-- VISION | SCRIPT COMPLETO
-- ESP, esqueleto, nomes, mira, velocidade, voo e veículos.
-- ESP requer Drawing no executor.
-- Não altera dinheiro nem concede ferramentas pelo servidor.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local Player = Players.LocalPlayer
local Mouse = Player:GetMouse()
local Env = getgenv and getgenv() or _G

-- Limpa versões anteriores.
for _, name in ipairs({
    "VisionVehicles",
    "VisionController",
    "BasicESPController",
}) do
    local previous = Env[name]
    if previous and previous.Stop then
        pcall(previous.Stop)
    end
end

local Config = {
    ESP = true,
    Skeleton = true,
    Names = true,
    NPCs = false,
    TeamCheck = false,
    Color = Color3.fromRGB(110, 150, 255),

    Aim = false,
    AimPart = "Head",
    AimRadius = 220,
    WallCheck = true,
    NoRecoil = false,
    RecoilStrength = 1,

    SpeedEnabled = false,
    Speed = 32,
    Fly = false,
    FlySpeed = 60,

    VehicleHeight = 10,
    VehicleDistance = 6,
    ThrowSpeed = 180,
    GrabRange = 45,
}

local Controller = {Config = Config}
Env.VisionController = Controller

local Connections = {}
local Visuals = {}
local Targets = {}
local Refreshers = {}

local Stopped = false
local Aiming = false
local LockedPart
local HeldVehicle

local FlightVelocity
local FlightAttachment
local FlightHumanoid
local OldAutoRotate

local SpeedHumanoid
local SavedSpeed
local SpeedApplied = false

local DrawingAvailable = Drawing and Drawing.new
local Binding = "VisionCompleteCamera"

local function connect(signal, callback)
    local connection = signal:Connect(callback)
    table.insert(Connections, connection)
    return connection
end

local function make(class, properties, parent)
    local object = Instance.new(class)
    for key, value in pairs(properties) do
        object[key] = value
    end
    object.Parent = parent
    return object
end

local function round(object, radius)
    make("UICorner", {
        CornerRadius = UDim.new(0, radius or 10),
    }, object)
end

local function characterParts()
    local character = Player.Character
    return character,
        character and character:FindFirstChildOfClass("Humanoid"),
        character and character:FindFirstChild("HumanoidRootPart")
end

-- =========================================================
-- ESP / ALVOS
-- =========================================================

local Bones = {
    {"Head", "UpperTorso"},
    {"UpperTorso", "LowerTorso"},

    {"UpperTorso", "LeftUpperArm"},
    {"LeftUpperArm", "LeftLowerArm"},
    {"LeftLowerArm", "LeftHand"},

    {"UpperTorso", "RightUpperArm"},
    {"RightUpperArm", "RightLowerArm"},
    {"RightLowerArm", "RightHand"},

    {"LowerTorso", "LeftUpperLeg"},
    {"LeftUpperLeg", "LeftLowerLeg"},
    {"LeftLowerLeg", "LeftFoot"},

    {"LowerTorso", "RightUpperLeg"},
    {"RightUpperLeg", "RightLowerLeg"},
    {"RightLowerLeg", "RightFoot"},

    {"Head", "Torso"},
    {"Torso", "Left Arm"},
    {"Torso", "Right Arm"},
    {"Torso", "Left Leg"},
    {"Torso", "Right Leg"},
}

local function draw(kind, properties)
    local object = Drawing.new(kind)
    for key, value in pairs(properties) do
        object[key] = value
    end
    return object
end

local function createVisual()
    local visual = {
        Box = draw("Square", {
            Visible = false,
            Filled = false,
            Thickness = 1.5,
            Color = Config.Color,
        }),
        Name = draw("Text", {
            Visible = false,
            Center = true,
            Outline = true,
            Size = 14,
            Color = Config.Color,
        }),
        Lines = {},
    }

    for index = 1, #Bones do
        visual.Lines[index] = draw("Line", {
            Visible = false,
            Thickness = 1.5,
            Color = Config.Color,
        })
    end

    return visual
