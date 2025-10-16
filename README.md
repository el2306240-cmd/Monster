-- =================================================================================
-- == FILE: ServerScriptService/MonsterHubSystem/AdminModule (ModuleScript)       ==
-- Paste as ModuleScript named: AdminModule under ServerScriptService.MonsterHubSystem
-- =================================================================================
local Players = game:GetService("Players")
local DataStoreService = game:GetService("DataStoreService")
local Workspace = game:GetService("Workspace")

local Admin = {}

-- ---------- CONFIG ----------
-- Substitua pelos seus UserIds (somente números)
Admin.Owners = {
    [12345678] = true, -- <-- substitua
    [87654321] = true, -- <-- substitua
}

Admin.HubKey = "MonsterHub_SafeCode_001"   -- deve bater com o cliente
Admin._SECRET_PASSWORD = "ghosthub002"     -- senha server-side (NUNCA coloque no cliente)
Admin.Admins = {}
Admin.Authenticated = {} -- [userId] = true
Admin.Commands = {}
Admin.Banned = {}
Admin.Jailed = {}
Admin.Protection = { AntiToolSteal = true, AntiKick = true }

-- DataStore para bans (opcional)
local ok, banStore = pcall(function() return DataStoreService:GetDataStore("MonsterHub_Bans_v1") end)
if not ok then banStore = nil end

-- ---------- HELPERS ----------
local function toId(x)
    if not x then return nil end
    if type(x) == "number" then return x end
    if typeof(x) == "Instance" and x.UserId then return x.UserId end
    if type(x) == "string" then
        local n = tonumber(x)
        if n then return n end
    end
    return nil
end

function Admin:IsOwner(playerOrId)
    local id = toId(playerOrId)
    return id and Admin.Owners[id] == true
end

function Admin:IsAdmin(playerOrId)
    local id = toId(playerOrId)
    if not id then return false end
    if Admin:IsOwner(id) then return true end
    if Admin.Admins[id] then return true end
    if Admin.Authenticated[id] then return true end
    return false
end

function Admin:RegisterCommand(name, func, minRank)
    name = string.lower(name)
    Admin.Commands[name] = { func = func, minRank = minRank or "Admin" }
end

local rankCheck = {
    Owner = function(_, player) return Admin:IsOwner(player) end,
    Admin = function(_, player) return Admin:IsAdmin(player) end,
    User  = function() return true end,
}

function Admin:CanExecute(player, cmdName)
    local cmd = Admin.Commands[string.lower(cmdName or "")]
    if not cmd then return false, "Comando não encontrado" end
    local checker = rankCheck[cmd.minRank] or rankCheck["Admin"]
    return checker(nil, player)
end

function Admin:Execute(player, cmdName, args)
    local cmd = Admin.Commands[string.lower(cmdName or "")]
    if not cmd then return false, "Comando não encontrado" end
    local ok, res = pcall(function() return cmd.func(player, args or {}) end)
    if not ok then
        warn("Admin command error:", res)
        return false, "Erro ao executar comando"
    end
    return true, res or "Executado"
end

-- ---------- AUTH via senha ----------
function Admin:AttemptPasswordAuth(player, password)
    if not player or not player.UserId then return false, "Inválido" end
    if tostring(password) == tostring(Admin._SECRET_PASSWORD) then
        Admin.Authenticated[player.UserId] = true
        return true, "Senha correta — acesso admin liberado"
    else
        return false, "Senha incorreta"
    end
end

function Admin:RevokeAuth(userId)
    Admin.Authenticated[userId] = nil
end

-- ---------- DataStore bans ----------
local function loadBans()
    if not banStore then return end
    local ok, data = pcall(function() return banStore:GetAsync("bans") end)
    if ok and type(data) == "table" then Admin.Banned = data end
end
local function saveBans()
    if not banStore then return end
    pcall(function() banStore:SetAsync("bans", Admin.Banned) end)
end
loadBans()
spawn(function()
    while true do
        wait(60)
        saveBans()
    end
end)

-- ---------- Ensure jail cell exists ----------
if not Workspace:FindFirstChild("Jails") then
    local j = Instance.new("Folder")
    j.Name = "Jails"
    j.Parent = Workspace
