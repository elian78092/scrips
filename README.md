-- =========================================================================
-- ELIAN HUB - INTERFAZ ULTRA PROFESIONAL V18.5 (AIMBOT CORREGIDO)
-- =========================================================================
if not game:IsLoaded() then
    game.Loaded:Wait()
end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local parentGui = pcall(function() return CoreGui.Name end) and CoreGui or player:WaitForChild("PlayerGui")
local ScreenGui = nil

local headHitboxEnabled = false      
local headHitboxSize = 10 

local aimbotEnabled = false
local aimDistance = 500
local isRightClicking = false
local currentTargetHead = nil

local espBoxEnabled = false
local espNameEnabled = false
local espBodyEnabled = false
local espDistance = 1500

local flyEnabled = false
local flySpeed = 50
local speedEnabled = false
local walkSpeedValue = 16
local noclipEnabled = false

local espBoxes = {}
local espNames = {}
local espBodies = {}
local originalHeadData = {} 
local connections = {}

local bv, bg

local function restoreHeads()
    for plr, data in pairs(originalHeadData) do
        pcall(function()
            if data and data.Head and data.Head.Parent then
                local head = data.Head
                head.Size = data.Size
                head.Transparency = data.Transparency
                head.Color = data.Color
                head.Material = data.Material
                head.CanCollide = data.CanCollide
                head.Massless = data.Massless
                
                local bp = head:FindFirstChild("ElianHitboxBP")
                if bp then bp:Destroy() end
            end
        end)
    end
    originalHeadData = {}
end

local function cleanESPVisuals()
    for _, boxGui in pairs(espBoxes) do pcall(function() if boxGui then boxGui:Destroy() end end) end
    espBoxes = {}

    for _, nameGui in pairs(espNames) do pcall(function() if nameGui then nameGui:Destroy() end end) end
    espNames = {}

    for _, highlight in pairs(espBodies) do pcall(function() if highlight then highlight:Destroy() end end) end
    espBodies = {}
end

local function fullCleanup()
    headHitboxEnabled = false
    aimbotEnabled = false
    flyEnabled = false
    speedEnabled = false
    noclipEnabled = false
    currentTargetHead = nil
    
    restoreHeads()
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
        if parentGui:FindFirstChild("ErrorMenu") then parentGui.ErrorMenu:Destroy() end
        if parentGui:FindFirstChild("ProfessionalMenuV12") then parentGui.ProfessionalMenuV12:Destroy() end
        if parentGui:FindFirstChild("ProfessionalMenuV13") then parentGui.ProfessionalMenuV13:Destroy() end
        if parentGui:FindFirstChild("ProfessionalMenuV14") then parentGui.ProfessionalMenuV14:Destroy() end
        if parentGui:FindFirstChild("ElianHubLoading") then parentGui.ElianHubLoading:Destroy() end
        if parentGui:FindFirstChild("ElianHubMenu") then parentGui.ElianHubMenu:Destroy() end
        if parentGui:FindFirstChild("ElianHubRestoreBtn") then parentGui.ElianHubRestoreBtn:Destroy() end
        if ScreenGui then ScreenGui:Destroy() end
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

