-- ╔══════════════════════════════════════════════════╗
-- ║               Q U I N C Y H U B                 ║
-- ║           Delta Mobile Compatible                ║
-- ╚══════════════════════════════════════════════════╝

-- SERVIÇOS
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")

-- ESTADO GLOBAL
local isFlying  = false
local flyConn   = nil
local flySpeed  = 55
local uiVisible = false

-- ══════════════════════════════════════════════════
-- DIAGNÓSTICO: por que a logo sumia antes?
-- → ClipsDescendants=true cortava os filhos
-- → ZIndex dos filhos precisam ser > pai
-- → Pétalas precisam estar ACIMA do círculo dourado
-- Solução: sem ClipsDescendants, ZIndex explícito alto
-- ══════════════════════════════════════════════════

-- FUNÇÃO QUE CONSTRÓI A LOGO (círculo dourado + 8 pétalas escuras)
-- baseIndex: ZIndex base do container pai
local function BuildLogo(parent, diameterPx, baseIndex)
    local Z = baseIndex or 50

    -- 1) Círculo dourado (fundo)
    local gold = Instance.new("Frame")
    gold.Name              = "GoldCircle"
    gold.Size              = UDim2.new(0, diameterPx, 0, diameterPx)
    gold.Position          = UDim2.new(0.5, -diameterPx/2, 0.5, -diameterPx/2)
    gold.BackgroundColor3  = Color3.fromRGB(212, 168, 38)
    gold.BorderSizePixel   = 0
    gold.ZIndex            = Z + 1   -- acima do pai
    gold.Parent            = parent
    Instance.new("UICorner", gold).CornerRadius = UDim.new(1, 0)

    -- 2) 8 pétalas escuras giradas (criam o efeito côncavo dourado)
    --    Cada pétala: retângulo arredondado, ~38% da largura, ~52% da altura
    --    posicionado levemente acima do centro e rotacionado i*45°
    local pw = math.floor(diameterPx * 0.38)
    local ph = math.floor(diameterPx * 0.54)
    -- offset vertical: pétala começa acima do centro para o efeito correto
    local py = -math.floor(ph * 0.86)

    for i = 0, 7 do
        local petal = Instance.new("Frame")
        petal.Name             = "Petal"..i
        petal.Size             = UDim2.new(0, pw, 0, ph)
        petal.Position         = UDim2.new(0.5, -pw/2, 0.5, py)
        petal.BackgroundColor3 = Color3.fromRGB(8, 12, 30)
        petal.BorderSizePixel  = 0
        petal.Rotation         = i * 45
        petal.ZIndex           = Z + 2   -- acima do círculo dourado
        petal.Parent           = gold    -- filhos do círculo dourado
        Instance.new("UICorner", petal).CornerRadius = UDim.new(0.5, 0)
    end

    return gold
end

-- ══════════════════════════════════════════════════
-- SCREEN GUI
-- ══════════════════════════════════════════════════
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name           = "QuincyHub"
ScreenGui.ResetOnSpawn   = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
ScreenGui.IgnoreGuiInset = true
ScreenGui.Parent         = PlayerGui

-- ══════════════════════════════════════════════════
-- BOTÃO LOGO FLUTUANTE (sempre visível, arrastável)
-- ══════════════════════════════════════════════════
-- ZIndex base do botão = 100
local LogoBtn = Instance.new("TextButton")
LogoBtn.Name             = "LogoBtn"
LogoBtn.Size             = UDim2.new(0, 70, 0, 70)
LogoBtn.Position         = UDim2.new(0, 16, 0.5, -35)
LogoBtn.BackgroundColor3 = Color3.fromRGB(8, 12, 30)
LogoBtn.BorderSizePixel  = 0
LogoBtn.Text             = ""
LogoBtn.ZIndex           = 100
LogoBtn.AutoButtonColor  = false
LogoBtn.Parent           = ScreenGui
Instance.new("UICorner", LogoBtn).CornerRadius = UDim.new(1, 0)

local logoBtnStroke = Instance.new("UIStroke", LogoBtn)
logoBtnStroke.Color     = Color3.fromRGB(212, 168, 38)
logoBtnStroke.Thickness = 2.5