end
if not Workspace.Jails:FindFirstChild("Cell") then
    local cell = Instance.new("Part")
    cell.Name = "Cell"
    cell.Size = Vector3.new(6,1,6)
    cell.Transparency = 1
    cell.Anchored = true
    cell.CanCollide = false
    cell.CFrame = CFrame.new(0,50,0)
    cell.Parent = Workspace.Jails
end

-- ---------- COMMANDS ----------
Admin:RegisterCommand("kick", function(player, args)
    local targetArg = args[1]
    if not targetArg then return "Uso: kick <UserId/Nome>" end
    local reason = table.concat({select(2, table.unpack(args))}, " ")
    local targetPlayer
    local id = tonumber(tostring(targetArg))
    if id then targetPlayer = Players:GetPlayerByUserId(id) end
    if not targetPlayer then
        for _,p in pairs(Players:GetPlayers()) do
            if string.lower(p.Name) == string.lower(tostring(targetArg)) then targetPlayer = p; break end
        end
    end
    if not targetPlayer then return "Player not found" end
    if Admin.Protection.AntiKick and Admin:IsOwner(targetPlayer) then return "Cannot kick owner (protected)" end
    pcall(function() targetPlayer:Kick((reason ~= "" and reason) or "Kicked by admin") end)
    return "Kicked "..tostring(targetPlayer.Name)
end, "Admin")

Admin:RegisterCommand("tempban", function(player, args)
    local id = tonumber(tostring(args[1])); if not id then return "Uso: tempban <userId> <minutes|'perm'>" end
    local durArg = args[2]; local expiry
    if durArg == "perm" then expiry = math.huge else
        local minutes = tonumber(durArg)
        if not minutes then return "Duração inválida" end
        expiry = os.time() + minutes*60
    end
    Admin.Banned[id] = expiry; saveBans()
    local pl = Players:GetPlayerByUserId(id)
    if pl then pcall(function() pl:Kick("Banned") end) end
    return "Banned "..tostring(id)
end, "Owner")

Admin:RegisterCommand("unban", function(player,args)
    local id = tonumber(tostring(args[1])); if not id then return "Uso: unban <userId>" end
    Admin.Banned[id] = nil; saveBans()
    return "Unbanned "..tostring(id)
end, "Owner")

Admin:RegisterCommand("jail", function(player,args)
    local id = tonumber(tostring(args[1])); if not id then return "Uso: jail <userId> [minutes]" end
    local mins = tonumber(args[2]) or 0
    local expiry = mins > 0 and (os.time() + mins*60) or nil
    local pl = Players:GetPlayerByUserId(id)
    if not pl then return "Jogador não online" end
    Admin.Jailed[id] = { expiry = expiry }
    pcall(function()
        local char = pl.Character
        if char and char.PrimaryPart then
            local cell = Workspace:FindFirstChild("Jails") and Workspace.Jails:FindFirstChild("Cell")
            if cell and cell:IsA("BasePart") then char:SetPrimaryPartCFrame(cell.CFrame + Vector3.new(0,3,0)) end
        end
        local hum = char and char:FindFirstChildWhichIsA("Humanoid")
        if hum then hum.WalkSpeed = 0; hum.JumpPower = 0 end
        if pl.Backpack then for _,it in pairs(pl.Backpack:GetChildren()) do if it:IsA("Tool") then it.Parent = nil end end end
        if char then for _,it in pairs(char:GetChildren()) do if it:IsA("Tool") then it:Destroy() end end end
    end)
    return "Jailed "..tostring(id)
end, "Admin")

Admin:RegisterCommand("free", function(player,args)
    local id = tonumber(tostring(args[1])); if not id then return "Uso: free <userId>" end
    if not Admin.Jailed[id] then return "Não estava preso" end
    Admin.Jailed[id] = nil
    local pl = Players:GetPlayerByUserId(id)
    if pl and pl.Character then
        local hum = pl.Character:FindFirstChildWhichIsA("Humanoid")
        if hum then hum.WalkSpeed = 16; hum.JumpPower = 50 end
    end
    return "Liberado "..tostring(id)
end, "Admin")

Admin:RegisterCommand("slap", function(player,args)
    local id = tonumber(tostring(args[1])); if not id then return "Uso: slap <userId> [dano]" end
    local dmg = tonumber(args[2]) or 20
    local pl = Players:GetPlayerByUserId(id)
    if not pl or not pl.Character then return "Jogador não online" end
    local hum = pl.Character:FindFirstChildWhichIsA("Humanoid")
    if hum then pcall(function() hum:TakeDamage(dmg) end) end
    return "Slapped "..tostring(id)
end, "Admin")

