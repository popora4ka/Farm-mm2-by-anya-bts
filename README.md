-- ╔══════════════════════════════════════════════════════╗
-- ║         COIN FARMER PRO — by Script Generator       ║
-- ║   Persistent GUI | AFK | Speed | Animation | Reset  ║
-- ╚══════════════════════════════════════════════════════╝

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local Humanoid = Character:WaitForChild("Humanoid")
local RootPart = Character:WaitForChild("HumanoidRootPart")

-- ══════════════════════════
--   НАСТРОЙКИ ПО УМОЛЧАНИЮ
-- ══════════════════════════
local Settings = {
    Enabled       = false,
    Speed         = 21,
    Style         = "Стоя",      -- "Стоя" | "Лёжа"
    AntiAFK       = true,
    NoRender      = false,
    LimitAction   = "Автосброс", -- "Автосброс" | "Стоп"
    CoinLimit     = 100,
}

local State = {
    Coins        = 0,
    CoinsTotal   = 0,
    Resets       = 0,
    Running      = false,
    CurrentCoin  = nil,
    AFKConn      = nil,
    MainConn     = nil,
    HoverConn    = nil,
    GUIVisible   = true,
}

-- ══════════════════════════════════
--  ЗАЩИТА GUI ОТ УДАЛЕНИЯ ПРИ RESET
-- ══════════════════════════════════
local ScreenGui
local function getOrCreateGui()
    -- Храним GUI в CoreGui чтобы не удалялось при Reset
    local coreGui = game:GetService("CoreGui")
    local existing = coreGui:FindFirstChild("CoinFarmerPro")
    if existing then return existing end
    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "CoinFarmerPro"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ScreenGui.Parent = coreGui
    return ScreenGui
end

ScreenGui = getOrCreateGui()

-- ══════════════════════════
--    ЦВЕТА / ТЕМА
-- ══════════════════════════
local C = {
    BG        = Color3.fromRGB(10,  12,  20),
    Panel     = Color3.fromRGB(16,  20,  35),
    Accent    = Color3.fromRGB(90,  200, 255),
    Accent2   = Color3.fromRGB(255, 180, 50),
    Green     = Color3.fromRGB(80,  230, 140),
    Red       = Color3.fromRGB(255, 80,  90),
    Text      = Color3.fromRGB(220, 230, 255),
    Muted     = Color3.fromRGB(100, 110, 140),
    Border    = Color3.fromRGB(40,  50,  80),
    Shadow    = Color3.fromRGB(5,   7,   14),
}

-- ══════════════════════════
--    УТИЛИТЫ
-- ══════════════════════════
local function makeTween(obj, props, t, style, dir)
    local info = TweenInfo.new(t or 0.25, style or Enum.EasingStyle.Quart, dir or Enum.EasingDirection.Out)
    return TweenService:Create(obj, info, props)
end

local function newInst(cls, props)
    local o = Instance.new(cls)
    for k, v in pairs(props) do o[k] = v end
    return o
end

local function round(frame, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius or 8)
    corner.Parent = frame
    return corner
end

local function stroke(frame, color, thickness)
    local s = Instance.new("UIStroke")
    s.Color = color or C.Border
    s.Thickness = thickness or 1
    s.Parent = frame
    return s
end

-- ══════════════════════════════════════
--   ПОСТРОЕНИЕ ГЛАВНОГО ОКНА
-- ══════════════════════════════════════

-- ── Мини-кнопка (когда свёрнуто) ──
local MiniBtn = newInst("TextButton", {
    Name = "MiniBtn",
    Size = UDim2.new(0, 46, 0, 46),
    Position = UDim2.new(0, 20, 0.5, -23),
    BackgroundColor3 = C.BG,
    Text = "💰",
    TextSize = 22,
    Font = Enum.Font.GothamBold,
    TextColor3 = C.Accent2,
    AutoButtonColor = false,
    Visible = false,
    ZIndex = 20,
    Parent = ScreenGui,
})
round(MiniBtn, 14)
stroke(MiniBtn, C.Accent2, 2)

-- пульсация мини-кнопки
local function pulseMini()
    while MiniBtn.Visible do
        makeTween(MiniBtn, {TextTransparency = 0.4}, 0.7, Enum.EasingStyle.Sine):Play()
        task.wait(0.7)
        makeTween(MiniBtn, {TextTransparency = 0}, 0.7, Enum.EasingStyle.Sine):Play()
        task.wait(0.7)
    end
