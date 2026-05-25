-- ╔══════════════════════════════════════════════════════╗
-- ║                  Q U I N C Y H U B                  ║
-- ║              Delta Mobile Compatible                 ║
-- ╚══════════════════════════════════════════════════════╝

-- SERVIÇOS
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")

-- PLAYER
local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")

-- ESTADO
local isFlying    = false
local flyConn     = nil
local flySpeed    = 55
local uiVisible   = false  -- UI começa OCULTA, só aparece ao clicar no botão logo

-- ══════════════════════════════════════════════════════
--  HELPER: desenha a logo (8 pétalas côncavas douradas
--  sobre fundo azul escuro, idêntica à imagem enviada)
-- ══════════════════════════════════════════════════════
local function BuildLogo(parent, size)
    -- Container circular dourado
    local circle = Instance.new("Frame")
    circle.Size           = UDim2.new(0, size, 0, size)
    circle.Position       = UDim2.new(0.5, -size/2, 0.5, -size/2)
    circle.BackgroundColor3 = Color3.fromRGB(210, 165, 40)
    circle.BorderSizePixel  = 0
    circle.ZIndex           = parent.ZIndex + 1
    circle.Parent           = parent
    Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)

    -- 8 pétalas (recortes escuros que criam o efeito côncavo dourado)
    local petalW = math.floor(size * 0.38)
    local petalH = math.floor(size * 0.52)
    for i = 0, 7 do
        local petal = Instance.new("Frame")
        petal.Size              = UDim2.new(0, petalW, 0, petalH)
        petal.Position          = UDim2.new(0.5, -petalW/2, 0.5, -(petalH * 0.88))
        petal.BackgroundColor3  = Color3.fromRGB(8, 12, 30)
        petal.BorderSizePixel   = 0
        petal.Rotation          = i * 45
        petal.ZIndex            = circle.ZIndex + 1
        petal.Parent            = circle
        Instance.new("UICorner", petal).CornerRadius = UDim.new(0.5, 0)
    end

    return circle
end

-- ══════════════════════════════════════════════════════
--  SCREEN GUI
-- ══════════════════════════════════════════════════════
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name            = "QuincyHub"
ScreenGui.ResetOnSpawn    = false
ScreenGui.ZIndexBehavior  = Enum.ZIndexBehavior.Sibling
ScreenGui.IgnoreGuiInset  = true
ScreenGui.Parent          = PlayerGui

-- ══════════════════════════════════════════════════════
--  BOTÃO LOGO FLUTUANTE (sempre visível, arrastável)
-- ══════════════════════════════════════════════════════
local LogoBtn = Instance.new("TextButton")
LogoBtn.Name              = "LogoBtn"
LogoBtn.Size              = UDim2.new(0, 68, 0, 68)
LogoBtn.Position          = UDim2.new(0, 16, 0.5, -34)
LogoBtn.BackgroundColor3  = Color3.fromRGB(8, 12, 30)
LogoBtn.BorderSizePixel   = 0
LogoBtn.Text              = ""
LogoBtn.ZIndex            = 200
LogoBtn.ClipsDescendants  = true
LogoBtn.Parent            = ScreenGui

Instance.new("UICorner", LogoBtn).CornerRadius = UDim.new(1, 0)

local logoBtnStroke = Instance.new("UIStroke", LogoBtn)
logoBtnStroke.Color     = Color3.fromRGB(212, 170, 40)
logoBtnStroke.Thickness = 2.5

-- Logo dentro do botão (8 pétalas, tamanho 60)
BuildLogo(LogoBtn, 60)

-- ══════════════════════════════════════════════════════
--  ARRASTAR O BOTÃO LOGO  (funciona em mobile e PC)
-- ══════════════════════════════════════════════════════
do
    local dragging   = false
    local touchStart = nil   -- Vector2: posição inicial do toque/mouse
    local btnStart   = nil   -- Vector2: posição inicial do botão em offset
    local moved      = false -- detecta se foi arrastar ou clique

    LogoBtn.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.Touch
        or inp.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging  = true
            moved     = false
            touchStart = Vector2.new(inp.Position.X, inp.Position.Y)
            -- guarda a posição do botão em pixels absolutos
            local abs = LogoBtn.AbsolutePosition
            btnStart  = Vector2.new(abs.X, abs.Y)
        end
    end)

    UserInputService.InputChanged:Connect(function(inp)
        if not dragging then return end
        if inp.UserInputType ~= Enum.UserInputType.Touch
        and inp.UserInputType ~= Enum.UserInputType.MouseMovement then return end

        local cur    = Vector2.new(inp.Position.X, inp.Position.Y)
        local delta  = cur - touchStart
        if delta.Magnitude > 4 then moved = true end

        local newX = btnStart.X + delta.X
        local newY = btnStart.Y + delta.Y

        -- Limita dentro da tela
        local vp = workspace.CurrentCamera.ViewportSize
        newX = math.clamp(newX, 0, vp.X - LogoBtn.AbsoluteSize.X)
        newY = math.clamp(newY, 0, vp.Y - LogoBtn.AbsoluteSize.Y)

        LogoBtn.Position = UDim2.new(0, newX, 0, newY)
    end)

    UserInputService.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.Touch
        or inp.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
            -- Se não arrastou, considera clique → toggle UI
            if not moved then
                uiVisible = not uiVisible
                MainFrame.Visible = uiVisible
                if uiVisible then
                    MainFrame.Size = UDim2.new(0, 0, 0, 0)
                    TweenService:Create(MainFrame,
                        TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
                        {Size = UDim2.new(0, 310, 0, 400)}
                    ):Play()
                end
            end
        end
    end)
