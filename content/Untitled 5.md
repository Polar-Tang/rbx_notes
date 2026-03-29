Okey, i fall down in the cyclic module dependency. 
```
  WeaponBinder
	/	   \
abilities  SomeWeapon
	\       /	
	constants
```
I think this problem occurs to most of the binder, that's why quenty emphasizes how to create a service. I'm not sure where i should start to migrate the module to service bag so let's start by the binder. I think ever file in the service bag requiring any must do it through the loader, but i'm not sure about it. 
The weapon binder goes like:
```lua
local special_require = require(script.Parent.loader).load(script)
local Binder = special_require("Binder")
```
We construct an object that later will be used by the binder provider
```lua
local require = require(script.Parent.loader).load(script)
-- local Binder = require("Binder")
local WeaponController = require("WeaponController")

local Weapon = {}
Weapon.ServiceName = "Weapon"
  
function Weapon.new(char: Model, serviceBag)
local self = setmetatable(WeaponController(char, serviceBag), Weapon)
self._serviceBag = assert(serviceBag, "No serviceBag")
return self
end
  
-- something like this should be done in the binder provider
-- https://quenty.github.io/NevermoreEngine/docs/servicebag#dependency-injection-in-binders
-- Binder.new("Armed", require(Weapon), serviceBag)
return Weapon
```
Now here's the weapon utilizing the controller:
```lua
function WeaponController.new(char: Model)
	local self = setmetatable(BaseObject.new(char), WeaponController)
	self.Char = char
	self._weapon = nil
	  
	return self
end
```
The idea of the controller is to be the context for the strategies, however i don't have seen any quenty's services using a strategy (there could be though). Any way, EquiTool is the method to set a strategy
```lua
local require = require(script.Parent.loader).load(script)
local Strategies = {
	Fist = require("Fist"),
}

function WeaponController:EquiTool(toolName)
	self._weapon = Strategies[toolName].new(self.Char)
end
```
The weapons are using a weapon absctract class that has .new and :`_createAbilities` method, and here comes the trouble because this method is overwrite by the custom weapon, for example Fist:
```lua
function Fist:_createAbilities()
	self.attacks =
		AbilityBuilder.new(self.char)
		:Add(Enum.UserInputType.MouseButton1, "M1", {
```
Here the weapon use a builder and i'm requiring something that could be a tottally new service
```lua
constants.ALL_ABILITIES = {
	["M1"] = require(skills.Basic),
	["Sprint"] = require(skills.Sprint),
	["Heavy"] = require(skills.Heavy),
	["Blocking"] = require(skills.Blocking),
}

function AbilityBuilder.new(char)
	return setmetatable({
		char = char,
		_map = {},
	}, AbilityBuilder)
end
  
function AbilityBuilder:Add(inputKey, abilityName, config)
	local AbilityClass = ALL_ABILITIES[abilityName]
	-- Get the cooldown controller from service bag to every ability
	config._cooldownController = CooldownController.new(self.char)
	self._map[inputKey] = AbilityClass.new(config)
	return self
end
```
Now this very funny, because later i realized that i didn't require the service bag after started, it could be globally avaible, so iremove the popDrill, but now i'm doing all of this and it must be prepared to all stages, so i should pop dril again
- Still need to pass animation handler to a binder else in the service bag
- Adapt every attack for requiring server thing

