
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
3. Your project table — shared by all files in the project.

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

Callbacks should return the result of the action. A successful `Cast()` tells the framework that the rotation has performed an action and no further logic should run for that iteration.

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
| `vayn.enemyHealer` | The first enemy healer found. Useful in arena; not recommended for battleground logic. |

### Unit Lists

Unit lists provide access to units that are not exposed through the static objects.

| List | Description |
|---|---|
| `vayn.enemies` | Enemy players in the PvP environment. |

Unit lists support methods such as `within()` and `find()`.

```lua
local enemy = vayn.enemies
    .within(40)
    .find(function(unit)
        return unit.viableEnemy
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
| `castLength` | Alias for `castTime`. |

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
| `jump` / `jumpOrMove` | Uses `vayn:JumpApex()`. |
| `ignoreCasting` | Stops the current cast. |
| `cancelCatForm` | Cancels Cat Form when rooted/slowed. |
| `castByID` | Uses `CastSpellByID`. |
| `name` | Uses an alternate spell name. |

Without `castByID`, casting normally uses `vayn.CastSpellByName()`.

Cast attempts are rate-limited per spell ID with an internal delay of approximately `0.25–0.50` seconds.

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

`vayn.debug` can also accept a search string.

`vayn.preCastWindow` controls how close a spell can be to becoming available while still being queued/cast, depending on its attributes.


# Unit API

A `unit` represents a WoW game object used by the rotation. Players, NPCs, pets, totems, and game objects are exposed through the same interface.

Common unit references include `vayn.player`, `vayn.target`, and members of unit lists such as `vayn.enemies`.

## Quick Start

```lua
-- Health
if vayn.player.hp < 50 then
    -- heal logic
end

-- Aura
if vayn.target.debuff(703) then
    local remains = vayn.target.debuffRemains(703)
end

-- Range and line of sight
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
| `guid` | WGG GUID. |
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

### Methods

| Method | Description |
|---|---|
| `isUnit(other)` | Compares two units. |
| `isTarget()` | Checks whether this is the current target. |
| `enemyTo(other)` / `friendTo(other)` | Faction relative to another unit. |
| `losTo(other)` / `losToRaw(other)` | Line of sight to another unit. |
| `losToPosition(pos)` / `losToPositionRaw(pos)` | Line of sight to `{x, y, z}`. |
| `losOf(other)` | Inverse of `other.losTo(self)`. |
| `icewallObstructingLosTo(unit)` | Checks for an Ice Wall obstruction. |
| `smokebombObstructingLosTo(unit)` | Checks for a Smoke Bomb obstruction. |
| `predictLos(other, time)` | Predicts line of sight after `time` seconds. |

## Position & Movement

| Property | Description |
|---|---|
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
| `trapTravelTime` | Estimated trap travel time at 19.281 yds/s. |
| `averageRange` | Spec-based average engagement range. |

### Movement Flags

Movement flags include:

`movingForward`, `movingBackward`, `stravingLeft`, `stravingRight`, `turningLeft`, `turningRight`, `falling`, `fallingFar`, `swimming`, `flying`, `canFly`, `ascending`, `descending`, `levitating`, `onTransport`

Most also have a corresponding `*Pending` variant.

### Methods

| Method | Description |
|---|---|
| `distanceTo(unit)` / `distance2DTo(unit)` | Distance to another unit. |
| `distanceToPosition(pos)` / `distance2DToPosition(pos)` | Distance to a position. |
| `predictDistance(time)` / `predictDistance2D(time)` | Predicted distance after `time`. |
| `predictDistanceTo(unit, time)` | Predicted distance to a unit. |
| `predictPosition(time)` | Predicted `{x, y, z}` position. |
| `movingTowards(unit)` / `movingAwayFrom(unit)` | Movement direction relative to a unit. |
| `hasMovementFlag(flag)` | Tests a `MovementFlags` value. |
| `facing(unit, options?)` | Checks the facing cone; default is 180°. |
| `facing45(unit)` / `facing90(unit)` | 45° / 90° facing checks. |
| `behind(unit)` | Checks whether this unit is behind another. |
| `angleTo(unit)` | Angle to another unit. |
| `setFace(unit?)` / `SetFacing(unit?)` | Faces a unit or numeric angle. |

### Distance to a Position

```lua
local distance = player.distanceToPosition({
    x = 1,
    y = 100,
    z = 300,
})
```

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
| `guardianSpirit` | Guardian Spirit state with heal-event tracking. |

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

### Methods

| Method | Description |
|---|---|
| `recentlyCast(spellID, window?)` | Whether the spell was cast within the window. |
| `recentlyCastTime(spellID)` | Time since the spell was cast. |
| `cooldown(spellID)` | Remaining tracked cooldown. |
| `canCastOverlappingSpell()` / `setOverlappingSpell()` | Helpers for overlapping cast windows. |

## Crowd Control

CC is detected from the aura tables in `vayn.ids.cc`. Most CC properties return an aura object when active and `nil`/`false` otherwise.

| Property | Description |
|---|---|
| `stun` / `incap` / `disorient` / `fear` / `horror` / `mindcontrol` | Hard CC. |
| `cyclone` / `sleep` / `charmed` | Special CC. |
| `disarm` / `root` / `slow` / `silence` | Soft CC. |
| `bcc` | Breakable CC. |
| `vbcc` | Valuable breakable CC. |
| `hardcc` | Non-breakable hard CC. |
| `cc` | Role-aware CC summary. |
| `longestCC` | Longest active CC aura. |
| `incomingCC` / `incomingCCRemains` | Predicted incoming CC. |

