# File Structure
The only file that is needed is your custom Loadorder in /vayn/classes/class_index.lua.
Here you determine which files will be loaded.

```lua
local paths = {
    "C:/WGG/vayn/classes/druid/balance",
}

return paths --its important to return the table here, so it can be used to load your files!
```

Every file you load gets passed 3 Tables. The original Unlocker table containing functions supplied by the unlocker, the vayn table, giving you access to all functions inside the framework and a empty table that is shared across all your files.

# Spellbook
Vayn itself contains a spell book already, used for the integrated rotations. You can create your own:

```lua
local WGG, vayn, myProject = ...

myProject.spellBook = {
    rogue = {
        assasination = {
            eviscerate = vayn.spell:New(12345)
        }
    }
}
```

You could also only do one layer, or even more splitting into categories like cc, damage, defensives and so on.
Check the spells segment later on for more info about creating spell objects.

# Callbacks
So we need some logic to decide what to do and when to cast a certain spell, right?
Thats what callbacks are for, we register our logic with the spell object our final decision will most likely always be calling :Cast() on a certain spell.

```lua
local assassination = myProject.spellBook.rogue.assassination
local target = vayn.target
local player = vayn.player

assassination.eviscerate:Callback("5ComboPoints", function(spell)
    if player.cp < 5 then return end
    return spell:Cast(target)
end)
```
You should always return out of the callback function. This tells the framework that a cast was sucessful (if it passed all checks that are hidden inside :Cast()) and no more logic needs to be performed this iteration.

You can also chain these returns with alerts. (Alerts always return true)

```lua
assassination.eviscerate:Callback("5ComboPoints", function(spell)
    if player.cp < 5 then return end
    return spell:Cast(target) and vayn.alert(spell.name, spell.id)
end)
```

# Rotation
The rotation is performed by an actor, its called "sync" in vayn. Here you call the spell objects, telling them wich callback you want to execute. (If you leave this empty, the default callback will be used. Its the same like creating a callback without giving it a name)


```lua
local assassination = myProject.spellBook.rogue.assassination
vayn.sync(671, function()
    assassination.eviscerate("5ComboPoints")
end)
```

Registering an actor requires 2 arguments, the specID for which your rotation is created and also the function that gets called on repeat. Inside this function you call the spell objects diretly, passing the Callbackname.

Why shouldn't I create all the logic in the actor instead of in the callbacks? 
The answer is simple, callbacks are checked first if the spell they are created for is even ready / castable before running the logic inside. So in complex rotations you gain a performance increase and its more readable aswell.

# Static objects
There are a few static objects we can use inside of vayn. 

vayn.player always references the player object
vayn.target always referecnes your current target
vayn.enemyHealer references the first enemy healer it can find, useful in arena - would not use it inside of battlegrounds


# Unit Lists
Like static objects there are unit lists that give you access to units that are not your current target or any other object referencable by static objects.

vayn.enemies (filtered for only players in pvp enviroment)


# Spell API

The `spell` module wraps WoW spells into **spell objects** used by rotation callbacks. Each spell knows its cooldown, range, cast rules, and optional AoE placement logic.

Spell objects are created via `vayn.spell:New(spellID, attributes)` and invoked from actor callbacks with `spell()` or `spell("callbackName")`.

---

## Quick start

```lua
-- Define in spellBook
kick = vayn.spell:New(1766, { interrupt = true, ignoreGCD = true })

-- Register a callback in an actor file
kick:Callback(function(spell)
    if not vayn.target.casting then return end
    return spell:Cast(vayn.target) and vayn.alert(spell.name, spell.id)
end)

-- Invoke from the actor tick (runs callback named "default")
kick()

-- Ground-targeted spell with smart placement
tarTrap:Callback("healer", function(spell)
    local healer = vayn.enemies.find(function(e) return e.healer end)
    if not healer then return end
    return spell:SmartAoE(healer, {
        minHits = 1,
        offsetMin = 1,
        offsetMax = 3,
        ignoreFriends = true,
        movePredTime = healer.trapTravelTime,
    })
end)

-- Simple ground cast near a unit
freeze:Callback(function(spell)
    return spell:FastAoE(vayn.target) and vayn.alert(spell.name, spell.id)
end)
```

---

## Construction

### `spell:New(id, attributes?)`

Creates a spell wrapper from a spell ID. Returns `nil` if the spell is not found in the client.

| Field | Type | Description |
|-------|------|-------------|
| `id` / `spellID` | number | Spell ID |
| `name` | string | Localized spell name from `C_Spell.GetSpellInfo` |
| `iconID` | number | Spell icon |
| `castTime` | number | Cast time in seconds |
| `info` | table | Raw `C_Spell.GetSpellInfo` result |
| `attributes` | table | Cast/placement behaviour flags (see below) |
| `callbacks` | table | Named callback functions |
| `target` | boolean | Optional macro target override |

Default attributes set in `New()`:

```lua
ignoreStun = false
ignoreControl = false
ignoreGCD = false
ignoreMoving = (castTime <= 0)   -- instant spells ignore movement by default
ignoreCasting = false
ignoreChanneling = false
ignoreLoS = false
ignoreFacing = false
ignoreDirectionalImmunity = false
ignorePreCast = false
ignoreEnemies = false
face = false
stopMoving = false
heal = false
beneficial = false
damage = false
effect = false
cc = false
interrupt = false
castByID = false
jump = false
jumpOrMove = false
heading = false
name = nil
```

Pass an `attributes` table to override any of these at creation time.



## Access model

### Properties (read-only)

Accessed without parentheses via `propertyMap`:

```lua
spell.cd
spell.known
spell.range
spell.charges
```

