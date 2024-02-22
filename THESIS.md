# What's a "nomic"?

For the uninitiated, a nomic is a special sort of social game. The main difference between a nomic and any other sort of game is that, in a Nomic, players can change the rules.

In [Agora Nomic](https://agoranomic.org), the nomic I normally participate in, there's been some off-and-on discussion about pragmatism and platonism, and occasionally it's brought up the idea of a code nomic. The oldest proposed "code-like" nomic in Agora's history would be [favor's "A Completely Formal Nomic"](https://agoranomic.org/Herald/theses/html/XXXX-XX-XX-favor.html), which defines a DSL (a "domain-specific language") to describe game state and game actions. Alexis [later conceptualized a nomic](https://agoranomic.org/Herald/theses/html/2009-11-23-Alexis.html) that used propositional statements to assert the gamestate to be a certain way. Between these two theses existed [PerlNomic](http://odbook.stanford.edu/viewing/filedocument/63), which used Perl to create user interfaces and to manage the gamestate. Finally, there existed [Nomyx](), a Haskell-based nomic with a GUI that saw activity between 2013 and 2017.

In this post, we're going to derive our own nomic, using Lua as the backing language.

A special thanks to:
- [Enrique García Cota](https://github.com/kikito), whose [Lua sandbox library](https://github.com/kikito/lua-sandbox) taught me the wonders of `sethook` and alerted me to a number of easily-missed exploits. This Nomic does not recycle code from the above library (at least not intentially), but by construction some similarity must exist.
- (peer reviewers go here)

# The Essential Nomic Experience

In order to be a nomic, we kind of need to be able to alter rules. Specifically, we need to be able to (a) submit a proposal, (b) vote on a proposal, and (c) resolve those proposals. We'll also want (d) a way to keep track of players, and (e) allow players to join and leave.

For the sake of organization, let's put all of the proposal information into a single table, and all of the player information into a single table:
```lua
Players = {}
Proposals = {}
```

## Date Time

As a brief aside, it'll probably be helpful to have some memory of when things happened; y'know, getting a jump on the archival system and all. Let's just define that now.

```lua
function datetime()
  return os.date() .. " " .. os.time()
end
```

## Joining & Leaving

Alright, let's let players join and leave the game. We're going to assume that there's some global `author` field that our interface can determine uniquely based on who's doing the action. We can worry about how that's done later.

As a nice convenience, we'll store the date using our handy dandy `datetime` function. Also, despite what I just said about the `Players` table holding all the player-related stuff, we're *not* going to put these functions in the `Players` table, for a couple reasons:
- We don't yet know what `author` will look like, and it's in theory possible that someone's unique ID is `register` or `deregister` (an odd choice, perhaps, but a choice nonetheless).
- If we keep the `Players` table pure, we can pretty easily iterate over all the existing players.

```lua
function register(alias)
  -- We create a table holding the alias because we can do some goofy things with memory management later.
  Players[author] = {
    alias=alias,
    joined=datetime()
  }
end

function deregister()
  -- Players who deregister are lost to history. Perhaps a good thing to change with an early proposal?
  Players[author] = nil
end
```

## Appending to lists?

We're going to be working a lot with automatically-assigned numerical IDs, and those are really nice to just be able to do quickly. Time for our second helper method!

```lua
-- Appends a value to a table, as if it were a list. Returns the resulting index.
function append(table, value)
  table[#table+1] = value
  return #table
end
```

## How about those Proposals?

Now to dive into the most important part: proposals. We'll want to track quite a bit for this, so let's start with a helpful method to create new proposals. The stuff we're storing should make a lot of sense, given what we know. Just to make it explicit, the `title` is what we call the proposal, the `code` is what would be executed, and `comment` is a submitter-described comment of what it does or why it's being submitted. We're not adding the proposal directly to the table because we're thinking ahead here. It'll become more obvious when we get to the user interface.

```lua
-- A helper method to give some structure to proposals.
function Proposals.new(title, code, comment)
  return {
    author = author,
    title = title,
    code = code,
    comment = comment,
    submitted = datetime(),
    resolved = false,
    votes = {}
  }
end
```

Alright, now for some egregiously long code for submitting, voting, and resolving. Starting with submitting, we're doing four things here:
- We're attempting to precompile the submitted code, since if it doesn't compile it's a waste of time to vote,
- We're creating the proposal and appending it to our `Proposals` list,
- We're printing out that the proposal was submitted, and
- We're returning the ID right away, so that our interface can allow players to vote without a second thought.
```lua
function Proposals.submit(title, code, comment)
  local test_fn = load(code, nil, 't', _G)
  if not test_fn then error("Could not compile code!") end

  local proposal = Proposals.build(title, code, comment)
  local id = append(Proposals, proposal)
  print("Motion submitted! Given ID " .. id .. ".")

  return id
end
```

Now for voting! This will have to do a lot, but we'll break it down into steps. First up is validating the proposal. We'll check that (a) it's actually a proposal, and (b) it hasn't already been resolved.
```lua
function Proposals.vote(id, outcome, comment)
  local proposal = Motions[id]
  if type(id) != 'number' or proposal == nil then
    error('Unknown proposal #' .. id .. '.')
  end
  if proposal.resolved then
    error('Cannot vote on resolved proposal #' .. id .. '.')
  end
```

Now to set the vote. We're going to use that unique author ID to keep things convenient, and we'll also need to clear votes in case someone changed their vote. For my sanity, we're also going to make the eligible outcomes "yay" or "nay", because typing `votes.for[author]` isn't valid and `votes["for"][author]` is obnoxious. Additionally, if players vote `nil` (or any other fals-y Lua value), we'll say they meant to abstain, and if they submit anything else (e.g., an integer, "for", or "against"), we'll give an error.
```lua
  if outcome == 'yay' then
    proposal.votes.yay[author] = comment or '(No comment.)'
    proposal.votes.nay[author] = nil
    print("Voted 'yay' on #" .. id .. '.')
  elseif outcome == 'nay' then
    proposal.votes.nay[author] = comment or '(No comment.)'
    proposal.votes.yay[author] = nil
    print("Voted 'nay' on #" .. id .. '.')
  elseif not outcome then
    proposal.votes.yay[author] = nil
    proposal.votes.nay[author] = nil
    print("Voted 'nil' on #" .. id .. '.')
  else
    error("Unrecognized ballot '" .. outcome .. "'.")
  end
end
```
Currently, this won't save comments for players choosing to abstain. Perhaps that's something to change with a proposal?

Finally, for resolutions. Once again, a lot happening, but we'll go through it piecewise. First, we'll make sure it's valid and hasn't already been resolved, same as with voting.

```lua
function Proposals.resolve(id)
  local proposal = Proposals[id]
  if type(id) != 'number' or proposal == nil then
    error('Unknown proposal #' .. id .. '.')
  end
  if proposal.resolved then
    error('Cannot resolve the already-resolved proposal #' .. id .. '.')
  end
```

Now we need to tally votes. For a tiny bit of scam prevention, we're only going to count currently-registered players votes. We're also going to borrow a simplified version of Agora's quorum, and say that a proposal is adopted if and only if 3 or more people voted and and there are two more "yay" votes than "nay".

```lua
  local counts = { yay={}, nay={} }
  for player in next, t do
    if proposal.votes.yay[player] then
      append(counts.yay, player)
    elseif proposal.votes.nay[player] then
      append(counts.nay, player)
    end
  end
  if #counts.yay + #counts.nay < 3 then
    error('A minimum of 3 votes are required to resolve a proposal. Currently there are ' .. tostring(#counts.yay + #counts.nay) .. ' votes.')
  end
```

Finally, we're going to resolve the proposal and note it as doing so.

```lua
  if #counts.yay >= #counts.nay + 2 then
    local fn = load(proposal.code, nil, 't', _G)
    if fn == nil then error('The code failed to compile!') end
    motion.resolved = datetime()
    print('Resolving motion #' .. tostring(id) .. '.')
    pcall(fn)
  else
    error("There must be at least 2 more 'yay' votes than 'nay' votes. Currently there are " .. tostring(#counts.yay) " 'yay' votes and " .. tostring(#counts.nay).. " 'nay' votes.")
  end
end
```

And huzzah! We've got a nomic! Well, almost...

# The Player Interface?

Okay, so we've got the core of the nomic, but players still need to be able to *do* things. Looking back at our predecessors, it seems our options are:
1. A structured data language, like JSON, XML, or TOML, like in "A Completely Formal Nomic".
2. A DSL, like in Nomyx.
3. A GUI, like in PerlNomic and Nomyx.

Each of these has their own pros and drawbacks, but the biggest drawbacks are (a) expandability, (b) time to learn, and (c) time to implement. We'd ideally like something as powerful as Lua itself, isn't too much trouble to learn, and wouldn't require us to implement a parser or abstract syntax tree.

Say, one thing comes to mind. Why don't we just let players submit Lua code to do their actions? Now, I know what you're saying, "Didn't we just make a whole system that required players to vote to run Lua code? Why would we let people just execute code anyways?" Well, hear me out. There's a whole subset of Lua we haven't touched once, and we're going to use the heck out of it.

## Our Public-Facing Interface

Let's start with what we know. Players need to be able to submit code, and we'll need to be able to uniquely distinguish users. Let's start by defining the basic structure:

```lua
-- [PROTO] This isn't the final implementation of this function!
function accept(message, author)
  local fn = load(message, nil, 't', _G)
  if not fn then return end
  _G.author = author
  pcall(fn)
  _G.author = nil
end
```
Once again we're kind of cheating. *Something* has to host the Lua instance, so we're handing this off to the implementer to decide how to distinguish. If you're thinking of implementing this, here's a few ideas:
- If it's a mailing list, consider a salted hash of the email address.
- If it's a Discord bot, well, Discord already has an authentication system---just use that.
- If it's a webapp, perhaps there should be a log-in?
Then, the interface has to just call the `accept` function with the message and author. Stupid, simple!

(Oh yeah, another message to the host: Make sure to override `print` to respond via your chosen medium, so players can actually get feedback!)

Another important thing to note with this is that this function is defined *in Lua*. That means a proposal can change it. It shouldn't change the parameters, though; that's a lot of work for the host, and could just prevent it from working. Be careful!

## Introducing the Sandbox

Okay, now that we've got our baseline figured out, let's see how we can make this safe. Well, first, we should limit what all players can actually interact with. For example, the `debug` table is kind of always a problem for everyone. Conveniently for us, we can just change what table our `load` call is treating as the global environment. Let's define ourselves a `Sandbox` that holds all the state available to the player.

```lua
Sandbox = {
  math = math,
  tostring = tostring,
  datetime = datetime,
  register = register,
  deregister = deregister,
  Players = Players,
  Proposals = Proposals
}
```
It might be interesting to note the absent `append` method; the reason for this will become more apparent later.

Now, we just need to update our method as so:

```lua
-- [PROTO] This isn't the final implementation of this function!
function accept(message, author)
  local fn = load(message, nil, 't', Sandbox)
  if not fn then return end
  _G.author = author
  Sandbox.author = author
  pcall(fn)
  Sandbox.author = nil
  _G.author = nil
end
```

Looking good, except players can still change tables however they wish. Let's fix that.

## A new helper method: `readonly`!

Alright, let's fix that problem area. Introducing metatables, one of the most powerful features of Lua!

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

`setmetatable` changes the behavior for the table. `__index` affects reading values, while `__newindex` affects adding new indeces. Since we're making a new empty table, it's impossible to set an index without `__newindex` being implicitly called and throwing the error. Now let's make that `Sandbox` read-only!
```lua
-- [PROTO] This isn't the final implementation of this function!
function accept(message, author)
  local fn = load(message, nil, 't', readonly(Sandbox))
  if not fn then return end
  _G.author = author
  Sandbox.author = author
  pcall(fn)
  Sandbox.author = nil
  _G.author = nil
end
```

Now we've got an immutable state, except for the functions in `Sandbox` that can mutate the state (such as `Proposals.submit` and `Proposals.resolve`.) We're also not making `append` available because it *might* be able to mutate arbitrary tables. It probably won't be able to, but it's not a risk I want to deal with. Once again, a possible proposal to consider?

## The Halting Problem, A Problem No More!

Okay, we've almost got everything. Just need to deal with the trouble makers that submit code like:
```lua
while true do end
```
But that's not too bad. We just need to solve the Halting Problem. [Easy enough.](https://en.wikipedia.org/wiki/Halting_problem)

Jokes aside, our solution is actually quite simple: All programs halt, if we force them to error after 500,000 instructions.

Time to use a second powerful feature of Lua: `sethook`. With this, we can define a function that runs after a certain number of instructions.
```lua
quota = 500000

function over_quota()
  sethook() -- clears the hook
  error('Exceeded quota: ' .. tostring(quota) .. '.')
end
```
Since this is, once again, defined within Lua, it's also quite easy to increase this limit as the systems become more complicated.

Now to add that hook!

```lua
function accept(message, author)
  local fn = load(message, nil, 't', readonly(Sandbox))
  if not fn then return end
  _G.author = author
  Sandbox.author = author
  sethook(over_quota, '', quota)
  is_ok, errmsg = pcall(fn)
  sethook()
  Sandbox.author = nil
  _G.author = nil
  if not is_ok then
    print('The message failed to execute correctly:\n  ' .. errmsg)
  end
end
```
Oh, we're also printing errors now. That's no big deal, though.

# Conclusions

And we're done! Wasn't that fun? And nowhere near as bad. I'll throw a summary of all that wonderful code at the end.

If you're planning on running Lua Nomic, you'll need to make sure you do the following:
- Set up a system to carry messages to Lua via the `accept` method.
- Return methods with `print`.
- Don't let the thing crash! Oops!
- By default, this assumes that players will have a restricted environment, so we're treating `file`/`os`/et cetera are safe. I recommend keeping these features, as it's more interesting for creating historical records and such, but if you don't want to set up the environment, you'll need to look at all of everything that needs to be disabled. A particularly ambitious host might make players have access to a directory that can be viewed from a website subdirectory.

# Appendix: The Codebase!

```lua
Players = {}
Proposals = {}

function datetime()
  return os.date() .. " " .. os.time()
end

function register(alias)
  -- We create a table holding the alias because we can do some goofy things with memory management later.
  Players[author] = {
    alias=alias,
    joined=datetime()
  }
end

function deregister()
  -- Players who deregister are lost to history. Perhaps a good thing to change with an early proposal?
  Players[author] = nil
end

-- Appends a value to a table, as if it were a list. Returns the resulting index.
function append(table, value)
  table[#table+1] = value
  return #table
end

-- A helper method to give some structure to proposals.
function Proposals.new(title, code, comment)
  return {
    author = author,
    title = title,
    code = code,
    comment = comment,
    submitted = datetime(),
    resolved = false,
    votes = {}
  }
end

function Proposals.submit(title, code, comment)
  local test_fn = load(code, nil, 't', _G)
  if not test_fn then error("Could not compile code!") end

  local proposal = Proposals.build(title, code, comment)
  local id = append(Proposals, proposal)
  print("Motion submitted! Given ID " .. id .. ".")

  return id
end

function Proposals.vote(id, outcome, comment)
  local proposal = Motions[id]
  if type(id) != 'number' or proposal == nil then
    error('Unknown proposal #' .. id .. '.')
  end
  if proposal.resolved then
    error('Cannot vote on resolved proposal #' .. id .. '.')
  end
  if outcome == 'yay' then
    proposal.votes.yay[author] = comment or '(No comment.)'
    proposal.votes.nay[author] = nil
    print("Voted 'yay' on #" .. id .. '.')
  elseif outcome == 'nay' then
    proposal.votes.nay[author] = comment or '(No comment.)'
    proposal.votes.yay[author] = nil
    print("Voted 'nay' on #" .. id .. '.')
  elseif not outcome then
    proposal.votes.yay[author] = nil
    proposal.votes.nay[author] = nil
    print("Voted 'nil' on #" .. id .. '.')
  else
    error("Unrecognized ballot '" .. outcome .. "'.")
  end
end

function Proposals.resolve(id)
  local proposal = Proposals[id]
  if type(id) != 'number' or proposal == nil then
    error('Unknown proposal #' .. id .. '.')
  end
  if proposal.resolved then
    error('Cannot resolve the already-resolved proposal #' .. id .. '.')
  end
  local counts = { yay={}, nay={} }
  for player in next, t do
    if proposal.votes.yay[player] then
      append(counts.yay, player)
    elseif proposal.votes.nay[player] then
      append(counts.nay, player)
    end
  end
  if #counts.yay + #counts.nay < 3 then
    error('A minimum of 3 votes are required to resolve a proposal. Currently there are ' .. tostring(#counts.yay + #counts.nay) .. ' votes.')
  end
  if #counts.yay >= #counts.nay + 2 then
    local fn = load(proposal.code, nil, 't', _G)
    if fn == nil then error('The code failed to compile!') end
    motion.resolved = datetime()
    print('Resolving motion #' .. tostring(id) .. '.')
    pcall(fn)
  else
    error("There must be at least 2 more 'yay' votes than 'nay' votes. Currently there are " .. tostring(#counts.yay) " 'yay' votes and " .. tostring(#counts.nay).. " 'nay' votes.")
  end
end

Sandbox = {
  math = math,
  tostring = tostring,
  datetime = datetime,
  register = register,
  deregister = deregister,
  Players = Players,
  Proposals = Proposals
}

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

quota = 500000

function over_quota()
  sethook() -- clears the hook
  error('Exceeded quota: ' .. tostring(quota) .. '.')
end

function accept(message, author)
  local fn = load(message, nil, 't', readonly(Sandbox))
  if not fn then return end
  _G.author = author
  Sandbox.author = author
  sethook(over_quota, '', quota)
  is_ok, errmsg = pcall(fn)
  sethook()
  Sandbox.author = nil
  _G.author = nil
  if not is_ok then
    print('The message failed to execute correctly:\n  ' .. errmsg)
  end
end
```
