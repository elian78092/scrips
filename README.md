-- Asegurarse de que el juego haya cargado
if not game:IsLoaded() then
    game.Loaded:Wait()
end

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local MarketPlaceService = game:GetService("MarketplaceService")
local CoreGui = game:GetService("CoreGui")
local player = Players.LocalPlayer

-- =========================================================================
-- CONFIGURACIÓN DE SEGURIDAD ESTRICTA POR ID Y NOMBRE DE JUEGO
-- =========================================================================
local DUELIST_PLACE_ID = 111166698622146 -- ID oficial de Duelista: PvP
local DUELIST_FALLBACK_ID = 18780604245 

local successName, gameInfo = pcall(function()
    return MarketPlaceService:GetProductInfo(game.PlaceId)
end)

local gameName = string.lower(successName and gameInfo and gameInfo.Name or "")
local esDuelistPorNombre = string.find(gameName, "duelist") ~= nil or string.find(gameName, "duelista") ~= nil
local esDuelistPorId = (game.PlaceId == DUELIST_PLACE_ID)

local function mostrarMensajeErrorYTeletransportar()
    pcall(function()
        if CoreGui:FindFirstChild("ErrorMenu") then
            CoreGui.ErrorMenu:Destroy()
        end

        local ErrorGui = Instance.new("ScreenGui")
        ErrorGui.Name = "ErrorMenu"
        ErrorGui.Parent = CoreGui
        ErrorGui.ResetOnSpawn = false

        local AlertFrame = Instance.new("Frame")
        AlertFrame.Size = UDim2.new(0, 420, 0, 150)
        AlertFrame.Position = UDim2.new(0.5, -210, 0.4, -75)
        AlertFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
        AlertFrame.BorderSizePixel = 0
        AlertFrame.Parent = ErrorGui

        local Corner = Instance.new("UICorner")
        Corner.CornerRadius = UDim.new(0, 12)
        Corner.Parent = AlertFrame

        local Stroke = Instance.new("UIStroke")
        Stroke.Color = Color3.fromRGB(255, 50, 50)
        Stroke.Thickness = 2
        Stroke.Parent = AlertFrame

        local Text = Instance.new("TextLabel")
        Text.Size = UDim2.new(1, -20, 1, -20)
        Text.Position = UDim2.new(0, 10, 0, 10)
        Text.BackgroundTransparency = 1
        Text.Font = Enum.Font.GothamBold
        Text.TextSize = 14
        Text.TextColor3 = Color3.fromRGB(255, 255, 255)
        Text.TextWrapped = true
        Text.Text = "⚠ ¡ESTE SCRIPT ES SOLAMENTE PARA DUELISTA: PvP!\n\nRedirigiéndote al juego correcto..."
        Text.Parent = AlertFrame
    end)

    task.wait(3)
    
    pcall(function()
        local teleportOptions = Instance.new("TeleportOptions")
        TeleportService:TeleportAsync(DUELIST_FALLBACK_ID, {player}, teleportOptions)
    end)
end

if not esDuelistPorId and not esDuelistPorNombre then
    mostrarMensajeErrorYTeletransportar()
    return 
end

-- =========================================================================
-- VARIABLES GLOBALES Y DE ESTADO
-- =========================================================================
local headHitboxEnabled = false
local headHitboxSize = 6 

local espBoxEnabled = false
local espNameEnabled = false
local espDistance = 80

local espBoxes = {}
local espNames = {}
local originalHeadSizes = {}
local characterConnections = {}
local connections = {}

-- =========================================================================
-- SISTEMA DE TEAM CHECK
-- =========================================================================
local function isTeammate(otherPlayer)
    if player.Team and otherPlayer.Team then
        return player.Team == otherPlayer.Team
    end
    return false
end