### Using game:getService in the service bag
This is an example about how to use the game service in the sercie bag, just don't use require 
```lua
--[=[

Makes the character look at nearby physical buttons

@class LookAtButtonsClient

]=]

  

local require = require(script.Parent.loader).load(script)

  

local Players = game:GetService("Players")

local RunService = game:GetService("RunService")

  

local AdorneeUtils = require("AdorneeUtils")

local BaseObject = require("BaseObject")

local CharacterUtils = require("CharacterUtils")

local GameBindersClient = require("GameBindersClient")

local IKAimPositionPriorites = require("IKAimPositionPriorites")

local IKServiceClient = require("IKServiceClient")

local Maid = require("Maid")

local Octree = require("Octree")

local ServiceBag = require("ServiceBag")

  

local LOOK_NEAR_DISTANCE = 15

  

local LookAtButtonsClient = setmetatable({}, BaseObject)

LookAtButtonsClient.ClassName = "LookAtButtonsClient"

LookAtButtonsClient.__index = LookAtButtonsClient

  

function LookAtButtonsClient.new(humanoid, serviceBag: ServiceBag.ServiceBag)

local self = setmetatable(BaseObject.new(humanoid), LookAtButtonsClient)

  

self._serviceBag = assert(serviceBag, "No serviceBag")

self._gameBindersClient = self._serviceBag:GetService(GameBindersClient)

self._ikServiceClient = self._serviceBag:GetService(IKServiceClient)

  

if CharacterUtils.getPlayerFromCharacter(humanoid) == Players.LocalPlayer then

self:_setupLocal()

end

  

return self

end

  

function LookAtButtonsClient:_setupLocal()

self._octree = Octree.new()

  

self._maid:GiveTask(self._gameBindersClient.PhysicalButton:GetClassAddedSignal():Connect(function(physicalButton)

self:_trackPhysicalButton(physicalButton)

end))

self._maid:GiveTask(self._gameBindersClient.PhysicalButton:GetClassRemovedSignal():Connect(function(physicalButton)

self:_stopTrackingPhysicalButton(physicalButton)

end))

  

for _, physicalButton in self._gameBindersClient.PhysicalButton:GetAll() do

self:_trackPhysicalButton(physicalButton)

end

  

self._maid:GiveTask(RunService.RenderStepped:Connect(function()

self:_lookAtNearByButton()

end))

end

  

function LookAtButtonsClient:_lookAtNearByButton()

local rootPart = self._obj.RootPart

if not rootPart then

return

end

  

local position = rootPart.Position

local nearestList = self._octree:KNearestNeighborsSearch(position, 1, LOOK_NEAR_DISTANCE)

local _, nearest = next(nearestList)

if nearest then

local adornee = nearest:GetAdornee()

local nearestPosition = AdorneeUtils.getCenter(adornee)

  

if nearestPosition then

self._ikServiceClient:SetAimPosition(nearestPosition, IKAimPositionPriorites.HIGH)

end

end

end

  

function LookAtButtonsClient:_trackPhysicalButton(class)

local maid = Maid.new()

  

local node

local function update()

local adornee = class:GetAdornee()

local position = AdorneeUtils.getCenter(adornee)

  

if position then

if not node then

node = self._octree:CreateNode(position, class)

end

else

if node then

node:Destroy()

node = nil

end

end

end

-- Obviously would be better to do this every like, 0.3 seconds or something, but we're lazy for this tech demo

maid:GiveTask(RunService.Heartbeat:Connect(update))

update()

  

maid:GiveTask(function()

if node then

node:Destroy()

node = nil

end

  

self._maid[class] = nil

end)

self._maid[class] = maid

end

  

function LookAtButtonsClient:_stopTrackingPhysicalButton(class)

self._maid[class] = nil

end

  

function LookAtButtonsClient:_updateLocal() end

  

return LookAtButtonsClient
````

### Initialized service bag only available in the server

So now i'm migrating the abilities to work comfortable with, also it will easier to require them globally, however i have a great misunderstanding about where place the modules, if `/src/Shared` or `/src/Server`, i was thinking that i'm initializing the service bag in Server, so it's global there, but for the npcs probably would require it from the replicatedStorage due to the performance. NeverMore is in RS, they do a tradeoff performance over security, so i'm considering two options
##### Move service bag to replicated storage
As i would require most of the binder services for a model in the workspace, does it make sense the binder provide would be there.
###### Share server by this way i think about
A simple solution could be to create a service bag init in the replicated storage, whenever it's called it store the service bag globally there:
```lua
-- /shared/serviceBg
serviceBg.init = function(ServiceBag)
	_G.serviceBag = _ServiceBag
