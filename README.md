task.spawn(function() pcall(function()
repeat task.wait() until game:IsLoaded()
local TweenService      = game:GetService("TweenService")
local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local UserInputService  = game:GetService("UserInputService")
local CoreGui           = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LP = Players.LocalPlayer
if _G.AstaStealConn then
    pcall(function() _G.AstaStealConn:Disconnect() end)
    _G.AstaStealConn = nil
end
if _G.AstaNormalSteal and _G.AstaNormalSteal.stealConn then
    pcall(function() _G.AstaNormalSteal.stealConn:Disconnect() end)
end
if _G.AstaSemiSteal and _G.AstaSemiSteal.conn then
    pcall(function() _G.AstaSemiSteal.conn:Disconnect() end)
end
pcall(function()
    local old = CoreGui:FindFirstChild("AstaStealUI")
    if old then old:Destroy() end
end)
local CONFIG_PATH = "Asta/asta_steal_config.json"
local Radii = {Semi = 9}
local selectedMode   = "Semi"
local isEnabled      = true
local uiScaleValue   = 100

local mainUIScale    = nil
local function saveConfig()
    pcall(function()
        local data = {
            semiRadius   = Radii.Semi,
            mode         = "Semi",
            enabled      = isEnabled,
            uiScale      = uiScaleValue,
            
        }
        writefile(CONFIG_PATH, game:GetService("HttpService"):JSONEncode(data))
    end)
end

local function loadConfig()
    pcall(function()
        if not isfile(CONFIG_PATH) then return end
        local raw = readfile(CONFIG_PATH)
        local ok, data = pcall(function() return game:GetService("HttpService"):JSONDecode(raw) end)
        if not ok or type(data) ~= "table" then return end
        if type(data.semiRadius)   == "number" then Radii.Semi   = data.semiRadius   end
        selectedMode = "Semi"
        if type(data.enabled) == "boolean" then isEnabled = data.enabled end
        if type(data.uiScale) == "number"  then uiScaleValue = math.clamp(math.floor(data.uiScale + 0.5), 50, 200) end
        
    end)
end
loadConfig()
local ProgressBarFill = nil
local ProgressPercentLabel = nil
local function setProgress(val)
    local progress = math.clamp(val, 0, 1)
    if ProgressBarFill then
        TweenService:Create(ProgressBarFill, TweenInfo.new(0.05), {
            Size = UDim2.new(progress, 0, 1, 0)
        }):Play()
    end
    if ProgressPercentLabel then
        ProgressPercentLabel.Text = tostring(math.floor(progress * 100 + 0.5)) .. "%"
    end
end
local function getHRP()
    local char = LP.Character
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart")
        or char:FindFirstChild("Torso")
        or char:FindFirstChild("UpperTorso")
end
local function isMyBase(plotName)
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return false end
    local plot = plots:FindFirstChild(plotName)
    if not plot then return false end
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") then return yb.Enabled == true end
    end
    return false
end
local function startNormalMode() end
local function stopNormalMode() end
_G.AstaSemiSteal = _G.AstaSemiSteal or {}
local A = _G.AstaSemiSteal
local function semiSetup()
    if A.conn then pcall(function() A.conn:Disconnect() end); A.conn = nil end
    A.enabled       = false
    A.holdMin       = 1.3
    A.holdMax       = 2.6
    A.entryDelay    = 0.3
    A.cooldown      = 0.05
    A.primeRange    = 80
    A.radius        = Radii.Semi
    A.plotSync      = A.plotSync or {caches = {}, connections = {}}
    A.animals       = A.animals or {}
    A.promptCache   = A.promptCache or {}
    A.internalCache = A.internalCache or {}
    A.state         = A.state or {active = false, startTime = 0, phase = "idle", label = ""}
    local function splitPath(path)
        if typeof(path) == "table" then return path end
        local out = {}
        for part in string.gmatch(tostring(path), "[^%.]+") do table.insert(out, tonumber(part) or part) end
        return out
    end
    local function resolvePath(path, root)
        local current, parent, key = root, nil, nil
        for _, part in ipairs(splitPath(path)) do parent = current; key = part; current = current and current[part] or nil end
        return current, parent, key
    end
    local function applySyncDiff(channelName, packet)
        local cache = A.plotSync.caches[channelName]
        if typeof(cache) ~= "table" then return end
        local path, action, a, b = packet[1], packet[2], packet[3], packet[4]
        local current, parent, key = resolvePath(path, cache)
        if action == "Changed" then if parent ~= nil then parent[key] = a end
        elseif action == "ArrayInsert" then if current ~= nil then table.insert(current, b, a) end
        elseif action == "ArrayRemoved" then if current ~= nil then table.remove(current, b) end
        elseif action == "DictionaryInsert" then if current ~= nil then current[b] = a end
        elseif action == "DictionaryRemoved" then if current ~= nil then current[b] = nil end end
    end
    local function attachPlotChannel(remote, plots, requestData)
        if A.plotSync.connections[remote] then return end
        local channelName = tostring(remote.Name)
        if not plots:FindFirstChild(channelName) then return end
        if requestData and A.plotSync.caches[channelName] == nil then
            local ok, data = pcall(function() return requestData:InvokeServer(channelName) end)
            A.plotSync.caches[channelName] = (ok and typeof(data) == "table") and data or {}
        elseif A.plotSync.caches[channelName] == nil then A.plotSync.caches[channelName] = {} end
        A.plotSync.connections[remote] = remote.OnClientEvent:Connect(function(queue)
            for _, packet in ipairs(queue) do applySyncDiff(channelName, packet) end
        end)
    end
    local function ensureSync()
        if A.syncReady then return true end
        local ok = pcall(function()
            local rs = ReplicatedStorage
            A.packages = rs:WaitForChild("Packages", 10); A.datas = rs:WaitForChild("Datas", 10); A.plots = workspace:WaitForChild("Plots", 10)
            if not (A.packages and A.datas and A.plots) then return end
            A.animalsData = require(A.datas:WaitForChild("Animals", 10))
            local sync = A.packages:WaitForChild("Synchronizer", 10)
            A.channelFolder = sync:WaitForChild("Channel", 10); A.routeRemote = sync:WaitForChild("CommunicationRoute", 10); A.requestData = sync:FindFirstChild("RequestData")
            for _, child in ipairs(A.channelFolder:GetChildren()) do if child:IsA("RemoteEvent") then attachPlotChannel(child, A.plots, A.requestData) end end
            A.channelFolder.ChildAdded:Connect(function(child) if child:IsA("RemoteEvent") then attachPlotChannel(child, A.plots, A.requestData) end end)
            A.routeRemote.OnClientEvent:Connect(function(actions)
                for _, action in ipairs(actions) do
                    local kind, channelName = action[1], tostring(action[2])
                    if A.plots and A.plots:FindFirstChild(channelName) then
                        if kind == "ListenerAdded" then local remote = A.channelFolder and A.channelFolder:FindFirstChild(channelName); if remote and remote:IsA("RemoteEvent") then attachPlotChannel(remote, A.plots, A.requestData) end
                        elseif kind == "ListenerRemoved" then for remote, conn in pairs(A.plotSync.connections) do if tostring(remote.Name) == channelName then pcall(function() conn:Disconnect() end); A.plotSync.connections[remote] = nil; A.plotSync.caches[channelName] = nil; break end end end
                    end
                end
            end)
            A.syncReady = true
        end)
        return ok and A.syncReady == true
    end
    local function getPlotOwner(plot)
        local sign = plot and plot:FindFirstChild("PlotSign"); local frame = sign and sign:FindFirstChild("SurfaceGui") and sign.SurfaceGui:FindFirstChild("Frame"); local label = frame and frame:FindFirstChild("TextLabel")
        if not label or label.Text == "Empty Base" then return nil end
        return label.Text:gsub("'s [Bb]ase$", ""):gsub("%s+$", "")
    end
    local function isMyBaseAnimal(animalData)
        if not animalData or not animalData.plot or not A.plots then return false end
        local plot = A.plots:FindFirstChild(animalData.plot); if not plot then return false end
        local owner = getPlotOwner(plot); return owner == LP.DisplayName or owner == LP.Name
    end
    local function podiumFor(animalData)
        local plot = A.plots and A.plots:FindFirstChild(animalData.plot); local podiums = plot and plot:FindFirstChild("AnimalPodiums"); return podiums and podiums:FindFirstChild(animalData.slot) or nil
    end
    local function animalPos(animalData) local podium = podiumFor(animalData); return podium and podium:GetPivot().Position or nil end
    local function distToAnimal(animalData) local root = LP.Character and (LP.Character:FindFirstChild("HumanoidRootPart") or LP.Character:FindFirstChild("UpperTorso")); local pos = animalPos(animalData); return root and pos and (root.Position - pos).Magnitude or math.huge end
    local function findPromptForAnimal(animalData)
        if not animalData then return nil end; local cached = A.promptCache[animalData.uid]; if cached and cached.Parent then return cached end
        local podium = podiumFor(animalData); local base = podium and podium:FindFirstChild("Base"); local spawn = base and base:FindFirstChild("Spawn"); local attach = spawn and spawn:FindFirstChild("PromptAttachment"); if not attach then return nil end
        for _, prompt in ipairs(attach:GetChildren()) do if prompt:IsA("ProximityPrompt") then A.promptCache[animalData.uid] = prompt; return prompt end end; return nil
    end
    local function scanAllPlots()
        if not ensureSync() then return end; local newCache = {}
        for _, plot in ipairs(A.plots:GetChildren()) do
            local cache = A.plotSync.caches[plot.Name]; local animalList = cache and cache.AnimalList
            if typeof(animalList) == "table" then
                for slot, animalData in pairs(animalList) do
                    if type(animalData) == "table" then local animalName = animalData.Index; local info = A.animalsData and A.animalsData[animalName]; if info then table.insert(newCache, {name = info.DisplayName or animalName, plot = plot.Name, slot = tostring(slot), uid = plot.Name .. "_" .. tostring(slot)}) end end
                end
            end
        end
        A.animals = newCache
    end
    local function pickClosest()
        local root = LP.Character and (LP.Character:FindFirstChild("HumanoidRootPart") or LP.Character:FindFirstChild("UpperTorso")); if not root then return nil end
        local best, bestDist = nil, math.huge
        for _, animalData in ipairs(A.animals) do
            if not isMyBaseAnimal(animalData) then local pos = animalPos(animalData); local dist = pos and (root.Position - pos).Magnitude or math.huge; if dist <= (A.primeRange or 80) and dist < bestDist then best, bestDist = animalData, dist end end
        end
        return best
    end
    local function buildCallbacks(prompt)
        if A.internalCache[prompt] then return end; local data = {holdCallbacks = {}, triggerCallbacks = {}, ready = true}
        local okHold, holds = pcall(getconnections, prompt.PromptButtonHoldBegan); if okHold and type(holds) == "table" then for _, conn in ipairs(holds) do if type(conn.Function) == "function" then table.insert(data.holdCallbacks, conn.Function) end end end
        local okTrig, trigs = pcall(getconnections, prompt.Triggered); if okTrig and type(trigs) == "table" then for _, conn in ipairs(trigs) do if type(conn.Function) == "function" then table.insert(data.triggerCallbacks, conn.Function) end end end
        if #data.holdCallbacks > 0 or #data.triggerCallbacks > 0 then A.internalCache[prompt] = data end
    end
    local function executeSemi(prompt, animalData)
        if not prompt or not prompt.Parent or not animalData then return false end
        buildCallbacks(prompt); local data = A.internalCache[prompt]; if not data or not data.ready then return false end
        data.ready = false; A.state.active = true; A.state.startTime = tick(); A.state.phase = "holding"; A.state.label = animalData.name or "Animal"
        task.spawn(function()
            local startTime = A.state.startTime
            for _, fn in ipairs(data.holdCallbacks) do task.spawn(function() pcall(fn) end) end
            local abortedHold = false
            while A.enabled and selectedMode == "Semi" and tick() - startTime < (A.holdMin or 1.3) do
                if distToAnimal(animalData) > (A.primeRange or 80) then
                    abortedHold = true; break
                end
                setProgress((tick() - startTime) / (A.holdMax or 2.6)); task.wait()
            end
            if abortedHold then
                A.state.active = false; A.state.phase = "idle"; data.ready = true; setProgress(0); return
            end
            A.state.phase = "waitingRange"; local alreadyInRange = distToAnimal(animalData) <= (tonumber(A.radius) or 10); local fired = false
            while A.enabled and selectedMode == "Semi" and prompt.Parent do
                local elapsed = tick() - startTime; if elapsed > (A.holdMax or 2.6) then break end
                if distToAnimal(animalData) > (A.primeRange or 80) then break end
                setProgress(elapsed / (A.holdMax or 2.6))
                if distToAnimal(animalData) <= (tonumber(A.radius) or 10) then
                    if not alreadyInRange then task.wait(A.entryDelay or 0.3) end
                    if A.enabled and selectedMode == "Semi" then for _, fn in ipairs(data.triggerCallbacks) do task.spawn(function() pcall(fn) end) end; fired = true end; break
                end; task.wait()
            end
            A.state.active = false; A.state.phase = "idle"; if fired then setProgress(1) end; task.wait(A.cooldown or 0.05); data.ready = true; setProgress(0)
        end)
        return true
    end
    local function ensureScanThread()
        if A.scanThread then return end
        A.scanThread = task.spawn(function() while _G.AstaSemiSteal do if A.enabled or selectedMode == "Semi" then pcall(scanAllPlots) end; task.wait(5) end end)
    end
    A.start = function()
        A.radius = Radii.Semi; A.enabled = true; ensureSync(); ensureScanThread(); pcall(scanAllPlots)
        if A.conn then A.conn:Disconnect(); A.conn = nil end
        A.conn = RunService.Heartbeat:Connect(function()
            if not isEnabled or not A.enabled then return end; if selectedMode ~= "Semi" then A.stop(); return end; if A.state.active then return end
            local target = pickClosest(); if not target then return end; local prompt = findPromptForAnimal(target); if prompt then executeSemi(prompt, target) end
        end)
    end
    A.stop = function()
        A.enabled = false; if A.conn then A.conn:Disconnect(); A.conn = nil end; A.state.active = false; A.state.phase = "idle"; setProgress(0)
    end
end
semiSetup()
local RadiusValueLabel = nil
local function currentRadius()
    return Radii.Semi
end
local function setRadius(v)
    v = math.clamp(v, 1, 1000)
    Radii.Semi = v
    if A then A.radius = v end
    if RadiusValueLabel then RadiusValueLabel.Text = tostring(v) end
    saveConfig()
end
local NormalBtn, SemiBtn
local ALL_MODE_BTNS = {}
local function refreshModeButtons()
    for mode, btn in pairs(ALL_MODE_BTNS) do
        local active = (mode == selectedMode)
        btn.BackgroundColor3 = active and Color3.fromRGB(0, 0, 0) or Color3.fromRGB(8, 20, 60)
        btn.TextColor3       = active and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(120, 120, 120)
    end
end
local function stopAll()
    if A and A.stop then A.stop() end
    setProgress(0)
end
local function startAll()
    if not isEnabled then return end
    if A and A.start then A.start() end
end
local function applyMode(mode)
    stopAll()
    selectedMode = mode
    if RadiusValueLabel then RadiusValueLabel.Text = tostring(currentRadius()) end
    refreshModeButtons()
    startAll()
    saveConfig()
end
local sg = Instance.new("ScreenGui")
sg.Name           = "AstaStealUI"
sg.ResetOnSpawn   = false
sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
sg.Parent         = CoreGui
local W, H = 320, 163
local main = Instance.new("Frame", sg)
main.Name                   = "Main"
main.Size                   = UDim2.new(0, W, 0, H)
main.Position               = UDim2.new(0.5, -W/2, 0, 20)
main.BackgroundColor3       = Color3.fromRGB(5, 15, 45)
main.BackgroundTransparency = 0
main.BorderSizePixel        = 0
main.Active                 = true
Instance.new("UICorner", main).CornerRadius = UDim.new(0, 10)
mainUIScale = Instance.new("UIScale", main)
mainUIScale.Scale = uiScaleValue / 100
local mainStroke = Instance.new("UIStroke", main)
mainStroke.Thickness       = 2
mainStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
local shimGrad = Instance.new("UIGradient", mainStroke)
shimGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0,    Color3.fromRGB(255, 255, 255)),
    ColorSequenceKeypoint.new(0.20, Color3.fromRGB(80, 80, 80)),
    ColorSequenceKeypoint.new(0.45, Color3.fromRGB(255, 255, 255)),
    ColorSequenceKeypoint.new(0.55, Color3.fromRGB(255, 255, 255)),
    ColorSequenceKeypoint.new(0.80, Color3.fromRGB(80, 80, 80)),
    ColorSequenceKeypoint.new(1,    Color3.fromRGB(255, 255, 255)),
})
shimGrad.Transparency = NumberSequence.new({
    NumberSequenceKeypoint.new(0,    0.80),
    NumberSequenceKeypoint.new(0.20, 0.40),
    NumberSequenceKeypoint.new(0.45, 0.85),
    NumberSequenceKeypoint.new(0.55, 0.05),
    NumberSequenceKeypoint.new(0.80, 0.40),
    NumberSequenceKeypoint.new(1,    0.80),
})
shimGrad.Rotation = 0
task.spawn(function()
    while main.Parent do shimGrad.Rotation = (shimGrad.Rotation + 1.2) % 360; task.wait(0.033) end
end)
local headerH = 30
local header  = Instance.new("Frame", main)
header.Size                   = UDim2.new(1, 0, 0, headerH)
header.BackgroundColor3       = Color3.fromRGB(8, 22, 55)
header.BackgroundTransparency = 0
header.BorderSizePixel        = 0
Instance.new("UICorner", header).CornerRadius = UDim.new(0, 10)
local hFix = Instance.new("Frame", header)
hFix.Size             = UDim2.new(1, 0, 0, 10)
hFix.Position         = UDim2.new(0, 0, 1, -10)
hFix.BackgroundColor3 = Color3.fromRGB(8, 8, 8)
hFix.BorderSizePixel  = 0
local titleLbl = Instance.new("TextLabel", header)
titleLbl.Size                   = UDim2.new(1, -150, 1, 0)
titleLbl.Position               = UDim2.new(0, 34, 0, 0)
titleLbl.BackgroundTransparency = 1
titleLbl.Text                   = "Asta"
titleLbl.TextColor3             = Color3.fromRGB(255, 255, 255)
titleLbl.Font                   = Enum.Font.GothamBlack
titleLbl.TextSize               = 13
titleLbl.TextXAlignment         = Enum.TextXAlignment.Left
titleLbl.ZIndex                 = 3
local titleGrad = Instance.new("UIGradient", titleLbl)
titleGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0,   Color3.fromRGB(150, 150, 150)),
    ColorSequenceKeypoint.new(0.4, Color3.fromRGB(255, 255, 255)),
    ColorSequenceKeypoint.new(0.6, Color3.fromRGB(255, 255, 255)),
    ColorSequenceKeypoint.new(1,   Color3.fromRGB(150, 150, 150)),
})
titleGrad.Transparency = NumberSequence.new({
    NumberSequenceKeypoint.new(0,   0.15),
    NumberSequenceKeypoint.new(0.4, 0.0),
    NumberSequenceKeypoint.new(0.6, 0.0),
    NumberSequenceKeypoint.new(1,   0.15),
})
titleGrad.Rotation = 90
task.spawn(function()
    while titleLbl.Parent do titleGrad.Rotation = (titleGrad.Rotation + 0.8) % 360; task.wait(0.016) end
end)
local statusDot = Instance.new("Frame", header)
statusDot.Size             = UDim2.new(0, 9, 0, 9)
statusDot.Position         = UDim2.new(1, -38, 0.5, -4)
statusDot.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
statusDot.BorderSizePixel  = 0
Instance.new("UICorner", statusDot).CornerRadius = UDim.new(1, 0)
task.spawn(function()
    while statusDot.Parent do
        if isEnabled then
            local p = (math.sin(tick() * 3) + 1) / 2
            statusDot.BackgroundColor3 = Color3.fromRGB(math.floor(180 + p * 40), math.floor(180 + p * 40), math.floor(180 + p * 40))
        else
            statusDot.BackgroundColor3 = Color3.fromRGB(120, 40, 40)
        end
        task.wait(0.033)
    end
end)
local onBtn = Instance.new("TextButton", header)
onBtn.Size                   = UDim2.new(0, 50, 1, 0)
onBtn.Position               = UDim2.new(1, -108, 0, 0)
onBtn.BackgroundTransparency = 1
onBtn.Text                   = isEnabled and "ON" or "OFF"
onBtn.TextColor3             = isEnabled and Color3.fromRGB(200, 200, 200) or Color3.fromRGB(255, 80, 80)
onBtn.Font                   = Enum.Font.GothamBold
onBtn.TextSize               = 11
onBtn.TextXAlignment         = Enum.TextXAlignment.Right
onBtn.ZIndex                 = 5
onBtn.MouseButton1Click:Connect(function()
    isEnabled = not isEnabled
    onBtn.Text       = isEnabled and "ON"  or "OFF"
    onBtn.TextColor3 = isEnabled and Color3.fromRGB(200, 200, 200) or Color3.fromRGB(255, 80, 80)
    if isEnabled then startAll() else stopAll() end
    saveConfig()
end)

local content = Instance.new("Frame", main)
content.Size                   = UDim2.new(1, 0, 1, -headerH)
content.Position               = UDim2.new(0, 0, 0, headerH)
content.BackgroundTransparency = 1
content.BorderSizePixel        = 0

