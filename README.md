-- =========================================================================
-- SCRIPS FREE - TAMANO EQUILIBRADO (PC / MOVIL) Y AIMBOT
-- =========================================================================

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local TeleportService = game:GetService("TeleportService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local function getGuiParent()
    local ok, res = pcall(function()
        if typeof(gethui) == "function" then
            return gethui()
        end
        return CoreGui
    end)
    if ok and res then
        return res
    end
    return LocalPlayer:WaitForChild("PlayerGui")
end

local GuiParent = getGuiParent()
local DISCORD_LINK = "https://discord.gg/rVDK6E5KC"

local Palette = {
    BgDark = Color3.fromRGB(9, 13, 23),
    BgPanel = Color3.fromRGB(15, 22, 38),
    BgItem = Color3.fromRGB(22, 32, 54),
    Blue = Color3.fromRGB(45, 138, 255),
    BlueBright = Color3.fromRGB(92, 172, 255),
    BlueDeep = Color3.fromRGB(22, 66, 138),
    TextWhite = Color3.fromRGB(242, 247, 255),
    TextSub = Color3.fromRGB(152, 174, 212),
    Border = Color3.fromRGB(36, 62, 108)
}

local function limpiarTodoAnterior()
    for _, n in ipairs({
        "ScripsFreeMainGui", "ScripsFreeFloatGui", "ScripsFreeToastGui",
        "ScripsFreeMobileAimGui", "ScripsFreeUI", "ScripsFreeFloat", "ScripsFreeToast"
    }) do
        pcall(function()
            local g = GuiParent:FindFirstChild(n)
            if g then g:Destroy() end
        end)
    end
end
limpiarTodoAnterior()

local function detectMobile()
    if UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled then
        return true
    end
    local vp = Camera and Camera.ViewportSize or Vector2.new(1280, 720)
    return UserInputService.TouchEnabled and vp.X < 800
end

local isMobile = detectMobile()

local Config = {
    hitboxEnabled = false,
    hitboxPart = "Head",
    hitboxSize = 10,

    aimbotEnabled = false,
    aimbotFov = 240,
    aimbotSmooth = 0.45,
    aimbotHoldActive = false,
    mobileAimToggle = false,
    showFovCircle = false,

    espBody = false,
    espName = false,
    espTracer = false,
    espDistance = 2500,

    flyEnabled = false,
    flySpeed = 60,
    speedEnabled = false,
    walkSpeed = 24,
    noclipEnabled = false,
    fovEnabled = false,
    fovValue = 70,

    ignored = {}
}

local Connections = {}
local EspBodyMap = {}
local EspNameMap = {}
local EspTracerMap = {}
local HitboxBackup = {}
local FlyBV, FlyBG
local FovDrawing = nil

local function regConn(sig, cb)
    local c = sig:Connect(cb)
    table.insert(Connections, c)
    return c
end

local function toast(title, msg, secs)
    pcall(function()
        local old = GuiParent:FindFirstChild("ScripsFreeToastGui")
        if old then old:Destroy() end

        local gui = Instance.new("ScreenGui")
        gui.Name = "ScripsFreeToastGui"
        gui.ResetOnSpawn = false
        gui.IgnoreGuiInset = true
        gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        gui.Parent = GuiParent

        local box = Instance.new("Frame")
        box.Size = UDim2.new(0, 260, 0, 58)
        box.Position = UDim2.new(1, 20, 1, -80)
        box.BackgroundColor3 = Palette.BgPanel
        box.BorderSizePixel = 0
        box.Parent = gui

        local cor = Instance.new("UICorner")
        cor.CornerRadius = UDim.new(0, 10)
        cor.Parent = box

        local st = Instance.new("UIStroke")
        st.Color = Palette.Blue
        st.Thickness = 1.4
        st.Parent = box

        local t1 = Instance.new("TextLabel")
        t1.Size = UDim2.new(1, -16, 0, 20)
        t1.Position = UDim2.new(0, 12, 0, 6)
        t1.BackgroundTransparency = 1
        t1.Font = Enum.Font.GothamBold
        t1.TextSize = 12
        t1.TextXAlignment = Enum.TextXAlignment.Left
        t1.TextColor3 = Palette.BlueBright
        t1.Text = title
        t1.Parent = box

        local t2 = Instance.new("TextLabel")
        t2.Size = UDim2.new(1, -16, 0, 20)
        t2.Position = UDim2.new(0, 12, 0, 28)
        t2.BackgroundTransparency = 1
        t2.Font = Enum.Font.Gotham
        t2.TextSize = 11
        t2.TextXAlignment = Enum.TextXAlignment.Left
        t2.TextColor3 = Palette.TextWhite
        t2.Text = msg
        t2.Parent = box

        local targetX = 1 - ((box.Size.X.Offset + 16) / math.max((Camera and Camera.ViewportSize.X or 1280), 1))
        TweenService:Create(box, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
            Position = UDim2.new(targetX, 0, 1, -80)
        }):Play()

        task.delay(secs or 2.2, function()
            pcall(function()
                TweenService:Create(box, TweenInfo.new(0.22, Enum.EasingStyle.Quint, Enum.EasingDirection.In), {
                    Position = UDim2.new(1, 20, 1, -80)
                }):Play()
                task.wait(0.24)
                gui:Destroy()
            end)
        end)
    end)
end

