
## Getting Started

### File Structure

Your project needs a custom load order at:

`/vayn/classes/class_index.lua`

This file returns the paths that should be loaded.

```lua
local paths = {
    "C:/WGG/vayn/classes/druid/balance",
}

return paths
```

> **Important:** The file must return the `paths` table so Vayn can load the files.

Each loaded file receives three tables:

1. `WGG` — the original Unlocker table and its supplied functions.
2. `vayn` — access to the Vayn framework.
3. `Your project table` — shared by all files in the project.

### File Headers

A simple header gives each file access to the common objects it needs:

```lua
local WGG, vayn, myProject = ...

if vayn.player.class2 ~= "ROGUE" then
    return "skipped"
end

local player = vayn.player
local target = vayn.target
local assassination = myProject.spellBook.rogue.assassination
```

This keeps the rest of the file concise. Returning early also prevents class-specific code from running for other classes.

### Spellbook

Create your own spellbook instead of repeatedly constructing spell objects throughout the rotation:

```lua
local WGG, vayn, myProject = ...

myProject.spellBook = {
    rogue = {
        assassination = {
            eviscerate = vayn.spell:New(12345),
        },
    },
}
```

The structure is up to you. You can use a single level, split by specialization, or add categories such as `damage`, `cc`, and `defensives`.

