local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
   Name = "AimBot By:@Kevinnofc",
   LoadingTitle = "Loading Systems...",
   LoadingSubtitle = "@Kevinnofc",
   ConfigurationSaving = {
      Enabled = true,
      FolderName = "AIMBOT",
      FileName = "MainConfig"
   }
})

-- --- SERVICES & VARIABLES ---
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local UserInputService = game:GetService("UserInputService")

local AimbotEnabled = false
local TurnBotEnabled = false
local ESPEnabled = false
local TeamCheck = true
local Smoothness = 0.2
local FOVRadius = 150
local ShowFOV = true

-- --- FOV CIRCLE DRAWING ---
local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness = 1
FOVCircle.Color = Color3.fromRGB(255, 255, 255)
FOVCircle.Transparency = 0.7
FOVCircle.Visible = false

-- --- UTILITIES ---
local function isEnemy(player)
    if not TeamCheck then return true end
    if not player.Team or not LocalPlayer.Team then return true end
    return player.Team ~= LocalPlayer.Team
end

local function isVisible(targetPart)
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {LocalPlayer.Character, targetPart.Parent}
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    local result = workspace:Raycast(Camera.CFrame.Position, (targetPart.Position - Camera.CFrame.Position), raycastParams)
    return result == nil
end

-- --- ESP GESTOR (evita conexões múltiplas e vazamento) ---
local playerBillboards = {}
local function setupESP(player)
    if playerBillboards[player] then return end -- já configurado

    local billboard
    local function createBillboard(head)
        if billboard and billboard.Parent == head then
            return billboard -- já anexado
        end
        -- Remove anterior se existir
        if billboard then
            billboard:Destroy()
        end
        billboard = Instance.new("BillboardGui", head)
        billboard.AlwaysOnTop = true
        billboard.Size = UDim2.new(0, 100, 0, 50)
        billboard.StudsOffset = Vector3.new(0, 2, 0)
        local label = Instance.new("TextLabel", billboard)
        label.BackgroundTransparency = 1
        label.Size = UDim2.new(1, 0, 1, 0)
        label.TextColor3 = Color3.fromRGB(255, 50, 50)
        label.Font = Enum.Font.Code
        label.TextStrokeTransparency = 0
        return billboard
    end

    local function onCharacterAdded(char)
        local head = char:WaitForChild("Head", 5)
        if head then
            createBillboard(head)
        end
    end

    player.CharacterAdded:Connect(onCharacterAdded)
    if player.Character then
        onCharacterAdded(player.Character)
    end

    playerBillboards[player] = true
end

-- Preenche jogadores atuais
for _, p in pairs(Players:GetPlayers()) do
    if p ~= LocalPlayer then setupESP(p) end
end
Players.PlayerAdded:Connect(function(p)
    if p ~= LocalPlayer then setupESP(p) end
end)
-- Limpeza quando jogador sai
Players.PlayerRemoving:Connect(function(p)
    playerBillboards[p] = nil
end)

-- Loop central do ESP (roda a cada 0.2s, sem sobrecarregar)
RunService.Heartbeat:Connect(function()
    local now = tick()
    if not ESPEnabled then return end

    -- Throttle para 5 updates por segundo
    if (espLastUpdate or 0) + 0.2 > now then return end
    espLastUpdate = now

    for _, player in pairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        local char = player.Character
        if not char or not char:IsDescendantOf(workspace) then continue end
        local head = char:FindFirstChild("Head")
        if not head then continue end

        -- Procura a BillboardGui no head
        local billboard = head:FindFirstChildOfClass("BillboardGui")
        if not billboard then continue end

        -- Atualiza visibilidade e texto
        billboard.Enabled = ESPEnabled and isEnemy(player)
        if billboard.Enabled then
            local dist = math.floor((head.Position - Camera.CFrame.Position).Magnitude)
            local label = billboard:FindFirstChildOfClass("TextLabel")
            if label then
                label.Text = string.format("%s\n[%dm]", player.Name, dist)
            end
        end
    end
end)

