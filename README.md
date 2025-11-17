_G.AutoFarmConfig = _G.AutoFarmConfig or {
    -- ระยะทาง
    ATTACK_RANGE = 100,
    BEHIND_DISTANCE = 5,
    FRONT_DISTANCE = 5,
    HEIGHT_OFFSET = 2.5,
    SAFE_HEIGHT = 100,
    
    -- เลือดและความปลอดภัย
    HEALTH_THRESHOLD = 80,
    SAFE_MODE_THRESHOLD = 25,
    SAFE_MODE_RECOVER = 70,
    
    -- ความเร็ว
    ATTACK_SPEED = 0.5,
    
    -- เปิด/ปิดระบบอัตโนมัติ
    AUTO_FARM = true,
    AUTO_COLLECT = true,
    AUTO_HEAL = true,
    AUTO_RETRY = true,
    SAFE_MODE = true,
}

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")
local Character = Player.Character or Player.CharacterAdded:Wait()
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")
local Humanoid = Character:WaitForChild("Humanoid")

local ATTACK_RANGE = _G.AutoFarmConfig.ATTACK_RANGE
local ATTACK_SPEED = _G.AutoFarmConfig.ATTACK_SPEED
local BEHIND_DISTANCE = _G.AutoFarmConfig.BEHIND_DISTANCE
local FRONT_DISTANCE = _G.AutoFarmConfig.FRONT_DISTANCE
local HEIGHT_OFFSET = _G.AutoFarmConfig.HEIGHT_OFFSET
local HEALTH_THRESHOLD = _G.AutoFarmConfig.HEALTH_THRESHOLD
local SAFE_MODE_THRESHOLD = _G.AutoFarmConfig.SAFE_MODE_THRESHOLD
local SAFE_MODE_RECOVER = _G.AutoFarmConfig.SAFE_MODE_RECOVER
local SAFE_HEIGHT = _G.AutoFarmConfig.SAFE_HEIGHT

local isRunning = _G.AutoFarmConfig.AUTO_FARM
local currentTarget = nil
local autoCollectEnabled = _G.AutoFarmConfig.AUTO_COLLECT
local autoHeal = _G.AutoFarmConfig.AUTO_HEAL
local autoRetry = _G.AutoFarmConfig.AUTO_RETRY
local uiVisible = true
local safeMode = false
local safeModeEnabled = _G.AutoFarmConfig.SAFE_MODE
local currentMonsterIndex = 1
local allMonsters = {}
local originalPosition = nil
local attackPosition = "behind"
local lastPositionSwitch = tick()
local safePlatform = nil
local safePlatformPosition = nil

-- =============================================
-- UTILITY FUNCTIONS (ประกาศก่อนใช้งาน)
-- =============================================

local function getHealthPercent()
    local success, result = pcall(function()
        local healthBar = PlayerGui:FindFirstChild("Main")
        if healthBar then
            healthBar = healthBar:FindFirstChild("PlayerBar")
            if healthBar then
                healthBar = healthBar:FindFirstChild("Main")
                if healthBar then
                    healthBar = healthBar:FindFirstChild("HealthBar")
                    if healthBar then
                        healthBar = healthBar:FindFirstChild("Amount")
                        if healthBar then
                            local healthText = healthBar.Text
                            local current, max = healthText:match("(%d+)/(%d+)")
                            if current and max then
                                local currentNum = tonumber(current)
                                local maxNum = tonumber(max)
                                if currentNum and maxNum and maxNum > 0 then
                                    return (currentNum / maxNum) * 100
                                end
                            end
                        end
                    end
                end
            end
        end
        return 100
    end)
    if success and result then
        return result
    end
    return 100
end

local function createSafePlatform()
    pcall(function()
        if safePlatform and safePlatform.Parent then
            safePlatform:Destroy()
        end
        
        local platform = Instance.new("Part")
        platform.Name = "SafePlatform"
        platform.Size = Vector3.new(20, 1, 20)
        platform.Anchored = true
        platform.CanCollide = true
        platform.Transparency = 0.3
        platform.Material = Enum.Material.ForceField
        platform.BrickColor = BrickColor.new("Bright blue")
        
        -- บันทึกตำแหน่งปัจจุบันถ้ายังไม่มี
        if not safePlatformPosition then
            safePlatformPosition = HumanoidRootPart.Position + Vector3.new(0, SAFE_HEIGHT, 0)
        end
        
        platform.Position = safePlatformPosition
        platform.Parent = Workspace
        
        safePlatform = platform
    end)
    return safePlatform
end

local function removeSafePlatform()
    pcall(function()
        if safePlatform and safePlatform.Parent then
            safePlatform:Destroy()
            safePlatform = nil
        end
    end)
end

local function teleportToSafeZone()
    pcall(function()
        if not originalPosition then
            originalPosition = HumanoidRootPart.Position
        end
        
        -- สร้าง platform ที่ตำแหน่งเดิม หรือตำแหน่งที่บันทึกไว้
        local platform = createSafePlatform()
        
        if platform then
            -- วาร์ปผู้เล่นไปยืนบน platform
            local standPos = platform.Position + Vector3.new(0, 3, 0)
            HumanoidRootPart.CFrame = CFrame.new(standPos)
            HumanoidRootPart.Velocity = Vector3.new(0, 0, 0)
            HumanoidRootPart.RotVelocity = Vector3.new(0, 0, 0)
        end
    end)
