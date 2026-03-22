if getgenv().AkatsukiFarmLoaded then
    getgenv().AkatsukiFarmLoaded()
end

local lastExec = getgenv().AkatsukiLastExec or 0
if tick() - lastExec < 3 then return end
getgenv().AkatsukiLastExec = tick()

local farmEnabled = false
local abilityThread = nil
local toolThread = nil
local activeSkills = {}

local RS = game:GetService("ReplicatedStorage")
local player = game:GetService("Players").LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local humanoid = char:WaitForChild("Humanoid")

local abilityRemote = RS:WaitForChild("AbilitySystem"):WaitForChild("Remotes"):WaitForChild("RequestAbility")
local portalRemote = RS:WaitForChild("Remotes"):WaitForChild("TeleportToPortal")

local SKILLS = { Z = 1, X = 2, C = 3, V = 4 }
local SKILL_KEYS = { "Z", "X", "C", "V" }

local function getTools()
    local tools = {}
    local holding = char:FindFirstChildWhichIsA("Tool")
    if holding then table.insert(tools, holding.Name) end
    for _, v in ipairs(player.Backpack:GetChildren()) do
        if v:IsA("Tool") and not table.find(tools, v.Name) then
            table.insert(tools, v.Name)
            if #tools >= 2 then break end
        end
    end
    return tools
end

local TOOLS = getTools()

local RED = Color3.fromRGB(200, 0, 0)
local DARK_RED = Color3.fromRGB(110, 0, 0)
local BLACK = Color3.fromRGB(10, 10, 10)
local GREEN = Color3.fromRGB(0, 200, 80)
local GRAY = Color3.fromRGB(160, 160, 160)

local gui = Instance.new("ScreenGui")
gui.Name = "AkatsukiEGF"
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.Parent = player:WaitForChild("PlayerGui")

local frameHeight = #TOOLS > 0 and 195 or 165

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 290, 0, frameHeight)
frame.Position = UDim2.new(0.5, -145, 0.05, 0)
frame.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
frame.BorderSizePixel = 0
frame.Active = true
frame.Draggable = true
frame.Parent = gui

Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 14)
local frameBorder = Instance.new("UIStroke", frame)
frameBorder.Color = RED
frameBorder.Thickness = 2.5

local function makeLabel(parent, text, size, pos, sz, xAlign)
    local l = Instance.new("TextLabel")
    l.BackgroundTransparency = 1
    l.Text = text
    l.TextColor3 = RED
    l.TextSize = size
    l.Font = Enum.Font.Gotham
    l.Position = pos
    l.Size = sz
    l.TextXAlignment = xAlign or Enum.TextXAlignment.Left
    l.Parent = parent
    return l
end

local function makeDivider(y)
    local d = Instance.new("Frame")
    d.Size = UDim2.new(0.88, 0, 0, 1)
    d.Position = UDim2.new(0.06, 0, 0, y)
    d.BackgroundColor3 = RED
    d.BorderSizePixel = 0
    d.Parent = frame
end

local titleLabel = makeLabel(frame, "Akatsuki Hub - End Game Farm", 14,
    UDim2.new(0, 10, 0, 0), UDim2.new(1, -40, 0, 38), Enum.TextXAlignment.Left)
titleLabel.Font = Enum.Font.GothamBold
titleLabel.TextWrapped = true

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 22, 0, 22)
closeBtn.Position = UDim2.new(1, -28, 0, 8)
closeBtn.BackgroundTransparency = 1
closeBtn.Text = "X"
closeBtn.TextColor3 = RED
closeBtn.TextSize = 15
closeBtn.Font = Enum.Font.GothamBold
closeBtn.BorderSizePixel = 0
closeBtn.Parent = frame

makeDivider(40)

local yOffset = 46
local selectedTool = 1
local toolBtns = {}

if #TOOLS > 0 then
    local toolRow = Instance.new("Frame")
    toolRow.Size = UDim2.new(1, -20, 0, 26)
    toolRow.Position = UDim2.new(0, 10, 0, yOffset)
    toolRow.BackgroundTransparency = 1
    toolRow.Parent = frame

    local function selectTool(idx)
        selectedTool = idx
        for i, t in ipairs(toolBtns) do
            t.TextColor3 = i == idx and RED or GRAY
        end
    end

    for i, name in ipairs(TOOLS) do
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0.48, 0, 1, 0)
        btn.Position = UDim2.new((i - 1) * 0.51, 0, 0, 0)
        btn.BackgroundTransparency = 1
        btn.Text = name
        btn.TextColor3 = i == 1 and RED or GRAY
        btn.TextSize = 12
        btn.Font = Enum.Font.Gotham
        btn.BorderSizePixel = 0
        btn.Parent = toolRow

        toolBtns[i] = btn

        btn.MouseButton1Click:Connect(function()
            selectTool(i)
        end)
    end

    yOffset = yOffset + 32
    makeDivider(yOffset - 2)