### Methods (callable)

Defined directly on the spell table:

```lua
spell:Cast(unit?, overwrites?)
spell:Castable(unit?, overwrites?)
spell:Callback(name?, fn)
spell:SmartAoE(unitOrPosition, overwrites?)
spell:FastAoE(unit, options?)
```

When `Cast(unit, overwrites)` receives a non-unit first argument, it treats that table as `overwrites` and casts without a target.

---

## Cooldown & charges

| Property | Returns |
|----------|---------|
| `cd` | Remaining cooldown in seconds (`math.huge` if unknown) |
| `baseCD` | Base cooldown from `GetSpellBaseCooldown` (does not work for charge spells) |
| `known` | Whether the spell is known (`IsSpellKnown`, `IsPlayerSpell`, overrides) |
| `usable` | Whether the spell is currently usable |
| `noMana` | Whether unusable due to insufficient resources |
| `cost` | Primary power cost from `C_Spell.GetSpellPowerCost` |
| `charges` | Current charges |
| `maxCharges` | Maximum charges |
| `chargesFrac` | Fractional charges including partial recharge progress |
| `nextChargeCD` | Time until the next charge is available |
| `fullRechargeTime` | Time until all charges are restored |
| `current` | Whether the spell is the current spell (`C_Spell.IsCurrentSpell`) |
| `count` | Cast count (`C_Spell.GetSpellCastCount`, when available) |

---

## Range

| Property | Returns |
|----------|---------|
| `minRange` | Minimum range from spell info |
| `maxRange` | Maximum range from spell info |
| `range` | Effective cast range: `max(minRange, maxRange)` plus player combat reach; falls back to `vayn.player.meleeRange` when both are 0 |
| `castLength` | Alias for `castTime` |

---

## Dispel flags

| Property | Returns |
|----------|---------|
| `curseDispell` | Spell ID is in `vayn.ids.dispellSpellsCurse` |
| `dispellMagic` | Spell ID is in `vayn.ids.dispellSpellsMagic` |
| `poisonDispell` | Spell ID is in `vayn.ids.dispellSpellsPoison` |

---

## Attribute properties

Each entry in `attributes` is also exposed as a readable property:

| Property | Purpose |
|----------|---------|
| `ignoreStun` / `ignoreControl` | Bypass player stun/CC checks in `Castable` |
| `ignoreMoving` | Allow casting while moving |
| `ignoreCasting` / `ignoreChanneling` | Allow casting during casts/channels |
| `ignoreLoS` / `ignoreFacing` | Skip line-of-sight and facing checks |
| `ignoreDirectionalImmunity` | Ignore directional physical immunity |
| `ignorePreCast` / `ignorePreCastWindow` | Cast even when CD is within pre-cast window |
| `ignoreGCD` | GCD-related queue handling in `Cast` |
| `face` / `heading` / `stopMoving` | Pre-cast movement/facing behaviour |
| `heal` / `beneficial` / `damage` / `effect` | Target validation (`"physical"` / `"magic"` for damage/effect) |
| `cc` | CC type string for immunity checks (`"stun"`, `"root"`, `"polymorph"`, etc.) |
| `interrupt` | Interrupt spell — requires target casting/channeling |
| `castByID` | Use `CastSpellByID` instead of `CastSpellByName` |
| `jump` / `jumpOrMove` | Jump before casting |
| `cancelCatForm` | Cancel Cat Form when rooted/slowed before casting |
| `ignoreEnemies` | SmartAoE: skip enemy hit counting |
| `radius` / `minHits` / `maxHits` / `offsetMin` / `offsetMax` | AoE placement parameters |

---

## Callbacks

### `spell:Callback(name?, callback)`

Registers a function invoked when the spell is called. If `name` is omitted, registers as `"default"`.

### `spell:Callbacks(callbacks)`

Bulk-registers a table of `{ name = fn, ... }`.

### `spell(name?, ...)`

Callbacks are smart, they only run if the spell is even usable.
---

## Casting

### `spell:Cast(unit?, overwrites?)`

Executes a targeted cast after `Castable` checks pass.

**Pre-cast behaviour** (controlled by attributes and `overwrites`):

- `stopMoving` — calls `vayn:ControlMoving(duration)` (default 0.5s, or numeric override)
- `jump` / `jumpOrMove` — calls `vayn:JumpApex()`
- `ignoreCasting` — stops current cast via `SpellStopCasting()`
- `cancelCatForm` — cancels Cat Form when rooted/slowed
- `castByID` — uses `CastSpellByID(spellID, unitToken)`
- `attributes.name` / `overwrites.name` — casts by alternate spell name
- otherwise — `vayn.CastSpellByName(name, unitToken)`

Cast attempts are rate-limited per spell ID via an internal delay (0.25–0.50s).

Returns `true` on cast attempt, `false`/`nil` on failure.

### `spell:Castable(unit?, overwrites?)`

Returns whether the spell can be cast right now. Merges `self.attributes` with `overwrites` for the check.

**Player checks:**

- Not stunned (unless `ignoreStun` / `ignoreControl`)
- Not in hard CC
- Not moving (unless `ignoreMoving`, `stopMoving`, or `canCastWhileMoving`)
- Not casting/channeling beyond pre-cast window

**Target checks** (when `unit` is provided):

- Range (uses `overwrites.range` or `self.range`)
- Not dead (unless `ignoreDead` in overwrites)
- CC spells cannot target totems
- `damage` → enemy only; `heal` / `beneficial` → friend only
- Searing Glare blocks damage/CC/effect (except interrupts)
- Line of sight and facing
- Directional, physical/magic damage/effect immunities
- CC-type immunities matching `attributes.cc`
- Heal/beneficial/interrupt immunities

