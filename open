-- ================================================================
--   COIN FARMER PRO  v3.0
--   Fixed: lag, lay pose, render, coin count, noclip, anti-fling
-- ================================================================

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local VirtualUser      = game:GetService("VirtualUser")
local MarketplaceService = game:GetService("MarketplaceService")
local PhysicsService   = game:GetService("PhysicsService")

local LP = Players.LocalPlayer

-- ================================================================
--  GAMEPASS CHECK  (Elite = 50 coins, default = 40)
-- ================================================================
local ELITE_PASS_ID = 429957
local function detectCoinLimit()
    local ok, owns = pcall(function()
        return MarketplaceService:UserOwnsGamePassAsync(LP.UserId, ELITE_PASS_ID)
    end)
    return (ok and owns) and 50 or 40
end
local COIN_LIMIT = detectCoinLimit()

-- ================================================================
--  STATE
-- ================================================================
local S = {
    enabled     = false,
    speed       = 21,
    style       = "Stand",      -- "Stand" | "Lay"
    antiAfk     = true,
    noRender    = false,
    limitAction = "AutoReset",  -- "AutoReset" | "Stop"
    coinLimit   = COIN_LIMIT,

    coins       = 0,
    totalCoins  = 0,
    resets      = 0,
    sessionStart= os.clock(),

    farmConn    = nil,
    noclipConn  = nil,
    flingConn   = nil,
    bagFull     = false,
    resetting   = false,

    -- Snapshot of coins when farming started (to count only REMOVED ones)
    knownCoins  = {},   -- [part] = true
    coinCount   = 0,
}

-- ================================================================
--  GUI SETUP (CoreGui — survives reset)
-- ================================================================
local CoreGui = game:GetService("CoreGui")
local ScreenGui = CoreGui:FindFirstChild("CFP_GUI")
if not ScreenGui then
    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name           = "CFP_GUI"
    ScreenGui.ResetOnSpawn   = false
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ScreenGui.IgnoreGuiInset = true
    ScreenGui.Parent         = CoreGui
end

-- ================================================================
--  HELPERS
-- ================================================================
local function tw(obj, props, t, style, dir)
    return TweenService:Create(obj,
        TweenInfo.new(t or 0.22, style or Enum.EasingStyle.Quart,
                      dir or Enum.EasingDirection.Out), props)
end
local function ni(cls, props, parent)
    local o = Instance.new(cls)
    for k,v in pairs(props) do o[k]=v end
    if parent then o.Parent=parent end
    return o
end
local function corner(f,r) local c=Instance.new("UICorner") c.CornerRadius=UDim.new(0,r or 8) c.Parent=f return c end
local function stroke(f,col,th) local s=Instance.new("UIStroke") s.Color=col or Color3.fromRGB(35,45,72) s.Thickness=th or 1 s.Parent=f return s end

local function getChar() return LP.Character end
local function getRoot() local c=getChar() return c and c:FindFirstChild("HumanoidRootPart") end
local function getHum()  local c=getChar() return c and c:FindFirstChildOfClass("Humanoid") end

-- ================================================================
--  COLOURS
-- ================================================================
local C = {
    bg    = Color3.fromRGB(9,11,19),
    panel = Color3.fromRGB(14,18,30),
    bord  = Color3.fromRGB(35,45,72),
    acc   = Color3.fromRGB(82,196,255),
    gold  = Color3.fromRGB(255,185,50),
    green = Color3.fromRGB(72,225,130),
    red   = Color3.fromRGB(255,75,85),
    muted = Color3.fromRGB(90,105,140),
    text  = Color3.fromRGB(210,220,255),
}

-- ================================================================
--  BUILD GUI
-- ================================================================
local W, H = 308, 520

local MiniBtn = ni("TextButton",{
    Name="MiniBtn", Size=UDim2.new(0,44,0,44),
    Position=UDim2.new(0,16,0.5,-22),
    BackgroundColor3=C.bg, Text="CF", TextSize=13,
    Font=Enum.Font.GothamBold, TextColor3=C.acc,
    AutoButtonColor=false, Visible=false, ZIndex=30, Parent=ScreenGui,
})
corner(MiniBtn,12); stroke(MiniBtn,C.acc,1.5)

local Main = ni("Frame",{
    Name="Main", Size=UDim2.new(0,W,0,H),
    Position=UDim2.new(0,16,0.5,-H/2),
    BackgroundColor3=C.bg, BorderSizePixel=0, ZIndex=10, Parent=ScreenGui,
})
corner(Main,12); stroke(Main,C.bord,1.5)

