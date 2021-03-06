﻿---
title: Lua modules in Defold
brief: Lua modules allow you to structure your project and create reusable library code. This manual explains how to do that in Defold.
---

# Lua modules

Lua modules allow you to structure your project and create reusable library code. It is generally a good idea to avoid duplication in your projects. Script code that replicates identical behavior between different game objects can simply be copied and pasted. However, if instead the script code could be shared between the game objects, any changes to the single script code instance would instantly affect both game objects.

Defold allows you to include script files into other script files which allows you to share code. Moreover, Lua modules can be used to encapsulate functionality and data in an external script file for reuse in game object and GUI script files.

## Requiring files

Suppose we are building an application featuring butterflies of different kinds. We want to create some behaviors for these butterflies and share those behaviors. To do so we create a game object with a script file:

![Blue butterfly](images/modules/modules_blue_butterfly.png)

We put the following code in *blue_butterfly.script*:

```lua
require "modules_example.flying"

function init(self)
    fly_randomly()
end

function on_message(self, message_id, message, sender)
    if message_id == hash("fly_randomly_done") then
        fly_randomly()
    end
end
```

The idea is that we call `fly_randomly()` at `init()` which will cause the butterfly to animate to a new random position some distance relative to its origin. When the animation is done, we get a `fly_randomly_done` message back and immediately send the butterfly to a new random position (some distance relative its origin).

The first line `require 'modules_example.flying'` reads the script file *flying.lua* in the folder *modules_example* (where the application logic is stored) into the engine.

::: sidenote
The syntax of the filename string provided to `require` is a bit special. Lua will replace '.' characters in the filename string with path separators: '/' on Mac OS X and Linux and '\' on Windows.
:::

To create *flying.lua*, just add a new Lua module file to your project and name it:

![New module](images/modules/modules_new.png)

![New module name](images/modules/modules_new_name.png)

```lua
-- We need to store the original position.
local origin

-- Crudely fly to a random position at "radius" distance
-- from the original position
-- This function sends back a "fly_randomly_done" message when it's done.
function fly_randomly(radius)
    -- Radius is 100 unless specified
    radius = radius or 100
    local go_id = go.get_id()
    -- Store original location
    if origin == nil then
        origin = go.get_world_position(go_id)
    end

    -- Figure out a random position at max distance
    -- "radius" from origin
    local rand_angle = math.random(0, 3.141592 * 2)
    local rand_radius = math.random(radius)
    local offset = vmath.rotate(vmath.quat_rotation_z(rand_angle),
                                vmath.vector3(rand_radius, 0, 0))
    local rand_pos = origin + offset
    -- Set a random duration scaled against the radius to prevent
    -- too fast animation.
    local rand_duration = math.random(radius) / 100 + (radius / 200)
    -- Animate, then send a message back when completed
    go.animate(".", "position", go.PLAYBACK_ONCE_FORWARD,
        rand_pos, go.EASING_INOUTSINE, rand_duration, 0.0,
        function ()
            msg.post("#", "fly_randomly_done")
        end)
end
```

This code works pretty well and it makes the butterfly animate between random positions around its original location. Now, we add a yellow butterfly game object to the collection:

![Blue and yellow butterflies](images/modules/modules_blue_and_yellow.png)

The code for flying randomly already exist, so creating the script for the yellow butterfly is straightforward:

```lua
require "modules_example.flying"

function init(self)
    -- Yellow butterflies have a wider flying range.
    fly_randomly(200)
end

function on_message(self, message_id, message, sender)
    if message_id == hash("fly_randomly_done") then
        fly_randomly(200)
    end
end
```

When we run the code we find that the two butterflies join up and start to fly relative to the same origin. To understand what is happening here, recall how Defold manages [Lua contexts](/manuals/lua).

Now, what happened with the blue and yellow butterflies was an unfortunate side effects of sharing global data between game objects. Here is the definition of the variable origin:

```lua
-- We need to store the original position.
local origin
```

It is defined "local" which means that it is local to the current Lua context. Since all game objects are evaluated in the same Lua context, the yellow and blue butterfly will use exactly the same variable origin. One of the butterfly objects will set the variable to its origin, then the second butterfly will use the same value for its origin:

```lua
-- "origin" is not set if someone in the same context
-- has already set it...
if origin == nil then
    origin = go.get_world_position(go_id)
end
```

We’ll soon see how we can easily fix this bug, but first we should address a more subtle problem.

## Name spaces

In our included Lua file, we define a variable origin that we use to store data. This name might lead to problems later on. Suppose we add other types of behavior to our butterflies and use the name "origin" in another situation. Suddenly we cause a name collision. One way to guard against name collisions is to prefix everything in our file:

```lua
local flying_origin

function flying_fly_randomly(radius)
    ...
end
```