Returns `true` when all checks pass.

---

## AoE casting

Ground-targeted spells use methods from the AoE extension modules.

### `spell:SmartAoE(unitOrPosition, overwrites?)`

Finds an optimal ground position and casts there.

1. Validates input is a table (unit or `{x, y, z}`)
2. If input is a **unit** (`stunDR` present), runs `Castable` and resolves to a predicted/actual position
3. Calls `GetSmartAoEPosition` to search candidate positions
4. Calls `AoECast` at the best result

**Placement parameters** (from attributes or `overwrites`):

| Key | Default | Description |
|-----|---------|-------------|
| `radius` | `8` | Hit evaluation radius |
| `offsetMin` | `0.5` | Min distance from anchor |
| `offsetMax` | `20` | Max distance from anchor |
| `distanceSteps` | `18` | Distance search granularity |
| `circleSteps` | `10` | Angular search granularity |
| `minHits` / `maxHits` | `0` / `∞` | Required hit count range |
| `movePredTime` | `0` | Unit position prediction time |
| `sort` | hits desc | Custom sort function for candidates |
| `ignoreHitCount` | `false` | Skip hit evaluation |
| `ignoreEnemies` / `ignoreFriends` | `false` | Limit which unit lists are counted |
| `filter` | nil | `(unit) → bool` hit filter |

Results are cached for 0.5s under `"SmartAoEPosition_<name>"`.

### `spell:AoECast(position, overwrites?)`
Raw spell cast at given position.

### `spell:FastAoE(unit, options?)`

Lightweight alternative to `SmartAoE` for single-target ground placement near a unit.

1. Runs `Castable(unit)` when input is a unit
2. Resolves unit to position (with optional `options.movePredTime` prediction)
3. Picks a random offset within `radius / 2`
4. Validates: target still in radius, player LoS, player in spell range
5. Calls `AoECast` on success

| Option | Default | Description |
|--------|---------|-------------|
| `radius` | `self.radius` or `5` | Placement variance and target check radius |
| `range` | `self.range` or `0` | Max cast range from player |
| `movePredTime` | nil | Unit position prediction |

Retries up to 10 times with uncached random offsets.

---

## Full property index

<details>
<summary>52 properties (click to expand)</summary>

baseCD
beneficial
cancelCatForm
castByID
castLength
cc
charges
chargesFrac
cost
count
cd
curseDispell
current
damage
dispellMagic
effect
face
fullRechargeTime
heading
heal
ignoreCasting
ignoreChanneling
ignoreControl
ignoreDirectionalImmunity
ignoreEnemies
ignoreFacing
ignoreGCD
ignoreLoS
ignoreMoving
ignorePreCast
ignorePreCastWindow
ignoreStun
interrupt
jump
jumpOrMove
known
maxCharges
maxHits
minHits
minRange
maxRange
nextChargeCD
noMana
offsetMax
offsetMin
poisonDispell
radius
range
stopMoving
usable

</details>

---

## Full method index

<details>
<summary>Core + AoE methods (click to expand)</summary>

Callback
Callbacks
Cast
Castable
New
SmartAoE
GetSmartAoEPosition
EvaluateAoEPosition
AoECast
ClickGround
FastAoE

</details>

---


---

## Notes

- **`vayn.preCastWindow`**: Spells with CD remaining within this window can still be queued/cast depending on attributes.
- **Overwrites**: Both `Cast` and AoE methods accept a temporary attribute override table; methods restore original values after casting.
- **Debug**: Set `vayn.print_cast_attempts = true` to log cast attempts; failed `Castable` checks log via `vayn.debug = true` (Can also take a search string).
---

# Unit API

The `unit` module (`frame/unit.lua`) is the core object wrapper around WoW game objects in vayn. Every entity the rotation logic cares about — players, NPCs, pets, totems, and game objects — is represented as a **unit** instance with a rich property and method API.

Global shortcuts like `vayn.player`, `vayn.target`, and list members (`vayn.enemies`, `vayn.fgroup`, etc.) all resolve to unit instances.

---

## Quick start

```lua
-- Player health as a percentage (0–100)
if vayn.player.hp < 50 then
    -- heal logic
end

-- Check a debuff on target by spell ID or name
if vayn.target.debuff(703) then          -- Garrote by ID
    local remains = vayn.target.debuffRemains(703)
end

-- Distance and line of sight relative to the player
if vayn.target.distance < 8 and vayn.target.los then
    -- in range and visible
end

-- CC state
if vayn.target.stun then
    local dr = vayn.target.stunDR          -- diminishing returns multiplier
    local left = vayn.target.stunRemains     -- seconds remaining
end
```

---

## Construction

### `unit:New(object)`

Creates a new unit wrapper around a WGG object handle, WoW unit token (`"player"`, `"target"`, …), or GUID string.

| Field | Type | Description |
|-------|------|-------------|
| `object` | any | Underlying WGG object handle |
| `wgg_guid` | string | WGG object GUID |
| `wow_guid` | string | WoW API GUID (when available) |
| `creationTime` | number | `vayn.time` when the unit was created |
| `name` | string | Cached object name |
| `buffs` / `debuffs` / `hiddenAuras` | table | Aura caches with `instanceLookup` and `lastRefresh` |
| `cooldowns` | table | Tracked spell cooldown timestamps |
| `lockouts` | table | School lockout tracking |
| `*DRValue` / `last*DR` | number | Diminishing-return state per CC category |

### Metamethods

| Method | Behavior |
|--------|----------|
| `unit:__eq(other)` | Equality via `self.isUnit(other)` |
| `unit:__tostring()` | Returns `"unit(<name>)"` |

