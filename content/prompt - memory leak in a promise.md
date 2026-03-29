Let's make an overview on the attack abilities, all the abilities inherits methods from an abstract class
```lua
function Basic.new(config)
	local self = setmetatable(AttackBaseAbility.new(config), Basic)

	return self
end
````
The they have an execute method that is called by the invoker of the command pattern, this method usually looks like this:
```lua
function Basic:Execute()
	local CHAR = self.character
	if CHAR:GetAttribute("Attacking") or CHAR:GetAttribute("Swing") then
		return
	end

	if self:GetCD() then
		return
	end
	-- start attack
	self:StartCD()
	--Settin attr
	self:SetAttributes()

	local detectionInfo = {}-- detectionInfo holds information about the source for the hitbox
	self:_countAttack()
	local hitFx = function()
		-- Use client passes the data needed to create the hitbox in the client, then the client fires back the hit targets
		self:CreateScopedPromise({ Attacker = self.character, hitboxes = { detectionInfo } })
			:Then(function(hitTarget, hitPart)
				hitTouchTemplate(data, hitTarget, hitPart)
			end)
	end
	animHandler:OnSignal({
		callback = hitFx,
		animationName = swingName,
		animationEventName = "Hit",
	})
end
```
I want to focues in the hitFx, this functionallity is executed in the Hit event, and its usually the most complex snippet of the code, which have the logic. Let's carefully analyze this pattern. :CreateScopedPromise() is a method inherited from AttackBaseAbility, and it's the core logic to trigger the client side hit detection behavior, core in this combat system
```lua
-- AttackBaseAbility
function AttackBaseAbility:CreateScopedPromise(detectionInfo)
	local scopeMaid = Maid.new()

	local PromiseTask = Promise.new(function(resolve, reject)
		useClient(resolve, reject, scopeMaid, detectionInfo)
	end)

	return PromiseTask
end
```
This utility does create a scoped maid, so the connections created during tha attack are cleaning, when the are cleaning? in this module utillity called useClient
```lua
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Events = ReplicatedStorage.Events
local HitboxResponseRemote = Events.HitboxResponseRemote

return function(resolve, reject, scopeMaid, detectionInfo)
	-- Listen to client on hit parts
	local character = detectionInfo.Attacker
	local attacker_player = Players:GetPlayerFromCharacter(character)

	local connection = HitboxResponseRemote.OnServerEvent:Connect(function(player, data)
		local attackID, hitTargets, hitPart = data.attackID, data.character, data.hitPart
		if player == attacker_player and attackID == detectionInfo.AttackID then
			resolve(hitTargets, hitPart) -- Promise resolves with hit targets
			-- targets
		end
	end)

	-- Add connection to scoped maid so it cleans up
	scopeMaid:GiveTask(connection)

	-- Fire to client to start detection
	HitboxResponseRemote:FireClient(attacker_player, detectionInfo)
end
```
In summary it uses the scoped maid for a server event connection and do fire the client with the detection info, the client creates the hitbox using detectionInfo table and fire backs on `HitboxResponseRemote.OnServerEvent`, that does resolve and call the function scoped to `Then`, `hitTouchTemplate`. But i want to focus in this pattern first because there's a bottle neck effect in the game, the hitboexes damages does nothing, is like if the damage is not counting, but it's actually stacking, and all the hitboxes accumulated does the damage at once. I'm not a experienced developer but i have seen memory leaks and when a connection is not disconnected does occur this bottle neck effect. i think such behavior is a memory leak, i'm not quite sure what's happening but you know the Promises library better than me, what is a promise? i think they are like a connection than yields until we resolve, so my hypothesis is that the promises connections are stacking live, and yield all the other promises from the hitboxes causing the bottle effect (memory leak). As you know the Promise better than me i want you to define me what is a Promise, you can include snippet of the quenty's library to confirm or deny my hypothesis, then we can discuss possible solutions as include some kind of timeout for the promise to cancel

### Fix

```lua
function AttackBaseAbility:CreateScopedPromise(detectionInfo)
	local scopeMaid = Maid.new()

	local PromiseTask = Promise.new(function(resolve, reject)
		useClient(resolve, reject, scopeMaid, detectionInfo)
	end):Finally(function()
		scopeMaid:DoCleaning()
	end)

	return PromiseTask
