-- =================================================================
-- SERVICIOS Y COMPATIBILIDAD SOLARA
-- =================================================================
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local SoundService = game:GetService("SoundService")
local ContentProvider = game:GetService("ContentProvider")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local StarterGui = game:GetService("StarterGui")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local ProximityPromptService = game:GetService("ProximityPromptService")
local CollectionService = game:GetService("CollectionService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local playerGui = player:WaitForChild("PlayerGui")
local targetGui = (gethui and pcall(gethui) and gethui()) or CoreGui

if not game:IsLoaded() then
    game.Loaded:Wait()
end

pcall(function()
    StarterGui:SetCore("SendNotification", {
        Title = "om.hub",
        Text = "Modulo Survivor, Anti-Void, Auto-Revive & Hitbox Cargados.",
        Duration = 5
    })
end)

-- Limpieza de interfaces previas
for _, name in pairs({"OutcomeMemoriesUI", "gay", "VoteMenuByPedroDraggableSonicFace", "UnifiedHubUI", "MobileControlsUI"}) do
    pcall(function()
        local old = targetGui:FindFirstChild(name) or playerGui:FindFirstChild(name)
        if old then old:Destroy() end
    end)
end

-- =================================================================
-- CONFIGURACIONES Y ESTADOS
-- =================================================================

local SETTINGS = {
    TriggerBot = {
        Enabled = false,
        Cooldown = 0.4,
        Distance = 8, 
        Width = 7,
    },
    Hitbox = {
        Enabled = false,
        Size = 10,
        TargetTeam = "All"
    },
    Visual = {
        ESP = false,
        Fullbright = false,
        MinesESP = false,
    }
}

-- Guardar estado original de iluminación
local originalLighting = {
    Ambient = Lighting.Ambient,
    OutdoorAmbient = Lighting.OutdoorAmbient,
    Brightness = Lighting.Brightness,
    ClockTime = Lighting.ClockTime,
    FogEnd = Lighting.FogEnd,
    GlobalShadows = Lighting.GlobalShadows
}

local originalHitboxSizes = {}
local probe = nil
local probeWeld = nil
local lastAttackTime = 0

-- Variables para Teleport
local savedCustomPosition = nil

-- Settings Auto-Pick & LMS
local VoteRemote = ReplicatedStorage:WaitForChild("Remotes", 5) and ReplicatedStorage.Remotes:WaitForChild("Voted", 5)

local lmsStates = {
    ["KnucklesSolo"] = true,
    ["SonicSolo"]    = true,
    ["BlazeSolo"]    = true,
    ["AmySolo"]      = true,
    ["EggmanSolo"]   = true,
    ["Fleetway"]     = true
}

-- ESTILOS DE TEXTO PARA CADA PERSONAJE EN LMS
local lmsStyles = {
    ["SonicSolo"] = {
        TextColor = Color3.fromRGB(150, 255, 255),
        StrokeColor = Color3.fromRGB(0, 0, 200),
        Font = Enum.Font.LuckiestGuy
    },
    ["KnucklesSolo"] = {
        TextColor = Color3.fromRGB(139, 0, 0),
        StrokeColor = Color3.fromRGB(40, 0, 0),
        Font = Enum.Font.GothamBlack
    },
    ["BlazeSolo"] = {
        TextColor = Color3.fromRGB(160, 32, 240),
        StrokeColor = Color3.fromRGB(255, 140, 0),
        Font = Enum.Font.Bodoni
    },
    ["AmySolo"] = {
        TextColor = Color3.fromRGB(255, 182, 193),
        StrokeColor = Color3.fromRGB(255, 20, 147),
        Font = Enum.Font.FredokaOne
    },
    ["EggmanSolo"] = {
        TextColor = Color3.fromRGB(255, 140, 0),
        StrokeColor = Color3.fromRGB(0, 0, 0),
        Font = Enum.Font.SciFi
    },
    ["Fleetway"] = {
        TextColor = Color3.fromRGB(255, 255, 0),
        StrokeColor = Color3.fromRGB(255, 0, 0),
        Font = Enum.Font.Creepster
    }
}

local autoPickEnabled = false
local selectedCharacter = nil
local currentIndex = 0

-- Estados de la pestaña Local
local speedUpEnabled = false
local currentSpeed = 35

local noJumpCooldownEnabled = false
local infJumpEnabled = false

-- Estados Fly Inf
local flyInfFleetwayEnabled = false
local flySilverInfEnabled = false
local flyInfLoop = nil

-- =================================================================
-- LÓGICA CURSOR LOCK / AIM LOCK (TECLA 'T')
-- =================================================================
local cursorLockEnabled = false

local LOCK_TARGETS = {
    ["2011x"] = true,
    ["fleetway"] = true,
    ["tailsdoll"] = true,
    ["kolossos"] = true,
    ["exeller"] = true,
    ["exe"] = true
}

local function getNearestLockTarget()
    local myChar = player.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(player.Name))
    if not myChar then return nil end

    local myRoot = myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end

    local nearest = nil
    local shortest = math.huge

    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= player then
            local char = plr.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(plr.Name))
            if char then
                local isKiller = char:GetAttribute("Team") == "EXE" or plr:GetAttribute("Team") == "EXE"
                local attr = char:GetAttribute("Character")
                
                if isKiller or (attr and LOCK_TARGETS[string.lower(attr)]) then
                    local root = char:FindFirstChild("HumanoidRootPart")
                    if root then
                        local dist = (root.Position - myRoot.Position).Magnitude
                        if dist < shortest then
                            shortest = dist
                            nearest = root
                        end
                    end
                end
            end
        end
    end

    return nearest
end

local function setCursorLockState(enabled)
    cursorLockEnabled = enabled
end

RunService.RenderStepped:Connect(function()
    if not cursorLockEnabled then return end

    local char = player.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(player.Name))
    if not char then return end

    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local targetRoot = getNearestLockTarget()
    if not targetRoot then return end

    local targetPos = targetRoot.Position + Vector3.new(0, 1.5, 0)

    camera.CFrame = CFrame.lookAt(camera.CFrame.Position, targetPos)
    root.CFrame = CFrame.new(root.Position, Vector3.new(targetPos.X, root.Position.Y, targetPos.Z))
end)

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.T then
        setCursorLockState(not cursorLockEnabled)
    end
end)

-- =================================================================
-- LÓGICA SONIC ANIMATION / HEADTURN
-- =================================================================
local sonicAnimEnabled = false
local sonicConnections = {}
local sonicLoopThread = nil

local function updateSonicChar(character)
    if not sonicAnimEnabled or not character then return end
    if character:GetAttribute("Character") == "Sonic" then
        local prepare = CollectionService:HasTag(character, "Prepare")
        local peelout = CollectionService:HasTag(character, "Peelout")

        character:SetAttribute("HeadTurn", not (prepare or peelout))
    end
end

local function setupSonicChar(character)
    if not sonicAnimEnabled or not character then return end
    updateSonicChar(character)

    table.insert(sonicConnections, character:GetAttributeChangedSignal("Character"):Connect(function()
        updateSonicChar(character)
    end))

    table.insert(sonicConnections, CollectionService:GetInstanceAddedSignal("Prepare"):Connect(function(instance)
        if instance == character then updateSonicChar(character) end
    end))

    table.insert(sonicConnections, CollectionService:GetInstanceRemovedSignal("Prepare"):Connect(function(instance)
        if instance == character then updateSonicChar(character) end
    end))

    table.insert(sonicConnections, CollectionService:GetInstanceAddedSignal("Peelout"):Connect(function(instance)
        if instance == character then updateSonicChar(character) end
    end))

    table.insert(sonicConnections, CollectionService:GetInstanceRemovedSignal("Peelout"):Connect(function(instance)
        if instance == character then updateSonicChar(character) end
    end))
end

local function setSonicAnimState(enabled)
    sonicAnimEnabled = enabled

    for _, conn in ipairs(sonicConnections) do
        pcall(function() conn:Disconnect() end)
    end
    sonicConnections = {}

    if sonicLoopThread then
        task.cancel(sonicLoopThread)
        sonicLoopThread = nil
    end

    if sonicAnimEnabled then
        if player.Character then
            setupSonicChar(player.Character)
        end

        table.insert(sonicConnections, player.CharacterAdded:Connect(setupSonicChar))

        sonicLoopThread = task.spawn(function()
            while sonicAnimEnabled do
                for _, instance in ipairs(CollectionService:GetTagged("Dropdash")) do
                    if instance:GetAttribute("Character") == "Sonic" then
                        CollectionService:AddTag(instance, "Charging")
                    end
                end

                for _, instance in ipairs(CollectionService:GetTagged("Peelout")) do
                    if instance:GetAttribute("Character") == "Sonic" then
                        CollectionService:AddTag(instance, "Charging")
                    end
                end

                for _, instance in ipairs(CollectionService:GetTagged("Charging")) do
                    if not CollectionService:HasTag(instance, "Dropdash")
                        and not CollectionService:HasTag(instance, "Peelout") then
                        CollectionService:RemoveTag(instance, "Charging")
                    end
                end
                task.wait()
            end
        end)
    end
end

-- =================================================================
-- LÓGICA EGGMAN BOOST (CORREGIDO SIN T-POSE & ACTIVACIÓN TECLA 'G')
-- =================================================================
local eggmanBoostEnabled = false
local TARGET_SPEEDBOOST = 2.65
local TARGET_STATE = "jetpack"

local function getEggmanPlayerModel()
    local playersFolder = workspace:FindFirstChild("Players")
    if playersFolder then
        return playersFolder:FindFirstChild(player.Name)
    end
    return nil
end

local function applyEggmanBoost(state)
    local model = getEggmanPlayerModel()
    if not model then return end
    
    if model:GetAttribute("Character") == "Eggman" then
        if state then
            model:SetAttribute("SpeedBoost", TARGET_SPEEDBOOST)
            model:SetAttribute("State", TARGET_STATE)
        else
            model:SetAttribute("SpeedBoost", 1)
            model:SetAttribute("State", "None")
        end
    end
end

local function toggleEggmanBoost()
    eggmanBoostEnabled = not eggmanBoostEnabled
    applyEggmanBoost(eggmanBoostEnabled)
end

local function setEggmanBoostState(enabled)
    eggmanBoostEnabled = enabled
    applyEggmanBoost(eggmanBoostEnabled)
end

player.CharacterAdded:Connect(function()
    task.wait(0.5)
    if eggmanBoostEnabled then
        applyEggmanBoost(true)
    end
end)

-- =================================================================
-- SISTEMA DE VUELO ADICIONAL (TAILS FLY GUI & KEYBIND G)
-- =================================================================
local FLY_SPEED_GUI = 50
local flyGuiEnabled = false
local isFlyGuiFlying = false
local FLY_ANIM_ID = "rbxassetid://134795550846208"

local flyGuiCharData = {}
local flyGuiBtn = nil
local flyScreenGui = nil

local function stopFlyGui()
    isFlyGuiFlying = false
    if flyGuiCharData.animTrack then flyGuiCharData.animTrack:Stop() end
    if flyGuiCharData.humanoid then flyGuiCharData.humanoid.PlatformStand = false end
    if flyGuiCharData.bv then flyGuiCharData.bv:Destroy() end
    if flyGuiCharData.bg then flyGuiCharData.bg:Destroy() end
    flyGuiCharData.bv = nil
    flyGuiCharData.bg = nil
