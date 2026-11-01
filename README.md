--!strict
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- Secure GUI container resolution (UNC gethui fallback)
local function getGuiParent(): Instance
    if typeof(gethui) == "function" then
        local success, hui = pcall(gethui)
        if success and hui then
            return hui
        end
    end
    return LocalPlayer:WaitForChild("PlayerGui")
end

-- ==========================================
-- 1. UI SETUP (ENGLISH LABELS)
-- ==========================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "TriggerbotGui"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = getGuiParent()

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0, 210, 0, 135)
mainFrame.Position = UDim2.new(0.05, 0, 0.2, 0)
mainFrame.BackgroundColor3 = Color3.fromRGB(22, 22, 26)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 8)
corner.Parent = mainFrame

local titleLabel = Instance.new("TextLabel")
titleLabel.Name = "TitleLabel"
titleLabel.Size = UDim2.new(1, 0, 0, 28)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "Auto Triggerbot"
titleLabel.TextColor3 = Color3.fromRGB(240, 240, 240)
titleLabel.Font = Enum.Font.GothamBold
titleLabel.TextSize = 13
titleLabel.Parent = mainFrame

-- Trigger Button (Toggle 1)
local triggerBtn = Instance.new("TextButton")
triggerBtn.Name = "TriggerButton"
triggerBtn.Size = UDim2.new(0.88, 0, 0, 38)
triggerBtn.Position = UDim2.new(0.06, 0, 0.28, 0)
triggerBtn.BackgroundColor3 = Color3.fromRGB(190, 40, 40)
triggerBtn.Text = "Trigger: DISABLED"
triggerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
triggerBtn.Font = Enum.Font.GothamBold
triggerBtn.TextSize = 12
triggerBtn.AutoButtonColor = true
triggerBtn.Parent = mainFrame

local triggerBtnCorner = Instance.new("UICorner")
triggerBtnCorner.CornerRadius = UDim.new(0, 6)
triggerBtnCorner.Parent = triggerBtn

-- Team Check Button (Toggle 2)
local teamCheckBtn = Instance.new("TextButton")
teamCheckBtn.Name = "TeamCheckButton"
teamCheckBtn.Size = UDim2.new(0.88, 0, 0, 38)
teamCheckBtn.Position = UDim2.new(0.06, 0, 0.63, 0)
teamCheckBtn.BackgroundColor3 = Color3.fromRGB(40, 160, 80)
teamCheckBtn.Text = "Team Check: ENABLED"
teamCheckBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
teamCheckBtn.Font = Enum.Font.GothamBold
teamCheckBtn.TextSize = 12
teamCheckBtn.AutoButtonColor = true
teamCheckBtn.Parent = mainFrame

local teamCheckCorner = Instance.new("UICorner")
teamCheckCorner.CornerRadius = UDim.new(0, 6)
teamCheckCorner.Parent = teamCheckBtn

-- ==========================================
-- 2. STATE & TOGGLE HANDLERS
-- ==========================================
local isTriggerEnabled: boolean = false
local isTeamCheckEnabled: boolean = true
local lastClickTime: number = 0
local MIN_DELAY: number = 0.05 -- Firing recovery delay

triggerBtn.MouseButton1Click:Connect(function()
    isTriggerEnabled = not isTriggerEnabled
    if isTriggerEnabled then
        triggerBtn.BackgroundColor3 = Color3.fromRGB(40, 160, 80)
        triggerBtn.Text = "Trigger: ENABLED"
    else
        triggerBtn.BackgroundColor3 = Color3.fromRGB(190, 40, 40)
        triggerBtn.Text = "Trigger: DISABLED"
    end
end)

teamCheckBtn.MouseButton1Click:Connect(function()
    isTeamCheckEnabled = not isTeamCheckEnabled
    if isTeamCheckEnabled then
        teamCheckBtn.BackgroundColor3 = Color3.fromRGB(40, 160, 80)
        teamCheckBtn.Text = "Team Check: ENABLED"
    else
        teamCheckBtn.BackgroundColor3 = Color3.fromRGB(190, 40, 40)
        teamCheckBtn.Text = "Team Check: DISABLED"
    end
end)

-- ==========================================
-- 3. ENEMY & TEAM IDENTIFICATION
-- ==========================================
local function isEnemy(player: Player): boolean
    if player == LocalPlayer then
        return false
    end

    if isTeamCheckEnabled then
        -- Check if teams are assigned and match
        if LocalPlayer.Team ~= nil and player.Team ~= nil then
            if LocalPlayer.Team == player.Team then
                return false
            end
        end

        -- Check TeamColor fallback
        if not LocalPlayer.Neutral and not player.Neutral then
            if LocalPlayer.TeamColor == player.TeamColor then
                return false
            end
        end
    end

    return true
end

local function getTargetPlayer(part: Instance?): (Player?, Model?)
    if not part then return nil, nil end

    local character = part:FindFirstAncestorOfClass("Model")
    if not character or character == LocalPlayer.Character then
        return nil, nil
    end

    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then
        return nil, nil
    end

    local player = Players:GetPlayerFromCharacter(character)
    if player and isEnemy(player) then
        return player, character
    end

    return nil, nil
end

-- ==========================================
-- 4. HARDWARE DISPATCH (M1 CLICK)
-- ==========================================
local function sendClick()
    local mousePos = UserInputService:GetMouseLocation()

    -- Native Roblox input event dispatch
    local vimSuccess = pcall(function()
        VirtualInputManager:SendMouseButtonEvent(mousePos.X, mousePos.Y, 0, true, game, 1)
        task.delay(0.015, function()
            VirtualInputManager:SendMouseButtonEvent(mousePos.X, mousePos.Y, 0, false, game, 1)
        end)
    end)

    if vimSuccess then return end

    -- Fallback UNC method
    if typeof(mouse1click) == "function" then
        mouse1click()
        return
    elseif typeof(mouse1press) == "function" and typeof(mouse1release) == "function" then
        mouse1press()
        task.delay(0.015, mouse1release)
        return
    end

    -- Classic Tool activation fallback
    local char = LocalPlayer.Character
    if char then
        local tool = char:FindFirstChildOfClass("Tool")
        if tool then
            tool:Activate()
        end
    end
end

-- ==========================================
-- 5. RENDER LOOP
-- ==========================================
RunService.RenderStepped:Connect(function()
    if not isTriggerEnabled then return end

    local now = os.clock()
    if now - lastClickTime < MIN_DELAY then
        return
    end

    local mousePos = UserInputService:GetMouseLocation()
    local unitRay = Camera:ViewportPointToRay(mousePos.X, mousePos.Y)

    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Exclude
    rayParams.IgnoreWater = true

    local ignoreList = {}
    if LocalPlayer.Character then
        table.insert(ignoreList, LocalPlayer.Character)
    end
    rayParams.FilterDescendantsInstances = ignoreList

    local result = Workspace:Raycast(unitRay.Origin, unitRay.Direction * 1500, rayParams)

    if result and result.Instance then
        local targetPlayer = getTargetPlayer(result.Instance)
        if targetPlayer then
            lastClickTime = now
            sendClick()
        end
    end
end)