local function restoreOriginalHeads()
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if originalHeadSizes[otherPlayer] and otherPlayer.Character then
            local head = otherPlayer.Character:FindFirstChild("Head")
            if head then
                head.Size = originalHeadSizes[otherPlayer].size
                head.Transparency = originalHeadSizes[otherPlayer].transparency
                head.CanCollide = originalHeadSizes[otherPlayer].cancollide
                head.Massless = originalHeadSizes[otherPlayer].massless
            end
        end
    end
    originalHeadSizes = {}
end

local function fullCleanup()
    headHitboxEnabled = false
    restoreOriginalHeads()

    for _, boxGui in pairs(espBoxes) do if boxGui then boxGui:Destroy() end end
    espBoxes = {}

    for _, nameGui in pairs(espNames) do if nameGui then nameGui:Destroy() end end
    espNames = {}

    for _, conn in pairs(characterConnections) do if conn then conn:Disconnect() end end
    characterConnections = {}

    for _, conn in pairs(connections) do if conn then conn:Disconnect() end end
    connections = {}
end

-- =========================================================================
-- DISEÑO DE LA INTERFAZ ESTILIZADA (ELIAN HUB)
-- =========================================================================
if CoreGui:FindFirstChild("ProfessionalMenuV9") then
    CoreGui.ProfessionalMenuV9:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ProfessionalMenuV9"
ScreenGui.Parent = CoreGui
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
MainFrame.Position = UDim2.new(0.5, -240, 0.5, -175)
MainFrame.Size = UDim2.new(0, 480, 0, 350)
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.BorderSizePixel = 0

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 14)
MainCorner.Parent = MainFrame

local MainGradient = Instance.new("UIGradient")
MainGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(14, 14, 18)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(22, 18, 26))
})
MainGradient.Rotation = 45
MainGradient.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Thickness = 2.5
MainStroke.Parent = MainFrame

local TopBar = Instance.new("Frame")
TopBar.Name = "TopBar"
TopBar.Parent = MainFrame
TopBar.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
TopBar.Size = UDim2.new(1, 0, 0, 48)
TopBar.BorderSizePixel = 0

local TopCorner = Instance.new("UICorner")
TopCorner.CornerRadius = UDim.new(0, 14)
TopCorner.Parent = TopBar

local TopFix = Instance.new("Frame")
TopFix.Parent = TopBar
TopFix.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
TopFix.Position = UDim2.new(0, 0, 0.7, 0)
TopFix.Size = UDim2.new(1, 0, 0.3, 0)
TopFix.BorderSizePixel = 0

local Title = Instance.new("TextLabel")
Title.Parent = TopBar
Title.BackgroundTransparency = 1
Title.Position = UDim2.new(0.04, 0, 0, 0)
Title.Size = UDim2.new(0.6, 0, 1, 0)
Title.Font = Enum.Font.GothamBold
Title.Text = "⚡ ELIAN HUB"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 16
Title.TextXAlignment = Enum.TextXAlignment.Left

local ControlsContainer = Instance.new("Frame")
ControlsContainer.Parent = TopBar
ControlsContainer.BackgroundTransparency = 1
ControlsContainer.Position = UDim2.new(0.80, 0, 0.22, 0)
ControlsContainer.Size = UDim2.new(0, 85, 0, 30)

local MinimizeBtn = Instance.new("TextButton")
MinimizeBtn.Parent = ControlsContainer
MinimizeBtn.BackgroundColor3 = Color3.fromRGB(28, 28, 36)
MinimizeBtn.Size = UDim2.new(0, 32, 0, 26)
MinimizeBtn.Font = Enum.Font.GothamBold
MinimizeBtn.Text = "—"
MinimizeBtn.TextColor3 = Color3.fromRGB(220, 220, 230)
MinimizeBtn.TextSize = 12
MinimizeBtn.BorderSizePixel = 0

local MinCorner = Instance.new("UICorner")
MinCorner.CornerRadius = UDim.new(0, 6)
MinCorner.Parent = MinimizeBtn

