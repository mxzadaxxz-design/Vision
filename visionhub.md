-- VISION | SCRIPT COMPLETO | INTERFACE MONOCROMÁTICA
-- ESP, contornos corporais, nomes, mira, velocidade, voo e veículos.
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
    Color = Color3.fromRGB(235, 235, 235),

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
    Noclip = false,

    VehicleHeight = 10,
    VehicleDistance = 6,
    ThrowSpeed = 3500,
    ThrowLift = 1800,
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

-- Partes corporais R6 e R15; exclui acessórios e HumanoidRootPart.
local BodyParts = {
    "Head", "Torso", "UpperTorso", "LowerTorso",
    "Left Arm", "Right Arm", "Left Leg", "Right Leg",
    "LeftUpperArm", "LeftLowerArm", "LeftHand",
    "RightUpperArm", "RightLowerArm", "RightHand",
    "LeftUpperLeg", "LeftLowerLeg", "LeftFoot",
    "RightUpperLeg", "RightLowerLeg", "RightFoot",
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
        PartOutlines = {},
    }

    return visual
end

local function hideVisual(visual)
    visual.Box.Visible = false
    visual.Name.Visible = false
    for _, outline in pairs(visual.PartOutlines) do
        outline.Enabled = false
    end
end

local function removeVisual(visual)
    visual.Box:Remove()
    visual.Name:Remove()
    for _, outline in pairs(visual.PartOutlines) do
        outline:Destroy()
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
    local frame, size
    if model:IsA("BasePart") then
        frame, size = model.CFrame, model.Size
    else
        frame, size = model:GetBoundingBox()
    end
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

-- Direção horizontal: acompanha a câmera sem inclinar o carro.
local function vehicleForward(root)
    local camera = workspace.CurrentCamera
    local look = camera and camera.CFrame.LookVector or root.CFrame.LookVector
    local forward = Vector3.new(look.X, 0, look.Z)
    if forward.Magnitude < 0.001 then
        look = root.CFrame.LookVector
        forward = Vector3.new(look.X, 0, look.Z)
    end
    if forward.Magnitude < 0.001 then
        return Vector3.new(0, 0, -1)
    end
    return forward.Unit
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
        -- Referência do assento, não dos eixos da raiz do veículo.
        CFrame = part.CFrame:ToObjectSpace(seat.CFrame),
    }, part)

    local position = make("AlignPosition", {
        Mode = Enum.PositionAlignmentMode.OneAttachment,
        Attachment0 = attachment,
        ApplyAtCenterOfMass = true,
        MaxForce = 1e9,
        MaxVelocity = 250,
        Responsiveness = 65,
        Position = root.Position
            + Vector3.new(0, Config.VehicleHeight, 0)
            + vehicleForward(root) * Config.VehicleDistance,
    }, part)

    local orientation = make("AlignOrientation", {
        Mode = Enum.OrientationAlignmentMode.OneAttachment,
        Attachment0 = attachment,
        MaxTorque = 1e9,
        MaxAngularVelocity = 60,
        Responsiveness = 65,
        CFrame = CFrame.lookAt(Vector3.zero, vehicleForward(root)),
    }, part)

    HeldVehicle = {
        Part = part,
        Model = model,
        Position = position,
        Orientation = orientation,
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
    if not camera or not part:IsDescendantOf(workspace) then
        releaseVehicle()
        vehicleMessage("Veículo ou câmera indisponível.")
        return
    end

    local direction = camera.CFrame.LookVector
    releaseVehicle()

    local success, problem = pcall(function()
        part.AssemblyAngularVelocity = Vector3.zero
        part.AssemblyLinearVelocity = direction * Config.ThrowSpeed
            + Vector3.new(0, Config.ThrowLift, 0)
    end)

    if success then
        vehicleMessage("Arremessado! Velocidade: " .. Config.ThrowSpeed)
    else
        vehicleMessage("Falha no arremesso: " .. tostring(problem))
    end
end

-- =========================================================
-- ATRAVESSAR PAREDES
-- =========================================================

local OriginalCollisions = {}

local function restoreCollisions()
    for part, original in pairs(OriginalCollisions) do
        if part.Parent then
            part.CanCollide = original
        end
    end
    table.clear(OriginalCollisions)
end

connect(RunService.PreSimulation, function()
    local character, humanoid = characterParts()
    if Stopped or not Config.Noclip or not character
        or not humanoid or humanoid.Health <= 0 then
        restoreCollisions()
        return
    end

    -- Restaura peças removidas ou de um personagem anterior.
    for part, original in pairs(OriginalCollisions) do
        if not part:IsDescendantOf(character) then
            if part.Parent then
                part.CanCollide = original
            end
            OriginalCollisions[part] = nil
        end
    end

    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            if OriginalCollisions[part] == nil then
                OriginalCollisions[part] = part.CanCollide
            end
            part.CanCollide = false
        end
    end
end)

connect(Player.CharacterRemoving, function()
    restoreCollisions()
end)

-- =========================================================
-- INTERFACE
-- =========================================================

local Colors = {
    Background = Color3.fromRGB(10, 10, 10),
    Surface = Color3.fromRGB(21, 21, 21),
    Field = Color3.fromRGB(32, 32, 32),
    Accent = Color3.fromRGB(210, 210, 210),
    Text = Color3.fromRGB(245, 245, 245),
    Muted = Color3.fromRGB(146, 146, 146),
    Stroke = Color3.fromRGB(55, 55, 55),
}

-- Cancela a animação anterior da mesma propriedade ao atualizar um controle.
local UITweens = setmetatable({}, {__mode = "k"})
local function animate(object, properties, duration)
    local previous = UITweens[object]
    if previous then previous:Cancel() end
    local tween = TweenService:Create(object,
        TweenInfo.new(duration or 0.18, Enum.EasingStyle.Quart,
            Enum.EasingDirection.Out), properties)
    UITweens[object] = tween
    tween:Play()
end

local function outline(object, transparency)
    return make("UIStroke", {
        Color = Colors.Stroke,
        Thickness = 1,
        Transparency = transparency or 0.3,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
    }, object)
end

local function label(parent, value, size, color)
    return make("TextLabel", {
        BackgroundTransparency = 1,
        Text = value,
        Font = Enum.Font.GothamMedium,
        TextSize = size or 12,
        TextColor3 = color or Colors.Text,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextTruncate = Enum.TextTruncate.AtEnd,
        BorderSizePixel = 0,
    }, parent)
end

local Gui = make("ScreenGui", {
    Name = "VisionComplete",
    ResetOnSpawn = false,
    DisplayOrder = 100,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
}, Player:WaitForChild("PlayerGui"))

local Window = make("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(760, 550),
    BackgroundColor3 = Colors.Background,
    BackgroundTransparency = 0.12,
    BorderSizePixel = 0,
    ClipsDescendants = true,
}, Gui)
round(Window, 16)
outline(Window, 0.05)
local Scale = make("UIScale", {Scale = 1}, Window)