end

local function hideVisual(visual)
    visual.Box.Visible = false
    visual.Name.Visible = false
    for _, line in ipairs(visual.Lines) do
        line.Visible = false
    end
end

local function removeVisual(visual)
    visual.Box:Remove()
    visual.Name:Remove()
    for _, line in ipairs(visual.Lines) do
        line:Remove()
    end
end

local function track(object)
    if object:IsA("Humanoid") then
        local model = object.Parent
        if model and model:IsA("Model") then
            Targets[model] = object
        end
    end
end

connect(workspace.DescendantAdded, track)

connect(workspace.DescendantRemoving, function(object)
    if object:IsA("Humanoid") then
        local model = object.Parent
        if model and Targets[model] == object then
            Targets[model] = nil
            if Visuals[model] then
                removeVisual(Visuals[model])
                Visuals[model] = nil
            end
        end
    end
end)

for _, object in ipairs(workspace:GetDescendants()) do
    track(object)
end

local function validTarget(model, humanoid)
    if not model:IsDescendantOf(workspace)
        or model == Player.Character
        or not humanoid.Parent
        or humanoid.Health <= 0 then
        return false
    end

    local owner = Players:GetPlayerFromCharacter(model)

    if not owner then
        return Config.NPCs
    end

    if Config.TeamCheck and Player.Team
        and owner.Team == Player.Team then
        return false
    end

    return true
end

local function screenBox(camera, model)
    local frame, size = model:GetBoundingBox()
    local half = size * 0.5
    local minimum = Vector2.new(math.huge, math.huge)
    local maximum = Vector2.new(-math.huge, -math.huge)

    for x = -1, 1, 2 do
        for y = -1, 1, 2 do
            for z = -1, 1, 2 do
                local point = camera:WorldToViewportPoint(
                    frame:PointToWorldSpace(Vector3.new(
                        half.X * x,
                        half.Y * y,
                        half.Z * z
                    ))
                )

                if point.Z <= 0 then return end

                minimum = Vector2.new(
                    math.min(minimum.X, point.X),
                    math.min(minimum.Y, point.Y)
                )
                maximum = Vector2.new(
                    math.max(maximum.X, point.X),
                    math.max(maximum.Y, point.Y)
                )
            end
        end
    end

    local viewport = camera.ViewportSize
    if maximum.X < 0 or maximum.Y < 0
        or minimum.X > viewport.X
        or minimum.Y > viewport.Y then
        return
    end

    return minimum, maximum - minimum
end

local function canSee(camera, model, part)
    if not Config.WallCheck then return true end

    local parameters = RaycastParams.new()
    parameters.FilterType = Enum.RaycastFilterType.Exclude

    local excluded = {camera}
    if Player.Character then
        table.insert(excluded, Player.Character)
    end
    parameters.FilterDescendantsInstances = excluded

    local origin = camera.CFrame.Position
    local result = workspace:Raycast(
        origin,
        part.Position - origin,
        parameters
    )

    return not result or result.Instance:IsDescendantOf(model)
end

-- =========================================================
-- VOO
-- =========================================================

local function stopFlight()
    if FlightVelocity then FlightVelocity:Destroy() end
    if FlightAttachment then FlightAttachment:Destroy() end

    if FlightHumanoid and FlightHumanoid.Parent then
        FlightHumanoid.AutoRotate = OldAutoRotate
    end

    FlightVelocity = nil
    FlightAttachment = nil
    FlightHumanoid = nil
end

local function startFlight(root, humanoid)
    stopFlight()

    FlightHumanoid = humanoid
    OldAutoRotate = humanoid.AutoRotate
    humanoid.AutoRotate = false

    FlightAttachment = make("Attachment", {
        Name = "VisionFlightAttachment",
    }, root)

    FlightVelocity = make("LinearVelocity", {
        Name = "VisionFlightVelocity",
        Attachment0 = FlightAttachment,
        RelativeTo = Enum.ActuatorRelativeTo.World,
        MaxForce = math.huge,
        VectorVelocity = Vector3.zero,
    }, root)