-- Logo dentro do botão: base ZIndex = 100 → pétalas ficam em 102
BuildLogo(LogoBtn, 62, 100)

-- ══════════════════════════════════════════════════
-- DRAG DO BOTÃO LOGO
-- Lógica:
--   InputBegan  → marca início (posição toque + posição botão)
--   InputChanged via UIS → move o botão se arrastando
--   InputEnded  → se não houve movimento real → toggle UI
-- ══════════════════════════════════════════════════
local dragging   = false
local touchStart = nil
local btnStart   = nil
local wasDrag    = false

-- Referência ao MainFrame definida abaixo; usamos upvalue
local MainFrame  -- declarado aqui, definido depois

local function toggleUI()
    uiVisible = not uiVisible
    MainFrame.Visible = uiVisible
    if uiVisible then
        MainFrame.Size = UDim2.new(0, 0, 0, 0)
        TweenService:Create(MainFrame,
            TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
            {Size = UDim2.new(0, 310, 0, 410)}
        ):Play()
    end
end

LogoBtn.InputBegan:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.Touch
    or inp.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging   = true
        wasDrag    = false
        touchStart = Vector2.new(inp.Position.X, inp.Position.Y)
        local abs  = LogoBtn.AbsolutePosition
        btnStart   = Vector2.new(abs.X, abs.Y)
    end
end)

UserInputService.InputChanged:Connect(function(inp)
    if not dragging then return end
    if inp.UserInputType ~= Enum.UserInputType.Touch
    and inp.UserInputType ~= Enum.UserInputType.MouseMovement then return end

    local cur   = Vector2.new(inp.Position.X, inp.Position.Y)
    local delta = cur - touchStart

    if delta.Magnitude > 6 then wasDrag = true end
    if not wasDrag then return end

    local vp = workspace.CurrentCamera.ViewportSize
    local nx = math.clamp(btnStart.X + delta.X, 0, vp.X - 70)
    local ny = math.clamp(btnStart.Y + delta.Y, 0, vp.Y - 70)
    LogoBtn.Position = UDim2.new(0, nx, 0, ny)
end)

UserInputService.InputEnded:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.Touch
    or inp.UserInputType == Enum.UserInputType.MouseButton1 then
        if dragging and not wasDrag then
            toggleUI()
        end
        dragging = false
    end
end)

-- ══════════════════════════════════════════════════
-- MAIN FRAME (começa invisível)
-- ZIndex base = 10; conteúdo interno usa 11+
-- ══════════════════════════════════════════════════
MainFrame = Instance.new("Frame")
MainFrame.Name             = "MainFrame"
MainFrame.Size             = UDim2.new(0, 310, 0, 410)
MainFrame.Position         = UDim2.new(0.5, -155, 0.5, -205)
MainFrame.BackgroundColor3 = Color3.fromRGB(8, 12, 30)
MainFrame.BorderSizePixel  = 0
MainFrame.ZIndex           = 10
MainFrame.Visible          = false
MainFrame.Parent           = ScreenGui
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 16)

local frameStroke = Instance.new("UIStroke", MainFrame)
frameStroke.Color     = Color3.fromRGB(212, 168, 38)
frameStroke.Thickness = 2.2

-- Gradiente fundo
local bgGrad = Instance.new("UIGradient", MainFrame)
bgGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(14, 21, 52)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(5, 8, 20)),
})
bgGrad.Rotation = 150

-- Barra superior dourada fina
local topBar = Instance.new("Frame", MainFrame)
topBar.Size             = UDim2.new(1, 0, 0, 4)
topBar.Position         = UDim2.new(0, 0, 0, 0)
topBar.BackgroundColor3 = Color3.fromRGB(212, 168, 38)
topBar.BorderSizePixel  = 0
topBar.ZIndex           = 11
Instance.new("UICorner", topBar).CornerRadius = UDim.new(0, 16)