end

local function startFlyGui()
    if not flyGuiCharData.hrp or not flyGuiCharData.humanoid then return end
    isFlyGuiFlying = true
    flyGuiCharData.humanoid.PlatformStand = true

    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(1e9,1e9,1e9)
    bv.Parent = flyGuiCharData.hrp

    local bg = Instance.new("BodyGyro")
    bg.MaxTorque = Vector3.new(1e9,1e9,1e9)
    bg.P = 1e5
    bg.Parent = flyGuiCharData.hrp

    flyGuiCharData.bv = bv
    flyGuiCharData.bg = bg

    if flyGuiCharData.animTrack then
        flyGuiCharData.animTrack:Play()
    end
end

local function toggleFlyGui()
    if not flyGuiEnabled then return end
    if isFlyGuiFlying then stopFlyGui() else startFlyGui() end
end

local function setupFlyGuiCharacter(char)
    stopFlyGui()

    local humanoid = char:WaitForChild("Humanoid")
    local hrp = char:WaitForChild("HumanoidRootPart")

    local anim = Instance.new("Animation")
    anim.AnimationId = FLY_ANIM_ID
    local animTrack = humanoid:LoadAnimation(anim)

    flyGuiCharData = {
        char = char,
        humanoid = humanoid,
        hrp = hrp,
        animTrack = animTrack,
        bv = nil,
        bg = nil
    }
end

local function setFlyGuiState(enabled)
    flyGuiEnabled = enabled
    if flyGuiEnabled then
        if not flyScreenGui then
            flyScreenGui = Instance.new("ScreenGui")
            flyScreenGui.Name = "FlyButtonGUI"
            flyScreenGui.Parent = player.PlayerGui
            flyScreenGui.ResetOnSpawn = false

            flyGuiBtn = Instance.new("TextButton")
            flyGuiBtn.Parent = flyScreenGui
            flyGuiBtn.Size = UDim2.fromScale(0.06, 0.06)
            flyGuiBtn.Position = UDim2.fromScale(0.88, 0.78)
            flyGuiBtn.Text = "▲"
            flyGuiBtn.TextScaled = true
            flyGuiBtn.BackgroundColor3 = Color3.fromRGB(20,20,20)
            flyGuiBtn.TextColor3 = Color3.new(1,1,1)
            flyGuiBtn.BorderSizePixel = 0
            flyGuiBtn.Font = Enum.Font.GothamBold
            flyGuiBtn.Active = true
            flyGuiBtn.Draggable = true

            flyGuiBtn.MouseButton1Click:Connect(toggleFlyGui)
        end
        flyScreenGui.Enabled = true
        setupFlyGuiCharacter(player.Character or player.CharacterAdded:Wait())
    else
        stopFlyGui()
        if flyScreenGui then
            flyScreenGui.Enabled = false
        end
    end
end

player.CharacterAdded:Connect(function(char)
    if flyGuiEnabled then
        task.wait(0.1)
        setupFlyGuiCharacter(char)
    end
end)

-- Mapeo de Tecla 'G' para Tails Fly y Eggman Boost
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.G then
        if flyGuiEnabled then
            toggleFlyGui()
        end
        
        local model = getEggmanPlayerModel()
        if model and model:GetAttribute("Character") == "Eggman" then
            toggleEggmanBoost()
        end
    end
end)

RunService.RenderStepped:Connect(function()
    if not flyGuiEnabled or not isFlyGuiFlying or not flyGuiCharData.hrp or not flyGuiCharData.bg or not flyGuiCharData.bv then return end
    
    local camCFrame = workspace.CurrentCamera.CFrame
    local lookVector = camCFrame.LookVector

    flyGuiCharData.bv.Velocity = lookVector * FLY_SPEED_GUI
    flyGuiCharData.bg.CFrame = camCFrame.Rotation + flyGuiCharData.hrp.Position
end)

-- LÓGICA FLY INF (FLEETWAY & SILVER)
local function applyFlyInf(character)
    pcall(function()
        local frame = character:WaitForChild("HumanoidRootPart")
            :WaitForChild("BillboardGui")
            :WaitForChild("Frame")
        local function lockText(label)
            label.Text = "inf"
            label:GetPropertyChangedSignal("Text"):Connect(function()
                if label.Text ~= "inf" then
                    label.Text = "inf"
                end
            end)
        end
        lockText(frame:WaitForChild("Amount"))
        lockText(frame:WaitForChild("Maximum"))
    end)
end

local function toggleFlyInf(state)
    if state then
        if player.Character then applyFlyInf(player.Character) end
        if not flyInfLoop then
            flyInfLoop = task.spawn(function()
                while flyInfFleetwayEnabled or flySilverInfEnabled do
                    pcall(function()
                        if player.Character then
                            player.Character:AddTag("Kill")
                            task.wait()
                            player.Character:RemoveTag("Kill")
                        end
                    end)
                    task.wait()
                end
            end)
        end
    else
        if not flyInfFleetwayEnabled and not flySilverInfEnabled and flyInfLoop then
            task.cancel(flyInfLoop)
            flyInfLoop = nil
        end
    end
end

-- Auto-Revive
local autoReviveEnabled = false
local REVIVE_DELAY = 0.02
local reviveConnection = nil

local function setAutoReviveState(enabled)
    autoReviveEnabled = enabled
    if autoReviveEnabled then
        if not reviveConnection then
            reviveConnection = ProximityPromptService.PromptShown:Connect(function(prompt)
                if not autoReviveEnabled then return end
                if not prompt or not prompt.Enabled then return end
                if prompt.ActionText ~= "Revive" then return end

                task.delay(REVIVE_DELAY, function()
                    if prompt and prompt.Parent and prompt.Enabled and autoReviveEnabled then
                        prompt.HoldDuration = 0
                        pcall(function()
                            fireproximityprompt(prompt)
                        end)
                    end
                end)
            end)
        end
    else
        if reviveConnection then
            reviveConnection:Disconnect()
            reviveConnection = nil
        end
    end
end

-- Anti-Void
local antiVoidEnabled = false
local fakeVoidPart = nil
local fakeVoidConnection = nil
local HITBOX_SIZE = Vector3.new(2048, 6, 2048) 
local FAKE_VOID_POSITION = Vector3.new(16620, 948, 308) 
local SAFE_SPAWN_CFRAME = CFrame.new(16437, 1013, 473) 

local function setAntiVoidState(enabled)
    antiVoidEnabled = enabled
    if antiVoidEnabled then
        if not fakeVoidPart or not fakeVoidPart.Parent then
            fakeVoidPart = Instance.new("Part")
            fakeVoidPart.Name = "RawAntiVoid"
            fakeVoidPart.Size = HITBOX_SIZE
            fakeVoidPart.Position = FAKE_VOID_POSITION
            fakeVoidPart.Anchored = true
            fakeVoidPart.CanCollide = false 
            fakeVoidPart.Transparency = 1 
            fakeVoidPart.BrickColor = BrickColor.new("Bright green")
            fakeVoidPart.Parent = workspace

            fakeVoidConnection = fakeVoidPart.Touched:Connect(function(hit)
                if not antiVoidEnabled then return end
                local character = player.Character
                if character and hit:IsDescendantOf(character) then
                    character:SetPrimaryPartCFrame(SAFE_SPAWN_CFRAME)
                end
            end)
        end
    else
        if fakeVoidConnection then
            fakeVoidConnection:Disconnect()
            fakeVoidConnection = nil
        end
        if fakeVoidPart then
            fakeVoidPart:Destroy()
            fakeVoidPart = nil
        end
    end
end

-- =================================================================
-- SISTEMA DE TRIPWIRE MINES ESP
-- =================================================================

if _G.ChaseTrapESP_Connections then 
    for _, v in pairs(_G.ChaseTrapESP_Connections) do pcall(function() v:Disconnect() end) end 
end
_G.ChaseTrapESP_Connections = {}

if _G.ChaseTrapESP_Objects then 
    for _, v in pairs(_G.ChaseTrapESP_Objects) do pcall(function() v:Destroy() end) end 
end
_G.ChaseTrapESP_Objects = {}

local TrapsESP = {}
local KnownTrapSize = Vector3.new(3, 3, 3)

local function clearAllMinesESP()
    for trap, espData in pairs(TrapsESP) do
        if espData then
            if espData.box then pcall(function() espData.box:Destroy() end) end
            if espData.glowBox then pcall(function() espData.glowBox:Destroy() end) end
            if espData.bill then pcall(function() espData.bill:Destroy() end) end
        end
    end
    TrapsESP = {}
    for _, v in pairs(_G.ChaseTrapESP_Objects) do
        pcall(function() v:Destroy() end)
    end
    _G.ChaseTrapESP_Objects = {}
end

local function MakeMineESP(trap)
    if not trap or trap.Name ~= "Union" or (trap.Parent and trap.Parent.Name ~= "Traps") then return end
    if TrapsESP[trap] then return end
    
    local espData = {}
    
    local box = Instance.new("BoxHandleAdornment")
    box.Size = KnownTrapSize
    box.Color3 = Color3.fromRGB(0, 220, 255)
    box.Transparency = 0.4
    box.ZIndex = 10
    box.Adornee = trap
    box.Parent = trap
    box.Visible = SETTINGS.Visual.MinesESP
    table.insert(_G.ChaseTrapESP_Objects, box)
    espData.box = box
    
    local glowBox = Instance.new("BoxHandleAdornment")
    glowBox.Size = KnownTrapSize + Vector3.new(0.5, 0.5, 0.5)
    glowBox.Color3 = Color3.fromRGB(0, 255, 200)
    glowBox.Transparency = 0.7
    glowBox.AlwaysOnTop = true
    glowBox.ZIndex = 5
    glowBox.Adornee = trap
    glowBox.Parent = trap
    glowBox.Visible = SETTINGS.Visual.MinesESP
    table.insert(_G.ChaseTrapESP_Objects, glowBox)
    espData.glowBox = glowBox
    
    local bill = Instance.new("BillboardGui")
    bill.Size = UDim2.new(0, 100, 0, 30)
    bill.StudsOffset = Vector3.new(0, 2.5, 0)
    bill.AlwaysOnTop = true
    bill.MaxDistance = 400
    bill.Parent = trap
    bill.Enabled = SETTINGS.Visual.MinesESP
    table.insert(_G.ChaseTrapESP_Objects, bill)
    espData.bill = bill
    
    local dist = Instance.new("TextLabel")
    dist.TextSize = 12
    dist.Size = UDim2.new(1, 0, 1, 0)
    dist.Position = UDim2.new(0, 0, 0, 0)
    dist.BackgroundTransparency = 1
    dist.Text = "TRIPWIRE MINE"
    dist.TextColor3 = Color3.fromRGB(0, 255, 255)
    dist.TextStrokeTransparency = 0
    dist.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    dist.Font = Enum.Font.GothamBold
    dist.TextXAlignment = Enum.TextXAlignment.Center
    dist.TextYAlignment = Enum.TextYAlignment.Center
    dist.Parent = bill
    espData.dist = dist
    
    TrapsESP[trap] = espData
end