-- --- UI TABS ---
local MainTab = Window:CreateTab("Main", 4483362458)

MainTab:CreateSection("Aimbot & TurnBot")
MainTab:CreateToggle({
   Name = "Enable Head-Lock",
   CurrentValue = false,
   Callback = function(Value) AimbotEnabled = Value end,
})

MainTab:CreateToggle({
   Name = "Enable Smart TurnBot",
   CurrentValue = false,
   Info = "Only turns if an enemy behind you is NOT in cover",
   Callback = function(Value) TurnBotEnabled = Value end,
})

MainTab:CreateSlider({
   Name = "FOV Size",
   Range = {50, 800},
   Increment = 10,
   CurrentValue = 150,
   Callback = function(Value) FOVRadius = Value end,
})

MainTab:CreateSlider({
   Name = "Smoothness",
   Range = {0.1, 1},
   Increment = 0.1,
   CurrentValue = 0.2,
   Callback = function(Value) Smoothness = Value end,
})

MainTab:CreateSection("Visuals")
MainTab:CreateToggle({
   Name = "Show FOV Circle",
   CurrentValue = true,
   Callback = function(Value) ShowFOV = Value end,
})

MainTab:CreateToggle({
   Name = "Enable ESP",
   CurrentValue = false,
   Callback = function(Value) ESPEnabled = Value end,
})

MainTab:CreateToggle({
   Name = "Team Check",
   CurrentValue = true,
   Callback = function(Value) TeamCheck = Value end,
})

-- --- MAIN RENDER LOOP OTIMIZADO ---
RunService.RenderStepped:Connect(function()
    -- Atualiza FOV circle
    FOVCircle.Position = UserInputService:GetMouseLocation()
    FOVCircle.Radius = FOVRadius
    FOVCircle.Visible = ShowFOV and AimbotEnabled

    if not (AimbotEnabled or TurnBotEnabled) then return end

    -- Pré-calcula referências fixas do frame
    local camCF = Camera.CFrame
    local camPos = camCF.Position
    local camLook = camCF.LookVector
    local mousePos = UserInputService:GetMouseLocation()

    local bestTarget = nil
    local targetIsBehind = false

    -- Itera só em personagens válidos
    for _, player in pairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        if not isEnemy(player) then continue end

        local char = player.Character
        if not char then continue end
        local head = char:FindFirstChild("Head")
        local hum = char:FindFirstChild("Humanoid")
        if not head or not hum or hum.Health <= 0 then continue end

        -- Vetor relativo (cabeça - câmera)
        local relVector = head.Position - camPos
        local relDir = relVector.Unit
        local isBehind = camLook:Dot(relDir) < 0

        -- PRIORIDADE 1: Aimbot (apenas se na tela e dentro da FOV)
        if AimbotEnabled then
            local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
            if onScreen then
                local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                if dist < FOVRadius then
                    -- SÓ FAZ RAYCAST SE O ALVO JÁ É CANDIDATO
                    if isVisible(head) then
                        bestTarget = head
                        targetIsBehind = false
                        break -- encontrou alvo visível na frente, para loop
                    end
                end
            end
        end

        -- PRIORIDADE 2: TurnBot (apenas se atrás e ainda não temos alvo)
        if TurnBotEnabled and isBehind and not bestTarget then
            -- SÓ FAZ RAYCAST SE ESTÁ ATRÁS
            if isVisible(head) then
                bestTarget = head
                targetIsBehind = true
                -- Não dá break, deixa continuar procurando alguém na frente (Aimbot)
                -- mas evita sobrescrever com outro atrás
            end
        end
    end

    if bestTarget then
        local targetCF = CFrame.new(camPos, bestTarget.Position)
        Camera.CFrame = Camera.CFrame:Lerp(targetCF, Smoothness)
    end
end)

Rayfield:LoadConfiguration()
