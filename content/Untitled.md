
https://www.youtube.com/watch?v=t45nG7xvjEE&list=PLG6c4yPY2xvQLVtT6RWfZhGUB7PvuVwOh&t=1608s
26:48
![[Pasted image 20250926162609.png]]
"See detailss"
![[Pasted image 20250926163924.png]]


grey = (58,53,50)
dark-grey = 54, 67, 73
white = 250,248,227
blue = 27, 82, 136
green = 61, 213, 136
light blue = 51,213,227

![[Pasted image 20250927115205.png]]
local BuffTables = {
	XP = {
		common = 1.5,
		uncommon = 2,
		rare = 2.5,
		epic = 3,
	},
	DMG = {
		common = 2,
		uncommon = 4,
		rare = 8,
		epic = 16,
	},
	-- actualSpeed = actualSpeed - (actualSpeed*speedBuff)

	speed = {
		common = 1/16,
		uncommon = 1/8, 
		rare = 1/4,
		epic = 1/2,
	},
	-- actualLife = actualLife + buffSpeed

	stamina = {
		common = 32,
		uncommon = 64, 
		rare = 128,
		epic = 256,
	},
	poisonHealth = {
		common = 128,
		uncommon = 256, 
		rare = 512,
		epic = 1024,
	},
}

local AspectNames = {
	XP = "ExperienceSeed",
	DMG = "GemOfPower",
	speed = "SpeedPoison",
	stamina = "EnchantedFood",
	poisonHealth = "PoisonHealth",
}