local function UpdateMinesESP()
    if not SETTINGS.Visual.MinesESP then return end
    
    local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    local rootPos = root and root.Position or Vector3.new(0, 0, 0)
    
    for trap, espData in pairs(TrapsESP) do
        if not trap or not trap.Parent then
            if espData.box then pcall(function() espData.box:Destroy() end) end
            if espData.glowBox then pcall(function() espData.glowBox:Destroy() end) end
            if espData.bill then pcall(function() espData.bill:Destroy() end) end
            TrapsESP[trap] = nil
            continue
        end
        
        espData.box.Visible = true
        espData.glowBox.Visible = true
        espData.bill.Enabled = true

        if root then
            local studsDist = (rootPos - trap.Position).Magnitude
            
            local color
            if studsDist < 25 then
                color = Color3.fromRGB(255, 40, 80)
            elseif studsDist < 60 then
                color = Color3.fromRGB(255, 170, 0)
            else
                color = Color3.fromRGB(0, 230, 255)
            end
            
            espData.box.Color3 = color
            espData.glowBox.Color3 = color
            espData.dist.TextColor3 = color
            espData.dist.Text = "💣 MINA 💣\n[" .. math.floor(studsDist) .. " studs]"
        end
    end
end

local function ScanExistingMines()
    local proj = workspace:FindFirstChild("Projectile")
    if proj then
        for _, trap in pairs(proj:GetDescendants()) do
            if trap.Name == "Union" and trap.Parent and trap.Parent.Name == "Traps" then
                MakeMineESP(trap)
            end
        end
    end
end

task.spawn(function()
    local proj = workspace:FindFirstChild("Projectile") or workspace:WaitForChild("Projectile", 10)
    if proj then
        ScanExistingMines()
        table.insert(_G.ChaseTrapESP_Connections, proj.DescendantAdded:Connect(function(child)
            if child.Name == "Union" and child.Parent and child.Parent.Name == "Traps" then
                task.wait(0.05)
                MakeMineESP(child)
            end
        end))
    end
end)

table.insert(_G.ChaseTrapESP_Connections, RunService.Heartbeat:Connect(UpdateMinesESP))

-- =================================================================
-- CONFIGURACIÓN Y MÓDULO KNUCKLES
-- =================================================================

local knucklesScriptDisabled = false
local knucklesFlightEnabled = true
local knucklesPunchTechEnabled = true

local HOLD_TIME = 0.3
local FLY_SPEED = 50
local FLY_ANIM  = "rbxassetid://90686495729962"

local KNUCKLES_ANIMATIONS = {
    Idle    = { id = "rbxassetid://97453636614232",  track = nil, shouldLoop = true  },
    Walk    = { id = "rbxassetid://114028642269057", track = nil, shouldLoop = true  },
    Run     = { id = "rbxassetid://92702778916591",  track = nil, shouldLoop = true  },
    Jump    = { id = "rbxassetid://139272524520189", track = nil, shouldLoop = false },
    Fall    = { id = "rbxassetid://97406687015420",  track = nil, shouldLoop = true  },
    FlyLoop = { id = FLY_ANIM,                       track = nil, shouldLoop = true  },
}

local ANIMS_TO_DISABLE = {
    "rbxassetid://126990789595827",
    "rbxassetid://83673430424599",
}

local isFlying          = false
local spaceHeld         = false
local spaceTimer        = 0
local bodyVelocity      = nil
local bodyGyro          = nil
local currentKnucklesAnim = nil
local shiftHeld         = false
local punchTimer        = 0

local function stopAllKnucklesAnimations()
    for _, data in pairs(KNUCKLES_ANIMATIONS) do
        if data.track then
            if data.track.IsPlaying then
                data.track:Stop()
            end
            data.track = nil
        end
    end
    currentKnucklesAnim = nil
end

local function isKnuckles()
    if knucklesScriptDisabled then return false end
    local playersFolder = workspace:FindFirstChild("Players")
    if not playersFolder then return false end
    local model = playersFolder:FindFirstChild(player.Name)
    return model and model:GetAttribute("Character") == "Knuckles"
end

local function disableDefaultKnucklesAnimations(humanoid)
    if knucklesScriptDisabled then return end
    if not (humanoid and humanoid:FindFirstChild("Animator")) then return end
    for _, track in pairs(humanoid:GetPlayingAnimationTracks()) do
        for _, id in ipairs(ANIMS_TO_DISABLE) do
            if track.Animation and track.Animation.AnimationId == id then
                track:Stop()
                break
            end
        end
    end
end

local function loadKnucklesAnimation(humanoid, animName)
    if knucklesScriptDisabled then return nil end
    local data = KNUCKLES_ANIMATIONS[animName]
    if not data or not humanoid then return nil end
    local anim = Instance.new("Animation")
    anim.AnimationId = data.id
    local track = humanoid:LoadAnimation(anim)
    track.Looped = data.shouldLoop
    data.track   = track
    return track
end

local function playKnucklesAnimation(humanoid, animName)
    if knucklesScriptDisabled then return end
    for _, data in pairs(KNUCKLES_ANIMATIONS) do
        if data.track and data.track.IsPlaying then
            data.track:Stop()
        end
    end
    local data = KNUCKLES_ANIMATIONS[animName]
    if not data then return end
    if not data.track then loadKnucklesAnimation(humanoid, animName) end
    if data.track then data.track:Play() end
end

local function updateKnucklesAnimations(humanoid)
    if knucklesScriptDisabled then return end
    local newAnim = nil
    if isFlying then
        newAnim = "FlyLoop"
    else
        local state = humanoid:GetState()
        if state == Enum.HumanoidStateType.Jumping then
            newAnim = "Jump"
        elseif state == Enum.HumanoidStateType.Freefall then
            newAnim = "Fall"
        elseif humanoid.MoveDirection.Magnitude > 0.1 then
            if shiftHeld or humanoid.WalkSpeed > 16 then
                newAnim = "Run"
            else
                newAnim = "Walk"
            end
        else
            newAnim = "Idle"
        end
    end

    if newAnim and newAnim ~= currentKnucklesAnim then
        playKnucklesAnimation(humanoid, newAnim)
        currentKnucklesAnim = newAnim
    end
end

local function startFly(rootPart, humanoid)
    if isFlying or not knucklesFlightEnabled or knucklesScriptDisabled then return end
    isFlying = true
    humanoid.PlatformStand = true

    bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.MaxForce = Vector3.new(1e5, 1e5, 1e5)
    bodyVelocity.Velocity = Vector3.zero
    bodyVelocity.Parent   = rootPart

    bodyGyro = Instance.new("BodyGyro")
    bodyGyro.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
    bodyGyro.D         = 100
    bodyGyro.CFrame    = rootPart.CFrame
    bodyGyro.Parent    = rootPart

    updateKnucklesAnimations(humanoid)
end

local function stopFly(humanoid)
    if not isFlying then return end
    isFlying = false
    if humanoid then humanoid.PlatformStand = false end

    if bodyVelocity then bodyVelocity:Destroy(); bodyVelocity = nil end
    if bodyGyro     then bodyGyro:Destroy();     bodyGyro     = nil end

    currentKnucklesAnim = nil
    if humanoid and not knucklesScriptDisabled then 
        updateKnucklesAnimations(humanoid) 
    end
end

local function applyPunchTech()
    if not knucklesPunchTechEnabled or knucklesScriptDisabled then return end
    local playersFolder = workspace:FindFirstChild("Players")
    local model = playersFolder and playersFolder:FindFirstChild(player.Name)
    if model then
        local state = model:GetAttribute("State")
        if state == "focus" then
            model:SetAttribute("State", "punch")
            model:SetAttribute("CanSprint", true)
        end
    end
end

-- =================================================================
-- BASE DE DATOS DE PERSONAJES Y IMÁGENES
-- =================================================================

local characters = {"Sonic", "Amy", "Eggman", "Cream", "Tails", "Knuckles", "MetalSonic", "Blaze", "Silver"}

local characterImages = {
    ["Sonic"]      = "rbxthumb://type=Asset&id=90698485266685&w=150&h=150",
    ["Amy"]        = "rbxthumb://type=Asset&id=139906522593901&w=150&h=150",
    ["MetalSonic"] = "rbxthumb://type=Asset&id=125424559322589&w=150&h=150",
    ["Eggman"]     = "rbxthumb://type=Asset&id=129053897954881&w=150&h=150",
    ["Knuckles"]   = "rbxthumb://type=Asset&id=99077689372417&w=150&h=150",
    ["Tails"]      = "rbxthumb://type=Asset&id=77385974180985&w=150&h=150",
    ["Cream"]      = "rbxthumb://type=Asset&id=74140125964289&w=150&h=150",
    ["Blaze"]      = "rbxthumb://type=Asset&id=139583952022775&w=150&h=150",
    ["Silver"]     = "rbxthumb://type=Asset&id=90790764418649&w=150&h=150"
}

