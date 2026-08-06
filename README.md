-- =========================================================================
-- ELIAN HUB V13.3 - ULTIMATE MOBILE & PC FIX (HITBOX + ESP)
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
        if parentGui:FindFirstChild("ProfessionalMenuV12") then parentGui.ProfessionalMenuV12:Destroy() end
        if parentGui:FindFirstChild("ProfessionalMenuV13") then parentGui.ProfessionalMenuV13:Destroy() end
        if parentGui:FindFirstChild("ElianHubRestoreBtn") then parentGui.ElianHubRestoreBtn:Destroy() end
        if ScreenGui then ScreenGui:Destroy() end
    end)
end

destruirMenus()

-- =========================================================================
-- DETECCIÓN HÍBRIDA ROBUSTA (MÓVIL / PC)
-- =========================================================================
local isMobileDevice = (not UserInputService.KeyboardEnabled) and (UserInputService.TouchEnabled or table.find({"Delta", "Hydrogen", "Codex", "Fluxus", "ArceusX", "KRNL", "Synapse"}, identifyexecutor and identifyexecutor() or "") ~= nil)
if UserInputService.TouchEnabled and not isMobileDevice then
    isMobileDevice = true
end

-- =========================================================================
-- VARIABLES GLOBALES Y DE ESTADO
-- =========================================================================
local headHitboxEnabled = false      
local headHitboxSize = 10 

local aimbotEnabled = false
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

