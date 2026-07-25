-- ================================================
--  SYLAX HUB — Golf Auto Hole  v2.3
--  Key System + Tela de Idioma + GUI Principal
-- ================================================

local RS       = game:GetService("ReplicatedStorage")
local Players  = game:GetService("Players")
local UIS      = game:GetService("UserInputService")
local RunSvc   = game:GetService("RunService")
local TweenSvc = game:GetService("TweenService")
local lp       = Players.LocalPlayer

local GolfCfg    = require(RS.Shared.Config.Golf)
local HoleTarget = require(RS.Client.System.GolfStrike.HoleTarget)
local BallHndlr  = require(RS.Client.System.GolfStrike.BallHandler)
local GolfStrike = require(RS.Client.System.GolfStrike)

local MIN_SPD = GolfCfg.minStrikeSpeed
local MAX_SPD = GolfCfg.maxStrikeSpeed

-- ════════════════════════════════════════════════
--  IDIOMAS
-- ════════════════════════════════════════════════
local LANG = "pt"
local TR = {
    pt = {
        scriptOn   = "SCRIPT: ATIVADO",    scriptOff  = "SCRIPT: DESATIVADO",
        logTitle   = "LOG",                force      = "FORCA",
        holdStrike = "SEGURE PARA TACAR",  striking   = "TACANDO...",
        auto       = "AUTO",               autoStop   = "PARAR AUTO",
        version    = "SYLAX 1.0",
        eyeOn      = "Espectar: ON",       eyeOff     = "Espectar: OFF",
        noball     = "Sem bola.",          nohole     = "Sem buraco.",
        inhole     = "No buraco!",         outofrange = "Fora de alcance.",
        shot       = "Tacada!",            timeout    = "Timeout.",
        autoOn     = "Auto: ON",           autoOff    = "Auto: OFF",
        activated  = "Script ativado",     deactivated= "Script desativado",
        colors     = "CORES",              color1     = "COR",
        preview    = "PREVIA",             apply      = "APLICAR",
        close      = "FECHAR",
        ready      = "Pronto! Segure TACAR para tacar",
        premiumRequired = "Adquira a versão Premium",
        tooFarStrike = "Muito longe para seu boneco dar a tacada.",
    },
    en = {
        scriptOn   = "SCRIPT: ON",         scriptOff  = "SCRIPT: OFF",
        logTitle   = "LOG",                force      = "FORCE",
        holdStrike = "HOLD TO STRIKE",     striking   = "STRIKING...",
        auto       = "AUTO",               autoStop   = "STOP AUTO",
        version    = "SYLAX 1.0",
        eyeOn      = "Spectate: ON",       eyeOff     = "Spectate: OFF",
        noball     = "No ball found.",     nohole     = "No hole found.",
        inhole     = "In the hole!",       outofrange = "Out of range.",
        shot       = "Strike!",            timeout    = "Timeout.",
        autoOn     = "Auto: ON",           autoOff    = "Auto: OFF",
        activated  = "Script activated",   deactivated= "Script deactivated",
        colors     = "COLORS",             color1     = "COLOR",
        preview    = "PREVIEW",            apply      = "APPLY",
        close      = "CLOSE",
        ready      = "Ready! Hold STRIKE to hit",
        premiumRequired = "Get the Premium version",
        tooFarStrike = "Too far for your character to take the shot.",
    },
    es = {
        scriptOn   = "SCRIPT: ACTIVADO",   scriptOff  = "SCRIPT: DESACTIVADO",
        logTitle   = "LOG",                force      = "FUERZA",
        holdStrike = "MANTEN PARA GOLPEAR",striking   = "GOLPEANDO...",
        auto       = "AUTO",               autoStop   = "PARAR AUTO",
        version    = "SYLAX 1.0",
        eyeOn      = "Espectador: ON",     eyeOff     = "Espectador: OFF",
        noball     = "Sin pelota.",        nohole     = "Sin hoyo.",
        inhole     = "En el hoyo!",        outofrange = "Fuera de alcance.",
        shot       = "Golpe!",             timeout    = "Timeout.",
        autoOn     = "Auto: ON",           autoOff    = "Auto: OFF",
        activated  = "Script activado",    deactivated= "Script desactivado",
        colors     = "COLORES",            color1     = "COLOR",
        preview    = "VISTA PREVIA",       apply      = "APLICAR",
        close      = "CERRAR",
        ready      = "Listo! Manten GOLPEAR para tirar",
        premiumRequired = "Adquiere la versión Premium",
        tooFarStrike = "Demasiado lejos para que tu personaje golpee.",
    },
}
local function T(k) return (TR[LANG] or TR.pt)[k] or TR.pt[k] or k end

-- ════════════════════════════════════════════════
--  CORES BASE
-- ════════════════════════════════════════════════
local BLACK = Color3.new(0, 0, 0)
local WHITE = Color3.new(1, 1, 1)
local CT    = Color3.fromRGB(225, 255, 232)
local CR    = Color3.fromRGB(200, 30,  30)

local themeC1 = Color3.fromRGB(0, 190, 65)
local themeC2 = themeC1:Lerp(BLACK, 0.62)
local LOCKED_BG = Color3.fromRGB(24, 24, 24)
local LOCK_ICON = "rbxassetid://113617763566572"

local PALETTE = {
    Color3.fromRGB(0,   200, 80),
    Color3.fromRGB(0,   190, 255),
    Color3.fromRGB(60,  110, 255),
    Color3.fromRGB(150, 40,  255),
    Color3.fromRGB(255, 50,  200),
    Color3.fromRGB(255, 45,  45),
    Color3.fromRGB(255, 145, 0),
    Color3.fromRGB(250, 230, 0),
    Color3.fromRGB(200, 255, 210),
    Color3.fromRGB(0,   220, 185),
}

-- ════════════════════════════════════════════════
--  ESTADO GLOBAL
-- ════════════════════════════════════════════════
local running        = false
local scriptOn       = true
local lastReach      = nil
local spectating     = false
local strikePower    = 1.0
local holdActive     = false
local autoOn         = false
local eyeOn          = false
local minimized      = false
local sliderDragging = false
local panelDragging  = false
local dragStart      = Vector2.new()
local panelStart     = UDim2.new()
local colorGuiOpen   = false

-- ════════════════════════════════════════════════
--  ALCANÇABILIDADE
-- ════════════════════════════════════════════════
local function isReachable(ballPos, holePos)
    if not ballPos or not holePos then return false end
    local dx, dz = holePos.X - ballPos.X, holePos.Z - ballPos.Z
    if Vector3.new(dx, 0, dz).Magnitude < 0.01 then return true end
    local flatDir = Vector3.new(dx, 0, dz).Unit
    for a = 5, 60, 5 do
        local ok, res = pcall(HoleTarget.ComputeTarget, ballPos, flatDir, math.rad(a), 1)
        if ok and res and res.powerRatio and res.powerRatio >= 0 and res.powerRatio <= 1 then
            return true
        end
    end
    return false
end

-- ════════════════════════════════════════════════
--  LINHA NEON (beams)
-- ════════════════════════════════════════════════
local vFolder = workspace:FindFirstChild("_AutoHoleVisual")
if vFolder then vFolder:Destroy() end
vFolder = Instance.new("Folder"); vFolder.Name = "_AutoHoleVisual"; vFolder.Parent = workspace

local function makeAnchor()
    local p = Instance.new("Part")
    p.Size = Vector3.new(0.05,0.05,0.05); p.Anchored = true
    p.CanCollide = false; p.Transparency = 1; p.CastShadow = false
    p.Parent = vFolder; return p
end
local ballAnchor = makeAnchor(); local holeAnchor = makeAnchor()
local att0 = Instance.new("Attachment"); att0.Parent = ballAnchor
local att1 = Instance.new("Attachment"); att1.Parent = holeAnchor

local beamCore = Instance.new("Beam")
beamCore.Attachment0 = att0; beamCore.Attachment1 = att1
beamCore.Width0 = 0.08; beamCore.Width1 = 0.08; beamCore.FaceCamera = true
beamCore.LightEmission = 1; beamCore.LightInfluence = 0; beamCore.Segments = 1
beamCore.Transparency = NumberSequence.new(0)
beamCore.Color = ColorSequence.new(themeC1); beamCore.Enabled = false; beamCore.Parent = vFolder

local beamGlow = Instance.new("Beam")
beamGlow.Attachment0 = att0; beamGlow.Attachment1 = att1
beamGlow.Width0 = 0.55; beamGlow.Width1 = 0.55; beamGlow.FaceCamera = true
beamGlow.LightEmission = 1; beamGlow.LightInfluence = 0; beamGlow.Segments = 1
beamGlow.Transparency = NumberSequence.new({
    NumberSequenceKeypoint.new(0,0.55), NumberSequenceKeypoint.new(0.5,0.65), NumberSequenceKeypoint.new(1,0.80)
})
beamGlow.Color = ColorSequence.new(themeC2); beamGlow.Enabled = false; beamGlow.Parent = vFolder

-- ════════════════════════════════════════════════
--  ALERTA DE CANTO
-- ════════════════════════════════════════════════
local IMG_REACH = "rbxassetid://76907825856100"
local IMG_FAR   = "rbxassetid://105523667401348"
local NEON_RED  = Color3.fromRGB(255, 0, 0)

local oldA = lp.PlayerGui:FindFirstChild("HoleAlertGui"); if oldA then oldA:Destroy() end
local alertSG = Instance.new("ScreenGui")
alertSG.Name = "HoleAlertGui"; alertSG.ResetOnSpawn = false
alertSG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; alertSG.Parent = lp.PlayerGui

local alertFrame = Instance.new("Frame")
alertFrame.Size = UDim2.new(0,150,0,150); alertFrame.Position = UDim2.new(1,-166,0,12)
alertFrame.BackgroundTransparency = 1; alertFrame.BorderSizePixel = 0
alertFrame.Visible = false; alertFrame.Parent = alertSG
Instance.new("UICorner", alertFrame).CornerRadius = UDim.new(0,12)

local alertImg = Instance.new("ImageLabel")
alertImg.Size = UDim2.new(1,0,1,0); alertImg.BackgroundTransparency = 1
alertImg.Image = IMG_FAR; alertImg.ScaleType = Enum.ScaleType.Fit; alertImg.Parent = alertFrame