end

-- ── Основное окно ──
local MainFrame = newInst("Frame", {
    Name = "Main",
    Size = UDim2.new(0, 320, 0, 480),
    Position = UDim2.new(0, 20, 0.5, -240),
    BackgroundColor3 = C.BG,
    BorderSizePixel = 0,
    ZIndex = 10,
    Parent = ScreenGui,
})
round(MainFrame, 14)
stroke(MainFrame, C.Border, 1.5)

-- Тень
local Shadow = newInst("Frame", {
    Size = UDim2.new(1, 16, 1, 16),
    Position = UDim2.new(0, -8, 0, 8),
    BackgroundColor3 = C.Shadow,
    BackgroundTransparency = 0.4,
    ZIndex = 9,
    Parent = MainFrame,
})
round(Shadow, 16)

-- ── Заголовок ──
local Header = newInst("Frame", {
    Size = UDim2.new(1, 0, 0, 52),
    BackgroundColor3 = C.Panel,
    BorderSizePixel = 0,
    ZIndex = 11,
    Parent = MainFrame,
})
round(Header, 14)
-- нижние углы плоские
local BottomMask = newInst("Frame", {
    Size = UDim2.new(1, 0, 0, 14),
    Position = UDim2.new(0, 0, 1, -14),
    BackgroundColor3 = C.Panel,
    BorderSizePixel = 0,
    ZIndex = 11,
    Parent = Header,
})

local TitleIcon = newInst("TextLabel", {
    Text = "💰",
    TextSize = 22,
    Position = UDim2.new(0, 14, 0, 0),
    Size = UDim2.new(0, 32, 1, 0),
    BackgroundTransparency = 1,
    TextColor3 = C.Accent2,
    Font = Enum.Font.GothamBold,
    ZIndex = 12,
    Parent = Header,
})

local TitleLabel = newInst("TextLabel", {
    Text = "COIN FARMER PRO",
    TextSize = 14,
    Position = UDim2.new(0, 50, 0, 0),
    Size = UDim2.new(1, -100, 1, 0),
    BackgroundTransparency = 1,
    TextColor3 = C.Accent,
    Font = Enum.Font.GothamBold,
    TextXAlignment = Enum.TextXAlignment.Left,
    ZIndex = 12,
    Parent = Header,
})

-- Кнопка свернуть
local MinBtn = newInst("TextButton", {
    Text = "─",
    TextSize = 18,
    Size = UDim2.new(0, 34, 0, 28),
    Position = UDim2.new(1, -42, 0, 12),
    BackgroundColor3 = C.Border,
    TextColor3 = C.Text,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    ZIndex = 12,
    Parent = Header,
})
round(MinBtn, 7)

-- ── Статистика ──
local StatsBar = newInst("Frame", {
    Size = UDim2.new(1, -24, 0, 52),
    Position = UDim2.new(0, 12, 0, 60),
    BackgroundColor3 = C.Panel,
    BorderSizePixel = 0,
    ZIndex = 11,
    Parent = MainFrame,
})
round(StatsBar, 10)
stroke(StatsBar, C.Border)

local function statLabel(text, xPos, icon)
    local f = newInst("Frame", {
        Size = UDim2.new(0.33, 0, 1, 0),
        Position = UDim2.new(xPos, 0, 0, 0),
        BackgroundTransparency = 1,
        ZIndex = 12,
        Parent = StatsBar,
    })
    local ico = newInst("TextLabel", {
        Text = icon,
        TextSize = 14,
        Size = UDim2.new(1, 0, 0, 18),
        Position = UDim2.new(0, 0, 0, 6),
        BackgroundTransparency = 1,
        TextColor3 = C.Muted,
        Font = Enum.Font.Gotham,
        ZIndex = 12,
        Parent = f,
    })
    local val = newInst("TextLabel", {
        Text = "0",
        TextSize = 15,
        Size = UDim2.new(1, 0, 0, 20),
        Position = UDim2.new(0, 0, 0, 26),
        BackgroundTransparency = 1,
        TextColor3 = C.Text,
        Font = Enum.Font.GothamBold,
        ZIndex = 12,
        Parent = f,
    })
    return val
end

