loadstring(game:HttpGet("https://raw.githubusercontent.com/Pixeluted/adoniscries/main/Source.lua", true))()

if not game:IsLoaded() then
    game.Loaded:Wait()
end

if not syn or not protectgui then
    getgenv().protectgui = function() end
end

local SilentAimSettings = {
    Enabled = false,
    ClassName = "Universal Silent Aim",
    ToggleKey = "RightAlt",
    TeamCheck = false,
    VisibleCheck = false,
    TargetPart = "HumanoidRootPart",
    SilentAimMethod = "Raycast",
    FOVRadius = 130,
    FOVVisible = true,
    ShowSilentAimTarget = false,
    MouseHitPrediction = false,
    MouseHitPredictionAmount = 0.165,
    HitChance = 100,
    HeadshotChanceEnabled = false,
    HeadshotChance = 0,
    FixedFOV = true,
    TargetIndicatorRadius = 20,
    CrosshairLength = 30,
    CrosshairGap = 5,
    IndicatorRotationEnabled = false,
    IndicatorRotationSpeed = 1,
    IndicatorRainbowEnabled = false,
    IndicatorRainbowSpeed = 1,
    MaxDistance = 500,
    PriorityMode = "准星最近",
    TargetInfoStyle = "面板",
    ShowTargetName = true,
    ShowTargetHealth = true,
    ShowTargetDistance = true,
    ShowTargetCategory = false,
    ShowDamageNotifier = false,
    HighlightEnabled = false,
    HighlightRainbowEnabled = false,
    HighlightColor = Color3.fromRGB(255, 255, 0),
    IndependentPanelPosition = "200,200",
    IndependentPanelPinned = false,
    LeakAndHitMode = false,
    Wallbang = false,
    EnableNameTargeting = false,
    WhitelistedNames = {},
    BlacklistedNames = {},
    ShowTracer = false,
    Tracer_Y_Offset = 0,
    WhitelistPath = {},
    IndicatorBreathingEnabled = true,
    IndicatorBreathingSpeed = 1,
    IndicatorBreathingMin = 0.8,
    IndicatorBreathingMax = 1.2,
    ThreeLineCrosshairEnabled = true,
    ThreeLineCrosshairLength = 30,
    ThreeLineCrosshairGap = 5
}

getgenv().SilentAimSettings = SilentAimSettings
local MainFileName = "UniversalSilentAim"