local Header = make("Frame", {
    Size = UDim2.new(1, 0, 0, 76),
    BackgroundColor3 = Colors.Surface,
    BackgroundTransparency = 0.15,
    BorderSizePixel = 0,
    Active = true,
}, Window)

make("Frame", {
    Position = UDim2.new(0, 0, 1, -1),
    Size = UDim2.new(1, 0, 0, 1),
    BackgroundColor3 = Colors.Stroke,
    BackgroundTransparency = 0.35,
    BorderSizePixel = 0,
}, Header)

do
    local icon = make("Frame", {
        Position = UDim2.fromOffset(20, 18),
        Size = UDim2.fromOffset(40, 40),
        BackgroundColor3 = Colors.Field,
        BorderSizePixel = 0,
    }, Header)
    round(icon, 11)
    outline(icon)
    local letter = label(icon, "V", 23)
    letter.Size = UDim2.fromScale(1, 1)
    letter.Font = Enum.Font.GothamBold
    letter.TextXAlignment = Enum.TextXAlignment.Center
    local title = label(Header, "VISION", 21)
    title.Position = UDim2.fromOffset(73, 16)
    title.Size = UDim2.fromOffset(220, 27)
    title.Font = Enum.Font.GothamBold
    local subtitle = label(Header, "PAINEL DE CONTROLE", 9, Colors.Muted)
    subtitle.Position = UDim2.fromOffset(74, 44)
    subtitle.Size = UDim2.fromOffset(260, 15)
    local badge = make("Frame", {
        Position = UDim2.new(1, -186, 0, 26),
        Size = UDim2.fromOffset(112, 24),
        BackgroundColor3 = Colors.Field,
        BackgroundTransparency = 0.25,
        BorderSizePixel = 0,
    }, Header)
    round(badge, 6)
    outline(badge, 0.5)
    local text = label(badge, "CLIENTE LOCAL", 9, Colors.Muted)
    text.Size = UDim2.fromScale(1, 1)
    text.TextXAlignment = Enum.TextXAlignment.Center
end

local Hide = make("TextButton", {
    Position = UDim2.new(1, -54, 0, 21),
    Size = UDim2.fromOffset(34, 34),
    BackgroundColor3 = Colors.Field,
    BackgroundTransparency = 0.3,
    BorderSizePixel = 0,
    Text = "—",
    TextSize = 17,
    Font = Enum.Font.GothamMedium,
    TextColor3 = Colors.Muted,
    AutoButtonColor = false,
}, Header)
round(Hide, 9)
outline(Hide, 0.6)
connect(Hide.MouseEnter, function()
    animate(Hide, {BackgroundTransparency = 0, TextColor3 = Colors.Text})
end)
connect(Hide.MouseLeave, function()
    animate(Hide, {BackgroundTransparency = 0.3, TextColor3 = Colors.Muted})
end)