local pulseTween = nil
local function startPulse()
    if pulseTween then pulseTween:Cancel() end
    pulseTween = TweenSvc:Create(alertImg,
        TweenInfo.new(0.7, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true),
        { ImageTransparency = 0.25 }); pulseTween:Play()
end

-- ════════════════════════════════════════════════
--  ESPECTAR
-- ════════════════════════════════════════════════
local spectateConn = nil
local function stopSpectate()
    spectating = false
    if spectateConn then spectateConn:Disconnect(); spectateConn = nil end
    local char = lp.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            workspace.CurrentCamera.CameraSubject = hum
            workspace.CurrentCamera.CameraType = Enum.CameraType.Custom
        end
    end
end
local function startSpectate()
    spectating = true
    spectateConn = RunSvc.RenderStepped:Connect(function()
        if not spectating then return end
        local ok, bp = pcall(BallHndlr.GetMyBallPart)
        if ok and bp and bp.Parent then
            workspace.CurrentCamera.CameraSubject = bp
            workspace.CurrentCamera.CameraType = Enum.CameraType.Follow
        end
    end)
end

-- ════════════════════════════════════════════════
--  TACADA
-- ════════════════════════════════════════════════
local addLog = function(msg) print("[SylaxHub] " .. msg) end

local function doAutoHole()
    if not GolfStrike.IsBallStrikeable() then
        local t = 0
        repeat
            task.wait(0.05); t += 0.05
            if not running then return end
            if t > 20 then addLog("X "..T("timeout")); return end
        until GolfStrike.IsBallStrikeable()
    end
    local ballPos = BallHndlr.GetMyBallPosition()
    if not ballPos then addLog("X "..T("noball")); return end
    local holePos = HoleTarget.GetEndPointPosition()
    if not holePos then addLog("X "..T("nohole")); return end
    local dx, dz = holePos.X-ballPos.X, holePos.Z-ballPos.Z
    if Vector3.new(dx,0,dz).Magnitude < 0.01 then addLog("* "..T("inhole")); return end
    local flatDir = Vector3.new(dx,0,dz).Unit
    local angles = {}
    for a=45,5,-5 do table.insert(angles,a) end
    for a=50,60,5 do table.insert(angles,a) end
    local bestResult, bestAngle
    for _,ad in ipairs(angles) do
        if not running then return end
        local ok, res = pcall(HoleTarget.ComputeTarget, ballPos, flatDir, math.rad(ad), 1)
        if ok and res and res.powerRatio and res.powerRatio >= 0 and res.powerRatio <= 1 then
            bestResult = res; bestAngle = ad; break
        end
    end
    if not bestResult then addLog("X "..T("outofrange")); return end
    local pr = bestResult.powerRatio; local ar = math.rad(bestAngle)
    local dir = bestResult.flatDir or flatDir
    local spd = (MIN_SPD + (MAX_SPD-MIN_SPD)*pr) * strikePower
    local vel = Vector3.new(dir.X*spd*math.cos(ar), spd*math.sin(ar), dir.Z*spd*math.cos(ar))
    local rp  = math.clamp(pr*100*strikePower, 0, 115)
    addLog(string.format("* %d | %.0f%% | %s %.0f%%", bestAngle, rp, T("force"), strikePower*100))
    local sd = { velocity=vel, rawPowerPercent=rp, displayRawPowerPercent=rp, powerPercent=rp, angle=bestAngle, rawAngle=bestAngle }
    local ok2, err = pcall(GolfStrike.Strike, "ball", sd, nil, "ball")
    if ok2 then addLog("* "..T("shot"))
    else local ok3 = pcall(BallHndlr.OnStrike, sd, nil); addLog(ok3 and "* OK" or "X "..tostring(err)) end
end

-- ════════════════════════════════════════════════
--  FORWARD DECLARATIONS
-- ════════════════════════════════════════════════
local buildLangScreen  -- declarada antes para ser chamada pelo key system
local buildMainGui