This works, but Lua provides a simple and elegant way to organize script code and data: modules.

## Modules

A Lua module is nothing more than a Lua table that we use as a container for our functions and behaviours. Here is the module version of "flying.lua", which also includes a fix to the data-sharing problem we hit earlier:

```lua
-- The table "M" contains the module.
local M = {}

-- We use a table to store original positions.
-- Note that the module becomes part of all game objects'
-- shared Lua context so we can't just store the origin in
-- a variable straight - it would be overwritten if more than
-- one GO used this module
M.origins = {}

-- Crudely fly to a random position at "radius" distance
-- from the original position
-- This function sends back a "fly_randomly_done" message when it's done.
function M.fly_randomly(radius)
    -- Radius is 100 unless specified
    radius = radius or 100
    -- We need current object id to index the original position.
    -- Can't use "."
    local go_id = go.get_id()
    -- Store origin position if it's not already stored.
    if flying.origins[go_id] == nil then
        flying.origins[go_id] = go.get_world_position(go_id)
    end

    -- Figure out a random position at max distance
    -- "radius" from origin
    local rand_angle = math.random(0, 3.141592 * 2)
    local rand_radius = math.random(radius)
    local offset = vmath.rotate(vmath.quat_rotation_z(rand_angle),
                                vmath.vector3(rand_radius, 0, 0))
    local rand_pos = flying.origins[go_id] + offset
    -- Set a random duration scaled against the radius to prevent
    -- too fast animation.
    local rand_duration = math.random(radius) / 100 + (radius / 200)
    -- Animate, then send a message back when completed
    go.animate(".", "position", go.PLAYBACK_ONCE_FORWARD,
        rand_pos, go.EASING_INOUTSINE, rand_duration, 0.0,
        function ()
            msg.post("#", "fly_randomly_done")
        end)
end

return M
```

The difference is that we create a table flying that we populate with our data (`flying.origins`) and functions (`flying.fly_randomly()`). We end the module by returning the table. To use the module we change the butterfly scripts to:

```lua
flying = require "modules_example.flying"

function init(self)
    flying.fly_randomly()
end

function on_message(self, message_id, message, sender)
    if message_id == hash("fly_randomly_done") then
        flying.fly_randomly()
    end
end
```

When requiring an external file, we assign the required script’s return value to a variable:

```lua
flying = require "modules_example.flying"
```

Since the module returns the table containing all the module code and data we get that table reference in the variable flying. We are free to name the table arbitrarily so we could just as well write:

```lua
banana = require "modules_example.flying"

function init(self)
    banana.fly_randomly()
end

function on_message(self, message_id, message, sender)
    if message_id == hash("fly_randomly_done") then
        banana.fly_randomly()
    end
end
```

This means that we can even use two modules with the same name--we just assign them locally to different variable names.

With the module mechanism we are now fully equipped to encapsulate shared functionality and avoid any name collisions.

## Naming conventions

