### New challenge feature
The new task that client gives me was a little unexpected for me. I need you to help me to plan how to implement this new feature, but before let me give you big picture about the game.
so there are two different states machines for different pet instances. Pet for combat and pet for plots, let's focus in pet for plots. The player stars in a place grow-garden like, there are different spots like little farms for each player, every spot farm is for a player and all the pets are placed by the server but the 
```lua
-- src/server/Mediator.luau
-- Called in CharacterAdded
function module.PlayerAdded(player: Player, playerData)
	-- Need to type playerData but is the data stored about the player
	InitPlot(player, playerData.pets) -- Every pet got an unique id as key and uses as directly reference. Pet is created from a template which is looked up as ServerStorage[playerPetData].PetName
	Registry:InitPlayer(player, playerData.pets) -- all the pet table information used for the server Registry handler
	
	local PetNew = ReplicatedStorage.RemoteEvents.PetNew
	-- The pet information is sent to the client asynchronous-like
	PetNew:FireClient(player, { -- (note that is a remote event but for this task we can consider to change to a RemoteFunction)
		allPets = playerData.pets,
	})
end
```

The client listen to this remove event for doing `_G["pets_registry"] = data.allPets` and that's the source of truth.
##### What about state machines?
As i mentioned earlier there are three state machines, one for the boss, petFollowing (the pet at the combat instance) and pet in the plot (what we are focusing on) this uses state machines. The plot pets are tagged from the server as NPC and the tag does replicate in the client.
ClassAddedSignal do call the Init method of this binder:
```lua
-- this method is called on nearby players
function StateMachineBinder:Init(player)
	-- the state machine is like the npc's brain

	-- Load the state machine starting at idle state
	local state = StateMachine.new(
		"Idle",
		StateMachine:LoadDirectory(StatesFolder),
		setCTX(self.char, self.serviceBag, player, self.maid)
	)
	-- setCTX create the table used for the context of the stateMachine + additional setup useful to see
	self.state = state
	self._initialized = true
	return self
end
```
and setCtx setup does set the context for the state machine, controllers and another stuff

```lua
return function(parent, serviceBag: any, player, maid)
	local current_pet_stats = petStats[parent:GetAttribute("petName")] -- static data for specific pet

	local player_owner = parent:WaitForChild("Owner").Value
	local isOwnerLocal = tostring(player_owner) == tostring(player.UserId)

	local petData = getStats(parent, maid, isOwnerLocal) 

	if isOwnerLocal then
		maid:GiveTask(createProximityPrompt(parent))

		repeat
			task.wait()
		until _G["pets_registry"] and _G["pets_registry"][parent.Name]
		local petInfo = _G["pets_registry"][parent.Name]

		-- another set up
	end
	
	return {
		maid = maid,
		------------------- timers -------------------
		timer = 0,
		lastIdleTime = 0,
		lastTriggerTime = 0,
		lastEatTime = 0,
		lastPatrolTime = 0,
		patrolStartTime = 0,
		eatingStartTime = 0,
		---- Saves the last time an state where trigger, the we can know to trigger the state based in that substracction
		patrol_interval = current_pet_stats.patrol_interval,
		eat_interval = current_pet_stats.eat_interval,
		patrol_duration = current_pet_stats.patrol_duration,
		eat_duration = current_pet_stats.eat_duration,
		---------------------------------------------------------
		-------- Controlles --------
		SoundController = isOwnerLocal and SoundController.new(petData, serviceBag) or nil,
		EatController = EatController.new(petData, serviceBag, player),
		MovementController = MovementController.new(petData, serviceBag),
		TargetingController = TargetingController.new(petData, serviceBag),
		PetAnimatorController = PetAnimatorController.new(petData, serviceBag),
	}
end
```
This is very important to know because the new task requires behavior that comes directly from current_pet_stats, static data for specific pet
##### Task request

Support / Non-Combat Pets

Pig
Passive: Money Boost – Earn 2x coins from all sources
Bonus: Boss rewards give extra coins

Giraffe
Passive: Discount – All kibble and shop items cost less
Bonus: Chance for purchases to cost nothing

Parrot
Passive: Speed Boost – 2x player walk speed
Bonus: Small chance to dodge hazards while moving

Bunny
Passive: Hatch Boost – 2x egg hatch speed
Bonus: Reduces cooldown between hatches

Turtle
Passive: Protection – Reduces damage taken outside of combat (traps, hazards)
Bonus: Grants a temporary shield when low health

Cat
Passive: Luck Boost – Higher chance of rare pets from eggs
Bonus: Increased chance for bonus rewards

Alpaca
Passive: Storage Boost – Increases inventory capacity
Bonus: Items stack higher and last longer

Elk
Passive: Kibble Efficiency – Kibble gives more XP to pets
Bonus: Small passive XP gain over time

Panda
Passive: Balance Boost – Small boost to coins, XP, and luck
Bonus: Slightly increases all other pet effects

Elephant
Passive: Resource Boost – Gain more resources (coins, drops) from all actions
Bonus: Chance to double collected rewards