end
```

### Quick overview
I want you to stop for a moment and have a look to my weapon class, it's like a context it's using _weapon as the current strategy:

```
--Services
-- The weapon context strategy
-- local SoundService = game:GetService("SoundService")
-- local SoundFolder = SoundService.SFX
-- local WeaponSoundFolder = SoundFolder.Weapons
local Debris = game:GetService("Debris")
local require = require(script.Parent.loader).load(script)
local EffectService = require("EffectService")
local MovementLock = require("MovementLock")

local BaseObject = require("BaseObject")
local Weapon = {}
Weapon.__index = Weapon

local Strategies = {
	batter = require("Batter"),
	shark = require("Shark"),
}

function Weapon.new(char: Model, serviceBag)
	local self = setmetatable(BaseObject.new(char), Weapon)
	self.Char = char
	self._weapon = nil
	self._serviceBag = serviceBag
	self.Humanoid = char:WaitForChild("Humanoid")

	self._character_state = MovementLock.new(char, serviceBag)
	self._defenseChain = {}

	if self.Humanoid.RigType == Enum.HumanoidRigType.R15 then
		self.armName = "RightLowerArm"
		self.torso = self.Char:FindFirstChild("UpperTorso")
	else
		self.armName = "Right Arm"
		self.torso = self.Char:FindFirstChild("Torso")
	end
	return self
end

function Weapon:Execute(action, params)
	self._weapon:Execute(action, params)
end
function Weapon:TakeHit(context)
	context.anchorState = self._character_state
	-- Must have a global damage counter for checking if player can transform
	EffectService:Apply(context)
end

function Weapon:increaseDmgCount(increase)
	assert(typeof(increase) == "number", "increase value must be a number")
	self._weapon:_increaseDmgCount(increase)
end

function Weapon:OnIncomingAttack(context)
	-- Context should be modified through the chain, and the last in the change is TakeHit
	task.defer(function()
		self:OnDefend(context)
	end)
end

function Weapon:OnDefend(context)
	-- Chain of Responsibility pattern
	local modifiedContext = context
	-- For the moment, the hit part always will be a torso
	context.HitPart = self.torso

	for _, description in ipairs(self._defenseChain) do
		local modifier = description.modifier
		if not modifier then
			return
		end
		print("description.modifier ", description.modifier)
		modifiedContext = modifier:Process(modifiedContext)
		print("modifiedContext ", modifiedContext)
		-- If a modifier blocks completely, stop the chain
		if modifiedContext.blocked then
			return
		end
	end
	self:TakeHit(modifiedContext)
end

function Weapon:SetActiveDefense(class)
	table.insert(self._defenseChain, {
		modifier = class,
		priority = class.priority,
		blockType = class,
	})

	table.sort(self._defenseChain, function(a, b)
		return a.priority < b.priority
	end)
end

function Weapon:ClearActiveDefense(priority)
	for index, desc in ipairs(self._defenseChain) do
		if desc.priority == priority then
			table.remove(self._defenseChain, index)
		end
	end
end

function Weapon:OnDefended()
	self._weapon:OnDefended()
end

function Weapon:EquiTool(toolName)
	self.tool_name = toolName
	if not Strategies[toolName] then
		warn("Weapon not found")
		return
	end
	self._weapon = Strategies[toolName].new(self.Char, self._serviceBag)
	local arm = self.Char:FindFirstChild(self.armName)
	if not arm then
		warn(string.format("The %s should has a %s", self.Char.Name, self.armName))
		return
	end

	-- local M6D = Instance.new("Motor6D")
	-- M6D.Name = "ToolGrip"
	-- M6D.Parent = arm

	-- arm.ToolGrip.Part0 = arm
	-- arm.ToolGrip.Part1 = self.Char[toolName].BodyAttach
end

function Weapon:UnequiTool()
	if not self.tool then
		warn("No tool equiped")
		return
	end
	self.tool.BodyAttach.Trail.Enabled = false

	local arm = self.Char:FindFirstChild(self.armName)
	arm:FindFirstChild("ToolGrip"):Destroy()

	local UnEquipSound = self.Folder:WaitForChild("UnEquip"):Clone()
	UnEquipSound.Parent = self.Char:WaitForChild("HumanoidRootPart")
	UnEquipSound:Play()
	Debris:AddItem(UnEquipSound, 1)
	self._weapon:UnEquip()