local Camera = workspace.CurrentCamera
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local GuiService = game:GetService("GuiService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local Debris = game:GetService("Debris")
local CoreGui = game:GetService("CoreGui")
local PathfindingService = game:GetService("PathfindingService")

local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

local GetPlayers = Players.GetPlayers
local WorldToViewportPoint = Camera.WorldToViewportPoint
local FindFirstChild = game.FindFirstChild
local RenderStepped = RunService.RenderStepped
local GetMouseLocation = UserInputService.GetMouseLocation

local resume = coroutine.resume
local create = coroutine.create

local ValidTargetParts = {"Head", "HumanoidRootPart"}
local PredictionAmount = 0.165

local currentTargetPart = nil
local currentHighlight = nil
local currentRotationAngle = 0
local currentIndicatorHue = 0
local npcList = {}
local targetMap = {}
local avatarCache = {}
local recentShots = {}
local pendingDamage = {}

local lockedTargetObject = nil

local target_indicator_circle = Drawing.new("Circle")
target_indicator_circle.Visible = false; target_indicator_circle.ZIndex = 1000; target_indicator_circle.Thickness = 2; target_indicator_circle.Filled = false
local target_indicator_lines = {}
for i = 1, 5 do local line = Drawing.new("Line"); line.Visible = false; line.ZIndex = 1000; line.Thickness = 2; table.insert(target_indicator_lines, line) end
local tracer_line = Drawing.new("Line")
tracer_line.Visible = false; tracer_line.ZIndex = 998; tracer_line.Color = Color3.fromRGB(255, 255, 0); tracer_line.Thickness = 1; tracer_line.Transparency = 1

local overhead_info_texts = {
    Name = Drawing.new("Text"),
    Health = Drawing.new("Text"),
    Distance = Drawing.new("Text"),
    Category = Drawing.new("Text")
}
for _, text in pairs(overhead_info_texts) do
    text.Visible = false; text.ZIndex = 1001; text.Font = Drawing.Fonts.Plex; text.Size = 14; text.Color = Color3.fromRGB(255, 255, 255); text.Center = true; text.Outline = true
end

local panel_info_bg = Drawing.new("Square")
panel_info_bg.Visible = false; panel_info_bg.ZIndex = 1002; panel_info_bg.Color = Color3.fromRGB(0, 0, 0); panel_info_bg.Thickness = 0; panel_info_bg.Filled = true; panel_info_bg.Transparency = 0.5
local panel_info_texts = {
    Name = Drawing.new("Text"),
    Health = Drawing.new("Text"),
    Distance = Drawing.new("Text"),
    Category = Drawing.new("Text")
}
for _, text in pairs(panel_info_texts) do
    text.Visible = false; text.ZIndex = 1003; text.Font = Drawing.Fonts.Plex; text.Size = 14; text.Color = Color3.fromRGB(255, 255, 255); text.Center = false; text.Outline = true
end

local FOVCircleGui = Instance.new("ScreenGui", LocalPlayer:WaitForChild("PlayerGui"))
FOVCircleGui.Name = "FOVCircleGui"; FOVCircleGui.ResetOnSpawn = false; FOVCircleGui.IgnoreGuiInset = true; FOVCircleGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
local FOVCircleFrame = Instance.new("Frame", FOVCircleGui)
FOVCircleFrame.Name = "FOVCircleFrame"; FOVCircleFrame.AnchorPoint = Vector2.new(0.5, 0.5); FOVCircleFrame.Position = UDim2.fromScale(0.5, 0.5); FOVCircleFrame.BackgroundTransparency = 1
local FOVStroke = Instance.new("UIStroke", FOVCircleFrame)
FOVStroke.Name = "FOVStroke"; FOVStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border; FOVStroke.Thickness = 1; FOVStroke.Transparency = 0.5
local FOVCorner = Instance.new("UICorner", FOVCircleFrame)
FOVCorner.Name = "FOVCorner"; FOVCorner.CornerRadius = UDim.new(1, 0)

local IndependentPanelGui = Instance.new("ScreenGui", LocalPlayer:WaitForChild("PlayerGui"))
IndependentPanelGui.Name = "IndependentPanelGui"; IndependentPanelGui.ResetOnSpawn = false; IndependentPanelGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
local IndependentPanelFrame = Instance.new("Frame", IndependentPanelGui)
IndependentPanelFrame.Name = "PanelFrame"; IndependentPanelFrame.Size = UDim2.fromOffset(160, 100);
IndependentPanelFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30); IndependentPanelFrame.BackgroundTransparency = 0.3; IndependentPanelFrame.BorderSizePixel = 1; IndependentPanelFrame.BorderColor3 = Color3.new(1,1,1)
IndependentPanelFrame.Visible = false; IndependentPanelFrame.Active = true
local IPCorner = Instance.new("UICorner", IndependentPanelFrame); IPCorner.CornerRadius = UDim.new(0, 4)
local IPListLayout = Instance.new("UIListLayout", IndependentPanelFrame)
IPListLayout.Padding = UDim.new(0, 5); IPListLayout.SortOrder = Enum.SortOrder.LayoutOrder; IPListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center; IPListLayout.VerticalAlignment = Enum.VerticalAlignment.Center

local independent_panel_texts = {}
for i, name in ipairs({"Name", "Health", "Distance", "Category"}) do
    local label = Instance.new("TextLabel", IndependentPanelFrame)
    label.Name = name; label.Size = UDim2.new(1, -10, 0, 15); label.BackgroundTransparency = 1
    label.Font = Enum.Font.SourceSans; label.TextSize = 14; label.TextColor3 = Color3.new(1,1,1); label.TextXAlignment = Enum.TextXAlignment.Left; label.LayoutOrder = i
    independent_panel_texts[name] = label
end
IndependentPanelFrame.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 and IndependentPanelFrame.Draggable then IndependentPanelFrame.Position = UDim2.fromOffset(UserInputService:GetMouseLocation().X, UserInputService:GetMouseLocation().Y) end end)
IndependentPanelFrame.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 and IndependentPanelFrame.Draggable then SilentAimSettings.IndependentPanelPosition = IndependentPanelFrame.Position.X.Offset .. "," .. IndependentPanelFrame.Position.Y.Offset end end)

