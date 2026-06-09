-- ================================================================
--   COIN FARMER PRO  v2.0
--   Universal | Persistent GUI | Anti-AFK | Lay/Stand | Auto-Reset
-- ================================================================

local Players         = game:GetService("Players")
local RunService      = game:GetService("RunService")
local UserInputService= game:GetService("UserInputService")
local TweenService    = game:GetService("TweenService")
local VirtualUser     = game:GetService("VirtualUser")
local StarterGui      = game:GetService("StarterGui")

local LP = Players.LocalPlayer

-- ================================================================
--  PERSISTENT GUI SETUP (survives reset via CoreGui)
-- ================================================================
local CoreGui = game:GetService("CoreGui")
local ScreenGui = CoreGui:FindFirstChild("CFP_GUI")
if not ScreenGui then
    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name            = "CFP_GUI"
    ScreenGui.ResetOnSpawn    = false
    ScreenGui.ZIndexBehavior  = Enum.ZIndexBehavior.Sibling
    ScreenGui.IgnoreGuiInset  = true
    ScreenGui.Parent          = CoreGui
end

-- ================================================================
--  GAMEPASS CHECK  (Elite pass = 50 coins, default = 40)
-- ================================================================
local MarketplaceService = game:GetService("MarketplaceService")
local ELITE_PASS_ID = 429957

local function detectCoinLimit()
    local ok, owns = pcall(function()
        return MarketplaceService:UserOwnsGamePassAsync(LP.UserId, ELITE_PASS_ID)
    end)
    if ok and owns then
        print("[CoinFarmerPro] Elite gamepass detected — limit set to 50")
        return 50
    else
        print("[CoinFarmerPro] No Elite gamepass — limit set to 40")
        return 40
    end
end

-- ================================================================
--  STATE
-- ================================================================
local S = {
    -- Settings
    enabled      = false,
    speed        = 21,
    style        = "Stand",   -- "Stand" | "Lay"
    antiAfk      = true,
    noRender     = false,
    limitAction  = "AutoReset", -- "AutoReset" | "Stop"
    coinLimit    = detectCoinLimit(),  -- auto: 40 or 50 based on gamepass

    -- Runtime
    coins        = 0,
    totalCoins   = 0,
    resets       = 0,
    sessionStart = os.clock(),

    -- Connections
    farmConn     = nil,
    afkConn      = nil,
    bagConn      = nil,
    roundConn    = nil,

    -- Flags
    inRound      = false,
    bagFull      = false,
    resetting    = false,
}

-- ================================================================
--  HELPERS
-- ================================================================
local function tw(obj, props, t, style, dir)
    return TweenService:Create(obj,
        TweenInfo.new(t or 0.22, style or Enum.EasingStyle.Quart, dir or Enum.EasingDirection.Out),
        props)
end

local function ni(cls, props, parent)
    local o = Instance.new(cls)
    for k,v in pairs(props) do o[k]=v end
    if parent then o.Parent = parent end
    return o
end

local function addCorner(f, r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, r or 8)
    c.Parent = f
    return c
end

local function addStroke(f, col, th)
    local s = Instance.new("UIStroke")
    s.Color     = col or Color3.fromRGB(50,60,90)
    s.Thickness = th  or 1
    s.Parent    = f
    return s
end

local function getChar()
    return LP.Character
end
local function getRoot()
    local c = getChar()
    return c and c:FindFirstChild("HumanoidRootPart")
end
local function getHum()
    local c = getChar()
    return c and c:FindFirstChildOfClass("Humanoid")
end

-- ================================================================
--  COLOURS
-- ================================================================
local COL = {
    bg      = Color3.fromRGB(9,  11, 19),
    panel   = Color3.fromRGB(14, 18, 30),
    border  = Color3.fromRGB(35, 45, 72),
    accent  = Color3.fromRGB(82, 196, 255),
    gold    = Color3.fromRGB(255,185, 50),
    green   = Color3.fromRGB(72, 225,130),
    red     = Color3.fromRGB(255, 75, 85),
    muted   = Color3.fromRGB(90, 105,140),
    text    = Color3.fromRGB(210,220,255),
}

-- ================================================================
--  BUILD GUI
-- ================================================================