-- AIMBOT MEJORADO Y OPTIMIZADO
local function updateClosestHead()
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
                    -- Validar distancia en el mundo 3D primero para respetar el aimDistance
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
-- INTERFAZ GRÁFICA PRINCIPAL Y PANTALLA DE CARGA DE 6 SEGUNDOS
-- =========================================================================
local function iniciarMenuPrincipal()
    destruirMenus()

    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "ElianHubMenu"
    ScreenGui.Parent = parentGui
    ScreenGui.ResetOnSpawn = false
    ScreenGui.IgnoreGuiInset = true

    -- PANTALLA DE CARGA ROBUSTA (6 SEGUNDOS EXACTOS)
    local LoadGui = Instance.new("ScreenGui")
    LoadGui.Name = "ElianHubLoading"
    LoadGui.Parent = parentGui
    LoadGui.ResetOnSpawn = false
    LoadGui.IgnoreGuiInset = true

    local LoadFrame = Instance.new("Frame")
    LoadFrame.Parent = LoadGui
    LoadFrame.BackgroundColor3 = Color3.fromRGB(11, 13, 19)
    LoadFrame.Position = UDim2.new(0.5, -170, 0.5, -65)
    LoadFrame.Size = UDim2.new(0, 340, 0, 130)
    LoadFrame.BorderSizePixel = 0

    local LoadCorner = Instance.new("UICorner")
    LoadCorner.CornerRadius = UDim.new(0, 16)
    LoadCorner.Parent = LoadFrame

    local LoadStroke = Instance.new("UIStroke")
    LoadStroke.Thickness = 2
    LoadStroke.Color = Color3.fromRGB(130, 90, 255)
    LoadStroke.Parent = LoadFrame

    local LoadTitle = Instance.new("TextLabel")
    LoadTitle.Parent = LoadFrame
    LoadTitle.BackgroundTransparency = 1
    LoadTitle.Position = UDim2.new(0, 0, 0, 20)
    LoadTitle.Size = UDim2.new(1, 0, 0, 25)
    LoadTitle.Font = Enum.Font.GothamBold
    LoadTitle.Text = "💎 ELIAN HUB"
    LoadTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
    LoadTitle.TextSize = 16

    local LoadSub = Instance.new("TextLabel")
    LoadSub.Parent = LoadFrame
    LoadSub.BackgroundTransparency = 1
    LoadSub.Position = UDim2.new(0, 0, 0, 45)
    LoadSub.Size = UDim2.new(1, 0, 0, 20)
    LoadSub.Font = Enum.Font.GothamMedium
    LoadSub.Text = "Cargando componentes..."
    LoadSub.TextColor3 = Color3.fromRGB(160, 170, 200)
    LoadSub.TextSize = 11

    local BarBack = Instance.new("Frame")
    BarBack.Parent = LoadFrame
    BarBack.BackgroundColor3 = Color3.fromRGB(20, 25, 35)
    BarBack.Position = UDim2.new(0.1, 0, 0, 78)
    BarBack.Size = UDim2.new(0.8, 0, 0, 8)
    BarBack.BorderSizePixel = 0

    local BarCorner = Instance.new("UICorner")
    BarCorner.CornerRadius = UDim.new(0, 4)
    BarCorner.Parent = BarBack

    local BarFill = Instance.new("Frame")
    BarFill.Parent = BarBack
    BarFill.BackgroundColor3 = Color3.fromRGB(0, 229, 255)
    BarFill.Size = UDim2.new(0, 0, 1, 0)
    BarFill.BorderSizePixel = 0

    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(0, 4)
    FillCorner.Parent = BarFill

    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Parent = ScreenGui
    MainFrame.BackgroundColor3 = Color3.fromRGB(11, 13, 19)
    
    if isMobileDevice then
        MainFrame.Position = UDim2.new(0.5, -240, 0.5, -185)
        MainFrame.Size = UDim2.new(0, 480, 0, 370)
    else
        MainFrame.Position = UDim2.new(0.5, -300, 0.5, -230)
        MainFrame.Size = UDim2.new(0, 600, 0, 460)
    end
    
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true
    MainFrame.Visible = false

    local MainCorner = Instance.new("UICorner")
    MainCorner.CornerRadius = UDim.new(0, 16)
    MainCorner.Parent = MainFrame

    local MainGradient = Instance.new("UIGradient")
    MainGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(16, 19, 28)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(8, 10, 15))
    })
    MainGradient.Rotation = 45
    MainGradient.Parent = MainFrame

    local MainStroke = Instance.new("UIStroke")
    MainStroke.Thickness = 1.8
    MainStroke.Color = Color3.fromRGB(130, 90, 255)
    MainStroke.Parent = MainFrame

    task.spawn(function()
        local cargaCompleta = false

        task.delay(7.0, function()
            if not cargaCompleta and LoadGui and LoadGui.Parent then
                cargaCompleta = true
                pcall(function() LoadGui:Destroy() end)
                if MainFrame then MainFrame.Visible = true end
            end
        end)

        pcall(function()
            TweenService:Create(BarFill, TweenInfo.new(5.7, Enum.EasingStyle.Linear, Enum.EasingDirection.Out), {Size = UDim2.new(1, 0, 1, 0)}):Play()
            task.wait(5.7)
            
            LoadSub.Text = "¡Listo!"
            task.wait(0.3)
            
            TweenService:Create(LoadFrame, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.new(0, 0, 0, 0), BackgroundTransparency = 1}):Play()
            task.wait(0.3)
        end)

        if not cargaCompleta then
            cargaCompleta = true
            pcall(function() LoadGui:Destroy() end)
            if MainFrame then MainFrame.Visible = true end
        end
    end)

    local TopBar = Instance.new("Frame")
    TopBar.Parent = MainFrame
    TopBar.BackgroundTransparency = 1
    TopBar.Size = UDim2.new(1, 0, 0, 46)

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
    LogoBadge.BackgroundColor3 = Color3.fromRGB(130, 90, 255)
    LogoBadge.Size = UDim2.new(0, 26, 0, 26)
    
    local LogoCorner = Instance.new("UICorner")
    LogoCorner.CornerRadius = UDim.new(0, 8)
    LogoCorner.Parent = LogoBadge

    local LogoText = Instance.new("TextLabel")
    LogoText.Parent = LogoBadge
    LogoText.BackgroundTransparency = 1
    LogoText.Size = UDim2.new(1, 0, 1, 0)
    LogoText.Font = Enum.Font.GothamBold
    LogoText.Text = "💎"
    LogoText.TextColor3 = Color3.fromRGB(255, 255, 255)
    LogoText.TextSize = 13

    local Title = Instance.new("TextLabel")
    Title.Parent = TitleContainer
    Title.BackgroundTransparency = 1
    Title.Size = UDim2.new(0, 180, 1, 0)
    Title.Font = Enum.Font.GothamBold
    Title.Text = "ELIAN HUB"
    Title.RichText = true
    Title.TextColor3 = Color3.fromRGB(255, 255, 255)
    Title.TextSize = 13
    Title.TextXAlignment = Enum.TextXAlignment.Left

    local TopActions = Instance.new("Frame")
    TopActions.Parent = TopBar
    TopActions.BackgroundTransparency = 1
    TopActions.Position = UDim2.new(0.45, 0, 0.15, 0)
    TopActions.Size = UDim2.new(0.51, 0, 0, 32)

    local UIListLayout = Instance.new("UIListLayout")
    UIListLayout.Parent = TopActions
    UIListLayout.FillDirection = Enum.FillDirection.Horizontal
    UIListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Right
    UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    UIListLayout.Padding = UDim.new(0, 6)

    local DiscordTopBtn = Instance.new("TextButton")
    DiscordTopBtn.Parent = TopActions
    DiscordTopBtn.BackgroundColor3 = Color3.fromRGB(88, 101, 242)
    DiscordTopBtn.Size = UDim2.new(0, isMobileDevice and 95 or 105, 0, 30)
    DiscordTopBtn.Font = Enum.Font.GothamBold
    DiscordTopBtn.Text = "💬 Discord"
    DiscordTopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    DiscordTopBtn.TextSize = 11
    DiscordTopBtn.BorderSizePixel = 0
    DiscordTopBtn.LayoutOrder = 1

    local DTopCorner = Instance.new("UICorner")
    DTopCorner.CornerRadius = UDim.new(0, 8)
    DTopCorner.Parent = DiscordTopBtn

    local MinimizeBtn = Instance.new("TextButton")
    MinimizeBtn.Parent = TopActions
    MinimizeBtn.BackgroundColor3 = Color3.fromRGB(18, 22, 32)
    MinimizeBtn.Size = UDim2.new(0, 30, 0, 30)
    MinimizeBtn.Font = Enum.Font.GothamBold
    MinimizeBtn.Text = "—"
    MinimizeBtn.TextColor3 = Color3.fromRGB(180, 190, 220)
    MinimizeBtn.TextSize = 12
    MinimizeBtn.BorderSizePixel = 0
    MinimizeBtn.LayoutOrder = 2

    local MinCorner = Instance.new("UICorner")
    MinCorner.CornerRadius = UDim.new(0, 8)
    MinCorner.Parent = MinimizeBtn

    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Parent = TopActions
    CloseBtn.BackgroundColor3 = Color3.fromRGB(235, 60, 90)
    CloseBtn.Size = UDim2.new(0, 30, 0, 30)
    CloseBtn.Font = Enum.Font.GothamBold
    CloseBtn.Text = "X"
    CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    CloseBtn.TextSize = 12
    CloseBtn.BorderSizePixel = 0
    CloseBtn.LayoutOrder = 3

    local CloseCorner = Instance.new("UICorner")
    CloseCorner.CornerRadius = UDim.new(0, 8)
    CloseCorner.Parent = CloseBtn

    local function copiarDiscordLink()
        local link = "https://discord.gg/ewc2Uh4Ff"
        pcall(function()
            if setclipboard then setclipboard(link) end
        end)
        DiscordTopBtn.Text = "✓ ¡Copiado!"
        task.wait(1.5)
        DiscordTopBtn.Text = "💬 Discord"
    end

    DiscordTopBtn.MouseButton1Click:Connect(copiarDiscordLink)
    DiscordTopBtn.TouchTap:Connect(copiarDiscordLink)

    local function crearBotonRestaurar()
        if parentGui:FindFirstChild("ElianHubRestoreBtn") then return end
        local RestoreGui = Instance.new("ScreenGui")
        RestoreGui.Name = "ElianHubRestoreBtn"
        RestoreGui.Parent = parentGui
        RestoreGui.ResetOnSpawn = false

        local OpenBtn = Instance.new("TextButton")
        OpenBtn.Parent = RestoreGui
        OpenBtn.BackgroundColor3 = Color3.fromRGB(14, 17, 24)
        OpenBtn.Position = UDim2.new(0.02, 0, 0.35, 0)
        OpenBtn.Size = UDim2.new(0, 130, 0, 38)
        OpenBtn.Font = Enum.Font.GothamBold
        OpenBtn.Text = "💎 ELIAN HUB"
        OpenBtn.TextColor3 = Color3.fromRGB(0, 229, 255)
        OpenBtn.TextSize = 12
        OpenBtn.Active = true
        OpenBtn.Draggable = true

        local OCorner = Instance.new("UICorner")
        OCorner.CornerRadius = UDim.new(0, 10)
        OCorner.Parent = OpenBtn

        local OStroke = Instance.new("UIStroke")
        OStroke.Color = Color3.fromRGB(130, 90, 255)
        OStroke.Thickness = 1.5
        OStroke.Parent = OpenBtn

        local function restaurar()
            MainFrame.Visible = true
            RestoreGui:Destroy()
        end

        OpenBtn.MouseButton1Click:Connect(restaurar)
        OpenBtn.TouchTap:Connect(restaurar)
    end

    MinimizeBtn.MouseButton1Click:Connect(function()
        MainFrame.Visible = false
        crearBotonRestaurar()
    end)
    MinimizeBtn.TouchTap:Connect(function()
        MainFrame.Visible = false
        crearBotonRestaurar()
    end)

    CloseBtn.MouseButton1Click:Connect(destruirMenus)
    CloseBtn.TouchTap:Connect(destruirMenus)

    local ContainerBody = Instance.new("Frame")
    ContainerBody.Parent = MainFrame
    ContainerBody.BackgroundTransparency = 1
    ContainerBody.Position = UDim2.new(0, 0, 0, 46)
    ContainerBody.Size = UDim2.new(1, 0, 1, -46)

    local TabContainer = Instance.new("Frame")
    TabContainer.Parent = ContainerBody
    TabContainer.BackgroundColor3 = Color3.fromRGB(8, 10, 15)
    TabContainer.Position = UDim2.new(0.03, 0, 0.03, 0)
    TabContainer.Size = UDim2.new(0, isMobileDevice and 130 or 150, 0, isMobileDevice and 305 or 390)
    TabContainer.BorderSizePixel = 0

    local TabCorner = Instance.new("UICorner")
    TabCorner.CornerRadius = UDim.new(0, 12)
    TabCorner.Parent = TabContainer

    local TabListLayout = Instance.new("UIListLayout")
    TabListLayout.Parent = TabContainer
    TabListLayout.FillDirection = Enum.FillDirection.Vertical
    TabListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    TabListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    TabListLayout.Padding = UDim.new(0, 6)

    local TabPadding = Instance.new("UIPadding")
    TabPadding.Parent = TabContainer
    TabPadding.PaddingTop = UDim.new(0, 10)

    local function createTabButton(name, text, order)
        local Btn = Instance.new("TextButton")
        Btn.Parent = TabContainer
        Btn.BackgroundColor3 = order == 1 and Color3.fromRGB(130, 90, 255) or Color3.fromRGB(14, 18, 27)
        Btn.Size = UDim2.new(0.88, 0, 0, isMobileDevice and 36 or 40)
        Btn.Font = Enum.Font.GothamBold
        Btn.Text = text
        Btn.TextColor3 = order == 1 and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(130, 140, 175)
        Btn.TextSize = 12
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
    PagesContainer.Position = UDim2.new(0.31, 0, 0.03, 0)
    PagesContainer.Size = UDim2.new(0, isMobileDevice and 320 or 390, 0, isMobileDevice and 305 or 390)
    PagesContainer.BackgroundTransparency = 1

    local function createPage()
        local page = Instance.new("ScrollingFrame")
        page.Parent = PagesContainer
        page.BackgroundTransparency = 1
        page.Size = UDim2.new(1, 0, 1, 0)
        page.CanvasSize = UDim2.new(0, 0, 1.6, 0)
        page.ScrollBarThickness = 2
        page.ScrollBarImageColor3 = Color3.fromRGB(130, 90, 255)
        page.Visible = false
        page.Active = true
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
            TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(14, 18, 27), TextColor3 = Color3.fromRGB(130, 140, 175)}):Play()
        end
        TweenService:Create(activeBtn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(130, 90, 255), TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
    end

    CombatTabBtn.MouseButton1Click:Connect(function() switchTab(CombatPage, CombatTabBtn) end)
    CombatTabBtn.TouchTap:Connect(function() switchTab(CombatPage, CombatTabBtn) end)

    ESPTabBtn.MouseButton1Click:Connect(function() switchTab(ESPPage, ESPTabBtn) end)
    ESPTabBtn.TouchTap:Connect(function() switchTab(ESPPage, ESPTabBtn) end)

    PlayersTabBtn.MouseButton1Click:Connect(function() switchTab(PlayersPage, PlayersTabBtn) end)
    PlayersTabBtn.TouchTap:Connect(function() switchTab(PlayersPage, PlayersTabBtn) end)

    EtcTabBtn.MouseButton1Click:Connect(function() switchTab(EtcPage, EtcTabBtn) end)
    EtcTabBtn.TouchTap:Connect(function() switchTab(EtcPage, EtcTabBtn) end)

    task.spawn(function()
        while ScreenGui and ScreenGui.Parent do
            pcall(function()
                local hue = (tick() * 0.15) % 1
                local colorRgb = Color3.fromHSV(hue, 0.5, 1)
                MainStroke.Color = colorRgb
            end)
            task.wait(0.1)
        end
    end)

    local function createToggle(parent, name, posY, initialVal, callback)
        local ToggleBtn = Instance.new("TextButton")
        ToggleBtn.Parent = parent
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(15, 19, 28)
        ToggleBtn.Position = UDim2.new(0.02, 0, 0, posY)
        ToggleBtn.Size = UDim2.new(0.96, 0, 0, 40)
        ToggleBtn.Font = Enum.Font.GothamMedium
        ToggleBtn.Text = "     " .. name
        ToggleBtn.TextColor3 = Color3.fromRGB(210, 215, 235)
        ToggleBtn.TextSize = 12
        ToggleBtn.TextXAlignment = Enum.TextXAlignment.Left
        ToggleBtn.BorderSizePixel = 0

        local TCorner = Instance.new("UICorner")
        TCorner.CornerRadius = UDim.new(0, 10)
        TCorner.Parent = ToggleBtn

        local StatusIndicator = Instance.new("Frame")
        StatusIndicator.Parent = ToggleBtn
        StatusIndicator.BackgroundColor3 = initialVal and Color3.fromRGB(0, 229, 255) or Color3.fromRGB(28, 35, 50)
        StatusIndicator.Position = UDim2.new(0.79, 0, 0.25, 0)
        StatusIndicator.Size = UDim2.new(0, 38, 0, 20)

        local SCorner = Instance.new("UICorner")
        SCorner.CornerRadius = UDim.new(0, 10)
        SCorner.Parent = StatusIndicator

        local Dot = Instance.new("Frame")
        Dot.Parent = StatusIndicator
        Dot.BackgroundColor3 = initialVal and Color3.fromRGB(11, 13, 19) or Color3.fromRGB(140, 150, 180)
        Dot.Position = initialVal and UDim2.new(0.46, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0)
        Dot.Size = UDim2.new(0, 16, 0, 16)

        local DCorner = Instance.new("UICorner")
        DCorner.CornerRadius = UDim.new(0, 8)
        DCorner.Parent = Dot

        local enabled = initialVal
        local function triggerToggle()
            enabled = not enabled
            if enabled then
                TweenService:Create(StatusIndicator, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(0, 229, 255)}):Play()
                TweenService:Create(Dot, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(11, 13, 19)}):Play()
                Dot:TweenPosition(UDim2.new(0.46, 0, 0.1, 0), "Out", "Quad", 0.2, true)
            else
                TweenService:Create(StatusIndicator, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(28, 35, 50)}):Play()
                TweenService:Create(Dot, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(140, 150, 180)}):Play()
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
        SliderFrame.BackgroundColor3 = Color3.fromRGB(15, 19, 28)
        SliderFrame.Position = UDim2.new(0.02, 0, 0, posY)
        SliderFrame.Size = UDim2.new(0.96, 0, 0, 52)
        SliderFrame.BorderSizePixel = 0

        local SFrameCorner = Instance.new("UICorner")
        SFrameCorner.CornerRadius = UDim.new(0, 10)
        SFrameCorner.Parent = SliderFrame

        local Label = Instance.new("TextLabel")
        Label.Parent = SliderFrame
        Label.BackgroundTransparency = 1
        Label.Position = UDim2.new(0.05, 0, 0.12, 0)
        Label.Text = name
        Label.TextColor3 = Color3.fromRGB(200, 210, 230)
        Label.TextSize = 12
        Label.TextXAlignment = Enum.TextXAlignment.Left
        Label.Font = Enum.Font.GothamMedium

        local ValueLabel = Instance.new("TextLabel")
        ValueLabel.Parent = SliderFrame
        ValueLabel.BackgroundTransparency = 1
        ValueLabel.Position = UDim2.new(0.05, 0, 0.12, 0)
        ValueLabel.Size = UDim2.new(0.9, 0, 0, 20)
        ValueLabel.Text = tostring(default)
        ValueLabel.TextColor3 = Color3.fromRGB(0, 229, 255)
        ValueLabel.TextSize = 12
        ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
        ValueLabel.Font = Enum.Font.GothamBold

        local Bar = Instance.new("Frame")
        Bar.Parent = SliderFrame
        Bar.BackgroundColor3 = Color3.fromRGB(24, 30, 44)
        Bar.Position = UDim2.new(0.05, 0, 0.65, 0)
        Bar.Size = UDim2.new(0.9, 0, 0, 6)

        local BarCorner = Instance.new("UICorner")
        BarCorner.CornerRadius = UDim.new(0, 3)
        BarCorner.Parent = Bar

        local Fill = Instance.new("Frame")
        Fill.Parent = Bar
        Fill.BackgroundColor3 = Color3.fromRGB(130, 90, 255)
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

    createToggle(CombatPage, "Hitbox Cabeza Enemigo", 6, headHitboxEnabled, function(state)
        headHitboxEnabled = state
        if not state then restoreHeads() end
    end)
    
    createSlider(CombatPage, "Tamaño Cabeza (Max 45)", 2, 45, headHitboxSize, 56, function(val) 
        headHitboxSize = val 
    end)

    if not isMobileDevice then
        createToggle(CombatPage, "Aimbot Directo (Click Dcho)", 116, aimbotEnabled, function(state) aimbotEnabled = state end)
    else
        createToggle(CombatPage, "Aimbot Automático Móvil", 116, aimbotEnabled, function(state) aimbotEnabled = state end)
    end

    createToggle(ESPPage, "ESP Cuerpo Azul", 6, espBodyEnabled, function(state) espBodyEnabled = state end)
    createToggle(ESPPage, "ESP Name", 56, espNameEnabled, function(state) espNameEnabled = state end)
    createSlider(ESPPage, "ESP Distancia", 108, 5000, espDistance, 108, function(val) espDistance = val end)

    createToggle(EtcPage, "Fly (Volar)", 6, flyEnabled, function(state) flyEnabled = state end)
    createSlider(EtcPage, "Velocidad Fly", 10, 300, flySpeed, 56, function(val) flySpeed = val end)

    createToggle(EtcPage, "Activar Speed (Velocidad)", 118, speedEnabled, function(state) 
        speedEnabled = state 
        if not state then
            pcall(function()
                local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
                if hum then hum.WalkSpeed = 16 end
            end)
        end
    end)
    createSlider(EtcPage, "WalkSpeed Valor", 16, 200, walkSpeedValue, 172, function(val) walkSpeedValue = val end)

    createToggle(EtcPage, "Noclip (Atravesar paredes)", 234, noclipEnabled, function(state)
        noclipEnabled = state
        if not state and player.Character then
            pcall(function()
                for _, part in ipairs(player.Character:GetDescendants()) do
                    if part:IsA("BasePart") then
                        part.CanCollide = true
                    end
                end
            end)
        end
    end)

    local playerItems = {}
    
    task.spawn(function()
        while ScreenGui and ScreenGui.Parent do
            pcall(function()
                if PlayersPage.Visible then
                    local currentY = 6
                    local activePlayers = Players:GetPlayers()
                    
                    for _, item in pairs(playerItems) do
                        pcall(function() item:Destroy() end)
                    end
                    playerItems = {}

                    for _, p in ipairs(activePlayers) do
                        if p ~= player then
                            local card = Instance.new("Frame")
                            card.Parent = PlayersPage
                            card.BackgroundColor3 = Color3.fromRGB(15, 19, 28)
                            card.Position = UDim2.new(0.02, 0, 0, currentY)
                            card.Size = UDim2.new(0.96, 0, 0, 40)
                            card.BorderSizePixel = 0
                            
                            local cCorner = Instance.new("UICorner")
                            cCorner.CornerRadius = UDim.new(0, 10)
                            cCorner.Parent = card

                            local nameLbl = Instance.new("TextLabel")
                            nameLbl.Parent = card
                            nameLbl.BackgroundTransparency = 1
                            nameLbl.Position = UDim2.new(0.04, 0, 0, 0)
                            nameLbl.Size = UDim2.new(0.55, 0, 1, 0)
                            nameLbl.Font = Enum.Font.GothamMedium
                            nameLbl.TextColor3 = Color3.fromRGB(230, 235, 255)
                            nameLbl.TextSize = 12
                            nameLbl.TextXAlignment = Enum.TextXAlignment.Left
                            nameLbl.Text = p.Name

                            local statusLbl = Instance.new("TextLabel")
                            statusLbl.Parent = card
                            statusLbl.BackgroundTransparency = 1
                            statusLbl.Position = UDim2.new(0.58, 0, 0, 0)
                            statusLbl.Size = UDim2.new(0.38, 0, 1, 0)
                            statusLbl.Font = Enum.Font.GothamBold
                            statusLbl.TextColor3 = Color3.fromRGB(0, 229, 255)
                            statusLbl.TextSize = 11
                            statusLbl.TextXAlignment = Enum.TextXAlignment.Right

                            local distTxt = "Fuera de rango"
                            pcall(function()
                                if p.Character and p.Character:FindFirstChild("HumanoidRootPart") and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                                    local dist = math.floor((p.Character.HumanoidRootPart.Position - player.Character.HumanoidRootPart.Position).Magnitude)
                                    local hp = math.floor(p.Character.Humanoid.Health)
                                    distTxt = hp .. " HP | " .. dist .. "m"
                                end
                            end)
                            statusLbl.Text = distTxt

                            table.insert(playerItems, card)
                            currentY = currentY + 46
                        end
                    end
                    PlayersPage.CanvasSize = UDim2.new(0, 0, 0, currentY + 20)
                end
            end)
            task.wait(1)
        end
    end)
