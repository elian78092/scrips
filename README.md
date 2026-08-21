-- =========================================================================
-- ELIAN HUB - ULTRA LUXURY EDITION V6 (ERROR FIXED & MAX BEAUTY)
-- =========================================================================

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local TeleportService = game:GetService("TeleportService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local parentGui = pcall(function() return CoreGui.Name end) and CoreGui or player:WaitForChild("PlayerGui")
local camera = workspace.CurrentCamera

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
        NotifFrame.BackgroundColor3 = Color3.fromRGB(12, 15, 25)
        NotifFrame.Position = UDim2.new(1, 20, 0.82, 0)
        NotifFrame.Size = UDim2.new(0, 330, 0, 76)
        NotifFrame.BorderSizePixel = 0

        local NotifCorner = Instance.new("UICorner")
        NotifCorner.CornerRadius = UDim.new(0, 16)
        NotifCorner.Parent = NotifFrame

        local NotifStroke = Instance.new("UIStroke")
        NotifStroke.Thickness = 1.5
        NotifStroke.Color = Color3.fromRGB(255, 75, 120)
        NotifStroke.Parent = NotifFrame

        local NotifGrad = Instance.new("UIGradient")
        NotifGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(28, 33, 52)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(10, 12, 18))
        })
        NotifGrad.Rotation = 90
        NotifGrad.Parent = NotifFrame

        local TitleLbl = Instance.new("TextLabel")
        TitleLbl.Parent = NotifFrame
        TitleLbl.BackgroundTransparency = 1
        TitleLbl.Position = UDim2.new(0, 18, 0, 12)
        TitleLbl.Size = UDim2.new(1, -20, 0, 20)
        TitleLbl.Font = Enum.Font.FredokaOne
        TitleLbl.Text = titulo
        TitleLbl.TextColor3 = Color3.fromRGB(255, 100, 140)
        TitleLbl.TextSize = 14
        TitleLbl.TextXAlignment = Enum.TextXAlignment.Left

        local MsgLbl = Instance.new("TextLabel")
        MsgLbl.Parent = NotifFrame
        MsgLbl.BackgroundTransparency = 1
        MsgLbl.Position = UDim2.new(0, 18, 0, 36)
        MsgLbl.Size = UDim2.new(1, -20, 0, 26)
        MsgLbl.Font = Enum.Font.GothamMedium
        MsgLbl.Text = mensaje
        MsgLbl.TextColor3 = Color3.fromRGB(210, 220, 240)
        MsgLbl.TextSize = 12
        MsgLbl.TextXAlignment = Enum.TextXAlignment.Left

        NotifFrame:TweenPosition(UDim2.new(1, -350, 0.82, 0), "Out", "Back", 0.4, true)

        task.delay(duracion or 4, function()
            pcall(function()
                NotifFrame:TweenPosition(UDim2.new(1, 20, 0.82, 0), "In", "Quart", 0.4, true)
                task.wait(0.4)
                NotifGui:Destroy()
            end)
        end)
    end)
end

if not game:IsLoaded() then game.Loaded:Wait() end

local hitboxMode = "Head"
local hitboxEnabled = false      
local hitboxSize = 10 

local aimbotEnabled = false
local aimDistance = 800
local isShootingOrClicking = false
local currentTargetPart = nil

local espGlobalBody = false
local espGlobalName = false
local espGlobalTracer = false
local espDistance = 2000

local flyEnabled = false
local flySpeed = 50
local speedEnabled = false
local walkSpeedValue = 16
local noclipEnabled = false
local customFovEnabled = false
local fovValue = 70

local blockedPlayersUI = {}
local espNames = {}
local espBodies = {}
local espTracers = {}
local originalHitboxData = {} 
local connections = {}
local bv, bg

local function aplicarBypass()
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
end
aplicarBypass()

local function restoreSingleHitbox(plr)
    pcall(function()
        local data = originalHitboxData[plr]
        if data and data.Part and data.Part.Parent then
            local part = data.Part
            part.Size = data.Size
            part.Transparency = data.Transparency
            part.Color = data.Color
            part.Material = data.Material
            part.CanCollide = data.CanCollide
            part.Massless = data.Massless
            
            local bp = part:FindFirstChild("ElianHitboxBP")
            if bp then bp:Destroy() end
        end
        originalHitboxData[plr] = nil
    end)
end

local function restoreHitboxes()
    for plr, _ in pairs(originalHitboxData) do restoreSingleHitbox(plr) end
    originalHitboxData = {}
end

local function cleanESPVisuals()
    for _, nameGui in pairs(espNames) do pcall(function() if nameGui then nameGui:Destroy() end end) end
    espNames = {}
    for _, highlight in pairs(espBodies) do pcall(function() if highlight then highlight:Destroy() end end) end
    espBodies = {}
    for _, tracer in pairs(espTracers) do pcall(function() if tracer then tracer:Remove() end end) end
    espTracers = {}
end

local function fullCleanup()
    hitboxEnabled = false
    aimbotEnabled = false
    flyEnabled = false
    speedEnabled = false
    noclipEnabled = false
    espGlobalBody = false
    espGlobalName = false
    espGlobalTracer = false
    customFovEnabled = false
    currentTargetPart = nil
    
    restoreHitboxes()
    cleanESPVisuals()

    pcall(function()
        local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum.WalkSpeed = 16 end
        camera.FieldOfView = 70
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

table.insert(connections, UserInputService.InputBegan:Connect(function(input, gpe)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then isShootingOrClicking = true end
end))

table.insert(connections, UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then isShootingOrClicking = false end
end))

local function getTargetPart(character)
    if not character then return nil end
    if hitboxMode == "Head" then
        return character:FindFirstChild("Head") or character:FindFirstChild("FakeHead")
    else
        return character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso") or character:FindFirstChild("HumanoidRootPart")
    end
end

local ScreenGui = nil