See [Spell API](#spell-api) for spell creation and configuration.

### Callbacks

Callbacks contain the conditions for using a spell. The rotation itself can then stay small and simply call the appropriate callback.

```lua
local assassination = myProject.spellBook.rogue.assassination
local player = vayn.player
local target = vayn.target

assassination.eviscerate:Callback("5ComboPoints", function(spell)
    if player.cp < 5 then return end

    return spell:Cast(target)
end)
```

Callbacks should always return the result of the action. A successful `Cast()` tells the framework that the rotation has performed an action and no further logic should run for that iteration.

Alerts can be chained onto the return value:

```lua
return spell:Cast(target) and vayn.alert(spell.name, spell.id)
```

### Rotation

The rotation is driven by `vayn.sync()`. Register the specialization ID and call the spells you want checked on each iteration.

```lua
local assassination = myProject.spellBook.rogue.assassination

vayn.sync(671, function()
    assassination.eviscerate("5ComboPoints")
end)
```

The first argument is the specialization ID. The second is the function executed repeatedly.

Keeping conditions in callbacks is preferable to putting everything in the actor. A callback is only evaluated when its spell is usable/castable, which can reduce unnecessary work and keeps the rotation easier to read.

## Core Objects

### Static Objects

Vayn exposes several commonly used unit objects directly:

| Object | Description |
|---|---|
| `vayn.player` | The player unit. |
| `vayn.target` | The current target. |
| `vayn.focus` | The current focus. |
| `vayn.healer` | The first friendly healer found. Useful in arena; not recommended for battleground logic. |
| `vayn.enemyHealer` | The first enemy healer found. Useful in arena; not recommended for battleground logic. |
| `vayn.pet` | The current player pet. |

### Unit Lists

Unit lists provide access to units that are not exposed through the static objects.

| List | Description |
|---|---|
| `vayn.enemies` | all enemies in OM, limited to players in the PvP environment |
| `vayn.allEnemies` | not filtered list of enemies |
| `vayn.group` | party or raidmembers, if not in raid all friendly units |
| `vayn.triggers` | area triggers |
| `vayn.totems` | enemy totems |
| `vayn.friendlyTotems` | friendly totems |
| `vayn.pets` | enemy pets |
| `vayn.friendlyPets` | friendly pets |
| `vayn.allObjects` | everything in OM |


### Methods

| List | Description |
|---|---|
| `.within(range)` | filtered by `range` |
| `.filter(function(unit))` | filtered by `function` |
| `.sort(function(a, b))` | sort the list |
| `.lowest` | lowest hp unit |
| `.highest` | lowest hp unit |
| `.find(unit)` | filtered by `function`, only returning the first viable unit |
| `.loop(function(unit))` | perform function for every unit |
| `.stomp(function(unit, unitUptime))` | same as loop, also passing uptime to your function |
| `.around(unit, range, function())` | number of units around a certain unit, filtered via function |
| `.aroundPosition(unit, range, function())` | number of units around a certain position, filtered via function  |
| `.count` | number of units |

Methods returning a viable group (not a single unit) can be chained endlessly.
```lua
local enemy = vayn.enemies
    .within(40)
    .filter(function(unit)
        return unit.rogue
    end)
    .find(function(unit)
        return not unit.cc
    end)
```

# Spell API

The `spell` module wraps WoW spells in spell objects. A spell object handles cooldowns, range, castability, target validation, callbacks, and optional ground-targeted AoE logic.

## Quick Start

```lua
local kick = vayn.spell:New(1766, {
    interrupt = true,
    ignoreGCD = true,
})

kick:Callback(function(spell)
    if not vayn.target.casting then return end

    return spell:Cast(vayn.target)
        and vayn.alert(spell.name, spell.id)
end)

kick()
```

For ground-targeted spells:

```lua
tarTrap:Callback("healer", function(spell)
    local healer = vayn.enemies.find(function(unit)
        return unit.healer
    end)

    if not healer then return end

    return spell:SmartAoE(healer, {
        minHits = 1,
        offsetMin = 1,
        offsetMax = 3,
        ignoreFriends = true,
        movePredTime = healer.trapTravelTime,
    })
end)
```

For a simple ground cast near a unit:

```lua
freeze:Callback(function(spell)
    return spell:FastAoE(vayn.target)
        and vayn.alert(spell.name, spell.id)
end)
```

## Creating Spells

### `vayn.spell:New(id, attributes?)`

Creates a spell object from a spell ID. Returns `nil` if the spell cannot be found in the client.

The object exposes spell information, attributes, callbacks, and casting methods.

```lua
local kick = vayn.spell:New(1766, {
    interrupt = true,
    ignoreGCD = true,
})
```

### Default Attributes

```lua
ignoreStun = false
ignoreControl = false
ignoreGCD = false
ignoreMoving = (castTime <= 0)
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

Attributes can be supplied when the spell is created or temporarily overridden when casting.

## Properties

Spell properties are accessed directly:

```lua
spell.cd
spell.known
spell.range
spell.charges
spell.chargesFrac
spell.nextChargeCD
spell.fullRechargeTime
```

Useful groups include:

### Cooldown & Charges

| Property | Description |
|---|---|
| `cd` | Remaining cooldown in seconds. |
| `baseCD` | Base cooldown from `GetSpellBaseCooldown`. Not available for charge spells. |
| `known` | Whether the spell is known. |
| `usable` | Whether the spell is currently usable. |
| `noMana` | Whether the spell is unusable because of insufficient resources. |
| `cost` | Primary power cost. |
| `charges` | Current charges. |
| `maxCharges` | Maximum charges. |
| `chargesFrac` | Charges including partial recharge progress. |
| `nextChargeCD` | Time until the next charge. |
| `fullRechargeTime` | Time until all charges are restored. |
| `current` | Whether this is the current spell. |
| `count` | Cast count, when available. |

### Range

| Property | Description |
|---|---|
| `minRange` | Minimum range from spell info. |
| `maxRange` | Maximum range from spell info. |
| `range` | Effective cast range, including player combat reach. Falls back to `vayn.player.meleeRange` when spell ranges are zero. |
| `castTime` | Duration a cast needs to complete |

### Dispel Flags

| Property | Description |
|---|---|
| `curseDispell` | Spell is registered as a Curse dispel. |
| `magicDispell` | Spell is registered as a Magic dispel. |
| `poisonDispell` | Spell is registered as a Poison dispel. |

## Callbacks

### `spell:Callback(name?, callback)`

Registers a callback. Omitting `name` registers the callback as `default`.

```lua
spell:Callback(function(spell)
    return spell:Cast(vayn.target)
end)
```

Named callbacks are useful when the same spell needs different conditions:

```lua
spell:Callback("interrupt", function(spell)
    if not vayn.target.casting then return end
    return spell:Cast(vayn.target)
end)
```

Call it with:

```lua
spell("interrupt")
```

Callbacks are checked only when the spell is usable/castable.

### `spell:Callbacks(callbacks)`

Registers multiple callbacks from a table:

```lua
spell:Callbacks({
    interrupt = function(spell)
        -- ...
    },
    default = function(spell)
        -- ...
    },
})
```

## Casting

### `spell:Cast(unit?, overwrites?)`

Attempts to cast the spell after the normal castability checks pass.

Important pre-cast attributes include:

| Attribute | Behaviour |
|---|---|
| `stopMoving` | Stops movement before casting. |
| `ignoreCasting` | Stops the current cast. |
| `castByID` | Casts spell via ID. |
| `name` | Uses an alternate spell name. |

Returns `true` on a cast attempt and `false`/`nil` on failure.

### `spell:Castable(unit?, overwrites?)`

Returns whether the spell can currently be cast. Temporary `overwrites` are merged with the spell's attributes for the check.

Player checks include:

- Stun and hard CC.
- Movement restrictions.
- Current casts/channels.

Target checks include:

- Range.
- Alive/dead state.
- Friend/enemy validation.
- Line of sight and facing.
- Damage/effect immunities.
- CC immunities.
- Healing, beneficial, and interrupt immunities.

Returns `true` when all relevant checks pass.

## AoE Casting

Ground-targeted spells expose several AoE helpers.

### `spell:SmartAoE(unitOrPosition, overwrites?)`

Finds a suitable ground position and casts there.

The input can be a unit or a `{x, y, z}` position. For units, Vayn checks castability and resolves the unit to its predicted/current position before searching for a placement.

Common placement options:

| Option | Default | Description |
|---|---:|---|
| `radius` | `8` | Hit evaluation radius. |
| `offsetMin` | `0.5` | Minimum distance from the anchor. |
| `offsetMax` | `20` | Maximum distance from the anchor. |
| `distanceSteps` | `18` | Distance search granularity. |
| `circleSteps` | `10` | Angular search granularity. |
| `minHits` | `0` | Minimum required hits. |
| `maxHits` | `∞` | Maximum allowed hits. |
| `movePredTime` | `0` | Position prediction time. |
| `sort` | hits desc | Candidate sorting function. |
| `ignoreHitCount` | `false` | Skip hit evaluation. |
| `ignoreEnemies` | `false` | Exclude enemies from hit counting. |
| `ignoreFriends` | `false` | Exclude friends from hit counting. |
| `filter` | `nil` | `(unit) -> bool` hit filter. |

### `spell:AoECast(position, overwrites?)`

Performs a raw ground-targeted cast at the supplied position.

### `spell:FastAoE(unit, options?)`

A lightweight alternative to `SmartAoE` for placing a ground spell near one unit.

It:

1. Checks `Castable(unit)`.
2. Resolves the unit to a position.
3. Optionally predicts the position.
4. Tries a random offset within `radius / 2`.
5. Checks radius, line of sight, and range.
6. Calls `AoECast()` on success.

Options:

| Option | Default | Description |
|---|---:|---|
| `radius` | `self.radius` or `5` | Placement variance and target radius. |
| `range` | `self.range` or `0` | Maximum cast range. |
| `movePredTime` | `nil` | Position prediction time. |

It retries up to 10 times using uncached random offsets.

## Temporary Overrides

`Cast()` and AoE methods accept temporary attribute overrides. The original spell attributes are restored after the operation.

```lua
spell:Cast(target, {
    ignoreFacing = true,
})
```

## Debugging

To log cast attempts:

```lua
vayn.print_cast_attempts = true
```

For failed `Castable()` checks:

```lua
vayn.debug = true
```

`vayn.debug` also accepts a search string to filter the results.
```lua
vayn.debug = "Eviscerate"
```

# Unit API

A `unit` represents a WoW game object used by the rotation. Players, NPCs, pets, totems, and game objects are exposed through the same interface.

Common unit references include `vayn.player`, `vayn.target`, and members of unit lists such as `vayn.enemies`.

## Examples

```lua
-- Health
if vayn.player.hp < 50 then
    -- heal logic