local RadiusLabel = Instance.new("TextLabel", content)
RadiusLabel.Size                   = UDim2.new(0, 65, 0, 18)
RadiusLabel.Position               = UDim2.new(0, 10, 0, 4)
RadiusLabel.BackgroundTransparency = 1
RadiusLabel.Text                   = "Range:"
RadiusLabel.TextColor3             = Color3.fromRGB(150, 150, 150)
RadiusLabel.Font                   = Enum.Font.GothamBold
RadiusLabel.TextSize               = 11
RadiusLabel.TextXAlignment         = Enum.TextXAlignment.Left
local MinusArea = Instance.new("TextButton", content)
MinusArea.Size             = UDim2.new(0, 26, 0, 20)
MinusArea.Position         = UDim2.new(1, -80, 0, 4)
MinusArea.BackgroundColor3 = Color3.fromRGB(10, 30, 90)
MinusArea.BorderSizePixel  = 0
MinusArea.Text             = "-"
MinusArea.TextColor3       = Color3.fromRGB(80, 80, 80)
MinusArea.Font             = Enum.Font.GothamBold
MinusArea.TextSize         = 14
Instance.new("UICorner", MinusArea).CornerRadius = UDim.new(0, 5)
local mStroke = Instance.new("UIStroke", MinusArea)
mStroke.Color = Color3.fromRGB(60, 60, 60); mStroke.Thickness = 1
RadiusValueLabel = Instance.new("TextLabel", content)
RadiusValueLabel.Size                   = UDim2.new(0, 22, 0, 20)
RadiusValueLabel.Position               = UDim2.new(1, -52, 0, 4)
RadiusValueLabel.BackgroundTransparency = 1
RadiusValueLabel.Text                   = tostring(currentRadius())
RadiusValueLabel.TextColor3             = Color3.fromRGB(255, 255, 255)
RadiusValueLabel.Font                   = Enum.Font.GothamBold
RadiusValueLabel.TextSize               = 13
RadiusValueLabel.TextXAlignment         = Enum.TextXAlignment.Center
local PlusArea = Instance.new("TextButton", content)
PlusArea.Size             = UDim2.new(0, 26, 0, 20)
PlusArea.Position         = UDim2.new(1, -28, 0, 4)
PlusArea.BackgroundColor3 = Color3.fromRGB(10, 30, 90)
PlusArea.BorderSizePixel  = 0
PlusArea.Text             = "+"
PlusArea.TextColor3       = Color3.fromRGB(80, 80, 80)
PlusArea.Font             = Enum.Font.GothamBold
PlusArea.TextSize         = 14
Instance.new("UICorner", PlusArea).CornerRadius = UDim.new(0, 5)
local pStroke = Instance.new("UIStroke", PlusArea)
pStroke.Color = Color3.fromRGB(60, 60, 60); pStroke.Thickness = 1
MinusArea.MouseButton1Click:Connect(function() setRadius(currentRadius() - 1) end)
PlusArea.MouseButton1Click:Connect(function() setRadius(currentRadius() + 1) end)
local modeSep = Instance.new("Frame", content)
modeSep.Size             = UDim2.new(1, -20, 0, 1)
modeSep.Position         = UDim2.new(0, 10, 0, 28)
modeSep.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
modeSep.BorderSizePixel  = 0
local function makeBtn(labelTxt, xOffset, yOffset, modeName)
    local btn = Instance.new("TextButton", content)
    btn.Size             = UDim2.new(0.5, -8, 0, 22)
    btn.Position         = UDim2.new(0, xOffset, 0, yOffset)
    btn.BackgroundColor3 = Color3.fromRGB(8, 20, 60)
    btn.BorderSizePixel  = 0
    btn.Text             = labelTxt
    btn.TextColor3       = Color3.fromRGB(120, 120, 120)
    btn.Font             = Enum.Font.GothamBold
    btn.TextSize         = 10
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    local s = Instance.new("UIStroke", btn)
    s.Color = Color3.fromRGB(50, 50, 50); s.Thickness = 1
    ALL_MODE_BTNS[modeName] = btn
    btn.MouseButton1Click:Connect(function() applyMode(modeName) end)
    return btn
end
SemiBtn   = makeBtn("Semi",    10,   34, "Semi")
SemiBtn.Size = UDim2.new(1, -20, 0, 22)
local modeSep2 = Instance.new("Frame", content)
modeSep2.Size             = UDim2.new(1, -20, 0, 1)
modeSep2.Position         = UDim2.new(0, 10, 0, 60)
modeSep2.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
modeSep2.BorderSizePixel  = 0
local uiScaleLbl = Instance.new("TextLabel", content)
uiScaleLbl.Size                   = UDim2.new(0, 70, 0, 20)
uiScaleLbl.Position               = UDim2.new(0, 10, 0, 64)
uiScaleLbl.BackgroundTransparency = 1
uiScaleLbl.Text                   = "UI Scale:"
uiScaleLbl.TextColor3             = Color3.fromRGB(150, 150, 150)
uiScaleLbl.Font                   = Enum.Font.GothamBold
uiScaleLbl.TextSize               = 11
uiScaleLbl.TextXAlignment         = Enum.TextXAlignment.Left
local uiScaleBox = Instance.new("TextBox", content)
uiScaleBox.Size                   = UDim2.new(0, 48, 0, 20)
uiScaleBox.Position               = UDim2.new(1, -62, 0, 64)
uiScaleBox.BackgroundColor3       = Color3.fromRGB(8, 20, 60)
uiScaleBox.BorderSizePixel        = 0
uiScaleBox.Text                   = tostring(uiScaleValue)
uiScaleBox.TextColor3             = Color3.fromRGB(255, 255, 255)
uiScaleBox.Font                   = Enum.Font.GothamBold
uiScaleBox.TextSize               = 12
uiScaleBox.TextXAlignment         = Enum.TextXAlignment.Center
uiScaleBox.ClearTextOnFocus       = false
Instance.new("UICorner", uiScaleBox).CornerRadius = UDim.new(0, 5)
local uiScaleStroke = Instance.new("UIStroke", uiScaleBox)
uiScaleStroke.Color = Color3.fromRGB(60, 60, 60); uiScaleStroke.Thickness = 1
local uiScalePct = Instance.new("TextLabel", content)
uiScalePct.Size                   = UDim2.new(0, 12, 0, 20)
uiScalePct.Position               = UDim2.new(1, -13, 0, 64)
uiScalePct.BackgroundTransparency = 1
uiScalePct.Text                   = "%"
uiScalePct.TextColor3             = Color3.fromRGB(120, 120, 120)
uiScalePct.Font                   = Enum.Font.GothamBold
uiScalePct.TextSize               = 11
uiScaleBox.FocusLost:Connect(function()
    local n = tonumber(uiScaleBox.Text)
    if n then
        n = math.clamp(math.floor(n + 0.5), 50, 200)
        uiScaleValue = n
        uiScaleBox.Text = tostring(n)
        if mainUIScale then mainUIScale.Scale = n / 100 end
        saveConfig()
    else
        uiScaleBox.Text = tostring(uiScaleValue)
    end
end)
local modeSep3 = Instance.new("Frame", content)
modeSep3.Size             = UDim2.new(1, -20, 0, 1)
modeSep3.Position         = UDim2.new(0, 10, 0, 88)
modeSep3.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
modeSep3.BorderSizePixel  = 0
local grabPercentageLabel = Instance.new("TextLabel", content)
grabPercentageLabel.Size                   = UDim2.new(1, -20, 0, 12)
grabPercentageLabel.Position               = UDim2.new(0, 10, 0, 91)
grabPercentageLabel.BackgroundTransparency = 1
grabPercentageLabel.Text                   = "GRAB PERCENTAGE"
grabPercentageLabel.TextColor3             = Color3.fromRGB(150, 150, 150)
grabPercentageLabel.Font                   = Enum.Font.GothamBold
grabPercentageLabel.TextSize               = 10
grabPercentageLabel.TextXAlignment         = Enum.TextXAlignment.Center
grabPercentageLabel.TextYAlignment         = Enum.TextYAlignment.Center
local ProgressBarBg = Instance.new("Frame", content)
ProgressBarBg.Size             = UDim2.new(1, -20, 0, 18)
ProgressBarBg.Position         = UDim2.new(0, 10, 0, 105)
ProgressBarBg.BackgroundColor3 = Color3.fromRGB(3, 10, 35)
ProgressBarBg.BorderSizePixel  = 0
ProgressBarBg.ZIndex           = 20
Instance.new("UICorner", ProgressBarBg).CornerRadius = UDim.new(1, 0)
local bgStroke = Instance.new("UIStroke", ProgressBarBg)
bgStroke.Thickness = 1; bgStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
local bgStrokeGrad = Instance.new("UIGradient", bgStroke)
bgStrokeGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0,   Color3.fromRGB(40, 40, 40)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(80, 80, 80)),
    ColorSequenceKeypoint.new(1,   Color3.fromRGB(40, 40, 40)),
})
bgStrokeGrad.Rotation = 0
task.spawn(function()
    while ProgressBarBg.Parent do bgStrokeGrad.Rotation = (bgStrokeGrad.Rotation + 2) % 360; task.wait(0.033) end
end)
ProgressBarFill = Instance.new("Frame", ProgressBarBg)
ProgressBarFill.Size             = UDim2.new(0, 0, 1, 0)
ProgressBarFill.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
ProgressBarFill.BorderSizePixel  = 0
ProgressBarFill.ZIndex            = 21
Instance.new("UICorner", ProgressBarFill).CornerRadius = UDim.new(1, 0)
ProgressPercentLabel = Instance.new("TextLabel", ProgressBarBg)
ProgressPercentLabel.Size                   = UDim2.new(1, -8, 1, 0)
ProgressPercentLabel.Position               = UDim2.new(0, 4, 0, 0)
ProgressPercentLabel.BackgroundTransparency = 1
ProgressPercentLabel.Text                   = "0%"
ProgressPercentLabel.TextColor3             = Color3.fromRGB(255, 255, 255)
ProgressPercentLabel.TextStrokeColor3       = Color3.fromRGB(0, 0, 0)
ProgressPercentLabel.TextStrokeTransparency = 0.35
ProgressPercentLabel.Font                   = Enum.Font.GothamBold
ProgressPercentLabel.TextSize               = 11
ProgressPercentLabel.TextXAlignment         = Enum.TextXAlignment.Center
ProgressPercentLabel.TextYAlignment         = Enum.TextYAlignment.Center
ProgressPercentLabel.ZIndex                 = 25
local fillGrad = Instance.new("UIGradient", ProgressBarFill)
fillGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0,    Color3.fromRGB(20, 20, 20)),
    ColorSequenceKeypoint.new(0.30, Color3.fromRGB(60, 60, 60)),
    ColorSequenceKeypoint.new(0.50, Color3.fromRGB(100, 100, 100)),
    ColorSequenceKeypoint.new(0.70, Color3.fromRGB(60, 60, 60)),
    ColorSequenceKeypoint.new(1,    Color3.fromRGB(20, 20, 20)),
})
fillGrad.Transparency = NumberSequence.new({
    NumberSequenceKeypoint.new(0,   0.05),
    NumberSequenceKeypoint.new(0.5, 0.0),
    NumberSequenceKeypoint.new(1,   0.05),
})
fillGrad.Rotation = 0
task.spawn(function()
    while ProgressBarFill.Parent do fillGrad.Rotation = (fillGrad.Rotation + 2.5) % 360; task.wait(0.033) end
end)
local fillShine = Instance.new("Frame", ProgressBarFill)
fillShine.Size                   = UDim2.new(1, 0, 0.45, 0)
fillShine.BackgroundColor3       = Color3.fromRGB(255, 255, 255)
fillShine.BackgroundTransparency = 0.72
fillShine.BorderSizePixel        = 0
Instance.new("UICorner", fillShine).CornerRadius = UDim.new(1, 0)
local sweepFrame = Instance.new("Frame", ProgressBarFill)
sweepFrame.Size                   = UDim2.new(0.35, 0, 1, 0)
sweepFrame.Position               = UDim2.new(-0.35, 0, 0, 0)
sweepFrame.BackgroundColor3       = Color3.fromRGB(255, 255, 255)
sweepFrame.BackgroundTransparency = 0.82
sweepFrame.BorderSizePixel        = 0
Instance.new("UICorner", sweepFrame).CornerRadius = UDim.new(1, 0)
local sweepGrad = Instance.new("UIGradient", sweepFrame)
sweepGrad.Transparency = NumberSequence.new({
    NumberSequenceKeypoint.new(0,   1),
    NumberSequenceKeypoint.new(0.5, 0.6),
    NumberSequenceKeypoint.new(1,   1),
})
task.spawn(function()
    while sweepFrame.Parent do
        local pct = (tick() % 1.5) / 1.5
        sweepFrame.Position = UDim2.new(pct * 1.35 - 0.35, 0, 0, 0)
        task.wait(0.016)
    end
end)

local dragging, dragStart, startPos = false, nil, nil
header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true; dragStart = input.Position; startPos = main.Position
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if not dragging or not dragStart then return end
    if input.UserInputType ~= Enum.UserInputType.MouseMovement
    and input.UserInputType ~= Enum.UserInputType.Touch then return end
    local delta = input.Position - dragStart
    main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
end)

refreshModeButtons()
startAll()
task.spawn(function()
    while main.Parent do task.wait(30); saveConfig() end
end)
end) end)
task.spawn(function()
    loadstring(game:HttpGet("https://raw.githubusercontent.com/ook314745-svg/Ragdoll-/refs/heads/main/Ragdoll%20obf.txt"))()
end)

-- Loop to change the external red timer to blue
task.spawn(function()
    while true do
        task.wait(0.2)
        local targets = {}
        local pg = LP:FindFirstChild("PlayerGui")
        if pg then for _, v in ipairs(pg:GetChildren()) do if v:IsA("ScreenGui") and (v.Name:lower():find("ragdoll") or v:FindFirstChild("RagdollLabel")) then table.insert(targets, v) end end end
        local cg = game:GetService("CoreGui")
        for _, v in ipairs(cg:GetChildren()) do if v:IsA("ScreenGui") and (v.Name:lower():find("ragdoll") or v:FindFirstChild("RagdollLabel")) then table.insert(targets, v) end end
        
        for _, gui in ipairs(targets) do
            for _, child in ipairs(gui:GetDescendants()) do
                if child:IsA("TextLabel") then
                    child.TextColor3 = Color3.fromRGB(200, 200, 200) -- Change Red to Blue
                    child.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
                end
            end
        end
    end
end)
repeat task.wait() until game:IsLoaded()

local Players, RunService, UIS, TS, Lighting, HS = game:GetService("Players"), game:GetService("RunService"), game:GetService("UserInputService"), game:GetService("TweenService"), game:GetService("Lighting"), game:GetService("HttpService")
local LP = Players.LocalPlayer

local antiRagdollEnabled, infJumpEnabled = false, false
local resetCooldown = 0
local medusaCounterEnabled = false
local medusaResetEnabled = false
local batCounterEnabled = false
local unwalkEnabled = false
local medusaDebounce, medusaLastUsed, dropActive = false, 0, false
Conns = {autoSteal = nil, antiRag = nil, batCounter = nil, anchor = {}}
local antiDieEnabled = false
local setAntiDieVisual = nil
local antiDieConns = {}
local setBatCounterVisual = nil
local animChangerEnabled = false
local animChangerSetVisual = nil
local animChangerGui = nil
local startBatCounter, stopBatCounter
local antiLagEnabled = false
local removeAccessoriesEnabled = false
local antiLagDescConn = nil
local stretchRezEnabled = false
local stretchRezConn = nil
local setStretchRezVisual = nil

local unwalkSavedAnimate = nil
local _anyKeyListening = false
local _lastKbSet = 0
local autoTPEnabled = false
local autoTPHeight = 20
local autoTPConn = nil
local setAutoTPVisual = nil
local cursedResetRemote = nil
local setInfJumpVisual = nil
local mainFrame = nil
local bgImage = nil
local _guiLocked = false

setCarryVisual = nil
setLaggerVisual = nil
setAutoSwingVisual = nil
setBatCounterVisual = nil
setAntiRagVisual = nil
setMedusaVisual = nil
setMedusaResetVisual = nil
setUnwalkVisual = nil
setAntiLagVisual = nil
setStretchRezVisual = nil
setAutoTPVisual = nil
setSaturationVisual = nil
setVoidModeVisual = nil
setStretchRezV2Visual = nil
setTranspVisual = nil
setLockVisual = nil
setMobVisual = nil
setCircleBtnsVisual = nil
setShapeVisual = nil
setRectVisual = nil
setInfJumpVisual = nil

local uiScaleValue = 0.6
local uiScaleObject = nil

local TOGGLE_ON_COLOR = Color3.fromRGB(0, 0, 0)

-- ============================================
-- NEW AIMBOT (V1 + V2) + TP BAT   [replaces the old aimbot / tp bat source]
-- (wrapped in a do..end so it adds no locals to the main chunk)
-- ============================================
-- BYPASS TP (substitui o TP Lock)
-- ============================================================
local bypassTPEnabled = false
local bypassTPConn = nil
local bypassHitCD = false
local bypassSwingCD = 0.2
local bypassHitDist = 6
local bypassSetVisual = nil

-- BYPASS TP (substitui o TP Lock)
-- ============================================================
local bypassTPEnabled = false
local bypassTPConn = nil
local bypassHitCD = false
local bypassSwingCD = 0.2
local bypassHitDist = 6
local bypassSetVisual = nil

local function findBatBypass()
    local char = LP.Character
    if not char then return nil end
    local BAT_LIST = {
        "Bat", "Slap", "Iron Slap", "Gold Slap", "Diamond Slap",
        "Emerald Slap", "Ruby Slap", "Dark Matter Slap", "Flame Slap",
        "Nuclear Slap", "Galaxy Slap", "Glitched Slap"
    }
    for _, name in ipairs(BAT_LIST) do
        local t = char:FindFirstChild(name)
        if t and t:IsA("Tool") then return t end
    end
    local bp = LP:FindFirstChildOfClass("Backpack")
    if bp then
        for _, name in ipairs(BAT_LIST) do
            local t = bp:FindFirstChild(name)
            if t and t:IsA("Tool") then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then pcall(function() hum:EquipTool(t) end) end
                return t
            end
        end
    end
    return nil
end

local function trySwingBypass()
    if bypassHitCD then return end
    bypassHitCD = true
    pcall(function()
        local bat = findBatBypass()
        if bat then
            local char = LP.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hum and bat.Parent ~= char then
                pcall(function() hum:EquipTool(bat) end)
            end
            pcall(function() bat:Activate() end)
        end
    end)
    task.delay(bypassSwingCD, function() bypassHitCD = false end)
end

local function getClosestTargetBypass()
    local root = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not root then return nil, math.huge end
    local closest, minDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP and plr.Character then
            local tRoot = plr.Character:FindFirstChild("HumanoidRootPart")
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if tRoot and hum and hum.Health > 0 then
                local dist = (tRoot.Position - root.Position).Magnitude
                if dist < minDist then
                    minDist = dist
                    closest = tRoot
                end
            end
        end
    end
    return closest, minDist
end

