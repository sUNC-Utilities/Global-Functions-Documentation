Functions that allow **inspection/modification/creation** of Luau closures

---

## hookfunction

Hooks a function with another wanted function, returning the original unhooked function.

> [!Note]
> The hook shouldn't have more upvalues than the function you want to hook.
>                       
> All possible hooking closure pairs should be supported throughout L, NC, C. (NC = newcclosure)

```luau
function hookfunction<A1..., R1..., A2..., R2...>(function_to_hook: (A1...) -> R1..., function_hook: (A2...) -> R2...): (A1...) -> R1...
```

### Parameters

- `function_to_hook` - The function that will be hooked
- `function_hook` - The function that will be used as a hook

### Example

```luau
local function DummyFunction()
    print("I am not hooked!")
end

local function DummyHook()
    print("I am hooked!")
end

DummyFunction() -- Output: I am not hooked!

local OldFunction = hookfunction(DummyFunction, DummyHook)

DummyFunction() -- Output: I am hooked!
OldFunction() -- Output: I am not hooked!
```

---

## hookmetamethod

This function takes any lua value that can have metatable and attempts to hook the specified metamethod of the lua value with the `hookfunction` function.

Function can be safely implemented in Lua if `hookfunction` is properly implemented in C code. 
```luau
function hookmetamethod(object: table | Instance | userdata, metamethod_name: string, hook: (...any) -> (...any)): (...any) -> (...any)
```

### Parameter

- `object` - The object which has the metatable.
- `metamethod_name` - The name of the metamethod to hook.
- `hook` - The function that will be used as a hook.

### Example
```luau
local Original; Original = hookmetamethod(game, "__index", function(...)
    local Key = select(2, ...)
    print(Key)
    return Original(...)
end)

local _ = game.PlaceId
hookmetamethod(game, "__index", Original) -- Restores game's __index

-- Output: PlaceId
```

---

## newcclosure

> [!WARNING]
> Many executors are implementing this function using `coroutine` functions in Lua; these implementations won't pass sUNC checks.
>
> The wrapped function should be yieldable (meaning that the function should be able to call `task.wait`, for example)

This function takes in a function and wraps it into a C closure.

When the returned function is called, the original Lua closure is called, and arguments are passed to the original closure, and then the original closure returned arguments are passed to the caller of the C closure.

```luau
function newcclosure<A..., R...>(function_to_wrap: (A...) -> R...): (A...) -> R...
```

### Parameter

- `function_to_wrap` - A function to be wrapped.

### Example

```luau
local DummyFunction = function(...)
    return ...
end

print(iscclosure(DummyFunction)) -- Output: false

local WrappedFunction = newcclosure(DummyFunction)

print(iscclosure(WrappedFunction)) -- Output: true

local FunctionResults = WrappedFunction("Hello")
print(FunctionResults) -- Output: Hello
```

```luau
local DummyYieldingFunction = newcclosure(function()
    print("Before")
    task.wait(1.5)
    print("After")
end)

DummyYieldingFunction()
-- Output:
-- Before
-- yield for 1.5 seconds
-- After
```

---

## iscclosure

Checks if a given function is a C closure.

```luau
function iscclosure(func: (...any) -> (...any)): boolean
```

### Parameter

- `func` - The function to check.

### Example

```luau
local function DummyLuaFunction()
    print("This is an executor Lua closure")
end

local DummyCFunction = newcclosure(function()
    print("This is an Executor C Closure")
end)

local DummyStandardCFunction = print
local DummyGlobalCFunction = getgc

print(iscclosure(DummyCFunction)) -- Output: true
print(iscclosure(DummyGlobalCFunction)) -- Output: true
print(iscclosure(DummyStandardCFunction)) -- Output: true
print(iscclosure(DummyLuaFunction)) -- Output: false
```

---

## islclosure

Checks if a given function is a L closure.

```luau
function islclosure(func: (...any) -> (...any)): boolean
```

### Parameter

- `func` - The function to check.

### Example

```luau
local function DummyLuaFunction()
    print("This is an executor Lua closure")
end

local DummyCFunction = newcclosure(function()
    print("This is an executor C closure")
end)

local DummyStandardCFunction = print

print(islclosure(DummyLuaFunction)) -- Output: true
print(islclosure(DummyStandardCFunction)) -- Output: false
print(islclosure(DummyCFunction)) -- Output: false
```

---

## isexecutorclosure

Checks if a given function is the executor's closure.

```luau
function isexecutorclosure(func: (...any) -> (...any)): boolean
```

### Parameter

- `func` - The function to check.

### Example

```luau
local function DummyLuaFunction()
    print("This is an executor Lua closure")
end

local DummyCFunction = newcclosure(function()
    print("This is an executor C closure")
end)

local DummyStandardCFunction = print
local DummyGlobalCFunction = getgc

print(isexecutorclosure(DummyLuaFunction)) -- Output: true
print(isexecutorclosure(DummyCFunction)) -- Output: true
print(isexecutorclosure(DummyGlobalCFunction)) -- Output: true
print(isexecutorclosure(DummyStandardCFunction)) -- Output: false
```

---

## clonefunction

Creates and returns a new function that has the same behaviour as the passed function.
> [!NOTE]
> The cloned function must have the same environment as the original.
> 
> Any sort of modification to the original shouldn't affect the clone. Meaning that stuff like hooking the original will leave the clone unaffected.

```luau
function clonefunction<A..., R...>(function_to_clone: (A...) -> R...): (A...) -> R...
```

### Parameter

- `func` - The function to clone.

### Example

```luau
local function DummyFunction()
    print("Hello")
end

local ClonedFunction = clonefunction(DummyFunction)

print(debug.info(ClonedFunction, "l")) -- Output: 1
print(debug.info(ClonedFunction, "n")) -- Output: DummyFunction
print(ClonedFunction == DummyFunction) -- Output: false
print(getfenv(ClonedFunction) == getfenv(DummyFunction)) -- Output: true
```

---

## getfunctionhash

Returns a Hex represented SHA384 hash of the provided function's re-built bytecode

```luau
function getfunctionhash(function_to_hash): string
```

### Example

```luau
local function isSHA384Hex(hash)
    if #hash ~= 96 then
        return false
    end
    if not hash:match("^[0-9a-fA-F]+$") then
        return false
    end
    return true
end

local DummyFunction0 = function() end
local DummyFunction1 = function(...) end
local DummyFunction2 = function() end

print(isSHA384Hex(getfunctionhash(DummyFunction0))) -- Output: true
print(getfunctionhash(DummyFunction0) == getfunctionhash(DummyFunction1)) -- Output: false
print(getfunctionhash(DummyFunction0) == getfunctionhash(DummyFunction2)) -- Output: true
```