end

local statusLabel = makeLabel(frame, "Status: OFF", 12,
    UDim2.new(0, 10, 0, yOffset + 2), UDim2.new(1, -20, 0, 20))

local skillRow = Instance.new("Frame")
skillRow.Size = UDim2.new(1, -20, 0, 22)
skillRow.Position = UDim2.new(0, 10, 0, yOffset + 26)
skillRow.BackgroundTransparency = 1
skillRow.Parent = frame

makeLabel(skillRow, "Skills:", 12, UDim2.new(0, 0, 0, 0), UDim2.new(0, 38, 1, 0))

for i, key in ipairs(SKILL_KEYS) do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 28, 1, 0)
    btn.Position = UDim2.new(0, 36 + (i - 1) * 34, 0, 0)
    btn.BackgroundTransparency = 1
    btn.Text = key
    btn.TextColor3 = RED
    btn.TextSize = 12
    btn.Font = Enum.Font.Gotham
    btn.BorderSizePixel = 0
    btn.Parent = skillRow

    btn.MouseButton1Click:Connect(function()
        activeSkills[key] = not activeSkills[key]
        btn.TextColor3 = activeSkills[key] and GREEN or RED
    end)
end

local divY = yOffset + 52
makeDivider(divY)

local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(0, 85, 0, 28)
toggleBtn.Position = UDim2.new(0.5, -42, 0, divY + 16)
toggleBtn.BackgroundColor3 = RED
toggleBtn.Text = "START"
toggleBtn.TextColor3 = BLACK
toggleBtn.TextSize = 13
toggleBtn.Font = Enum.Font.Gotham
toggleBtn.BorderSizePixel = 0
toggleBtn.Parent = frame
Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(0, 6)

local function equipSelectedTool()
    pcall(function()
        local name = TOOLS[selectedTool]
        if not name then return end
        local equipped = char:FindFirstChildWhichIsA("Tool")
        if equipped and equipped.Name == name then return end
        local item = player.Backpack:FindFirstChild(name)
        if item then
            humanoid:EquipTool(item)
        end
    end)
end

local function fireAbilities()
    while farmEnabled do
        pcall(function()
            for _, key in ipairs(SKILL_KEYS) do
                if activeSkills[key] then
                    abilityRemote:FireServer(SKILLS[key])
                end
            end
        end)
        task.wait(0.1)
    end
end

local function watchTool()
    while farmEnabled do
        equipSelectedTool()
        task.wait(1)
    end
end

local function runFarm()
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    while farmEnabled do
        statusLabel.Text = "Status: Changing Island"
        pcall(function() portalRemote:FireServer("Judgement") end)
        task.wait(0.5)
        if not farmEnabled then break end

        statusLabel.Text = "Status: Teleporting"
        pcall(function() hrp.CFrame = CFrame.new(1071, 2, 1273) end)
        task.wait(2)
        if not farmEnabled then break end

        statusLabel.Text = "Status: Changing Island"
        pcall(function() portalRemote:FireServer("Judgement") end)
        task.wait(0.5)
        if not farmEnabled then break end

        statusLabel.Text = "Status: Teleporting"
        pcall(function() hrp.CFrame = CFrame.new(-1272, 1, -1192) end)
        task.wait(2)
    end
end

local function stopFarm()
    farmEnabled = false
    if abilityThread then task.cancel(abilityThread) abilityThread = nil end
    if toolThread then task.cancel(toolThread) toolThread = nil end
    statusLabel.Text = "Status: OFF"
    toggleBtn.Text = "START"
    toggleBtn.BackgroundColor3 = RED
end

getgenv().AkatsukiFarmLoaded = function()
    stopFarm()
    pcall(function() gui:Destroy() end)
end

closeBtn.MouseButton1Click:Connect(function()
    stopFarm()
    gui:Destroy()
    getgenv().AkatsukiFarmLoaded = nil
end)

toggleBtn.MouseButton1Click:Connect(function()
    farmEnabled = not farmEnabled
    if farmEnabled then
        toggleBtn.Text = "STOP"
        toggleBtn.BackgroundColor3 = DARK_RED
        task.spawn(runFarm)
        abilityThread = task.spawn(fireAbilities)
        toolThread = task.spawn(watchTool)
    else
        stopFarm()
    end
end)
