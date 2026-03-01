-- 服务
local Workspace = game:GetService("Workspace")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- 等待Drawing库加载
if not Drawing then
    while not Drawing do
        game:GetService("RunService").RenderStepped:Wait()
    end
end

-- 变量
local old
local main = {
    enable = false,
    teamcheck = false,
    friendcheck = false,
    enablenpc = false,
    wallbang = true,
    fov = 150,
    fovVisible = true,
    lockedPlayer = nil,
    lockMode = true,
    fovCircle = nil,
    targetIndicator = nil,
    lockOnIndicator = nil,
}

-- 安全创建FOV圈
local function createFOVCircle()
    if main.fovCircle then
        pcall(function() main.fovCircle:Remove() end)
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