local ExpectedArguments = {
    FindPartOnRayWithIgnoreList = { ArgCountRequired = 3, Args = {"Instance", "Ray", "table", "boolean", "boolean"} },
    FindPartOnRayWithWhitelist = { ArgCountRequired = 3, Args = {"Instance", "Ray", "table", "boolean"} },
    FindPartOnRay = { ArgCountRequired = 2, Args = {"Instance", "Ray", "Instance", "boolean", "boolean"} },
    Raycast = { ArgCountRequired = 3, Args = {"Instance", "Vector3", "Vector3", "RaycastParams"} }
}

local HitSounds = {
    ["bell"] = "rbxassetid://8679627751",
    ["metal"] = "rbxassetid://3125624765",
    ["click"] = "rbxassetid://17755696142",
    ["exp"] = "rbxassetid://10070796384"
}

local rainbowColor = Color3.fromHSV(0, 1, 1)
task.spawn(function()
    while task.wait() do
        if Library and Library.Unloaded then break end
        local hue = (tick() % 6) / 6
        rainbowColor = Color3.fromHSV(hue, 1, 1)
    end
end)

local function playHitSound(soundId)
    local sound = Instance.new("Sound")
    sound.Parent = CoreGui
    sound.SoundId = soundId
    sound.Volume = 0.6
    sound:Play()
    Debris:AddItem(sound, sound.TimeLength + 0.2)
end

function CalculateChance(Percentage)
    Percentage = math.floor(Percentage)
    return math.random() <= Percentage / 100
end

do
    if not isfolder(MainFileName) then makefolder(MainFileName) end
    if not isfolder(string.format("%s/%s", MainFileName, tostring(game.PlaceId))) then makefolder(string.format("%s/%s", MainFileName, tostring(game.PlaceId))) end
end

local function getPositionOnScreen(Vector)
    local Vec3, OnScreen = WorldToViewportPoint(Camera, Vector)
    return Vector2.new(Vec3.X, Vec3.Y), OnScreen
end

local function ValidateArguments(Args, RayMethod)
    local Matches = 0
    if #Args < RayMethod.ArgCountRequired then return false end
    for Pos, Argument in next, Args do if typeof(Argument) == RayMethod.Args[Pos] then Matches = Matches + 1 end end
    return Matches >= RayMethod.ArgCountRequired
end

local function getDirection(Origin, Position)
    return (Position - Origin).Unit * 1000
end

local function isNPC(obj)
    return obj:IsA("Model") and obj:FindFirstChild("Humanoid") and obj.Humanoid.Health > 0 and obj:FindFirstChild("HumanoidRootPart") and not Players:GetPlayerFromCharacter(obj)
end

function getTargetCategory(character)
    if not character then return "无" end

    if Players:GetPlayerFromCharacter(character) then
        return "玩家"
    end

    if SilentAimSettings.EnableNameTargeting then
        local name = character.Name:lower()
        for _, whitelistedName in ipairs(SilentAimSettings.WhitelistedNames) do
            if whitelistedName and whitelistedName ~= "" and string.find(name, whitelistedName:lower(), 1, true) then
                return "添加的"
            end
        end
    end
    
    for _, path in ipairs(SilentAimSettings.WhitelistPath) do
        local obj = workspace:FindFirstChild(path)
        if obj and obj == character then
            return "路径白名单"
        end
    end
    
    if character:FindFirstChild("Humanoid") then
         return "NPC"
    end

    return "未知"
end

local function updateNPCs()
    local newNpcList = {}
    local addedNpcs = {}

    if SilentAimSettings.EnableNameTargeting and #SilentAimSettings.WhitelistedNames > 0 then
        for _, model in ipairs(workspace:GetDescendants()) do
            if isNPC(model) then
                for _, substring in ipairs(SilentAimSettings.WhitelistedNames) do
                    if substring and substring ~= "" and string.find(model.Name:lower(), substring:lower(), 1, true) then
                        if not addedNpcs[model] then
                            table.insert(newNpcList, model)
                            addedNpcs[model] = true
                            break
                        end
                    end
                end
            end
        end
    end

    for _, path in ipairs(SilentAimSettings.WhitelistPath) do
        local obj = workspace:FindFirstChild(path)
        if obj and isNPC(obj) and not addedNpcs[obj] then
            table.insert(newNpcList, obj)
            addedNpcs[obj] = true
        end
    end

    for _, v in ipairs(workspace:GetChildren()) do
        if isNPC(v) then
            if not addedNpcs[v] then
                table.insert(newNpcList, v)
                addedNpcs[v] = true
            end
        end
    end
    
    npcList = newNpcList