-- ── LOGO NO TOPO DA UI ──────────────────────────────
-- Container: frame sem clip, ZIndex=11
-- Logo dentro: ZIndex base = 11 → pétalas ficam em 13
local logoHolder = Instance.new("Frame", MainFrame)
logoHolder.Name             = "LogoHolder"
logoHolder.Size             = UDim2.new(0, 90, 0, 90)
logoHolder.Position         = UDim2.new(0.5, -45, 0, 16)
logoHolder.BackgroundColor3 = Color3.fromRGB(8, 12, 30)
logoHolder.BorderSizePixel  = 0
logoHolder.ZIndex           = 11
logoHolder.Parent           = MainFrame
Instance.new("UICorner", logoHolder).CornerRadius = UDim.new(1, 0)

local holderStroke = Instance.new("UIStroke", logoHolder)
holderStroke.Color     = Color3.fromRGB(212, 168, 38)
holderStroke.Thickness = 2.5

-- Constrói logo com ZIndex base 11 → círculo=12, pétalas=13
BuildLogo(logoHolder, 82, 11)

-- ── TÍTULO ──────────────────────────────────────────
local titleLbl = Instance.new("TextLabel", MainFrame)
titleLbl.Size               = UDim2.new(1, 0, 0, 30)
titleLbl.Position           = UDim2.new(0, 0, 0, 118)
titleLbl.BackgroundTransparency = 1
titleLbl.Text               = "QuincyHub"
titleLbl.TextColor3         = Color3.fromRGB(220, 178, 50)
titleLbl.TextSize           = 24
titleLbl.Font               = Enum.Font.GothamBold
titleLbl.TextXAlignment     = Enum.TextXAlignment.Center
titleLbl.ZIndex             = 11

local titleStroke = Instance.new("UIStroke", titleLbl)
titleStroke.Color     = Color3.fromRGB(90, 60, 0)
titleStroke.Thickness = 1

-- ── SUBTÍTULO ───────────────────────────────────────
local subLbl = Instance.new("TextLabel", MainFrame)
subLbl.Size               = UDim2.new(1, 0, 0, 18)
subLbl.Position           = UDim2.new(0, 0, 0, 150)
subLbl.BackgroundTransparency = 1
subLbl.Text               = "v1.0  ·  Delta Mobile"
subLbl.TextColor3         = Color3.fromRGB(135, 108, 32)
subLbl.TextSize           = 12
subLbl.Font               = Enum.Font.Gotham
subLbl.TextXAlignment     = Enum.TextXAlignment.Center
subLbl.ZIndex             = 11

-- ── DIVISOR ─────────────────────────────────────────
local divider = Instance.new("Frame", MainFrame)
divider.Size              = UDim2.new(0.78, 0, 0, 1)
divider.Position          = UDim2.new(0.11, 0, 0, 180)
divider.BackgroundColor3  = Color3.fromRGB(212, 168, 38)
divider.BackgroundTransparency = 0.5
divider.BorderSizePixel   = 0
divider.ZIndex            = 11

-- ── LABEL SEÇÃO ─────────────────────────────────────
local secLbl = Instance.new("TextLabel", MainFrame)
secLbl.Size               = UDim2.new(1, -32, 0, 18)
secLbl.Position           = UDim2.new(0, 16, 0, 194)
secLbl.BackgroundTransparency = 1
secLbl.Text               = "✦  MOVIMENTO"
secLbl.TextColor3         = Color3.fromRGB(170, 132, 40)
secLbl.TextSize           = 11
secLbl.Font               = Enum.Font.GothamBold
secLbl.TextXAlignment     = Enum.TextXAlignment.Left
secLbl.ZIndex             = 11

-- ── FLY CARD ────────────────────────────────────────
local flyCard = Instance.new("Frame", MainFrame)
flyCard.Name             = "FlyCard"
flyCard.Size             = UDim2.new(1, -32, 0, 110)
flyCard.Position         = UDim2.new(0, 16, 0, 220)
flyCard.BackgroundColor3 = Color3.fromRGB(13, 20, 46)
flyCard.BorderSizePixel  = 0
flyCard.ZIndex           = 11
Instance.new("UICorner", flyCard).CornerRadius = UDim.new(0, 12)

