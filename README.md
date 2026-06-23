local repo = 'https://raw.githubusercontent.com/mstudio45/LinoriaLib/main/'

local Library = loadstring(game:HttpGet(repo .. 'Library.lua'))()
local ThemeManager = loadstring(game:HttpGet(repo .. 'addons/ThemeManager.lua'))()
local SaveManager = loadstring(game:HttpGet(repo .. 'addons/SaveManager.lua'))()

local Options = Library.Options
local Toggles = Library.Toggles

Library.ShowToggleFrameInKeybinds = true
Library.ShowCustomCursor = true
Library.NotifySide = "Left"

local Window = Library:CreateWindow({
    Title = 'GUB.gg',
    Center = true,
    AutoShow = true,
    Resizable = true,
    TabPadding = 8,
    MenuFadeTime = 0.2
})

local Tabs = {
    Main = Window:AddTab('Main'),
    Combat = Window:AddTab('Ragebot'),
    Movement = Window:AddTab('Movement'),
    Esp = Window:AddTab('Visuals'),
    Misc = Window:AddTab('Misc'),
    ['UI Settings'] = Window:AddTab('UI Settings'),
}

-- 安全检查：只在支持的游戏中运行
local function IsSupportedGame()
    local gameId = game.PlaceId
    local supportedGames = {
        301549746, -- Criminality
        2788229376 -- Da Hood
    }
    for _, id in pairs(supportedGames) do
        if gameId == id then
            return true
        end
    end
    if game:GetService("Players") and game:GetService("ReplicatedStorage") then
        return true
    end
    return false
end

if not IsSupportedGame() then
    Library:Notify("This script may not be fully compatible with this game", 5)
end

-- 基础服务
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer or Players.LocalPlayer
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")

-- =============================================
-- Aimbot 系统
-- =============================================
local aimbotSettings = {
    Enabled = false,
    TeamCheck = false,
    FOV = 120,
    Smoothness = 5,
    BodyAim = "Head",
    WallCheck = true,
    ActivationKey = "MouseButton2",
    ShowFOV = true,
    FOVColor = Color3.fromRGB(0, 180, 180),
}

local fovCircle
local aimbotState = { currentTarget = nil, isActive = false }

local function isTeammate(player)
    if not player.Team or not LocalPlayer.Team then return false end
    return player.Team == LocalPlayer.Team
end

local function isVisible(camera, part)
    local origin = camera.CFrame.Position
    local dir = part.Position - origin
    local rp = RaycastParams.new()
    rp.FilterType = Enum.RaycastFilterType.Exclude
    local ex = {}
    if LocalPlayer.Character then table.insert(ex, LocalPlayer.Character) end
    if part.Parent then table.insert(ex, part.Parent) end
    rp.FilterDescendantsInstances = ex
    return Workspace:Raycast(origin, dir, rp) == nil
end

local function getAimPart(player)
    if not player.Character then return nil end
    local ba = aimbotSettings.BodyAim
    if ba == "Nearest" then
        local cam = Workspace.CurrentCamera
        local mousePos = UserInputService:GetMouseLocation()
        local best, bestD = nil, math.huge
        for _, p in ipairs(player.Character:GetChildren()) do
            if p:IsA("BasePart") then
                local s, v = cam:WorldToViewportPoint(p.Position)
                if v then
                    local d = (Vector2.new(s.X, s.Y) - mousePos).Magnitude
                    if d < bestD then
                        bestD = d
                        best = p
                    end
                end
            end
        end
        return best
    end
    local name = ba
    if ba == "Torso" then
        name = player.Character:FindFirstChild("UpperTorso") and "UpperTorso" or "Torso"
    end
    return player.Character:FindFirstChild(name)
end

local function getAimbotTarget(camera, mousePos)
    local best, bestD = nil, aimbotSettings.FOV
    for _, p in ipairs(Players:GetPlayers()) do
        if p == LocalPlayer then continue end
        if aimbotSettings.TeamCheck and isTeammate(p) then continue end
        if not p.Character then continue end
        local h = p.Character:FindFirstChildOfClass("Humanoid")
        if not h or h.Health <= 0 then continue end
        local ap = getAimPart(p)
        if not ap then continue end
        local s, v = camera:WorldToViewportPoint(ap.Position)
        if not v then continue end
        local d = (Vector2.new(s.X, s.Y) - mousePos).Magnitude
        if d > bestD then continue end
        if aimbotSettings.WallCheck and not isVisible(camera, ap) then continue end
        bestD = d
        best = p
    end
    return best
