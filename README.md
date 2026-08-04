-- =========================================================================
-- ELIAN HUB V12.1 - TEAM CHECK FIX, SPEED OPTIMIZATION & ETC TAB
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

local function destruirMenus()
    pcall(function()
        if parentGui:FindFirstChild("ErrorMenu") then parentGui.ErrorMenu:Destroy() end
        if parentGui:FindFirstChild("DeviceSelectorGui") then parentGui.DeviceSelectorGui:Destroy() end
        if parentGui:FindFirstChild("ProfessionalMenuV12") then parentGui.ProfessionalMenuV12:Destroy() end
        if parentGui:FindFirstChild("ElianHubRestoreBtn") then parentGui.ElianHubRestoreBtn:Destroy() end
        if ScreenGui then ScreenGui:Destroy() end
    end)
end

destruirMenus()

-- =========================================================================
-- VARIABLES GLOBALES
-- =========================================================================
local isMobileDevice = false

local headHitboxEnabled = false     
local headHitboxSize = 10 

local aimbotEnabled = false
local silentAimEnabled = false
local teamCheckEnabled = false 
local aimDistance = 500
local isRightClicking = false

local currentTargetHead = nil

local espBoxEnabled = false
local espNameEnabled = false
local espBodyEnabled = false
local espDistance = 500 

local flyEnabled = false
local flySpeed = 50
local speedEnabled = false
local walkSpeedValue = 16

local espBoxes = {}
local espNames = {}
local espBodies = {}
local originalHeadData = {} 
local connections = {}