end

-- =========================================================
-- VEÍCULOS
-- =========================================================

local VehicleStatus

local function vehicleMessage(message)
    if VehicleStatus then
        VehicleStatus.Text = message
    end
end

local function releaseVehicle()
    if not HeldVehicle then return end

    for _, object in ipairs(HeldVehicle.Constraints) do
        object:Destroy()
    end

    HeldVehicle = nil
end

local function findVehicle(part)
    local ancestor = part and part.Parent

    while ancestor and ancestor ~= workspace do
        if ancestor:IsA("Model") then
            local seat = ancestor:FindFirstChildWhichIsA("VehicleSeat", true)
                or ancestor:FindFirstChildWhichIsA("Seat", true)

            if seat then
                return ancestor, seat
            end
        end
        ancestor = ancestor.Parent
    end
end

local function grabVehicle()
    if HeldVehicle then
        releaseVehicle()
        vehicleMessage("Veículo solto.")
        return
    end

    local _, humanoid, root = characterParts()
    local target = Mouse.Target

    if not root or not humanoid or humanoid.Health <= 0 or not target then
        vehicleMessage("Aponte para um veículo próximo.")
        return
    end

    local model, seat = findVehicle(target)
    if not model then
        vehicleMessage("Nenhum veículo com assento identificado.")
        return
    end

    if (seat.Position - root.Position).Magnitude > Config.GrabRange then
        vehicleMessage("Aproxime-se do veículo.")
        return
    end

    -- Evita controlar a própria estrutura do personagem ao estar sentado.
    if seat.Occupant == humanoid then
        vehicleMessage("Saia do veículo antes de segurá-lo.")
        return
    end

    local part = seat.AssemblyRootPart or seat

    if part.Anchored then
        vehicleMessage("A estrutura do veículo está ancorada.")
        return
    end

    for _, connected in ipairs(part:GetConnectedParts(true)) do
        if connected.Anchored then
            vehicleMessage("Veículo preso a uma peça ancorada.")
            return
        end
    end

    local attachment = make("Attachment", {
        Name = "VisionVehicleAttachment",
    }, part)

    local position = make("AlignPosition", {
        Mode = Enum.PositionAlignmentMode.OneAttachment,
        Attachment0 = attachment,
        ApplyAtCenterOfMass = true,
        MaxForce = 1e9,
        MaxVelocity = 100,
        Responsiveness = 35,
        Position = root.Position
            + Vector3.new(0, Config.VehicleHeight, 0)
            + root.CFrame.LookVector * Config.VehicleDistance,
    }, part)

    local orientation = make("AlignOrientation", {
        Mode = Enum.OrientationAlignmentMode.OneAttachment,
        Attachment0 = attachment,
        MaxTorque = 1e9,
        MaxAngularVelocity = 30,
        Responsiveness = 25,
        CFrame = part.CFrame.Rotation,
    }, part)

    HeldVehicle = {
        Part = part,
        Model = model,
        Position = position,
        Constraints = {position, orientation, attachment},
    }

    vehicleMessage("Tentando segurar: " .. model.Name)
end

local function throwVehicle()
    if not HeldVehicle then
        vehicleMessage("Pegue um veículo primeiro.")
        return
    end

    local part = HeldVehicle.Part
    local camera = workspace.CurrentCamera

    releaseVehicle()

    if part.Parent and camera then
        part.AssemblyLinearVelocity =
            camera.CFrame.LookVector * Config.ThrowSpeed
            + Vector3.new(0, Config.ThrowSpeed * 0.15, 0)

        vehicleMessage("Impulso aplicado. O servidor pode corrigir.")
    end
end

-- =========================================================
-- INTERFACE
-- =========================================================