---

## Access model

Unit members are resolved through a custom `__index` metamethod. There are three access styles:

### 1. Properties (read-only)

Accessed without parentheses. Implemented in `propertyMap`.

```lua
vayn.target.hp
vayn.target.enemy
vayn.target.casting
```

When the underlying object no longer exists, many properties return safe defaults from `defaultReturns` (e.g. `hp = 100`, `distance = math.huge`).

### 2. Methods (callable)

Accessed with parentheses. Implemented in `functionMap`. The metamethod wraps them so extra arguments are forwarded.

```lua
vayn.target.buff(45438)                    -- Ice Block
vayn.target.distanceTo(vayn.player)
vayn.enemies.within(40).find(function(u) return u.viableEnemy end)
```

When the object is missing, method calls return safe stubs (e.g. `buffRemains()` → `0`, `distanceTo()` → `math.huge`).

### 3. Direct instance methods

Defined on the unit table itself:

| Method | Description |
|--------|-------------|
| `unit:New(object)` | Constructor |
| `unit:Interact()` | Calls `ObjectInteract` if the unit exists |
| `unit:SetTarget()` | Targets the unit via `TargetUnit` if not already targeted |

### Special resolved keys

| Key | Description |
|-----|-------------|
| `guid` / `wgg_guid` / `wow_guid` | GUID accessors with lazy caching |
| `omToken` | WGG object token used for WoW API calls |

---

## Identity & existence

| Property | Returns |
|----------|---------|
| `exists` | Whether the WGG object still exists |
| `valid` | `omToken` is set and object exists |
| `alive` / `dead` | Life state (Feign Death on hunters is treated as alive) |
| `name` | Object name, or `"Unknown"` |
| `id` | NPC/object ID |
| `guid` | WGG GUID |
| `type` / `typeName` | Object type index and label (`Player`, `Unit`, `GameObject`, …) |
| `omToken` | Token string for WoW API |
| `uptime` / `existsSince` | Seconds since unit wrapper was created |
| `player` | Whether the unit is a player character |
| `pet` / `battlePet` / `totem` | Pet-related checks |
| `dummy` | Whether NPC ID is in `vayn.ids.dummies` |
| `level` | Unit level |

---

## Faction & targeting

| Property | Returns |
|----------|---------|
| `enemy` | Can the player attack this unit? |
| `friend` | Is this unit friendly to the player? |
| `target` | Unit wrapper for this unit's target |
| `creator` | Unit wrapper for the object's creator |
| `viable` / `viableEnemy` / `viableFriend` | Combat-viability filters (LoS, alive, player-only checks) |
| `combat` / `combatTime` | In-combat state and duration |
| `los` | Line of sight from player to this unit (`vayn.player.losTo(self)`) |

### Methods

| Method | Description |
|--------|-------------|
| `isUnit(other)` | Same unit comparison |
| `isTarget()` | Whether this unit is the current target |
| `enemyTo(other)` / `friendTo(other)` | Faction relative to another unit |
| `losTo(other)` / `losToRaw(other)` | Line of sight to another unit |
| `losToPosition(pos)` / `losToPositionRaw(pos)` | LoS to `{x, y, z}` |
| `losOf(other)` | Inverse of `other.losTo(self)` |
| `icewallObstructingLosTo(unit)` | Whether an Ice Wall blocks LoS |
| `smokebombObstructingLosTo(unit)` | Whether a Smoke Bomb blocks LoS |
| `predictLos(other, time)` | Predicted LoS after `time` seconds |

---

## Position, movement & range

| Property | Returns |
|----------|---------|
| `position` | `{x, y, z}` table |
| `positionRaw` | Raw `x, y, z` from `ObjectPosition` |
| `x` / `y` / `z` | Individual coordinates |
| `rotation` | Facing angle (radians) |
| `distance` | 3D distance from player |
| `distance2D` | 2D distance from player |
| `combatReach` | Object combat reach |
| `meleeRange` | Calculated melee range including reach and lag buffer |
| `speed` / `maxSpeed` | Current and run speed |
| `moving` | `speed > 0` |
| `timeStandingStill` | Seconds standing still |
| `altitude` | Height above ground |
| `timeToUnit` / `timeToUnitRaw` | Estimated intercept time from player |
| `trapTravelTime` | Estimated trap travel time at 19.281 yds/s |
| `averageRange` | Spec-based average engagement range |

### Movement flags

Boolean properties backed by `MovementFlags`: `movingForward`, `movingBackward`, `stravingLeft`, `stravingRight`, `turningLeft`, `turningRight`, `falling`, `fallingFar`, `swimming`, `flying`, `canFly`, `ascending`, `descending`, `levitating`, `onTransport`, and their `*Pending` variants.

| Method | Description |
|--------|-------------|
| `distanceTo(unit)` / `distance2DTo(unit)` | Distance to another unit |
| `distanceToPosition(pos)` / `distance2DToPosition(pos)` | Distance to a point |
| `predictDistance(time)` / `predictDistance2D(time)` | Predicted distance after `time` |
| `predictDistanceTo(unit, time)` | Predicted distance to unit |
| `predictPosition(time)` | Predicted `{x, y, z}` |
| `movingTowards(unit)` / `movingAwayFrom(unit)` | Movement direction relative to unit |
| `hasMovementFlag(flag)` | Test a `MovementFlags` value |
| `facing(unit, options?)` | Whether `unit` is within facing cone (default 180°) |
| `facing45(unit)` / `facing90(unit)` | 45° / 90° cone variants |
| `behind(unit)` | Whether this unit is behind `unit` |
| `angleTo(unit)` | Angle to another unit |
| `setFace(unit?)` / `SetFacing(unit?)` | Face a unit or numeric angle |