local function restorePlayerHitbox(plr)
    pcall(function()
        local b = HitboxBackup[plr]
        if b and b.Part and b.Part.Parent then
            b.Part.Size = b.Size
            b.Part.Transparency = b.Transparency
            b.Part.Color = b.Color
            b.Part.Material = b.Material
            b.Part.CanCollide = b.CanCollide
            b.Part.Massless = b.Massless
            local bp = b.Part:FindFirstChild("ScripsFreeHitAnchor")
            if bp then bp:Destroy() end
        end
        HitboxBackup[plr] = nil
    end)
end

local function restoreAllHitboxes()
    for p in pairs(HitboxBackup) do
        restorePlayerHitbox(p)
    end
end

local function clearAllEsp()
    for k, v in pairs(EspBodyMap) do
        pcall(function() if v then v:Destroy() end end)
        EspBodyMap[k] = nil
    end
    for k, v in pairs(EspNameMap) do
        pcall(function() if v then v:Destroy() end end)
        EspNameMap[k] = nil
    end
    for k, v in pairs(EspTracerMap) do
        pcall(function() if v then v:Remove() end end)
        EspTracerMap[k] = nil
    end
end

local function stopFly()
    pcall(function()
        if FlyBV then FlyBV:Destroy(); FlyBV = nil end
        if FlyBG then FlyBG:Destroy(); FlyBG = nil end
        local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum.PlatformStand = false end
    end)
end

local function shutdownScript()
    Config.hitboxEnabled = false
    Config.aimbotEnabled = false
    Config.showFovCircle = false
    Config.espBody = false
    Config.espName = false
    Config.espTracer = false
    Config.flyEnabled = false
    Config.speedEnabled = false
    Config.noclipEnabled = false
    Config.fovEnabled = false

    restoreAllHitboxes()
    clearAllEsp()
    stopFly()

    pcall(function()
        if FovDrawing then
            FovDrawing:Remove()
            FovDrawing = nil
        end
        local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum.WalkSpeed = 16 end
        if Camera then Camera.FieldOfView = 70 end
    end)

    for _, c in ipairs(Connections) do
        pcall(function() c:Disconnect() end)
    end
    table.clear(Connections)
    limpiarTodoAnterior()
end

regConn(UserInputService.InputBegan, function(input, gpe)
    if gpe then return end
    if input.UserInputType == Enum.UserInputType.MouseButton2 or input.KeyCode == Enum.KeyCode.LeftAlt then
        Config.aimbotHoldActive = true
    end
end)

regConn(UserInputService.InputEnded, function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 or input.KeyCode == Enum.KeyCode.LeftAlt then
        Config.aimbotHoldActive = false
    end
end)

local function crearBotonAimMovil()
    if not isMobile then return end
    if GuiParent:FindFirstChild("ScripsFreeMobileAimGui") then return end

    local mGui = Instance.new("ScreenGui")
    mGui.Name = "ScripsFreeMobileAimGui"
    mGui.ResetOnSpawn = false
    mGui.IgnoreGuiInset = true
    mGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    mGui.Parent = GuiParent

    local aimBtn = Instance.new("TextButton")
    aimBtn.Size = UDim2.new(0, 82, 0, 36)
    aimBtn.Position = UDim2.new(0.8, 0, 0.58, 0)
    aimBtn.BackgroundColor3 = Palette.BgPanel
    aimBtn.BorderSizePixel = 0
    aimBtn.Font = Enum.Font.GothamBold
    aimBtn.TextSize = 11
    aimBtn.TextColor3 = Palette.TextWhite
    aimBtn.Text = "Aim: OFF"
    aimBtn.Active = true
    aimBtn.Draggable = true
    aimBtn.Parent = mGui

    local cor = Instance.new("UICorner")
    cor.CornerRadius = UDim.new(0, 9)
    cor.Parent = aimBtn

    local st = Instance.new("UIStroke")
    st.Color = Palette.Blue
    st.Thickness = 1.4
    st.Parent = aimBtn

    aimBtn.MouseButton1Click:Connect(function()
        Config.mobileAimToggle = not Config.mobileAimToggle
        aimBtn.Text = Config.mobileAimToggle and "Aim: ON" or "Aim: OFF"
        TweenService:Create(aimBtn, TweenInfo.new(0.15), {
            BackgroundColor3 = Config.mobileAimToggle and Palette.Blue or Palette.BgPanel
        }):Play()
    end)
end
crearBotonAimMovil()

local function getTargetPart(char)
    if not char then return nil end
    if Config.hitboxPart == "Head" then
        return char:FindFirstChild("Head") or char:FindFirstChild("FakeHead")
    end
    return char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso") or char:FindFirstChild("HumanoidRootPart")
end

-- =========================================================================
-- VENTANA PRINCIPAL (TAMANO NORMAL / EQUILIBRADO)
-- =========================================================================
local MainGui = Instance.new("ScreenGui")
MainGui.Name = "ScripsFreeMainGui"
MainGui.ResetOnSpawn = false
MainGui.IgnoreGuiInset = true
MainGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
MainGui.Parent = GuiParent

local scaleObj = Instance.new("UIScale")
scaleObj.Scale = isMobile and 0.78 or 1.0
scaleObj.Parent = MainGui

local baseW = 580
local baseH = 420

local Window = Instance.new("Frame")
Window.Name = "Window"
Window.Size = UDim2.new(0, baseW, 0, baseH)
Window.Position = UDim2.new(0.5, -baseW / 2, 0.5, -baseH / 2)
Window.BackgroundColor3 = Palette.BgDark
Window.BorderSizePixel = 0
Window.Active = true
Window.Draggable = true
Window.ClipsDescendants = true
Window.Parent = MainGui

