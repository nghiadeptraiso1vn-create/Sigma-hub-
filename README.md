# Sigma-hub-
Script xịn☠️
-- Sigma Hub (clean) -- by nghia
-- Features:
-- 1) CheckKey GUI (GET KEY / CHECK KEY)
-- 2) Permanent key "sigmaboss" + optional remote key list (pastebin/raw URL)
-- 3) On valid key -> open Hub: rotating markers, auto-attack when holding Tool, auto pickup (ProximityPrompt),
--    HP billboard following selected enemy, toggle keys
-- 4) Compatible with executors like KRNL (parents GUI to CoreGui)

-- CONFIG (edit these values)
local PERMANENT_KEY = "sigmaboss"                         -- vĩnh viễn
local KEY_LIST_URL = nil                                  -- ví dụ: "https://pastebin.com/raw/XXXXX" or nil to disable
local CHECK_REMOTE = true                                 -- nếu true, script sẽ pcall HttpGet(KEY_LIST_URL) và so sánh
local AUTO_PICKUP_INTERVAL = 0.6                          -- giây giữa lần scan ProximityPrompt
local PICKUP_RANGE = 16                                   -- studs
local ROTATE_MARKER_COUNT = 18
local ROTATE_RADIUS = 6
local ROTATE_SPEED = 1.6                                  -- revolutions per second
local ATTACK_DELAY = 0.8                                  -- giây giữa các activate tool
local HUB_GUI_TITLE = "Sigma Hub 💀 by nghia🤡"

-- ===== services & helpers =====
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local CoreGui = game:GetService("CoreGui")
local workspace = workspace

local localPlayer = Players.LocalPlayer
assert(localPlayer, "This script must run in a player context (executor or LocalScript).")

local function safeDestroy(obj)
    if obj and obj.Parent then
        pcall(function() obj:Destroy() end)
    end
end

local function shortMsg(parent, text, color)
    color = color or Color3.fromRGB(255,255,255)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -40, 0, 28)
    lbl.Position = UDim2.new(0, 20, 0, parent.AbsoluteSize.Y - 38)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.Font = Enum.Font.GothamBold
    lbl.TextSize = 18
    lbl.TextColor3 = color
    lbl.TextXAlignment = Enum.TextXAlignment.Center
    lbl.Parent = parent
    spawn(function()
        wait(2.2)
        pcall(function() lbl:Destroy() end)
    end)
end

-- ===== KEY CHECKING =====
local function remoteKeysFetch(url)
    if not url then return nil end
    local ok, res = pcall(function()
        return game:HttpGet(url)
    end)
    if not ok or not res then return nil end
    -- split lines, trim
    local t = {}
    for line in res:gmatch("[^\r\n]+") do
        local s = string.gsub(line, "^%s*(.-)%s*$", "%1")
        if s ~= "" then t[string.lower(s)] = true end
    end
    return t
end

local function isKeyValid(inputKey)
    if not inputKey then return false end
    local k = string.lower(tostring(inputKey))
    if k == string.lower(PERMANENT_KEY) then return true end
    if CHECK_REMOTE and KEY_LIST_URL then
        local keys = remoteKeysFetch(KEY_LIST_URL)
        if keys and keys[k] then return true end
    end
    return false
end