end

local function isBlacklisted(name)
    local lowerName = name:lower()
    for _, blacklistedName in ipairs(SilentAimSettings.BlacklistedNames) do
        if blacklistedName:lower() == lowerName then
            return true
        end
    end
    return false
end

local function isPartVisible(part, customOrigin)
    if not part then return false end
    local localCharacter = LocalPlayer.Character
    if not localCharacter then return false end
    local origin = customOrigin or Camera.CFrame.Position
    local direction = part.Position - origin
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    raycastParams.FilterDescendantsInstances = {localCharacter, part.Parent}
    local raycastResult = workspace:Raycast(origin, direction.Unit * direction.Magnitude, raycastParams)
    return not raycastResult
end

local function getClosestPlayer()
    local LocalPlayerCharacter = LocalPlayer.Character
    if not LocalPlayerCharacter or not LocalPlayerCharacter:FindFirstChild("HumanoidRootPart") then return nil end
    local localRoot = LocalPlayerCharacter.HumanoidRootPart
    
    local AimPoint = SilentAimSettings.FixedFOV and (Camera.ViewportSize / 2) or GetMouseLocation(UserInputService)
    local candidates = {}
    
    for _, Player in ipairs(GetPlayers(Players)) do
        if Player ~= LocalPlayer and not (SilentAimSettings.TeamCheck and Player.Team == LocalPlayer.Team) and not isBlacklisted(Player.Name) then
            local Character = Player.Character
            local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
            if Character and Humanoid and Humanoid.Health > 0 then
                local partForChecks = Character:FindFirstChild(SilentAimSettings.TargetPart) or Character:FindFirstChild("HumanoidRootPart")
                if not partForChecks then continue end

                if not (SilentAimSettings.VisibleCheck and not isPartVisible(partForChecks, LocalPlayerCharacter.Head.Position)) then
                    local physicalDist = (localRoot.Position - partForChecks.Position).Magnitude
                    if physicalDist <= SilentAimSettings.MaxDistance then
                        if SilentAimSettings.PriorityMode == "最近的人(无FOV)" then
                            table.insert(candidates, {character = Character, fov = math.huge, dist = physicalDist, health = Humanoid.Health})
                        else
                            local ScreenPosition, OnScreen = getPositionOnScreen(partForChecks.Position)
                            if OnScreen then
                                local fovDist = (AimPoint - ScreenPosition).Magnitude
                                if fovDist <= SilentAimSettings.FOVRadius then
                                    table.insert(candidates, {character = Character, fov = fovDist, dist = physicalDist, health = Humanoid.Health})
                                end
                            end
                        end
                    end
                end
            end
        end
    end

    if #candidates == 0 then return nil end
    table.sort(candidates, function(a, b)
        if SilentAimSettings.PriorityMode == "最低血量" then
            return a.health < b.health
        elseif SilentAimSettings.PriorityMode == "距离最近" or SilentAimSettings.PriorityMode == "最近的人(无FOV)" then
            return a.dist < b.dist
        else
            return a.fov < b.fov
        end
    end)
    return candidates[1].character
end