local Sidebar = make("Frame", {
    Position = UDim2.fromOffset(0, 76),
    Size = UDim2.new(0, 176, 1, -118),
    BackgroundColor3 = Colors.Surface,
    BackgroundTransparency = 0.4,
    BorderSizePixel = 0,
}, Window)
make("Frame", {
    Position = UDim2.new(1, -1, 0, 0),
    Size = UDim2.new(0, 1, 1, 0),
    BackgroundColor3 = Colors.Stroke,
    BackgroundTransparency = 0.4,
    BorderSizePixel = 0,
}, Sidebar)

do
    local title = label(Sidebar, "NAVEGAÇÃO", 9, Colors.Muted)
    title.Position = UDim2.fromOffset(20, 20)
    title.Size = UDim2.fromOffset(140, 15)
    local hint = label(Sidebar, "SHIFT DIREITO\nMostrar / ocultar painel", 10, Colors.Muted)
    hint.Position = UDim2.new(0, 20, 1, -57)
    hint.Size = UDim2.new(1, -34, 0, 36)
    hint.TextWrapped = true
    hint.TextTruncate = Enum.TextTruncate.None
    hint.TextYAlignment = Enum.TextYAlignment.Top
end

local Tabs = make("Frame", {
    Position = UDim2.fromOffset(12, 49),
    Size = UDim2.new(1, -24, 0, 255),
    BackgroundTransparency = 1,
}, Sidebar)
make("UIListLayout", {
    FillDirection = Enum.FillDirection.Vertical,
    Padding = UDim.new(0, 7),
    SortOrder = Enum.SortOrder.LayoutOrder,
}, Tabs)

local PageTitle = label(Window, "Visual", 21)
PageTitle.Position = UDim2.fromOffset(198, 96)
PageTitle.Size = UDim2.new(1, -222, 0, 27)
PageTitle.Font = Enum.Font.GothamBold
local PageDescription = label(Window, "", 11, Colors.Muted)
PageDescription.Position = UDim2.fromOffset(199, 128)
PageDescription.Size = UDim2.new(1, -224, 0, 18)

local PageInfo = {
    ["VISUAL"] = {"Visual", "Indicadores e aparência dos alvos.", "01"},
    ["MIRA"] = {"Mira", "Seleção de alvos e controle da câmera.", "02"},
    ["MOVIMENTO"] = {"Movimento", "Velocidade, voo e colisão.", "03"},
    ["VEÍCULOS"] = {"Veículos", "Posicionamento e arremesso de veículos.", "04"},
    ["EXTRAS"] = {"Extras", "Ferramentas locais e ações adicionais.", "05"},
}
local Pages, TabButtons, TabMarkers, TabNumbers, TabCaptions = {}, {}, {}, {}, {}
local SelectedPage
local TabOrder = 0

local function selectPage(name)
    SelectedPage = name
    local info = PageInfo[name]
    PageTitle.Text = info and info[1] or name
    PageDescription.Text = info and info[2] or ""
    for key, page in pairs(Pages) do
        local active = key == name
        page.Visible = active
        animate(TabButtons[key], {BackgroundTransparency = active and 0 or 1})
        animate(TabCaptions[key], {TextColor3 = active and Colors.Text or Colors.Muted})
        animate(TabNumbers[key], {TextColor3 = active and Colors.Text or Colors.Muted})
        TabMarkers[key].Visible = active
    end
end

local function newPage(name)
    TabOrder += 1
    local info = PageInfo[name]
    local page = make("ScrollingFrame", {
        Position = UDim2.fromOffset(198, 165),
        Size = UDim2.new(1, -218, 1, -218),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ScrollBarThickness = 3,
        ScrollBarImageColor3 = Colors.Accent,
        ScrollBarImageTransparency = 0.5,
        CanvasSize = UDim2.new(),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
        ScrollingDirection = Enum.ScrollingDirection.Y,
        Visible = false,
    }, Window)
    make("UIListLayout", {
        Padding = UDim.new(0, 9),
        SortOrder = Enum.SortOrder.LayoutOrder,
    }, page)
    make("UIPadding", {
        PaddingTop = UDim.new(0, 1),
        PaddingLeft = UDim.new(0, 1),
        PaddingRight = UDim.new(0, 9),
        PaddingBottom = UDim.new(0, 10),
    }, page)
    local button = make("TextButton", {
        Size = UDim2.new(1, 0, 0, 43),
        BackgroundColor3 = Colors.Field,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
        LayoutOrder = TabOrder,
    }, Tabs)
    round(button, 8)
    local number = label(button, info and info[3] or tostring(TabOrder), 10, Colors.Muted)
    number.Position = UDim2.fromOffset(14, 0)
    number.Size = UDim2.fromOffset(24, 43)
    number.Font = Enum.Font.Code
    local caption = label(button, name, 10, Colors.Muted)
    caption.Position = UDim2.fromOffset(43, 0)
    caption.Size = UDim2.new(1, -48, 1, 0)
    caption.Font = Enum.Font.GothamSemibold
    local marker = make("Frame", {
        Position = UDim2.fromOffset(1, 11),
        Size = UDim2.fromOffset(2, 21),
        BackgroundColor3 = Colors.Text,
        BorderSizePixel = 0,
        Visible = false,
    }, button)
    round(marker, 2)
    Pages[name], TabButtons[name] = page, button
    TabMarkers[name], TabNumbers[name], TabCaptions[name] = marker, number, caption
    connect(button.Activated, function() selectPage(name) end)
    connect(button.MouseEnter, function()
        if SelectedPage ~= name then animate(button, {BackgroundTransparency = 0.55}) end
    end)
    connect(button.MouseLeave, function()
        if SelectedPage ~= name then animate(button, {BackgroundTransparency = 1}) end
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
        Size = UDim2.new(1, 0, 0, 57),
        BackgroundColor3 = Colors.Surface,
        BackgroundTransparency = 0.16,
        BorderSizePixel = 0,
        LayoutOrder = Order,
    }, parent)
    round(frame, 9)
    local stroke = outline(frame, 0.65)
    local caption = label(frame, name, 12)
    caption.Position = UDim2.fromOffset(16, 0)
    caption.Size = UDim2.new(1, -180, 1, 0)
    caption.TextWrapped = true
    caption.TextTruncate = Enum.TextTruncate.None
    connect(frame.MouseEnter, function() animate(stroke, {Transparency = 0.15}) end)
    connect(frame.MouseLeave, function() animate(stroke, {Transparency = 0.65}) end)
    return frame