-- ── Mini button (collapsed state) ──────────────────────────────
local MiniBtn = ni("TextButton", {
    Name              = "MiniBtn",
    Size              = UDim2.new(0,44,0,44),
    Position          = UDim2.new(0,16,0.5,-22),
    BackgroundColor3  = COL.bg,
    Text              = "CF",
    TextSize          = 13,
    Font              = Enum.Font.GothamBold,
    TextColor3        = COL.accent,
    AutoButtonColor   = false,
    Visible           = false,
    ZIndex            = 30,
    Parent            = ScreenGui,
})
addCorner(MiniBtn, 12)
addStroke(MiniBtn, COL.accent, 1.5)

-- ── Main frame ─────────────────────────────────────────────────
local W = 308
local H = 510
local Main = ni("Frame", {
    Name             = "Main",
    Size             = UDim2.new(0,W,0,H),
    Position         = UDim2.new(0,16,0.5,-H/2),
    BackgroundColor3 = COL.bg,
    BorderSizePixel  = 0,
    ZIndex           = 10,
    Parent           = ScreenGui,
})
addCorner(Main, 12)
addStroke(Main, COL.border, 1.5)

-- drop shadow
local Shadow = ni("Frame",{
    Size=UDim2.new(1,20,1,20),Position=UDim2.new(0,-10,0,10),
    BackgroundColor3=Color3.new(0,0,0),BackgroundTransparency=0.55,
    ZIndex=9,Parent=Main,
})
addCorner(Shadow,14)

-- ── Header ─────────────────────────────────────────────────────
local Header = ni("Frame",{
    Size=UDim2.new(1,0,0,48),BackgroundColor3=COL.panel,
    BorderSizePixel=0,ZIndex=11,Parent=Main,
})
addCorner(Header,12)
ni("Frame",{  -- mask bottom corners
    Size=UDim2.new(1,0,0,12),Position=UDim2.new(0,0,1,-12),
    BackgroundColor3=COL.panel,BorderSizePixel=0,ZIndex=11,Parent=Header,
})

local TitleLbl = ni("TextLabel",{
    Text="COIN FARMER PRO", TextSize=13, Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,-80,1,0), Position=UDim2.new(0,14,0,0),
    BackgroundTransparency=1, TextColor3=COL.accent,
    TextXAlignment=Enum.TextXAlignment.Left, ZIndex=12, Parent=Header,
})

local VerLbl = ni("TextLabel",{
    Text="v2.0", TextSize=10, Font=Enum.Font.Gotham,
    Size=UDim2.new(0,30,1,0), Position=UDim2.new(0,160,0,0),
    BackgroundTransparency=1, TextColor3=COL.muted,
    TextXAlignment=Enum.TextXAlignment.Left, ZIndex=12, Parent=Header,
})

local CollapseBtn = ni("TextButton",{
    Text="—", TextSize=16, Font=Enum.Font.GothamBold,
    Size=UDim2.new(0,30,0,24), Position=UDim2.new(1,-38,0.5,-12),
    BackgroundColor3=COL.border, TextColor3=COL.text,
    AutoButtonColor=false, ZIndex=12, Parent=Header,
})
addCorner(CollapseBtn,6)

-- ── Stats bar ──────────────────────────────────────────────────
local StatsPanel = ni("Frame",{
    Size=UDim2.new(1,-20,0,50), Position=UDim2.new(0,10,0,56),
    BackgroundColor3=COL.panel, BorderSizePixel=0, ZIndex=11, Parent=Main,
})
addCorner(StatsPanel,8)
addStroke(StatsPanel,COL.border)

local function statCell(labelTxt, xRatio)
    local f = ni("Frame",{
        Size=UDim2.new(1/4,0,1,0), Position=UDim2.new(xRatio,0,0,0),
        BackgroundTransparency=1, ZIndex=12, Parent=StatsPanel,
    })
    ni("TextLabel",{
        Text=labelTxt, TextSize=9, Font=Enum.Font.Gotham,
        Size=UDim2.new(1,0,0,16), Position=UDim2.new(0,0,0,5),
        BackgroundTransparency=1, TextColor3=COL.muted, ZIndex=12, Parent=f,
    })
    local val = ni("TextLabel",{
        Text="0", TextSize=15, Font=Enum.Font.GothamBold,
        Size=UDim2.new(1,0,0,22), Position=UDim2.new(0,0,0,23),
        BackgroundTransparency=1, TextColor3=COL.text, ZIndex=12, Parent=f,
    })
    return val