local function getNPCTarget()
    local LocalPlayerCharacter = LocalPlayer.Character
    if not LocalPlayerCharacter or not LocalPlayerCharacter:FindFirstChild("HumanoidRootPart") then return nil end
    local localRoot = LocalPlayerCharacter.HumanoidRootPart

    local AimPoint = SilentAimSettings.FixedFOV and (Camera.ViewportSize / 2) or GetMouseLocation(UserInputService)
    local candidates = {}

    for _, NPCModel in ipairs(npcList) do
        if not (SilentAimSettings.TeamCheck and NPCModel.Team and NPCModel.Team == LocalPlayer.Team) and not isBlacklisted(NPCModel.Name) then
            local Humanoid = NPCModel and NPCModel:FindFirstChildOfClass("Humanoid")
            if NPCModel and Humanoid and Humanoid.Health > 0 then
                local partForChecks = NPCModel:FindFirstChild(SilentAimSettings.TargetPart) or NPCModel.PrimaryPart or NPCModel:FindFirstChild("HumanoidRootPart")
                if not partForChecks then continue end

                if not (SilentAimSettings.VisibleCheck and not isPartVisible(partForChecks, LocalPlayerCharacter.Head.Position)) then
                    local physicalDist = (localRoot.Position - partForChecks.Position).Magnitude
                    if physicalDist <= SilentAimSettings.MaxDistance then
                         if SilentAimSettings.PriorityMode == "最近的人(无FOV)" then
                            table.insert(candidates, {character = NPCModel, fov = math.huge, dist = physicalDist, health = Humanoid.Health})
                        else
                            local ScreenPosition, OnScreen = getPositionOnScreen(partForChecks.Position)
                            if OnScreen then
                                local fovDist = (AimPoint - ScreenPosition).Magnitude
                                if fovDist <= SilentAimSettings.FOVRadius then
                                    table.insert(candidates, {character = NPCModel, fov = fovDist, dist = physicalDist, health = Humanoid.Health})
                                end
                            end
                        end
                    end
                end
            end
        end
    end

    if #candidates == 0 then return nil end
    table.sort(candidates, function(a, b)
        if SilentAimSettings.PriorityMode == "最低血量" then
            return a.health < b.health
        elseif SilentAimSettings.PriorityMode == "距离最近" or SilentAimSettings.PriorityMode == "最近的人(无FOV)" then
            return a.dist < b.dist
        else
            return a.fov < b.fov
        end
    end)
    return candidates[1].character
end

function getPolygonPoints(center, radius, sides)
    local points = {}
    local rotationOffset = SilentAimSettings.IndicatorRotationEnabled and currentRotationAngle or 0
    for i = 1, sides do
        local angle = (i - 1) * (2 * math.pi / sides) - (math.pi / 2) + rotationOffset
        table.insert(points, Vector2.new(center.X + radius * math.cos(angle), center.Y + radius * math.sin(angle)))
    end
    return points
end

function hideAllVisuals()
    target_indicator_circle.Visible = false
    for _, line in ipairs(target_indicator_lines) do line.Visible = false end
    for _, text in pairs(overhead_info_texts) do text.Visible = false end
    panel_info_bg.Visible = false
    for _, text in pairs(panel_info_texts) do text.Visible = false end
    if IndependentPanelFrame then IndependentPanelFrame.Visible = false end
end

local repo = "https://raw.githubusercontent.com/ATLASTEAM01/Obsidian/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()
local ThemeManager = loadstring(game:HttpGet(repo .. "addons/ThemeManager.lua"))()
local SaveManager = loadstring(game:HttpGet(repo .. "addons/SaveManager.lua"))()

local Options = Library.Options
local Toggles = Library.Toggles

local Window = Library:CreateWindow({ Title = "Universal Silent Aim", Footer = "1.3", Center = true, AutoShow = true })

local Tabs = {
    Main = Window:AddTab("主页", "user"),
    Visuals = Window:AddTab("视觉", "camera"),
    Management = Window:AddTab("管理", "users"),
    Misc = Window:AddTab("杂项", "box"),
    ["UI Settings"] = Window:AddTab("UI设置", "settings"),
}