local function iniciarMenuPrincipal()
    destruirMenus()

    -- PANTALLA DE CARGA MEJORADA Y FLUIDA (10 Segundos exactos)
    local LoadGui = Instance.new("ScreenGui")
    LoadGui.Name = "ElianHubLoading"
    LoadGui.Parent = parentGui
    LoadGui.ResetOnSpawn = false
    LoadGui.IgnoreGuiInset = true

    local RainContainer = Instance.new("Folder")
    RainContainer.Name = "RainContainer"
    RainContainer.Parent = LoadGui

    task.spawn(function()
        while LoadGui and LoadGui.Parent do
            pcall(function()
                local drop = Instance.new("Frame")
                drop.Parent = RainContainer
                drop.BackgroundColor3 = Color3.fromRGB(100, 200, 255)
                drop.BackgroundTransparency = 0.3
                drop.BorderSizePixel = 0
                drop.Size = UDim2.new(0, 2, 0, math.random(30, 60))
                local startX = math.random(0, workspace.CurrentCamera.ViewportSize.X)
                drop.Position = UDim2.new(0, startX, 0, -80)
                local fallTime = math.random(8, 15) / 10
                drop:TweenPosition(UDim2.new(0, startX - 15, 1, 80), "InOut", "Linear", fallTime, true)
                task.delay(fallTime, function() if drop then drop:Destroy() end end)
            end)
            task.wait(0.02)
        end
    end)

    local LoadFrame = Instance.new("Frame")
    LoadFrame.Parent = LoadGui
    LoadFrame.BackgroundColor3 = Color3.fromRGB(6, 8, 12)
    LoadFrame.Position = UDim2.new(0.5, -230, 0.5, -115)
    LoadFrame.Size = UDim2.new(0, 0, 0, 0)
    LoadFrame.BorderSizePixel = 0
    LoadFrame.ClipsDescendants = true

    local LCorner = Instance.new("UICorner")
    LCorner.CornerRadius = UDim.new(0, 24)
    LCorner.Parent = LoadFrame

    local LGradient = Instance.new("UIGradient")
    LGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(28, 35, 55)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(5, 7, 10))
    })
    LGradient.Rotation = 45
    LGradient.Parent = LoadFrame

    local LStroke = Instance.new("UIStroke")
    LStroke.Thickness = 2.5
    LStroke.Color = Color3.fromRGB(255, 75, 120)
    LStroke.Parent = LoadFrame

    local LoadTitle = Instance.new("TextLabel")
    LoadTitle.Parent = LoadFrame
    LoadTitle.BackgroundTransparency = 1
    LoadTitle.Position = UDim2.new(0, 0, 0, 25)
    LoadTitle.Size = UDim2.new(1, 0, 0, 35)
    LoadTitle.Font = Enum.Font.FredokaOne
    LoadTitle.Text = "⚡ ELIAN HUB ⚡"
    LoadTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
    LoadTitle.TextSize = 24

    local SubTitle = Instance.new("TextLabel")
    SubTitle.Parent = LoadFrame
    SubTitle.BackgroundTransparency = 1
    SubTitle.Position = UDim2.new(0, 0, 0, 65)
    SubTitle.Size = UDim2.new(1, 0, 0, 20)
    SubTitle.Font = Enum.Font.GothamMedium
    SubTitle.Text = "Preparando interfaz de lujo..."
    SubTitle.TextColor3 = Color3.fromRGB(170, 190, 220)
    SubTitle.TextSize = 12

    local BarBackground = Instance.new("Frame")
    BarBackground.Parent = LoadFrame
    BarBackground.BackgroundColor3 = Color3.fromRGB(10, 13, 20)
    BarBackground.Position = UDim2.new(0.08, 0, 0, 105)
    BarBackground.Size = UDim2.new(0.84, 0, 0, 14)
    BarBackground.BorderSizePixel = 0
    BarBackground.ClipsDescendants = true

    local BCorner = Instance.new("UICorner")
    BCorner.CornerRadius = UDim.new(0, 7)
    BCorner.Parent = BarBackground

    local BarFill = Instance.new("Frame")
    BarFill.Parent = BarBackground
    BarFill.BackgroundColor3 = Color3.fromRGB(255, 75, 120)
    BarFill.Size = UDim2.new(0, 0, 1, 0)
    BarFill.BorderSizePixel = 0

    local FCorner = Instance.new("UICorner")
    FCorner.CornerRadius = UDim.new(0, 7)
    FCorner.Parent = BarFill

    local BarGradient = Instance.new("UIGradient")
    BarGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 140, 170)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(235, 40, 90))
    })
    BarGradient.Parent = BarFill

    local PercentLbl = Instance.new("TextLabel")
    PercentLbl.Parent = LoadFrame
    PercentLbl.BackgroundTransparency = 1
    PercentLbl.Position = UDim2.new(0, 0, 0, 135)
    PercentLbl.Size = UDim2.new(1, 0, 0, 25)
    PercentLbl.Font = Enum.Font.FredokaOne
    PercentLbl.Text = "0%"
    PercentLbl.TextColor3 = Color3.fromRGB(245, 250, 255)
    PercentLbl.TextSize = 16

    TweenService:Create(LoadFrame, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, 460, 0, 210)
    }):Play()

    task.spawn(function()
        local tiempoTotal = 10.0
        local pasos = 100
        local esperaPorPaso = tiempoTotal / pasos

        for i = 1, 100 do
            PercentLbl.Text = i .. "%"
            TweenService:Create(BarFill, TweenInfo.new(esperaPorPaso, Enum.EasingStyle.Linear, Enum.EasingDirection.Out), {
                Size = UDim2.new(i / 100, 0, 1, 0)
            }):Play()
            
            if i == 25 then SubTitle.Text = "Optimizando funciones gráficas..." end
            if i == 50 then SubTitle.Text = "Configurando Hitboxes y ESP..." end
            if i == 80 then SubTitle.Text = "¡Carga completada con éxito!" end
            
            task.wait(esperaPorPaso)
        end
        
        task.wait(0.3)
        TweenService:Create(LoadFrame, TweenInfo.new(0.4, Enum.EasingStyle.Sine, Enum.EasingDirection.In), {BackgroundTransparency = 1, Size = UDim2.new(0, 480, 0, 220)}):Play()
        task.wait(0.4)
        LoadGui:Destroy()
        
        mostrarNotificacion("⚡ Elian Hub", "¡Bienvenido! Interfaz cargada perfectamente.", 3)
    end)

    task.wait(10.7)

    -- MENÚ PRINCIPAL CON TAMAÑOS CORREGIDOS Y ESTÉTICA ELEVADA
    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "ElianHubMenu"
    ScreenGui.Parent = parentGui
    ScreenGui.ResetOnSpawn = false
    ScreenGui.IgnoreGuiInset = true

    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Parent = ScreenGui
    MainFrame.BackgroundColor3 = Color3.fromRGB(6, 8, 12)
    
    local widthMenu = isMobileDevice and 460 or 640
    local heightMenu = isMobileDevice and 360 or 470

    MainFrame.Position = UDim2.new(0.5, -widthMenu / 2, 0.5, -heightMenu / 2)
    MainFrame.Size = UDim2.new(0, 0, 0, 0)
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true
    MainFrame.Visible = true

    local MainCorner = Instance.new("UICorner")
    MainCorner.CornerRadius = UDim.new(0, 22)
    MainCorner.Parent = MainFrame

    local MainGradient = Instance.new("UIGradient")
    MainGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(22, 28, 46)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(4, 6, 10))
    })
    MainGradient.Rotation = 45
    MainGradient.Parent = MainFrame

    local MainStroke = Instance.new("UIStroke")
    MainStroke.Thickness = 2
    MainStroke.Color = Color3.fromRGB(255, 75, 120)
    MainStroke.Parent = MainFrame

    TweenService:Create(MainFrame, TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, widthMenu, 0, heightMenu)
    }):Play()

    local TopBar = Instance.new("Frame")
    TopBar.Parent = MainFrame
    TopBar.BackgroundTransparency = 1
    TopBar.Size = UDim2.new(1, 0, 0, 54)

    local TitleContainer = Instance.new("Frame")
    TitleContainer.Parent = TopBar
    TitleContainer.BackgroundTransparency = 1
    TitleContainer.Position = UDim2.new(0.03, 0, 0, 0)
    TitleContainer.Size = UDim2.new(0.4, 0, 1, 0)

    local TitleLayout = Instance.new("UIListLayout")
    TitleLayout.Parent = TitleContainer
    TitleLayout.FillDirection = Enum.FillDirection.Horizontal
    TitleLayout.VerticalAlignment = Enum.VerticalAlignment.Center
    TitleLayout.Padding = UDim.new(0, 10)

    local LogoBadge = Instance.new("Frame")
    LogoBadge.Parent = TitleContainer
    LogoBadge.BackgroundColor3 = Color3.fromRGB(255, 75, 120)
    LogoBadge.Size = UDim2.new(0, 32, 0, 32)
    
    local LogoCorner = Instance.new("UICorner")
    LogoCorner.CornerRadius = UDim.new(0, 10)
    LogoCorner.Parent = LogoBadge

    local LogoText = Instance.new("TextLabel")
    LogoText.Parent = LogoBadge
    LogoText.BackgroundTransparency = 1
    LogoText.Size = UDim2.new(1, 0, 1, 0)
    LogoText.Font = Enum.Font.FredokaOne
    LogoText.Text = "⚡"
    LogoText.TextColor3 = Color3.fromRGB(255, 255, 255)
    LogoText.TextSize = 15

    local Title = Instance.new("TextLabel")
    Title.Parent = TitleContainer
    Title.BackgroundTransparency = 1
    Title.Size = UDim2.new(0, 180, 1, 0)
    Title.Font = Enum.Font.FredokaOne
    Title.Text = "ELIAN HUB"
    Title.TextColor3 = Color3.fromRGB(255, 255, 255)
    Title.TextSize = 17
    Title.TextXAlignment = Enum.TextXAlignment.Left

    local TopActions = Instance.new("Frame")
    TopActions.Parent = TopBar
    TopActions.BackgroundTransparency = 1
    TopActions.Position = UDim2.new(0.45, 0, 0.15, 0)
    TopActions.Size = UDim2.new(0.52, 0, 0, 34)

    local UIListLayout = Instance.new("UIListLayout")
    UIListLayout.Parent = TopActions
    UIListLayout.FillDirection = Enum.FillDirection.Horizontal
    UIListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Right
    UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    UIListLayout.Padding = UDim.new(0, 6)

    local DiscordTopBtn = Instance.new("TextButton")
    DiscordTopBtn.Parent = TopActions
    DiscordTopBtn.BackgroundColor3 = Color3.fromRGB(12, 16, 26)
    DiscordTopBtn.Size = UDim2.new(0, isMobileDevice and 85 or 95, 0, 32)
    DiscordTopBtn.Font = Enum.Font.GothamBold
    DiscordTopBtn.Text = "💬 Discord"
    DiscordTopBtn.TextColor3 = Color3.fromRGB(220, 230, 250)
    DiscordTopBtn.TextSize = 11
    DiscordTopBtn.BorderSizePixel = 0
    DiscordTopBtn.LayoutOrder = 1

    local DTopCorner = Instance.new("UICorner")
    DTopCorner.CornerRadius = UDim.new(0, 10)
    DTopCorner.Parent = DiscordTopBtn

    local MinimizeBtn = Instance.new("TextButton")
    MinimizeBtn.Parent = TopActions
    MinimizeBtn.BackgroundColor3 = Color3.fromRGB(12, 16, 26)
    MinimizeBtn.Size = UDim2.new(0, 32, 0, 32)
    MinimizeBtn.Font = Enum.Font.GothamBold
    MinimizeBtn.Text = "—"
    MinimizeBtn.TextColor3 = Color3.fromRGB(210, 220, 250)
    MinimizeBtn.TextSize = 12
    MinimizeBtn.BorderSizePixel = 0
    MinimizeBtn.LayoutOrder = 2

    local MinCorner = Instance.new("UICorner")
    MinCorner.CornerRadius = UDim.new(0, 10)
    MinCorner.Parent = MinimizeBtn

    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Parent = TopActions
    CloseBtn.BackgroundColor3 = Color3.fromRGB(45, 15, 25)
    CloseBtn.Size = UDim2.new(0, 32, 0, 32)
    CloseBtn.Font = Enum.Font.GothamBold
    CloseBtn.Text = "✕"
    CloseBtn.TextColor3 = Color3.fromRGB(255, 75, 120)
    CloseBtn.TextSize = 13
    CloseBtn.BorderSizePixel = 0
    CloseBtn.LayoutOrder = 3

    local CloseCorner = Instance.new("UICorner")
    CloseCorner.CornerRadius = UDim.new(0, 10)
    CloseCorner.Parent = CloseBtn

    local function copiarDiscordLink()
        pcall(function() if setclipboard then setclipboard("https://discord.gg/ewc2Uh4Ff") end end)
        DiscordTopBtn.Text = "✓ Copiado"
        mostrarNotificacion("Discord", "¡Enlace copiado con éxito!", 2)
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
        OpenBtn.BackgroundColor3 = Color3.fromRGB(10, 13, 20)
        OpenBtn.Position = UDim2.new(0.02, 0, 0.35, 0)
        OpenBtn.Size = UDim2.new(0, 135, 0, 44)
        OpenBtn.Font = Enum.Font.FredokaOne
        OpenBtn.Text = "⚡ ELIAN HUB"
        OpenBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        OpenBtn.TextSize = 12
        OpenBtn.Active = true
        OpenBtn.Draggable = true

        local OCorner = Instance.new("UICorner")
        OCorner.CornerRadius = UDim.new(0, 12)
        OCorner.Parent = OpenBtn

        local OStroke = Instance.new("UIStroke")
        OStroke.Color = Color3.fromRGB(255, 75, 120)
        OStroke.Thickness = 1.8
        OStroke.Parent = OpenBtn

        local function restaurar()
            MainFrame.Visible = true
            TweenService:Create(MainFrame, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                Size = UDim2.new(0, widthMenu, 0, heightMenu)
            }):Play()
            RestoreGui:Destroy()
        end

        OpenBtn.MouseButton1Click:Connect(restaurar)
        OpenBtn.TouchTap:Connect(restaurar)
    end

    MinimizeBtn.MouseButton1Click:Connect(function() 
        TweenService:Create(MainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Sine, Enum.EasingDirection.In), {Size = UDim2.new(0, 0, 0, 0)}):Play()
        task.wait(0.2)
        MainFrame.Visible = false 
        crearBotonRestaurar() 
    end)
    MinimizeBtn.TouchTap:Connect(function() 
        TweenService:Create(MainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Sine, Enum.EasingDirection.In), {Size = UDim2.new(0, 0, 0, 0)}):Play()
        task.wait(0.2)
        MainFrame.Visible = false 
        crearBotonRestaurar() 
    end)
    CloseBtn.MouseButton1Click:Connect(destruirMenus)
    CloseBtn.TouchTap:Connect(destruirMenus)

    local ContainerBody = Instance.new("Frame")
    ContainerBody.Parent = MainFrame
    ContainerBody.BackgroundTransparency = 1
    ContainerBody.Position = UDim2.new(0, 0, 0, 54)
    ContainerBody.Size = UDim2.new(1, 0, 1, -54)

    local tabWidth = isMobileDevice and 110 or 135
    
    local TabContainer = Instance.new("Frame")
    TabContainer.Parent = ContainerBody
    TabContainer.BackgroundColor3 = Color3.fromRGB(4, 6, 9)
    TabContainer.Position = UDim2.new(0.025, 0, 0.025, 0)
    TabContainer.Size = UDim2.new(0, tabWidth, 0, heightMenu - 68)
    TabContainer.BorderSizePixel = 0

    local TabCorner = Instance.new("UICorner")
    TabCorner.CornerRadius = UDim.new(0, 16)
    TabCorner.Parent = TabContainer

    local TabListLayout = Instance.new("UIListLayout")
    TabListLayout.Parent = TabContainer
    TabListLayout.FillDirection = Enum.FillDirection.Vertical
    TabListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    TabListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    TabListLayout.Padding = UDim.new(0, 8)

    local TabPadding = Instance.new("UIPadding")
    TabPadding.Parent = TabContainer
    TabPadding.PaddingTop = UDim.new(0, 12)

    local function createTabButton(name, text, order)
        local Btn = Instance.new("TextButton")
        Btn.Parent = TabContainer
        Btn.BackgroundColor3 = order == 1 and Color3.fromRGB(255, 75, 120) or Color3.fromRGB(10, 14, 22)
        Btn.Size = UDim2.new(0.9, 0, 0, isMobileDevice and 38 or 42)
        Btn.Font = Enum.Font.FredokaOne
        Btn.Text = text
        Btn.TextColor3 = order == 1 and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(140, 155, 185)
        Btn.TextSize = isMobileDevice and 11 or 12
        Btn.BorderSizePixel = 0
        Btn.LayoutOrder = order

        local Corner = Instance.new("UICorner")
        Corner.CornerRadius = UDim.new(0, 12)
        Corner.Parent = Btn
        return Btn
    end

    local CombatTabBtn = createTabButton("Combat", "⚔ Combat", 1)
    local ESPTabBtn = createTabButton("ESP", "🎯 ESP", 2)
    local PlayersTabBtn = createTabButton("Players", "👥 Players", 3)
    local EtcTabBtn = createTabButton("Misc", "⚙ Misc", 4)

    local pagesWidth = widthMenu - tabWidth - 30
    local PagesContainer = Instance.new("Frame")
    PagesContainer.Parent = ContainerBody
    PagesContainer.Position = UDim2.new(0, tabWidth + 18, 0.025, 0)
    PagesContainer.Size = UDim2.new(0, pagesWidth, 0, heightMenu - 68)
    PagesContainer.BackgroundTransparency = 1
    PagesContainer.ClipsDescendants = true

    local function createPage()
        local page = Instance.new("ScrollingFrame")
        page.Parent = PagesContainer
        page.BackgroundTransparency = 1
        page.Size = UDim2.new(1, 0, 1, 0)
        page.CanvasSize = UDim2.new(0, 0, 2.5, 0)
        page.ScrollBarThickness = 3
        page.ScrollBarImageColor3 = Color3.fromRGB(255, 75, 120)
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
        selectedPage.Position = UDim2.new(0.04, 0, 0, 0)
        
        TweenService:Create(selectedPage, TweenInfo.new(0.25, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
            Position = UDim2.new(0, 0, 0, 0)
        }):Play()

        for _, btn in ipairs({CombatTabBtn, ESPTabBtn, PlayersTabBtn, EtcTabBtn}) do
            TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(10, 14, 22), TextColor3 = Color3.fromRGB(140, 155, 185)}):Play()
        end
        TweenService:Create(activeBtn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(255, 75, 120), TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
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
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(10, 14, 22)
        ToggleBtn.Position = UDim2.new(0, 0, 0, posY)
        ToggleBtn.Size = UDim2.new(0.98, 0, 0, 44)
        ToggleBtn.Font = Enum.Font.GothamMedium
        ToggleBtn.Text = "   " .. name
        ToggleBtn.TextColor3 = Color3.fromRGB(235, 242, 255)
        ToggleBtn.TextSize = 12
        ToggleBtn.TextXAlignment = Enum.TextXAlignment.Left
        ToggleBtn.BorderSizePixel = 0

        local TCorner = Instance.new("UICorner")
        TCorner.CornerRadius = UDim.new(0, 12)
        TCorner.Parent = ToggleBtn

        local StatusIndicator = Instance.new("Frame")
        StatusIndicator.Parent = ToggleBtn
        StatusIndicator.BackgroundColor3 = initialVal and Color3.fromRGB(255, 75, 120) or Color3.fromRGB(18, 24, 38)
        StatusIndicator.Position = UDim2.new(0.72, 0, 0.2, 0)
        StatusIndicator.Size = UDim2.new(0, 44, 0, 26)

        local SCorner = Instance.new("UICorner")
        SCorner.CornerRadius = UDim.new(0, 13)
        SCorner.Parent = StatusIndicator

        local Dot = Instance.new("Frame")
        Dot.Parent = StatusIndicator
        Dot.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        Dot.Position = initialVal and UDim2.new(0.48, 0, 0.12, 0) or UDim2.new(0.08, 0, 0.12, 0)
        Dot.Size = UDim2.new(0, 20, 0, 20)

        local DCorner = Instance.new("UICorner")
        DCorner.CornerRadius = UDim.new(0, 10)
        DCorner.Parent = Dot

        local enabled = initialVal
        local function triggerToggle()
            enabled = not enabled
            if enabled then
                TweenService:Create(StatusIndicator, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(255, 75, 120)}):Play()
                Dot:TweenPosition(UDim2.new(0.48, 0, 0.12, 0), "Out", "Quart", 0.2, true)
            else
                TweenService:Create(StatusIndicator, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(18, 24, 38)}):Play()
                Dot:TweenPosition(UDim2.new(0.08, 0, 0.12, 0), "Out", "Quart", 0.2, true)
            end
            callback(enabled)
        end

        ToggleBtn.MouseButton1Click:Connect(triggerToggle)
        ToggleBtn.TouchTap:Connect(triggerToggle)
    end

    local function createSlider(parent, name, min, max, default, posY, callback)
        local SliderFrame = Instance.new("Frame")
        SliderFrame.Parent = parent
        SliderFrame.BackgroundColor3 = Color3.fromRGB(10, 14, 22)
        SliderFrame.Position = UDim2.new(0, 0, 0, posY)
        SliderFrame.Size = UDim2.new(0.98, 0, 0, 56)
        SliderFrame.BorderSizePixel = 0

        local SFrameCorner = Instance.new("UICorner")
        SFrameCorner.CornerRadius = UDim.new(0, 12)
        SFrameCorner.Parent = SliderFrame

        local Label = Instance.new("TextLabel")
        Label.Parent = SliderFrame
        Label.BackgroundTransparency = 1
        Label.Position = UDim2.new(0.04, 0, 0.12, 0)
        Label.Text = name
        Label.TextColor3 = Color3.fromRGB(235, 242, 255)
        Label.TextSize = 12
        Label.TextXAlignment = Enum.TextXAlignment.Left
        Label.Font = Enum.Font.GothamMedium

        local ValueLabel = Instance.new("TextLabel")
        ValueLabel.Parent = SliderFrame
        ValueLabel.BackgroundTransparency = 1
        ValueLabel.Position = UDim2.new(0.04, 0, 0.12, 0)
        ValueLabel.Size = UDim2.new(0.92, 0, 0, 20)
        ValueLabel.Text = tostring(default)
        ValueLabel.TextColor3 = Color3.fromRGB(255, 75, 120)
        ValueLabel.TextSize = 12
        ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
        ValueLabel.Font = Enum.Font.FredokaOne

        local Bar = Instance.new("Frame")
        Bar.Parent = SliderFrame
        Bar.BackgroundColor3 = Color3.fromRGB(18, 24, 36)
        Bar.Position = UDim2.new(0.04, 0, 0.65, 0)
        Bar.Size = UDim2.new(0.92, 0, 0, 8)

        local BarCorner = Instance.new("UICorner")
        BarCorner.CornerRadius = UDim.new(0, 4)
        BarCorner.Parent = Bar

        local Fill = Instance.new("Frame")
        Fill.Parent = Bar
        Fill.BackgroundColor3 = Color3.fromRGB(255, 75, 120)
        Fill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)

        local FillCorner = Instance.new("UICorner")
        FillCorner.CornerRadius = UDim.new(0, 4)
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

    local CombatInfoLbl = Instance.new("TextLabel")
    CombatInfoLbl.Parent = CombatPage
    CombatInfoLbl.BackgroundTransparency = 1
    CombatInfoLbl.Position = UDim2.new(0, 0, 0, 4)
    CombatInfoLbl.Size = UDim2.new(0.98, 0, 0, 24)
    CombatInfoLbl.Font = Enum.Font.FredokaOne
    CombatInfoLbl.Text = "⚔ Selector de Hitbox Global"
    CombatInfoLbl.TextColor3 = Color3.fromRGB(255, 75, 120)
    CombatInfoLbl.TextSize = 12

    local ModeToggleBtn = Instance.new("TextButton")
    ModeToggleBtn.Parent = CombatPage
    ModeToggleBtn.BackgroundColor3 = Color3.fromRGB(10, 14, 22)
    ModeToggleBtn.Position = UDim2.new(0, 0, 0, 34)
    ModeToggleBtn.Size = UDim2.new(0.98, 0, 0, 44)
    ModeToggleBtn.Font = Enum.Font.FredokaOne
    ModeToggleBtn.Text = "Hitbox Parte: [ Head ]"
    ModeToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    ModeToggleBtn.TextSize = 12
    ModeToggleBtn.BorderSizePixel = 0

    local ModeCorner = Instance.new("UICorner")
    ModeCorner.CornerRadius = UDim.new(0, 12)
    ModeCorner.Parent = ModeToggleBtn

    local function cambiarModoHitbox()
        restoreHitboxes()
        if hitboxMode == "Head" then
            hitboxMode = "Torso"
            ModeToggleBtn.Text = "Hitbox Parte: [ Torso ]"
            mostrarNotificacion("Combat", "Hitbox cambiado a Torso", 2)
        else
            hitboxMode = "Head"
            ModeToggleBtn.Text = "Hitbox Parte: [ Head ]"
            mostrarNotificacion("Combat", "Hitbox cambiado a Cabeza", 2)
        end
    end
    ModeToggleBtn.MouseButton1Click:Connect(cambiarModoHitbox)
    ModeToggleBtn.TouchTap:Connect(cambiarModoHitbox)

    createToggle(CombatPage, "Hitbox Global Activo", 86, hitboxEnabled, function(state)
        hitboxEnabled = state
        if not state then restoreHitboxes() end
    end)
    createSlider(CombatPage, "Tamaño Hitbox", 2, 25, hitboxSize, 138, function(val) hitboxSize = val end)

    if not isMobileDevice then
        createToggle(CombatPage, "Aimbot (Click Dcho)", 202, aimbotEnabled, function(state) aimbotEnabled = state end)
    else
        createToggle(CombatPage, "Aimbot (Al Disparar)", 202, aimbotEnabled, function(state) aimbotEnabled = state end)
    end

    createToggle(ESPPage, "ESP Cuerpo Global", 4, espGlobalBody, function(state) espGlobalBody = state end)
    createToggle(ESPPage, "ESP Nombre Global", 52, espGlobalName, function(state) espGlobalName = state end)
    createToggle(ESPPage, "ESP Tracers Global", 100, espGlobalTracer, function(state)
        espGlobalTracer = state
        if not state then
            for _, tracer in pairs(espTracers) do pcall(function() tracer:Remove() end) end
            espTracers = {}
        end
    end)
    createSlider(ESPPage, "ESP Distancia", 148, 5000, espDistance, 148, function(val) espDistance = val end)

    local PlayersListLayoutContainer = Instance.new("ScrollingFrame")
    PlayersListLayoutContainer.Parent = PlayersPage
    PlayersListLayoutContainer.BackgroundTransparency = 1
    PlayersListLayoutContainer.Position = UDim2.new(0, 0, 0, 4)
    PlayersListLayoutContainer.Size = UDim2.new(0.98, 0, 1, -10)
    PlayersListLayoutContainer.CanvasSize = UDim2.new(0, 0, 0, 0)
    PlayersListLayoutContainer.ScrollBarThickness = 3
    PlayersListLayoutContainer.ScrollBarImageColor3 = Color3.fromRGB(255, 75, 120)

    local UIListPLayout = Instance.new("UIListLayout")
    UIListPLayout.Parent = PlayersListLayoutContainer
    UIListPLayout.SortOrder = Enum.SortOrder.LayoutOrder
    UIListPLayout.Padding = UDim.new(0, 6)

    local function actualizarListaJugadores()
        for _, child in ipairs(PlayersListLayoutContainer:GetChildren()) do
            if child:IsA("Frame") then child:Destroy() end
        end

        local count = 0
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= player then
                count = count + 1
                local pRow = Instance.new("Frame")
                pRow.Parent = PlayersListLayoutContainer
                pRow.BackgroundColor3 = Color3.fromRGB(10, 14, 22)
                pRow.Size = UDim2.new(1, 0, 0, 44)
                pRow.BorderSizePixel = 0
                pRow.LayoutOrder = count

                local rCorner = Instance.new("UICorner")
                rCorner.CornerRadius = UDim.new(0, 12)
                rCorner.Parent = pRow

                local nameLbl = Instance.new("TextLabel")
                nameLbl.Parent = pRow
                nameLbl.BackgroundTransparency = 1
                nameLbl.Position = UDim2.new(0.04, 0, 0, 0)
                nameLbl.Size = UDim2.new(0.6, 0, 1, 0)
                nameLbl.Font = Enum.Font.GothamMedium
                nameLbl.Text = p.Name
                nameLbl.TextColor3 = Color3.fromRGB(230, 240, 255)
                nameLbl.TextSize = 12
                nameLbl.TextXAlignment = Enum.TextXAlignment.Left

                local uiToggleBtn = Instance.new("TextButton")
                uiToggleBtn.Parent = pRow
                uiToggleBtn.BackgroundColor3 = Color3.fromRGB(18, 40, 28)
                uiToggleBtn.Position = UDim2.new(0.65, 0, 0.15, 0)
                uiToggleBtn.Size = UDim2.new(0.32, 0, 0, 31)
                uiToggleBtn.Font = Enum.Font.FredokaOne
                uiToggleBtn.Text = "UI: OFF"
                uiToggleBtn.TextColor3 = Color3.fromRGB(80, 255, 140)
                uiToggleBtn.TextSize = 11
                uiToggleBtn.BorderSizePixel = 0

                local btnCorner = Instance.new("UICorner")
                btnCorner.CornerRadius = UDim.new(0, 10)
                btnCorner.Parent = uiToggleBtn

                local function togglePlayerUI()
                    if blockedPlayersUI[p] then
                        blockedPlayersUI[p] = nil
                        uiToggleBtn.Text = "UI: OFF"
                        uiToggleBtn.BackgroundColor3 = Color3.fromRGB(18, 40, 28)
                        uiToggleBtn.TextColor3 = Color3.fromRGB(80, 255, 140)
                        mostrarNotificacion("Players", "Filtro desactivado para " .. p.Name, 2)
                    else
                        blockedPlayersUI[p] = true
                        uiToggleBtn.Text = "UI: ON"
                        uiToggleBtn.BackgroundColor3 = Color3.fromRGB(45, 18, 28)
                        uiToggleBtn.TextColor3 = Color3.fromRGB(255, 75, 120)
                        mostrarNotificacion("Players", "Ignorando a " .. p.Name, 2)
                    end
                end

                uiToggleBtn.MouseButton1Click:Connect(togglePlayerUI)
                uiToggleBtn.TouchTap:Connect(togglePlayerUI)
            end
        end
        PlayersListLayoutContainer.CanvasSize = UDim2.new(0, 0, 0, count * 50)
    end

    actualizarListaJugadores()
    table.insert(connections, Players.PlayerAdded:Connect(actualizarListaJugadores))
    table.insert(connections, Players.PlayerRemoving:Connect(actualizarListaJugadores))

    local PingFrame = Instance.new("Frame")
    PingFrame.Parent = EtcPage
    PingFrame.BackgroundColor3 = Color3.fromRGB(10, 14, 22)
    PingFrame.Position = UDim2.new(0, 0, 0, 4)
    PingFrame.Size = UDim2.new(0.98, 0, 0, 46)
    PingFrame.BorderSizePixel = 0

    local PingCorner = Instance.new("UICorner")
    PingCorner.CornerRadius = UDim.new(0, 12)
    PingCorner.Parent = PingFrame

    local PingTextLbl = Instance.new("TextLabel")
    PingTextLbl.Parent = PingFrame
    PingTextLbl.BackgroundTransparency = 1
    PingTextLbl.Position = UDim2.new(0.04, 0, 0, 0)
    PingTextLbl.Size = UDim2.new(0.92, 0, 1, 0)
    PingTextLbl.Font = Enum.Font.FredokaOne
    PingTextLbl.Text = "📶 Ping: Calculando..."
    PingTextLbl.TextColor3 = Color3.fromRGB(220, 235, 255)
    PingTextLbl.TextSize = 12
    PingTextLbl.TextXAlignment = Enum.TextXAlignment.Left

    task.spawn(function()
        while task.wait(1) do
            pcall(function()
                local pingVal = math.floor(player:GetNetworkPing() * 1000)
                local fpsVal = math.floor(1 / RunService.RenderStepped:Wait())
                PingTextLbl.Text = string.format("📶 Ping: %d ms | ⚡ FPS: %d", pingVal, fpsVal)
            end)
        end
    end)

    createToggle(EtcPage, "Fly (Volar)", 58, flyEnabled, function(state) 
        flyEnabled = state 
        if not state then
            pcall(function()
                if bv then bv:Destroy() end
                if bg then bg:Destroy() end
                bv = nil; bg = nil
                local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
                if hum then hum.PlatformStand = false end
            end)
        end
    end)
    createSlider(EtcPage, "Velocidad Fly", 10, 300, flySpeed, 108, function(val) flySpeed = val end)
    createToggle(EtcPage, "Activar Speed", 170, speedEnabled, function(state)
        speedEnabled = state
        if not state then
            pcall(function()
                local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
                if hum then hum.WalkSpeed = 16 end
            end)
        end
    end)
    createSlider(EtcPage, "WalkSpeed Valor", 16, 200, walkSpeedValue, 220, function(val) walkSpeedValue = val end)
    createToggle(EtcPage, "Noclip", 282, noclipEnabled, function(state)
        noclipEnabled = state
        if not state and player.Character then
            pcall(function()
                for _, part in ipairs(player.Character:GetDescendants()) do
                    if part:IsA("BasePart") then part.CanCollide = true end
                end
            end)
        end
    end)
    createToggle(EtcPage, "Custom FOV", 332, customFovEnabled, function(state)
        customFovEnabled = state
        if not state then camera.FieldOfView = 70 end
    end)
    createSlider(EtcPage, "FOV Valor", 50, 120, fovValue, 382, function(val) fovValue = val end)

    local RejoinBtn = Instance.new("TextButton")
    RejoinBtn.Parent = EtcPage
    RejoinBtn.BackgroundColor3 = Color3.fromRGB(255, 75, 120)
    RejoinBtn.Position = UDim2.new(0, 0, 0, 448)
    RejoinBtn.Size = UDim2.new(0.98, 0, 0, 44)
    RejoinBtn.Font = Enum.Font.FredokaOne
    RejoinBtn.Text = "🔄 Rejoin Server"
    RejoinBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    RejoinBtn.TextSize = 12
    RejoinBtn.BorderSizePixel = 0

    local rCorner = Instance.new("UICorner")
    rCorner.CornerRadius = UDim.new(0, 12)
    rCorner.Parent = RejoinBtn

    local function hacerRejoin()
        mostrarNotificacion("⚡ Rejoin", "Reconectando...", 2)
        task.wait(1)
        pcall(function() TeleportService:Teleport(game.PlaceId, player) end)
    end
    RejoinBtn.MouseButton1Click:Connect(hacerRejoin)
    RejoinBtn.TouchTap:Connect(hacerRejoin)