end

local function disableAnimations()
    pcall(function()
        for _, track in pairs(Humanoid:GetPlayingAnimationTracks()) do
            track:Stop()
        end
    end)
end

local function setupNoClip()
    pcall(function()
        for _, part in pairs(Character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
    end)
end

local function collectAllDrops()
    if not autoCollectEnabled then return end
    task.spawn(function()
        pcall(function()
            local cameraDrops = Workspace.Camera:FindFirstChild("Drops")
            if not cameraDrops then return end
            for _, drop in pairs(cameraDrops:GetChildren()) do
                if drop.Name == "Drop" and drop:FindFirstChild("Center") then
                    local center = drop.Center
                    local prompt = center:FindFirstChildOfClass("ProximityPrompt")
                    if prompt then
                        fireproximityprompt(prompt, 0)
                    end
                end
            end
        end)
    end)
end

local function findAllMonsters()
    local monsters = {}
    if not Workspace:FindFirstChild("Mobs") then
        return monsters
    end
    for _, mob in pairs(Workspace.Mobs:GetChildren()) do
        if mob:FindFirstChild("Humanoid") and mob.Humanoid.Health > 0 and mob:FindFirstChild("HumanoidRootPart") then
            table.insert(monsters, mob)
        end
    end
    return monsters
end

local function getNextMonster()
    allMonsters = findAllMonsters()
    if #allMonsters == 0 then
        return nil
    end
    
    currentMonsterIndex = currentMonsterIndex + 1
    if currentMonsterIndex > #allMonsters then
        currentMonsterIndex = 1
    end
    
    return allMonsters[currentMonsterIndex]
end

local function lockOnTarget(monster)
    if not monster or not monster:FindFirstChild("HumanoidRootPart") then
        return
    end
    pcall(function()
        local monsterPos = monster.HumanoidRootPart.Position
        local currentPos = HumanoidRootPart.Position
        HumanoidRootPart.CFrame = CFrame.new(
            currentPos,
            Vector3.new(monsterPos.X, currentPos.Y, monsterPos.Z)
        )
    end)
end

local function teleportBehindMonster(monster)
    if not monster or not monster:FindFirstChild("HumanoidRootPart") then
        return false
    end
    pcall(function()
        local monsterHRP = monster.HumanoidRootPart
        local monsterPos = monsterHRP.Position
        local monsterCFrame = monsterHRP.CFrame
        
        local currentTime = tick()
        if currentTime - lastPositionSwitch >= 0.5 then
            if attackPosition == "behind" then
                attackPosition = "front"
            else
                attackPosition = "behind"
            end
            lastPositionSwitch = currentTime
        end
        
        local offset
        if attackPosition == "behind" then
            offset = monsterCFrame.LookVector * -BEHIND_DISTANCE
        else
            offset = monsterCFrame.LookVector * FRONT_DISTANCE
        end
        
        local targetPos = monsterPos + offset + Vector3.new(0, HEIGHT_OFFSET, 0)
        local targetCFrame = CFrame.new(targetPos, monsterPos)
        
        HumanoidRootPart.CFrame = targetCFrame
        HumanoidRootPart.Velocity = Vector3.new(0, 0, 0)
        HumanoidRootPart.RotVelocity = Vector3.new(0, 0, 0)
    end)
    return true
end

local function attackMonster()
    if not currentTarget or not currentTarget:FindFirstChild("HumanoidRootPart") then return end
    task.spawn(function()
        pcall(function()
            local monsterHRP = currentTarget.HumanoidRootPart
            local monsterPos = monsterHRP.Position
            local playerPos = HumanoidRootPart.Position
            local direction = (monsterPos - playerPos).Unit
            
            local mobService = ReplicatedStorage:FindFirstChild("ReplicatedStorage")
            if mobService then
                mobService = mobService:FindFirstChild("Packages")
                if mobService then
                    mobService = mobService:FindFirstChild("Knit")
                    if mobService then
                        mobService = mobService:FindFirstChild("Services")
                        if mobService then
                            mobService = mobService:FindFirstChild("MobService")
                            if mobService then
                                local signal = mobService:FindFirstChild("AttackVFXSignal")
                                if signal then
                                    signal:Fire("SwordHit", monsterPos, {
                                        Player = Player,
                                        Direction = direction
                                    })
                                end
                            end
                        end
                    end
                end
            end
            
            pcall(function()
                local knit = require(ReplicatedStorage.ReplicatedStorage.Packages.Knit)
                local attackService = knit.GetService("AttackService")
                if attackService then
                    local mouseData = {
                        Hit = monsterPos,
                        Target = monsterHRP,
                        Origin = playerPos
                    }
                    attackService:UseAbility("Slot_M1", mouseData)
                end
            end)
            
            for _, tool in pairs(Character:GetChildren()) do
                if tool:IsA("Tool") then
                    pcall(function()
                        tool:Activate()
                    end)
                end
            end
        end)
    end)
end

-- =============================================
-- UI CREATION
-- =============================================

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AutoFarmUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.DisplayOrder = 999999999
ScreenGui.Parent = PlayerGui

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 420, 0, 600)
MainFrame.Position = UDim2.new(0.5, -210, 0.5, -300)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
MainFrame.BorderSizePixel = 0
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 16)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(100, 150, 255)
MainStroke.Thickness = 2
MainStroke.Transparency = 0.3
MainStroke.Parent = MainFrame