-- ════════════════════════════════════════════════
--  KEY SYSTEM (tela 1)
-- ════════════════════════════════════════════════
local function buildKeyGui()
    local oldK = lp.PlayerGui:FindFirstChild("SylaxKeyGui"); if oldK then oldK:Destroy() end
    local keySG = Instance.new("ScreenGui", lp.PlayerGui)
    keySG.Name = "SylaxKeyGui"; keySG.ResetOnSpawn = false
    keySG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    keySG.DisplayOrder = 100

    -- Overlay escuro
    local backdrop = Instance.new("Frame", keySG)
    backdrop.Size = UDim2.new(1,0,1,0); backdrop.BackgroundColor3 = BLACK
    backdrop.BackgroundTransparency = 0.75; backdrop.BorderSizePixel = 0; backdrop.ZIndex = 100

    -- Container de partículas (atrás do painel)
    local pCont = Instance.new("Frame", backdrop)
    pCont.Size = UDim2.new(1,0,1,0); pCont.BackgroundTransparency = 1; pCont.ZIndex = 101

    -- ── Painel principal ─────────────────────────
    local CW, CH = 338, 418
    local panel = Instance.new("Frame", keySG)
    panel.Size = UDim2.new(0,CW,0,CH)
    panel.Position = UDim2.new(0.5,-CW/2, 0.5,-CH/2)
    panel.BackgroundColor3 = themeC1:Lerp(BLACK, 0.93)
    panel.BorderSizePixel = 0; panel.ZIndex = 110
    Instance.new("UICorner", panel).CornerRadius = UDim.new(0,20)

    -- Borda estática verde
    local pStroke = Instance.new("UIStroke", panel)
    pStroke.Color = themeC1; pStroke.Thickness = 1.5; pStroke.Transparency = 0.32

    -- Gradiente sutil no painel
    local pGrad = Instance.new("UIGradient", panel)
    pGrad.Color = ColorSequence.new(themeC1, themeC2); pGrad.Rotation = 135
    pGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0,0.85), NumberSequenceKeypoint.new(1,0.93)})

    -- Anel animado ao redor do painel
    local ringFrame = Instance.new("Frame", panel)
    ringFrame.Size = UDim2.new(1,8,1,8); ringFrame.Position = UDim2.new(0,-4,0,-4)
    ringFrame.BackgroundTransparency = 1; ringFrame.ZIndex = 109
    Instance.new("UICorner", ringFrame).CornerRadius = UDim.new(0,24)
    local ringStroke = Instance.new("UIStroke", ringFrame)
    ringStroke.Thickness = 2; ringStroke.Transparency = 0.20
    local ringGrad = Instance.new("UIGradient", ringStroke)
    ringGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0,   themeC1),
        ColorSequenceKeypoint.new(0.5, WHITE),
        ColorSequenceKeypoint.new(1,   themeC1),
    })
    ringGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0,   0.88),
        NumberSequenceKeypoint.new(0.22, 0.05),
        NumberSequenceKeypoint.new(0.78, 0.05),
        NumberSequenceKeypoint.new(1,   0.88),
    })

    -- ── Logo ──────────────────────────────────────
    local iconBg = Instance.new("Frame", panel)
    iconBg.Size = UDim2.new(0,62,0,62); iconBg.Position = UDim2.new(0.5,-31,0,20)
    iconBg.BackgroundColor3 = themeC1:Lerp(BLACK, 0.76)
    iconBg.BorderSizePixel = 0; iconBg.ZIndex = 112
    Instance.new("UICorner", iconBg).CornerRadius = UDim.new(0,15)

    -- Anel de brilho ao redor do ícone
    local iRing = Instance.new("Frame", iconBg)
    iRing.Size = UDim2.new(1,14,1,14); iRing.Position = UDim2.new(0,-7,0,-7)
    iRing.BackgroundTransparency = 1; iRing.ZIndex = 111
    Instance.new("UICorner", iRing).CornerRadius = UDim.new(0,22)
    local iRingStroke = Instance.new("UIStroke", iRing)
    iRingStroke.Color = themeC1; iRingStroke.Thickness = 3; iRingStroke.Transparency = 0.22
    local iRingGrad = Instance.new("UIGradient", iRingStroke)
    iRingGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, themeC1),
        ColorSequenceKeypoint.new(0.5, WHITE),
        ColorSequenceKeypoint.new(1, themeC1),
    })
    iRingGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0,   0.82),
        NumberSequenceKeypoint.new(0.25, 0.04),
        NumberSequenceKeypoint.new(0.75, 0.04),
        NumberSequenceKeypoint.new(1,   0.82),
    })

    local iconImg = Instance.new("ImageLabel", iconBg)
    iconImg.Size = UDim2.new(0.80,0,0.80,0); iconImg.Position = UDim2.new(0.10,0,0.10,0)
    iconImg.BackgroundTransparency = 1
    iconImg.Image = "rbxassetid://90423747596924"
    iconImg.ScaleType = Enum.ScaleType.Fit; iconImg.ZIndex = 113

    -- ── Título ────────────────────────────────────
    local titleLbl = Instance.new("TextLabel", panel)
    titleLbl.Size = UDim2.new(1,-20,0,26); titleLbl.Position = UDim2.new(0,10,0,90)
    titleLbl.BackgroundTransparency = 1; titleLbl.Text = "SYLAX HUB"
    titleLbl.TextSize = 22; titleLbl.Font = Enum.Font.GothamBlack
    titleLbl.TextColor3 = CT; titleLbl.TextXAlignment = Enum.TextXAlignment.Center
    titleLbl.ZIndex = 112
    local titleTextGrad = Instance.new("UIGradient", titleLbl)
    titleTextGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0,   CT),
        ColorSequenceKeypoint.new(0.54, CT),
        ColorSequenceKeypoint.new(0.55, themeC1),
        ColorSequenceKeypoint.new(1,   themeC1),
    })

    local subLbl = Instance.new("TextLabel", panel)
    subLbl.Size = UDim2.new(1,-20,0,16); subLbl.Position = UDim2.new(0,10,0,118)
    subLbl.BackgroundTransparency = 1; subLbl.Text = "Chave de Acesso Necessária"
    subLbl.TextSize = 11; subLbl.Font = Enum.Font.GothamSemibold
    subLbl.TextColor3 = themeC1:Lerp(WHITE, 0.20)
    subLbl.TextXAlignment = Enum.TextXAlignment.Center; subLbl.ZIndex = 112

    -- Separador
    local sep = Instance.new("Frame", panel)
    sep.Size = UDim2.new(1,-40,0,1); sep.Position = UDim2.new(0,20,0,144)
    sep.BackgroundColor3 = themeC1; sep.BackgroundTransparency = 0.52
    sep.BorderSizePixel = 0; sep.ZIndex = 112

    -- ── Input ─────────────────────────────────────
    local inputBg = Instance.new("Frame", panel)
    inputBg.Size = UDim2.new(1,-38,0,44); inputBg.Position = UDim2.new(0,19,0,157)
    inputBg.BackgroundColor3 = themeC1:Lerp(BLACK, 0.88)
    inputBg.BorderSizePixel = 0; inputBg.ZIndex = 112
    Instance.new("UICorner", inputBg).CornerRadius = UDim.new(0,11)
    local inputStroke = Instance.new("UIStroke", inputBg)
    inputStroke.Color = themeC1; inputStroke.Thickness = 1.2; inputStroke.Transparency = 0.55

    -- Ícone de cadeado
    local lockIco = Instance.new("TextLabel", inputBg)
    lockIco.Size = UDim2.new(0,24,1,0); lockIco.Position = UDim2.new(0,8,0,0)
    lockIco.BackgroundTransparency = 1; lockIco.Text = "🔑"
    lockIco.TextSize = 14; lockIco.Font = Enum.Font.GothamBold; lockIco.ZIndex = 114

    local inputBox = Instance.new("TextBox", inputBg)
    inputBox.Size = UDim2.new(1,-44,1,0); inputBox.Position = UDim2.new(0,36,0,0)
    inputBox.BackgroundTransparency = 1; inputBox.Text = ""
    inputBox.PlaceholderText = "Digite sua chave..."
    inputBox.TextColor3 = CT
    inputBox.PlaceholderColor3 = themeC1:Lerp(BLACK, 0.25)
    inputBox.TextSize = 13; inputBox.Font = Enum.Font.Gotham
    inputBox.TextXAlignment = Enum.TextXAlignment.Left
    inputBox.ClearTextOnFocus = false; inputBox.ZIndex = 114

    -- Contador de caracteres
    local charLbl = Instance.new("TextLabel", panel)
    charLbl.Size = UDim2.new(0,70,0,15); charLbl.Position = UDim2.new(1,-87,0,205)
    charLbl.BackgroundTransparency = 1; charLbl.Text = "0/50"
    charLbl.TextSize = 9; charLbl.Font = Enum.Font.Gotham
    charLbl.TextColor3 = themeC1:Lerp(BLACK, 0.35)
    charLbl.TextXAlignment = Enum.TextXAlignment.Right; charLbl.ZIndex = 112

    -- Status
    local statusLbl = Instance.new("TextLabel", panel)
    statusLbl.Size = UDim2.new(1,-38,0,28); statusLbl.Position = UDim2.new(0,19,0,224)
    statusLbl.BackgroundTransparency = 1; statusLbl.Text = ""
    statusLbl.TextSize = 11; statusLbl.Font = Enum.Font.GothamSemibold
    statusLbl.TextColor3 = themeC1
    statusLbl.TextXAlignment = Enum.TextXAlignment.Center
    statusLbl.TextWrapped = true; statusLbl.TextTransparency = 1; statusLbl.ZIndex = 112

    local function showStatus(msg, isErr, isOk)
        statusLbl.Text = msg
        statusLbl.TextColor3 = isOk and themeC1 or (isErr and Color3.fromRGB(220,55,55) or Color3.fromRGB(220,175,35))
        statusLbl.TextTransparency = 1
        TweenSvc:Create(statusLbl, TweenInfo.new(0.22), {TextTransparency = 0}):Play()
    end

    -- ── Botão VERIFICAR ───────────────────────────
    local verifyBtn = Instance.new("TextButton", panel)
    verifyBtn.Size = UDim2.new(1,-38,0,42); verifyBtn.Position = UDim2.new(0,19,0,258)
    verifyBtn.BackgroundColor3 = themeC1:Lerp(BLACK, 0.56)
    verifyBtn.BorderSizePixel = 0; verifyBtn.Text = "VERIFICAR CHAVE"
    verifyBtn.TextColor3 = CT; verifyBtn.TextSize = 13
    verifyBtn.Font = Enum.Font.GothamBold; verifyBtn.AutoButtonColor = false; verifyBtn.ZIndex = 113
    Instance.new("UICorner", verifyBtn).CornerRadius = UDim.new(0,11)
    local verifyGrad = Instance.new("UIGradient", verifyBtn)
    verifyGrad.Color = ColorSequence.new(themeC1:Lerp(BLACK,0.38), themeC1:Lerp(BLACK,0.68))
    verifyGrad.Rotation = 90
    verifyGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0,0.0), NumberSequenceKeypoint.new(1,0.18)})
    local verifyStroke = Instance.new("UIStroke", verifyBtn)
    verifyStroke.Color = themeC1; verifyStroke.Thickness = 1.5; verifyStroke.Transparency = 0.28

    -- Spinner de carregamento
    local spinner = Instance.new("Frame", verifyBtn)
    spinner.Size = UDim2.new(0,18,0,18); spinner.Position = UDim2.new(0.5,-9,0.5,-9)
    spinner.BackgroundColor3 = CT; spinner.BorderSizePixel = 0
    spinner.Visible = false; spinner.ZIndex = 115
    Instance.new("UICorner", spinner).CornerRadius = UDim.new(1,0)
    local spinGrad = Instance.new("UIGradient", spinner)
    spinGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0,0), NumberSequenceKeypoint.new(0.75,0.75), NumberSequenceKeypoint.new(1,1)})

    -- ── GET KEY + DISCORD ─────────────────────────
    local getKeyBtn = Instance.new("TextButton", panel)
    getKeyBtn.Size = UDim2.new(0.5,-24,0,38); getKeyBtn.Position = UDim2.new(0,19,0,312)
    getKeyBtn.BackgroundColor3 = themeC1:Lerp(BLACK, 0.66)
    getKeyBtn.BorderSizePixel = 0; getKeyBtn.Text = "🔑  GET KEY"
    getKeyBtn.TextColor3 = themeC1:Lerp(WHITE, 0.30)
    getKeyBtn.TextSize = 12; getKeyBtn.Font = Enum.Font.GothamBold
    getKeyBtn.AutoButtonColor = false; getKeyBtn.ZIndex = 113
    Instance.new("UICorner", getKeyBtn).CornerRadius = UDim.new(0,10)
    local gkStroke = Instance.new("UIStroke", getKeyBtn)
    gkStroke.Color = themeC1; gkStroke.Thickness = 1.2; gkStroke.Transparency = 0.45

    local discordBtn = Instance.new("TextButton", panel)
    discordBtn.Size = UDim2.new(0.5,-24,0,38); discordBtn.Position = UDim2.new(0.5,5,0,312)
    discordBtn.BackgroundColor3 = Color3.fromRGB(55,62,160)
    discordBtn.BorderSizePixel = 0; discordBtn.Text = "💬  DISCORD"
    discordBtn.TextColor3 = Color3.fromRGB(200,205,255)
    discordBtn.TextSize = 12; discordBtn.Font = Enum.Font.GothamBold
    discordBtn.AutoButtonColor = false; discordBtn.ZIndex = 113
    Instance.new("UICorner", discordBtn).CornerRadius = UDim.new(0,10)
    local dcStroke = Instance.new("UIStroke", discordBtn)
    dcStroke.Color = Color3.fromRGB(110,120,255); dcStroke.Thickness = 1.2; dcStroke.Transparency = 0.45

    -- Rodapé
    local footerLbl = Instance.new("TextLabel", panel)
    footerLbl.Size = UDim2.new(1,-20,0,16); footerLbl.Position = UDim2.new(0,10,0,366)
    footerLbl.BackgroundTransparency = 1; footerLbl.Text = "SYLAX 1.0  •  Golf Auto Hole"
    footerLbl.TextSize = 9; footerLbl.Font = Enum.Font.Gotham
    footerLbl.TextColor3 = themeC1:Lerp(BLACK, 0.42)
    footerLbl.TextXAlignment = Enum.TextXAlignment.Center; footerLbl.ZIndex = 112

    -- Linha divisória do rodapé
    local footSep = Instance.new("Frame", panel)
    footSep.Size = UDim2.new(1,-40,0,1); footSep.Position = UDim2.new(0,20,0,358)
    footSep.BackgroundColor3 = themeC1; footSep.BackgroundTransparency = 0.68
    footSep.BorderSizePixel = 0; footSep.ZIndex = 112

    -- ── Partículas verdes ─────────────────────────
    local GREEN_SH = {
        Color3.fromRGB(0,220,80), Color3.fromRGB(0,190,65),
        Color3.fromRGB(50,230,120), Color3.fromRGB(30,255,100),
    }
    local particles = {}; local particlesDead = false

    local function spawnParticle()
        if particlesDead or not pCont.Parent then return end
        local sz = math.random(5,18)
        local pf = Instance.new("Frame", pCont)
        pf.Size = UDim2.new(0,sz,0,sz)
        pf.Position = UDim2.new(math.random()*1.4-0.2, 0, 1.15, 0)
        pf.BackgroundColor3 = GREEN_SH[math.random(#GREEN_SH)]
        pf.BackgroundTransparency = math.random(55,82)/100
        pf.BorderSizePixel = 0; pf.ZIndex = 102
        Instance.new("UICorner", pf).CornerRadius = UDim.new(1,0)
        table.insert(particles, {
            frame=pf, vx=(math.random()-0.5)*0.0028, vy=-math.random(14,42)/10000,
            created=tick(), lifetime=math.random(22,52),
            origSz=sz, wobble=math.random()*math.pi*2, pulse=math.random()*math.pi*2,
        })
    end

    local partConn = RunSvc.Heartbeat:Connect(function()
        if particlesDead then return end
        for i=#particles,1,-1 do
            local pd = particles[i]
            if not pd.frame.Parent then table.remove(particles,i)
            else
                local pos = pd.frame.Position
                if pos.Y.Scale < -0.25 or (tick()-pd.created) > pd.lifetime then
                    pd.frame:Destroy(); table.remove(particles,i)
                else
                    local nx = pos.X.Scale + pd.vx + math.sin(tick()*1.4+pd.wobble)*0.0013
                    local ny = pos.Y.Scale + pd.vy
                    if nx < -0.2 then nx = 1.2 elseif nx > 1.2 then nx = -0.2 end
                    pd.frame.Position = UDim2.new(nx,0,ny,0)
                    local b = math.sin(tick()*2.0+pd.pulse)*0.07+1
                    pd.frame.Size = UDim2.new(0,pd.origSz*b,0,pd.origSz*b)
                end
            end
        end
    end)

    task.spawn(function()
        for i=1,18 do spawnParticle(); task.wait(0.06) end
        while not particlesDead and pCont.Parent do
            if #particles < 36 then spawnParticle() end
            task.wait(0.55)
        end
        partConn:Disconnect()
    end)

    -- ── Animações de borda ────────────────────────
    task.spawn(function()
        while not particlesDead and ringFrame.Parent do
            local tw = TweenSvc:Create(ringGrad, TweenInfo.new(4, Enum.EasingStyle.Linear),
                {Rotation = ringGrad.Rotation + 360}); tw:Play(); tw.Completed:Wait()
            if ringFrame.Parent then ringGrad.Rotation = ringGrad.Rotation % 360 end
            task.wait(0.04)
        end
    end)
    task.spawn(function()
        while not particlesDead and iRing.Parent do
            local tw = TweenSvc:Create(iRingGrad, TweenInfo.new(3, Enum.EasingStyle.Linear),
                {Rotation = iRingGrad.Rotation + 360}); tw:Play(); tw.Completed:Wait()
            if iRing.Parent then iRingGrad.Rotation = iRingGrad.Rotation % 360 end
            task.wait(0.04)
        end
    end)

    -- ── Eventos do input ──────────────────────────
    inputBox:GetPropertyChangedSignal("Text"):Connect(function()
        local txt = inputBox.Text
        if #txt > 50 then inputBox.Text = txt:sub(1,50) end
        charLbl.Text = #inputBox.Text.."/50"
        charLbl.TextColor3 = (#inputBox.Text >= 50) and Color3.fromRGB(220,55,55)
            or themeC1:Lerp(BLACK, 0.35)
    end)
    inputBox.Focused:Connect(function()
        TweenSvc:Create(inputStroke, TweenInfo.new(0.18), {Transparency=0.05}):Play()
    end)
    inputBox.FocusLost:Connect(function()
        TweenSvc:Create(inputStroke, TweenInfo.new(0.18), {Transparency=0.55}):Play()
    end)

    -- ── Lógica de verificação ─────────────────────
    local isLoading = false
    local spinTween = nil

    local function doVerify()
        if isLoading then return end
        local key = inputBox.Text
        if key == "" then
            showStatus("⚠  Digite sua chave de acesso", true, false)
            return
        end
        isLoading = true; verifyBtn.Text = ""; spinner.Visible = true
        spinTween = TweenSvc:Create(spinner,
            TweenInfo.new(0.7, Enum.EasingStyle.Linear, Enum.EasingDirection.InOut, -1),
            {Rotation=360}); spinTween:Play()
        showStatus("Validando chave...", false, false)

        task.spawn(function()
            task.wait(1.8)
            if spinTween then spinTween:Cancel(); spinTween = nil end
            spinner.Visible = false; isLoading = false
            if key == "sylax012026" then
                showStatus("✓  Acesso liberado!", false, true)
                verifyBtn.Text = "VERIFICAR CHAVE"
                task.wait(0.50)
                -- Transição: fade out e abre tela de idioma
                particlesDead = true
                partConn:Disconnect()
                TweenSvc:Create(backdrop, TweenInfo.new(0.26, Enum.EasingStyle.Quint),
                    {BackgroundTransparency=1}):Play()
                TweenSvc:Create(panel, TweenInfo.new(0.26, Enum.EasingStyle.Quint),
                    {Position=UDim2.new(0.5,-CW/2, 0.45,-CH/2)}):Play()
                task.delay(0.28, function() keySG:Destroy(); buildLangScreen() end)
            else
                showStatus("✕  Chave inválida.", true, false)
                verifyBtn.Text = "VERIFICAR CHAVE"
            end
        end)
    end

    verifyBtn.MouseButton1Click:Connect(doVerify)
    UIS.InputBegan:Connect(function(inp, gp)
        if gp or not keySG.Parent then return end
        if inp.KeyCode == Enum.KeyCode.Return and inputBox:IsFocused() then doVerify() end
    end)

    -- Hover nos botões
    verifyBtn.MouseEnter:Connect(function()
        TweenSvc:Create(verifyBtn, TweenInfo.new(0.14), {BackgroundColor3=themeC1:Lerp(BLACK,0.38)}):Play()
    end)
    verifyBtn.MouseLeave:Connect(function()
        TweenSvc:Create(verifyBtn, TweenInfo.new(0.18), {BackgroundColor3=themeC1:Lerp(BLACK,0.56)}):Play()
    end)
    getKeyBtn.MouseEnter:Connect(function()
        TweenSvc:Create(getKeyBtn, TweenInfo.new(0.14), {BackgroundColor3=themeC1:Lerp(BLACK,0.46)}):Play()
        gkStroke.Transparency = 0.10
    end)
    getKeyBtn.MouseLeave:Connect(function()
        TweenSvc:Create(getKeyBtn, TweenInfo.new(0.18), {BackgroundColor3=themeC1:Lerp(BLACK,0.66)}):Play()
        gkStroke.Transparency = 0.45
    end)
    discordBtn.MouseEnter:Connect(function()
        TweenSvc:Create(discordBtn, TweenInfo.new(0.14), {BackgroundColor3=Color3.fromRGB(75,82,200)}):Play()
        dcStroke.Transparency = 0.10
    end)
    discordBtn.MouseLeave:Connect(function()
        TweenSvc:Create(discordBtn, TweenInfo.new(0.18), {BackgroundColor3=Color3.fromRGB(55,62,160)}):Play()
        dcStroke.Transparency = 0.45
    end)

    -- Get Key copia link
    getKeyBtn.MouseButton1Click:Connect(function()
        showStatus("🔗  Link copiado! Acesse para pegar a key.", false, true)
        pcall(function() if setclipboard then setclipboard("https://link-center.net/4792489/avqLmZkM336o") end end)
    end)
    -- Discord copia invite
    discordBtn.MouseButton1Click:Connect(function()
        showStatus("💬  Link Discord copiado!", false, true)
        pcall(function() if setclipboard then setclipboard("https://discord.gg/D8v4YQACc") end end)
    end)

    -- ── Animação de entrada ───────────────────────
    panel.Size = UDim2.new(0,0,0,0); panel.BackgroundTransparency = 1
    backdrop.BackgroundTransparency = 1
    TweenSvc:Create(backdrop, TweenInfo.new(0.24, Enum.EasingStyle.Quad),
        {BackgroundTransparency=0.75}):Play()
    task.wait(0.08)
    TweenSvc:Create(panel, TweenInfo.new(0.36, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
        {Size=UDim2.new(0,CW,0,CH), BackgroundTransparency=0}):Play()
    task.wait(0.48)
    inputBox:CaptureFocus()
end

-- ════════════════════════════════════════════════
--  TELA DE SELEÇÃO DE IDIOMA (tela 2)
-- ════════════════════════════════════════════════
buildLangScreen = function()
    local oldLang = lp.PlayerGui:FindFirstChild("SylaxLangGui"); if oldLang then oldLang:Destroy() end
    local langSG = Instance.new("ScreenGui", lp.PlayerGui)
    langSG.Name = "SylaxLangGui"; langSG.ResetOnSpawn = false
    langSG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

    local overlay = Instance.new("Frame", langSG)
    overlay.Size = UDim2.new(1,0,1,0); overlay.BackgroundColor3 = BLACK
    overlay.BackgroundTransparency = 0.18; overlay.BorderSizePixel = 0; overlay.ZIndex = 1

    local CARD_W = 330; local CARD_H = 104; local CARD_GAP = 10
    local TITLE_H = 62
    local CONTAINER_H = TITLE_H + 3*(CARD_H + CARD_GAP) - CARD_GAP

    local container = Instance.new("Frame", langSG)
    container.Size = UDim2.new(0,CARD_W,0,CONTAINER_H)
    container.Position = UDim2.new(0.5,-CARD_W/2, 0.5,-CONTAINER_H/2)
    container.BackgroundTransparency = 1; container.ZIndex = 2

    -- Título
    local titleMain = Instance.new("TextLabel", container)
    titleMain.Size = UDim2.new(1,0,0,36); titleMain.Position = UDim2.new(0,0,0,0)
    titleMain.BackgroundTransparency = 1; titleMain.Text = "⛳  SYLAX HUB"
    titleMain.TextSize = 27; titleMain.Font = Enum.Font.GothamBlack
    titleMain.TextColor3 = WHITE; titleMain.ZIndex = 3
    titleMain.TextStrokeTransparency = 0.45; titleMain.TextStrokeColor3 = BLACK

    local titleSub = Instance.new("TextLabel", container)
    titleSub.Size = UDim2.new(1,0,0,16); titleSub.Position = UDim2.new(0,0,0,38)
    titleSub.BackgroundTransparency = 1; titleSub.Text = "Select  ·  Selecione  ·  Selecciona"
    titleSub.TextSize = 10; titleSub.Font = Enum.Font.GothamSemibold
    titleSub.TextColor3 = Color3.fromRGB(155,155,155); titleSub.ZIndex = 3

    -- Dados dos idiomas
    local LANG_DATA = {
        { code="pt", flag="🇧🇷", label="PORTUGUÊS", img="rbxassetid://102667164652557" },
        { code="en", flag="🇺🇸", label="ENGLISH",   img="rbxassetid://84588457966864"  },
        { code="es", flag="🇪🇸", label="ESPAÑOL",   img="rbxassetid://139681142849800" },
    }

    for i, ld in ipairs(LANG_DATA) do
        local yPos = TITLE_H + (i-1)*(CARD_H + CARD_GAP)

        local card = Instance.new("Frame", container)
        card.Name = "Card_"..ld.code
        card.Size = UDim2.new(1,0,0,CARD_H); card.Position = UDim2.new(0,0,0,yPos)
        card.BackgroundColor3 = BLACK; card.BorderSizePixel = 0
        card.ClipsDescendants = true; card.ZIndex = 3
        Instance.new("UICorner", card).CornerRadius = UDim.new(0,16)

        local imgBg = Instance.new("ImageLabel", card)
        imgBg.Size = UDim2.new(1,0,1,0); imgBg.BackgroundTransparency = 1
        imgBg.Image = ld.img; imgBg.ScaleType = Enum.ScaleType.Crop
        imgBg.ImageTransparency = 0.10; imgBg.ZIndex = 3

        -- Véu lateral
        local veil = Instance.new("Frame", card)
        veil.Size = UDim2.new(1,0,1,0); veil.BackgroundColor3 = BLACK
        veil.BackgroundTransparency = 1; veil.BorderSizePixel = 0; veil.ZIndex = 4
        local veilG = Instance.new("UIGradient", veil)
        veilG.Color = ColorSequence.new(BLACK, BLACK)
        veilG.Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0,   0.10),
            NumberSequenceKeypoint.new(0.28, 0.72),
            NumberSequenceKeypoint.new(0.72, 0.72),
            NumberSequenceKeypoint.new(1,   0.10),
        })
        veilG.Rotation = 0

        -- Véu inferior
        local veilBot = Instance.new("Frame", card)
        veilBot.Size = UDim2.new(1,0,1,0); veilBot.BackgroundColor3 = BLACK
        veilBot.BackgroundTransparency = 1; veilBot.BorderSizePixel = 0; veilBot.ZIndex = 5
        local veilBG = Instance.new("UIGradient", veilBot)
        veilBG.Color = ColorSequence.new(BLACK, BLACK)
        veilBG.Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0,   0.92),
            NumberSequenceKeypoint.new(0.6,  0.74),
            NumberSequenceKeypoint.new(1,   0.05),
        })
        veilBG.Rotation = 90

        local cardStroke = Instance.new("UIStroke", card)
        cardStroke.Color = Color3.fromRGB(65,65,65); cardStroke.Thickness = 1.8; cardStroke.Transparency = 0.20

        local flagLbl = Instance.new("TextLabel", card)
        flagLbl.Size = UDim2.new(0,50,1,0); flagLbl.Position = UDim2.new(0,13,0,0)
        flagLbl.BackgroundTransparency = 1; flagLbl.Text = ld.flag
        flagLbl.TextSize = 36; flagLbl.Font = Enum.Font.GothamBold
        flagLbl.TextColor3 = WHITE; flagLbl.ZIndex = 7

        local nameLbl = Instance.new("TextLabel", card)
        nameLbl.Size = UDim2.new(1,-90,0,CARD_H); nameLbl.Position = UDim2.new(0,74,0,0)
        nameLbl.BackgroundTransparency = 1; nameLbl.Text = ld.label
        nameLbl.TextSize = 22; nameLbl.Font = Enum.Font.GothamBlack
        nameLbl.TextColor3 = WHITE; nameLbl.ZIndex = 7
        nameLbl.TextXAlignment = Enum.TextXAlignment.Left
        nameLbl.TextStrokeTransparency = 0.50; nameLbl.TextStrokeColor3 = BLACK

        local arrowLbl = Instance.new("TextLabel", card)
        arrowLbl.Size = UDim2.new(0,26,1,0); arrowLbl.Position = UDim2.new(1,-36,0,0)
        arrowLbl.BackgroundTransparency = 1; arrowLbl.Text = "›"
        arrowLbl.TextSize = 42; arrowLbl.Font = Enum.Font.GothamBold
        arrowLbl.TextColor3 = Color3.fromRGB(200,200,200); arrowLbl.ZIndex = 7

        local hitBtn = Instance.new("TextButton", card)
        hitBtn.Size = UDim2.new(1,0,1,0); hitBtn.BackgroundTransparency = 1
        hitBtn.Text = ""; hitBtn.ZIndex = 8; hitBtn.BorderSizePixel = 0

        hitBtn.MouseEnter:Connect(function()
            TweenSvc:Create(imgBg, TweenInfo.new(0.14), {ImageTransparency=0}):Play()
            cardStroke.Color = themeC1; cardStroke.Transparency = 0; cardStroke.Thickness = 2.6
            TweenSvc:Create(nameLbl, TweenInfo.new(0.14), {TextColor3=themeC1}):Play()
            TweenSvc:Create(arrowLbl, TweenInfo.new(0.14), {TextColor3=themeC1}):Play()
        end)
        hitBtn.MouseLeave:Connect(function()
            TweenSvc:Create(imgBg, TweenInfo.new(0.20), {ImageTransparency=0.10}):Play()
            cardStroke.Color = Color3.fromRGB(65,65,65); cardStroke.Transparency = 0.20; cardStroke.Thickness = 1.8
            TweenSvc:Create(nameLbl,  TweenInfo.new(0.20), {TextColor3=WHITE}):Play()
            TweenSvc:Create(arrowLbl, TweenInfo.new(0.20), {TextColor3=Color3.fromRGB(200,200,200)}):Play()
        end)
        hitBtn.MouseButton1Click:Connect(function()
            LANG = ld.code
            TweenSvc:Create(overlay,    TweenInfo.new(0.26, Enum.EasingStyle.Quint), {BackgroundTransparency=1}):Play()
            TweenSvc:Create(container,  TweenInfo.new(0.26, Enum.EasingStyle.Quint),
                {Position=UDim2.new(0.5,-CARD_W/2, 0.44,-CONTAINER_H/2)}):Play()
            task.delay(0.28, function() langSG:Destroy(); buildMainGui() end)
        end)
    end