end

-- Aura
if vayn.target.debuff(703, player) then
    local remains = vayn.target.debuffRemains(703)
end

-- Range and line of sight, most likely covered by :Castable anyway
if vayn.target.distance < 8 and vayn.target.los then
    -- target is in range and visible
end

-- Crowd control
if vayn.target.stun then
    local dr = vayn.target.stunDR
    local remains = vayn.target.stunRemains
end
```

## Accessing Units

### Properties

Properties are read directly:

```lua
vayn.target.hp
vayn.target.enemy
vayn.target.casting
```

Many properties provide safe defaults when the underlying object no longer exists. For example, `hp` defaults to `100` and `distance` to `math.huge`.

### Methods

Methods are called with parentheses:

```lua
vayn.target.buff(45438)
vayn.target.distanceTo(vayn.player)
vayn.enemies.within(40).find(function(unit)
    return unit.viableEnemy
end)
```

Missing units return safe method stubs where applicable, such as `buffRemains()` returning `0` and `distanceTo()` returning `math.huge`.

### Direct Unit Methods

| Method | Description |
|---|---|
| `unit:Interact()` | Calls `ObjectInteract` if the unit exists. |
| `unit:SetTarget()` | Targets the unit if it is not already targeted. |

## Identity & Existence

| Property | Description |
|---|---|
| `exists` | Whether the WGG object exists. |
| `valid` | Whether `omToken` is set and the object exists. |
| `alive` / `dead` | Life state. Feign Death is treated as alive. |
| `name` | Object name, or `"Unknown"`. |
| `id` | NPC/object ID. |
| `type` / `typeName` | Object type index and label. |
| `omToken` | WoW object token. |
| `uptime` / `existsSince` | Time since the wrapper was created. |
| `player` | Whether the unit is a player. |
| `pet` / `battlePet` / `totem` | Pet/totem checks. |
| `level` | Unit level. |

## Faction & Targeting

| Property | Description |
|---|---|
| `enemy` | Whether the player can attack the unit. |
| `friend` | Whether the unit is friendly to the player. |
| `target` | The unit's current target. |
| `creator` | The object's creator. |
| `viable` / `viableEnemy` / `viableFriend` | Combat viability filters. |
| `combat` / `combatTime` | Combat state and duration. |
| `los` | Line of sight from the player. |

You can chain these like:
```lua
    local someUnit = vayn.enemies.lowest
    if someUnit.target.los