end

local function setupAimbot()
    fovCircle = Drawing.new("Circle")
    fovCircle.Visible = false
    fovCircle.Color = Color3.fromRGB(0, 180, 180)
    fovCircle.Thickness = 1
    fovCircle.Filled = false
    fovCircle.NumSides = 64
    fovCircle.Transparency = 0.8

    local keyMap = {
        MouseButton1 = Enum.UserInputType.MouseButton1,
        MouseButton2 = Enum.UserInputType.MouseButton2,
        MouseButton3 = Enum.UserInputType.MouseButton3,
        Q = Enum.KeyCode.Q, E = Enum.KeyCode.E, X = Enum.KeyCode.X,
        C = Enum.KeyCode.C, F = Enum.KeyCode.F, R = Enum.KeyCode.R,
        T = Enum.KeyCode.T, V = Enum.KeyCode.V, Z = Enum.KeyCode.Z,
    }
    local mouseKeys = {MouseButton1=true, MouseButton2=true, MouseButton3=true}

    UserInputService.InputBegan:Connect(function(input, gpe)
        if gpe then return end
        if not aimbotSettings.Enabled then return end
        local k = aimbotSettings.ActivationKey
        local m = keyMap[k]
        if not m then return end
        if mouseKeys[k] then
            if input.UserInputType == m then aimbotState.isActive = true end
        else
            if input.KeyCode == m then aimbotState.isActive = true end
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if not aimbotSettings.Enabled then return end
        local k = aimbotSettings.ActivationKey
        local m = keyMap[k]
        if not m then return end
        local released = false
        if mouseKeys[k] then
            released = input.UserInputType == m
        else
            released = input.KeyCode == m
        end
        if released then
            aimbotState.isActive = false
            aimbotState.currentTarget = nil
        end
    end)

    RunService.RenderStepped:Connect(function()
        local cam = Workspace.CurrentCamera
        local mousePos = UserInputService:GetMouseLocation()

        if aimbotSettings.Enabled and aimbotSettings.ShowFOV then
            fovCircle.Position = mousePos
            fovCircle.Radius = aimbotSettings.FOV
            fovCircle.Color = aimbotSettings.FOVColor
            fovCircle.Visible = true
        else
            fovCircle.Visible = false
        end

        if not aimbotSettings.Enabled or not aimbotState.isActive then return end

        local target = getAimbotTarget(cam, mousePos)
        if not target then
            aimbotState.currentTarget = nil
            return
        end
        aimbotState.currentTarget = target

        local aimPart = getAimPart(target)
        if not aimPart then return end

        local aimPos = aimPart.Position
        local smooth = math.max(aimbotSettings.Smoothness, 1)
        local alpha = 1 / smooth

        local sp = cam:WorldToViewportPoint(aimPos)
        local targetScreen = Vector2.new(sp.X, sp.Y)
        local diff = targetScreen - mousePos
        local moveBy = diff * alpha
        mousemoverel(moveBy.X, moveBy.Y)
    end)
end

-- =============================================
-- ESP 系统
-- =============================================
local cheats = {
    b_b = false;
    b_f = false;
    b_f_t = 1;
    b_sd = false;
    b_sn = false;
    b_sh = false;
    b_ht = "Bar";
    b_rt = false;
    b_tc = false;
    teamColor = Color3.new(1, 1, 1);
    enemyColor = Color3.new(1, 0, 0);
}

local playerFFEnabled = false
local playerFFColor = Color3.new(1, 1, 1)

local cheatsf = Instance.new("Folder", game.CoreGui) cheatsf.Name = "cheats"
local espf = Instance.new("Folder", cheatsf) espf.Name = "esp"

