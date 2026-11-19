-- language: Luau, target: Roblox (Sniper Arena), runtime: executor
-- ESP com scan/render separados — atualiza todo frame

----------------------------------------------------------------------
-- SERVICES
----------------------------------------------------------------------
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace        = game:GetService("Workspace")
local CoreGui          = game:GetService("CoreGui")
local Camera           = Workspace.CurrentCamera
local LP               = Players.LocalPlayer

----------------------------------------------------------------------
-- CLEANUP ANTERIOR
----------------------------------------------------------------------
if getgenv().SniperESPCleanup then
    pcall(getgenv().SniperESPCleanup)
end

----------------------------------------------------------------------
-- CONFIG
----------------------------------------------------------------------
local CFG = {
    Enabled      = true,
    Boxes        = true,
    Names        = true,
    Health       = true,
    Distance     = true,
    Tracers      = false,
    HeadDot      = true,
    TeamCheck    = true,
    TeamColor    = false,
    MaxDist      = 1200,
    BoxColor     = Color3.fromRGB(255, 40, 40),
    TracerOrigin = "Bottom",
}

----------------------------------------------------------------------
-- STATE
----------------------------------------------------------------------
local MenuOpen   = true
local ESPCache   = {}
local GameChars  = {}
local CharFolder = nil
local LastScan   = 0

----------------------------------------------------------------------
-- CHARACTER DETECTION
----------------------------------------------------------------------
local function findCharFolder()
    local names = {
        "Characters", "Chars", "GameCharacters", "Alive",
        "InGame", "PlayerModels", "players", "chars",
        "Soldiers", "Units", "Snipers"
    }
    for _, n in ipairs(names) do
        local f = Workspace:FindFirstChild(n)
        if f then
            for _, c in ipairs(f:GetChildren()) do
                if c:IsA("Model") and c:FindFirstChildOfClass("Humanoid") then
                    return f
                end
            end
        end
    end
    for _, child in ipairs(Workspace:GetChildren()) do
        if child:IsA("Folder") or child:IsA("Model") then
            for _, n in ipairs(names) do
                local f = child:FindFirstChild(n)
                if f then
                    for _, c in ipairs(f:GetChildren()) do
                        if c:IsA("Model") and c:FindFirstChildOfClass("Humanoid") then
                            return f
                        end
                    end
                end
            end
        end
    end
    return nil
end

local function modelToPlayer(model)
    local p = Players:GetPlayerFromCharacter(model)
    if p then return p end
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr.Name == model.Name or plr.DisplayName == model.Name then
            return plr
        end
    end
    return nil
end

local function isLocal(model)
    if model == LP.Character then return true end
    if model.Name == LP.Name or model.Name == LP.DisplayName then return true end
    local p = Players:GetPlayerFromCharacter(model)
    return p and p == LP
end

local function isEnemy(model)
    if isLocal(model) then return false end
    local hum = model:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    if not model:FindFirstChild("HumanoidRootPart") then return false end
    if not CFG.TeamCheck then return true end
    local plr = modelToPlayer(model)
    if plr and LP.Team and plr.Team then
        return plr.Team ~= LP.Team
    end
    return true
end

local function refreshChars()
    local found, seen = {}, {}
    local function scan(folder)
        for _, obj in ipairs(folder:GetChildren()) do
            if obj:IsA("Model") and not seen[obj]
               and obj:FindFirstChildOfClass("Humanoid")
               and obj:FindFirstChild("HumanoidRootPart")
               and not isLocal(obj) then
                local hum = obj:FindFirstChildOfClass("Humanoid")
                if hum.Health > 0 then
                    seen[obj] = true
                    table.insert(found, obj)
                end
            end
        end
    end
    if CharFolder and CharFolder.Parent then scan(CharFolder) end
    scan(Workspace)
    for _, child in ipairs(Workspace:GetChildren()) do
        if (child:IsA("Folder") or child:IsA("Model"))
           and child ~= CharFolder
           and not child:FindFirstChildOfClass("Humanoid") then
            scan(child)
        end
    end
    GameChars = found
end

local function getMyRoot()
    if LP.Character then
        local r = LP.Character:FindFirstChild("HumanoidRootPart")
        if r then return r end
    end
    if CharFolder and CharFolder.Parent then
        for _, obj in ipairs(CharFolder:GetChildren()) do
            if obj:IsA("Model") and isLocal(obj) then
                local r = obj:FindFirstChild("HumanoidRootPart")
                if r then return r end
            end
        end
    end
    return nil
