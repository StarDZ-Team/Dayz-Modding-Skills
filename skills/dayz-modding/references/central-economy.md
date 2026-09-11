# Central Economy (CE) & Mission Files

The Central Economy decides what exists in the world: loot, vehicles, animals,
infected, dynamic events, and when any of it is deleted. A custom item with a
perfect `config.cpp` and a perfect script **will never spawn** until the CE knows
about it.

**Verification note.** Every structure and value in this file was read from the
vanilla mission shipped with DayZ Server
(`mpmissions\dayzOffline.chernarusplus`). Paths are relative to the mission root.

---

## 1. Mission File Map

Measured file tree of the vanilla Chernarus mission (persistence folders omitted):

| File | Size | Role |
|---|---|---|
| `init.c` | 2.7 KB | Mission entry point. `void main()` is mandatory |
| `cfgeconomycore.xml` | 2 KB | **CE master file** — root classes, defaults, and mod CE registration |
| `cfglimitsdefinition.xml` | 1.3 KB | The controlled vocabularies (categories, tags, usages, tiers) |
| `cfglimitsdefinitionuser.xml` | 1.4 KB | User-defined groupings over those vocabularies |
| `db\types.xml` | 880 KB | Per-item spawn rules — the file everyone edits |
| `db\events.xml` | 53 KB | Dynamic events (vehicles, heli crashes, animal herds, infected) |
| `db\globals.xml` | 1.7 KB | Global CE tuning variables |
| `db\economy.xml` | 0.5 KB | Which CE subsystems init / load / respawn / save |
| `db\messages.xml` | 1.2 KB | Scheduled server messages and restart warnings |
| `cfgspawnabletypes.xml` | 128 KB | Cargo, attachments and damage applied when an item spawns |
| `cfgrandompresets.xml` | 34 KB | Named random loot bundles referenced by spawnable types |
| `cfgeventspawns.xml` | 81 KB | World positions for each event |
| `cfgeventgroups.xml` | 119 KB | Multi-object event compositions |
| `cfgplayerspawnpoints.xml` | 13 KB | Fresh-spawn and respawn locations |
| `cfgignorelist.xml` | 0.8 KB | Types never persisted/restored |
| `cfgweather.xml` | 4.7 KB | Weather ranges — **XML, not JSON** |
| `cfgenvironment.xml` | 4.3 KB | Animal/infected territory file registration |
| `cfggameplay.json` | 2.8 KB | Runtime gameplay tuning (stamina, base damage, UI) |
| `cfgeffectarea.json` | 6.3 KB | Contaminated / static effect areas |
| `cfgundergroundtriggers.json` | 35 B | Underground area triggers |
| `env\*.xml` | 7–86 KB | Per-species territories |
| `mapgroupproto.xml` | 1.2 MB | **Loot positions inside every building type** |
| `mapgrouppos.xml` | 1.5 MB | Which building instance sits where on the map |
| `mapclusterproto.xml`, `mapgroupcluster*.xml` | 34 KB – 4.6 MB | Tree/bush clusters (dynamic events terrain data) |
| `areaflags.map` | 84 MB | Binary area flags |
| `storage_<instanceId>\` | — | **Live persistence. Back it up before touching the economy.** |

---

## 2. `cfgeconomycore.xml` — and How a Mod Ships Its Own CE

This is the file that tells the engine which CE files to load. Verified content:

```xml
<economycore>
    <classes>
        <rootclass name="DefaultWeapon" />
        <rootclass name="DefaultMagazine" />
        <rootclass name="Inventory_Base" />
        <rootclass name="HouseNoDestruct" reportMemoryLOD="no" />
        <rootclass name="SurvivorBase"  act="character" reportMemoryLOD="no" />
        <rootclass name="DZ_LightAI"    act="character" reportMemoryLOD="no" />
        <rootclass name="CarScript"     act="car"       reportMemoryLOD="no" />
        <rootclass name="BoatScript"    act="car"       reportMemoryLOD="no" />
    </classes>
    <defaults>
        <default name="dyn_radius" value="30" />
        <default name="log_ce_lootspawn" value="false"/>
        <default name="save_types_startup" value="true"/>
        <!-- … -->
    </defaults>
    <ce folder="MyMod_CE">
        <file name="MyMod_Types.xml"          type="types" />
        <file name="MyMod_SpawnableTypes.xml" type="spawnabletypes" />
    </ce>