end

local function updateClosestTarget()
    local closest = nil
    local shortestDist = math.huge
    local mousePos = UserInputService:GetMouseLocation()

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character and not blockedPlayersUI[otherPlayer] then
            pcall(function()
                local part = getTargetPart(otherPlayer.Character)
                local humanoid = otherPlayer.Character:FindFirstChildOfClass("Humanoid")
                local rootPart = otherPlayer.Character:FindFirstChild("HumanoidRootPart")
                
                if part and rootPart and humanoid and humanoid.Health > 0 then
                    local worldDist = (rootPart.Position - camera.CFrame.Position).Magnitude
                    if worldDist <= aimDistance then
                        local screenPos, onScreen = camera:WorldToViewportPoint(part.Position)
                        if onScreen then
                            local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                            if dist < shortestDist then
                                shortestDist = dist
                                closest = part
                            end
                        end
                    end
                end
            end)
        end
    end
    currentTargetPart = closest
end

RunService.RenderStepped:Connect(function(dt)
    if not ScreenGui or not ScreenGui.Parent then return end

    if speedEnabled then
        pcall(function()
            local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
            if hum then hum.WalkSpeed = walkSpeedValue end
        end)
    end

    if noclipEnabled then
        pcall(function()
            if player.Character then
                for _, part in ipairs(player.Character:GetDescendants()) do
                    if part:IsA("BasePart") then part.CanCollide = false end
                end
            end
        end)
    end

    if flyEnabled then
        pcall(function()
            local char = player.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hrp and hum then
                hum.PlatformStand = true
                if not bv or not bv.Parent then
                    bv = Instance.new("BodyVelocity")
                    bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                    bv.Parent = hrp
                end
                if not bg or not bg.Parent then
                    bg = Instance.new("BodyGyro")
                    bg.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
                    bg.Parent = hrp
                end
                bg.CFrame = camera.CFrame
                local moveDir = hum.MoveDirection
                if moveDir.Magnitude > 0 then bv.Velocity = moveDir * flySpeed else bv.Velocity = Vector3.new(0, 0.1, 0) end
            end
        end)
    else
        pcall(function()
            if bv then bv:Destroy(); bv = nil end
            if bg then bg:Destroy(); bg = nil end
            local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
            if hum then hum.PlatformStand = false end
        end)
    end

    if customFovEnabled then pcall(function() camera.FieldOfView = fovValue end) end

    if hitboxEnabled then
        for _, otherPlayer in ipairs(Players:GetPlayers()) do
            if otherPlayer ~= player then
                if blockedPlayersUI[otherPlayer] then
                    restoreSingleHitbox(otherPlayer)
                else
                    pcall(function()
                        local targetPart = getTargetPart(otherPlayer.Character)
                        local hum = otherPlayer.Character:FindFirstChildOfClass("Humanoid")
                        if targetPart and hum and hum.Health > 0 then
                            if not originalHitboxData[otherPlayer] then
                                originalHitboxData[otherPlayer] = {
                                    Part = targetPart, Size = targetPart.Size, Transparency = targetPart.Transparency,
                                    Color = targetPart.Color, Material = targetPart.Material, CanCollide = targetPart.CanCollide, Massless = targetPart.Massless
                                }
                            end
                            targetPart.Size = Vector3.new(hitboxSize, hitboxSize * 0.8, hitboxSize)
                            targetPart.Transparency = 0.5
                            targetPart.Color = Color3.fromRGB(255, 75, 120)
                            targetPart.Material = Enum.Material.Neon
                            targetPart.CanCollide = false
                            targetPart.Massless = true

                            local hrp = otherPlayer.Character:FindFirstChild("HumanoidRootPart")
                            if hrp then
                                if not targetPart:FindFirstChild("ElianHitboxBP") then
                                    local bp = Instance.new("BodyPosition")
                                    bp.Name = "ElianHitboxBP"
                                    bp.MaxForce = Vector3.new(40000, 0, 40000)
                                    bp.Position = hrp.Position
                                    bp.Parent = targetPart
                                else
                                    targetPart.ElianHitboxBP.Position = hrp.Position
                                end
                            end
                        end
                    end)
                end
            end
        end
    else
        restoreHitboxes()
    end

    if aimbotEnabled then
        if isShootingOrClicking then
            updateClosestTarget()
            if currentTargetPart then
                pcall(function() camera.CFrame = CFrame.new(camera.CFrame.Position, currentTargetPart.Position) end)
            end
        else
            currentTargetPart = nil
        end
    end

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and not blockedPlayersUI[otherPlayer] then
            pcall(function()
                local char = otherPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if hrp and hum and hum.Health > 0 and espGlobalBody then
                    local dist = (hrp.Position - camera.CFrame.Position).Magnitude
                    if dist <= espDistance then
                        local highlight = espBodies[otherPlayer]
                        if not highlight then
                            highlight = Instance.new("Highlight")
                            highlight.Name = "ElianEspBody"
                            highlight.Adornee = char
                            highlight.FillColor = Color3.fromRGB(255, 75, 120)
                            highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
                            highlight.FillTransparency = 0.5
                            highlight.OutlineTransparency = 0.2
                            highlight.Parent = CoreGui
                            espBodies[otherPlayer] = highlight
                        end
                        highlight.Enabled = true
                    else
                        if espBodies[otherPlayer] then espBodies[otherPlayer].Enabled = false end
                    end
                else
                    if espBodies[otherPlayer] then espBodies[otherPlayer]:Destroy(); espBodies[otherPlayer] = nil end
                end
            end)
        else
            if espBodies[otherPlayer] then espBodies[otherPlayer]:Destroy(); espBodies[otherPlayer] = nil end
        end
    end

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and not blockedPlayersUI[otherPlayer] then
            pcall(function()
                local char = otherPlayer.Character
                local head = char and (char:FindFirstChild("Head") or char:FindFirstChild("FakeHead"))
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if head and hum and hum.Health > 0 and espGlobalName then
                    local dist = (head.Position - camera.CFrame.Position).Magnitude
                    if dist <= espDistance then
                        local bgGui = espNames[otherPlayer]
                        if not bgGui then
                            bgGui = Instance.new("BillboardGui")
                            bgGui.Name = "ElianEspName"
                            bgGui.Adornee = head
                            bgGui.Size = UDim2.new(0, 100, 0, 40)
                            bgGui.StudsOffset = Vector3.new(0, 2.6, 0)
                            bgGui.AlwaysOnTop = true
                            bgGui.Parent = CoreGui

                            local txt = Instance.new("TextLabel")
                            txt.Name = "EspText"
                            txt.Parent = bgGui
                            txt.BackgroundTransparency = 1
                            txt.Size = UDim2.new(1, 0, 1, 0)
                            txt.Font = Enum.Font.FredokaOne
                            txt.TextColor3 = Color3.fromRGB(255, 255, 255)
                            txt.TextSize = 13
                            txt.TextStrokeTransparency = 0.3
                            espNames[otherPlayer] = bgGui
                        end
                        local txt = bgGui:FindFirstChild("EspText")
                        if txt then txt.Text = otherPlayer.Name .. " [" .. math.floor(dist) .. "m]" end
                    else
                        if espNames[otherPlayer] then espNames[otherPlayer]:Destroy(); espNames[otherPlayer] = nil end
                    end
                else
                    if espNames[otherPlayer] then espNames[otherPlayer]:Destroy(); espNames[otherPlayer] = nil end
                end
            end)
        else
            if espNames[otherPlayer] then espNames[otherPlayer]:Destroy(); espNames[otherPlayer] = nil end
        end
    end

    if Drawing then
        for _, otherPlayer in ipairs(Players:GetPlayers()) do
            if otherPlayer ~= player and not blockedPlayersUI[otherPlayer] then
                pcall(function()
                    local char = otherPlayer.Character
                    local head = char and (char:FindFirstChild("Head") or char:FindFirstChild("FakeHead"))
                    local hum = char and char:FindFirstChildOfClass("Humanoid")
                    if head and hum and hum.Health > 0 and espGlobalTracer then
                        local dist = (head.Position - camera.CFrame.Position).Magnitude
                        if dist <= espDistance then
                            local vector, onScreen = camera:WorldToViewportPoint(head.Position)
                            local tracer = espTracers[otherPlayer]
                            if not tracer then
                                tracer = Drawing.new("Line")
                                tracer.Visible = false
                                tracer.Color = Color3.fromRGB(255, 75, 120)
                                tracer.Thickness = 1.5
                                tracer.Transparency = 0.75
                                espTracers[otherPlayer] = tracer
                            end
                            if onScreen then
                                tracer.From = Vector2.new(camera.ViewportSize.X / 2, 0)
                                tracer.To = Vector2.new(vector.X, vector.Y)
                                tracer.Visible = true
                            else
                                tracer.Visible = false
                            end
                        else
                            if espTracers[otherPlayer] then espTracers[otherPlayer].Visible = false end
                        end
                    else
                        if espTracers[otherPlayer] then espTracers[otherPlayer]:Remove(); espTracers[otherPlayer] = nil end
                    end
                end)
            else
                if espTracers[otherPlayer] then espTracers[otherPlayer]:Remove(); espTracers[otherPlayer] = nil end
            end
        end
    end
end)

iniciarMenuPrincipal()