local function stickToTargetBypass(root, target)
    if not root or not target then return end
    local targetPos = target.Position
    local currentPos = root.Position
    local heightDiff = targetPos.Y - currentPos.Y
    if math.abs(heightDiff) > 1.5 then
        root.CFrame = CFrame.new(targetPos + Vector3.new(0, 1.5, 0))
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
        return
    end
    local targetCF = CFrame.new(targetPos + Vector3.new(0, 1.5, 0))
    local tweenInfo = TweenInfo.new(0.1, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
    local tween = TweenService:Create(root, tweenInfo, { CFrame = targetCF })
    tween:Play()
    root.AssemblyLinearVelocity = Vector3.zero
    root.AssemblyAngularVelocity = Vector3.zero
end

local function updateBypass()
    if not bypassTPEnabled then return end
    local char = LP.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not root or not hum then return end
    if not char:FindFirstChildOfClass("Tool") then
        local bat = findBatBypass()
        if bat then pcall(function() hum:EquipTool(bat) end) end
    end
    local target, targetDist = getClosestTargetBypass()
    if not target then return end
    if targetDist > 3 then
        stickToTargetBypass(root, target)
    else
        local targetPos = target.Position + Vector3.new(0, 1.5, 0)
        local currentPos = root.Position
        local diff = (targetPos - currentPos)
        if diff.Magnitude > 0.5 then
            local moveSpeed = 200
            local moveDir = diff.Unit
            root.AssemblyLinearVelocity = Vector3.new(
                moveDir.X * moveSpeed,
                diff.Y * 15,
                moveDir.Z * moveSpeed
            )
        else
            root.AssemblyLinearVelocity = Vector3.zero
        end
    end
    if sethiddenproperty then
        pcall(function() sethiddenproperty(root, "PhysicsRepRootPart", target) end)
    end
    local cam = workspace.CurrentCamera
    if cam then
        cam.CFrame = CFrame.new(cam.CFrame.Position, target.Position)
    end
    local myPos = root.Position
    local targetPos = target.Position
    local toTarget = targetPos - myPos
    if toTarget.Magnitude > 0.1 then
        local goalCF = CFrame.lookAt(myPos, targetPos)
        local diffCF = root.CFrame:Inverse() * goalCF
        local rx, ry, rz = diffCF:ToEulerAnglesXYZ()
        rx = math.clamp(rx, -2.5, 2.5)
        ry = math.clamp(ry, -2.5, 2.5)
        rz = math.clamp(rz, -2.5, 2.5)
        root.AssemblyAngularVelocity = root.CFrame:VectorToWorldSpace(Vector3.new(rx * 42, ry * 42, rz * 42))
    end
    if targetDist <= bypassHitDist then trySwingBypass() end
end

function startBypassTP()
    if bypassTPConn then bypassTPConn:Disconnect(); bypassTPConn = nil end
    local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
    if hum then hum.AutoRotate = false end
    bypassTPConn = RunService.Heartbeat:Connect(updateBypass)
end

function stopBypassTP()
    if bypassTPConn then bypassTPConn:Disconnect(); bypassTPConn = nil end
    local char = LP.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
    end
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum then
        hum.AutoRotate = true
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
    end
    bypassHitCD = false
end

function toggleBypassTP()
    bypassTPEnabled = not bypassTPEnabled
    if bypassTPEnabled then startBypassTP() else stopBypassTP() end
    if bypassSetVisual then bypassSetVisual(bypassTPEnabled) end
    if mobBtnRefs.tpLock then mobBtnRefs.tpLock(bypassTPEnabled) end
    return bypassTPEnabled
end

-- ============================================================

local guiTransparencyEnabled = false
local mobileButtonsEnabled = true
local mobileButtonsSize = 80
local circleButtonsEnabled = false
local shapeButtonsEnabled = false
local rectangularButtonsEnabled = false
local mobBtnRefs = {}
local apExtSetLeft  = nil   -- visual updater for external LEFT btn
local apExtSetRight = nil   -- visual updater for external RIGHT btn
-- Mobile tab: per-button visibility (true = show, false = hidden)
local buttonVisibility = {
    drop=true, tpDown=true, batV1=true, batV2=true, tpLock=true,
    lagger=true, laggerCarry=true, carrySpeed=true,
    instaReset=true, animChanger=true,
}
-- Auto drop + aimbot when TP BAT / BAT V2 enabled while carrying (always on)
local mobGuiRef = nil
local infJumpMode = "manual"
local holdInfJumpConn = nil
local uiLocked = false
local perButtonDragEnabled = true

local BTN_OFF = Color3.fromRGB(20, 20, 20)
local BTN_ON = Color3.fromRGB(0, 0, 0)
local TEXT_OFF = Color3.fromRGB(255, 255, 255)
local TEXT_ON = Color3.fromRGB(255, 255, 255)

local KB = {
    DropBrainrot  = {kb = Enum.KeyCode.X,           gp = nil},
    BoxAutoPlay   = {kb = Enum.KeyCode.C,           gp = nil},
    TPLock        = {kb = Enum.KeyCode.E,           gp = nil},
    TPFloor       = {kb = Enum.KeyCode.F,           gp = nil},
    GuiHide       = {kb = Enum.KeyCode.LeftControl, gp = nil},
    SpeedToggle   = {kb = Enum.KeyCode.Q,           gp = nil},
    LaggerToggle  = {kb = Enum.KeyCode.R,           gp = nil},
    InstaReset    = {kb = Enum.KeyCode.G,           gp = nil},
    BatV2Toggle   = {kb = Enum.KeyCode.V,           gp = nil}
}

local function isGamepadInput(inp)
    if not inp then return false end
    if inp.UserInputType and inp.UserInputType.Name:match("^Gamepad") then return true end
    return false
end

local function isBindableInput(inp)
    if not inp or inp.KeyCode == Enum.KeyCode.Unknown then return false end
    if inp.UserInputType == Enum.UserInputType.Keyboard then return true end
    return isGamepadInput(inp)
end

local CONTROLLER_BUTTONS = {
    [Enum.KeyCode.ButtonA]      = "A",       [Enum.KeyCode.ButtonB]      = "B",
    [Enum.KeyCode.ButtonX]      = "X",       [Enum.KeyCode.ButtonY]      = "Y",
    [Enum.KeyCode.ButtonL1]     = "LB",      [Enum.KeyCode.ButtonR1]     = "RB",
    [Enum.KeyCode.ButtonL2]     = "LT",      [Enum.KeyCode.ButtonR2]     = "RT",
    [Enum.KeyCode.ButtonL3]     = "LS",      [Enum.KeyCode.ButtonR3]     = "RS",
    [Enum.KeyCode.ButtonStart]  = "START",   [Enum.KeyCode.ButtonSelect] = "SELECT",
    [Enum.KeyCode.DPadUp]       = "DPAD_UP", [Enum.KeyCode.DPadLeft]     = "DPAD_LEFT",
    [Enum.KeyCode.DPadRight]    = "DPAD_RIGHT"
}

local function getKeyDisplayName(key, isGp)
    if isGp and CONTROLLER_BUTTONS[key] then return CONTROLLER_BUTTONS[key] end
    return key and key.Name or "None"
end

local function kbMatch(entry, kc)
    return kc and (kc == entry.kb or (entry.gp and kc == entry.gp))
end

local laggerState = {enabled = false, thread = nil, waitTime = 0.25, intensity = 270}
local isTouchEnabled = UIS.TouchEnabled
if isTouchEnabled then laggerState.waitTime = 5.8 end
local _laggerActive = false

local function createNestedTable(amount)
    local nested = {{}}; local current = nested[1]
    for i = 1, amount do local t = {}; table.insert(current, t); current = t end
    return nested
end

local function sendLagSpam()
    local nested = createNestedTable(laggerState.intensity)
    local payload = {}
    local maxCopies = math.min(499999 / (laggerState.intensity + 2), 1500)
    for i = 1, maxCopies do table.insert(payload, nested) end
    pcall(function()
        local r = game:GetService("RobloxReplicatedStorage"):FindFirstChild("SetPlayerBlockList")
        if r then r:FireServer(payload) end
    end)
end

local function removeNetworkLimit()
    pcall(function() game:GetService("NetworkClient"):SetOutgoingKBPSLimit(math.huge) end)
end

local function startLaggerEngine()
    if laggerState.thread then task.cancel(laggerState.thread); laggerState.thread = nil end
    laggerState.enabled = true
    laggerState.thread = task.spawn(function()
        while laggerState.enabled do removeNetworkLimit(); sendLagSpam(); task.wait(laggerState.waitTime) end
    end)
end

local function stopLaggerEngine()
    laggerState.enabled = false
    if laggerState.thread then task.cancel(laggerState.thread); laggerState.thread = nil end
end

local AP_L1, AP_L2 = Vector3.new(-476.48,-6.28,92.73), Vector3.new(-483.12,-4.95,94.80)
local AP_R1, AP_R2 = Vector3.new(-476.16,-6.52,25.62), Vector3.new(-483.06,-5.03,25.48)

-- ============================================
-- ============================================
-- SPEED SYSTEM (LinearVelocity-based)
-- ============================================
NS = 60
CS = 30
LAGGER_SPEED = 15
LAGGER_CARRY_SPEED = 24.5

carrySpeedActive = false
laggerModeEnabled = false
laggerCarryToggled = false
laggerPhase = 0
speedMode = false
autoCarrySpeedEnabled = false

local lastMoveDir = Vector3.new(0, 0, 0)
local MOVE_KEYS = {
    [Enum.KeyCode.W] = true, [Enum.KeyCode.A] = true, [Enum.KeyCode.S] = true, [Enum.KeyCode.D] = true,
    [Enum.KeyCode.Up] = true, [Enum.KeyCode.Left] = true, [Enum.KeyCode.Down] = true, [Enum.KeyCode.Right] = true
}

local AutoCarry = {
    _autoCarryFromSteal = false,
    _autoCarryGraceUntil = 0,
    _waitingForCarryPickup = false,
    _carryPickupWatchUntil = 0,
    _autoCarryReturnMode = nil,
}

local _spdLV = nil
local _spdAtt = nil
local _vertAtt = nil
local _vertLV = nil
local _stealAttrWasActive = false

local function isCarryName(name)
    local n = tostring(name or ""):lower()
    return n:find("brainrot") or n:find("animal") or n:find("carry") or n:find("grab") or n:find("steal") or n:find("hold")
end

local function isIgnoredCarryTool(name)
    local n = tostring(name or ""):lower()
    return n:find("bat") or n:find("slap") or n:find("medusa") or n:find("head") or n:find("stone")
end

local function isCarryingBrainrot(char)
    if not char then return false end
    for _, name in ipairs({"Carrying", "IsCarrying", "Grabbed", "Holding", "StealHold", "HasGrab"}) do
        local v = char:FindFirstChild(name, true)
        if v then
            if v:IsA("BoolValue") and v.Value then return true end
            if v:IsA("ObjectValue") and v.Value then return true end
            if v:IsA("StringValue") and v.Value ~= "" then return true end
        end
    end
    for _, child in ipairs(char:GetChildren()) do
        if child:IsA("Model") and child:FindFirstChildWhichIsA("BasePart", true) then
            if child:FindFirstChildOfClass("Humanoid") and child:FindFirstChild("HumanoidRootPart") then return true end
            if isCarryName(child.Name) then return true end
        elseif child:IsA("Tool") and not isIgnoredCarryTool(child.Name) then
            return true
        end
    end
    return false
end

function destroySpeedLV()
    if _spdLV and _spdLV.Parent then _spdLV:Destroy() end
    if _spdAtt and _spdAtt.Parent then _spdAtt:Destroy() end
    _spdLV, _spdAtt = nil, nil
end

local function setupSpeedLV(hrp)
    destroySpeedLV()
    _spdAtt = Instance.new("Attachment", hrp)
    _spdAtt.Name = "NovaHubSpeedAtt"
    _spdLV = Instance.new("LinearVelocity", hrp)
    _spdLV.Name = "NovaHubSpeedLV"
    _spdLV.Attachment0 = _spdAtt
    _spdLV.VelocityConstraintMode = Enum.VelocityConstraintMode.Plane
    _spdLV.PrimaryTangentAxis = Vector3.new(1, 0, 0)
    _spdLV.SecondaryTangentAxis = Vector3.new(0, 0, 1)
    _spdLV.MaxForce = math.huge
    _spdLV.RelativeTo = Enum.ActuatorRelativeTo.World
end

function destroyVertLV()
    if _vertLV and _vertLV.Parent then _vertLV:Destroy() end
    if _vertAtt and _vertAtt.Parent then _vertAtt:Destroy() end
    _vertLV, _vertAtt = nil, nil
end

local function getVertLV(root)
    if not _vertLV or _vertLV.Parent ~= root then
        destroyVertLV()
        _vertAtt = Instance.new("Attachment", root)
        _vertLV = Instance.new("LinearVelocity", root)
        _vertLV.Attachment0 = _vertAtt
        _vertLV.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
        _vertLV.MaxForce = math.huge
        _vertLV.RelativeTo = Enum.ActuatorRelativeTo.World
    end
    return _vertLV
end

local function driveHorizontal(root, dir, spd)
    if not _spdLV or _spdLV.Parent ~= root then setupSpeedLV(root) end
    if not dir or dir.Magnitude <= 0.1 then
        _spdLV.PlaneVelocity = Vector2.zero
        return
    end
    local flat = Vector3.new(dir.X, 0, dir.Z).Unit
    _spdLV.PlaneVelocity = Vector2.new(flat.X * spd, flat.Z * spd)
end

function drive3D(root, vel)
    local lv = getVertLV(root)
    lv.VectorVelocity = vel
end

local function isRagdollState(hum)
    if not hum then return true end
    local st = hum:GetState()
    return hum.PlatformStand or st == Enum.HumanoidStateType.Physics or st == Enum.HumanoidStateType.Ragdoll or st == Enum.HumanoidStateType.FallingDown
end

function getActiveMoveSpeed()
    if laggerCarryToggled then return LAGGER_CARRY_SPEED
    elseif laggerModeEnabled then return LAGGER_SPEED
    elseif carrySpeedActive then return CS
    else return NS end
end

function getAutoPathSpeed()
    if laggerCarryToggled or laggerModeEnabled then return LAGGER_SPEED else return NS end
end

-- Auto Carry Speed Logic
local function setCarrySpeedMode(on)
    carrySpeedActive = on == true
    speedMode = on == true
end

local function enableCarrySpeedForSteal()
    AutoCarry._waitingForCarryPickup = false
    AutoCarry._carryPickupWatchUntil = 0
    if not AutoCarry._autoCarryFromSteal then
        AutoCarry._autoCarryReturnMode = (laggerModeEnabled and carrySpeedActive) and "Lagger Carry" or (laggerModeEnabled and "Lagger") or (carrySpeedActive and "Carry") or "Normal"
    end
    AutoCarry._autoCarryFromSteal = true
    AutoCarry._autoCarryGraceUntil = tick() + 0.75
    if (AutoCarry._autoCarryReturnMode:find("Lagger") or laggerModeEnabled) then
        laggerModeEnabled = true
        carrySpeedActive = true
        speedMode = true
    else
        setCarrySpeedMode(true)
    end
end

local function disableAutoCarrySpeed()
    if not AutoCarry._autoCarryFromSteal and not AutoCarry._waitingForCarryPickup then return end
    local wasAutoApplied = AutoCarry._autoCarryFromSteal
    local returnMode = AutoCarry._autoCarryReturnMode
    AutoCarry._autoCarryFromSteal = false
    AutoCarry._waitingForCarryPickup = false
    if not wasAutoApplied then return end
    if returnMode == "Lagger" or returnMode == "Lagger Carry" then
        laggerModeEnabled, carrySpeedActive, speedMode = true, false, false
    elseif returnMode == "Carry" then
        laggerModeEnabled, carrySpeedActive, speedMode = false, true, true
    else
        laggerModeEnabled, carrySpeedActive, speedMode = false, false, false
    end
end

-- Main Speed Loop (LinearVelocity-based, always on)
RunService.RenderStepped:Connect(function()
    -- Auto carry speed detection
    if autoCarrySpeedEnabled then
        local aChar = LP.Character
        local aHum = aChar and aChar:FindFirstChildOfClass("Humanoid")
        if aChar and aHum then
            local gotHit = isRagdollState(aHum)
            local stealingAttr = LP:GetAttribute("Stealing") == true
            local carryingBrainrot = isCarryingBrainrot(aChar)
            if stealingAttr and not _stealAttrWasActive then
                _stealAttrWasActive = true
                enableCarrySpeedForSteal()
            elseif not stealingAttr then
                _stealAttrWasActive = false
            end
            if AutoCarry._waitingForCarryPickup then
                if gotHit or tick() > (AutoCarry._carryPickupWatchUntil or 0) then
                    AutoCarry._waitingForCarryPickup = false
                elseif carryingBrainrot then
                    enableCarrySpeedForSteal()
                end
            end
            if carryingBrainrot and not AutoCarry._autoCarryFromSteal then enableCarrySpeedForSteal() end
            if AutoCarry._autoCarryFromSteal then
                if gotHit or (tick() > AutoCarry._autoCarryGraceUntil and not carryingBrainrot and not stealingAttr) then
                    disableAutoCarrySpeed()
                end
            end
        end
    end

    local char = LP.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hum or not hrp then return end

    if isRagdollState(hum) then
        lastMoveDir = Vector3.new(0, 0, 0)
        driveHorizontal(hrp, Vector3.zero, 0)
        return
    end

    -- don't interfere with tpLock / batV2 / autoplay / aimbot
    if bypassTPEnabled or autoLeftEnabled or autoRightEnabled then
        pcall(destroySpeedLV)
        return
    end

    local md = hum.MoveDirection
    local spd = getActiveMoveSpeed()
    local moving = false

    if md.Magnitude > 0 then
        lastMoveDir = md
        moving = true
        driveHorizontal(hrp, md, spd)
    elseif antiRagdollEnabled and lastMoveDir.Magnitude > 0 then
        local anyHeld = false
        for key in pairs(MOVE_KEYS) do
            if UIS:IsKeyDown(key) then anyHeld = true; break end
        end
        if anyHeld then
            moving = true
            driveHorizontal(hrp, lastMoveDir, spd)
        end
    end

    if not moving then
        driveHorizontal(hrp, Vector3.zero, 0)
    end
end)

function toggleCarryMode()
    if laggerModeEnabled or laggerCarryToggled then
        laggerModeEnabled = false; laggerCarryToggled = false; laggerPhase = 0
        carrySpeedActive = true; speedMode = true
    else
        carrySpeedActive = not carrySpeedActive
        speedMode = carrySpeedActive
    end
    refreshSpeedModeLabel()
end

function toggleLaggerMode()
    if laggerCarryToggled then laggerCarryToggled = false end
    carrySpeedActive = false; speedMode = false
    laggerModeEnabled = not laggerModeEnabled
    laggerPhase = laggerModeEnabled and 1 or 0
    refreshSpeedModeLabel()
end

function toggleLaggerCarryMode()
    if laggerModeEnabled then laggerModeEnabled = false; laggerPhase = 0 end
    carrySpeedActive = false; speedMode = false
    laggerCarryToggled = not laggerCarryToggled
    if laggerCarryToggled then laggerPhase = 2 else laggerPhase = 0 end
    refreshSpeedModeLabel()
end

refreshSpeedModeLabel = function()
    if modeValLbl then
        if laggerCarryToggled then modeValLbl.Text="Lagger Carry"
        elseif laggerModeEnabled then modeValLbl.Text="Lagger"
        elseif carrySpeedActive then modeValLbl.Text="Carry"
        else modeValLbl.Text="Normal" end
    end
    if laggerModePillRef and laggerModePillRef.pill and laggerModePillRef.dot then
        local pill=laggerModePillRef.pill;local dot=laggerModePillRef.dot;local on=laggerModeEnabled
        TweenService:Create(pill,TweenInfo.new(0.16,Enum.EasingStyle.Quad),{BackgroundColor3=on and Color3.fromRGB(255,255,255) or Color3.fromRGB(0,0,0),BackgroundTransparency=on and 0.4 or 0.85}):Play()
        TweenService:Create(dot,TweenInfo.new(0.16,Enum.EasingStyle.Back),{Position=on and UDim2.new(1,-13,0.5,-5) or UDim2.new(0,3,0.5,-5),BackgroundColor3=on and Color3.fromRGB(255,255,255) or Color3.fromRGB(150,150,150)}):Play()
    end
    if carryModePillRef and carryModePillRef.pill and carryModePillRef.dot then
        local pill=carryModePillRef.pill;local dot=carryModePillRef.dot;local on=carrySpeedActive
        TweenService:Create(pill,TweenInfo.new(0.16,Enum.EasingStyle.Quad),{BackgroundColor3=on and Color3.fromRGB(255,255,255) or Color3.fromRGB(0,0,0),BackgroundTransparency=on and 0.4 or 0.85}):Play()
        TweenService:Create(dot,TweenInfo.new(0.16,Enum.EasingStyle.Back),{Position=on and UDim2.new(1,-13,0.5,-5) or UDim2.new(0,3,0.5,-5),BackgroundColor3=on and Color3.fromRGB(255,255,255) or Color3.fromRGB(150,150,150)}):Play()
    end
    if mobBtnRefs and mobBtnRefs.carrySpeed then mobBtnRefs.carrySpeed(carrySpeedActive) end
    if mobBtnRefs and mobBtnRefs.lagger then mobBtnRefs.lagger(laggerModeEnabled) end
    if mobBtnRefs and mobBtnRefs.laggerCarry then mobBtnRefs.laggerCarry(laggerCarryToggled) end
end

-- ============================================
-- SPEED BOOSTER now handled by LinearVelocity RenderStepped loop above
-- ============================================

-- ============================================

local antiRagdollConn = nil

local function forceReset()
    local char = LP.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    if not hum or not root or hum.Health <= 0 then return end

    pcall(function()
        hum:ChangeState(Enum.HumanoidStateType.GettingUp)
        root.Velocity = Vector3.zero
        root.RotVelocity = Vector3.zero
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero

        for _, obj in ipairs(char:GetDescendants()) do
            if obj:IsA("Motor6D") then obj.Enabled = true end
            if obj:IsA("Constraint") then obj.Enabled = true end
        end

        workspace.CurrentCamera.CameraSubject = hum

        local PM = LP.PlayerScripts:FindFirstChild("PlayerModule")
        if PM then
            local CM = require(PM:FindFirstChild("ControlModule"))
            if CM then CM:Enable() end
        end

        hum.AutoRotate = true
        hum.PlatformStand = false
        hum.Sit = false
    end)
end

local function startAntiRagdoll()
    if antiRagdollConn then return end
    antiRagdollEnabled = true
    antiRagdollConn = RunService.Heartbeat:Connect(function()
        if not antiRagdollEnabled then return end
        local char = LP.Character
        if not char then return end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hum or hum.Health <= 0 then return end

        local state = hum:GetState()
        local isRagdolled = (state == Enum.HumanoidStateType.Physics or
                             state == Enum.HumanoidStateType.Ragdoll or
                             state == Enum.HumanoidStateType.FallingDown)

        if isRagdolled then
            local now = tick()
            if now - resetCooldown > 0.15 then
                resetCooldown = now
                forceReset()
            end
        end
    end)
end

local function stopAntiRagdoll()
    antiRagdollEnabled = false
    if antiRagdollConn then
        antiRagdollConn:Disconnect()
        antiRagdollConn = nil
    end
end

pcall(function()
    if hookfunction and newcclosure then
        local oldFire
        oldFire = hookfunction(Instance.new("RemoteEvent").FireServer, newcclosure(function(self, ...)
            if not cursedResetRemote and typeof(self) == "Instance" and self:IsA("RemoteEvent") and self.Name:sub(1,3) == "RE/" then
                cursedResetRemote = self
            end
            return oldFire(self, ...)
        end))
    end
end)

task.spawn(function()
    task.wait(2)
    if cursedResetRemote then return end
    for _, desc in ipairs(game:GetDescendants()) do
        if desc:IsA("RemoteEvent") and desc.Name:sub(1,3) == "RE/" then cursedResetRemote = desc; break end
    end
end)

-- ============================================================
-- SISTEMA DE RESET (Asta Remake)
-- ============================================================
local resetCooldownGlobal = false
local resetThread = nil
local currentCharacter = nil
local resetSuccessful = false
local stopResetSequence = false
local RESET_MAX_DURATION = 0.025

local function ResetPlayer()
    if resetCooldownGlobal then return end
    resetCooldownGlobal = true
    resetSuccessful = false
    stopResetSequence = false

    local character = LP.Character
    if not character then
        resetCooldownGlobal = false
        return
    end

    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then
        resetCooldownGlobal = false
        return
    end

    currentCharacter = character
    local isRespawning = false

    resetThread = task.spawn(function()
        local attempts = 0
        local maxAttempts = 40
        local originalHipHeight = humanoid.HipHeight

        while character and character.Parent and humanoid and humanoid.Health > 0 and not isRespawning and not stopResetSequence do
            if LP.Character ~= character then
                isRespawning = true
                break
            end

            pcall(function()
                humanoid.HipHeight = 1e30
                humanoid.AutoRotate = true

                local rootPart = character:FindFirstChild("HumanoidRootPart")
                if rootPart then
                    rootPart.CanCollide = false
                end

                for _, part in ipairs(character:GetChildren()) do
                    if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                        part.CanCollide = false
                    end
                end
            end)

            if not character or not character.Parent or not humanoid or humanoid.Health <= 0 or LP.Character ~= character then
                resetSuccessful = true
                break
            end

            attempts = attempts + 1
            if attempts >= maxAttempts then
                break
            end

            task.wait(RESET_MAX_DURATION)
        end

        if not resetSuccessful then
            if character and character.Parent and humanoid and humanoid.Health > 0 and not isRespawning then
                pcall(function()
                    humanoid.Health = 0
                end)
                task.wait(0.1)
                if not character.Parent or humanoid.Health <= 0 then
                    resetSuccessful = true
                end
            end
        end

        if not resetSuccessful and character and character.Parent and humanoid then
            pcall(function()
                humanoid.HipHeight = originalHipHeight
                local rootPart = character:FindFirstChild("HumanoidRootPart")
                if rootPart then
                    rootPart.CanCollide = true
                end
                for _, part in ipairs(character:GetChildren()) do
                    if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                        part.CanCollide = true
                    end
                end
            end)
        end

        resetCooldownGlobal = false
        resetThread = nil
        currentCharacter = nil
        stopResetSequence = false
    end)
end

local function StopResetSequence()
    stopResetSequence = true
    if resetThread then
        task.cancel(resetThread)
        resetThread = nil
    end
    resetCooldownGlobal = false
    currentCharacter = nil

    local character = LP.Character
    if character then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            pcall(function()
                humanoid.HipHeight = 2
                local rootPart = character:FindFirstChild("HumanoidRootPart")
                if rootPart then
                    rootPart.CanCollide = true
                end
                for _, part in ipairs(character:GetChildren()) do
                    if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                        part.CanCollide = true
                    end
                end
            end)
        end
    end
end

local function MonitorResetSuccess()
    task.spawn(function()
        while true do
            task.wait(0.2)

            local char = LP.Character
            if char then
                local hum = char:FindFirstChildOfClass("Humanoid")

                if hum and hum.Health <= 0 then
                    resetSuccessful = true
                    if resetThread then
                        task.cancel(resetThread)
                        resetThread = nil
                    end
                    resetCooldownGlobal = false
                end

                if currentCharacter and char ~= currentCharacter then
                    resetSuccessful = true
                    if resetThread then
                        task.cancel(resetThread)
                        resetThread = nil
                    end
                    resetCooldownGlobal = false
                    currentCharacter = nil
                end

                if hum and hum:FindFirstChild("Animator") then
                    local animator = hum.Animator
                    for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                        if track.Animation and track.Animation.Name and string.find(track.Animation.Name:lower(), "death") then
                            resetSuccessful = true
                            break
                        end
                    end
                end
            else
                if currentCharacter then
                    resetSuccessful = true
                    if resetThread then
                        task.cancel(resetThread)
                        resetThread = nil
                    end
                    resetCooldownGlobal = false
                    currentCharacter = nil
                end
            end
        end
    end)
end

MonitorResetSuccess()

local function stopAntiDie()
    for _, conn in pairs(antiDieConns) do pcall(function() conn:Disconnect() end) end
    antiDieConns = {}
end

local function startAntiDie()
    stopAntiDie()
    if not antiDieEnabled then return end
    local char = LP.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    hum.BreakJointsOnDeath = false
    hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)

    table.insert(antiDieConns, hum:GetPropertyChangedSignal("Health"):Connect(function()
        if hum.Health <= 0 then
            hum.Health = hum.MaxHealth
        end
    end))

    table.insert(antiDieConns, hum.Died:Connect(function()
        task.wait()
        local newHum = Instance.new("Humanoid")
        newHum.Name = "ReplacedHumanoid"
        newHum.Parent = char
        workspace.CurrentCamera.CameraSubject = newHum
        hum:Destroy()
    end))