end

local CoinsLbl  = statCell("COINS",   0)
local TotalLbl  = statCell("TOTAL",   0.25)
local ResetsLbl = statCell("RESETS",  0.50)
local TimeLbl   = statCell("SESSION", 0.75)

-- dividers
for _, x in ipairs({0.25,0.5,0.75}) do
    ni("Frame",{
        Size=UDim2.new(0,1,0.6,0), Position=UDim2.new(x,0,0.2,0),
        BackgroundColor3=COL.border, BorderSizePixel=0, ZIndex=12, Parent=StatsPanel,
    })
end

-- ── Progress bar ───────────────────────────────────────────────
local ProgBG = ni("Frame",{
    Size=UDim2.new(1,-20,0,14), Position=UDim2.new(0,10,0,114),
    BackgroundColor3=COL.panel, BorderSizePixel=0, ZIndex=11, Parent=Main,
})
addCorner(ProgBG,7)
addStroke(ProgBG,COL.border)

local ProgFill = ni("Frame",{
    Size=UDim2.new(0,0,1,0), BackgroundColor3=COL.green,
    BorderSizePixel=0, ZIndex=12, Parent=ProgBG,
})
addCorner(ProgFill,7)

local ProgTxt = ni("TextLabel",{
    Text="0 / ...", TextSize=9, Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,0,1,0), BackgroundTransparency=1,
    TextColor3=COL.text, ZIndex=13, Parent=ProgBG,
})

-- ── Section label helper ────────────────────────────────────────
local function sectionLbl(txt, y)
    ni("TextLabel",{
        Text=txt, TextSize=10, Font=Enum.Font.GothamBold,
        Size=UDim2.new(1,-20,0,14), Position=UDim2.new(0,10,0,y),
        BackgroundTransparency=1, TextColor3=COL.muted,
        TextXAlignment=Enum.TextXAlignment.Left, ZIndex=11, Parent=Main,
    })
end

-- ── Speed ──────────────────────────────────────────────────────
sectionLbl("MOVEMENT SPEED", 135)

local SpeedRow = ni("Frame",{
    Size=UDim2.new(1,-20,0,34), Position=UDim2.new(0,10,0,151),
    BackgroundColor3=COL.panel, BorderSizePixel=0, ZIndex=11, Parent=Main,
})
addCorner(SpeedRow,8)
addStroke(SpeedRow,COL.border)

local SpeedMinus = ni("TextButton",{
    Text="−", TextSize=18, Font=Enum.Font.GothamBold,
    Size=UDim2.new(0,34,1,0), BackgroundColor3=COL.border,
    TextColor3=COL.accent, AutoButtonColor=false, ZIndex=12, Parent=SpeedRow,
})
addCorner(SpeedMinus,8)

local SpeedVal = ni("TextLabel",{
    Text="21", TextSize=15, Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,-68,1,0), Position=UDim2.new(0,34,0,0),
    BackgroundTransparency=1, TextColor3=COL.accent, ZIndex=12, Parent=SpeedRow,
})

local SpeedPlus = ni("TextButton",{
    Text="+", TextSize=18, Font=Enum.Font.GothamBold,
    Size=UDim2.new(0,34,1,0), Position=UDim2.new(1,-34,0,0),
    BackgroundColor3=COL.border, TextColor3=COL.accent,
    AutoButtonColor=false, ZIndex=12, Parent=SpeedRow,
})
addCorner(SpeedPlus,8)

-- ── Style ──────────────────────────────────────────────────────
sectionLbl("MOVEMENT STYLE", 193)

local StyleRow = ni("Frame",{
    Size=UDim2.new(1,-20,0,34), Position=UDim2.new(0,10,0,209),
    BackgroundColor3=COL.panel, BorderSizePixel=0, ZIndex=11, Parent=Main,
})
addCorner(StyleRow,8)