CC categories also expose matching `*Remains` and `*Uptime` properties and, where applicable, `*DR` / `*DRRemains` properties.

### Diminishing Returns

DR values are tracked by category, including `stunDR`, `incapDR`, `disorientDR`, `fearDR`, `horrorDR`, `cycloneDR`, `disarmDR`, `rootDR`, `silenceDR`, and `knockbackDR`.

Values follow the WoW DR tiers: `1 → 0.5 → 0.25 → 0`.

| Property | Description |
|---|---|
| `stunBreak` | Whether a stun break effect is active. |
| `safe` | Whether the unit is considered safe due to major defensives, CC, etc. |

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
| `bccImmunity` | BCC immunity. |

## Auras

### Methods

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

Aura objects created by `vayn.aura:New` expose properties such as `.remains`, `.uptime`, `.stacks`, `.creator`, and `.id`.

### Common Aura Properties

| Property | Description |
|---|---|
| `stealth` | Stealth buff. |
| `bloodlust` | Bloodlust/Heroism. |
| `trinket` / `trinketRemains` | PvP trinket state. |
| `humanRacial` / `humanRacialRemains` | Will to Survive tracking. |
| `defensiveCDs` / `majorDefensiveCDs` / `offensiveCDs` | Cooldown aura groups. |
| `purgable` | Stealable buff. |
| `dotted` | Bleed/DoT from `vayn.ids`. |
| `ams` | Anti-Magic Shell. |
| `smokeBomb` / `speared` / `searingGlare` | Arena-specific debuff checks. |

## Power & Resources

Generic resource methods:

| Method | Description |
|---|---|
| `power(type)` | Current power for a `POWER_TYPES` index. |
| `powerMax(type)` | Maximum power. |
| `powerPct(type)` | Power percentage. |
| `powerDeficit(type)` | Missing power. |

Each resource also exposes `<resource>`, `<resource>Max`, `<resource>Pct`, and `<resource>Deficit` properties where applicable.

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

`warrior`, `paladin`, `hunter`, `rogue`, `priest`, `deathKnight`, `shaman`, `mage`, `warlock`, `monk`, `druid`, `demonHunter`, `evoker`

`class1`, `class2`, and `class3` return the class index, English token, and localized name.

### Specialization

Every PvP-relevant specialization has a boolean property such as `armsWarrior`, `frostMage`, `restorationDruid`, `subtletyRogue`, and `havocDemonhunter`.

| Property | Description |
|---|---|
| `specID` | Specialization ID. |
| `specName` / `specNameShort` | Full and short specialization names. |
| `specColor` | Specialization color from `vayn.ids.specs`. |

### Role

| Property | Description |
|---|---|
| `healer` / `dps` / `tank` | Spec-based role. |
| `melee` / `ranged` / `caster` | Range archetype. |
| `magicDPS` / `physicalDPS` | Damage type. |
| `healerRole` / `damageRole` / `tankRole` | WoW group role. |
| `inGroup` / `raid` / `party` | Group membership. |

### Race

`race` returns the race name. Race booleans include `human`, `orc`, `nightElf`, `bloodElf`, `undead`, `dwarf`, `gnome`, `draenei`, `worgen`, `pandaren`, `tauren`, `troll`, `goblin`, `voidElf`, `lightforgedDraenei`, `darkIronDwarf`, `kulTiran`, `mechagnome`, `nightborne`, `highmountainTauren`, `magharOrc`, `zandalariTroll`, `vulpera`, and `dracthyr`.

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

Depending on the option, it returns a count or `(melee, ranged, cds, total)`.

## World Objects

| Property | Description |
|---|---|
| `gatherable` | Herb/ore node that is not depleted. |
| `lootable` | Can be looted. |
| `skinnable` | Can be skinned. |
| `tapDenied` | Mob tap is denied. |
| `creatureType` | Creature type such as `"Humanoid"` or `"Beast"`. |
| `humanoid` / `beast` / `critter` / `undead` | Creature-type shortcuts. |
| `outdoors` | Placeholder; currently always `false`. |
| `locked` | Placeholder; currently always `false`. |

## Talents & Totems

| Method | Description |
|---|---|
| `hasTalent(talentID)` | Checks whether the unit has a talent. |
| `hasTotem(totemID)` | Checks whether the unit has a totem. |

`talent` is an alias for `hasTalent`.

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

### Method aliases

| Alias | Canonical |
|---|---|
| `distanceToUnit` | `distanceTo` |
| `distanceToPoint` | `distanceToPosition` |
| `movingTo` / `isMovingTowards` | `movingTowards` |
| `recentlyUsed` / `used` | `recentlyCast` |
| `v2attackers` | `v2Attackers` |
| `predictLoS` | `predictLos` |

Accessing an unknown property throws `Unit property <name> not found`.

## Player-only APIs

Some APIs are only valid on the player, including `gcd`, `mounted`, and power regeneration properties. Calling these on non-player units can call `vayn.error` or `error()`.

## Missing Units

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

# Reference

The sections below contain the complete property and method indexes. Use the categorized sections above for normal development; use these indexes when looking for an API name.

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

magicDispell

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