```

## Comparison, Positions, Los

| Method | Description |
|---|---|
| `isUnit(other)` | Compares two units. |
| `enemyTo(other)` / `friendTo(other)` | Faction relative to another unit. |
| `losTo(other)` / `losToRaw(other)` | Line of sight to another unit. |
| `losToPosition(pos)` / `losToPositionRaw(pos)` | Line of sight to `{x, y, z}`. |
| `losOf(other)` | Inverse of `other.losTo(self)`. |
| `predictLos(other, time)` | Predicts line of sight after `time` seconds. |
| `position` | `{x, y, z}` position. |
| `positionRaw` | Raw position from `ObjectPosition`. |
| `x` / `y` / `z` | Individual coordinates. |
| `rotation` | Facing angle in radians. |
| `distance` | 3D distance from the player. |
| `distance2D` | 2D distance from the player. |
| `combatReach` | Object combat reach. |
| `meleeRange` | Melee range including reach and lag buffer. |
| `speed` / `maxSpeed` | Current and maximum movement speed. |
| `moving` | Whether `speed > 0`. |
| `timeStandingStill` | Time spent standing still. |
| `altitude` | Height above ground. |
| `timeToUnit` / `timeToUnitRaw` | Estimated intercept time. |
| `trapTravelTime` | Estimated trap travel time. |
| `averageRange` | Spec-based average engagement range. |
| `distanceTo(unit)` / `distance2DTo(unit)` | Distance to another unit. |
| `distanceToPosition(pos)` / `distance2DToPosition(pos)` | Distance to a position. |
| `predictDistance(time)` / `predictDistance2D(time)` | Predicted distance after `time`. |
| `predictDistanceTo(unit, time)` | Predicted distance to a unit. |
| `predictPosition(time)` | Predicted `{x, y, z}` position. |
| `movingTowards(unit)` / `movingAwayFrom(unit)` | Movement direction relative to a unit. |
| `hasMovementFlag(flag)` | Tests a `MovementFlags` value. |
| `facing(unit, angle?)` | Checks the facing cone; default is 180°. |
| `facing45(unit)` / `facing90(unit)` | 45° / 90° facing checks. |
| `behind(unit)` | Checks whether this unit is behind another. |
| `angleTo(unit)` | Angle to another unit. |
| `setFace(unit?)` / `SetFacing(unit?)` | Faces a unit or numeric angle. |


## Health & Absorbs

| Property | Description |
|---|---|
| `health` | Raw health. |
| `maxHealth` | Maximum health. |
| `hp` | Health percentage, `0–100`. |
| `hpa` | Effective health percentage including absorbs minus heal absorbs. |
| `realHealth` | `health - healAbsorb + absorb`. |
| `healthMissing` / `hpMissing` / `realHealthMissing` | Missing-health values. |
| `healAbsorb` | Total heal absorption. |
| `absorb` | Total damage absorption. |

## Casting & Channels

| Property | Description |
|---|---|
| `casting` | Cast spell name, or `nil`. |
| `castID` | Cast spell ID. |
| `castTimeRemains` / `castTimeComplete` | Cast timing. |
| `castPct` / `castPctRemains` | Cast progress. |
| `channeling` | Channel spell name, or `false`. |
| `channelID` | Channel spell ID. |
| `channelTimeRemains` / `channelTimeComplete` | Channel timing. |
| `channelPct` / `channelPctRemains` | Channel progress. |
| `castTarget` | Unit being cast at. |
| `canCastWhileMoving` | Whether the unit has a cast-while-moving buff. |
| `lastCast` | Last recorded cast ID. |
| `gcd` | Remaining GCD; player only. |
| `maxGCD` | Maximum GCD for the unit's spec/haste. |
| `recentlyCast(spellID, window?)` | Whether the spell was cast within the window. |
| `recentlyCastTime(spellID)` | Time since the spell was cast. |
| `cooldown(spellID)` | Remaining tracked cooldown. |


## Auras

| Method | Description |
|---|---|
| `buff(id, creator?, uptime?)` | Longest matching buff aura. Accepts spell ID or lowercase name. |
| `debuff(id, creator?, uptime?)` | Longest matching debuff aura. |
| `buffFrom(table, creator?, uptime?)` | First matching buff from a spell ID table. |
| `debuffFrom(table, creator?, uptime?)` | First matching debuff from a spell ID table. |
| `buffRemains(id, creator?)` | Buff time remaining. |
| `debuffRemains(id, creator?)` | Debuff time remaining. |
| `buffUptime(id, creator?)` | Buff uptime. |
| `debuffUptime(id, creator?)` | Debuff uptime. |
| `buffStacks(id, creator?)` | Buff stack count. |
| `debuffStacks(id, creator?)` | Debuff stack count. |
| `hiddenAura(id)` | Hidden/private aura. |
| `hiddenAuraFrom(table)` | First matching hidden aura. |
| `hiddenAuraRemains(id)` / `hiddenAuraUptime(id)` | Hidden-aura timing helpers. |

Aura objects expose properties such as `.remains`, `.uptime`, `.stacks`, `.creator`, and `.id`.

### Crowd Control

CC is detected from the aura tables in `vayn.ids.cc`. Most CC properties return an aura object when active and `nil`/`false` otherwise.

| Property           | Description                            |
|--------------------|----------------------------------------|
| **Hard Crowd Control:** | |
| `stun`             | Stun aura object             |
| `incap`            | Incapacitate aura object     |
| `disorient`        | Disorient aura object        |
| `fear`             | Fear aura object            |
| `horror`           | Horror aura object          |
| `mindcontrol`      | Mind Control aura object    |
| **Soft Crowd Control:**  | |
| `disarm`           | Disarm aura object          |
| `root`             | Root aura object            |
| `slow`             | Slow aura object            |
| `silence`          | Silence aura object         |
| **Special CC Types:**     |                               |
| `bcc`              | Breakable Crowd Control                |
| `vbcc`             | Valuable Breakable Crowd Control       |
| `hardcc`           | Non-breakable Hard Crowd Control       |
| `cc`               | Role-aware CC summary                  |
| `incomingCC`       | Predicted incoming CC                  |
| **Time Remaining for Each CC Category:** |                 |
| `stunRemains`            | Time remaining on stun           |
| `incapRemains`           | Time remaining on incapacitate   |
| `disorientRemains`       | Time remaining on disorient      |
| `fearRemains`            | Time remaining on fear           |
| `horrorRemains`          | Time remaining on horror         |
| `mindcontrolRemains`     | Time remaining on mind control   |
| `disarmRemains`          | Time remaining on disarm         |
| `rootRemains`            | Time remaining on root           |
| `slowRemains`            | Time remaining on slow           |
| `silenceRemains`         | Time remaining on silence        |
| `bccRemains`             | Time remaining on breakable CC   |
| `vbccRemains`            | Time remaining on valuable BCC   |
| `hardccRemains`          | Time remaining on non-breakable CC|
| `ccRemains`              | Time remaining on general CC     |
| **Uptime for Each CC Category:**         |                  |
| `stunUptime`             | Uptime for stun                  |
| `incapUptime`            | Uptime for incapacitate          |
| `disorientUptime`        | Uptime for disorient             |
| `fearUptime`             | Uptime for fear                  |
| `horrorUptime`           | Uptime for horror                |
| `mindcontrolUptime`      | Uptime for mind control          |
| `disarmUptime`           | Uptime for disarm                |
| `rootUptime`             | Uptime for root                  |
| `slowUptime`             | Uptime for slow                  |
| `silenceUptime`          | Uptime for silence               |
| `bccUptime`              | Uptime for breakable CC          |
| `vbccUptime`             | Uptime for valuable BCC          |
| `hardccUptime`           | Uptime for non-breakable CC      |
| `ccUptime`               | Uptime for general CC            |

## Diminishing Returns

DR values are tracked by category, including

| Property         | Description                                 |
|------------------|---------------------------------------------|
| `stunDR`         | Stun diminishing returns tier (1, 0.5, 0).  |
| `stunDRRemains`  | Time remaining on stun DR.                  |
| `incapDR`        | Incapacitate diminishing returns tier.      |
| `incapDRRemains` | Time remaining on incap DR.                 |
| `disorientDR`    | Disorient diminishing returns tier.         |
| `disorientDRRemains` | Time remaining on disorient DR.         |
| `disarmDR`       | Disarm diminishing returns tier.            |
| `disarmDRRemains`| Time remaining on disarm DR.                |
| `rootDR`         | Root diminishing returns tier.              |
| `rootDRRemains`  | Time remaining on root DR.                  |
| `silenceDR`      | Silence diminishing returns tier.           |
| `silenceDRRemains` | Time remaining on silence DR.             |
| `knockbackDR`    | Knockback diminishing returns tier.         |
| `knockbackDRRemains` | Time remaining on knockback DR.         |

Values follow the WoW DR tiers: `1 → 0.5 → 0`.
If a unit is on full stun DR it will be also considered stun immune, :Castable(unit) checks will fail during this time.

## Immunities

Immunity properties combine aura checks, DR state, and untouchable CC debuffs. Immunity properties generally have matching `*Remains` and `*Uptime` variants.

| Property | Description |
|---|---|
| `physicalDamageImmunity` | Physical damage immunity. |
| `magicDamageImmunity` | Magic damage immunity. |
| `physicalEffectImmunity` | Physical effect immunity. |
| `magicEffectImmunity` | Magic effect immunity. |
| `directionalPhysicalDamageImmunity` | Directional physical immunity. |
| `stunImmunity` / `incapImmunity` / `polymorphImmunity` | CC immunities. |
| `disorientImmunity` / `fearImmunity` | Disorient/fear immunities. |
| `disarmImmunity` / `rootImmunity` / `slowImmunity` | Soft CC immunities. |
| `silenceImmunity` / `interruptImmunity` / `knockbackImmunity` | Cast/interrupt immunities. |
| `healImmunity` / `beneficialImmunity` | Healing/beneficial immunities. |
| `untouchableCC` | Untouchable CC debuff. |
| `bccImmunity` | BCC immunity. (Spells like Sacrifice will break our bcc, so we consider the unit immune to bcc) |

## Common Unit Properties

| Property | Description |
|---|---|
| `stealth` | Stealth buff. |
| `bloodlust` | Bloodlust/Heroism. |
| `trinket` / `trinketRemains` | PvP trinket state. |
| `humanRacial` / `humanRacialRemains` | Will to Survive tracking. |
| `defensiveCDs` / `majorDefensiveCDs` / `offensiveCDs` | Cooldown aura groups. |
| `purgable` | Stealable buff. |
| `ams` | Anti-Magic Shell. |
| `smokeBomb` / `speared` / `searingGlare` | pvp-specific debuff checks. |

## Power & Resources

Generic resource methods:

| Method | Description |
|---|---|
| `power(type)` | Current power for a `POWER_TYPES` index. |
| `powerMax(type)` | Maximum power. |
| `powerPct(type)` | Power percentage. |
| `powerDeficit(type)` | Missing power. |

| Resource | Common properties |
|---|---|
| Mana | `mana`, `manaMax`, `manaPct`, `manaDeficit` |
| Rage | `rage`, `rageMax`, `ragePct`, `rageDeficit` |
| Focus | `focus`, `focusMax`, `focusPct`, `focusDeficit` |
| Energy | `energy`, `energyMax`, `energyPct`, `energyDeficit` |
| Combo Points | `comboPoints`, `comboPointsMax`, `comboPointsPct`, `comboPointsDeficit`, `chargedComboPoints` |
| Runes | `runes`, `runesMax`, `runesPct`, `runesDeficit` |
| Runic Power | `runicPower`, `runicPowerMax`, `runicPowerPct`, `runicPowerDeficit` |
| Soul Shards | `soulShards`, `soulShardsMax`, `soulShardsPct`, `soulShardsDeficit`, `predictedSoulShards` |
| Astral Power | `astralPower`, `astralPowerMax`, `astralPowerPct`, `astralPowerDeficit` |
| Holy Power | `holyPower`, `holyPowerMax`, `holyPowerPct`, `holyPowerDeficit` |
| Maelstrom | `maelstrom`, `maelstromMax`, `maelstromPct`, `maelstromDeficit` |
| Chi | `chi`, `chiMax`, `chiPct`, `chiDeficit` |
| Insanity | `insanity`, `insanityMax`, `insanityPct`, `insanityDeficit` |
| Arcane Charges | `arcaneCharges`, `arcaneChargesMax`, `arcaneChargesPct`, `arcaneChargesDeficit` |
| Fury | `fury`, `furyMax`, `furyPct`, `furyDeficit` |
| Pain | `pain`, `painMax`, `painPct`, `painDeficit` |
| Essence | `essence`, `essenceMax`, `essencePct`, `essenceDeficit` |
| DK Runes | `runeBlood`, `runeFrost`, `runeUnholy` and their Max/Pct/Deficit variants |

Player-only resource properties include `powerRegen`, `basePowerRegen`, `combatPowerRegen`, `mounted`, and `gcd`.

## Class, Spec & Role

### Class

Boolean class properties include:

| Property | Description |
|---|---|
| `warrior`       | Warrior class    |
| `paladin`       | Paladin class   |
| `hunter`        | Hunter class    |
| `rogue`         | Rogue class     |
| `priest`        | Priest class    |
| `deathKnight`   | Death Knight class |
| `shaman`        | Shaman class    |
| `mage`          | Mage class      |
| `warlock`       | Warlock class   |
| `monk`          | Monk class      |
| `druid`         | Druid class     |
| `demonHunter`   | Demon Hunter class |
| `evoker`        | Evoker class    |

`class1`, `class2`, and `class3` return the class index, English token, and localized name.

### Specialization

Every specialization has a boolean property such as `armsWarrior`, `frostMage`, `restorationDruid`, `subtletyRogue`, and `havocDemonhunter`.

| Property | Description |
|---|---|
| `specID` | Specialization ID. |
| `specName` / `specNameShort` | Full and short specialization names. |

### Role

| Property | Description |
|---|---|
| `healer` / `dps` / `tank` | Spec-based role. |
| `melee` / `ranged` / `caster` | Range archetype. |
| `magicDPS` / `physicalDPS` | Damage type. |
| `healerRole` / `damageRole` / `tankRole` | WoW group role. |

### Race

| Property                | Description         |
|-------------------------|--------------------|
| `race`                  | Returns the race name. |
| `human`                 | Human race boolean |
| `orc`                   | Orc race boolean   |
| `nightElf`              | Night Elf race boolean |
| `bloodElf`              | Blood Elf race boolean |
| `undead`                | Undead race boolean |
| `dwarf`                 | Dwarf race boolean |
| `gnome`                 | Gnome race boolean |
| `draenei`               | Draenei race boolean |
| `worgen`                | Worgen race boolean |
| `pandaren`              | Pandaren race boolean |
| `tauren`                | Tauren race boolean |
| `troll`                 | Troll race boolean |
| `goblin`                | Goblin race boolean |
| `voidElf`               | Void Elf race boolean |
| `lightforgedDraenei`    | Lightforged Draenei race boolean |
| `darkIronDwarf`         | Dark Iron Dwarf race boolean |
| `kulTiran`              | Kul Tiran race boolean |
| `mechagnome`            | Mechagnome race boolean |
| `nightborne`            | Nightborne race boolean |
| `highmountainTauren`    | Highmountain Tauren race boolean |
| `magharOrc`             | Mag'har Orc race boolean |
| `zandalariTroll`        | Zandalari Troll race boolean |
| `vulpera`               | Vulpera race boolean |
| `dracthyr`              | Dracthyr race boolean |

## Attackers & PvP Utility

| Property | Description |
|---|---|
| `attackers` | Total DPS attackers. |
| `meleeAttackers` | Melee attackers. |
| `rangedAttackers` | Ranged attackers. |
| `cooldownAttackers` | Attackers with offensive cooldowns active. |
| `fc` / `hasFlag` | Flag carrier. |
| `orbOfPower` | Orb of Power buff. |
| `capping` / `cappingRemains` | Node capping state. |
| `drinking` | Drinking/regen state. |

### `v2Attackers(option?, ignoreRange?)`

Returns detailed attacker counts. Options include:

- `"totalUnits"`
- `"meleeUnits"`
- `"rangedUnits"`
- `"cooldowns"`
- `"magic"`

Depending on the option, it returns a count or without anything passed to the function `(melee, ranged, cds, total)`.

## World Objects

| Property | Description |
|---|---|
| `gatherable` | Herb/ore node that is not depleted. |
| `lootable` | Can be looted. |
| `skinnable` | Can be skinned. |
| `tapDenied` | Mob tap is denied. |
| `creatureType` | Creature type such as `"Humanoid"` or `"Beast"`. |
| `humanoid` / `beast` / `critter` / `undead` | Creature-type shortcuts. |
| `outdoors` | Placeholder; currently always `false`, till we get an implementation from WGG |

## Talents & Totems

| Method | Description |
|---|---|
| `hasTalent(talentID)` | Checks whether the unit has a talent. (only player) |
| `hasTotem(totemID)` | Checks whether the unit has a totem. |


## Aliases

Aliases exist for shorthand and backwards compatibility. They behave the same as their canonical properties or methods.

| Alias | Canonical |
|---|---|
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
| `dk` / `frostdk` / `sub` / `disc` | Class/spec shortcuts |
| `distanceToUnit` | `distanceTo` |
| `distanceToPoint` | `distanceToPosition` |
| `movingTo` / `isMovingTowards` | `movingTowards` |
| `recentlyUsed` / `used` | `recentlyCast` |
| `v2attackers` | `v2Attackers` |
| `predictLoS` | `predictLos` |


## Player-only APIs

Some APIs are only valid on vayn.player, including `gcd`, `mounted`, and power regeneration properties. Calling these on other units will result in an lua error.

## Default returns

The unit metatable provides safe defaults for many properties when an object no longer exists, allowing rotation code to avoid repetitive nil checks.

# Macro System

Macros are temporary flags set through slash commands.

### Registering a Macro

```lua
vayn.RegisterMacro("burst", 1)
```

The second argument is the duration in seconds.

### Checking a Macro

```lua
if not vayn.MacrosQueued["burst"] then return end
```

### Using the Macro In-Game

```text
/username burst
```

# Timing & Humanization

## Delays

Create a randomized delay with `vayn.delay()`:

```lua
local delay = vayn.delay(0.2, 0.4, 5)
```

This creates a delay between `0.2` and `0.4` seconds and changes the selected value every `5` seconds.

Use the current delay with `.now`:

```lua
if target.buffUptime(123) < delay.now then return end
```

## Debounce

`vayn.debounce()` tracks time from when a decision first becomes viable rather than depending on the conditions remaining true.

```lua
return vayn.debounce(
    spell.name,
    vayn.around3.now,
    5,
    function()
        return spell:Cast(player)
    end
)
```

Signature:

```lua
vayn.debounce(key, min, reset, func)
```

| Argument | Description |
|---|---|
| `key` | Unique identifier for the debounce. |
| `min` | Minimum time before the function can execute. |
| `reset` | Time after which the debounce state is forgotten. |
| `func` | Function to execute. |


## Integrated Helper Functions

| Function | Description |
|---|---|
| `vayn:SmartInterrupt(spell)`                 | Intelligently interrupts the enemy's spell cast.          |
| `vayn:GetInterruptUnit(spell)`               | Finds the best enemy unit to interrupt with given spell.  |
| `vayn:SmartAutoHit()`                        | Activates auto-attack/hit on an appropriate target.       |
| `vayn:SmartDispell(spell)`                   | Detects and dispels debuffs on allies automatically.      |
| `vayn:SmartTotemStomp(spell, directCast, options)` | Seeks and destroys enemy totems or similar entities.      |
| `vayn:SmartTotemStompGrounding(spell, directCast, options)` | Destroys Grounding Totems.   |
| `vayn:UseDPSTrinket()`                       | Uses your DPS trinket if ready.              |
| `vayn:StopMoving()`                          | Stops all player movement.                                |
| `vayn:BlockMovement()`                       | Temporarily blocks all movement inputs.                   |
| `vayn:ResumeMovement()`                      | Restores and allows movement inputs again.                |

## Update Ticker

`vayn.addUpdateCallback(function())` runs before any rotation actor gets invoked, useful for running point based object loops and caching the result for that frame

## Instances
| Function | Description |
|---|---|
| `vayn.arena`         | True in any arena match.            |
| `vayn.ratedArena`    | True in rated arena.                |
| `vayn.battleground`  | True in battlegrounds.              |
| `vayn.flagMap`       | True on flag-based BG maps.         |
| `vayn.pvp`           | True in any PvP zone or mode.       |
| `vayn.pve`           | True outside of PvP (PvE) zones.    |
| `vayn.soloRBG`       | True in solo rated battlegrounds.   |
| `vayn.soloShuffle`   | True in solo shuffle matches.       |

## Alerts

| Function | Description |
|---|---|
| `vayn.alert(message, spellId, time, options)` | Shows a large alert with customizable options. |
| `vayn.smallAlert(message, spellId, time)` | Shows a small alert with given message and spell. |
| `vayn.largeAlert(message, spellId, time)` | Shows a large (prominent) alert. |
| `vayn.utilityAlert(message, spellId, time)` | Small alert with utility color/style. |
| `vayn.defensiveAlert(message, spellId, time)` | Small alert with defensive color/style. |
| `vayn.ccAlert(message, spellId, time)` | Small alert styled for crowd control. |
| `vayn.dangerAlert(message, spellId, time)` | Small alert for dangerous situations. |
| `vayn.burstAlert(message, spellId, time)` | Large alert indicating burst effects. |
| `vayn.quickAlert(message, spell)` | Shortcut large alert for the provided spell. |