local function styleToggle(txt, xscale)
    local b = ni("TextButton",{
        Text=txt, TextSize=12, Font=Enum.Font.GothamBold,
        Size=UDim2.new(0.5,-2,1,0), Position=UDim2.new(xscale, xscale==0 and 0 or 2,0,0),
        BackgroundColor3=COL.border, TextColor3=COL.muted,
        AutoButtonColor=false, ZIndex=12, Parent=StyleRow,
    })
    addCorner(b,8)
    return b
end

local BtnStand = styleToggle("STAND", 0)
local BtnLay   = styleToggle("LAY",  0.5)

local function refreshStyleBtns()
    if S.style == "Stand" then
        BtnStand.BackgroundColor3 = COL.accent
        BtnStand.TextColor3       = COL.bg
        BtnLay.BackgroundColor3   = COL.border
        BtnLay.TextColor3         = COL.muted
    else
        BtnLay.BackgroundColor3   = COL.accent
        BtnLay.TextColor3         = COL.bg
        BtnStand.BackgroundColor3 = COL.border
        BtnStand.TextColor3       = COL.muted
    end
end
refreshStyleBtns()

-- ── Toggles ────────────────────────────────────────────────────
sectionLbl("OPTIONS", 251)

local function buildToggle(labelTxt, yPos, initOn, activeCol)
    local row = ni("Frame",{
        Size=UDim2.new(1,-20,0,32), Position=UDim2.new(0,10,0,yPos),
        BackgroundColor3=COL.panel, BorderSizePixel=0, ZIndex=11, Parent=Main,
    })
    addCorner(row,8)

    ni("TextLabel",{
        Text=labelTxt, TextSize=11, Font=Enum.Font.Gotham,
        Size=UDim2.new(1,-58,1,0), Position=UDim2.new(0,10,0,0),
        BackgroundTransparency=1, TextColor3=COL.text,
        TextXAlignment=Enum.TextXAlignment.Left, ZIndex=12, Parent=row,
    })

    local track = ni("Frame",{
        Size=UDim2.new(0,40,0,20), Position=UDim2.new(1,-48,0.5,-10),
        BackgroundColor3= initOn and (activeCol or COL.green) or COL.border,
        ZIndex=12, Parent=row,
    })
    addCorner(track,10)

    local knob = ni("Frame",{
        Size=UDim2.new(0,16,0,16),
        Position= initOn and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8),
        BackgroundColor3=Color3.new(1,1,1), ZIndex=13, Parent=track,
    })
    addCorner(knob,8)

    local state = {on=initOn}
    local aColor = activeCol or COL.green

    local function doToggle()
        state.on = not state.on
        local kp = state.on and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8)
        local tc = state.on and aColor or COL.border
        tw(knob,  {Position=kp}, 0.18):Play()
        tw(track, {BackgroundColor3=tc}, 0.18):Play()
        return state.on
    end

    row.InputBegan:Connect(function(i)
        if i.UserInputType==Enum.UserInputType.MouseButton1
        or i.UserInputType==Enum.UserInputType.Touch then
            doToggle()
        end
    end)

    return function() return state.on end, doToggle
end

local getAntiAFK,  togAntiAFK  = buildToggle("ANTI-AFK PROTECTION",  267, true)
local getNoRender, togNoRender = buildToggle("DISABLE RENDERING",     305, false, COL.accent)

-- ── Limit action ───────────────────────────────────────────────
sectionLbl("ON COIN LIMIT", 345)

local LimitRow = ni("Frame",{
    Size=UDim2.new(1,-20,0,34), Position=UDim2.new(0,10,0,361),
    BackgroundColor3=COL.panel, BorderSizePixel=0, ZIndex=11, Parent=Main,
})
addCorner(LimitRow,8)

local function limitBtn(txt, xs)
    local b = ni("TextButton",{
        Text=txt, TextSize=11, Font=Enum.Font.GothamBold,
        Size=UDim2.new(0.5,-2,1,0), Position=UDim2.new(xs, xs==0 and 0 or 2,0,0),
        BackgroundColor3=COL.border, TextColor3=COL.muted,
        AutoButtonColor=false, ZIndex=12, Parent=LimitRow,
    })
    addCorner(b,8)
    return b
end

local BtnAutoReset = limitBtn("AUTO RESET", 0)
local BtnStop      = limitBtn("STOP",       0.5)

