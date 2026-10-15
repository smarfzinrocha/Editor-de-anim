--[[
═══════════════════════════════════════════════════════════════
                    ANIM EDIT v1.0
          Editor de Animações + Botões Flutuantes
                  Para Delta / Executors
═══════════════════════════════════════════════════════════════

TUTORIAL RÁPIDO:
1. Execute o script → aparece a janela principal (arrastável).
2. Clique no "−" para minimizar (vira bolinha flutuante arrastável).
3. Aba "Animações": cole IDs de animações/emotes para Idle, Walk, Run, Jump etc.
   (deixe em branco o que não quiser alterar).
4. Aba "Botões": crie botões flutuantes personalizados.
   - Digite o texto (quanto maior o texto, menor o TextSize automaticamente).
   - Defina se tem UIStroke / UICorner e a cor do stroke.
   - Cole o ID da animação/emote que o botão vai tocar.
   - Marque em quais animações o ID se aplica (pode ser mais de uma).
   - Marque se o botão será arrastável.
5. Botão FINALIZAR: copia para a área de transferência um script
   novo pronto para usar (com todos os IDs e botões que você configurou).

Dicas:
- Para evitar conflito arraste × clique: o botão só clica se você
  soltar o mouse sem ter movido muito.
- Emotes de "PlayOnce" tocam uma vez. Os outros substituem a animação
  correspondente no Animate do personagem.
═══════════════════════════════════════════════════════════════
]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Mouse = LocalPlayer:GetMouse()

-- ============================================================
-- CONFIGURAÇÃO E DADOS
-- ============================================================
local ANIM_TYPES = {
    "PlayOnce", "Idle", "Walk", "Run", "Jump", "Fall", "Climb", "Swim", "SwimIdle"
}

local DEFAULT_ANIM_PATHS = {
    Idle     = {"idle", "Animation1"},
    Walk     = {"walk", "WalkAnim"},
    Run      = {"run", "RunAnim"},
    Jump     = {"jump", "JumpAnim"},
    Fall     = {"fall", "FallAnim"},
    Climb    = {"climb", "ClimbAnim"},
    Swim     = {"swim", "Swim"},
    SwimIdle = {"swimidle", "SwimIdle"},
}

-- Dados que serão exportados
local Data = {
    AnimIDs = {
        Idle = "", Walk = "", Run = "", Jump = "",
        Fall = "", Climb = "", Swim = "", SwimIdle = ""
    },
    Buttons = {}, -- {text, size, pos, stroke, corner, strokeColor, animMap, draggable}
}

-- ============================================================
-- UTILITÁRIOS
-- ============================================================
local function Create(class, props)
    local inst = Instance.new(class)
    for k, v in pairs(props or {}) do
        if k ~= "Parent" then
            inst[k] = v
        end
    end
    if props and props.Parent then
        inst.Parent = props.Parent
    end
    return inst
end

local function AddCorner(parent, radius)
    return Create("UICorner", {
        CornerRadius = UDim.new(0, radius or 8),
        Parent = parent
    })
end

local function AddStroke(parent, color, thickness)
    return Create("UIStroke", {
        Color = color or Color3.fromRGB(120, 80, 255),
        Thickness = thickness or 1.5,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
        Parent = parent
    })
end

local function MakeDraggable(frame, dragHandle)
    dragHandle = dragHandle or frame
    local dragging, dragStart, startPos
    local dragThreshold = 6

    local function update(input)
        local delta = input.Position - dragStart
        frame.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + delta.X,
            startPos.Y.Scale, startPos.Y.Offset + delta.Y
        )
    end

    dragHandle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    dragHandle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            if dragging then
                update(input)
            end
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            if dragging then
                update(input)
            end
        end
    end)
end

-- Clique seguro (não dispara se arrastou)
local function SafeClick(button, callback)
    local startPos, moved = nil, false
    local threshold = 8

    button.MouseButton1Down:Connect(function()
        startPos = Vector2.new(Mouse.X, Mouse.Y)
        moved = false
    end)

    button.MouseMoved:Connect(function()
        if startPos then
            local dist = (Vector2.new(Mouse.X, Mouse.Y) - startPos).Magnitude
            if dist > threshold then
                moved = true
            end
        end
    end)

    button.MouseButton1Up:Connect(function()
        if startPos and not moved then
            callback()
        end
        startPos = nil
        moved = false
    end)
end

local function AutoTextSize(button, maxSize)
    maxSize = maxSize or 16
    local text = button.Text
    local size = button.AbsoluteSize
    local len = #text
    if len <= 4 then
        button.TextSize = maxSize
    elseif len <= 8 then
        button.TextSize = math.clamp(maxSize - 2, 10, maxSize)
    elseif len <= 14 then
        button.TextSize = math.clamp(maxSize - 4, 9, maxSize)
    else
        button.TextSize = math.clamp(maxSize - 6, 8, maxSize)
    end
end

local function Notify(msg)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = "Anim Edit v1.0",
            Text = msg,
            Duration = 4
        })
    end)
end