function Admin:PlayGlobalSound(audioId, volume, pitch)
    local id = tonumber(tostring(audioId)) or tonumber(string.match(tostring(audioId), "%d+"))
    if not id then return false, "AudioId inválido" end
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxassetid://"..tostring(id)
    sound.Volume = volume or 1
    sound.PlaybackSpeed = pitch or 1
    sound.Parent = Workspace
    sound:Play()
    sound.Ended:Connect(function() pcall(function() sound:Destroy() end) end)
    delay(60, function() if sound and sound.Parent then pcall(function() sound:Destroy() end) end)
    return true, "Tocando"
end

-- ---------- maintenance loops ----------
spawn(function()
    while true do
        local now = os.time()
        for id, info in pairs(Admin.Jailed) do
            if info.expiry and info.expiry <= now then
                Admin.Jailed[id] = nil
                local pl = Players:GetPlayerByUserId(id)
                if pl and pl.Character then
                    local hum = pl.Character:FindFirstChildWhichIsA("Humanoid")
                    if hum then hum.WalkSpeed = 16; hum.JumpPower = 50 end
                end
            end
        end
        wait(5)
    end
end)

Players.PlayerAdded:Connect(function(pl)
    local id = pl.UserId
    local expiry = Admin.Banned[id]
    if expiry then
        if expiry == math.huge or expiry > os.time() then
            wait(0.5)
            pcall(function() pl:Kick("Banned") end)
            return
        else
            Admin.Banned[id] = nil
            saveBans()
        end
    end
    local j = Admin.Jailed[id]
    if j then
        wait(1)
        pcall(function()
            if pl.Character and pl.Character.PrimaryPart then
                local cell = Workspace:FindFirstChild("Jails") and Workspace.Jails:FindFirstChild("Cell")
                if cell then pl.Character:SetPrimaryPartCFrame(cell.CFrame + Vector3.new(0,3,0)) end
                local hum = pl.Character:FindFirstChildWhichIsA("Humanoid")
                if hum then hum.WalkSpeed = 0; hum.JumpPower = 0 end
            end
        end)
    end
end)

return Admin

-- =================================================================================
-- == FILE: ServerScriptService/MonsterHubSystem/AdminServer (Script)            ==
-- Paste as Script named: AdminServer under ServerScriptService.MonsterHubSystem
-- =================================================================================
local Replicated = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local folder = Replicated:WaitForChild("MonsterHub")
local AdminRequest = folder:WaitForChild("AdminRequest")
local PlayGlobalSoundEvent = folder:WaitForChild("PlayGlobalSound")
local HubFeedback = folder:WaitForChild("HubFeedback")
local PasswordAttempt = folder:WaitForChild("PasswordAttempt")

local Admin = require(script.Parent:WaitForChild("AdminModule"))

local function validHubKey(key)
    return key and tostring(key) == tostring(Admin.HubKey)
end

-- AdminRequest signature: (player, commandName, argsTable, hubKey)
AdminRequest.OnServerEvent:Connect(function(player, commandName, args, hubKey)
    if not validHubKey(hubKey) then
        HubFeedback:FireClient(player, false, "HubKey inválido — comando recusado")
        return
    end
    if type(commandName) ~= "string" then
        HubFeedback:FireClient(player, false, "Comando inválido")
        return
    end
    args = args or {}
    local can, why = Admin:CanExecute(player, commandName)
    if not can then HubFeedback:FireClient(player, false, why or "Sem permissão"); return end
    local ok, res = Admin:Execute(player, commandName, args)
    HubFeedback:FireClient(player, ok, res)
end)

-- PlayGlobalSoundEvent signature: (player, audioId, volume, pitch, hubKeyOrNil)
PlayGlobalSoundEvent.OnServerEvent:Connect(function(player, audioId, volume, pitch, hubKey)
    local ok, res = pcall(function() return Admin:PlayGlobalSound(audioId, volume, pitch) end)
    HubFeedback:FireClient(player, ok and true or false, ok and "Tocando" or (res or "Erro"))
end)

-- PasswordAttempt signature: (player, password, hubKey)
PasswordAttempt.OnServerEvent:Connect(function(player, password, hubKey)
    if not validHubKey(hubKey) then
        HubFeedback:FireClient(player, false, "HubKey inválido")
        return
    end
    local ok, msg = Admin:AttemptPasswordAuth(player, password)
    HubFeedback:FireClient(player, ok, msg)
end)