local function refreshLimitBtns()
    if S.limitAction == "AutoReset" then
        BtnAutoReset.BackgroundColor3 = COL.gold
        BtnAutoReset.TextColor3       = COL.bg
        BtnStop.BackgroundColor3      = COL.border
        BtnStop.TextColor3            = COL.muted
    else
        BtnStop.BackgroundColor3      = COL.red
        BtnStop.TextColor3            = Color3.new(1,1,1)
        BtnAutoReset.BackgroundColor3 = COL.border
        BtnAutoReset.TextColor3       = COL.muted
    end
end
refreshLimitBtns()

-- ── Status bar ─────────────────────────────────────────────────
local StatusBar = ni("Frame",{
    Size=UDim2.new(1,-20,0,24), Position=UDim2.new(0,10,0,403),
    BackgroundColor3=COL.panel, BorderSizePixel=0, ZIndex=11, Parent=Main,
})
addCorner(StatusBar,6)

local StatusLbl = ni("TextLabel",{
    Text="STATUS: IDLE", TextSize=10, Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,0,1,0), BackgroundTransparency=1,
    TextColor3=COL.muted, ZIndex=12, Parent=StatusBar,
})

-- ── Start / Stop button ────────────────────────────────────────
local StartBtn = ni("TextButton",{
    Text="START FARMING", TextSize=14, Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,-20,0,38), Position=UDim2.new(0,10,1,-48),
    BackgroundColor3=COL.green, TextColor3=COL.bg,
    AutoButtonColor=false, ZIndex=11, Parent=Main,
})
addCorner(StartBtn,10)

-- ── Hover effects ──────────────────────────────────────────────
local hoverBtns = {StartBtn,SpeedMinus,SpeedPlus,CollapseBtn,MiniBtn,BtnStand,BtnLay,BtnAutoReset,BtnStop}
for _,b in ipairs(hoverBtns) do
    b.MouseEnter:Connect(function() tw(b,{BackgroundTransparency=0.18},0.12):Play() end)
    b.MouseLeave:Connect(function() tw(b,{BackgroundTransparency=0},   0.12):Play() end)
end

-- ================================================================
--  DRAGGING
-- ================================================================
local dragActive, dragStart, dragOrigin = false, nil, nil
Header.InputBegan:Connect(function(i)
    if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
        dragActive = true
        dragStart  = i.Position
        dragOrigin = Main.Position
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if dragActive and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
        local d = i.Position - dragStart
        Main.Position = UDim2.new(dragOrigin.X.Scale, dragOrigin.X.Offset+d.X, dragOrigin.Y.Scale, dragOrigin.Y.Offset+d.Y)
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
        dragActive=false
    end
end)

-- ================================================================
--  COLLAPSE / EXPAND
-- ================================================================
local guiOpen = true

local function collapseGui()
    guiOpen = false
    tw(Main,{Size=UDim2.new(0,W,0,0),BackgroundTransparency=1},0.28,Enum.EasingStyle.Quart):Play()
    task.delay(0.28,function()
        Main.Visible = false
        MiniBtn.Visible = true
        tw(MiniBtn,{BackgroundTransparency=0},0.2):Play()
    end)
end

local function expandGui()
    guiOpen = true
    MiniBtn.Visible = false
    Main.Visible    = true
    Main.Size = UDim2.new(0,W,0,0)
    tw(Main,{Size=UDim2.new(0,W,0,H),BackgroundTransparency=0},0.3,Enum.EasingStyle.Back):Play()
end

CollapseBtn.MouseButton1Click:Connect(collapseGui)
MiniBtn.MouseButton1Click:Connect(expandGui)

-- intro animation
Main.BackgroundTransparency = 1
Main.Size = UDim2.new(0,W,0,0)
task.defer(function()
    tw(Main,{Size=UDim2.new(0,W,0,H),BackgroundTransparency=0},0.45,Enum.EasingStyle.Back):Play()
end)

-- ================================================================
--  UI UPDATES
-- ================================================================
local function setStatus(txt, col)
    StatusLbl.Text       = "STATUS: "..txt
    StatusLbl.TextColor3 = col or COL.muted
end

