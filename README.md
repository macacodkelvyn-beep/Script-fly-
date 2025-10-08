# Script-fly-
-- FlySystem.lua (LocalScript em StarterPlayerScripts)
-- Controle: E = toggle, W/A/S/D = mover relativo à câmera, Space = sobe, LeftControl = desce
-- Configurações
local TOGGLE_KEY = Enum.KeyCode.E
local ASCEND_KEY = Enum.KeyCode.Space
local DESCEND_KEY = Enum.KeyCode.LeftControl

local TOP_SPEED = 100          -- velocidade máxima (studs/s)
local CRUISE_SPEED = 50        -- velocidade padrão ao andar (studs/s)
local ACCELERATION = 8         -- quão rápido atinge velocidade alvo
local VERTICAL_SPEED = 60      -- velocidade vertical ao subir/descer
local SMOOTHING = 10           -- suavização do movimento (maior = mais rígido)
local ENABLE_PLATFORM_STAND = true -- desativa animações quando voando

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local flying = false
local currentConnection = nil

-- Helper
local function getCharacterParts()
    local character = player.Character
    if not character then return nil end
    local root = character:FindFirstChild("HumanoidRootPart")
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if root and humanoid then
        return character, root, humanoid
    end
    return nil
end

local function clamp(v, a, b) return math.max(a, math.min(b, v)) end

-- Calcula entrada do jogador em cada frame
local function startFlyLoop()
    if currentConnection then return end
    local character, root, humanoid = getCharacterParts()
    if not (character and root and humanoid) then return end

    if ENABLE_PLATFORM_STAND then
        pcall(function() humanoid.PlatformStand = true end)
    end

    local velocity = root.Velocity

    currentConnection = RunService.RenderStepped:Connect(function(dt)
        -- atualiza referências (caso character seja trocado)
        if not player.Character or player.Character ~= character then
            -- character mudou/died
            flying = false
            if currentConnection then
                currentConnection:Disconnect()
                currentConnection = nil
            end
            if humanoid and ENABLE_PLATFORM_STAND then
                pcall(function() humanoid.PlatformStand = false end)
            end
            return
        end

        -- leitura de inputs
        local moveVector = Vector3.new(0,0,0)
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveVector = moveVector + Vector3.new(0,0,-1) end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveVector = moveVector + Vector3.new(0,0,1) end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveVector = moveVector + Vector3.new(-1,0,0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveVector = moveVector + Vector3.new(1,0,0) end

        local ascend = UserInputService:IsKeyDown(ASCEND_KEY) and 1 or 0
        local descend = UserInputService:IsKeyDown(DESCEND_KEY) and -1 or 0
        local verticalInput = ascend + descend

        -- converte movimento relativo à câmera
        local camCFrame = camera.CFrame
        local forward = Vector3.new(camCFrame.LookVector.X, 0, camCFrame.LookVector.Z).Unit
        if forward ~= forward then forward = Vector3.new(0,0,-1) end -- fallback
        local right = Vector3.new(camCFrame.RightVector.X, 0, camCFrame.RightVector.Z).Unit
        if right ~= right then right = Vector3.new(1,0,0) end

        local horizontalDir = (forward * -moveVector.Z) + (right * moveVector.X) -- note: W maps to -Z
        if horizontalDir.Magnitude > 1 then horizontalDir = horizontalDir.Unit end

        -- velocidade alvo
        local targetHorizontalSpeed = CRUISE_SPEED
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
            targetHorizontalSpeed = TOP_SPEED -- sprint while holding shift
        end

        local targetVel = horizontalDir * targetHorizontalSpeed
        targetVel = Vector3.new(targetVel.X, 0, targetVel.Z)

        -- vertical
        local targetY = verticalInput * VERTICAL_SPEED

        local desiredVelocity = Vector3.new(targetVel.X, targetY, targetVel.Z)

        -- suaviza e aplica
        local lerpAlpha = clamp(ACCELERATION * dt, 0, 1)
        local newVel = root.Velocity:Lerp(desiredVelocity, lerpAlpha * (SMOOTHING/10))

        -- mantém um pouco de inércia na direção atual do jogador (melhora sensação)
        root.Velocity = Vector3.new(newVel.X, newVel.Y, newVel.Z)
    end)
end

local function stopFlyLoop()
    if currentConnection then
        currentConnection:Disconnect()
        currentConnection = nil
    end
    local _, _, humanoid = getCharacterParts()
    if humanoid and ENABLE_PLATFORM_STAND then
        pcall(function() humanoid.PlatformStand = false end)
    end
end

-- Toggle de fly
local function toggleFly()
    flying = not flying
    if flying then
        startFlyLoop()
    else
        stopFlyLoop()
    end
end

-- Input handler para tecla de toggle (pode também usar GUI)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == TOGGLE_KEY then
            toggleFly()
        end
    end
end)

-- Recomeçar loop quando o personagem reaparecer
player.CharacterAdded:Connect(function(char)
    -- garante que voar seja desligado ao reaparecer
    if flying then
        flying = false
        stopFlyLoop()
    end
end)

-- Fallback: desliga ao fechar sessão
game:BindToClose(function()
    stopFlyLoop()
end)