local wCorner = Instance.new("UICorner")
wCorner.CornerRadius = UDim.new(0, 14)
wCorner.Parent = Window

local wStroke = Instance.new("UIStroke")
wStroke.Color = Palette.Blue
wStroke.Thickness = 1.5
wStroke.Parent = Window

Window.Size = UDim2.new(0, baseW * 0.94, 0, baseH * 0.94)
TweenService:Create(Window, TweenInfo.new(0.24, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
    Size = UDim2.new(0, baseW, 0, baseH)
}):Play()

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 46)
Header.BackgroundColor3 = Palette.BgPanel
Header.BorderSizePixel = 0
Header.Parent = Window

local hCorner = Instance.new("UICorner")
hCorner.CornerRadius = UDim.new(0, 14)
hCorner.Parent = Header

local hFix = Instance.new("Frame")
hFix.Size = UDim2.new(1, 0, 0, 12)
hFix.Position = UDim2.new(0, 0, 1, -12)
hFix.BackgroundColor3 = Palette.BgPanel
hFix.BorderSizePixel = 0
hFix.Parent = Header

local BrandLbl = Instance.new("TextLabel")
BrandLbl.Size = UDim2.new(0.45, 0, 1, 0)
BrandLbl.Position = UDim2.new(0, 16, 0, 0)
BrandLbl.BackgroundTransparency = 1
BrandLbl.Font = Enum.Font.GothamBold
BrandLbl.TextSize = 15
BrandLbl.TextXAlignment = Enum.TextXAlignment.Left
BrandLbl.TextColor3 = Palette.TextWhite
BrandLbl.Text = "Scrips Free"
BrandLbl.Parent = Header

local DiscordBtn = Instance.new("TextButton")
DiscordBtn.Size = UDim2.new(0, 92, 0, 30)
DiscordBtn.Position = UDim2.new(1, -172, 0.5, -15)
DiscordBtn.BackgroundColor3 = Palette.Blue
DiscordBtn.BorderSizePixel = 0
DiscordBtn.Font = Enum.Font.GothamBold
DiscordBtn.TextSize = 12
DiscordBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
DiscordBtn.Text = "Discord"
DiscordBtn.AutoButtonColor = false
DiscordBtn.Parent = Header

local dCor = Instance.new("UICorner")
dCor.CornerRadius = UDim.new(0, 8)
dCor.Parent = DiscordBtn

regConn(DiscordBtn.MouseEnter, function()
    TweenService:Create(DiscordBtn, TweenInfo.new(0.14), {BackgroundColor3 = Palette.BlueBright}):Play()
end)
regConn(DiscordBtn.MouseLeave, function()
    TweenService:Create(DiscordBtn, TweenInfo.new(0.14), {BackgroundColor3 = Palette.Blue}):Play()
end)

DiscordBtn.MouseButton1Click:Connect(function()
    pcall(function()
        if setclipboard then
            setclipboard(DISCORD_LINK)
        end
    end)
    DiscordBtn.Text = "Copiado"
    toast("Scrips Free", "Enlace copiado: " .. DISCORD_LINK, 2.2)
    task.delay(1.3, function()
        if DiscordBtn and DiscordBtn.Parent then
            DiscordBtn.Text = "Discord"
        end
    end)
end)

local MinBtn = Instance.new("TextButton")
MinBtn.Size = UDim2.new(0, 30, 0, 30)
MinBtn.Position = UDim2.new(1, -72, 0.5, -15)
MinBtn.BackgroundColor3 = Palette.BgItem
MinBtn.BorderSizePixel = 0
MinBtn.Font = Enum.Font.GothamBold
MinBtn.TextSize = 14
MinBtn.TextColor3 = Palette.TextWhite
MinBtn.Text = "-"
MinBtn.Parent = Header

local mCor = Instance.new("UICorner")
mCor.CornerRadius = UDim.new(0, 8)
mCor.Parent = MinBtn

local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.new(0, 30, 0, 30)
CloseBtn.Position = UDim2.new(1, -36, 0.5, -15)
CloseBtn.BackgroundColor3 = Palette.BlueDeep
CloseBtn.BorderSizePixel = 0
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.TextSize = 12
CloseBtn.TextColor3 = Palette.TextWhite
CloseBtn.Text = "X"
CloseBtn.Parent = Header

local cCor = Instance.new("UICorner")
cCor.CornerRadius = UDim.new(0, 8)
cCor.Parent = CloseBtn

local function showRestoreButton()
    if GuiParent:FindFirstChild("ScripsFreeFloatGui") then return end
    local fGui = Instance.new("ScreenGui")
    fGui.Name = "ScripsFreeFloatGui"
    fGui.ResetOnSpawn = false
    fGui.IgnoreGuiInset = true
    fGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    fGui.Parent = GuiParent

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 114, 0, 38)
    btn.Position = UDim2.new(0.03, 0, 0.38, 0)
    btn.BackgroundColor3 = Palette.Blue
    btn.BorderSizePixel = 0
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 12
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Text = "Scrips Free"
    btn.Active = true
    btn.Draggable = true
    btn.Parent = fGui

    local bCor = Instance.new("UICorner")
    bCor.CornerRadius = UDim.new(0, 9)
    bCor.Parent = btn

    btn.MouseButton1Click:Connect(function()
        Window.Visible = true
        Window.Size = UDim2.new(0, baseW * 0.94, 0, baseH * 0.94)
        TweenService:Create(Window, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Size = UDim2.new(0, baseW, 0, baseH)
        }):Play()
        fGui:Destroy()
    end)
