--// 💰 ระบบโยกเงิน + กันตาย (Fluent UI) + Rejoin by สคริป (ปรับระบบเดินใหม่ Tween + Vault)

-- ====== Services ======
local Players = game:GetService("Players")
local PathfindingService = game:GetService("PathfindingService")
local RunService = game:GetService("RunService")
local TeleportService = game:GetService("TeleportService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local hum = char:WaitForChild("Humanoid")
local playerGui = LocalPlayer:WaitForChild("PlayerGui")

-- ====== Fluent UI ======
local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
local SaveManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua"))()
local InterfaceManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua"))()

local Window = Fluent:CreateWindow({
    Title="โยกเงิน",
    SubTitle="by สคริป",
    TabWidth=160,
    Size=UDim2.fromOffset(580,460),
    Acrylic=true,
    Theme="Dark"
})

-- ====== Variables ======
local amount = 1 -- เริ่มต้นขั้นต่ำ 1
local walkSpeed = 25
local autoWithdraw = false
local antiDeathEnabled = false

-- ====== Anti-Death Config ======
local HP_THRESHOLD = 30
local EVADE_TELEPORT_Y = -30
local DEBOUNCE_TIME = 0.3
local lastEvadeTime = 0

-- ====== Tween + Pathfinding + Vault ระบบเดิน ======
local activeTween
local isWalking = false
local WALK_SPEED = walkSpeed
local VAULT_HEIGHT = 10
local VAULT_TIME = 0.4

local function vaultOver(obstaclePos)
    local upPos = Vector3.new(obstaclePos.X, obstaclePos.Y + VAULT_HEIGHT, obstaclePos.Z)
    local tweenUp = TweenService:Create(hrp, TweenInfo.new(VAULT_TIME), {CFrame = CFrame.new(upPos)})
    tweenUp:Play()
    tweenUp.Completed:Wait()

    local downTween = TweenService:Create(hrp, TweenInfo.new(VAULT_TIME), {CFrame = CFrame.new(obstaclePos)})
    downTween:Play()
    downTween.Completed:Wait()
end

local function moveToPosition(pos)
    if activeTween then activeTween:Cancel() end
    local dist = (hrp.Position - pos).Magnitude
    local t = dist / WALK_SPEED
    local tween = TweenService:Create(hrp, TweenInfo.new(t, Enum.EasingStyle.Linear), {CFrame = CFrame.new(pos)})
    activeTween = tween
    tween:Play()
    tween.Completed:Wait()
end

local function followPath(path)
    local waypoints = path:GetWaypoints()
    for _, wp in ipairs(waypoints) do
        if not isWalking then break end
        if wp.Action == Enum.PathWaypointAction.Jump then
            vaultOver(wp.Position)
        else
            moveToPosition(wp.Position)
        end
    end
end

function smartWalkTo(targetPos, speed)
    if isWalking then return end
    isWalking = true
    WALK_SPEED = speed or WALK_SPEED

    local path = PathfindingService:CreatePath({
        AgentRadius = 2,
        AgentHeight = 5,
        AgentCanJump = true,
        WaypointSpacing = 4
    })
    path:ComputeAsync(hrp.Position, targetPos)

    if path.Status == Enum.PathStatus.Success then
        followPath(path)
    else
        warn("❌ Pathfinding ล้มเหลว:", targetPos)
    end

    isWalking = false
end

-- ====== ฟังก์ชัน Anti-Death ======
local function doEvade()
    if not hrp or not hrp.Parent then return end
    if tick() - lastEvadeTime < DEBOUNCE_TIME then return end
    lastEvadeTime = tick()
    pcall(function()
        hrp.CFrame = hrp.CFrame + Vector3.new(0, EVADE_TELEPORT_Y, 0)
    end)
end

RunService.Heartbeat:Connect(function()
    if antiDeathEnabled and hum and hum.Health <= HP_THRESHOLD then
        doEvade()
    end
end)

-- ====== Tabs ======
local Tabs = {}
Tabs.Main = Window:AddTab({Title="Main"})
Tabs.Rejoin = Window:AddTab({Title="Rejoin"})
Tabs.Settings = Window:AddTab({Title="Settings"})

-- ====== Main Tab ======
Tabs.Main:AddInput("AmountInput",{
    Title="จำนวนเงินที่ต้องการถอน (ขั้นต่ำ 1)",
    Placeholder="เช่น 500",
    Default="1",
    Description="💡 แนะนำ: ประมาณ 10000",
    Callback=function(value)
        local val = tonumber(value)
        if val and val >= 1 then
            amount = val
        else
            amount = 1
        end
    end
})

Tabs.Main:AddToggle("AutoWithdrawToggle",{Title="ถอนเงินอัตโนมัติ",Default=false,Callback=function(v) autoWithdraw=v end})
Tabs.Main:AddToggle("AntiDeathToggle",{Title="กันตายอัตโนมัติ",Default=false,Callback=function(v)
    antiDeathEnabled = v
end})

Tabs.Main:AddButton({
    Title="▶ เริ่มระบบอัตโนมัติ",
    Description="เดินไปถอนเงิน → ถอน → เดินไปจุดใหม่ → วาปตามลำดับ",
    Callback=function()
        task.spawn(function()
            hum.WalkSpeed = walkSpeed
            if antiDeathEnabled then
                Fluent:Notify({Title="กันตาย",Content="เปิดระบบกันตายชั่วคราว ✅",Duration=2})
            end

            if autoWithdraw and amount >= 1 then
                -- ===== จุดถอนเงิน =====
                local withdrawPos = Vector3.new(834.0,254.7,-162.1)
                Fluent:Notify({Title="ถอนเงิน",Content="เดินไปยังจุดถอนเงิน...",Duration=3})
                smartWalkTo(withdrawPos, walkSpeed)
                task.wait(1)

                -- ===== ถอนเงิน =====
                local args = {8,"transfer_funds","bank","hand",amount}
                pcall(function()
                    ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("Get"):InvokeServer(unpack(args))
                end)
                Fluent:Notify({Title="ถอนเงิน",Content="ถอนเงินสำเร็จ: "..amount.." ✅",Duration=3})

                -- ===== เดินไปจุดก่อนวาป =====
                local firstPos = Vector3.new(920.4, 254.7, -150.4)
                Fluent:Notify({Title="กำลังเดิน",Content="ไปจุดเตรียมวาป...",Duration=2})
                smartWalkTo(firstPos, walkSpeed)
                task.wait(0.5)

                -- ===== วาปออโต้ตามจุด =====
                local teleportPoints = {
                    Vector3.new(914.8, 254.7, -150.3),
                    Vector3.new(914.1, 254.7, -149.3),
                    Vector3.new(907.1, 254.7, -151.4),
                    Vector3.new(898.1, 254.7, -152.5),
                    Vector3.new(890.6, 254.7, -149.9),
                    Vector3.new(883.6, 254.7, -147.5)
                }

                for _, point in ipairs(teleportPoints) do
                    if not hrp or not hrp.Parent then break end
                    hrp.CFrame = CFrame.new(point)
                    task.wait(0.2)
                end

                Fluent:Notify({Title="วาปสำเร็จ",Content="วาปครบทุกจุดเรียบร้อย ✅",Duration=3})
            end
        end)
    end
})

-- ====== Rejoin Tab ======
local rejoinTime = 1
local rejoinRunning = false

Tabs.Rejoin:AddSlider("RejoinTimeSlider",{
    Title="เวลา Rejoin (นาที)",
    Description="ตั้งเวลา 1-5 นาที",
    Default=1, Min=1, Max=5, Rounding=1,
    Callback=function(value)
        rejoinTime = value
    end
})

Tabs.Rejoin:AddButton({
    Title="▶ เริ่มนับถอยหลัง Rejoin",
    Description="นับถอยหลังแล้วรีจอยอัตโนมัติ",
    Callback=function()
        if rejoinRunning then return end
        rejoinRunning = true

        local ScreenGui = Instance.new("ScreenGui")
        ScreenGui.Name = "RejoinCountdownUI"
        ScreenGui.ResetOnSpawn = false
        ScreenGui.Parent = playerGui

        local Frame = Instance.new("Frame")
        Frame.Size = UDim2.new(0,150,0,60)
        Frame.Position = UDim2.new(0,10,0,50)
        Frame.BackgroundColor3 = Color3.fromRGB(40,40,40)
        Frame.BorderSizePixel = 0
        Frame.Parent = ScreenGui

        local Label = Instance.new("TextLabel")
        Label.Size = UDim2.new(1,0,1,0)
        Label.BackgroundTransparency = 1
        Label.TextColor3 = Color3.new(1,1,1)
        Label.Font = Enum.Font.GothamBold
        Label.TextScaled = true
        Label.Text = tostring(rejoinTime*60)
        Label.Parent = Frame

        -- Dragging
        local dragging = false
        local dragInput, mousePos, framePos

        Frame.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = true
                mousePos = input.Position
                framePos = Frame.Position
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragging = false
                    end
                end)
            end
        end)

        Frame.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement then
                dragInput = input
            end
        end)

        UserInputService.InputChanged:Connect(function(input)
            if input == dragInput and dragging then
                local delta = input.Position - mousePos
                Frame.Position = UDim2.new(framePos.X.Scale, framePos.X.Offset + delta.X,
                                           framePos.Y.Scale, framePos.Y.Offset + delta.Y)
            end
        end)

        -- Countdown
        task.spawn(function()
            local totalSeconds = rejoinTime * 60
            while totalSeconds >= 0 do
                Label.Text = tostring(totalSeconds)
                task.wait(1)
                totalSeconds = totalSeconds - 1
            end
            TeleportService:Teleport(game.PlaceId, LocalPlayer)
        end)
    end
})

-- ====== Save & Interface Manager ======
SaveManager:SetLibrary(Fluent)
InterfaceManager:SetLibrary(Fluent)
SaveManager:IgnoreThemeSettings()
SaveManager:SetFolder("FluentScriptHub/specific-game")
InterfaceManager:SetFolder("FluentScriptHub")
InterfaceManager:BuildInterfaceSection(Tabs.Settings)
SaveManager:BuildConfigSection(Tabs.Settings)
Window:SelectTab(1)

Fluent:Notify({Title="Fluent",Content="Script โหลดเสร็จแล้ว ✅",Duration=5})