There are many different ways to define the module table, but we recommend using the standard of naming the table containing the public functions and values `M` (See http://lua-users.org/wiki/ModuleDefinition). The use of the name `M` helps, since it prevents mistakes like accidentally shadowing the module table by a function argument with the same name.

```lua
--- Module description
-- @module foobar
local M = {}

--- Constant for bar
-- @field BAR
local M.BAR = "bar"

local function private_function()
end

--- If the module needs to retain a non-singleton state it should have a
-- create function that returns an "instance" with the state. This instance can
-- then be passed to the module functions.
-- @return Instance of foobar
function M.create()
    local foobar = {
        foo = "foo"
    }
    return foobar
end

--- This function does something.
-- @function do_something
-- @param foobar
-- @return foo
function M.do_something(foobar)
    return foobar.foo
end

--- This function does something else.
-- @function do_something_else
-- @param foobar
-- @return foobar
function M.do_something_else(foobar)
    return M.do_something(foobar) + M.BAR
end

return M
```

## Allowing monkey patching

>   "A monkey patch is a way to extend or modify the run-time code of dynamic languages 
>   without altering the original source code." [Wikipedia]

Lua is a dynamic language and as such it is possible to modify all of the builtin modules. This is extremely powerful and very useful, primarily for testing and debugging purposes. Monkey patching easily leads to strong coupling, which is usually a bad idea. However, if you write a module, you should make it possible to monkey patch your custom modules as well. Lua allows you to do things like:

```lua
-- mymodule.lua
local M = {}

M.foo = function()
    print('this is a public module function')
end

setmetatable(M, {
    __newindex = function(m, t)
        error('The user has tried to add the attribute ' .. t .. ' to the module!')
    end
})

return M
```

Tricks like the above is usually not a good idea. You should leave the decision on what to do to your module to the user.

## Beware of locals

If you decide to not define modules using the style mentioned above (by defining functions as table fields directly) you should be wary of how you use locals. Here is an example (from the blog [kiki.to](http://kiki.to/blog/2014/04/04/rule-3-allow-monkeypatching/)):

```lua
local M = {}

local function sum(a, b)
    return a + b
end

local function mult(a, b)
    local result = 0
    for i=1,b do
        result = sum(result, a)
    end
    return result
end

M.sum = sum
M.mult = mult

return M
```

This is a very simple calculator module. Notice how the module table is assigned its public functions at the end. Also notice how the local `mult()` function uses the local `sum()` function. Doing so might be marginally faster, but it also introduces a subtle issue. If we, for some reason, were to monkey patch the module like this:

```lua
local summult = require("summult")
summult.sum = function(a,b) return 1 end
print(summult.mult(5,2))
```

Now, even after being overridden, `summult.mult(5,2)` still returns `10` (while you would expect 1). The problem is that the user changes the `sum()` function on the module table, but internally in `multi()` the local and unmodified function is still used.

## Don’t pollute the global scope

This best practice is not specific to modules, but it’s worth reiterating the importance of not using the global scope to store state or define functions. One obvious risk of storing state in the global scope is that you expose the state of the module. Another is the risk of two modules using the same global variables (coupling). Note, though that Defold shares the Lua context only between objects in the same collection, so there is no truly global scope.

It’s a good practice during development to monitor the global table and raise an `error()` whenever the global table is modified.

::: sidenote
For more info, read the Lua Wiki page http://lua-users.org/wiki/DetectingUndefinedVariables
:::

This code can be used to guard the global table:

```lua
--- A module for guarding tables from new indices. Typical use is to guard the global scope.
-- Get the latest version from https://gist.github.com/britzl/546d2a7e32a3d75bab45
-- @module superstrict
-- @usage
--
--  -- Defold specific example. Allow the gameobject and gui script lifecycle functions. Also allow assignment of
--  -- facebook and iap modules for dummy implementations on desktop. The whitelist handles pattern matching and in the
--  -- example all functions prefixed with '__' will also be allowed in the global scope
--  local superstrict = require("superstrict")
--  superstrict.lock(_G, { "go", "gui", "msg", "url", "sys", "render", "factory", "particlefx", "physics", "sound", "sprite", "image", "tilemap", "vmath", "matrix4", "vector3", "vector4", "quat", "hash", "hash_to_hex", "hashmd5", "pprint", "iap", "facebook", "push", "http", "json", "spine", "zlib", "init", "final", "update", "on_input", "on_message", "on_reload", "__*" })
--
--  -- this will be allowed
--  __allowed = "abc"
--
--  -- and this
--  facebook = dummy_fb
--
--  -- this is ok
--  function init(self)
--  end
--
--  -- this will throw an error
--  if foo == "bar" then
--  end
--
--  -- this will also throw an error
--  function global_function_meant_to_be_local()
--  end
--
local M = {}

-- mapping between tables and whitelisted names
local whitelisted_names = {}

-- make sure that no one messes with the error function since we need it to communicate illegal access to locked tables
local _error = error

--- Check if a key in a table is whitelisted or not
-- @param t The table to check for a whitelisted name
-- @param n Name of the variable to check
-- @return true if n is whitelisted on t
local function is_whitelisted(t, n)
    for _,whitelisted_name in pairs(whitelisted_names[t] or {}) do
        if n:find(whitelisted_name) then
            return true
        end
    end
    return false
end

--- Guarded newindex
-- Will check if the specified name is whitelisted for the table. If the name isn't whitelisted
-- an error will be thrown.
-- @param t The table on which a new index is being set
-- @param n Name of the variable to set on the table
-- @param v The value to set
local function lock_newindex(t, n, v)
    if is_whitelisted(t, n) then
        rawset(t, n, v)
        return
    end

    _error("Table [" .. tostring(t) .. "] is locked. Attempting to write value to '" .. n .. "'. You must declare '" .. n .. "' as local or added it to the whitelist.", 2)
end

--- Guarded __index
-- Will throw an error if trying to read undefined value
-- @param t The table on which a new index is being set
-- @param n Name of the variable to set on the table
local function lock_index(t, n)
    if is_whitelisted(t, n) then
        return rawget(t, n)
    end

    _error("Table [" .. tostring(t) .. "] is locked. Attempting to read undefined value '" .. n .. "'.", 2)
end

--- Lock a table. This will prevent the table from being assigned new values (functions and variables)
-- Typical use is to call lock(_G) to guard the global scope from accidental assignments
-- @param t Table to lock
-- @param whitelist List of names that are allowed on the table
function M.lock(t, whitelist)
    assert(t, "You must pass a table to lock")
    whitelisted_names[t] = whitelist or {}
    local mt = getmetatable(t) or {}
    mt.__newindex = lock_newindex
    mt.__index = lock_index
    setmetatable(t, mt)
end

---
-- Unlock a table
-- @param t Table to unlock
function M.unlock(t)
    assert(t, "You must pass a table to unlock")
    local mt = getmetatable(t) or {}
    mt.__newindex = rawset
    mt.__index = rawget
    setmetatable(t, mt)
end

return M
```

Modules should also try to provide functions to manipulate internal state, instead of exposing it to the user. If you provide functions to manipulate internal state it becomes trivial to refactor the inner workings of the module without affecting users of the module. This practice of encapsulating state is a common practice in OOP languages, but it applies just as much to Lua module development.

## Stateless or stateful modules?

Stateful modules keep an internal state that is shared between all users of the module and can be compared to singletons:

```lua
local M = {}

-- all users of the module will share this table
local state = {}

function M.do_something(foobar)
    table.insert(state, foobar)
end

return M
```

A stateless module on the other hand doesn’t keep any internal state. Instead it provides a mechanism to externalize the state into a separate table that is local to the module user. Most approaches to stateless modules rely on a constructor function within the module, often called `create()` or `new()`. The constructor function returns a table where state is stored, and depending on the implementation sometimes also the module functions themselves.

## Stateless modules using only a state table

Perhaps the easiest approach is to use a constructor function that returns a new table containing only state. The state is explicitly passed to the module as the first parameter of every function that manipulates the state table.

```lua
local M = {}

local function private_function(self, bar)
    return self.public_variable .. bar
end

function M.public_function(self, bar)
    return private_function(self, bar)
end

function M.new(foo)
    local instance = {
        public_variable = foo
    }
    return instance
end

return M
```

You use the module like this:

```lua
local foobar = require(“foobar”)
local fb = foobar.new(“foo”)
print(fb.public_variable)
print(foobar.public_function(fb, “bar”))
```

## Stateless modules using metatables

::: sidenote
_Metatables_ is a powerful feature in Lua. A good tutorial on how they work can be found here: http://nova-fusion.com/2011/06/30/lua-metatables-tutorial/
:::

Another approach is to use a constructor function that returns a new table with state _and_ the public functions of the module each time it’s called:

```lua
local M = {}

local function private_function(self, bar)
    return self.public_variable .. bar
end

function M:public_function(bar)
    return private_function(self, bar)
end

function M.new(foo)
    local instance = {
        public_variable = foo
    }
    return setmetatable(instance, { __index = M })
end

return M
```

You use this module in the following manner:

```lua
local foobar = require(“foobar”)
local fb = foobar.new(“foo”)
print(fb.public_variable)
print(fb:public_function(“bar”))
```

::: sidenote
Note the use of the colon operator when calling and defining functions on the module. An expression like `o:foo(x)` is just another way to write `o.foo(o, x)`, that is, to call `o.foo()` adding `o` as a first extra argument. In a function declaration, a first parameter `self` is added.

Read more here: http://www.lua.org/pil/5.html
:::

## Stateless modules using closures

A third way to define a module is by using Lua closures (See http://www.lua.org/pil/6.1.html for details). A function returns the instance and the closure contains the instance, private and public data and functions---there is no need to pass the instance as an argument (either explicitly or implicitly using the colon operator) like when using metatables. This method is also somewhat faster than using metatables since function calls does not need to go through the `__index` metamethods. One drawback is that each closure contains its own copy of the methods so memory consumption is higher. Another is that it's not possible to monkey-patch the instance methods in a clean way.

```lua
local M = {}

function M.new(foo)
    local instance = {
        public_variable = foo
    }

    local private_variable = ""

    local private_function = function(bar)
        return instance.public_variable .. private_variable .. bar
    end

    instance.public_function = function(bar)
        return private_function(bar)
    end

    return instance
end

return M
```

Use the module in the following manner:

```lua
local foobar = require(“foobar”)
local fb = foobar.new(“foo”)
print(fb.public_variable)
print(fb.public_function(“bar”))
```

## Modules should behave expectedly

Modules are primarily intended to simplify reuse and to encapsulate behavior. When writing a module, it is important to design the module and name its functions in such a way that the expected behavior can be inferred simply by looking at module, function and argument names.

If a function in the module account is named `login(username, password)` the expected behavior is that the function should perform a login to an account using the specified username and password---and nothing else.

## Document the module

::: sidenote
Details on the LDoc documentation standard can be found here: https://github.com/stevedonovan/LDoc
:::

When documenting a module or function, remember to write documentation only when it adds information that can’t be inferred from function and argument names. In many cases short, well named functions and well named function arguments are enough to infer how the modules works, what to use and what to expect from a function call. Document return values, preconditions and examples of usage. Write documentation using the LDoc standard.