local function addEsp(player)
    local bbg = Instance.new("BillboardGui", espf)
    bbg.Name = player.Name
    bbg.AlwaysOnTop = true
    bbg.Size = UDim2.new(4,0,5.4,0)
    bbg.ClipsDescendants = false
    
    local outlines = Instance.new("Frame", bbg)
    outlines.Size = UDim2.new(1,0,1,0)
    outlines.BorderSizePixel = 0
    outlines.BackgroundTransparency = 1
    local left = Instance.new("Frame", outlines)
    left.BorderSizePixel = 0
    left.Size = UDim2.new(0,1,1,0)
    local right = left:Clone()
    right.Parent = outlines
    right.Size = UDim2.new(0,-1,1,0)
    right.Position = UDim2.new(1,0,0,0)
    local up = left:Clone()
    up.Parent = outlines
    up.Size = UDim2.new(1,0,0,1)
    local down = left:Clone()
    down.Parent = outlines
    down.Size = UDim2.new(1,0,0,-1)
    down.Position = UDim2.new(0,0,1,0)
    
    local info = Instance.new("BillboardGui", bbg)
    info.Name = "info"
    info.Size = UDim2.new(3,0,0,54)
    info.StudsOffset = Vector3.new(3.6,-3,0)
    info.AlwaysOnTop = true
    info.ClipsDescendants = false
    local namelabel = Instance.new("TextLabel", info)
    namelabel.Name = "namelabel"
    namelabel.BackgroundTransparency = 1
    namelabel.TextStrokeTransparency = 0
    namelabel.TextXAlignment = Enum.TextXAlignment.Left
    namelabel.Size = UDim2.new(0,100,0,18)
    namelabel.Position = UDim2.new(0,0,0,0)
    namelabel.Text = player.Name
    local distancel = Instance.new("TextLabel", info)
    distancel.Name = "distancelabel"
    distancel.BackgroundTransparency = 1
    distancel.TextStrokeTransparency = 0
    distancel.TextXAlignment = Enum.TextXAlignment.Left
    distancel.Size = UDim2.new(0,100,0,18)
    distancel.Position = UDim2.new(0,0,0,18)
    local healthl = Instance.new("TextLabel", info)
    healthl.Name = "healthlabel"
    healthl.BackgroundTransparency = 1
    healthl.TextStrokeTransparency = 0
    healthl.TextXAlignment = Enum.TextXAlignment.Left
    healthl.Size = UDim2.new(0,100,0,18)
    healthl.Position = UDim2.new(0,0,0,36)
    
    local uill = Instance.new("UIListLayout", info)
    
    local forhealth = Instance.new("BillboardGui", bbg)
    forhealth.Name = "forhealth"
    forhealth.Size = UDim2.new(5,0,6,0)
    forhealth.AlwaysOnTop = true
    forhealth.ClipsDescendants = false
    
    local healthbar = Instance.new("Frame", forhealth)
    healthbar.Name = "healthbar"
    healthbar.BackgroundColor3 = Color3.fromRGB(40,40,40)
    healthbar.BorderColor3 = Color3.fromRGB(0,0,0)
    healthbar.Size = UDim2.new(0.04,0,0.9,0)
    healthbar.Position = UDim2.new(0,0,0.05,0)
    local bar = Instance.new("Frame", healthbar)
    bar.Name = "bar"
    bar.BorderSizePixel = 0
    bar.BackgroundColor3 = Color3.fromRGB(94,255,69)
    bar.AnchorPoint = Vector2.new(0,1)
    bar.Position = UDim2.new(0,0,1,0)
    bar.Size = UDim2.new(1,0,1,0)
    
    local co = coroutine.create(function()
        while wait(0.1) do
            if (player.Character and player.Character:FindFirstChild"HumanoidRootPart") then
                bbg.Adornee = player.Character.HumanoidRootPart
                info.Adornee = player.Character.HumanoidRootPart
                forhealth.Adornee = player.Character.HumanoidRootPart
                
                if (player.Team ~= LocalPlayer.Team) then
                    bbg.Enabled = true
                    info.Enabled = true
                    forhealth.Enabled = true
                end
                if player.Character:FindFirstChild("ForceField") then
                    outlines.BackgroundTransparency = 0.4
                    left.BackgroundTransparency = 0.4
                    right.BackgroundTransparency = 0.4
                    up.BackgroundTransparency = 0.4
                    down.BackgroundTransparency = 0.4
                    healthl.TextTransparency = 0.4
                    healthl.TextStrokeTransparency = 0.8
                    distancel.TextTransparency = 0.4
                    distancel.TextStrokeTransparency = 0.8
                    namelabel.TextTransparency = 0.4
                    namelabel.TextStrokeTransparency = 0.8
                    bar.BackgroundTransparency = 0.4
                    healthbar.BackgroundTransparency = 0.8
                else
                    outlines.BackgroundTransparency = 0
                    left.BackgroundTransparency = 0
                    right.BackgroundTransparency = 0
                    up.BackgroundTransparency = 0
                    down.BackgroundTransparency = 0
                    healthl.TextTransparency = 0
                    healthl.TextStrokeTransparency = 0
                    distancel.TextTransparency = 0
                    distancel.TextStrokeTransparency = 0
                    namelabel.TextTransparency = 0
                    namelabel.TextStrokeTransparency = 0
                    bar.BackgroundTransparency = 0
                    healthbar.BackgroundTransparency = 0
                end
                if cheats.b_b == true then
                    outlines.Visible = true
                else
                    outlines.Visible = false
                end
                if cheats.b_f == true then
                    if player.Character:FindFirstChild("ForceField") then
                        outlines.BackgroundTransparency = 0.9
                    else
                        outlines.BackgroundTransparency = cheats.b_f_t
                    end
                else
                    outlines.BackgroundTransparency = 1
                end
                if cheats.b_sh == true then
                    if (player.Character:FindFirstChild"Humanoid") then
                        healthl.Text = "Health: "..math.floor(player.Character:FindFirstChild"Humanoid".Health)
                        healthbar.bar.Size = UDim2.new(1,0,player.Character:FindFirstChild"Humanoid".Health/player.Character:FindFirstChild"Humanoid".MaxHealth,0)
                    end
                    if cheats.b_ht == "Bar" then
                        healthbar.Visible = true
                        healthl.Visible = false
                    elseif cheats.b_ht == "Text" then
                        healthbar.Visible = false
                        healthl.Visible = true
                    elseif cheats.b_ht == "Both" then
                        healthl.Visible = true
                        healthbar.Visible = true
                    end
                else
                    healthl.Visible = false
                    healthbar.Visible = false
                end
                if cheats.b_sn then
                    namelabel.Visible = true
                else
                    namelabel.Visible = false
                end
                if cheats.b_sd == true then
                    distancel.Visible = true
                    if (LocalPlayer.Character and LocalPlayer.Character:FindFirstChild"HumanoidRootPart") then
                        distancel.Text = "Distance: "..math.floor(0.5+(LocalPlayer.Character:FindFirstChild"HumanoidRootPart".Position - player.Character:FindFirstChild"HumanoidRootPart".Position).magnitude)
                    end
                else
                    distancel.Visible = false
                end
                if cheats.b_rt == true then
                    if (player.Team == LocalPlayer.Team) then
                        bbg.Enabled = true
                        info.Enabled = true
                        forhealth.Enabled = true
                    end
                else
                    if (player.Team == LocalPlayer.Team) then
                        bbg.Enabled = false
                        info.Enabled = false
                        forhealth.Enabled = false
                    end
                end
                if cheats.b_tc == true then
                    outlines.BackgroundColor3 = player.TeamColor.Color
                    left.BackgroundColor3 = player.TeamColor.Color
                    right.BackgroundColor3 = player.TeamColor.Color
                    up.BackgroundColor3 = player.TeamColor.Color
                    down.BackgroundColor3 = player.TeamColor.Color
                    healthl.TextColor3 = player.TeamColor.Color
                    distancel.TextColor3 = player.TeamColor.Color
                    namelabel.TextColor3 = player.TeamColor.Color
                else
                    local colorToUse = cheats.teamColor
                    if player.Team ~= LocalPlayer.Team then
                        colorToUse = cheats.enemyColor
                    end
                    outlines.BackgroundColor3 = colorToUse
                    left.BackgroundColor3 = colorToUse
                    right.BackgroundColor3 = colorToUse
                    up.BackgroundColor3 = colorToUse
                    down.BackgroundColor3 = colorToUse
                    healthl.TextColor3 = colorToUse
                    distancel.TextColor3 = colorToUse
                    namelabel.TextColor3 = colorToUse
                end
            end
            if not (Players:FindFirstChild(player.Name)) then
                print(player.Name.." has left. Clearing esp.")
                espf:FindFirstChild(player.Name):Destroy()
                coroutine.yield()
            end
        end
    end)
    coroutine.resume(co)