end

-- ══════════════════════════════════════════════════════
--  MAIN FRAME  (começa invisível)
-- ══════════════════════════════════════════════════════
local MainFrame = Instance.new("Frame")
MainFrame.Name             = "MainFrame"
MainFrame.Size             = UDim2.new(0, 310, 0, 400)
MainFrame.Position         = UDim2.new(0.5, -155, 0.5, -200)
MainFrame.BackgroundColor3 = Color3.fromRGB(8, 12, 30)
MainFrame.BorderSizePixel  = 0
MainFrame.ZIndex           = 10
MainFrame.Visible          = false   -- ← UI OCULTA AO INICIAR
MainFrame.Parent           = ScreenGui

Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 16)

local frameStroke = Instance.new("UIStroke", MainFrame)
frameStroke.Color     = Color3.fromRGB(212, 170, 40)
frameStroke.Thickness = 2

-- Gradiente de fundo
local bgGrad = Instance.new("UIGradient", MainFrame)
bgGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0,   Color3.fromRGB(13, 20, 48)),
    ColorSequenceKeypoint.new(1,   Color3.fromRGB(5,  8,  20)),
})
bgGrad.Rotation = 150

-- Barra superior dourada
local topBar = Instance.new("Frame", MainFrame)
topBar.Size              = UDim2.new(1, 0, 0, 4)
topBar.BackgroundColor3  = Color3.fromRGB(212, 170, 40)
topBar.BorderSizePixel   = 0
topBar.ZIndex            = 11
Instance.new("UICorner", topBar).CornerRadius = UDim.new(0, 16)

-- ── LOGO NO TOPO DA UI (86px) ──────────────────────────
local logoHolder = Instance.new("Frame", MainFrame)
logoHolder.Size              = UDim2.new(0, 86, 0, 86)
logoHolder.Position          = UDim2.new(0.5, -43, 0, 18)
logoHolder.BackgroundColor3  = Color3.fromRGB(8, 12, 30)
logoHolder.BorderSizePixel   = 0
logoHolder.ZIndex            = 12
logoHolder.ClipsDescendants  = true
Instance.new("UICorner", logoHolder).CornerRadius = UDim.new(1, 0)

local holderStroke = Instance.new("UIStroke", logoHolder)
holderStroke.Color     = Color3.fromRGB(212, 170, 40)
holderStroke.Thickness = 2.5

BuildLogo(logoHolder, 80)

-- ── TÍTULO ─────────────────────────────────────────────
local title = Instance.new("TextLabel", MainFrame)
title.Size               = UDim2.new(1, 0, 0, 30)
title.Position           = UDim2.new(0, 0, 0, 114)
title.BackgroundTransparency = 1
title.Text               = "QuincyHub"
title.TextColor3         = Color3.fromRGB(220, 178, 55)
title.TextSize           = 25
title.Font               = Enum.Font.GothamBold
title.TextXAlignment     = Enum.TextXAlignment.Center
title.ZIndex             = 12

local titleStroke = Instance.new("UIStroke", title)
titleStroke.Color     = Color3.fromRGB(100, 70, 0)
titleStroke.Thickness = 1

-- ── SUBTÍTULO ──────────────────────────────────────────
local sub = Instance.new("TextLabel", MainFrame)
sub.Size               = UDim2.new(1, 0, 0, 18)
sub.Position           = UDim2.new(0, 0, 0, 146)
sub.BackgroundTransparency = 1
sub.Text               = "v1.0  ·  Delta Mobile"
sub.TextColor3         = Color3.fromRGB(140, 110, 35)
sub.TextSize           = 12
sub.Font               = Enum.Font.Gotham
sub.TextXAlignment     = Enum.TextXAlignment.Center
sub.ZIndex             = 12