-- Base de datos de letras
local songLyrics = {
    ["SonicSolo"] = {
        { time = 27.76, text = "Rollin' around at the speed of sound" },
        { time = 30.06, text = "You wonder what's runnin' by you!" },
        { time = 32.61, text = "In a flash!" },
        { time = 35.31, text = "Ooh! Ooh!" },
        { time = 37.66, text = "Breakin' the rules, guess I'll switch my pace!" },
        { time = 40.26, text = "Some things in life you just can't replace" },
        { time = 42.61, text = "Oh nooo!" },
        { time = 44.91, text = "You can't replace, no!" },
        { time = 48.31, text = "You wanna dig my skull under your heel" },
        { time = 52.46, text = "But in time you'll find" },
        { time = 54.61, text = "There aren't any tricks left to pull on me!" },
        { time = 60.31, text = "Fight to be free!" },
        { time = 63.46, text = "And I'm always thinkin'" },
        { time = 65.41, text = "Two steps ahead!" },
        { time = 67.71, text = "Don't care who stands in my path" },
        { time = 70.36, text = "I'll never change who I am!" },
        { time = 73.11, text = "And who they all believe in, push me to carry on!" },
        { time = 77.76, text = "Crashing till the break of dawn" },
        { time = 80.46, text = "Fighting with the terms" },
        { time = 82.41, text = "That are gone" },
        { time = 85.01, text = "Gone, gone, goooone!" },
        { time = 86.20, text = " " },
        { time = 108.56, text = "I don't let uncertainty slow me down!" },
        { time = 111.36, text = "Or make me second guess!" },
        { time = 114.06, text = "I'll pave my own way forward" },
        { time = 116.56, text = "And clean up this mess!" },
        { time = 119.31, text = "Gotta keep on rollin' (rollin' on!)" },
        { time = 122.06, text = "Gotta keep singing my song! (It's true!)" },
        { time = 125.21, text = "If you think that you're faster" },
        { time = 129.91, text = "Prove me wrong!" },
        { time = 133.31, text = "Lost it all" },
        { time = 134.41, text = "Still I stand" },
        { time = 135.56, text = "No circumstance I'll die by your hands!" },
        { time = 140.66, text = "With the strength burned in my mind" },
        { time = 143.41, text = "Bring your worst, I won't hold back!" },
        { time = 145.76, text = "You can't twist the way I feel" },
        { time = 148.16, text = "I keep movin' on!" },
        { time = 172.31, text = "There's no more holdin' back" },
        { time = 174.56, text = "All the souls you've toyed with" },
        { time = 177.11, text = "Siding with violence" },
        { time = 179.06, text = "Isn't this what you wanted from me?!" },
        { time = 182.26, text = "There's no mercy left no more" },
        { time = 184.66, text = "Nothing can bring them back" },
        { time = 186.86, text = "Be as fast as you are" },
        { time = 188.81, text = "You can't outrun me!" },
        { time = 190.06, text = "No! No!" },
        { time = 192.16, text = "Cut all the twisted games!" },
        { time = 194.61, text = "I'll break free of the chains!" },
        { time = 197.06, text = "That held me back from showing you what I'm made of!" },
        { time = 202.06, text = "I'll live to see the day!" },
        { time = 204.41, text = "I'll escape from this place!" },
        { time = 207.06, text = "That's who I am" },
        { time = 209.56, text = "The master of defy-in...'" },
        { time = 212.41, text = "Defyin' fate!" },
        { time = 221.31, text = "(¡I'VE GOT YOUR LIMIT!)" },
        { time = 222.33, text ="(¡RIGHT HEREEEEE....!)"},
        { time = 224.06, text = "I'll live my own way!" },
        { time = 226.16, text = "And by my own rules!" },
        { time = 228.56, text = "Copy my face, can't get my ideals now, can you?" },
        { time = 232.41, text = "No!" },
        { time = 233.61, text = "Don't stand down, endure the pain!" },
        { time = 236.21, text = "Bring the thunder like a hurricane!" },
        { time = 238.71, text = "Live and learn!" },
        { time = 241.31, text = "Hangin' on the edge of tomorrow!" },
        { time = 244.16, text = "Never go your way!" },
        { time = 245.61, text = "So don't blink...!" },
    },
    ["KnucklesSolo"] = {
        { time = 10, text = "Come on, bring it!!!!1." },
        { time = 37, text = "Let's go toe-to-toe with you all mighty," },
        { time = 39, text = "there's something you must know." },
        { time = 40, text = "With the real Lucifer, I never joke" },
        { time = 42, text = "because it's my ammo. I'm throwing blows" },
        { time = 44, text = "while all you do is hesitate." },
        { time = 46, text = "While you know in my plateau, I'M DOING" },
        { time = 48, text = "WHAT IS NECESSARY. AND IF I ENTER" },
        { time = 50, text = "THE GAME, I ain't staying. Cuz I" },
        { time = 52, text = "don't play a joke or chuckle, keep my" },
        { time = 54, text = "fists blazing with rage. We" },
        { time = 56, text = "stand, we withstand the fury. So, knock," },
        { time = 58, text = "knock, cuz knuckles there's no need to" },
        { time = 61, text = "hurry. I'm right here." },
        { time = 62, text = "Guess I'supposed to be down. Leave you" },
        { time = 64, text = "in the streets how? Yeah, you let them" },
        { time = 66, text = "bleed now. See how? Whip in the" },
        { time = 68, text = "floor with you. Yeah, I'll put you" },
        { time = 70, text = "straight down. So, tell me who's" },
        { time = 71, text = "weak now. Let's see who can cease now." },
        { time = 74, text = "What you think I struggle and buckle" },
        { time = 76, text = "under your arm again? I'm only getting" },
        { time = 78, text = "started, guardian angel out and" },
        { time = 80, text = "defending my heaven crate to 11, I never" },
        { time = 82, text = "step down. Make sure this world keeps" },
        { time = 84, text = "spinning and goes round and back." },
        { time = 87, text = "We stand." },
        { time = 89, text = "Here repeating." },
        { time = 90, text = "Defeating every member with every" },
        { time = 92, text = "side of" },
        { time = 95, text = "Always singing and searching for a way." },
        { time = 98, text = "Isn't it tiring to have a fire" },
        { time = 100, text = "extinguisher be" },
        { time = 103, text = "in the millions of times right" },
        { time = 105, text = "where" },
        { time = 106, text = "we stand." },
        { time = 107, text = "Always hoping and praying that we'll" },
        { time = 109, text = "find a find a little peace." },
        { time = 125, text = "And like I said, you're going to face" },
        { time = 127, text = "the" },
        { time = 128, text = "Left, right, left, right, left, RIGHT," },
        { time = 129, text = "WHAT?" },
        { time = 130, text = "YOU THOUGHT I PUT UP THE gloves and" },
        { time = 131, text = "fetch your thirst out? I fight for my" },
        { time = 133, text = "life until I'm dead, I give you the" },
        { time = 135, text = "worst and do it again. I never" },
        { time = 136, text = "stop going. That's just who I am. Anyone" },
        { time = 138, text = "who got their ducks in a row I can." },
        { time = 140, text = "In the end I'll corner and I'll die." },
        { time = 143, text = "Fighting and trying and still" },
        { time = 147, text = "hit me with the horrors and they'll" },
        { time = 148, text = "rise." },
        { time = 150, text = "Violent and violently" },
        { time = 153, text = "we're living tight in this moment every" },
        { time = 154, text = "night." },
        { time = 156, text = "Tired and silently" },
        { time = 159, text = "pleading for a moment of our lives." },
        { time = 162, text = "Back in the good times now." },
        { time = 165, text = "We stand." },
        { time = 166, text = "Here repeating. Defeating every member" },
        { time = 168, text = "with every" },
        { time = 170, text = "side of" },
        { time = 172, text = "Always singing and searching for a way." },
        { time = 176, text = "Isn't it tiring to have a fire" },
        { time = 178, text = "extinguisher" },
        { time = 180, text = "in the millions of times right where" },
        { time = 184, text = "we stand." },
        { time = 185, text = "Always hoping and praying that we'll" },
        { time = 187, text = "find a find a" },
        { time = 217, text = "Every night we stand." },
        { time = 220, text = "Here repeating. Defeating every" },
        { time = 222, text = "member with every" },
        { time = 224, text = "side of" },
        { time = 227, text = "Always singing and searching for a" },
        { time = 230, text = "Isn't it tiring to have a fire" },
        { time = 232, text = "extinguisher" },
        { time = 234, text = "in the millions of times right where" },
        { time = 238, text = "we stand." },
        { time = 239, text = "Always hoping and praying that" },
        { time = 241, text = "we'll find" },
        { time = 246, text = " " },
    },
    ["BlazeSolo"] = {
        { time = 0.74, text = "Carrying on my light" },
        { time = 2.39, text = "Was meant to last" },
        { time = 4.28, text = "To show that when all the stars have died" },
        { time = 8.5, text = "A lone flame" },
        { time = 10.0, text = "Can burn bright" },
        { time = 12.0, text = " " },
        { time = 34.4, text = "And the Sol still burns bright" },
        { time = 36.5, text = "You can't take what's mine" },
        { time = 37.9, text = "My fire is a right" },
        { time = 39.4, text = "So divine" },
        { time = 40.4, text = "And I know I own my own spark" },
        { time = 43.1, text = "That flames" },
        { time = 44.5, text = "The Sol that still burns" },
        { time = 46.9, text = " " },
        { time = 47.4, text = "The Sol that still burns" },
        { time = 56.0, text = "Oh" },
        { time = 57.3, text = "Back to back" },
        { time = 58.0, text = "There's no one to hold" },
        { time = 58.98, text = "Back to back" },
        { time = 59.78, text = "My hands are so cold" },
        { time = 60.68, text = "Back to back" },
        { time = 61.48, text = "There's no one to hold" },
        { time = 62.38, text = "Back to back" },
        { time = 63.08, text = "My hands so cold" },
        { time = 64.08, text = "Back to back" },
        { time = 64.78, text = "There's no one to hold" },
        { time = 65.78, text = "Back to back" },
        { time = 67.18, text = "Back to back" },
        { time = 67.88, text = "Back to back" },
        { time = 68.58, text = "My hands are so cold" },
        { time = 69.58, text = "Back to back" },
        { time = 70.28, text = "Back to back" },
        { time = 70.98, text = "My hands are so cold" },
        { time = 71.98, text = "Back to back" },
        { time = 72.68, text = "Back to back" },
        { time = 73.38, text = "My hands are so cold" },
        { time = 74.38, text = "Back to back" },
        { time = 75.08, text = "My hands are" },
        { time = 76.28, text = "Back to back" },
        { time = 76.98, text = "Back to back" },
        { time = 77.68, text = "Back to back" },
        { time = 78.38, text = "Back to back" },
        { time = 79.58, text = "Back to back" },
        { time = 80.28, text = " " },
        { time = 114.7, text = "Back to back" },
        { time = 115.3, text = "Back to back" },
        { time = 115.9, text = "Back to back" },
        { time = 116.5, text = "Back to back" },
        { time = 117.1, text = "Back to back" },
        { time = 117.7, text = "Back to back" },
        { time = 118.3, text = "Back to back" },
        { time = 118.9, text = "Back to back" },
        { time = 119.5, text = "Back to back" },
        { time = 120.1, text = "Back to back" },
        { time = 120.7, text = "Back to back" },
        { time = 121.3, text = "Back to back" },
        { time = 121.9, text = "Back to back" },
        { time = 122.5, text = "Back to back" },
        { time = 123.1, text = "Back to back" },
        { time = 123.7, text = "Back to back" },
        { time = 124.3, text = "Back to back" },
        { time = 124.9, text = "Back to back" },
        { time = 124.11, text = " " },
    },
    ["AmySolo"] = {
        { time = 62.13, text = "What did you say?" },
        { time = 64.13, text = "'Cause every time I hear you" },
        { time = 66.63, text = "I drift away" },
        { time = 69.13, text = "And hallucinate" },
        { time = 71.63, text = "What did you say?" },
        { time = 73.63, text = "'Cause every time I hear you" },
        { time = 76.13, text = "I drift away" },
        { time = 78.63, text = "And hallucinate" },
        { time = 81.33, text = "Listen" },
        { time = 82.83, text = "I've got a secret" },
        { time = 85.53, text = "Can I promise" },
        { time = 88.03, text = "That you could keep it?" },
        { time = 91.03, text = "But if" },
        { time = 92.73, text = "You tried to speak it" },
        { time = 95.43, text = "I'll make you swallow" },
        { time = 98.03, text = "A thousand needles" },
        { time = 100.03, text = " " },
    },
    ["EggmanSolo"] = {
        { time = 45, text = "EGGMAN" },
        { time = 47, text = "it's what i am" },
        { time = 48, text = "i am the" },
        { time = 49, text = "EGGMAN" },
        { time = 50, text = "i got the master plan" },
        { time = 50.5, text = "i am the" },
        { time = 51, text = "EGGMAN" },
        { time = 52, text = "it's what i am" },
        { time = 53, text = "i am the" },
        { time = 54, text = "EGGMAN" },
        { time = 55, text = "i got the master plan" },
        { time = 56, text = "EGGMAN is't what i am" },
        { time = 58, text = "i am the" },
        { time = 59, text = "EGGMAN" },
        { time = 60, text = "I GOT THE MASTER PLAN" },
        { time = 61, text = "I AM THE" },
        { time = 62, text = "EGGMAN" },
        { time = 63, text = "IS'T WHAT I AM" },
        { time = 64, text = "I AM THE" },
        { time = 65, text = "EGGMAN" },
        { time = 157, text = "I AM THE" },
        { time = 158, text = "EGGMAN" },
        { time = 159, text = "IS'T WHAT I AM" },
        { time = 160, text = "I AM THE EGGMAN" },
        { time = 162, text = "I GOT THE MASTER PLAN" },
        { time = 163, text = "I AM THE EGGMAN" },
        { time = 165, text = "IS'T WHAT I AM" },
        { time = 166, text = "I AM THE EGGMAN" },
        { time = 167, text = "I GOT THE-" },
        { time = 173, text = "I GOT THE MASTER PLAN" },
        { time = 184, text = "HUAH HUAH" },
        { time = 185, text = "WHY DON'T YOU GIVE UP" },
        { time = 187, text = "YOU NEVER DEFEAT ME" },
        { time = 190, text = "HUAH HUAH HUAH" },
    },
}