</economycore>
```

### The `<ce folder>` block is the answer to "how do I ship types.xml with my mod"

A mod **cannot** ship `db\types.xml` — it is a mission file, not a PBO file, and
overwriting the server owner's copy would destroy their edits. The supported
pattern is:

1. Ship a folder of CE files with your mod (e.g. `MyMod_CE\MyMod_Types.xml`).
2. Instruct the server owner to drop that folder into the mission root and add a
   single `<ce folder="MyMod_CE">` block to `cfgeconomycore.xml`.

Your entries are then merged at load. The owner keeps their `db\types.xml`
untouched, and uninstalling your mod is one deleted folder plus one deleted block.

`type=` values verified in use: `types`, `spawnabletypes`. Others exist for events
and random presets **(convention — not observed in the sampled missions)**.

### Root classes

An entity only participates in the CE if it inherits from a declared `rootclass`.
A custom item extending `Inventory_Base` is covered. A custom vehicle must extend
`CarScript` (`act="car"`) or the CE will not persist or respawn it.

---

## 3. `types.xml` — the Per-Item Contract

```xml
<type name="MyCustomRifle">
    <nominal>8</nominal>
    <lifetime>10800</lifetime>
    <restock>1800</restock>
    <min>4</min>
    <quantmin>-1</quantmin>
    <quantmax>-1</quantmax>
    <cost>100</cost>
    <flags count_in_cargo="0" count_in_hoarder="0" count_in_map="1"
           count_in_player="0" crafted="0" deloot="0"/>
    <category name="weapons"/>
    <usage name="Military"/>
    <usage name="Police"/>
    <value name="Tier3"/>
    <value name="Tier4"/>
</type>
```

| Element | Meaning | Notes |
|---|---|---|
| `nominal` | Target number of this item alive in the world | The CE spawns toward this number |
| `min` | Refill trigger — CE tops up when the count falls to this | `min` > 0 with `nominal` 0 does nothing useful |
| `lifetime` | Seconds an untouched item survives before cleanup | 10800 = 3 h |
| `restock` | Seconds before the CE may respawn after removal | `0` = immediately eligible |
| `quantmin` / `quantmax` | Fill percentage range for quantity items | `-1` = not applicable |
| `cost` | Spawn priority weight | Higher = more likely when several types compete |
| `category` | One value from `cfglimitsdefinition.xml` | Wrong name ⇒ file rejected |
| `usage` | Location classes where it may spawn | Multiple allowed |
| `value` | Tier(s) | Multiple allowed |
| `tag` | `floor` / `shelves` / `ground` | Restricts container type |

### Flags

| Flag | `1` means |
|---|---|
| `count_in_cargo` | Copies inside containers count toward `nominal` |
| `count_in_hoarder` | Copies in tents/barrels/stashes count |
| `count_in_map` | Copies lying in the world count |
| `count_in_player` | Copies in player inventories count |
| `crafted` | Item is craftable — CE will not spawn it |
| `deloot` | Item is dynamic-event loot only |

The usual mistake: `count_in_player="1"` on a common item. Players hoard it, the
CE believes the target is met, and the world empties out.

---

## 4. Controlled Vocabularies — `cfglimitsdefinition.xml`

**A single invalid name can make the engine reject the whole `types.xml`.** These
are the complete verified lists:

**Categories:** `tools`, `containers`, `clothes`, `lootdispatch`, `food`,
`weapons`, `books`, `explosives`

**Tags:** `floor`, `shelves`, `ground`

**Usage flags:** `Military`, `Police`, `Medic`, `Firefighter`, `Industrial`,
`Farm`, `Coast`, `Town`, `Village`, `Hunting`, `Office`, `School`, `Prison`,
`Lunapark`, `SeasonalEvent`, `ContaminatedArea`, `Historical`

**Value flags (tiers):** `Tier1`, `Tier2`, `Tier3`, `Tier4`, `Unique`

Names are case-sensitive. `military` is not `Military`.

Custom values require editing `cfglimitsdefinition.xml` itself — which is a
mission file, so the same "instruct the server owner" rule from §2 applies.
`cfglimitsdefinitionuser.xml` defines named groupings over these values for use
in `mapgroupproto.xml`.

---

## 5. `cfgspawnabletypes.xml` — What an Item Carries When It Spawns

`types.xml` decides *whether* something spawns. `cfgspawnabletypes.xml` decides
what is *in* it and what condition it is in.

```xml
<type name="PlateCarrierVest_Camo">
    <damage min="0.1" max="0.6" />
    <attachments chance="0.85">
        <item name="PlateCarrierHolster_Camo" chance="1.00" />
    </attachments>
    <attachments chance="0.85">
        <item name="PlateCarrierPouches_Camo" chance="1.00" />
    </attachments>