local MainSettingsBox = Tabs.Main:AddLeftGroupbox("主设置")
MainSettingsBox:AddToggle("EnabledToggle", { Text = "启用", Default = SilentAimSettings.Enable        pcall(function() main.fovCircle:Remove() end)
        main.fovCircle = nil
    end
    
    if not Drawing then return end
    
    pcall(function()
        main.fovCircle = Drawing.new("Circle")
        main.fovCircle.Visible = false
        main.fovCircle.Thickness = 2
        main.fovCircle.Color = Color3.fromRGB(255, 0, 120)
        main.fovCircle.Filled = false
        main.fovCircle.Transparency = 0.8
        main.fovCircle.Radius = main.fov
        main.fovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    end)
end

-- 创建目标指示器
local function createTargetIndicator()
    if main.targetIndicator then
        pcall(function() main.targetIndicator:Remove() end)
        main.targetIndicator = nil
    end
    
    if not Drawing then return end
    
    pcall(function()
        main.targetIndicator = Drawing.new("Square")
        main.targetIndicator.Visible = false
        main.targetIndicator.Thickness = 2
        main.targetIndicator.Color = Color3.fromRGB(0, 255, 0)
        main.targetIndicator.Filled = false
        main.targetIndicator.Size = Vector2.new(20, 20)
    end)
end

-- 创建锁定指示器
local function createLockOnIndicator()
    if main.lockOnIndicator then
        pcall(function() main.lockOnIndicator:Remove() end)
        main.lockOnIndicator = nil
    end
    
    if not Drawing then return end
    
    pcall(function()
        main.lockOnIndicator = Drawing.new("Circle")
        main.lockOnIndicator.Visible = false
        main.lockOnIndicator.Thickness = 3
        main.lockOnIndicator.Color = Color3.fromRGB(255, 0, 0)
        main.lockOnIndicator.Filled = false
        main.lockOnIndicator.Radius = 15
    end)
end

-- 更新FOV圈位置和可见性
local function updateFOVCircle()
    if not main.fovCircle or not Drawing then return end
    
    pcall(function()
        main.fovCircle.Visible = (main.enable and main.fovVisible)
        main.fovCircle.Radius = main.fov
        local vpSize = Camera.ViewportSize
        main.fovCircle.Position = Vector2.new(vpSize.X / 2, vpSize.Y / 2)
    end)
end

-- 检查点是否在FOV圈内
local function isPointInFOV(screenPos)
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local distance = (screenPos - center).Magnitude
    return distance <= main.fov
end

-- 获取最近玩家头部（在FOV圈内）
local function getClosestHeadInFOV()
    if not LocalPlayer.Character then return nil end
    if not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then return nil end

    local closestHead
    local closestDistance = math.huge
    
    -- 先检查当前锁定的玩家是否在FOV圈内
    if main.lockedPlayer and main.lockMode and main.lockedPlayer.Character then
        local character = main.lockedPlayer.Character
        local head = character:FindFirstChild("Head")
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        
        if head and humanoid and humanoid.Health > 0 then
            local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
            if onScreen and isPointInFOV(Vector2.new(screenPos.X, screenPos.Y)) then
                pcall(function()
                    if main.lockOnIndicator then
                        main.lockOnIndicator.Visible = true
                        main.lockOnIndicator.Position = Vector2.new(screenPos.X, screenPos.Y)
                    end
                end)
                return head
            end
        end
    end
    
    -- 寻找FOV圈内的最近玩家
    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer or not player.Character then continue end
        
        -- 团队检查
        if main.teamcheck and player.Team == LocalPlayer.Team then
            continue
        end
        
        -- 好友检查
        if main.friendcheck and LocalPlayer:IsFriendsWith(player.UserId) then
            continue
        end
        
        local character = player.Character
        local head = character:FindFirstChild("Head")
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        
        if head and humanoid and humanoid.Health > 0 then
            local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
            if onScreen and isPointInFOV(Vector2.new(screenPos.X, screenPos.Y)) then
                local distance = (head.Position - Camera.CFrame.Position).Magnitude
                if distance < closestDistance then
                    closestHead = head
                    closestDistance = distance
                    
                    pcall(function()
                        if main.targetIndicator then
                            main.targetIndicator.Visible = true
                            main.targetIndicator.Position = Vector2.new(screenPos.X - 10, screenPos.Y - 10)
                        end
                    end)
                end
            end
        end
    end
    
    if not main.lockedPlayer and main.lockOnIndicator then
        pcall(function()
            main.lockOnIndicator.Visible = false
        end)
    end
    
    return closestHead
end

-- 获取最近NPC头部
local function getClosestNpcHead()
    local closestNPC
    local closestDistance = math.huge
    local localHrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not localHrp then return nil end
    
    for _, object in ipairs(Workspace:GetDescendants()) do
        if object:IsA("Model") then
            local humanoid = object:FindFirstChildOfClass("Humanoid")
            local hrp = object:FindFirstChild("HumanoidRootPart") or object.PrimaryPart
            
            if humanoid and hrp and humanoid.Health > 0 then
                local isPlayer = false
                for _, pl in ipairs(Players:GetPlayers()) do
                    if pl.Character == object then
                        isPlayer = true
                        break
                    end
                end
                
                if not isPlayer then
                    local distance = (hrp.Position - localHrp.Position).Magnitude
                    if distance < closestDistance then
                        closestDistance = distance
                        closestNPC = object
                    end
                end
            end
        end
    end
    
    return closestNPC and closestNPC:FindFirstChild("Head") or nil
end

-- Hook Raycast（修复版本）
old = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
    local method = getnamecallmethod()
    
    if (main.enable or main.enablenpc) and method == "Raycast" then
        local args = {...}
        
        if not args[1] or not args[2] then
            return old(self, ...)
        end
        
        local origin = args[1]
        local direction = args[2]
        
        if typeof(origin) ~= "Vector3" or typeof(direction) ~= "Vector3" then
            return old(self, ...)
        end
        
        local closestHead = nil
        
        if main.enable then
            closestHead = getClosestHeadInFOV()
        elseif main.enablenpc then
            closestHead = getClosestNpcHead()
        end
        
        if closestHead then
            local toTarget = closestHead.Position - origin
            
            if main.wallbang then
                -- 穿墙模式
                return {
                    Instance = closestHead,
                    Position = closestHead.Position,
                    Normal = (origin - closestHead.Position).Unit,
                    Material = Enum.Material.Plastic,
                    Distance = (closestHead.Position - origin).Magnitude
                }
            else
                -- 检查角度
                local dirUnit = direction.Unit
                local toTargetUnit = toTarget.Unit
                local dot = toTargetUnit:Dot(dirUnit)
                
                if dot >= 0.94 then  -- 约20度范围
                    local raycastParams = RaycastParams.new()
                    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
                    raycastParams.FilterDescendantsInstances = {LocalPlayer.Character}
                    
                    local result = Workspace:Raycast(origin, toTarget, raycastParams)
                    
                    if not result or result.Instance:IsDescendantOf(closestHead.Parent) then
                        return {
                            Instance = closestHead,
                            Position = closestHead.Position,
                            Normal = (origin - closestHead.Position).Unit,
                            Material = Enum.Material.Plastic,
                            Distance = (closestHead.Position - origin).Magnitude
                        }
                    end
                end
            end
        end
    end
    
    return old(self, ...)
end))