end
```
Please choose one of these approaches considering [the nevermore documenation](https://quenty.github.io/NevermoreEngine/docs/servicebag/) 


`_ga_9SLXM9GQ7R=GS2.1.s1765911426$o16$g1$t1765911439$j47$l0$h0; _ga=GA1.1.992468134.1759064775; PHPSESSID=q1ust4nedhrbho9hfv8b4qjon3; SESSID=nodo7`
```sh
sudo docker run -v $(pwd)/wordlist:/wordlist/ -it ghcr.io/xmendez/wfuzz wfuzz -b "_ga_9SLXM9GQ7R=GS2.1.s1765911426$o16$g1$t1765911439$j47$l0$h0; _ga=GA1.1.992468134.1759064775; PHPSESSID=q1ust4nedhrbho9hfv8b4qjon3; SESSID=nodo7" -z range,50-70 -s 3 -t 1 -u "https://mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30257/a/INGRESO_2021014%5C_RESOLUCION--EXAMEN-SEMINARIO-10-hs-DICIEMBRE-2025--TEMAS-1-Y-2.pdf/id/73FUZZ"
````
````
clickEvent.OnServerEvent:Connect(function(player: Player, data: clickEvenetData)
	if not data or not data.clickPosition then
		return
	end

	-- player must have a character with humanoid
	local character = player.Character or player.CharacterAdded:Wait()
	local humanoid = character:WaitForChild("Humanoid")
	-- client can send stop thruty to cancel the operation
	if data.stop then
		humanoid:UnequipTools()
		return
	end

	-- check if the ball is equipped
	local equippedTool = getTool(player)
	if not equippedTool or equippedTool.Name ~= "MagicBall" then
		return
	end

	local HRP: BasePart = character.PrimaryPart
	-- This is like a plain binder
	local PlayerHeart = PlayerHearthRegistry:GetHandler(player)

	-- check isApeasse, avoid several tasks by several clicks
	if PlayerHeart.isAppease then
		warn("Must wait")
		return
	else
		PlayerHeart:Appease(2)
	end

	-- create a ball to trhow
	local pokeClone: Model = MagicBall:Clone()
	pokeClone.Parent = Workspace

	-- create a scoped Maid using time stamps as a keys
	local timeStamp = tick()
	pokeClone:SetAttribute("Stamp", timeStamp)
	local _, scopeMaid = PlayerHeart:CreateInteractionScope(15, timeStamp)

	-- the backend sign the ball
	local ballId = httpService:GenerateGUID()
	registry[ballId] = {
		ball = pokeClone,
		id = ballId,
	}
	-- collection of the ball
	Debris:AddItem(pokeClone, 15)
	task.delay(15, function()
		registry[ballId] = nil
	end)

	-- play throw animation
	local Animator: Animator = humanoid:WaitForChild("Animator")
	local animation = Instance.new("Animation")
	animation.AnimationId = "rbxassetid://90384503768373"
	local animationTack = Animator:LoadAnimation(animation)
	animationTack:Play()

	-- initialize the throw ball functionallity
	-- Create a Maid to manage all connections/tasks for this throw

	local markerConn
	markerConn = animationTack:GetMarkerReachedSignal("throw"):Connect(function()
		humanoid:UnequipTools()
		Throw:Play()

		--------------- variables ---------------
		local startPosition = HRP.Position
		local lastY = 0

		--------------- constants ---------------
		local T = 0.5 -- a fixed amount of time for the ball to hit the ground
		local BOUNCINESS = 0.4 -- elasticity of a collision
		local clickPosition = data.clickPosition -- P1 or position goal
		local GRAVITY = Vector3.new(0, -workspace.Gravity, 0)

		-- CALCULATE DIRECTION AND HOW FAST THE OBJECT MOVES
		--V1 = (P1 - P0 - 0.5*G*T^2)/ T
		local initialVelocity = (clickPosition - startPosition - 0.5 * GRAVITY * T ^ 2) / T
		local t = 0

		-- This function is ccall if a pet is caputred, it plays effects and creates a prompt connection
		local function capturePet(pet: Model, animDur: number?)
			--------------- variables ---------------
			local Dur = animDur or 2
			local petPrimaryPart = pet.PrimaryPart
			--------------- shine vfx ---------------
			local brightPart: Part = treasureVFX(petPrimaryPart, Dur)
			Debris:AddItem(brightPart, Dur)

			--------------- ball prompt ---------------
			local pickBallPrompt: ProximityPrompt = Instance.new("ProximityPrompt")
			pickBallPrompt.ActionText = "Pick Ball"
			pickBallPrompt.Name = "Pick Ball"
			pickBallPrompt.KeyboardKeyCode = Enum.KeyCode.Q
			pickBallPrompt.HoldDuration = 0
			pickBallPrompt.Style = Enum.ProximityPromptStyle.Custom
			pickBallPrompt.RequiresLineOfSight = false
			pickBallPrompt.MaxActivationDistance = 8
			pickBallPrompt.ClickablePrompt = true
			pickBallPrompt.ObjectText = ballId
			pickBallPrompt.Parent = pokeClone
			--------------- here's a pickPet ---------------

			local prompCOnn = function()
				pickPet(player, pet, pokeClone)
			end
			scopeMaid:GiveTask(pickBallPrompt.Triggered:Connect(prompCOnn))

			--------------- shake animation ---------------
			do
				local primary = pokeClone.PrimaryPart

				-- original orientation
				local originalCFrame = primary.CFrame

				-- how far it tilts left/right (in radians)
				local angle = math.rad(10)
				local tweenInfo = TweenInfo.new(0.2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)

				local goal = { CFrame = originalCFrame * CFrame.Angles(0, 0, angle) }
				local tween = TweenService:Create(primary, tweenInfo, goal)
				tween:Play()
			end
		end

		--call after T, uses A BuoyancySensor to call capturePet if a model is hit
		local function lastHit(pos: Vector3)
			-- clean the projectile trayectory
			scopeMaid:__index("trajectoryConnection"):Disconnect()
			-- detects parts in 10 radius
			local animDur = 2 -- dur for the pet captured
			local pet: Model -- the pet model that might get hit
			local parts = workspace:GetPartBoundsInRadius(pos, 10)

			-- loop all over the parts and detetct if its parent (a model or worksapace) has the isAPet attribute
			for _, part in ipairs(parts) do
				if not part.Parent:IsA("Model") then
					continue
				end
				if not part.Parent:GetAttribute("isAPet") then
					continue
				end
				pet = part.Parent

				local tween =
					TweenService:Create(part, TweenInfo.new(animDur), { Size = part.Size - Vector3.new(1, 1, 1) })
				tween:Play()

				capturePet(pet, animDur)
			end
		end

		-- use the connection for pivoting the ball, with kinematic equation, while t < T
		scopeMaid:__newindex(
			"trajectoryConnection",
			RunService.Heartbeat:Connect(function(deltaTime)
				t += deltaTime

				-- calculate the current position based in the cinematic function, this trajectory is also predicted by the client in a StarterPack.MagicBall.localScript
				--// s = s0 + v0*t + 0.5*a*t^2
				local newPos = startPosition + (initialVelocity * t) + 0.5 * GRAVITY * (t ^ 2)

				--Bouncing
				if lastY > 0 and newPos.Y <= 0 then
					-- This is the current velocity at the moment of impact
					-- The v0 (last initial velocity) at current time * gravity
					-- ImpactV = v0 + a*t
					local impactVelocity = initialVelocity + GRAVITY * t
					local bounceVelY = -impactVelocity.Y * BOUNCINESS

					if math.abs(bounceVelY) < 15 then
						lastHit(newPos)
						return
					end
					-- Keep on bouncing
					-- redefines de variable for newPos calculation, so newPosition changes
					-- Every bounce is creating a new Throw event

					-- impact point, from the ground
					startPosition = Vector3.new(newPos.X, 0, newPos.Z)
					-- bounce direction + bounce strength
					initialVelocity = Vector3.new(impactVelocity.X, bounceVelY, impactVelocity.Z)
					t = 0
					lastY = 0
					return
				end

				pokeClone:PivotTo(CFrame.new(newPos))
				lastY = newPos.Y
			end)
		)
	end)

	-- Add the marker connection to the maid for cleanup
	scopeMaid:GiveTask(markerConn)
end)
```