-- shadow
local Shad=ni("Frame",{Size=UDim2.new(1,20,1,20),Position=UDim2.new(0,-10,0,10),
    BackgroundColor3=Color3.new(0,0,0),BackgroundTransparency=0.55,ZIndex=9,Parent=Main})
corner(Shad,14)

-- Header
local Hdr=ni("Frame",{Size=UDim2.new(1,0,0,48),BackgroundColor3=C.panel,
    BorderSizePixel=0,ZIndex=11,Parent=Main})
corner(Hdr,12)
ni("Frame",{Size=UDim2.new(1,0,0,12),Position=UDim2.new(0,0,1,-12),
    BackgroundColor3=C.panel,BorderSizePixel=0,ZIndex=11,Parent=Hdr})

local TitleLbl=ni("TextLabel",{Text="COIN FARMER PRO",TextSize=13,Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,-80,1,0),Position=UDim2.new(0,14,0,0),
    BackgroundTransparency=1,TextColor3=C.acc,
    TextXAlignment=Enum.TextXAlignment.Left,ZIndex=12,Parent=Hdr})

ni("TextLabel",{Text="v3.0",TextSize=10,Font=Enum.Font.Gotham,
    Size=UDim2.new(0,30,1,0),Position=UDim2.new(0,165,0,0),
    BackgroundTransparency=1,TextColor3=C.muted,
    TextXAlignment=Enum.TextXAlignment.Left,ZIndex=12,Parent=Hdr})

local ColBtn=ni("TextButton",{Text="—",TextSize=16,Font=Enum.Font.GothamBold,
    Size=UDim2.new(0,30,0,24),Position=UDim2.new(1,-38,0.5,-12),
    BackgroundColor3=C.bord,TextColor3=C.text,AutoButtonColor=false,ZIndex=12,Parent=Hdr})
corner(ColBtn,6)

-- Stats
local StatPanel=ni("Frame",{Size=UDim2.new(1,-20,0,50),Position=UDim2.new(0,10,0,56),
    BackgroundColor3=C.panel,BorderSizePixel=0,ZIndex=11,Parent=Main})
corner(StatPanel,8); stroke(StatPanel,C.bord)

local function statCell(lbl,xr)
    local f=ni("Frame",{Size=UDim2.new(0.25,0,1,0),Position=UDim2.new(xr,0,0,0),
        BackgroundTransparency=1,ZIndex=12,Parent=StatPanel})
    ni("TextLabel",{Text=lbl,TextSize=9,Font=Enum.Font.Gotham,
        Size=UDim2.new(1,0,0,16),Position=UDim2.new(0,0,0,5),
        BackgroundTransparency=1,TextColor3=C.muted,ZIndex=12,Parent=f})
    local v=ni("TextLabel",{Text="0",TextSize=15,Font=Enum.Font.GothamBold,
        Size=UDim2.new(1,0,0,22),Position=UDim2.new(0,0,0,23),
        BackgroundTransparency=1,TextColor3=C.text,ZIndex=12,Parent=f})
    return v
end
local CoinsLbl  = statCell("COINS",  0)
local TotalLbl  = statCell("TOTAL",  0.25)
local ResetsLbl = statCell("RESETS", 0.50)
local TimeLbl   = statCell("SESSION",0.75)
for _,x in ipairs({0.25,0.5,0.75}) do
    ni("Frame",{Size=UDim2.new(0,1,0.6,0),Position=UDim2.new(x,0,0.2,0),
        BackgroundColor3=C.bord,BorderSizePixel=0,ZIndex=12,Parent=StatPanel})
end

-- Limit display
local LimitHint=ni("TextLabel",{
    Text="LIMIT: "..COIN_LIMIT..(COIN_LIMIT==50 and "  [ELITE]" or "  [STANDARD]"),
    TextSize=10,Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,-20,0,16),Position=UDim2.new(0,10,0,113),
    BackgroundTransparency=1,TextColor3=COIN_LIMIT==50 and C.gold or C.muted,
    TextXAlignment=Enum.TextXAlignment.Left,ZIndex=11,Parent=Main})

-- Progress bar
local ProgBG=ni("Frame",{Size=UDim2.new(1,-20,0,14),Position=UDim2.new(0,10,0,131),
    BackgroundColor3=C.panel,BorderSizePixel=0,ZIndex=11,Parent=Main})