local fleetwayLyrics = {
    { time = 7.50,  text = "(You might not get it, but your tricks cannot stop me!)" },
    { time = 11.66, text = "Trying to stand when you're bound to fall?" },
    { time = 13.68, text = "Know your fate is my call!" },
    { time = 14.99, text = "Hanging there on a thin line?" },
    { time = 16.89, text = "Tick-tock, you're out of time!" },
    { time = 17.94, text = "I wanna see you try" },
    { time = 19.64, text = "I need to hear you cry" },
    { time = 20.73, text = "Clinging on the edge of the line?" },
    { time = 22.56, text = "Let me say it one more time" },
    { time = 23.63, text = "(Go! Go! Super Sonic!)" },
    { time = 25.04, text = "Pure power, raw and erratic!" },
    { time = 26.54, text = "(Go! Go! Super Sonic!)" },
    { time = 28.10, text = "I am feeling manic!" },
    { time = 29.58, text = "(Go! Go! Super Sonic!)" },
    { time = 31.06, text = "I want to see you trying" },
    { time = 32.55, text = "(Go! Go! Super Sonic!)" },
    { time = 34.02, text = "I want to see you panic!" },
    { time = 35.40, text = "AIM HIGHER" },
    { time = 36.43, text = "FULL POWER" },
    { time = 37.78, text = "Speeding faster than light" },
    { time = 42.36, text = "Witness the might!" },
    { time = 43.76, text = "Burning brighter than stars" },
    { time = 47.88, text = "I'm the end of your fight" },
    { time = 49.46, text = "Feeding off your fright" },
    { time = 54.00, text = "In the chaos I bask!" },
    { time = 55.51, text = "I'll show you my way" },
    { time = 59.61, text = "And make you pay!" },
    { time = 62.10, text = " " },
    { time = 66.88, text = "(More, more! Hurt me more, it'll only make me stronger!)" },
    { time = 73.38, text = " " },
    { time = 74.28, text = "Never stopping, still going Super Sonic" },
    { time = 75.90, text = "Striking, biting and loving the fighting" },
    { time = 77.50, text = "Manic and erratic, terror's exciting" },
    { time = 79.24, text = "It is my power you are lacking" },
    { time = 80.57, text = "World is shaking, earth is quaking" },
    { time = 82.20, text = "Look at the risk you're finally taking" },
    { time = 83.70, text = "Slipping? Never, I'm clever and better" },
    { time = 85.43, text = "I'll snap your neck and break any tether" },
    { time = 86.49, text = "(Go! Go! Super Sonic!)" },
    { time = 87.99, text = "Pure power, raw and erratic!" },
    { time = 89.49, text = "(Go! Go! Super Sonic!)" },
    { time = 91.06, text = "I am beyond manic!" },
    { time = 92.55, text = "(Go! Go! Super Sonic!)" },
    { time = 94.02, text = "Keep on bleeding and trying!" },
    { time = 95.50, text = "I will make you panic!" },
    { time = 98.38, text = "AIM HIGHER" },
    { time = 99.40, text = "FULL POWER" },
    { time = 100.75, text = "Speeding faster than light" },
    { time = 104.80, text = "I am the might!" },
    { time = 106.19, text = "Burning brighter than stars" },
    { time = 110.31, text = "I'll turn you to ash!" },
    { time = 111.87, text = "Feeding off your fright" },
    { time = 116.40, text = "In the chaos I bask!" },
    { time = 117.90, text = "I'll show you my way" },
    { time = 122.00, text = "Now you must pay!" },
    { time = 126.99, text = " " },
    { time = 127.82, text = "(Don't act surprised now!)" },
    { time = 130.67, text = "(did you really think I'd go down easily?)" },
    { time = 133.47, text = "(Pathetic fools!)" },
    { time = 136.32, text = "(Now)" },
    { time = 137.67, text = "(PREPARE FOR MY WRATH!)" },
    { time = 139.67, text = "Speeding faster than light" },
    { time = 144.00, text = "Devour the might!" },
    { time = 145.40, text = "Burning brighter than stars" },
    { time = 149.74, text = "I'll erase you in sight!" },
    { time = 151.14, text = "Speeding faster than light" },
    { time = 155.30, text = "Devour the might!" },
    { time = 156.69, text = "Burning brighter than stars" },
    { time = 160.90, text = "I'll erase you in sight!" },
    { time = 165.89, text = "Trying to stand when you're bound to fall?" },
    { time = 167.88, text = "Know your fate is my call!" },
    { time = 169.20, text = "Hanging now when you're on the line?" },
    { time = 171.09, text = "You're running out of time!" },
    { time = 172.14, text = "I wanna see you try" },
    { time = 173.84, text = "I want to hear you cry" },
    { time = 174.93, text = "Hanging on when you're on the line?" },
    { time = 176.76, text = "Saying it one more time.." },
    { time = 177.80, text = "(Go! Go! More power!)" },
    { time = 178.20, text = "I'm raw, pure power!" },
    { time = 180.70, text = "(Go! Go! More power!)" },
    { time = 183.68, text = "(Go! Go! More power!)" },
    { time = 185.18, text = "I want to see you trying" },
    { time = 186.67, text = "(Go! Go! More power!)" },
    { time = 188.17, text = "I want to see you panic!" },
    { time = 189.68, text = "NOW!" },
    { time = 193.72, text = " " },
}

-- =================================================================
-- LÓGICA ESP MEJORADO (UBICADO LIGERAMENTE ENCIMA DE LA CABEZA)
-- =================================================================

local function clearAllESP()
    for _, p in pairs(Players:GetPlayers()) do
        local char = p.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(p.Name))
        if char then
            local espFolder = char:FindFirstChild("OutcomeESP")
            if espFolder then espFolder:Destroy() end
        end
    end
end

local function updateESP()
    if not SETTINGS.Visual.ESP then
        clearAllESP()
        return
    end
    
    local myChar = player.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(player.Name))
    local myHrp = myChar and myChar:FindFirstChild("HumanoidRootPart")
    
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player then
            local char = p.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(p.Name))
            if char and char:FindFirstChild("HumanoidRootPart") then
                local targetHrp = char.HumanoidRootPart
                local targetHead = char:FindFirstChild("Head")
                local adorneePart = targetHead or targetHrp
                local offsetPos = targetHead and Vector3.new(0, 1.8, 0) or Vector3.new(0, 3.5, 0)
                
                local isAlive = true
                local hum = char:FindFirstChildOfClass("Humanoid")
                local healthVal = char:FindFirstChild("Health")
                
                if healthVal and healthVal:IsA("ValueBase") then
                    if healthVal.Value <= 0 then isAlive = false end
                elseif hum then
                    if hum.Health <= 0 then isAlive = false end
                end

                if isAlive then
                    local espFolder = char:FindFirstChild("OutcomeESP")
                    local isKiller = char:GetAttribute("Team") == "EXE" or p:GetAttribute("Team") == "EXE"
                    local charAttr = char:GetAttribute("Character")
                    if charAttr and LOCK_TARGETS[string.lower(charAttr)] then
                        isKiller = true
                    end

                    local color = isKiller and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(0, 255, 0)
                    local roleText = isKiller and "ASESINO" or "SUPERVIVIENTE"
                    
                    local distStuds = myHrp and math.floor((myHrp.Position - targetHrp.Position).Magnitude) or 0
                    local distText = distStuds .. " studs"
                    
                    if not espFolder then
                        espFolder = Instance.new("Folder")
                        espFolder.Name = "OutcomeESP"
                        espFolder.Parent = char

                        local hl = Instance.new("Highlight")
                        hl.Name = "ESPHighlight"
                        hl.Adornee = char
                        hl.FillTransparency = 0.5
                        hl.OutlineColor = Color3.fromRGB(0, 0, 0)
                        hl.OutlineTransparency = 0
                        hl.Parent = espFolder

                        local bb = Instance.new("BillboardGui")
                        bb.Name = "Tag"
                        bb.Size = UDim2.new(0, 180, 0, 50)
                        bb.AlwaysOnTop = true
                        bb.Adornee = adorneePart
                        bb.StudsOffset = offsetPos
                        bb.Parent = espFolder
                        
                        local lbl = Instance.new("TextLabel")
                        lbl.Name = "NameLabel"
                        lbl.BackgroundTransparency = 1
                        lbl.Size = UDim2.new(1, 0, 1, 0)
                        lbl.Font = Enum.Font.GothamBold
                        lbl.TextSize = 11
                        lbl.TextStrokeTransparency = 0
                        lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
                        lbl.Parent = bb
                    end
                    
                    if espFolder then
                        local hl = espFolder:FindFirstChild("ESPHighlight")
                        if hl then 
                            hl.FillColor = color 
                            hl.OutlineColor = Color3.fromRGB(0, 0, 0)
                            hl.OutlineTransparency = 0
                        end
                        local bb = espFolder:FindFirstChild("Tag")
                        if bb then
                            bb.Adornee = adorneePart
                            bb.StudsOffset = offsetPos
                            local lbl = bb:FindFirstChild("NameLabel")
                            if lbl then
                                lbl.Text = p.DisplayName .. "\n[" .. roleText .. "]\n" .. distText
                                lbl.TextColor3 = color
                                lbl.TextStrokeTransparency = 0
                                lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
                            end
                        end
                    end
                else
                    local espFolder = char:FindFirstChild("OutcomeESP")
                    if espFolder then espFolder:Destroy() end
                end
            end
        end
    end
end

-- Lógica de Hitbox Expander
local function updateHitboxes()
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player then
            local char = p.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(p.Name))
            if char then
                local hrp = char:FindFirstChild("HumanoidRootPart")
                if hrp then
                    if not originalHitboxSizes[hrp] then
                        originalHitboxSizes[hrp] = hrp.Size
                    end

                    local isKiller = char:GetAttribute("Team") == "EXE" or p:GetAttribute("Team") == "EXE"
                    local shouldExpand = false

                    if SETTINGS.Hitbox.Enabled then
                        if SETTINGS.Hitbox.TargetTeam == "All" then
                            shouldExpand = true
                        elseif SETTINGS.Hitbox.TargetTeam == "EXE" and isKiller then
                            shouldExpand = true
                        elseif SETTINGS.Hitbox.TargetTeam == "Survivor" and not isKiller then
                            shouldExpand = true
                        end
                    end

                    if shouldExpand then
                        hrp.Size = Vector3.new(SETTINGS.Hitbox.Size, SETTINGS.Hitbox.Size, SETTINGS.Hitbox.Size)
                        hrp.Transparency = 0.7
                        hrp.Color = isKiller and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(0, 255, 0)
                        hrp.CanCollide = false
                    else
                        if originalHitboxSizes[hrp] then
                            hrp.Size = originalHitboxSizes[hrp]
                            hrp.Transparency = 1
                        end
                    end
                end
            end
        end
    end