local CoinsLabel  = statLabel("Монеты", 0,      "🪙 Монеты")
local TotalLabel  = statLabel("Всего",  0.33,   "📦 Всего")
local ResetsLabel = statLabel("Сбросы", 0.66,   "🔄 Сбросы")

-- ── Прогресс-бар монет ──
local ProgBG = newInst("Frame", {
    Size = UDim2.new(1, -24, 0, 18),
    Position = UDim2.new(0, 12, 0, 120),
    BackgroundColor3 = C.Panel,
    BorderSizePixel = 0,
    ZIndex = 11,
    Parent = MainFrame,
})
round(ProgBG, 9)
stroke(ProgBG, C.Border)

local ProgFill = newInst("Frame", {
    Size = UDim2.new(0, 0, 1, 0),
    BackgroundColor3 = C.Accent2,
    BorderSizePixel = 0,
    ZIndex = 12,
    Parent = ProgBG,
})
round(ProgFill, 9)

local ProgLabel = newInst("TextLabel", {
    Size = UDim2.new(1, 0, 1, 0),
    Text = "0 / 100",
    TextSize = 11,
    BackgroundTransparency = 1,
    TextColor3 = C.Text,
    Font = Enum.Font.GothamBold,
    ZIndex = 13,
    Parent = ProgBG,
})

-- ── Секция: Скорость ──
local function sectionTitle(text, y)
    local l = newInst("TextLabel", {
        Text = text,
        TextSize = 11,
        Size = UDim2.new(1, -24, 0, 16),
        Position = UDim2.new(0, 12, 0, y),
        BackgroundTransparency = 1,
        TextColor3 = C.Muted,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 11,
        Parent = MainFrame,
    })
    return l
end

sectionTitle("⚡  СКОРОСТЬ ДВИЖЕНИЯ", 148)

local SpeedRow = newInst("Frame", {
    Size = UDim2.new(1, -24, 0, 36),
    Position = UDim2.new(0, 12, 0, 166),
    BackgroundColor3 = C.Panel,
    BorderSizePixel = 0,
    ZIndex = 11,
    Parent = MainFrame,
})
round(SpeedRow, 10)
stroke(SpeedRow, C.Border)

local SpeedMinus = newInst("TextButton", {
    Text = "−",
    TextSize = 20,
    Size = UDim2.new(0, 36, 1, 0),
    Position = UDim2.new(0, 0, 0, 0),
    BackgroundColor3 = C.Border,
    TextColor3 = C.Accent,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    ZIndex = 12,
    Parent = SpeedRow,
})
round(SpeedMinus, 10)

local SpeedVal = newInst("TextLabel", {
    Text = tostring(Settings.Speed),
    TextSize = 16,
    Size = UDim2.new(1, -72, 1, 0),
    Position = UDim2.new(0, 36, 0, 0),
    BackgroundTransparency = 1,
    TextColor3 = C.Accent,
    Font = Enum.Font.GothamBold,
    ZIndex = 12,
    Parent = SpeedRow,
})

local SpeedPlus = newInst("TextButton", {
    Text = "+",
    TextSize = 20,
    Size = UDim2.new(0, 36, 1, 0),
    Position = UDim2.new(1, -36, 0, 0),
    BackgroundColor3 = C.Border,
    TextColor3 = C.Accent,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    ZIndex = 12,
    Parent = SpeedRow,
})
round(SpeedPlus, 10)

-- ── Стиль движения ──
sectionTitle("🧍  СТИЛЬ ДВИЖЕНИЯ", 210)

local StyleRow = newInst("Frame", {
    Size = UDim2.new(1, -24, 0, 36),
    Position = UDim2.new(0, 12, 0, 228),
    BackgroundColor3 = C.Panel,
    BorderSizePixel = 0,
    ZIndex = 11,
    Parent = MainFrame,
})
round(StyleRow, 10)

local function styleBtn(text, xPos)
    local b = newInst("TextButton", {
        Text = text,
        TextSize = 13,
        Size = UDim2.new(0.5, -2, 1, 0),
        Position = UDim2.new(xPos, 2, 0, 0),
        BackgroundColor3 = C.Border,
        TextColor3 = C.Muted,
        Font = Enum.Font.GothamBold,
        AutoButtonColor = false,
        ZIndex = 12,
        Parent = StyleRow,
    })
    round(b, 10)
    return b