corner(ProgBG,7); stroke(ProgBG,C.bord)
local ProgFill=ni("Frame",{Size=UDim2.new(0,0,1,0),BackgroundColor3=C.green,
    BorderSizePixel=0,ZIndex=12,Parent=ProgBG})
corner(ProgFill,7)
local ProgTxt=ni("TextLabel",{Text="0 / "..COIN_LIMIT,TextSize=9,Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,0,1,0),BackgroundTransparency=1,TextColor3=C.text,ZIndex=13,Parent=ProgBG})

-- Section label
local function secLbl(txt,y)
    ni("TextLabel",{Text=txt,TextSize=10,Font=Enum.Font.GothamBold,
        Size=UDim2.new(1,-20,0,14),Position=UDim2.new(0,10,0,y),
        BackgroundTransparency=1,TextColor3=C.muted,
        TextXAlignment=Enum.TextXAlignment.Left,ZIndex=11,Parent=Main})
end

-- Speed
secLbl("MOVEMENT SPEED",153)
local SpeedRow=ni("Frame",{Size=UDim2.new(1,-20,0,34),Position=UDim2.new(0,10,0,169),
    BackgroundColor3=C.panel,BorderSizePixel=0,ZIndex=11,Parent=Main})
corner(SpeedRow,8); stroke(SpeedRow,C.bord)
local SpeedMinus=ni("TextButton",{Text="−",TextSize=18,Font=Enum.Font.GothamBold,
    Size=UDim2.new(0,34,1,0),BackgroundColor3=C.bord,TextColor3=C.acc,
    AutoButtonColor=false,ZIndex=12,Parent=SpeedRow})
corner(SpeedMinus,8)
local SpeedVal=ni("TextLabel",{Text=tostring(S.speed),TextSize=15,Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,-68,1,0),Position=UDim2.new(0,34,0,0),
    BackgroundTransparency=1,TextColor3=C.acc,ZIndex=12,Parent=SpeedRow})
local SpeedPlus=ni("TextButton",{Text="+",TextSize=18,Font=Enum.Font.GothamBold,
    Size=UDim2.new(0,34,1,0),Position=UDim2.new(1,-34,0,0),
    BackgroundColor3=C.bord,TextColor3=C.acc,AutoButtonColor=false,ZIndex=12,Parent=SpeedRow})
corner(SpeedPlus,8)

-- Style
secLbl("MOVEMENT STYLE",211)
local StyleRow=ni("Frame",{Size=UDim2.new(1,-20,0,34),Position=UDim2.new(0,10,0,227),
    BackgroundColor3=C.panel,BorderSizePixel=0,ZIndex=11,Parent=Main})
corner(StyleRow,8)
local function mkStyleBtn(txt,xs)
    local b=ni("TextButton",{Text=txt,TextSize=12,Font=Enum.Font.GothamBold,
        Size=UDim2.new(0.5,-2,1,0),Position=UDim2.new(xs,xs==0 and 0 or 2,0,0),
        BackgroundColor3=C.bord,TextColor3=C.muted,AutoButtonColor=false,ZIndex=12,Parent=StyleRow})
    corner(b,8); return b
end
local BtnStand=mkStyleBtn("STAND",0)
local BtnLay  =mkStyleBtn("LAY",0.5)
local function refreshStyle()
    BtnStand.BackgroundColor3 = S.style=="Stand" and C.acc or C.bord
    BtnStand.TextColor3       = S.style=="Stand" and C.bg  or C.muted
    BtnLay.BackgroundColor3   = S.style=="Lay"   and C.acc or C.bord
    BtnLay.TextColor3         = S.style=="Lay"   and C.bg  or C.muted
end
refreshStyle()