-- 渲染循环
local connection
local function update()
    updateFOVCircle()
    
    if not main.enable then
        pcall(function()
            if main.targetIndicator then
                main.targetIndicator.Visible = false
            end
            if main.lockOnIndicator then
                main.lockOnIndicator.Visible = false
            end
        end)
    end
end

-- 选择最近玩家
local function selectClosestPlayerInFOV()
    local closestPlayer
    local closestDistance = math.huge
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer or not player.Character then continue end
        
        local character = player.Character
        local head = character:FindFirstChild("Head")
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        
        if head and humanoid and humanoid.Health > 0 then
            local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
            if onScreen and isPointInFOV(Vector2.new(screenPos.X, screenPos.Y)) then
                local distance = (head.Position - Camera.CFrame.Position).Magnitude
                if distance < closestDistance then
                    closestPlayer = player
                    closestDistance = distance
                end
            end
        end
    end
    
    if closestPlayer then
        main.lockedPlayer = closestPlayer
        main.lockMode = true
        print("✓ 已锁定玩家: " .. closestPlayer.Name)
    else
        print("✗ FOV圈内没有找到玩家")
    end
end

-- UI加载
local function loadUI()
    local success, WindUI = pcall(function()
        return loadstring(game:HttpGet("https://github.com/Footagesus/WindUI/releases/latest/download/main.lua"))()
    end)
    
    if not success or not WindUI then
        print("✗ UI加载失败")
        return
    end
    
    local Window = WindUI:CreateWindow({
        Title = "通用子弹追踪",
        Icon = "rbxassetid://129260712070622",
        IconThemed = true,
        Author = "idk",
        Folder = "CloudHub",
        Size = UDim2.fromOffset(300, 480),
        Transparent = true,
        Theme = "Dark",
        User = {
            Enabled = true,
            Callback = function() print("UI opened") end,
            Anonymous = false
        },
        SideBarWidth = 200,
        ScrollBarEnabled = true,
    })
    
    Window:EditOpenButton({
        Title = "打开UI",
        Icon = "monitor",
        CornerRadius = UDim.new(0, 16),
        StrokeThickness = 2,
        Color = ColorSequence.new(
            Color3.fromHex("FF0F7B"),
            Color3.fromHex("F89B29")
        ),
        Draggable = true,
    })
    
    local MainSection = Window:Section({
        Title = "main",
        Opened = true,
    })
    local Main = MainSection:Tab({ Title = "Main", Icon = "Sword" })
    
    -- 开启子弹追踪
    Main:Toggle({
        Title = "开启子弹追踪",
        Image = "bird",
        Value = false,
        Callback = function(state)
            main.enable = state
            if state then
                if not connection then
                    createFOVCircle()
                    createTargetIndicator()
                    createLockOnIndicator()
                    connection = RunService.RenderStepped:Connect(update)
                    print("✓ 子弹追踪已启用")
                end
            else
                if connection then
                    connection:Disconnect()
                    connection = nil
                end
                pcall(function()
                    if main.fovCircle then main.fovCircle:Remove() main.fovCircle = nil end
                    if main.targetIndicator then main.targetIndicator:Remove() main.targetIndicator = nil end
                    if main.lockOnIndicator then main.lockOnIndicator:Remove() main.lockOnIndicator = nil end
                end)
                print("✓ 子弹追踪已禁用")
            end
        end
    })
    
    Main:Toggle({
        Title = "开启队伍验证",
        Image = "bird",
        Value = false,
        Callback = function(state)
            main.teamcheck = state
        end
    })
    
    Main:Toggle({
        Title = "开启好友验证",
        Image = "bird",
        Value = false,
        Callback = function(state)
            main.friendcheck = state
        end
    })
    
    Main:Toggle({
        Title = "开启NPC子弹追踪",
        Image = "bird",
        Value = false,
        Callback = function(state)
            main.enablenpc = state
        end
    })
    
    Main:Toggle({
        Title = "穿墙模式",
        Image = "wall",
        Value = main.wallbang,
        Callback = function(state)
            main.wallbang = state
        end
    })
    
    Main:Slider({
        Title = "FOV圈大小",
        Min = 50,
        Max = 300,
        Value = main.fov,
        Callback = function(value)
            main.fov = value
        end
    })
    
    Main:Toggle({
        Title = "显示FOV圈",
        Image = "circle",
        Value = main.fovVisible,
        Callback = function(state)
            main.fovVisible = state
            updateFOVCircle()
        end
    })
    
    Main:Toggle({
        Title = "锁定模式",
        Image = "target",
        Value = main.lockMode,
        Callback = function(state)
            main.lockMode = state
        end
    })
    
    Main:Button({
        Title = "选择FOV内最近玩家",
        Callback = function()
            selectClosestPlayerInFOV()
        end
    })
    
    Main:Button({
        Title = "清除锁定",
        Callback = function()
            main.lockedPlayer = nil
            print("✓ 已清除锁定")
        end
    })
    
    Main:Button({
        Title = "切换下一个玩家",
        Callback = function()
            local players = {}
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= LocalPlayer and player.Character then
                    local character = player.Character
                    local head = character:FindFirstChild("Head")
                    local humanoid = character:FindFirstChildOfClass("Humanoid")
                    
                    if head and humanoid and humanoid.Health > 0 then
                        local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
                        if onScreen and isPointInFOV(Vector2.new(screenPos.X, screenPos.Y)) then
                            table.insert(players, player)
                        end
                    end
                end
            end
            
            if #players > 0 then
                local currentIndex = 0
                if main.lockedPlayer then
                    for i, player in ipairs(players) do
                        if player == main.lockedPlayer then
                            currentIndex = i
                            break
                        end
                    end
                end
                
                local nextIndex = (currentIndex % #players) + 1
                main.lockedPlayer = players[nextIndex]
                print("✓ 已切换锁定到玩家: " .. main.lockedPlayer.Name)
            else
                print("✗ FOV圈内没有找到玩家")
            end
        end
    })
    
    print("✓ 子弹追踪脚本已加载")
end

-- 启动UI
task.spawn(loadUI)