local Colors = {
    Background = Color3.fromRGB(14, 16, 25),
    Surface = Color3.fromRGB(24, 27, 40),
    Field = Color3.fromRGB(35, 39, 57),
    Accent = Color3.fromRGB(118, 108, 255),
    Text = Color3.fromRGB(240, 242, 255),
    Muted = Color3.fromRGB(147, 155, 182),
}

local Gui = make("ScreenGui", {
    Name = "VisionComplete",
    ResetOnSpawn = false,
    DisplayOrder = 100,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
}, Player:WaitForChild("PlayerGui"))

local Window = make("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(500, 550),
    BackgroundColor3 = Colors.Background,
    BorderSizePixel = 0,
    ClipsDescendants = true,
}, Gui)
round(Window, 18)

make("UIStroke", {
    Color = Color3.fromRGB(66, 67, 101),
    Thickness = 1,
}, Window)

local Scale = make("UIScale", {Scale = 1}, Window)

local function label(parent, value, size, color)
    return make("TextLabel", {
        BackgroundTransparency = 1,
        Text = value,
        Font = Enum.Font.GothamMedium,
        TextSize = size or 12,
        TextColor3 = color or Colors.Text,
        TextXAlignment = Enum.TextXAlignment.Left,
    }, parent)
end

local Header = make("Frame", {
    Size = UDim2.new(1, 0, 0, 86),
    BackgroundColor3 = Colors.Surface,
    BorderSizePixel = 0,
    Active = true,
}, Window)

make("UIGradient", {
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(49, 39, 85)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(24, 27, 40)),
    }),
    Rotation = 15,
}, Header)

local Title = label(Header, "VISION", 26)
Title.Position = UDim2.fromOffset(24, 17)
Title.Size = UDim2.fromOffset(280, 32)

local Subtitle = label(
    Header,
    "VISUAL  /  MOVIMENTO  /  CONTROLE FÍSICO",
    10,
    Colors.Muted
)
Subtitle.Position = UDim2.fromOffset(25, 53)
Subtitle.Size = UDim2.fromOffset(365, 17)

local Hide = make("TextButton", {
    Position = UDim2.new(1, -49, 0, 26),
    Size = UDim2.fromOffset(32, 30),
    BackgroundColor3 = Colors.Field,
    BorderSizePixel = 0,
    Text = "—",
    TextSize = 18,
    Font = Enum.Font.Gotham,
    TextColor3 = Colors.Text,
}, Header)
round(Hide, 8)

local Tabs = make("Frame", {
    Position = UDim2.fromOffset(16, 99),
    Size = UDim2.new(1, -32, 0, 36),
    BackgroundTransparency = 1,
}, Window)

make("UIListLayout", {
    FillDirection = Enum.FillDirection.Horizontal,
    Padding = UDim.new(0, 6),
    SortOrder = Enum.SortOrder.LayoutOrder,
}, Tabs)

local Pages = {}
local TabButtons = {}

local function selectPage(name)
    for key, page in pairs(Pages) do
        page.Visible = key == name
        TabButtons[key].BackgroundColor3 =
            key == name and Colors.Accent or Colors.Surface
        TabButtons[key].TextColor3 =
            key == name and Colors.Text or Colors.Muted
    end
end

local function newPage(name)
    local page = make("ScrollingFrame", {
        Position = UDim2.fromOffset(16, 150),
        Size = UDim2.new(1, -32, 1, -192),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ScrollBarThickness = 3,
        ScrollBarImageColor3 = Colors.Accent,
        CanvasSize = UDim2.new(),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
        Visible = false,
    }, Window)

    make("UIListLayout", {
        Padding = UDim.new(0, 8),
        SortOrder = Enum.SortOrder.LayoutOrder,
    }, page)

    make("UIPadding", {
        PaddingRight = UDim.new(0, 7),
        PaddingBottom = UDim.new(0, 8),
    }, page)

    local button = make("TextButton", {
        Size = UDim2.new(0.25, -5, 1, 0),
        BackgroundColor3 = Colors.Surface,
        BorderSizePixel = 0,
        Text = name,
        Font = Enum.Font.GothamSemibold,
        TextSize = 10,
        TextColor3 = Colors.Muted,
        AutoButtonColor = false,
    }, Tabs)
    round(button, 9)

    Pages[name] = page
    TabButtons[name] = button

    connect(button.MouseButton1Click, function()
        selectPage(name)
    end)

    return page
