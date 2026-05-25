-- ╔══════════════════════════════════════════╗
-- ║           QuincyHub Script               ║
-- ║         Compatible: Delta Mobile         ║
-- ╚══════════════════════════════════════════╝

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")
local Humanoid = Character:WaitForChild("Humanoid")

-- ══════════════════════════════════════
--  STATE
-- ══════════════════════════════════════
local isFlying = false
local flyConnection = nil
local flySpeed = 50
local uiVisible = true

-- ══════════════════════════════════════
--  LOGO SVG (inline as ImageLabel via DrawingLib fallback)
--  We embed the logo as a base64 encoded decal or use
--  a custom render. For Delta compatibility we use
--  a pure Roblox GUI approach with vector-like shapes.
-- ══════════════════════════════════════

-- ══════════════════════════════════════
--  SCREEN GUI
-- ══════════════════════════════════════
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "QuincyHub"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = PlayerGui

-- ══════════════════════════════════════
--  FLOATING LOGO BUTTON (always visible, draggable)
-- ══════════════════════════════════════
local LogoButton = Instance.new("ImageButton")
LogoButton.Name = "LogoToggle"
LogoButton.Size = UDim2.new(0, 64, 0, 64)
LogoButton.Position = UDim2.new(0, 20, 0.5, -32)
LogoButton.BackgroundColor3 = Color3.fromRGB(10, 15, 35)
LogoButton.BorderSizePixel = 0
LogoButton.Image = "" -- will be drawn with children
LogoButton.ZIndex = 100
LogoButton.Parent = ScreenGui

local LogoCorner = Instance.new("UICorner")
LogoCorner.CornerRadius = UDim.new(1, 0)
LogoCorner.Parent = LogoButton

local LogoStroke = Instance.new("UIStroke")
LogoStroke.Color = Color3.fromRGB(212, 170, 60)
LogoStroke.Thickness = 2.5
LogoStroke.Parent = LogoButton

-- Golden glow gradient inside logo button
local LogoGrad = Instance.new("UIGradient")
LogoGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 215, 80)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(180, 130, 20)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(212, 170, 60)),
})
LogoGrad.Rotation = 135
LogoGrad.Parent = LogoButton

-- Logo symbol (simplified QuincyHub emblem using rotated frames)
local function CreatePetal(parent, rotation, zindex)
    local petal = Instance.new("Frame")
    petal.Size = UDim2.new(0, 18, 0, 28)
    petal.Position = UDim2.new(0.5, -9, 0.5, -24)
    petal.BackgroundColor3 = Color3.fromRGB(10, 15, 35)
    petal.BorderSizePixel = 0
    petal.Rotation = rotation
    petal.ZIndex = zindex or 101
    petal.Parent = parent

    local pc = Instance.new("UICorner")
    pc.CornerRadius = UDim.new(0.5, 0)
    pc.Parent = petal
    return petal
end

for i = 0, 7 do
    CreatePetal(LogoButton, i * 45, 101)
end

-- ══════════════════════════════════════
--  MAIN UI FRAME
-- ══════════════════════════════════════
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 320, 0, 420)
MainFrame.Position = UDim2.new(0.5, -160, 0.5, -210)
MainFrame.BackgroundColor3 = Color3.fromRGB(8, 12, 28)
MainFrame.BorderSizePixel = 0
MainFrame.ZIndex = 10
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 18)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(212, 170, 60)
MainStroke.Thickness = 2
MainStroke.Parent = MainFrame

-- Background gradient
local BgGrad = Instance.new("UIGradient")
BgGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(12, 18, 42)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(5, 8, 20)),
})
BgGrad.Rotation = 160
BgGrad.Parent = MainFrame

-- Top accent bar
local TopBar = Instance.new("Frame")
TopBar.Size = UDim2.new(1, 0, 0, 3)
TopBar.Position = UDim2.new(0, 0, 0, 0)
TopBar.BackgroundColor3 = Color3.fromRGB(212, 170, 60)
TopBar.BorderSizePixel = 0
TopBar.ZIndex = 11
TopBar.Parent = MainFrame