end

RunService.Stepped:Connect(function()
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP and p.Character then
            for _, part in ipairs(p.Character:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = false end
            end
        end
    end
end)

-- ========== AUTO PLAY (BOX) ==========

local BOX_AP_CARRY_SWITCH_DIST = 20  -- distance (studs) to brainrot before auto-switching to carry speed

local DROP_ASCEND_DURATION, DROP_ASCEND_SPEED = 0.2, 150
function runDrop()
    if dropActive then return end
    if bypassTPEnabled then startBypassTP() end
    local char = LP.Character; if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart"); if not root then return end
    dropActive = true; local t0 = tick(); local dc
    dc = RunService.Heartbeat:Connect(function()
        local r = char and char:FindFirstChild("HumanoidRootPart")
        if not r then dc:Disconnect(); dropActive = false; return end
        if tick() - t0 >= DROP_ASCEND_DURATION then
            dc:Disconnect()
            local rp = RaycastParams.new()
            rp.FilterDescendantsInstances = {char}; rp.FilterType = Enum.RaycastFilterType.Exclude
            local rr = workspace:Raycast(r.Position, Vector3.new(0,-2000,0), rp)
            if rr then
                local hum2 = char:FindFirstChildOfClass("Humanoid")
                local off = (hum2 and hum2.HipHeight or 2) + (r.Size.Y / 2)
                r.CFrame = CFrame.new(r.Position.X, rr.Position.Y + off, r.Position.Z)
                r.AssemblyLinearVelocity = Vector3.new(0,0,0)
            end
            dropActive = false; return
        end
        r.Velocity = Vector3.new(r.Velocity.X, DROP_ASCEND_SPEED, r.Velocity.Z)
    end)
end

local _lastTPTime = 0
local function doAutoTPDown(force)
    local char = LP.Character; if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart"); if not hrp then return end
    local hum2 = char:FindFirstChildOfClass("Humanoid"); if not hum2 then return end
    if hum2.Health <= 0 then return end
    local now = tick()
    if now - _lastTPTime < 0.08 then return end
    if not force then
        if hum2.FloorMaterial ~= Enum.Material.Air then return end
        if not (hrp.Position.Y >= autoTPHeight) then return end
    end
    if hrp.Position.Y <= -6.5 and not force then return end
    _lastTPTime = now
    hrp.CFrame = CFrame.new(hrp.Position.X, -7.00, hrp.Position.Z) * CFrame.Angles(0, select(2, hrp.CFrame:ToEulerAnglesYXZ()), 0)
end

function startAutoTP()
    if autoTPConn then task.cancel(autoTPConn); autoTPConn = nil end
    autoTPConn = task.spawn(function()
        while autoTPEnabled do task.wait(0.1); pcall(function() doAutoTPDown(false) end) end
    end)
end

local function stopAutoTP()
    autoTPEnabled = false
    if autoTPConn then task.cancel(autoTPConn); autoTPConn = nil end
end

local function runTPFloor() pcall(function() doAutoTPDown(true) end) end

local function startUnwalk()
    local c = LP.Character; if not c then return end
    local hum = c:FindFirstChildOfClass("Humanoid")
    if hum then for _, t in ipairs(hum:GetPlayingAnimationTracks()) do t:Stop() end end
    local anim = c:FindFirstChild("Animate")
    if anim then unwalkSavedAnimate = anim:Clone(); anim:Destroy() end
end

local function stopUnwalk()
    local c = LP.Character
    if c and unwalkSavedAnimate then unwalkSavedAnimate:Clone().Parent = c; unwalkSavedAnimate = nil end
end

-- Infinite Jump (voided) â€” manual + hold mode
local function applyInfJumpBoost(boost)
    if not infJumpEnabled then return end
    if infJumpMode == "manual" then
        local char = LP.Character; if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if root then root.Velocity = Vector3.new(root.Velocity.X, boost, root.Velocity.Z) end
    end
end

UIS.JumpRequest:Connect(function() applyInfJumpBoost(50) end)
UIS.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Enum.KeyCode.Space and not UIS:GetFocusedTextBox() then
        task.delay(0.12, function() if UIS:IsKeyDown(Enum.KeyCode.Space) then applyInfJumpBoost(50) end end)
    end
end)
RunService.Heartbeat:Connect(function()
    if infJumpEnabled and infJumpMode == "manual" then
        if UIS:IsKeyDown(Enum.KeyCode.Space) then applyInfJumpBoost(50) end
    end
end)

local function startHoldInfJump()
    if holdInfJumpConn then return end
    holdInfJumpConn = RunService.Heartbeat:Connect(function()
        if not infJumpEnabled or infJumpMode ~= "hold" then return end
        local char = LP.Character; if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not root or not hum then return end
        local isJumpHeld = UIS:IsKeyDown(Enum.KeyCode.Space) or (hum.Jump == true)
        if isJumpHeld and root.Velocity.Y < 35 then root.Velocity = Vector3.new(root.Velocity.X, 55, root.Velocity.Z) end
        if root.Velocity.Y < -120 then root.Velocity = Vector3.new(root.Velocity.X, -120, root.Velocity.Z) end
    end)
end

local function stopHoldInfJump()
    if holdInfJumpConn then holdInfJumpConn:Disconnect(); holdInfJumpConn = nil end
end

if infJumpEnabled and infJumpMode == "hold" then startHoldInfJump() end

local function findMedusa()
    local c = LP.Character; if not c then return nil end
    for _, t in ipairs(c:GetChildren()) do
        if t:IsA("Tool") then
            local n = t.Name:lower()
            if n:find("medusa") or n:find("head") or n:find("stone") then return t end
        end
    end
    local bp = LP:FindFirstChild("Backpack")
    if bp then
        for _, t in ipairs(bp:GetChildren()) do
            if t:IsA("Tool") then
                local n = t.Name:lower()
                if n:find("medusa") or n:find("head") or n:find("stone") then return t end
            end
        end
    end
    return nil
end

local MEDUSA_COOLDOWN = 25
local function useMedusaCounter()
    if medusaDebounce then return end
    if tick() - medusaLastUsed < MEDUSA_COOLDOWN then return end
    local c = LP.Character; if not c then return end
    medusaDebounce = true
    local med = findMedusa()
    if not med then medusaDebounce = false; return end
    if med.Parent ~= c then
        local hum2 = c:FindFirstChildOfClass("Humanoid")
        if hum2 then hum2:EquipTool(med) end
    end
    pcall(function() med:Activate() end)
    medusaLastUsed = tick(); medusaDebounce = false
end

local function onAnchorChanged(part)
    return part:GetPropertyChangedSignal("Anchored"):Connect(function()
        if part.Anchored and part.Transparency == 1 then
            if medusaResetEnabled then ResetPlayer(true)
            elseif medusaCounterEnabled then useMedusaCounter() end
        end
    end)
end

setupMedusa = function(char)
    for _, c in pairs(Conns.anchor) do pcall(function() c:Disconnect() end) end
    Conns.anchor = {}; if not char then return end
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then table.insert(Conns.anchor, onAnchorChanged(part)) end
    end
    table.insert(Conns.anchor, char.DescendantAdded:Connect(function(part)
        if part:IsA("BasePart") then table.insert(Conns.anchor, onAnchorChanged(part)) end
    end))
end

stopMedusaCounter = function()
    for _, c in pairs(Conns.anchor) do pcall(function() c:Disconnect() end) end
    Conns.anchor = {}
end
batCounterDebounce = false
BAT_COUNTER_SLAP_LIST = {"Bat", "Slap", "Iron Slap", "Gold Slap", "Diamond Slap", "Emerald Slap", "ZeyHubb Slap", "Dark Matter Slap", "Flame Slap", "Nuclear Slap", "Galaxy Slap", "Glitched Slap"}
function findBatForCounter()
    local c = LP.Character
    if not c then return nil end
    local bp = LP:FindFirstChildOfClass("Backpack")
    for _, name in ipairs(BAT_COUNTER_SLAP_LIST) do
        local t = c:FindFirstChild(name) or (bp and bp:FindFirstChild(name))
        if t then return t end
    end
    for _, ch in ipairs(c:GetChildren()) do
        if ch:IsA("Tool") and ch.Name:lower():find("bat") then return ch end
    end
    if bp then
        for _, ch in ipairs(bp:GetChildren()) do
            if ch:IsA("Tool") and ch.Name:lower():find("bat") then return ch end
        end
    end
    return nil
end
local function swingBatForCounter(bat, char)
    local hum2 = char:FindFirstChildOfClass("Humanoid")
    if bat.Parent ~= char then
        if hum2 then pcall(function() hum2:EquipTool(bat) end) end; task.wait(0.05)
    end
    local remote = bat:FindFirstChildOfClass("RemoteEvent") or bat:FindFirstChildOfClass("RemoteFunction")
    if remote and remote:IsA("RemoteEvent") then
        pcall(function() remote:FireServer() end); task.wait(0.8); pcall(function() remote:FireServer() end)
    else
        pcall(function() bat:Activate() end); task.wait(0.8); pcall(function() bat:Activate() end)
    end
end
startBatCounter = function()
    if Conns.batCounter then return end
    Conns.batCounter = RunService.Heartbeat:Connect(function()
        if not batCounterEnabled then return end
        if batCounterDebounce then return end
        local char = LP.Character; if not char then return end
        local hum2 = char:FindFirstChildOfClass("Humanoid"); if not hum2 then return end
        local st = hum2:GetState()
        if st == Enum.HumanoidStateType.Physics or st == Enum.HumanoidStateType.Ragdoll or st == Enum.HumanoidStateType.FallingDown then
            batCounterDebounce = true
            task.spawn(function()
                local bat = findBatForCounter()
                if bat then swingBatForCounter(bat, char) end
                task.wait(0.5); batCounterDebounce = false
            end)
        end
    end)
end
stopBatCounter = function()
    if Conns.batCounter then Conns.batCounter:Disconnect(); Conns.batCounter = nil end
    batCounterDebounce = false
end

local SKY_TAG = "VynxSkyTheme"
local currentSkyTheme = "Off"
local SKY_PRESETS = {
    ["Off"]           = {kind="off"},
    ["Night"]         = {clock=22,brightness=2,ambient={110,100,130},outAmb={120,110,140},sky={stars=4000,moon=18,sun=0,moonTex=true},atm={dens=0.45,color={120,60,180},decay={60,20,100},glare=0.5,haze=1.2}},
    ["Aurora"]        = {clock=14,brightness=3,ambient={150,120,150},outAmb={160,130,150},atm={dens=0.55,color={255,80,200},decay={255,20,150},glare=2.5,haze=3},clouds={cover=0.7,dens=0.7,color={255,240,250}}},
    ["Sunset"]        = {clock=17.2,brightness=2.5,ambient={170,120,100},outAmb={180,130,110},sky={stars=0,sun=25,moon=0},atm={dens=0.5,color={255,130,60},decay={255,80,30},glare=2,haze=2.5},clouds={cover=0.55,dens=0.55,color={255,200,140}}},
    ["Galaxy"]        = {clock=0,brightness=1.5,ambient={70,60,100},outAmb={80,70,110},sky={stars=10000,moon=30,sun=0},atm={dens=0.15,color={40,20,80},decay={20,10,50},glare=0.3,haze=0.5}},
    ["Cyber"]         = {clock=21,brightness=2.2,ambient={90,130,170},outAmb={100,140,180},sky={stars=2000,moon=12},atm={dens=0.4,color={0,200,255},decay={150,0,255},glare=2,haze=2},clouds={cover=0.4,dens=0.6,color={100,200,255}}},
    ["Sakura"]        = {clock=11,brightness=3.5,ambient={170,150,160},outAmb={180,160,170},sky={sun=8},atm={dens=0.3,color={255,200,220},decay={255,170,200},glare=1,haze=1.5},clouds={cover=0.6,dens=0.4,color={255,250,252}}},
    ["Pink Night"]    = {clock=23,brightness=2.2,ambient={120,60,110},outAmb={140,70,120},sky={stars=5000,moon=22,sun=0,moonTex=true},atm={dens=0.5,color={255,80,180},decay={140,30,100},glare=0.7,haze=1.4},clouds={cover=0.3,dens=0.5,color={180,90,150}}},
    ["Blood Moon"]    = {clock=22.5,brightness=1.6,ambient={130,40,40},outAmb={150,50,50},sky={stars=1500,moon=28,sun=0,moonTex=true},atm={dens=0.6,color={220,30,30},decay={120,10,10},glare=1.4,haze=2},clouds={cover=0.5,dens=0.7,color={120,30,30}}},
    ["Emerald Dawn"]  = {clock=6.5,brightness=2.8,ambient={130,170,140},outAmb={140,180,150},sky={sun=18,moon=0,stars=0},atm={dens=0.4,color={80,200,140},decay={40,150,90},glare=1.8,haze=2.2},clouds={cover=0.5,dens=0.5,color={200,255,220}}},
    ["Volcanic"]      = {clock=19,brightness=2,ambient={180,80,40},outAmb={200,90,50},sky={stars=200,sun=12,moon=0},atm={dens=0.75,color={255,60,0},decay={180,20,0},glare=3,haze=3.5},clouds={cover=0.8,dens=0.9,color={120,40,20}}},
    ["Arctic"]        = {clock=9,brightness=3.2,ambient={200,220,235},outAmb={210,230,245},sky={sun=10,stars=0,moon=0},atm={dens=0.3,color={180,220,255},decay={140,200,240},glare=1.5,haze=1.8},clouds={cover=0.7,dens=0.6,color={250,253,255}}},
    ["Midnight Ocean"]= {clock=1.5,brightness=1.7,ambient={60,90,130},outAmb={70,100,140},sky={stars=6000,moon=24,sun=0,moonTex=true},atm={dens=0.5,color={20,60,140},decay={10,30,90},glare=0.6,haze=1.5}},
    ["Vaporwave"]     = {clock=19.5,brightness=2.4,ambient={180,120,200},outAmb={190,130,210},sky={stars=1000,moon=14},atm={dens=0.45,color={255,100,220},decay={120,60,255},glare=2.2,haze=2.4},clouds={cover=0.5,dens=0.55,color={200,150,255}}},
    ["Toxic"]         = {clock=13,brightness=2.5,ambient={140,180,80},outAmb={150,190,90},atm={dens=0.55,color={100,220,40},decay={60,150,20},glare=1.8,haze=2.6},clouds={cover=0.65,dens=0.7,color={180,255,120}}},
    ["Solar Eclipse"] = {clock=12,brightness=0.9,ambient={50,40,60},outAmb={60,50,70},sky={stars=3500,sun=22,moon=0},atm={dens=0.5,color={255,140,40},decay={30,20,40},glare=2.8,haze=1.8}},
    ["Hellscape"]     = {clock=18,brightness=1.8,ambient={200,60,30},outAmb={220,70,40},sky={stars=100,sun=30,moon=0},atm={dens=0.85,color={255,30,0},decay={120,0,0},glare=3.5,haze=4},clouds={cover=0.95,dens=0.95,color={80,20,10}}},
    ["Heaven"]        = {clock=12,brightness=4,ambient={240,235,210},outAmb={250,245,220},sky={sun=16,moon=0,stars=0},atm={dens=0.25,color={255,250,220},decay={255,240,200},glare=3,haze=1.5},clouds={cover=0.85,dens=0.5,color={255,255,255}}},
    ["Storm"]         = {clock=15,brightness=1.4,ambient={90,90,110},outAmb={100,100,120},sky={stars=0,sun=6,moon=0},atm={dens=0.65,color={80,90,120},decay={40,50,80},glare=0.5,haze=3},clouds={cover=0.95,dens=0.95,color={60,65,80}}},
    ["Sunrise"]       = {clock=6.2,brightness=2.8,ambient={220,180,130},outAmb={230,190,140},sky={sun=22,stars=0,moon=0},atm={dens=0.45,color={255,180,100},decay={255,140,80},glare=2.4,haze=2.2},clouds={cover=0.4,dens=0.4,color={255,220,180}}},
    ["Deep Space"]    = {clock=0,brightness=1,ambient={30,25,50},outAmb={40,35,60},sky={stars=15000,moon=0,sun=0},atm={dens=0.08,color={15,5,40},decay={5,0,20},glare=0.2,haze=0.3}},
    ["Lavender Dream"]= {clock=18.5,brightness=2.6,ambient={180,160,220},outAmb={190,170,230},sky={stars=800,moon=16,sun=0},atm={dens=0.4,color={200,160,255},decay={160,120,220},glare=1.4,haze=1.8},clouds={cover=0.55,dens=0.5,color={220,200,255}}},
    ["Inferno"]       = {clock=17.5,brightness=2.2,ambient={220,100,40},outAmb={235,110,50},sky={sun=26,moon=0,stars=0},atm={dens=0.6,color={255,90,20},decay={200,40,0},glare=3,haze=3.2},clouds={cover=0.7,dens=0.7,color={200,80,40}}},
    ["Mint Sky"]      = {clock=10,brightness=3.2,ambient={180,230,210},outAmb={190,240,220},sky={sun=10},atm={dens=0.32,color={150,255,210},decay={100,220,180},glare=1.6,haze=1.6},clouds={cover=0.55,dens=0.45,color={240,255,250}}},
}
local SKY_ORDER = {"Off","Night","Aurora","Sunset","Galaxy","Cyber","Sakura","Pink Night","Blood Moon","Emerald Dawn","Volcanic","Arctic","Midnight Ocean","Vaporwave","Toxic","Solar Eclipse","Hellscape","Heaven","Storm","Sunrise","Deep Space","Lavender Dream","Inferno","Mint Sky"}

local function _color3(rgb) return Color3.fromRGB(rgb[1], rgb[2], rgb[3]) end

local function applySkyTheme(mode)
    for _, child in ipairs(Lighting:GetChildren()) do
        if child:GetAttribute(SKY_TAG) then pcall(function() child:Destroy() end) end
    end
    local terrain = workspace:FindFirstChildOfClass("Terrain")
    if terrain then
        for _, child in ipairs(terrain:GetChildren()) do
            if child:GetAttribute(SKY_TAG) then pcall(function() child:Destroy() end) end
        end
    end
    local preset = SKY_PRESETS[mode]
    if not preset or preset.kind == "off" then
        Lighting.ClockTime = 14; Lighting.Brightness = 2
        Lighting.OutdoorAmbient = Color3.fromRGB(127,127,127)
        Lighting.Ambient = Color3.fromRGB(127,127,127)
        Lighting.FogEnd = 100000; Lighting.GlobalShadows = true
        currentSkyTheme = "Off"; return
    end
    Lighting.FogStart = 0; Lighting.FogEnd = 100000; Lighting.FogColor = Color3.fromRGB(100,100,100)
    Lighting.ColorShift_Top = Color3.fromRGB(0,0,0); Lighting.ColorShift_Bottom = Color3.fromRGB(0,0,0)
    Lighting.GlobalShadows = true; Lighting.ClockTime = preset.clock or 14; Lighting.Brightness = preset.brightness or 2
    if preset.outAmb then Lighting.OutdoorAmbient = _color3(preset.outAmb) end
    if preset.ambient then Lighting.Ambient = _color3(preset.ambient) end
    if preset.sky then
        local sky = Instance.new("Sky"); sky:SetAttribute(SKY_TAG, true)
        if preset.sky.stars then sky.StarCount = preset.sky.stars end
        if preset.sky.moon then sky.MoonAngularSize = preset.sky.moon end
        if preset.sky.sun then sky.SunAngularSize = preset.sky.sun end
        if preset.sky.moonTex then sky.MoonTextureId = "rbxasset://sky/moon.jpg" end
        sky.Parent = Lighting
    end
    if preset.atm then
        local atm = Instance.new("Atmosphere"); atm:SetAttribute(SKY_TAG, true)
        atm.Density = preset.atm.dens or 0.3; atm.Color = _color3(preset.atm.color)
        atm.Decay = _color3(preset.atm.decay); atm.Glare = preset.atm.glare or 1; atm.Haze = preset.atm.haze or 1
        atm.Parent = Lighting
    end
    if preset.clouds and terrain then
        local clouds = Instance.new("Clouds"); clouds:SetAttribute(SKY_TAG, true)
        clouds.Cover = preset.clouds.cover or 0.5; clouds.Density = preset.clouds.dens or 0.5
        clouds.Color = _color3(preset.clouds.color); clouds.Parent = terrain
    end
    currentSkyTheme = mode
end

local defLightBrightness, defLightClock, defLightAmbient

local function applyAntiLagDerender(obj)
    pcall(function()
        if obj:IsA("Accessory") or obj:IsA("Hat") then obj:Destroy()
        elseif obj:IsA("BasePart") then obj.Material = Enum.Material.Plastic; obj.Reflectance = 0; obj.CastShadow = false
        elseif obj:IsA("Decal") or obj:IsA("Texture") then obj.Transparency = 1
        elseif obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Fire") or obj:IsA("Smoke") or obj:IsA("Sparkles") then obj.Enabled = false
        elseif obj:IsA("AnimationController") or obj:IsA("Animator") then
            for _,t in ipairs(obj:GetPlayingAnimationTracks()) do pcall(function() t:Stop(0) end) end
        end
    end)
end