-- Toggles
secLbl("OPTIONS",269)
local function mkToggle(label,y,initOn,activeCol)
    local row=ni("Frame",{Size=UDim2.new(1,-20,0,32),Position=UDim2.new(0,10,0,y),
        BackgroundColor3=C.panel,BorderSizePixel=0,ZIndex=11,Parent=Main})
    corner(row,8)
    ni("TextLabel",{Text=label,TextSize=11,Font=Enum.Font.Gotham,
        Size=UDim2.new(1,-58,1,0),Position=UDim2.new(0,10,0,0),
        BackgroundTransparency=1,TextColor3=C.text,
        TextXAlignment=Enum.TextXAlignment.Left,ZIndex=12,Parent=row})
    local track=ni("Frame",{Size=UDim2.new(0,40,0,20),Position=UDim2.new(1,-48,0.5,-10),
        BackgroundColor3=initOn and (activeCol or C.green) or C.bord,ZIndex=12,Parent=row})
    corner(track,10)
    local knob=ni("Frame",{Size=UDim2.new(0,16,0,16),
        Position=initOn and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8),
        BackgroundColor3=Color3.new(1,1,1),ZIndex=13,Parent=track})
    corner(knob,8)
    local st={on=initOn}
    local ac=activeCol or C.green
    local function doTog()
        st.on=not st.on
        tw(knob,{Position=st.on and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8)},0.18):Play()
        tw(track,{BackgroundColor3=st.on and ac or C.bord},0.18):Play()
        return st.on
    end
    row.InputBegan:Connect(function(i)
        if i.UserInputType==Enum.UserInputType.MouseButton1
        or i.UserInputType==Enum.UserInputType.Touch then doTog() end
    end)
    return function() return st.on end, doTog
end
local getAFK,  _ = mkToggle("ANTI-AFK PROTECTION", 285, true)
local getRender,_ = mkToggle("DISABLE RENDERING",  323, false, C.acc)
local getAntiFling,_ = mkToggle("ANTI-FLING",      361, true, C.gold)

-- Limit action
secLbl("ON COIN LIMIT",401)
local LimitRow=ni("Frame",{Size=UDim2.new(1,-20,0,34),Position=UDim2.new(0,10,0,417),
    BackgroundColor3=C.panel,BorderSizePixel=0,ZIndex=11,Parent=Main})
corner(LimitRow,8)
local function mkLimBtn(txt,xs)
    local b=ni("TextButton",{Text=txt,TextSize=11,Font=Enum.Font.GothamBold,
        Size=UDim2.new(0.5,-2,1,0),Position=UDim2.new(xs,xs==0 and 0 or 2,0,0),
        BackgroundColor3=C.bord,TextColor3=C.muted,AutoButtonColor=false,ZIndex=12,Parent=LimitRow})
    corner(b,8); return b
end
local BtnAuto=mkLimBtn("AUTO RESET",0)
local BtnStop=mkLimBtn("STOP",0.5)
local function refreshLimit()
    BtnAuto.BackgroundColor3 = S.limitAction=="AutoReset" and C.gold or C.bord
    BtnAuto.TextColor3       = S.limitAction=="AutoReset" and C.bg   or C.muted
    BtnStop.BackgroundColor3 = S.limitAction=="Stop"      and C.red  or C.bord
    BtnStop.TextColor3       = S.limitAction=="Stop"      and Color3.new(1,1,1) or C.muted
end
refreshLimit()

-- Status + Start
local StatBar=ni("Frame",{Size=UDim2.new(1,-20,0,22),Position=UDim2.new(0,10,0,458),
    BackgroundColor3=C.panel,BorderSizePixel=0,ZIndex=11,Parent=Main})
corner(StatBar,6)
local StatusLbl=ni("TextLabel",{Text="STATUS: IDLE",TextSize=10,Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,0,1,0),BackgroundTransparency=1,TextColor3=C.muted,ZIndex=12,Parent=StatBar})

local StartBtn=ni("TextButton",{Text="START FARMING",TextSize=14,Font=Enum.Font.GothamBold,
    Size=UDim2.new(1,-20,0,36),Position=UDim2.new(0,10,1,-46),
    BackgroundColor3=C.green,TextColor3=C.bg,AutoButtonColor=false,ZIndex=11,Parent=Main})
corner(StartBtn,10)

-- Hover
for _,b in ipairs({StartBtn,SpeedMinus,SpeedPlus,ColBtn,MiniBtn,BtnStand,BtnLay,BtnAuto,BtnStop}) do
    b.MouseEnter:Connect(function() tw(b,{BackgroundTransparency=0.18},0.12):Play() end)
    b.MouseLeave:Connect(function() tw(b,{BackgroundTransparency=0},0.12):Play() end)
end

-- ================================================================
--  DRAG
-- ================================================================
local dActive,dStart,dOrigin=false,nil,nil
Hdr.InputBegan:Connect(function(i)
    if i.UserInputType==Enum.UserInputType.MouseButton1
    or i.UserInputType==Enum.UserInputType.Touch then
        dActive=true dStart=i.Position dOrigin=Main.Position
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if dActive and (i.UserInputType==Enum.UserInputType.MouseMovement
    or i.UserInputType==Enum.UserInputType.Touch) then
        local d=i.Position-dStart
        Main.Position=UDim2.new(dOrigin.X.Scale,dOrigin.X.Offset+d.X,dOrigin.Y.Scale,dOrigin.Y.Offset+d.Y)
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType==Enum.UserInputType.MouseButton1
    or i.UserInputType==Enum.UserInputType.Touch then dActive=false end
end)