local TopBarCorner = Instance.new("UICorner")
TopBarCorner.CornerRadius = UDim.new(0, 18)
TopBarCorner.Parent = TopBar

-- ── HEADER LOGO DISPLAY ──
local HeaderLogo = Instance.new("Frame")
HeaderLogo.Size = UDim2.new(0, 72, 0, 72)
HeaderLogo.Position = UDim2.new(0.5, -36, 0, 22)
HeaderLogo.BackgroundColor3 = Color3.fromRGB(10, 15, 35)
HeaderLogo.BorderSizePixel = 0
HeaderLogo.ZIndex = 12
HeaderLogo.Parent = MainFrame

local HLCorner = Instance.new("UICorner")
HLCorner.CornerRadius = UDim.new(1, 0)
HLCorner.Parent = HeaderLogo

local HLStroke = Instance.new("UIStroke")
HLStroke.Color = Color3.fromRGB(212, 170, 60)
HLStroke.Thickness = 2.5
HLStroke.Parent = HeaderLogo

local HLGrad = Instance.new("UIGradient")
HLGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 215, 80)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(180, 130, 20)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(212, 170, 60)),
})
HLGrad.Rotation = 135
HLGrad.Parent = HeaderLogo

-- Petals on header logo
for i = 0, 7 do
    local petal = Instance.new("Frame")
    petal.Size = UDim2.new(0, 20, 0, 32)
    petal.Position = UDim2.new(0.5, -10, 0.5, -26)
    petal.BackgroundColor3 = Color3.fromRGB(8, 12, 28)
    petal.BorderSizePixel = 0
    petal.Rotation = i * 45
    petal.ZIndex = 13
    petal.Parent = HeaderLogo

    local pc = Instance.new("UICorner")
    pc.CornerRadius = UDim.new(0.5, 0)
    pc.Parent = petal
end

-- ── TITLE ──
local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, 0, 0, 32)
TitleLabel.Position = UDim2.new(0, 0, 0, 106)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "QuincyHub"
TitleLabel.TextColor3 = Color3.fromRGB(212, 170, 60)
TitleLabel.TextSize = 26
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextXAlignment = Enum.TextXAlignment.Center
TitleLabel.ZIndex = 12
TitleLabel.Parent = MainFrame

-- Golden gradient on title via UIGradient (text color override)
local TitleStroke = Instance.new("UIStroke")
TitleStroke.Color = Color3.fromRGB(120, 80, 0)
TitleStroke.Thickness = 1
TitleStroke.Parent = TitleLabel

-- ── SUBTITLE / VERSION ──
local SubLabel = Instance.new("TextLabel")
SubLabel.Size = UDim2.new(1, 0, 0, 20)
SubLabel.Position = UDim2.new(0, 0, 0, 138)
SubLabel.BackgroundTransparency = 1
SubLabel.Text = "v1.0  •  Delta Mobile"
SubLabel.TextColor3 = Color3.fromRGB(150, 120, 40)
SubLabel.TextSize = 13
SubLabel.Font = Enum.Font.Gotham
SubLabel.TextXAlignment = Enum.TextXAlignment.Center
SubLabel.ZIndex = 12
SubLabel.Parent = MainFrame

-- ── DIVIDER ──
local Divider = Instance.new("Frame")
Divider.Size = UDim2.new(0.8, 0, 0, 1)
Divider.Position = UDim2.new(0.1, 0, 0, 172)
Divider.BackgroundColor3 = Color3.fromRGB(212, 170, 60)
Divider.BackgroundTransparency = 0.6
Divider.BorderSizePixel = 0
Divider.ZIndex = 12
Divider.Parent = MainFrame

-- ── FLY SECTION LABEL ──
local SectionLabel = Instance.new("TextLabel")
SectionLabel.Size = UDim2.new(1, -40, 0, 20)
SectionLabel.Position = UDim2.new(0, 20, 0, 190)
SectionLabel.BackgroundTransparency = 1
SectionLabel.Text = "✦  LOCOMOTION"
SectionLabel.TextColor3 = Color3.fromRGB(180, 145, 55)
SectionLabel.TextSize = 11
SectionLabel.Font = Enum.Font.GothamBold
SectionLabel.TextXAlignment = Enum.TextXAlignment.Left
SectionLabel.LetterSpacingOffset = 3
SectionLabel.ZIndex = 12
SectionLabel.Parent = MainFrame

