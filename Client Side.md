# Main local scripts

Tools
```luau
local player = game.Players.LocalPlayer
local tools = player.Backpack:GetChildren()
local abilities = game.ReplicatedStorage.AbilityModules

for _, tool : Tool in tools do
	require(abilities[tool.Name])(tool, player)
end
```

Effects
```luau
repeat task.wait() until game:IsLoaded()

local tweenservice = game:GetService("TweenService")
local tinfo = TweenInfo.new(1, Enum.EasingStyle.Linear)

local player = game.Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()

local parts = {}

local function renew(char : Model)
	parts = {}

	for i, v in char:GetDescendants() do
		if v:IsA("MeshPart") or v:IsA("Part") then
			table.insert(parts, v)
		end
	end
end

renew(char)

player.CharacterAdded:Connect(renew)

for i, model : Model in workspace.Targets:GetChildren() do
	local humanoid : Humanoid = model.Humanoid
	local oldhealth = humanoid.Health
	humanoid:GetPropertyChangedSignal("Health"):Connect(function()		
		local clone = game.ReplicatedStorage.Other.Gui:Clone()
		clone.Gui.TextLabel.Text = "-" .. oldhealth - humanoid.Health
		clone.Position = model:GetPivot().Position
		clone.Parent = workspace.Effects
		
		game.SoundService.Hit:Play()
		oldhealth = humanoid.Health
		
		local tween = tweenservice:Create(clone, TweenInfo.new(0.5), {Position = clone.Position + Vector3.new(math.random(-3,3), math.random(5,10), math.random(-3,3))})
		tween:Play()
		tween.Completed:Wait()
		
		tweenservice:Create(clone.Gui.TextLabel.UIStroke, TweenInfo.new(1), {Transparency = 1}):Play()
		tween = tweenservice:Create(clone.Gui.TextLabel, TweenInfo.new(1), {TextTransparency = 1})
		
		tween:Play()
		tween.Completed:Wait()
		clone:Destroy()
	end)
end

while task.wait(1 / 12) do
	local model = Instance.new("Model")
	
	for i, v : Part in parts do -- directly cloning the model has cframe issues for some reason????
		local clone = v:Clone()
		
		if clone.Name == "HumanoidRootPart" then model.PrimaryPart = clone end
		
		for _, vv in clone:GetDescendants() do
			if vv:IsA("Attachment") or vv:IsA("Motor6D") or vv:IsA("Decal") then
				vv:Destroy()
			end
		end
		
		clone.Material = Enum.Material.SmoothPlastic
		clone.CFrame = v.CFrame
		clone.CanCollide = false
		clone.CastShadow = false
		clone.Anchored = true
		
		clone.Parent = model
	end
	
	model.Parent = workspace.Effects

	task.defer(function()
		for i, v in model:GetChildren() do
			tweenservice:Create(v, tinfo, {Transparency = 1}):Play()
		end
		
		for i = 1, 200 do
			model:PivotTo(model:GetPivot() * CFrame.new(0, 0, 0.02))
			task.wait()
		end
		
		model:Destroy()
	end)
end
```

# Tools
Slash
```luau
local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local events = game.ReplicatedStorage.Events

local animation = game.ReplicatedStorage.Animations.Slash
local oncooldown = false

return function(tool : Tool, player : Player)
	tool.Activated:Connect(function()
		if oncooldown then return end
		oncooldown = true

		local char = player.Character or player.CharacterAdded:Wait()

		local track : AnimationTrack = char.Humanoid.Animator:LoadAnimation(animation)
		track:Play()
		track:AdjustSpeed(1.25)

		local conn = track:GetMarkerReachedSignal("Attack"):Connect(function()
			events.Ability:FireServer(tool.Name)
			game.SoundService.Slash:Play()
		end)

		task.delay(abilityinfo_module[tool.Name].Cooldown, function()
			oncooldown = false
			conn:Disconnect()
		end)
	end)
end
```

Star
```luau
local tweenservice = game:GetService("TweenService")

local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local events = game.ReplicatedStorage.Events

local animation = game.ReplicatedStorage.Animations.Star
local oncooldown = false

return function(tool : Tool, player : Player)
	local gui = player.PlayerGui:WaitForChild("ScreenGui")

	tool.Activated:Connect(function()
		if oncooldown then return end
		oncooldown = true

		local char = player.Character or player.CharacterAdded:Wait()

		local track : AnimationTrack = char.Humanoid.Animator:LoadAnimation(animation)
		track:Play()

		game.SoundService.Star:Play()
		task.wait(1)

		task.delay(abilityinfo_module[tool.Name].Cooldown, function()
			oncooldown = false
		end)

		local startpositions : {any} = events.AbilityFunction:InvokeServer(tool.Name)
		local finished = 0

		char.Torso.Anchored = true

		for i, targetpos in startpositions do
			local clone = game.ReplicatedStorage.Other.Star:Clone()
			clone.Size = clone.Size * 0.1
			clone.Position = char["Left Arm"].Position
			clone.Parent = workspace.Effects

			task.defer(function()
				tweenservice:Create(clone, TweenInfo.new(4), {Color = Color3.fromRGB(255)}):Play()
				local tween = tweenservice:Create(clone, TweenInfo.new(2), {Size = clone.Size * 10, Position = targetpos})
				tween:Play()
				tween.Completed:Wait()

				local part = Instance.new("Part")
				part.Size = Vector3.new(0.1, 0.1, 0.1)
				part.Position = clone.Position
				part.Color = Color3.fromRGB(255,255,255)
				part.Material = "Neon"
				part.Shape = Enum.PartType.Ball
				part.CanCollide = false
				part.CastShadow = false
				part.Anchored = true
				part.Parent = workspace.Effects

				tweenservice:Create(part, TweenInfo.new(0.2), {Size = Vector3.new(7,7,7)}):Play()

				local tween = tweenservice:Create(clone, TweenInfo.new(0.2), {Transparency = 1})
				tween:Play()
				tween.Completed:Wait()
				clone:Destroy()

				game.SoundService.StarExplode:Play()

				local tween = tweenservice:Create(part, TweenInfo.new(0.2), {Transparency = 1})
				tween:Play()
				tween.Completed:Wait()
				part:Destroy()

				finished += 1

				if finished >= #startpositions then
					track.Looped = false
					track:Stop()
					char.Torso.Anchored = false
				end
			end)	

			task.wait(0.1)
		end
	end)
end
```

