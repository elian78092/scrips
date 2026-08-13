-- =========================================================================
-- ELIAN HUB - CORRECCIÓN DE ERRORES, CURSOR Y CIERRE TOTAL
-- =========================================================================

local DUELISTS_PLACE_ID = 0 

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LogService = game:GetService("LogService")

local player = Players.LocalPlayer
local parentGui = pcall(function() return CoreGui.Name end) and CoreGui or player:WaitForChild("PlayerGui")
local camera = workspace.CurrentCamera

-- Notificación flotante estilizada
local function mostrarNotificacion(titulo, mensaje, duracion)
    pcall(function()
        local oldNotif = parentGui:FindFirstChild("ElianNotification")
        if oldNotif then oldNotif:Destroy() end

        local NotifGui = Instance.new("ScreenGui")
        NotifGui.Name = "ElianNotification"
        NotifGui.Parent = parentGui
        NotifGui.ResetOnSpawn = false
        NotifGui.IgnoreGuiInset = true

        local NotifFrame = Instance.new("Frame")
        NotifFrame.Parent = NotifGui
        NotifFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
        NotifFrame.Position = UDim2.new(1, 20, 0.85, 0)
        NotifFrame.Size = UDim2.new(0, 300, 0, 75)
        NotifFrame.BorderSizePixel = 0

        local NotifCorner = Instance.new("UICorner")
        NotifCorner.CornerRadius = UDim.new(0, 14)
        NotifCorner.Parent = NotifFrame

        local NotifStroke = Instance.new("UIStroke")
        NotifStroke.Thickness = 1.5
        NotifStroke.Color = Color3.fromRGB(120, 80, 255)
        NotifStroke.Parent = NotifFrame

        local TitleLbl = Instance.new("TextLabel")
        TitleLbl.Parent = NotifFrame
        TitleLbl.BackgroundTransparency = 1
        TitleLbl.Position = UDim2.new(0, 18, 0, 12)
        TitleLbl.Size = UDim2.new(1, -25, 0, 20)
        TitleLbl.Font = Enum.Font.GothamBold
        TitleLbl.Text = titulo
        TitleLbl.TextColor3 = Color3.fromRGB(160, 120, 255)
        TitleLbl.TextSize = 13
        TitleLbl.TextXAlignment = Enum.TextXAlignment.Left

        local MsgLbl = Instance.new("TextLabel")
        MsgLbl.Parent = NotifFrame
        MsgLbl.BackgroundTransparency = 1
        MsgLbl.Position = UDim2.new(0, 18, 0, 36)
        MsgLbl.Size = UDim2.new(1, -25, 0, 25)
        MsgLbl.Font = Enum.Font.GothamMedium
        MsgLbl.Text = mensaje
        MsgLbl.TextColor3 = Color3.fromRGB(180, 190, 210)
        MsgLbl.TextSize = 11
        MsgLbl.TextXAlignment = Enum.TextXAlignment.Left

        NotifFrame:TweenPosition(UDim2.new(1, -320, 0.85, 0), "Out", "Quad", 0.4, true)

        task.delay(duracion or 4, function()
            pcall(function()
                NotifFrame:TweenPosition(UDim2.new(1, 20, 0.85, 0), "In", "Quad", 0.4, true)
                task.wait(0.4)
                NotifGui:Destroy()
            end)
        end)
    end)
end

-- =========================================================================
-- VALIDACIÓN DE JUEGO
-- =========================================================================
local nombreJuego = ""
pcall(function()
    nombreJuego = game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId).Name
end)

local esDuelist = string.find(string.lower(nombreJuego), "duelist") ~= nil or string.find(string.lower(nombreJuego), "duels") ~= nil or game.PlaceId == DUELISTS_PLACE_ID

if not esDuelist then
    mostrarNotificacion("¡Error de Juego!", "Este script es exclusivo para Duelist.", 3)
    task.wait(2)
    player:Kick("pendejo es para duelist el scrips ._.")
    return
end

if not game:IsLoaded() then
    game.Loaded:Wait()
end

-- Variables de Estado
local headHitboxEnabled = false      
local headHitboxSize = 10 
local torsoHitboxEnabled = false
local torsoHitboxSize = 10

local aimbotEnabled = false
local aimDistance = 500
local isRightClicking = false
local currentTargetHead = nil

local espNameEnabled = false
local espBodyEnabled = false
local espDistance = 1500

local flyEnabled = false
local flySpeed = 50
local speedEnabled = false
local walkSpeedValue = 16
local noclipEnabled = false

local espNames = {}
local espBodies = {}
local originalPartsData = {} 
local connections = {}

local bv, bg

-- =========================================================================
-- BYPASS Y ANTI-DETECCIÓN
-- =========================================================================
pcall(function()
    local mt = getrawmetatable(game)
    if mt then
        setreadonly(mt, false)
        local oldNamecall = mt.__namecall
        mt.__namecall = newcclosure(function(self, ...)
            local method = getnamecallmethod()
            if (method == "Kick" or method == "kick") and self ~= player then
                return nil
            end
            return oldNamecall(self, ...)
        end)
        setreadonly(mt, true)
    end
end)

local function restoreParts()
    for plr, dataMap in pairs(originalPartsData) do
        for partName, data in pairs(dataMap) do
            pcall(function()
                if data and data.Part and data.Part.Parent then
                    local part = data.Part
                    part.Size = data.Size
                    part.Transparency = data.Transparency
                    part.Color = data.Color
                    part.Material = data.Material
                    part.CanCollide = data.CanCollide
                    part.Massless = data.Massless
                end
            end)
        end
    end
    originalPartsData = {}
