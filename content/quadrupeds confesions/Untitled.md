Hey, i programmed some pets in a farm that only needs to patrol and stay idle, i thing it may be a good idea to completly avoid an expensive class such as Humanoid so i endup manually adding the same features like humanoid Hip (necesary for quadrupeds) and other things such as custom MoveTo:

```
function MovementController:PlainMove(targetPosition)
	if not self._AnimationHandler then
		self:getAnimHandler()
	end

	-- Cancel previous movement
	if self._currentMovePromise then
		self._currentMovePromise:Destroy()
	end
	local startPos = self._root.Position
	local delta = targetPosition - startPos
	local dir = delta.Unit
	local distance = delta.Magnitude
	local timeToReach = distance / self.velocity

	self._currentMovePromise = Promise.new(function(resolve, reject)
		
		-- CLEAN
		self._maid:DoCleaning()

		-- Do some checks because we fear to NaN values
		do
			if not isValidPosition(startPos) then
				reject("Root position corrupted (void vector)")
				return
			end

			if not isValidPosition(targetPosition) then
				reject("Target position invalid")
				return
			end

			if timeToReach ~= timeToReach then -- Check for NaN
				reject("TimeToReach is NaN (infinite distance)")
				return
			end
		end

		-- ✅ Create cancellable delay using thread
		local rotateTime = 0.5

		local lookThatShit = CFrame.lookAt(startPos, targetPosition) * CFrame.Angles(0, math.rad(90), 0)
		local rotateTween = TweenService:Create(
			self._root,
			TweenInfo.new(rotateTime, Enum.EasingStyle.Linear, Enum.EasingDirection.InOut),
			{ CFrame = lookThatShit }
		)
		rotateTween:Play()

		-- TODO: add this task to maid
		-- CONNECT THE MOTION PIVOTING AFTER THE ROTATION
		local rotationCOmplete = rotateTween.Completed:Once(function(playbackState)
			self.isMoving = true

			-- Walking anim
			local animationTrask = self._AnimationHandler:LoadAnimation(true, "walk")
			animationTrask:AdjustSpeed(self.speed)
			local elapsed = 0

			self._moveConn = RunService.Heartbeat:Connect(function(dt)
				elapsed += dt

				if elapsed >= timeToReach then
					self._moveConn:Disconnect()
					self._moveConn = nil
					resolve(timeToReach)
					print("Position ", self._root.Position)
					self._AnimationHandler:LoadAnimation(false, "walk")
					self.isMoving = false
					return
				end

				local newPos = startPos + dir * self.velocity * elapsed
				self.Model:PivotTo(CFrame.lookAt(newPos, targetPosition) * CFrame.Angles(0, math.rad(90), 0))
			end)

			self._maid:GiveTask(self._moveConn)
		end)
		self._maid:GiveTask(rotationCOmplete)
	end)
	return timeToReach
end
```

But new models have came and they have some bugs which the only fix that comes up to my mind is to migrate all the pets to using humanoid so i guess i should slap my face. So the issue is that you can see somewhere the code i use PivotTo to move the model and the cframe is the current pet cframe but rotated with Cframe.Angles(0,90,0) and i still don't know why this pet models are so doom but i need to rotate them again, could rotate their anglle exactly 90 grades at axis y using align orientation:

```
`
local att = Instance.new("Attachment")
	att.Parent = self._root
	local AlignOrientation = Instance.new("AlignOrientation")
	AlignOrientation.Attachment0 = att
	AlignOrientation.Mode = Enum.OrientationAl
```

HUmanoid fail
### Rotate hrp with tweens
Here's a detailed report of an issue where i don't find the right solution and i hope you to help me to find a right solution. I encorouge you to double check any information you provide and exploring the internet for results for any internet source supporting your idea please include the link
If you use any tween on hrp it does work to face the direction however humanoid completly misunderstand this HRP rotation and gives up to walk. I simply will not use the tween and try to solve the issue where humanoid:MoveTo doesn't work. For some reason the game requires some attempts of MoveTo to start working, i just adjust HipHumanoid 
```
Cat 0.6
Elephant 3.1

```
Also i completly remove resizing model functions (in theory hip humanoid does adapt to but i'm not sure).
I'm completly sure the hip shit is not the problem, i have no clue why the moveTo is not working on the firstAttempts. Humanoid:MoveTo requires a HumanoidRootPart, all parts unanchored however the internally functionally for moving the rig is quite misterious such properties as AssemblyAngularVelocity and AssemblyLinearVelocity in hrp remains as Vector3.zero, the only i understand is that it have some custom phisics properties that are needed to avoid any kind of pet floating shit i really inspect the explorer during the game session hoping to find any kind of phisic constraint that was impediment to MoveTo working properly, i repeat that i have no idea what's the phisics behind this method, i only know it uses HRP, however i keep trying weird shits to see if the moveto starts working in the first intents (it requires many intents to start working) i though i can be a phisics issue like require more force so i made this little push to the pet:
```lua
local lookThatShit = CFrame.lookAt(self._root.Position, targetPosition) * CFrame.Angles(0, math.rad(90), 0)  

local hrp: BasePart = self._root

hrp:ApplyImpulse(lookThatShit.LookVector * hrp.AssemblyMass)

````
Obviously it doesn't work. I just think it might work because i will describe how this bug look to me: it looks like some kind of align position trying to move a part of mass heavier than the force but finally it start moving it when the force overcome the assembly mass. Any information about Humanoid:MoveTo working improperly will be helpful. thank you