---

## Health & absorbs

| Property | Returns |
|----------|---------|
| `health` | Raw health |
| `maxHealth` | Maximum health |
| `hp` | Health percentage (0–100) |
| `hpa` | Effective health percentage including absorbs minus heal absorbs |
| `realHealth` | `health - healAbsorb + absorb` |
| `healthMissing` / `hpMissing` / `realHealthMissing` | Missing health variants |
| `healAbsorb` | Total heal absorption |
| `absorb` | Total damage absorption shields |
| `guardianSpirit` | Guardian Spirit buff (with heal-event tracking) |

---

## Casting & channels

| Property | Returns |
|----------|---------|
| `casting` | Cast spell name, or nil |
| `castID` | Cast spell ID |
| `castTimeRemains` / `castTimeComplete` | Cast timing (seconds) |
| `castPct` / `castPctRemains` | Cast progress percentage |
| `channeling` | Channel spell name, or `false` |
| `channelID` | Channel spell ID |
| `channelTimeRemains` / `channelTimeComplete` | Channel timing |
| `channelPct` / `channelPctRemains` | Channel progress percentage |
| `castTarget` | Unit being cast at |
| `canCastWhileMoving` | Has a cast-while-moving buff |
| `lastCast` | Last recorded cast spell ID |
| `gcd` | Remaining GCD (**player only**) |
| `maxGCD` | Maximum GCD for this unit's spec/haste |

### Methods

| Method | Description |
|--------|-------------|
| `recentlyCast(spellID, window?)` | Cast within time window |
| `recentlyCastTime(spellID)` | Time since cast |
| `cooldown(spellID)` | Remaining tracked cooldown from `vayn.ids.trackedCDs` |
| `canCastOverlappingSpell()` / `setOverlappingSpell()` | Overlapping cast window helpers |

---

## Crowd control

CC is detected via aura tables in `vayn.ids.cc`. Most CC properties return an **aura object** (truthy) or `nil`/`false`.

### CC categories

| Property | Type |
|----------|------|
| `stun` / `incap` / `disorient` / `fear` / `horror` / `mindcontrol` | Hard CC |
| `cyclone` / `sleep` / `charmed` | Special CC |
| `disarm` / `root` / `slow` / `silence` | Soft CC |
| `bcc` | Breakable CC (incap, breakable stun/disorient/fear/horror) |
| `vbcc` | Valuable breakable CC |
| `hardcc` | Non-breakable hard CC |
| `cc` | Role-aware CC summary (varies by caster/healer/melee) |
| `longestCC` | Longest active CC aura |
| `incomingCC` / `incomingCCRemains` | Predicted incoming CC from cast target |

Each category has matching `*Remains`, `*Uptime`, and (where applicable) `*DR` / `*DRRemains` properties.

### Diminishing returns

DR is tracked per category: `stunDR`, `incapDR`, `disorientDR`, `fearDR`, `horrorDR`, `cycloneDR`, `disarmDR`, `rootDR`, `silenceDR`, `knockbackDR` and their `*Remains` counterparts. Values follow WoW DR tiers (1 → 0.5 → 0.25 → 0).

| Property | Description |
|----------|-------------|
| `stunBreak` | Whether a stun break effect is active |
| `safe` | Unit is in a "safe" state (major defensives, CC, etc.) |

---

## Immunities

Immunities combine aura checks from `vayn.ids.immunities`, DR state, and untouchable CC debuffs. Each immunity has `*Remains` and `*Uptime` variants.

| Property | Checks |
|----------|--------|
| `physicalDamageImmunity` | Physical damage immunity |
| `magicDamageImmunity` | Magic damage immunity |
| `physicalEffectImmunity` | Physical effect immunity |
| `magicEffectImmunity` | Magic effect immunity |
| `directionalPhysicalDamageImmunity` | Directional physical immunity (e.g. Fists of Fury) |
| `stunImmunity` / `incapImmunity` / `polymorphImmunity` | CC immunities |
| `disorientImmunity` / `fearImmunity` | Disorient/fear immunities |
| `disarmImmunity` / `rootImmunity` / `slowImmunity` | Soft CC immunities |
| `silenceImmunity` / `interruptImmunity` / `knockbackImmunity` | Cast/interrupt immunities |
| `healImmunity` / `beneficialImmunity` | Healing/beneficial immunities |
| `untouchableCC` | Untouchable CC debuff |
| `bccImmunity` | BCC immunity (Sac, SW:D, etc.) |