end

local espUpdateInterval = 0.1
local lastESPUpdate = 0
local renderConnection

renderConnection = RunService.RenderStepped:Connect(function(dt)
    local now = tick()
    
    if now - lastESPUpdate >= espUpdateInterval then
        updateESP()
        updateHitboxes()
        lastESPUpdate = now
    end

    if SETTINGS.Visual.Fullbright then
        Lighting.Ambient = Color3.fromRGB(255, 255, 255)
        Lighting.OutdoorAmbient = Color3.fromRGB(255, 255, 255)
        Lighting.Brightness = 2
        Lighting.ClockTime = 14
        Lighting.FogEnd = 100000
        Lighting.GlobalShadows = false
    end

    local char = player.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(player.Name))
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local humanoid = char and char:FindFirstChild("Humanoid")

    if speedUpEnabled and hrp and humanoid then
        local moveDir = humanoid.MoveDirection
        if moveDir.Magnitude > 0 then
            hrp.CFrame = hrp.CFrame + (moveDir * currentSpeed * dt)
        end
    end

    -- Control de Knuckles
    if not knucklesScriptDisabled and isKnuckles() and char and humanoid and hrp then
        disableDefaultKnucklesAnimations(humanoid)

        if spaceHeld and not isFlying and knucklesFlightEnabled then
            spaceTimer = spaceTimer + dt
            if spaceTimer >= HOLD_TIME then
                startFly(hrp, humanoid)
            end
        end

        if isFlying and bodyVelocity then
            local camCF = camera.CFrame
            bodyVelocity.Velocity = camCF.LookVector * FLY_SPEED
            if bodyGyro then
                bodyGyro.CFrame = CFrame.new(hrp.Position, hrp.Position + camCF.LookVector)
            end
        end

        punchTimer = punchTimer + dt
        if punchTimer >= 1 then
            punchTimer = 0
            applyPunchTech()
        end

        updateKnucklesAnimations(humanoid)
    else
        if isFlying then stopFly(humanoid) end
    end

    -- TriggerBot
    if SETTINGS.TriggerBot.Enabled and hrp then
        if not probe or probe.Parent ~= char then
            if probe then probe:Destroy(); probe = nil; probeWeld = nil end
            
            probe = Instance.new("Part")
            probe.Name = "Outcome_Trigger_Fixed"
            probe.CanCollide = false
            probe.CanQuery = false
            probe.CanTouch = false
            probe.Massless = true
            probe.Transparency = 0.7
            probe.Color = Color3.fromRGB(255, 255, 0)
            probe.Parent = char

            probeWeld = Instance.new("Weld")
            probeWeld.Part0 = hrp
            probeWeld.Part1 = probe
            probeWeld.Parent = probe
        end

        local dist = SETTINGS.TriggerBot.Distance
        local width = SETTINGS.TriggerBot.Width 
        probe.Size = Vector3.new(width, 7, dist)
        probeWeld.C1 = CFrame.new(0, 0, dist/2)

        if now - lastAttackTime >= SETTINGS.TriggerBot.Cooldown then
            local foundEnemy = false
            for _, part in ipairs(workspace:GetPartsInPart(probe)) do
                local m = part.Parent
                if m and m:FindFirstChild("Humanoid") and m ~= char then
                    foundEnemy = true
                    break
                end
            end

            if foundEnemy then
                lastAttackTime = now
                pcall(function()
                    if mouse1press then 
                        mouse1press()
                        task.wait(0.01)
                        mouse1release()
                    else 
                        game:GetService("VirtualUser"):ClickButton1(Vector2.new(0,0)) 
                    end
                end)
            end
        end
    else
        if probe then probe:Destroy(); probe = nil; probeWeld = nil end
    end
end)

Players.PlayerRemoving:Connect(function(p)
    local char = p.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(p.Name))
    if char then
        local espFolder = char:FindFirstChild("OutcomeESP")
        if espFolder then espFolder:Destroy() end
    end
end)

-- =================================================================
-- MÚSICA DE FLEETWAY (LOW HP - SOLARA SAFE)
-- =================================================================
local LOW_HP_THRESHOLD = 150
local TENSION_VOLUME = 1
local PLAY_DELAY = 0.5
local FREEZE_TIME_REQUIRED = 1

local function loadCustomAssetSafe(url, filename)
    local success, result = pcall(function()
        if isfile and writefile and game.HttpGet and not isfile(filename) then
            writefile(filename, game:HttpGet(url))
        end
        return (getcustomasset and getcustomasset(filename)) or ""
    end)
    return success and result or ""
end

local FLEETWAY_MUSIC_ID = loadCustomAssetSafe(
    "https://raw.githubusercontent.com/sonicmodder456/test/main/video-output-72638258-7064-4008-8024-C6328C3026D0-1.mp3",
    "fleetway.mp3"
)

local tensionMusic = Instance.new("Sound")
tensionMusic.Name = "FleetwayLowHP"
tensionMusic.SoundId = FLEETWAY_MUSIC_ID
tensionMusic.Looped = true
tensionMusic.Volume = 0
tensionMusic.Parent = SoundService

pcall(function() ContentProvider:PreloadAsync({tensionMusic}) end)

local mutedSounds = {}
local triggeredThisRound = false
local lastTimerValue = nil
local freezeStartTime = nil

local function getFleetwayModel()
    local folder = workspace:FindFirstChild("Players")
    if not folder then return nil end
    for _, child in ipairs(folder:GetChildren()) do
        if child:GetAttribute("Character") == "Fleetway" then
            return child
        end
    end
    return nil
end

local function getTimer()
    local gp = workspace:FindFirstChild("GameProperties")
    return gp and gp:FindFirstChild("Time")
end

local function muteGameSounds()
    mutedSounds = {}
    pcall(function()
        local soloFolder = ReplicatedStorage:FindFirstChild("ClientAssets")
        if soloFolder then soloFolder = soloFolder:FindFirstChild("Sounds") end
        if soloFolder then soloFolder = soloFolder:FindFirstChild("mus") end
        if soloFolder then soloFolder = soloFolder:FindFirstChild("Game") end
        if soloFolder then soloFolder = soloFolder:FindFirstChild("Round") end
        if soloFolder then soloFolder = soloFolder:FindFirstChild("SoloTheme") end
        if soloFolder then
            for _, sound in ipairs(soloFolder:GetDescendants()) do
                if sound:IsA("Sound") then
                    mutedSounds[sound] = sound.Volume
                    sound.Volume = 0
                end
            end
        end
    end)
    pcall(function()
        for _, sound in ipairs(workspace:GetDescendants()) do
            if sound:IsA("Sound") and sound ~= tensionMusic then
                mutedSounds[sound] = sound.Volume
                sound.Volume = 0
            end
        end
    end)
end

local function restoreSounds()
    for sound, volume in pairs(mutedSounds) do
        pcall(function()
            if sound and sound.Parent then
                sound.Volume = volume
            end
        end)
    end
    mutedSounds = {}
end

local function stopFleetwayMusic()
    pcall(function()
        tensionMusic:Stop()
        tensionMusic.Volume = 0
    end)
    restoreSounds()
    triggeredThisRound = false
end

task.spawn(function()
    while true do
        task.wait(0.2)
        if not lmsStates["Fleetway"] then
            stopFleetwayMusic()
            continue
        end

        local model = getFleetwayModel()
        local timer = getTimer()

        if not model or not timer then
            stopFleetwayMusic()
            lastTimerValue = nil
            freezeStartTime = nil
            continue
        end

        local currentTimer = timer.Value
        if currentTimer <= 0 then
            stopFleetwayMusic()
            lastTimerValue = nil
            freezeStartTime = nil
            continue
        end

        if lastTimerValue == nil then
            lastTimerValue = currentTimer
        else
            if currentTimer == lastTimerValue then
                if not freezeStartTime then
                    freezeStartTime = tick()
                elseif tick() - freezeStartTime >= FREEZE_TIME_REQUIRED then
                    stopFleetwayMusic()
                end
            else
                lastTimerValue = currentTimer
                freezeStartTime = nil
            end
        end

        local health = model:FindFirstChild("Health")
        if not health then continue end

        if health.Value > 0 and health.Value <= LOW_HP_THRESHOLD and not triggeredThisRound then
            triggeredThisRound = true
            muteGameSounds()
            task.wait(PLAY_DELAY)
            if timer and timer.Value > 0 then
                tensionMusic.Volume = TENSION_VOLUME
                if not tensionMusic.IsPlaying then 
                    tensionMusic:Play() 
                end
            end
        end
    end
end)

-- =================================================================
-- BOTONES FLOTANTES DE CONTROL MÓVIL
-- =================================================================

local mobileGui = Instance.new("ScreenGui")
mobileGui.Name = "MobileControlsUI"
mobileGui.ResetOnSpawn = false
mobileGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

pcall(function() mobileGui.Parent = targetGui end)
if not mobileGui.Parent then mobileGui.Parent = playerGui end

local menuToggleBtn = Instance.new("TextButton")
menuToggleBtn.Size = UDim2.new(0, 50, 0, 50)
menuToggleBtn.Position = UDim2.new(0.02, 0, 0.25, 0)
menuToggleBtn.BackgroundColor3 = Color3.fromRGB(139, 0, 0)
menuToggleBtn.Text = "MENU"
menuToggleBtn.TextColor3 = Color3.new(1, 1, 1)
menuToggleBtn.Font = Enum.Font.GothamBold
menuToggleBtn.TextSize = 10
menuToggleBtn.Active = true
menuToggleBtn.Draggable = true
menuToggleBtn.Parent = mobileGui

local menuCorner = Instance.new("UICorner")
menuCorner.CornerRadius = UDim.new(0, 10)
menuCorner.Parent = menuToggleBtn

local aimBtn = Instance.new("TextButton")
aimBtn.Size = UDim2.new(0, 50, 0, 50)
aimBtn.Position = UDim2.new(0.02, 0, 0.35, 0)
aimBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
aimBtn.Text = "AIM [T]"
aimBtn.TextColor3 = Color3.new(1, 1, 1)
aimBtn.Font = Enum.Font.GothamBold
aimBtn.TextSize = 9
aimBtn.Active = true
aimBtn.Draggable = true
aimBtn.Parent = mobileGui

local aimCorner = Instance.new("UICorner")
aimCorner.CornerRadius = UDim.new(0, 10)
aimCorner.Parent = aimBtn

aimBtn.MouseButton1Click:Connect(function()
    setCursorLockState(not cursorLockEnabled)
    aimBtn.BackgroundColor3 = cursorLockEnabled and Color3.fromRGB(255, 204, 0) or Color3.fromRGB(40, 40, 40)
    aimBtn.TextColor3 = cursorLockEnabled and Color3.fromRGB(0, 0, 0) or Color3.new(1, 1, 1)
end)

local gBtn = Instance.new("TextButton")
gBtn.Size = UDim2.new(0, 50, 0, 50)
gBtn.Position = UDim2.new(0.02, 0, 0.45, 0)
gBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
gBtn.Text = "BOOST\n[G]"
gBtn.TextColor3 = Color3.new(1, 1, 1)
gBtn.Font = Enum.Font.GothamBold
gBtn.TextSize = 8
gBtn.Active = true
gBtn.Draggable = true
gBtn.Parent = mobileGui