-- ── DIVISOR ────────────────────────────────────────────
local div = Instance.new("Frame", MainFrame)
div.Size              = UDim2.new(0.8, 0, 0, 1)
div.Position          = UDim2.new(0.1, 0, 0, 176)
div.BackgroundColor3  = Color3.fromRGB(212, 170, 40)
div.BackgroundTransparency = 0.55
div.BorderSizePixel   = 0
div.ZIndex            = 12

-- ── LABEL SEÇÃO ────────────────────────────────────────
local secLabel = Instance.new("TextLabel", MainFrame)
secLabel.Size               = UDim2.new(1, -32, 0, 18)
secLabel.Position           = UDim2.new(0, 16, 0, 190)
secLabel.BackgroundTransparency = 1
secLabel.Text               = "✦  MOVIMENTO"
secLabel.TextColor3         = Color3.fromRGB(170, 135, 45)
secLabel.TextSize           = 11
secLabel.Font               = Enum.Font.GothamBold
secLabel.TextXAlignment     = Enum.TextXAlignment.Left
secLabel.ZIndex             = 12

-- ── CARD DO FLY ────────────────────────────────────────
local flyCard = Instance.new("Frame", MainFrame)
flyCard.Size              = UDim2.new(1, -32, 0, 108)
flyCard.Position          = UDim2.new(0, 16, 0, 215)
flyCard.BackgroundColor3  = Color3.fromRGB(13, 19, 44)
flyCard.BorderSizePixel   = 0
flyCard.ZIndex            = 12
Instance.new("UICorner", flyCard).CornerRadius = UDim.new(0, 12)

local cardStroke = Instance.new("UIStroke", flyCard)
cardStroke.Color       = Color3.fromRGB(212, 170, 40)
cardStroke.Thickness   = 1
cardStroke.Transparency = 0.5

-- Ícone 🕊
local flyIcon = Instance.new("TextLabel", flyCard)
flyIcon.Size               = UDim2.new(0, 40, 0, 40)
flyIcon.Position           = UDim2.new(0, 10, 0.5, -20)
flyIcon.BackgroundTransparency = 1
flyIcon.Text               = "🕊"
flyIcon.TextSize           = 28
flyIcon.Font               = Enum.Font.GothamBold
flyIcon.TextColor3         = Color3.fromRGB(210, 165, 40)
flyIcon.ZIndex             = 13

local flyCardTitle = Instance.new("TextLabel", flyCard)
flyCardTitle.Size               = UDim2.new(0, 120, 0, 22)
flyCardTitle.Position           = UDim2.new(0, 58, 0, 18)
flyCardTitle.BackgroundTransparency = 1
flyCardTitle.Text               = "Fly Mode"
flyCardTitle.TextColor3         = Color3.fromRGB(230, 190, 70)
flyCardTitle.TextSize           = 15
flyCardTitle.Font               = Enum.Font.GothamBold
flyCardTitle.TextXAlignment     = Enum.TextXAlignment.Left
flyCardTitle.ZIndex             = 13

local flyCardDesc = Instance.new("TextLabel", flyCard)
flyCardDesc.Size               = UDim2.new(0, 150, 0, 16)
flyCardDesc.Position           = UDim2.new(0, 58, 0, 44)
flyCardDesc.BackgroundTransparency = 1
flyCardDesc.Text               = "Voa livremente pelo mapa"
flyCardDesc.TextColor3         = Color3.fromRGB(110, 90, 40)
flyCardDesc.TextSize           = 11
flyCardDesc.Font               = Enum.Font.Gotham
flyCardDesc.TextXAlignment     = Enum.TextXAlignment.Left
flyCardDesc.ZIndex             = 13

-- ══════════════════════════════════════════════════════
--  BOTÃO FLY  ← botão real, centralizado no card
-- ══════════════════════════════════════════════════════
local FlyButton = Instance.new("TextButton", flyCard)
FlyButton.Size              = UDim2.new(0, 82, 0, 34)
FlyButton.Position          = UDim2.new(1, -96, 0.5, -17)
FlyButton.BackgroundColor3  = Color3.fromRGB(212, 170, 40)
FlyButton.BorderSizePixel   = 0
FlyButton.Text              = "FLY"
FlyButton.TextColor3        = Color3.fromRGB(8, 12, 30)
FlyButton.TextSize          = 15
FlyButton.Font              = Enum.Font.GothamBold
FlyButton.ZIndex            = 14
FlyButton.AutoButtonColor   = false
Instance.new("UICorner", FlyButton).CornerRadius = UDim.new(0, 10)

