-- Cosmic Hub | Fluxo PvP
-- 100% tudo desativado
-- Delta Compatible

local Library = loadstring(game:HttpGet("https://sirius.menu/gen2"))()
local Window = Library:CreateWindow({
    Name = "Cosmic Hub",
    Themeable = {
        Info = "Fluxo PvP | Cosmic Hub"
    }
})

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

local Settings = {
    SilentAim = false,
    FOV = 120,
    TeamCheck = false,
    VisCheck = false,
    ESP = false,
    ESPBox = false,
    ESPTracer = false,
    ESPName = false,
    InfiniteAmmo = false,
    Hitbox = false,
    HitboxSize = 8,
    Speed = false,
    SpeedValue = 30,
    Fly = false,
    FlySpeed = 50
}

local Connections = {}
local ESPObjects = {}

local function IsAlive(char)
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0
end

local function IsVisible(char)
    if not Settings.VisCheck then return true end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return false end
    local origin = Camera.CFrame.Position
    local direction = hrp.Position - origin
    local params = RaycastParams.new()
    params.FilterDescendantsInstances = {LocalPlayer.Character, Camera}
    params.FilterType = Enum.RaycastFilterType.Blacklist
    local result = Workspace:Raycast(origin, direction, params)
    if result then
        return result.Instance:IsDescendantOf(char)
    end
    return true
end

local function GetClosestPlayer()
    local closest, shortest = nil, Settings.FOV
    local center = Camera.ViewportSize / 2
    for _, plr in pairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        if Settings.TeamCheck and plr.Team == LocalPlayer.Team then continue end
        local char = plr.Character
        if not IsAlive(char) then continue end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end
        local pos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
        if not onScreen then continue end
        local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
        if dist < shortest and IsVisible(char) then
            shortest = dist
            closest = plr
        end
    end
    return closest
end

local function SilentAim()
    if not Settings.SilentAim then return end
    local target = GetClosestPlayer()
    if target and target.Character then
        local hrp = target.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            local mt = getrawmetatable(game)
            local old = mt.__namecall
            setreadonly(mt, false)
            mt.__namecall = newcclosure(function(self, ...)
                local args = {...}
                local method = getnamecallmethod()
                if method == "FireServer" or method == "InvokeServer" then
                    if typeof(args[1]) == "Vector3" or typeof(args[2]) == "Vector3" then
                        args[1] = hrp.Position
                    end
                end
                return old(self, unpack(args))
            end)
            setreadonly(mt, true)
        end
    end
end

local function CreateESP(player)
    if ESPObjects[player] then return end
    local box = Drawing.new("Square")
    box.Thickness = 1
    box.Filled = false
    box.Color = Color3.fromRGB(0, 255, 128)
    box.Visible = false
    local tracer = Drawing.new("Line")
    tracer.Thickness = 1
    tracer.Color = Color3.fromRGB(0, 255, 128)
    tracer.Visible = false
    local name = Drawing.new("Text")
    name.Size = 14
    name.Center = true
    name.Outline = true
    name.Color = Color3.fromRGB(255, 255, 255)
    name.Visible = false
    ESPObjects[player] = {Box = box, Tracer = tracer, Name = name}
end

local function RemoveESP(player)
    local data = ESPObjects[player]
    if data then
        data.Box:Remove()
        data.Tracer:Remove()
        data.Name:Remove()
        ESPObjects[player] = nil
    end
end

local function UpdateESP()
    if not Settings.ESP then
        for plr, _ in pairs(ESPObjects) do
            RemoveESP(plr)
        end
        return
    end
    for _, plr in pairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        CreateESP(plr)
        local data = ESPObjects[plr]
        local char = plr.Character
        if not IsAlive(char) then
            data.Box.Visible = false
            data.Tracer.Visible = false
            data.Name.Visible = false
            continue
        end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local head = char:FindFirstChild("Head")
        if not hrp or not head then continue end
        local pos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
        local headPos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
        local footPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))
        if onScreen then
            local height = math.abs(headPos.Y - footPos.Y)
            local width = height / 2
            data.Box.Size = Vector2.new(width, height)
            data.Box.Position = Vector2.new(pos.X - width/2, pos.Y - height/2)
            data.Box.Visible = Settings.ESPBox
            data.Tracer.From = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y)
            data.Tracer.To = Vector2.new(pos.X, pos.Y)
            data.Tracer.Visible = Settings.ESPTracer
            data.Name.Text = plr.Name
            data.Name.Position = Vector2.new(pos.X, headPos.Y - 15)
            data.Name.Visible = Settings.ESPName
        else
            data.Box.Visible = false
            data.Tracer.Visible = false
            data.Name.Visible = false
        end
    end
end

local function InfiniteAmmo()
    if not Settings.InfiniteAmmo then return end
    local char = LocalPlayer.Character
    if not char then return end
    for _, tool in pairs(char:GetChildren()) do
        if tool:IsA("Tool") then
            pcall(function()
                if tool:FindFirstChild("Ammo") then tool.Ammo.Value = 999 end
                if tool:FindFirstChild("Clip") then tool.Clip.Value = 999 end
                if tool:FindFirstChild("MaxAmmo") then tool.MaxAmmo.Value = 999 end
            end)
        end
    end
end