local cardStroke = Instance.new("UIStroke", flyCard)
cardStroke.Color        = Color3.fromRGB(212, 168, 38)
cardStroke.Thickness    = 1
cardStroke.Transparency = 0.45

-- Ícone 🕊 no card
local flyIcon = Instance.new("TextLabel", flyCard)
flyIcon.Size               = UDim2.new(0, 38, 0, 38)
flyIcon.Position           = UDim2.new(0, 10, 0.5, -19)
flyIcon.BackgroundTransparency = 1
flyIcon.Text               = "🕊"
flyIcon.TextSize           = 26
flyIcon.Font               = Enum.Font.GothamBold
flyIcon.ZIndex             = 12

local flyCardTitle = Instance.new("TextLabel", flyCard)
flyCardTitle.Size               = UDim2.new(0, 110, 0, 22)
flyCardTitle.Position           = UDim2.new(0, 56, 0, 16)
flyCardTitle.BackgroundTransparency = 1
flyCardTitle.Text               = "Fly Mode"
flyCardTitle.TextColor3         = Color3.fromRGB(228, 186, 60)
flyCardTitle.TextSize           = 15
flyCardTitle.Font               = Enum.Font.GothamBold
flyCardTitle.TextXAlignment     = Enum.TextXAlignment.Left
flyCardTitle.ZIndex             = 12

local flyCardDesc = Instance.new("TextLabel", flyCard)
flyCardDesc.Size               = UDim2.new(0, 140, 0, 16)
flyCardDesc.Position           = UDim2.new(0, 56, 0, 42)
flyCardDesc.BackgroundTransparency = 1
flyCardDesc.Text               = "Voa pelo mapa livremente"
flyCardDesc.TextColor3         = Color3.fromRGB(105, 85, 35)
flyCardDesc.TextSize           = 11
flyCardDesc.Font               = Enum.Font.Gotham
flyCardDesc.TextXAlignment     = Enum.TextXAlignment.Left
flyCardDesc.ZIndex             = 12

-- ── BOTÃO FLY ───────────────────────────────────────
local FlyButton = Instance.new("TextButton", flyCard)
FlyButton.Name             = "FlyButton"
FlyButton.Size             = UDim2.new(0, 80, 0, 34)
FlyButton.Position         = UDim2.new(1, -92, 0.5, -17)
FlyButton.BackgroundColor3 = Color3.fromRGB(212, 168, 38)
FlyButton.BorderSizePixel  = 0
FlyButton.Text             = "FLY"
FlyButton.TextColor3       = Color3.fromRGB(8, 12, 30)
FlyButton.TextSize         = 15
FlyButton.Font             = Enum.Font.GothamBold
FlyButton.ZIndex           = 13
FlyButton.AutoButtonColor  = false
Instance.new("UICorner", FlyButton).CornerRadius = UDim.new(0, 10)

-- Gradiente dourado no botão
local flyBtnGrad = Instance.new("UIGradient", FlyButton)
flyBtnGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 215, 75)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(170, 128, 18)),
})
flyBtnGrad.Rotation = 90

-- ── RODAPÉ ──────────────────────────────────────────
local footer = Instance.new("TextLabel", MainFrame)
footer.Size               = UDim2.new(1, 0, 0, 18)
footer.Position           = UDim2.new(0, 0, 1, -26)
footer.BackgroundTransparency = 1
footer.Text               = "QuincyHub  ·  Made with 🌙"
footer.TextColor3         = Color3.fromRGB(65, 52, 20)
footer.TextSize           = 11
footer.Font               = Enum.Font.Gotham
footer.TextXAlignment     = Enum.TextXAlignment.Center
footer.ZIndex             = 11

-- ══════════════════════════════════════════════════
-- LÓGICA DE VOO
-- ══════════════════════════════════════════════════
local function GetHRP()
    local c = LocalPlayer.Character
    return c and c:FindFirstChild("HumanoidRootPart")
end

local function GetHum()
    local c = LocalPlayer.Character
    return c and c:FindFirstChildOfClass("Humanoid")
end