-- ===== BUILD CHECK-KEY GUI =====
local function buildKeyGui()
    safeDestroy(CoreGui:FindFirstChild("SigmaKeyGui"))
    local gui = Instance.new("ScreenGui")
    gui.Name = "SigmaKeyGui"
    gui.ResetOnSpawn = false
    gui.Parent = CoreGui

    -- root frame (center)
    local root = Instance.new("Frame")
    root.Size = UDim2.new(1, -60, 0, 260)
    root.Position = UDim2.new(0, 30, 0.04, 0)
    root.BackgroundTransparency = 0.32
    root.BackgroundColor3 = Color3.fromRGB(8,8,8)
    root.BorderSizePixel = 0
    root.Parent = gui

    -- top buttons and title
    local btnLeft = Instance.new("TextButton")
    btnLeft.Name = "GetKey"
    btnLeft.Size = UDim2.new(0.16, 0, 0, 60)
    btnLeft.Position = UDim2.new(0.02, 0, 0, 10)
    btnLeft.BackgroundColor3 = Color3.fromRGB(35,35,35)
    btnLeft.BorderSizePixel = 0
    btnLeft.Font = Enum.Font.GothamBold
    btnLeft.TextSize = 24
    btnLeft.Text = "GET KEY"
    btnLeft.Parent = root

    local btnRight = Instance.new("TextButton")
    btnRight.Name = "CheckKey"
    btnRight.Size = UDim2.new(0.16, 0, 0, 60)
    btnRight.Position = UDim2.new(0.82, 0, 0, 10)
    btnRight.BackgroundColor3 = Color3.fromRGB(35,35,35)
    btnRight.BorderSizePixel = 0
    btnRight.Font = Enum.Font.GothamBold
    btnRight.TextSize = 24
    btnRight.Text = "CHECK KEY"
    btnRight.Parent = root

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(0.6, 0, 0, 60)
    title.Position = UDim2.new(0.2, 0, 0, 10)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.TextSize = 30
    title.Text = "CHECK KEY"
    title.TextColor3 = Color3.fromRGB(255,255,255)
    title.TextXAlignment = Enum.TextXAlignment.Center
    title.Parent = root

    -- description
    local desc = Instance.new("TextLabel")
    desc.Size = UDim2.new(1, -40, 0, 40)
    desc.Position = UDim2.new(0, 20, 0, 80)
    desc.BackgroundTransparency = 1
    desc.Font = Enum.Font.Gotham
    desc.TextSize = 20
    desc.TextColor3 = Color3.fromRGB(200,200,200)
    desc.Text = "Nhập mật khẩu ở ô giữa. GET KEY sẽ copy link nhận key."
    desc.TextXAlignment = Enum.TextXAlignment.Center
    desc.Parent = root

    -- big centered text box look
    local boxBg = Instance.new("Frame")
    boxBg.Size = UDim2.new(0.9, 0, 0, 80)
    boxBg.Position = UDim2.new(0.05, 0, 0, 140)
    boxBg.BackgroundColor3 = Color3.fromRGB(30,30,30)
    boxBg.BorderSizePixel = 0
    boxBg.Parent = root
    local corner = Instance.new("UICorner", boxBg)
    corner.CornerRadius = UDim.new(0, 14)

    local input = Instance.new("TextBox")
    input.Parent = boxBg
    input.BackgroundTransparency = 1
    input.Size = UDim2.new(1, -30, 1, -18)
    input.Position = UDim2.new(0, 15, 0, 9)
    input.Font = Enum.Font.Gotham
    input.PlaceholderText = "Nhập mật khẩu ở đây..."
    input.TextSize = 26
    input.TextColor3 = Color3.fromRGB(255,255,255)
    input.TextXAlignment = Enum.TextXAlignment.Center

    -- small message label
    local msg = Instance.new("TextLabel")
    msg.Parent = root
    msg.Size = UDim2.new(1, -40, 0, 28)
    msg.Position = UDim2.new(0, 20, 0, 230)
    msg.BackgroundTransparency = 1
    msg.Font = Enum.Font.Gotham
    msg.TextSize = 18
    msg.TextColor3 = Color3.fromRGB(200,200,200)
    msg.TextXAlignment = Enum.TextXAlignment.Center

    -- handlers
    btnLeft.MouseButton1Click:Connect(function()
        -- try to copy link (some executors provide setclipboard)
        local ok = pcall(function()
            if setclipboard then setclipboard("https://example.com/getkey") end
        end)
        msg.Text = "Key vĩnh viễn: " .. PERMANENT_KEY .. (ok and " (link đã copy)" or "")
        delay(2.0, function() if msg then msg.Text = "" end end)
    end)

    local function showMsgText(t, col)
        msg.TextColor3 = col or Color3.fromRGB(255,255,255)
        msg.Text = t
        delay(2.0, function() if msg then msg.Text = "" end end)
    end

    local function openHub()
        safeDestroy(gui)
        spawn(function() startHub() end)
    end

    btnRight.MouseButton1Click:Connect(function()
        local key = tostring(input.Text or "")
        if isKeyValid(key) then
            showMsgText("✅ Key chính xác! Mở hub...", Color3.fromRGB(0,255,0))
            wait(0.8)
            openHub()
        else
            showMsgText("❌ Sai key rồi 🤡", Color3.fromRGB(255,80,80))
        end
    end)

    -- also allow Enter key to validate
    input.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            btnRight.MouseButton1Click:Fire()
        end
    end)

    return gui
end

-- ===== HUB (rotate markers + auto features) =====
local hubGuiInstance = nil
local rotateModel = nil
local hubRunning = false

-- state
local currentTool = nil
local lastAttackTime = 0
local selectedHumanoid = nil
local hpBillboard = nil

-- tool tracking
local function onToolEquipped(tool)
    currentTool = tool