end

local BtnStand = styleBtn("🧍 Стоя", 0)
local BtnLay   = styleBtn("🛏 Лёжа", 0.5)

local function updateStyleBtns()
    if Settings.Style == "Стоя" then
        BtnStand.BackgroundColor3 = C.Accent
        BtnStand.TextColor3 = C.BG
        BtnLay.BackgroundColor3 = C.Border
        BtnLay.TextColor3 = C.Muted
    else
        BtnLay.BackgroundColor3 = C.Accent
        BtnLay.TextColor3 = C.BG
        BtnStand.BackgroundColor3 = C.Border
        BtnStand.TextColor3 = C.Muted
    end
end
updateStyleBtns()

-- ── Переключатели ──
sectionTitle("⚙️  ПАРАМЕТРЫ", 272)

local function toggleRow(labelText, y, defaultOn, color)
    local row = newInst("Frame", {
        Size = UDim2.new(1, -24, 0, 34),
        Position = UDim2.new(0, 12, 0, y),
        BackgroundColor3 = C.Panel,
        BorderSizePixel = 0,
        ZIndex = 11,
        Parent = MainFrame,
    })
    round(row, 10)

    local lbl = newInst("TextLabel", {
        Text = labelText,
        TextSize = 12,
        Size = UDim2.new(1, -60, 1, 0),
        Position = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        TextColor3 = C.Text,
        Font = Enum.Font.Gotham,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 12,
        Parent = row,
    })

    local trackBG = newInst("Frame", {
        Size = UDim2.new(0, 42, 0, 22),
        Position = UDim2.new(1, -52, 0.5, -11),
        BackgroundColor3 = defaultOn and (color or C.Green) or C.Border,
        ZIndex = 12,
        Parent = row,
    })
    round(trackBG, 11)

    local knob = newInst("Frame", {
        Size = UDim2.new(0, 18, 0, 18),
        Position = defaultOn and UDim2.new(1, -20, 0.5, -9) or UDim2.new(0, 2, 0.5, -9),
        BackgroundColor3 = Color3.new(1,1,1),
        ZIndex = 13,
        Parent = trackBG,
    })
    round(knob, 9)

    local state = {on = defaultOn}

    local function toggle()
        state.on = not state.on
        local targetX = state.on and UDim2.new(1, -20, 0.5, -9) or UDim2.new(0, 2, 0.5, -9)
        makeTween(knob, {Position = targetX}, 0.2):Play()
        makeTween(trackBG, {BackgroundColor3 = state.on and (color or C.Green) or C.Border}, 0.2):Play()
        return state.on
    end

    row.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            toggle()
        end
    end)

    return row, function() return state.on end, toggle
end

local AfkRow,      afkGet,      afkToggle    = toggleRow("🛡  Защита от AFK",       290, Settings.AntiAFK)
local RenderRow,   renderGet,   renderToggle = toggleRow("🖥  Отключить рендер",    330, Settings.NoRender,  C.Accent)

-- ── Лимит: Автосброс / Стоп ──
sectionTitle("🎯  ДЕЙСТВИЕ ПРИ ЛИМИТЕ", 372)

local LimitRow = newInst("Frame", {
    Size = UDim2.new(1, -24, 0, 36),
    Position = UDim2.new(0, 12, 0, 390),
    BackgroundColor3 = C.Panel,
    BorderSizePixel = 0,
    ZIndex = 11,
    Parent = MainFrame,
})
round(LimitRow, 10)

local function limitBtn(text, xPos)
    local b = newInst("TextButton", {
        Text = text,
        TextSize = 12,
        Size = UDim2.new(0.5, -2, 1, 0),
        Position = UDim2.new(xPos, 2, 0, 0),
        BackgroundColor3 = C.Border,
        TextColor3 = C.Muted,
        Font = Enum.Font.GothamBold,
        AutoButtonColor = false,
        ZIndex = 12,
        Parent = LimitRow,
    })
    round(b, 10)
    return b
end

local BtnAutoReset = limitBtn("🔄 Автосброс", 0)
local BtnStop      = limitBtn("⏹ Стоп",       0.5)