local function updateStats()
    CoinsLbl.Text  = tostring(S.coins)
    TotalLbl.Text  = tostring(S.totalCoins)
    ResetsLbl.Text = tostring(S.resets)
    local pct = math.clamp(S.coins / S.coinLimit, 0, 1)
    tw(ProgFill, {Size=UDim2.new(pct,0,1,0)}, 0.25):Play()
    ProgTxt.Text = S.coins.." / "..S.coinLimit
    local pc = pct < 0.55 and COL.green or pct < 0.85 and COL.gold or COL.red
    tw(ProgFill,{BackgroundColor3=pc},0.25):Play()
end

-- Session timer (runs always)
task.spawn(function()
    while true do
        local elapsed = os.clock() - S.sessionStart
        local h = math.floor(elapsed/3600)
        local m = math.floor((elapsed%3600)/60)
        local sec = math.floor(elapsed%60)
        TimeLbl.Text = string.format("%02d:%02d:%02d",h,m,sec)
        task.wait(1)
    end
end)

-- Title colour pulse
task.spawn(function()
    while true do
        tw(TitleLbl,{TextColor3=COL.gold},1.4,Enum.EasingStyle.Sine):Play() task.wait(1.4)
        tw(TitleLbl,{TextColor3=COL.accent},1.4,Enum.EasingStyle.Sine):Play() task.wait(1.4)
    end
end)

-- ================================================================
--  ANTI-AFK  (robust — catches Idled + periodic input)
-- ================================================================
local function startAntiAFK()
    if S.afkConn then S.afkConn:Disconnect() end
    -- Roblox fires Idled after ~2 min; we catch it and simulate input
    S.afkConn = LP.Idled:Connect(function()
        if getAntiAFK() then
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new())
        end
    end)
end

local function stopAntiAFK()
    if S.afkConn then S.afkConn:Disconnect(); S.afkConn=nil end
end

-- periodic jump to stay active (every 55 s)
task.spawn(function()
    while true do
        task.wait(55)
        if S.enabled and getAntiAFK() then
            local hum = getHum()
            if hum then hum.Jump = true end
        end
    end
end)

-- ================================================================
--  RENDERING TOGGLE
-- ================================================================
local function applyRender()
    if getNoRender() then
        -- Disable distance/quality to reduce load
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        workspace.StreamingEnabled = false
        -- Hide non-essential models (set LOD)
        for _,v in ipairs(workspace:GetDescendants()) do
            if v:IsA("BasePart") and not v:IsDescendantOf(getChar() or game) then
                v.LODLevel = 4
            end
        end
    else
        settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
    end
end
-- ================================================================
--  "COIN BAG FULL" DETECTION via PlayerGui
-- ================================================================
local function isBagFull()
    local pg = LP:FindFirstChild("PlayerGui")
    if not pg then return false end
    for _, gui in ipairs(pg:GetDescendants()) do
        if (gui:IsA("TextLabel") or gui:IsA("TextButton") or gui:IsA("Frame"))
        and gui.Visible then
            local txt = (gui:IsA("TextLabel") or gui:IsA("TextButton")) and gui.Text or ""
            if txt:lower():find("coin bag full") or txt:lower():find("bag full") then
                return true
            end
        end
    end
    return false
end

-- ================================================================
--  COIN FINDER  (universal — finds any coin-like Part in workspace)
--  Since coins spawn server-side, we look for Parts that:
--   • are NOT anchored (they simulate) OR are anchored but small
--   • have a yellow/gold colour, or name contains coin/gem/money
--   • are inside workspace (not in players / lighting)
-- ================================================================
local COIN_NAMES = {"coin","gem","money","cash","token","diamond","crystal","orb","bag"}

local function looksLikeCoin(part)
    if not part:IsA("BasePart") then return false end
    local n = part.Name:lower()
    for _, kw in ipairs(COIN_NAMES) do
        if n:find(kw) then return true end
    end
    -- Colour heuristic: gold/yellow/green
    local c = part.Color
    if c.R > 0.6 and c.G > 0.6 and c.B < 0.3 then return true end  -- yellow/gold
    if c.G > 0.6 and c.R < 0.4 and c.B < 0.4 then return true end  -- green
    -- Small spinning parts (common coin pattern)
    local sz = part.Size
    if sz.X < 5 and sz.Y < 5 and sz.Z < 5 and not part.Anchored then return true end
    return false
