A body that do not deform on force being applied. 
```lua
local parts = PhysicsUtils.getConnectedParts(part)
```
- Distances between points stay constant
- The body can translate and rotate    
- But it cannot bend or stretch

From a rigid body, every forces decomposes in two
1. Linear acceleration
	Linear motion depends only on mass and center of mass
2. Angular acceleration

#### Torque
what forces do when applied to a distance