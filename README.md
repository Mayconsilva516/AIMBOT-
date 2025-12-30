-- Script de Exemplo para Roblox: ESP (Highlight), Visualização de FOV, Aimbot e Kill All (Loop Kill)
-- Este script é destinado a fins educacionais para aprender sobre GUI, Highlights, RunService, Câmera e Manipulação de Personagem.
-- Coloque este código em um LocalScript dentro de StarterPlayerScripts ou StarterGui.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Configurações
local SETTINGS = {
    ESPEnabled = false,
    FOVEnabled = false,
    AimbotEnabled = false,
    FOVRadius = 150, -- Tamanho do círculo de FOV (Raio em pixels)
    DetectionRange = 1000, -- Distância máxima para detectar/highlight
    AimbotSmoothness = 0.2 -- Suavidade do Aimbot (0 a 1, onde 1 é instantâneo)
}

-- Variáveis de controle
local highlights = {} -- Armazena as instâncias de Highlight
local fovCircle = nil -- Referência ao círculo de FOV
local espButtonRef = nil 
local aimbotButtonRef = nil

-- 1. Criação da Interface (GUI)
local function createInterface()
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "EspFovGui"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

    -- Frame Principal
    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Size = UDim2.new(0, 200, 0, 140) -- Reduzido
    MainFrame.Position = UDim2.new(0, 10, 0.5, -70)
    MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    MainFrame.BorderSizePixel = 0
    MainFrame.Active = true
    MainFrame.Parent = ScreenGui

    -- Função de arrastar personalizada
    local dragging, dragInput, dragStart, startPos
    local function update(input)
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
    
    MainFrame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = MainFrame.Position
            
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    
    MainFrame.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            update(input)
        end
    end)

    -- Título
    local Title = Instance.new("TextLabel")
    Title.Text = "MKZZ STORE V1"
    Title.Size = UDim2.new(1, 0, 0, 30)
    Title.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    Title.TextColor3 = Color3.fromRGB(255, 255, 255)
    Title.Font = Enum.Font.SourceSansBold
    Title.TextSize = 18
    Title.Parent = MainFrame

    -- Botão Toggle ESP (Tecla X)
    local EspButton = Instance.new("TextButton")
    EspButton.Name = "EspButton"
    EspButton.Size = UDim2.new(0.9, 0, 0, 35)
    EspButton.Position = UDim2.new(0.05, 0, 0, 40)
    EspButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50) -- Vermelho (Desligado)
    EspButton.Text = "ESP (X): OFF"
    EspButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    EspButton.Font = Enum.Font.SourceSans
    EspButton.TextSize = 16
    EspButton.Parent = MainFrame
    espButtonRef = EspButton

    -- Botão Toggle Aimbot (Tecla V)
    local AimbotButton = Instance.new("TextButton")
    AimbotButton.Name = "AimbotButton"
    AimbotButton.Size = UDim2.new(0.9, 0, 0, 35)
    AimbotButton.Position = UDim2.new(0.05, 0, 0, 85)
    AimbotButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50) -- Vermelho (Desligado)
    AimbotButton.Text = "AIMBOT (V): OFF"
    AimbotButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    AimbotButton.Font = Enum.Font.SourceSans
    AimbotButton.TextSize = 16
    AimbotButton.Parent = MainFrame
    aimbotButtonRef = AimbotButton

    -- Círculo FOV (Visual)
    fovCircle = Instance.new("Frame")
    fovCircle.Name = "FOVCircle"
    fovCircle.AnchorPoint = Vector2.new(0.5, 0.5)
    fovCircle.Position = UDim2.new(0.5, 0, 0.5, 0)
    fovCircle.Size = UDim2.new(0, SETTINGS.FOVRadius * 2, 0, SETTINGS.FOVRadius * 2)
    fovCircle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    fovCircle.BackgroundTransparency = 1
    fovCircle.BorderSizePixel = 0
    fovCircle.Visible = false
    fovCircle.Parent = ScreenGui
    
    -- Deixar o frame redondo
    local uiCorner = Instance.new("UICorner")
    uiCorner.CornerRadius = UDim.new(1, 0)
    uiCorner.Parent = fovCircle
    
    -- Borda (Stroke) para ficar mais visível
    local uiStroke = Instance.new("UIStroke")
    uiStroke.Color = Color3.fromRGB(255, 0, 0)
    uiStroke.Thickness = 2
    uiStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    uiStroke.Parent = fovCircle
    
    -- Funções para alternar estados
    local function toggleESP()
        SETTINGS.ESPEnabled = not SETTINGS.ESPEnabled
        if SETTINGS.ESPEnabled then
            EspButton.Text = "ESP (X): ON"
            EspButton.BackgroundColor3 = Color3.fromRGB(50, 200, 50) -- Verde
        else
            EspButton.Text = "ESP (X): OFF"
            EspButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50) -- Vermelho
            clearHighlights()
        end
    end

    local function toggleAimbot()
        SETTINGS.AimbotEnabled = not SETTINGS.AimbotEnabled
        if SETTINGS.AimbotEnabled then
            AimbotButton.Text = "AIMBOT (V): ON"
            AimbotButton.BackgroundColor3 = Color3.fromRGB(50, 200, 50) -- Verde
            -- Ativa o FOV automaticamente se o Aimbot for ativado
            if not SETTINGS.FOVEnabled then 
                SETTINGS.FOVEnabled = true
                fovCircle.Visible = true
            end
        else
            AimbotButton.Text = "AIMBOT (V): OFF"
            AimbotButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50) -- Vermelho
            -- Desativa o FOV automaticamente se o Aimbot for desativado
            if SETTINGS.FOVEnabled then 
                SETTINGS.FOVEnabled = false
                fovCircle.Visible = false
            end
        end
    end

    -- Eventos dos Botões
    EspButton.MouseButton1Click:Connect(toggleESP)
    AimbotButton.MouseButton1Click:Connect(toggleAimbot)

    -- Eventos de Teclado
    UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.KeyCode == Enum.KeyCode.X then
            toggleESP()
        elseif input.KeyCode == Enum.KeyCode.V then
            toggleAimbot()
        elseif input.KeyCode == Enum.KeyCode.LeftAlt then
            if UserInputService.MouseBehavior == Enum.MouseBehavior.LockCenter then
                UserInputService.MouseBehavior = Enum.MouseBehavior.Default
            else
                UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
            end
        end
    end)