end

local function findNearestCoin()
    local root = getRoot()
    if not root then return nil end
    local rp    = root.Position
    local best, bestDist = nil, math.huge

    for _, obj in ipairs(workspace:GetDescendants()) do
        if looksLikeCoin(obj) then
            -- skip if inside any player character
            local inChar = false
            for _, pl in ipairs(Players:GetPlayers()) do
                if pl.Character and obj:IsDescendantOf(pl.Character) then
                    inChar = true break
                end
            end
            if not inChar then
                local d = (rp - obj.Position).Magnitude
                if d < bestDist then
                    bestDist = d
                    best     = obj
                end
            end
        end
    end
    return best, bestDist
end
-- ================================================================
--  DETECT ROUND STATE
--  "In round" = coins exist in workspace
--  "In lobby" = no coins exist
-- ================================================================
local function coinsExist()
    for _, obj in ipairs(workspace:GetDescendants()) do
        if looksLikeCoin(obj) then return true end
    end
    return false
end

-- ================================================================
--  LAY STYLE: flatten character like "—" hovering UNDER coin
-- ================================================================
local function applyLayStyle(coinPos)
    local root = getRoot()
    local hum  = getHum()
    if not root or not hum then return end

    hum.PlatformStand = true

    -- Position: directly under the coin, rotated flat (like a plank)
    local targetCF = CFrame.new(coinPos.X, coinPos.Y - 2.5, coinPos.Z)
                   * CFrame.Angles(math.rad(90), 0, 0)
    root.CFrame = targetCF
end

local function applyStandStyle()
    local hum = getHum()
    if hum then hum.PlatformStand = false end
end

-- ================================================================
--  RESET ACTION  (auto-reset: character resets to lobby)
-- ================================================================
local function doAutoReset()
    if S.resetting then return end
    S.resetting = true
    setStatus("RESETTING...", COL.gold)
    S.coins  = 0
    S.resets = S.resets + 1
    updateStats()
    -- Kill character to trigger respawn (most reliable reset method)
    local hum = getHum()
    if hum then
        hum.Health = 0
    end
    task.wait(0.5)
    S.resetting = false
end

-- ================================================================
--  MAIN FARM LOOP
-- ================================================================
local function stopFarm()
    S.enabled = false
    if S.farmConn then S.farmConn:Disconnect(); S.farmConn=nil end
    stopAntiAFK()
    applyStandStyle()
    local hum = getHum()
    if hum then hum.WalkSpeed = 16; hum.JumpPower = 50 end
    tw(StartBtn,{BackgroundColor3=COL.green},0.2):Play()
    StartBtn.Text = "START FARMING"
    setStatus("IDLE", COL.muted)
end

local function startFarm()
    S.enabled = true
    tw(StartBtn,{BackgroundColor3=COL.red},0.2):Play()
    StartBtn.Text = "STOP FARMING"
    setStatus("SEARCHING...", COL.accent)
    startAntiAFK()
    applyRender()

    -- Disconnect old loop if any
    if S.farmConn then S.farmConn:Disconnect() end

    S.farmConn = RunService.Heartbeat:Connect(function()
        if not S.enabled then return end

        -- Refresh character refs each tick (handles respawn)
        local char = getChar()
        local root = getRoot()
        local hum  = getHum()
        if not char or not root or not hum then
            setStatus("WAITING FOR CHARACTER...", COL.gold)
            return
        end

        -- Apply speed
        hum.WalkSpeed  = S.speed
        hum.JumpPower  = 50

        -- Check if bag is full
        if isBagFull() then
            if not S.bagFull then
                S.bagFull = true
                if S.limitAction == "AutoReset" then
                    doAutoReset()
                else
                    stopFarm()
                end
            end
            return
        else
            S.bagFull = false
        end

        -- Check if we're in round (coins exist on map)
        if not coinsExist() then
            applyStandStyle()
            hum.WalkSpeed = 16
            setStatus("WAITING FOR ROUND...", COL.gold)
            return
        end

        setStatus("FARMING", COL.green)

        -- Find nearest coin
        local coin, dist = findNearestCoin()
        if not coin then
            setStatus("NO COINS FOUND", COL.muted)
            return
        end

        local coinPos = coin.Position

        if S.style == "Lay" then
            -- LAY mode: teleport directly under coin and hover flat
            applyLayStyle(coinPos)

            -- Collect: move root into coin hitbox
            if dist < 8 then
                local touchCF = CFrame.new(coinPos.X, coinPos.Y, coinPos.Z)
                root.CFrame = touchCF
            else
                -- Move toward coin
                local dir = (coinPos - root.Position).Unit
                root.CFrame = CFrame.new(root.Position + dir * math.min(S.speed * 0.15, dist))
                              * CFrame.Angles(math.rad(90), 0, 0)
            end
        else
            -- STAND mode: walk normally toward coin
            applyStandStyle()
            if dist > 2.5 then
                local dir = (coinPos - root.Position)
                dir = Vector3.new(dir.X, 0, dir.Z).Unit
                root.CFrame = CFrame.new(root.Position + dir * math.min(S.speed * 0.12, dist))
                              * CFrame.new(0, 0, 0)
                              * CFrame.fromEulerAnglesYXZ(0, math.atan2(-dir.X, -dir.Z), 0)
            end
        end
    end)
