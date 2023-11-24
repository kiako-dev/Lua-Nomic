# Lua Nomic

In this thesis I hope to outline fully, clearly, and with justification a starting codebase for Lua Nomic, a code Nomic based entirely in Lua.

This thesis will be broken up into Chapters and Sections as follows:
1. Our Goal Ruleset
    1. Why "yay" and "nay" instead of "for" and "against"?
    2. Where are my CFJs?
2. Designing the Code
    1. Interfacing
    2. Registration and Deregistration
    3. Motions and Resolutions
3. Speculating and Philosophizing
    1. On the Nature of Scams
    2. What's left to do?
3. Testing the Limits
    1. [Agora](https://agoranomic.com) and ???
    2. [Nomini](https://crowsworth.itch.io/nomini) and Supported Actions
    3. [Nomyx](https://github.com/nomyx/Nomyx) and Event Based Architecture
4. (Appendix) The Full Codebase

## Special Thanks

A special thanks to:
- [Enrique García Cota](https://github.com/kikito), whose [Lua sandbox library](https://github.com/kikito/lua-sandbox) taught me the wonders of `sethook` and alerted me to a number of easily-missed exploits. This Nomic does not recycle code from the above library (at least not intentially), but by construction some similarity must exist.
- [Chloe Baldwin](https://crowsworth.itch.io/), whose [Nomini Nomic ruleset](https://crowsworth.itch.io/nomini) inspired me to consider a drastically simpler core ruleset. Some key terms have been borrowed from Nomini, and some specific features are recreated in **4.2**.
- (peer reviewers go here)

# Our Goal Ruleset

Our core ruleset shall be as follows:

1. A player CAN register by announcement, specifying an alias. 
2. A player CAN deregister by announcement.
2. A player CAN submit a motion by announcement, specifying some plaintext code, and optionally providing a title and a comment. The motion is assigned an ID.
3. A player CAN vote on a motion not yet resolved, specifying "yay", "nay", or nil.
4. A player CAN resolve a motion by announcement if at least 3 players have voted. If the number of "yay" votes is at least two more than the number of "nay" votes, the code is executed on `GLOBAL` (and so may make significant and permanent changes to the gamestate.)

This is the complete ruleset we will implement in Chapter 2, and is all that is truly necessary for the core game of Nomic. In Chapter 4, we will explore some ways to implement a few of the more complex systems found in Agora, Nomini, and Nomyx.

### Why "yay" and "nay" instead of "for" and "against"?

In Lua, `for` is what is considered a "keyword". This ends up causing the code to look way worse. We can compare:

```lua
vote_tallies = {yay=0, nay=0}
-- Note that "for" describes the loop here; this is why it has issues in maps.
for vote in votes do 
  if vote == 'yay' then
    vote_tallies.yay = vote_tallies.yay + 1
  end
end
```

```lua
vote_tallies = {['for']=0, ['against']=0}
for vote in votes do
  if vote == 'for' then
    vote_tallies['for'] = vote_tallies['for'] + 1
  end
end
```

Plus, "yay" and "nay" are just fun words.

### Where are my CFJs?

Since Lua Nomic is a code nomic, it is _adjudicated by code_. This means, plainly, that success is permission. Therefore, players don't need to CFJ if something was "possible" or "allowed", since the outcome is plainly determined from the gamestate. This is the largest selling point of a code nomic (but changes the nature of scams entirely—we'll discuss this more in Chapter 3).

# Designing the Code

Let's dive in to how to make this concrete.

## Interfacing

We now must ask: 

> How does one actually do something "by announcement"? 

We will assume the context of a mailing list, as this is most familiar to Agorans. I've turned over in my head a number of methods, including sending data via JSON or another format, designing a DSL (a "domain specific language", a programming language specifically designed for this game), and many much worse formats. 

However, only one format I found to be completely fair to the context (and also not completely outside the scope of the project): if the game operates in code, then players should take their actions through that code. Fortunately for us, Lua offers the very powerful `load` function to turn any text into a function!

Thus, our first method:

```lua
-- [PROTO] This is not the final iteration of this feature!
-- Called by the host, passing in the content of the message received.
function accept(message)
  -- Compile the message.
  local fn = load(message, nil, 't', _G)
  -- If the message failed to compile, we return early.
  if not fn then return end
  -- Run the message! (`pcall` prevents errors from happening.)
  pcall(fn)
end
```

The arguments of `load` are as follows:
1. The text to turn into a function.
2. The "source" or name of the function; this is somewhat irrelevant for this project, so we'll leave it as `nil`.
3. The third argument tells Lua how to read it; `'t'` means it must be text (in oppose to effectively binary ones and zeroes), so that actions cannot be hidden.
4. The final argument is the environment the action behaves in (think of it as affecting Agora versus affecting Tournaments or Contracts.) `_G` means it applies to the entirety of the system.

Now, that's last one's a concern. We don't want a player to be able to do _anything_ with any action, just a few things. Since the solution is a bit more complicated, we'll address another concern first: 
> How do we determine who's who?

We'll go with the simplest option: the host determines who the author is, and simply tells us. We then set a global variable to keep track of it.

```lua
-- [PROTO] This is not the final iteration fo this feature!
-- Called by the host, passing in the content of the message received, and a unique identifier for the author.
function accept(message, author)
  local fn = load(message, nil, 't', _G)
  if not fn then return end
  _G.author = author -- Author while running.
  pcall(fn)
  _G.author = nil -- No author after running.
end
```

Now it's time to figure out that little issue of limiting access to the world. We'll start with the following helper method:
```lua
-- Makes any table read-only.
function readonly(source)
  -- Any Lua-native non-table type is immutable.
  if type(source) != 'table' then
    return source
  end
  return setmetatable({}, {
    __index = function(t, k) return readonly(source[k]) end,
    __newindex = function(t, k, v) err('Cannot mutate a read-only table!') end
  })
end
```
`setmetatable` is another very powerful Lua method; it takes a table (in this case, `{}`), and assigns it some special properties:
- `__index` changes the behavior whenever code reads a value. In this case, it will return a read-only copy of whatever the source table's value was (ignoring the actual table `{}`'s contents.)
- `__newindex` changes the behavior whenever code writes a value to a new key.
Since the table `{}` is empty, every key is new, so it always calls `__newindex` (and subsequently errors) when a player tries to change any value.

We can then alter our `accept` method to only allow read-only access to `GLOBAL`:

```lua
-- [PROTO] This is not the final iteration fo this feature!
-- Called by the host, passing in the content of the message received, and a unique identifier for the author.
function accept(message, author)
  local fn = load(message, nil, 't', readonly(_G))
  if not fn then return end
  _G.author = author -- Author while running.
  pcall(fn)
  _G.author = nil -- No author after running.
end
```

Unfortunately our work is not complete: because of the powerful `debug` library, it's trivial to get write access to `GLOBAL`. Users also have access to the filesystem due to the `os` and `file` tables. Very scary. Let's instead define a smaller, lesser environment for users to have access to, without these dangerous features:

```lua
Sandbox = {}
-- Math functions are generally safe to provide, as long as they can't be overwritten.
Sandbox.math = math
Sandbox.tostring = tostring

-- [PROTO] This is not the final iteration fo this feature!
-- Called by the host, passing in the content of the message received, and a unique identifier for the author.
function accept(message, author)
  local fn = load(message, nil, 't', readonly(Sandbox))
  if not fn then return end
  Sandbox.author = author
  pcall(fn)
  Sandbox.author = nil
end
```

We're just about there! We only have one _teensy_ issue to address: what do we do about players who submit code such as
> ```lua
> while true do end
> ```
This wonderful 18-character segment of code runs forever. In fact, there's an infinite family of never-halting code, and it's [provable](https://simple.wikipedia.org/wiki/Halting_problem) that we can't determine which programs will and won't halt. We need another solution.

Introducing another wonderful tool of the standard Lua library: `sethook`. With this, we can define a function that runs after a certain number of instructions. Let's start by setting this limit arbitrarily high, at about 500,000 instructions. This shouldn't be a problem until much iteration of Lua Nomic, and the implementation means this is easily fixed with a motion.

```lua
quota = 500000

-- Our error method for when we've used up our operations.
function over_quota()
  sethook()
  error('Exceeded quota: ' .. tostring(quota))
end

-- Called by the host, passing in the content of the message received, and a unique identifier for the author.
function accept(message, author)
  local fn = load(message, nil, 't', readonly(Sandbox))
  if not fn then return end
  Sandbox.author = author
  sethook(over_quota, '', quota)
  is_ok, errmsg = pcall(fn)
  sethook()
  Sandbox.author = nil
  -- We're also now printing any errors.
  if not is_ok then
    print('The message failed to execute correctly:\n  ' .. errmsg)
  end
end
```

(To complete the interface, a mailing list-based implementation should use any `print` statements to generate a reply.)

## Registration and Deregistration

To track registered players, we'll just make a `Players` table.

```lua
Players = {}
Sandbox.Players = {}
```

Then, the methods to register and deregister:

```lua
function datetime()
  return os.date() .. " " .. os.time()
end
Sandbox.datetime = datetime

function Sandbox.register(alias)
  -- We create a table holding the alias because we can do some goofy things with memory management later.
  Players[author] = {
    alias=alias,
    joined=datetime()
  }
end

function Sandbox.deregister()
  -- Players who deregister are lost to history. Perhaps a good thing to change with an early motion?
  Players[author] = nil
end
```

## Motions and Resolutions

Now the meat and potatoes of Nomic. We'll start by defining our global table to store motions:
```lua
Motions = {}
Sandbox.Motions = {}

-- A helper method to give some structure to motions.
function Motions.build(title, code, comment)
  return {
    author = Sandbox.author,
    title = title,
    code = code,
    comment = comment,
    submitted = datetime(),
    resolved = false,
    votes = {}
  }
end
```

Since we'll probably want to keep motions in a specific order, we're going to define this helper method:

```lua
-- Appends a value to a table, as if it were a list. Returns the resulting index.
local function append(table, value)
  table[#table+1] = value
  return #table
end
```

And the methods to submit, vote, and resolve motions:

```lua
function Motions.submit(title, code, comment)
  -- We check for syntax errors.
  local test_fn = load(code, nil, 't', _G)
  if not test_fn then error("Could not compile code!") end
  -- We add it to the list of Motions.
  local id = append(Motions, Motions.build(title, code, comment))
  print("Motion submitted! Given ID " .. id .. ".")
  -- We'll return the ID in the same message so players can vote at the same time as submitting.
  return id
end

function Motions.vote(id, outcome, comment)
  local motion = Motions[id]
  -- Checking that the motion actually exists.
  if type(id) != 'number' or motion == nil then
    error('Unknown motion #' .. id .. '.')
  end
  -- Checking if we've already resolved the motion.
  if motion.resolved then
    error('Cannot vote on resolved motion #' .. id .. '.')
  end
  -- Set the vote
  if outcome == 'yay' then
    motion.votes.yay[Sandbox.author] = comment or '(No comment.)'
    motion.votes.nay[Sandbox.author] = nil
    print("Voted 'yay' on #" .. id .. '.')
  elseif outcome == 'nay' then
    motion.votes.nay[Sandbox.author] = comment or '(No comment.)'
    motion.votes.yay[Sandbox.author] = nil
    print("Voted 'nay' on #" .. id .. '.')
  elseif not outcome then
    motion.votes.yay[Sandbox.author] = nil
    motion.votes.nay[Sandbox.author] = 
    print("Voted 'nil' on #" .. id .. '.')
  else
    error("Unrecognized ballot '" .. outcome .. "'.")
  end
end

function Motions.resolve(id)
  local motion = Motions[id]
  -- Ensure it's a valid motion.
  if type(id) != 'number' or motion == nil then
    error('Unknown motion #' .. id .. '.')
  end
  -- No re-resolving motions.
  if motion.resolved then
    error('Cannot resolve the already-resolved motion #' .. id .. '.')
  end
  -- Tally votes.
  local counts = { yay={}, nay={} }
  for voter in pairs(motion.votes.yay) do
    append(counts.yay, voter)
  end
  for voter in pairs(motion.votes.nay) do
    append(counts.nay, voter)
  end
  -- Ensure at least 3 people voted
  if #counts.yay + #counts.nay < 3 then
    error('A minimum of 3 votes are required to resolve a motion. Currently there are ' .. tostring(#counts.yay + #counts.nay) .. ' votes.')
  end
  -- If there are at least two more 'yay' votes than 'nay' votes, run the code.
  if #counts.yay >= #counts.nay + 2 then
    local fn = load(motion.code, nil, 't', _G)
    if fn == nil then error('The code failed to compile!') end
    motion.resolved = datetime()
    print('Resolving motion #' .. tostring(id) .. '.')
    pcall(fn)
  else
    error("There must be at least 2 more 'yay' votes than 'nay' votes. Currently there are " .. tostring(#counts.yay) " 'yay' votes and " .. tostring(#counts.nay).. " 'nay' votes.")
  end
end
```

# Speculating and Philosophizing

## On the Nature of Scams

## So what's left to do?

# Testing the Limits

## [Agora](https://agoranomic.com) and Tabled Actions

## [Nomini](https://crowsworth.itch.io/nomini) and ???

## [Nomyx](https://github.com/nomyx/Nomyx) and Event Based Architecture

```lua
Events = {}
Events.hook = {
  pre = setmetatable({}, {__mode = "k"}),
  post = setmetatable({}, {__mode = "k"}),
}
local function pop_and_pack(...)
  local t = table.pack(...)
  local pop = t[1]
  for i = 1, i <= t.n do
    t[i] = t[i + 1]
  end
  t.n = t.n - 1
  return pop, t
end
setmetatable(Events.hook, {
  __call = function (self, fn)
    self.pre[fn] = {}
    self.post[fn] = {}
    return setmetatable({}, {
      __call = function (self2, ...)
        local params = table.pack(...)
        for mixin in pairs(self.pre[self2.raw]) do
          local cancelled
          cancelled, params = pop_and_pack(mixin(table.unpack(params, 1, params.n)))
          if cancelled then return end
        end
        self2.raw(params)
        for mixin in pairs(self.post[self2.raw]) do
          local cancelled
          cancelled, params = pop_and_pack(mixin(table.unpack(params, 1, params.n)))
          if cancelled then return end
        end
      end,
      __index = { raw = fn },
      __newindex = function (t, k, v) end
    })
  end
})

function Events.add_mixin(hook, handler, pre_or_post)
  if pre_or_post == "pre" then
    Events.hook.pre[hook.raw][handler] = true
  elseif pre_or_post == "post" then
    Events.hook.post[hook.raw][handler] = true
  end
end
```

Example usage:

```lua
local function fn(x)
  print("A: " .. x)
end
local function pre_mixin(x)
  print("B: " .. x)
  if x < 5 then
    print("Because " .. x .. " < 5, we are cancelling the main call.")
    return true, nil
  end
  return false, x - 3
end
local function post_mixin(x)
  print("C: " .. x)
  return false, x
end

fn = Events.hook(fn)
Events.add_mixin(fn, pre_mixin, "pre")
Events.add_mixin(fn, post_mixin, "post")
fn(3)
-- B: 3
-- Because 3 < 5, we are cancelling the main call.
fn(6)
-- B: 6
-- A: 3
-- C: 3
```

# Appendix: The Full Codebase

```lua
function datetime()
  return os.date() .. " " .. os.time()
end

Players = {}
Motions = {}

Sandbox = {
  -- Libraries
  math = math,
  -- Methods
  type = type,
  datetime = datetime,
  -- Tables
  Motions = Motions,
  Players = Players,
}

quota = 500000

-- Error method for when we've used up our operations.
function over_quota()
  sethook()
  error('Exceeded quota: ' .. tostring(quota))
end

-- Called by the host, passing in the content of the message received, and a unique identifier for the author.
function accept(message, author)
  local fn = load(message, nil, 't', readonly(Sandbox))
  if not fn then return end
  Sandbox.author = author
  sethook(over_quota, '', quota)
  is_ok, errmsg = pcall(fn)
  sethook()
  Sandbox.author = nil
  if not is_ok then
    print('The message failed to execute correctly:\n  ' .. errmsg)
  end
end

-- Registers oneself under a specified alias.
function Sandbox.register(alias)
  Players[Sandbox.author] = {
    alias=alias,
  }
end

-- Deregisters oneself.
function Sandbox.deregister()
  Players[Sandbox.author] = nil
end

local function append(table, value)
  table[#table+1] = value
  return #table
end

-- A helper method to give some structure to motions.
function Motions.build(title, code, comment)
  return {
    author = Sandbox.author,
    title = title,
    code = code,
    comment = comment,
    submitted = datetime(),
    resolved = false,
    votes = {}
  }
end

function Motions.submit(title, code, comment)
  local test_fn = load(code, nil, 't', _G)
  if not test_fn then error("Could not compile code!") end
  local id = append(Motions, Motions.build(title, code, comment))
  print("Motion submitted! Given ID " .. id .. ".")
  return id
end

function Motions.vote(id, outcome, comment)
  local motion = Motions[id]
  if type(id) != 'number' or motion == nil then
    error("Unknown motion #" .. id .. ".")
  end
  if motion.resolved then
    error("Cannot vote on resolved motion #" .. id .. ".")
  end
  if outcome == 'yay' then
    motion.votes.yay[Sandbox.author] = comment or "(No comment.)"
    motion.votes.nay[Sandbox.author] = nil
    print("Voted 'yay' on #" .. id .. '.')
  elseif outcome == 'nay' then
    motion.votes.nay[Sandbox.author] = comment or "(No comment.)"
    motion.votes.yay[Sandbox.author] = nil
    print("Voted 'nay' on #" .. id .. '.')
  elseif not outcome then
    motion.votes.yay[Sandbox.author] = nil
    motion.votes.nay[Sandbox.author] = nil
  else
    error("Unrecognized ballot '" .. outcome .. "'.")
  end
end

function Motions.resolve(id)
  local motion = Motions[id]
  if type(id) != 'number' or motion == nil then
    error('Unknown motion #' .. id .. '.')
  end
  if motion.resolved then
    error('Cannot resolve the already-resolved motion #' .. id .. '.')
  end
  local counts = { yay={}, nay={} }
  for voter in pairs(motion.votes.yay) do
    append(counts.yay, voter)
  end
  for voter in pairs(motion.votes.nay) do
    append(counts.nay, voter)
  end
  if #counts.yay + #counts.nay < 3 then
    error('A minimum of 3 votes are required to resolve a motion. Currently there are ' .. tostring(#counts.yay + #counts.nay) .. ' votes.')
  end
  if #counts.yay >= #counts.nay + 2 then
    local fn = load(motion.code, nil, 't', _G)
    if fn == nil then error('The code failed to compile!') end
    motion.resolved = datetime()
    pcall(fn)
  end
end
```