Many aliases exist (see [Aliases](#aliases)).

---

## Auras

Aura lookups are cached per unit with a randomized refresh interval (`auraCacheTime`). Buff/debuff scans use `AuraUtil.ForEachAura`; hidden auras use `WGG.GetAllPrivateAuras`.

### Methods

| Method | Signature | Returns |
|--------|-----------|---------|
| `buff(id, creator?, uptime?)` | Spell ID or lowercase name | Longest matching aura object |
| `debuff(id, creator?, uptime?)` | Spell ID or lowercase name | Longest matching aura object |
| `buffFrom(table, creator?, uptime?)` | Table of spell IDs | First matching buff |
| `debuffFrom(table, creator?, uptime?)` | Table of spell IDs | First matching debuff |
| `buffRemains(id, creator?)` | | Seconds remaining |
| `debuffRemains(id, creator?)` | | Seconds remaining |
| `buffUptime(id, creator?)` | | Seconds active |
| `debuffUptime(id, creator?)` | | Seconds active |
| `buffStacks(id, creator?)` | | Stack count |
| `debuffStacks(id, creator?)` | | Stack count |
| `hiddenAura(id)` | Hidden/private aura ID | Aura table |
| `hiddenAuraFrom(table)` | Table of IDs | First matching hidden aura |
| `hiddenAuraRemains(id)` / `hiddenAuraUptime(id)` | | Timing helpers |

Aura objects (from `vayn.aura:New`) expose `.remains`, `.uptime`, `.stacks`, `.creator`, `.id`, etc.

### Notable aura-based properties

| Property | Description |
|----------|-------------|
| `stealth` | Stealth buff from `vayn.ids.stealth` |
| `bloodlust` | Bloodlust/Heroism |
| `trinket` / `trinketRemains` | PvP trinket availability |
| `humanRacial` / `humanRacialRemains` | Will to Survive tracking |
| `defensiveCDs` / `majorDefensiveCDs` / `offensiveCDs` | CD aura groups (+ `*Uptime`) |
| `purgable` | Stealable buff |
| `dotted` | Has a bleed or DoT from `vayn.ids` |
| `ams` | Anti-Magic Shell |
| `smokeBomb` / `speared` / `searingGlare` | Arena-specific debuff checks |

---

## Power & resources

Generic power methods:

| Method | Description |
|--------|-------------|
| `power(type)` | Current power for `POWER_TYPES` index |
| `powerMax(type)` | Maximum power |
| `powerPct(type)` | Percentage (0–100) |
| `powerDeficit(type)` | Missing power |

Each resource type also has dedicated properties: `<resource>`, `<resource>Max`, `<resource>Pct`, `<resource>Deficit`.

| Resource | Properties |
|----------|------------|
| Mana | `mana`, `manaMax`, `manaPct`, `manaDeficit` |
| Rage | `rage`, … |
| Focus | `focus`, … |
| Energy | `energy`, … |
| Combo Points | `comboPoints`, `chargedComboPoints`, … |
| Runes | `runes`, `runesMax`, `runesPct`, `runesDeficit` |
| Runic Power | `runicPower`, … |
| Soul Shards | `soulShards`, `predictedSoulShards`, … |
| Astral Power | `astralPower`, … |
| Holy Power | `holyPower`, … |
| Maelstrom | `maelstrom`, … |
| Chi | `chi`, … |
| Insanity | `insanity`, … |
| Arcane Charges | `arcaneCharges`, … |
| Fury | `fury` (DH or warrior context) |
| Pain | `pain`, … |
| Essence | `essence`, … |
| Death Knight runes | `runeBlood`, `runeFrost`, `runeUnholy` (+ Max/Pct/Deficit) |

Player-only: `powerRegen`, `basePowerRegen`, `combatPowerRegen`, `mounted`, `gcd`.

---

## Class, spec & role

### Class checks

Boolean properties: `warrior`, `paladin`, `hunter`, `rogue`, `priest`, `deathKnight`, `shaman`, `mage`, `warlock`, `monk`, `druid`, `demonHunter`, `evoker`.

`class1` / `class2` / `class3` return class index, English token, and localized name.

### Spec checks

Boolean properties for every PvP-relevant spec, e.g. `armsWarrior`, `frostMage`, `restorationDruid`, `subtletyRogue`, `havocDemonhunter`, etc.

| Property | Description |
|----------|-------------|
| `specID` | Specialization ID |
| `specName` / `specNameShort` | Full and short spec names |
| `specColor` | Spec color from `vayn.ids.specs` |

### Role checks

| Property | Description |
|----------|-------------|
| `healer` / `dps` / `tank` | Spec-based role from `vayn.ids.specs` |
| `melee` / `ranged` / `caster` | Range archetype |
| `magicDPS` / `physicalDPS` | Damage type |
| `healerRole` / `damageRole` / `tankRole` | WoW group role assignment |
| `inGroup` / `raid` / `party` | Group membership |

### Race checks

`race` returns the race name. Individual booleans: `human`, `orc`, `nightElf`, `bloodElf`, `undead`, `dwarf`, `gnome`, `draenei`, `worgen`, `pandaren`, `tauren`, `troll`, `goblin`, `voidElf`, `lightforgedDraenei`, `darkIronDwarf`, `kulTiran`, `mechagnome`, `nightborne`, `highmountainTauren`, `magharOrc`, `zandalariTroll`, `vulpera`, `dracthyr`.

---

## Attackers & PvP utility

| Property | Description |
|----------|-------------|
| `attackers` | Total DPS attackers on this unit |
| `meleeAttackers` | Melee attackers |
| `rangedAttackers` | Ranged attackers |
| `cooldownAttackers` | Attackers with offensive CDs active |

| Method | Description |
|--------|-------------|
| `v2Attackers(option?, ignoreRange?)` | Detailed attacker counts. Options: `"totalUnits"`, `"meleeUnits"`, `"rangedUnits"`, `"cooldowns"`, `"magic"`. Returns count or `(melee, ranged, cds, total)` |

| Property | Description |
|----------|-------------|
| `fc` / `hasFlag` | Flag carrier |
| `orbOfPower` | Orb of Power buff |
| `capping` / `cappingRemains` | Node capping hidden aura |
| `drinking` | Drinking regen buff |

---

## World objects

Properties for non-combat interactions:

| Property | Description |
|----------|-------------|
| `gatherable` | Herb/ore node not depleted |
| `lootable` | Can be looted |
| `skinnable` | Can be skinned |
| `tapDenied` | Tap denied on mob |
| `creatureType` | `"Humanoid"`, `"Beast"`, etc. |
| `humanoid` / `beast` / `critter` / `undead` | Creature type shortcuts |
| `outdoors` | Currently always `false` (placeholder) |
| `locked` | Placeholder, always `false` |

---

## Talents & totems

| Method | Description |
|--------|-------------|
| `hasTalent(talentID)` | Whether the unit has a talent |
| `hasTotem(totemID)` | Whether the unit has a totem |

Alias: `talent` → `hasTalent`.

---

## Aliases

The module registers **189 property aliases** and several method aliases for backwards compatibility and shorthand. Aliases point to the canonical property/method and behave identically.

Common examples:

| Alias | Canonical |
|-------|-----------|
| `healthPercent` | `hp` |
| `realHealthPercent` | `hpa` |
| `dist` / `distance2d` | `distance` / `distance2D` |
| `cp` / `cpMax` / `cpPct` | `comboPoints` / `comboPointsMax` / `comboPointsPct` |
| `stunned` / `feared` / `rooted` / `silenced` | `stun` / `fear` / `root` / `silence` |
| `gcdRemains` | `gcd` |
| `cds` | `offensiveCDs` |
| `token` | `omToken` |
| `friendly` | `friend` |
| `source` | `creator` |
| `isFC` / `isDummy` | `fc` / `dummy` |
| `dk` / `frostdk` / `sub` / `disc` | class/spec shortcuts |

Method aliases:

| Alias | Canonical |
|-------|-----------|
| `distanceToUnit` | `distanceTo` |
| `distanceToPoint` | `distanceToPosition` |
| `movingTo` / `isMovingTowards` | `movingTowards` |
| `recentlyUsed` / `used` | `recentlyCast` |
| `v2attackers` | `v2Attackers` |
| `predictLoS` | `predictLos` |

Accessing an unknown property throws: `Unit property <name> not found`.

---

## Full property index

<details>
<summary>476 properties (click to expand)</summary>

absorb
afflictionWarlock
alive
altitude
ams
arcaneCharges
arcaneChargesDeficit
arcaneChargesMax
arcaneChargesPct
arcaneMage
armsWarrior
ascending
assassinationRogue
astralPower
astralPowerDeficit
astralPowerMax
astralPowerPct
attackers
augmentationEvoker
auraInfo
averageRange
balanceDruid
base
basePowerRegen
baseRegen
battlePet
bcc
bccImmunity
bccImmunityRemains
bccRemains
beast
beastmasteryHunter
beneficialImmunity
beneficialImmunityRemains
beneficialImmunityUptime
bloodDeathknight
bloodElf
bloodlust
brewmasterMonk
canCastWhileMoving
canFly
capping
cappingRemains
caster
castID
casting
castPct
castPctRemains
castTarget
castTimeComplete
castTimeRemains
cc
ccRemains
ccUptime
channelID
channeling
channelName
channelNotInterruptible
channelPct
channelPctRemains
channelTimeComplete
channelTimeRemains
chargedComboPoints
chargedCPs
charmed
charmedRemains
charmedUptime
chi
chiDeficit
chiMax
chiPct
class1
class2
class3
combat
combatPowerRegen
combatReach
combatRegen
combatTime
comboPoints
comboPointsDeficit
comboPointsMax
comboPointsPct
cooldownAttackers
creator
creatureType
critter
cyclone
cycloneDR
cycloneDRRemains
cycloneRemains
cycloneUptime
damageRole
darkIronDwarf
dead
deathKnight
defensiveCDs
defensiveCDsUptime
demonHunter
demonologyWarlock
descending
destructionWarlock
devastationEvoker
directionalPhysicalDamageImmunity
directionalPhysicalDamageImmunityRemains
directionalPhysicalDamageImmunityUptime
disarm
disarmDR
disarmDRRemains
disarmImmunity
disarmImmunityRemains
disarmImmunityUptime
disarmRemains
disarmUptime
disciplinePriest
disorient
disorientDR
disorientDRRemains
disorientImmunity
disorientImmunityRemains
disorientImmunityUptime
disorientRemains
disorientUptime
distance
distance2D
dotted
dps
dracthyr
draenei
drDuration
drinking
druid
dummy
duration
dwarf
elementalShaman
endTime
endTimeMS
enemy
enemyTo
energy
energyDeficit
energyMax
energyPct
enhancementShaman
essence
essenceDeficit
essenceMax
essencePct
evoker
exists
existsSince
falling
fallingFar
fc
fear
fearDR
fearDRRemains
fearImmunity
fearImmunityRemains
fearImmunityUptime
fearRemains
fearUptime
feralDruid
fireMage
flying
focus
focusDeficit
focusMax
focusPct
friend
friendTo
frostDeathknight
frostMage
fury
furyDeficit
furyMax
furyPct
furyWarrior
gatherable
gcd
gnome
goblin
guardianDruid
guardianSpirit
guid
hardcc
hasFlag
haste
havocDemonhunter
healAbsorb
healer
healerRole
healImmunity
healImmunityRemains
healImmunityUptime
health
healthMissing
highmountainTauren
holyPaladin
holyPower
holyPowerDeficit
holyPowerMax
holyPowerPct
holyPriest
horror
horrorDR
horrorDRRemains
horrorRemains
horrorUptime
hp
hpa
hpMissing
human
humanoid
humanRacial
humanRacialRemains
hunter
id
incap
incapDR
incapDRRemains
incapImmunity
incapImmunityRemains
incapImmunityUptime
incapRemains
incapUptime
incomingCC
incomingCCRemains
inGroup
insanity
insanityDeficit
insanityMax
insanityPct
interruptImmunity
interruptImmunityRemains
interruptImmunityUptime
knockbackDR
knockbackDRRemains
knockbackImmunity
knockbackImmunityRemains
knockbackImmunityUptime
kulTiran
lastCast
lastTrinketUse
level
levitating
lightforgedDraenei
locked
longestCC
lootable
los
maelstrom
maelstromDeficit
maelstromMax
maelstromPct
mage
magharOrc
magicDamageImmunity
magicDamageImmunityRemains
magicDamageImmunityUptime
magicDPS
magicEffectImmunity
magicEffectImmunityRemains
magicEffectImmunityUptime
majorDefensiveCDs
majorDefensiveCDsUptime
mana
manaDeficit
manaMax
manaPct
marksmanshipHunter
maxGCD
maxHealth
maxRemaining
maxSpeed
mechagnome
melee
meleeAttackers
meleeRange
mindcontrol
mindcontrolRemains
mindcontrolUptime
mistweaverMonk
monk
mounted
movementFlag
moving
movingBackward
movingBackwardPending
movingForward
movingForwardPending
name
nightborne
nightElf
notInterruptible
offensiveCDs
offensiveCDsUptime
omToken
onTransport
orbOfPower
orc
outdoors
outlawRogue
pain
painDeficit
painMax
painPct
paladin
pandaren
party
pet
physicalDamageImmunity
physicalDamageImmunityRemains
physicalDamageImmunityUptime
physicalDPS
physicalEffectImmunity
physicalEffectImmunityRemains
physicalEffectImmunityUptime
player
polymorphImmunity
polymorphImmunityRemains
polymorphImmunityUptime
position
positionRaw
powerRegen
powerType
predictedSoulShards
preservationEvoker
priest
protectionPaladin
protectionWarrior
purgable
race
racialRemaining
rage
rageDeficit
rageMax
ragePct
raid
ranged
rangedAttackers
ready
realHealth
realHealthMissing
restorationDruid
restorationShaman
retributionPaladin
rogue
role
root
rootDR
rootDRRemains
rootImmunity
rootImmunityRemains
rootImmunityUptime
rootRemains
rootUptime
rotation
runeBlood
runeBloodDeficit
runeBloodMax
runeBloodPct
runeFrost
runeFrostDeficit
runeFrostMax
runeFrostPct
runes
runesDeficit
runesMax
runesPct
runeUnholy
runeUnholyDeficit
runeUnholyMax
runeUnholyPct
runicPower
runicPowerDeficit
runicPowerMax
runicPowerPct
safe
searingGlare
shadowPriest
shaman
sharedRemaining
silence
silenceDR
silenceDRRemains
silenceImmunity
silenceImmunityRemains
silenceImmunityUptime
silenceRemains
silenceUptime
skinnable
sleep
sleepRemains
sleepUptime
slow
slowImmunity
slowImmunityRemains
slowImmunityUptime
slowRemains
slowUptime
smokeBomb
soulShards
soulShardsDeficit
soulShardsMax
soulShardsPct
speared
specColor
specID
specName
specNameShort
speed
spellId
start
startTimeMS
staticField
stealth
straving
stravingLeft
stravingLeftPending
stravingRight
stravingRightPending
stun
stunBreak
stunDR
stunDRRemains
stunImmunity
stunImmunityRemains
stunImmunityUptime
stunRemains
stunUptime
subtletyRogue
survivalHunter
swimming
tank
tankRole
tapDenied
target
tauren
timeStandingStill
timeToUnit
timeToUnitRaw
totem
trapTravelTime
trinket
trinketRemains
troll
turning
turningLeft
turningRight
type
typeName
undead
unholyDeathknight
untouchableCC
untouchableCCRemains
untouchableCCUptime
uptime
valid
vbcc
vbccRemains
vengeanceDemonhunter
viable
viableEnemy
viableFriend
voidElf
vulpera
warlock
warrior
windwalkerMonk
worgen
x
y
z
zandalariTroll

</details>

---

## Full method index

<details>
<summary>58 methods (click to expand)</summary>

angleTo
behind
buff
buffFrom
buffRemains
buffStacks
buffUptime
canCastOverlappingSpell
cooldown
debuff
debuffFrom
debuffRemains
debuffStacks
debuffUptime
distance2DTo
distance2DToPosition
distanceTo
distanceToPosition
enemyTo
facing
facing45
facing90
friendTo
hasMovementFlag
hasTalent
hasTotem
hiddenAura
hiddenAuraFrom
hiddenAuraRemains
hiddenAuraUptime
icewallObstructingLosTo
icewallObstructingLosToPosition
isTarget
isUnit
losOf
losTo
losToPosition
losToPositionRaw
losToRaw
movingAwayFrom
movingTowards
power
powerDeficit
powerMax
powerPct
predictDistance
predictDistance2D
predictDistance2DTo
predictDistanceTo
predictLos
predictPosition
recentlyCast
recentlyCastTime
setFace
SetFacing
setOverlappingSpell
smokebombObstructingLosTo
v2Attackers

</details>

---

## Related modules

| Module | Role |
|--------|------|
| `frame/unitManager.lua` | Unit cache, GUID/token resolution, `get()` |
| `frame/lists.lua` | Enemy/friendly lists yielding units |
| `vayn.aura` | Aura object returned by `buff()` / `debuff()` |
| `vayn.ids` | CC, immunity, spec, and tracked CD tables |

---

## Notes

- **Player-only APIs** (`gcd`, `mounted`, `powerRegen`) call `vayn.error` or `error()` when used on non-player units.
- **Secret values**: WoW secret values/tables are unwrapped via `vayn.unwrap` or rejected with a console warning.
- **Caching**: Aura and DR lookups use `vayn.cache` with GUID-scoped keys to avoid repeated API scans within a frame.
- **Non-existent objects**: The `__index` fallback returns safe defaults so rotation code can read properties without nil-checking every field.