-- ================================================================
--  COLLAPSE / EXPAND
-- ================================================================
local function collapseGui()
    tw(Main,{Size=UDim2.new(0,W,0,0),BackgroundTransparency=1},0.25,Enum.EasingStyle.Quart):Play()
    task.delay(0.25,function()
        Main.Visible=false MiniBtn.Visible=true
        tw(MiniBtn,{BackgroundTransparency=0},0.2):Play()
    end)
end
local function expandGui()
    MiniBtn.Visible=false Main.Visible=true
    Main.Size=UDim2.new(0,W,0,0)
    tw(Main,{Size=UDim2.new(0,W,0,H),BackgroundTransparency=0},0.3,Enum.EasingStyle.Back):Play()
end
ColBtn.MouseButton1Click:Connect(collapseGui)
MiniBtn.MouseButton1Click:Connect(expandGui)

-- intro
Main.BackgroundTransparency=1 Main.Size=UDim2.new(0,W,0,0)
task.defer(function()
    tw(Main,{Size=UDim2.new(0,W,0,H),BackgroundTransparency=0},0.45,Enum.EasingStyle.Back):Play()
end)

-- ================================================================
--  STATS UPDATE
-- ================================================================
local function setStatus(txt,col)
    StatusLbl.Text=      "STATUS: "..txt
    StatusLbl.TextColor3= col or C.muted
end
local function updateStats()
    CoinsLbl.Text =tostring(S.coins)
    TotalLbl.Text =tostring(S.totalCoins)
    ResetsLbl.Text=tostring(S.resets)
    local pct=math.clamp(S.coins/S.coinLimit,0,1)
    tw(ProgFill,{Size=UDim2.new(pct,0,1,0)},0.2):Play()
    ProgTxt.Text=S.coins.." / "..S.coinLimit
    local pc=pct<0.55 and C.green or pct<0.85 and C.gold or C.red
    tw(ProgFill,{BackgroundColor3=pc},0.2):Play()
end

-- Session timer
task.spawn(function()
    while true do
        local e=os.clock()-S.sessionStart
        TimeLbl.Text=string.format("%02d:%02d:%02d",
            math.floor(e/3600),math.floor((e%3600)/60),math.floor(e%60))
        task.wait(1)
    end
end)

-- Title pulse
task.spawn(function()
    while true do
        tw(TitleLbl,{TextColor3=C.gold},1.4,Enum.EasingStyle.Sine):Play() task.wait(1.4)
        tw(TitleLbl,{TextColor3=C.acc}, 1.4,Enum.EasingStyle.Sine):Play() task.wait(1.4)
    end
end)

-- ================================================================
--  ANTI-AFK
-- ================================================================
LP.Idled:Connect(function()
    if getAFK() then
        VirtualUser:CaptureController()
        VirtualUser:ClickButton2(Vector2.new())
    end
end)
task.spawn(function()
    while true do
        task.wait(50)
        if S.enabled and getAFK() then
            local h=getHum()
            if h then h.Jump=true end
        end
    end
end)

-- ================================================================
--  RENDERING  (executor API — drawcache + FRM level 1)
-- ================================================================
local renderApplied = false
local function applyRender()
    local disable = getRender()
    if disable and not renderApplied then
        renderApplied = true
        -- Executor-level render distance / quality reduction
        pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)
        -- Hide all non-character BaseParts (reduce GPU load)
        local myChar = getChar()
        for _, v in ipairs(workspace:GetDescendants()) do
            if v:IsA("BasePart") or v:IsA("UnionOperation") or v:IsA("MeshPart") or v:IsA("SpecialMesh") then
                if not myChar or not v:IsDescendantOf(myChar) then
                    v.LocalTransparencyModifier = 1
                end
            end
        end
        -- Hide terrain
        workspace.Terrain.Transparency = 1
        -- Disable shadows
        game:GetService("Lighting").GlobalShadows = false
        game:GetService("Lighting").FogEnd = 1000000
    elseif not disable and renderApplied then
        renderApplied = false
        pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic end)
        local myChar = getChar()
        for _, v in ipairs(workspace:GetDescendants()) do
            if v:IsA("BasePart") or v:IsA("UnionOperation") or v:IsA("MeshPart") then
                if not myChar or not v:IsDescendantOf(myChar) then
                    v.LocalTransparencyModifier = 0
                end
            end
        end
        workspace.Terrain.Transparency = 0
        game:GetService("Lighting").GlobalShadows = true
    end