end

MinBtn.MouseButton1Click:Connect(function()
    local tw = TweenService:Create(Window, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
        Size = UDim2.new(0, baseW * 0.88, 0, 10)
    })
    tw:Play()
    tw.Completed:Wait()
    Window.Visible = false
    showRestoreButton()
end)

CloseBtn.MouseButton1Click:Connect(shutdownScript)

regConn(UserInputService.InputBegan, function(input, gpe)
    if not gpe and input.KeyCode == Enum.KeyCode.RightShift then
        if Window.Visible then
            Window.Visible = false
            showRestoreButton()
        else
            local fl = GuiParent:FindFirstChild("ScripsFreeFloatGui")
            if fl then fl:Destroy() end
            Window.Visible = true
        end
    end
end)

local navW = 132
local NavFrame = Instance.new("Frame")
NavFrame.Size = UDim2.new(0, navW, 1, -60)
NavFrame.Position = UDim2.new(0, 10, 0, 52)
NavFrame.BackgroundColor3 = Palette.BgPanel
NavFrame.BorderSizePixel = 0
NavFrame.Parent = Window

local nCor = Instance.new("UICorner")
nCor.CornerRadius = UDim.new(0, 11)
nCor.Parent = NavFrame

local nStroke = Instance.new("UIStroke")
nStroke.Color = Palette.Border
nStroke.Thickness = 1
nStroke.Parent = NavFrame

local navLayout = Instance.new("UIListLayout")
navLayout.Padding = UDim.new(0, 7)
navLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
navLayout.Parent = NavFrame

local navPad = Instance.new("UIPadding")
navPad.PaddingTop = UDim.new(0, 10)
navPad.Parent = NavFrame

local ContentHolder = Instance.new("Frame")
ContentHolder.Size = UDim2.new(1, -(navW + 24), 1, -60)
ContentHolder.Position = UDim2.new(0, navW + 16, 0, 52)
ContentHolder.BackgroundTransparency = 1
ContentHolder.Parent = Window

local Pages = {}
local NavButtons = {}

local function makePage(name, idx)
    local page = Instance.new("ScrollingFrame")
    page.Name = name
    page.Size = UDim2.new(1, 0, 1, 0)
    page.BackgroundTransparency = 1
    page.BorderSizePixel = 0
    page.ScrollBarThickness = 4
    page.ScrollBarImageColor3 = Palette.Blue
    page.CanvasSize = UDim2.new(0, 0, 0, 500)
    page.Visible = (idx == 1)
    page.Parent = ContentHolder
    Pages[name] = page

    local list = Instance.new("UIListLayout")
    list.Padding = UDim.new(0, 7)
    list.HorizontalAlignment = Enum.HorizontalAlignment.Center
    list.Parent = page

    local pad = Instance.new("UIPadding")
    pad.PaddingRight = UDim.new(0, 6)
    pad.PaddingBottom = UDim.new(0, 10)
    pad.Parent = page

    regConn(list:GetPropertyChangedSignal("AbsoluteContentSize"), function()
        page.CanvasSize = UDim2.new(0, 0, 0, list.AbsoluteContentSize.Y + 18)
    end)

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.88, 0, 0, 36)
    btn.BackgroundColor3 = (idx == 1) and Palette.Blue or Palette.BgItem
    btn.BorderSizePixel = 0
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 12
    btn.TextColor3 = (idx == 1) and Color3.fromRGB(255, 255, 255) or Palette.TextSub
    btn.Text = name
    btn.AutoButtonColor = false
    btn.Parent = NavFrame
    NavButtons[name] = btn

    local bCor = Instance.new("UICorner")
    bCor.CornerRadius = UDim.new(0, 8)
    bCor.Parent = btn

    btn.MouseButton1Click:Connect(function()
        for pName, pFrame in pairs(Pages) do
            local active = (pName == name)
            pFrame.Visible = active
            if active then
                pFrame.Position = UDim2.new(0, 10, 0, 0)
                TweenService:Create(pFrame, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                    Position = UDim2.new(0, 0, 0, 0)
                }):Play()
            end
            local nb = NavButtons[pName]
            if nb then
                TweenService:Create(nb, TweenInfo.new(0.14), {
                    BackgroundColor3 = active and Palette.Blue or Palette.BgItem,
                    TextColor3 = active and Color3.fromRGB(255, 255, 255) or Palette.TextSub
                }):Play()
            end
        end
    end)

    return page
end