end

local VisualPage = newPage("VISUAL")
local AimPage = newPage("MIRA")
local MovementPage = newPage("MOVIMENTO")
local VehiclePage = newPage("VEÍCULOS")

local Order = 0

local function row(parent, name)
    Order += 1

    local frame = make("Frame", {
        Size = UDim2.new(1, 0, 0, 52),
        BackgroundColor3 = Colors.Surface,
        BorderSizePixel = 0,
        LayoutOrder = Order,
    }, parent)
    round(frame, 10)

    local caption = label(frame, name)
    caption.Position = UDim2.fromOffset(14, 0)
    caption.Size = UDim2.new(1, -166, 1, 0)

    return frame
end

local function refreshKey(key)
    for _, update in ipairs(Refreshers[key] or {}) do
        update()
    end
end

local function toggle(parent, name, key, callback)
    local frame = row(parent, name)

    local button = make("TextButton", {
        Size = UDim2.fromOffset(46, 26),
        Position = UDim2.new(1, -60, 0.5, -13),
        BackgroundColor3 = Colors.Field,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
    }, frame)
    round(button, 13)

    local knob = make("Frame", {
        Size = UDim2.fromOffset(18, 18),
        BackgroundColor3 = Colors.Text,
        BorderSizePixel = 0,
    }, button)
    round(knob, 9)

    local function refresh()
        local enabled = Config[key] == true

        TweenService:Create(button, TweenInfo.new(0.12), {
            BackgroundColor3 =
                enabled and Colors.Accent or Colors.Field,
        }):Play()

        TweenService:Create(knob, TweenInfo.new(0.12), {
            Position = UDim2.fromOffset(enabled and 24 or 4, 4),
        }):Play()
    end

    Refreshers[key] = Refreshers[key] or {}
    table.insert(Refreshers[key], refresh)

    connect(button.MouseButton1Click, function()
        Config[key] = not Config[key]
        if callback then callback(Config[key]) end
        refreshKey(key)
    end)

    refresh()
end

local function numberField(parent, name, key, minimum, maximum)
    local frame = row(parent, name)

    local field = make("TextBox", {
        Size = UDim2.fromOffset(118, 30),
        Position = UDim2.new(1, -132, 0.5, -15),
        BackgroundColor3 = Colors.Field,
        BorderSizePixel = 0,
        Text = tostring(Config[key]),
        ClearTextOnFocus = false,
        Font = Enum.Font.GothamMedium,
        TextSize = 13,
        TextColor3 = Colors.Text,
    }, frame)
    round(field, 8)

    connect(field.FocusLost, function()
        local value = tonumber((field.Text:gsub(",", ".")))

        if value and value == value
            and value > -math.huge and value < math.huge then

            if minimum ~= nil then
                value = math.max(minimum, value)
            end
            if maximum ~= nil then
                value = math.min(maximum, value)
            end

            Config[key] = value
        end

        field.Text = tostring(Config[key])
    end)
end