end

-- ════════════════════════════════════════════════
--  GUI PRINCIPAL (tela 3)
-- ════════════════════════════════════════════════
local renderConn = nil
local mainSG     = nil

buildMainGui = function()
    local oldG = lp.PlayerGui:FindFirstChild("AutoHoleGui"); if oldG then oldG:Destroy() end

    local PW = 232; local HH = 46
    local FH = HH + 290

    mainSG = Instance.new("ScreenGui")
    mainSG.Name = "AutoHoleGui"; mainSG.ResetOnSpawn = false
    mainSG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; mainSG.Parent = lp.PlayerGui

    local Wrapper = Instance.new("Frame")
    Wrapper.Name = "Wrapper"; Wrapper.Size = UDim2.new(0,PW,0,FH)
    Wrapper.Position = UDim2.new(0,16,0.5,-FH/2)
    Wrapper.BackgroundColor3 = themeC1:Lerp(BLACK, 0.94)
    Wrapper.BorderSizePixel = 0; Wrapper.ClipsDescendants = true; Wrapper.Parent = mainSG
    Instance.new("UICorner", Wrapper).CornerRadius = UDim.new(0,15)

    local wrapGrad = Instance.new("UIGradient", Wrapper)
    wrapGrad.Color = ColorSequence.new(themeC1, themeC2); wrapGrad.Rotation = 135
    wrapGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0,0.82), NumberSequenceKeypoint.new(1,0.91)})

    local panelStroke = Instance.new("UIStroke", Wrapper)
    panelStroke.Color = themeC1; panelStroke.Thickness = 1.6; panelStroke.Transparency = 0.10

    -- ── Header ───────────────────────────────────
    local Bar = Instance.new("Frame", Wrapper)
    Bar.Name = "Header"; Bar.Size = UDim2.new(1,0,0,HH)
    Bar.BackgroundColor3 = themeC1:Lerp(BLACK, 0.88); Bar.BorderSizePixel = 0; Bar.ZIndex = 3

    local barGrad = Instance.new("UIGradient", Bar)
    barGrad.Color = ColorSequence.new(themeC1, themeC2); barGrad.Rotation = 90
    barGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0,0.72), NumberSequenceKeypoint.new(1,0.88)})

    local hSep = Instance.new("Frame", Bar)
    hSep.Size = UDim2.new(1,0,0,1); hSep.Position = UDim2.new(0,0,1,-1)
    hSep.BackgroundColor3 = themeC1; hSep.BorderSizePixel = 0; hSep.ZIndex = 4

    local Ico = Instance.new("TextLabel", Bar)
    Ico.Size = UDim2.new(0,28,0,28); Ico.Position = UDim2.new(0,7,0.5,-14)
    Ico.BackgroundTransparency = 1; Ico.Text = "⛳"; Ico.TextSize = 19
    Ico.Font = Enum.Font.GothamBold; Ico.ZIndex = 4

    local TtlS = Instance.new("TextLabel", Bar)
    TtlS.Size = UDim2.new(0,47,0,19); TtlS.Position = UDim2.new(0,37,0,7)
    TtlS.BackgroundTransparency = 1; TtlS.Text = "SYLAX"; TtlS.TextSize = 13
    TtlS.Font = Enum.Font.GothamBlack; TtlS.TextColor3 = CT
    TtlS.TextXAlignment = Enum.TextXAlignment.Left; TtlS.ZIndex = 4

    local TtlH = Instance.new("TextLabel", Bar)
    TtlH.Size = UDim2.new(0,34,0,19); TtlH.Position = UDim2.new(0,82,0,7)
    TtlH.BackgroundTransparency = 1; TtlH.Text = "HUB"; TtlH.TextSize = 13
    TtlH.Font = Enum.Font.GothamBlack; TtlH.TextColor3 = themeC1
    TtlH.TextXAlignment = Enum.TextXAlignment.Left; TtlH.ZIndex = 4

    local function makeHdrBtn(xOff, label, tsz)
        local b = Instance.new("TextButton", Bar)
        b.Size = UDim2.new(0,22,0,22); b.Position = UDim2.new(1,xOff,0.5,-11)
        b.BackgroundColor3 = themeC1:Lerp(BLACK, 0.78)
        b.Text = label; b.TextSize = tsz or 12; b.Font = Enum.Font.GothamBold
        b.TextColor3 = CT; b.BorderSizePixel = 0; b.ZIndex = 5
        Instance.new("UICorner", b).CornerRadius = UDim.new(0,6)
        return b
    end
    local EyeBtn   = makeHdrBtn(-116, "👁",  12)
    local ColorBtn = makeHdrBtn(-90,  "🎨", 12)
    local MinBtn   = makeHdrBtn(-64,  "—",   12)
    local Xbtn     = makeHdrBtn(-38,  "✕",  10)
    Xbtn.BackgroundColor3 = CR

    -- Botão de cores bloqueado para usuários sem a versão Premium.
    local colorLock = Instance.new("ImageLabel", ColorBtn)
    colorLock.Size = UDim2.new(0,14,0,14)
    colorLock.Position = UDim2.new(0.5,-7,0.5,-7)
    colorLock.BackgroundTransparency = 1
    colorLock.Image = LOCK_ICON
    colorLock.ScaleType = Enum.ScaleType.Fit
    colorLock.ZIndex = 7
    ColorBtn.Text = ""
    ColorBtn.BackgroundColor3 = LOCKED_BG

    -- ── Conteúdo ─────────────────────────────────
    local Content = Instance.new("Frame", Wrapper)
    Content.Name = "Content"; Content.Size = UDim2.new(1,0,1,-HH)
    Content.Position = UDim2.new(0,0,0,HH); Content.BackgroundTransparency = 1

    local listLayout = Instance.new("UIListLayout", Content)
    listLayout.FillDirection = Enum.FillDirection.Vertical
    listLayout.SortOrder = Enum.SortOrder.LayoutOrder; listLayout.Padding = UDim.new(0,6)

    local listPad = Instance.new("UIPadding", Content)
    listPad.PaddingTop = UDim.new(0,7); listPad.PaddingBottom = UDim.new(0,7)
    listPad.PaddingLeft = UDim.new(0,7); listPad.PaddingRight = UDim.new(0,7)

    local themed = {
        gradients  = { wrapGrad, barGrad },
        strokes    = { panelStroke },
        bgFrames   = {},
        textAccent = { TtlH },
        sepFrames  = { hSep },
        rowGrads   = {},
        sliderGrads= {},
    }

    local rowOrder = 0
    local function makeRow(h, darkFactor)
        rowOrder += 1
        local r = Instance.new("Frame", Content)
        r.Size = UDim2.new(1,0,0,h)
        r.BackgroundColor3 = themeC1:Lerp(BLACK, darkFactor or 0.86)
        r.BorderSizePixel = 0; r.LayoutOrder = rowOrder
        Instance.new("UICorner", r).CornerRadius = UDim.new(0,9)
        local rg = Instance.new("UIGradient", r)
        rg.Color = ColorSequence.new(themeC1, themeC2); rg.Rotation = 135
        rg.Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0,0.78), NumberSequenceKeypoint.new(1,0.88)})
        table.insert(themed.bgFrames, {frame=r, factor=darkFactor or 0.86})
        table.insert(themed.rowGrads, rg)
        return r, rg
    end
    local function addStroke(inst, thin, tr)
        local s = Instance.new("UIStroke", inst)
        s.Color = themeC1; s.Thickness = thin or 1.2; s.Transparency = tr or 0.35
        table.insert(themed.strokes, s); return s
    end

    -- ROW 1 — Script ON/OFF
    local rowScript = makeRow(30, 0.87)
    addStroke(rowScript, 1.2, 0.30)
    local ScriptBtn = Instance.new("TextButton", rowScript)
    ScriptBtn.Size = UDim2.new(1,0,1,0); ScriptBtn.BackgroundTransparency = 1
    ScriptBtn.Text = "🟢  "..T("scriptOn"); ScriptBtn.TextColor3 = CT
    ScriptBtn.TextSize = 10; ScriptBtn.Font = Enum.Font.GothamBold
    ScriptBtn.BorderSizePixel = 0; ScriptBtn.ZIndex = 2

    -- ROW 2 — Log
    local rowLog = makeRow(84, 0.92)
    addStroke(rowLog, 1.0, 0.50)
    local logHdr = Instance.new("Frame", rowLog)
    logHdr.Size = UDim2.new(1,0,0,16)
    logHdr.BackgroundColor3 = themeC1:Lerp(BLACK, 0.94)
    logHdr.BorderSizePixel = 0; logHdr.ZIndex = 2
    Instance.new("UICorner", logHdr).CornerRadius = UDim.new(0,9)
    table.insert(themed.bgFrames, {frame=logHdr, factor=0.94})
    local logHdrLbl = Instance.new("TextLabel", logHdr)
    logHdrLbl.Size = UDim2.new(1,-8,1,0); logHdrLbl.Position = UDim2.new(0,8,0,0)
    logHdrLbl.BackgroundTransparency = 1; logHdrLbl.Text = "[ "..T("logTitle").." ]"
    logHdrLbl.TextSize = 8; logHdrLbl.Font = Enum.Font.GothamSemibold
    logHdrLbl.TextColor3 = themeC1:Lerp(WHITE, 0.40)
    logHdrLbl.TextXAlignment = Enum.TextXAlignment.Left; logHdrLbl.ZIndex = 3
    table.insert(themed.textAccent, logHdrLbl)

    local Scroll = Instance.new("ScrollingFrame", rowLog)
    Scroll.Size = UDim2.new(1,-4,1,-18); Scroll.Position = UDim2.new(0,2,0,17)
    Scroll.BackgroundTransparency = 1; Scroll.BorderSizePixel = 0
    Scroll.ScrollBarThickness = 2; Scroll.ScrollBarImageColor3 = themeC1:Lerp(BLACK,0.50)
    Scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    Scroll.CanvasSize = UDim2.new(0,0,0,0); Scroll.ZIndex = 2
    local sLayout = Instance.new("UIListLayout", Scroll); sLayout.Padding = UDim.new(0,2)
    local sPad = Instance.new("UIPadding", Scroll)
    sPad.PaddingLeft = UDim.new(0,5); sPad.PaddingTop = UDim.new(0,2)

    local logIdx = 0
    addLog = function(msg)
        logIdx += 1
        local l = Instance.new("TextLabel", Scroll)
        l.Size = UDim2.new(1,-8,0,0); l.AutomaticSize = Enum.AutomaticSize.Y
        l.BackgroundTransparency = 1; l.Text = msg
        l.TextColor3 = themeC1:Lerp(WHITE, 0.55)
        l.TextSize = 9; l.Font = Enum.Font.Code
        l.TextXAlignment = Enum.TextXAlignment.Left; l.TextWrapped = true
        l.LayoutOrder = logIdx; l.ZIndex = 3; l.Parent = Scroll
        task.defer(function() Scroll.CanvasPosition = Vector2.new(0, Scroll.AbsoluteCanvasSize.Y) end)
    end

    -- ROW 3 — Slider
    local rowSlider = makeRow(40, 0.89)
    addStroke(rowSlider, 1.0, 0.45)
    local sliderLabel = Instance.new("TextLabel", rowSlider)
    sliderLabel.Size = UDim2.new(1,-10,0,14); sliderLabel.Position = UDim2.new(0,8,0,3)
    sliderLabel.BackgroundTransparency = 1; sliderLabel.Text = T("force")..":  100%"
    sliderLabel.TextSize = 9; sliderLabel.Font = Enum.Font.GothamSemibold
    sliderLabel.TextColor3 = themeC1:Lerp(WHITE, 0.55)
    sliderLabel.TextXAlignment = Enum.TextXAlignment.Left; sliderLabel.ZIndex = 2
    table.insert(themed.textAccent, sliderLabel)

    local sliderTrack = Instance.new("Frame", rowSlider)
    sliderTrack.Size = UDim2.new(1,-16,0,7); sliderTrack.Position = UDim2.new(0,8,0,25)
    sliderTrack.BackgroundColor3 = themeC1:Lerp(BLACK, 0.82)
    sliderTrack.BorderSizePixel = 0; sliderTrack.ZIndex = 2
    Instance.new("UICorner", sliderTrack).CornerRadius = UDim.new(0.5,0)
    table.insert(themed.bgFrames, {frame=sliderTrack, factor=0.82})

    local sliderFill = Instance.new("Frame", sliderTrack)
    sliderFill.Size = UDim2.new(1,0,1,0); sliderFill.BackgroundColor3 = themeC1
    sliderFill.BorderSizePixel = 0; sliderFill.ZIndex = 3
    Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(0.5,0)
    local sliderFillGrad = Instance.new("UIGradient", sliderFill)
    sliderFillGrad.Color = ColorSequence.new(themeC1, themeC2)
    table.insert(themed.sliderGrads, sliderFillGrad)
    table.insert(themed.bgFrames, {frame=sliderFill, factor=0.0})

    local sliderThumb = Instance.new("Frame", sliderTrack)
    sliderThumb.Size = UDim2.new(0,15,0,15); sliderThumb.AnchorPoint = Vector2.new(0.5,0.5)
    sliderThumb.Position = UDim2.new(1,0,0.5,0); sliderThumb.BackgroundColor3 = CT
    sliderThumb.BorderSizePixel = 0; sliderThumb.ZIndex = 5
    Instance.new("UICorner", sliderThumb).CornerRadius = UDim.new(0.5,0)
    local thumbStroke = Instance.new("UIStroke", sliderThumb)
    thumbStroke.Color = themeC1; thumbStroke.Thickness = 2.2
    table.insert(themed.strokes, thumbStroke)

    local thumbHit = Instance.new("TextButton", sliderThumb)
    thumbHit.Size = UDim2.new(1,14,1,14); thumbHit.Position = UDim2.new(0,-7,0,-7)
    thumbHit.BackgroundTransparency = 1; thumbHit.Text = ""; thumbHit.ZIndex = 7

    local trackHit = Instance.new("TextButton", sliderTrack)
    trackHit.Size = UDim2.new(1,0,0,26); trackHit.Position = UDim2.new(0,0,0.5,-13)
    trackHit.BackgroundTransparency = 1; trackHit.Text = ""; trackHit.ZIndex = 4

    local function updateSlider(rawX)
        local abs = sliderTrack.AbsolutePosition; local sz = sliderTrack.AbsoluteSize
        local ratio = math.clamp((rawX - abs.X) / sz.X, 0, 1)
        strikePower = 0.1 + ratio * 1.4
        sliderFill.Size = UDim2.new(ratio,0,1,0)
        sliderThumb.Position = UDim2.new(ratio,0,0.5,0)
        sliderLabel.Text = T("force")..":  "..tostring(math.round(ratio * 100)).."%"
    end

    -- Drag do slider com conexão temporária (corrige bug de toque)
    local sliderMoveConn = nil
    local function stopSliderDrag()
        sliderDragging = false
        if sliderMoveConn then sliderMoveConn:Disconnect(); sliderMoveConn = nil end
    end
    local function startSliderDrag(initX)
        if sliderDragging then return end
        sliderDragging = true; updateSlider(initX)
        sliderMoveConn = UIS.InputChanged:Connect(function(inp)
            if inp.UserInputType == Enum.UserInputType.MouseMovement
            or inp.UserInputType == Enum.UserInputType.Touch then
                updateSlider(inp.Position.X)
            end
        end)
    end
    thumbHit.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            startSliderDrag(inp.Position.X)
        end
    end)
    trackHit.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            startSliderDrag(inp.Position.X)
        end
    end)

    -- ROW 4 — TACAR
    local rowTacar = makeRow(44, 0.76)
    local tacarStroke = addStroke(rowTacar, 1.8, 0.05)
    local tacarBar = Instance.new("Frame", rowTacar)
    tacarBar.Size = UDim2.new(0,0,1,0); tacarBar.BackgroundColor3 = themeC1
    tacarBar.BorderSizePixel = 0; tacarBar.ZIndex = 1
    Instance.new("UICorner", tacarBar).CornerRadius = UDim.new(0,9)
    local tacarBarGrad = Instance.new("UIGradient", tacarBar)
    tacarBarGrad.Color = ColorSequence.new(themeC1, themeC2)
    tacarBarGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0,0.25), NumberSequenceKeypoint.new(1,0.10)})
    table.insert(themed.sliderGrads, tacarBarGrad)
    table.insert(themed.bgFrames, {frame=tacarBar, factor=0.0})

    local tacEmoji = Instance.new("TextLabel", rowTacar)
    tacEmoji.Size = UDim2.new(0,24,1,0); tacEmoji.Position = UDim2.new(0,5,0,0)
    tacEmoji.BackgroundTransparency = 1; tacEmoji.Text = "🏌"
    tacEmoji.TextSize = 17; tacEmoji.Font = Enum.Font.GothamBold; tacEmoji.ZIndex = 3

    local MainBtn = Instance.new("TextButton", rowTacar)
    MainBtn.Size = UDim2.new(1,0,1,0); MainBtn.BackgroundTransparency = 1
    MainBtn.Text = T("holdStrike"); MainBtn.TextColor3 = CT
    MainBtn.TextSize = 12; MainBtn.Font = Enum.Font.GothamBlack
    MainBtn.BorderSizePixel = 0; MainBtn.ZIndex = 3

    -- ROW 5 — AUTO
    local rowAuto = makeRow(28, 0.88)
    local autoStroke = addStroke(rowAuto, 1.2, 0.42)
    rowAuto.BackgroundColor3 = LOCKED_BG
    local AutoBtn = Instance.new("TextButton", rowAuto)
    AutoBtn.Size = UDim2.new(1,0,1,0); AutoBtn.BackgroundTransparency = 1
    AutoBtn.Text = ""; AutoBtn.TextColor3 = Color3.fromRGB(145,145,145)
    AutoBtn.TextSize = 10; AutoBtn.Font = Enum.Font.GothamBold
    AutoBtn.BorderSizePixel = 0; AutoBtn.ZIndex = 2
    table.insert(themed.textAccent, AutoBtn)
    local autoLock = Instance.new("ImageLabel", AutoBtn)
    autoLock.Size = UDim2.new(0,16,0,16)
    autoLock.Position = UDim2.new(0.5,-8,0.5,-8)
    autoLock.BackgroundTransparency = 1
    autoLock.Image = LOCK_ICON
    autoLock.ScaleType = Enum.ScaleType.Fit
    autoLock.ZIndex = 3

    -- ROW 6 — Status/versão
    local rowStatus, _ = makeRow(10, 0.94)
    rowStatus.BackgroundTransparency = 1
    local statusLbl = Instance.new("TextLabel", rowStatus)
    statusLbl.Size = UDim2.new(1,0,1,0); statusLbl.BackgroundTransparency = 1
    statusLbl.Text = T("version"); statusLbl.TextSize = 8; statusLbl.Font = Enum.Font.Gotham
    statusLbl.TextColor3 = themeC1:Lerp(BLACK,0.50); statusLbl.ZIndex = 2

    -- ════════════════════════════════════════════════
    --  FUNÇÃO CENTRAL DE TEMA
    -- ════════════════════════════════════════════════
    local RD1 = Color3.fromRGB(215,30,30)
    local RD2 = Color3.fromRGB(100,8,8)

    local function applyFullTheme(c1, c2, isRed)
        for _, g in ipairs(themed.gradients) do g.Color = ColorSequence.new(c1, c2) end
        for _, g in ipairs(themed.rowGrads) do
            g.Color = ColorSequence.new(c1, c2)
            g.Transparency = NumberSequence.new({
                NumberSequenceKeypoint.new(0, isRed and 0.60 or 0.78),
                NumberSequenceKeypoint.new(1, isRed and 0.70 or 0.88)})
        end
        for _, g in ipairs(themed.sliderGrads) do g.Color = ColorSequence.new(c1, c2) end
        for _, s in ipairs(themed.strokes) do s.Color = c1 end
        panelStroke.Color = c1
        for _, f in ipairs(themed.sepFrames) do f.BackgroundColor3 = c1 end
        hSep.BackgroundColor3 = c1
        for _, lbl in ipairs(themed.textAccent) do lbl.TextColor3 = c1:Lerp(WHITE, 0.45) end
        TtlH.TextColor3 = c1
        local TD = TweenInfo.new(0.35, Enum.EasingStyle.Quint)
        for _, entry in ipairs(themed.bgFrames) do
            local target = (entry.factor == 0.0) and c1 or c1:Lerp(isRed and BLACK or BLACK, entry.factor)
            TweenSvc:Create(entry.frame, TD, {BackgroundColor3=target}):Play()
        end
        TweenSvc:Create(Wrapper, TD, {BackgroundColor3=c1:Lerp(BLACK,0.94)}):Play()
        TweenSvc:Create(Bar,     TD, {BackgroundColor3=c1:Lerp(BLACK,0.88)}):Play()
        for _, btn in ipairs({EyeBtn, MinBtn}) do
            btn.BackgroundColor3 = c1:Lerp(BLACK, 0.78)
        end
        ColorBtn.BackgroundColor3 = LOCKED_BG
        rowAuto.BackgroundColor3 = LOCKED_BG
        autoStroke.Color = Color3.fromRGB(70,70,70)
        beamCore.Color = ColorSequence.new(c1)
        beamGlow.Color = ColorSequence.new(c2)
    end

    -- ════════════════════════════════════════════════
    --  RENDER LOOP
    -- ════════════════════════════════════════════════
    local function connectRenderLoop()
        if renderConn then renderConn:Disconnect() end
        renderConn = RunSvc.Heartbeat:Connect(function()
            if not scriptOn then
                beamCore.Enabled = false; beamGlow.Enabled = false
                alertFrame.Visible = false; return
            end
            local ballPos = nil
            local ok, bp = pcall(BallHndlr.GetMyBallPosition); if ok then ballPos = bp end
            local holePos = HoleTarget.GetEndPointPosition()
            if not ballPos or not holePos then
                beamCore.Enabled = false; beamGlow.Enabled = false
                alertFrame.Visible = false; lastReach = nil; return
            end
            ballAnchor.CFrame = CFrame.new(ballPos); holeAnchor.CFrame = CFrame.new(holePos)
            beamCore.Enabled = true; beamGlow.Enabled = true
            local reach = isReachable(ballPos, holePos)
            if reach ~= lastReach then
                lastReach = reach
                if reach then
                    beamCore.Color = ColorSequence.new(themeC1)
                    beamGlow.Color = ColorSequence.new(themeC2)
                    alertImg.Image = IMG_REACH
                else
                    beamCore.Color = ColorSequence.new(NEON_RED)
                    beamGlow.Color = ColorSequence.new(NEON_RED)
                    alertImg.Image = IMG_FAR
                end
                alertImg.ImageTransparency = 0; startPulse()
            end
            alertFrame.Visible = true
        end)
    end
    connectRenderLoop()

    -- ════════════════════════════════════════════════
    --  INPUT GLOBAL
    -- ════════════════════════════════════════════════
    UIS.InputChanged:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseMovement and panelDragging then
            local d = Vector2.new(inp.Position.X, inp.Position.Y) - dragStart
            Wrapper.Position = UDim2.new(panelStart.X.Scale, panelStart.X.Offset+d.X,
                                          panelStart.Y.Scale, panelStart.Y.Offset+d.Y)
        end
    end)
    UIS.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            stopSliderDrag(); panelDragging = false
        end
    end)

    -- ════════════════════════════════════════════════
    --  BOTÕES
    -- ════════════════════════════════════════════════
    Bar.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then
            panelDragging = true
            dragStart = Vector2.new(inp.Position.X, inp.Position.Y)
            panelStart = Wrapper.Position
        end
    end)
    EyeBtn.MouseButton1Click:Connect(function()
        eyeOn = not eyeOn
        if eyeOn then startSpectate(); EyeBtn.TextColor3 = themeC1; addLog("👁 "..T("eyeOn"))
        else stopSpectate(); EyeBtn.TextColor3 = CT; addLog("👁 "..T("eyeOff")) end
    end)
    MinBtn.MouseButton1Click:Connect(function()
        minimized = not minimized
        TweenSvc:Create(Wrapper, TweenInfo.new(0.20, Enum.EasingStyle.Quint),
            {Size=UDim2.new(0,PW,0, minimized and HH or FH)}):Play()
        MinBtn.Text = minimized and "+" or "—"
        Content.Visible = not minimized
    end)
    Xbtn.MouseButton1Click:Connect(function()
        running = false; holdActive = false; autoOn = false; scriptOn = false
        stopSpectate()
        if pulseTween   then pulseTween:Cancel() end
        if renderConn   then renderConn:Disconnect() end
        if spectateConn then spectateConn:Disconnect() end
        if vFolder      then vFolder:Destroy() end
        alertSG:Destroy(); mainSG:Destroy()
    end)
    ScriptBtn.MouseButton1Click:Connect(function()
        scriptOn = not scriptOn
        if scriptOn then
            connectRenderLoop()
            ScriptBtn.Text = "🟢  "..T("scriptOn"); ScriptBtn.TextColor3 = CT
            applyFullTheme(themeC1, themeC2, false); addLog("🟢 "..T("activated"))
        else
            running = false; holdActive = false; autoOn = false
            if renderConn then renderConn:Disconnect(); renderConn = nil end
            beamCore.Enabled = false; beamGlow.Enabled = false; alertFrame.Visible = false
            ScriptBtn.Text = "🔴  "..T("scriptOff"); ScriptBtn.TextColor3 = Color3.fromRGB(255,110,110)
            applyFullTheme(RD1, RD2, true); addLog("🔴 "..T("deactivated"))
        end
    end)

    local strikeTween = nil
    MainBtn.MouseButton1Down:Connect(function()
        if not scriptOn or holdActive then return end
        local okBall, ballPos = pcall(BallHndlr.GetMyBallPosition)
        local char = lp.Character
        local root = char and (char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart)
        if not okBall or not ballPos or not root or (root.Position - ballPos).Magnitude > 10 then
            addLog("⚠ "..T("tooFarStrike"))
            return
        end
        holdActive = true; tacarBar.Size = UDim2.new(0,0,1,0); MainBtn.Text = T("striking")
        strikeTween = TweenSvc:Create(tacarBar, TweenInfo.new(1, Enum.EasingStyle.Linear),
            {Size=UDim2.new(1,0,1,0)}); strikeTween:Play()
        strikeTween.Completed:Connect(function(state)
            if state == Enum.PlaybackState.Completed and holdActive then
                running = true; task.spawn(function() doAutoHole(); running = false end)
            end
            holdActive = false; tacarBar.Size = UDim2.new(0,0,1,0); MainBtn.Text = T("holdStrike")
        end)
    end)
    MainBtn.MouseButton1Up:Connect(function()
        if strikeTween then strikeTween:Cancel(); strikeTween = nil end
        holdActive = false; running = false
        tacarBar.Size = UDim2.new(0,0,1,0); MainBtn.Text = T("holdStrike")
    end)
    AutoBtn.MouseButton1Click:Connect(function()
        autoOn = false
        running = false
        AutoBtn.Text = ""
        addLog("🪙 "..T("premiumRequired"))
    end)

    -- ════════════════════════════════════════════════
    --  GUI DE CORES — cor única
    -- ════════════════════════════════════════════════
    local colorSG = nil
    local selC1 = themeC1

    ColorBtn.MouseButton1Click:Connect(function()
        addLog("🪙 "..T("premiumRequired"))
        if true then return end
        if colorGuiOpen then
            if colorSG then colorSG:Destroy(); colorSG = nil end
            colorGuiOpen = false; return
        end
        colorGuiOpen = true
        local oldCG = lp.PlayerGui:FindFirstChild("SylaxColorGui"); if oldCG then oldCG:Destroy() end
        colorSG = Instance.new("ScreenGui", lp.PlayerGui)
        colorSG.Name = "SylaxColorGui"; colorSG.ResetOnSpawn = false
        colorSG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

        local CW2, CH2 = 204, 214
        local CPanel = Instance.new("Frame", colorSG)
        CPanel.Size = UDim2.new(0,CW2,0,CH2)
        CPanel.Position = UDim2.new(0, 16+PW+6, 0.5, -CH2/2)
        CPanel.BackgroundColor3 = selC1:Lerp(BLACK, 0.92)
        CPanel.BorderSizePixel = 0
        Instance.new("UICorner", CPanel).CornerRadius = UDim.new(0,14)
        local cpGrad = Instance.new("UIGradient", CPanel)
        cpGrad.Color = ColorSequence.new(selC1, selC1:Lerp(BLACK,0.62)); cpGrad.Rotation = 135
        cpGrad.Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0,0.82), NumberSequenceKeypoint.new(1,0.90)})
        local cpStroke = Instance.new("UIStroke", CPanel)
        cpStroke.Color = selC1; cpStroke.Thickness = 1.6

        local cpTitle = Instance.new("TextLabel", CPanel)
        cpTitle.Size = UDim2.new(1,-34,0,24); cpTitle.Position = UDim2.new(0,7,0,6)
        cpTitle.BackgroundTransparency = 1; cpTitle.Text = "🎨  "..T("colors")
        cpTitle.TextSize = 11; cpTitle.Font = Enum.Font.GothamBlack; cpTitle.TextColor3 = CT
        cpTitle.TextXAlignment = Enum.TextXAlignment.Left

        local cpClose = Instance.new("TextButton", CPanel)
        cpClose.Size = UDim2.new(0,20,0,20); cpClose.Position = UDim2.new(1,-26,0,7)
        cpClose.BackgroundColor3 = CR; cpClose.Text = "✕"; cpClose.TextSize = 9
        cpClose.Font = Enum.Font.GothamBold; cpClose.TextColor3 = CT; cpClose.BorderSizePixel = 0
        Instance.new("UICorner", cpClose).CornerRadius = UDim.new(0,6)
        cpClose.MouseButton1Click:Connect(function()
            colorGuiOpen = false; colorSG:Destroy(); colorSG = nil
        end)

        local swatchFrame = Instance.new("Frame", CPanel)
        swatchFrame.Size = UDim2.new(1,-14,0,13); swatchFrame.Position = UDim2.new(0,7,0,34)
        swatchFrame.BackgroundColor3 = selC1; swatchFrame.BorderSizePixel = 0
        Instance.new("UICorner", swatchFrame).CornerRadius = UDim.new(0,5)
        local swatchGrad = Instance.new("UIGradient", swatchFrame)
        swatchGrad.Color = ColorSequence.new(selC1, selC1:Lerp(BLACK,0.62)); swatchGrad.Rotation = 90

        local grid = Instance.new("Frame", CPanel)
        grid.Size = UDim2.new(1,-14,0,74); grid.Position = UDim2.new(0,7,0,54)
        grid.BackgroundTransparency = 1
        local gl = Instance.new("UIGridLayout", grid)
        gl.CellSize = UDim2.new(0,33,0,33); gl.CellPadding = UDim2.new(0,3,0,3)
        gl.FillDirection = Enum.FillDirection.Horizontal
        gl.HorizontalAlignment = Enum.HorizontalAlignment.Left

        local prevGrad = nil
        local function refreshPicker()
            local c2a = selC1:Lerp(BLACK, 0.62)
            cpStroke.Color = selC1; cpGrad.Color = ColorSequence.new(selC1, c2a)
            CPanel.BackgroundColor3 = selC1:Lerp(BLACK, 0.92)
            swatchFrame.BackgroundColor3 = selC1; swatchGrad.Color = ColorSequence.new(selC1, c2a)
            if prevGrad then prevGrad.Color = ColorSequence.new(selC1, c2a) end
        end

        for _, col in ipairs(PALETTE) do
            local cb = Instance.new("TextButton", grid)
            cb.BackgroundColor3 = col; cb.Size = UDim2.new(0,33,0,33)
            cb.Text = ""; cb.BorderSizePixel = 0
            Instance.new("UICorner", cb).CornerRadius = UDim.new(0,8)
            local cbs = Instance.new("UIStroke", cb)
            cbs.Thickness = col == selC1 and 2.4 or 1.4
            cbs.Color = col == selC1 and CT or Color3.fromRGB(40,40,40)
            cb.MouseEnter:Connect(function() cbs.Color = CT; cbs.Thickness = 2.4 end)
            cb.MouseLeave:Connect(function()
                cbs.Color = (col==selC1) and CT or Color3.fromRGB(40,40,40)
                cbs.Thickness = (col==selC1) and 2.4 or 1.4
            end)
            cb.MouseButton1Click:Connect(function() selC1=col; refreshPicker() end)
        end

        local prevLbl = Instance.new("TextLabel", CPanel)
        prevLbl.Size = UDim2.new(1,-14,0,10); prevLbl.Position = UDim2.new(0,7,0,134)
        prevLbl.BackgroundTransparency = 1; prevLbl.Text = T("preview")
        prevLbl.TextSize = 8; prevLbl.Font = Enum.Font.GothamSemibold
        prevLbl.TextColor3 = selC1:Lerp(WHITE,0.50); prevLbl.TextXAlignment = Enum.TextXAlignment.Left

        local prevFrame = Instance.new("Frame", CPanel)
        prevFrame.Size = UDim2.new(1,-14,0,14); prevFrame.Position = UDim2.new(0,7,0,146)
        prevFrame.BackgroundColor3 = BLACK; prevFrame.BorderSizePixel = 0
        Instance.new("UICorner", prevFrame).CornerRadius = UDim.new(0,5)
        prevGrad = Instance.new("UIGradient", prevFrame)
        prevGrad.Color = ColorSequence.new(selC1, selC1:Lerp(BLACK,0.62)); prevGrad.Rotation = 90

        local applyBtn = Instance.new("TextButton", CPanel)
        applyBtn.Size = UDim2.new(1,-14,0,26); applyBtn.Position = UDim2.new(0,7,0,173)
        applyBtn.BackgroundColor3 = selC1; applyBtn.Text = T("apply")
        applyBtn.TextSize = 11; applyBtn.Font = Enum.Font.GothamBold
        applyBtn.TextColor3 = BLACK; applyBtn.BorderSizePixel = 0
        Instance.new("UICorner", applyBtn).CornerRadius = UDim.new(0,7)
        local applyGrad = Instance.new("UIGradient", applyBtn)
        applyGrad.Color = ColorSequence.new(selC1, selC1:Lerp(BLACK,0.62)); applyGrad.Rotation = 90

        applyBtn.MouseButton1Click:Connect(function()
            themeC1 = selC1; themeC2 = selC1:Lerp(BLACK, 0.62)
            applyFullTheme(themeC1, themeC2, false)
            sliderLabel.TextColor3       = themeC1:Lerp(WHITE, 0.55)
            logHdrLbl.TextColor3         = themeC1:Lerp(WHITE, 0.40)
            statusLbl.TextColor3         = themeC1:Lerp(BLACK, 0.50)
            Scroll.ScrollBarImageColor3  = themeC1:Lerp(BLACK, 0.50)
            addLog("🎨 "..T("colors").." OK")
            colorGuiOpen = false; colorSG:Destroy(); colorSG = nil
        end)
        refreshPicker()
    end)

    addLog("* "..T("ready"))
end

-- ════════════════════════════════════════════════
--  INICIAR — começa pelo Key System
-- ════════════════════════════════════════════════
print("[SylaxHub] v2.3 — carregando key system")
buildKeyGui()