local function enableAntiLag()
    removeAccessoriesEnabled = true; antiLagEnabled = true
    defLightBrightness = defLightBrightness or Lighting.Brightness
    defLightClock = defLightClock or Lighting.ClockTime
    defLightAmbient = defLightAmbient or Lighting.OutdoorAmbient
    Lighting.GlobalShadows = false; Lighting.FogEnd = 1e10; Lighting.Brightness = 1
    Lighting.EnvironmentDiffuseScale = 0; Lighting.EnvironmentSpecularScale = 0
    for _, e in pairs(Lighting:GetChildren()) do
        pcall(function()
            if e:IsA("BlurEffect") or e:IsA("SunRaysEffect") or e:IsA("ColorCorrectionEffect") or e:IsA("BloomEffect") or e:IsA("DepthOfFieldEffect") then e.Enabled = false end
        end)
    end
    for _, obj in ipairs(workspace:GetDescendants()) do applyAntiLagDerender(obj) end
    if antiLagDescConn then antiLagDescConn:Disconnect() end
    antiLagDescConn = workspace.DescendantAdded:Connect(function(obj)
        if removeAccessoriesEnabled then applyAntiLagDerender(obj) end
    end)
end

local function disableAntiLag()
    removeAccessoriesEnabled = false; antiLagEnabled = false
    if antiLagDescConn then antiLagDescConn:Disconnect(); antiLagDescConn = nil end
    pcall(function()
        if defLightBrightness then Lighting.Brightness = defLightBrightness end
        if defLightClock then Lighting.ClockTime = defLightClock end
        if defLightAmbient then Lighting.OutdoorAmbient = defLightAmbient end
        Lighting.ExposureCompensation = 0
    end)
end

local function enableStretchRez()
    stretchRezEnabled = true; workspace.CurrentCamera.FieldOfView = 107
    if stretchRezConn then stretchRezConn:Disconnect() end
    stretchRezConn = RunService.RenderStepped:Connect(function()
        if not stretchRezEnabled then stretchRezConn:Disconnect(); stretchRezConn = nil; return end
        workspace.CurrentCamera.FieldOfView = 107
    end)
end

local function disableStretchRez()
    stretchRezEnabled = false
    if stretchRezConn then stretchRezConn:Disconnect(); stretchRezConn = nil end
    workspace.CurrentCamera.FieldOfView = 70
end

function saveConfig()
    local function ks(e) return {kb = e.kb and e.kb.Name or nil, gp = e.gp and e.gp.Name or nil} end
    local cfg = {
        normalSpeed = NS, carrySpeed = CS,
        dropBrainrotKey = ks(KB.DropBrainrot),
        tpLockKey = ks(KB.TPLock), laggerToggleKey = ks(KB.LaggerToggle),
        instaResetKey = ks(KB.InstaReset), tpFloorKey = ks(KB.TPFloor), guiHideKey = ks(KB.GuiHide),
        speedToggleKey = ks(KB.SpeedToggle), batV2Key = ks(KB.BatV2Toggle),
        antiRagdoll = antiRagdollEnabled,
        infiniteJump = infJumpEnabled, medusaCounter = medusaCounterEnabled,
        medusaReset = medusaResetEnabled, batCounter = batCounterEnabled,
        carryMode = carrySpeedActive, laggerMode = laggerModeEnabled, laggerCarryMode = (laggerModeEnabled and carrySpeedActive),
        laggerSpeed = LAGGER_SPEED, laggerCarrySpeed = LAGGER_CARRY_SPEED,
        bypassTPEnabled = bypassTPEnabled, autoSwing = autoSwingEnabled,
        unwalkEnabled = unwalkEnabled, antiLag = antiLagEnabled, stretchRez = stretchRezEnabled,
        autoTPEnabled = autoTPEnabled, autoTPHeight = autoTPHeight,
        guiTransparencyEnabled = guiTransparencyEnabled,
        mobileButtonsEnabled = mobileButtonsEnabled, mobileButtonsSize = mobileButtonsSize,
        circleButtonsEnabled = circleButtonsEnabled, shapeButtonsEnabled = shapeButtonsEnabled,
        rectangularButtonsEnabled = rectangularButtonsEnabled,
        infJumpMode = infJumpMode,
        antiDie = antiDieEnabled,
        uiLocked = uiLocked, perButtonDragEnabled = perButtonDragEnabled,
        ,
        ,
        skyTheme = currentSkyTheme, uiScale = uiScaleValue,
        buttonVisibility = buttonVisibility,
    }
    pcall(function() writefile("VynxPC.json", HS:JSONEncode(cfg)) end)
end

task.spawn(function() while task.wait(5) do saveConfig() end end)

local overheadGui = nil
local overheadSpeedLabel = nil

local function createHeadLabels(char)
    if overheadGui then
        pcall(function() overheadGui:Destroy() end)
        overheadGui = nil
        overheadSpeedLabel = nil
    end
    if not char then return end
    local head = char:FindFirstChild("Head") or char:WaitForChild("Head", 5)
    if not head then return end
    
    overheadGui = Instance.new("BillboardGui")
    overheadGui.Name = "yousef_duels_OverheadInfo"
    overheadGui.Size = UDim2.new(0, 250, 0, 100)
    overheadGui.StudsOffset = Vector3.new(0, 3, 0)
    overheadGui.AlwaysOnTop = true
    overheadGui.LightInfluence = 0
    overheadGui.Parent = head
    
    -- Discord Link (Middle)
    local discordLbl = Instance.new("TextLabel")
    discordLbl.Name = "Discord"
    discordLbl.Size = UDim2.new(1, 0, 0, 26)
    discordLbl.Position = UDim2.new(0, 0, 0, 26)
    discordLbl.BackgroundTransparency = 1
    discordLbl.Text = "discord.gg/VMB5ywPY7"
    discordLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    discordLbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    discordLbl.TextStrokeTransparency = 0
    discordLbl.Font = Enum.Font.GothamBold
    discordLbl.TextSize = 14
    discordLbl.TextXAlignment = Enum.TextXAlignment.Center
    discordLbl.ZIndex = 10
    discordLbl.Parent = overheadGui
    
    -- Speed (Bottom)
    overheadSpeedLabel = Instance.new("TextLabel")
    overheadSpeedLabel.Name = "Speed"
    overheadSpeedLabel.Size = UDim2.new(1, 0, 0, 26)
    overheadSpeedLabel.Position = UDim2.new(0, 0, 0, 52)
    overheadSpeedLabel.BackgroundTransparency = 1
    overheadSpeedLabel.Text = "Speed: 0"
    overheadSpeedLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    overheadSpeedLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    overheadSpeedLabel.TextStrokeTransparency = 0
    overheadSpeedLabel.Font = Enum.Font.GothamBold
    overheadSpeedLabel.TextSize = 16
    overheadSpeedLabel.TextXAlignment = Enum.TextXAlignment.Center
    overheadSpeedLabel.ZIndex = 10
    overheadSpeedLabel.Parent = overheadGui
    
    local conn
    conn = RunService.RenderStepped:Connect(function()
        if not char or not char.Parent or not overheadSpeedLabel then conn:Disconnect(); return end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        local spd
        if hrp then
            local vel = hrp.Velocity
            spd = Vector3.new(vel.X, 0, vel.Z).Magnitude
        else
            spd = 0
        end
        if overheadSpeedLabel then overheadSpeedLabel.Text = string.format("Speed: %.0f", spd) end

    end)
end

LP.CharacterAdded:Connect(function(char)
    StopResetSequence()
    resetCooldownGlobal = false
    currentCharacter = nil
    resetSuccessful = false
    stopResetSequence = false
    task.wait(0.5)
    createHeadLabels(char)
end)

if LP.Character then
    task.spawn(function()
        task.wait(0.5)
        createHeadLabels(LP.Character)
    end)
end

-- Ragdoll countdown billboard (voided) â€” blue color
do
    local RAGDOLL_DURATION = 2.5
    local ragdollBillboard = nil
    local ragdollTextLbl = nil
    local ragdollConn = nil

    local function createRagdollBillboard(char)
        if ragdollBillboard then ragdollBillboard:Destroy(); ragdollBillboard = nil end
        local head = char:FindFirstChild("Head"); if not head then return end
        local gui = Instance.new("BillboardGui")
        gui.Name = "RagdollCountdownGui"
        gui.Adornee = head
        gui.Size = UDim2.new(0, 100, 0, 40)
        gui.StudsOffset = Vector3.new(0, 2.5, 0)
        gui.AlwaysOnTop = true
        gui.ResetOnSpawn = false
        gui.Enabled = false
        gui.Parent = head

        local lbl = Instance.new("TextLabel")
        lbl.Size = UDim2.new(1, 0, 1, 0)
        lbl.BackgroundTransparency = 1
        lbl.Text = ""
        lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
        lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        lbl.TextStrokeTransparency = 0
        lbl.Font = Enum.Font.GothamBold
        lbl.TextScaled = true
        lbl.ZIndex = 10
        lbl.Parent = gui

        ragdollBillboard = gui
        ragdollTextLbl = lbl
    end

    local function startRagdollCountdown()
        if ragdollConn then ragdollConn:Disconnect(); ragdollConn = nil end
        if not ragdollBillboard or not ragdollTextLbl then return end
        local t0 = tick()
        ragdollBillboard.Enabled = true
        ragdollConn = RunService.Heartbeat:Connect(function()
            local elapsed = tick() - t0
            local remaining = RAGDOLL_DURATION - elapsed
            if remaining <= 0 then
                ragdollBillboard.Enabled = false
                ragdollTextLbl.Text = ""
                ragdollConn:Disconnect(); ragdollConn = nil
                return
            end
            ragdollTextLbl.Text = string.format("RAGDOLL %.1f", remaining)
        end)
    end

    LP.CharacterAdded:Connect(function(char)
        task.wait(0.3)
        createRagdollBillboard(char)
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum.StateChanged:Connect(function(_, new)
                if new == Enum.HumanoidStateType.Physics or new == Enum.HumanoidStateType.Ragdoll or new == Enum.HumanoidStateType.FallingDown then
                    startRagdollCountdown()
                end
            end)
        end
    end)

    if LP.Character then
        task.spawn(function()
            task.wait(0.3)
            createRagdollBillboard(LP.Character)
            local hum = LP.Character:FindFirstChildOfClass("Humanoid")
            if hum then
                hum.StateChanged:Connect(function(_, new)
                    if new == Enum.HumanoidStateType.Physics or new == Enum.HumanoidStateType.Ragdoll or new == Enum.HumanoidStateType.FallingDown then
                        startRagdollCountdown()
                    end
                end)
            end
        end)
    end
end

local playerSpeedLabels = {}

local function createPlayerSpeedLabel(plr)
    if plr == LP then return end
    local char = plr.Character
    if not char then return end
    local head = char:FindFirstChild("Head")
    if not head then return end

    if playerSpeedLabels[plr] then
        playerSpeedLabels[plr].gui:Destroy()
        playerSpeedLabels[plr] = nil
    end

    local speedGui = Instance.new("BillboardGui")
    speedGui.Name = "VynxSpeedLabel_Other"
    speedGui.Adornee = head
    speedGui.Size = UDim2.new(0, 160, 0, 28)
    speedGui.StudsOffset = Vector3.new(0, 2.8, 0)
    speedGui.AlwaysOnTop = true
    speedGui.Parent = char

    local speedLbl = Instance.new("TextLabel")
    speedLbl.Size = UDim2.new(1,0,1,0)
    speedLbl.BackgroundTransparency = 1
    speedLbl.Text = "Speed: 0"
    speedLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    speedLbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    speedLbl.TextStrokeTransparency = 0
    speedLbl.Font = Enum.Font.GothamBold
    speedLbl.TextSize = 14
    speedLbl.TextScaled = true
    speedLbl.TextXAlignment = Enum.TextXAlignment.Center
    speedLbl.Parent = speedGui

    local conn
    conn = RunService.RenderStepped:Connect(function()
        if not char or not char.Parent then
            conn:Disconnect()
            return
        end
        local hum = char:FindFirstChildOfClass("Humanoid")
        local spd = hum and hum.WalkSpeed or 0
        speedLbl.Text = string.format("Speed: %.0f", spd)
    end)

    playerSpeedLabels[plr] = {gui = speedGui, lbl = speedLbl, conn = conn}
end

Players.PlayerAdded:Connect(function(plr)
    if plr ~= LP then
        plr.CharacterAdded:Connect(function()
            task.wait(0.5)
            createPlayerSpeedLabel(plr)
        end)
    end
end)

Players.PlayerRemoving:Connect(function(plr)
    if playerSpeedLabels[plr] then
        playerSpeedLabels[plr].gui:Destroy()
        if playerSpeedLabels[plr].conn then playerSpeedLabels[plr].conn:Disconnect() end
        playerSpeedLabels[plr] = nil
    end
end)

local _cachedChar, _cachedHum, _cachedHrp = nil, nil, nil

LP.CharacterAdded:Connect(function(char)
    _cachedChar = nil; _cachedHum = nil; _cachedHrp = nil
    task.wait(0.3)
    _cachedChar = char
    _cachedHum = char:FindFirstChildOfClass("Humanoid")
    _cachedHrp = char:FindFirstChild("HumanoidRootPart")
    if medusaCounterEnabled or medusaResetEnabled then setupMedusa(char) end
    if batCounterEnabled then startBatCounter() end
    if unwalkEnabled then task.wait(0.5); startUnwalk() end
    if antiRagdollEnabled then stopAntiRagdoll(); startAntiRagdoll() end
    startAntiDie()
end)

UIS.InputBegan:Connect(function(input, gpe)
    if _anyKeyListening then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if gpe or UIS:GetFocusedTextBox() then return end
    end
    if not isGamepadInput(input) then
        if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    end
    if not isBindableInput(input) then return end
    local kc = input.KeyCode
    if kbMatch(KB.LaggerToggle, kc) then toggleLaggerMode(); saveConfig()
    elseif kbMatch(KB.SpeedToggle, kc) then toggleCarryMode(); saveConfig()
    elseif kbMatch(KB.DropBrainrot, kc) then runDrop()
    elseif kbMatch(KB.InstaReset, kc) then
        if tick() - _lastKbSet > 0.5 then pcall(ResetPlayer) end
    elseif kbMatch(KB.TPFloor, kc) then runTPFloor()
    elseif kbMatch(KB.TPLock, kc) then
        toggleBypassTP(); saveConfig()
    elseif kbMatch(KB.GuiHide, kc) then
        if mainFrame then mainFrame.Visible = not mainFrame.Visible end
    
    end
end)

local MOB_POS_FILE = "vynx_btnpos.json"
local DRAG_THRESHOLD = 15

local function loadBtnPositions()
    if not (isfile and isfile(MOB_POS_FILE)) then return {} end
    local ok, data = pcall(function() return HS:JSONDecode(readfile(MOB_POS_FILE)) end)
    if ok and type(data) == "table" then return data end
    return {}
end

local function saveBtnPositions()
    if not writefile then return end; if not mobGuiRef then return end
    local out = {}
    for _, child in ipairs(mobGuiRef:GetDescendants()) do
        if child:IsA("Frame") and child:GetAttribute("BtnKey") then
            local key = child:GetAttribute("BtnKey")
            out[key] = {xo = child.Position.X.Offset, yo = child.Position.Y.Offset}
        end
    end
    pcall(function() writefile(MOB_POS_FILE, HS:JSONEncode(out)) end)
end

local function destroyMobileButtons()
    if mobGuiRef then pcall(function() mobGuiRef:Destroy() end); mobGuiRef = nil end
    mobBtnRefs = {}
end

local function resetBtnPositions()
    pcall(function() if writefile then writefile(MOB_POS_FILE, "{}") end end)
    if mobileButtonsEnabled then buildMobileButtons() end
end

local function buildMobileButtons()
    destroyMobileButtons()
    if not mobileButtonsEnabled then return end
    local savedPositions = loadBtnPositions()
    local BTN_SIZE = math.floor(mobileButtonsSize * 0.65)
    local fontSize = math.max(9, math.floor(mobileButtonsSize * 0.13))
    local spacing = 20; local cols = 2
    local isRect = rectangularButtonsEnabled
    local btnW = BTN_SIZE; local btnH = BTN_SIZE
    if isRect then btnW = math.floor(BTN_SIZE*1.4); btnH = math.floor(BTN_SIZE*0.75); spacing = 16 end
    local viewport = workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize or Vector2.new(800,600)
    local totalW = cols * btnW + (cols-1) * spacing
    local startX = viewport.X - totalW - 30; local startY = 30
    local buttonOrder = {
        {key="drop",         label="DROP BR",         toggle=false},
        {key="tpDown",       label="TP DOWN",         toggle=false},
        
        {key="tpLock",       label="TP BAT",          toggle=true},
        {key="lagger",       label="LAGGER",          toggle=true},
        {key="laggerCarry",  label="LAGGER CARRY",    toggle=true},
        {key="carrySpeed",   label="CARRY SPD",       toggle=true},
        {key="instaReset",   label="INSTA RESET",     toggle=false},
        {key="animChanger",  label="ANIM CHANGER",    toggle=true},
    }
    local mobGui = Instance.new("ScreenGui")
    mobGui.Name = "VynxMobileButtons"; mobGui.ResetOnSpawn = false
    mobGui.DisplayOrder = 15; mobGui.IgnoreGuiInset = true
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(mobGui) end end)
    if not pcall(function() mobGui.Parent = game:GetService("CoreGui") end) then
        mobGui.Parent = LP:WaitForChild("PlayerGui")
    end
    mobGuiRef = mobGui

    -- only render buttons that are visible
    local visibleOrder = {}
    for _, def in ipairs(buttonOrder) do
        if buttonVisibility[def.key] ~= false then
            visibleOrder[#visibleOrder+1] = def
        end
    end
    for i, def in ipairs(visibleOrder) do
        local col = (i-1) % cols; local row = math.floor((i-1)/cols)
        local xPos = startX + col*(btnW+spacing); local yPos = startY + row*(btnH+spacing)
        local saved = savedPositions[def.key]
        local initX = saved and saved.xo or xPos; local initY = saved and saved.yo or yPos
        local frame = Instance.new("Frame", mobGui)
        frame.Name = "MobBtn_"..def.key
        frame.Size = UDim2.new(0,btnW,0,btnH)
        frame.Position = UDim2.new(0,initX,0,initY)
        frame.BackgroundColor3 = BTN_OFF; frame.BackgroundTransparency = 0
        frame.BorderSizePixel = 0; frame.Active = true; frame.ZIndex = 102
        frame:SetAttribute("BtnKey", def.key)
        local cornerRadius
        if circleButtonsEnabled then cornerRadius = UDim.new(1,0)
        elseif shapeButtonsEnabled then cornerRadius = UDim.new(0,4)
        elseif rectangularButtonsEnabled then cornerRadius = UDim.new(0,9)
        else cornerRadius = UDim.new(0, math.max(8, math.floor(math.min(btnW,btnH)*0.2))) end
        local uic = Instance.new("UICorner", frame); uic.CornerRadius = cornerRadius

        local fstroke = Instance.new("UIStroke", frame)
        fstroke.Color = Color3.fromRGB(80,80,80); fstroke.Thickness = 1.5
        fstroke.Transparency = 1

        local btn = Instance.new("TextButton", frame)
        btn.Size = UDim2.new(1,0,1,0); btn.BackgroundTransparency = 1
        btn.Text = def.label; btn.TextColor3 = TEXT_OFF
        btn.Font = Enum.Font.GothamBold; btn.TextSize = fontSize
        btn.LineHeight = 1.1; btn.TextWrapped = true
        btn.AutoButtonColor = false; btn.ZIndex = 105
        local isOn = false
        local function setOn(v)
            isOn = v
            if v then
                frame:SetAttribute("BtnIsOn", true)
                TS:Create(frame, TweenInfo.new(0.12,Enum.EasingStyle.Quad), {BackgroundColor3 = BTN_ON}):Play()
                TS:Create(fstroke, TweenInfo.new(0.12), {Color=Color3.fromRGB(80,80,80), Thickness=2.5, Transparency=0}):Play()
                TS:Create(btn, TweenInfo.new(0.12), {TextColor3 = TEXT_ON}):Play()
            else
                frame:SetAttribute("BtnIsOn", false)
                TS:Create(frame, TweenInfo.new(0.12,Enum.EasingStyle.Quad), {BackgroundColor3 = BTN_OFF}):Play()
                TS:Create(fstroke, TweenInfo.new(0.12), {Color=Color3.fromRGB(80,80,80), Thickness=1.5, Transparency=1}):Play()
                TS:Create(btn, TweenInfo.new(0.12), {TextColor3 = TEXT_OFF}):Play()
            end
        end
        mobBtnRefs[def.key] = setOn
        local dragData = {start=nil,startPos=nil,moved=false,down=false}
        btn.InputBegan:Connect(function(input)
            if uiLocked or not perButtonDragEnabled then return end
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragData.moved=false; dragData.down=true
                dragData.start=input.Position; dragData.startPos=frame.Position
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragData.down=false
                        if dragData.moved then pcall(saveBtnPositions) end
                    end
                end)
            end
        end)
        btn.InputChanged:Connect(function(input)
            if dragData.down and not uiLocked and perButtonDragEnabled and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local delta = input.Position - dragData.start
                if delta.Magnitude > DRAG_THRESHOLD then dragData.moved = true end
                if dragData.moved then
                    frame.Position = UDim2.new(dragData.startPos.X.Scale, dragData.startPos.X.Offset+delta.X, dragData.startPos.Y.Scale, dragData.startPos.Y.Offset+delta.Y)
                end
            end
        end)
        local function flashWhite()
            TS:Create(frame, TweenInfo.new(0.06), {BackgroundColor3=Color3.fromRGB(120, 120, 120)}):Play()
            TS:Create(btn, TweenInfo.new(0.06), {TextColor3=Color3.fromRGB(255,255,255)}):Play()
            task.delay(0.15, function()
                if isOn then
                    TS:Create(frame, TweenInfo.new(0.12), {BackgroundColor3=BTN_ON}):Play()
                    TS:Create(btn, TweenInfo.new(0.12), {TextColor3=TEXT_ON}):Play()
                else
                    TS:Create(frame, TweenInfo.new(0.12), {BackgroundColor3=BTN_OFF}):Play()
                    TS:Create(btn, TweenInfo.new(0.12), {TextColor3=TEXT_OFF}):Play()
                end
            end)
        end
        btn.MouseButton1Click:Connect(function()
            if dragData.moved then dragData.moved=false; return end
            flashWhite()
            if def.key == "tpLock" then
                toggleBypassTP(); setOn(bypassTPEnabled); saveConfig(); return
            end
            
            if def.key == "carrySpeed" then
                toggleCarryMode(); setOn(carrySpeedActive)
                if mobBtnRefs.lagger then mobBtnRefs.lagger(laggerModeEnabled) end
                if mobBtnRefs.laggerCarry then mobBtnRefs.laggerCarry(laggerModeEnabled and carrySpeedActive) end
                saveConfig(); return
            end
            if def.key == "lagger" then
                toggleLaggerMode(); setOn(laggerModeEnabled)
                if mobBtnRefs.carrySpeed then mobBtnRefs.carrySpeed(carrySpeedActive) end
                if mobBtnRefs.laggerCarry then mobBtnRefs.laggerCarry(laggerModeEnabled and carrySpeedActive) end
                saveConfig(); return
            end
            if def.key == "laggerCarry" then
                toggleLaggerCarryMode(); setOn(laggerModeEnabled and carrySpeedActive)
                if mobBtnRefs.carrySpeed then mobBtnRefs.carrySpeed(carrySpeedActive) end
                if mobBtnRefs.lagger then mobBtnRefs.lagger(laggerModeEnabled) end
                saveConfig(); return
            end
            if def.key == "animChanger" then
                animChangerEnabled = not animChangerEnabled
                if animChangerGui then animChangerGui.Enabled = animChangerEnabled end
                setOn(animChangerEnabled)
                if animChangerSetVisual then animChangerSetVisual(animChangerEnabled) end
                saveConfig(); return
            end
            if def.key == "drop" then runDrop(); return end
            if def.key == "tpDown" then runTPFloor(); return end
            if def.key == "instaReset" then ResetPlayer(); return end
        end)
    end
    if mobBtnRefs.tpLock then mobBtnRefs.tpLock(bypassTPEnabled) end
    if mobBtnRefs.carrySpeed then mobBtnRefs.carrySpeed(carrySpeedActive) end
    if mobBtnRefs.lagger then mobBtnRefs.lagger(laggerModeEnabled) end
    if mobBtnRefs.laggerCarry then mobBtnRefs.laggerCarry(laggerCarryToggled) end
    if mobBtnRefs.animChanger then mobBtnRefs.animChanger(animChangerEnabled) end