-- ── FLY CARD ──
local FlyCard = Instance.new("Frame")
FlyCard.Size = UDim2.new(1, -40, 0, 100)
FlyCard.Position = UDim2.new(0, 20, 0, 218)
FlyCard.BackgroundColor3 = Color3.fromRGB(14, 20, 45)
FlyCard.BorderSizePixel = 0
FlyCard.ZIndex = 12
FlyCard.Parent = MainFrame

local FlyCardCorner = Instance.new("UICorner")
FlyCardCorner.CornerRadius = UDim.new(0, 12)
FlyCardCorner.Parent = FlyCard

local FlyCardStroke = Instance.new("UIStroke")
FlyCardStroke.Color = Color3.fromRGB(212, 170, 60)
FlyCardStroke.Thickness = 1
FlyCardStroke.Transparency = 0.5
FlyCardStroke.Parent = FlyCard

-- Fly card icon
local FlyIcon = Instance.new("TextLabel")
FlyIcon.Size = UDim2.new(0, 36, 0, 36)
FlyIcon.Position = UDim2.new(0, 14, 0.5, -18)
FlyIcon.BackgroundTransparency = 1
FlyIcon.Text = "🕊"
FlyIcon.TextSize = 26
FlyIcon.Font = Enum.Font.GothamBold
FlyIcon.TextColor3 = Color3.fromRGB(212, 170, 60)
FlyIcon.ZIndex = 13
FlyIcon.Parent = FlyCard

local FlyCardTitle = Instance.new("TextLabel")
FlyCardTitle.Size = UDim2.new(0, 120, 0, 22)
FlyCardTitle.Position = UDim2.new(0, 58, 0, 18)
FlyCardTitle.BackgroundTransparency = 1
FlyCardTitle.Text = "Fly Mode"
FlyCardTitle.TextColor3 = Color3.fromRGB(230, 195, 80)
FlyCardTitle.TextSize = 16
FlyCardTitle.Font = Enum.Font.GothamBold
FlyCardTitle.TextXAlignment = Enum.TextXAlignment.Left
FlyCardTitle.ZIndex = 13
FlyCardTitle.Parent = FlyCard

local FlyCardDesc = Instance.new("TextLabel")
FlyCardDesc.Size = UDim2.new(0, 140, 0, 18)
FlyCardDesc.Position = UDim2.new(0, 58, 0, 42)
FlyCardDesc.BackgroundTransparency = 1
FlyCardDesc.Text = "Voa livremente pelo mapa"
FlyCardDesc.TextColor3 = Color3.fromRGB(120, 100, 50)
FlyCardDesc.TextSize = 11
FlyCardDesc.Font = Enum.Font.Gotham
FlyCardDesc.TextXAlignment = Enum.TextXAlignment.Left
FlyCardDesc.ZIndex = 13
FlyCardDesc.Parent = FlyCard

-- ── FLY BUTTON ──
local FlyButton = Instance.new("TextButton")
FlyButton.Size = UDim2.new(0, 80, 0, 36)
FlyButton.Position = UDim2.new(1, -94, 0.5, -18)
FlyButton.BackgroundColor3 = Color3.fromRGB(212, 170, 60)
FlyButton.BorderSizePixel = 0
FlyButton.Text = "FLY"
FlyButton.TextColor3 = Color3.fromRGB(8, 12, 28)
FlyButton.TextSize = 15
FlyButton.Font = Enum.Font.GothamBold
FlyButton.ZIndex = 14
FlyButton.Parent = FlyCard

local FlyBtnCorner = Instance.new("UICorner")
FlyBtnCorner.CornerRadius = UDim.new(0, 10)
FlyBtnCorner.Parent = FlyButton