local function UpdateHitbox()
    for _, plr in pairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        local char = plr.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            local hrp = char.HumanoidRootPart
            if Settings.Hitbox then
                hrp.Size = Vector3.new(Settings.HitboxSize, Settings.HitboxSize, Settings.HitboxSize)
                hrp.Transparency = 0.7
                hrp.CanCollide = false
            else
                hrp.Size = Vector3.new(2, 2, 1)
                hrp.Transparency = 1
            end
        end
    end
end

local function UpdateSpeed()
    local char = LocalPlayer.Character
    if char and char:FindFirstChildOfClass("Humanoid") then
        char.Humanoid.WalkSpeed = Settings.Speed and Settings.SpeedValue or 16
    end
end

local FlyBV, FlyBG
local function UpdateFly(state)
    local char = LocalPlayer.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    if state then
        FlyBV = Instance.new("BodyVelocity")
        FlyBV.MaxForce = Vector3.new(9e9, 9e9, 9e9)
        FlyBV.Velocity = Vector3.zero
        FlyBV.Parent = hrp
        FlyBG = Instance.new("BodyGyro")
        FlyBG.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
        FlyBG.P = 9e4
        FlyBG.Parent = hrp
        Connections.Fly = RunService.RenderStepped:Connect(function()
            if not Settings.Fly then return end
            local move = Vector3.zero
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then move += Camera.CFrame.LookVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then move -= Camera.CFrame.LookVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then move -= Camera.CFrame.RightVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then move += Camera.CFrame.RightVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then move += Vector3.new(0,1,0) end
            if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then move -= Vector3.new(0,1,0) end
            if move.Magnitude > 0 then move = move.Unit end
            FlyBV.Velocity = move * Settings.FlySpeed
            FlyBG.CFrame = Camera.CFrame
        end)
    else
        if Connections.Fly then Connections.Fly:Disconnect() end
        if FlyBV then FlyBV:Destroy() end
        if FlyBG then FlyBG:Destroy() end
    end
end

local CombatTab = Window:CreateTab({Name = "Combat"})
local VisualTab = Window:CreateTab({Name = "Visuals"})
local PlayerTab = Window:CreateTab({Name = "Player"})
local SettingsTab = Window:CreateTab({Name = "Settings"})

CombatTab:CreateToggle({
    Name = "Silent Aim",
    CurrentValue = false,
    Callback = function(v) Settings.SilentAim = v end
})

CombatTab:CreateToggle({
    Name = "Team Check",
    CurrentValue = false,
    Callback = function(v) Settings.TeamCheck = v end
})

CombatTab:CreateToggle({
    Name = "Visibility Check",
    CurrentValue = false,
    Callback = function(v) Settings.VisCheck = v end
})

CombatTab:CreateSlider({
    Name = "FOV",
    Range = {50, 400},
    Increment = 5,
    CurrentValue = 120,
    Callback = function(v) Settings.FOV = v end
})

CombatTab:CreateToggle({
    Name = "Hitbox Expander",
    CurrentValue = false,
    Callback = function(v) Settings.Hitbox = v end
})

CombatTab:CreateSlider({
    Name = "Hitbox Size",
    Range = {3, 20},
    Increment = 1,
    CurrentValue = 8,
    Callback = function(v) Settings.HitboxSize = v end
})

CombatTab:CreateToggle({
    Name = "Munição Infinita",
    CurrentValue = false,
    Callback = function(v) Settings.InfiniteAmmo = v end
})

VisualTab:CreateToggle({
    Name = "ESP Master",
    CurrentValue = false,
    Callback = function(v) Settings.ESP = v end
})

VisualTab:CreateToggle({
    Name = "ESP Box",
    CurrentValue = false,
    Callback = function(v) Settings.ESPBox = v end
})

VisualTab:CreateToggle({
    Name = "ESP Tracer",
    CurrentValue = false,
    Callback = function(v) Settings.ESPTracer = v end
})

VisualTab:CreateToggle({
    Name = "ESP Name",
    CurrentValue = false,
    Callback = function(v) Settings.ESPName = v end
})

PlayerTab:CreateToggle({
    Name = "Speed Hack",
    CurrentValue = false,
    Callback = function(v)
        Settings.Speed = v
        UpdateSpeed()
    end
})

PlayerTab:CreateSlider({
    Name = "Speed Value",
    Range = {16, 100},
    Increment = 1,
    CurrentValue = 30,
    Callback = function(v)
        Settings.SpeedValue = v
        if Settings.Speed then UpdateSpeed() end
    end
})

PlayerTab:CreateToggle({
    Name = "Fly",
    CurrentValue = false,
    Callback = function(v)
        Settings.Fly = v
        UpdateFly(v)
    end
})

PlayerTab:CreateSlider({
    Name = "Fly Speed",
    Range = {20, 150},
    Increment = 5,
    CurrentValue = 50,
    Callback = function(v) Settings.FlySpeed = v end
})

SettingsTab:CreateLabel({Name = "Cosmic Hub | Fluxo PvP"})
SettingsTab:CreateLabel({Name = "100% tudo desativado"})

RunService.RenderStepped:Connect(function()
    UpdateESP()
    UpdateHitbox()
    InfiniteAmmo()
    if Settings.SilentAim then
        SilentAim()
    end
end)

Players.PlayerRemoving:Connect(function(plr)
    RemoveESP(plr)
end)

print("Cosmic Hub carregado | Tudo false")