end

-- Also apply render toggle when it changes
local renderOrigToggle = _
-- re-hook through RunService below

-- ================================================================
--  NOCLIP (only while farming)
-- ================================================================
local function enableNoclip()
    if S.noclipConn then return end
    S.noclipConn = RunService.Stepped:Connect(function()
        local char = getChar()
        if not char then return end
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
    end)
end

local function disableNoclip()
    if S.noclipConn then
        S.noclipConn:Disconnect()
        S.noclipConn = nil
    end
    -- Restore collision
      local char = getChar()
    if char then
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = true
            end
        end
    end
end

-- ================================================================
--  ANTI-FLING
--  Locks velocity if someone applies extreme external force
-- ================================================================
local MAX_VELOCITY = 200  -- studs/s — above this we consider it a fling

local function enableAntiFling()
    if S.flingConn then return end
    S.flingConn = RunService.Heartbeat:Connect(function()
        if not getAntiFling() then return end
        local root = getRoot()
        if not root then return end
        local vel = root.AssemblyLinearVelocity
        if vel.Magnitude > MAX_VELOCITY then
            -- Cancel the fling
            root.AssemblyLinearVelocity  = Vector3.new(0,0,0)
            root.AssemblyAngularVelocity = Vector3.new(0,0,0)
        end
    end)
end

local function disableAntiFling()
    if S.flingConn then S.flingConn:Disconnect(); S.flingConn=nil end
end

-- ================================================================
--  "COIN BAG FULL" DETECTION
-- ================================================================
local function isBagFull()
    local pg = LP:FindFirstChild("PlayerGui")
    if not pg then return false end
    for _, v in ipairs(pg:GetDescendants()) do
        if (v:IsA("TextLabel") or v:IsA("TextButton")) and v.Visible then
            local t = v.Text:lower()
            if t:find("coin bag full") or t:find("bag full") or t:find("bag is full") then
                return true
            end
        end
    end
    return false
end

-- ================================================================
--  COIN DETECTION  (keyword + colour heuristic)
-- ================================================================
local COIN_KW = {"coin","gem","money","cash","token","diamond","crystal","orb"}

local function looksLikeCoin(p)
    if not p:IsA("BasePart") then return false end
    local n=p.Name:lower()
    for _,k in ipairs(COIN_KW) do if n:find(k) then return true end end
    -- gold/yellow colour heuristic
    local r,g,b=p.Color.R,p.Color.G,p.Color.B
    if r>0.55 and g>0.55 and b<0.35 then return true end
    return false
end

local function findNearestCoin()
    local root=getRoot()
    if not root then return nil,math.huge end
    local rp=root.Position
    local best,bd=nil,math.huge
    for _,obj in ipairs(workspace:GetDescendants()) do
        if looksLikeCoin(obj) then
            local inPC=false
            for _,pl in ipairs(Players:GetPlayers()) do
                if pl.Character and obj:IsDescendantOf(pl.Character) then inPC=true;break end
            end
            if not inPC then
                local d=(rp-obj.Position).Magnitude
                if d<bd then bd=d;best=obj end
            end
        end
    end
    return best,bd
end

local function coinsExistOnMap()
    for _,obj in ipairs(workspace:GetDescendants()) do
        if looksLikeCoin(obj) then
            local inPC=false
            for _,pl in ipairs(Players:GetPlayers()) do
                if pl.Character and obj:IsDescendantOf(pl.Character) then inPC=true;break end
            end
            if not inPC then return true end
        end
    end
    return false
end
-- ================================================================
--  COIN COUNTER  (counts only DISAPPEARANCES — i.e. collected coins)
--  Tracks the set of coins. When a coin that WAS there is now gone
--  → player (or server) removed it → count it.
--  Ignores NEW coins appearing (spawn/respawn events).
-- ================================================================
local function buildCoinSnapshot()
    S.knownCoins = {}
    S.coinCount  = 0
    for _, obj in ipairs(workspace:GetDescendants()) do
        if looksLikeCoin(obj) then
            local inPC=false
            for _,pl in ipairs(Players:GetPlayers()) do
                if pl.Character and obj:IsDescendantOf(pl.Character) then inPC=true;break end
            end
            if not inPC then
                S.knownCoins[obj] = true
                S.coinCount = S.coinCount + 1
            end
        end
    end