local GradientBG = Instance.new("Frame")
GradientBG.Size = UDim2.new(1, 0, 1, 0)
GradientBG.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
GradientBG.BorderSizePixel = 0
GradientBG.ZIndex = 0
GradientBG.Parent = MainFrame

local BGCorner = Instance.new("UICorner")
BGCorner.CornerRadius = UDim.new(0, 16)
BGCorner.Parent = GradientBG

local BGGradient = Instance.new("UIGradient")
BGGradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(30, 30, 50)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(15, 15, 25))
}
BGGradient.Rotation = 45
BGGradient.Parent = GradientBG

local TitleBar = Instance.new("Frame")
TitleBar.Name = "TitleBar"
TitleBar.Size = UDim2.new(1, 0, 0, 70)
TitleBar.BackgroundColor3 = Color3.fromRGB(25, 25, 40)
TitleBar.BorderSizePixel = 0
TitleBar.Parent = MainFrame

local TitleCorner = Instance.new("UICorner")
TitleCorner.CornerRadius = UDim.new(0, 16)
TitleCorner.Parent = TitleBar

local TitleGradient = Instance.new("UIGradient")
TitleGradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(100, 150, 255)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(150, 100, 255)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 100, 150))
}
TitleGradient.Rotation = 90
TitleGradient.Parent = TitleBar

local TitleOverlay = Instance.new("Frame")
TitleOverlay.Size = UDim2.new(1, 0, 1, 0)
TitleOverlay.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
TitleOverlay.BackgroundTransparency = 0.4
TitleOverlay.BorderSizePixel = 0
TitleOverlay.Parent = TitleBar

local TitleOverlayCorner = Instance.new("UICorner")
TitleOverlayCorner.CornerRadius = UDim.new(0, 16)
TitleOverlayCorner.Parent = TitleOverlay

local dragging = false
local dragInput, mousePos, framePos

TitleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        mousePos = input.Position
        framePos = MainFrame.Position
        
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

TitleBar.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - mousePos
        MainFrame.Position = UDim2.new(
            framePos.X.Scale, 
            framePos.X.Offset + delta.X, 
            framePos.Y.Scale, 
            framePos.Y.Offset + delta.Y
        )
    end
end)

task.spawn(function()
    while wait() do
        TweenService:Create(TitleGradient, TweenInfo.new(3, Enum.EasingStyle.Linear, Enum.EasingDirection.InOut, -1, true), {Rotation = TitleGradient.Rotation + 360}):Play()
        wait(3)
    end
end)

local TitleIcon = Instance.new("TextLabel")
TitleIcon.Size = UDim2.new(0, 50, 0, 50)
TitleIcon.Position = UDim2.new(0, 15, 0.5, -25)
TitleIcon.BackgroundTransparency = 1
TitleIcon.Text = "⚡"
TitleIcon.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleIcon.TextSize = 32
TitleIcon.Font = Enum.Font.GothamBold
TitleIcon.ZIndex = 5
TitleIcon.Parent = TitleBar

task.spawn(function()
    while wait(1) do
        TweenService:Create(TitleIcon, TweenInfo.new(0.5, Enum.EasingStyle.Bounce, Enum.EasingDirection.Out), {
            TextSize = 36
        }):Play()
        wait(0.5)
        TweenService:Create(TitleIcon, TweenInfo.new(0.5, Enum.EasingStyle.Bounce, Enum.EasingDirection.Out), {
            TextSize = 32
        }):Play()
    end
end)

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, -140, 0, 30)
TitleLabel.Position = UDim2.new(0, 70, 0, 12)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "AUTO FARM SYSTEM"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.TextSize = 22
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.ZIndex = 5
TitleLabel.Parent = TitleBar

local SubtitleLabel = Instance.new("TextLabel")
SubtitleLabel.Size = UDim2.new(1, -140, 0, 20)
SubtitleLabel.Position = UDim2.new(0, 70, 0, 42)
SubtitleLabel.BackgroundTransparency = 1
SubtitleLabel.Text = "Premium Edition v2.2 - Fixed"
SubtitleLabel.TextColor3 = Color3.fromRGB(150, 200, 255)
SubtitleLabel.TextSize = 13
SubtitleLabel.Font = Enum.Font.Gotham
SubtitleLabel.TextXAlignment = Enum.TextXAlignment.Left
SubtitleLabel.TextTransparency = 0.3
SubtitleLabel.ZIndex = 5
SubtitleLabel.Parent = TitleBar

local CloseButton = Instance.new("TextButton")
CloseButton.Name = "CloseButton"
CloseButton.Size = UDim2.new(0, 45, 0, 45)
CloseButton.Position = UDim2.new(1, -60, 0.5, -22.5)
CloseButton.BackgroundColor3 = Color3.fromRGB(255, 60, 80)
CloseButton.BorderSizePixel = 0
CloseButton.Text = "✕"
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.TextSize = 24
CloseButton.Font = Enum.Font.GothamBold
CloseButton.ZIndex = 5
CloseButton.Parent = TitleBar

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 10)
CloseCorner.Parent = CloseButton