local function StartFly()
    local hrp = GetHRP()
    local hum = GetHum()
    if not hrp or not hum then return end

    hum.PlatformStand = true

    -- Remove instâncias antigas se existirem
    for _, n in ipairs({"QH_BV","QH_BG"}) do
        local old = hrp:FindFirstChild(n)
        if old then old:Destroy() end
    end

    local bv = Instance.new("BodyVelocity", hrp)
    bv.Name      = "QH_BV"
    bv.Velocity  = Vector3.zero
    bv.MaxForce  = Vector3.new(1e5, 1e5, 1e5)

    local bg = Instance.new("BodyGyro", hrp)
    bg.Name      = "QH_BG"
    bg.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
    bg.P         = 9000
    bg.CFrame    = hrp.CFrame

    local cam = workspace.CurrentCamera

    flyConn = RunService.Heartbeat:Connect(function()
        local r = GetHRP()
        if not r then return end
        local bvr = r:FindFirstChild("QH_BV")
        local bgr = r:FindFirstChild("QH_BG")
        if not bvr or not bgr then return end

        local cf  = cam.CFrame
        local dir = Vector3.zero

        -- Teclado / PC
        if UserInputService:IsKeyDown(Enum.KeyCode.W)         then dir += cf.LookVector  end
        if UserInputService:IsKeyDown(Enum.KeyCode.S)         then dir -= cf.LookVector  end
        if UserInputService:IsKeyDown(Enum.KeyCode.A)         then dir -= cf.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.D)         then dir += cf.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space)     then dir += Vector3.yAxis  end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then dir -= Vector3.yAxis  end

        -- Mobile: MoveDirection do Humanoid
        local h = GetHum()
        if h then
            local mv = h.MoveDirection
            if mv.Magnitude > 0.05 then
                dir += cf.LookVector  * (-mv.Z)
                dir += cf.RightVector * mv.X
            end
        end

        if dir.Magnitude > 0 then dir = dir.Unit end
        bvr.Velocity = dir * flySpeed
        bgr.CFrame   = cf
    end)
end

local function StopFly()
    if flyConn then flyConn:Disconnect(); flyConn = nil end
    local hrp = GetHRP()
    local hum = GetHum()
    if hum then hum.PlatformStand = false end
    if hrp then
        local bv = hrp:FindFirstChild("QH_BV")
        local bg = hrp:FindFirstChild("QH_BG")
        if bv then bv:Destroy() end
        if bg then bg:Destroy() end
    end
end

-- Cores do botão: dourado / verde
local GOLD_SEQ = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 215, 75)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(170, 128, 18)),
})
local GREEN_SEQ = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(80, 230, 120)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(20, 155, 65)),
})

local function SetFlyButtonActive(active)
    if active then
        flyBtnGrad.Color       = GREEN_SEQ
        FlyButton.TextColor3   = Color3.fromRGB(255, 255, 255)
        FlyButton.Text         = "FLY ✓"
        TweenService:Create(flyCard, TweenInfo.new(0.25),
            {BackgroundColor3 = Color3.fromRGB(8, 28, 16)}):Play()
    else
        flyBtnGrad.Color       = GOLD_SEQ
        FlyButton.TextColor3   = Color3.fromRGB(8, 12, 30)
        FlyButton.Text         = "FLY"
        TweenService:Create(flyCard, TweenInfo.new(0.25),
            {BackgroundColor3 = Color3.fromRGB(13, 20, 46)}):Play()
    end
end

-- ══════════════════════════════════════════════════
-- CLIQUE NO BOTÃO FLY
-- Activated funciona tanto em mouse quanto em touch
-- ══════════════════════════════════════════════════
FlyButton.Activated:Connect(function()
    isFlying = not isFlying
    SetFlyButtonActive(isFlying)
    if isFlying then StartFly() else StopFly() end
end)

-- ══════════════════════════════════════════════════
-- RESET AO MORRER / RESPAWN
-- ══════════════════════════════════════════════════
LocalPlayer.CharacterAdded:Connect(function()
    isFlying = false
    if flyConn then flyConn:Disconnect(); flyConn = nil end
    SetFlyButtonActive(false)
end)

print("✦ QuincyHub iniciado! Toque no botão logo para abrir a UI. ✦")