end

local function refreshKey(key)
    for _, update in ipairs(Refreshers[key] or {}) do update() end
end

local function toggle(parent, name, key, callback)
    local frame = row(parent, name)
    local status = label(frame, "", 9, Colors.Muted)
    status.Position = UDim2.new(1, -112, 0, 0)
    status.Size = UDim2.new(0, 35, 1, 0)
    status.TextXAlignment = Enum.TextXAlignment.Right
    local button = make("TextButton", {
        Size = UDim2.fromOffset(43, 24),
        Position = UDim2.new(1, -59, 0.5, -12),
        BackgroundColor3 = Colors.Field,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
    }, frame)
    round(button, 12)
    local knob = make("Frame", {
        Size = UDim2.fromOffset(16, 16),
        Position = UDim2.fromOffset(4, 4),
        BackgroundColor3 = Colors.Muted,
        BorderSizePixel = 0,
    }, button)
    round(knob, 8)
    local function refresh()
        local enabled = Config[key] == true
        status.Text = enabled and "ON" or "OFF"
        animate(button, {BackgroundColor3 = enabled and Colors.Accent or Colors.Field})
        animate(knob, {
            Position = UDim2.fromOffset(enabled and 23 or 4, 4),
            BackgroundColor3 = enabled and Colors.Background or Colors.Muted,
        })
    end
    Refreshers[key] = Refreshers[key] or {}
    table.insert(Refreshers[key], refresh)
    connect(button.Activated, function()
        Config[key] = not Config[key]
        if callback then callback(Config[key]) end
        refreshKey(key)
    end)
    refresh()
end

local function numberField(parent, name, key, minimum, maximum)
    local frame = row(parent, name)
    local field = make("TextBox", {
        Size = UDim2.fromOffset(118, 32),
        Position = UDim2.new(1, -134, 0.5, -16),
        BackgroundColor3 = Colors.Background,
        BackgroundTransparency = 0.15,
        BorderSizePixel = 0,
        Text = tostring(Config[key]),
        ClearTextOnFocus = false,
        Font = Enum.Font.Code,
        TextSize = 13,
        TextColor3 = Colors.Text,
    }, frame)
    round(field, 7)
    local stroke = outline(field, 0.45)
    connect(field.Focused, function()
        animate(stroke, {Color = Colors.Accent, Transparency = 0})
    end)
    connect(field.FocusLost, function()
        local value = tonumber((field.Text:gsub(",", ".")))
        if value and value == value and value > -math.huge and value < math.huge then
            if minimum ~= nil then value = math.max(minimum, value) end
            if maximum ~= nil then value = math.min(maximum, value) end
            Config[key] = value
        end
        field.Text = tostring(Config[key])
        animate(stroke, {Color = Colors.Stroke, Transparency = 0.45})
    end)
end