-- ============================================================
-- APLICAR ANIMAÇÕES NO PERSONAGEM
-- ============================================================
local function ApplyAnimIDs()
    local char = LocalPlayer.Character
    if not char then return end
    local animate = char:FindFirstChild("Animate")
    if not animate then return end

    for name, id in pairs(Data.AnimIDs) do
        if id and id ~= "" and DEFAULT_ANIM_PATHS[name] then
            local folderName, animName = unpack(DEFAULT_ANIM_PATHS[name])
            local folder = animate:FindFirstChild(folderName)
            if folder then
                local anim = folder:FindFirstChild(animName)
                if anim and anim:IsA("Animation") then
                    anim.AnimationId = "rbxassetid://" .. tostring(id):gsub("%D", "")
                end
            end
        end
    end
end

-- animMap: { ["Idle"] = "123", ["Walk"] = "456", ["PlayOnce"] = "789" }
local function PlayAnimation(animMap)
    local char = LocalPlayer.Character
    if not char then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    local animator = humanoid:FindFirstChildOfClass("Animator") or humanoid

    if type(animMap) ~= "table" then return end

    local didReplace = false

    for animType, animId in pairs(animMap) do
        local cleanId = tostring(animId or ""):gsub("%D", "")
        if cleanId ~= "" then
            if animType == "PlayOnce" or not DEFAULT_ANIM_PATHS[animType] then
                local anim = Instance.new("Animation")
                anim.AnimationId = "rbxassetid://" .. cleanId
                local track = animator:LoadAnimation(anim)
                track.Priority = Enum.AnimationPriority.Action
                track:Play()
                track.Stopped:Connect(function()
                    anim:Destroy()
                end)
            else
                Data.AnimIDs[animType] = cleanId
                didReplace = true
            end
        end
    end

    if didReplace then
        ApplyAnimIDs()
        local animate = char:FindFirstChild("Animate")
        if animate then
            pcall(function()
                humanoid:ChangeState(Enum.HumanoidStateType.Freefall)
                task.wait(0.05)
                humanoid:ChangeState(Enum.HumanoidStateType.Running)
            end)
        end
    end
end

