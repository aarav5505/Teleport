local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local RS = game:GetService("ReplicatedStorage")
local Config = require(script.Parent.Config)
local Destination = require(script.Parent.Destination)
local Visual = require(script.Parent.Visual)
assert(not RS:FindFirstChild("BeaconRemote"), "Install only one BeaconKit")
local remote = Instance.new("RemoteEvent")
remote.Name = "BeaconRemote"
remote.Parent = RS
local effects = Instance.new("Folder")
effects.Name = "BeaconEffects"
effects.Parent = workspace
local states = {}
local function clear(s)
    if s.beacon then s.beacon:Destroy() end
    s.beacon, s.flight, s.point, s.support, s.supportFrame = nil, nil, nil, nil, nil
    s.visual, s.canTeleport, s.expiryTime = nil, false, 0
end
local function tell(player, s, message)
    s.tool:SetAttribute("BeaconState", s.flight and "Flying" or s.point and "Ready" or "Empty")
    s.tool:SetAttribute("CooldownUntil", s.cooldown)
    s.tool:SetAttribute("CanTeleport", s.canTeleport == true)
    s.tool:SetAttribute("ExpiresAt", s.expiryTime or 0)
    s.tool:SetAttribute("Lifetime", Config.Lifetime)
    s.tool:SetAttribute("CooldownDuration", Config.Cooldown)
    s.tool:SetAttribute("Status", message)
    s.tool.ToolTip = message
end
local function characterFor(player, s)
    local c = player.Character
    local h = c and c:FindFirstChildOfClass("Humanoid")
    local r = c and c:FindFirstChild("HumanoidRootPart")
    if c and h and r and h.Health > 0 and not h.Sit and s.tool.Parent == c then return c,h,r end
end
local function finite(v)
    return typeof(v) == "Vector3" and v.X == v.X and v.Y == v.Y and v.Z == v.Z
        and math.abs(v.X) < 1e6 and math.abs(v.Y) < 1e6 and math.abs(v.Z) < 1e6
end
local function setup(player)
    if states[player] then return end
    local tool = script.Parent.Beacon:Clone()
    local s = {tool = tool, cooldown = 0, lastRequest = -math.huge, lastThrow = -math.huge, connections = {}}
    states[player] = s
    local function spawn(character)
        clear(s)
        if s.death then s.death:Disconnect(); s.death = nil end
        local h = character:WaitForChild("Humanoid", 10)
        local backpack = player:WaitForChild("Backpack", 10)
        if states[player] ~= s or player.Character ~= character or not h or not backpack then return end
        if s.tool then s.tool:Destroy() end
        s.tool = script.Parent.Beacon:Clone()
        s.tool.Parent = backpack
        tell(player, s, "Click: throw | E: teleport | R: replace")
        s.death = h.Died:Connect(function() clear(s); tell(player,s,"Beacon cleared on death") end)
    end
    table.insert(s.connections, player.CharacterAdded:Connect(spawn))
    table.insert(s.connections, player.CharacterRemoving:Connect(function() clear(s) end))
    if player.Character then task.spawn(spawn, player.Character) end