local function choice(parent, name, key, options)
    local frame = row(parent, name)
    local button = make("TextButton", {
        Size = UDim2.fromOffset(140, 32),
        Position = UDim2.new(1, -156, 0.5, -16),
        BackgroundColor3 = Colors.Field,
        BorderSizePixel = 0,
        Text = tostring(Config[key]) .. "  ›",
        Font = Enum.Font.GothamMedium,
        TextSize = 11,
        TextColor3 = Colors.Text,
        AutoButtonColor = false,
    }, frame)
    round(button, 7)
    outline(button, 0.5)
    connect(button.Activated, function()
        local index = table.find(options, Config[key]) or 0
        Config[key] = options[index % #options + 1]
        button.Text = tostring(Config[key]) .. "  ›"
    end)
    connect(button.MouseEnter, function()
        animate(button, {BackgroundColor3 = Color3.fromRGB(43, 43, 43)})
    end)
    connect(button.MouseLeave, function() animate(button, {BackgroundColor3 = Colors.Field}) end)
end

local function note(parent, message)
    Order += 1
    local object = label(parent, message, 11, Colors.Muted)
    object.Size = UDim2.new(1, -8, 0, 0)
    object.AutomaticSize = Enum.AutomaticSize.Y
    object.TextWrapped = true
    object.TextTruncate = Enum.TextTruncate.None
    object.TextYAlignment = Enum.TextYAlignment.Top
    object.LayoutOrder = Order
    make("UIPadding", {
        PaddingTop = UDim.new(0, 7),
        PaddingBottom = UDim.new(0, 9),
        PaddingLeft = UDim.new(0, 3),
        PaddingRight = UDim.new(0, 3),
    }, object)
    return object
end

local function action(parent, name, callback, color)
    Order += 1
    local base = color or Colors.Field
    local button = make("TextButton", {
        Size = UDim2.new(1, 0, 0, 43),
        BackgroundColor3 = base,
        BackgroundTransparency = 0.1,
        BorderSizePixel = 0,
        Text = name,
        Font = Enum.Font.GothamSemibold,
        TextSize = 10,
        TextColor3 = Colors.Text,
        LayoutOrder = Order,
        AutoButtonColor = false,
    }, parent)
    round(button, 8)
    local stroke = outline(button, 0.35)
    connect(button.Activated, callback)
    connect(button.MouseEnter, function()
        animate(button, {BackgroundColor3 = base:Lerp(Color3.new(1, 1, 1), 0.08)})
        animate(stroke, {Transparency = 0})
    end)
    connect(button.MouseLeave, function()
        animate(button, {BackgroundColor3 = base})
        animate(stroke, {Transparency = 0.35})
    end)
    return button
end

-- Construção local e ações com jogadores.
local ExtraPage = newPage("EXTRAS")
local ExtraStatus = note(ExtraPage, "Construção local: selecione uma peça com o menu fechado.")
local function extraMessage(message)
    ExtraStatus.Text = message
end

local TargetName = ""
local TargetRow = row(ExtraPage, "Nome do jogador")
local TargetField = make("TextBox", {
    Size = UDim2.fromOffset(140, 30),
    Position = UDim2.new(1, -154, 0.5, -15),
    BackgroundColor3 = Colors.Field,
    BorderSizePixel = 0,
    Text = "",
    PlaceholderText = "Nome ou @nome",
    ClearTextOnFocus = false,
    Font = Enum.Font.Gotham,
    TextSize = 12,
    TextColor3 = Colors.Text,
}, TargetRow)
round(TargetField, 8)

local function resolveExtraPlayer()
    local query = TargetField.Text:match("^%s*(.-)%s*$"):gsub("^@", ""):lower()
    if query == "" then
        extraMessage("Digite o nome de um jogador.")
        return
    end
    local candidates = {}
    for _, other in ipairs(Players:GetPlayers()) do
        if other ~= Player then
            if other.Name:lower() == query then return other end
            if other.Name:lower():sub(1, #query) == query
                or other.DisplayName:lower():sub(1, #query) == query then
                table.insert(candidates, other)
            end
        end
    end
    if #candidates == 1 then return candidates[1] end
    extraMessage(#candidates > 1 and "Nome ambíguo: digite o @nome completo."
        or "Jogador não encontrado.")
end

local function extraRoots()
    local other = resolveExtraPlayer()
    if not other then return end
    local _, humanoid, root = characterParts()
    local otherRoot = other.Character and other.Character:FindFirstChild("HumanoidRootPart")
    if not root or not humanoid or humanoid.Health <= 0 or not otherRoot then
        extraMessage("Personagem indisponível.")
        return
    end
    return root, otherRoot, other
end

action(ExtraPage, "IR ATÉ O JOGADOR", function()
    local root, otherRoot, other = extraRoots()
    if not root then return end
    local ok, problem = pcall(function()
        root.CFrame = otherRoot.CFrame * CFrame.new(0, 0, 4)
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
    end)
    extraMessage(ok and ("TP solicitado até " .. other.Name .. ". O servidor pode corrigir.")
        or tostring(problem))
end)

action(ExtraPage, "TENTAR PUXAR JOGADOR", function()
    local root, otherRoot, other = extraRoots()
    if not root then return end
    local ok, problem = pcall(function()
        otherRoot.CFrame = root.CFrame * CFrame.new(0, 0, -5)
        otherRoot.AssemblyLinearVelocity = Vector3.zero
    end)
    extraMessage(ok and ("Movimento local tentado em " .. other.Name .. "; pode não replicar.")
        or tostring(problem))
end)

local ToolSourcesCopied = setmetatable({}, {__mode = "k"})
action(ExtraPage, "TENTAR COPIAR TOOLS ACESSÍVEIS", function()
    local backpack = Player:FindFirstChildOfClass("Backpack")
    if not backpack then extraMessage("Backpack indisponível.") return end
    local copied, failed = 0, 0
    for _, source in ipairs({game:GetService("ReplicatedStorage"), workspace}) do
        for _, item in ipairs(source:GetDescendants()) do
            if item:IsA("Tool") and not ToolSourcesCopied[item] then
                local belongsToCharacter = false
                for _, other in ipairs(Players:GetPlayers()) do
                    if other.Character and item:IsDescendantOf(other.Character) then
                        belongsToCharacter = true
                        break
                    end
                end
                if not belongsToCharacter then
                    local ok, clone = pcall(function() return item:Clone() end)
                    if ok and clone then
                        -- A cópia herda Anchored do modelo de armazenamento.
                        -- Corrige apenas suas peças antes de equipar, preservando soldas.
                        local prepared, problem = pcall(function()
                            for _, object in ipairs(clone:GetDescendants()) do
                                if object:IsA("BasePart") then
                                    object.Anchored = false
                                    object.CanCollide = false
                                end
                            end
                            clone:SetAttribute("VisionCopiedTool", true)
                            -- Scripts de servidor não passam a executar nesta cópia local.
                            clone.Parent = backpack
                        end)
                        if prepared then
                            ToolSourcesCopied[item] = true
                            copied += 1
                        else
                            clone:Destroy()
                            failed += 1
                            warn("[VISION] Falha ao preparar Tool: " .. tostring(problem))
                        end
                    else
                        failed += 1
                    end
                end
            end
        end
    end
    extraMessage("Cópias locais: " .. copied .. ". Falhas: " .. failed
        .. ". Não garante armas funcionais nem acessa ServerStorage.")
end)

Config.BuildEnabled = false
Config.BuildMode = "Mover"
Config.BuildStep = 1
Config.BuildAngle = 15
local BuildSelected
local BuildStartFrame
local BuildStartSize
local BuildDragAxis
local BuildHistory = {}
local BuildBox = make("SelectionBox", {
    Name = "VisionBuildSelection",
    Color3 = Colors.Accent,
    LineThickness = 0.04,
    SurfaceTransparency = 0.9,
}, Gui)
local BuildHandles = make("Handles", {
    Name = "VisionBuildHandles",
    Color3 = Colors.Accent,
    Style = Enum.HandlesStyle.Movement,
    Visible = false,
}, Gui)
local BuildArcs = make("ArcHandles", {
    Name = "VisionBuildRotation",
    Color3 = Colors.Accent,
    Visible = false,
}, Gui)
local function updateBuild()
    local part = BuildSelected
    local active = Config.BuildEnabled and part and part:IsDescendantOf(workspace)
    BuildBox.Adornee = active and part or nil
    BuildHandles.Adornee = active and part or nil
    BuildArcs.Adornee = active and part or nil
    BuildHandles.Visible = active and Config.BuildMode ~= "Girar" or false
    BuildArcs.Visible = active and Config.BuildMode == "Girar" or false
    BuildHandles.Style = Config.BuildMode == "Tamanho"
        and Enum.HandlesStyle.Resize or Enum.HandlesStyle.Movement
end
local function rememberBuild()
    if not BuildSelected or not BuildSelected.Parent then return end
    BuildStartFrame = BuildSelected.CFrame
    BuildStartSize = BuildSelected.Size
    table.insert(BuildHistory, {
        Part = BuildSelected, Frame = BuildStartFrame, Size = BuildStartSize,
    })
    if #BuildHistory > 100 then table.remove(BuildHistory, 1) end
end
local function snapBuild(value, step)
    step = math.max(step, 0.01)
    return math.round(value / step) * step
end

toggle(ExtraPage, "Construção local", "BuildEnabled", updateBuild)
choice(ExtraPage, "Ferramenta", "BuildMode", {"Mover", "Girar", "Tamanho"})
numberField(ExtraPage, "Passo / studs", "BuildStep", 0.01)
numberField(ExtraPage, "Passo / graus", "BuildAngle", 0.1, 180)
action(ExtraPage, "DESFAZER ÚLTIMA EDIÇÃO", function()
    local previous = table.remove(BuildHistory)
    if previous and previous.Part.Parent then
        previous.Part.Size = previous.Size
        previous.Part.CFrame = previous.Frame
        extraMessage("Última edição local desfeita.")
    end
end)
note(ExtraPage, "Feche o menu, clique na peça e arraste as setas ou anéis. Edições locais podem ser corrigidas pelo servidor.")

connect(Mouse.Button1Down, function()
    if not Config.BuildEnabled or Window.Visible or BuildDragAxis then return end
    local part = Mouse.Target
    if not part or not part:IsA("BasePart") then return end
    if Player.Character and part:IsDescendantOf(Player.Character) then return end
    BuildSelected = part
    extraMessage("Selecionada: " .. part.Name)
    updateBuild()
end)
connect(BuildHandles.MouseButton1Down, function(face)
    BuildDragAxis = face
    rememberBuild()
end)
connect(BuildHandles.MouseDrag, function(face, distance)
    if not Config.BuildEnabled or not BuildSelected or not BuildSelected.Parent
        or not BuildStartFrame or BuildDragAxis ~= face then return end
    local direction = Vector3.FromNormalId(face)
    local amount = snapBuild(distance, Config.BuildStep)
    if Config.BuildMode == "Mover" then
        BuildSelected.CFrame = BuildStartFrame * CFrame.new(direction * amount)
    elseif Config.BuildMode == "Tamanho" then
        local axis = Vector3.new(math.abs(direction.X), math.abs(direction.Y), math.abs(direction.Z))
        local original = BuildStartSize:Dot(axis)
        amount = math.max(0.05 - original, amount)
        BuildSelected.Size = BuildStartSize + axis * amount
        BuildSelected.CFrame = BuildStartFrame * CFrame.new(direction * amount * 0.5)
    end
end)
connect(BuildArcs.MouseButton1Down, function(axis)
    BuildDragAxis = axis
    rememberBuild()
end)
connect(BuildArcs.MouseDrag, function(axis, angle)
    if not Config.BuildEnabled or Config.BuildMode ~= "Girar"
        or not BuildSelected or not BuildSelected.Parent
        or not BuildStartFrame or BuildDragAxis ~= axis then return end
    local direction = axis == Enum.Axis.X and Vector3.xAxis
        or axis == Enum.Axis.Y and Vector3.yAxis or Vector3.zAxis
    local snapped = math.rad(snapBuild(math.deg(angle), Config.BuildAngle))
    BuildSelected.CFrame = BuildStartFrame * CFrame.fromAxisAngle(direction, snapped)
end)
connect(UIS.InputEnded, function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        BuildDragAxis = nil
    end
end)
connect(RunService.RenderStepped, updateBuild)


-- Visual
toggle(VisualPage, "ESP / quadrado", "ESP")
toggle(VisualPage, "Contorno branco / sem preenchimento", "Skeleton")
toggle(VisualPage, "Nickname", "Names")
toggle(VisualPage, "Incluir NPCs", "NPCs")
toggle(VisualPage, "Ignorar mesma equipe", "TeamCheck")

local ColorRow = row(VisualPage, "Cor do ESP")
local Palette = {
    Color3.fromRGB(255, 255, 255),
    Color3.fromRGB(220, 220, 220),
    Color3.fromRGB(185, 185, 185),
    Color3.fromRGB(150, 150, 150),
    Color3.fromRGB(115, 115, 115),
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
toggle(MovementPage, "Atravessar paredes", "Noclip", function(enabled)
    if not enabled then restoreCollisions() end
end)

note(MovementPage,
    "Atravessar paredes também desativa colisão com o chão. "
    .. "Use junto com o voo e desative fora das paredes."
)

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
numberField(VehiclePage, "Impulso para cima", "ThrowLift", 0)
numberField(VehiclePage, "Distância para pegar", "GrabRange", 0)

note(VehiclePage,
    "Para segurar sobre a cabeça, coloque distância à frente em 0. "
    .. "Veículos ancorados ou controlados pelo servidor podem não responder."
)

make("Frame", {
    Position = UDim2.new(0, 0, 1, -42),
    Size = UDim2.new(1, 0, 0, 42),
    BackgroundColor3 = Colors.Surface,
    BackgroundTransparency = 0.1,
    BorderSizePixel = 0,
}, Window)
make("Frame", {
    Position = UDim2.new(0, 0, 1, -42),
    Size = UDim2.new(1, 0, 0, 1),
    BackgroundColor3 = Colors.Stroke,
    BackgroundTransparency = 0.4,
    BorderSizePixel = 0,
}, Window)
local Footer = label(
    Window,
    "CTRL + ASPAS: MENU   /   CTRL + BACKSPACE: DESATIVAR",
    9,
    Colors.Muted
)
Footer.Position = UDim2.new(0, 18, 1, -30)
Footer.Size = UDim2.new(1, -36, 0, 18)

local function showMenu(visible)
    Window.Visible = visible
end

-- Sem botão flutuante; abra somente pelo atalho.
showMenu(false)
connect(Hide.MouseButton1Click, function()
    showMenu(false)
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

local function deactivateAll()
    -- Desliga todas as opções booleanas, mantendo valores numéricos.
    for key, value in pairs(Config) do
        if type(value) == "boolean" then
            Config[key] = false
            refreshKey(key)
        end
    end
    Aiming = false
    LockedPart = nil
    Dragging = false
    DragTouch = nil
    BuildDragAxis = nil
    BuildSelected = nil
    stopFlight()
    releaseVehicle()
    restoreCollisions()
    updateBuild()
    if SpeedApplied and SpeedHumanoid and SpeedHumanoid.Parent
        and SavedSpeed then
        SpeedHumanoid.WalkSpeed = SavedSpeed
    end
    SpeedApplied = false
    for _, visual in pairs(Visuals) do
        hideVisual(visual)
    end
    showMenu(false)
    extraMessage("Todas as opções desligadas. Ações anteriores não são desfeitas.")
end

local function controlHeld()
    return UIS:IsKeyDown(Enum.KeyCode.LeftControl)
        or UIS:IsKeyDown(Enum.KeyCode.RightControl)
end

local function quoteKey(input)
    if input.KeyCode == Enum.KeyCode.Quote
        or input.KeyCode == Enum.KeyCode.QuotedDouble then
        return true
    end
    -- Permite o mapeamento local das aspas em outros layouts.
    local ok, key = pcall(function()
        return UIS:GetStringForKeyCode(input.KeyCode)
    end)
    return ok and (key == "\"" or key == "'")
end

connect(UIS.InputBegan, function(input, processed)
    -- O atalho de desligamento funciona inclusive ao editar um campo.
    if controlHeld() and input.KeyCode == Enum.KeyCode.Backspace then
        deactivateAll()
        return
    end
    if UIS:GetFocusedTextBox() then return end

    if controlHeld() and quoteKey(input) then
        showMenu(not Window.Visible)
        return
    end

    if processed or controlHeld() then return end

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
            math.max(0.1, (viewport.X - 24) / 760),
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
                    visual.Box.Visible = not Config.Skeleton

                    local owner = Players:GetPlayerFromCharacter(model)

                    visual.Name.Text = owner and owner.Name or model.Name
                    visual.Name.Position = Vector2.new(
                        position.X + size.X * 0.5,
                        position.Y - 18
                    )
                    visual.Name.Color = Config.Color
                    visual.Name.Visible = Config.Names

                    if Config.Skeleton then
                        for index, name in ipairs(BodyParts) do
                            local part = model:FindFirstChild(name)
                            if part and part:IsA("BasePart") then
                                local outline = visual.PartOutlines[index]
                                if outline and outline.Adornee ~= part then
                                    outline:Destroy()
                                    outline = nil
                                end
                                if not outline then
                                    outline = make("Highlight", {
                                        Name = "VisionBodyOutline",
                                        Adornee = part,
                                        FillTransparency = 1,
                                        OutlineTransparency = 0,
                                        OutlineColor = Color3.new(1, 1, 1),
                                        DepthMode = Enum.HighlightDepthMode.AlwaysOnTop,
                                    }, part)
                                    visual.PartOutlines[index] = outline
                                end
                                outline.Enabled = true
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
            or FlightAttachment.Parent ~= root then
            startFlight(root, humanoid)
        end

        local direction = Vector3.zero

        if not UIS:GetFocusedTextBox() then
            if UIS:IsKeyDown(Enum.KeyCode.W) then
                direction += camera.CFrame.LookVector
            end
            if UIS:IsKeyDown(Enum.KeyCode.S) then
                direction -= camera.CFrame.LookVector
            end
            if UIS:IsKeyDown(Enum.KeyCode.D) then
                direction += camera.CFrame.RightVector
            end
            if UIS:IsKeyDown(Enum.KeyCode.A) then
                direction -= camera.CFrame.RightVector
            end
            if UIS:IsKeyDown(Enum.KeyCode.Space) then
                direction += Vector3.yAxis
            end
            if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then
                direction -= Vector3.yAxis
            end
        end

        FlightVelocity.VectorVelocity = direction.Magnitude > 0
            and direction.Unit * Config.FlySpeed
            or Vector3.zero

        root.AssemblyAngularVelocity = Vector3.zero
    else
        stopFlight()
    end

    if HeldVehicle then
        if not alive
            or not HeldVehicle.Part:IsDescendantOf(workspace)
            or not HeldVehicle.Model:IsDescendantOf(workspace)
            or not HeldVehicle.Position.Parent
            or not HeldVehicle.Orientation.Parent then

            releaseVehicle()
            vehicleMessage("Veículo liberado.")
        else
            local forward = vehicleForward(root)
            HeldVehicle.Position.Position =
                root.Position
                + Vector3.new(0, Config.VehicleHeight, 0)
                + forward * Config.VehicleDistance
            HeldVehicle.Orientation.CFrame =
                CFrame.lookAt(Vector3.zero, forward)
        end
    end
end)

-- =========================================================
-- ENCERRAMENTO
-- =========================================================

function Controller.Stop()
    if Stopped then return end
    Stopped = true
    Config.BuildEnabled = false
    BuildSelected = nil
    table.clear(BuildHistory)
    Config.Noclip = false
    restoreCollisions()

    LockedPart = nil
    stopFlight()
    releaseVehicle()

    RunService:UnbindFromRenderStep(Binding)
    RunService:UnbindFromRenderStep(Binding .. "Recoil")

    for _, connection in ipairs(Connections) do
        connection:Disconnect()
    end

    for _, visual in pairs(Visuals) do
        removeVisual(visual)
    end

    if SpeedApplied and SpeedHumanoid
        and SpeedHumanoid.Parent and SavedSpeed then
        SpeedHumanoid.WalkSpeed = SavedSpeed
    end

    table.clear(Visuals)
    table.clear(Targets)
    for _, tween in pairs(UITweens) do tween:Cancel() end
    table.clear(UITweens)
    Gui:Destroy()

    if Env.VisionController == Controller then
        Env.VisionController = nil
    end
end

action(
    MovementPage,
    "ENCERRAR SCRIPT",
    Controller.Stop,
    Color3.fromRGB(42, 42, 42)
)

selectPage("VISUAL")

print("[VISION] Carregado.")
print("Ctrl + aspas: menu | Ctrl + Backspace: desativar todas as opções")
