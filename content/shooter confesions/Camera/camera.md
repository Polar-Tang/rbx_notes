Allright. We got how this library can be used, the example is at `node_modules/@quenty/camera/src/Client/Effects/ImpulseCamera.story.lua`, the important thing is at 
```lua
local impulseCamera = ImpulseCamera.new() -- creates a camera like 
maid:GiveTask(stack:Add((defaultCamera + impulseCamera):SetMode("Relative"))) -- add this camera to the top. acamera + otherCamera are both effects at the set time
  
maid:GiveTask(RunService.RenderStepped:Connect(function()
	local topState = stack:GetTopState()
	
	if topState then
		topState:Set(Workspace.CurrentCamera) -- updates the camera stack
	end
end))
```
I need to create a crosshair camera, please find a camera like that works to create the effect of zoom default camera. This will look like this:
```lua
	local cameraService = _G.CameraService

	local stack = cameraService:GetCameraStack()
	maid:GiveTask(stack)

	local impulseCamera = ImpulseCamera.new()
	maid:GiveTask(defaultCamera)

	maid:GiveTask(defaultCamera:BindToRenderStep())

	local ZoomEffect = ZoomEffect.new() -- this effect doesnt exist, you should find a effect at node_modules/@quenty/camera/src/Client/Effects that zooms the camera
	maid:GiveTask(stack:Add((impulseCamera + ZoomEffect):SetMode("Relative")))

	maid:GiveTask(RunService.RenderStepped:Connect(function()
		local topState = stack:GetTopState()
		if topState then
			topState:Set(Workspace.CurrentCamera)
		end
	end))
```

GetTopState gets the current camera effect and returns its state. Set sets the current cframe and FieldOfView to the state
```lua
function CameraState.Set(self: CameraState, camera: Camera)
	camera.FieldOfView = self.FieldOfView
	camera.CFrame = self.CFrame
end
```
Then the real answer is how the internal Spring does change the cameraState.
Why `impulseCamera:Impulse(Vector3.new(0, math.random() - 0.5, math.random() - 0.5) * 50, 50, 0.2)` worked but 


### spring to first person
I'm thinking about how to create a zoom effect, but not the effect you got by using SmoothZoomedCamera:ZoomIn i do mean to use a smooth transition effect to third person to first person and vicersa, it's the same which happen when rolling up the wheel and also is similar to tween the camera, however that second one is not an option. I really think spring can add this smoothness. The internal zoomethCamera expose its spring
```
function SmoothZoomedCamera.__index(self: SmoothZoomedCamera, index)
-- code..
elseif index == "Target" or index == "TargetZoom" then
        return self.Spring.Target
    end
    end
```
I try to use spring however i totally ignore the spring usage, by this plain explanation and example provided at the top of the spring module.
    """"
    A physical model of a spring, useful in many applications.
    
A spring is an object that will compute based upon Hooke's law. Properties only evaluate
        upon index making this model good for lazy applications.
    
```lua
        local RunService = game:GetService("RunService")
        local UserInputService = game:GetService("UserInputService")
    
        local spring = Spring.new(Vector3.zero)
    
        RunService.RenderStepped:Connect(function()
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then
                spring.Target = Vector3.new(0, 0, 1)
            else
                spring.Target = Vector3.zero
            end
    
            print(spring.Position) -- A smoothed out version of the input keycode W
        end)
```
""" end quote
that's why i assume i can get the player position and trigger first person by doing this:
```
        local cameraService = _G.CameraService
            local stack = cameraService:GetCameraStack()
            local maid = Maid.new()
            
            local defaultCamera = DefaultCamera.new()
            local zoomEffect = SmoothZoomedCamera.new()
        
            local player_char = Players.LocalPlayer.Character or Players.LocalPlayer.CharacterAdded:Wait()
            local player_pos = player_char.PrimaryPart.Position
            
            zoomEffect.Target = player_pos
        
            maid:GiveTask(stack:Add((defaultCamera + zoomEffect):SetMode("Relative")))
            
            maid:GiveTask(RunService.RenderStepped:Connect(function()
                local topState = stack:GetTopState()
                    topState:Set(Workspace.CurrentCamera)
            end))
        
            return function()
                maid:DoCleaning()
            end
```
however i also ignore how the spring properties would have any effect to the internal camera state, for sure there's a hidden methamethod behind the scenes, by doing this we are impulsing the camera:
```
        function SmoothZoomedCamera.__newindex(self: SmoothZoomedCamera, index, value)
            if index == "TargetZoom" or index == "Target" then
                local target = math.clamp(value, self.MinZoom, self.MaxZoom)
                self.Spring.Target = target
        
                if self.BounceAtEnd then
                    if target < value then
                        self:Impulse(self.MaxZoom)
                    elseif target > value then
                        self:Impulse(-self.MinZoom)
                    end
                end
                -- more code..
                end
                end
                function SmoothZoomedCamera.Impulse(self: SmoothZoomedCamera, value)
                    self.Spring:Impulse(value)
                end
```

