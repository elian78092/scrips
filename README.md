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
local Camera = workspace.CurrentCamera
local player = Players.LocalPlayer

-- =========================================================================
-- CONFIGURACIÓN DE SEGURIDAD ESTRICTA POR ID Y NOMBRE DE JUEGO
-- =========================================================================
local DUELIST_PLACE_ID = 111166698622146 
local DUELIST_FALLBACK_ID = 18780604245 

local successName, gameInfo = pcall(function()
    return MarketPlaceService:GetProductInfo(game.PlaceId)
end)

local gameName = string.lower(successName and gameInfo and gameInfo.Name or "")
local esDuelistPorNombre = string.find(gameName, "duelist") ~= nil or string.find(gameName, "duelista") ~= nil
local esDuelistPorId = (game.PlaceId == DUELIST_PLACE_ID)

-- Variables globales de control para el menú principal y la advertencia
local DeviceGui = nil
local ScreenGui = nil

local function destruirMenus()
    pcall(function()
        if CoreGui:FindFirstChild("ErrorMenu") then CoreGui.ErrorMenu:Destroy() end
        if CoreGui:FindFirstChild("DeviceSelectorGui") then CoreGui.DeviceSelectorGui:Destroy() end
        if CoreGui:FindFirstChild("ProfessionalMenuV9") then CoreGui.ProfessionalMenuV9:Destroy() end
        if DeviceGui then DeviceGui:Destroy() end
        if ScreenGui then ScreenGui:Destroy() end
    end)
end

local function mostrarMensajeErrorYTeletransportar()
    destruirMenus()
    pcall(function()
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
        Text.Text = "⚠ ¡ESTE SCRIPT ES SOLAMENTE PARA DUELISTA: PvP!\n\nEscribe 'dueñoe' en el chat para forzar la carga del menú o serás redirigido..."
        Text.Parent = AlertFrame
    end)
end

-- =========================================================================
-- VARIABLES GLOBALES Y DE ESTADO
-- =========================================================================
local aimbotEnabled = false           
local headHitboxEnabled = false       
local silentAimHitboxEnabled = false  
local headHitboxSize = 6 

local espBoxEnabled = false
local espNameEnabled = false
local espBodyEnabled = false
local espDistance = 300 

local espBoxes = {}
local espNames = {}
local espBodies = {}
local originalHeadSizes = {}
local characterConnections = {}
local connections = {}

local dispositivoSeleccionado = "PC" -- Por defecto al forzar con comando

-- =========================================================================
-- SISTEMA DE TEAM CHECK DINÁMICO
-- =========================================================================
local function isTeammate(otherPlayer)
    if otherPlayer == player then return true end
    
    local success, result = pcall(function()
        if player.Team and otherPlayer.Team then
            if player.Team == otherPlayer.Team then
                return true
            end
        end

        if otherPlayer:GetAttribute("Team") and player:GetAttribute("Team") then
            if otherPlayer:GetAttribute("Team") == player:GetAttribute("Team") then
                return true
            end
        end

        local myChar = player.Character
        local otherChar = otherPlayer.Character
        
        if myChar and otherChar then
            if otherChar:FindFirstChild("Team") and myChar:FindFirstChild("Team") then
                if otherChar.Team.Value == myChar.Team.Value then
                    return true
                end
            end
        end

        return false
    end)

    return success and result or false
end

local function restoreOriginalHeads()
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        pcall(function()
            if originalHeadSizes[otherPlayer] and otherPlayer.Character then
                local head = otherPlayer.Character:FindFirstChild("Head")
                if head then
                    head.Size = originalHeadSizes[otherPlayer].size
                    head.Transparency = originalHeadSizes[otherPlayer].transparency
                    head.CanCollide = originalHeadSizes[otherPlayer].cancollide
                    head.Massless = originalHeadSizes[otherPlayer].massless
                    head.CastShadow = originalHeadSizes[otherPlayer].castshadow
                end
            end
        end)
    end
    originalHeadSizes = {}
end