end
remote.OnServerEvent:Connect(function(player, action, direction)
    local s = states[player]
    if not s or type(action) ~= "string" then return end
    local now = os.clock()
    if now - s.lastRequest < 0.15 then return end
    s.lastRequest = now
    local character, _, root = characterFor(player,s)
    if not character then return end
    if workspace:GetServerTimeNow() < s.cooldown then tell(player,s,"Cooling down"); return end
    if action == "Throw" then
        if not finite(direction) or direction.Magnitude < 0.01 or now-s.lastThrow < Config.ThrowInterval then return end
        direction = direction.Unit
        s.lastThrow = now
        clear(s)
        local bead = Instance.new("Part")
        bead.Name = "Beacon_" .. player.UserId
        bead.Shape = Enum.PartType.Ball
        bead.Size = Vector3.one * 0.8
        bead.Material = Enum.Material.Neon
        bead.Color = Color3.fromRGB(255,180,50)
        bead.Anchored, bead.CanCollide, bead.CanTouch, bead.CanQuery = true, false, false, false
        bead.Position = root.Position + Vector3.new(0,1,0)
        bead.Parent = effects
        s.visual = Visual.attach(script.Parent:FindFirstChild("BeaconVisual"), bead)
        bead:SetAttribute("OwnerId",player.UserId)
        bead:SetAttribute("Ready",false)
        local light = Instance.new("PointLight",bead)
        light.Color, light.Range, light.Brightness = bead.Color, 10, 2
        local ray = RaycastParams.new()
        ray.FilterType = Enum.RaycastFilterType.Exclude
        ray.FilterDescendantsInstances = {character,effects}
        ray.RespectCanCollide = true
        s.beacon = bead
        s.flight = {position=bead.Position, origin=bead.Position, velocity=direction*Config.ThrowSpeed+Vector3.new(0,Config.Lift,0), time=0, ray=ray}
        tell(player,s,"Beacon flying…")
    elseif action == "Teleport" then
        if not s.point then tell(player,s,"Throw onto a flat surface first"); return end
        if now >= s.expires then clear(s); tell(player,s,"Beacon expired"); return end
        if not s.support or not s.support.Parent or s.support.CFrame ~= s.supportFrame then
            clear(s); tell(player,s,"Landing surface moved"); return
        end
        local target, reason = Destination.find(character,root,s.point,s.support,effects,Config)
        if not target then tell(player,s,reason); return end
        character:PivotTo(target)
        root.AssemblyLinearVelocity, root.AssemblyAngularVelocity = Vector3.zero, Vector3.zero
        clear(s)
        s.cooldown = workspace:GetServerTimeNow()+Config.Cooldown
        tell(player,s,"Teleported! Cooldown " .. Config.Cooldown .. "s")
    end
end)
RunService.Heartbeat:Connect(function(dt)
    for player,s in pairs(states) do
        local f = s.flight
        if f then
            -- Bounded substeps; a long stall expires the projectile instead of skipping walls.
            if dt > 0.3 then clear(s); tell(player,s,"Throw interrupted; try again"); continue end
            local steps = math.max(1,math.ceil(dt/(1/60)))
            local step = dt/steps
            for _ = 1,steps do
                f.time += step
                local gravity = Vector3.new(0,-workspace.Gravity,0)
                local delta = f.velocity*step + gravity*(0.5*step*step)
                local hit = workspace:Raycast(f.position,delta,f.ray)
                if hit then
                    local surface = hit.Instance
                    if hit.Normal.Y >= Config.MinNormalY and surface:IsA("BasePart") and surface.Anchored
                        and not surface:FindFirstAncestorOfClass("Tool") then
                        s.point,s.support,s.supportFrame = hit.Position,surface,surface.CFrame
                        s.expires = os.clock()+Config.Lifetime
                        s.expiryTime = workspace:GetServerTimeNow()+Config.Lifetime
                        s.beacon:SetAttribute("ExpiresAt",s.expiryTime)
                        s.beacon:SetAttribute("Lifetime",Config.Lifetime)
                        s.flight = nil
                        Visual.move(s.beacon,s.visual,hit.Position+Vector3.new(0,0.45,0))
                        s.beacon.Color = Color3.fromRGB(60,240,190)
                        s.beacon.PointLight.Color = s.beacon.Color
                        tell(player,s,"Ready — E or click to teleport; R to replace")
                    else clear(s); tell(player,s,"Aim at a flat, anchored platform") end
                    break
                end
                f.position += delta
                f.velocity += gravity*step
                if f.time > Config.MaxFlight or (f.position-f.origin).Magnitude > Config.MaxRange then
                    clear(s); tell(player,s,"Beacon missed; throw again"); break
                end
                Visual.move(s.beacon,s.visual,f.position)
            end
        elseif s.point then
            if os.clock() >= s.expires then
                clear(s); tell(player,s,"Beacon expired; throw again")
            elseif os.clock() >= (s.nextCheck or 0) then
                s.nextCheck = os.clock()+0.25
                local c, _, r = characterFor(player,s)
                local target, reason
                if c and s.support and s.support.Parent and s.support.CFrame == s.supportFrame then
                    target, reason = Destination.find(c,r,s.point,s.support,effects,Config)
                else reason = "Equip Beacon; landing surface must stay still" end
                s.canTeleport = target ~= nil
                s.beacon:SetAttribute("Ready",s.canTeleport)
                s.beacon:SetAttribute("Reason",reason or "Ready")
                tell(player,s,target and "Ready — E or click to teleport; R to replace" or reason)
            end
        end
    end
end)
Players.PlayerAdded:Connect(setup)
Players.PlayerRemoving:Connect(function(player)
    local s = states[player]
    if not s then return end
    clear(s)
    if s.death then s.death:Disconnect() end
    for _,connection in ipairs(s.connections) do connection:Disconnect() end
    s.tool:Destroy()
    states[player] = nil
end)
for _,player in ipairs(Players:GetPlayers()) do setup(player) end