local CloseStroke = Instance.new("UIStroke")
CloseStroke.Color = Color3.fromRGB(255, 255, 255)
CloseStroke.Thickness = 2
CloseStroke.Transparency = 0.7
CloseStroke.Parent = CloseButton

CloseButton.MouseEnter:Connect(function()
    TweenService:Create(CloseButton, TweenInfo.new(0.2), {
        Size = UDim2.new(0, 50, 0, 50),
        BackgroundColor3 = Color3.fromRGB(255, 80, 100)
    }):Play()
    TweenService:Create(CloseStroke, TweenInfo.new(0.2), {Transparency = 0.3}):Play()
end)

CloseButton.MouseLeave:Connect(function()
    TweenService:Create(CloseButton, TweenInfo.new(0.2), {
        Size = UDim2.new(0, 45, 0, 45),
        BackgroundColor3 = Color3.fromRGB(255, 60, 80)
    }):Play()
    TweenService:Create(CloseStroke, TweenInfo.new(0.2), {Transparency = 0.7}):Play()
end)

local StatusFrame = Instance.new("Frame")
StatusFrame.Name = "StatusFrame"
StatusFrame.Size = UDim2.new(1, -30, 0, 80)
StatusFrame.Position = UDim2.new(0, 15, 0, 85)
StatusFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 45)
StatusFrame.BorderSizePixel = 0
StatusFrame.Parent = MainFrame

local StatusCorner = Instance.new("UICorner")
StatusCorner.CornerRadius = UDim.new(0, 12)
StatusCorner.Parent = StatusFrame

local StatusStroke = Instance.new("UIStroke")
StatusStroke.Color = Color3.fromRGB(100, 255, 100)
StatusStroke.Thickness = 2
StatusStroke.Transparency = 0.6
StatusStroke.Parent = StatusFrame

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, -20, 0, 25)
StatusLabel.Position = UDim2.new(0, 10, 0, 10)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "🛡 Status: Normal Mode"
StatusLabel.TextColor3 = Color3.fromRGB(100, 255, 100)
StatusLabel.TextSize = 16
StatusLabel.Font = Enum.Font.GothamBold
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.Parent = StatusFrame

local HealthLabel = Instance.new("TextLabel")
HealthLabel.Size = UDim2.new(1, -20, 0, 20)
HealthLabel.Position = UDim2.new(0, 10, 0, 35)
HealthLabel.BackgroundTransparency = 1
HealthLabel.Text = "❤ Health: 100%"
HealthLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
HealthLabel.TextSize = 14
HealthLabel.Font = Enum.Font.Gotham
HealthLabel.TextXAlignment = Enum.TextXAlignment.Left
HealthLabel.Parent = StatusFrame

local TargetLabel = Instance.new("TextLabel")
TargetLabel.Size = UDim2.new(1, -20, 0, 20)
TargetLabel.Position = UDim2.new(0, 10, 0, 55)
TargetLabel.BackgroundTransparency = 1
TargetLabel.Text = "🎯 Target: None"
TargetLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TargetLabel.TextSize = 14
TargetLabel.Font = Enum.Font.Gotham
TargetLabel.TextXAlignment = Enum.TextXAlignment.Left
TargetLabel.Parent = StatusFrame

local Container = Instance.new("ScrollingFrame")
Container.Name = "Container"
Container.Size = UDim2.new(1, -30, 1, -180)
Container.Position = UDim2.new(0, 15, 0, 180)
Container.BackgroundTransparency = 1
Container.BorderSizePixel = 0
Container.ScrollBarThickness = 8
Container.ScrollBarImageColor3 = Color3.fromRGB(100, 150, 255)
Container.CanvasSize = UDim2.new(0, 0, 0, 0)
Container.Parent = MainFrame

local UIListLayout = Instance.new("UIListLayout")
UIListLayout.Padding = UDim.new(0, 12)
UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
UIListLayout.Parent = Container

local function updateCanvasSize()
    Container.CanvasSize = UDim2.new(0, 0, 0, UIListLayout.AbsoluteContentSize.Y + 15)
end

UIListLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(updateCanvasSize)