local function makeToggle(parent, labelText, initial, cb)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 40)
    row.BackgroundColor3 = Palette.BgPanel
    row.BorderSizePixel = 0
    row.Parent = parent

    local rCor = Instance.new("UICorner")
    rCor.CornerRadius = UDim.new(0, 9)
    rCor.Parent = row

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -72, 1, 0)
    lbl.Position = UDim2.new(0, 12, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Font = Enum.Font.GothamMedium
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.TextColor3 = Palette.TextWhite
    lbl.Text = labelText
    lbl.Parent = row

    local sw = Instance.new("TextButton")
    sw.Size = UDim2.new(0, 44, 0, 24)
    sw.Position = UDim2.new(1, -56, 0.5, -12)
    sw.BackgroundColor3 = initial and Palette.Blue or Palette.BgItem
    sw.BorderSizePixel = 0
    sw.Text = ""
    sw.AutoButtonColor = false
    sw.Parent = row

    local swCor = Instance.new("UICorner")
    swCor.CornerRadius = UDim.new(1, 0)
    swCor.Parent = sw

    local dot = Instance.new("Frame")
    dot.Size = UDim2.new(0, 18, 0, 18)
    dot.Position = initial and UDim2.new(1, -21, 0.5, -9) or UDim2.new(0, 3, 0.5, -9)
    dot.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    dot.BorderSizePixel = 0
    dot.Parent = sw

    local dCor = Instance.new("UICorner")
    dCor.CornerRadius = UDim.new(1, 0)
    dCor.Parent = dot

    local state = initial
    sw.MouseButton1Click:Connect(function()
        state = not state
        TweenService:Create(sw, TweenInfo.new(0.16), {
            BackgroundColor3 = state and Palette.Blue or Palette.BgItem
        }):Play()
        TweenService:Create(dot, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Position = state and UDim2.new(1, -21, 0.5, -9) or UDim2.new(0, 3, 0.5, -9)
        }):Play()
        cb(state)
    end)
end