local gCorner = Instance.new("UICorner")
gCorner.CornerRadius = UDim.new(0, 10)
gCorner.Parent = gBtn

gBtn.MouseButton1Click:Connect(function()
    if flyGuiEnabled then
        toggleFlyGui()
    end
    
    local model = getEggmanPlayerModel()
    if model and model:GetAttribute("Character") == "Eggman" then
        toggleEggmanBoost()
    end
end)

-- =================================================================
-- INTERFAZ UNIFICADA EN UN SOLO PANEL
-- =================================================================

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "UnifiedHubUI"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

pcall(function() screenGui.Parent = targetGui end)
if not screenGui.Parent then screenGui.Parent = playerGui end

menuToggleBtn.MouseButton1Click:Connect(function()
    screenGui.Enabled = not screenGui.Enabled
end)

-- Subtítulos / Letras en pantalla
local lyricsLabel = Instance.new("TextLabel")
lyricsLabel.Size = UDim2.new(0.6, 0, 0.08, 0)
lyricsLabel.Position = UDim2.new(0.2, 0, 0.77, 0)
lyricsLabel.BackgroundTransparency = 1
lyricsLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
lyricsLabel.TextStrokeTransparency = 0
lyricsLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
lyricsLabel.Font = Enum.Font.GothamBold
lyricsLabel.TextScaled = true
lyricsLabel.Text = ""
lyricsLabel.ZIndex = 19
lyricsLabel.Parent = screenGui

-- Contenedor Principal
local mainContainer = Instance.new("Frame")
mainContainer.Size = UDim2.new(0, 360, 0, 260)
mainContainer.Position = UDim2.new(0.5, -180, 0.3, 0)
mainContainer.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
mainContainer.BorderSizePixel = 0
mainContainer.Parent = screenGui

local containerCorner = Instance.new("UICorner")
containerCorner.CornerRadius = UDim.new(0, 8)
containerCorner.Parent = mainContainer

local dragging, dragInput, dragStart, startPos

local function updateInput(input)
    local delta = input.Position - dragStart
    mainContainer.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end

local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 35)
header.BackgroundColor3 = Color3.fromRGB(139, 0, 0) 
header.BorderSizePixel = 0
header.Parent = mainContainer

local headerCorner = Instance.new("UICorner")
headerCorner.CornerRadius = UDim.new(0, 8)
headerCorner.Parent = header
local headerFix = Instance.new("Frame")
headerFix.Size = UDim2.new(1, 0, 0, 10)
headerFix.Position = UDim2.new(0, 0, 1, -10)
headerFix.BackgroundColor3 = Color3.fromRGB(139, 0, 0)
headerFix.BorderSizePixel = 0
headerFix.Parent = header

header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = mainContainer.Position
        
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)
header.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        updateInput(input)
    end
end)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -100, 1, 0)
title.Position = UDim2.new(0, 15, 0, 0)
title.Text = "om.hub | dc: @deadshizik | Tecla: Insert"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 12
title.TextXAlignment = Enum.TextXAlignment.Left
title.BackgroundTransparency = 1
title.Parent = header

local mainBody = Instance.new("Frame")
mainBody.Size = UDim2.new(1, 0, 1, -35)
mainBody.Position = UDim2.new(0, 0, 0, 35)
mainBody.BackgroundTransparency = 1
mainBody.Parent = mainContainer

local minimizeBtn = Instance.new("TextButton")
minimizeBtn.Size = UDim2.new(0, 25, 0, 25)
minimizeBtn.Position = UDim2.new(1, -60, 0, 5)
minimizeBtn.BackgroundColor3 = Color3.fromRGB(255, 204, 0)
minimizeBtn.BorderSizePixel = 0
minimizeBtn.Text = "-"
minimizeBtn.TextColor3 = Color3.new(0, 0, 0)
minimizeBtn.Font = Enum.Font.GothamBold
minimizeBtn.TextSize = 18
minimizeBtn.Parent = header

local minCorner = Instance.new("UICorner")
minCorner.CornerRadius = UDim.new(0, 4)
minCorner.Parent = minimizeBtn

minimizeBtn.MouseButton1Click:Connect(function()
    mainBody.Visible = not mainBody.Visible
    mainContainer.Size = mainBody.Visible and UDim2.new(0, 360, 0, 260) or UDim2.new(0, 360, 0, 35)
    minimizeBtn.Text = mainBody.Visible and "-" or "+"
end)

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 25, 0, 25)
closeBtn.Position = UDim2.new(1, -30, 0, 5)
closeBtn.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
closeBtn.BorderSizePixel = 0
closeBtn.Text = "X"
closeBtn.TextColor3 = Color3.new(1, 1, 1)
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 14
closeBtn.Parent = header

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 4)
closeCorner.Parent = closeBtn

closeBtn.MouseButton1Click:Connect(function()
    screenGui:Destroy()
    mobileGui:Destroy()
end)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    if input.KeyCode == Enum.KeyCode.Insert then
        mainContainer.Visible = not mainContainer.Visible
    elseif input.KeyCode == Enum.KeyCode.Space then
        spaceHeld = true
        spaceTimer = 0
    elseif input.KeyCode == Enum.KeyCode.LeftShift or input.KeyCode == Enum.KeyCode.RightShift then
        shiftHeld = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.Space then
        spaceHeld = false
        spaceTimer = 0
        local char = player.Character
        local hum = char and char:FindFirstChild("Humanoid")
        stopFly(hum)
    elseif input.KeyCode == Enum.KeyCode.LeftShift or input.KeyCode == Enum.KeyCode.RightShift then
        shiftHeld = false
        currentKnucklesAnim = nil
    end
end)

local sideBar = Instance.new("Frame")
sideBar.Size = UDim2.new(0, 90, 1, 0)
sideBar.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
sideBar.BorderSizePixel = 0
sideBar.Parent = mainBody

local sideBarCorner = Instance.new("UICorner")
sideBarCorner.CornerRadius = UDim.new(0, 8)
sideBarCorner.Parent = sideBar
local sideBarFix1 = Instance.new("Frame")
sideBarFix1.Size = UDim2.new(0, 10, 1, 0)
sideBarFix1.Position = UDim2.new(1, -10, 0, 0)
sideBarFix1.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
sideBarFix1.BorderSizePixel = 0
sideBarFix1.Parent = sideBar
local sideBarFix2 = Instance.new("Frame")
sideBarFix2.Size = UDim2.new(1, 0, 0, 10)
sideBarFix2.Position = UDim2.new(0, 0, 0, 0)
sideBarFix2.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
sideBarFix2.BorderSizePixel = 0
sideBarFix2.Parent = sideBar

local function createToggle(parent, text, default, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 30)
    frame.BackgroundTransparency = 1
    frame.Parent = parent
    
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0, 130, 1, 0)
    lbl.Text = text:lower()
    lbl.TextColor3 = Color3.fromRGB(180, 180, 180)
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 11
    lbl.BackgroundTransparency = 1
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = frame
    
    local bg = Instance.new("Frame")
    bg.Size = UDim2.new(0, 34, 0, 16)
    bg.Position = UDim2.new(1, -34, 0.5, -8)
    bg.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    bg.BorderSizePixel = 1
    bg.BorderColor3 = Color3.fromRGB(50, 50, 50)
    bg.Parent = frame
    
    local bgCorner = Instance.new("UICorner")
    bgCorner.CornerRadius = UDim.new(1, 0)
    bgCorner.Parent = bg
    
    local box = Instance.new("Frame")
    box.Size = UDim2.new(0, 10, 0, 10)
    box.Position = default and UDim2.new(1, -12, 0.5, -5) or UDim2.new(0, 2, 0.5, -5)
    box.BackgroundColor3 = default and Color3.fromRGB(255, 204, 0) or Color3.fromRGB(60, 60, 60)
    box.BorderSizePixel = 0
    box.Parent = bg
    
    local boxCorner = Instance.new("UICorner")
    boxCorner.CornerRadius = UDim.new(1, 0)
    boxCorner.Parent = box
    
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 1, 0)
    btn.BackgroundTransparency = 1
    btn.Text = ""
    btn.Parent = bg
    
    btn.MouseButton1Click:Connect(function()
        default = not default
        box.Position = default and UDim2.new(1, -12, 0.5, -5) or UDim2.new(0, 2, 0.5, -5)
        box.BackgroundColor3 = default and Color3.fromRGB(255, 204, 0) or Color3.fromRGB(60, 60, 60)
        callback(default)
    end)
end

-- =================================================================
-- CONTROL DESLIZANTE ESTILO VERTICAL iOS (PARA SPEED UP)
-- =================================================================
local function createSlider(parent, text, minV, maxV, default, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 130)
    frame.BackgroundTransparency = 1
    frame.Parent = parent
    
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, 0, 0, 15)
    lbl.Position = UDim2.new(0, 0, 0, 0)
    lbl.Text = text:lower()
    lbl.TextColor3 = Color3.fromRGB(180, 180, 180)
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.BackgroundTransparency = 1
    lbl.Parent = frame
    
    local iosBack = Instance.new("Frame")
    iosBack.Size = UDim2.new(0, 48, 0, 105)
    iosBack.Position = UDim2.new(0, 10, 0, 20)
    iosBack.BackgroundColor3 = Color3.fromRGB(35, 33, 30)
    iosBack.BorderSizePixel = 0
    iosBack.ClipsDescendants = true
    iosBack.Parent = frame
    
    local iosCorner = Instance.new("UICorner")
    iosCorner.CornerRadius = UDim.new(0, 16)
    iosCorner.Parent = iosBack
    
    local fillBar = Instance.new("Frame")
    local startRel = math.clamp((default - minV) / (maxV - minV), 0, 1)
    fillBar.Size = UDim2.new(1, 0, startRel, 0)
    fillBar.Position = UDim2.new(0, 0, 1 - startRel, 0)
    fillBar.BackgroundColor3 = Color3.fromRGB(240, 238, 233)
    fillBar.BorderSizePixel = 0
    fillBar.Parent = iosBack
    
    local icon = Instance.new("ImageLabel")
    icon.Size = UDim2.new(0, 22, 0, 22)
    icon.Position = UDim2.new(0.5, -11, 1, -30)
    icon.BackgroundTransparency = 1
    icon.Image = "rbxassetid://6031071057"
    icon.ImageColor3 = Color3.fromRGB(80, 80, 80)
    icon.ZIndex = 3
    icon.Parent = iosBack
    
    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(0, 80, 0, 20)
    valLbl.Position = UDim2.new(0, 70, 0, 60)
    valLbl.Text = "Valor: " .. tostring(default)
    valLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    valLbl.Font = Enum.Font.GothamBold
    valLbl.TextSize = 12
    valLbl.TextXAlignment = Enum.TextXAlignment.Left
    valLbl.BackgroundTransparency = 1
    valLbl.Parent = frame

    local draggingSlider = false

    local function updateFromInput(input)
        local totalY = iosBack.AbsoluteSize.Y
        local offsetY = math.clamp(input.Position.Y - iosBack.AbsolutePosition.Y, 0, totalY)
        local rel = 1 - (offsetY / totalY)
        local val = math.round(minV + (maxV - minV) * rel)
        
        fillBar.Size = UDim2.new(1, 0, rel, 0)
        fillBar.Position = UDim2.new(0, 0, 1 - rel, 0)
        valLbl.Text = "Valor: " .. tostring(val)
        
        if rel > 0.3 then
            icon.ImageColor3 = Color3.fromRGB(100, 100, 100)
        else
            icon.ImageColor3 = Color3.fromRGB(180, 180, 180)
        end

        callback(val)
    end

    iosBack.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = true
            updateFromInput(input)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if draggingSlider and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            updateFromInput(input)
        end
    end)