-- ============================================================
-- GUI PRINCIPAL
-- ============================================================
local function CreateMainGUI()
    -- Remove GUI antiga se existir
    local old = CoreGui:FindFirstChild("AnimEditV1") or PlayerGui:FindFirstChild("AnimEditV1")
    if old then old:Destroy() end

    local ScreenGui = Create("ScreenGui", {
        Name = "AnimEditV1",
        ResetOnSpawn = false,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        Parent = (gethui and gethui()) or CoreGui or PlayerGui
    })

    -- ========== JANELA PRINCIPAL ==========
    local Main = Create("Frame", {
        Name = "Main",
        Size = UDim2.new(0, 300, 0, 360),
        Position = UDim2.new(0.5, -150, 0.5, -180),
        BackgroundColor3 = Color3.fromRGB(22, 22, 30),
        BorderSizePixel = 0,
        Active = true,
        Parent = ScreenGui
    })
    AddCorner(Main, 10)
    AddStroke(Main, Color3.fromRGB(130, 90, 255), 1.5)

    -- Título / barra de arraste
    local TitleBar = Create("Frame", {
        Name = "TitleBar",
        Size = UDim2.new(1, 0, 0, 28),
        BackgroundColor3 = Color3.fromRGB(32, 28, 48),
        BorderSizePixel = 0,
        Parent = Main
    })
    AddCorner(TitleBar, 10)

    local Title = Create("TextLabel", {
        Size = UDim2.new(1, -70, 1, 0),
        Position = UDim2.new(0, 8, 0, 0),
        BackgroundTransparency = 1,
        Text = "Anim Edit v1.0",
        TextColor3 = Color3.fromRGB(220, 200, 255),
        Font = Enum.Font.GothamBold,
        TextSize = 13,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = TitleBar
    })

    local MinimizeBtn = Create("TextButton", {
        Name = "Minimize",
        Size = UDim2.new(0, 22, 0, 22),
        Position = UDim2.new(1, -50, 0.5, -11),
        BackgroundColor3 = Color3.fromRGB(60, 50, 90),
        Text = "−",
        TextColor3 = Color3.fromRGB(255, 255, 255),
        Font = Enum.Font.GothamBold,
        TextSize = 16,
        Parent = TitleBar
    })
    AddCorner(MinimizeBtn, 5)

    local CloseBtn = Create("TextButton", {
        Name = "Close",
        Size = UDim2.new(0, 22, 0, 22),
        Position = UDim2.new(1, -24, 0.5, -11),
        BackgroundColor3 = Color3.fromRGB(160, 50, 70),
        Text = "×",
        TextColor3 = Color3.fromRGB(255, 255, 255),
        Font = Enum.Font.GothamBold,
        TextSize = 14,
        Parent = TitleBar
    })
    AddCorner(CloseBtn, 5)

    MakeDraggable(Main, TitleBar)

    -- ========== BOLINHA MINIMIZADA ==========
    local Ball = Create("TextButton", {
        Name = "Ball",
        Size = UDim2.new(0, 40, 0, 40),
        Position = UDim2.new(0, 16, 0.5, -20),
        BackgroundColor3 = Color3.fromRGB(90, 60, 180),
        Text = "AE",
        TextColor3 = Color3.fromRGB(255, 255, 255),
        Font = Enum.Font.GothamBold,
        TextSize = 13,
        Visible = false,
        Parent = ScreenGui
    })
    AddCorner(Ball, 20)
    AddStroke(Ball, Color3.fromRGB(180, 140, 255), 1.5)
    MakeDraggable(Ball)

    MinimizeBtn.MouseButton1Click:Connect(function()
        Main.Visible = false
        Ball.Visible = true
        Ball.Position = UDim2.new(0, Main.AbsolutePosition.X, 0, Main.AbsolutePosition.Y)
    end)

    Ball.MouseButton1Click:Connect(function()
        Ball.Visible = false
        Main.Visible = true
        Main.Position = UDim2.new(0, Ball.AbsolutePosition.X, 0, Ball.AbsolutePosition.Y)
    end)

    CloseBtn.MouseButton1Click:Connect(function()
        ScreenGui:Destroy()
    end)

    -- ========== ABAS ==========
    local TabBar = Create("Frame", {
        Size = UDim2.new(1, -12, 0, 24),
        Position = UDim2.new(0, 6, 0, 32),
        BackgroundTransparency = 1,
        Parent = Main
    })

    local Content = Create("Frame", {
        Name = "Content",
        Size = UDim2.new(1, -12, 1, -70),
        Position = UDim2.new(0, 6, 0, 60),
        BackgroundTransparency = 1,
        Parent = Main
    })

    local tabs = {}
    local currentTab = nil

    local function SwitchTab(name)
        for n, frame in pairs(tabs) do
            frame.Visible = (n == name)
        end
        currentTab = name
    end

    local function CreateTab(name, order)
        local btn = Create("TextButton", {
            Size = UDim2.new(0, 68, 1, 0),
            Position = UDim2.new(0, (order - 1) * 72, 0, 0),
            BackgroundColor3 = Color3.fromRGB(45, 40, 70),
            Text = name,
            TextColor3 = Color3.fromRGB(220, 210, 255),
            Font = Enum.Font.GothamSemibold,
            TextSize = 11,
            Parent = TabBar
        })
        AddCorner(btn, 5)

        local page = Create("ScrollingFrame", {
            Name = name,
            Size = UDim2.new(1, 0, 1, 0),
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            ScrollBarThickness = 3,
            CanvasSize = UDim2.new(0, 0, 0, 500),
            Visible = false,
            Parent = Content
        })
        tabs[name] = page

        btn.MouseButton1Click:Connect(function()
            SwitchTab(name)
            for _, b in pairs(TabBar:GetChildren()) do
                if b:IsA("TextButton") then
                    b.BackgroundColor3 = Color3.fromRGB(45, 40, 70)
                end
            end
            btn.BackgroundColor3 = Color3.fromRGB(90, 60, 160)
        end)

        return page, btn
    end

    local pageAnims, btnAnims = CreateTab("Animações", 1)
    local pageButtons, btnButtons = CreateTab("Botões", 2)
    local pageTut, btnTut = CreateTab("Tutorial", 3)

    SwitchTab("Animações")
    btnAnims.BackgroundColor3 = Color3.fromRGB(90, 60, 160)

    -- ========== ABA ANIMAÇÕES ==========
    local y = 4
    Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 18),
        Position = UDim2.new(0, 0, 0, y),
        BackgroundTransparency = 1,
        Text = "Cole o ID (só números). Vazio = não altera.",
        TextColor3 = Color3.fromRGB(180, 170, 210),
        Font = Enum.Font.Gotham,
        TextSize = 11,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = pageAnims
    })
    y = y + 22

    local animBoxes = {}
    for _, animName in ipairs({"Idle", "Walk", "Run", "Jump", "Fall", "Climb", "Swim", "SwimIdle"}) do
        Create("TextLabel", {
            Size = UDim2.new(0, 70, 0, 22),
            Position = UDim2.new(0, 0, 0, y),
            BackgroundTransparency = 1,
            Text = animName,
            TextColor3 = Color3.fromRGB(210, 200, 255),
            Font = Enum.Font.GothamSemibold,
            TextSize = 11,
            TextXAlignment = Enum.TextXAlignment.Left,
            Parent = pageAnims
        })

        local box = Create("TextBox", {
            Size = UDim2.new(1, -78, 0, 22),
            Position = UDim2.new(0, 74, 0, y),
            BackgroundColor3 = Color3.fromRGB(35, 32, 50),
            Text = Data.AnimIDs[animName] or "",
            PlaceholderText = "ID...",
            TextColor3 = Color3.fromRGB(230, 230, 255),
            PlaceholderColor3 = Color3.fromRGB(120, 110, 150),
            Font = Enum.Font.Gotham,
            TextSize = 11,
            ClearTextOnFocus = false,
            Parent = pageAnims
        })
        AddCorner(box, 5)
        AddStroke(box, Color3.fromRGB(80, 60, 130), 1)
        animBoxes[animName] = box

        box.FocusLost:Connect(function()
            Data.AnimIDs[animName] = box.Text:gsub("%D", "")
        end)

        y = y + 26
    end

    local ApplyBtn = Create("TextButton", {
        Size = UDim2.new(1, 0, 0, 26),
        Position = UDim2.new(0, 0, 0, y + 4),
        BackgroundColor3 = Color3.fromRGB(90, 60, 180),
        Text = "Aplicar Animações",
        TextColor3 = Color3.fromRGB(255, 255, 255),
        Font = Enum.Font.GothamBold,
        TextSize = 12,
        Parent = pageAnims
    })
    AddCorner(ApplyBtn, 6)
    ApplyBtn.MouseButton1Click:Connect(function()
        for name, box in pairs(animBoxes) do
            Data.AnimIDs[name] = box.Text:gsub("%D", "")
        end
        ApplyAnimIDs()
        Notify("Animações aplicadas!")
    end)

    pageAnims.CanvasSize = UDim2.new(0, 0, 0, y + 40)

    -- ========== ABA BOTÕES ==========
    y = 8
    Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 20),
        Position = UDim2.new(0, 0, 0, y),
        BackgroundTransparency = 1,
        Text = "Crie botões flutuantes personalizados",
        TextColor3 = Color3.fromRGB(180, 170, 210),
        Font = Enum.Font.Gotham,
        TextSize = 12,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = pageButtons
    })
    y = y + 26

    -- Campos de criação
    local function MakeLabel(text, py)
        Create("TextLabel", {
            Size = UDim2.new(0, 110, 0, 22),
            Position = UDim2.new(0, 0, 0, py),
            BackgroundTransparency = 1,
            Text = text,
            TextColor3 = Color3.fromRGB(200, 190, 240),
            Font = Enum.Font.Gotham,
            TextSize = 12,
            TextXAlignment = Enum.TextXAlignment.Left,
            Parent = pageButtons
        })
    end

    MakeLabel("Texto do botão:", y)
    local TextBox = Create("TextBox", {
        Size = UDim2.new(1, -120, 0, 24),
        Position = UDim2.new(0, 115, 0, y),
        BackgroundColor3 = Color3.fromRGB(35, 32, 50),
        Text = "Emote",
        TextColor3 = Color3.fromRGB(230, 230, 255),
        Font = Enum.Font.Gotham,
        TextSize = 13,
        Parent = pageButtons
    })
    AddCorner(TextBox, 5)
    y = y + 30

    -- Uma TextBox de ID para cada tipo de animação
    Create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 16),
        Position = UDim2.new(0, 0, 0, y),
        BackgroundTransparency = 1,
        Text = "ID por animação (deixe vazio o que não usar):",
        TextColor3 = Color3.fromRGB(200, 190, 240),
        Font = Enum.Font.Gotham,
        TextSize = 11,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = pageButtons
    })
    y = y + 18

    local TYPE_OPTIONS = {
        {id = "PlayOnce", label = "PlayOnce"},
        {id = "Idle", label = "Idle"},
        {id = "Walk", label = "Walk"},
        {id = "Run", label = "Run"},
        {id = "Jump", label = "Jump"},
        {id = "Fall", label = "Fall"},
        {id = "Climb", label = "Climb"},
        {id = "Swim", label = "Swim"},
    }
    local typeIdBoxes = {}

    for _, opt in ipairs(TYPE_OPTIONS) do
        Create("TextLabel", {
            Size = UDim2.new(0, 62, 0, 22),
            Position = UDim2.new(0, 0, 0, y),
            BackgroundTransparency = 1,
            Text = opt.label,
            TextColor3 = Color3.fromRGB(210, 200, 255),
            Font = Enum.Font.GothamSemibold,
            TextSize = 11,
            TextXAlignment = Enum.TextXAlignment.Left,
            Parent = pageButtons
        })
        local box = Create("TextBox", {
            Size = UDim2.new(1, -70, 0, 22),
            Position = UDim2.new(0, 66, 0, y),
            BackgroundColor3 = Color3.fromRGB(35, 32, 50),
            Text = "",
            PlaceholderText = "ID...",
            TextColor3 = Color3.fromRGB(230, 230, 255),
            PlaceholderColor3 = Color3.fromRGB(120, 110, 150),
            Font = Enum.Font.Gotham,
            TextSize = 11,
            ClearTextOnFocus = false,
            Parent = pageButtons
        })
        AddCorner(box, 5)
        AddStroke(box, Color3.fromRGB(80, 60, 130), 1)
        typeIdBoxes[opt.id] = box
        y = y + 26
    end
    y = y + 4

    local function GetAnimMap()
        local map = {}
        for _, opt in ipairs(TYPE_OPTIONS) do
            local box = typeIdBoxes[opt.id]
            if box then
                local id = box.Text:gsub("%D", "")
                if id ~= "" then
                    map[opt.id] = id
                end
            end
        end
        return map
    end

    MakeLabel("Tamanho (X,Y):", y)
    local SizeBox = Create("TextBox", {
        Size = UDim2.new(1, -120, 0, 24),
        Position = UDim2.new(0, 115, 0, y),
        BackgroundColor3 = Color3.fromRGB(35, 32, 50),
        Text = "80,40",
        TextColor3 = Color3.fromRGB(230, 230, 255),
        Font = Enum.Font.Gotham,
        TextSize = 13,
        Parent = pageButtons
    })
    AddCorner(SizeBox, 5)
    y = y + 30

    -- Toggles
    local StrokeToggle = true
    local CornerToggle = true
    local DraggableToggle = true
    local StrokeColor = Color3.fromRGB(140, 90, 255)

    local function MakeToggle(label, py, default, callback)
        local frame = Create("Frame", {
            Size = UDim2.new(1, 0, 0, 26),
            Position = UDim2.new(0, 0, 0, py),
            BackgroundTransparency = 1,
            Parent = pageButtons
        })
        Create("TextLabel", {
            Size = UDim2.new(0, 140, 1, 0),
            BackgroundTransparency = 1,
            Text = label,
            TextColor3 = Color3.fromRGB(200, 190, 240),
            Font = Enum.Font.Gotham,
            TextSize = 12,
            TextXAlignment = Enum.TextXAlignment.Left,
            Parent = frame
        })
        local btn = Create("TextButton", {
            Size = UDim2.new(0, 50, 0, 22),
            Position = UDim2.new(0, 150, 0.5, -11),
            BackgroundColor3 = default and Color3.fromRGB(70, 160, 90) or Color3.fromRGB(90, 50, 60),
            Text = default and "ON" or "OFF",
            TextColor3 = Color3.fromRGB(255, 255, 255),
            Font = Enum.Font.GothamBold,
            TextSize = 12,
            Parent = frame
        })
        AddCorner(btn, 5)
        local state = default
        btn.MouseButton1Click:Connect(function()
            state = not state
            btn.Text = state and "ON" or "OFF"
            btn.BackgroundColor3 = state and Color3.fromRGB(70, 160, 90) or Color3.fromRGB(90, 50, 60)
            callback(state)
        end)
        return function() return state end
    end

    local getStroke = MakeToggle("UIStroke:", y, true, function(v) StrokeToggle = v end)
    y = y + 28
    local getCorner = MakeToggle("UICorner:", y, true, function(v) CornerToggle = v end)
    y = y + 28
    local getDrag = MakeToggle("Arrastável:", y, true, function(v) DraggableToggle = v end)
    y = y + 32

    MakeLabel("Cor Stroke (R,G,B):", y)
    local ColorBox = Create("TextBox", {
        Size = UDim2.new(1, -120, 0, 24),
        Position = UDim2.new(0, 115, 0, y),
        BackgroundColor3 = Color3.fromRGB(35, 32, 50),
        Text = "140,90,255",
        TextColor3 = Color3.fromRGB(230, 230, 255),
        Font = Enum.Font.Gotham,
        TextSize = 13,
        Parent = pageButtons
    })
    AddCorner(ColorBox, 5)
    y = y + 36

    -- Lista de botões criados
    local ButtonsList = Create("Frame", {
        Name = "ButtonsList",
        Size = UDim2.new(1, 0, 0, 120),
        Position = UDim2.new(0, 0, 0, y),
        BackgroundColor3 = Color3.fromRGB(28, 26, 40),
        BorderSizePixel = 0,
        Parent = pageButtons
    })
    AddCorner(ButtonsList, 8)
    local ListLayout = Create("UIListLayout", {
        Padding = UDim.new(0, 4),
        Parent = ButtonsList
    })

    local function RefreshButtonsList()
        for _, c in pairs(ButtonsList:GetChildren()) do
            if c:IsA("Frame") then c:Destroy() end
        end
        for i, btnData in ipairs(Data.Buttons) do
            local row = Create("Frame", {
                Size = UDim2.new(1, -8, 0, 28),
                BackgroundColor3 = Color3.fromRGB(40, 36, 55),
                Parent = ButtonsList
            })
            AddCorner(row, 5)
            Create("TextLabel", {
                Size = UDim2.new(1, -60, 1, 0),
                Position = UDim2.new(0, 6, 0, 0),
                BackgroundTransparency = 1,
                Text = (function()
                    local parts = {}
                    if btnData.animMap then
                        for k, v in pairs(btnData.animMap) do
                            table.insert(parts, k)
                        end
                    end
                    table.sort(parts)
                    return btnData.text .. "  [" .. (#parts > 0 and table.concat(parts, ",") or "—") .. "]"
                end)(),
                TextColor3 = Color3.fromRGB(220, 210, 255),
                Font = Enum.Font.Gotham,
                TextSize = 12,
                TextXAlignment = Enum.TextXAlignment.Left,
                Parent = row
            })
            local del = Create("TextButton", {
                Size = UDim2.new(0, 50, 0, 22),
                Position = UDim2.new(1, -54, 0.5, -11),
                BackgroundColor3 = Color3.fromRGB(160, 50, 70),
                Text = "Excluir",
                TextColor3 = Color3.fromRGB(255, 255, 255),
                Font = Enum.Font.GothamBold,
                TextSize = 11,
                Parent = row
            })
            AddCorner(del, 4)
            del.MouseButton1Click:Connect(function()
                table.remove(Data.Buttons, i)
                -- Remove o botão visual também
                local floating = ScreenGui:FindFirstChild("FloatBtn_" .. i)
                if floating then floating:Destroy() end
                RefreshButtonsList()
            end)
        end
        ButtonsList.Size = UDim2.new(1, 0, 0, math.max(40, #Data.Buttons * 32 + 8))
        pageButtons.CanvasSize = UDim2.new(0, 0, 0, y + ButtonsList.Size.Y.Offset + 80)
    end

    local CreateBtn = Create("TextButton", {
        Size = UDim2.new(1, 0, 0, 34),
        Position = UDim2.new(0, 0, 0, y + 130),
        BackgroundColor3 = Color3.fromRGB(70, 140, 90),
        Text = "+ Criar Botão Flutuante",
        TextColor3 = Color3.fromRGB(255, 255, 255),
        Font = Enum.Font.GothamBold,
        TextSize = 14,
        Parent = pageButtons
    })
    AddCorner(CreateBtn, 8)

    CreateBtn.MouseButton1Click:Connect(function()
        local text = TextBox.Text
        local animMap = GetAnimMap()
        local sizeParts = SizeBox.Text:split(",")
        local sx = tonumber(sizeParts[1]) or 80
        local sy = tonumber(sizeParts[2]) or 40

        local colorParts = ColorBox.Text:split(",")
        local r = tonumber(colorParts[1]) or 140
        local g = tonumber(colorParts[2]) or 90
        local b = tonumber(colorParts[3]) or 255

        local hasAny = false
        for _ in pairs(animMap) do hasAny = true break end
        if not hasAny then
            Notify("Coloque pelo menos 1 ID em alguma animação!")
            return
        end

        local btnData = {
            text = text,
            size = Vector2.new(sx, sy),
            pos = UDim2.new(0.5, -sx/2, 0.7, 0),
            stroke = StrokeToggle,
            corner = CornerToggle,
            strokeColor = Color3.fromRGB(r, g, b),
            animMap = animMap,
            draggable = DraggableToggle
        }
        table.insert(Data.Buttons, btnData)

        -- Cria o botão visual flutuante
        local idx = #Data.Buttons
        local floatBtn = Create("TextButton", {
            Name = "FloatBtn_" .. idx,
            Size = UDim2.new(0, sx, 0, sy),
            Position = btnData.pos,
            BackgroundColor3 = Color3.fromRGB(45, 40, 70),
            Text = text,
            TextColor3 = Color3.fromRGB(255, 255, 255),
            Font = Enum.Font.GothamBold,
            AutoButtonColor = false,
            Parent = ScreenGui
        })
        AutoTextSize(floatBtn, 15)
        if CornerToggle then AddCorner(floatBtn, 8) end
        if StrokeToggle then AddStroke(floatBtn, Color3.fromRGB(r, g, b), 1.5) end

        if DraggableToggle then
            MakeDraggable(floatBtn)
        end

        SafeClick(floatBtn, function()
            PlayAnimation(animMap)
            Notify("Tocando: " .. text)
        end)

        -- Atualiza posição quando arrasta
        floatBtn:GetPropertyChangedSignal("Position"):Connect(function()
            btnData.pos = floatBtn.Position
        end)

        RefreshButtonsList()
        Notify("Botão criado!")
    end)

    pageButtons.CanvasSize = UDim2.new(0, 0, 0, y + 200)

    -- ========== ABA TUTORIAL ==========
    local tutText = [[
ANIM EDIT v1.0 - COMO USAR

1. JANELA PRINCIPAL
   • Arraste pela barra de título.
   • Clique em "−" para minimizar (vira bolinha).
   • Clique na bolinha para reabrir.
   • "×" fecha o editor.

2. ABA ANIMAÇÕES
   • Cole apenas o número do ID da animação.
   • Deixe em branco o que não quiser mudar.
   • Clique em "Aplicar Animações Agora".
   • Os IDs substituem as animações padrão do Animate.

3. ABA BOTÕES
   • Texto: o que aparece no botão (texto grande = fonte menor).
   • ID por animação: cada tipo tem sua própria caixinha
        PlayOnce = emote que toca uma vez
        Idle/Walk/Run... = troca só aquela animação
        Deixe vazio o que não quiser usar no botão
   • Tamanho: largura,altura (ex: 90,45)
   • UIStroke / UICorner / Arrastável: ON ou OFF
   • Cor Stroke: R,G,B (ex: 140,90,255)
   • Clique em "+ Criar Botão Flutuante"
   • Os botões aparecem na tela e podem ser arrastados.
   • Clique no botão (sem arrastar) para tocar a animação.
   • Na lista abaixo você pode excluir botões.

4. BOTÃO FINALIZAR
   • Gera um script novo com tudo que você configurou.
   • Copia automaticamente para a área de transferência.
   • Cole em um novo executor / LocalScript e use!

DICA: Emotes de "PlayOnce" são ideais para danças e poses.
Use Idle/Walk/Run quando quiser trocar o movimento padrão do personagem.
]]

    Create("TextLabel", {
        Size = UDim2.new(1, -10, 0, 900),
        Position = UDim2.new(0, 4, 0, 4),
        BackgroundTransparency = 1,
        Text = tutText,
        TextColor3 = Color3.fromRGB(200, 195, 230),
        Font = Enum.Font.Gotham,
        TextSize = 12,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Top,
        TextWrapped = true,
        Parent = pageTut
    })
    pageTut.CanvasSize = UDim2.new(0, 0, 0, 920)

    -- ========== BOTÃO FINALIZAR ==========
    local FinalBtn = Create("TextButton", {
        Size = UDim2.new(1, -12, 0, 28),
        Position = UDim2.new(0, 6, 1, -34),
        BackgroundColor3 = Color3.fromRGB(180, 80, 50),
        Text = "FINALIZAR (copiar script)",
        TextColor3 = Color3.fromRGB(255, 255, 255),
        Font = Enum.Font.GothamBold,
        TextSize = 12,
        Parent = Main
    })
    AddCorner(FinalBtn, 6)
    AddStroke(FinalBtn, Color3.fromRGB(255, 140, 80), 1)

    FinalBtn.MouseButton1Click:Connect(function()
        -- Atualiza dados das TextBoxes
        for name, box in pairs(animBoxes) do
            Data.AnimIDs[name] = box.Text:gsub("%D", "")
        end

        -- Gera o script exportado
        local lines = {}
        table.insert(lines, "--[[")
        table.insert(lines, "    Script gerado pelo Anim Edit v1.0")
        table.insert(lines, "    Cole este script no executor e execute.")
        table.insert(lines, "]]")
        table.insert(lines, "")
        table.insert(lines, "local Players = game:GetService(\"Players\")")
        table.insert(lines, "local UserInputService = game:GetService(\"UserInputService\")")
        table.insert(lines, "local LocalPlayer = Players.LocalPlayer")
        table.insert(lines, "local Mouse = LocalPlayer:GetMouse()")
        table.insert(lines, "")
        table.insert(lines, "local function Create(class, props)")
        table.insert(lines, "    local inst = Instance.new(class)")
        table.insert(lines, "    for k,v in pairs(props or {}) do if k ~= \"Parent\" then inst[k] = v end end")
        table.insert(lines, "    if props and props.Parent then inst.Parent = props.Parent end")
        table.insert(lines, "    return inst")
        table.insert(lines, "end")
        table.insert(lines, "")
        table.insert(lines, "-- Animações padrão")
        table.insert(lines, "local AnimIDs = {")
        for name, id in pairs(Data.AnimIDs) do
            if id and id ~= "" then
                table.insert(lines, string.format("    %s = \"%s\",", name, id))
            end
        end
        table.insert(lines, "}")
        table.insert(lines, "")
        table.insert(lines, "local DEFAULT_PATHS = {")
        table.insert(lines, "    Idle={\"idle\",\"Animation1\"}, Walk={\"walk\",\"WalkAnim\"}, Run={\"run\",\"RunAnim\"},")
        table.insert(lines, "    Jump={\"jump\",\"JumpAnim\"}, Fall={\"fall\",\"FallAnim\"}, Climb={\"climb\",\"ClimbAnim\"},")
        table.insert(lines, "    Swim={\"swim\",\"Swim\"}, SwimIdle={\"swimidle\",\"SwimIdle\"},")
        table.insert(lines, "}")
        table.insert(lines, "")
        table.insert(lines, "local function ApplyAnims()")
        table.insert(lines, "    local char = LocalPlayer.Character")
        table.insert(lines, "    if not char then return end")
        table.insert(lines, "    local animate = char:FindFirstChild(\"Animate\")")
        table.insert(lines, "    if not animate then return end")
        table.insert(lines, "    for name, id in pairs(AnimIDs) do")
        table.insert(lines, "        if id ~= \"\" and DEFAULT_PATHS[name] then")
        table.insert(lines, "            local folder = animate:FindFirstChild(DEFAULT_PATHS[name][1])")
        table.insert(lines, "            if folder then")
        table.insert(lines, "                local anim = folder:FindFirstChild(DEFAULT_PATHS[name][2])")
        table.insert(lines, "                if anim and anim:IsA(\"Animation\") then")
        table.insert(lines, "                    anim.AnimationId = \"rbxassetid://\" .. id")
        table.insert(lines, "                end")
        table.insert(lines, "            end")
        table.insert(lines, "        end")
        table.insert(lines, "    end")
        table.insert(lines, "end")
        table.insert(lines, "")
        table.insert(lines, "local function PlayAnim(animMap)")
        table.insert(lines, "    local char = LocalPlayer.Character")
        table.insert(lines, "    if not char then return end")
        table.insert(lines, "    local hum = char:FindFirstChildOfClass(\"Humanoid\")")
        table.insert(lines, "    if not hum then return end")
        table.insert(lines, "    local animator = hum:FindFirstChildOfClass(\"Animator\") or hum")
        table.insert(lines, "    if type(animMap) ~= \"table\" then return end")
        table.insert(lines, "    local replaced = false")
        table.insert(lines, "    for tipo, id in pairs(animMap) do")
        table.insert(lines, "        id = tostring(id):gsub(\"%D\", \"\")")
        table.insert(lines, "        if id ~= \"\" then")
        table.insert(lines, "            if tipo == \"PlayOnce\" or not DEFAULT_PATHS[tipo] then")
        table.insert(lines, "                local a = Instance.new(\"Animation\")")
        table.insert(lines, "                a.AnimationId = \"rbxassetid://\" .. id")
        table.insert(lines, "                local t = animator:LoadAnimation(a)")
        table.insert(lines, "                t.Priority = Enum.AnimationPriority.Action")
        table.insert(lines, "                t:Play()")
        table.insert(lines, "            else")
        table.insert(lines, "                AnimIDs[tipo] = id")
        table.insert(lines, "                replaced = true")
        table.insert(lines, "            end")
        table.insert(lines, "        end")
        table.insert(lines, "    end")
        table.insert(lines, "    if replaced then ApplyAnims() end")
        table.insert(lines, "end")
        table.insert(lines, "")
        table.insert(lines, "-- Aplica animações ao spawnar")
        table.insert(lines, "if LocalPlayer.Character then ApplyAnims() end")
        table.insert(lines, "LocalPlayer.CharacterAdded:Connect(function() task.wait(1) ApplyAnims() end)")
        table.insert(lines, "")
        table.insert(lines, "-- Botões flutuantes")
        table.insert(lines, "local ScreenGui = Create(\"ScreenGui\", {Name=\"AnimEditGenerated\", ResetOnSpawn=false, Parent=(gethui and gethui()) or game:GetService(\"CoreGui\") or LocalPlayer:WaitForChild(\"PlayerGui\")})")
        table.insert(lines, "")
        table.insert(lines, "local function MakeDraggable(gui)")
        table.insert(lines, "    local dragging, start, startPos")
        table.insert(lines, "    gui.InputBegan:Connect(function(i)")
        table.insert(lines, "        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then")
        table.insert(lines, "            dragging = true; start = i.Position; startPos = gui.Position")
        table.insert(lines, "            i.Changed:Connect(function() if i.UserInputState == Enum.UserInputState.End then dragging = false end end)")
        table.insert(lines, "        end")
        table.insert(lines, "    end)")
        table.insert(lines, "    UserInputService.InputChanged:Connect(function(i)")
        table.insert(lines, "        if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then")
        table.insert(lines, "            local d = i.Position - start")
        table.insert(lines, "            gui.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)")
        table.insert(lines, "        end")
        table.insert(lines, "    end)")
        table.insert(lines, "end")
        table.insert(lines, "")
        table.insert(lines, "local function SafeClick(btn, cb)")
        table.insert(lines, "    local startP, moved")
        table.insert(lines, "    btn.MouseButton1Down:Connect(function() startP = Vector2.new(Mouse.X, Mouse.Y); moved = false end)")
        table.insert(lines, "    btn.MouseMoved:Connect(function() if startP and (Vector2.new(Mouse.X, Mouse.Y)-startP).Magnitude > 8 then moved = true end end)")
        table.insert(lines, "    btn.MouseButton1Up:Connect(function() if startP and not moved then cb() end; startP = nil end)")
        table.insert(lines, "end")
        table.insert(lines, "")

        for i, b in ipairs(Data.Buttons) do
            local pos = b.pos
            table.insert(lines, string.format("-- Botão %d: %s", i, b.text))
            table.insert(lines, string.format("do"))
            table.insert(lines, string.format("    local btn = Create(\"TextButton\", {"))
            table.insert(lines, string.format("        Size = UDim2.new(0, %d, 0, %d),", b.size.X, b.size.Y))
            table.insert(lines, string.format("        Position = UDim2.new(%.3f, %d, %.3f, %d),", pos.X.Scale, pos.X.Offset, pos.Y.Scale, pos.Y.Offset))
            table.insert(lines, string.format("        BackgroundColor3 = Color3.fromRGB(45,40,70),"))
            table.insert(lines, string.format("        Text = %q,", b.text))
            table.insert(lines, string.format("        TextColor3 = Color3.fromRGB(255,255,255),"))
            table.insert(lines, string.format("        Font = Enum.Font.GothamBold,"))
            table.insert(lines, string.format("        TextSize = 14,"))
            table.insert(lines, string.format("        AutoButtonColor = false,"))
            table.insert(lines, string.format("        Parent = ScreenGui"))
            table.insert(lines, string.format("    })"))
            if b.corner then
                table.insert(lines, "    Create(\"UICorner\", {CornerRadius=UDim.new(0,8), Parent=btn})")
            end
            if b.stroke then
                local c = b.strokeColor
                table.insert(lines, string.format("    Create(\"UIStroke\", {Color=Color3.fromRGB(%d,%d,%d), Thickness=1.5, Parent=btn})", math.floor(c.R*255), math.floor(c.G*255), math.floor(c.B*255)))
            end
            if b.draggable then
                table.insert(lines, "    MakeDraggable(btn)")
            end
            local mapParts = {}
            if b.animMap then
                for k, v in pairs(b.animMap) do
                    table.insert(mapParts, string.format("[%q]=%q", k, tostring(v)))
                end
            elseif b.animId and b.animId ~= "" then
                table.insert(mapParts, string.format("[%q]=%q", b.animType or "PlayOnce", b.animId))
            end
            local mapLua = "{" .. table.concat(mapParts, ",") .. "}"
            table.insert(lines, "    SafeClick(btn, function() PlayAnim(" .. mapLua .. ") end)")
            table.insert(lines, "end")
            table.insert(lines, "")
        end


        table.insert(lines, "")
        table.insert(lines, "print(\"[Anim Edit] Script gerado carregado com sucesso!\")")

        local finalScript = table.concat(lines, "\n")

        -- Copia para clipboard
        local ok = pcall(function()
            if setclipboard then
                setclipboard(finalScript)
            elseif toclipboard then
                toclipboard(finalScript)
            else
                -- Fallback: cria um TextBox temporário
                local sg = Instance.new("ScreenGui")
                sg.Parent = (gethui and gethui()) or CoreGui
                local tb = Instance.new("TextBox")
                tb.Size = UDim2.new(0, 1, 0, 1)
                tb.Position = UDim2.new(2, 0, 2, 0)
                tb.Text = finalScript
                tb.Parent = sg
                tb:CaptureFocus()
                task.wait(0.1)
                tb:Destroy()
                sg:Destroy()
            end
        end)

        if ok then
            Notify("Script copiado para a área de transferência!")
        else
            Notify("Não foi possível copiar automaticamente. Veja o console.")
            print("========== SCRIPT GERADO ==========")
            print(finalScript)
            print("===================================")
        end
    end)

    Notify("Anim Edit v1.0 carregado!")
end

-- Inicia
CreateMainGUI()