local function updateLimitBtns()
    if Settings.LimitAction == "Автосброс" then
        BtnAutoReset.BackgroundColor3 = C.Accent2
        BtnAutoReset.TextColor3 = C.BG
        BtnStop.BackgroundColor3 = C.Border
        BtnStop.TextColor3 = C.Muted
    else
        BtnStop.BackgroundColor3 = C.Red
        BtnStop.TextColor3 = Color3.new(1,1,1)
        BtnAutoReset.BackgroundColor3 = C.Border
        BtnAutoReset.TextColor3 = C.Muted
    end
end
updateLimitBtns()

-- ── Главная кнопка СТАРТ/СТОП ──
local StartBtn = newInst("TextButton", {
    Text = "▶  НАЧАТЬ ФАРМ",
    TextSize = 15,
    Size = UDim2.new(1, -24, 0, 40),
    Position = UDim2.new(0, 12, 1, -52),
    BackgroundColor3 = C.Green,
    TextColor3 = C.BG,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    ZIndex = 11,
    Parent = MainFrame,
})
round(StartBtn, 12)

-- ══════════════════════════
--   ПЕРЕТАСКИВАНИЕ ОКНА
-- ══════════════════════════
local dragging, dragStart, startPos
Header.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = i.Position
        startPos = MainFrame.Position
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
        local delta = i.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

-- ══════════════════════════
--   АНИМАЦИЯ ПОЯВЛЕНИЯ
-- ══════════════════════════
MainFrame.BackgroundTransparency = 1
MainFrame:TweenPosition(UDim2.new(0, 20, 0.5, -240), "Out", "Back", 0.5, true)
makeTween(MainFrame, {BackgroundTransparency = 0}, 0.4):Play()

-- ══════════════════════════
--   СВЕРНУТЬ / РАЗВЕРНУТЬ
-- ══════════════════════════
local function collapse()
    State.GUIVisible = false
    makeTween(MainFrame, {Size = UDim2.new(0, 320, 0, 0), BackgroundTransparency = 1}, 0.3, Enum.EasingStyle.Quart):Play()
    task.wait(0.3)
    MainFrame.Visible = false
    MiniBtn.Visible = true
    MiniBtn.BackgroundTransparency = 1
    makeTween(MiniBtn, {BackgroundTransparency = 0}, 0.2):Play()
    task.spawn(pulseMini)
end

local function expand()
    State.GUIVisible = true
    MiniBtn.Visible = false
    MainFrame.Visible = true
    MainFrame.Size = UDim2.new(0, 320, 0, 0)
    makeTween(MainFrame, {Size = UDim2.new(0, 320, 0, 480), BackgroundTransparency = 0}, 0.35, Enum.EasingStyle.Back):Play()
end

MinBtn.MouseButton1Click:Connect(collapse)
MiniBtn.MouseButton1Click:Connect(expand)

-- ══════════════════════════
--   ОБНОВЛЕНИЕ СТАТИСТИКИ
-- ══════════════════════════
local function updateStats()
    CoinsLabel.Text  = tostring(State.Coins)
    TotalLabel.Text  = tostring(State.CoinsTotal)
    ResetsLabel.Text = tostring(State.Resets)
    local pct = math.clamp(State.Coins / Settings.CoinLimit, 0, 1)
    makeTween(ProgFill, {Size = UDim2.new(pct, 0, 1, 0)}, 0.3):Play()
    ProgLabel.Text = State.Coins .. " / " .. Settings.CoinLimit
    -- цвет прогресса
    local progColor = pct < 0.6 and C.Green or pct < 0.85 and C.Accent2 or C.Red
    makeTween(ProgFill, {BackgroundColor3 = progColor}, 0.3):Play()
end

-- ══════════════════════════
--   КНОПКИ УПРАВЛЕНИЯ
-- ══════════════════════════
SpeedMinus.MouseButton1Click:Connect(function()
    Settings.Speed = math.max(16, Settings.Speed - 1)
    SpeedVal.Text = tostring(Settings.Speed)
    makeTween(SpeedVal, {TextColor3 = C.Accent2}, 0.1):Play()
    makeTween(SpeedVal, {TextColor3 = C.Accent},  0.2):Play()
end)

SpeedPlus.MouseButton1Click:Connect(function()
    Settings.Speed = math.min(30, Settings.Speed + 1)
    SpeedVal.Text = tostring(Settings.Speed)
    makeTween(SpeedVal, {TextColor3 = C.Accent2}, 0.1):Play()
    makeTween(SpeedVal, {TextColor3 = C.Accent},  0.2):Play()
end)