Players.PlayerRemoving:Connect(function(pl)
    if pl and pl.UserId then Admin:RevokeAuth(pl.UserId) end
end)

-- =================================================================================
-- == FILE: StarterGui/MonsterHubClient (LocalScript)                          ==
-- Paste as LocalScript named: MonsterHubClient under StarterGui
-- =================================================================================
-- LocalScript creates GUI, dragon button, draggable behavior, sends hubKey with admin requests
local Players = game:GetService("Players")
local Replicated = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- ---------- CONFIG ----------
local HUB_KEY = "MonsterHub_SafeCode_001" -- must match AdminModule.HubKey
local SECRET_PASSWORD_DISPLAY = "secret (server-side)" -- don't use this; server checks password
local DRAGON_IMAGE_ID = "rbxassetid://PUT_YOUR_DRAGON_IMAGE_ID_HERE" -- substitute with your uploaded image
local GUI_COLOR = Color3.fromRGB(165,16,16)
local BUTTON_COLOR = Color3.fromRGB(130,8,8)
local BUTTON_HOVER = Color3.fromRGB(150,10,10)
local AUDIO_COOLDOWN = 5 -- seconds
local ROUND_PIXELS = 10

-- Ensure Remote folder exists (server script will also expect these names)
local folder = Replicated:FindFirstChild("MonsterHub")
if not folder then
    folder = Instance.new("Folder", Replicated)
    folder.Name = "MonsterHub"
end
local function getOrCreateRemote(name)
    local r = folder:FindFirstChild(name)
    if not r then
        r = Instance.new("RemoteEvent", folder)
        r.Name = name
    end
    return r
end
local AdminRequest = getOrCreateRemote("AdminRequest")
local PlayGlobalSoundEvent = getOrCreateRemote("PlayGlobalSound")
local HubFeedback = getOrCreateRemote("HubFeedback")
local PasswordAttempt = getOrCreateRemote("PasswordAttempt")

-- Remove old GUI
local old = playerGui:FindFirstChild("MonsterHubGUI")
if old then old:Destroy() end

-- ---------- GUI CREATION ----------
local screenGui = Instance.new("ScreenGui", playerGui)
screenGui.Name = "MonsterHubGUI"
screenGui.ResetOnSpawn = false
screenGui.DisplayOrder = 100

-- Main frame
local main = Instance.new("Frame", screenGui)
main.Name = "Main"
main.Size = UDim2.new(0,460,0,360)
main.Position = UDim2.new(0.6,0,0.25,0)
main.AnchorPoint = Vector2.new(0.5,0)
main.BackgroundColor3 = GUI_COLOR
main.BorderSizePixel = 0
local mainCorner = Instance.new("UICorner", main); mainCorner.CornerRadius = UDim.new(0, ROUND_PIXELS)

-- Top bar
local top = Instance.new("Frame", main)
top.Name = "Top"
top.Size = UDim2.new(1,0,0,44)
top.Position = UDim2.new(0,0,0,0)
top.BackgroundTransparency = 1

local title = Instance.new("TextLabel", top)
title.Size = UDim2.new(0.6,0,1,0)
title.Position = UDim2.new(0,12,0,0)
title.BackgroundTransparency = 1
title.Text = "Monster Hub"
title.TextSize = 20
title.TextColor3 = Color3.fromRGB(255,255,255)
title.Font = Enum.Font.GothamBold
title.TextXAlignment = Enum.TextXAlignment.Left

local closeBtn = Instance.new("TextButton", top)
closeBtn.Name = "Close"
closeBtn.Size = UDim2.new(0,36,0,28)
closeBtn.Position = UDim2.new(1,-48,0,8)
closeBtn.Text = "▢"
closeBtn.Font = Enum.Font.Gotham
closeBtn.TextSize = 20
closeBtn.TextColor3 = Color3.fromRGB(255,255,255)
closeBtn.BackgroundColor3 = BUTTON_COLOR
closeBtn.BorderSizePixel = 0
local closeCorner = Instance.new("UICorner", closeBtn); closeCorner.CornerRadius = UDim.new(0,8)

closeBtn.MouseButton1Click:Connect(function()
    main.Visible = false
    dragonButton.Visible = true
end)

