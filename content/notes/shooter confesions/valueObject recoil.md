Set a recoil as a valueObject and Rx 
Find out how to subscribe to the value of cooldown controller

- Create a heart beat connection in input manager that checks every cooldown time if the mouse1 button is still pressed
Problems? cooldown depends in the current strategy so basically is imposible for this heartbeat to be in InputManager, in another case i may call it continuosly in a heartbeat per frame but this may introduce a probloce for performan even if the attack is checking its cooldown an early returning, or maybe i can expose it's cooldown through a public method and and do in input manager 


### New task
It works great and the Execute ability is ready to call execute continously, however inputManager only fires once. Create a heartbeat connection when mousebutton1 is pressed and it starts a new timer, the heartbeat connection will look like this:
```lua
local distanceCheckConnection = RunService.Heartbeat:Connect(function(deltaTime)
	local timer = self.timer
	local lastTime = self.lastTime
	self.timer = timer + deltaTime
	if timer - lastTime >= 1 then
```
where self.timer is the time  elapsed since the button was first pressed