end

local function applyPlayerForceField()
    local character = LocalPlayer.Character
    if not character then return end
    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            if playerFFEnabled then
                part.Material = Enum.Material.ForceField
                part.Color = playerFFColor
            end
        end
    end
end
RunService.Heartbeat:Connect(applyPlayerForceField)

local function initESP()
    for _, v in pairs(Players:GetPlayers()) do
        if v ~= LocalPlayer and not espf:FindFirstChild(v.Name) then
            addEsp(v)
        end
    end
    task.spawn(function()
        while true do
            wait(10)
            for _, v in pairs(Players:GetPlayers()) do
                if v ~= LocalPlayer and not espf:FindFirstChild(v.Name) then
                    addEsp(v)
                end
            end
        end
    end)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.KeyCode == Enum.KeyCode.KeypadOne and not gameProcessed then
        Window.ScreenGui.Enabled = not Window.ScreenGui.Enabled
    end
end)

-- =============================================
-- Visuals 标签页 UI
-- =============================================
local visualsTab = Tabs.Esp
local leftESP = visualsTab:AddLeftGroupbox('ESP')

leftESP:AddToggle('b_rt', {
    Text = 'Render team',
    Default = false,
    Callback = function(v) cheats.b_rt = v end
})
leftESP:AddToggle('b_b', {
    Text = 'Bounding box',
    Default = false,
    Callback = function(v) cheats.b_b = v end
})
leftESP:AddToggle('b_f', {
    Text = 'Fill alpha',
    Default = false,
    Callback = function(v) cheats.b_f = v end
})
leftESP:AddSlider('b_f_t', {
    Text = 'Fill transparency',
    Default = 1,
    Min = 0,
    Max = 1,
    Rounding = 2,
    Suffix = '',
    Callback = function(v) cheats.b_f_t = v end
})
leftESP:AddDivider()
leftESP:AddToggle('b_sd', {
    Text = 'Show distance',
    Default = false,
    Callback = function(v) cheats.b_sd = v end
})
leftESP:AddToggle('b_sn', {
    Text = 'Show name',
    Default = false,
    Callback = function(v) cheats.b_sn = v end
})
leftESP:AddToggle('b_sh', {
    Text = 'Show health',
    Default = false,
    Callback = function(v) cheats.b_sh = v end
})
leftESP:AddDropdown('b_ht', {
    Text = 'Health type',
    Values = {'Text', 'Bar', 'Both'},
    Default = 'Bar',
    Callback = function(v) cheats.b_ht = v end
})