end

-- 2. Funções de Highlight (ESP)
local function addHighlight(character)
    if not character then return end
    if highlights[character] then return end -- Já tem highlight

    local highlight = Instance.new("Highlight")
    highlight.Name = "ESPHighlight"
    highlight.FillColor = Color3.fromRGB(255, 0, 0)
    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
    highlight.FillTransparency = 0.5
    highlight.OutlineTransparency = 0
    highlight.Parent = character
    
    highlights[character] = highlight
end

local function removeHighlight(character)
    if highlights[character] then
        highlights[character]:Destroy()
        highlights[character] = nil
    end
end

function clearHighlights()
    for char, highlight in pairs(highlights) do
        highlight:Destroy()
    end
    highlights = {}
end

-- Função para pegar o jogador mais próximo do centro da tela (dentro do FOV)
local function getClosestPlayerToCenter()
    local closestPlayer = nil
    local shortestDistance = SETTINGS.FOVRadius

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local char = player.Character
            local rootPart = char:FindFirstChild("HumanoidRootPart")
            local humanoid = char:FindFirstChild("Humanoid")
            
            if rootPart and humanoid and humanoid.Health > 0 then
                local screenPos, onScreen = Camera:WorldToViewportPoint(rootPart.Position)
                
                if onScreen then
                    local mouseLocation = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
                    local targetPos = Vector2.new(screenPos.X, screenPos.Y)
                    local distanceToCenter = (targetPos - mouseLocation).Magnitude
                    
                    if distanceToCenter < shortestDistance then
                        shortestDistance = distanceToCenter
                        closestPlayer = player
                    end
                end
            end
        end
    end
    
    return closestPlayer
end

-- 3. Loop Principal (RenderStepped)
RunService.RenderStepped:Connect(function()
    -- Atualizar ESP
    if SETTINGS.ESPEnabled then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                local char = player.Character
                local rootPart = char:FindFirstChild("HumanoidRootPart")
                local humanoid = char:FindFirstChild("Humanoid")

                if rootPart and humanoid and humanoid.Health > 0 then
                    local distance = (rootPart.Position - Camera.CFrame.Position).Magnitude
                    if distance <= SETTINGS.DetectionRange then
                        addHighlight(char)
                    else
                        removeHighlight(char)
                    end
                else
                    removeHighlight(char)
                end
            end
        end
    end

    -- Lógica de FOV e Aimbot
    local target = getClosestPlayerToCenter()
    local detected = (target ~= nil)

    if SETTINGS.FOVEnabled and fovCircle then
        if detected then
            fovCircle:FindFirstChildOfClass("UIStroke").Color = Color3.fromRGB(0, 255, 0) -- Verde
        else
            fovCircle:FindFirstChildOfClass("UIStroke").Color = Color3.fromRGB(255, 0, 0) -- Vermelho
        end
    end

    if SETTINGS.AimbotEnabled and target and target.Character then
        local targetPart = target.Character:FindFirstChild("Head") or target.Character:FindFirstChild("HumanoidRootPart")
        if targetPart then
            local currentCFrame = Camera.CFrame
            local targetCFrame = CFrame.new(currentCFrame.Position, targetPart.Position)
            Camera.CFrame = currentCFrame:Lerp(targetCFrame, SETTINGS.AimbotSmoothness)
        end
    end
end)

-- Limpar highlights quando jogadores saírem
Players.PlayerRemoving:Connect(function(player)
    if player.Character then
        removeHighlight(player.Character)
    end
end)

-- Inicializar Interface
createInterface()