end

----------------------------------------------------------------------
-- UTILS
----------------------------------------------------------------------
local function w2s(pos)
    local v, on = Camera:WorldToViewportPoint(pos)
    return Vector2.new(v.X, v.Y), on, v.Z
end

local function vpCenter()
    local vp = Camera.ViewportSize
    return Vector2.new(vp.X / 2, vp.Y / 2)
end

----------------------------------------------------------------------
-- UI
----------------------------------------------------------------------
if CoreGui:FindFirstChild("SniperESP") then
    CoreGui:FindFirstChild("SniperESP"):Destroy()
end

local Gui = Instance.new("ScreenGui")
Gui.Name = "SniperESP"; Gui.ResetOnSpawn = false
Gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
pcall(function() Gui.Parent = CoreGui end)
if not Gui.Parent then Gui.Parent = LP:WaitForChild("PlayerGui") end

local Main = Instance.new("Frame")
Main.Size = UDim2.new(0, 220, 0, 0)
Main.Position = UDim2.new(0, 16, 0.5, -150)
Main.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
Main.BorderSizePixel = 0; Main.AutomaticSize = Enum.AutomaticSize.Y
Main.Active = true; Main.Parent = Gui
Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 8)
local stk = Instance.new("UIStroke", Main)
stk.Color = Color3.fromRGB(200, 60, 20); stk.Thickness = 1.5; stk.Transparency = 0.3
Instance.new("UIPadding", Main).PaddingBottom = UDim.new(0, 6)
local lay = Instance.new("UIListLayout", Main)
lay.Padding = UDim.new(0, 2); lay.SortOrder = Enum.SortOrder.LayoutOrder

-- title / drag handle
local Title = Instance.new("TextButton")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundColor3 = Color3.fromRGB(200, 50, 15)
Title.BorderSizePixel = 0
Title.Text = "  🎯  SNIPER ARENA ESP"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 13; Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.AutoButtonColor = false; Title.LayoutOrder = 0; Title.Parent = Main
Instance.new("UICorner", Title).CornerRadius = UDim.new(0, 8)

local dragging, dragStart, startPos
Title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true; dragStart = input.Position; startPos = Main.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local d = input.Position - dragStart
        Main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X,
                                   startPos.Y.Scale, startPos.Y.Offset + d.Y)
    end
end)

-- UI builders
local ord = 0
local function no() ord += 1; return ord end

local function addSep(text)
    local s = Instance.new("TextLabel")
    s.Size = UDim2.new(1, -12, 0, 20); s.Position = UDim2.new(0, 6, 0, 0)
    s.BackgroundTransparency = 1; s.BorderSizePixel = 0
    s.Text = text; s.TextColor3 = Color3.fromRGB(200, 60, 20)
    s.TextSize = 11; s.Font = Enum.Font.GothamBold
    s.TextXAlignment = Enum.TextXAlignment.Left
    s.LayoutOrder = no(); s.Parent = Main
end

local function addToggle(label, default, cb)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -12, 0, 26); b.Position = UDim2.new(0, 6, 0, 0)
    b.BackgroundColor3 = Color3.fromRGB(24, 24, 30); b.BorderSizePixel = 0
    b.TextSize = 12; b.Font = Enum.Font.Gotham
    b.TextXAlignment = Enum.TextXAlignment.Left; b.AutoButtonColor = false
    b.LayoutOrder = no(); b.Parent = Main
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 5)
    local st = default
    local function r()
        if st then
            b.Text = "    ● " .. label
            b.TextColor3 = Color3.fromRGB(80, 255, 100)
            b.BackgroundColor3 = Color3.fromRGB(24, 34, 26)
        else
            b.Text = "    ○ " .. label
            b.TextColor3 = Color3.fromRGB(180, 80, 80)
            b.BackgroundColor3 = Color3.fromRGB(34, 24, 24)
        end
    end
    r()
    b.MouseButton1Click:Connect(function() st = not st; r(); cb(st) end)
end