local function fullCleanup()
    aimbotEnabled = false
    headHitboxEnabled = false
    silentAimHitboxEnabled = false
    restoreOriginalHeads()

    for _, boxGui in pairs(espBoxes) do pcall(function() if boxGui then boxGui:Destroy() end end) end
    espBoxes = {}

    for _, nameGui in pairs(espNames) do pcall(function() if nameGui then nameGui:Destroy() end end) end
    espNames = {}

    for _, highlight in pairs(espBodies) do pcall(function() if highlight then highlight:Destroy() end end) end
    espBodies = {}

    for _, conn in pairs(characterConnections) do pcall(function() if conn then conn:Disconnect() end end) end
    characterConnections = {}

    for _, conn in pairs(connections) do pcall(function() if conn then conn:Disconnect() end end) end
    connections = {}
end

-- =========================================================================
-- FUNCIÓN PARA INICIALIZAR EL MENÚ PRINCIPAL
-- =========================================================================
local function iniciarMenuPrincipal()
    destruirMenus()

    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "ProfessionalMenuV9"
    ScreenGui.Parent = CoreGui
    ScreenGui.ResetOnSpawn = false
    ScreenGui.IgnoreGuiInset = true

    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Parent = ScreenGui
    MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
    MainFrame.Position = UDim2.new(0.5, -240, 0.5, -175)
    MainFrame.Size = UDim2.new(0, 480, 0, 375)
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true

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
    TopBar.ClipsDescendants = true

    local TopCorner = Instance.new("UICorner")
    TopCorner.CornerRadius = UDim.new(0, 14)
    TopCorner.Parent = TopBar

    local TopFix = Instance.new("Frame")
    TopFix.Parent = TopBar
    TopFix.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
    TopFix.Position = UDim2.new(0, 0, 0.7, 0)
    TopFix.Size = UDim2.new(1, 0, 0.3, 0)
    TopFix.BorderSizePixel = 0

    local Title = Instance.new("TextLabel")
    Title.Parent = TopBar
    Title.BackgroundTransparency = 1
    Title.Position = UDim2.new(0.04, 0, 0, 0)
    Title.Size = UDim2.new(0.6, 0, 1, 0)
    Title.Font = Enum.Font.GothamBold
    Title.Text = "⚡ ELIAN HUB (" .. dispositivoSeleccionado .. ")"
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
    TabContainer.Size = UDim2.new(0, 125, 0, 300)
    TabContainer.BorderSizePixel = 0

    local TabCorner = Instance.new("UICorner")
    TabCorner.CornerRadius = UDim.new(0, 10)
    TabCorner.Parent = TabContainer

    local CombatTabBtn = Instance.new("TextButton")
    CombatTabBtn.Parent = TabContainer
    CombatTabBtn.BackgroundColor3 = Color3.fromRGB(255, 30, 70)
    CombatTabBtn.Position = UDim2.new(0.08, 0, 0.05, 0)
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
    VisualTabBtn.Position = UDim2.new(0.08, 0, 0.20, 0)
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
    PagesContainer.Size = UDim2.new(0, 315, 0, 300)
    PagesContainer.BackgroundTransparency = 1

    local CombatPage = Instance.new("ScrollingFrame")
    CombatPage.Parent = PagesContainer
    CombatPage.BackgroundTransparency = 1
    CombatPage.Size = UDim2.new(1, 0, 1, 0)
    CombatPage.CanvasSize = UDim2.new(0, 0, 1.4, 0)
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

    local minimized = false
    MinimizeBtn.MouseButton1Click:Connect(function()
        minimized = not minimized
        ContainerBody.Visible = not minimized
        if minimized then
            MainFrame:TweenSize(UDim2.new(0, 480, 0, 48), "Out", "Quart", 0.25, true)
            MinimizeBtn.Text = "+"
        else
            MainFrame:TweenSize(UDim2.new(0, 480, 0, 375), "Out", "Quart", 0.25, true)
            MinimizeBtn.Text = "—"
        end
    end)

    CloseBtn.MouseButton1Click:Connect(function()
        fullCleanup()
        ScreenGui:Destroy()
    end)

    task.spawn(function()
        while ScreenGui and ScreenGui.Parent do
            pcall(function()
                local velocidad = 0.8
                local hue = (tick() * velocidad) % 1
                local colorRgb = Color3.fromHSV(hue, 0.7, 1)
                
                Title.TextColor3 = colorRgb
                MainStroke.Color = colorRgb
            end)
            RunService.RenderStepped:Wait()
        end
    end)

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

        local dragging = false
        local function updateValue(input)
            local pos = math.clamp((input.Position.X - Bar.AbsolutePosition.X) / Bar.AbsoluteSize.X, 0, 1)
            local val = math.floor(min + ((max - min) * pos))
            Fill.Size = UDim2.new(pos, 0, 1, 0)
            ValueLabel.Text = tostring(val)
            callback(val)
        end

        table.insert(connections, Bar.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then 
                dragging = true 
                updateValue(input) 
            end
        end))
        table.insert(connections, UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then 
                dragging = false 
            end
        end))
        table.insert(connections, UserInputService.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then 
                updateValue(input) 
            end
        end))
    end

    createToggle(CombatPage, "Aimbot Suave (Mantener Clic Derecho)", 10, function(state) aimbotEnabled = state end)
    createToggle(CombatPage, "Hitbox", 60, function(state)
        headHitboxEnabled = state
        if not state then restoreOriginalHeads() end
    end)
    createSlider(CombatPage, "Head Hitbox Size", 2, 35, 6, 110, function(val) headHitboxSize = val end)
    createToggle(CombatPage, "Aimbot Silencioso", 170, function(state)
        silentAimHitboxEnabled = state
        if not state then restoreOriginalHeads() end
    end)

    createToggle(VisualPage, "ESP Neon Box", 10, function(state) espBoxEnabled = state end)
    createToggle(VisualPage, "ESP Name", 60, function(state) espNameEnabled = state end)
    createToggle(VisualPage, "ESP Cuerpo Azul", 110, function(state) espBodyEnabled = state end)
    createSlider(VisualPage, "ESP Max Distance", 50, 300, 300, 160, function(val) espDistance = val end)