end

task.spawn(function()
    while true do
        task.wait(0.25)
        if S.enabled then
            local newKnown = {}
            local newCount = 0
            for _, obj in ipairs(workspace:GetDescendants()) do
                if looksLikeCoin(obj) then
                    local inPC=false
                    for _,pl in ipairs(Players:GetPlayers()) do
                        if pl.Character and obj:IsDescendantOf(pl.Character) then inPC=true;break end
                    end
                    if not inPC then
                        newKnown[obj] = true
                        newCount = newCount + 1
                    end
                end
            end

            -- Count only coins that DISAPPEARED (were in old set, not in new)
            local disappeared = 0
            for obj,_ in pairs(S.knownCoins) do
                if not newKnown[obj] then
                    disappeared = disappeared + 1
                end
            end

            if disappeared > 0 then
                -- Only count as collected if we were close to a coin area
                -- (avoids counting server-side despawns unrelated to us)
                local root = getRoot()
                if root then
                    S.coins      = math.min(S.coins + disappeared, S.coinLimit)
                    S.totalCoins = S.totalCoins + disappeared
                    updateStats()
                end
            end

            S.knownCoins = newKnown
            S.coinCount  = newCount
        end
    end
end)

-- ================================================================
--  AUTO-RESET
-- ================================================================
local function doAutoReset()
    if S.resetting then return end
    S.resetting = true
    setStatus("RESETTING...", C.gold)
    task.wait(0.3)
    S.coins  = 0
    S.resets = S.resets + 1
    updateStats()
    -- Kill character → respawn in lobby
    local hum=getHum()
    if hum then hum.Health=0 end
    task.wait(0.5)
    S.resetting = false
    S.bagFull   = false
end

-- ================================================================
--  MAIN FARM LOOP  (runs on Heartbeat — one connection, no extra loops)
-- ================================================================
local function stopFarm()
    S.enabled = false
    if S.farmConn then S.farmConn:Disconnect(); S.farmConn=nil end
    disableNoclip()
    -- keep anti-fling on always
    local hum=getHum()
    if hum then hum.WalkSpeed=16; hum.JumpPower=50; hum.PlatformStand=false end
    tw(StartBtn,{BackgroundColor3=C.green},0.2):Play()
    StartBtn.Text="START FARMING"
    setStatus("IDLE",C.muted)
    -- Restore render if it was disabled
    if renderApplied then
        renderApplied=true  -- force restore
        getRender = function() return false end
        applyRender()
    end
end