local function makeSlider(parent, labelText, minV, maxV, initial, cb)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 52)
    row.BackgroundColor3 = Palette.BgPanel
    row.BorderSizePixel = 0
    row.Parent = parent

    local rCor = Instance.new("UICorner")
    rCor.CornerRadius = UDim.new(0, 9)
    rCor.Parent = row

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.68, 0, 0, 20)
    lbl.Position = UDim2.new(0, 12, 0, 6)
    lbl.BackgroundTransparency = 1
    lbl.Font = Enum.Font.GothamMedium
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.TextColor3 = Palette.TextWhite
    lbl.Text = labelText
    lbl.Parent = row

    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(0.25, 0, 0, 20)
    valLbl.Position = UDim2.new(0.72, 0, 0, 6)
    valLbl.BackgroundTransparency = 1
    valLbl.Font = Enum.Font.GothamBold
    valLbl.TextSize = 12
    valLbl.TextXAlignment = Enum.TextXAlignment.Right
    valLbl.TextColor3 = Palette.BlueBright
    valLbl.Text = tostring(initial)
    valLbl.Parent = row

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1, -24, 0, 8)
    bar.Position = UDim2.new(0, 12, 0, 34)
    bar.BackgroundColor3 = Palette.BgItem
    bar.BorderSizePixel = 0
    bar.Parent = row

    local bCor = Instance.new("UICorner")
    bCor.CornerRadius = UDim.new(1, 0)
    bCor.Parent = bar

    local pct = math.clamp((initial - minV) / math.max(maxV - minV, 1), 0, 1)
    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(pct, 0, 1, 0)
    fill.BackgroundColor3 = Palette.Blue
    fill.BorderSizePixel = 0
    fill.Parent = bar

    local fCor = Instance.new("UICorner")
    fCor.CornerRadius = UDim.new(1, 0)
    fCor.Parent = fill

    local dragging = false
    local function setFromInput(input)
        local sc = scaleObj.Scale > 0 and scaleObj.Scale or 1
        local rel = math.clamp((input.Position.X - bar.AbsolutePosition.X) / math.max(bar.AbsoluteSize.X * sc, 1), 0, 1)
        local v = math.floor(minV + ((maxV - minV) * rel) + 0.5)
        fill.Size = UDim2.new(rel, 0, 1, 0)
        valLbl.Text = tostring(v)
        cb(v)
    end

    regConn(bar.InputBegan, function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            setFromInput(input)
        end
    end)

    regConn(UserInputService.InputEnded, function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    regConn(UserInputService.InputChanged, function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            setFromInput(input)
        end
    end)
end

local CombatPage = makePage("Combate", 1)
local EspPage = makePage("ESP", 2)
local PlayersPage = makePage("Jugadores", 3)
local MiscPage = makePage("Misc", 4)

local PartBtn = Instance.new("TextButton")
PartBtn.Size = UDim2.new(1, 0, 0, 38)
PartBtn.BackgroundColor3 = Palette.BgPanel
PartBtn.BorderSizePixel = 0
PartBtn.Font = Enum.Font.GothamBold
PartBtn.TextSize = 12
PartBtn.TextColor3 = Palette.BlueBright
PartBtn.Text = "Objetivo: Head"
PartBtn.Parent = CombatPage

local pBtnCor = Instance.new("UICorner")
pBtnCor.CornerRadius = UDim.new(0, 9)
pBtnCor.Parent = PartBtn

PartBtn.MouseButton1Click:Connect(function()
    restoreAllHitboxes()
    if Config.hitboxPart == "Head" then
        Config.hitboxPart = "Torso"
        PartBtn.Text = "Objetivo: Torso"
    else
        Config.hitboxPart = "Head"
        PartBtn.Text = "Objetivo: Head"
    end
end)

makeToggle(CombatPage, "Aimbot Activo", Config.aimbotEnabled, function(v)
    Config.aimbotEnabled = v
end)

makeToggle(CombatPage, "Mostrar Circulo FOV", Config.showFovCircle, function(v)
    Config.showFovCircle = v
end)

makeSlider(CombatPage, "FOV Aimbot", 60, 600, Config.aimbotFov, function(v)
    Config.aimbotFov = v
end)

makeSlider(CombatPage, "Suavizado Aimbot (%)", 10, 100, math.floor(Config.aimbotSmooth * 100), function(v)
    Config.aimbotSmooth = v / 100
end)

makeToggle(CombatPage, "Hitbox Global", Config.hitboxEnabled, function(v)
    Config.hitboxEnabled = v
    if not v then restoreAllHitboxes() end
end)

makeSlider(CombatPage, "Tamano Hitbox", 2, 25, Config.hitboxSize, function(v)
    Config.hitboxSize = v
end)

makeToggle(EspPage, "ESP Cuerpo", Config.espBody, function(v)
    Config.espBody = v
    if not v then clearAllEsp() end
end)

makeToggle(EspPage, "ESP Nombre y Distancia", Config.espName, function(v)
    Config.espName = v
    if not v then clearAllEsp() end
end)

makeToggle(EspPage, "ESP Tracers", Config.espTracer, function(v)
    Config.espTracer = v
    if not v then clearAllEsp() end
end)

makeSlider(EspPage, "Distancia Maxima ESP", 150, 5000, Config.espDistance, function(v)
    Config.espDistance = v
end)

local function rebuildPlayersList()
    for _, child in ipairs(PlayersPage:GetChildren()) do
        if child:IsA("Frame") then
            child:Destroy()
        end
    end

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            local row = Instance.new("Frame")
            row.Size = UDim2.new(1, 0, 0, 40)
            row.BackgroundColor3 = Palette.BgPanel
            row.BorderSizePixel = 0
            row.Parent = PlayersPage

            local rCor = Instance.new("UICorner")
            rCor.CornerRadius = UDim.new(0, 9)
            rCor.Parent = row

            local lbl = Instance.new("TextLabel")
            lbl.Size = UDim2.new(0.65, 0, 1, 0)
            lbl.Position = UDim2.new(0, 12, 0, 0)
            lbl.BackgroundTransparency = 1
            lbl.Font = Enum.Font.GothamMedium
            lbl.TextSize = 12
            lbl.TextXAlignment = Enum.TextXAlignment.Left
            lbl.TextColor3 = Palette.TextWhite
            lbl.Text = plr.Name
            lbl.Parent = row

            local ign = Config.ignored[plr] == true
            local b = Instance.new("TextButton")
            b.Size = UDim2.new(0, 84, 0, 26)
            b.Position = UDim2.new(1, -96, 0.5, -13)
            b.BackgroundColor3 = ign and Palette.Blue or Palette.BgItem
            b.BorderSizePixel = 0
            b.Font = Enum.Font.GothamBold
            b.TextSize = 11
            b.TextColor3 = Color3.fromRGB(255, 255, 255)
            b.Text = ign and "Ignorado" or "Objetivo"
            b.Parent = row

            local bCor = Instance.new("UICorner")
            bCor.CornerRadius = UDim.new(0, 7)
            bCor.Parent = b

            b.MouseButton1Click:Connect(function()
                local nextState = not (Config.ignored[plr] == true)
                Config.ignored[plr] = nextState or nil
                if nextState then restorePlayerHitbox(plr) end
                b.Text = nextState and "Ignorado" or "Objetivo"
                b.BackgroundColor3 = nextState and Palette.Blue or Palette.BgItem
            end)
        end
    end
end

rebuildPlayersList()
regConn(Players.PlayerAdded, rebuildPlayersList)
regConn(Players.PlayerRemoving, function(plr)
    Config.ignored[plr] = nil
    restorePlayerHitbox(plr)
    pcall(function() if EspBodyMap[plr] then EspBodyMap[plr]:Destroy(); EspBodyMap[plr] = nil end end)
    pcall(function() if EspNameMap[plr] then EspNameMap[plr]:Destroy(); EspNameMap[plr] = nil end end)
    pcall(function() if EspTracerMap[plr] then EspTracerMap[plr]:Remove(); EspTracerMap[plr] = nil end end)
    rebuildPlayersList()
end)

local InfoBox = Instance.new("Frame")
InfoBox.Size = UDim2.new(1, 0, 0, 38)
InfoBox.BackgroundColor3 = Palette.BgPanel
InfoBox.BorderSizePixel = 0
InfoBox.Parent = MiscPage

local iCor = Instance.new("UICorner")
iCor.CornerRadius = UDim.new(0, 9)
iCor.Parent = InfoBox

local InfoLbl = Instance.new("TextLabel")
InfoLbl.Size = UDim2.new(1, -24, 1, 0)
InfoLbl.Position = UDim2.new(0, 12, 0, 0)
InfoLbl.BackgroundTransparency = 1
InfoLbl.Font = Enum.Font.GothamBold
InfoLbl.TextSize = 12
InfoLbl.TextXAlignment = Enum.TextXAlignment.Left
InfoLbl.TextColor3 = Palette.BlueBright
InfoLbl.Text = "FPS: --  |  Ping: -- ms"
InfoLbl.Parent = InfoBox

task.spawn(function()
    while MainGui and MainGui.Parent do
        pcall(function()
            local ping = math.floor(LocalPlayer:GetNetworkPing() * 1000)
            local fps = math.floor(1 / math.max(RunService.RenderStepped:Wait(), 0.001))
            InfoLbl.Text = string.format("FPS: %d  |  Ping: %d ms", fps, ping)
        end)
        task.wait(1)
    end
end)

makeToggle(MiscPage, "Fly", Config.flyEnabled, function(v)
    Config.flyEnabled = v
    if not v then stopFly() end
end)

makeSlider(MiscPage, "Velocidad Fly", 15, 260, Config.flySpeed, function(v)
    Config.flySpeed = v
end)

makeToggle(MiscPage, "Speed", Config.speedEnabled, function(v)
    Config.speedEnabled = v
    if not v then
        pcall(function()
            local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
            if hum then hum.WalkSpeed = 16 end
        end)
    end
end)

makeSlider(MiscPage, "WalkSpeed", 16, 180, Config.walkSpeed, function(v)
    Config.walkSpeed = v
end)

makeToggle(MiscPage, "Noclip", Config.noclipEnabled, function(v)
    Config.noclipEnabled = v
    if not v and LocalPlayer.Character then
        pcall(function()
            for _, p in ipairs(LocalPlayer.Character:GetDescendants()) do
                if p:IsA("BasePart") then p.CanCollide = true end
            end
        end)
    end
end)

makeToggle(MiscPage, "Custom FOV", Config.fovEnabled, function(v)
    Config.fovEnabled = v
    if not v and Camera then Camera.FieldOfView = 70 end
end)

makeSlider(MiscPage, "FOV Camara", 50, 120, Config.fovValue, function(v)
    Config.fovValue = v
end)

local RejoinB = Instance.new("TextButton")
RejoinB.Size = UDim2.new(1, 0, 0, 38)
RejoinB.BackgroundColor3 = Palette.Blue
RejoinB.BorderSizePixel = 0
RejoinB.Font = Enum.Font.GothamBold
RejoinB.TextSize = 12
RejoinB.TextColor3 = Color3.fromRGB(255, 255, 255)
RejoinB.Text = "Rejoin Server"
RejoinB.Parent = MiscPage

local rejCor = Instance.new("UICorner")
rejCor.CornerRadius = UDim.new(0, 9)
rejCor.Parent = RejoinB

RejoinB.MouseButton1Click:Connect(function()
    toast("Scrips Free", "Reconectando...", 2)
    task.wait(0.5)
    pcall(function()
        TeleportService:Teleport(game.PlaceId, LocalPlayer)
    end)
end)

local function shouldRunAimbot()
    if not Config.aimbotEnabled then return false end
    if isMobile then
        return Config.mobileAimToggle
    end
    return Config.aimbotHoldActive
end

local function findAimTarget()
    if not Camera then return nil end
    local vp = Camera.ViewportSize
    local origin2D = isMobile and Vector2.new(vp.X * 0.5, vp.Y * 0.45) or UserInputService:GetMouseLocation()
    local bestPart = nil
    local bestDist = Config.aimbotFov

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and not Config.ignored[plr] then
            local char = plr.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local part = getTargetPart(char)

            if part and hrp and hum and hum.Health > 0 then
                local dist3D = (hrp.Position - Camera.CFrame.Position).Magnitude
                if dist3D <= 1500 then
                    local pos2D, onScreen = Camera:WorldToViewportPoint(part.Position)
                    if onScreen and pos2D.Z > 0 then
                        local dist2D = (Vector2.new(pos2D.X, pos2D.Y) - origin2D).Magnitude
                        if dist2D <= bestDist then
                            bestDist = dist2D
                            bestPart = part
                        end
                    end
                end
            end
        end
    end

    return bestPart
end

regConn(RunService.RenderStepped, function()
    if not MainGui or not MainGui.Parent then return end
    Camera = workspace.CurrentCamera or Camera
    if not Camera then return end

    local vp = Camera.ViewportSize
    local aimCenter = isMobile and Vector2.new(vp.X * 0.5, vp.Y * 0.45) or UserInputService:GetMouseLocation()

    if Drawing then
        if Config.aimbotEnabled and Config.showFovCircle then
            if not FovDrawing then
                FovDrawing = Drawing.new("Circle")
                FovDrawing.Thickness = 1.5
                FovDrawing.NumSides = 48
                FovDrawing.Filled = false
                FovDrawing.Transparency = 0.75
                FovDrawing.Color = Palette.BlueBright
            end
            FovDrawing.Visible = true
            FovDrawing.Radius = Config.aimbotFov
            FovDrawing.Position = aimCenter
        elseif FovDrawing then
            FovDrawing.Visible = false
        end
    end

    if Config.speedEnabled then
        pcall(function()
            local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
            if hum then hum.WalkSpeed = Config.walkSpeed end
        end)
    end

    if Config.noclipEnabled then
        pcall(function()
            local char = LocalPlayer.Character
            if char then
                for _, p in ipairs(char:GetDescendants()) do
                    if p:IsA("BasePart") then p.CanCollide = false end
                end
            end
        end)
    end

    if Config.flyEnabled then
        pcall(function()
            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hrp and hum then
                hum.PlatformStand = true
                if not FlyBV or not FlyBV.Parent then
                    FlyBV = Instance.new("BodyVelocity")
                    FlyBV.MaxForce = Vector3.new(1e6, 1e6, 1e6)
                    FlyBV.Parent = hrp
                end
                if not FlyBG or not FlyBG.Parent then
                    FlyBG = Instance.new("BodyGyro")
                    FlyBG.MaxTorque = Vector3.new(1e6, 1e6, 1e6)
                    FlyBG.P = 1e4
                    FlyBG.Parent = hrp
                end
                FlyBG.CFrame = Camera.CFrame
                local dir = hum.MoveDirection
                FlyBV.Velocity = dir.Magnitude > 0 and (dir.Unit * Config.flySpeed) or Vector3.zero
            end
        end)
    end

    if Config.fovEnabled then
        pcall(function()
            Camera.FieldOfView = Config.fovValue
        end)
    end

    if Config.hitboxEnabled then
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                if Config.ignored[plr] then
                    restorePlayerHitbox(plr)
                else
                    pcall(function()
                        local char = plr.Character
                        local hum = char and char:FindFirstChildOfClass("Humanoid")
                        local hrp = char and char:FindFirstChild("HumanoidRootPart")
                        local part = getTargetPart(char)

                        if part and hum and hum.Health > 0 then
                            if not HitboxBackup[plr] then
                                HitboxBackup[plr] = {
                                    Part = part,
                                    Size = part.Size,
                                    Transparency = part.Transparency,
                                    Color = part.Color,
                                    Material = part.Material,
                                    CanCollide = part.CanCollide,
                                    Massless = part.Massless
                                }
                            end
                            part.Size = Vector3.new(Config.hitboxSize, Config.hitboxSize * 0.85, Config.hitboxSize)
                            part.Transparency = 0.55
                            part.Color = Palette.Blue
                            part.Material = Enum.Material.Neon
                            part.CanCollide = false
                            part.Massless = true

                            if hrp then
                                local bp = part:FindFirstChild("ScripsFreeHitAnchor")
                                if not bp then
                                    bp = Instance.new("BodyPosition")
                                    bp.Name = "ScripsFreeHitAnchor"
                                    bp.MaxForce = Vector3.new(40000, 0, 40000)
                                    bp.Parent = part
                                end
                                bp.Position = hrp.Position
                            end
                        end
                    end)
                end
            end
        end
    end

    if shouldRunAimbot() then
        pcall(function()
            local target = findAimTarget()
            if target then
                local camPos = Camera.CFrame.Position
                local desired = CFrame.lookAt(camPos, target.Position)
                local alpha = math.clamp(Config.aimbotSmooth, 0.05, 1)
                Camera.CFrame = Camera.CFrame:Lerp(desired, alpha)
            end
        end)
    end

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and not Config.ignored[plr] then
            pcall(function()
                local char = plr.Character
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                local head = char and (char:FindFirstChild("Head") or char:FindFirstChild("FakeHead"))
                local anchor = head or hrp

                local valid = char and hum and hum.Health > 0 and anchor and
                    ((anchor.Position - Camera.CFrame.Position).Magnitude <= Config.espDistance)

                if Config.espBody and valid then
                    local hl = EspBodyMap[plr]
                    if not hl then
                        hl = Instance.new("Highlight")
                        hl.Name = "ScripsFreeBodyHL"
                        hl.FillColor = Palette.Blue
                        hl.OutlineColor = Color3.fromRGB(255, 255, 255)
                        hl.FillTransparency = 0.55
                        hl.OutlineTransparency = 0.15
                        hl.Parent = GuiParent
                        EspBodyMap[plr] = hl
                    end
                    hl.Adornee = char
                    hl.Enabled = true
                elseif EspBodyMap[plr] then
                    EspBodyMap[plr]:Destroy()
                    EspBodyMap[plr] = nil
                end

                if Config.espName and valid then
                    local bb = EspNameMap[plr]
                    if not bb then
                        bb = Instance.new("BillboardGui")
                        bb.Name = "ScripsFreeNameBB"
                        bb.Size = UDim2.new(0, 140, 0, 30)
                        bb.StudsOffset = Vector3.new(0, 2.5, 0)
                        bb.AlwaysOnTop = true
                        bb.Parent = GuiParent

                        local txt = Instance.new("TextLabel")
                        txt.Name = "Txt"
                        txt.Size = UDim2.new(1, 0, 1, 0)
                        txt.BackgroundTransparency = 1
                        txt.Font = Enum.Font.GothamBold
                        txt.TextSize = 12
                        txt.TextColor3 = Color3.fromRGB(255, 255, 255)
                        txt.TextStrokeTransparency = 0.25
                        txt.Parent = bb
                        EspNameMap[plr] = bb
                    end
                    bb.Adornee = anchor
                    local d = math.floor((anchor.Position - Camera.CFrame.Position).Magnitude)
                    local t = bb:FindFirstChild("Txt")
                    if t then t.Text = plr.Name .. " [" .. d .. "m]" end
                elseif EspNameMap[plr] then
                    EspNameMap[plr]:Destroy()
                    EspNameMap[plr] = nil
                end

                if Drawing and Config.espTracer and valid then
                    local ln = EspTracerMap[plr]
                    if not ln then
                        ln = Drawing.new("Line")
                        ln.Thickness = 1.5
                        ln.Transparency = 0.8
                        ln.Color = Palette.BlueBright
                        EspTracerMap[plr] = ln
                    end
                    local p2, vis = Camera:WorldToViewportPoint(anchor.Position)
                    if vis and p2.Z > 0 then
                        ln.From = Vector2.new(vp.X * 0.5, vp.Y)
                        ln.To = Vector2.new(p2.X, p2.Y)
                        ln.Visible = true
                    else
                        ln.Visible = false
                    end
                elseif EspTracerMap[plr] then
                    EspTracerMap[plr]:Remove()
                    EspTracerMap[plr] = nil
                end
            end)
        else
            pcall(function() if EspBodyMap[plr] then EspBodyMap[plr]:Destroy(); EspBodyMap[plr] = nil end end)
            pcall(function() if EspNameMap[plr] then EspNameMap[plr]:Destroy(); EspNameMap[plr] = nil end end)
            pcall(function() if EspTracerMap[plr] then EspTracerMap[plr]:Remove(); EspTracerMap[plr] = nil end end)
        end
    end
end)

toast("Scrips Free", "Listo con tamano normal para PC y movil.", 2.2)