end

-- =========================================================================
-- VALIDACIÓN INICIAL Y DETECTOR DEL COMANDO EN CHAT ("dueñoe")
-- =========================================================================
local forzadoPorComando = false

local function configurarEscuchaChat()
    local function procesarTexto(texto)
        if string.lower(texto) == "dueñoe" then
            forzadoPorComando = true
            iniciarMenuPrincipal()
        end
    end

    local successTextChat, textChatService = pcall(function() return game:GetService("TextChatService") end)
    if successTextChat and textChatService then
        pcall(function()
            textChatService.MessageReceived:Connect(function(message)
                if message.TextSource and message.TextSource.UserId == player.UserId then
                    procesarTexto(message.Text)
                end
            end)
        end)
    end

    pcall(function()
        player.Chatted:Connect(function(mensaje)
            procesarTexto(mensaje)
        end)
    end)
end

configurarEscuchaChat()

if not esDuelistPorId and not esDuelistPorNombre then
    mostrarMensajeErrorYTeletransportar()
    
    task.spawn(function()
        local tiempoTranscurrido = 0
        while tiempoTranscurrido < 4 do
            if forzadoPorComando then return end
            task.wait(0.5)
            tiempoTranscurrido = tiempoTranscurrido + 0.5
        end
        
        if not forzadoPorComando then
            pcall(function()
                local teleportOptions = Instance.new("TeleportOptions")
                TeleportService:TeleportAsync(DUELIST_FALLBACK_ID, {player}, teleportOptions)
            end)
        end
    end)
    return 
else
    iniciarMenuPrincipal()
end