end
local function onToolUnequipped(tool)
    if currentTool == tool then currentTool = nil end
end

local function attachToolListeners(character)
    if not character then return end
    character.ChildAdded:Connect(function(c)
        if c:IsA("Tool") then onToolEquipped(c) end
    end)
    character.ChildRemoved:Connect(function(c)
        if c:IsA("Tool") and currentTool == c then onToolUnequipped(c) end
    end)
    -- initial
    for _,c in pairs(character:GetChildren()) do
        if c:IsA("Tool") then currentTool = c break end
    end
end

-- rotating markers
local function createRotateMarkers(count)
    safeDestroy(rotateModel)
    local model = Instance.new("Model")
    model.Name = "SigmaRotate"
    model.Parent = workspace
    local parts = {}
    for i = 1, count do
        local p = Instance.new("Part")
        p.Size = Vector3.new(0.3,0.3,0.3)
        p.Anchored = true
        p.CanCollide = false
        p.Material = Enum.Material.Neon
        p.Shape = Enum.PartType.Ball
        p.Transparency = 0
        p.Parent = model
        table.insert(parts, p)
    end
    return model, parts
end

local rotateParts = {}
local rotateAngle = 0

-- attack loop (RenderStepped)
local function attackTick(dt)
    if not currentTool then return end
    lastAttackTime = lastAttackTime + dt
    if lastAttackTime >= ATTACK_DELAY then
        lastAttackTime = 0
        pcall(function()
            if currentTool and currentTool.Parent then
                if currentTool.Activate then
                    currentTool:Activate()
                elseif currentTool:FindFirstChild("Activate") and type(currentTool.Activate) == "function" then
                    currentTool:Activate()
                end
            end
        end)
    end
end

-- auto pickup loop
local function autoPickupLoop()
    while hubRunning do
        if not workspace or not localPlayer.Character or not localPlayer.Character:FindFirstChild("HumanoidRootPart") then
            wait(AUTO_PICKUP_INTERVAL)
        else
            local hrp = localPlayer.Character.HumanoidRootPart
            for _, obj in ipairs(workspace:GetDescendants()) do
                if obj:IsA("ProximityPrompt") then
                    local parentPart = obj.Parent
                    if parentPart and parentPart:IsA("BasePart") then
                        local ok, dist = pcall(function() return (parentPart.Position - hrp.Position).Magnitude end)
                        if ok and dist and dist <= PICKUP_RANGE then
                            pcall(function()
                                -- try InputHoldBegin/End where supported, else :Triggered()
                                if obj.HoldDuration and obj.HoldDuration > 0 then
                                    obj:InputHoldBegin()
                                    wait(0.05)
                                    obj:InputHoldEnd()
                                else
                                    obj:InputHoldBegin()
                                    obj:InputHoldEnd()
                                end
                            end)
                        end
                    end
                end
            end
            wait(AUTO_PICKUP_INTERVAL)
        end
    end
end

-- HP Billboard creation
local function createHPBillboardForHumanoid(humanoid)
    safeDestroy(hpBillboard)
    if not humanoid or not humanoid.Parent then return end
    local adornee = humanoid.Parent:FindFirstChild("HumanoidRootPart") or humanoid.Parent:FindFirstChildWhichIsA("BasePart")
    if not adornee then return end

    local bg = Instance.new("BillboardGui")
    bg.Name = "SigmaHP"
    bg.Adornee = adornee
    bg.Size = UDim2.new(0,140,0,40)
    bg.StudsOffset = Vector3.new(0, 3.2, 0)
    bg.AlwaysOnTop = true
    bg.Parent = workspace

    local frame = Instance.new("Frame", bg)
    frame.Size = UDim2.new(1,0,1,0)
    frame.BackgroundTransparency = 0.25
    frame.BackgroundColor3 = Color3.fromRGB(0,0,0)

    local hpBar = Instance.new("Frame", frame)
    hpBar.Size = UDim2.new(1,0,0,12)
    hpBar.Position = UDim2.new(0,0,0.12,0)
    hpBar.BackgroundColor3 = Color3.fromRGB(40,200,80)
    hpBar.BorderSizePixel = 0
    local hpLabel = Instance.new("TextLabel", frame)
    hpLabel.Size = UDim2.new(1,0,0,18)
    hpLabel.Position = UDim2.new(0,0,0,18)
    hpLabel.BackgroundTransparency = 1
    hpLabel.Font = Enum.Font.Gotham
    hpLabel.TextSize = 13
    hpLabel.TextColor3 = Color3.fromRGB(255,255,255)

    -- updater
    spawn(function()
        while bg and humanoid and humanoid.Parent and humanoid.Health >= 0 do
            pcall(function()
                local pct = 0
                if humanoid.MaxHealth > 0 then pct = math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1) end
                hpBar.Size = UDim2.new(pct, 0, 0, 12)
                hpLabel.Text = string.format("HP: %d / %d", math.floor(humanoid.Health), math.floor(humanoid.MaxHealth))
            end)
            wait(0.13)
        end
        safeDestroy(bg)
    end)
    hpBillboard = bg