end

iniciarMenuPrincipal()

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
                if part:IsA("BasePart") then
                    part.CanCollide = false
                end
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

table.insert(connections, RunService.RenderStepped:Connect(function()
    updateClosestHead()

    for p, hl in pairs(espBodies) do
        if not p or not p.Parent or not p.Character or not p.Character.Parent then
            pcall(function() hl:Destroy() end)
            espBodies[p] = nil
        end
    end
    for p, gui in pairs(espNames) do
        if not p or not p.Parent or not p.Character or not p.Character.Parent then
            pcall(function() gui:Destroy() end)
            espNames[p] = nil
        end
    end

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            pcall(function()
                local head = getRealHead(otherPlayer.Character)
                local hrp = otherPlayer.Character:FindFirstChild("HumanoidRootPart")
                
                if head and hrp then
                    if headHitboxEnabled then
                        if not originalHeadData[otherPlayer] then
                            originalHeadData[otherPlayer] = {
                                Head = head,
                                Size = head.Size,
                                Transparency = head.Transparency,
                                Color = head.Color,
                                Material = head.Material,
                                CanCollide = head.CanCollide,
                                Massless = head.Massless
                            }
                        end
                        head.Size = Vector3.new(headHitboxSize, headHitboxSize, headHitboxSize)
                        head.Transparency = 0.5
                        head.Color = Color3.fromRGB(130, 90, 255)
                        head.Material = Enum.Material.Neon
                        head.CanCollide = false
                        head.Massless = true

                        local bp = head:FindFirstChild("ElianHitboxBP")
                        if not bp then
                            bp = Instance.new("BodyPosition")
                            bp.Name = "ElianHitboxBP"
                            bp.MaxForce = Vector3.new(20000, 20000, 20000)
                            bp.P = 8000
                            bp.Parent = head
                        end

                        local targetPos = hrp.Position + Vector3.new(0, 2, 0)
                        local raycastParams = RaycastParams.new()
                        raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
                        raycastParams.FilterDescendantsInstances = {otherPlayer.Character, player.Character}
                        raycastParams.IgnoreWater = true

                        local direction = targetPos - hrp.Position
                        local raycastResult = workspace:Raycast(hrp.Position, direction, raycastParams)

                        if raycastResult then
                            bp.Position = raycastResult.Position - (direction.Unit * 0.5)
                        else
                            bp.Position = targetPos
                        end

                    elseif originalHeadData[otherPlayer] then
                        local data = originalHeadData[otherPlayer]
                        head.Size = data.Size
                        head.Transparency = data.Transparency
                        head.Color = data.Color
                        head.Material = data.Material
                        head.CanCollide = data.CanCollide
                        head.Massless = data.Massless
                        
                        local bp = head:FindFirstChild("ElianHitboxBP")
                        if bp then bp:Destroy() end
                        
                        originalHeadData[otherPlayer] = nil
                    end
                end
            end)
        elseif originalHeadData[otherPlayer] then
            pcall(function()
                local data = originalHeadData[otherPlayer]
                if data and data.Head and data.Head.Parent then
                    data.Head.Size = data.Size
                    data.Head.Transparency = data.Transparency
                    data.Head.Color = data.Color
                    data.Head.Material = data.Material
                    data.Head.CanCollide = data.CanCollide
                end
            end)
            originalHeadData[otherPlayer] = nil
        end
    end

    if aimbotEnabled and currentTargetHead then
        if not isMobileDevice then
            if isRightClicking then
                pcall(function()
                    camera.CFrame = CFrame.new(camera.CFrame.Position, currentTargetHead.Position)
                end)
            end
        else
            pcall(function()
                camera.CFrame = CFrame.new(camera.CFrame.Position, currentTargetHead.Position)
            end)
        end
    end

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
                                hl.FillColor = Color3.fromRGB(0, 180, 255)
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
                            if espBodies[otherPlayer] then
                                espBodies[otherPlayer]:Destroy()
                                espBodies[otherPlayer] = nil
                            end
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
                                txt.TextColor3 = Color3.fromRGB(0, 229, 255)
                                txt.Font = Enum.Font.GothamBold
                                txt.TextSize = isMobileDevice and 11 or 12
                                txt.TextStrokeTransparency = 0.4
                                espNames[otherPlayer] = bGui
                            else
                                bGui.Enabled = true
                                local txt = bGui:FindFirstChild("ESPText") or bGui:FindFirstChild("EspText")
                                if txt then
                                    txt.Text = otherPlayer.Name .. " [" .. math.floor(dist) .. "m]"
                                end
                            end
                        else
                            if espNames[otherPlayer] then
                                espNames[otherPlayer]:Destroy()
                                espNames[otherPlayer] = nil
                            end
                        end
                    else
                        if espBodies[otherPlayer] then espBodies[otherPlayer]:Destroy(); espBodies[otherPlayer] = nil end
                        if espNames[otherPlayer] then espNames[otherPlayer]:Destroy(); espNames[otherPlayer] = nil end
                    end
                else
                    if espBodies[otherPlayer] then espBodies[otherPlayer]:Destroy(); espBodies[otherPlayer] = nil end
                    if espNames[otherPlayer] then espNames[otherPlayer]:Destroy(); espNames[otherPlayer] = nil end
                end
            end)
        else
            if espBodies[otherPlayer] then pcall(function() espBodies[otherPlayer]:Destroy() end); espBodies[otherPlayer] = nil end
            if espNames[otherPlayer] then pcall(function() espNames[otherPlayer]:Destroy() end); espNames[otherPlayer] = nil end
        end
    end
end))

table.insert(connections, player.CharacterAdded:Connect(function(newChar)
    task.wait(1)
    cleanESPVisuals()
end))