end

local function loadConfig()
    if not (isfile and isfile("VynxPC.json")) then return end
    local ok, cfg = pcall(function() return HS:JSONDecode(readfile("VynxPC.json")) end)
    if not ok or type(cfg) ~= "table" then return end
    if cfg.normalSpeed then NS = cfg.normalSpeed end
    if cfg.carrySpeed then CS = cfg.carrySpeed end
    if cfg.laggerSpeed then LAGGER_SPEED = cfg.laggerSpeed end
    if cfg.laggerCarrySpeed then LAGGER_CARRY_SPEED = cfg.laggerCarrySpeed end
    if cfg.autoTPHeight then autoTPHeight = cfg.autoTPHeight end
    if cfg.mobileButtonsSize then mobileButtonsSize = cfg.mobileButtonsSize end
    if cfg.antiRagdoll ~= nil then antiRagdollEnabled = cfg.antiRagdoll end
    if cfg.infiniteJump ~= nil then infJumpEnabled = cfg.infiniteJump end
    if cfg.medusaCounter ~= nil then medusaCounterEnabled = cfg.medusaCounter end
    if cfg.medusaReset ~= nil then medusaResetEnabled = cfg.medusaReset end
    if cfg.batCounter ~= nil then batCounterEnabled = cfg.batCounter end
    if cfg.carryMode ~= nil then carrySpeedActive = cfg.carryMode; speedMode = cfg.carryMode end
    if cfg.laggerMode ~= nil then laggerModeEnabled = cfg.laggerMode end
    if cfg.laggerCarryMode ~= nil then laggerCarryToggled = cfg.laggerCarryMode end
    if cfg.bypassTPEnabled ~= nil then bypassTPEnabled = cfg.bypassTPEnabled end
    if cfg.autoSwing ~= nil then autoSwingEnabled = cfg.autoSwing end
    if cfg.unwalkEnabled ~= nil then unwalkEnabled = cfg.unwalkEnabled end
    if cfg.antiLag ~= nil then antiLagEnabled = cfg.antiLag end
    if cfg.stretchRez ~= nil then stretchRezEnabled = cfg.stretchRez end
    if cfg.autoTPEnabled ~= nil then autoTPEnabled = cfg.autoTPEnabled end
    if cfg.guiTransparencyEnabled ~= nil then guiTransparencyEnabled = cfg.guiTransparencyEnabled end
    if cfg.mobileButtonsEnabled ~= nil then mobileButtonsEnabled = cfg.mobileButtonsEnabled end
    if cfg.circleButtonsEnabled ~= nil then circleButtonsEnabled = cfg.circleButtonsEnabled end
    if cfg.shapeButtonsEnabled ~= nil then shapeButtonsEnabled = cfg.shapeButtonsEnabled end
    if cfg.rectangularButtonsEnabled ~= nil then rectangularButtonsEnabled = cfg.rectangularButtonsEnabled end
    if cfg.antiDie ~= nil then antiDieEnabled = cfg.antiDie end
    if cfg.infJumpMode ~= nil then infJumpMode = cfg.infJumpMode end
    if cfg.uiLocked ~= nil then uiLocked = cfg.uiLocked end
    if cfg.perButtonDragEnabled ~= nil then perButtonDragEnabled = cfg.perButtonDragEnabled end
    if 
    if 
    if cfg.skyTheme ~= nil then currentSkyTheme = cfg.skyTheme end
    if cfg.uiScale ~= nil then uiScaleValue = cfg.uiScale end
    if type(cfg.buttonVisibility) == "table" then
        for k, v in pairs(cfg.buttonVisibility) do
            if buttonVisibility[k] ~= nil then buttonVisibility[k] = v end
        end
    end
    local function applyKey(target, saved)
        if not saved or not target then return end
        if saved.kb and Enum.KeyCode[saved.kb] then target.kb = Enum.KeyCode[saved.kb] else target.kb = nil end
        if saved.gp and Enum.KeyCode[saved.gp] then target.gp = Enum.KeyCode[saved.gp] else target.gp = nil end
    end
    applyKey(KB.DropBrainrot, cfg.dropBrainrotKey); applyKey(KB.BoxAutoPlay, cfg.boxAutoPlayKey)
    applyKey(KB.TPLock, cfg.tpLockKey)
    applyKey(KB.LaggerToggle, cfg.laggerToggleKey);
    applyKey(KB.InstaReset, cfg.instaResetKey); applyKey(KB.TPFloor, cfg.tpFloorKey)
    applyKey(KB.GuiHide, cfg.guiHideKey); applyKey(KB.SpeedToggle, cfg.speedToggleKey)
    applyKey(KB.BatV2Toggle, cfg.batV2Key)
end

local function applyState()
    if antiRagdollEnabled then stopAntiRagdoll(); startAntiRagdoll() end
    if setAntiRagVisual then setAntiRagVisual(antiRagdollEnabled) end
    if setInfJumpVisual then setInfJumpVisual(infJumpEnabled) end
    if infJumpEnabled and infJumpMode == "hold" then startHoldInfJump() end
    if medusaCounterEnabled or medusaResetEnabled then
        setupMedusa(LP.Character)
        if setMedusaVisual then setMedusaVisual(medusaCounterEnabled) end
        if setMedusaResetVisual then setMedusaResetVisual(medusaResetEnabled) end
    end
    if batCounterEnabled then startBatCounter() end
    if setBatCounterVisual then setBatCounterVisual(batCounterEnabled) end
    if bypassTPEnabled then startBypassTP() end
    if unwalkEnabled then startUnwalk() end
    if setUnwalkVisual then setUnwalkVisual(unwalkEnabled) end
    if antiLagEnabled then enableAntiLag() end
    if setAntiLagVisual then setAntiLagVisual(antiLagEnabled) end
    if stretchRezEnabled then enableStretchRez() end
    if setStretchRezVisual then setStretchRezVisual(stretchRezEnabled) end
    if autoTPEnabled then startAutoTP() end
    if setAutoTPVisual then setAutoTPVisual(autoTPEnabled) end
    
    if antiDieEnabled then startAntiDie() end
    if setAntiDieVisual then setAntiDieVisual(antiDieEnabled) end
    refreshSpeedModeLabel()
    if currentSkyTheme and currentSkyTheme ~= "Off" then applySkyTheme(currentSkyTheme) end
    if mainFrame and uiScaleObject then uiScaleObject.Scale = uiScaleValue end
end

if _G.VYNX_LOADED then return end
_G.VYNX_LOADED = true

local bgImageRef = nil
local backgroundEnabled = false
local backgroundIndex = 0

local BG_IMAGES = {
    [1] = "82570501613757",
    [2] = "89455917077259",
    [3] = "140011519343966",
    [4] = "122541342511357",
    [5] = "91186886252449",
    [6] = "121087678749100",
    [7] = "113351045442552",
    [8] = "123175449101989",
    [9] = "113133243302321"
}

local function applyBackgroundImage(index)
    backgroundIndex = index or 0
    if not bgImageRef then return end
    if backgroundIndex == 0 then
        bgImageRef.Visible = false
        backgroundEnabled = false
    else
        local imgId = BG_IMAGES[backgroundIndex]
        if imgId then
            bgImageRef.Image = "rbxassetid://" .. imgId
            bgImageRef.Visible = true
            backgroundEnabled = true
        end
    end
end