local function choice(parent, name, key, options)
    local frame = row(parent, name)

    local button = make("TextButton", {
        Size = UDim2.fromOffset(140, 30),
        Position = UDim2.new(1, -154, 0.5, -15),
        BackgroundColor3 = Colors.Field,
        BorderSizePixel = 0,
        Text = tostring(Config[key]) .. "  >",
        Font = Enum.Font.GothamMedium,
        TextSize = 11,
        TextColor3 = Colors.Text,
    }, frame)
    round(button, 8)

    connect(button.MouseButton1Click, function()
        local index = table.find(options, Config[key]) or 0
        Config[key] = options[index % #options + 1]
        button.Text = tostring(Config[key]) .. "  >"
    end)
end

local function note(parent, message)
    Order += 1
    local object = label(parent, message, 11, Colors.Muted)
    object.Size = UDim2.new(1, -8, 0, 48)
    object.TextWrapped = true
    object.LayoutOrder = Order
    return object
end

local function action(parent, name, callback, color)
    Order += 1

    local button = make("TextButton", {
        Size = UDim2.new(1, 0, 0, 40),
        BackgroundColor3 = color or Colors.Accent,
        BorderSizePixel = 0,
        Text = name,
        Font = Enum.Font.GothamSemibold,
        TextSize = 11,
        TextColor3 = Colors.Text,
        LayoutOrder = Order,
    }, parent)
    round(button, 10)

    connect(button.MouseButton1Click, callback)
    return button
end

-- Visual
toggle(VisualPage, "ESP / quadrado", "ESP")
toggle(VisualPage, "Esqueleto corporal", "Skeleton")
toggle(VisualPage, "Nickname", "Names")
toggle(VisualPage, "Incluir NPCs", "NPCs")
toggle(VisualPage, "Ignorar mesma equipe", "TeamCheck")

local ColorRow = row(VisualPage, "Cor do ESP")
local Palette = {
    Color3.fromRGB(110, 150, 255),
    Color3.fromRGB(70, 225, 255),
    Color3.fromRGB(255, 110, 155),
    Color3.fromRGB(100, 255, 170),
    Color3.fromRGB(255, 220, 100),
}

for index, color in ipairs(Palette) do
    local button = make("TextButton", {
        Size = UDim2.fromOffset(20, 20),
        Position = UDim2.new(1, -148 + (index - 1) * 26, 0.5, -10),
        BackgroundColor3 = color,
        BorderSizePixel = 0,
        Text = "",
    }, ColorRow)
    round(button, 10)

    connect(button.MouseButton1Click, function()
        Config.Color = color
    end)
end

if not DrawingAvailable then
    note(VisualPage, "Drawing indisponível: o ESP não será desenhado.")
end

-- Mira
toggle(AimPage, "Ativar aimbot", "Aim")
toggle(AimPage, "Verificar paredes", "WallCheck")
toggle(AimPage, "Incluir NPCs", "NPCs")

choice(AimPage, "Parte do corpo", "AimPart", {
    "Head",
    "HumanoidRootPart",
    "UpperTorso",
    "Torso",
})

numberField(AimPage, "Raio / pixels", "AimRadius", 0)
toggle(AimPage, "Correção extra de recoil", "NoRecoil")
numberField(AimPage, "Compensação / 0–1", "RecoilStrength", 0, 1)

note(AimPage,
    "Segure o botão direito. A correção extra atua na câmera; "
    .. "a mira instantânea também pode reduzir recoil visual."
)

-- Movimento
toggle(MovementPage, "Velocidade personalizada", "SpeedEnabled")
numberField(MovementPage, "Velocidade", "Speed", 0)

toggle(MovementPage, "Ativar voo / F", "Fly", function(enabled)
    if not enabled then stopFlight() end
end)

numberField(MovementPage, "Velocidade do voo", "FlySpeed", 0)

note(MovementPage,
    "Voo: WASD, Espaço para subir e Ctrl esquerdo para descer. "
    .. "O servidor pode corrigir o movimento."
)

-- Veículos
VehicleStatus = note(VehiclePage, "Aponte para um carro ou moto com assento.")

action(VehiclePage, "PEGAR / SOLTAR VEÍCULO  [G]", grabVehicle)
action(VehiclePage, "ARREMESSAR VEÍCULO  [H]", throwVehicle)

numberField(VehiclePage, "Altura acima do jogador", "VehicleHeight", 0)
numberField(VehiclePage, "Distância à frente", "VehicleDistance", 0)
numberField(VehiclePage, "Força do arremesso", "ThrowSpeed", 0)
numberField(VehiclePage, "Distância para pegar", "GrabRange", 0)

note(VehiclePage,
    "Para segurar sobre a cabeça, coloque distância à frente em 0. "
    .. "Veículos ancorados ou controlados pelo servidor podem não responder."
)

local Footer = label(
    Window,
    "SHIFT DIR: MENU   /   F: VOO   /   G: PEGAR   /   H: ARREMESSAR",
    9,
    Colors.Muted
)
Footer.Position = UDim2.new(0, 18, 1, -30)
Footer.Size = UDim2.new(1, -36, 0, 18)

local Launcher = make("TextButton", {
    Size = UDim2.fromOffset(44, 44),
    Position = UDim2.fromOffset(18, 180),
    BackgroundColor3 = Colors.Accent,
    BorderSizePixel = 0,
    Text = "V",
    Font = Enum.Font.GothamBold,
    TextSize = 20,
    TextColor3 = Colors.Text,
    Visible = false,
}, Gui)
round(Launcher, 14)

local function showMenu(visible)
    Window.Visible = visible
    Launcher.Visible = not visible
end

connect(Hide.MouseButton1Click, function()
    showMenu(false)
end)

connect(Launcher.MouseButton1Click, function()
    showMenu(true)
end)

-- Arrastar pelo cabeçalho.
local Dragging = false
local DragOrigin
local WindowOrigin
local DragTouch

connect(Header.InputBegan, function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        Dragging = true
        DragOrigin = input.Position
        WindowOrigin = Window.Position
        DragTouch = input.UserInputType == Enum.UserInputType.Touch
            and input or nil
    end
end)

connect(UIS.InputChanged, function(input)
    if Dragging and (
        input.UserInputType == Enum.UserInputType.MouseMovement
        or input == DragTouch
    ) then
        local delta = input.Position - DragOrigin

        Window.Position = UDim2.new(
            WindowOrigin.X.Scale,
            WindowOrigin.X.Offset + delta.X,
            WindowOrigin.Y.Scale,
            WindowOrigin.Y.Offset + delta.Y
        )
    end
end)

-- =========================================================
-- CONTROLES
-- =========================================================

connect(UIS.InputBegan, function(input, processed)
    if UIS:GetFocusedTextBox() then return end

    if input.KeyCode == Enum.KeyCode.RightShift then
        showMenu(not Window.Visible)
        return
    end

    if processed then return end

    if input.KeyCode == Enum.KeyCode.F then
        Config.Fly = not Config.Fly
        refreshKey("Fly")
        if not Config.Fly then stopFlight() end

    elseif input.KeyCode == Enum.KeyCode.G then
        grabVehicle()

    elseif input.KeyCode == Enum.KeyCode.H then
        throwVehicle()

    elseif input.UserInputType == Enum.UserInputType.MouseButton2 then
        Aiming = true
    end
end)

connect(UIS.InputEnded, function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input == DragTouch then
        Dragging = false
        DragTouch = nil
    end

    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        Aiming = false
        LockedPart = nil
    end
end)

connect(UIS.WindowFocusReleased, function()
    Aiming = false
    LockedPart = nil
    Dragging = false

    Config.Fly = false
    refreshKey("Fly")
    stopFlight()
    releaseVehicle()
end)

-- =========================================================
-- RENDERIZAÇÃO / MIRA
-- =========================================================

RunService:BindToRenderStep(
    Binding,
    Enum.RenderPriority.Camera.Value + 1,
    function()
        local camera = workspace.CurrentCamera
        if not camera then return end

        local viewport = camera.ViewportSize

        Scale.Scale = math.min(
            1,
            math.max(0.1, (viewport.X - 24) / 500),
            math.max(0.1, (viewport.Y - 24) / 550)
        )

        local center = viewport * 0.5
        local nearest = Config.AimRadius
        local selected

        local canAim = Config.Aim and Aiming
            and not UIS:GetFocusedTextBox()

        for model, humanoid in pairs(Targets) do
            local visual = Visuals[model]

            if visual then hideVisual(visual) end
            if not validTarget(model, humanoid) then continue end

            if Config.ESP and DrawingAvailable then
                if not visual then
                    visual = createVisual()
                    Visuals[model] = visual
                end

                local position, size = screenBox(camera, model)

                if position then
                    visual.Box.Position = position
                    visual.Box.Size = size
                    visual.Box.Color = Config.Color
                    visual.Box.Visible = true

                    local owner = Players:GetPlayerFromCharacter(model)

                    visual.Name.Text = owner and owner.Name or model.Name
                    visual.Name.Position = Vector2.new(
                        position.X + size.X * 0.5,
                        position.Y - 18
                    )
                    visual.Name.Color = Config.Color
                    visual.Name.Visible = Config.Names

                    if Config.Skeleton then
                        for index, pair in ipairs(Bones) do
                            local first = model:FindFirstChild(pair[1])
                            local second = model:FindFirstChild(pair[2])

                            if first and second
                                and first:IsA("BasePart")
                                and second:IsA("BasePart") then

                                local a, visibleA =
                                    camera:WorldToViewportPoint(first.Position)

                                local b, visibleB =
                                    camera:WorldToViewportPoint(second.Position)

                                local line = visual.Lines[index]
                                line.From = Vector2.new(a.X, a.Y)
                                line.To = Vector2.new(b.X, b.Y)
                                line.Color = Config.Color
                                line.Visible = visibleA and visibleB
                            end
                        end
                    end
                end
            end

            if canAim then
                local part = model:FindFirstChild(Config.AimPart)

                if part and part:IsA("BasePart") then
                    local point, onScreen =
                        camera:WorldToViewportPoint(part.Position)

                    if onScreen then
                        local distance =
                            (Vector2.new(point.X, point.Y) - center).Magnitude

                        if distance < nearest
                            and canSee(camera, model, part) then
                            nearest = distance
                            selected = part
                        end
                    end
                end
            end
        end

        LockedPart = selected

        if selected then
            camera.CFrame = CFrame.lookAt(
                camera.CFrame.Position,
                selected.Position
            )
        end
    end
)

RunService:BindToRenderStep(
    Binding .. "Recoil",
    Enum.RenderPriority.Last.Value,
    function()
        local camera = workspace.CurrentCamera
        local part = LockedPart

        if camera and Config.NoRecoil and Config.Aim
            and Aiming and part and part.Parent
            and not UIS:GetFocusedTextBox() then

            camera.CFrame = camera.CFrame:Lerp(
                CFrame.lookAt(camera.CFrame.Position, part.Position),
                math.clamp(Config.RecoilStrength, 0, 1)
            )
        end
    end
)

-- =========================================================
-- MOVIMENTO / FÍSICA
-- =========================================================

connect(RunService.Heartbeat, function()
    local _, humanoid, root = characterParts()
    local camera = workspace.CurrentCamera

    if humanoid ~= SpeedHumanoid then
        if SpeedApplied and SpeedHumanoid
            and SpeedHumanoid.Parent and SavedSpeed then
            SpeedHumanoid.WalkSpeed = SavedSpeed
        end

        SpeedHumanoid = humanoid
        SavedSpeed = humanoid and humanoid.WalkSpeed
        SpeedApplied = false
    end

    if humanoid then
        if Config.SpeedEnabled then
            if not SpeedApplied then
                SavedSpeed = humanoid.WalkSpeed
            end

            humanoid.WalkSpeed = Config.Speed
            SpeedApplied = true

        elseif SpeedApplied then
            humanoid.WalkSpeed = SavedSpeed
            SpeedApplied = false
        end
    end

    local alive = humanoid and humanoid.Health > 0 and root

    if Config.Fly and alive and camera then
        if not FlightVelocity
            or FlightVelocity.Parent ~= root
            or not FlightAttachment
            or FlightAttachment.Parent ~= root t