end

local function cleanESPVisuals()
    for _, nameGui in pairs(espNames) do pcall(function() if nameGui then nameGui:Destroy() end end) end
    espNames = {}
    for _, highlight in pairs(espBodies) do pcall(function() if highlight then highlight:Destroy() end end) end
    espBodies = {}
end

local function fullCleanup()
    headHitboxEnabled = false
    torsoHitboxEnabled = false
    aimbotEnabled = false
    flyEnabled = false
    speedEnabled = false
    noclipEnabled = false
    currentTargetHead = nil
    
    restoreParts()
    cleanESPVisuals()

    pcall(function()
        local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum.WalkSpeed = 16 end
    end)

    pcall(function()
        if bv then bv:Destroy() end
        if bg then bg:Destroy() end
        local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            local oldBv = hrp:FindFirstChildOfClass("BodyVelocity")
            local oldBg = hrp:FindFirstChildOfClass("BodyGyro")
            if oldBv then oldBv:Destroy() end
            if oldBg then oldBg:Destroy() end
        end
        local humanoid = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
        if humanoid then humanoid.PlatformStand = false end
    end)
    bv = nil
    bg = nil

    for _, conn in pairs(connections) do pcall(function() if conn then conn:Disconnect() end end) end
    connections = {}
end

local function destruirMenus()
    fullCleanup()
    pcall(function()
        if parentGui:FindFirstChild("ElianHubLoading") then parentGui.ElianHubLoading:Destroy() end
        if parentGui:FindFirstChild("ElianHubMenu") then parentGui.ElianHubMenu:Destroy() end
        if parentGui:FindFirstChild("ElianHubRestoreBtn") then parentGui.ElianHubRestoreBtn:Destroy() end
        if parentGui:FindFirstChild("ElianNotification") then parentGui.ElianNotification:Destroy() end
    end)
end

destruirMenus()

local rawExecutor = (identifyexecutor and identifyexecutor()) or "Desconocido"
local isMobileDevice = (not UserInputService.KeyboardEnabled) and (UserInputService.TouchEnabled or table.find({"Delta", "Hydrogen", "Codex", "Fluxus", "ArceusX", "KRNL", "Synapse"}, rawExecutor) ~= nil)
if UserInputService.TouchEnabled and not isMobileDevice then
    isMobileDevice = true
end

table.insert(connections, UserInputService.InputBegan:Connect(function(input, gpe)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isRightClicking = true
    end
end))

table.insert(connections, UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isRightClicking = false
    end
end))

local function getRealHead(character)
    if not character then return nil end
    return character:FindFirstChild("Head") or character:FindFirstChild("FakeHead")
end

local function getTorsoPart(character)
    if not character then return nil end
    return character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso") or character:FindFirstChild("HumanoidRootPart")
end

local function updateClosestHead()
    if isMobileDevice then return end
    local closest = nil
    local shortestDist = math.huge
    local mousePos = UserInputService:GetMouseLocation()

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            pcall(function()
                local head = getRealHead(otherPlayer.Character)
                local humanoid = otherPlayer.Character:FindFirstChildOfClass("Humanoid")
                local rootPart = otherPlayer.Character:FindFirstChild("HumanoidRootPart")
                
                if head and rootPart and humanoid and humanoid.Health > 0 then
                    local worldDist = (rootPart.Position - camera.CFrame.Position).Magnitude
                    if worldDist <= aimDistance then
                        local screenPos, onScreen = camera:WorldToViewportPoint(head.Position)
                        if onScreen then
                            local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                            if dist < shortestDist then
                                shortestDist = dist
                                closest = head
                            end
                        end
                    end
                end
            end)
        end
    end
    currentTargetHead = closest
end

-- =========================================================================
-- INTERFAZ MODERNA (CORREGIDA SIN BLOQUEO DE MOUSE)
-- =========================================================================
local ScreenGui = nil