local function addSlider(label, mn, mx, step, default, cb)
    local f = Instance.new("Frame")
    f.Size = UDim2.new(1, -12, 0, 26); f.Position = UDim2.new(0, 6, 0, 0)
    f.BackgroundColor3 = Color3.fromRGB(24, 24, 30); f.BorderSizePixel = 0
    f.LayoutOrder = no(); f.Parent = Main
    Instance.new("UICorner", f).CornerRadius = UDim.new(0, 5)
    local l = Instance.new("TextLabel", f)
    l.Size = UDim2.new(0.5, 0, 1, 0); l.Position = UDim2.new(0, 10, 0, 0)
    l.BackgroundTransparency = 1; l.Text = label
    l.TextColor3 = Color3.fromRGB(180, 180, 200); l.TextSize = 11
    l.Font = Enum.Font.Gotham; l.TextXAlignment = Enum.TextXAlignment.Left
    local v = default
    local vl = Instance.new("TextLabel", f)
    vl.Size = UDim2.new(0, 44, 1, 0); vl.Position = UDim2.new(0.5, 4, 0, 0)
    vl.BackgroundTransparency = 1; vl.Text = tostring(v)
    vl.TextColor3 = Color3.fromRGB(255, 255, 255); vl.TextSize = 12; vl.Font = Enum.Font.GothamBold
    local function mkB(txt, pos, col, fn)
        local bt = Instance.new("TextButton", f)
        bt.Size = UDim2.new(0, 24, 0, 20); bt.Position = pos
        bt.BackgroundColor3 = col; bt.Text = txt
        bt.TextColor3 = Color3.fromRGB(255,255,255); bt.TextSize = 14
        bt.Font = Enum.Font.GothamBold; bt.BorderSizePixel = 0
        Instance.new("UICorner", bt).CornerRadius = UDim.new(0, 4)
        bt.MouseButton1Click:Connect(fn)
    end
    mkB("−", UDim2.new(1,-56,0,3), Color3.fromRGB(60,20,20), function()
        v = math.max(v - step, mn); vl.Text = tostring(v); cb(v)
    end)
    mkB("+", UDim2.new(1,-28,0,3), Color3.fromRGB(20,60,20), function()
        v = math.min(v + step, mx); vl.Text = tostring(v); cb(v)
    end)
end