BtnStand.MouseButton1Click:Connect(function()
    Settings.Style = "Стоя"
    updateStyleBtns()
end)

BtnLay.MouseButton1Click:Connect(function()
    Settings.Style = "Лёжа"
    updateStyleBtns()
end)

BtnAutoReset.MouseButton1Click:Connect(function()
    Settings.LimitAction = "Автосброс"
    updateLimitBtns()
end)

BtnStop.MouseButton1Click:Connect(function()
    Settings.LimitAction = "Стоп"
    updateLimitBtns()
end)

-- ══════════════════════════════════════
--   ПОИСК МОНЕТ В ВОРКСПЕЙСЕ
-- ══════════════════════════════════════
local function findNearestCoin()
    local nearest, dist = nil, math.huge
    for _, obj in ipairs(workspace:GetDescendants()) do
        -- Ищем объекты с именем содержащим "Coin"/"coin"/"монет"
        if (obj:IsA("BasePart") or obj:IsA("Model")) and
           (obj.Name:lower():find("coin") or obj.Name:lower():find("монет")) then
            local pos
            if obj:IsA("BasePart") then
                pos = obj.Position
            elseif obj:IsA("Model") and obj.PrimaryPart then
                pos = obj.PrimaryPart.Position
            end
            if pos then
                local d = (RootPart.Position - pos).Magnitude
                if d < dist then
                    dist = d
                    nearest = obj
                end
            end
        end
    end
    return nearest, dist
end

-- ══════════════════════════════════════
--   ПРИМЕНЕНИЕ СТИЛЯ ДВИЖЕНИЯ
-- ══════════════════════════════════════
local function applyStyle()
    if Settings.Style == "Лёжа" then
        -- Поворачиваем персонажа горизонтально (эффект парения)
        if Humanoid then
        Humanoid.PlatformStand = false
        end
    end
end

local function resetStyle()
    if Humanoid then
        Humanoid.PlatformStand = false
    end
end

-- ══════════════════════════
--   AFK ЗАЩИТА
-- ══════════════════════════
local function startAntiAFK()
    State.AFKConn = RunService.Heartbeat:Connect(function()
        if afkGet() then
            -- Симулируем движение мыши для предотвращения кика
            local vp = workspace.CurrentCamera.ViewportSize
            local fakeMove = Vector2.new(math.random(-1,1), math.random(-1,1))
        end
    end)
end

local function stopAntiAFK()
    if State.AFKConn then
        State.AFKConn:Disconnect()
        State.AFKConn = nil
    end
end

-- Альтернативный метод Anti-AFK через VirtualUser
local ok, VU = pcall(function() return game:GetService("VirtualUser") end)
if ok and VU then
    Players.LocalPlayer.Idled:Connect(function()
        if afkGet() then
            VU:CaptureController()
            VU:ClickButton2(Vector2.new())
        end
    end)
end

-- ══════════════════════════
--   РЕНДЕРИНГ
-- ══════════════════════════
local function setRender(enabled)
    local lp = Players.LocalPlayer
    if not enabled then
        -- Снижаем качество рендера
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        workspace.StreamingEnabled = false
    else
        settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
    end
end