-- Side buttons
local side = Instance.new("Frame", main)
side.Size = UDim2.new(0,140,1,-64)
side.Position = UDim2.new(0,12,0,52)
side.BackgroundTransparency = 1

local function makeSideButton(text, y)
    local b = Instance.new("TextButton", side)
    b.Size = UDim2.new(1,-12,0,44)
    b.Position = UDim2.new(0,6,0,(y-1)*56)
    b.Text = text
    b.Font = Enum.Font.GothamSemibold
    b.TextSize = 16
    b.TextColor3 = Color3.fromRGB(255,255,255)
    b.BackgroundColor3 = BUTTON_COLOR
    b.BorderSizePixel = 0
    local corner = Instance.new("UICorner", b); corner.CornerRadius = UDim.new(0,8)
    b.MouseEnter:Connect(function() b.BackgroundColor3 = BUTTON_HOVER end)
    b.MouseLeave:Connect(function() b.BackgroundColor3 = BUTTON_COLOR end)
    return b
end

local btnFuncs = makeSideButton("Funções", 1)
local btnAdmin = makeSideButton("Administrador", 2)
local btnAudios = makeSideButton("Áudios", 3)
local btnConfig = makeSideButton("Config", 4)

-- Content
local content = Instance.new("Frame", main)
content.Size = UDim2.new(1, -176, 1, -72)
content.Position = UDim2.new(0,164,0,52)
content.BackgroundTransparency = 1

local function clearContent()
    for _,c in pairs(content:GetChildren()) do if not c:IsA("UIListLayout") then pcall(function() c:Destroy() end) end end
end

local function notify(msg)
    local label = Instance.new("TextLabel", main)
    label.Size = UDim2.new(1,-24,0,28)
    label.Position = UDim2.new(0,12,1,-40)
    label.BackgroundTransparency = 0.35
    label.BackgroundColor3 = Color3.fromRGB(60,10,10)
    label.TextColor3 = Color3.fromRGB(255,255,255)
    label.Font = Enum.Font.Gotham
    label.TextSize = 14
    label.Text = msg
    label.ZIndex = 5
    delay(4, function() pcall(function() label:Destroy() end) end)
end

local function makeInput(labelText, placeholder, y)
    local lbl = Instance.new("TextLabel", content)
    lbl.Size = UDim2.new(0.35,0,0,28)
    lbl.Position = UDim2.new(0,8,0,(y-1)*40 +8)
    lbl.BackgroundTransparency = 1
    lbl.Text = labelText
    lbl.TextColor3 = Color3.fromRGB(255,255,255)
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 14
    lbl.TextXAlignment = Enum.TextXAlignment.Left

    local input = Instance.new("TextBox", content)
    input.Size = UDim2.new(0.62,0,0,28)
    input.Position = UDim2.new(0.35,8,0,(y-1)*40 +8)
    input.PlaceholderText = placeholder or ""
    input.ClearTextOnFocus = false
    input.BackgroundColor3 = Color3.fromRGB(200,20,20)
    input.TextColor3 = Color3.fromRGB(255,255,255)
    input.Font = Enum.Font.Gotham
    input.TextSize = 14
    local corner = Instance.new("UICorner", input); corner.CornerRadius = UDim.new(0,6)
    return input
end

-- ---------- Build sections ----------
local function buildFuncs()
    clearContent()
    local title = Instance.new("TextLabel", content)
    title.Size = UDim2.new(1,-12,0,28)
    title.Position = UDim2.new(0,8,0,4)
    title.BackgroundTransparency = 1
    title.Text = "Funções Básicas"
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    title.TextColor3 = Color3.fromRGB(255,255,255)

    local speedInput = makeInput("Velocidade", "ex: 24", 1)
    local setSpeed = Instance.new("TextButton", content)
    setSpeed.Size = UDim2.new(0.48,0,0,36)
    setSpeed.Position = UDim2.new(0,8,0, 88)
    setSpeed.Text = "Set Velocidade"
    setSpeed.Font = Enum.Font.GothamSemibold
    setSpeed.TextColor3 = Color3.fromRGB(255,255,255)
    setSpeed.BackgroundColor3 = BUTTON_COLOR
    local corner = Instance.new("UICorner", setSpeed); corner.CornerRadius = UDim.new(0,8)
    setSpeed.MouseButton1Click:Connect(fun