local CloseBtn = Instance.new("TextButton")
CloseBtn.Parent = ControlsContainer
CloseBtn.BackgroundColor3 = Color3.fromRGB(255, 45, 45)
CloseBtn.Position = UDim2.new(0, 40, 0, 0)
CloseBtn.Size = UDim2.new(0, 32, 0, 26)
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.Text = "✕"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.TextSize = 12
CloseBtn.BorderSizePixel = 0

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 6)
CloseCorner.Parent = CloseBtn

local ContainerBody = Instance.new("Frame")
ContainerBody.Parent = MainFrame
ContainerBody.BackgroundTransparency = 1
ContainerBody.Position = UDim2.new(0, 0, 0, 48)
ContainerBody.Size = UDim2.new(1, 0, 1, -48)

local TabContainer = Instance.new("Frame")
TabContainer.Parent = ContainerBody
TabContainer.BackgroundColor3 = Color3.fromRGB(16, 16, 22)
TabContainer.Position = UDim2.new(0.03, 0, 0.04, 0)
TabContainer.Size = UDim2.new(0, 125, 0, 275)
TabContainer.BorderSizePixel = 0

local TabCorner = Instance.new("UICorner")
TabCorner.CornerRadius = UDim.new(0, 10)
TabCorner.Parent = TabContainer

local CombatTabBtn = Instance.new("TextButton")
CombatTabBtn.Parent = TabContainer
CombatTabBtn.BackgroundColor3 = Color3.fromRGB(255, 30, 70)
CombatTabBtn.Position = UDim2.new(0.08, 0, 0.06, 0)
CombatTabBtn.Size = UDim2.new(0.84, 0, 0, 38)
CombatTabBtn.Font = Enum.Font.GothamBold
CombatTabBtn.Text = "⚔  Combat"
CombatTabBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CombatTabBtn.TextSize = 12
CombatTabBtn.BorderSizePixel = 0

local CombatCorner = Instance.new("UICorner")
CombatCorner.CornerRadius = UDim.new(0, 8)
CombatCorner.Parent = CombatTabBtn

local VisualTabBtn = Instance.new("TextButton")
VisualTabBtn.Parent = TabContainer
VisualTabBtn.BackgroundColor3 = Color3.fromRGB(24, 24, 32)
VisualTabBtn.Position = UDim2.new(0.08, 0, 0.23, 0)
VisualTabBtn.Size = UDim2.new(0.84, 0, 0, 38)
VisualTabBtn.Font = Enum.Font.GothamBold
VisualTabBtn.Text = "👁  Visual"
VisualTabBtn.TextColor3 = Color3.fromRGB(140, 140, 160)
VisualTabBtn.TextSize = 12
VisualTabBtn.BorderSizePixel = 0

local VisualCorner = Instance.new("UICorner")
VisualCorner.CornerRadius = UDim.new(0, 8)
VisualCorner.Parent = VisualTabBtn

local PagesContainer = Instance.new("Frame")
PagesContainer.Parent = ContainerBody
PagesContainer.Position = UDim2.new(0.31, 0, 0.04, 0)
PagesContainer.Size = UDim2.new(0, 315, 0, 275)
PagesContainer.BackgroundTransparency = 1

local CombatPage = Instance.new("ScrollingFrame")
CombatPage.Parent = PagesContainer
CombatPage.BackgroundTransparency = 1
CombatPage.Size = UDim2.new(1, 0, 1, 0)
CombatPage.CanvasSize = UDim2.new(0, 0, 1.2, 0)
CombatPage.ScrollBarThickness = 0

local VisualPage = Instance.new("ScrollingFrame")
VisualPage.Parent = PagesContainer
VisualPage.BackgroundTransparency = 1
VisualPage.Size = UDim2.new(1, 0, 1, 0)
VisualPage.CanvasSize = UDim2.new(0, 0, 1.2, 0)
VisualPage.ScrollBarThickness = 0
VisualPage.Visible = false

