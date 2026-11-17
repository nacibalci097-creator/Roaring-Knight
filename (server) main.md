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