local rightPlayer = visualsTab:AddRightGroupbox('Player')
rightPlayer:AddToggle('ForceFieldEnabled', {
    Text = 'ForceField BasePart',
    Default = false,
    Callback = function(v) playerFFEnabled = v end
})
rightPlayer:AddLabel('Color'):AddColorPicker('ForceFieldColor', {
    Default = playerFFColor,
    Callback = function(c) playerFFColor = c end
})

getgenv().forceFieldColor = Color3.fromRGB(255, 255, 255)
getgenv().forceFieldToggle = false

rightPlayer:AddToggle('ForceFieldToggle', {
    Text = 'ForceField ViewModel',
    Default = false,
    Tooltip = 'Apply ForceField material to ViewModel arms',
    Callback = function(value)
        getgenv().forceFieldToggle = value
        if value then
            task.spawn(function()
                while getgenv().forceFieldToggle do
                    local camera = workspace.CurrentCamera
                    if camera then
                        local viewModel = camera:FindFirstChild("ViewModel")
                        if viewModel then
                            for _, child in pairs(viewModel:GetChildren()) do
                                if child:IsA("BasePart") and (child.Name:find("Arm") or child.Name:find("Hand")) then
                                    child.Material = Enum.Material.ForceField
                                    child.Color = getgenv().forceFieldColor
                                end
                            end
                        end
                    end
                    task.wait(0.5)
                end
            end)
        end
    end
})
rightPlayer:AddLabel('ForceField Color'):AddColorPicker('ForceFieldColor', {
    Default = Color3.new(1, 1, 1),
    Title = 'ForceField Color',
    Callback = function(Value)
        getgenv().forceFieldColor = Value
    end
})

local rightAlt = visualsTab:AddRightGroupbox('Alt Features')
rightAlt:AddToggle('TPkillToggle', {
    Text = 'TPkill',
    Default = false,
    Tooltip = 'Teleport to nearby players',
    Callback = function(value)
        if getgenv().tpKillLoop then
            getgenv().tpKillLoop:Disconnect()
            getgenv().tpKillLoop = nil
        end
        if value then
            getgenv().tpKillLoop = RunService.RenderStepped:Connect(function()
                local char = LocalPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                if not hrp then return end
                for _, player in pairs(Players:GetPlayers()) do
                    if player ~= LocalPlayer and player.Character then
                        local target = player.Character:FindFirstChild("HumanoidRootPart")
                        if target and (hrp.Position - target.Position).Magnitude <= 23 then
                            hrp.CFrame = target.CFrame + Vector3.new(0, 2, 0)
                            break
                        end
                    end
                end
            end)
        end
    end
})