</type>

<type name="Barrel_Blue">
    <hoarder />
</type>
```

- Each `<attachments>` block is an **independent roll** — the outer `chance` is
  whether the slot is filled at all, the inner per-item `chance` picks which one.
- `<damage min max>` randomises spawn health (0 = pristine, 1 = ruined).
- `<hoarder />` marks the type as a storage container for `count_in_hoarder`.
- `<cargo>` blocks fill container contents, and can reference a named preset from
  `cfgrandompresets.xml`.

`cfgrandompresets.xml` holds reusable bundles:

```xml
<cargo chance="0.15" name="foodHermit">
    <item name="TunaCan"     chance="0.11" />
    <item name="SardinesCan" chance="0.11" />
    <item name="Apple"       chance="0.07" />
</cargo>
```

A custom weapon that should spawn with a magazine belongs here, not in
`types.xml`.

---

## 6. Events — `db\events.xml` + `cfgeventspawns.xml`

An event is a spawner: vehicles, wrecks, animal herds, infected groups,
contaminated areas.

```xml
<event name="StaticHeliCrash">
    <nominal>3</nominal>
    <min>0</min>
    <max>0</max>
    <lifetime>2100</lifetime>
    <restock>0</restock>
    <saferadius>1000</saferadius>
    <distanceradius>1000</distanceradius>
    <cleanupradius>1000</cleanupradius>
    <secondary>InfectedArmy</secondary>
    <flags deletable="1" init_random="0" remove_damaged="0"/>
    <position>fixed</position>
    <limit>child</limit>
    <active>1</active>
    <children>
        <child lootmax="15" lootmin="10" max="3" min="1" type="Wreck_UH1Y"/>
    </children>
</event>
```

| Element | Meaning |
|---|---|
| `nominal` | How many instances of the event are alive at once |
| `saferadius` | Minimum distance from players to spawn |
| `distanceradius` | Minimum distance between two instances of this event |
| `cleanupradius` | Radius cleaned when the event is removed |
| `secondary` | Another event triggered alongside (here: infected around the crash) |
| `position` | `fixed` (uses `cfgeventspawns.xml`) or `player` |
| `limit` | `custom` / `child` / `mixed` — how `nominal` is counted |
| `children` | The actual classnames, with per-child min/max and loot counts |
| `active` | `1` enables the event |

Positions live in `cfgeventspawns.xml`, keyed by the same event name:

```xml
<eventposdef>
    <event name="VehicleCivilianSedan">
        <pos x="12071.933594" z="9129.989258" a="317.953339" />
    </event>
</eventposdef>
```

`x`/`z` are world coordinates (DayZ is Y-up: `z` is the second ground axis),
`a` is yaw in degrees.

Event naming convention: an event that spawns vehicles is conventionally named
`Vehicle*`, infected `Infected*`, animals `Ambient*` **(convention)**.

---

## 7. Building Loot — `mapgroupproto.xml`

Loot does not spawn "in a building" — it spawns at explicit points defined per
building type:

```xml
<group name="Land_House_2B02">
    <usage name="Village" />
    <container name="lootFloor">
        <category name="tools" />
        <category name="containers" />
        <category name="clothes" />
        <tag name="floor" />
        <point pos="1.475813 -5.575520 -3.226645" range="1.199951" height="2.010249" />
        <!-- 8 points total for this container -->
    </container>
    <container name="lootshelves">
        <tag name="shelves" />
        <!-- … -->
    </container>