-- ══════════════════════════════════════
--   ОСНОВНОЙ ФАРМ ЦИКЛ
-- ══════════════════════════════════════
local function startFarm()
    State.Running = true
    startAntiAFK()

    State.MainConn = RunService.Heartbeat:Connect(function()
        -- Синхронизируем персонажа (может поменяться после Reset)
        Character = LocalPlayer.Character
        if not Character then return end
        Humanoid = Character:FindFirstChildOfClass("Humanoid")
        RootPart = Character:FindFirstChild("HumanoidRootPart")
        if not Humanoid or not RootPart then return end

        -- Скорость
        Humanoid.WalkSpeed = Settings.Speed

        -- Стиль
        applyStyle()

        -- Рендер
        setRender(not renderGet())

        -- Найти монету
        local coin, dist = findNearestCoin()
        if coin then
            State.CurrentCoin = coin
            local targetPos
            if coin:IsA("BasePart") then
                targetPos = coin.Position
            elseif coin:IsA("Model") and coin.PrimaryPart then
                targetPos = coin.PrimaryPart.Position
            end

            if targetPos then
                local offset = Settings.Style == "Лёжа" and Vector3.new(0, 2.5, 0) or Vector3.new(0, 0, 0)
                -- Телепортируем к монете с плавным движением
                if dist > 3 then
                    local dir = (targetPos - RootPart.Position).Unit
                    RootPart.CFrame = CFrame.new(RootPart.Position + dir * math.min(Settings.Speed * 0.1, dist) + offset)
                else
                    -- Подбираем монету (Touch)
                    if coin:IsA("BasePart") then
                        local fakePart = Instance.new("Part")
                        fakePart.Size = Vector3.new(1,1,1)
                        fakePart.CFrame = coin.CFrame
                        fakePart.Parent = workspace
                        fakePart.CanCollide = false
                        fakePart.Anchored = true
                        coin.Touched:Fire(RootPart)
                        fakePart:Destroy()
                    end
                    -- Считаем монету
                    State.Coins = State.Coins + 1
                    State.CoinsTotal = State.CoinsTotal + 1
                    updateStats()

                    -- Проверяем лимит
                    if State.Coins >= Settings.CoinLimit then
                        if Settings.LimitAction == "Автосброс" then
                            State.Coins = 0
                            State.Resets = State.Resets + 1
                            updateStats()
                            -- Ищем кнопку сброса
                            for _, v in ipairs(workspace:GetDescendants()) do
                                if v:IsA("BasePart") and (v.Name:lower():find("reset") or v.Name:lower():find("сброс")) then
                                    v.Touched:Fire(RootPart)
                                    break
                                end
                            end
                        else
                            stopFarm()
                        end
                    end
                end
            end
        end
    end)
end

function stopFarm()
    State.Running = false
    stopAntiAFK()
    if State.MainConn then
        State.MainConn:Disconnect()
        State.MainConn = nil
    end
    resetStyle()
    if Character and Character:FindFirstChildOfClass("Humanoid") then
        Character:FindFirstChildOfClass("Humanoid").WalkSpeed = 16
    end
end

-- ══════════════════════════
--   КНОПКА СТАРТ / СТОП
-- ══════════════════════════
StartBtn.MouseButton1Click:Connect(function()
    if State.Running then
        stopFarm()
        makeTween(StartBtn, {BackgroundColor3 = C.Green}, 0.2):Play()
        StartBtn.Text = "▶  НАЧАТЬ ФАРМ"
    else
        startFarm()
        makeTween(StartBtn, {BackgroundColor3 = C.Red}, 0.2):Play()
        StartBtn.Text = "⏹  ОСТАНОВИТЬ"
    end
end)

-- Hover эффект кнопок
for _, btn in ipairs({StartBtn, SpeedMinus, SpeedPlus, MinBtn, MiniBtn, BtnStand, BtnLay, BtnAutoReset, BtnStop}) do
    btn.MouseEnter:Connect(function()
        makeTween(btn, {BackgroundTransparency = 0.15}, 0.15):Play()
    end)
    btn.MouseLeave:Connect(function()
        makeTween(btn, {BackgroundTransparency = 0}, 0.15):Play()
    end)
end

-- ══════════════════════════════════════════════════════
--  ПЕРЕСОЗДАНИЕ ССЫЛОК ПОСЛЕ RESET (GUI СОХРАНЯЕТСЯ!)
-- ══════════════════════════════════════════════════════
LocalPlayer.CharacterAdded:Connect(function(newChar)
    Character = newChar
    Humanoid = newChar:WaitForChild("Humanoid")
    RootPart = newChar:WaitForChild("HumanoidRootPart")

    -- Если фарм был активен — перезапускаем
    if State.Running then
        task.wait(1)
        startFarm()
    end
    -- GUI остаётся нетронутым
end)

-- ══════════════════════════
--   АНИМАЦИЯ ЗАГОЛОВКА
-- ══════════════════════════
task.spawn(function()
    while true do
        makeTween(TitleLabel, {TextColor3 = C.Accent2}, 1.5, Enum.EasingStyle.Sine):Play()
        task.wait(1.5)
        makeTween(TitleLabel, {TextColor3 = C.Accent}, 1.5, Enum.EasingStyle.Sine):Play()
        task.wait(1.5)
    end
end)

updateStats()
print("[CoinFarmerPro] Скрипт успешно загружен! GUI защищён от сброса.")