local function createToggle(name, icon, color, callback)
    local Toggle = Instance.new("Frame")
    Toggle.Name = name
    Toggle.Size = UDim2.new(1, 0, 0, 70)
    Toggle.BackgroundColor3 = Color3.fromRGB(30, 30, 45)
    Toggle.BorderSizePixel = 0
    Toggle.Parent = Container

    local ToggleCorner = Instance.new("UICorner")
    ToggleCorner.CornerRadius = UDim.new(0, 12)
    ToggleCorner.Parent = Toggle

    local ToggleStroke = Instance.new("UIStroke")
    ToggleStroke.Color = color
    ToggleStroke.Thickness = 2
    ToggleStroke.Transparency = 0.6
    ToggleStroke.Parent = Toggle

    local IconLabel = Instance.new("TextLabel")
    IconLabel.Size = UDim2.new(0, 50, 0, 50)
    IconLabel.Position = UDim2.new(0, 10, 0.5, -25)
    IconLabel.BackgroundTransparency = 1
    IconLabel.Text = icon
    IconLabel.TextColor3 = color
    IconLabel.TextSize = 28
    IconLabel.Font = Enum.Font.GothamBold
    IconLabel.Parent = Toggle

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, -150, 1, 0)
    Label.Position = UDim2.new(0, 65, 0, 0)
    Label.BackgroundTransparency = 1
    Label.Text = name
    Label.TextColor3 = Color3.fromRGB(255, 255, 255)
    Label.TextSize = 18
    Label.Font = Enum.Font.GothamBold
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.TextYAlignment = Enum.TextYAlignment.Center
    Label.Parent = Toggle

    local StatusBG = Instance.new("Frame")
    StatusBG.Size = UDim2.new(0, 70, 0, 35)
    StatusBG.Position = UDim2.new(1, -80, 0.5, -17.5)
    StatusBG.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
    StatusBG.BorderSizePixel = 0
    StatusBG.Parent = Toggle

    local StatusCorner = Instance.new("UICorner")
    StatusCorner.CornerRadius = UDim.new(1, 0)
    StatusCorner.Parent = StatusBG

    local StatusCircle = Instance.new("Frame")
    StatusCircle.Size = UDim2.new(0, 27, 0, 27)
    StatusCircle.Position = UDim2.new(0, 4, 0.5, -13.5)
    StatusCircle.BackgroundColor3 = Color3.fromRGB(255, 70, 70)
    StatusCircle.BorderSizePixel = 0
    StatusCircle.Parent = StatusBG

    local CircleCorner = Instance.new("UICorner")
    CircleCorner.CornerRadius = UDim.new(1, 0)
    CircleCorner.Parent = StatusCircle

    local StatusLabel = Instance.new("TextLabel")
    StatusLabel.Size = UDim2.new(1, -35, 1, 0)
    StatusLabel.Position = UDim2.new(0, 35, 0, 0)
    StatusLabel.BackgroundTransparency = 1
    StatusLabel.Text = "OFF"
    StatusLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    StatusLabel.TextSize = 13
    StatusLabel.Font = Enum.Font.GothamBold
    StatusLabel.Parent = StatusBG

    local Button = Instance.new("TextButton")
    Button.Size = UDim2.new(1, 0, 1, 0)
    Button.BackgroundTransparency = 1
    Button.Text = ""
    Button.Parent = Toggle

    local isEnabled = false

    Button.MouseButton1Click:Connect(function()
        isEnabled = not isEnabled
        
        if isEnabled then
            TweenService:Create(StatusCircle, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                Position = UDim2.new(1, -31, 0.5, -13.5),
                BackgroundColor3 = Color3.fromRGB(100, 255, 100)
            }):Play()
            TweenService:Create(StatusBG, TweenInfo.new(0.3), {
                BackgroundColor3 = color
            }):Play()
            StatusLabel.Text = "ON"
            TweenService:Create(ToggleStroke, TweenInfo.new(0.3), {Transparency = 0.2}):Play()
            TweenService:Create(Toggle, TweenInfo.new(0.1), {Size = UDim2.new(1, 5, 0, 70)}):Play()
            wait(0.1)
            TweenService:Create(Toggle, TweenInfo.new(0.1), {Size = UDim2.new(1, 0, 0, 70)}):Play()
        else
            TweenService:Create(StatusCircle, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                Position = UDim2.new(0, 4, 0.5, -13.5),
                BackgroundColor3 = Color3.fromRGB(255, 70, 70)
            }):Play()
            TweenService:Create(StatusBG, TweenInfo.new(0.3), {
                BackgroundColor3 = Color3.fromRGB(50, 50, 70)
            }):Play()
            StatusLabel.Text = "OFF"
            TweenService:Create(ToggleStroke, TweenInfo.new(0.3), {Transparency = 0.6}):Play()
        end
        
        callback(isEnabled)
    end)

    Button.MouseEnter:Connect(function()
        TweenService:Create(Toggle, TweenInfo.new(0.2), {
            BackgroundColor3 = Color3.fromRGB(35, 35, 55)
        }):Play()
        TweenService:Create(IconLabel, TweenInfo.new(0.2), {TextSize = 32}):Play()
    end)

    Button.MouseLeave:Connect(function()
        TweenService:Create(Toggle, TweenInfo.new(0.2), {
            BackgroundColor3 = Color3.fromRGB(30, 30, 45)
        }):Play()
        TweenService:Create(IconLabel, TweenInfo.new(0.2), {TextSize = 28}):Play()
    end)

    return Toggle
end