local Camera = workspace.CurrentCamera
local originalFOV = Camera.FieldOfView or 70
getgenv().FOVChangerEnabled = false
getgenv().DesiredFOV = originalFOV

RunService.RenderStepped:Connect(function()
    if getgenv().FOVChangerEnabled and Camera then
        Camera.FieldOfView = getgenv().DesiredFOV
    elseif Camera then
        Camera.FieldOfView = originalFOV
    end
end)

rightAlt:AddToggle('FOVChangerToggle', {
    Text = 'FOV Changer',
    Default = false,
    Tooltip = 'Toggle custom Field of View',
    Callback = function(enabled)
        getgenv().FOVChangerEnabled = enabled
    end
})
rightAlt:AddSlider('FOVSlider', {
    Text = 'FOV Value',
    Default = 90,
    Min = 30,
    Max = 120,
    Rounding = 1,
    Tooltip = 'Adjust the camera FOV',
    Callback = function(val)
        getgenv().DesiredFOV = val
        if getgenv().FOVChangerEnabled then
            Camera.FieldOfView = val
        end
    end
})
rightAlt:AddButton({
    Text = 'Remove Camshake',
    Func = function()
        local char = LocalPlayer.Character
        if char then
            for _, part in pairs(char:GetDescendants()) do
                if part.Name:find("Shake") or part.Name:find("Cam") then
                    part:Destroy()
                end
            end
            Library:Notify("Camera shake removed if present")
        end
    end,
    Tooltip = 'Remove camera shake effects',
})
rightAlt:AddButton({
    Text = 'Hide Head',
    Func = function()
        local char = LocalPlayer.Character
        if char then
            local head = char:FindFirstChild("Head")
            if head then
                head.Transparency = 1
                for _, decal in pairs(head:GetChildren()) do
                    if decal:IsA("Decal") then
                        decal.Transparency = 1
                    end
                end
            end
            Library:Notify("Head hidden")
        end
    end,
    Tooltip = 'Hide your character\'s head',
})

-- =============================================
-- Main 标签页
-- =============================================
local MainLeft = Tabs.Main:AddLeftGroupbox('Combat')
MainLeft:AddToggle('AimbotEnabled', {
    Text = 'Enabled',
    Default = false,
    Callback = function(v) aimbotSettings.Enabled = v end
})
MainLeft:AddSlider('AimbotFOV', {
    Text = 'FOV',
    Default = 120,
    Min = 10,
    Max = 500,
    Rounding = 0,
    Suffix = 'px',
    Callback = function(v) aimbotSettings.FOV = v end
})
MainLeft:AddToggle('ShowFOVCircle', {
    Text = 'Show FOV Circle',
    Default = true,
    Callback = function(v) aimbotSettings.ShowFOV = v end
})
MainLeft:AddSlider('Smoothness', {
    Text = 'Smoothness',
    Default = 5,
    Min = 1,
    Max = 50,
    Rounding = 0,
    Callback = function(v) aimbotSettings.Smoothness = v end
})
MainLeft:AddDropdown('ActivationKey', {
    Text = 'Activation Key',
    Values = {'MouseButton1','MouseButton2','MouseButton3','Q','E','X','C','F','R','T','V','Z'},
    Default = 'MouseButton2',
    Callback = function(v) aimbotSettings.ActivationKey = v end
})
MainLeft:AddLabel('FOV Color'):AddColorPicker('FOVColor', {
    Default = aimbotSettings.FOVColor,
    Callback = function(c) aimbotSettings.FOVColor = c end
})

local MainLeftOpt = Tabs.Main:AddLeftGroupbox('Aimbot Options')
MainLeftOpt:AddDropdown('BodyAim', {
    Text = 'Body Aim',
    Values = {'Head','Torso','HumanoidRootPart','Nearest'},
    Default = 'Head',
    Callback = function(v) aimbotSettings.BodyAim = v end
})
MainLeftOpt:AddToggle('WallCheck', {
    Text = 'Wall Check',
    Default = true,
    Callback = function(v) aimbotSettings.WallCheck = v end
})
MainLeftOpt:AddToggle('TeamCheckAim', {
    Text = 'Team Check',
    Default = false,
    Callback = function(v) aimbotSettings.TeamCheck = v end
})