local function addCycle(label, opts, default, cb)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -12, 0, 26); b.Position = UDim2.new(0, 6, 0, 0)
    b.BackgroundColor3 = Color3.fromRGB(24, 24, 30); b.BorderSizePixel = 0
    b.TextSize = 12; b.Font = Enum.Font.Gotham
    b.TextXAlignment = Enum.TextXAlignment.Left; b.AutoButtonColor = false
    b.TextColor3 = Color3.fromRGB(150, 150, 255)
    b.LayoutOrder = no(); b.Parent = Main
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 5)
    local i = table.find(opts, default) or 1
    b.Text = "    ◆ " .. label .. ": " .. opts[i]
    b.MouseButton1Click:Connect(function()
        i = (i % #opts) + 1
        b.Text = "    ◆ " .. label .. ": " .. opts[i]; cb(opts[i])
    end)
end

----------------------------------------------------------------------
-- BUILD UI
----------------------------------------------------------------------
addSep("  DISPLAY")
addToggle("ESP Enabled", CFG.Enabled, function(v) CFG.Enabled = v end)
addToggle("Boxes", CFG.Boxes, function(v) CFG.Boxes = v end)
addToggle("Names", CFG.Names, function(v) CFG.Names = v end)
addToggle("Health Bars", CFG.Health, function(v) CFG.Health = v end)
addToggle("Distance", CFG.Distance, function(v) CFG.Distance = v end)
addToggle("Tracers", CFG.Tracers, function(v) CFG.Tracers = v end)
addToggle("Head Dot", CFG.HeadDot, function(v) CFG.HeadDot = v end)

addSep("  FILTERS")
addToggle("Team Check", CFG.TeamCheck, function(v) CFG.TeamCheck = v end)
addToggle("Use Team Colors", CFG.TeamColor, function(v) CFG.TeamColor = v end)
addSlider("Max Dist", 100, 3000, 100, CFG.MaxDist, function(v) CFG.MaxDist = v end)

addSep("  STYLE")
addCycle("Tracer Origin", {"Bottom", "Center"}, CFG.TracerOrigin, function(v) CFG.TracerOrigin = v end)

local statusLbl = Instance.new("TextLabel")
statusLbl.Size = UDim2.new(1, -12, 0, 18); statusLbl.Position = UDim2.new(0, 6, 0, 0)
statusLbl.BackgroundTransparency = 1; statusLbl.Text = "  detecting..."
statusLbl.TextColor3 = Color3.fromRGB(90, 90, 90)
statusLbl.TextSize = 10; statusLbl.Font = Enum.Font.Gotham
statusLbl.TextXAlignment = Enum.TextXAlignment.Left
statusLbl.LayoutOrder = no(); statusLbl.Parent = Main

local hintLbl = Instance.new("TextLabel")
hintLbl.Size = UDim2.new(1, -12, 0, 16); hintLbl.Position = UDim2.new(0, 6, 0, 0)
hintLbl.BackgroundTransparency = 1; hintLbl.Text = "  [Insert] toggle menu"
hintLbl.TextColor3 = Color3.fromRGB(60, 60, 60)
hintLbl.TextSize = 10; hintLbl.Font = Enum.Font.Gotham
hintLbl.TextXAlignment = Enum.TextXAlignment.Left
hintLbl.LayoutOrder = no(); hintLbl.Parent = Main

----------------------------------------------------------------------
-- ESP DRAWING
----------------------------------------------------------------------
local function newESP()
    local e = { Lines = {} }
    for i = 1, 4 do
        local l = Drawing.new("Line")
        l.Thickness = 1.2; l.Visible = false
        e.Lines[i] = l
    end
    e.Name = Drawing.new("Text")
    e.Name.Size = 13; e.Name.Center = true; e.Name.Outline = true
    e.Name.Color = Color3.fromRGB(255, 255, 255)
    pcall(function() e.Name.Font = Drawing.Fonts.Plex end)

    e.Dist = Drawing.new("Text")
    e.Dist.Size = 11; e.Dist.Center = true; e.Dist.Outline = true
    e.Dist.Color = Color3.fromRGB(180, 180, 180)
    pcall(function() e.Dist.Font = Drawing.Fonts.Plex end)

    e.HPBG = Drawing.new("Line"); e.HPBG.Thickness = 3.5
    e.HPBG.Color = Color3.fromRGB(30, 30, 30)
    e.HPBar = Drawing.new("Line"); e.HPBar.Thickness = 1.8
    e.Tracer = Drawing.new("Line"); e.Tracer.Thickness = 1
    e.Tracer.Color = Color3.fromRGB(255, 200, 50)
    e.HeadDot = Drawing.new("Circle"); e.HeadDot.Radius = 3
    e.HeadDot.NumSides = 12; e.HeadDot.Filled = true
    e.HeadDot.Color = Color3.fromRGB(255, 255, 255)
    return e
end

local function hideESP(e)
    for _, l in ipairs(e.Lines) do l.Visible = false end
    e.Name.Visible = false; e.Dist.Visible = false
    e.HPBG.Visible = false; e.HPBar.Visible = false
    e.Tracer.Visible = false; e.HeadDot.Visible = false
end

local function removeESP(e)
    for _, l in ipairs(e.Lines) do l:Remove() end
    e.Name:Remove(); e.Dist:Remove()
    e.HPBG:Remove(); e.HPBar:Remove()
    e.Tracer:Remove(); e.HeadDot:Remove()
end

local function getColor(model)
    if CFG.TeamColor then
        local plr = modelToPlayer(model)
        if plr and plr.Team then return plr.TeamColor.Color end
    end
    return CFG.BoxColor
end

local function renderESP(model)
    if not ESPCache[model] then ESPCache[model] = newESP() end
    local e = ESPCache[model]

    if not CFG.Enabled or not isEnemy(model) then hideESP(e); return end

    local root = model:FindFirstChild("HumanoidRootPart")
    local head = model:FindFirstChild("Head")
    local hum  = model:FindFirstChildOfClass("Humanoid")
    local mr   = getMyRoot()
    if not root or not hum or not mr then hideESP(e); return end

    local dist = (root.Position - mr.Position).Magnitude
    if dist > CFG.MaxDist then hideESP(e); return end

    local top2D, topOn = w2s(root.Position + Vector3.new(0, 3.2, 0))
    local bot2D, botOn = w2s(root.Position - Vector3.new(0, 3.2, 0))
    if not topOn and not botOn then hideESP(e); return end

    local h  = math.abs(bot2D.Y - top2D.Y)
    local w  = h * 0.5
    local cx = (top2D.X + bot2D.X) / 2
    local color = getColor(model)

    -- box
    if CFG.Boxes then
        local tl = Vector2.new(cx - w/2, top2D.Y)
        local tr = Vector2.new(cx + w/2, top2D.Y)
        local bl = Vector2.new(cx - w/2, bot2D.Y)
        local br = Vector2.new(cx + w/2, bot2D.Y)
        e.Lines[1].From = tl; e.Lines[1].To = tr
        e.Lines[2].From = tr; e.Lines[2].To = br
        e.Lines[3].From = br; e.Lines[3].To = bl
        e.Lines[4].From = bl; e.Lines[4].To = tl
        for _, l in ipairs(e.Lines) do l.Color = color; l.Visible = true end
    else
        for _, l in ipairs(e.Lines) do l.Visible = false end
    end

    -- name
    if CFG.Names then
        e.Name.Text = model.Name
        e.Name.Position = Vector2.new(cx, top2D.Y - 16)
        e.Name.Visible = true
    else e.Name.Visible = false end

    -- health
    if CFG.Health then
        local hpF = math.clamp(hum.Health / hum.MaxHealth, 0, 1)
        local bx = cx - w/2 - 6
        e.HPBG.From = Vector2.new(bx, bot2D.Y)
        e.HPBG.To   = Vector2.new(bx, top2D.Y); e.HPBG.Visible = true
        e.HPBar.From = Vector2.new(bx, bot2D.Y)
        e.HPBar.To   = Vector2.new(bx, bot2D.Y - h * hpF)
        e.HPBar.Color = Color3.fromRGB(255 * (1 - hpF), 255 * hpF, 0)
        e.HPBar.Visible = true
    else e.HPBG.Visible = false; e.HPBar.Visible = false end

    -- distance
    if CFG.Distance then
        e.Dist.Text = string.format("[%d m]", math.floor(dist))
        e.Dist.Position = Vector2.new(cx, bot2D.Y + 3)
        e.Dist.Visible = true
    else e.Dist.Visible = false end

    -- tracer
    if CFG.Tracers then
        local vp = Camera.ViewportSize
        if CFG.TracerOrigin == "Bottom" then
            e.Tracer.From = Vector2.new(vp.X / 2, vp.Y)
        else
            e.Tracer.From = Vector2.new(vp.X / 2, vp.Y / 2)
        end
        e.Tracer.To = Vector2.new(cx, bot2D.Y)
        e.Tracer.Color = color; e.Tracer.Visible = true
    else e.Tracer.Visible = false end

    -- head dot
    if CFG.HeadDot and head then
        local hp, hOn = w2s(head.Position + Vector3.new(0, 0.3, 0))
        if hOn then
            e.HeadDot.Position = hp
            e.HeadDot.Radius = math.clamp(1200 / dist, 2, 8)
            e.HeadDot.Color = color; e.HeadDot.Visible = true
        else e.HeadDot.Visible = false end
    else e.HeadDot.Visible = false end
end

----------------------------------------------------------------------
-- STALE CLEANUP
----------------------------------------------------------------------
local function cleanStale()
    local alive = {}
    for _, m in ipairs(GameChars) do alive[m] = true end
    for model, e in pairs(ESPCache) do
        if not alive[model] or not model.Parent then
            pcall(hideESP, e); pcall(removeESP, e)
            ESPCache[model] = nil
        end
    end
end

----------------------------------------------------------------------
-- INPUT
----------------------------------------------------------------------
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        MenuOpen = not MenuOpen; Main.Visible = MenuOpen
    end
end)