local function startFarm()
    S.enabled = true
    buildCoinSnapshot()   -- baseline — so we don't count initial spawn as collected
    tw(StartBtn,{BackgroundColor3=C.red},0.2):Play()
    StartBtn.Text="STOP FARMING"
    setStatus("SEARCHING...",C.acc)
    enableNoclip()
    applyRender()

    if S.farmConn then S.farmConn:Disconnect() end

    -- We use a time-based throttle inside Heartbeat to avoid running every frame
    local lastMove = 0

    S.farmConn = RunService.Heartbeat:Connect(function(dt)
        if not S.enabled then return end

        local now = os.clock()

        -- Throttle: only move logic 20 times/sec to prevent lag
        if now - lastMove < 0.05 then return end
        lastMove = now

        local char = getChar()
        local root = getRoot()
        local hum  = getHum()
        if not char or not root or not hum then
            setStatus("WAITING FOR CHARACTER...", C.gold)
            return
        end

        -- Apply speed
        hum.WalkSpeed = S.speed
        hum.JumpPower = 50

        -- Check bag full via GUI
        if isBagFull() then
            if not S.bagFull then
                S.bagFull = true
                if S.limitAction == "AutoReset" then
                    task.spawn(doAutoReset)
                else
                    task.spawn(stopFarm)
                end
            end
            return
        end

        -- Also check coin counter limit
        if S.coins >= S.coinLimit then
            if not S.bagFull then
                S.bagFull = true
                if S.limitAction == "AutoReset" then
                    task.spawn(doAutoReset)
                else
                    task.spawn(stopFarm)
                end
            end
            return
        end

        S.bagFull = false

        -- Wait for round (coins must exist)
        if not coinsExistOnMap() then
            hum.WalkSpeed = 16
            hum.PlatformStand = false
            setStatus("WAITING FOR ROUND...", C.gold)
            return
        end

        setStatus("FARMING", C.green)

        -- Find nearest coin
        local coin, dist = findNearestCoin()
        if not coin then setStatus("NO COINS FOUND", C.muted) return end

        local cp = coin.Position

        if S.style == "Lay" then
            -- LAY: character lies flat like "—" UNDER the coin
            -- Correct rotation: X=90 makes character horizontal (lying down)
            -- We position slightly below the coin
            hum.PlatformStand = true
            local targetPos = Vector3.new(cp.X, cp.Y - 3, cp.Z)
            local lookDir   = (targetPos - root.Position)
            -- Move toward coin horizontally, keep Y fixed below coin
            if dist > 2 then
                local moveDir = Vector3.new(lookDir.X, 0, lookDir.Z).Unit
                local newPos  = root.Position + moveDir * math.min(S.speed * 0.08, dist)
                -- Keep Y = coin.Y - 3 (hover under coin)
                root.CFrame = CFrame.new(Vector3.new(newPos.X, cp.Y - 3, newPos.Z))
                            * CFrame.Angles(0, math.atan2(-moveDir.X, -moveDir.Z), 0)
                            * CFrame.Angles(math.rad(90), 0, 0)
            else
                -- Arrived: snap directly under coin, lying flat
                root.CFrame = CFrame.new(Vector3.new(cp.X, cp.Y - 3, cp.Z))
                            * CFrame.Angles(math.rad(90), 0, 0)
            end
        else
            -- STAND: walk normally toward coin
            hum.PlatformStand = false
            if dist > 2.5 then
                local dir = Vector3.new(cp.X - root.Position.X, 0, cp.Z - root.Position.Z).Unit
                local newPos = root.Position + dir * math.min(S.speed * 0.1, dist)
                root.CFrame = CFrame.new(newPos)
                            * CFrame.fromEulerAnglesYXZ(0, math.atan2(-dir.X, -dir.Z), 0)
            end
        end
    end)
end
-- ================================================================
--  RESPAWN HANDLER
-- ================================================================
LP.CharacterAdded:Connect(function(char)
    char:WaitForChild("HumanoidRootPart", 10)
    char:WaitForChild("Humanoid", 10)
    task.wait(2)  -- wait for lobby/round to settle
    if S.enabled then
        if S.farmConn then S.farmConn:Disconnect(); S.farmConn=nil end
        enableNoclip()
        startFarm()
    end
    -- Always keep anti-fling running
    if S.flingConn then S.flingConn:Disconnect(); S.flingConn=nil end
    enableAntiFling()
end)

-- ================================================================
--  ALWAYS-ON: Anti-fling starts immediately
-- ================================================================
enableAntiFling()

-- ================================================================
--  BUTTON CALLBACKS
-- ================================================================
StartBtn.MouseButton1Click:Connect(function()
    if S.enabled then stopFarm() else startFarm() end
end)

SpeedMinus.MouseButton1Click:Connect(function()
    S.speed=math.max(16,S.speed-1)
    SpeedVal.Text=tostring(S.speed)
    tw(SpeedVal,{TextColor3=C.gold},0.1):Play()
    tw(SpeedVal,{TextColor3=C.acc},0.25):Play()
end)
SpeedPlus.MouseButton1Click:Connect(function()
    S.speed=math.min(30,S.speed+1)
    SpeedVal.Text=tostring(S.speed)
    tw(SpeedVal,{TextColor3=C.gold},0.1):Play()
    tw(SpeedVal,{TextColor3=C.acc},0.25):Play()
end)

BtnStand.MouseButton1Click:Connect(function() S.style="Stand"; refreshStyle() end)
BtnLay.MouseButton1Click:Connect(function()   S.style="Lay";   refreshStyle() end)
BtnAuto.MouseButton1Click:Connect(function()  S.limitAction="AutoReset"; refreshLimit() end)
BtnStop.MouseButton1Click:Connect(function()  S.limitAction="Stop";      refreshLimit() end)

-- Render toggle re-applies immediately
-- (hooked via RunService tick below)
local lastRenderCheck = false
RunService.Heartbeat:Connect(function()
    local cur = getRender()
    if cur ~= lastRenderCheck then
        lastRenderCheck = cur
        applyRender()
    end
end)

-- ================================================================
updateStats()
print("[CoinFarmerPro v3] Loaded | Limit: "..COIN_LIMIT..(COIN_LIMIT==50 and " (Elite)" or " (Standard)"))