local function switchTab(selectedPage, activeBtn, inactiveBtn)
    CombatPage.Visible = false
    VisualPage.Visible = false
    selectedPage.Visible = true
    TweenService:Create(activeBtn, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundColor3 = Color3.fromRGB(255, 30, 70), TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
    TweenService:Create(inactiveBtn, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundColor3 = Color3.fromRGB(24, 24, 32), TextColor3 = Color3.fromRGB(140, 140, 160)}):Play()
end

CombatTabBtn.MouseButton1Click:Connect(function() switchTab(CombatPage, CombatTabBtn, VisualTabBtn) end)
VisualTabBtn.MouseButton1Click:Connect(function() switchTab(VisualPage, VisualTabBtn, CombatTabBtn) end)

local function addHoverEffect(btn, baseColor, hoverColor)
    btn.MouseEnter:Connect(function()
        TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = hoverColor}):Play()
    end)
    btn.MouseLeave:Connect(function()
        TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = baseColor}):Play()
    end)
end
addHoverEffect(MinimizeBtn, Color3.fromRGB(28, 28, 36), Color3.fromRGB(40, 40, 52))
addHoverEffect(CloseBtn, Color3.fromRGB(255, 45, 45), Color3.fromRGB(200, 30, 30))

local minimized = false
MinimizeBtn.MouseButton1Click:Connect(function()
    minimized = not minimized
    ContainerBody.Visible = not minimized
    if minimized then
        MainFrame:TweenSize(UDim2.new(0, 480, 0, 48), "Out", "Quart", 0.25, true)
        MinimizeBtn.Text = "+"
    else
        MainFrame:TweenSize(UDim2.new(0, 480, 0, 350), "Out", "Quart", 0.25, true)
        MinimizeBtn.Text = "—"
    end
end)

CloseBtn.MouseButton1Click:Connect(function()
    fullCleanup()
    ScreenGui:Destroy()
end)

-- =========================================================================
-- SISTEMA DE LUCES RGB MÁS RÁPIDO ("AL PASO") PARA TÍTULO Y BORDES
-- =========================================================================
task.spawn(function()
    while ScreenGui and ScreenGui.Parent do
        local velocidad = 3.5 
        local hue = (tick() * velocidad) % 1
        local colorRgb = Color3.fromHSV(hue, 0.9, 1)
        
        Title.TextColor3 = colorRgb
        MainStroke.Color = colorRgb
        
        RunService.RenderStepped:Wait()
    end
end)