end

-- select target by click
local function enableTargetSelection()
    local mouse = localPlayer:GetMouse()
    mouse.Button1Down:Connect(function()
        local target = mouse.Target
        if not target then return end
        local mdl = target:FindFirstAncestorOfClass("Model")
        if mdl and mdl:FindFirstChildOfClass("Humanoid") then
            selectedHumanoid = mdl:FindFirstChildOfClass("Humanoid")
            createHPBillboardForHumanoid(selectedHumanoid)
        end
    end)
end

-- toggles
local rotateEnabled = true
local attackEnabled = true

-- main startHub
function startHub()
    if hubRunning then return end
    hubRunning = true

    -- small notification GUI
    local notifyGui = Instance.new("ScreenGui")
    notifyGui.Name = "SigmaHubGUI"
    notifyGui.Parent = CoreGui
    local label = Instance.new("TextLabel")
    label.Parent = notifyGui
    label.Size = UDim2.new(0, 300, 0, 28)
    label.Position = UDim2.new(0, 10, 0, 10)
    label.BackgroundTransparency = 0.3
    label.BackgroundColor3 = Color3.fromRGB(12,12,12)
    label.Text = HUB_GUI_TITLE .. " - OPEN"
    label.Font = Enum.Font.Gotham
    label.TextSize = 16
    label.TextColor3 = Color3.fromRGB(255,255,255)

    -- prepare rotating markers
    rotateModel, rotateParts = createRotateMarkers(ROTATE_MARKER_COUNT)

    -- connect character tool tracking
    if localPlayer.Character then attachToolListeners(localPlayer.Character) end
    localPlayer.CharacterAdded:Connect(function(char)
        attachToolListeners(char)
    end)

    -- enable selecting target
    enableTargetSelection()

    -- start auto pickup
    spawn(autoPickupLoop)

    -- RenderStepped loop for rotate & attack
    RunService.RenderStepped:Connect(function(dt)
        -- rotation
        if rotateEnabled and localPlayer.Character and localPlayer.Character:FindFirstChild("HumanoidRootPart") then
            rotateAngle = rotateAngle + dt * (ROTATE_SPEED * math.pi * 2)
            local hrp = localPlayer.Character.HumanoidRootPart
            for i,p in ipairs(rotateParts) do
                local frac = (i-1)/#rotateParts
                local angle = rotateAngle + frac * math.pi * 2
                local x = math.cos(angle) * ROTATE_RADIUS
                local z = math.sin(angle) * ROTATE_RADIUS
                p.CFrame = hrp.CFrame * CFrame.new(x, -1.3, z)
            end
        else
            for _,p in ipairs(rotateParts) do
                p.CFrame = CFrame.new(1e9,1e9,1e9)
            end
        end

        -- attack
        if attackEnabled then
            attackTick(dt)
        end
    end)

    -- input toggles
    UserInputService.InputBegan:Connect(function(input, processed)
        if processed then return end
        if input.KeyCode == Enum.KeyCode.RightControl then
            rotateEnabled = not rotateEnabled
            shortMsg(label, "Rotate: " .. (rotateEnabled and "ON" or "OFF"))
        elseif input.KeyCode == Enum.KeyCode.RightAlt then
            attackEnabled = not attackEnabled
            shortMsg(label, "AutoAttack: " .. (attackEnabled and "ON" or "OFF"))
        end
    end)

    -- cleanup on player leaving
    localPlayer.AncestryChanged:Connect(function()
        if not localPlayer:IsDescendantOf(game) then
            hubRunning = false
            safeDestroy(rotateModel)
            safeDestroy(notifyGui)
        end
    end)
end

-- ===== start =====
-- Build and display key GUI
local keyGui = buildKeyGui()

-- small helper: if you want to auto-open hub when running locally and you know the key, uncomment:
-- if isKeyValid("sigmaboss") then startHub() end

print("[Sigma Hub] Loaded. Check Key GUI shown.")