-- =========================================================================
-- LOGICA DE AIMBOT SUAVE Y FLUIDO (SOLO PC) + TEAM CHECK
-- =========================================================================
local function getClosestEnemyInFOV()
    local closestTarget = nil
    local shortestDistance = 300 
    local myRoot = player.Character and player.Character:FindFirstChild("HumanoidRootPart")

    if not myRoot then return nil end

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and not isTeammate(otherPlayer) then
            local char = otherPlayer.Character
            if char then
                local hum = char:FindFirstChild("Humanoid")
                local head = char:FindFirstChild("Head")
                local rootPart = char:FindFirstChild("HumanoidRootPart")

                if hum and hum.Health > 0 and head and rootPart then
                    local distance = (rootPart.Position - myRoot.Position).Magnitude
                    if distance <= shortestDistance then
                        shortestDistance = distance
                        closestTarget = head
                    end
                end
            end
        end
    end

    return closestTarget
end

table.insert(connections, RunService.RenderStepped:Connect(function(dt)
    if aimbotEnabled then
        if UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
            local targetHead = getClosestEnemyInFOV()
            if targetHead then
                local currentCFrame = Camera.CFrame
                local targetCFrame = CFrame.new(currentCFrame.Position, targetHead.Position)
                
                local smoothness = math.clamp(dt * 15, 0.05, 1)
                Camera.CFrame = currentCFrame:Lerp(targetCFrame, smoothness)
            end
        end
    end
end))

-- =========================================================================
-- SISTEMA DE GESTIÓN DE TAMAÑO Y VISIBILIDAD DE CABEZAS
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
                massless = head.Massless,
                castshadow = head.CastShadow
            }
        end
        
        local isAlive = hum.Health > 0
        local isEnemy = isAlive and not isTeammate(otherPlayer)
        local distance = myRoot and (rootPart.Position - myRoot.Position).Magnitude or 9999
        local inRange = distance <= espDistance

        if isEnemy and inRange then
            if silentAimHitboxEnabled then
                head.Size = Vector3.new(25, 25, 25)
                head.Transparency = 1 
                head.CanCollide = false
                head.Massless = true
                head.CastShadow = false
            elseif headHitboxEnabled then
                local safeSize = math.clamp(headHitboxSize, 2, 35)
                head.Size = Vector3.new(safeSize, safeSize, safeSize)
                head.Transparency = 0.5 
                head.CanCollide = false
                head.Massless = true
                head.CastShadow = false
            else
                if originalHeadSizes[otherPlayer] then
                    head.Size = originalHeadSizes[otherPlayer].size
                    head.Transparency = originalHeadSizes[otherPlayer].transparency
                    head.CanCollide = originalHeadSizes[otherPlayer].cancollide
                    head.Massless = originalHeadSizes[otherPlayer].massless
                    head.CastShadow = originalHeadSizes[otherPlayer].castshadow
                end
            end
        else
            if originalHeadSizes[otherPlayer] then
                head.Size = originalHeadSizes[otherPlayer].size
                head.Transparency = originalHeadSizes[otherPlayer].transparency
                head.CanCollide = originalHeadSizes[otherPlayer].cancollide
                head.Massless = originalHeadSizes[otherPlayer].massless
                head.CastShadow = originalHeadSizes[otherPlayer].castshadow
            end
        end
    end
end

-- =========================================================================
-- BUCLE DE RENDERIZADO (ESP Y CUERPO AZUL)
-- =========================================================================
local function aplicarEfectosAEnemigo(otherPlayer)
    pcall(function()
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

        if espBodyEnabled and isEnemy and inRange then
            local highlight = espBodies[otherPlayer]
            if not highlight or highlight.Parent ~= char then
                if highlight then highlight:Destroy() end
                highlight = Instance.new("Highlight")
                highlight.Name = "ESP_Body_Blue"
                highlight.Adornee = char
                highlight.FillColor = Color3.fromRGB(0, 120, 255)
                highlight.FillTransparency = 0.5
                highlight.OutlineColor = Color3.fromRGB(0, 200, 255)
                highlight.OutlineTransparency = 0
                highlight.Parent = char
                espBodies[otherPlayer] = highlight
            end
            highlight.Enabled = true
        else
            if espBodies[otherPlayer] then espBodies[otherPlayer].Enabled = false end
        end

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
    end)
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
    if espBodies[p] then espBodies[p]:Destroy() espBodies[p] = nil end
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
