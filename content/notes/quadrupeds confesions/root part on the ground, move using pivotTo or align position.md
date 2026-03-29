You are quite right, the humanoid where giving me a lot of problems. My only problems of my current approach was the pets floatin but this can be solved by unanchoring hrp, using collision groups, roblox will still use gravity and phisics mechanics that are useful for what we want. All the descendats has simply a collision group
```lua

local function onDescendantAdded(descendant)
-- Set collision group for any part descendant

	if descendant:IsA("BasePart") then
		descendant.CanCollide = true
		descendant.CollisionGroup = "Npcs"
	end
end
```
But i still moving the model using pivot to causes the model to go to that exact position, however :MoveTo is the model property to move the rootpart to the specific position, 
`local lookThatShit = CFrame.lookAt(startPos, targetPosition) * CFrame.Angles(0, math.rad(90), 0)`
Now instead of using PivoTo i use MoveTo in the runservice connection which can corrects the Y position 
```lua
local newPos = startPos + dir * self.velocity * elapsed
--self.Model:PivotTo(CFrame.lookAt(newPos, targetPosition) * CFrame.Angles(0, math.rad(90), 0))
self.Model:MoveTo(newPos)
```
By doing this i can mainly avoid manual setting a Y offset accordingly to the model (like a custom hipHeight property). However i don't know why there's a model that does not play the tween at all, every model except for this one plays the tween but i have no idea why this model didn't run the tween at all
```lua
local rotateTween = TweenService:Create(
	self._root,
	TweenInfo.new(rotateTime, Enum.EasingStyle.Linear, Enum.EasingDirection.InOut),
	{ CFrame = lookThatShit }
	)
rotateTween:Play()

--task.delay(rotateTime,function() -- fired but the tween didn't happen
--local rotationCOmplete = rotateTween.Completed:Connect() fired instantly
-- after the tween creates the run service connection which for this pet happens without tweeing
```
The only thing that comes up to my mind is that self._root is not a valid instance. The self.root is waited and is porperly initialized, the only thing i see is unusual and different to literally all the pets is its orientation, `-0, -0, 0` this one is the only rootpart that have this (don't know why this happens) but as it is the only difference i think is worth to mention it. Anyway any information you can find about a tween don't playing after :Play is useful, please doublecheck what your saying and if it comes from any source in the internet you can include the link

### Geometric problem
This is going so good, the pet is moving nicely above the ground, walking with no humanoid, i'm not using PivotTo or rootpart anchored. However the position to an unreachable Y cause the pet using MoveTo to jitters. I realized that instead of hard coding an Y value that does it fit with the pet i can arrange the offsets between the rootpart and pet legs manually in the studio. So as the rootpart are above the ground, not elevetad in the same torso position, is literally like a box on the ground that is moved and i need to adjust its size to allow so its position can be 2.351 and the rig is still visually above the ground
### Improperly riging
This is the bug that worry me the most, i always dislike riging and i'm pretty sure the problem comes from riging. The models are fully animated in the client as they also moves in the client, there's a function done by the server were a teleporting, all the pets are fully animated in the client and some models (i don't know why some models have this issue, i studied the inspector looking for differences between the models that have the bug which the ones that don't and i have no idea why this happens) teleport back to the server position in this event
the real quiestion is how does roblox know the Y distance rootpart has and tries to keep it. 

Hey! here are a list of what i added to the game 
DOg
0.208 15

CAT
1.243 15

0.488, 9
lion
1.821 4

![[Pasted image 20260310182559.png]]

![[Pasted image 20260310182741.png]]****


❌ Death respanwn UI or something
✅ enchacing pet movement with custom physics (no pets floating)
✅ when a player leaves their farm is destroyed
✅ there's not pet jump over the fence or something like that

Hey, i've been testing the game a long while and i have fixed some bugs regarding the pet movement, there's some issue that i cannot fix about the rigs though where the pet crouches down to eat but the humanoid rootpart don't change the model orientation (i'm speaking about the eat animation). I can directly describe how to solve this rig problem but it had happened me a lot and i can attach the same model where is happening a similar issue and where is not, also you can notice i optimize the models totally avoiding the use of Humanoid class. Hope you find this useful