This new task are a big problem, because each pet ability has no relantionship and i though it can be a simple factor variable that can be nil but the events to boost are drastically unrelated from each other and if i use "variable that can be nil" from everywhere the boost is required the code will turn messy and hard to scale.
As i'm still not sure where to call this "effects" you can ask for more context because i'm still not very sure where Bonus is called. Also i'm not sure how "friend boost" does work (client also wants that) maybe we should define clearly what a boost looks like and then wonder about the right pattern that does it fit better here. I think 
of kinda facade that depends on the player Registry to stack a boost based in this list. However i still got doubts about its integration, i wonder does this facade/god class/whatever expose public method? if so we should called every specific method related to its operation where a boost may be required. From eatCotroller, to a shop list, and every fucking remote Event at src/server/Mediator.luau (yes we have a mediator). What do you think? 

Okay, i've been researching, friend boost is simple a percentage factor that do increase income rewards. What income rewards? probably just Coins, here are incommings resources
currently this game got the following event happen most of the time and they can be boosted. **implemented**:
- Xp, EatController from the state machine do send a remote event asking the server to increase the xp gained by the pet ( Panda)
- Coins, the player gains a specific amount of coins per seconds (Money Boost, pig, Panda, Elephant)
That's incommings currently in the game, here are another sources that are implemented but need to be redefined, **implemented but needs poolishing**:
- Egg rng, currently i have a very basic version where you directly buys the pet rarity, but i think there are differents eggs and the rarity does come from rng (Luck Boost, Cat)
- Egg duration, currently all eggs last 10 second to hatch (Hatch Boost, bunny)
- Storage -- not implemented at all, there's a default backpack inventory with a single item avaible (Storage Boost, Alpaca)
- Shop keepers -- i wasn't sure about how to handle prices and make it smooth with the player progess so all the items are currently free and they are simply some eggs and an item (speed coil) (Discound, giraffe)
Here are player.Character.Humanoid factors that increase something related to the player character Humanoid. **player humanoid**:
- Walk speed -- this is humanoid.WalkSpeed = newValue (speed bost, parrot)
- Reduce damage -- there are no enemies, traps or any kind of hazards, also there's no defined what the player lost if dies, it will simply respawn (Protection, turtle)
Incommings that need to be programmed, **not implemented**.
- Kibbly, and item that "rebirths" the pet, (Kibbly efficiency, Elk)
- luck may use for spin roulettes luck (Panda).

Please make a table for the thins that can be boost in the first row. 
these are row, please fill it as its listed
Pet is a row
state in the game (implemented, implemented but needs poolishing, player humanoid, not implemented)
Here's the boost coverage table:

| Boost Target               | State           | Pig  | Giraffe    | Parrot | Bunny | Turtle   | Cat  | Alpaca      | Elk   | Panda   | Elephant |
| -------------------------- | --------------- | ---- | ---------- | ------ | ----- | -------- | ---- | ----------- | ----- | ------- | -------- |
| **Coins/sec**              | Implemented     | 2x ✅ |            |        |       |          |      |             |       | small ✅ | more ✅   |
| **Pet XP (eat)**           | Implemented     |      |            |        |       |          |      |             |       | small ✅ |          |
| **Egg RNG / Rarity**       | Needs polish    |      |            |        |       |          | 2x ✅ |             |       | luck ✅  |          |
| **Egg hatch duration**     | Needs polish    |      |            |        | 2x ✅  |          |      |             |       |         |          |
| **Shop prices**            | Needs polish    |      | discount ✅ |        |       |          |      |             |       |         |          |
| **Inventory storage**      | Not implemented |      |            |        |       |          |      | +capacity ✅ |       |         |          |
| **Kibble efficiency**      | Not implemented |      |            |        |       |          |      |             | +XP ✅ |         |          |
| **Spin roulette luck**     | Not implemented |      |            |        |       |          |      |             |       | luck ✅  |          |
| **WalkSpeed**              | Humanoid        |      |            | 2x ✅   |       |          |      |             |       |         |          |
| **Damage reduction**       | Humanoid        |      |            |        |       | reduce ✅ |      |             |       |         |          |
| **Boss coin bonus**        | Bonus proc      | ✅    |            |        |       |          |      |             |       |         |          |
| **Free purchase proc**     | Bonus proc      |      | ✅          |        |       |          |      |             |       |         |          |
| **Dodge hazard proc**      | Bonus proc      |      |            | ✅      |       |          |      |             |       |         |          |
| **Reduced hatch cooldown** | Bonus proc      |      |            |        | ✅     |          |      |             |       |         |          |
| **Temp shield (low HP)**   | Bonus proc      |      |            | ✅      |       |          |      |             |       |         |          |
| **Bonus rare pet proc**    | Bonus proc      |      |            |        |       |          | ✅    |             |       |         |          |
| **Item stack bonus**       | Bonus proc      |      |            |        |       |          |      | ✅           |       |         |          |
| **Passive XP over time**   | Bonus proc      |      |            |        |       |          |      |             | ✅     |         |          |
| **All effects +small**     | Bonus proc      |      |            |        |       |          |      |             |       | ✅       |          |
| **Double reward proc**     | Bonus proc      |      |            |        |       |          |      |             |       |         | ✅        |
| **Friend boost (coins)**   | Implemented     | —    | —          | —      | —     | —        | —    | —           | —     | —       | —        |