-- COMPONENTES CREADORES MEJORADOS
local function createToggle(parent, name, posY, callback)
    local ToggleBtn = Instance.new("TextButton")
    ToggleBtn.Parent = parent
    ToggleBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 26)
    ToggleBtn.Position = UDim2.new(0.02, 0, 0, posY)
    ToggleBtn.Size = UDim2.new(0.96, 0, 0, 42)
    ToggleBtn.Font = Enum.Font.GothamMedium
    ToggleBtn.Text = "     " .. name
    ToggleBtn.TextColor3 = Color3.fromRGB(235, 235, 245)
    ToggleBtn.TextSize = 12
    ToggleBtn.TextXAlignment = Enum.TextXAlignment.Left
    ToggleBtn.BorderSizePixel = 0

    local TCorner = Instance.new("UICorner")
    TCorner.CornerRadius = UDim.new(0, 8)
    TCorner.Parent = ToggleBtn
    
    local TStroke = Instance.new("UIStroke")
    TStroke.Color = Color3.fromRGB(35, 35, 45)
    TStroke.Thickness = 1
    TStroke.Parent = ToggleBtn

    local StatusIndicator = Instance.new("Frame")
    StatusIndicator.Parent = ToggleBtn
    StatusIndicator.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    StatusIndicator.Position = UDim2.new(0.85, 0, 0.30, 0)
    StatusIndicator.Size = UDim2.new(0, 30, 0, 16)

    local SCorner = Instance.new("UICorner")
    SCorner.CornerRadius = UDim.new(0, 8)
    SCorner.Parent = StatusIndicator

    local Dot = Instance.new("Frame")
    Dot.Parent = StatusIndicator
    Dot.BackgroundColor3 = Color3.fromRGB(180, 180, 190)
    Dot.Position = UDim2.new(0.08, 0, 0.15, 0)
    Dot.Size = UDim2.new(0, 11, 0, 11)

    local DCorner = Instance.new("UICorner")
    DCorner.CornerRadius = UDim.new(0, 6)
    DCorner.Parent = Dot

    ToggleBtn.MouseEnter:Connect(function()
        TweenService:Create(ToggleBtn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(26, 26, 34)}):Play()
        TweenService:Create(TStroke, TweenInfo.new(0.2), {Color = Color3.fromRGB(50, 50, 65)}):Play()
    end)
    ToggleBtn.MouseLeave:Connect(function()
        TweenService:Create(ToggleBtn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(20, 20, 26)}):Play()
        TweenService:Create(TStroke, TweenInfo.new(0.2), {Color = Color3.fromRGB(35, 35, 45)}):Play()
    end)

    local enabled = false
    ToggleBtn.MouseButton1Click:Connect(function()
        enabled = not enabled
        if enabled then
            TweenService:Create(StatusIndicator, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(255, 30, 70)}):Play()
            TweenService:Create(Dot, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(255, 255, 255)}):Play()
            Dot:TweenPosition(UDim2.new(0.54, 0, 0.15, 0), "Out", "Quad", 0.18, true)
        else
            TweenService:Create(StatusIndicator, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(40, 40, 50)}):Play()
            TweenService:Create(Dot, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(180, 180, 190)}):Play()
            Dot:TweenPosition(UDim2.new(0.08, 0, 0.15, 0), "Out", "Quad", 0.18, true)
        end
        callback(enabled)
    end)
end

local function createSlider(parent, name, min, max, default, posY, callback)
    local SliderFrame = Instance.new("Frame")
    SliderFrame.Parent = parent
    SliderFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 26)
    SliderFrame.Position = UDim2.new(0.02, 0, 0, posY)
    SliderFrame.Size = UDim2.new(0.96, 0, 0, 54)
    SliderFrame.BorderSizePixel = 0

    local SFrameCorner = Instance.new("UICorner")
    SFrameCorner.CornerRadius = UDim.new(0, 8)
    SFrameCorner.Parent = SliderFrame
    
    local SStroke = Instance.new("UIStroke")
    SStroke.Color = Color3.fromRGB(35, 35, 45)
    SStroke.Thickness = 1
    SStroke.Parent = SliderFrame

    local Label = Instance.new("TextLabel")
    Label.Parent = SliderFrame
    Label.BackgroundTransparency = 1
    Label.Position = UDim2.new(0.05, 0, 0.12, 0)
    Label.Text = name
    Label.TextColor3 = Color3.fromRGB(200, 200, 210)
    Label.TextSize = 12
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.Font = Enum.Font.GothamMedium

    local ValueLabel = Instance.new("TextLabel")
    ValueLabel.Parent = SliderFrame
    ValueLabel.BackgroundTransparency = 1
    ValueLabel.Position = UDim2.new(0.05, 0, 0.12, 0)
    ValueLabel.Size = UDim2.new(0.9, 0, 0, 20)
    ValueLabel.Text = tostring(default)
    ValueLabel.TextColor3 = Color3.fromRGB(255, 30, 70)
    ValueLabel.TextSize = 12
    ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
    ValueLabel.Font = Enum.Font.GothamBold

    local Bar = Instance.new("Frame")
    Bar.Parent = SliderFrame
    Bar.BackgroundColor3 = Color3.fromRGB(40, 40, 52)
    Bar.Position = UDim2.new(0.05, 0, 0.70, 0)
    Bar.Size = UDim2.new(0.9, 0, 0, 6)

    local BarCorner = Instance.new("UICorner")
    BarCorner.CornerRadius = UDim.new(0, 3)
    BarCorner.Parent = Bar

    local Fill = Instance.new("Frame")
    Fill.Parent = Bar
    Fill.BackgroundColor3 = Color3.fromRGB(255, 30, 70)
    Fill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)

    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(0, 3)
    FillCorner.Parent = Fill

    SliderFrame.MouseEnter:Connect(function()
        TweenService:Create(SStroke, TweenInfo.new(0.2), {Color = Color3.fromRGB(50, 50, 65)}):Play()
    end)
    SliderFrame.MouseLeave:Connect(function()
        TweenService:Create(SStroke, TweenInfo.new(0.2), {Color = Color3.fromRGB(35, 35, 45)}):Play()
    end)

    local dragging = false
    local function updateValue(input)
        local pos = math.clamp((input.Position.X - Bar.AbsolutePosition.X) / Bar.AbsoluteSize.X, 0, 1)
        local val = math.floor(min + ((max - min) * pos))
        Fill.Size = UDim2.new(pos, 0, 1, 0)
        ValueLabel.Text = tostring(val)
        callback(val)
    end

    table.insert(connections, Bar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = true updateValue(input) end
    end))
    table.insert(connections, UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end))
    table.insert(connections, UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then updateValue(input) end
    end))