local function iniciarMenuPrincipal()
    destruirMenus()

    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "ElianHubMenu"
    ScreenGui.Parent = parentGui
    ScreenGui.ResetOnSpawn = false
    ScreenGui.IgnoreGuiInset = true
    -- IMPORTANTE: Evita que la interfaz bloquee el movimiento de la cámara/mouse del juego
    ScreenGui.DisplayOrder = 999

    local LoadGui = Instance.new("ScreenGui")
    LoadGui.Name = "ElianHubLoading"
    LoadGui.Parent = parentGui
    LoadGui.ResetOnSpawn = false
    LoadGui.IgnoreGuiInset = true

    local LoadFrame = Instance.new("Frame")
    LoadFrame.Parent = LoadGui
    LoadFrame.BackgroundColor3 = Color3.fromRGB(12, 14, 20)
    LoadFrame.Position = UDim2.new(0.5, -160, 0.5, -60)
    LoadFrame.Size = UDim2.new(0, 320, 0, 120)
    LoadFrame.BorderSizePixel = 0

    local LoadCorner = Instance.new("UICorner")
    LoadCorner.CornerRadius = UDim.new(0, 16)
    LoadCorner.Parent = LoadFrame

    local LoadStroke = Instance.new("UIStroke")
    LoadStroke.Thickness = 1.5
    LoadStroke.Color = Color3.fromRGB(110, 70, 255)
    LoadStroke.Parent = LoadFrame

    local LoadTitle = Instance.new("TextLabel")
    LoadTitle.Parent = LoadFrame
    LoadTitle.BackgroundTransparency = 1
    LoadTitle.Position = UDim2.new(0, 0, 0, 18)
    LoadTitle.Size = UDim2.new(1, 0, 0, 25)
    LoadTitle.Font = Enum.Font.GothamBold
    LoadTitle.Text = "💎 ELIAN HUB"
    LoadTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
    LoadTitle.TextSize = 15

    local LoadSub = Instance.new("TextLabel")
    LoadSub.Parent = LoadFrame
    LoadSub.BackgroundTransparency = 1
    LoadSub.Position = UDim2.new(0, 0, 0, 42)
    LoadSub.Size = UDim2.new(1, 0, 0, 20)
    LoadSub.Font = Enum.Font.GothamMedium
    LoadSub.Text = "Cargando Bypass..."
    LoadSub.TextColor3 = Color3.fromRGB(150, 160, 185)
    LoadSub.TextSize = 11

    local BarBack = Instance.new("Frame")
    BarBack.Parent = LoadFrame
    BarBack.BackgroundColor3 = Color3.fromRGB(20, 23, 33)
    BarBack.Position = UDim2.new(0.1, 0, 0, 75)
    BarBack.Size = UDim2.new(0.8, 0, 0, 6)
    BarBack.BorderSizePixel = 0

    local BarCorner = Instance.new("UICorner")
    BarCorner.CornerRadius = UDim.new(0, 3)
    BarCorner.Parent = BarBack

    local BarFill = Instance.new("Frame")
    BarFill.Parent = BarBack
    BarFill.BackgroundColor3 = Color3.fromRGB(130, 90, 255)
    BarFill.Size = UDim2.new(0, 0, 1, 0)
    BarFill.BorderSizePixel = 0

    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(0, 3)
    FillCorner.Parent = BarFill

    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Parent = ScreenGui
    MainFrame.BackgroundColor3 = Color3.fromRGB(12, 14, 20)
    
    if isMobileDevice then
        MainFrame.Position = UDim2.new(0.5, -230, 0.5, -175)
        MainFrame.Size = UDim2.new(0, 460, 0, 350)
    else
        MainFrame.Position = UDim2.new(0.5, -280, 0.5, -210)
        MainFrame.Size = UDim2.new(0, 560, 0, 420)
    end
    
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true
    MainFrame.Visible = false

    local MainCorner = Instance.new("UICorner")
    MainCorner.CornerRadius = UDim.new(0, 16)
    MainCorner.Parent = MainFrame

    local MainStroke = Instance.new("UIStroke")
    MainStroke.Thickness = 1.5
    MainStroke.Color = Color3.fromRGB(110, 70, 255)
    MainStroke.Parent = MainFrame

    -- PANTALLA DE CARGA DE 10 SEGUNDOS EXACTOS
    task.spawn(function()
        local steps = {"Cargando Bypass...", "Aplicando ganchos...", "Verificando seguridad...", "Iniciando Elian Hub..."}
        for i = 1, 10 do
            LoadSub.Text = steps[math.floor((i - 1) / 3) + 1] or "Iniciando Elian Hub..."
            task.wait(1)
        end
        TweenService:Create(BarFill, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.new(1, 0, 1, 0)}):Play()
        task.wait(0.5)
        pcall(function() LoadGui:Destroy() end)
        if MainFrame then MainFrame.Visible = true end
        mostrarNotificacion("💎 Elian Hub", "Cargado de forma segura.", 2.5)
    end)

    local TopBar = Instance.new("Frame")
    TopBar.Parent = MainFrame
    TopBar.BackgroundTransparency = 1
    TopBar.Size = UDim2.new(1, 0, 0, 45)

    local TitleContainer = Instance.new("Frame")
    TitleContainer.Parent = TopBar
    TitleContainer.BackgroundTransparency = 1
    TitleContainer.Position = UDim2.new(0.04, 0, 0, 0)
    TitleContainer.Size = UDim2.new(0.4, 0, 1, 0)

    local TitleLayout = Instance.new("UIListLayout")
    TitleLayout.Parent = TitleContainer
    TitleLayout.FillDirection = Enum.FillDirection.Horizontal
    TitleLayout.VerticalAlignment = Enum.VerticalAlignment.Center
    TitleLayout.Padding = UDim.new(0, 8)

    local LogoBadge = Instance.new("Frame")
    LogoBadge.Parent = TitleContainer
    LogoBadge.BackgroundColor3 = Color3.fromRGB(110, 70, 255)
    LogoBadge.Size = UDim2.new(0, 24, 0, 24)
    
    local LogoCorner = Instance.new("UICorner")
    LogoCorner.CornerRadius = UDim.new(0, 6)
    LogoCorner.Parent = LogoBadge

    local LogoText = Instance.new("TextLabel")
    LogoText.Parent = LogoBadge
    LogoText.BackgroundTransparency = 1
    LogoText.Size = UDim2.new(1, 0, 1, 0)
    LogoText.Font = Enum.Font.GothamBold
    LogoText.Text = "💎"
    LogoText.TextColor3 = Color3.fromRGB(255, 255, 255)
    LogoText.TextSize = 12

    local Title = Instance.new("TextLabel")
    Title.Parent = TitleContainer
    Title.BackgroundTransparency = 1
    Title.Size = UDim2.new(0, 180, 1, 0)
    Title.Font = Enum.Font.GothamBold
    Title.Text = "ELIAN HUB"
    Title.TextColor3 = Color3.fromRGB(240, 240, 255)
    Title.TextSize = 12
    Title.TextXAlignment = Enum.TextXAlignment.Left

    local TopActions = Instance.new("Frame")
    TopActions.Parent = TopBar
    TopActions.BackgroundTransparency = 1
    TopActions.Position = UDim2.new(0.45, 0, 0.15, 0)
    TopActions.Size = UDim2.new(0.51, 0, 0, 30)

    local UIListLayout = Instance.new("UIListLayout")
    UIListLayout.Parent = TopActions
    UIListLayout.FillDirection = Enum.FillDirection.Horizontal
    UIListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Right
    UIListLayout.Padding = UDim.new(0, 6)

    local MinimizeBtn = Instance.new("TextButton")
    MinimizeBtn.Parent = TopActions
    MinimizeBtn.BackgroundColor3 = Color3.fromRGB(18, 22, 32)
    MinimizeBtn.Size = UDim2.new(0, 28, 0, 28)
    MinimizeBtn.Font = Enum.Font.GothamBold
    MinimizeBtn.Text = "—"
    MinimizeBtn.TextColor3 = Color3.fromRGB(160, 170, 200)
    MinimizeBtn.TextSize = 11

    local MinCorner = Instance.new("UICorner")
    MinCorner.CornerRadius = UDim.new(0, 6)
    MinCorner.Parent = MinimizeBtn

    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Parent = TopActions
    CloseBtn.BackgroundColor3 = Color3.fromRGB(235, 65, 90)
    CloseBtn.Size = UDim2.new(0, 28, 0, 28)
    CloseBtn.Font = Enum.Font.GothamBold
    CloseBtn.Text = "✕"
    CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    CloseBtn.TextSize = 11

    local CloseCorner = Instance.new("UICorner")
    CloseCorner.CornerRadius = UDim.new(0, 6)
    CloseCorner.Parent = CloseBtn

    local function crearBotonRestaurar()
        if parentGui:FindFirstChild("ElianHubRestoreBtn") then return end
        local RestoreGui = Instance.new("ScreenGui")
        RestoreGui.Name = "ElianHubRestoreBtn"
        RestoreGui.Parent = parentGui
        RestoreGui.ResetOnSpawn = false
        RestoreGui.IgnoreGuiInset = true

        local OpenBtn = Instance.new("TextButton")
        OpenBtn.Parent = RestoreGui
        OpenBtn.BackgroundColor3 = Color3.fromRGB(12, 14, 20)
        OpenBtn.Position = UDim2.new(0.02, 0, 0.35, 0)
        OpenBtn.Size = UDim2.new(0, isMobileDevice and 100 or 110, 0, 36)
        OpenBtn.Font = Enum.Font.GothamBold
        OpenBtn.Text = "💎 ABRIR"
        OpenBtn.TextColor3 = Color3.fromRGB(130, 90, 255)
        OpenBtn.TextSize = 11
        OpenBtn.AutoButtonColor = false

        local OCorner = Instance.new("UICorner")
        OCorner.CornerRadius = UDim.new(1, 0)
        OCorner.Parent = OpenBtn

        local OStroke = Instance.new("UIStroke")
        OStroke.Color = Color3.fromRGB(110, 70, 255)
        OStroke.Thickness = 1.2
        OStroke.Parent = OpenBtn

        local dragging = false
        local dragInput, dragStart, startPos
        local hasMoved = false

        OpenBtn.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                hasMoved = false
                dragStart = input.Position
                startPos = OpenBtn.Position

                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragging = false
                    end
                end)
            end
        end)

        OpenBtn.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
                dragInput = input
            end
        end)

        UserInputService.InputChanged:Connect(function(input)
            if dragging and input == dragInput then
                local delta = input.Position - dragStart
                if delta.Magnitude > 5 then
                    hasMoved = true
                end
                OpenBtn.Position = UDim2.new(
                    startPos.X.Scale, 
                    startPos.X.Offset + delta.X, 
                    startPos.Y.Scale, 
                    startPos.Y.Offset + delta.Y
                )
            end
        end)

        local function restaurar()
            if not hasMoved then
                MainFrame.Visible = true
                RestoreGui:Destroy()
            end
        end

        OpenBtn.MouseButton1Up:Connect(restaurar)
        OpenBtn.TouchEnded:Connect(restaurar)
    end

    MinimizeBtn.MouseButton1Click:Connect(function()
        MainFrame.Visible = false
        crearBotonRestaurar()
    end)
    MinimizeBtn.TouchTap:Connect(function()
        MainFrame.Visible = false
        crearBotonRestaurar()
    end)

    -- Cierre total real: borra la GUI y desactiva todas las funciones para que no siga corriendo nada
    CloseBtn.MouseButton1Click:Connect(destruirMenus)
    CloseBtn.TouchTap:Connect(destruirMenus)

    local ContainerBody = Instance.new("Frame")
    ContainerBody.Parent = MainFrame
    ContainerBody.BackgroundTransparency = 1
    ContainerBody.Position = UDim2.new(0, 0, 0, 45)
    ContainerBody.Size = UDim2.new(1, 0, 1, -45)

    local TabContainer = Instance.new("Frame")
    TabContainer.Parent = ContainerBody
    TabContainer.BackgroundColor3 = Color3.fromRGB(8, 10, 15)
    TabContainer.Position = UDim2.new(0.03, 0, 0.03, 0)
    TabContainer.Size = UDim2.new(0, isMobileDevice and 115 or 130, 0, isMobileDevice and 285 or 345)

    local TabCorner = Instance.new("UICorner")
    TabCorner.CornerRadius = UDim.new(0, 12)
    TabCorner.Parent = TabContainer

    local TabListLayout = Instance.new("UIListLayout")
    TabListLayout.Parent = TabContainer
    TabListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    TabListLayout.Padding = UDim.new(0, 5)

    local TabPadding = Instance.new("UIPadding")
    TabPadding.Parent = TabContainer
    TabPadding.PaddingTop = UDim.new(0, 8)

    local function createTabButton(name, text, order)
        local Btn = Instance.new("TextButton")
        Btn.Parent = TabContainer
        Btn.BackgroundColor3 = order == 1 and Color3.fromRGB(110, 70, 255) or Color3.fromRGB(14, 17, 24)
        Btn.Size = UDim2.new(0.88, 0, 0, isMobileDevice and 34 or 38)
        Btn.Font = Enum.Font.GothamBold
        Btn.Text = text
        Btn.TextColor3 = order == 1 and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(120, 130, 160)
        Btn.TextSize = 11
        Btn.BorderSizePixel = 0
        Btn.LayoutOrder = order

        local Corner = Instance.new("UICorner")
        Corner.CornerRadius = UDim.new(0, 8)
        Corner.Parent = Btn
        return Btn
    end

    local CombatTabBtn = createTabButton("Combat", "⚔ Combat", 1)
    local ESPTabBtn = createTabButton("ESP", "🎯 ESP", 2)
    local PlayersTabBtn = createTabButton("Players", "👥 Players", 3)
    local EtcTabBtn = createTabButton("Misc", "⚙ Misc", 4)

    local PagesContainer = Instance.new("Frame")
    PagesContainer.Parent = ContainerBody
    PagesContainer.Position = UDim2.new(0.28, 0, 0.03, 0)
    PagesContainer.Size = UDim2.new(0, isMobileDevice and 315 or 385, 0, isMobileDevice and 285 or 345)
    PagesContainer.BackgroundTransparency = 1

    local function createPage()
        local page = Instance.new("ScrollingFrame")
        page.Parent = PagesContainer
        page.BackgroundTransparency = 1
        page.Size = UDim2.new(1, 0, 1, 0)
        page.CanvasSize = UDim2.new(0, 0, 2.0, 0)
        page.ScrollBarThickness = 2
        page.ScrollBarImageColor3 = Color3.fromRGB(110, 70, 255)
        page.Visible = false
        return page
    end

    local CombatPage = createPage()
    CombatPage.Visible = true
    local ESPPage = createPage()
    local PlayersPage = createPage()
    local EtcPage = createPage()

    local function switchTab(selectedPage, activeBtn)
        CombatPage.Visible = false
        ESPPage.Visible = false
        PlayersPage.Visible = false
        EtcPage.Visible = false
        selectedPage.Visible = true

        for _, btn in ipairs({CombatTabBtn, ESPTabBtn, PlayersTabBtn, EtcTabBtn}) do
            TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(14, 17, 24), TextColor3 = Color3.fromRGB(120, 130, 160)}):Play()
        end
        TweenService:Create(activeBtn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(110, 70, 255), TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
    end

    CombatTabBtn.MouseButton1Click:Connect(function() switchTab(CombatPage, CombatTabBtn) end)
    CombatTabBtn.TouchTap:Connect(function() switchTab(CombatPage, CombatTabBtn) end)

    ESPTabBtn.MouseButton1Click:Connect(function() switchTab(ESPPage, ESPTabBtn) end)
    ESPTabBtn.TouchTap:Connect(function() switchTab(ESPPage, ESPTabBtn) end)

    PlayersTabBtn.MouseButton1Click:Connect(function() switchTab(PlayersPage, PlayersTabBtn) end)
    PlayersTabBtn.TouchTap:Connect(function() switchTab(PlayersPage, PlayersTabBtn) end)

    EtcTabBtn.MouseButton1Click:Connect(function() switchTab(EtcPage, EtcTabBtn) end)
    EtcTabBtn.TouchTap:Connect(function() switchTab(EtcPage, EtcTabBtn) end)

    local function createToggle(parent, name, posY, initialVal, callback)
        local ToggleBtn = Instance.new("TextButton")
        ToggleBtn.Parent = parent
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(15, 18, 26)
        ToggleBtn.Position = UDim2.new(0.02, 0, 0, posY)
        ToggleBtn.Size = UDim2.new(0.96, 0, 0, 36)
        ToggleBtn.Font = Enum.Font.GothamMedium
        ToggleBtn.Text = "     " .. name
        ToggleBtn.TextColor3 = Color3.fromRGB(200, 210, 230)
        ToggleBtn.TextSize = 11
        ToggleBtn.TextXAlignment = Enum.TextXAlignment.Left

        local TCorner = Instance.new("UICorner")
        TCorner.CornerRadius = UDim.new(0, 8)
        TCorner.Parent = ToggleBtn

        local StatusIndicator = Instance.new("Frame")
        StatusIndicator.Parent = ToggleBtn
        StatusIndicator.BackgroundColor3 = initialVal and Color3.fromRGB(110, 70, 255) or Color3.fromRGB(25, 30, 42)
        StatusIndicator.Position = UDim2.new(0.81, 0, 0.22, 0)
        StatusIndicator.Size = UDim2.new(0, 34, 0, 18)

        local SCorner = Instance.new("UICorner")
        SCorner.CornerRadius = UDim.new(0, 9)
        SCorner.Parent = StatusIndicator

        local Dot = Instance.new("Frame")
        Dot.Parent = StatusIndicator
        Dot.BackgroundColor3 = initialVal and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(120, 130, 160)
        Dot.Position = initialVal and UDim2.new(0.48, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0)
        Dot.Size = UDim2.new(0, 14, 0, 14)

        local DCorner = Instance.new("UICorner")
        DCorner.CornerRadius = UDim.new(0, 7)
        DCorner.Parent = Dot

        local enabled = initialVal
        local function triggerToggle()
            enabled = not enabled
            if enabled then
                TweenService:Create(StatusIndicator, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(110, 70, 255)}):Play()
                TweenService:Create(Dot, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(255, 255, 255)}):Play()
                Dot:TweenPosition(UDim2.new(0.48, 0, 0.1, 0), "Out", "Quad", 0.2, true)
            else
                TweenService:Create(StatusIndicator, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(25, 30, 42)}):Play()
                TweenService:Create(Dot, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(120, 130, 160)}):Play()
                Dot:TweenPosition(UDim2.new(0.08, 0, 0.1, 0), "Out", "Quad", 0.2, true)
            end
            callback(enabled)
        end

        ToggleBtn.MouseButton1Click:Connect(triggerToggle)
        ToggleBtn.TouchTap:Connect(triggerToggle)
    end

    local function createSlider(parent, name, min, max, default, posY, callback)
        local SliderFrame = Instance.new("Frame")
        SliderFrame.Parent = parent
        SliderFrame.BackgroundColor3 = Color3.fromRGB(15, 18, 26)
        SliderFrame.Position = UDim2.new(0.02, 0, 0, posY)
        SliderFrame.Size = UDim2.new(0.96, 0, 0, 48)

        local SFrameCorner = Instance.new("UICorner")
        SFrameCorner.CornerRadius = UDim.new(0, 8)
        SFrameCorner.Parent = SliderFrame

        local Label = Instance.new("TextLabel")
        Label.Parent = SliderFrame
        Label.BackgroundTransparency = 1
        Label.Position = UDim2.new(0.05, 0, 0.12, 0)
        Label.Text = name
        Label.TextColor3 = Color3.fromRGB(180, 190, 210)
        Label.TextSize = 11
        Label.TextXAlignment = Enum.TextXAlignment.Left
        Label.Font = Enum.Font.GothamMedium

        local ValueLabel = Instance.new("TextLabel")
        ValueLabel.Parent = SliderFrame
        ValueLabel.BackgroundTransparency = 1
        ValueLabel.Position = UDim2.new(0.05, 0, 0.12, 0)
        ValueLabel.Size = UDim2.new(0.9, 0, 0, 20)
        ValueLabel.Text = tostring(default)
        ValueLabel.TextColor3 = Color3.fromRGB(150, 110, 255)
        ValueLabel.TextSize = 11
        ValueLabel.Font = Enum.Font.GothamBold
        ValueLabel.TextXAlignment = Enum.TextXAlignment.Right

        local Bar = Instance.new("Frame")
        Bar.Parent = SliderFrame
        Bar.BackgroundColor3 = Color3.fromRGB(22, 26, 38)
        Bar.Position = UDim2.new(0.05, 0, 0.65, 0)
        Bar.Size = UDim2.new(0.9, 0, 0, 5)

        local BarCorner = Instance.new("UICorner")
        BarCorner.CornerRadius = UDim.new(0, 2.5)
        BarCorner.Parent = Bar

        local Fill = Instance.new("Frame")
        Fill.Parent = Bar
        Fill.BackgroundColor3 = Color3.fromRGB(110, 70, 255)
        Fill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)

        local FillCorner = Instance.new("UICorner")
        FillCorner.CornerRadius = UDim.new(0, 2.5)
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

    -- Elementos Combat
    createToggle(CombatPage, "Hitbox Cabeza", 6, headHitboxEnabled, function(state)
        headHitboxEnabled = state
        if not state then restoreParts() end
    end)
    createSlider(CombatPage, "Tamaño Cabeza (Max 45)", 2, 45, headHitboxSize, 50, function(val) headHitboxSize = val end)

    createToggle(CombatPage, "Hitbox Torso", 104, torsoHitboxEnabled, function(state)
        torsoHitboxEnabled = state
        if not state then restoreParts() end
    end)
    createSlider(CombatPage, "Tamaño Torso (Max 45)", 2, 45, torsoHitboxSize, 148, function(val) torsoHitboxSize = val end)

    if not isMobileDevice then
        createToggle(CombatPage, "Aimbot Directo (Click Dcho)", 202, aimbotEnabled, function(state) aimbotEnabled = state end)
    end

    -- Elementos ESP
    createToggle(ESPPage, "ESP Cuerpo Azul", 6, espBodyEnabled, function(state) espBodyEnabled = state end)
    createToggle(ESPPage, "ESP Nombre", 48, espNameEnabled, function(state) espNameEnabled = state end)
    createSlider(ESPPage, "Distancia ESP", 100, 5000, espDistance, 92, function(val) espDistance = val end)

    -- Elementos Misc
    createToggle(EtcPage, "Fly (Volar)", 6, flyEnabled, function(state) flyEnabled = state end)
    createSlider(EtcPage, "Velocidad Fly", 10, 300, flySpeed, 50, function(val) flySpeed = val end)

    createToggle(EtcPage, "Speed (Velocidad)", 104, speedEnabled, function(state) 
        speedEnabled = state 
        if not state then
            pcall(function()
                local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
                if hum then hum.WalkSpeed = 16 end
            end)
        end
    end)
    createSlider(EtcPage, "Valor WalkSpeed", 16, 200, walkSpeedValue, 148, function(val) walkSpeedValue = val end)

    createToggle(EtcPage, "Noclip (Atravesar)", 202, noclipEnabled, function(state)
        noclipEnabled = state
        if not state and player.Character then
            pcall(function()
                for _, part in ipairs(player.Character:GetDescendants()) do
                    if part:IsA("BasePart") then part.CanCollide = true end
                end
            end)
        end
    end)
