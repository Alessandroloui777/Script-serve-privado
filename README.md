-- 🛡️ Script QA Unificado (coloque em ServerScriptService)

-- ======= CONFIGURAÇÃO =======
local ALLOWED_USERS = {
    [12345678] = true, -- coloque aqui os UserIds autorizados
    [87654321] = true, -- exemplo
}
local BASE_CFRAME = CFrame.new(0, 5, 0) -- posição da base

-- ======= SERVIDOR =======
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

-- Criar pasta de Remotes
local remotesFolder = ReplicatedStorage:FindFirstChild("QA_Remotes") or Instance.new("Folder", ReplicatedStorage)
remotesFolder.Name = "QA_Remotes"
remotesFolder.Parent = ReplicatedStorage

local function ensureRemote(name, className)
    local r = remotesFolder:FindFirstChild(name)
    if not r then
        r = Instance.new(className)
        r.Name = name
        r.Parent = remotesFolder
    end
    return r
end

local RE_Teleport = ensureRemote("QA_Teleport", "RemoteEvent")
local RE_Buy = ensureRemote("QA_Buy", "RemoteEvent")
local RE_Sell = ensureRemote("QA_Sell", "RemoteEvent")
local RF_State = ensureRemote("QA_RequestState", "RemoteFunction")

-- Catálogo de Brainrots
local Catalog = {
    {id=1, name="Common Smirk", rarity="Common", dps=5, price=100},
    {id=2, name="Side Eye", rarity="Common", dps=7, price=150},
    {id=3, name="Grimace", rarity="Uncommon", dps=15, price=350},
    {id=4, name="Rizz Master", rarity="Rare", dps=35, price=900},
    {id=5, name="Sigma Stare", rarity="Epic", dps=75, price=2200},
    {id=6, name="Giga Chad", rarity="Legendary", dps=160, price=5200},
}
local RarityOrder = {Common=1, Uncommon=2, Rare=3, Epic=4, Legendary=5}

local function sortByRarityThenDps()
    local t = table.clone(Catalog)
    table.sort(t, function(a,b)
        if RarityOrder[a.rarity] == RarityOrder[b.rarity] then
            return a.dps > b.dps
        end
        return RarityOrder[a.rarity] > RarityOrder[b.rarity]
    end)
    return t
end

local function getByRarity(rar)
    for _, it in ipairs(Catalog) do
        if it.rarity == rar then return it end
    end
end

-- Estado dos jogadores
local PlayerState = {}
local BASE_CAPACITY = 8

local function ensurePlayerState(plr)
    if not PlayerState[plr.UserId] then
        PlayerState[plr.UserId] = {
            cash = 5000,
            inventory = {},
        }
    end
    return PlayerState[plr.UserId]
end

local function totalDps(inv)
    local s = 0
    for _, it in ipairs(inv) do s += it.dps end
    return s
end

local function findLowestDpsIndex(inv)
    local idx, minDps = nil, math.huge
    for i, it in ipairs(inv) do
        if it.dps < minDps then
            minDps = it.dps
            idx = i
        end
    end
    return idx
end

-- Eventos
RE_Teleport.OnServerEvent:Connect(function(plr)
    if not ALLOWED_USERS[plr.UserId] then return end
    local st = ensurePlayerState(plr)
    local char = plr.Character
    if not char or not char.PrimaryPart then return end
    char:SetPrimaryPartCFrame(BASE_CFRAME)
end)

RE_Buy.OnServerEvent:Connect(function(plr, byRarity, wantedRarity)
    if not ALLOWED_USERS[plr.UserId] then return end
    local st = ensurePlayerState(plr)
    if #st.inventory >= BASE_CAPACITY then
        local idx = findLowestDpsIndex(st.inventory)
        if idx then
            local sold = table.remove(st.inventory, idx)
            st.cash += math.floor(sold.price * 0.6)
        end
    end
    local candidate
    if byRarity and wantedRarity then
        candidate = getByRarity(wantedRarity) or sortByRarityThenDps()[1]
    else
        candidate = sortByRarityThenDps()[1]
    end
    if candidate and st.cash >= candidate.price then
        st.cash -= candidate.price
        table.insert(st.inventory, candidate)
    end
end)

