Okay, let's analyze service bag now. Starting by the GetService method there's a tons of thing happening, service bag uses a registry of service, self._services, where the key seems to be a table:
```lua
function ServiceBag:GetService(serviceType)
		-- after if statements and error checks we fall here
		-- Try to add the service if we're still initializing services
		self:_addServiceType(serviceType)
		self:_ensureInitialization(serviceType)
		return self._services[serviceType]
		end
```
`_addServiceType` add the new service to `_services` and initialize it, this is interesting:
```lua
function ServiceBag:_addServiceType(serviceType)
	-- asserts&ifStatements
	-- Construct a new version of this service so we're isolated
	local service = setmetatable({}, { __index = serviceType })
	self._services[serviceType] = service

	self:_ensureInitialization(serviceType)
end
```
I assume that `_services` is a table referencing for the already initializing and presumably working services, while `_ensureInitialization(serviceType)` is only using during the init phase
```lua
function ServiceBag:_ensureInitialization(serviceType)
	if self._initializedServiceTypeSet[serviceType] then
		return
	end

	if self._initializing then -- this is only true in the :Init() phase
		self._serviceTypesToInitializeSet[serviceType] = nil
		self._initializedServiceTypeSet[serviceType] = true
		self:_initService(serviceType) -- here i artificially added an Init method to binder, which inserts service bag into __args, it worked, but it just show me a different error
	elseif self._serviceTypesToInitializeSet then -- this is the registry of the services not initialized
		self._serviceTypesToInitializeSet[serviceType] = true
	else
		error("[ServiceBag._ensureInitialization] - Cannot initialize past initializing phase ")
	end
end
````