end

iniciarMenuPrincipal()

-- Controles Fly y Noclip
local keysPressed = {W = false, A = false, S = false, D = false, Space = false, Shift = false}

table.insert(connections, UserInputService.InputBegan:Connect(function(input, gpe)
    if not gpe then
        if input.KeyCode == Enum.KeyCode.W then keysPressed.W = true end
        if input.KeyCode == Enum.KeyCode.S then keysPressed.S = true end
        if input.KeyCode == Enum.KeyCode.A then keysPressed.A = true end
        if input.KeyCode == Enum.KeyCode.D then keysPressed.D = true end
        if input.KeyCode == Enum.KeyCode.Space then keysPressed.Space = true end
        if input.KeyCode == Enum.KeyCode.LeftShift then keysPressed.Shift = true end
    end
end))

table.insert(connections, UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.W then keysPressed.W = false end
    if input.KeyCode == Enum.KeyCode.S then keysPressed.S = false end
    if input.KeyCode == Enum.KeyCode.A then keysPressed.A = false end
    if input.KeyCode == Enum.KeyCode.D then keysPressed.D = false end
    if input.KeyCode == Enum.KeyCode.Space then keysPressed.Space = false end
    if input.KeyCode == Enum.KeyCode.LeftShift then keysPressed.Shift = false end
end))