A few things worth flagging from filling this out:

**Turtle's Bonus** — I had to leave it on both Parrot's dodge row and Turtle's own row because the doc lists dodge under Parrot and shield under Turtle. Turtle's shield is a _reactive_ proc (triggers at low HP), which is mechanically very different from every other bonus — it needs a health threshold watcher, not just a RNG roll at an event point. Worth flagging that separately when you design the bonus system.

**Panda stacks on everything** — its bonus ("slightly increases all other pet effects") is a meta-multiplier, not a direct boost target. That's a special case in the registry: it's not a key you query, it's a modifier applied to every other key's final value.

**Friend boost** — I left it as `—` across all pets since it's a flat coin income percentage applied independently of which pet the player owns. It belongs in the registry but as its own source, not tied to a specific pet column.

Want me to now sketch the `BoostRegistry` module structure based on this table?

The bouns are interesting, they will not generate on demand, they will generate randomly in the server so it notifies the client with time. For example suddenly the player gets a bonus where he can buy something for free. It will not generate on demand because it happens in the server, let me explain myself, imagine it generates on demand when player opens the shop ui, it would require a fireserver to tell the server that's open the shop and fireClient back, this introduces a delay (not big, but a delay) and additionally complexity that can be avoided in this other design. Generate bonus randomly, the server generates a bonus with a task delay where timer does come from rng 30, 300 secs (all bonus can happen between a half of a minute and 5 minutes), when it does happen it tells the client which should do some kind of alert to the client. Less complex and at the end of the day is practically the same but helps to keep the things in the server.
That's simply events called after some time. what still worries me are the pet passive abilities. You're idea of querying a boost registry is good but also considering this design.
Xp, Coins, Egg rng, Egg duration, Storage, Shop keeper prices, Kibly factors, luck rng = everything is an observable/valueObjects that are changed by rx.combineLatest current pets in the register. 
If we do something like
```lua
Rx.combineLatest({PETSOBERVABLES}):Pipe({
	-- here's pseudo code ,idk how to do this
	Rx.filter(function(intValue)
	if intValue.Name == "xp_boost"
		local someBost = 20
		intValue.Value += someBost
	end)
})
```
What do you think?
### Pet non-combat abilities **$300**
I offer you doing the base system for boosting + 4 pet abilities 

**Pet abilities**
pig, Giraffe, Panda, elephant 

Every pet got a passive effect and a bonuses
**Passive effects**:
- full server-authoritative
- boost while the pet is active (in the player plot)
- Full reactive programming
**Bonus effect**
- Triggered  from rng 30, 300 secs (all bonus can happen between a half of a minute and 5 minutes)
- The triggering happens randomly and alerts the client
- This will happen after a given amount of time, if the pet is sold before the schedule bonus does cancel

[Here's a table](https://www.notion.so/pet-abilities-non-combat-3328ac3aaccd80ee9d4efcae3ef15674?showMoveTo=true&saveParent=true) (use vertical and horizontal scroll) it's about the status for every pet ability, two of the pets with state not implemented cannot be programmed before a new system (inventory system, connects UI with a registry in the server and elk which needs kibly system) There are two other pet abilities that can be programmed but will be incomplete, the turtle (there's no hazzard system ) and panda abilities increase on luck an the only rng is in the eggs but there can be a spin roulette in the future. 

**Deadline**: This will take me easily 1 week, and i can do the system for boost and the bonus effects schedulers

## Design idea
### Core idea: Effects are _named slots_, not method calls

Instead of the boost system exposing `applySpeedBoost()`, `applyKibbleEfficiency()` etc., every game system that _can be boosted_ queries a **BoostRegistry** for a specific named key:

lua

```lua
-- Any system asks: "what's my current multiplier for X?"
local multiplier = BoostRegistry:Get(player, "walk_speed")      -- Parrot
local multiplier = BoostRegistry:Get(player, "coin_gain")       -- Pig / Elephant / Panda
local multiplier = BoostRegistry:Get(player, "kibble_xp")       -- Elk
```

Every BoostRegistry will be called should be called in the server so this benefits are server authorative


### Design
- Need to find out how to turn a table into an observable
- Turn registry.pets:Observe
- Create a classes called Incommings. It has all the value listed here:
	Xp, Coins incomming, Egg rng, Egg duration, Storage, Shop keeper prices, Kibly factors, luck 
- This class should be flawless dependent. It may only require Registry or viceversa but the class will be hardly required. May also be posible registrying it in `_G`
- Start designing the Pipe that changes the valueObject values. 
- First find the proper method for these values, may be Rx.map. Idk
- For each pig multiply the Coins
- Use that multiplier in that class that have the task delay recursion
- That task is almost done