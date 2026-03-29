Sorry for the late response, the testing were way more longer than i was expecting
  
i got a couple of questions about the gameplays, the character are interchangable, right? do you want a Pre-Game Screen for selecting between the batter slayer and the shark warrior? (it will simple)




-- I was directly loading the bat animas (always wrong), after the batter model is reaparanted to the player.Character this fires toolUnequip causing to load the weapon and the UI do bug

  
Also we need to talk a little bit about the balance of the game.
I'm not sure if you will use the same model from the video for the game, anyway if we are using a r-6 we'll fine for playing the animations. As the weapon for this model are two we can uses two hitbox 
![[Pasted image 20251228212343.png]]
the swings uses a single hitbox (makes 12 damage per hit, considering default's value is 100 points) , but the 4th hit of the combo may use these two hitboxes, duplicating its damage, so
- 1st hit 12
- 2nd hit 12
- 3rd hit 12
- 4th hit 24
or we can just create a single hitbox and make 4 hits of 12 damage
- 1st hit 12
- 2nd hit 12
- 3rd hit 12
- 4th hit 12

There are another adir (still working in the attacks)
![[Pasted image 20251228204152.png]]
I will working in the attacks functionalities until you answer me 

another question, there are sounds in the studio?

### The hitboxes problem
Okay, i'm facing an sorfware architecture problem and don't know what software pattern does fit here. The hipotesis is: "Client needs to know where to attach the hitboxes, and server know it". So i got a command pattern where the attacks abilities does something like this
```lua
function Basic:Execute()
	animHandler:LoadAnimation(true, swingName)
	-- Use client passes the data needed to create the hitbox in the client, then the client fires back the hit targets
	local detectionInfo = {
		Time = 2,
		Size = self.hitbox_size,
		CFrame = self.hitbox_offset,
		AttackID = attackId,
		Attacker = self.character,
	}
	self:CreateScopedPromise(detectionInfo):Then(function(hitTarget)
		hitTouchTemplate(data, hitTarget)
	end)
```
So here we are using client side hit detection. Now, original my current aproach were for weapons however this is character based. The same class with strategy pattern still does a perfect fit though.
```lua
function Weapon.new(char: Model, serviceBag)
end
function Weapon:Execute(action)
end
function Weapon:OnIncomingAttack(context)
end
```
There are several methods, however i'm pretty sure we need to check for the following method:
```lua
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
```
I have never changed their name, but i repeat you, it still does a perfect fit for the character-based, see:
```lua
function module:InitChar(char, char_name)
	
	--blabla, get binder
	local WeaponBinder = BinderProvider:Get("Armed")
	
	if WeaponBinder then
		local weaponHandler = WeaponBinder:Get(char)

		weaponHandler:registryTool(char_name)
		weaponHandler:EquiTool(char_name)
	end

end
```
As you can see, i only need a way for using the correct instances structure to know where are the weapons in the "model tree"
```lua
-- a strategy
function Batter.new(char, serviceBag)
	local self = setmetatable(CharacterBase.new(char, serviceBag), Batter)
	self:_createAbilities()
	return self
end
```