table.insert(connections, RunService.Stepped:Connect(function()
    local char = player.Character
    if noclipEnabled and char then
        pcall(function()
            for _, part in ipairs(char:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = false end
            end
        end)
    end
end))

table.insert(connections, RunService.RenderStepped:Connect(function()
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local humanoid = char and char:FindFirstChildOfClass("Humanoid")

    if speedEnabled and humanoid then
        humanoid.WalkSpeed = walkSpeedValue
    end

    if flyEnabled and hrp and humanoid then
        if not bv or not bg or bv.Parent ~= hrp then
            pcall(function()
                if bv then bv:Destroy() end
                if bg then bg:Destroy() end
            end)
            bv = Instance.new("BodyVelocity")
            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bv.Velocity = Vector3.new(0, 0, 0)
            bv.Parent = hrp

            bg = Instance.new("BodyGyro")
            bg.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
            bg.P = 9000
            bg.Parent = hrp
        end

        humanoid.PlatformStand = true
        
        local moveDir = Vector3.new(0, 0, 0)
        local camCFrame = camera.CFrame

        if isMobileDevice then
            if humanoid.MoveDirection.Magnitude > 0 then
                moveDir = (camCFrame.LookVector * Vector3.new(1, 0, 1)).Unit * humanoid.MoveDirection.Z + (camCFrame.RightVector * Vector3.new(1, 0, 1)).Unit * humanoid.MoveDirection.X
            end
        else
            if keysPressed.W then moveDir = moveDir + camCFrame.LookVector end
            if keysPressed.S then moveDir = moveDir - camCFrame.LookVector end
            if keysPressed.A then moveDir = moveDir - camCFrame.RightVector end
            if keysPressed.D then moveDir = moveDir + camCFrame.RightVector end
            if keysPressed.Space then moveDir = moveDir + Vector3.new(0, 1, 0) end
            if keysPressed.Shift then moveDir = moveDir - Vector3.new(0, 1, 0) end
        end

        if moveDir.Magnitude > 0 then
            bv.Velocity = moveDir.Unit * flySpeed
        else
            bv.Velocity = Vector3.new(0, 0.1, 0)
        end
        bg.CFrame = camCFrame
    else
        if bv then pcall(function() bv:Destroy() end) bv = nil end
        if bg then pcall(function() bg:Destroy() end) bg = nil end
        if humanoid then humanoid.PlatformStand = false end
    end
end))

-- =========================================================================
-- BUCLE PRINCIPAL DE HITBOXES Y ESP
-- =========================================================================
table.insert(connections, RunService.RenderStepped:Connect(function()
    if not isMobileDevice then
        updateClosestHead()
    end

    local function applyPartHitbox(otherPlayer, part, partName, isEnabled, sizeValue)
        if not part then return end
        if not originalPartsData[otherPlayer] then
            originalPartsData[otherPlayer] = {}
        end

        if isEnabled then
            if not originalPartsData[otherPlayer][partName] then
                originalPartsData[otherPlayer][partName] = {
                    Part = part,
                    Size = part.Size,
                    Transparency = part.Transparency,
                    Color = part.Color,
                    Material = part.Material,
                    CanCollide = part.CanCollide,
                    Massless = part.Massless
                }
            end

            part.Size = Vector3.new(sizeValue, sizeValue, sizeValue)
            part.Transparency = 0.6
            part.Color = partName == "Head" and Color3.fromRGB(130, 90, 255) or Color3.fromRGB(90, 180, 255)
            part.Material = Enum.Material.Neon
            part.CanCollide = false
            part.Massless = true
        elseif originalPartsData[otherPlayer] and originalPartsData[otherPlayer][partName] then
            local data = originalPartsData[otherPlayer][partName]
            pcall(function()
                if data and data.Part and data.Part.Parent then
                    data.Part.Size = data.Size
                    data.Part.Transparency = data.Transparency
                    data.Part.Color = data.Color
                    data.Part.Material = data.Material
                    data.Part.CanCollide = data.CanCollide
                    data.Part.Massless = data.Massless
                end
            end)
            originalPartsData[otherPlayer][partName] = nil
        end
    end

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            pcall(function()
                local character = otherPlayer.Character
                local humanoid = character:FindFirstChildOfClass("Humanoid")
                local head = getRealHead(character)
                local torso = getTorsoPart(character)
                
                if not humanoid or humanoid.Health <= 0 then
                    if originalPartsData[otherPlayer] then
                        for partName, data in pairs(originalPartsData[otherPlayer]) do
                            pcall(function()
                                if data and data.Part and data.Part.Parent then
                                    data.Part.Size = data.Size
                                    data.Part.Transparency = data.Transparency
                                    data.Part.Color = data.Color
                                    data.Part.Material = data.Material
                                    data.Part.CanCollide = data.CanCollide
                                    data.Part.Massless = data.Massless
                                end
                            end)
                        end
                        originalPartsData[otherPlayer] = nil
                    end
                else
                    applyPartHitbox(otherPlayer, head, "Head", headHitboxEnabled, headHitboxSize)
                    applyPartHitbox(otherPlayer, torso, "Torso", torsoHitboxEnabled, torsoHitboxSize)
                end
            end)
        elseif originalPartsData[otherPlayer] then
            pcall(function()
                for partName, data in pairs(originalPartsData[otherPlayer]) do
                    if data and data.Part and data.Part.Parent then
                        data.Part.Size = data.Size
                        data.Part.Transparency = data.Transparency
                        data.Part.Color = data.Color
                        data.Part.Material = data.Material
                        data.Part.CanCollide = data.CanCollide
                        data.Part.Massless = data.Massless
                    end
                end
            end)
            originalPartsData[otherPlayer] = nil
        end
    end

    if not isMobileDevice and aimbotEnabled and currentTargetHead then
        if isRightClicking then
            pcall(function()
                camera.CFrame = CFrame.new(camera.CFrame.Position, currentTargetHead.Position)
            end)
        end
    end

    -- Render ESP Visuals
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            pcall(function()
                local char = otherPlayer.Character
                local hrp = char:FindFirstChild("HumanoidRootPart")
                local head = getRealHead(char)
                local humanoid = char:FindFirstChildOfClass("Humanoid")
                
                if hrp and head and humanoid and humanoid.Health > 0 then
                    local dist = (hrp.Position - camera.CFrame.Position).Magnitude
                    if dist <= espDistance then
                        if espBodyEnabled then
                            local hl = espBodies[otherPlayer]
                            if not hl or not hl.Parent or hl.Adornee ~= char then
                                if hl then pcall(function() hl:Destroy() end) end
                                hl = Instance.new("Highlight")
                                hl.Name = "ElianESP_HL"
                                hl.FillColor = Color3.fromRGB(110, 70, 255)
                                hl.OutlineColor = Color3.fromRGB(255, 255, 255)
                                hl.FillTransparency = 0.5
                                hl.OutlineTransparency = 0
                                hl.Adornee = char
                                hl.Parent = char
                                espBodies[otherPlayer] = hl
                            else
                                hl.Enabled = true
                            end
                        else
                            if espBodies[otherPlayer] then espBodies[otherPlayer]:Destroy(); espBodies[otherPlayer] = nil end
                        end

                        if espNameEnabled then
                            local bGui = espNames[otherPlayer]
                            if not bGui or not bGui.Parent or bGui.Adornee ~= head then
                                if bGui then pcall(function() bGui:Destroy() end) end
                                bGui = Instance.new("BillboardGui")
                                bGui.Name = "ElianESP_Name"
                                bGui.Size = UDim2.new(0, 140, 0, 30)
                                bGui.AlwaysOnTop = true
                                bGui.Adornee = head
                                bGui.StudsOffset = Vector3.new(0, 2.2, 0)
                                bGui.Parent = char
                                
                                local txt = Instance.new("TextLabel")
                                txt.Name = "EspText"
                                txt.Parent = bGui
                                txt.Size = UDim2.new(1, 0, 1, 0)
                                txt.BackgroundTransparency = 1
                                txt.TextColor3 = Color3.fromRGB(180, 140, 255)
                                txt.Font = Enum.Font.GothamBold
                                txt.TextSize = isMobileDevice and 11 or 12
                                txt.TextStrokeTransparency = 0.4
                                espNames[otherPlayer] = bGui
                            else
                                bGui.Enabled = true
                                local txt = bGui:FindFirstChild("EspText")
                                if txt then txt.Text = otherPlayer.Name .. " [" .. math.floor(dist) .. "m]" end
                            end
                        else
                            if espNames[otherPlayer] then espNames[otherPlayer]:Destroy(); espNames[otherPlayer] = nil end
                        end
                    else
                        if espBodies[otherPlayer] then espBodies[otherPlayer]:Destroy(); espBodies[otherPlayer] = nil end
                        if espNames[otherPlayer] then espNames[otherPlayer]:Destroy(); espNames[otherPlayer] = nil end
                    end
                end
            end)
        end
    end
end))

table.insert(connections, player.CharacterAdded:Connect(function()
    task.wait(1)
    cleanESPVisuals()
end))