local FlyBtnGrad = Instance.new("UIGradient")
FlyBtnGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 220, 90)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(180, 135, 25)),
})
FlyBtnGrad.Rotation = 90
FlyBtnGrad.Parent = FlyButton

-- ── STATUS INDICATOR ──
local StatusDot = Instance.new("Frame")
StatusDot.Size = UDim2.new(0, 10, 0, 10)
StatusDot.Position = UDim2.new(0, 20, 1, -28)
StatusDot.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
StatusDot.BorderSizePixel = 0
StatusDot.ZIndex = 12
StatusDot.Parent = MainFrame

local SDCorner = Instance.new("UICorner")
SDCorner.CornerRadius = UDim.new(1, 0)
SDCorner.Parent = StatusDot

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(0, 200, 0, 18)
StatusLabel.Position = UDim2.new(0, 36, 1, -31)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Status: Idle"
StatusLabel.TextColor3 = Color3.fromRGB(120, 100, 60)
StatusLabel.TextSize = 12
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.ZIndex = 12
StatusLabel.Parent = MainFrame

-- ── FOOTER ──
local Footer = Instance.new("TextLabel")
Footer.Size = UDim2.new(1, 0, 0, 18)
Footer.Position = UDim2.new(0, 0, 1, -22)
Footer.BackgroundTransparency = 1
Footer.Text = "QuincyHub  •  Made with 🌙"
Footer.TextColor3 = Color3.fromRGB(80, 65, 30)
Footer.TextSize = 11
Footer.Font = Enum.Font.Gotham
Footer.TextXAlignment = Enum.TextXAlignment.Center
Footer.ZIndex = 12
Footer.Parent = MainFrame

-- ══════════════════════════════════════
--  FLY LOGIC
-- ══════════════════════════════════════
local function StartFly()
    isFlying = true
    Humanoid.PlatformStand = true

    local BodyVelocity = Instance.new("BodyVelocity")
    BodyVelocity.Name = "QuincyFlyVelocity"
    BodyVelocity.Velocity = Vector3.new(0, 0, 0)
    BodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    BodyVelocity.Parent = HumanoidRootPart

    local BodyGyro = Instance.new("BodyGyro")
    BodyGyro.Name = "QuincyFlyGyro"
    BodyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
    BodyGyro.P = 1e4
    BodyGyro.Parent = HumanoidRootPart

    local Camera = workspace.CurrentCamera

    flyConnection = RunService.Heartbeat:Connect(function()
        if not isFlying then return end

        local cf = Camera.CFrame
        local moveDir = Vector3.new(0, 0, 0)

        if UserInputService:IsKeyDown(Enum.KeyCode.W) then
            moveDir = moveDir + cf.LookVector
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then
            moveDir = moveDir - cf.LookVector
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then
            moveDir = moveDir - cf.RightVector
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then
            moveDir = moveDir + cf.RightVector
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
            moveDir = moveDir + Vector3.new(0, 1, 0)
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
            moveDir = moveDir - Vector3.new(0, 1, 0)
        end

        -- Touch / Mobile support
        local thumbstick = LocalPlayer.PlayerGui:FindFirstChild("TouchGui")
        if thumbstick then
            local MoveVector = LocalPlayer.Move
            if MoveVector then
                moveDir = moveDir + cf.LookVector * MoveVector.Z
                moveDir = moveDir + cf.RightVector * MoveVector.X
            end
        end

        if moveDir.Magnitude > 0 then
            moveDir = moveDir.Unit
        end

        BodyVelocity.Velocity = moveDir * flySpeed
        BodyGyro.CFrame = cf
    end)
end

local function StopFly()
    isFlying = false
    Humanoid.PlatformStand = false

    if flyConnection then
        flyConnection:Disconnect()
        flyConnection = nil
    end

    local bv = HumanoidRootPart:FindFirstChild("QuincyFlyVelocity")
    local bg = HumanoidRootPart:FindFirstChild("QuincyFlyGyro")

    if bv then bv:Destroy() end
    if bg then bg:Destroy() end
end

