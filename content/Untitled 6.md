Okey, that's is working so godamn good, the position is always right in front of the enemy. However the animation is pushing the enemy backgroud, which breaks all the hit lock mechanic (the enemy flyes and don't stay for reciving the hits) so idecided to anchor their HRP:

```

defender_hrp.Anchored = true

task.delay(context.hitLock.duration, function()

defender_hrp.Anchored = false

end)

```

It works, but its happening an interesting bug though, if the player were already anchorec in a different task, let's say anchored at second 3 during 3 seconds, and then is anchored again after 5, the task delay from the first anchor (on 3d second) will break the second anchor (from the 5 second). I need to adjust this for effectService, called from the weapon, to anchor the parts, weapon should kinda register this, but let me tell you the current structure.

ability:Execute()

local data = {

serviceBag = self._serviceBag,

attacker = self.character,

stun_time = self.stun_time,

hitLock = {

duration = 2,

},

}

self:CreateScopedPromise({ Attacker = self.character, hitboxes = { detectionInfo } })

:Then(function(hitTarget, hitPart)

hitTouchTemplate(data, hitTarget, hitPart)

self:DoFrenzy()

end)

end

```

The last part may look a little complex but is just a template wrapped in a promise, this template uses the data variable and the hitbox information combined to ask to the defender if he can be hit:

```

--hitTemplate.luau

return function(parentClass, hitTarget, hitPart)

-- does shared logic between all the attacks

-- uses the service bag (passed thtugh the data) for calling the defender's binder

weaponEnemy:OnIncomingAttack(context)

```

The defender binder passes this data into a chain of responsability called _defenseChain

```

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

modifiedContext = modifier:Process(modifiedContext)

-- If a modifier blocks completely, stop the chain

if modifiedContext.blocked then

return

end

end

self:TakeHit(modifiedContext)

end

function Weapon:TakeHit(context)

EffectService:Apply(context)

end

```

So what we need to improve? weapon is calling effect service to apply all the hit logic, but this effect service is also generating states on the character of the binder, we need this weapon binder to be aware of such states, (e.g. character is anchored so we can queue new attacks that anchor him, increasing its anchor duration. But you consider the cleanest solution for achiving this?