end

local function createActionButton(parent, text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -10, 0, 26)
    btn.Text = text:lower()
    btn.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 10
    btn.Parent = parent
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = btn

    btn.MouseButton1Click:Connect(callback)
end

local tabs = {}

local function createTabFrame()
    local f = Instance.new("ScrollingFrame")
    f.Size = UDim2.new(1, -110, 1, -20)
    f.Position = UDim2.new(0, 100, 0, 10)
    f.BackgroundTransparency = 1
    f.BorderSizePixel = 0
    f.ScrollBarThickness = 2
    f.Visible = false
    f.Parent = mainBody
    
    local layout = Instance.new("UIListLayout")
    layout.Parent = f
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Padding = UDim.new(0, 6)
    
    layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        f.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 10)
    end)
    
    return f
end

local killerTab    = createTabFrame()
local survivorTab  = createTabFrame()
local visualTab    = createTabFrame()
local autoPickTab  = createTabFrame()
local lmsTab       = createTabFrame()
local localTab     = createTabFrame()
local teleportsTab = createTabFrame()

killerTab.Visible = true

local tabCount = 0
local function makeTabBtn(name, frame)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, 0, 0, 28)
    b.Position = UDim2.new(0, 0, 0, tabCount * 28)
    b.Text = name:lower()
    b.Font = Enum.Font.GothamBold
    b.TextSize = 10
    b.BorderSizePixel = 0
    b.BackgroundColor3 = frame.Visible and Color3.fromRGB(255, 204, 0) or Color3.fromRGB(100, 0, 0)
    b.TextColor3 = frame.Visible and Color3.fromRGB(0, 0, 0) or Color3.new(1, 1, 1)
    b.Parent = sideBar
    
    table.insert(tabs, {btn = b, frame = frame})
    tabCount = tabCount + 1

    b.MouseButton1Click:Connect(function()
        for _, t in ipairs(tabs) do
            t.frame.Visible = false
            t.btn.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
            t.btn.TextColor3 = Color3.new(1, 1, 1)
        end
        frame.Visible = true
        b.BackgroundColor3 = Color3.fromRGB(255, 204, 0)
        b.TextColor3 = Color3.fromRGB(0, 0, 0)
    end)
end

makeTabBtn("killer", killerTab)
makeTabBtn("survivor", survivorTab)
makeTabBtn("visual", visualTab)
makeTabBtn("auto pick", autoPickTab)
makeTabBtn("lms", lmsTab)
makeTabBtn("local", localTab)
makeTabBtn("teleportes", teleportsTab)

-- Opciones de KILLER (Hitbox, TriggerBot)
createToggle(killerTab, "hitbox expander", SETTINGS.Hitbox.Enabled, function(v) SETTINGS.Hitbox.Enabled = v; updateHitboxes() end)
createSlider(killerTab, "hitbox size", 2, 30, SETTINGS.Hitbox.Size, function(v) SETTINGS.Hitbox.Size = v; updateHitboxes() end)
createToggle(killerTab, "trigger bot", SETTINGS.TriggerBot.Enabled, function(v) SETTINGS.TriggerBot.Enabled = v end)
createSlider(killerTab, "width", 1, 25, SETTINGS.TriggerBot.Width, function(v) SETTINGS.TriggerBot.Width = v end)
createSlider(killerTab, "distance", 1, 25, SETTINGS.TriggerBot.Distance, function(v) SETTINGS.TriggerBot.Distance = v end)

-- Opciones de SURVIVOR
createToggle(survivorTab, "aimlock / cursor lock (exe) [t]", false, function(v) setCursorLockState(v) end)
createToggle(survivorTab, "sonic headturn / tags", false, function(v) setSonicAnimState(v) end)
createToggle(survivorTab, "eggman boost [g]", false, function(v) setEggmanBoostState(v) end)
createToggle(survivorTab, "tails fly [g]", false, function(v) setFlyGuiState(v) end)
createSlider(survivorTab, "tails fly speed", 10, 200, FLY_SPEED_GUI, function(v) FLY_SPEED_GUI = v end)

createToggle(survivorTab, "fly inf fleetway", false, function(v)
    flyInfFleetwayEnabled = v
    toggleFlyInf(v)
end)
createToggle(survivorTab, "fly silver inf", false, function(v)
    flySilverInfEnabled = v
    toggleFlyInf(v)
end)

createToggle(survivorTab, "disable knuckles script", false, function(v)
    knucklesScriptDisabled = v
    if v then
        local char = player.Character
        local hum = char and char:FindFirstChild("Humanoid")
        stopFly(hum)
        stopAllKnucklesAnimations()
    end
end)
createToggle(survivorTab, "knuckles flight", knucklesFlightEnabled, function(v) knucklesFlightEnabled = v end)
createToggle(survivorTab, "punch tech", knucklesPunchTechEnabled, function(v) knucklesPunchTechEnabled = v end)
createSlider(survivorTab, "fly speed", 10, 150, FLY_SPEED, function(v) FLY_SPEED = v end)

createToggle(visualTab, "esp", SETTINGS.Visual.ESP, function(v) SETTINGS.Visual.ESP = v end)
createToggle(visualTab, "tripwire mines esp", SETTINGS.Visual.MinesESP, function(v) 
    SETTINGS.Visual.MinesESP = v 
    if not v then
        clearAllMinesESP()
    else
        ScanExistingMines()
    end
end)
createToggle(visualTab, "fullbright", SETTINGS.Visual.Fullbright, function(v) 
    SETTINGS.Visual.Fullbright = v 
    if not v then
        Lighting.Ambient = originalLighting.Ambient
        Lighting.OutdoorAmbient = originalLighting.OutdoorAmbient
        Lighting.Brightness = originalLighting.Brightness
        Lighting.ClockTime = originalLighting.ClockTime
        Lighting.FogEnd = originalLighting.FogEnd
        Lighting.GlobalShadows = originalLighting.GlobalShadows
    end
end)

-- =================================================================
-- LÓGICA DE TELEPORTES
-- =================================================================

local function teleportTo(targetCFrame)
    local char = player.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(player.Name))
    if char and char:FindFirstChild("HumanoidRootPart") then
        char:SetPrimaryPartCFrame(targetCFrame)
    end
end

local tpPickContainer = Instance.new("Frame")
tpPickContainer.Size = UDim2.new(1, -10, 0, 100)
tpPickContainer.BackgroundTransparency = 1
tpPickContainer.Parent = teleportsTab

local tpImgPreview = Instance.new("ImageLabel")
tpImgPreview.Size = UDim2.new(0, 70, 0, 70)
tpImgPreview.Position = UDim2.new(0.5, -35, 0, 0)
tpImgPreview.BackgroundTransparency = 1
tpImgPreview.Parent = tpPickContainer

local tpPrevBtn = Instance.new("TextButton")
tpPrevBtn.Size = UDim2.new(0, 30, 0, 30)
tpPrevBtn.Position = UDim2.new(0, 10, 0, 20)
tpPrevBtn.Text = "<"
tpPrevBtn.TextColor3 = Color3.new(1, 1, 1)
tpPrevBtn.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
tpPrevBtn.Parent = tpPickContainer

local tpPrevCorner = Instance.new("UICorner")
tpPrevCorner.CornerRadius = UDim.new(0, 4)
tpPrevCorner.Parent = tpPrevBtn

local tpNextBtn = Instance.new("TextButton")
tpNextBtn.Size = UDim2.new(0, 30, 0, 30)
tpNextBtn.Position = UDim2.new(1, -40, 0, 20)
tpNextBtn.Text = ">"
tpNextBtn.TextColor3 = Color3.new(1, 1, 1)
tpNextBtn.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
tpNextBtn.Parent = tpPickContainer

local tpNextCorner = Instance.new("UICorner")
tpNextCorner.CornerRadius = UDim.new(0, 4)
tpNextCorner.Parent = tpNextBtn

local tpNameLabel = Instance.new("TextLabel")
tpNameLabel.Size = UDim2.new(1, 0, 0, 20)
tpNameLabel.Position = UDim2.new(0, 0, 0, 75)
tpNameLabel.TextColor3 = Color3.new(1, 1, 1)
tpNameLabel.Font = Enum.Font.GothamBold
tpNameLabel.TextSize = 12
tpNameLabel.BackgroundTransparency = 1
tpNameLabel.Parent = tpPickContainer

local tpTargetBtn = Instance.new("TextButton")
tpTargetBtn.Size = UDim2.new(1, -10, 0, 26)
tpTargetBtn.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
tpTargetBtn.TextColor3 = Color3.new(1, 1, 1)
tpTargetBtn.Font = Enum.Font.GothamBold
tpTargetBtn.TextSize = 10
tpTargetBtn.Parent = teleportsTab

local tpTargetCorner = Instance.new("UICorner")
tpTargetCorner.CornerRadius = UDim.new(0, 6)
tpTargetCorner.Parent = tpTargetBtn

local tpCurrentIndex = 0
local tpSelectedCharacter = characters[1]

local function updateTpPicker()
    tpSelectedCharacter = characters[tpCurrentIndex + 1]
    tpNameLabel.Text = tpSelectedCharacter
    tpImgPreview.Image = characterImages[tpSelectedCharacter] or ""
    tpTargetBtn.Text = "tp a " .. string.lower(tpSelectedCharacter)
end

updateTpPicker()

tpPrevBtn.MouseButton1Click:Connect(function()
    tpCurrentIndex = (tpCurrentIndex - 1) % #characters
    updateTpPicker()
end)

tpNextBtn.MouseButton1Click:Connect(function()
    tpCurrentIndex = (tpCurrentIndex + 1) % #characters
    updateTpPicker()
end)

tpTargetBtn.MouseButton1Click:Connect(function()
    local found = false
    for _, p in pairs(Players:GetPlayers()) do
        local char = p.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(p.Name))
        if p ~= player and char and char:FindFirstChild("HumanoidRootPart") then
            if char:GetAttribute("Character") == tpSelectedCharacter then
                teleportTo(char.HumanoidRootPart.CFrame * CFrame.new(0, 0, 3))
                found = true
                break
            end
        end
    end
    if not found then
        pcall(function()
            StarterGui:SetCore("SendNotification", {
                Title = "om.hub",
                Text = "No se encontró a nadie jugando como " .. tpSelectedCharacter,
                Duration = 3
            })
        end)
    end
end)

createActionButton(teleportsTab, "tp al asesino", function()
    for _, p in pairs(Players:GetPlayers()) do
        local char = p.Character or (workspace:FindFirstChild("Players") and workspace.Players:FindFirstChild(p.Name))
        if p ~= player and char and char:FindFirstChild("HumanoidRootPart") then
            local isKiller = char:GetAttribute("Team") == "EXE" or p:GetAttribute("Team") == "EXE"
            if isKiller then
                teleportTo(char.HumanoidRootPart.CFrame * CFrame.new(0