end

createToggle(CombatPage, "Activar Hitbox", 10, function(state)
    headHitboxEnabled = state
    if not state then
        restoreOriginalHeads()
    end
end)

createSlider(CombatPage, "Head Hitbox Size", 2, 30, 6, 60, function(val) headHitboxSize = val end)
createToggle(VisualPage, "ESP Neon Box", 10, function(state) espBoxEnabled = state end)
createToggle(VisualPage, "ESP Name", 60, function(state) espNameEnabled = state end)
createSlider(VisualPage, "ESP Max Distance", 10, 150, 80, 110, function(val) espDistance = val end)

-- =========================================================================
-- SISTEMA DE EXPANSIÓN FÍSICA DE CABEZA (SIN ANCLAR AL PISO)
-- =========================================================================
local function modificarCabeza(otherPlayer)
    if otherPlayer == player then return end
    local char = otherPlayer.Character
    if not char then return end
    
    local head = char:FindFirstChild("Head")
    local hum = char:FindFirstChild("Humanoid")
    local rootPart = char:FindFirstChild("HumanoidRootPart")
    local myRoot = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    
    if head and hum and rootPart then
        if not originalHeadSizes[otherPlayer] then
            originalHeadSizes[otherPlayer] = {
                size = head.Size,
                transparency = head.Transparency,
                cancollide = head.CanCollide,
                massless = head.Massless
            }
        end
        
        local isAlive = hum.Health > 0
        local isEnemy = isAlive and not isTeammate(otherPlayer)
        local distance = myRoot and (rootPart.Position - myRoot.Position).Magnitude or 9999
        local inRange = distance <= espDistance

        if headHitboxEnabled and isEnemy and inRange then
            head.Size = Vector3.new(headHitboxSize, headHitboxSize, headHitboxSize)
            head.Transparency = 0.6
            head.CanCollide = false
            head.Massless = true 
        else
            if originalHeadSizes[otherPlayer] then
                head.Size = originalHeadSizes[otherPlayer].size
                head.Transparency = originalHeadSizes[otherPlayer].transparency
                head.CanCollide = originalHeadSizes[otherPlayer].cancollide
                head.Massless = originalHeadSizes[otherPlayer].massless
            end
        end
    end
end