end

function Weapon:GetAction(action)
	return self._weapon:GetAction(action)
end

function Weapon:registryTool(toolName)
	-- local f = WeaponSoundFolder:FindFirstChild(toolName)
	self.tool_name = toolName
	-- if not f then
	-- 	warn("Sound folder not found")
	-- 	return
	-- end

	-- self.Folder = f
end

-- If the sound thing causes issue we move it to a facade
function Weapon:PlaySound(lifeTime: number?, nameSound: string)
	local dur = lifeTime or 2
	local Sound = self.Folder[nameSound]:Clone()
	Sound.Parent = self.Char:WaitForChild("HumanoidRootPart")
	Sound:Play()
	Debris:AddItem(Sound, dur)
end

function Weapon:PlaySwingSound(lifeTime: number?)
	print("Do you meant to use PlaySound ", debug.traceback())
	local dur = lifeTime or 1
	local SwordSwingSound = self.Folder:WaitForChild("Swing"):Clone()
	SwordSwingSound.Parent = self.Char:WaitForChild("HumanoidRootPart")
	SwordSwingSound:Play()
	Debris:AddItem(SwordSwingSound, dur)
end

function Weapon:PlayHeavySound(lifeTime: number?)
	print("Do you meant to use PlaySound ", debug.traceback())
	return
	--[[
		local dur = lifeTime or 4
		local HeavySound = WeaponSoundFolder:WaitForChild(self.tool_name):WaitForChild("Heavy"):Clone() -- the sound is very important
		HeavySound.Parent = self.Char.HumanoidRootPart
		HeavySound:Play()
		Debris:AddItem(HeavySound, dur)
	]]
end

return Weapon
```

Notice the onDefend, that's the final method called in hitTouchTemplate and decide the target to take its damage or not, this will be important for the incoming prompt

---


That's right, but Execute structure will be change, and OnDefend method may change as well
### Creates multiple server connections is wrong
The promises connections are successfully cleaned, but this error is being throwed:

```
Remote event invocation queue exhausted for [ReplicatedStorage.Events](http://ReplicatedStorage.Events).HitboxResponseRemote; did you forget to implement OnServerEvent?
```
And it makes perfect sense, after all `OnServerEvent` is not thinked to be connected and disconnected, they are usually a single connection in the server, stateless, that last all the game life time.
We need to turn this:
```lua
local connection = HitboxResponseRemote.OnServerEvent:Connect(function(player, data)
		local attackID, hitTargets, hitPart = data.attackID, data.character, data.hitPart
		if player == attacker_player and attackID == detectionInfo.AttackID then
			resolve(hitTargets, hitPart) -- Promise resolves with hit targets
			-- targets
		end
	end)

	-- Add connection to scoped maid so it cleans up
	scopeMaid:GiveTask(connection)
```
Into something like this (is just a representation):
```lua
HitboxResponseRemote.OnServerEvent:Connect(function(player, data)
	local BinderProvider = self._serviceBag:GetService(_G.BinderProvider)
	local WeaponBinder = BinderProvider:Get("Armed")
	local Binder = WeaponBinder:Get(self.character)
	Binder:ResolveCurrentAttack()
end
```
I like the current desing of the attacks, we are not using `RemoteFunction`, we are creating different coroutines that do not yield Scripts invoking a RemoteFunction yielding until they receive a response. Instead every Attack:Execute is describing intent in a promise, this fires an event to the client and the state is pending until it's resolves from the frontend, which creates the hitbox and detect who's hit, firing the server back. In the current approach thatcomunication is created at the same attack coroutine, connecting and disconnecting the `HitboxResponseRemote.OnServerEvent` several times, causing errors. As you can see the idea is good, but the attack need to expose the coroutine, if the corutine is public it can be resolved via the weapon binder (the class that holds all the abilities) in a stateless `HitboxResponseRemote.OnServerEvent`. Fortunately the Attack:Execute already creates and id, so this promises may be registered. Help me to refactor the weapon binder, registering the promises and resolving them from 