-- =============================================
-- Movement 标签页 (原 Misc 的 Player Features 移至此处)
-- =============================================
local MovementGroup = Tabs.Movement:AddLeftGroupbox('Player Features')

-- 体力、速度、旋转相关功能定义 (保持原逻辑)
local staminaLoop
local StaminaTbl = {}
local function UpdateStaminaTables()
    table.clear(StaminaTbl)
    for _, v in pairs(getgc(true)) do
        if type(v) == "table" and rawget(v, "S") then
            table.insert(StaminaTbl, v)
        end
    end
end
local function SetupInfiniteStamina()
    UpdateStaminaTables()
    if staminaLoop then staminaLoop:Disconnect() end
    staminaLoop = RunService.RenderStepped:Connect(function()
        if Toggles.InfiniteStaminaToggle.Value then
            for _, tbl in pairs(StaminaTbl) do
                tbl.S = 100
            end
        end
    end)
end
if LocalPlayer.Character then
    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(1)
        if Toggles.InfiniteStaminaToggle and Toggles.InfiniteStaminaToggle.Value then
            SetupInfiniteStamina()
        end
    end)
end

local DesiredSpeed = 35
local OriginalWalkspeed = 16
local _Humanoid
local function ApplyWalkSpeed()
    if not LocalPlayer.Character then return end
    local humanoid = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
    if humanoid then
        _Humanoid = humanoid
        humanoid.WalkSpeed = DesiredSpeed
    end
end
local function RemoveWalkSpeed()
    if _Humanoid then _Humanoid.WalkSpeed = OriginalWalkspeed end
end
local WalkSpeedConnection
local function SetupWalkSpeed()
    if WalkSpeedConnection then WalkSpeedConnection:Disconnect() end
    WalkSpeedConnection = RunService.Heartbeat:Connect(function()
        if LocalPlayer.Character and getgenv().WalkSpeedToggle then
            ApplyWalkSpeed()
        end
    end)
end
getgenv().WalkSpeedToggle = false

local spinEnabled = false
local spinSpeed = 100
local spinLoop
local function spinCharacter()
    if not LocalPlayer.Character then return end
    local root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if root then
        root.CFrame = root.CFrame * CFrame.Angles(0, math.rad(spinSpeed), 0)
    end
end
local function startSpin()
    if spinLoop then spinLoop:Disconnect() end
    spinLoop = RunService.RenderStepped:Connect(function()
        if spinEnabled and LocalPlayer.Character then spinCharacter() end
    end)
end
local function stopSpin()
    if spinLoop then spinLoop:Disconnect() spinLoop = nil end
end
if LocalPlayer.Character then
    LocalPlayer.CharacterAdded:Connect(function()
        if spinEnabled then startSpin() end
    end)
end

-- 添加控件到 Movement 标签页
MovementGroup:AddToggle('InfiniteStaminaToggle', {
    Text = 'Infinite Stamina',
    Default = false,
    Tooltip = 'Prevents stamina from decreasing.',
    Callback = function(Value)
        if Value then
            SetupInfiniteStamina()
        elseif staminaLoop then
            staminaLoop:Disconnect()
            staminaLoop = nil
        end
    end
})
MovementGroup:AddToggle('WalkSpeedToggle', {
    Text = 'Walk Speed',
    Default = false,
    Callback = function(enabled)
        getgenv().WalkSpeedToggle = enabled
        if enabled then
            SetupWalkSpeed()
            ApplyWalkSpeed()
        else
            RemoveWalkSpeed()
        end
    end,
    Tooltip = 'Toggle Walk Speed Bypass',
})
MovementGroup:AddSlider('WalkSpeedSlider', {
    Text = 'Walk Speed Value',
    Default = 35,
    Min = 0,
    Max = 100,
    Rounding = 1,
    Callback = function(val)
        DesiredSpeed = val
        if getgenv().WalkSpeedToggle then ApplyWalkSpeed() end
    end,
    Tooltip = 'Adjust your walk speed',
})
MovementGroup:AddToggle('SpinbotToggle', {
    Text = 'Enable Spinbot',
    Default = false,
    Tooltip = 'Toggle spinbot on/off',
    Callback = function(value)
        spinEnabled = value
        if value then startSpin() else stopSpin() end
    end
})
MovementGroup:AddSlider('SpinbotSpeed', {
    Text = 'Spinbot Speed',
    Default = spinSpeed,
    Min = 0,
    Max = 500,
    Rounding = 1,
    Tooltip = 'Adjust the spin speed',
    Callback = function(value)
        spinSpeed = math.max(value, 0)
    end
})