local function buildGui()
    local WHITE    = Color3.fromRGB(255, 255, 255)
    local BLACK    = Color3.fromRGB(0,   0,   0)
    local DARK     = Color3.fromRGB(10,  10,  14)
    local DARKER   = Color3.fromRGB(15,  10,  18)
    local MID      = Color3.fromRGB(40,  40,  40)
    local DIM_LINE = Color3.fromRGB(40, 40, 40)
    local CHECK_OFF= Color3.fromRGB(0, 40, 110)
    local BTN_ACT  = Color3.fromRGB(0, 0, 0)
    local INP_BG   = Color3.fromRGB(20,  20,  26)

    for _, name in ipairs({"Asta", "YousefMobileButtons", "yousefStealBar", "yousefRagdollTimer"}) do
        local old = game:GetService("CoreGui"):FindFirstChild(name)
        if old then old:Destroy() end
        local pg = LP:FindFirstChild("PlayerGui")
        if pg then local o = pg:FindFirstChild(name); if o then o:Destroy() end end
    end

    local gui = Instance.new("ScreenGui")
    gui.Name = "Asta"; gui.ResetOnSpawn = false
    gui.DisplayOrder = 10; gui.IgnoreGuiInset = true
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
    if not pcall(function() gui.Parent = game:GetService("CoreGui") end) then
        gui.Parent = LP:WaitForChild("PlayerGui")
    end

    uiScaleObject = Instance.new("UIScale", gui)
    uiScaleObject.Scale = uiScaleValue

    mainFrame = Instance.new("Frame", gui)
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 500, 0, 550)
    mainFrame.Position = UDim2.new(0.02, 0, 0.45, -275)
    mainFrame.BackgroundColor3 = DARK
    mainFrame.BorderSizePixel = 0
    mainFrame.ClipsDescendants = true
    Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 24)

    bgImage = Instance.new("ImageLabel", mainFrame)
    bgImage.Size = UDim2.new(1,0,1,0); bgImage.Position = UDim2.new(0,0,0,0)
    bgImage.BackgroundTransparency = 1
    bgImage.Image = ""
    bgImage.ScaleType = Enum.ScaleType.Crop; bgImage.ZIndex = 1
    bgImage.ImageTransparency = 0.45
    Instance.new("UICorner", bgImage).CornerRadius = UDim.new(0,24)

    -- Background usando rbxassetid (sistema do Asta Remake)
    bgImageRef = bgImage
    applyBackgroundImage(1)  -- Aplica a primeira imagem de fundo por padrÃ£o

    local bgOverlay = Instance.new("Frame", mainFrame)
    bgOverlay.Size = UDim2.new(1,0,1,0); bgOverlay.BackgroundColor3 = DARK
    bgOverlay.BackgroundTransparency = 1
    bgOverlay.BorderSizePixel = 0; bgOverlay.ZIndex = 2
    Instance.new("UICorner", bgOverlay).CornerRadius = UDim.new(0,24)

    local dragging, dragInput2, dragStart2, startPos2
    mainFrame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart2 = input.Position
            startPos2 = mainFrame.Position
            dragInput2 = input
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                    dragInput2 = nil
                end
            end)
        end
    end)
    mainFrame.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            local delta = input.Position - dragStart2
            if delta.Magnitude > 5 then
                mainFrame.Position = UDim2.new(
                    startPos2.X.Scale,
                    startPos2.X.Offset + delta.X,
                    startPos2.Y.Scale,
                    startPos2.Y.Offset + delta.Y
                )
            end
        end
    end)

    local Header = Instance.new("Frame", mainFrame)
    Header.Name = "Header"; Header.Size = UDim2.new(1,0,0,65)
    Header.BackgroundTransparency = 1; Header.ZIndex = 3

    local Title = Instance.new("TextLabel", Header)
    Title.Size = UDim2.new(0,180,1,0); Title.Position = UDim2.new(0,14,0,0)
    Title.BackgroundTransparency = 1; Title.Text = "Asta"
    Title.TextColor3 = WHITE  -- mantido branco; Title.Font = Enum.Font.GothamBlack
    Title.TextSize = 20; Title.TextXAlignment = Enum.TextXAlignment.Left; Title.ZIndex = 4

    local HeaderLine = Instance.new("Frame", Header)
    HeaderLine.Size = UDim2.new(1,-28,0,1); HeaderLine.Position = UDim2.new(0,14,1,0)
    HeaderLine.BackgroundColor3 = Color3.fromRGB(60, 60, 60); HeaderLine.BackgroundTransparency = 0.5
    HeaderLine.BorderSizePixel = 0; HeaderLine.ZIndex = 3

    local closeBtn = Instance.new("TextButton", Header)
    closeBtn.Size = UDim2.new(0,24,0,24); closeBtn.Position = UDim2.new(1,-34,0.5,-12)
    closeBtn.BackgroundColor3 = MID; closeBtn.BorderSizePixel = 0
    closeBtn.Text = "-"; closeBtn.TextColor3 = WHITE
    closeBtn.Font = Enum.Font.GothamBold; closeBtn.TextSize = 18; closeBtn.ZIndex = 6
    Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0,5)
    local closeBtnStroke = Instance.new("UIStroke", closeBtn)
    closeBtnStroke.Color = WHITE; closeBtnStroke.Thickness = 1

    local miniToggleBtn = Instance.new("TextButton", gui)
    miniToggleBtn.Name = "MiniToggle"
    miniToggleBtn.Size = UDim2.new(0, 240, 0, 52)
    miniToggleBtn.Position = UDim2.new(0.02, 0, 0.45, 60)
    miniToggleBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    miniToggleBtn.Text = "Asta"
    miniToggleBtn.TextColor3 = WHITE
    miniToggleBtn.Font = Enum.Font.GothamBlack
    miniToggleBtn.TextSize = 25
    miniToggleBtn.BorderSizePixel = 0
    miniToggleBtn.ZIndex = 100
    miniToggleBtn.Visible = false
    Instance.new("UICorner", miniToggleBtn).CornerRadius = UDim.new(0, 16)
    local miniStroke = Instance.new("UIStroke", miniToggleBtn)
    miniStroke.Color = Color3.fromRGB(80, 80, 80); miniStroke.Thickness = 2.5

    local miniShadow = Instance.new("UIStroke", miniToggleBtn)
    miniShadow.Color = Color3.fromRGB(30, 30, 30); miniShadow.Thickness = 4
    miniShadow.Transparency = 0.7

    miniToggleBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = true
        miniToggleBtn.Visible = false
    end)

    local miniDrag, miniDragStart, miniStartPos
    miniToggleBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            miniDrag = true
            miniDragStart = input.Position
            miniStartPos = miniToggleBtn.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then miniDrag = false end
            end)
        end
    end)
    UIS.InputChanged:Connect(function(input)
        if miniDrag and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - miniDragStart
            miniToggleBtn.Position = UDim2.new(miniStartPos.X.Scale, miniStartPos.X.Offset + delta.X, miniStartPos.Y.Scale, miniStartPos.Y.Offset + delta.Y)
        end
    end)

    closeBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = false
        miniToggleBtn.Visible = true
    end)

    local Sidebar = Instance.new("Frame", mainFrame)
    Sidebar.Name = "Sidebar"; Sidebar.Size = UDim2.new(0,150,1,-68)
    Sidebar.Position = UDim2.new(0,0,0,67); Sidebar.BackgroundTransparency = 1; Sidebar.ZIndex = 3

    local SidebarLine = Instance.new("Frame", Sidebar)
    SidebarLine.Size = UDim2.new(0,1,1,0); SidebarLine.Position = UDim2.new(1,0,0,0)
    SidebarLine.BackgroundColor3 = Color3.fromRGB(60, 60, 60); SidebarLine.BorderSizePixel = 0; SidebarLine.ZIndex = 3

    local TabContainer = Instance.new("Frame", Sidebar)
    TabContainer.Size = UDim2.new(1,-20,1,-110); TabContainer.Position = UDim2.new(0,10,0,100)
    TabContainer.BackgroundTransparency = 1; TabContainer.ZIndex = 3
    local TabListLayout = Instance.new("UIListLayout", TabContainer)
    TabListLayout.Padding = UDim.new(0,6); TabListLayout.SortOrder = Enum.SortOrder.LayoutOrder

    local ContentContainer = Instance.new("Frame", mainFrame)
    ContentContainer.Size = UDim2.new(1,-165,1,-84); ContentContainer.Position = UDim2.new(0,155,0,74)
    ContentContainer.BackgroundTransparency = 1; ContentContainer.ZIndex = 3

    local currentTab = nil
    local TabRefs = {contents = {}, btns = {}, underlines = {}, active = "Speed"}
    local TabList = {"Speed","Combat","Mechns","Visual","Extra","Mobile"}

    for i, name in ipairs(TabList) do
        local wrapper = Instance.new("Frame", TabContainer)
        wrapper.Size = UDim2.new(1,0,0,30)
        wrapper.BackgroundTransparency = 1
        wrapper.LayoutOrder = i

        local btn = Instance.new("TextButton", wrapper)
        btn.Size = UDim2.new(1,0,0,30)
        btn.BackgroundTransparency = (i == 1) and 0.6 or 1
        btn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        btn.Text = name
        btn.TextColor3 = WHITE
        btn.TextTransparency = (i == 1) and 0 or 0.5
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 11
        btn.TextXAlignment = Enum.TextXAlignment.Left
        btn.ZIndex = 5
        local bp = Instance.new("UIPadding", btn)
        bp.PaddingLeft = UDim.new(0,10)
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0,6)

        TabRefs.btns[name] = btn

        local page = Instance.new("ScrollingFrame", ContentContainer)
        page.Size = UDim2.new(1,0,1,0)
        page.BackgroundTransparency = 1
        page.CanvasSize = UDim2.new(0,0,0,0)
        page.AutomaticCanvasSize = Enum.AutomaticSize.Y
        page.ScrollBarThickness = 2
        page.ScrollBarImageColor3 = WHITE
        page.Visible = (i == 1)
        page.ZIndex = 3
        local pageLayout = Instance.new("UIListLayout", page)
        pageLayout.Padding = UDim.new(0,10)
        pageLayout.SortOrder = Enum.SortOrder.LayoutOrder
        local pagePad = Instance.new("UIPadding", page)
        pagePad.PaddingLeft = UDim.new(0,4)
        pagePad.PaddingRight = UDim.new(0,8)
        pagePad.PaddingTop = UDim.new(0,6)
        pagePad.PaddingBottom = UDim.new(0,6)

        TabRefs.contents[name] = page

        if i == 1 then
            currentTab = {Btn = btn, Pg = page}
        end

        btn.MouseButton1Click:Connect(function()
            if currentTab then
                currentTab.Btn.BackgroundTransparency = 1
                currentTab.Btn.TextTransparency = 0.5
                currentTab.Pg.Visible = false
            end
            btn.BackgroundTransparency = 0.6
            btn.TextTransparency = 0
            page.Visible = true
            currentTab = {Btn = btn, Pg = page}
            TabRefs.active = name
        end)
    end

    local function mkSect(parent, txt)
        local f = Instance.new("Frame", parent)
        f.Size = UDim2.new(1,0,0,16); f.BackgroundTransparency = 1
        f.LayoutOrder = #parent:GetChildren() + 1
        local l = Instance.new("TextLabel", f)
        l.Size = UDim2.new(1,0,1,0); l.BackgroundTransparency = 1
        l.Text = txt:upper(); l.TextColor3 = WHITE
        l.Font = Enum.Font.GothamBlack; l.TextSize = 9
        l.TextXAlignment = Enum.TextXAlignment.Left
        local line = Instance.new("Frame", f)
        line.Size = UDim2.new(1,0,0,1); line.Position = UDim2.new(0,0,1,-1)
        line.BackgroundColor3 = WHITE; line.BackgroundTransparency = 0.75; line.BorderSizePixel = 0
    end

    local function mkRow(parent)
        local f = Instance.new("Frame", parent)
        f.Size = UDim2.new(1,0,0,32); f.BackgroundTransparency = 1
        f.LayoutOrder = #parent:GetChildren() + 1
        return f
    end

    local function mkLabel(row, txt)
        local l = Instance.new("TextLabel", row)
        l.Size = UDim2.new(0.6,0,1,0); l.Position = UDim2.new(0,0,0,0)
        l.BackgroundTransparency = 1; l.Text = txt
        l.TextColor3 = WHITE; l.Font = Enum.Font.GothamBold
        l.TextSize = 12; l.TextXAlignment = Enum.TextXAlignment.Left
    end

    local function mkToggle(parent, txt, cb, initVal)
        local row = mkRow(parent)
        mkLabel(row, txt)
        local chk = Instance.new("TextButton", row)
        chk.Size = UDim2.new(0,18,0,18); chk.Position = UDim2.new(1,-24,0.5,-9)
        chk.BackgroundColor3 = initVal and TOGGLE_ON_COLOR or CHECK_OFF
        chk.BorderSizePixel = 0; chk.Text = ""; chk.ZIndex = 5
        Instance.new("UICorner", chk).CornerRadius = UDim.new(0,4)
        local chkStroke = Instance.new("UIStroke", chk)
        chkStroke.Color = WHITE; chkStroke.Thickness = 1
        local state = initVal == true
        local function sv(s)
            state = s
            TS:Create(chk, TweenInfo.new(0.15), {BackgroundColor3 = s and TOGGLE_ON_COLOR or CHECK_OFF}):Play()
        end
        sv(state)
        chk.MouseButton1Click:Connect(function()
            if _anyKeyListening then return end
            state = not state; sv(state); cb(state)
        end)
        return sv
    end

    local function mkBoxRow(parent, txt, default, cb)
        local row = mkRow(parent)
        mkLabel(row, txt)
        local tb = Instance.new("TextBox", row)
        tb.Size = UDim2.new(0,60,0,22); tb.Position = UDim2.new(1,-65,0.5,-11)
        tb.BackgroundColor3 = INP_BG; tb.Text = tostring(default)
        tb.TextColor3 = WHITE; tb.Font = Enum.Font.GothamBold
        tb.TextSize = 11; tb.ClearTextOnFocus = false; tb.ZIndex = 5
        Instance.new("UICorner", tb).CornerRadius = UDim.new(0,6)
        local bs = Instance.new("UIStroke", tb); bs.Color = DIM_LINE; bs.Thickness = 1
        tb.Focused:Connect(function() TS:Create(bs, TweenInfo.new(0.12), {Color=WHITE}):Play() end)
        tb.FocusLost:Connect(function()
            TS:Create(bs, TweenInfo.new(0.12), {Color=DIM_LINE}):Play()
            if cb then local n = tonumber(tb.Text); if n then cb(n) else tb.Text = tostring(default) end end
        end)
        return tb
    end

    local function mkKB(parent, kbEntry, cb)
        local btn = Instance.new("TextButton", parent)
        btn.Size = UDim2.new(0,40,0,18); btn.Position = UDim2.new(1,-110,0.5,-9)
        btn.BackgroundColor3 = INP_BG; btn.BorderSizePixel = 0
        local function getLabel()
            return (kbEntry.gp and getKeyDisplayName(kbEntry.gp, true)) or (kbEntry.kb and kbEntry.kb.Name) or "None"
        end
        btn.Text = getLabel(); btn.TextColor3 = WHITE
        btn.Font = Enum.Font.GothamBold; btn.TextSize = 8; btn.ZIndex = 5
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0,4)
        local bs = Instance.new("UIStroke", btn); bs.Color = DIM_LINE; bs.Thickness = 1
        local li, lc, pv, listenStart = false, nil, btn.Text, 0
        btn.MouseButton1Click:Connect(function()
            if li then
                li = false; _anyKeyListening = false
                if lc then lc:Disconnect(); lc = nil end; btn.Text = pv; return
            end
            pv = btn.Text; li = true; _anyKeyListening = true; listenStart = tick(); btn.Text = "..."
            lc = UIS.InputBegan:Connect(function(inp)
                if not li then return end
                if inp.KeyCode == Enum.KeyCode.Escape then
                    li = false; _anyKeyListening = false
                    if lc then lc:Disconnect(); lc = nil end; btn.Text = pv; return
                end
                local isGp = isGamepadInput(inp)
                if isGp and tick()-listenStart < 0.15 then return end
                if not isBindableInput(inp) then return end
                btn.Text = getKeyDisplayName(inp.KeyCode, isGp)
                pv = btn.Text; li = false; _anyKeyListening = false
                if lc then lc:Disconnect(); lc = nil end
                if cb then cb(inp.KeyCode, isGp) end
            end)
        end)
        return btn
    end

    local function mkToggleKB(parent, txt, kbEntry, onToggle, onKB)
        local row = mkRow(parent)
        mkLabel(row, txt)
        if kbEntry then
            mkKB(row, kbEntry, function(k, isGp)
                if isGp then kbEntry.gp = k; kbEntry.kb = nil else kbEntry.kb = k; kbEntry.gp = nil end
                if onKB then onKB(k, isGp) end
            end)
        end
        local chk = Instance.new("TextButton", row)
        chk.Size = UDim2.new(0,18,0,18); chk.Position = UDim2.new(1,-24,0.5,-9)
        chk.BackgroundColor3 = CHECK_OFF; chk.BorderSizePixel = 0; chk.Text = ""; chk.ZIndex = 5
        Instance.new("UICorner", chk).CornerRadius = UDim.new(0,4)
        local chkStroke = Instance.new("UIStroke", chk); chkStroke.Color = WHITE; chkStroke.Thickness = 1
        local state = false
        local function sv(s)
            state = s
            TS:Create(chk, TweenInfo.new(0.15), {BackgroundColor3 = s and TOGGLE_ON_COLOR or CHECK_OFF}):Play()
        end
        chk.MouseButton1Click:Connect(function()
            if _anyKeyListening then return end
            state = not state; sv(state); if onToggle then onToggle(state) end
        end)
        return sv
    end

    local function mkActionRow(parent, txt, onActivate, kbEntry)
        local row = mkRow(parent)
        mkLabel(row, txt)
        if kbEntry then
            mkKB(row, kbEntry, function(k, isGp)
                if isGp then kbEntry.gp = k; kbEntry.kb = nil else kbEntry.kb = k; kbEntry.gp = nil end
                saveConfig()
            end)
        end
        local btn = Instance.new("TextButton", row)
        btn.Size = UDim2.new(0,40,0,20); btn.Position = UDim2.new(1,-46,0.5,-10)
        btn.BackgroundColor3 = BTN_ACT; btn.BorderSizePixel = 0
        btn.Text = ">"; btn.TextColor3 = WHITE
        btn.Font = Enum.Font.GothamBlack; btn.TextSize = 10
        btn.AutoButtonColor = false; btn.ZIndex = 5
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0,5)
        local bs = Instance.new("UIStroke", btn); bs.Color = WHITE; bs.Thickness = 1
        btn.MouseButton1Click:Connect(function()
            TS:Create(btn, TweenInfo.new(0.06), {BackgroundColor3 = WHITE}):Play()
            TS:Create(btn, TweenInfo.new(0.06), {TextColor3 = BLACK}):Play()
            task.delay(0.14, function()
                TS:Create(btn, TweenInfo.new(0.1), {BackgroundColor3 = BTN_ACT}):Play()
                TS:Create(btn, TweenInfo.new(0.1), {TextColor3 = WHITE}):Play()
            end)
            if onActivate then onActivate() end
        end)
        return row
    end

    local sfSpeed   = TabRefs.contents["Speed"]
    local sfCombat  = TabRefs.contents["Combat"]
    local sfMechns  = TabRefs.contents["Mechns"]
    local sfVisual  = TabRefs.contents["Visual"]
    local sfExtra   = TabRefs.contents["Extra"]
    local sfMobile  = TabRefs.contents["Mobile"]

    -- Speed control helper: label + [-] [value box] [+] buttons
    local function mkSpeedCtrl(parent, txt, getVal, setVal, step)
        step = step or 5
        local row = Instance.new("Frame", parent)
        row.Size = UDim2.new(1,0,0,32); row.BackgroundTransparency = 1
        row.LayoutOrder = #parent:GetChildren() + 1
        mkLabel(row, txt)
        local minBtn = Instance.new("TextButton", row)
        minBtn.Size = UDim2.new(0,24,0,22); minBtn.Position = UDim2.new(1,-148,0.5,-11)
        minBtn.BackgroundColor3 = BTN_ACT; minBtn.BorderSizePixel = 0
        minBtn.Text = "-"; minBtn.TextColor3 = WHITE
        minBtn.Font = Enum.Font.GothamBlack; minBtn.TextSize = 15
        minBtn.AutoButtonColor = false; minBtn.ZIndex = 5
        Instance.new("UICorner", minBtn).CornerRadius = UDim.new(0,5)
        local mStroke = Instance.new("UIStroke", minBtn); mStroke.Color = WHITE; mStroke.Thickness = 1
        local valTb = Instance.new("TextBox", row)
        valTb.Size = UDim2.new(0,52,0,22); valTb.Position = UDim2.new(1,-120,0.5,-11)
        valTb.BackgroundColor3 = INP_BG; valTb.Text = tostring(getVal())
        valTb.TextColor3 = WHITE; valTb.Font = Enum.Font.GothamBold
        valTb.TextSize = 11; valTb.ClearTextOnFocus = false
        valTb.TextXAlignment = Enum.TextXAlignment.Center; valTb.ZIndex = 5
        Instance.new("UICorner", valTb).CornerRadius = UDim.new(0,5)
        local tbStroke = Instance.new("UIStroke", valTb); tbStroke.Color = DIM_LINE; tbStroke.Thickness = 1
        valTb.Focused:Connect(function() TS:Create(tbStroke, TweenInfo.new(0.12), {Color=WHITE}):Play() end)
        valTb.FocusLost:Connect(function()
            TS:Create(tbStroke, TweenInfo.new(0.12), {Color=DIM_LINE}):Play()
            local n = tonumber(valTb.Text)
            if n then setVal(math.clamp(math.floor(n), 1, 500)) end
            valTb.Text = tostring(getVal())
        end)
        local plusBtn = Instance.new("TextButton", row)
        plusBtn.Size = UDim2.new(0,24,0,22); plusBtn.Position = UDim2.new(1,-64,0.5,-11)
        plusBtn.BackgroundColor3 = BTN_ACT; plusBtn.BorderSizePixel = 0
        plusBtn.Text = "+"; plusBtn.TextColor3 = WHITE
        plusBtn.Font = Enum.Font.GothamBlack; plusBtn.TextSize = 15
        plusBtn.AutoButtonColor = false; plusBtn.ZIndex = 5
        Instance.new("UICorner", plusBtn).CornerRadius = UDim.new(0,5)
        local pStroke = Instance.new("UIStroke", plusBtn); pStroke.Color = WHITE; pStroke.Thickness = 1
        local function flash(b)
            TS:Create(b, TweenInfo.new(0.07), {BackgroundColor3=WHITE}):Play()
            task.delay(0.14, function() TS:Create(b, TweenInfo.new(0.10), {BackgroundColor3=BTN_ACT}):Play() end)
        end
        minBtn.MouseButton1Click:Connect(function()
            setVal(math.max(1, getVal() - step)); valTb.Text = tostring(getVal()); flash(minBtn)
        end)
        plusBtn.MouseButton1Click:Connect(function()
            setVal(math.min(500, getVal() + step)); valTb.Text = tostring(getVal()); flash(plusBtn)
        end)
    end

    mkSect(sfSpeed, "Speed Values")
    mkSpeedCtrl(sfSpeed, "Normal Speed", function() return NS end, function(v) NS=v; CS=math.max(1, NS/2); saveConfig() end, 5)
    mkSpeedCtrl(sfSpeed, "Grab Speed",   function() return CS end, function(v) CS=v; saveConfig() end, 5)
    mkBoxRow(sfSpeed, "Lagger Normal", LAGGER_SPEED, function(v) if v>0 and v<=500 then LAGGER_SPEED=v end; saveConfig() end)
    mkBoxRow(sfSpeed, "Lagger Carry", LAGGER_CARRY_SPEED, function(v) if v>0 and v<=500 then LAGGER_CARRY_SPEED=v end; saveConfig() end)

    mkSect(sfCombat, "Keybinds")
    mkActionRow(sfCombat, "Speed Toggle",  nil, KB.SpeedToggle)
    mkActionRow(sfCombat, "Lagger Toggle", nil, KB.LaggerToggle)
    mkActionRow(sfCombat, "Insta Reset",   nil, KB.InstaReset)

    mkSect(sfCombat, "Combat")
    setAutoSwingVisual   = mkToggle(sfCombat, "Auto Swing",     function(on) autoSwingEnabled=on; saveConfig() end, autoSwingEnabled)
    setAntiRagVisual     = mkToggle(sfCombat, "Anti Ragdoll",   function(on) antiRagdollEnabled=on; if on then startAntiRagdoll() else stopAntiRagdoll() end; saveConfig() end, antiRagdollEnabled)
    setMedusaResetVisual = mkToggle(sfCombat, "Medusa Reset",   function(on) medusaResetEnabled=on; saveConfig() end, medusaResetEnabled)
    setUnwalkVisual      = mkToggle(sfCombat, "Unwalk",         function(on) unwalkEnabled=on; if on then startUnwalk() else stopUnwalk() end; saveConfig() end, unwalkEnabled)

    mkSect(sfCombat, "Yousef")
    setBatCounterVisual  = mkToggle(sfCombat, "Bat Counter",    function(on) batCounterEnabled=on; if on then startBatCounter() else stopBatCounter() end; saveConfig() end, batCounterEnabled)
    setMedusaVisual      = mkToggle(sfCombat, "Medusa Counter", function(on) medusaCounterEnabled=on; if on then setupMedusa(LP.Character) else stopMedusaCounter() end; saveConfig() end, medusaCounterEnabled)

    bypassSetVisual = mkToggleKB(sfCombat, "Bypass TP", KB.TPLock, function(on)
        if _anyKeyListening then return end
        if on then
            if bypassTPEnabled then stopBypassTP() end
            startBypassTP()
            bypassTPEnabled = true
        else
            if bypassTPEnabled then stopBypassTP() end
            bypassTPEnabled = false
        end
        if bypassSetVisual then bypassSetVisual(bypassTPEnabled) end
        if mobBtnRefs.tpLock then mobBtnRefs.tpLock(bypassTPEnabled) end
        saveConfig()
    end, function() saveConfig() end)
    bypassSetVisual(bypassTPEnabled)

    

    setAntiDieVisual = mkToggle(sfCombat, "Anti Die", function(on)
        antiDieEnabled = on
        if on then startAntiDie() else stopAntiDie() end
        saveConfig()
    end, antiDieEnabled)

    mkSect(sfMechns, "Misc")
    setInfJumpVisual = mkToggle(sfMechns, "Infinite Jump", function(on)
        infJumpEnabled = on
        if infJumpEnabled then if infJumpMode == "hold" then startHoldInfJump() end
        else stopHoldInfJump() end
        saveConfig()
    end, infJumpEnabled)

    -- Jump Mode selector: Manual / Hold
    do
        local jRow = mkRow(sfMechns); mkLabel(jRow, "Jump Mode")
        local jBtns = {}
        local function mkJumpModeBtn(lbl, xOff, mode)
            local b = Instance.new("TextButton", jRow)
            b.Size = UDim2.new(0,52,0,20); b.Position = UDim2.new(1,xOff,0.5,-10)
            b.BackgroundColor3 = (infJumpMode==mode) and WHITE or BTN_ACT  -- preto/branco
            b.BorderSizePixel = 0; b.Text = lbl
            b.TextColor3 = (infJumpMode==mode) and BLACK or Color3.fromRGB(180,180,200)
            b.Font = Enum.Font.GothamBold; b.TextSize = 8; b.AutoButtonColor = false; b.ZIndex = 5
            Instance.new("UICorner", b).CornerRadius = UDim.new(0,4)
            table.insert(jBtns, b)
            b.MouseButton1Click:Connect(function()
                if infJumpMode == mode then return end
                infJumpMode = mode
                b.BackgroundColor3 = WHITE; b.TextColor3 = BLACK
                for _, ob in ipairs(jBtns) do
                    if ob ~= b then
                        ob.BackgroundColor3 = BTN_ACT; ob.TextColor3 = Color3.fromRGB(180,180,200)
                    end
                end
                stopHoldInfJump()
                if infJumpEnabled and mode == "hold" then startHoldInfJump() end
                saveConfig()
            end)
            return b
        end
        mkJumpModeBtn("Manual", -110, "manual")
        mkJumpModeBtn("Hold",    -54,  "hold")
    end

    mkSect(sfMechns, "Movement")

    -- Direction selector (Left / Right)
    do
        local dirRow = mkRow(sfMechns)
        mkLabel(dirRow, "Direction")
        local function mkDirBtn(label, dir, xOffset)
            local b = Instance.new("TextButton", dirRow)
            b.Size = UDim2.new(0, 48, 0, 22)
            b.Position = UDim2.new(1, xOffset, 0.5, -11)
            b.BorderSizePixel = 0
            b.Font = Enum.Font.GothamBold
            b.TextSize = 11
            b.AutoButtonColor = false
            b.ZIndex = 5
            b.Text = label
            Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
            local bs = Instance.new("UIStroke", b); bs.Thickness = 1
            return b
        end
        local btnRight = mkDirBtn("RIGHT", "right", -4)
        local btnLeft  = mkDirBtn("LEFT",  "left",  -56)
        local function refreshDirBtns()
            local ACTIVE   = Color3.fromRGB(200, 200, 200)
            local INACTIVE = Color3.fromRGB(40, 40, 60)
            btnRight.BackgroundColor3 = (BOX_AP_Direction == "right") and ACTIVE or INACTIVE
            btnLeft.BackgroundColor3  = (BOX_AP_Direction == "left")  and ACTIVE or INACTIVE
        end
        refreshDirBtns()
        btnRight.MouseButton1Click:Connect(function()
            BOX_AP_Direction = "right"; refreshDirBtns(); saveConfig()
        end)
        btnLeft.MouseButton1Click:Connect(function()
            BOX_AP_Direction = "left";  refreshDirBtns(); saveConfig()
        end)
    end
        
    mkActionRow(sfMechns, "Drop Brainrot", function() runDrop() end,     KB.DropBrainrot)
    mkActionRow(sfMechns, "TP Down",       function() runTPFloor() end,  KB.TPFloor)
    setAutoTPVisual = mkToggle(sfMechns, "Auto TP", function(on) autoTPEnabled=on; if on then startAutoTP() else stopAutoTP() end; saveConfig() end, autoTPEnabled)
    mkBoxRow(sfMechns, "TP Height", autoTPHeight, function(v) if v>=0 and v<=500 then autoTPHeight=v end; saveConfig() end)

    mkSect(sfVisual, "Visual Effects")
    setAntiLagVisual    = mkToggle(sfVisual, "Anti Lag",    function(on) if on then enableAntiLag() else disableAntiLag() end; saveConfig() end, antiLagEnabled)
    setStretchRezVisual = mkToggle(sfVisual, "Stretch Rez", function(on) if on then enableStretchRez() else disableStretchRez() end; saveConfig() end, stretchRezEnabled)

    mkSect(sfVisual, "Button Shapes")
    setCircleBtnsVisual = mkToggle(sfVisual, "Circle Buttons", function(on)
        circleButtonsEnabled=on
        if on then shapeButtonsEnabled=false; rectangularButtonsEnabled=false
            if setShapeVisual then setShapeVisual(false) end
            if setRectVisual then setRectVisual(false) end
        end
        if mobGuiRef then buildMobileButtons() end; saveConfig()
    end, circleButtonsEnabled)

    setShapeVisual = mkToggle(sfVisual, "Shape Buttons", function(on)
        shapeButtonsEnabled=on
        if on then circleButtonsEnabled=false; rectangularButtonsEnabled=false
            if setCircleBtnsVisual then setCircleBtnsVisual(false) end
            if setRectVisual then setRectVisual(false) end
        end
        if mobGuiRef then buildMobileButtons() end; saveConfig()
    end, shapeButtonsEnabled)

    setRectVisual = mkToggle(sfVisual, "Rectangular Buttons", function(on)
        rectangularButtonsEnabled=on
        if on then circleButtonsEnabled=false; shapeButtonsEnabled=false
            if setCircleBtnsVisual then setCircleBtnsVisual(false) end
            if setShapeVisual then setShapeVisual(false) end
        end
        if mobGuiRef then buildMobileButtons() end; saveConfig()
    end, rectangularButtonsEnabled)

    mkSect(sfVisual, "Sky Theme")
    do
        local row = mkRow(sfVisual); mkLabel(row, "Current Theme")
        local currentLbl = Instance.new("TextLabel", row)
        currentLbl.Size = UDim2.new(0,120,1,0); currentLbl.Position = UDim2.new(1,-124,0,0)
        currentLbl.BackgroundTransparency = 1; currentLbl.Text = currentSkyTheme or "Off"
        currentLbl.TextColor3 = WHITE; currentLbl.Font = Enum.Font.GothamBlack
        currentLbl.TextSize = 10; currentLbl.TextXAlignment = Enum.TextXAlignment.Right; currentLbl.ZIndex = 5

        local gridContainer = Instance.new("Frame", sfVisual)
        gridContainer.Size = UDim2.new(1,0,0,0); gridContainer.AutomaticSize = Enum.AutomaticSize.Y
        gridContainer.BackgroundTransparency = 1; gridContainer.LayoutOrder = #sfVisual:GetChildren()+1
        local grid = Instance.new("UIGridLayout", gridContainer)
        grid.CellSize = UDim2.new(1/3,-4,0,26); grid.CellPadding = UDim2.new(0,4,0,4)
        grid.SortOrder = Enum.SortOrder.LayoutOrder; grid.FillDirection = Enum.FillDirection.Horizontal

        local function refreshSkyBtns()
            for _, child in ipairs(gridContainer:GetChildren()) do
                if child:IsA("TextButton") then
                    local theme = child:GetAttribute("Theme"); local isActive = (theme == currentSkyTheme)
                    child.BackgroundColor3 = isActive and WHITE or BTN_ACT
                    child.TextColor3 = isActive and BLACK or Color3.fromRGB(180,180,200)
                end
            end
            currentLbl.Text = currentSkyTheme or "Off"
        end

        for _, theme in ipairs(SKY_ORDER) do
            local btn = Instance.new("TextButton", gridContainer)
            btn.BackgroundColor3 = (theme==currentSkyTheme) and WHITE or BTN_ACT
            btn.BorderSizePixel = 0; btn.Text = theme
            btn.TextColor3 = (theme==currentSkyTheme) and BLACK or Color3.fromRGB(180,180,200)
            btn.Font = Enum.Font.GothamBold; btn.TextSize = 8; btn.TextWrapped = true
            btn.AutoButtonColor = false; btn.ZIndex = 5
            btn:SetAttribute("Theme", theme)
            Instance.new("UICorner", btn).CornerRadius = UDim.new(0,4)
            local stroke = Instance.new("UIStroke", btn)
            stroke.Color = WHITE; stroke.Thickness = 1
            stroke.Transparency = (theme==currentSkyTheme) and 0 or 0.7
            btn.MouseButton1Click:Connect(function()
                currentSkyTheme = theme; applySkyTheme(theme); refreshSkyBtns(); saveConfig()
            end)
        end
        refreshSkyBtns()
    end

    animChangerSetVisual = mkToggle(sfExtra, "Animation Changer", function(on)
        animChangerEnabled = on
        if animChangerGui then animChangerGui.Enabled = on end
        if mobBtnRefs.animChanger then mobBtnRefs.animChanger(on) end
        saveConfig()
    end, animChangerEnabled)

    mkSect(sfExtra, "Performance")
    mkToggle(sfExtra, "Nuke Optimizer", function(on)
        if on then
            pcall(function()
                Lighting.GlobalShadows = false; Lighting.FogEnd = 9e9
                Lighting.EnvironmentDiffuseScale = 0; Lighting.EnvironmentSpecularScale = 0
                for _, e in ipairs(Lighting:GetChildren()) do
                    if e:IsA("BloomEffect") or e:IsA("BlurEffect") or e:IsA("ColorCorrectionEffect") or e:IsA("SunRaysEffect") or e:IsA("DepthOfFieldEffect") or e:IsA("Atmosphere") or e:IsA("Clouds") then
                        e:Destroy()
                    end
                end
                for _, obj in ipairs(workspace:GetDescendants()) do
                    if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Fire") or obj:IsA("Smoke") or obj:IsA("Sparkles") or obj:IsA("Explosion") then
                        pcall(function() obj:Destroy() end)
                    end
                    if obj:IsA("BasePart") and not obj:IsDescendantOf(LP.Character) then
                        pcall(function() obj.CastShadow = false end)
                    end
                end
            end)
        end; saveConfig()
    end, false)

    mkSect(sfExtra, "UI Scale")
    mkBoxRow(sfExtra, "UI Scale", uiScaleValue, function(v)
        if v>=0.4 and v<=1.3 then
            uiScaleValue=v; if uiScaleObject then uiScaleObject.Scale=v end; saveConfig()
        end
    end)



    -- â”€â”€â”€ Mobile Tab â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
    do
        mkSect(sfMobile, "Button Visibility")
        local mobBtnDefs = {
            {key="drop",         label="DROP BR"},
            {key="tpDown",       label="TP DOWN"},
            {key="batV2",        label="BAT V1"},
            {key="tpLock",       label="TP BAT"},
            {key="lagger",       label="LAGGER"},
            {key="laggerCarry",  label="LAGGER CARRY"},
            {key="carrySpeed",   label="CARRY SPD"},
            {key="instaReset",   label="INSTA RESET"},
            {key="animChanger",  label="ANIM CHANGER"},
        }
        for _, def in ipairs(mobBtnDefs) do
            local key = def.key
            mkToggle(sfMobile, def.label, function(on)
                buttonVisibility[key] = on
                if mobileButtonsEnabled then buildMobileButtons() end
                saveConfig()
            end, buttonVisibility[key] ~= false)
        end

        -- â”€â”€â”€ Mobile Settings â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
        mkSect(sfMobile, "Mobile Settings")

        local mobShowSet = mkToggle(sfMobile, "Show Mobile UI", function(on)
            mobileButtonsEnabled = on
            if on then buildMobileButtons() else destroyMobileButtons() end
            if setMobVisual then setMobVisual(on) end
            saveConfig()
        end, mobileButtonsEnabled)

        mkBoxRow(sfMobile, "Btn Size", mobileButtonsSize, function(v)
            if v >= 10 and v <= 200 then
                mobileButtonsSize = v
                if mobileButtonsEnabled then buildMobileButtons() end
                saveConfig()
            end
        end)

        local mobDragSet = mkToggle(sfMobile, "Btn Drag Per Btn", function(on)
            perButtonDragEnabled = on
            if mobileButtonsEnabled then buildMobileButtons() end
            saveConfig()
        end, perButtonDragEnabled)

        local mobLockSet = mkToggle(sfMobile, "Lock UI", function(on)
            uiLocked = on
            if setLockVisual then setLockVisual(on) end
            saveConfig()
        end, uiLocked)

        do
            local row = mkRow(sfMobile); mkLabel(row, "Reset Mobile Btns")
            local resetBtn = Instance.new("TextButton", row)
            resetBtn.Size = UDim2.new(0,60,0,20); resetBtn.Position = UDim2.new(1,-65,0.5,-10)
            resetBtn.BackgroundColor3 = BTN_ACT; resetBtn.BorderSizePixel = 0
            resetBtn.Text = "Reset"; resetBtn.TextColor3 = WHITE
            resetBtn.Font = Enum.Font.GothamBold; resetBtn.TextSize = 9
            resetBtn.AutoButtonColor = false; resetBtn.ZIndex = 5
            Instance.new("UICorner", resetBtn).CornerRadius = UDim.new(0,5)
            local rStroke = Instance.new("UIStroke", resetBtn); rStroke.Color = WHITE; rStroke.Thickness = 1
            resetBtn.MouseButton1Click:Connect(function()
                resetBtnPositions()
                TS:Create(resetBtn, TweenInfo.new(0.1), {BackgroundColor3=WHITE}):Play()
                task.delay(0.15, function()
                    TS:Create(resetBtn, TweenInfo.new(0.1), {BackgroundColor3=BTN_ACT}):Play()
                end)
            end)
        end

        -- â”€â”€â”€ Button Shapes â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
        mkSect(sfMobile, "Button Shapes")

        local mobCircleSet = mkToggle(sfMobile, "Circle Buttons", function(on)
            circleButtonsEnabled = on
            if on then
                shapeButtonsEnabled = false
                rectangularButtonsEnabled = false
                if setShapeVisual then setShapeVisual(false) end
                if setRectVisual then setRectVisual(false) end
            end
            if mobileButtonsEnabled then buildMobileButtons() end
            saveConfig()
        end, circleButtonsEnabled)

        local mobShapeSet = mkToggle(sfMobile, "Shape Buttons", function(on)
            shapeButtonsEnabled = on
            if on then
                circleButtonsEnabled = false
                rectangularButtonsEnabled = false
                if setCircleBtnsVisual then setCircleBtnsVisual(false) end
                if setRectVisual then setRectVisual(false) end
            end
            if mobileButtonsEnabled then buildMobileButtons() end
            saveConfig()
        end, shapeButtonsEnabled)

        local mobRectSet = mkToggle(sfMobile, "Rectangular Buttons", function(on)
            rectangularButtonsEnabled = on
            if on then
                circleButtonsEnabled = false
                shapeButtonsEnabled = false
                if setCircleBtnsVisual then setCircleBtnsVisual(false) end
                if setShapeVisual then setShapeVisual(false) end
            end
            if mobileButtonsEnabled then buildMobileButtons() end
            saveConfig()
        end, rectangularButtonsEnabled)

    end

    do
        local acGui = Instance.new("ScreenGui")
        acGui.Name = "AnimChangerHub"; acGui.ResetOnSpawn = false; acGui.Enabled = false
        acGui.DisplayOrder = 50
        if not pcall(function() acGui.Parent = game:GetService("CoreGui") end) then
            acGui.Parent = LP:WaitForChild("PlayerGui")
        end
        animChangerGui = acGui

        local AC_BG    = Color3.fromRGB(10, 10, 14)
        local AC_ROW   = Color3.fromRGB(20, 20, 28)
        local AC_ON    = Color3.fromRGB(0, 0, 0)
        local AC_OFF   = Color3.fromRGB(30, 30, 45)

        local acMain = Instance.new("Frame", acGui)
        acMain.Size = UDim2.new(0, 250, 0, 330)
        acMain.Position = UDim2.new(0.5, -125, 0.5, -165)
        acMain.BackgroundColor3 = AC_BG
        acMain.BorderSizePixel = 0
        acMain.ClipsDescendants = true
        acMain.Active = true
        acMain.Draggable = true
        Instance.new("UICorner", acMain).CornerRadius = UDim.new(0, 10)
        local acStroke = Instance.new("UIStroke", acMain)
        acStroke.Color = Color3.fromRGB(60, 60, 60); acStroke.Thickness = 1.5

        local acTitleBar = Instance.new("Frame", acMain)
        acTitleBar.Size = UDim2.new(1, 0, 0, 36)
        acTitleBar.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        acTitleBar.BorderSizePixel = 0
        Instance.new("UICorner", acTitleBar).CornerRadius = UDim.new(0, 10)
        local acFix = Instance.new("Frame", acTitleBar)
        acFix.Size = UDim2.new(1,0,0.5,0); acFix.Position = UDim2.new(0,0,0.5,0)
        acFix.BackgroundColor3 = Color3.fromRGB(0,0,0); acFix.BorderSizePixel = 0

        local acTitle = Instance.new("TextLabel", acTitleBar)
        acTitle.Size = UDim2.new(1,-36,1,0); acTitle.Position = UDim2.new(0,10,0,0)
        acTitle.BackgroundTransparency = 1; acTitle.Text = "Animation Changer"
        acTitle.TextColor3 = WHITE  -- mantido branco; acTitle.Font = Enum.Font.GothamBlack
        acTitle.TextSize = 13; acTitle.TextXAlignment = Enum.TextXAlignment.Left; acTitle.ZIndex = 4

        local acClose = Instance.new("TextButton", acTitleBar)
        acClose.Size = UDim2.new(0,22,0,22); acClose.Position = UDim2.new(1,-28,0.5,-11)
        acClose.BackgroundColor3 = Color3.fromRGB(180,30,30); acClose.BorderSizePixel = 0
        acClose.Text = "X"; acClose.TextColor3 = WHITE
        acClose.Font = Enum.Font.GothamBlack; acClose.TextSize = 14; acClose.ZIndex = 6
        Instance.new("UICorner", acClose).CornerRadius = UDim.new(0,5)
        acClose.MouseButton1Click:Connect(function()
            animChangerEnabled = false; acGui.Enabled = false
            if animChangerSetVisual then animChangerSetVisual(false) end
            if mobBtnRefs.animChanger then mobBtnRefs.animChanger(false) end
        end)

        local acTabBar = Instance.new("Frame", acMain)
        acTabBar.Size = UDim2.new(1,-12,0,26); acTabBar.Position = UDim2.new(0,6,0,42)
        acTabBar.BackgroundTransparency = 1

        local function makeACTab(lbl, xScale)
            local b = Instance.new("TextButton", acTabBar)
            b.Size = UDim2.new(0.33,-2,1,0); b.Position = UDim2.new(xScale,0,0,0)
            b.BackgroundColor3 = AC_OFF; b.BorderSizePixel = 0
            b.AutoButtonColor = false; b.Text = lbl
            b.Font = Enum.Font.GothamBold; b.TextSize = 10; b.TextColor3 = WHITE
            Instance.new("UICorner", b).CornerRadius = UDim.new(0,4)
            return b
        end
        local acTabNormal  = makeACTab("Normal",  0)
        local acTabSpecial = makeACTab("Special", 0.33)
        local acTabOther   = makeACTab("Other",   0.67)

        local acContent = Instance.new("Frame", acMain)
        acContent.Size = UDim2.new(1,-12,1,-78); acContent.Position = UDim2.new(0,6,0,72)
        acContent.BackgroundTransparency = 1; acContent.ClipsDescendants = true

        local function makeACScroll()
            local sf = Instance.new("ScrollingFrame", acContent)
            sf.Size = UDim2.new(1,0,1,0); sf.BackgroundTransparency = 1
            sf.BorderSizePixel = 0; sf.CanvasSize = UDim2.new(0,0,0,0)
            sf.ScrollBarThickness = 3; sf.ScrollBarImageTransparency = 0.5
            return sf
        end

        local function makeACAnimBtn(parent, yPos, lbl, anims)
            local btn = Instance.new("TextButton", parent)
            btn.Size = UDim2.new(1,-4,0,26); btn.Position = UDim2.new(0,2,0,yPos)
            btn.BackgroundColor3 = AC_ROW; btn.BorderSizePixel = 0
            btn.AutoButtonColor = false; btn.Text = ""; btn.ZIndex = 3
            Instance.new("UICorner", btn).CornerRadius = UDim.new(0,5)
            local bStroke = Instance.new("UIStroke", btn)
            bStroke.Color = Color3.fromRGB(60,60,60); bStroke.Thickness = 1

            local bLbl = Instance.new("TextLabel", btn)
            bLbl.Size = UDim2.new(1,-8,1,0); bLbl.Position = UDim2.new(0,6,0,0)
            bLbl.BackgroundTransparency = 1; bLbl.Text = lbl
            bLbl.Font = Enum.Font.GothamBold; bLbl.TextSize = 11
            bLbl.TextColor3 = WHITE; bLbl.TextXAlignment = Enum.TextXAlignment.Left; bLbl.ZIndex = 4

            local busy = false
            btn.MouseButton1Click:Connect(function()
                if busy then return end
                local char = LP.Character
                local animate = char and char:FindFirstChild("Animate")
                if not animate then return end
                busy = true
                local orig = bLbl.Text; bLbl.Text = "done!"
                TS:Create(btn, TweenInfo.new(0.12), {BackgroundColor3 = AC_ON}):Play()
                for key, id in pairs(anims) do
                    local path = string.split(key, ".")
                    local cur = animate
                    for _, seg in ipairs(path) do cur = cur and cur:FindFirstChild(seg) end
                    if cur and cur:IsA("Animation") then
                        cur.AnimationId = "http://www.roblox.com/asset/?id=" .. id
                    end
                end
                if char:FindFirstChildOfClass("Humanoid") then
                    char:FindFirstChildOfClass("Humanoid").Jump = true
                end
                task.delay(0.35, function()
                    bLbl.Text = orig
                    TS:Create(btn, TweenInfo.new(0.12), {BackgroundColor3 = AC_ROW}):Play()
                    busy = false
                end)
            end)
            return btn
        end

        local acNormalScroll = makeACScroll()
        local normalAnims = {
            Ninja      = {["idle.Animation1"]="656117400",["idle.Animation2"]="656118341",["walk.WalkAnim"]="656121766",["run.RunAnim"]="656118852",["jump.JumpAnim"]="656117878",["climb.ClimbAnim"]="656114359",["fall.FallAnim"]="656115606"},
            Levitation  = {["idle.Animation1"]="616006778",["idle.Animation2"]="616008087",["walk.WalkAnim"]="616013216",["run.RunAnim"]="616010382",["jump.JumpAnim"]="616008936",["climb.ClimbAnim"]="616003713",["fall.FallAnim"]="616005863"},
            Werewolf    = {["idle.Animation1"]="1083195517",["idle.Animation2"]="1083214717",["walk.WalkAnim"]="1083178339",["run.RunAnim"]="1083216690",["jump.JumpAnim"]="1083218792",["climb.ClimbAnim"]="1083182000",["fall.FallAnim"]="1083189019"},
            Stylish     = {["idle.Animation1"]="616136790",["idle.Animation2"]="616138447",["walk.WalkAnim"]="616146177",["run.RunAnim"]="616140816",["jump.JumpAnim"]="616139451",["climb.ClimbAnim"]="616133594",["fall.FallAnim"]="616134815"},
            Bubbly      = {["idle.Animation1"]="910004836",["idle.Animation2"]="910009958",["walk.WalkAnim"]="910034870",["run.RunAnim"]="910025107",["jump.JumpAnim"]="910016857",["fall.FallAnim"]="910001910",["swimidle.SwimIdle"]="910030921",["swim.Swim"]="910028158"},
            Cartoony    = {["idle.Animation1"]="742637544",["idle.Animation2"]="742638445",["walk.WalkAnim"]="742640026",["run.RunAnim"]="742638842",["jump.JumpAnim"]="742637942",["climb.ClimbAnim"]="742636889",["fall.FallAnim"]="742637151"},
            SuperHero   = {["idle.Animation1"]="616111295",["idle.Animation2"]="616113536",["walk.WalkAnim"]="616122287",["run.RunAnim"]="616117076",["jump.JumpAnim"]="616115533",["climb.ClimbAnim"]="616104706",["fall.FallAnim"]="616108001"},
            Knight      = {["idle.Animation1"]="657595757",["idle.Animation2"]="657568135",["walk.WalkAnim"]="657552124",["run.RunAnim"]="657564596",["jump.JumpAnim"]="658409194",["climb.ClimbAnim"]="658360781",["fall.FallAnim"]="657600338"},
            Zombie      = {["idle.Animation1"]="616158929",["idle.Animation2"]="616160636",["walk.WalkAnim"]="616168032",["run.RunAnim"]="616163682",["jump.JumpAnim"]="616161997",["climb.ClimbAnim"]="616156119",["fall.FallAnim"]="616157476"},
            Elder       = {["idle.Animation1"]="845397899",["idle.Animation2"]="845400520",["walk.WalkAnim"]="845403856",["run.RunAnim"]="845386501",["jump.JumpAnim"]="845398858",["climb.ClimbAnim"]="845392038",["fall.FallAnim"]="845396048"},
            Astronaut   = {["idle.Animation1"]="891621366",["idle.Animation2"]="891633237",["walk.WalkAnim"]="891667138",["run.RunAnim"]="891636393",["jump.JumpAnim"]="891627522",["climb.ClimbAnim"]="891609353",["fall.FallAnim"]="891617961"},
            Adidas      = {["idle.Animation1"]="18537376492",["idle.Animation2"]="18537371272",["walk.WalkAnim"]="18537392113",["run.RunAnim"]="18537384940",["jump.JumpAnim"]="18537380791",["climb.ClimbAnim"]="18537363391",["fall.FallAnim"]="18537367238",["swim.Swim"]="18537389531",["swimidle.SwimIdle"]="18537387180"},
            Toy         = {["idle.Animation1"]="782841498",["idle.Animation2"]="782845736",["walk.WalkAnim"]="782843345",["run.RunAnim"]="782842708",["jump.JumpAnim"]="782847020",["climb.ClimbAnim"]="782843869",["fall.FallAnim"]="782846423"},
            Pirate      = {["idle.Animation1"]="750781874",["idle.Animation2"]="750782770",["walk.WalkAnim"]="750785693",["run.RunAnim"]="750783738",["jump.JumpAnim"]="750782230",["climb.ClimbAnim"]="750779899",["fall.FallAnim"]="750780242"},
            Tryhard     = {["idle.Animation1"]="133806214992291",["idle.Animation2"]="94970088341563",["walk.WalkAnim"]="707897309",["run.RunAnim"]="707861613",["jump.JumpAnim"]="116936326516985",["climb.ClimbAnim"]="116936326516985",["fall.FallAnim"]="116936326516985",["swim.Swim"]="116936326516985",["swimidle.SwimIdle"]="116936326516985"},
            Vampire     = {["idle.Animation1"]="1083445855",["idle.Animation2"]="1083450166",["walk.WalkAnim"]="1083473930",["run.RunAnim"]="1083462077",["jump.JumpAnim"]="1083455352",["climb.ClimbAnim"]="1083439238",["fall.FallAnim"]="1083443587"},
        }
        local ny = 4
        for name, data in pairs(normalAnims) do
            makeACAnimBtn(acNormalScroll, ny, name, data); ny = ny + 30
        end
        acNormalScroll.CanvasSize = UDim2.new(0,0,0,ny+4)

        local acSpecialScroll = makeACScroll()
        local specialAnims = {
            Patrol     = {["idle.Animation1"]="1149612882",["idle.Animation2"]="1150842221",["walk.WalkAnim"]="1151231493",["run.RunAnim"]="1150967949",["jump.JumpAnim"]="1148811837",["climb.ClimbAnim"]="1148811837",["fall.FallAnim"]="1148863382"},
            Confident  = {["idle.Animation1"]="1069977950",["idle.Animation2"]="1069987858",["walk.WalkAnim"]="1070017263",["run.RunAnim"]="1070001516",["jump.JumpAnim"]="1069984524",["climb.ClimbAnim"]="1069946257",["fall.FallAnim"]="1069973677"},
            Popstar    = {["idle.Animation1"]="1212900985",["idle.Animation2"]="1150842221",["walk.WalkAnim"]="1212980338",["run.RunAnim"]="1212980348",["jump.JumpAnim"]="1212954642",["climb.ClimbAnim"]="1213044953",["fall.FallAnim"]="1212900995"},
            Sneaky     = {["idle.Animation1"]="1132473842",["idle.Animation2"]="1132477671",["walk.WalkAnim"]="1132510133",["run.RunAnim"]="1132494274",["jump.JumpAnim"]="1132489853",["climb.ClimbAnim"]="1132461372",["fall.FallAnim"]="1132469004"},
            Princess   = {["idle.Animation1"]="941003647",["idle.Animation2"]="941013098",["walk.WalkAnim"]="941028902",["run.RunAnim"]="941015281",["jump.JumpAnim"]="941008832",["climb.ClimbAnim"]="940996062",["fall.FallAnim"]="941000007"},
            Cowboy     = {["idle.Animation1"]="1014390418",["idle.Animation2"]="1014398616",["walk.WalkAnim"]="1014421541",["run.RunAnim"]="1014401683",["jump.JumpAnim"]="1014394726",["climb.ClimbAnim"]="1014380606",["fall.FallAnim"]="1014384571"},
            Ghost      = {["idle.Animation1"]="616006778",["idle.Animation2"]="616008087",["walk.WalkAnim"]="616013216",["run.RunAnim"]="616013216",["jump.JumpAnim"]="616008936",["fall.FallAnim"]="616005863",["swimidle.SwimIdle"]="616012453",["swim.Swim"]="616011509"},
        }
        local sy = 4
        for name, data in pairs(specialAnims) do
            makeACAnimBtn(acSpecialScroll, sy, name, data); sy = sy + 30
        end
        acSpecialScroll.CanvasSize = UDim2.new(0,0,0,sy+4)

        local acOtherScroll = makeACScroll()
        local otherAnims = {
            None   = {["idle.Animation1"]="0",["idle.Animation2"]="0",["walk.WalkAnim"]="0",["run.RunAnim"]="0",["jump.JumpAnim"]="0",["fall.FallAnim"]="0",["swimidle.SwimIdle"]="0",["swim.Swim"]="0"},
            Anthro = {["idle.Animation1"]="2510196951",["idle.Animation2"]="2510197257",["walk.WalkAnim"]="2510202577",["run.RunAnim"]="2510198475",["jump.JumpAnim"]="2510197830",["climb.ClimbAnim"]="2510192778",["fall.FallAnim"]="2510195892",["swim.Swim"]="10921264784",["swimidle.SwimIdle"]="10921265698"},
        }
        local oy = 4
        for name, data in pairs(otherAnims) do
            makeACAnimBtn(acOtherScroll, oy, name, data); oy = oy + 30
        end
        acOtherScroll.CanvasSize = UDim2.new(0,0,0,oy+4)

        local function acSwitchTab(name)
            acNormalScroll.Visible  = (name == "Normal")
            acSpecialScroll.Visible = (name == "Special")
            acOtherScroll.Visible   = (name == "Other")
            TS:Create(acTabNormal,  TweenInfo.new(0.1), {BackgroundColor3 = name=="Normal"  and AC_ON or AC_OFF}):Play()
            TS:Create(acTabSpecial, TweenInfo.new(0.1), {BackgroundColor3 = name=="Special" and AC_ON or AC_OFF}):Play()
            TS:Create(acTabOther,   TweenInfo.new(0.1), {BackgroundColor3 = name=="Other"   and AC_ON or AC_OFF}):Play()
        end
        acTabNormal.MouseButton1Click:Connect(function()  acSwitchTab("Normal")  end)
        acTabSpecial.MouseButton1Click:Connect(function() acSwitchTab("Special") end)
        acTabOther.MouseButton1Click:Connect(function()   acSwitchTab("Other")   end)

        LP.CharacterAdded:Connect(function(char)
            task.wait(0.5)
        end)

        acSwitchTab("Normal")
    end

    mainFrame.Visible = true
    miniToggleBtn.Visible = false
    print("Asta GUI Fully Loaded")
end



buildGui()
applyState()
if mobileButtonsEnabled then buildMobileButtons() end
print("Asta Fully Loaded")