-- ══════════════════════════════════════
--  FLY BUTTON LOGIC
-- ══════════════════════════════════════
FlyButton.MouseButton1Click:Connect(function()
    isFlying = not isFlying

    if isFlying then
        -- Activate: green button
        FlyButton.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
        FlyBtnGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(74, 222, 128)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(22, 163, 74)),
        })
        FlyButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        FlyButton.Text = "FLY ✓"

        StatusDot.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
        StatusLabel.Text = "Status: Voando 🕊"
        StatusLabel.TextColor3 = Color3.fromRGB(34, 197, 94)

        TweenService:Create(FlyCard, TweenInfo.new(0.3), {
            BackgroundColor3 = Color3.fromRGB(10, 30, 20)
        }):Play()

        StartFly()
    else
        -- Deactivate: gold button
        FlyButton.BackgroundColor3 = Color3.fromRGB(212, 170, 60)
        FlyBtnGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 220, 90)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(180, 135, 25)),
        })
        FlyButton.TextColor3 = Color3.fromRGB(8, 12, 28)
        FlyButton.Text = "FLY"

        StatusDot.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
        StatusLabel.Text = "Status: Idle"
        StatusLabel.TextColor3 = Color3.fromRGB(120, 100, 60)

        TweenService:Create(FlyCard, TweenInfo.new(0.3), {
            BackgroundColor3 = Color3.fromRGB(14, 20, 45)
        }):Play()

        StopFly()
    end
end)

-- ══════════════════════════════════════
--  LOGO BUTTON: TOGGLE UI
-- ══════════════════════════════════════
LogoButton.MouseButton1Click:Connect(function()
    uiVisible = not uiVisible

    if uiVisible then
        MainFrame.Visible = true
        TweenService:Create(MainFrame, TweenInfo.new(0.25, Enum.EasingStyle.Back), {
            Size = UDim2.new(0, 320, 0, 420)
        }):Play()
    else
        TweenService:Create(MainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {
            Size = UDim2.new(0, 0, 0, 0)
        }):Play()

        task.delay(0.2, function()
            MainFrame.Visible = false
            MainFrame.Size = UDim2.new(0, 320, 0, 420)
        end)
    end
end)

-- ══════════════════════════════════════
--  LOGO BUTTON: DRAGGABLE
-- ══════════════════════════════════════
local dragging = false
local dragInput, dragStart, startPos

LogoButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = LogoButton.Position

        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

LogoButton.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement
    or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - dragStart
        LogoButton.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
    end
end)

-- ══════════════════════════════════════
--  RESPAWN: clean up fly
-- ══════════════════════════════════════
LocalPlayer.CharacterAdded:Connect(function(newChar)
    Character = newChar
    HumanoidRootPart = newChar:WaitForChild("HumanoidRootPart")
    Humanoid = newChar:WaitForChild("Humanoid")

    isFlying = false
    if flyConnection then
        flyConnection:Disconnect()
        flyConnection = nil
    end

    -- Reset button state
    FlyButton.BackgroundColor3 = Color3.fromRGB(212, 170, 60)
    FlyButton.TextColor3 = Color3.fromRGB(8, 12, 28)
    FlyButton.Text = "FLY"
    StatusDot.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
    StatusLabel.Text = "Status: Idle"
    StatusLabel.TextColor3 = Color3.fromRGB(120, 100, 60)
    FlyCard.BackgroundColor3 = Color3.fromRGB(14, 20, 45)
end)

-- ══════════════════════════════════════
--  STARTUP ANIMATION
-- ══════════════════════════════════════
MainFrame.Size = UDim2.new(0, 0, 0, 0)
MainFrame.Visible = true

TweenService:Create(MainFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
    Size = UDim2.new(0, 320, 0, 420)
}):Play()

-- Logo button pulse animation
local function PulseLogo()
    TweenService:Create(LogoButton, TweenInfo.new(1.2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {
        Size = UDim2.new(0, 68, 0, 68)
    }):Play()
end

PulseLogo()

print("✦ QuincyHub carregado com sucesso! ✦")
