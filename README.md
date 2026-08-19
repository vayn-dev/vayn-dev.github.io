# Unit

## `unit.name`

Returns the name of the unit.

## `unit.id`

Returns the ID of the unit.

## `unit.distance`

Returns the distance from the unit to the player.

## `unit.distanceTo(other)`

Returns the distance from the unit to another unit.

**Parameters:**

* `other` — The unit to measure the distance to.

---

# Distance to Position

## `unit.distanceToPosition(position)`

Returns the distance from the unit to a given position.

The `position` parameter is a table containing `x`, `y`, and `z` coordinates.

### Example

```lua
player.distanceToPosition({
    x = 1,
    y = 100,
    z = 300
})
```

**Position format:**

```lua
{
    x = 1,
    y = 100,
    z = 300
}
```
