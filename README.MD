# Roblox Hot-Reloader for plugins!

This package hot-reloads modules in roblox studio with an isolated environment and avoids module caching.

_This module requires access to `loadstring` and it's recommended to be used only in **Plugins**. For hot-reloading in games, check [Rewire](https://github.com/sayhisam1/Rewire)_

This package is used in [UI Labs](https://ui-labs-roblox.github.io/ui-labs-docs/)

# How to use:

## Loading a module:

**Let's see how you can load a module**

First let's create an Environment where all the modules will live

```lua
local HMR = require("hmr")
local Environment = HMR.Environment

local env = Environment.new()
```

Now that we have an environment where our required modules can live, rather than using `require(...)` we load a module using `env:LoadDependency(module)`. This will return a [Promise](https://eryn.io/roblox-lua-promise/) that resolves with the result of the module, or rejects if the module failed to run.

_Note that this will also use `LoadDependency` recursively on any `require` call used inside of the module._

```lua
local env = Environment.new()

local module = ...
local dependency = env:LoadDependency(module)
local dependencyReturn = dependency:expect()
```

You can require more modules in the Environment by calling `env:LoadDependency()` again. Modules that have already been loaded (_required_) by either you or automatically will return the cached result. If a module is in the proccess of being required it will return the same promise

```lua
local module = ...

local dep = env:LoadDependency(module):expect()
local dep2 = env:LoadDependency(module):expect() --returns the cached result

print(dep == dep2) --true
```

### Accessing the `_G` table

Modules that are required in this environment will access a separated `_G` table instead of the real one. You can access this table in `env.Shared`

### Modifying globals

HMR environment replaces the globals `script`, `require` and `_G` in the modules to work, if you want to modify the globals with your custom values you can do this by calling `env:InjectGlobal(key, value)`.

_you cant modify the globals that HMR already replaces_ (`script`, `require`, `_G`)

```lua
local env = Environment.new()

env:InjectGlobal("print", function(...)
   print("printed from environment:", ...)
end)
env:InjectGlobal("customglobal", 10)



------ [[ in the required module ]] ------

print("hello world") -- printed from environment: hello world

local injected = customglobal --10
local injected = getfenv()["customglobal"] --10
```

`env:InjectGlobal()` will call `env:EnableGlobalInjection()` if it hasnt been enabled, however, if you require a module while global injections arent enabled, these changes wont work for that module, so, if you plan to replace globals at some point after you already loaded a module, you should call `env:EnableGlobalInjection()` before

### Listening for changes

HMR environment listens for changes in the module scripts that have been required. you can check when any of the required modules have been changed using the signal `env.OnDependencyChanged`. This can be used to reload the modules when any of the modules change. You can registry a module to listen by calling `env:ListenDependency(module)`. This will not load the module, it only listens for `env.OnDependencyChanged`

```lua
local env = Environment.new()

env.OnDependencyChanged:Connect(function(module)
    print("Module", module.Name, "has changed")
end)
```

### Reloading the modules

When you want to reload these modules again (_by listening `env.OnDependencyChanged` for example_) you need to create a new Environment and `Destroy()` the old one.

Calling `env:Destroy()` will not pause or cancel any code running in the required modules and will still receive env:LoadDependency() calls.

This will only disconnect changes for `env.OnDependencyChanged`, so make sure any running code in the modules gets cleaned.

```lua
local connection = nil
local env = nil
local module = ...

function LoadModule()
    if connnection then
        connection:Disconnect()
    end
    if env then
        env:Destroy()
    end

    env = Environment.new()
    env:LoadDependency(module)

    connection = env.OnDependencyChanged:Connect(function()
        LoadModule()
    end)
end

LoadModule()
```

### Hook on destroyed

You can use `env:HookOnDestroyed(callback)` to call a callback before the environment gets destroyed. Note that this is a callback and not a connection, so if you call it twice, only the last callback will be invoked

## Using the Hot-Reloader

HMR also exports a **HotReloader** that requires a module and automatically listens for env.OnDependencyChanged, managing and destroying the Environment for you. When the HotReloader is reloaded `HotReloader.OnReloadStarted` connection is fired HMR

This is used in [UI Labs](https://ui-labs-roblox.github.io/ui-labs-docs/) to automatically hot-reload stories

```lua
local HMR = require("hmr")
local HotReloader = HMR.HotReloader

local reloader = HotReloader.new(module) -- Entry module is required

reloader.OnReloadStarted:Connect(function(result)
    result:andThen(function(val)
        print("Reloaded", val)
    end)
end)

-- Use this to start the hot-reloader, you can also call this to force it to reload again
local dependency = reloader:Reload()


```

### Managing the Environment in the Hot-Reloader

The Hot-Reloader has `HotReloader:BeforeReload(callback)`, this will invoke the callback function _before_ loading the module but _after_ creating a new environment. Useful for setting up the environment before requiring the modules, this is a callback and not a connection, delaying this callback will delay the reloading and can cause unexpected results

```lua
local HMR = require("hmr")
local HotReloader = HMR.HotReloader

local reloader = HotReloader.new(module) -- Entry module is required

-- This should be called before reloading
reloader:BeforeReload(function(env)
    env:InjectGlobal("print", function(...)
       print("printed from environment:", ...)
    end)
end)

local dependency = reloader:Reload()
```

### Turning off AutoReloading

You can turn off Auto-Reloading with `HotReloader.AutoReload`, this is true by default, setting it to false will skip `env.OnDependencyChanged` events. Setting it to false will still accept `HotReloader:Reload()` calls

```lua
local reloader = HotReloader.new(module)

-- No longer listens changes
reloader.AutoReloader = false

reloader:Reload() -- It will still reload the module

-- Listens changes again
reloader.AutoRelaoder = true
```

## Extra

**Environment:**

-   `Environment:GetDependencyResult(module)` returns the loaded module result if exist without requiring it
-   `Environment:IsDependency(module)` returns if a module is loaded
-   `Environment.EnvironmentUID` unique [HttpService GUID](https://create.roblox.com/docs/reference/engine/classes/HttpService#GenerateGUID) for this environment

**Hot Reloader:**

-   `HotReloader.Module` module passed in HotReloader.new()
-   `HotReloader.GetEnvironment()` returns the current environment.
-   `HotReloader:Destroy()` will destroy the current environment and the reload connections
-   `HotReloader.OnDependencyChanged` fires when `env.OnDependencyChanged` is fired
