# Unit API

The `unit` module (`frame/unit.lua`) is the core object wrapper around WoW game objects in vayn. Every entity the rotation logic cares about — players, NPCs, pets, totems, and game objects — is represented as a **unit** instance with a rich property and method API.

Units are created via `vayn.unit:New(object)` and are usually obtained through `vayn.unitManager.get(...)`. Global shortcuts like `vayn.player`, `vayn.target`, and list members (`vayn.enemies`, `vayn.fgroup`, etc.) all resolve to unit instances.

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

```
absorb, afflictionWarlock, alive, altitude, ams, arcaneCharges, arcaneChargesDeficit,
arcaneChargesMax, arcaneChargesPct, arcaneMage, armsWarrior, ascending, assassinationRogue,
astralPower, astralPowerDeficit, astralPowerMax, astralPowerPct, attackers, augmentationEvoker,
auraInfo, averageRange, balanceDruid, base, basePowerRegen, baseRegen, battlePet, bcc,
bccImmunity, bccImmunityRemains, bccRemains, beast, beastmasteryHunter, beneficialImmunity,
beneficialImmunityRemains, beneficialImmunityUptime, bloodDeathknight, bloodElf, bloodlust,
brewmasterMonk, canCastWhileMoving, canFly, capping, cappingRemains, caster, castID, casting,
castPct, castPctRemains, castTarget, castTimeComplete, castTimeRemains, cc, ccRemains, ccUptime,
channelID, channeling, channelName, channelNotInterruptible, channelPct, channelPctRemains,
channelTimeComplete, channelTimeRemains, chargedComboPoints, chargedCPs, charmed,
charmedRemains, charmedUptime, chi, chiDeficit, chiMax, chiPct, class1, class2, class3, combat,
combatPowerRegen, combatReach, combatRegen, combatTime, comboPoints, comboPointsDeficit,
comboPointsMax, comboPointsPct, cooldownAttackers, creator, creatureType, critter, cyclone,
cycloneDR, cycloneDRRemains, cycloneRemains, cycloneUptime, damageRole, darkIronDwarf, dead,
deathKnight, defensiveCDs, defensiveCDsUptime, demonHunter, demonologyWarlock, descending,
destructionWarlock, devastationEvoker, directionalPhysicalDamageImmunity,
directionalPhysicalDamageImmunityRemains, directionalPhysicalDamageImmunityUptime, disarm,
disarmDR, disarmDRRemains, disarmImmunity, disarmImmunityRemains, disarmImmunityUptime,
disarmRemains, disarmUptime, disciplinePriest, disorient, disorientDR, disorientDRRemains,
disorientImmunity, disorientImmunityRemains, disorientImmunityUptime, disorientRemains,
disorientUptime, distance, distance2D, dotted, dps, dracthyr, draenei, drDuration, drinking,
druid, dummy, duration, dwarf, elementalShaman, endTime, endTimeMS, enemy, enemyTo, energy,
energyDeficit, energyMax, energyPct, enhancementShaman, essence, essenceDeficit, essenceMax,
essencePct, evoker, exists, existsSince, falling, fallingFar, fc, fear, fearDR, fearDRRemains,
fearImmunity, fearImmunityRemains, fearImmunityUptime, fearRemains, fearUptime, feralDruid,
fireMage, flying, focus, focusDeficit, focusMax, focusPct, friend, friendTo, frostDeathknight,
frostMage, fury, furyDeficit, furyMax, furyPct, furyWarrior, gatherable, gcd, gnome, goblin,
guardianDruid, guardianSpirit, guid, hardcc, hasFlag, haste, havocDemonhunter, healAbsorb,
healer, healerRole, healImmunity, healImmunityRemains, healImmunityUptime, health,
healthMissing, highmountainTauren, holyPaladin, holyPower, holyPowerDeficit, holyPowerMax,
holyPowerPct, holyPriest, horror, horrorDR, horrorDRRemains, horrorRemains, horrorUptime,
hp, hpa, hpMissing, human, humanoid, humanRacial, humanRacialRemains, hunter, id, incap,
incapDR, incapDRRemains, incapImmunity, incapImmunityRemains, incapImmunityUptime, incapRemains,
incapUptime, incomingCC, incomingCCRemains, inGroup, insanity, insanityDeficit, insanityMax,
insanityPct, interruptImmunity, interruptImmunityRemains, interruptImmunityUptime, knockbackDR,
knockbackDRRemains, knockbackImmunity, knockbackImmunityRemains, knockbackImmunityUptime,
kulTiran, lastCast, lastTrinketUse, level, levitating, lightforgedDraenei, locked, longestCC,
lootable, los, maelstrom, maelstromDeficit, maelstromMax, maelstromPct, mage, magharOrc,
magicDamageImmunity, magicDamageImmunityRemains, magicDamageImmunityUptime, magicDPS,
magicEffectImmunity, magicEffectImmunityRemains, magicEffectImmunityUptime, majorDefensiveCDs,
majorDefensiveCDsUptime, mana, manaDeficit, manaMax, manaPct, marksmanshipHunter, maxGCD,
maxHealth, maxRemaining, maxSpeed, mechagnome, melee, meleeAttackers, meleeRange, mindcontrol,
mindcontrolRemains, mindcontrolUptime, mistweaverMonk, monk, mounted, movementFlag, moving,
movingBackward, movingBackwardPending, movingForward, movingForwardPending, name, nightborne,
nightElf, notInterruptible, offensiveCDs, offensiveCDsUptime, omToken, onTransport,
orbOfPower, orc, outdoors, outlawRogue, pain, painDeficit, painMax, painPct, paladin, pandaren,
party, pet, physicalDamageImmunity, physicalDamageImmunityRemains, physicalDamageImmunityUptime,
physicalDPS, physicalEffectImmunity, physicalEffectImmunityRemains, physicalEffectImmunityUptime,
player, polymorphImmunity, polymorphImmunityRemains, polymorphImmunityUptime, position,
positionRaw, powerRegen, powerType, predictedSoulShards, preservationEvoker, priest,
protectionPaladin, protectionWarrior, purgable, race, racialRemaining, rage, rageDeficit,
rageMax, ragePct, raid, ranged, rangedAttackers, ready, realHealth, realHealthMissing,
restorationDruid, restorationShaman, retributionPaladin, rogue, role, root, rootDR,
rootDRRemains, rootImmunity, rootImmunityRemains, rootImmunityUptime, rootRemains, rootUptime,
rotation, runeBlood, runeBloodDeficit, runeBloodMax, runeBloodPct, runeFrost, runeFrostDeficit,
runeFrostMax, runeFrostPct, runes, runesDeficit, runesMax, runesPct, runeUnholy,
runeUnholyDeficit, runeUnholyMax, runeUnholyPct, runicPower, runicPowerDeficit,
runicPowerMax, runicPowerPct, safe, searingGlare, shadowPriest, shaman, sharedRemaining,
silence, silenceDR, silenceDRRemains, silenceImmunity, silenceImmunityRemains,
silenceImmunityUptime, silenceRemains, silenceUptime, skinnable, sleep, sleepRemains,
sleepUptime, slow, slowImmunity, slowImmunityRemains, slowImmunityUptime, slowRemains,
slowUptime, smokeBomb, soulShards, soulShardsDeficit, soulShardsMax, soulShardsPct, speared,
specColor, specID, specName, specNameShort, speed, spellId, start, startTimeMS, staticField,
stealth, straving, stravingLeft, stravingLeftPending, stravingRight, stravingRightPending,
stun, stunBreak, stunDR, stunDRRemains, stunImmunity, stunImmunityRemains, stunImmunityUptime,
stunRemains, stunUptime, subtletyRogue, survivalHunter, swimming, tank, tankRole, tapDenied,
target, tauren, timeStandingStill, timeToUnit, timeToUnitRaw, totem, trapTravelTime, trinket,
trinketRemains, troll, turning, turningLeft, turningRight, type, typeName, undead,
unholyDeathknight, untouchableCC, untouchableCCRemains, untouchableCCUptime, uptime, valid,
vbcc, vbccRemains, vengeanceDemonhunter, viable, viableEnemy, viableFriend, voidElf, vulpera,
warlock, warrior, windwalkerMonk, worgen, x, y, z, zandalariTroll
```

</details>

---

## Full method index

<details>
<summary>58 methods (click to expand)</summary>

```
angleTo, behind, buff, buffFrom, buffRemains, buffStacks, buffUptime,
canCastOverlappingSpell, cooldown, debuff, debuffFrom, debuffRemains, debuffStacks,
debuffUptime, distance2DTo, distance2DToPosition, distanceTo, distanceToPosition,
enemyTo, facing, facing45, facing90, friendTo, hasMovementFlag, hasTalent, hasTotem,
hiddenAura, hiddenAuraFrom, hiddenAuraRemains, hiddenAuraUptime, icewallObstructingLosTo,
icewallObstructingLosToPosition, isTarget, isUnit, losOf, losTo, losToPosition,
losToPositionRaw, losToRaw, movingAwayFrom, movingTowards, power, powerDeficit, powerMax,
powerPct, predictDistance, predictDistance2D, predictDistance2DTo, predictDistanceTo,
predictLos, predictPosition, recentlyCast, recentlyCastTime, setFace, SetFacing,
setOverlappingSpell, smokebombObstructingLosTo, v2Attackers
```

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
