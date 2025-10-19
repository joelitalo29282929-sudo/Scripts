-- Server: HandlePurchases.lua
-- Coloque em ServerScriptService
-- Substitua PRODUCT_ID pelo id do Developer Product que você criou.

local MarketplaceService = game:GetService("MarketplaceService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local PRODUCT_ID = 12345678 -- <<-- trocar pelo seu ProductId (número)
local BOOST_DURATION = 60   -- duração do boost em segundos
local BOOST_SPEED = 50      -- velocidade nova enquanto o boost estiver ativo
local NORMAL_SPEED = 16     -- velocidade padrão (padrão do Roblox)

-- RemoteEvent para avisar cliente sobre mudança de UI/status (cria se não existir)
local remoteName = "BananaRubyPurchaseEvent"
local remote = ReplicatedStorage:FindFirstChild(remoteName)
if not remote then
    remote = Instance.new("RemoteEvent")
    remote.Name = remoteName
    remote.Parent = ReplicatedStorage
end

-- Mantém rastreamento de jogadores com boost ativo para evitar múltiplos boosts
local activeBoosts = {} -- [player] = os cronos/flags

local function applySpeedBoost(player)
    if not player.Character or not player.Character.Parent then return end
    local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end

    -- guarda velocidade atual para restaurar depois (caso seja diferente)
    local oldSpeed = humanoid.WalkSpeed
    humanoid.WalkSpeed = BOOST_SPEED

    -- avisa cliente pra atualizar UI
    pcall(function()
        remote:FireClient(player, {
            event = "BoostStarted",
            boostedSpeed = BOOST_SPEED,
            duration = BOOST_DURATION
        })
    end)

    -- agendar restauração
    spawn(function()
        wait(BOOST_DURATION)
        -- apenas restaura se o humanoid ainda existir
        if player and player.Character and player.Character.Parent then
            local h = player.Character:FindFirstChildOfClass("Humanoid")
            if h then
                -- só restaura se a velocidade ainda corresponde ao boost (evita sobrescritas)
                if math.abs(h.WalkSpeed - BOOST_SPEED) < 0.1 then
                    h.WalkSpeed = (oldSpeed and oldSpeed > 0) and oldSpeed or NORMAL_SPEED
                end
            end
        end

        -- atualiza flag
        activeBoosts[player] = nil
        pcall(function()
            remote:FireClient(player, { event = "BoostEnded" })
        end)
    end)
end

-- Função chamada pelo ProcessReceipt quando o Roblox confirma pagamento
local function processReceipt(receiptInfo)
    local playerUserId = receiptInfo.PlayerId
    local productId = receiptInfo.ProductId

    -- somente nosso produto
    if productId ~= PRODUCT_ID then
        return Enum.ProductPurchaseDecision.NotProcessedYet
    end

    -- procura o player atual (pode retornar nil se o player saiu)
    local player = Players:GetPlayerByUserId(playerUserId)
    if player then
        -- previne multi-aplicação simultânea
        if activeBoosts[player] then
            -- já ativo — você pode decidir estender a duração ou negar.
            -- Aqui vamos somar um aviso e estender: simplesmente re-executar applySpeedBoost novamente.
            -- Para simplificar, só enviamos um aviso e retornamos que processamos.
            warn("[BananaRuby] Player already had an active boost; purchase processed and queued.")
            -- opcional: estender contadores, etc.
        else
            activeBoosts[player] = true
            applySpeedBoost(player)
        end
    end

    -- retorna que o recebimento foi processado com sucesso
    return Enum.ProductPurchaseDecision.PurchaseGranted
end

MarketplaceService.ProcessReceipt = processReceipt

-- Ao jogador entrar, restaura variáveis / limpa flags
Players.PlayerRemoving:Connect(function(player)
    activeBoosts[player] = nil
end)-- Cliente: BananaRubyClient.lua
-- Coloque em StarterPlayerScripts ou StartGui (LocalScript)
-- Substitua PRODUCT_ID pelo mesmo número do Dev Product.

local MarketplaceService = game:GetService("MarketplaceService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

local PRODUCT_ID = 12345678 -- <<-- mesmo ProductId do ServerScript
local remoteName = "BananaRubyPurchaseEvent"

-- cria GUI minimal (se quiser usar a GUI que já tem, adapte)
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "BananaRubyCathubDemo"
screenGui.ResetOnSpawn = false
screenGui.Parent = PlayerGui

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 360, 0, 220)
frame.Position = UDim2.new(0.5, -180, 0.5, -110)
frame.AnchorPoint = Vector2.new(0.5, 0.5)
frame.BackgroundColor3 = Color3.fromRGB(9, 19, 30)
frame.BorderSizePixel = 0
frame.Parent = screenGui
local uic = Instance.new("UICorner", frame); uic.CornerRadius = UDim.new(0, 12)

local title = Instance.new("TextLabel", frame)
title.Size = UDim2.new(1, -24, 0, 48)
title.Position = UDim2.new(0, 12, 0, 12)
title.BackgroundTransparency = 1
title.Text = "Banana Ruby / Cathub"
title.Font = Enum.Font.GothamBold
title.TextSize = 20
title.TextColor3 = Color3.fromRGB(230, 238, 248)
title.TextXAlignment = Enum.TextXAlignment.Left

local sub = Instance.new("TextLabel", frame)
sub.Size = UDim2.new(1, -24, 0, 18)
sub.Position = UDim2.new(0, 12, 0, 42)
sub.BackgroundTransparency = 1
sub.Text = "Seller (local): JOEL"
sub.Font = Enum.Font.Gotham
sub.TextSize = 13
sub.TextColor3 = Color3.fromRGB(191, 227, 255)
sub.TextXAlignment = Enum.TextXAlignment.Left

local price = Instance.new("TextLabel", frame)
price.Size = UDim2.new(0, 74, 0, 28)
price.Position = UDim2.new(1, -86, 0, 14)
price.BackgroundColor3 = Color3.fromRGB(12, 26, 40)
price.BorderSizePixel = 0
price.Text = "£4.99"
price.Font = Enum.Font.GothamBold
price.TextSize = 14
price.TextColor3 = Color3.fromRGB(230, 238, 248)
price.TextXAlignment = Enum.TextXAlignment.Center
local priceCorner = Instance.new("UICorner", price); priceCorner.CornerRadius = UDim.new(0,8)

local statusText = Instance.new("TextLabel", frame)
statusText.Size = UDim2.new(0, 120, 0, 18)
statusText.Position = UDim2.new(0, 12, 0, 64)
statusText.BackgroundTransparency = 1
statusText.Text = "Online"
statusText.Font = Enum.Font.Gotham
statusText.TextSize = 14
statusText.TextColor3 = Color3.fromRGB(255,255,255)
statusText.TextXAlignment = Enum.TextXAlignment.Left

local speedLabel = Instance.new("TextLabel", frame)
speedLabel.Size = UDim2.new(0, 200, 0, 44)
speedLabel.Position = UDim2.new(0, 12, 0, 98)
speedLabel.BackgroundTransparency = 1
speedLabel.Text = "Current Speed\n16"
speedLabel.Font = Enum.Font.GothamBold
speedLabel.TextSize = 18
speedLabel.TextColor3 = Color3.fromRGB(230, 238, 248)
speedLabel.TextXAlignment = Enum.TextXAlignment.Left
speedLabel.TextYAlignment = Enum.TextYAlignment.Top

local toggleBtn = Instance.new("TextButton", frame)
toggleBtn.Size = UDim2.new(1, -24, 0, 44)
toggleBtn.Position = UDim2.new(0, 12, 0, 150)
toggleBtn.BackgroundColor3 = Color3.fromRGB(18, 199, 111)
toggleBtn.BorderSizePixel = 0
toggleBtn.Text = "Enable Speed"
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextSize = 16
toggleBtn.TextColor3 = Color3.fromRGB(255,255,255)
local btnCorner = Instance.new("UICorner", toggleBtn); btnCorner.CornerRadius = UDim.new(0,10)

local remote = ReplicatedStorage:WaitForChild(remoteName)

-- Quando o servidor confirmar ou avisar, atualiza a UI
remote.OnClientEvent:Connect(function(data)
    if type(data) ~= "table" then return end
    if data.event == "BoostStarted" then
        toggleBtn.Text = "Disable Speed"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(99,102,106)
        speedLabel.Text = "Current Speed\n" .. tostring(data.boostedSpeed or "?")
    elseif data.event == "BoostEnded" then
        toggleBtn.Text = "Enable Speed"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(18,199,111)
        speedLabel.Text = "Current Speed\n16"
    end
end)

-- Ao clicar, abre a janela de compra do Dev Product
toggleBtn.MouseButton1Click:Connect(function()
    -- Se quiser checar estado local (opcional) você pode checar texto do botão
    if toggleBtn.Text == "Enable Speed" then
        -- Solicita compra do Developer Product
        MarketplaceService:PromptProductPurchase(Player, PRODUCT_ID)
    else
        -- Se já está ativado, aqui você pode implementar uma opção para "cancelar"
        -- mas normalmente a compra é consumida e expira após duration no servidor.
        -- Apenas mostra uma mensagem local:
        toggleBtn.Text = "Processing..."
        wait(0.5)
        toggleBtn.Text = "Disable Speed"
    end
end)

-- Opcional: atualização inicial do status (mostra Online)
statusText.Text = "Online"