-- =========================================================================
-- BUCLE DE RENDERIZADO DE ESP Y HITBOX DE CABEZA
-- =========================================================================
local function aplicarEfectosAEnemigo(otherPlayer)
    local char = otherPlayer.Character
    if not char then return end 
    
    local hum = char:FindFirstChild("Humanoid")
    local rootPart = char:FindFirstChild("HumanoidRootPart")
    local myRoot = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    
    local isAlive = (hum and hum.Health > 0 and rootPart)
    local isEnemy = isAlive and not isTeammate(otherPlayer)
    
    local distance = (myRoot and rootPart) and (rootPart.Position - myRoot.Position).Magnitude or 9999
    local inRange = distance <= espDistance

    modificarCabeza(otherPlayer)

    if espBoxEnabled and isEnemy and inRange and rootPart then
        local boxGui = espBoxes[otherPlayer]
        if not boxGui or boxGui.Parent ~= rootPart then
            if boxGui then boxGui:Destroy() end
            boxGui = Instance.new("BillboardGui")
            boxGui.Name = "ESP_Box"
            boxGui.AlwaysOnTop = true
            boxGui.Size = UDim2.new(4, 0, 5.5, 0)
            boxGui.Parent = rootPart
            
            local frame = Instance.new("Frame", boxGui)
            frame.BackgroundTransparency = 1
            frame.Size = UDim2.new(1, 0, 1, 0)
            
            local stroke = Instance.new("UIStroke", frame)
            stroke.Color = Color3.fromRGB(255, 30, 70)
            stroke.Thickness = 2
            
            espBoxes[otherPlayer] = boxGui
        end
        boxGui.Enabled = true
    else
        if espBoxes[otherPlayer] then espBoxes[otherPlayer].Enabled = false end
    end

    if espNameEnabled and isEnemy and inRange and rootPart then
        local nameGui = espNames[otherPlayer]
        if not nameGui or nameGui.Parent ~= rootPart then
            if nameGui then nameGui:Destroy() end
            nameGui = Instance.new("BillboardGui")
            nameGui.Name = "ESP_Name"
            nameGui.AlwaysOnTop = true
            nameGui.Size = UDim2.new(0, 200, 0, 40)
            nameGui.StudsOffset = Vector3.new(0, 3.8, 0)
            nameGui.Parent = rootPart
            
            local textLabel = Instance.new("TextLabel", nameGui)
            textLabel.BackgroundTransparency = 1
            textLabel.Size = UDim2.new(1, 0, 1, 0)
            textLabel.Font = Enum.Font.GothamBold
            textLabel.Text = otherPlayer.Name .. " [" .. math.floor(distance) .. "m]"
            textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
            textLabel.TextSize = 13
            
            local tStroke = Instance.new("UIStroke", textLabel)
            tStroke.Color = Color3.fromRGB(0, 0, 0)
            tStroke.Thickness = 1
            
            espNames[otherPlayer] = nameGui
        else
            local textLabel = nameGui:FindFirstChildOfClass("TextLabel")
            if textLabel then
                textLabel.Text = otherPlayer.Name .. " [" .. math.floor(distance) .. "m]"
            end
        end
        nameGui.Enabled = true
    else
        if espNames[otherPlayer] then espNames[otherPlayer].Enabled = false end
    end
end

local function trackearJugador(oPlayer)
    if oPlayer == player then return end
    
    characterConnections[oPlayer] = oPlayer.CharacterAdded:Connect(function()
        task.wait(0.2)
        if oPlayer.Character then
            aplicarEfectosAEnemigo(oPlayer)
        end
    end)
end

for _, p in ipairs(Players:GetPlayers()) do trackearJugador(p) end
table.insert(connections, Players.PlayerAdded:Connect(trackearJugador))

table.insert(connections, Players.PlayerRemoving:Connect(function(p)
    if espBoxes[p] then espBoxes[p]:Destroy() espBoxes[p] = nil end
    if espNames[p] then espNames[p]:Destroy() espNames[p] = nil end
    if originalHeadSizes[p] then originalHeadSizes[p] = nil end
    if characterConnections[p] then characterConnections[p]:Disconnect() characterConnections[p] = nil end
end))

table.insert(connections, RunService.RenderStepped:Connect(function()
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            aplicarEfectosAEnemigo(otherPlayer)
        end
    end
end))

print("¡Elian Hub cargado correctamente!")