RE_Sell.OnServerEvent:Connect(function(plr, sellLowest)
    if not ALLOWED_USERS[plr.UserId] then return end
    local st = ensurePlayerState(plr)
    if #st.inventory == 0 then return end
    local idx = sellLowest and findLowestDpsIndex(st.inventory) or #st.inventory
    local sold = table.remove(st.inventory, idx)
    st.cash += math.floor(sold.price * 0.6)
end)

RF_State.OnServerInvoke = function(plr)
    if not ALLOWED_USERS[plr.UserId] then return end
    local st = ensurePlayerState(plr)
    return {
        cash = st.cash,
        inventory = st.inventory,
        capacity = BASE_CAPACITY,
        totalDps = totalDps(st.inventory),
    }
end

-- ======= CLIENTE (UI) =======
Players.PlayerAdded:Connect(function(plr)
    if not ALLOWED_USERS[plr.UserId] then return end
    plr.CharacterAdded:Wait()

    local guiScript = [[
        local player = game.Players.LocalPlayer
        local remotes = game.ReplicatedStorage:WaitForChild("QA_Remotes")
        local RE_Teleport = remotes:WaitForChild("QA_Teleport")
        local RE_Buy = remotes:WaitForChild("QA_Buy")
        local RE_Sell = remotes:WaitForChild("QA_Sell")
        local RF_State = remotes:WaitForChild("QA_RequestState")

        local screen = Instance.new("ScreenGui")
        screen.Name = "QA_Tools"
        screen.Parent = player.PlayerGui

        local frame = Instance.new("Frame")
        frame.Size = UDim2.fromOffset(300, 360)
        frame.Position = UDim2.new(0, 20, 0.5, -180)
        frame.Parent = screen

        local title = Instance.new("TextLabel")
        title.Size = UDim2.new(1, 0, 0, 32)
        title.Text = "Painel QA"
        title.Parent = frame

        local lblInfo = Instance.new("TextLabel")
        lblInfo.Size = UDim2.new(1, -20, 0, 60)
        lblInfo.Position = UDim2.new(0, 10, 0, 40)
        lblInfo.TextWrapped = true
        lblInfo.Parent = frame

        local function mkBtn(text, y)
            local b = Instance.new("TextButton")
            b.Size = UDim2.new(1, -20, 0, 30)
            b.Position = UDim2.new(0, 10, 0, y)
            b.Text = text
            b.Parent = frame
            return b
        end

        local btnTp = mkBtn("Teleportar à Base", 110)
        local btnBuyBest = mkBtn("Comprar Melhor", 150)
        local rarityBox = Instance.new("TextBox")
        rarityBox.Size = UDim2.new(1, -20, 0, 30)
        rarityBox.Position = UDim2.new(0, 10, 0, 190)
        rarityBox.PlaceholderText = "Raridade"
        rarityBox.Parent = frame
        local btnBuyRar = mkBtn("Comprar por Raridade", 230)
        local btnSellLow = mkBtn("Vender Menor DPS", 270)
        local autoToggle = mkBtn("Auto-Compra: OFF", 310)

        local auto = false
        local function refresh()
            local state = RF_State:InvokeServer()
            if state then
                lblInfo.Text = string.format("Cash: %d | DPS: %d\nInv: %d/%d",
                    state.cash, state.totalDps, #state.inventory, state.capacity)
            end
        end

        btnTp.MouseButton1Click:Connect(function()
            RE_Teleport:FireServer()
            refresh()
        end)
        btnBuyBest.MouseButton1Click:Connect(function()
            RE_Buy:FireServer(false, nil)
            refresh()
        end)
        btnBuyRar.MouseButton1Click:Connect(function()
            RE_Buy:FireServer(true, rarityBox.Text)
            refresh()
        end)
        btnSellLow.MouseButton1Click:Connect(function()
            RE_Sell:FireServer(true)
            refresh()
        end)
        autoToggle.MouseButton1Click:Connect(function()
            auto = not auto
            autoToggle.Text = "Auto-Compra: " .. (auto and "ON" or "OFF")
        end)

        task.spawn(function()
            while screen.Parent do
                if auto then
                    RE_Buy:FireServer(false, nil)
                    refresh()
                end
                task.wait(2)
            end
        end)
        refresh()
    ]]

    local localScript = Instance.new("LocalScript")
    localScript.Source = guiScript
    localScript.Parent = plr:WaitForChild("PlayerGui")
end)
