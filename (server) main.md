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