-- Captura de Clic Derecho (Para PC)
table.insert(connections, UserInputService.InputBegan:Connect(function(input, gpe)
    if not gpe and input.UserInputType == Enum.UserInputType.MouseButton2 then
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

-- =========================================================================
-- CORRECCIÓN DEL TEAM CHECK Y BYPASS DE AMIGOS
-- =========================================================================
local function esAliadoOGeneroValido(otherPlayer)
    if otherPlayer == player then return true end
    
    -- Si el Team Check está desactivado, tratamos a todos los demás como enemigos válidos
    if not teamCheckEnabled then
        local success, isFriend = pcall(function()
            return player:IsFriendsWith(otherPlayer.UserId)
        end)
        if success and isFriend then
            return true -- Omite amigos si no se desea atacar a amigos
        end
        return false
    end

    -- Validación estricta de Equipo (Team Check Activado)
    if player.Team and otherPlayer.Team then
        if player.Team == otherPlayer.Team then
            return true
        end
    end

    -- Validación secundaria por Colores de Equipo en el Character (por si el Team de Roblox no se actualiza igual)
    local pChar = player.Character
    local oChar = otherPlayer.Character
    if pChar and oChar then
        local pRoot = pChar:FindFirstChild("HumanoidRootPart")
        local oRoot = oChar:FindFirstChild("HumanoidRootPart")
        if pRoot and oRoot and player.TeamColor and otherPlayer.TeamColor then
            if player.TeamColor == otherPlayer.TeamColor then
                return true
            end
        end
    end

    local success, isFriend = pcall(function()
        return player:IsFriendsWith(otherPlayer.UserId)
    end)
    if success and isFriend then
        return true
    end

    return false
end

local function restoreHeads()
    for plr, data in pairs(originalHeadData) do
        pcall(function()
            if data and data.Head and data.Head.Parent then
                data.Head.Size = data.Size
                data.Head.Transparency = data.Transparency
                data.Head.Color = data.Color
                data.Head.Material = data.Material
                data.Head.CanCollide = data.CanCollide
                data.Head.Massless = data.Massless
            end
        end)
    end
    originalHeadData = {}
end

local function fullCleanup()
    headHitboxEnabled = false
    aimbotEnabled = false
    silentAimEnabled = false
    teamCheckEnabled = false
    flyEnabled = false
    speedEnabled = false
    currentTargetHead = nil
    restoreHeads()

    for _, boxGui in pairs(espBoxes) do pcall(function() if boxGui then boxGui:Destroy() end end) end
    espBoxes = {}

    for _, nameGui in pairs(espNames) do pcall(function() if nameGui then nameGui:Destroy() end end) end
    espNames = {}

    for _, highlight in pairs(espBodies) do pcall(function() if highlight then highlight:Destroy() end end) end
    espBodies = {}

    for _, conn in pairs(connections) do pcall(function() if conn then conn:Disconnect() end end) end
    connections = {}
end

local function updateClosestHead()
    local closest = nil
    local shortestDist = aimDistance
    local mousePos = UserInputService:GetMouseLocation()

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if not esAliadoOGeneroValido(otherPlayer) and otherPlayer.Character then
            pcall(function()
                local head = getRealHead(otherPlayer.Character)
                local humanoid = otherPlayer.Character:FindFirstChildOfClass("Humanoid")
                
                if head and humanoid and humanoid.Health > 0 then
                    local screenPos, onScreen = camera:WorldToViewportPoint(head.Position)
                    if onScreen then
                        local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                        if dist < shortestDist then
                            shortestDist = dist
                            closest = head
                        end
                    end
                end
            end)
        end
    end
    
    currentTargetHead = closest
end

-- =========================================================================
-- INTERFAZ PRINCIPAL
-- =========================================================================
local function iniciarMenuPrincipal()
    destruirMenus()

    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "ProfessionalMenuV12"
    ScreenGui.Parent = parentGui
    ScreenGui.ResetOnSpawn = false
    ScreenGui.IgnoreGuiInset = true

    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Parent = ScreenGui
    MainFrame.BackgroundColor3 = Color3.fromRGB(7, 8, 12)
    MainFrame.Position = UDim2.new(0.5, -250, 0.5, -190)
    MainFrame.Size = UDim2.new(0, 500, 0, 380)
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true

    local MainCorner = Instance.new("UICorner")
    MainCorner.CornerRadius = UDim.new(0, 18)
    MainCorner.Parent = MainFrame

    local MainStroke = Instance.new("UIStroke")
    MainStroke.Thickness = 2
    MainStroke.Color = Color3.fromRGB(130, 80, 255)
    MainStroke.Parent = MainFrame

    local GlowEffect = Instance.new("ImageLabel")
    GlowEffect.Name = "Glow"
    GlowEffect.Parent = MainFrame
    GlowEffect.BackgroundTransparency = 1
    GlowEffect.Position = UDim2.new(0, -15, 0, -15)
    GlowEffect.Size = UDim2.new(1, 30, 1, 30)
    GlowEffect.ZIndex = -1
    GlowEffect.Image = "rbxassetid://6014261993"
    GlowEffect.ImageColor3 = Color3.fromRGB(110, 60, 255)
    GlowEffect.ImageTransparency = 0.5

    -- Barra Superior
    local TopBar = Instance.new("Frame")
    TopBar.Parent = MainFrame
    TopBar.BackgroundColor3 = Color3.fromRGB(12, 14, 20)
    TopBar.Size = UDim2.new(1, 0, 0, 52)
    TopBar.BorderSizePixel = 0

    local TopCorner = Instance.new("UICorner")
    TopCorner.CornerRadius = UDim.new(0, 18)
    TopCorner.Parent = TopBar

    local Title = Instance.new("TextLabel")
    Title.Parent = TopBar
    Title.BackgroundTransparency = 1
    Title.Position = UDim2.new(0.04, 0, 0, 0)
    Title.Size = UDim2.new(0.65, 0, 1, 0)
    Title.Font = Enum.Font.GothamBold
    Title.Text = "💎 ELIAN HUB V12.1  |  " .. (isMobileDevice and "MOBILE" or "PC")
    Title.TextColor3 = Color3.fromRGB(240, 240, 255)
    Title.TextSize = 13
    Title.TextXAlignment = Enum.TextXAlignment.Left

    local MinimizeBtn = Instance.new("TextButton")
    MinimizeBtn.Parent = TopBar
    MinimizeBtn.BackgroundColor3 = Color3.fromRGB(22, 25, 38)
    MinimizeBtn.Position = UDim2.new(0.78, 0, 0.22, 0)
    MinimizeBtn.Size = UDim2.new(0, 32, 0, 28)
    MinimizeBtn.Font = Enum.Font.GothamBold
    MinimizeBtn.Text = "─"
    MinimizeBtn.TextColor3 = Color3.fromRGB(180, 190, 220)
    MinimizeBtn.TextSize = 12
    MinimizeBtn.BorderSizePixel = 0

    local MinCorner = Instance.new("UICorner")
    MinCorner.CornerRadius = UDim.new(0, 8)
    MinCorner.Parent = MinimizeBtn

    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Parent = TopBar
    CloseBtn.BackgroundColor3 = Color3.fromRGB(235, 45, 80)
    CloseBtn.Position = UDim2.new(0.88, 0, 0.22, 0)
    CloseBtn.Size = UDim2.new(0, 32, 0, 28)
    CloseBtn.Font = Enum.Font.GothamBold
    CloseBtn.Text = "✕"
    CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    CloseBtn.TextSize = 12
    CloseBtn.BorderSizePixel = 0

    local CloseCorner = Instance.new("UICorner")
    CloseCorner.CornerRadius = UDim.new(0, 8)
    CloseCorner.Parent = CloseBtn

    local function crearBotonRestaurar()
        local RestoreGui = Instance.new("ScreenGui")
        RestoreGui.Name = "ElianHubRestoreBtn"
        RestoreGui.Parent = parentGui
        RestoreGui.ResetOnSpawn = false

        local OpenBtn = Instance.new("TextButton")
        OpenBtn.Parent = RestoreGui
        OpenBtn.BackgroundColor3 = Color3.fromRGB(10, 12, 18)
        OpenBtn.Position = UDim2.new(0.02, 0, 0.35, 0)
        OpenBtn.Size = UDim2.new(0, 155, 0, 46)
        OpenBtn.Font = Enum.Font.GothamBold
        OpenBtn.Text = "⚡ OPEN ELIAN HUB"
        OpenBtn.TextColor3 = Color3.fromRGB(0, 230, 255)
        OpenBtn.TextSize = 12
        OpenBtn.Active = true
        OpenBtn.Draggable = true

        local OCorner = Instance.new("UICorner")
        OCorner.CornerRadius = UDim.new(0, 14)
        OCorner.Parent = OpenBtn

        local OStroke = Instance.new("UIStroke")
        OStroke.Color = Color3.fromRGB(130, 80, 255)
        OStroke.Thickness = 2
        OStroke.Parent = OpenBtn

        local function restaurar()
            MainFrame.Visible = true
            RestoreGui:Destroy()
        end

        OpenBtn.MouseButton1Click:Connect(restaurar)
        OpenBtn.TouchTap:Connect(restaurar)
    end

    local function minimizarmenu()
        MainFrame.Visible = false
        crearBotonRestaurar()
    end

    MinimizeBtn.MouseButton1Click:Connect(minimizarmenu)
    MinimizeBtn.TouchTap:Connect(minimizarmenu)

    local ContainerBody = Instance.new("Frame")
    ContainerBody.Parent = MainFrame
    ContainerBody.BackgroundTransparency = 1
    ContainerBody.Position = UDim2.new(0, 0, 0, 52)
    ContainerBody.Size = UDim2.new(1, 0, 1, -52)

    -- Menú Lateral
    local TabContainer = Instance.new("Frame")
    TabContainer.Parent = ContainerBody
    TabContainer.BackgroundColor3 = Color3.fromRGB(10, 12, 18)
    TabContainer.Position = UDim2.new(0.03, 0, 0.04, 0)
    TabContainer.Size = UDim2.new(0, 130, 0, 305)
    TabContainer.BorderSizePixel = 0

    local TabCorner = Instance.new("UICorner")
    TabCorner.CornerRadius = UDim.new(0, 14)
    TabCorner.Parent = TabContainer

    local CombatTabBtn = Instance.new("TextButton")
    CombatTabBtn.Parent = TabContainer
    CombatTabBtn.BackgroundColor3 = Color3.fromRGB(120, 70, 255)
    CombatTabBtn.Position = UDim2.new(0.08, 0, 0.05, 0)
    CombatTabBtn.Size = UDim2.new(0.84, 0, 0, 36)
    CombatTabBtn.Font = Enum.Font.GothamBold
    CombatTabBtn.Text = "⚔ Combat"
    CombatTabBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    CombatTabBtn.TextSize = 12
    CombatTabBtn.BorderSizePixel = 0

    local CombatCorner = Instance.new("UICorner")
    CombatCorner.CornerRadius = UDim.new(0, 8)
    CombatCorner.Parent = CombatTabBtn

    local VisualTabBtn = Instance.new("TextButton")
    VisualTabBtn.Parent = TabContainer
    VisualTabBtn.BackgroundColor3 = Color3.fromRGB(16, 19, 28)
    VisualTabBtn.Position = UDim2.new(0.08, 0, 0.18, 0)
    VisualTabBtn.Size = UDim2.new(0.84, 0, 0, 36)
    VisualTabBtn.Font = Enum.Font.GothamBold
    VisualTabBtn.Text = "👁 Visuals"
    VisualTabBtn.TextColor3 = Color3.fromRGB(130, 140, 170)
    VisualTabBtn.TextSize = 12
    VisualTabBtn.BorderSizePixel = 0

    local VisualCorner = Instance.new("UICorner")
    VisualCorner.CornerRadius = UDim.new(0, 8)
    VisualCorner.Parent = VisualTabBtn

    local EtcTabBtn = Instance.new("TextButton")
    EtcTabBtn.Parent = TabContainer
    EtcTabBtn.BackgroundColor3 = Color3.fromRGB(16, 19, 28)
    EtcTabBtn.Position = UDim2.new(0.08, 0, 0.31, 0)
    EtcTabBtn.Size = UDim2.new(0.84, 0, 0, 36)
    EtcTabBtn.Font = Enum.Font.GothamBold
    EtcTabBtn.Text = "⚙ ETC"
    EtcTabBtn.TextColor3 = Color3.fromRGB(130, 140, 170)
    EtcTabBtn.TextSize = 12
    EtcTabBtn.BorderSizePixel = 0

    local EtcCorner = Instance.new("UICorner")
    EtcCorner.CornerRadius = UDim.new(0, 8)
    EtcCorner.Parent = EtcTabBtn

    local PagesContainer = Instance.new("Frame")
    PagesContainer.Parent = ContainerBody
    PagesContainer.Position = UDim2.new(0.32, 0, 0.04, 0)
    PagesContainer.Size = UDim2.new(0, 325, 0, 305)
    PagesContainer.BackgroundTransparency = 1

    local CombatPage = Instance.new("ScrollingFrame")
    CombatPage.Parent = PagesContainer
    CombatPage.BackgroundTransparency = 1
    CombatPage.Size = UDim2.new(1, 0, 1, 0)
    CombatPage.CanvasSize = UDim2.new(0, 0, 1.4, 0)
    CombatPage.ScrollBarThickness = 2
    CombatPage.ScrollBarImageColor3 = Color3.fromRGB(120, 70, 255)
    CombatPage.Active = true

    local VisualPage = Instance.new("ScrollingFrame")
    VisualPage.Parent = PagesContainer
    VisualPage.BackgroundTransparency = 1
    VisualPage.Size = UDim2.new(1, 0, 1, 0)
    VisualPage.CanvasSize = UDim2.new(0, 0, 1.3, 0)
    VisualPage.ScrollBarThickness = 2
    VisualPage.ScrollBarImageColor3 = Color3.fromRGB(120, 70, 255)
    VisualPage.Visible = false
    VisualPage.Active = true

    local EtcPage = Instance.new("ScrollingFrame")
    EtcPage.Parent = PagesContainer
    EtcPage.BackgroundTransparency = 1
    EtcPage.Size = UDim2.new(1, 0, 1, 0)
    EtcPage.CanvasSize = UDim2.new(0, 0, 1.3, 0)
    EtcPage.ScrollBarThickness = 2
    EtcPage.ScrollBarImageColor3 = Color3.fromRGB(120, 70, 255)
    EtcPage.Visible = false
    EtcPage.Active = true

    local function switchTab(selectedPage, activeBtn)
        CombatPage.Visible = false
        VisualPage.Visible = false
        EtcPage.Visible = false
        selectedPage.Visible = true

        for _, btn in ipairs({CombatTabBtn, VisualTabBtn, EtcTabBtn}) do
            TweenService:Create(btn, TweenInfo.new(0.25), {BackgroundColor3 = Color3.fromRGB(16, 19, 28), TextColor3 = Color3.fromRGB(130, 140, 170)}):Play()
        end
        TweenService:Create(activeBtn, TweenInfo.new(0.25), {BackgroundColor3 = Color3.fromRGB(120, 70, 255), TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
    end

    CombatTabBtn.MouseButton1Click:Connect(function() switchTab(CombatPage, CombatTabBtn) end)
    CombatTabBtn.TouchTap:Connect(function() switchTab(CombatPage, CombatTabBtn) end)
    
    VisualTabBtn.MouseButton1Click:Connect(function() switchTab(VisualPage, VisualTabBtn) end)
    VisualTabBtn.TouchTap:Connect(function() switchTab(VisualPage, VisualTabBtn) end)

    EtcTabBtn.MouseButton1Click:Connect(function() switchTab(EtcPage, EtcTabBtn) end)
    EtcTabBtn.TouchTap:Connect(function() switchTab(EtcPage, EtcTabBtn) end)

    CloseBtn.MouseButton1Click:Connect(function() fullCleanup(); ScreenGui:Destroy() end)
    CloseBtn.TouchTap:Connect(function() fullCleanup(); ScreenGui:Destroy() end)

    task.spawn(function()
        while ScreenGui and ScreenGui.Parent do
            pcall(function()
                local hue = (tick() * 0.35) % 1
                local colorRgb = Color3.fromHSV(hue, 0.75, 1)
                MainStroke.Color = colorRgb
            end)
            task.wait(0.08)
        end
    end)

    local function createToggle(parent, name, posY, initialVal, callback)
        local ToggleBtn = Instance.new("TextButton")
        ToggleBtn.Parent = parent
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(12, 14, 22)
        ToggleBtn.Position = UDim2.new(0.02, 0, 0, posY)
        ToggleBtn.Size = UDim2.new(0.95, 0, 0, 42)
        ToggleBtn.Font = Enum.Font.GothamMedium
        ToggleBtn.Text = "     " .. name
        ToggleBtn.TextColor3 = Color3.fromRGB(220, 225, 245)
        ToggleBtn.TextSize = 12
        ToggleBtn.TextXAlignment = Enum.TextXAlignment.Left
        ToggleBtn.BorderSizePixel = 0

        local TCorner = Instance.new("UICorner")
        TCorner.CornerRadius = UDim.new(0, 10)
        TCorner.Parent = ToggleBtn

        local StatusIndicator = Instance.new("Frame")
        StatusIndicator.Parent = ToggleBtn
        StatusIndicator.BackgroundColor3 = initialVal and Color3.fromRGB(0, 230, 255) or Color3.fromRGB(28, 32, 48)
        StatusIndicator.Position = UDim2.new(0.84, 0, 0.28, 0)
        StatusIndicator.Size = UDim2.new(0, 32, 0, 18)

        local SCorner = Instance.new("UICorner")
        SCorner.CornerRadius = UDim.new(0, 9)
        SCorner.Parent = StatusIndicator

        local Dot = Instance.new("Frame")
        Dot.Parent = StatusIndicator
        Dot.BackgroundColor3 = initialVal and Color3.fromRGB(7, 8, 12) or Color3.fromRGB(140, 150, 180)
        Dot.Position = initialVal and UDim2.new(0.52, 0, 0.12, 0) or UDim2.new(0.08, 0, 0.12, 0)
        Dot.Size = UDim2.new(0, 13, 0, 13)

        local DCorner = Instance.new("UICorner")
        DCorner.CornerRadius = UDim.new(0, 7)
        DCorner.Parent = Dot

        local enabled = initialVal
        local function triggerToggle()
            enabled = not enabled
            if enabled then
                TweenService:Create(StatusIndicator, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(0, 230, 255)}):Play()
                TweenService:Create(Dot, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(7, 8, 12)}):Play()
                Dot:TweenPosition(UDim2.new(0.52, 0, 0.12, 0), "Out", "Quad", 0.18, true)
            else
                TweenService:Create(StatusIndicator, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(28, 32, 48)}):Play()
                TweenService:Create(Dot, TweenInfo.new(0.18), {BackgroundColor3 = Color3.fromRGB(140, 150, 180)}):Play()
                Dot:TweenPosition(UDim2.new(0.08, 0, 0.12, 0), "Out", "Quad", 0.18, true)
            end
            callback(enabled)
        end

        ToggleBtn.MouseButton1Click:Connect(triggerToggle)
        ToggleBtn.TouchTap:Connect(triggerToggle)
    end

    local function createSlider(parent, name, min, max, default, posY, callback)
        local SliderFrame = Instance.new("Frame")
        SliderFrame.Parent = parent
        SliderFrame.BackgroundColor3 = Color3.fromRGB(12, 14, 22)
        SliderFrame.Position = UDim2.new(0.02, 0, 0, posY)
        SliderFrame.Size = UDim2.new(0.95, 0, 0, 56)
        SliderFrame.BorderSizePixel = 0

        local SFrameCorner = Instance.new("UICorner")
        SFrameCorner.CornerRadius = UDim.new(0, 10)
        SFrameCorner.Parent = SliderFrame

        local Label = Instance.new("TextLabel")
        Label.Parent = SliderFrame
        Label.BackgroundTransparency = 1
        Label.Position = UDim2.new(0.05, 0, 0.10, 0)
        Label.Text = name
        Label.TextColor3 = Color3.fromRGB(200, 210, 230)
        Label.TextSize = 12
        Label.TextXAlignment = Enum.TextXAlignment.Left
        Label.Font = Enum.Font.GothamMedium

        local ValueLabel = Instance.new("TextLabel")
        ValueLabel.Parent = SliderFrame
        ValueLabel.BackgroundTransparency = 1
        ValueLabel.Position = UDim2.new(0.05, 0, 0.10, 0)
        ValueLabel.Size = UDim2.new(0.9, 0, 0, 20)
        ValueLabel.Text = tostring(default)
        ValueLabel.TextColor3 = Color3.fromRGB(0, 230, 255)
        ValueLabel.TextSize = 12
        ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
        ValueLabel.Font = Enum.Font.GothamBold

        local Bar = Instance.new("Frame")
        Bar.Parent = SliderFrame
        Bar.BackgroundColor3 = Color3.fromRGB(25, 29, 44)
        Bar.Position = UDim2.new(0.05, 0, 0.68, 0)
        Bar.Size = UDim2.new(0.9, 0, 0, 6)

        local BarCorner = Instance.new("UICorner")
        BarCorner.CornerRadius = UDim.new(0, 3)
        BarCorner.Parent = Bar

        local Fill = Instance.new("Frame")
        Fill.Parent = Bar
        Fill.BackgroundColor3 = Color3.fromRGB(120, 70, 255)
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

    -- OPCIONES DE COMBATE
    createToggle(CombatPage, "Hitbox Cabeza Enemigo", 10, headHitboxEnabled, function(state)
        headHitboxEnabled = state
        if not state then restoreHeads() end
    end)
    
    createSlider(CombatPage, "Tamaño Cabeza (Max 45)", 2, 45, headHitboxSize, 60, function(val) 
        headHitboxSize = val 
    end)

    createToggle(CombatPage, "🛡 Team Check Estricto", 125, teamCheckEnabled, function(state)
        teamCheckEnabled = state
    end)

    if not isMobileDevice then
        createToggle(CombatPage, "Aimbot Directo (Click Dcho)", 180, aimbotEnabled, function(state) aimbotEnabled = state end)
        createToggle(CombatPage, "Aimbot Silent (Cabeza)", 230, silentAimEnabled, function(state) silentAimEnabled = state end)
    end

    -- OPCIONES VISUALES
    createToggle(VisualPage, "ESP Neon Box", 10, espBoxEnabled, function(state) espBoxEnabled = state end)
    createToggle(VisualPage, "ESP Name", 60, espNameEnabled, function(state) espNameEnabled = state end)
    createToggle(VisualPage, "ESP Cuerpo Azul", 110, espBodyEnabled, function(state) espBodyEnabled = state end)
    createSlider(VisualPage, "ESP Distancia", 50, 1000, espDistance, 160, function(val) espDistance = val end)

    -- OPCIONES ETC (FLY Y SPEED MEJORADO)
    createToggle(EtcPage, "Fly (Volar)", 10, flyEnabled, function(state) 
        flyEnabled = state 
    end)
    createSlider(EtcPage, "Velocidad Fly", 10, 300, flySpeed, 60, function(val) 
        flySpeed = val 
    end)

    createToggle(EtcPage, "Activar Speed (Velocidad)", 125, speedEnabled, function(state) 
        speedEnabled = state 
        if not state then
            pcall(function()
                local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
                if hum then hum.WalkSpeed = 16 end
            end)
        end
    end)
    createSlider(EtcPage, "WalkSpeed Valor", 16, 200, walkSpeedValue, 175, function(val) 
        walkSpeedValue = val 
    end)
end

-- =========================================================================
-- SELECTOR DE DISPOSITIVO
-- =========================================================================
local function mostrarSelectorDispositivo()
    destruirMenus()

    local DeviceGui = Instance.new("ScreenGui")
    DeviceGui.Name = "DeviceSelectorGui"
    DeviceGui.Parent = parentGui
    DeviceGui.ResetOnSpawn = false

    local Frame = Instance.new("Frame")
    Frame.Size = UDim2.new(0, 310, 0, 135)
    Frame.Position = UDim2.new(0.5, -155, 0.5, -67)
    Frame.BackgroundColor3 = Color3.fromRGB(7, 8, 12)
    Frame.BorderSizePixel = 0
    Frame.Parent = DeviceGui

    local Corner = Instance.new("UICorner")
    Corner.CornerRadius = UDim.new(0, 16)
    Corner.Parent = Frame

    local Stroke = Instance.new("UIStroke")
    Stroke.Color = Color3.fromRGB(130, 80, 255)
    Stroke.Thickness = 2
    Stroke.Parent = Frame

    local Title = Instance.new("TextLabel")
    Title.Size = UDim2.new(1, 0, 0, 42)
    Title.BackgroundTransparency = 1
    Title.Font = Enum.Font.GothamBold
    Title.Text = "SELECCIONA TU PLATAFORMA"
    Title.TextColor3 = Color3.fromRGB(240, 240, 255)
    Title.TextSize = 12
    Title.Parent = Frame

    local MobileBtn = Instance.new("TextButton")
    MobileBtn.Size = UDim2.new(0.42, 0, 0, 42)
    MobileBtn.Position = UDim2.new(0.06, 0, 0.48, 0)
    MobileBtn.BackgroundColor3 = Color3.fromRGB(120, 70, 255)
    MobileBtn.Font = Enum.Font.GothamBold
    MobileBtn.Text = "📱 MÓVIL"
    MobileBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    MobileBtn.TextSize = 12
    MobileBtn.Parent = Frame

    local MCorner = Instance.new("UICorner")
    MCorner.CornerRadius = UDim.new(0, 10)
    MCorner.Parent = MobileBtn

    local PcBtn = Instance.new("TextButton")
    PcBtn.Size = UDim2.new(0.42, 0, 0, 42)
    PcBtn.Position = UDim2.new(0.52, 0, 0.48, 0)
    PcBtn.BackgroundColor3 = Color3.fromRGB(0, 190, 230)
    PcBtn.Font = Enum.Font.GothamBold
    PcBtn.Text = "💻 PC"
    PcBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    PcBtn.TextSize = 12
    PcBtn.Parent = Frame

    local PCorner = Instance.new("UICorner")
    PCorner.CornerRadius = UDim.new(0, 10)
    PCorner.Parent = PcBtn

    local function seleccionarMovil()
        isMobileDevice = true
        DeviceGui:Destroy()
        iniciarMenuPrincipal()
    end

    local function seleccionarPC()
        isMobileDevice = false
        DeviceGui:Destroy()
        iniciarMenuPrincipal()
    end

    MobileBtn.MouseButton1Click:Connect(seleccionarMovil)
    MobileBtn.TouchTap:Connect(seleccionarMovil)

    PcBtn.MouseButton1Click:Connect(seleccionarPC)
    PcBtn.TouchTap:Connect(seleccionarPC)
end

mostrarSelectorDispositivo()

-- =========================================================================
-- LOGICA DE FLY Y SPEED OPTIMIZADO (ANTI-RESET)
-- =========================================================================
local bv, bg
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

table.insert(connections, RunService.RenderStepped:Connect(function()
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local humanoid = char and char:FindFirstChildOfClass("Humanoid")

    -- Aplicar Speed con persistencia por frame (evita que el juego lo sobrescriba)
    if speedEnabled and humanoid then
        humanoid.WalkSpeed = walkSpeedValue
    end

    -- Sistema de Fly
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
            moveDir = camCFrame.LookVector * (humanoid.MoveDirection.Magnitude > 0 and 1 or 0)
            if humanoid.MoveDirection.Magnitude > 0 then
                moveDir = (camCFrame.LookVector * humanoid.MoveDirection.Z + camCFrame.RightVector * humanoid.MoveDirection.X).Unit
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
-- EXPANSION DE HITBOX CON FILTRO CORREGIDO
-- =========================================================================
table.insert(connections, RunService.RenderStepped:Connect(function()
    if headHitboxEnabled then
        for _, otherPlayer in ipairs(Players:GetPlayers()) do
            if not esAliadoOGeneroValido(otherPlayer) and otherPlayer.Character then
                pcall(function()
                    local realHead = getRealHead(otherPlayer.Character)
                    local humanoid = otherPlayer.Character:FindFirstChildOfClass("Humanoid")
                    
                    if realHead and humanoid and humanoid.Health > 0 then
                        if not originalHeadData[otherPlayer] then
                            originalHeadData[otherPlayer] = {
                                Head = realHead,
                                Size = realHead.Size,
                                Transparency = realHead.Transparency,
                                Color = realHead.Color,
                                Material = realHead.Material,
                                CanCollide = realHead.CanCollide,
                                Massless = realHead.Massless
                            }
                        end
                        
                        realHead.Massless = true           
                        realHead.CanCollide = false        
                        
                        pcall(function()
                            realHead.CanTouch = false
                            realHead.CanQuery = true
                        end)

                        realHead.Size = Vector3.new(headHitboxSize, headHitboxSize, headHitboxSize)
                        realHead.Transparency = 0.65
                        realHead.Color = Color3.fromRGB(0, 230, 255)
                        realHead.Material = Enum.Material.Neon
                    end
                end)
            end
        end
    else
        if next(originalHeadData) then
            restoreHeads()
        end
    end
end))

-- =========================================================================
-- BUCLE PRINCIPAL, AIMBOT Y ESP CON FILTRO CORREGIDO
-- =========================================================================
table.insert(connections, RunService.RenderStepped:Connect(function()
    if not isMobileDevice and (aimbotEnabled or silentAimEnabled) then
        updateClosestHead()
    else
        currentTargetHead = nil
    end

    if not isMobileDevice and aimbotEnabled and isRightClicking and currentTargetHead and currentTargetHead.Parent then
        camera.CFrame = CFrame.new(camera.CFrame.Position, currentTargetHead.Position)
    end

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if esAliadoOGeneroValido(otherPlayer) then
            if espBodies[otherPlayer] then espBodies[otherPlayer].Enabled = false end
            if espBoxes[otherPlayer] then espBoxes[otherPlayer].Enabled = false end
            if espNames[otherPlayer] then espNames[otherPlayer].Enabled = false end
        elseif otherPlayer.Character then
            pcall(function()
                local rootPart = otherPlayer.Character:FindFirstChild("HumanoidRootPart") or getRealHead(otherPlayer.Character)
                local myRoot = player.Character and (player.Character:FindFirstChild("HumanoidRootPart") or getRealHead(player.Character))
                
                if rootPart and myRoot then
                    local distance = (rootPart.Position - myRoot.Position).Magnitude
                    local inRange = distance <= espDistance

                    if espBodyEnabled and inRange then
                        local highlight = espBodies[otherPlayer]
                        if not highlight or highlight.Parent ~= otherPlayer.Character then
                            if highlight then highlight:Destroy() end
                            highlight = Instance.new("Highlight")
                            highlight.Adornee = otherPlayer.Character
                            highlight.FillColor = Color3.fromRGB(120, 70, 255)
                            highlight.FillTransparency = 0.5
                            highlight.OutlineColor = Color3.fromRGB(0, 230, 255)
                            highlight.Parent = otherPlayer.Character
                            espBodies[otherPlayer] = highlight
                        end
                        highlight.Enabled = true
                    else
                        if espBodies[otherPlayer] then espBodies[otherPlayer].Enabled = false end
                    end

                    if espBoxEnabled and inRange then
                        local boxGui = espBoxes[otherPlayer]
                        if not boxGui or boxGui.Parent ~= rootPart then
                            if boxGui then boxGui:Destroy() end
                            boxGui = Instance.new("BillboardGui")
                            boxGui.AlwaysOnTop = true
                            boxGui.Size = UDim2.new(4, 0, 5.5, 0)
                            boxGui.Parent = rootPart
                            local frame = Instance.new("Frame", boxGui)
                            frame.BackgroundTransparency = 1
                            frame.Size = UDim2.new(1, 0, 1, 0)
                            local stroke = Instance.new("UIStroke", frame)
                            stroke.Color = Color3.fromRGB(0, 230, 255)
                            stroke.Thickness = 2
                            espBoxes[otherPlayer] = boxGui
                        end
                        espBoxes[otherPlayer].Enabled = true
                    else
                        if espBoxes[otherPlayer] then espBoxes[otherPlayer].Enabled = false end
                    end

                    if espNameEnabled and inRange then
                        local nameGui = espNames[otherPlayer]
                        if not nameGui or nameGui.Parent ~= rootPart then
                            if nameGui then nameGui:Destroy() end
                            nameGui = Instance.new("BillboardGui")
                            nameGui.AlwaysOnTop = true
                            nameGui.Size = UDim2.new(0, 200, 0, 40)
                            nameGui.StudsOffset = Vector3.new(0, 3.8, 0)
                            nameGui.Parent = rootPart
                            local textLabel = Instance.new("TextLabel", nameGui)
                            textLabel.BackgroundTransparency = 1
                            textLabel.Size = UDim2.new(1, 0, 1, 0)
                            textLabel.Font = Enum.Font.GothamBold
                            textLabel.TextSize = 13
                            textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
                            espNames[otherPlayer] = nameGui
                        end
                        nameGui.Enabled = true
                        local lbl = nameGui:FindFirstChildOfClass("TextLabel")
                        if lbl then lbl.Text = otherPlayer.Name .. " [" .. tostring(math.floor(distance)) .. "m]" end
                    else
                        if espNames[otherPlayer] then espNames[otherPlayer].Enabled = false end
                    end
                end
            end)
        end
    end
end))

-- =========================================================================
-- HOOKS SILENT AIM (PC)
-- =========================================================================
if hookmetamethod then
    local oldIndex
    oldIndex = hookmetamethod(game, "__index", function(self, key)
        if not checkcaller() and not isMobileDevice and silentAimEnabled and currentTargetHead and currentTargetHead.Parent then
            if self:IsA("Mouse") then
                if key == "Hit" then
                    return currentTargetHead.CFrame
                elseif key == "Target" then
                    return currentTargetHead
                end
            end
        end
        return oldIndex(self, key)
    end)

    local oldNamecall
    oldNamecall = hookmetamethod(game, "__namecall", function(self, ...)
        local method = getnamecallmethod()
        local args = {...}

        if not checkcaller() and not isMobileDevice and silentAimEnabled and currentTargetHead and currentTargetHead.Parent then
            if method == "Raycast" and #args >= 2 then
                local origin = args[1]
                if typeof(origin) == "Vector3" then
                    args[2] = (currentTargetHead.Position - origin).Unit * 1000
                    return oldNamecall(self, unpack(args))
                end
            elseif method == "FindPartOnWithIgnoreList" or method == "FindPartOnWithRay" then
                local val = args[1]
                if typeof(val) == "Ray" then
                    args[1] = Ray.new(val.Origin, (currentTargetHead.Position - val.Origin).Unit * 1000)
                    return oldNamecall(self, unpack(args))
                end
            end
        end

        return oldNamecall(self, ...)
    end)
end