-- =============================================
-- Misc 标签页 (原 Player Features 已移除)
-- =============================================
local MiscLeft = Tabs.Misc:AddLeftGroupbox('Misc Features')

local teleportLoop = false
local teleportTarget = nil
local function updatePlayerList()
    local playerNames = {}
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            table.insert(playerNames, player.Name)
        end
    end
    return playerNames
end

MiscLeft:AddDropdown('TeleportPlayerList', {
    Values = updatePlayerList(),
    Default = 1,
    Multi = false,
    Text = 'Select Player',
    Tooltip = 'Choose player to teleport to',
    Callback = function(value) teleportTarget = value end
})
MiscLeft:AddToggle('TeleportToPlayerToggle', {
    Text = 'Teleport to Player',
    Default = false,
    Tooltip = 'Teleport to selected player',
    Callback = function(enabled)
        teleportLoop = enabled
        if enabled then
            spawn(function()
                while teleportLoop and teleportTarget do
                    local targetPlayer = Players:FindFirstChild(teleportTarget)
                    if targetPlayer and targetPlayer.Character then
                        local targetHrp = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
                        local myChar = LocalPlayer.Character
                        local myHrp = myChar and myChar:FindFirstChild("HumanoidRootPart")
                        if targetHrp and myHrp then
                            myHrp.CFrame = targetHrp.CFrame + Vector3.new(0, 5, 0)
                        end
                    end
                    wait(1)
                end
            end)
        end
    end
})
MiscLeft:AddButton({
    Text = 'Refresh Player List',
    Func = function()
        Options.TeleportPlayerList:SetValues(updatePlayerList())
    end,
    Tooltip = 'Refresh the player list'
})

local noclipEnabled = false
local noclipConnection
MiscLeft:AddToggle('NoClipToggle', {
    Text = 'NoClip',
    Default = false,
    Tooltip = 'Walk through walls',
    Callback = function(enabled)
        noclipEnabled = enabled
        if noclipConnection then noclipConnection:Disconnect() noclipConnection = nil end
        if enabled then
            noclipConnection = RunService.Stepped:Connect(function()
                if LocalPlayer.Character then
                    for _, part in pairs(LocalPlayer.Character:GetDescendants()) do
                        if part:IsA("BasePart") then part.CanCollide = false end
                    end
                end
            end)
        end
    end
})

local autoRespawnEnabled = false
local autoRespawnConnection
MiscLeft:AddToggle('AutoRespawnToggle', {
    Text = 'Auto Respawn',
    Default = false,
    Tooltip = 'Automatically respawn when dead',
    Callback = function(enabled)
        autoRespawnEnabled = enabled
        if autoRespawnConnection then autoRespawnConnection:Disconnect() autoRespawnConnection = nil end
        if enabled then
            autoRespawnConnection = RunService.Heartbeat:Connect(function()
                local char = LocalPlayer.Character
                if char then
                    local humanoid = char:FindFirstChildOfClass("Humanoid")
                    if humanoid and humanoid.Health <= 0 then
                        LocalPlayer:LoadCharacter()
                    end
                end
            end)
        end
    end
})

-- =============================================
-- UI Settings
-- =============================================
local MenuGroup = Tabs['UI Settings']:AddLeftGroupbox('Menu')
MenuGroup:AddButton('Unload', function() Library:Unload() end)
MenuGroup:AddLabel('Menu bind'):AddKeyPicker('MenuKeybind', {
    Default = 'RightShift',
    NoUI = true,
    Text = 'Menu keybind'
})
Library.ToggleKeybind = Options.MenuKeybind

ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)
ThemeManager:SetFolder('UndetectedWare')
SaveManager:SetFolder('UndetectedWare')
SaveManager:BuildConfigSection(Tabs['UI Settings'])
ThemeManager:ApplyToTab(Tabs['UI Settings'])

setupAimbot()
initESP()
Library:Notify("UndetectedWare Loaded Successfully", 5)