end

-- ================================================================
--  COIN COUNT: detect when a coin disappears (was collected)
-- ================================================================
-- We track coin count changes to update the stat
local prevCoinCount = 0
task.spawn(function()
    while true do
        task.wait(0.3)
        if S.enabled then
            local count = 0
            for _, obj in ipairs(workspace:GetDescendants()) do
                if looksLikeCoin(obj) then count = count + 1 end
            end
            if count < prevCoinCount then
                -- A coin disappeared — we assume we collected it
                local diff = prevCoinCount - count
                S.coins      = S.coins      + diff
                S.totalCoins = S.totalCoins + diff
                updateStats()
            end
            prevCoinCount = count
        end
    end
end)

-- ================================================================
--  RESPAWN HANDLER — restart farm after death
-- ================================================================
LP.CharacterAdded:Connect(function(char)
    -- Wait for character to fully load
    char:WaitForChild("HumanoidRootPart", 10)
    char:WaitForChild("Humanoid", 10)
    task.wait(1.5)

    -- If farming was active, restart the loop
    if S.enabled then
        -- Reconnect farm loop with new character
        if S.farmConn then S.farmConn:Disconnect(); S.farmConn=nil end
        startFarm()
    end
end)

-- ================================================================
--  BUTTON LOGIC
-- ================================================================
StartBtn.MouseButton1Click:Connect(function()
    if S.enabled then
        stopFarm()
    else
        startFarm()
    end
end)

SpeedMinus.MouseButton1Click:Connect(function()
    S.speed = math.max(16, S.speed - 1)
    SpeedVal.Text = tostring(S.speed)
    tw(SpeedVal,{TextColor3=COL.gold},0.1):Play()
    tw(SpeedVal,{TextColor3=COL.accent},0.25):Play()
end)

SpeedPlus.MouseButton1Click:Connect(function()
    S.speed = math.min(30, S.speed + 1)
    SpeedVal.Text = tostring(S.speed)
    tw(SpeedVal,{TextColor3=COL.gold},0.1):Play()
    tw(SpeedVal,{TextColor3=COL.accent},0.25):Play()
end)

BtnStand.MouseButton1Click:Connect(function()
    S.style = "Stand"
    refreshStyleBtns()
end)

BtnLay.MouseButton1Click:Connect(function()
    S.style = "Lay"
    refreshStyleBtns()
end)

BtnAutoReset.MouseButton1Click:Connect(function()
    S.limitAction = "AutoReset"
    refreshLimitBtns()
end)

BtnStop.MouseButton1Click:Connect(function()
    S.limitAction = "Stop"
    refreshLimitBtns()
end)

-- Render toggle callback
local origNoRender = getNoRender
togNoRender = (function()
    local orig = togNoRender
    return function()
        orig()
        applyRender()
    end
end)()

-- ================================================================
updateStats()
print("[CoinFarmerPro v2] Loaded. GUI is persistent and survives resets.")