</group>
```

`mapgrouppos.xml` then places instances of `Land_House_2B02` at map coordinates.

**Consequence for custom buildings:** shipping a building `.p3d` with `pointfloor`
memory points is not enough. Without a `mapgroupproto.xml` group for the
classname, no loot will ever spawn inside it. (The same house has 132 `pointfloor`
memory points but only 8 `lootFloor` proto entries — the two are not generated
from each other at runtime.) See `model-asset-pipeline.md` §5.

---

## 8. `db\globals.xml` and `db\economy.xml`

`globals.xml` — verified vanilla values worth knowing:

| Variable | Vanilla | Meaning |
|---|---|---|
| `CleanupLifetimeDefault` | 45 | Seconds before a dropped item with no `lifetime` is deleted |
| `CleanupLifetimeRuined` | 330 | Ruined items |
| `CleanupLifetimeDeadPlayer` | 3600 | Player corpses |
| `CleanupAvoidance` | 100 | Radius around players where cleanup is suppressed |
| `LootSpawnAvoidance` | 100 | Radius around players where loot will not spawn |
| `ZombieMaxCount` | 1000 | Infected cap |
| `AnimalMaxCount` | 200 | Animal cap |
| `RespawnAttempt` / `RespawnLimit` / `RespawnTypes` | 2 / 20 / 12 | Loot respawn batching |
| `FlagRefreshFrequency` / `FlagRefreshMaxDuration` | 432000 / 3456000 | Base-building flag lifetime refresh |
| `TimeLogin` / `TimeLogout` | 15 / 15 | Login and logout timers |
| `IdleModeStartup` / `IdleModeCountdown` | 1 / 60 | Empty-server idle mode |

`type="0"` is integer, `type="1"` is float.

`economy.xml` switches whole subsystems:

```xml
<economy>
    <dynamic  init="1" load="1" respawn="1" save="1"/>
    <animals  init="1" load="0" respawn="1" save="0"/>
    <zombies  init="1" load="0" respawn="1" save="0"/>
    <vehicles init="1" load="1" respawn="1" save="1"/>
    <building init="1" load="1" respawn="0" save="1"/>
    <player   init="1" load="1" respawn="1" save="1"/>
</economy>
```

Setting `vehicles save="0"` is how servers stop persisting vehicles — a common
support question that has nothing to do with any mod.

---

## 9. `init.c` — the Mission Entry Point

Verified vanilla shape:

```c
void main()
{
    Hive ce = CreateHive();
    if ( ce )
        ce.InitOffline();
    // … date handling …
}

class CustomMission: MissionServer
{
    override PlayerBase CreateCharacter(PlayerIdentity identity, vector pos,
                                        ParamsReadContext ctx, string characterName)
    { /* … */ }

    override void StartingEquipSetup(PlayerBase player, bool clothesChosen)
    { /* starting gear */ }
};

Mission CreateCustomMission(string path)
{
    return new CustomMission();
}
```

Notes:

- **`void main()` is mandatory.** Without it the server logs *"Mission script has
  no main function. PlayerConnect will stay disabled"* and nobody can join.
- `CreateHive()` + `InitOffline()` is what starts the CE. Removing it disables
  loot spawning entirely.
- Starting gear lives in `StartingEquipSetup`, and `init.c` is a **mission** file.
  A mod that wants to change the spawn loadout should do it from a
  `modded class MissionServer` inside the mod, not by asking every server owner to
  hand-edit `init.c`.
- `init.c` is compiled as part of the mission, so it participates in the normal
  script layer rules and can be broken by a mod that changes `MissionServer`.

---

## 10. Shipping a Custom Item: the Full CE Checklist

- [ ] `config.cpp` entry has `scope = 2` (scope 0 = never spawns, scope 1 = spawns
      only as an attachment/cargo)
- [ ] `types.xml` entry exists, with `nominal` > 0 and `min` > 0
- [ ] `category` / `usage` / `value` names copied exactly from
      `cfglimitsdefinition.xml`
- [ ] Flags reflect reality (`count_in_cargo` for attachments, `crafted="1"` if it
      is only craftable)
- [ ] `cfgspawnabletypes.xml` entry if it should spawn with attachments or cargo
- [ ] The CE files ship in a `<ce folder="MyMod_CE">` block, not as a replacement
      `db\types.xml`
- [ ] Custom buildings have a `mapgroupproto.xml` group, or their loot points are
      dead
- [ ] Custom vehicles inherit from a declared `rootclass` (`CarScript`)
- [ ] Server owner backed up `storage_<instanceId>\` before the first restart with
      the new economy

---

## Symptom → Cause Quick Table

| Symptom | First thing to check |
|---|---|
| Custom item never spawns | `scope = 2`? `types.xml` entry present? `nominal`/`min` > 0? |
| A whole `types.xml` seems ignored | One invalid `category`/`usage`/`value` name rejects the file — check the RPT |
| Item spawns, but always empty | Needs a `cfgspawnabletypes.xml` entry |
| Item spawns once and never again | `restock` too high, or `count_in_player` inflating the live count |
| Loot dries up after a while | `count_in_*` flags counting hoarded copies toward `nominal` |
| Custom building has no loot | Missing `mapgroupproto.xml` group |
| Vehicles disappear on restart | `economy.xml` `<vehicles save="0">` |
| Nobody can join, no script error | `init.c` missing `void main()` |
| Items vanish seconds after dropping | `CleanupLifetimeDefault` (45 s) — the type has no `lifetime` |
| Mod's CE edits lost on server update | The mod overwrote `db\types.xml` instead of using `<ce folder>` |