local function fullCleanup()
    headHitboxEnabled = false
    aimbotEnabled = false
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
        if otherPlayer ~= player and otherPlayer.Character then
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
-- INTERFAZ GRÁFICA (ELIAN HUB V13.3)
-- =========================================================================
local function iniciarMenuPrincipal()
    destruirMenus()

    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "ProfessionalMenuV13"
    ScreenGui.Parent = parentGui
    ScreenGui.ResetOnSpawn = false
    ScreenGui.IgnoreGuiInset = true

    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Parent = ScreenGui
    MainFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
    MainFrame.Position = UDim2.new(0.5, -260, 0.5, -200)
    MainFrame.Size = UDim2.new(0, 520, 0, 400)
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true

    local MainCorner = Instance.new("UICorner")
    MainCorner.CornerRadius = UDim.new(0, 16)
    MainCorner.Parent = MainFrame

    local MainGradient = Instance.new("UIGradient")
    MainGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(20, 23, 33)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(11, 13, 18))
    })
    MainGradient.Rotation = 45
    MainGradient.Parent = MainFrame

    local MainStroke = Instance.new("UIStroke")
    MainStroke.Thickness = 1.8
    MainStroke.Color = Color3.fromRGB(140, 90, 255)
    MainStroke.Parent = MainFrame

    local TopGlow = Instance.new("Frame")
    TopGlow.Parent = MainFrame
    TopGlow.BackgroundColor3 = Color3.fromRGB(140, 90, 255)
    TopGlow.BackgroundTransparency = 0.85
    TopGlow.Size = UDim2.new(1, 0, 0, 3)
    TopGlow.BorderSizePixel = 0

    local TopBar = Instance.new("Frame")
    TopBar.Parent = MainFrame
    TopBar.BackgroundTransparency = 1
    TopBar.Size = UDim2.new(1, 0, 0, 50)

    local Title = Instance.new("TextLabel")
    Title.Parent = TopBar
    Title.BackgroundTransparency = 1
    Title.Position = UDim2.new(0.04, 0, 0, 0)
    Title.Size = UDim2.new(0.65, 0, 1, 0)
    Title.Font = Enum.Font.GothamBold
    Title.Text = "💎 ELIAN HUB V13.3 <font color='#8c5aff'>•</font> " .. (isMobileDevice and "MOBILE" or "PC")
    Title.RichText = true
    Title.TextColor3 = Color3.fromRGB(245, 245, 255)
    Title.TextSize = 13
    Title.TextXAlignment = Enum.TextXAlignment.Left

    local MinimizeBtn = Instance.new("TextButton")
    MinimizeBtn.Parent = TopBar
    MinimizeBtn.BackgroundColor3 = Color3.fromRGB(26, 30, 42)
    MinimizeBtn.Position = UDim2.new(0.79, 0, 0.2, 0)
    MinimizeBtn.Size = UDim2.new(0, 34, 0, 30)
    MinimizeBtn.Font = Enum.Font.GothamBold
    MinimizeBtn.Text = "—"
    MinimizeBtn.TextColor3 = Color3.fromRGB(180, 190, 220)
    MinimizeBtn.TextSize = 12
    MinimizeBtn.BorderSizePixel = 0

    local MinCorner = Instance.new("UICorner")
    MinCorner.CornerRadius = UDim.new(0, 8)
    MinCorner.Parent = MinimizeBtn

    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Parent = TopBar
    CloseBtn.BackgroundColor3 = Color3.fromRGB(235, 60, 90)
    CloseBtn.Position = UDim2.new(0.89, 0, 0.2, 0)
    CloseBtn.Size = UDim2.new(0, 34, 0, 30)
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
        OpenBtn.BackgroundColor3 = Color3.fromRGB(15, 17, 23)
        OpenBtn.Position = UDim2.new(0.02, 0, 0.35, 0)
        OpenBtn.Size = UDim2.new(0, 150, 0, 44)
        OpenBtn.Font = Enum.Font.GothamBold
        OpenBtn.Text = "⚡ ELIAN HUB"
        OpenBtn.TextColor3 = Color3.fromRGB(0, 230, 255)
        OpenBtn.TextSize = 12
        OpenBtn.Active = true
        OpenBtn.Draggable = true

        local OCorner = Instance.new("UICorner")
        OCorner.CornerRadius = UDim.new(0, 12)
        OCorner.Parent = OpenBtn

        local OStroke = Instance.new("UIStroke")
        OStroke.Color = Color3.fromRGB(140, 90, 255)
        OStroke.Thickness = 1.5
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
    ContainerBody.Position = UDim2.new(0, 0, 0, 50)
    ContainerBody.Size = UDim2.new(1, 0, 1, -50)

    local TabContainer = Instance.new("Frame")
    TabContainer.Parent = ContainerBody
    TabContainer.BackgroundColor3 = Color3.fromRGB(11, 13, 18)
    TabContainer.Position = UDim2.new(0.03, 0, 0.03, 0)
    TabContainer.Size = UDim2.new(0, 130, 0, 325)
    TabContainer.BorderSizePixel = 0

    local TabCorner = Instance.new("UICorner")
    TabCorner.CornerRadius = UDim.new(0, 12)
    TabCorner.Parent = TabContainer

    local CombatTabBtn = Instance.new("TextButton")
    CombatTabBtn.Parent = TabContainer
    CombatTabBtn.BackgroundColor3 = Color3.fromRGB(130, 80, 255)
    CombatTabBtn.Position = UDim2.new(0.08, 0, 0.04, 0)
    CombatTabBtn.Size = UDim2.new(0.84, 0, 0, 38)
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
    VisualTabBtn.BackgroundColor3 = Color3.fromRGB(18, 21, 30)
    VisualTabBtn.Position = UDim2.new(0.08, 0, 0.16, 0)
    VisualTabBtn.Size = UDim2.new(0.84, 0, 0, 38)
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
    EtcTabBtn.BackgroundColor3 = Color3.fromRGB(18, 21, 30)
    EtcTabBtn.Position = UDim2.new(0.08, 0, 0.28, 0)
    EtcTabBtn.Size = UDim2.new(0.84, 0, 0, 38)
    EtcTabBtn.Font = Enum.Font.GothamBold
    EtcTabBtn.Text = "⚙ Misc"
    EtcTabBtn.TextColor3 = Color3.fromRGB(130, 140, 170)
    EtcTabBtn.TextSize = 12
    EtcTabBtn.BorderSizePixel = 0

    local EtcCorner = Instance.new("UICorner")
    EtcCorner.CornerRadius = UDim.new(0, 8)
    EtcCorner.Parent = EtcTabBtn

    local PagesContainer = Instance.new("Frame")
    PagesContainer.Parent = ContainerBody
    PagesContainer.Position = UDim2.new(0.32, 0, 0.03, 0)
    PagesContainer.Size = UDim2.new(0, 335, 0, 325)
    PagesContainer.BackgroundTransparency = 1

    local CombatPage = Instance.new("ScrollingFrame")
    CombatPage.Parent = PagesContainer
    CombatPage.BackgroundTransparency = 1
    CombatPage.Size = UDim2.new(1, 0, 1, 0)
    CombatPage.CanvasSize = UDim2.new(0, 0, 1.4, 0)
    CombatPage.ScrollBarThickness = 2
    CombatPage.ScrollBarImageColor3 = Color3.fromRGB(130, 80, 255)
    CombatPage.Active = true

    local VisualPage = Instance.new("ScrollingFrame")
    VisualPage.Parent = PagesContainer
    VisualPage.BackgroundTransparency = 1
    VisualPage.Size = UDim2.new(1, 0, 1, 0)
    VisualPage.CanvasSize = UDim2.new(0, 0, 1.4, 0)
    VisualPage.ScrollBarThickness = 2
    VisualPage.ScrollBarImageColor3 = Color3.fromRGB(130, 80, 255)
    VisualPage.Visible = false
    VisualPage.Active = true

    local EtcPage = Instance.new("ScrollingFrame")
    EtcPage.Parent = PagesContainer
    EtcPage.BackgroundTransparency = 1
    EtcPage.Size = UDim2.new(1, 0, 1, 0)
    EtcPage.CanvasSize = UDim2.new(0, 0, 1.4, 0)
    EtcPage.ScrollBarThickness = 2
    EtcPage.ScrollBarImageColor3 = Color3.fromRGB(130, 80, 255)
    EtcPage.Visible = false
    EtcPage.Active = true

    local function switchTab(selectedPage, activeBtn)
        CombatPage.Visible = false
        VisualPage.Visible = false
        EtcPage.Visible = false
        selectedPage.Visible = true

        for _, btn in ipairs({CombatTabBtn, VisualTabBtn, EtcTabBtn}) do
            TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(18, 21, 30), TextColor3 = Color3.fromRGB(130, 140, 170)}):Play()
        end
        TweenService:Create(activeBtn, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(130, 80, 255), TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
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
                local hue = (tick() * 0.25) % 1
                local colorRgb = Color3.fromHSV(hue, 0.65, 1)
                MainStroke.Color = colorRgb
                TopGlow.BackgroundColor3 = colorRgb
            end)
            task.wait(0.1)
        end
    end)

    local function createToggle(parent, name, posY, initialVal, callback)
        local ToggleBtn = Instance.new("TextButton")
        ToggleBtn.Parent = parent
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(18, 21, 30)
        ToggleBtn.Position = UDim2.new(0.02, 0, 0, posY)
        ToggleBtn.Size = UDim2.new(0.96, 0, 0, 44)
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
        StatusIndicator.BackgroundColor3 = initialVal and Color3.fromRGB(0, 230, 255) or Color3.fromRGB(32, 38, 55)
        StatusIndicator.Position = UDim2.new(0.82, 0, 0.28, 0)
        StatusIndicator.Size = UDim2.new(0, 36, 0, 20)

        local SCorner = Instance.new("UICorner")
        SCorner.CornerRadius = UDim.new(0, 10)
        SCorner.Parent = StatusIndicator

        local Dot = Instance.new("Frame")
        Dot.Parent = StatusIndicator
        Dot.BackgroundColor3 = initialVal and Color3.fromRGB(15, 17, 23) or Color3.fromRGB(140, 150, 180)
        Dot.Position = initialVal and UDim2.new(0.48, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0)
        Dot.Size = UDim2.new(0, 16, 0, 16)

        local DCorner = Instance.new("UICorner")
        DCorner.CornerRadius = UDim.new(0, 8)
        DCorner.Parent = Dot

        local enabled = initialVal
        local function triggerToggle()
            enabled = not enabled
            if enabled then
                TweenService:Create(StatusIndicator, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(0, 230, 255)}):Play()
                TweenService:Create(Dot, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(15, 17, 23)}):Play()
                Dot:TweenPosition(UDim2.new(0.48, 0, 0.1, 0), "Out", "Quad", 0.2, true)
            else
                TweenService:Create(StatusIndicator, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(32, 38, 55)}):Play()
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
        SliderFrame.BackgroundColor3 = Color3.fromRGB(18, 21, 30)
        SliderFrame.Position = UDim2.new(0.02, 0, 0, posY)
        SliderFrame.Size = UDim2.new(0.96, 0, 0, 58)
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
        ValueLabel.TextColor3 = Color3.fromRGB(0, 230, 255)
        ValueLabel.TextSize = 12
        ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
        ValueLabel.Font = Enum.Font.GothamBold

        local Bar = Instance.new("Frame")
        Bar.Parent = SliderFrame
        Bar.BackgroundColor3 = Color3.fromRGB(28, 33, 48)
        Bar.Position = UDim2.new(0.05, 0, 0.68, 0)
        Bar.Size = UDim2.new(0.9, 0, 0, 6)

        local BarCorner = Instance.new("UICorner")
        BarCorner.CornerRadius = UDim.new(0, 3)
        BarCorner.Parent = Bar

        local Fill = Instance.new("Frame")
        Fill.Parent = Bar
        Fill.BackgroundColor3 = Color3.fromRGB(130, 80, 255)
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

    createToggle(CombatPage, "Hitbox Cabeza Enemigo", 10, headHitboxEnabled, function(state)
        headHitboxEnabled = state
        if not state then restoreHeads() end
    end)
    
    createSlider(CombatPage, "Tamaño Cabeza (Max 45)", 2, 45, headHitboxSize, 65, function(val) 
        headHitboxSize = val 
    end)

    if not isMobileDevice then
        createToggle(CombatPage, "Aimbot Directo (Click Dcho)", 135, aimbotEnabled, function(state) aimbotEnabled = state end)
    end

    createToggle(VisualPage, "ESP Neon Box", 10, espBoxEnabled, function(state) espBoxEnabled = state end)
    createToggle(VisualPage, "ESP Name", 65, espNameEnabled, function(state) espNameEnabled = state end)
    createToggle(VisualPage, "ESP Cuerpo Azul", 120, espBodyEnabled, function(state) espBodyEnabled = state end)
    createSlider(VisualPage, "ESP Distancia", 50, 1000, espDistance, 180, function(val) espDistance = val end)

    createToggle(EtcPage, "Fly (Volar)", 10, flyEnabled, function(state) flyEnabled = state end)
    createSlider(EtcPage, "Velocidad Fly", 10, 300, flySpeed, 65, function(val) flySpeed = val end)

    createToggle(EtcPage, "Activar Speed (Velocidad)", 135, speedEnabled, function(state) 
        speedEnabled = state 
        if not state then
            pcall(function()
                local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
                if hum then hum.WalkSpeed = 16 end
            end)
        end
    end)
    createSlider(EtcPage, "WalkSpeed Valor", 16, 200, walkSpeedValue, 190, function(val) walkSpeedValue = val end)