local function createSlider(name, icon, color, min, max, default, callback)
    local Slider = Instance.new("Frame")
    Slider.Name = name
    Slider.Size = UDim2.new(1, 0, 0, 85)
    Slider.BackgroundColor3 = Color3.fromRGB(30, 30, 45)
    Slider.BorderSizePixel = 0
    Slider.Parent = Container

    local SliderCorner = Instance.new("UICorner")
    SliderCorner.CornerRadius = UDim.new(0, 12)
    SliderCorner.Parent = Slider

    local SliderStroke = Instance.new("UIStroke")
    SliderStroke.Color = color
    SliderStroke.Thickness = 2
    SliderStroke.Transparency = 0.6
    SliderStroke.Parent = Slider

    local IconLabel = Instance.new("TextLabel")
    IconLabel.Size = UDim2.new(0, 40, 0, 40)
    IconLabel.Position = UDim2.new(0, 15, 0, 10)
    IconLabel.BackgroundTransparency = 1
    IconLabel.Text = icon
    IconLabel.TextColor3 = color
    IconLabel.TextSize = 24
    IconLabel.Font = Enum.Font.GothamBold
    IconLabel.Parent = Slider

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, -70, 0, 25)
    Label.Position = UDim2.new(0, 60, 0, 10)
    Label.BackgroundTransparency = 1
    Label.Text = name
    Label.TextColor3 = Color3.fromRGB(255, 255, 255)
    Label.TextSize = 16
    Label.Font = Enum.Font.GothamBold
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.Parent = Slider

    local ValueLabel = Instance.new("TextLabel")
    ValueLabel.Size = UDim2.new(0, 60, 0, 25)
    ValueLabel.Position = UDim2.new(1, -70, 0, 10)
    ValueLabel.BackgroundColor3 = color
    ValueLabel.Text = default .. "%"
    ValueLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    ValueLabel.TextSize = 16
    ValueLabel.Font = Enum.Font.GothamBold
    ValueLabel.Parent = Slider

    local ValueCorner = Instance.new("UICorner")
    ValueCorner.CornerRadius = UDim.new(0, 8)
    ValueCorner.Parent = ValueLabel

    local SliderBar = Instance.new("Frame")
    SliderBar.Size = UDim2.new(1, -30, 0, 10)
    SliderBar.Position = UDim2.new(0, 15, 1, -25)
    SliderBar.BackgroundColor3 = Color3.fromRGB(40, 40, 60)
    SliderBar.BorderSizePixel = 0
    SliderBar.Parent = Slider

    local BarCorner = Instance.new("UICorner")
    BarCorner.CornerRadius = UDim.new(1, 0)
    BarCorner.Parent = SliderBar

    local Fill = Instance.new("Frame")
    Fill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
    Fill.BackgroundColor3 = color
    Fill.BorderSizePixel = 0
    Fill.Parent = SliderBar

    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(1, 0)
    FillCorner.Parent = Fill

    local FillGradient = Instance.new("UIGradient")
    FillGradient.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, color),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 255, 255))
    }
    FillGradient.Parent = Fill

    local Dragging = false

    SliderBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            Dragging = true
            TweenService:Create(SliderStroke, TweenInfo.new(0.2), {Transparency = 0.2}):Play()
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            Dragging = false
            TweenService:Create(SliderStroke, TweenInfo.new(0.2), {Transparency = 0.6}):Play()
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if Dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local mouse = UserInputService:GetMouseLocation()
            local relativeX = math.clamp(mouse.X - SliderBar.AbsolutePosition.X, 0, SliderBar.AbsoluteSize.X)
            local percent = relativeX / SliderBar.AbsoluteSize.X
            local value = math.floor(min + (max - min) * percent)
            
            Fill.Size = UDim2.new(percent, 0, 1, 0)
            ValueLabel.Text = value .. "%"
            callback(value)
        end
    end)

    return Slider
end

local ToggleButton = Instance.new("TextButton")
ToggleButton.Name = "ToggleButton"
ToggleButton.Size = UDim2.new(0, 65, 0, 65)
ToggleButton.Position = UDim2.new(0, 15, 0, 15)
ToggleButton.BackgroundColor3 = Color3.fromRGB(30, 30, 45)
ToggleButton.BorderSizePixel = 0
ToggleButton.Text = ""
ToggleButton.Font = Enum.Font.GothamBold
ToggleButton.Active = true
ToggleButton.Draggable = true
ToggleButton.Parent = ScreenGui

local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(0, 15)
ToggleCorner.Parent = ToggleButton

local ToggleStroke = Instance.new("UIStroke")
ToggleStroke.Color = Color3.fromRGB(100, 150, 255)
ToggleStroke.Thickness = 3
ToggleStroke.Transparency = 0.4
ToggleStroke.Parent = ToggleButton

local ToggleGradient = Instance.new("UIGradient")
ToggleGradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(100, 150, 255)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(150, 100, 255))
}
ToggleGradient.Rotation = 45
ToggleGradient.Parent = ToggleButton

local OpenIcon = Instance.new("TextLabel")
OpenIcon.Size = UDim2.new(1, 0, 1, 0)
OpenIcon.BackgroundTransparency = 1
OpenIcon.Text = "⚙"
OpenIcon.TextColor3 = Color3.fromRGB(255, 255, 255)
OpenIcon.TextSize = 32
OpenIcon.Font = Enum.Font.GothamBold
OpenIcon.Visible = false
OpenIcon.Parent = ToggleButton

local CloseIcon = Instance.new("TextLabel")
CloseIcon.Size = UDim2.new(1, 0, 1, 0)
CloseIcon.BackgroundTransparency = 1
CloseIcon.Text = "✓"
CloseIcon.TextColor3 = Color3.fromRGB(100, 255, 100)
CloseIcon.TextSize = 36
CloseIcon.Font = Enum.Font.GothamBold
CloseIcon.Visible = true
CloseIcon.Parent = ToggleButton