local flyBtnGrad = Instance.new("UIGradient", FlyButton)
flyBtnGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 215, 80)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(175, 130, 20)),
})
flyBtnGrad.Rotation = 90

-- ── RODAPÉ ─────────────────────────────────────────────
local footer = Instance.new("TextLabel", MainFrame)
footer.Size               = UDim2.new(1, 0, 0, 18)
footer.Position           = UDim2.new(0, 0, 1, -24)
footer.BackgroundTransparency = 1
footer.Text               = "QuincyHub  ·  Made with 🌙"
footer.TextColor3         = Color3.fromRGB(70, 58, 25)
footer.TextSize           = 11
footer.Font               = Enum.Font.Gotham
footer.TextXAlignment     = Enum.TextXAlignment.Center
footer.ZIndex             = 12

-- ══════════════════════════════════════════════════════
--  LÓGICA DE VOO
-- ══════════════════════════════════════════════════════
local function GetChar()
    return LocalPlayer.Character
end

local function StartFly()
    local char = GetChar()
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChild("Humanoid")
    if not hrp or not hum then return end

    hum.PlatformStand = true

    local bv = Instance.new("BodyVelocity")
    bv.Name      = "QH_BV"
    bv.Velocity  = Vector3.zero
    bv.MaxForce  = Vector3.new(1e5, 1e5, 1e5)
    bv.Parent    = hrp

    local bg = Instance.new("BodyGyro")
    bg.Name      = "QH_BG"
    bg.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
    bg.P         = 9000
    bg.Parent    = hrp

    local cam = workspace.CurrentCamera

    flyConn = RunService.Heartbeat:Connect(function()
        local c = GetChar()
        if not c then return end
        local r = c:FindFirstChild("HumanoidRootPart")
        if not r then return end
        local bvr = r:FindFirstChild("QH_BV")
        local bgr = r:FindFirstChild("QH_BG")
        if not bvr or not bgr then return end

        local cf  = cam.CFrame
        local dir = Vector3.zero

        -- Teclado (PC)
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then dir += cf.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then dir -= cf.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then dir -= cf.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then dir += cf.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.yAxis end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then dir -= Vector3.yAxis end

        -- Mobile: usa MoveVector do Humanoid
        local h2 = c:FindFirstChild("Humanoid")
        if h2 then
            local mv = h2.MoveDirection
            if mv.Magnitude > 0.1 then
                dir += cf.LookVector * mv.Z * -1
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

    local char = GetChar()
    if char then
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChild("Humanoid")
        if hum then hum.PlatformStand = false end
        if hrp then
            local bv = hrp:FindFirstChild("QH_BV")
            local bg = hrp:FindFirstChild("QH_BG")
            if bv then bv:Destroy() end
            if bg then bg:Destroy() end
        end
    end
end

-- ══════════════════════════════════════════════════════
--  BOTÃO FLY – CLIQUE
-- ══════════════════════════════════════════════════════
FlyButton.Activated:Connect(function()
    isFlying = not isFlying

    if isFlying then
        -- ── ATIVO: fica verde ──
        flyBtnGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(80, 230, 130)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(22, 160, 70)),
        })
        FlyButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        FlyButton.Text       = "FLY ✓"

        TweenService:Create(flyCard, TweenInfo.new(0.25), {
            BackgroundColor3 = Color3.fromRGB(8, 28, 18)
        }):Play()

        StartFly()
    else
        -- ── INATIVO: volta dourado ──
        flyBtnGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 215, 80)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(175, 130, 20)),
        })
        FlyButton.TextColor3 = Color3.fromRGB(8, 12, 30)
        FlyButton.Text       = "FLY"

        TweenService:Create(flyCard, TweenInfo.new(0.25), {
            BackgroundColor3 = Color3.fromRGB(13, 19, 44)
        }):Play()

        StopFly()
    end
end)

-- ══════════════════════════════════════════════════════
--  RESET AO RESPAWN
-- ══════════════════════════════════════════════════════
LocalPlayer.CharacterAdded:Connect(function()
    isFlying = false
    if flyConn then flyConn:Disconnect(); flyConn = nil end
    FlyButton.Text       = "FLY"
    FlyButton.TextColor3 = Color3.fromRGB(8, 12, 30)
    flyBtnGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 215, 80)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(175, 130, 20)),
    })
    flyCard.BackgroundColor3 = Color3.fromRGB(13, 19, 44)
end)

print("✦ QuincyHub carregado! Clique no botão logo para abrir. ✦")