That's why the code from above will error because math.clamp its expecting a number instead of a vector3. However event after set the max Zoom number i can i still don't fire the "first person" mode that roblox uses by default, this zommed camera seems to have some kind of cap that i do not understand. Also the first person mode is triggered automatically when i spin the mousewheel enough to trigger it. If i could spring the camera exactly where the head is, that would mean i will at the camera subject and that will make the body not visible (that's practically first person camera)

Please create a effect at src/client/UI/cameraHandlers/camerasStates i'm not sure about the name, but it does inherit from node_modules/@quenty/camera/src/Client/Effects/CustomCameraEffect.lua and expose a method to spring to a position (i give a position and the camera gets to that position)

### damng spring
I'm trying to spring my camera however i do wonder if has something to do with a spring properties (it contracts if the distance is longer than the spring length) because the positions are right however the camera seems to fall short. In sumary it does spring to
```lua
--[=[
	Springs the camera to the specified position.
	@param position Vector3
]=]
function SpringPositionCamera:SpringTo(position: Vector3)
	assert(typeof(position) == "Vector3", "Bad position")
	self.Spring.Target = position
end
```
but springing a camera:
```lua
springEffect.Spring.Position = Workspace.CurrentCamera.CFrame.Position

-- Spring to head position
springEffect:SpringTo(head.Position)
```
I'm considering using phisics simulated to see if the problem is in my current setup for the cameras or the problem is in spring, because the positions are perfect.
Thank you so much for the debbugin snippets of code, i did:
```lua
while true do
	print(self.Spring.Position)
	task.wait()
end
```
and the prints for the goal position, end position and the last while print are the same.

#### Understand impulseCamera 
Okay, before continue doing anything i really need to compehend how spring.Position does change cameraState cframe. Your steps go like
```lua
RenderStepped fires
    → stack:GetTopState()
    → reads .CameraState on the top entry
    → top entry is SummedCamera(defaultCamera + impulseCamera)
    → SummedCamera reads defaultCamera.CameraState + impulseCamera.CameraState
    -- let's stop here
    → impulseCamera.CameraState is computed from spring position RIGHT NOW
    
```
camera state is created on demand-like
```lua
function ImpulseCamera._aggregateSprings(self: ImpulseCamera): Vector3
	local position = self._defaultSpring.Position

	for i = #self._springs, 1, -1 do
		local spring = self._springs[i]
		local animating, springPosition = SpringUtils.animating(spring, EPSILON)

		position = position + springPosition
		if not animating then
			table.remove(self._springs, i)
		end
	end

	return position
end

function ImpulseCamera.__index(self: ImpulseCamera, index)
	if index == "CameraState" then
		local newState = CameraState.new()

		local position = self:_aggregateSprings()
		newState.CFrame = CFrame.Angles(0, position.Y, 0)
			* CFrame.Angles(position.X, 0, 0)
			* CFrame.Angles(0, 0, position.Z)

		return newState
		-- more code...
```
_aggregateSprings is pure black magic. Please explain `_aggregateSprings` and how do i create my custom ImpulseCamera with Spring.Target support
I'm triying to make my own Impulse camera with this method but i get the error  attempt to call missing method 'SpringTo' of table on SpringTo calling:
```lua
--!strict
--[=[
	Spring position camera effect for smooth transitions to specific positions, e.g., first-person mode.
	@class SpringPositionCamera
]=]

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Spring = require(ReplicatedStorage.Nevermore.Quenty.spring.Shared.Spring)
local CameraState = require(ReplicatedStorage.Nevermore.Quenty.camera.Client.CameraState)
local CustomCameraEffect = require(ReplicatedStorage.Nevermore.Quenty.camera.Client.Effects.CustomCameraEffect)
local SummedCamera = require(ReplicatedStorage.Nevermore.Quenty.camera.Client.Effects.SummedCamera)

local SpringPositionCamera = {}
SpringPositionCamera.ClassName = "SpringPositionCamera"

export type SpringPositionCamera =
	typeof(setmetatable(
		{} :: {
			Spring: Spring.Spring<Vector3>,
			_effect: CustomCameraEffect.CustomCameraEffect,
		},
		{} :: typeof({ __index = SpringPositionCamera })
	))

function SpringPositionCamera.new(): SpringPositionCamera
	local self: SpringPositionCamera = setmetatable({} :: any, SpringPositionCamera)

	self.Spring = Spring.new(Vector3.zero)
	self.Spring.Speed = 20
	self.Spring.Damper = 1


	return self
end

--[=[
	Springs the camera to the specified position.
	@param position Vector3
]=]
function SpringPositionCamera:SpringTo(position: Vector3)
	assert(typeof(position) == "Vector3", "Bad position")
	self.Spring.Target = position
   
end

function SpringPositionCamera.__add(self: SpringPositionCamera, other)
	return SummedCamera.new(self._effect, other)
end

function SpringPositionCamera.__index(self: SpringPositionCamera, index)
	if index == "CameraState" then
		local newState = CameraState.new()

		local position = self.Spring.Position
		newState.CFrame = CFrame.new(position)

		return newState
    end
end


return SpringPositionCamera
```

#### Todo for tmr
- Do something like 