task.spawn(function()
    while wait(0.05) do
        TweenService:Create(ToggleGradient, TweenInfo.new(2, Enum.EasingStyle.Linear), {Rotation = ToggleGradient.Rotation + 360}):Play()
        wait(2)
    end
end)

ToggleButton.MouseEnter:Connect(function()
    TweenService:Create(ToggleButton, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, 75, 0, 75)
    }):Play()
    if uiVisible then
        TweenService:Create(CloseIcon, TweenInfo.new(0.3), {TextSize = 42}):Play()
    else
        TweenService:Create(OpenIcon, TweenInfo.new(0.3), {TextSize = 38}):Play()
    end
    TweenService:Create(ToggleStroke, TweenInfo.new(0.3), {Transparency = 0}):Play()
end)

ToggleButton.MouseLeave:Connect(function()
    TweenService:Create(ToggleButton, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, 65, 0, 65)
    }):Play()
    if uiVisible then
        TweenService:Create(CloseIcon, TweenInfo.new(0.3), {TextSize = 36}):Play()
    else
        TweenService:Create(OpenIcon, TweenInfo.new(0.3), {TextSize = 32}):Play()
    end
    TweenService:Create(ToggleStroke, TweenInfo.new(0.3), {Transparency = 0.4}):Play()
end)

ToggleButton.MouseButton1Click:Connect(function()
    uiVisible = not uiVisible
    
    TweenService:Create(ToggleButton, TweenInfo.new(0.3, Enum.EasingStyle.Back), {Rotation = ToggleButton.Rotation + 180}):Play()
    
    if uiVisible then
        OpenIcon.Visible = false
        CloseIcon.Visible = true
        MainFrame.Visible = true
        TweenService:Create(MainFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Size = UDim2.new(0, 420, 0, 600)
        }):Play()
        TweenService:Create(ToggleButton, TweenInfo.new(0.3), {
            BackgroundColor3 = Color3.fromRGB(30, 50, 30)
        }):Play()
        TweenService:Create(ToggleStroke, TweenInfo.new(0.3), {
            Color = Color3.fromRGB(100, 255, 100)
        }):Play()
    else
        OpenIcon.Visible = true
        CloseIcon.Visible = false
        TweenService:Create(MainFrame, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.In), {
            Size = UDim2.new(0, 0, 0, 0)
        }):Play()
        wait(0.3)
        MainFrame.Visible = false
        TweenService:Create(ToggleButton, TweenInfo.new(0.3), {
            BackgroundColor3 = Color3.fromRGB(30, 30, 45)
        }):Play()
        TweenService:Create(ToggleStroke, TweenInfo.new(0.3), {
            Color = Color3.fromRGB(100, 150, 255)
        }):Play()
    end
end)

CloseButton.MouseButton1Click:Connect(function()
    uiVisible = false
    OpenIcon.Visible = true
    CloseIcon.Visible = false
    TweenService:Create(ToggleButton, TweenInfo.new(0.3, Enum.EasingStyle.Back), {Rotation = ToggleButton.Rotation + 180}):Play()
    TweenService:Create(MainFrame, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.In), {
        Size = UDim2.new(0, 0, 0, 0)
    }):Play()
    wait(0.3)
    MainFrame.Visible = false
    TweenService:Create(ToggleButton, TweenInfo.new(0.3), {
        BackgroundColor3 = Color3.fromRGB(30, 30, 45)
    }):Play()
    TweenService:Create(ToggleStroke, TweenInfo.new(0.3), {
        Color = Color3.fromRGB(100, 150, 255)
    }):Play()
end)

-- =============================================
-- CREATE TOGGLES AND SLIDERS
-- =============================================

createToggle("Auto Farm", "⚔", Color3.fromRGB(100, 200, 255), function(enabled)
    isRunning = enabled
    if not enabled then
        currentTarget = nil
        currentMonsterIndex = 1
        allMonsters = {}
    end
end)

createToggle("Auto Collect", "💎", Color3.fromRGB(255, 200, 100), function(enabled)
    autoCollectEnabled = enabled
end)

createToggle("Auto Heal", "❤", Color3.fromRGB(255, 100, 100), function(enabled)
    autoHeal = enabled
end)

createToggle("Auto Retry", "🔄", Color3.fromRGB(150, 255, 150), function(enabled)
    autoRetry = enabled
end)

createToggle("Safe Mode", "🛡", Color3.fromRGB(255, 150, 50), function(enabled)
    safeModeEnabled = enabled
    if not enabled then
        safeMode = false
        removeSafePlatform()
    end
end)

createSlider("Health Threshold", "💊", Color3.fromRGB(255, 100, 150), 1, 100, HEALTH_THRESHOLD, function(value)
    HEALTH_THRESHOLD = value
end)

-- =============================================
-- BACKGROUND TASKS
-- =============================================

-- Auto Collect Loop
task.spawn(function()
    while task.wait(0.3) do
        if isRunning and autoCollectEnabled then
            collectAllDrops()
        end
    end
end)