end

iniciarMenuPrincipal()

-- =========================================================================
-- LOGICA DE FLY Y SPEED
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
-- HITBOX ANTITRABA / CERO PESO + ESP UNIVERSAL REPARADO (MÓVIL & PC)
-- =========================================================================
table.insert(connections, RunService.RenderStepped:Connect(function()
    updateClosestHead()

    -- 1. GESTIÓN DE HITBOX Y PESO DE CABEZAS
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
                        head.Color = Color3.fromRGB(140, 90, 255)
                        head.Material = Enum.Material.Neon
                        head.CanCollide = false
                        head.Massless = true

                        -- Evita que la cabeza pese, se hunda o trabe al contrincante
                        local bp = head:FindFirstChild("ElianHitboxBP")
                        if not bp then
                            bp = Instance.new("BodyPosition")
                            bp.Name = "ElianHitboxBP"
                            bp.MaxForce = Vector3.new(5000, 5000, 5000)
                            bp.P = 4000
                            bp.Parent = head
                        end
                        bp.Position = hrp.Position + Vector3.new(0, 2, 0)

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

    -- 2. AIMBOT (SOLO PC)
    if aimbotEnabled and isRightClicking and currentTargetHead then
        pcall(function()
            camera.CFrame = CFrame.new(camera.CFrame.Position, currentTargetHead.Position)
        end)
    end

    -- 3. SISTEMA ESP REPARADO PARA MÓVIL Y PC (BILLBOARDS Y HIGHLIGHTS ASEGURADOS)
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
                        -- ESP Cuerpo (Highlight Azul)
                        if espBodyEnabled then
                            if not espBodies[otherPlayer] then
                                local hl = Instance.new("Highlight")
                                hl.Name = "ElianESP_HL"
                                hl.FillColor = Color3.fromRGB(0, 180, 255)
                                hl.OutlineColor = Color3.fromRGB(255, 255, 255)
                                hl.FillTransparency = 0.5
                                hl.OutlineTransparency = 0
                                hl.Adornee = char
                                hl.Parent = char
                                espBodies[otherPlayer] = hl
                            else
                                espBodies[otherPlayer].Adornee = char
                            end
                        elseif espBodies[otherPlayer] then
                            espBodies[otherPlayer]:Destroy()
                            espBodies[otherPlayer] = nil
                        end

                        -- ESP Nombre y Distancia Adaptado para Móvil
                        if espNameEnabled then
                            if not espNames[otherPlayer] then
                                local bGui = Instance.new("BillboardGui")
                                bGui.Name = "ElianESP_Name"
                                bGui.Size = UDim2.new(0, 200, 0, 40)
                                bGui.AlwaysOnTop = true
                                bGui.Adornee = head
                                bGui.StudsOffset = Vector3.new(0, 2.5, 0)
                                bGui.Parent = char
                                
                                local txt = Instance.new("TextLabel")
                                txt.Name = "EspText"
                                txt.Parent = bGui
                                txt.Size = UDim2.new(1, 0, 1, 0)
                                txt.BackgroundTransparency = 1
                                txt.TextColor3 = Color3.fromRGB(0, 230, 255)
                                txt.Font = Enum.Font.GothamBold
                                txt.TextSize = 13
                                txt.TextStrokeTransparency = 0.4
                                espNames[otherPlayer] = bGui
                            else
                                local bGui = espNames[otherPlayer]
                                bGui.Adornee = head
                                local txt = bGui:FindFirstChild("EspText")
                                if txt then
                                    txt.Text = otherPlayer.Name .. " [" .. math.floor(dist) .. "m]"
                                end
                            end
                        elseif espNames[otherPlayer] then
                            espNames[otherPlayer]:Destroy()
                            espNames[otherPlayer] = nil
                        end
                    else
                        if espBodies[otherPlayer] then
                            espBodies[otherPlayer]:Destroy()
                            espBodies[otherPlayer] = nil
                        end
                        if espNames[otherPlayer] then
                            espNames[otherPlayer]:Destroy()
                            espNames[otherPlayer] = nil
                        end
                    end
                else
                    if espBodies[otherPlayer] then
                        espBodies[otherPlayer]:Destroy()
                        espBodies[otherPlayer] = nil
                    end
                    if espNames[otherPlayer] then
                        espNames[otherPlayer]:Destroy()
                        espNames[otherPlayer] = nil
                    end
                end
            end)
        else
            if espBodies[otherPlayer] then
                pcall(function() espBodies[otherPlayer]:Destroy() end)
                espBodies[otherPlayer] = nil
            end
            if espNames[otherPlayer] then
                pcall(function() espNames[otherPlayer]:Destroy() end)
                espNames[otherPlayer] = nil
            end
        end
    end
end))