----------------------------------------------------------------------
-- SCAN (Heartbeat — a cada 0.35s, acha personagens)
----------------------------------------------------------------------
local scanConn = RunService.Heartbeat:Connect(function()
    local now = tick()
    if now - LastScan < 0.35 then return end
    LastScan = now

    pcall(function()
        if not CharFolder or not CharFolder.Parent then
            CharFolder = findCharFolder()
        end
        refreshChars()
        cleanStale()

        local fn = CharFolder and CharFolder.Name or "broad"
        statusLbl.Text = string.format("  targets: %d | src: %s", #GameChars, fn)
    end)
end)

----------------------------------------------------------------------
-- RENDER (RenderStepped — TODO FRAME, posições em tempo real)
----------------------------------------------------------------------
local renderConn = RunService.RenderStepped:Connect(function()
    Camera = Workspace.CurrentCamera

    local chars = GameChars
    for _, model in ipairs(chars) do
        local ok = pcall(renderESP, model)
        if not ok and ESPCache[model] then
            pcall(hideESP, ESPCache[model])
        end
    end
end)

----------------------------------------------------------------------
-- CLEANUP GLOBAL
----------------------------------------------------------------------
getgenv().SniperESPCleanup = function()
    if scanConn then scanConn:Disconnect() end
    if renderConn then renderConn:Disconnect() end
    for _, e in pairs(ESPCache) do pcall(removeESP, e) end
    if Gui then Gui:Destroy() end
    ESPCache = {}
    GameChars = {}
end

print("[LUNAR] Sniper Arena ESP loaded — scan/render split")
