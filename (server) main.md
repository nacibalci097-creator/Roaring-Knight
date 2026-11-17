Main server
```luau
local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local iscolliding_module = require(game.ReplicatedStorage.Modules.IsColliding)

local events = game.ReplicatedStorage.Events

local targets = workspace.Targets:GetChildren()
local cooldowns = {}

game.Players.PlayerAdded:Connect(function(player)
	cooldowns[player] = {}
end)

events.Ability.OnServerEvent:Connect(function(player, ability : string)
	if not abilityinfo_module[ability] then return end
	if cooldowns[player][ability] then return end
	
	cooldowns[player][ability] = true
	
	task.delay(abilityinfo_module[ability].Cooldown, function()
		cooldowns[player][ability] = nil
	end)

	require(game.ServerScriptService.HitboxModules[ability])(player, ability, targets)
end)

events.AbilityFunction.OnServerInvoke = function(player : Player, ability : string)	
	if not abilityinfo_module[ability] then return end
	if cooldowns[player][ability] then return end

	cooldowns[player][ability] = true

	task.delay(abilityinfo_module[ability].Cooldown, function()
		cooldowns[player][ability] = nil
	end)

	local result = require(game.ServerScriptService.HitboxModules[ability])(player, ability, targets)

	return result
end
```

# Tool Modules:

Slash
```luau
local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local iscolliding_module = require(game.ReplicatedStorage.Modules.IsColliding)
local damage_module = require(game.ServerScriptService.Modules.Damage)
local events = game.ReplicatedStorage.Events

return function(player : Player, ability : string, targets : {any})
	local hitbox : Part = game.ReplicatedStorage.Hitboxes[ability]:Clone()
	local char = player.Character or player.CharacterAdded:Wait()

	hitbox.CFrame = char:GetPivot() * (abilityinfo_module[ability].HitboxOffset or CFrame.new())

	for _, model : Model in targets do
		if iscolliding_module(model:GetPivot().Position, hitbox) then
			damage_module(model, ability)
		end
	end

	hitbox:Destroy()
end
```

Star
```luau
local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local damage_module = require(game.ServerScriptService.Modules.Damage)

local function target(center: Vector3, radius: number)
	local x = center.X
	local y = center.Y
	local z = center.Z

	return Vector3.new(math.random(x - radius, x + radius), math.random(y, y + 1), math.random(z - radius, z + radius))
end

return function(player : Player, ability : string, targets : {any})
	local char = player.Character or player.CharacterAdded:Wait()
	local stars = {}
	
	for i = 1, abilityinfo_module.Star.Amount do
		local pos = target((char:GetPivot() * CFrame.new(0,2, -30)).Position, 15)

		stars[i] = pos

		task.delay(2 + 0.1 * (i - 1), function()
			for i, model : Model in targets do
				if (model:GetPivot().Position - pos).Magnitude <= 5 then
					damage_module(model, ability)
				end
			end
		end)
	end
	
	return stars
end
```

Swoon
```luau
local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local iscolliding_module = require(game.ReplicatedStorage.Modules.IsColliding)
local damage_module = require(game.ServerScriptService.Modules.Damage)

return function(player : Player, ability : string, targets : {any})
	local hitbox : Part = game.ReplicatedStorage.Hitboxes[ability]:Clone()
	local char = player.Character or player.CharacterAdded:Wait()

	hitbox.CFrame = char:GetPivot() * (abilityinfo_module[ability].HitboxOffset or CFrame.new())

	for _, model : Model in targets do
		if iscolliding_module(model:GetPivot().Position, hitbox) then
			damage_module(model, ability)
		end
	end

	hitbox:Destroy()
end
 ```

Sword Aim
```luau
local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local damage_module = require(game.ServerScriptService.Modules.Damage)

return function(player : Player, ability : string, targets : {any})
	local info = {}
	local models = {}
	
	local newtargets = {}
	
	for i, v : Model in targets do
		if v.Humanoid.Health > 0 then
			table.insert(newtargets, v)
		end
	end
	
	if #newtargets <= 0 then
		return {}
	end
	
	for i = 1, abilityinfo_module.SwordAim.Amount do
		local target : Model = newtargets[math.random(1, #newtargets)]
		local pos = target:GetPivot().Position

		table.insert(info, pos)
		table.insert(models, target)
	end
	
	task.defer(function()
		for i, pos : Vector3 in info do
			local cooldown = 1 - i * 0.1
			local target : Model = models[i]
			
			task.delay(1, function()
				if (pos - target:GetPivot().Position).Magnitude <= 1 then
					damage_module(target, ability)
				end
			end)
			
			task.wait(math.max(0.2, cooldown))
		end
	end)

	return info
end
```

Tunnel
```luau
local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local damage_module = require(game.ServerScriptService.Modules.Damage)

return function(player : Player, ability : string, targets : {any})	
	local swordpositions = {}

	local ignored = {}

	for i = 1, abilityinfo_module["Tunnel"].Amount do
		local even = i % 2 == 0
		local rng = math.random(-3,3)
		local offset = CFrame.new(even and 5 or -5, 3, 15) * CFrame.new(rng, 0, 0)
		local rotation = CFrame.Angles(math.rad(0), math.rad(0), even and math.rad(90) or math.rad(-90))

		swordpositions[i] = {
			Offset = offset,
			Rotation = rotation
		}
	end

	task.defer(function()
		for i, info in swordpositions do
			for ii = 1, 2 do
				local clone = game.ReplicatedStorage.Other.Blade:Clone()
				local char = player.Character or player.CharacterAdded:Wait()
				local target = char:GetPivot() * info.Offset * info.Rotation

				clone.CFrame = target

				task.defer(function()
					for i = 1, 400 do
						clone.CFrame *= CFrame.new(0,0, i <= 50 and -0.2 or -0.5)

						for _, model : Model in targets do
							if (model:GetPivot().Position - clone.Position).Magnitude <= 4 and not ignored[model] then
								ignored[model] = true

								task.delay(0.1, function()
									ignored[model] = nil
								end)
								
								damage_module(model, ability)
							end
						end

						task.wait()
					end
					clone:Destroy()
				end)
			end
			task.wait(0.025)
		end
	end)

	return swordpositions
end
```