Swoon
```luau
local tweenservice = game:GetService("TweenService")

local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local events = game.ReplicatedStorage.Events

local animation = game.ReplicatedStorage.Animations.Slash
local oncooldown = false

return function(tool : Tool, player : Player)
	local gui = player.PlayerGui:WaitForChild("ScreenGui")

	tool.Activated:Connect(function()
		if oncooldown then return end
		oncooldown = true

		local char = player.Character or player.CharacterAdded:Wait()

		local track : AnimationTrack = char.Humanoid.Animator:LoadAnimation(animation)
		track:Play()
		track:AdjustSpeed(1.5)

		local conn = track:GetMarkerReachedSignal("Attack"):Connect(function()
			events.Ability:FireServer(tool.Name)
			game.SoundService.Swoon:Play()

			gui.Swoon.BackgroundTransparency = 0
			gui.Swoon.ImageTransparency = 0

			tweenservice:Create(gui.Swoon, TweenInfo.new(2), {BackgroundTransparency = 1, ImageTransparency = 1}):Play()
		end)

		task.delay(abilityinfo_module[tool.Name].Cooldown, function()
			oncooldown = false
			conn:Disconnect()
		end)
	end)
end
```

Sword Aim
```luau
local tweenservice = game:GetService("TweenService")

local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local events = game.ReplicatedStorage.Events

local oncooldown = false

return function(tool : Tool, player : Player)
	local gui = player.PlayerGui:WaitForChild("ScreenGui")

	tool.Activated:Connect(function()
		if oncooldown then return end
		oncooldown = true

		local char = player.Character or player.CharacterAdded:Wait()

		task.delay(abilityinfo_module[tool.Name].Cooldown, function()
			oncooldown = false
		end)
		
		local positions : {any} = events.AbilityFunction:InvokeServer(tool.Name)

		for i, target : Vector3 in positions do
			task.defer(function()
				local clone = game.ReplicatedStorage.Other.Blade:Clone()

				clone.CFrame = char:GetPivot() * CFrame.new(0, math.random(6,10), math.random(-6, 6))
				clone.CFrame = CFrame.new(clone.Position, target) * CFrame.Angles(0, math.rad(-90), math.rad(90))
				clone.Parent = workspace.Effects
				game.SoundService.SwordSpawn:Play()
				tweenservice:Create(clone, TweenInfo.new(0.5), {Transparency = 0}):Play()

				task.wait(0.5)

				local tween = tweenservice:Create(clone, TweenInfo.new(0.5, Enum.EasingStyle.Linear), {Color = Color3.fromRGB(255)})
				tween:Play()
				tween.Completed:Wait()

				game.SoundService.SwordTarget:Play()

				tween = tweenservice:Create(clone, TweenInfo.new(0.05, Enum.EasingStyle.Linear), {Position = target})
				tween:Play()
				tween.Completed:Wait()
				clone:Destroy()

			end)

			local cooldown = 1 - i * 0.1

			task.wait(math.max(0.2, cooldown))
		end
	end)
end
```

Tunnel
```luau
local tweenservice = game:GetService("TweenService")

local abilityinfo_module = require(game.ReplicatedStorage.Modules.AbilityInfo)
local events = game.ReplicatedStorage.Events

local animation = game.ReplicatedStorage.Animations.Star
local oncooldown = false

return function(tool : Tool, player : Player)
	local gui = player.PlayerGui:WaitForChild("ScreenGui")

	tool.Activated:Connect(function()
		if oncooldown then return end
		oncooldown = true

		local char = player.Character or player.CharacterAdded:Wait()

		local track : AnimationTrack = char.Humanoid.Animator:LoadAnimation(animation)
		track:Play()

		task.wait(1)

		task.delay(abilityinfo_module[tool.Name].Cooldown, function()
			oncooldown = false
		end)

		local swordpositions = events.AbilityFunction:InvokeServer(tool.Name)

		char.Torso.Anchored = true
		
		for i, info in swordpositions do
			local clone = game.ReplicatedStorage.Other.Blade:Clone()
			local offset = info.Offset
			local rotation = info.Rotation
			local target = char:GetPivot() * offset * rotation

			clone.CFrame = target
			clone.Parent = workspace.Effects

			for i, v in clone:GetChildren() do
				if v.Name == "Hitbox" or v:IsA("Highlight") then continue end
				tweenservice:Create(v, TweenInfo.new(0.25), {Transparency = 0}):Play()
			end

			tweenservice:Create(clone, TweenInfo.new(0.25), {Transparency = 0}):Play()

			task.defer(function()
				for i = 1, 400 do
					clone.CFrame *= CFrame.new(0,0, i <= 50 and -0.2 or -0.5)
					task.wait()
				end

				clone:Destroy()
			end)
			
			task.wait(0.025)
		end

		char.Torso.Anchored = false
		track.Looped = false
		track:Stop()
	end)
end
```