-- Health Monitor and Safe Mode
task.spawn(function()
    while task.wait(0.5) do
        local healthPercent = getHealthPercent()
        HealthLabel.Text = "❤ Health: " .. math.floor(healthPercent) .. "%"
        
        if safeModeEnabled then
            if healthPercent <= SAFE_MODE_THRESHOLD and not safeMode then
                safeMode = true
                isRunning = false
                currentTarget = nil
                teleportToSafeZone()
                StatusLabel.Text = "🛡 Status: SAFE MODE - Healing!"
                StatusLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
                StatusStroke.Color = Color3.fromRGB(255, 100, 100)
                
                task.spawn(function()
                    while safeMode and getHealthPercent() < SAFE_MODE_RECOVER do
                        pcall(function()
                            ReplicatedStorage:WaitForChild("ReplicatedStorage"):WaitForChild("Packages"):WaitForChild("Knit"):WaitForChild("Services"):WaitForChild("HealingService"):WaitForChild("RF"):WaitForChild("UseHeal"):InvokeServer()
                        end)
                        task.wait(1)
                    end
                end)
                
            elseif healthPercent >= SAFE_MODE_RECOVER and safeMode then
                safeMode = false
                removeSafePlatform()
                safePlatformPosition = nil
                StatusLabel.Text = "🛡 Status: Normal Mode"
                StatusLabel.TextColor3 = Color3.fromRGB(100, 255, 100)
                StatusStroke.Color = Color3.fromRGB(100, 255, 100)
            end
        end
        
        if currentTarget then
            TargetLabel.Text = "🎯 Target: " .. currentTarget.Name .. " (" .. currentMonsterIndex .. "/" .. #allMonsters .. ")"
        else
            TargetLabel.Text = "🎯 Target: Searching..."
        end
    end
end)

-- Auto Heal Loop
task.spawn(function()
    while task.wait(1) do
        if autoHeal and isRunning and not safeMode then
            local healthPercent = getHealthPercent()
            if healthPercent <= HEALTH_THRESHOLD then
                pcall(function()
                    ReplicatedStorage:WaitForChild("ReplicatedStorage"):WaitForChild("Packages"):WaitForChild("Knit"):WaitForChild("Services"):WaitForChild("HealingService"):WaitForChild("RF"):WaitForChild("UseHeal"):InvokeServer()
                    task.wait(0.5)
                end)
            end
        end
    end
end)

-- Auto Start Dungeon
local dungeonStarting = false
task.spawn(function()
    while task.wait(0.5) do
        if isRunning then
            pcall(function()
                local startBtn = PlayerGui:FindFirstChild("Main")
                if startBtn then
                    startBtn = startBtn:FindFirstChild("StartBtn")
                    if startBtn and startBtn.Visible then
                        if not dungeonStarting then
                            dungeonStarting = true
                            ReplicatedStorage:WaitForChild("ReplicatedStorage"):WaitForChild("Packages"):WaitForChild("Knit"):WaitForChild("Services"):WaitForChild("DungeonService"):WaitForChild("RF"):WaitForChild("StartDungeon"):InvokeServer()
                            task.wait(2)
                        end
                    else
                        dungeonStarting = false
                    end
                end
            end)
        end
    end
end)

-- Auto Retry Loop
task.spawn(function()
    while task.wait(2) do
        if autoRetry then
            pcall(function()
                local args = {"Retry"}
                ReplicatedStorage:WaitForChild("ReplicatedStorage"):WaitForChild("Packages"):WaitForChild("Knit"):WaitForChild("Services"):WaitForChild("PartyService"):WaitForChild("RF"):WaitForChild("VoteOn"):InvokeServer(unpack(args))
            end)
        end
    end
end)

-- Can Attack Monitor
local canAttack = true
task.spawn(function()
    while task.wait(0.1) do
        pcall(function()
            local startBtn = PlayerGui:FindFirstChild("Main")
            if startBtn then
                startBtn = startBtn:FindFirstChild("StartBtn")
                if startBtn and startBtn.Visible then
                    canAttack = false
                else
                    canAttack = true
                end
            end
        end)
    end
end)

-- Main Farm Loop (Heartbeat)
RunService.Heartbeat:Connect(function()
    if not isRunning or safeMode then return end
    pcall(function()
        Character = Player.Character
        if not Character then return end
        HumanoidRootPart = Character:FindFirstChild("HumanoidRootPart")
        Humanoid = Character:FindFirstChild("Humanoid")
        if not HumanoidRootPart or not Humanoid then return end
        
        setupNoClip()
        disableAnimations()
        
        if not currentTarget or not currentTarget.Parent or 
           not currentTarget:FindFirstChild("Humanoid") or 
           currentTarget.Humanoid.Health <= 0 then
            currentTarget = getNextMonster()
            task.wait(0.5)
        end
        
        if currentTarget and canAttack then
            teleportBehindMonster(currentTarget)
            lockOnTarget(currentTarget)
        end
    end)
end)

-- Attack Loop
task.spawn(function()
    while task.wait(ATTACK_SPEED) do
        if isRunning and currentTarget and canAttack and not safeMode then
            attackMonster()
        end
    end
end)

-- Character Respawn Handler
Player.CharacterAdded:Connect(function(newChar)
    Character = newChar
    task.wait(0.5)
    HumanoidRootPart = newChar:WaitForChild("HumanoidRootPart")
    Humanoid = newChar:WaitForChild("Humanoid")
    currentTarget = nil
    currentMonsterIndex = 1
    allMonsters = {}
    safeMode = false
    removeSafePlatform()
    safePlatformPosition = nil
end)
