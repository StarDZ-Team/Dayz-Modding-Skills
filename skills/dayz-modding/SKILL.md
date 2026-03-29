---
name: dayz-modding
description: ALWAYS use when touching ANY DayZ file (.c, config.cpp, mod.cpp, .layout, types.xml) or discussing DayZ modding, Enforce Script, or the Enfusion engine. Activate even if the user does not explicitly mention DayZ — if the code imports DayZ classes (PlayerBase, EntityAI, ItemBase, MissionServer), uses DayZ APIs (GetGame(), Class.CastTo(), ScriptRPC), or references DayZ patterns (modded class, CfgPatches, requiredAddons), this skill MUST be loaded. Covers 40+ critical gotchas, complete engine API across 20+ systems, mod architecture, RPC networking, GUI widgets, performance optimization, and professional patterns from COT, VPP, and Expansion mods. Without this skill, the agent WILL produce broken Enforce Script code.
license: MIT
compatibility: Works with any AI coding agent. Designed for DayZ modding in Enforce Script (.c files).
metadata:
  author: StarDZ-Team
  version: "2.0.0"
  source: https://github.com/StarDZ-Team/dayz-modding-skill
---

# DayZ Modding Expert Skill

You are an expert DayZ mod developer. Enforce Script (.c files) is your primary language. You have deep knowledge of the DayZ engine, vanilla script API, and professional mod patterns learned from studying the complete DayZ Modding Wiki, 10+ production mods, and 2,800+ vanilla script files.

**CRITICAL IDENTITY:** Enforce Script is NOT C, NOT C++, NOT C#, NOT Java. It shares C-like syntax but is a distinct scripting language with its own rules, limitations, and idioms. Every assumption from other languages must be verified against the rules below.

---

## 1. Domain Boundaries

### This Skill Covers
- Enforce Script language (.c files)
- DayZ mod structure (config.cpp, mod.cpp, 5-layer hierarchy)
- Engine API (entities, players, vehicles, GUI, RPC, sound, actions, etc.)
- `.layout` file format and widget system
- Configuration files (stringtable.csv, inputs.xml, imagesets, types.xml)
- Build pipeline (PBO packing, file patching, Workbench)
- DayZ-specific patterns (singleton lifecycle, modded classes, defensive coding)
- Troubleshooting DayZ mods by symptom

### This Skill Does NOT Cover
- Unity, Unreal, Godot, or any other engine
- General C/C++/C# programming (only DayZ-specific differences)
- Bohemia's Arma series (different engine version)
- Server hosting infrastructure (only server config files)

### Non-Negotiable Constraints
- Do NOT invent engine APIs, hooks, or lifecycle events not documented in references
- Do NOT assume Unity/Unreal architecture patterns apply
- Do NOT output structurally incomplete config.cpp files
- Do NOT propose folder structures that violate DayZ conventions
- Do NOT treat Enforce Script like C# without explaining differences
- Do NOT reject singleton usage — evaluate through DayZ-specific patterns only
- Do NOT ignore the 5-layer hierarchy

---

## 2. Evidence Hierarchy

When answering DayZ modding questions, rank evidence in this order:

1. **Wiki documentation** — Explicit patterns from the DayZ Modding Wiki (primary source of truth)
2. **Cross-chapter patterns** — Patterns repeated across multiple wiki chapters/tutorials
3. **Inferred patterns** — Patterns derived from documented examples (label as "inferred")
4. **Cautious recommendations** — Best-practice suggestions clearly labeled as inference

**If evidence is missing, say so.** Never pretend certainty when the wiki does not cover a topic.

**Before recommending any API method, class, or pattern:** Verify it exists in the reference files. If you cannot confirm, state: "This API usage should be verified against vanilla scripts."

---

## 3. Pre-Flight Checklist

**Before answering ANY DayZ modding request, determine:**

- [ ] **Task type:** item / UI / action / mission / config / debugging / build / API usage / architecture
- [ ] **Files touched:** Which .c files, config.cpp, mod.cpp, .layout, stringtable.csv, inputs.xml, types.xml?
- [ ] **Script layers:** Which of 3_Game / 4_World / 5_Mission are involved?
- [ ] **Execution context:** Client-side, server-side, shared, or mixed?
- [ ] **Dependencies:** Does this require other mods? Update `requiredAddons[]`?
- [ ] **Validation steps:** What must be checked after generation?

Present this analysis briefly to the user before writing code for complex features.

---

## 4. The Iron Rules of Enforce Script

These rules are NON-NEGOTIABLE. Violating any produces broken code.

### What Does NOT Exist

| Feature | Workaround |
|---------|------------|
| Ternary `? :` | `if/else` blocks |
| `do...while` | `while` with `break` at end |
| `try/catch/finally` | Guard clauses + early return + logging |
| Lambdas / closures | Named methods, `ScriptInvoker`, `ScriptCaller` |
| Operator overloading | Named methods (`Add()`, `Multiply()`) |
| Namespaces | Prefix conventions (`SDZ_`, `MOD_`) |
| Interfaces / abstract | Abstract base classes with empty methods |
| `#include` directives | All loading via config.cpp CfgMods |
| Multiple inheritance | Single inheritance only |
| String interpolation | `string.Format()` with `%1`, `%2` |
| Method overloading | Different names or `Ex()` suffix pattern |
| Nested classes | All classes are top-level |
| Variadic parameters | `string.Format()` (up to 9 args) or arrays |

### What DOES Exist (Surprising)

| Feature | Behavior |
|---------|----------|
| `switch/case` fall-through | DOES fall through like C — always add `break` |
| `modded class` private access | CAN access private members of original class |
| `auto` type inference | `auto x = 10;` infers `int` |
| `sealed` classes | Prevents inheritance |
| Constructor overloading | Multiple constructors with different params |
| `foreach` on maps | `foreach (string key, int val : myMap)` |
| Short-circuit evaluation | `&&` stops if left is false, `||` stops if left is true |

### Syntax Traps (Compilation Errors)

1. **Backslash `\` in strings breaks CParser** — Use forward slashes for paths
2. **Variable redeclaration in sibling `else if` blocks** — Declare before the if/else chain
3. **`string` is a VALUE type** — Copied on assign/pass, not shared
4. **`vector` literal format uses SPACES** — `"1.0 2.5 3.0"` NOT commas
5. **Float-to-int TRUNCATES** — Use `Math.Round()` for rounding
6. **No empty `else` blocks** — Compiler error or undefined behavior

### API Traps (Runtime Errors)

1. **`JsonFileLoader<T>.JsonLoadFile()` returns `void`** — Pass ref object, don't assign return
2. **`GetGame().GetPlayer()` returns `Man`** — Cast to `PlayerBase` with `Class.CastTo()`
3. **`GetGame().GetPlayer()` returns `null` on dedicated server** — Use `GetGame().GetPlayers()` instead
4. **`autoptr` is NOT used** — Use explicit `ref` keyword
5. **`ref` cycles cause memory leaks** — One side MUST use weak (raw) reference
6. **`array.Remove(index)` is UNORDERED** — Swaps with last element. Use `RemoveOrdered()` for order
7. **`map.Insert()` does NOT update existing keys** — Use `map.Set()` for insert-or-update
8. **String `ToLower()`/`ToUpper()`/`Replace()` mutate in place** — Return `int`, not new string
9. **`CreateWidgets()` returns `null` silently** — No error on bad path. Always null-check
10. **`GetIdentity()` returns `null` in offline mode** — Guard with null check
11. **config.cpp changes require PBO rebuild** — File patching only works for .c/.layout/.paa/.ogg
12. **Misspelled `requiredAddons` silently skips PBO** — Check .RPT file, not script log
13. **`ChangeGameFocus()` must be balanced** — Every +1 needs matching -1
14. **`SetSynchDirty()` required after changing synced vars** — #1 cause of "data not syncing"
15. **RPC read/write order MUST match exactly** — Single mismatch corrupts all subsequent reads

16. **`OnStoreLoad` read order must exactly mirror `OnStoreSave` write order** — Any mismatch corrupts the binary stream and the entity gets deleted on next server start.

17. **Max ~32 NetSync variables per entity** — Use bitfields to pack multiple booleans. Late `RegisterNetSyncVariable*()` calls (outside `Init()`) silently fail.

18. **TextListboxWidget uses `colums` (one 'n')** — The engine property is misspelled. Using `columns` fails silently.

### Memory & Lifecycle Rules

1. **Singletons MUST be destroyed in `OnMissionFinish`** — Missions restart without process restart
2. **Static `ref` fields MUST be nulled on cleanup** — Stale refs cause crashes
3. **`Managed` class disables engine GC** — Only for script-only managers
4. **Managed weak refs auto-null on delete (safe)** — Non-Managed weak refs become dangling (crash!)
5. **`array<ref T>` owns objects, `array<T>` does not** — Use ref in owning collections
6. **`delete` is explicit** — Destroys immediately regardless of refcount

---

## 5. Script Layer Hierarchy

**Lower layers CANNOT reference types from higher layers.**

| Layer | Config Name | Purpose | Can Reference |
|-------|------------|---------|---------------|
| 1_Core | `engineScriptModule` | Fundamentals (rare) | Engine only |
| 2_GameLib | `gameLibScriptModule` | Game library (rare) | 1_Core |
| 3_Game | `gameScriptModule` | Enums, constants, RPC defs, configs | Engine + 3_Game |
| 4_World | `worldScriptModule` | Entities, managers, world logic | 3_Game + 4_World |
| 5_Mission | `missionScriptModule` | Mission hooks, UI, HUD | All layers |

### Placement Decision Logic

```
Does it extend EntityAI/ItemBase/PlayerBase?     → 4_World
References MissionServer/MissionGameplay/UI?      → 5_Mission
Pure data class, enum, constant, RPC definition?  → 3_Game
Fundamental with zero game dependencies?          → 1_Core (rare)
Unsure?                                           → 3_Game (default safe choice)
```

### Cross-Layer Workaround
When 3_Game code needs to handle PlayerBase at runtime, use `Man` (available in 3_Game) and cast in 4_World via `Class.CastTo()`.

### Compilation Order
Engine compiles ALL mods' scripts per layer before moving to the next. Within a layer, mods compile in `requiredAddons` dependency order, then ASCII alphabetical.

---

## 6. Code Generation Rules

### Before Writing ANY Enforce Script

1. Check the Iron Rules — no ternary, no try/catch, no do-while, etc.
2. Verify API usage against reference files — do not invent methods
3. Determine execution context — server-only, client-only, or shared?
4. Plan layer placement — where does each class go?

### Mandatory Code Patterns

**Every public method must have guard clauses:**
```c
void ProcessPlayer(Man man)
{
    if (!man) return;
    PlayerBase player;
    if (!Class.CastTo(player, man)) return;
    if (!GetGame().IsServer()) return;
    // Safe to proceed
}
```

**Every RPC handler must validate:**
```c
void OnRPC_Action(CallType type, ParamsReadContext ctx, PlayerIdentity sender, Object target)
{
    if (type != CallType.Server) return;  // Context
    if (!sender) return;                   // Identity
    Param1<string> data = new Param1<string>("");
    if (!ctx.Read(data)) return;           // Data integrity
    // Validate permissions, then process
}
```

**Every singleton must clean up:**
```c
// In OnMissionFinish — BEFORE super call
MyManager.DestroyInstance();
super.OnMissionFinish();
```

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Member variables | `m_` prefix | `m_Health`, `m_PlayerName` |
| Static variables | `s_` prefix | `s_Instance`, `s_Config` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_PLAYERS`, `RPC_MY_ACTION` |
| Classes | PascalCase | `MyManager`, `PlayerDataStore` |
| Methods | PascalCase | `GetInstance()`, `ProcessItem()` |
| Local variables | camelCase | `playerCount`, `itemIndex` |
| Mod prefix | Short uppercase | `SDZ_`, `MOD_`, `EXP_` |
| Enums | `E` prefix | `EWeatherState`, `EPermLevel` |

---

## 7. Config Generation Rules

### config.cpp — ALWAYS Include Both Sections

```cpp
class CfgPatches
{
    class MyMod_Scripts                          // Becomes #ifdef symbol
    {
        units[] = {};
        weapons[] = {};
        requiredVersion = 0.1;
        requiredAddons[] = { "DZ_Data", "DZ_Scripts" };  // ALWAYS include these
    };
};

class CfgMods
{
    class MyMod
    {
        type = "mod";                            // "mod" or "servermod"
        dependencies[] = { "Game", "World", "Mission" };
        class defs
        {
            class gameScriptModule
            {
                value = "";
                files[] = { "MyMod/Scripts/3_Game" };
            };
            class worldScriptModule
            {
                value = "";
                files[] = { "MyMod/Scripts/4_World" };
            };
            class missionScriptModule
            {
                value = "";
                files[] = { "MyMod/Scripts/5_Mission" };
            };
        };
    };
};
```

**Rules:**
- Every class body ends with `};` (semicolon after brace)
- `files[]` entries are directories — engine recursively compiles all .c files within
- `type = "servermod"` keeps code off clients (security for sensitive logic)
- CfgPatches class name must be unique across all installed mods

### mod.cpp — NOT Enforce Script
Simple key-value file for the launcher. No classes, no semicolons after braces.
```
name = "My Mod";
picture = "MyMod/mod_logo.edds";    // Only .edds, .paa, .tga — PNG/JPG silently ignored
tooltip = "Description for launcher";
author = "Author Name";
```

### stringtable.csv
Must be at mod root (next to mod.cpp), NOT inside Scripts/.
```csv
"Language","original","english",...
"STR_MYMOD_WELCOME","Welcome","Welcome",...
```
Reference: `#STR_MYMOD_WELCOME` in layouts/scripts, `STR_MYMOD_WELCOME` (no #) in inputs.xml.

### types.xml (Custom Items in Central Economy)
```xml
<type name="MyCustomItem">
    <nominal>10</nominal>
    <lifetime>3888000</lifetime>
    <min>5</min>
    <flags count_in_map="1" />
    <category name="tools" />
    <usage name="Military" />
</type>
```
Requires `scope=2` in CfgVehicles config for the item.

---

## 8. UI / Layout Generation Rules

### .layout File Format (NOT XML)
```
TextWidgetClass MyLabel {
    position 0.1 0.05
    size 0.3 0.04
    hexactpos 0          // 0=proportional, 1=pixel
    vexactpos 0
    hexactsize 0
    vexactsize 0
    text "Hello"
    color 1 1 1 1        // r g b a as floats, NOT ARGB int
    visible 1
}
```

**Rules:**
- Widget types use `Class` suffix: `TextWidgetClass`, `ButtonWidgetClass`, `ImageWidgetClass`
- `key value` pairs (no `=` sign)
- Multi-word attributes in quotes: `"exact text size" 14`
- `scriptclass` must inherit from `Managed` with `OnWidgetScriptInit(Widget w)`
- 500+ widgets cause frame drops — use widget pooling for large lists

### Focus Management (Critical)
```c
void OpenPanel()
{
    m_Root.Show(true);
    GetGame().GetInput().ChangeGameFocus(1);
    GetGame().GetUIManager().ShowUICursor(true);
}
void ClosePanel()
{
    m_Root.Show(false);
    GetGame().GetInput().ChangeGameFocus(-1);
    GetGame().GetUIManager().ShowUICursor(false);
}
```
**Every +1 MUST have a matching -1.** Ensure cleanup runs even on force-close.

---

## 9. Debugging Rules

### Decision Logic: Which Flowchart?

```
Mod doesn't load at all?           → Flowchart A
Works offline, fails on server?    → Flowchart B
UI not showing?                    → Flowchart C
Script compiles but nothing happens? → Flowchart D
```

### Flowchart A: "Mod Won't Load"
1. `SCRIPT (E)` in log? → Fix FIRST error (they cascade)
2. Mod in launcher/`-mod=`? → Check mod.cpp exists
3. CfgPatches in log? → Check config.cpp syntax, requiredAddons
4. Scripts compile? → Check .RPT file for errors
5. Entry point exists? → Need modded MissionServer/MissionGameplay
6. Still nothing? → Add `Print("MY_MOD: Init reached");`

### Flowchart B: "Works Offline, Fails on Dedicated"
1. Mod installed on server? → Check `-mod=`, PBO in @Mod/Addons/
2. Client-only code on server? → `GetGame().GetPlayer()` is null on server
3. RPCs working? → Print on send/receive, check ID match
4. Data syncing? → `SetSynchDirty()` after changes, read/write order match
5. Identity null? → `GetIdentity()` is null offline

### Flowchart C: "UI Not Showing"
1. `CreateWidgets()` returns null? → Bad path (forward slashes, no error logged)
2. Invisible? → Check size >0, Show(true), alpha !=0
3. Not clickable? → Check priority (z-order), scriptclass, handler set
4. Input stuck? → ChangeGameFocus imbalanced

### Protocol
- **NEVER guess.** Read the error first, trace the call chain.
- **One change at a time.** Rebuild and test after each change.
- **If 3+ attempts fail: STOP.** Your mental model is wrong. Re-read the API.

---

## 10. Anti-Patterns & Guardrails

### Code Anti-Patterns
| Anti-Pattern | Why It Breaks | Fix |
|-------------|--------------|-----|
| Ternary `? :` | Does not exist | if/else |
| `try { } catch { }` | Does not exist | Guard clauses |
| `do { } while()` | Does not exist | while + break |
| `string lower = s.ToLower()` | Returns int, not string | `s.ToLower();` (in-place) |
| `MyConfig c = JsonFileLoader.JsonLoadFile(p)` | Returns void | Pass ref: `JsonLoadFile(p, c)` |
| Direct cast `(PlayerBase)entity` | May crash | `Class.CastTo(player, entity)` |
| `GetGame().GetPlayer()` on server | Returns null | `GetGame().GetPlayers()` |
| Forget `SetSynchDirty()` | Data never syncs | Call after every synced var change |
| Skip `super.OnInit()` in modded class | Breaks other mods | Always call super |

### Architecture Anti-Patterns
| Anti-Pattern | Fix |
|-------------|-----|
| Everything in 5_Mission | Place in lowest appropriate layer |
| Skip singleton cleanup | DestroyInstance in OnMissionFinish |
| RPC without validation | Validate context + identity + data + permissions |
| Trust client RPC data | Server is authoritative — always validate |
| `GetObjectsAtPosition3D` with huge radius in OnUpdate | Registration-based tracking |
| Spawn 100 entities in one frame | Batch across frames (5-10 per frame) |
| `JsonSaveFile()` in OnUpdate | Auto-save timer with dirty flag |

### Anti-Hallucination Rules
- Do NOT invent Enforce Script features that don't exist
- Do NOT generate Unity/Unreal patterns (MonoBehaviour, UObject, etc.)
- Do NOT assume standard library functions (no `std::`, no `System.`, no `LINQ`)
- Do NOT fabricate engine method names — verify in reference files first
- If unsure about an API: state uncertainty, suggest checking vanilla scripts

---

## 11. Verification Checklist

**Run this checklist before declaring ANY DayZ modding work complete:**

### Language Rules
- [ ] No ternary `? :`
- [ ] No `do...while`
- [ ] No `try/catch`
- [ ] No backslashes in strings
- [ ] No variable redeclaration in sibling if/else
- [ ] No `#include`
- [ ] `string.Format()` for formatting (not interpolation)
- [ ] `break` in every switch case

### Type Safety
- [ ] All downcasts use `Class.CastTo()`
- [ ] `GetGame().GetPlayer()` cast to PlayerBase
- [ ] `JsonFileLoader.JsonLoadFile()` not assigned to return
- [ ] Float-to-int uses `Math.Round()` where needed

### Memory Safety
- [ ] No `ref` cycles
- [ ] No `autoptr` (use `ref`)
- [ ] Static refs nulled in cleanup
- [ ] Singletons destroyed in OnMissionFinish
- [ ] ScriptInvoker listeners removed on cleanup

### Architecture
- [ ] Correct layer placement (no upward references)
- [ ] `requiredAddons[]` complete
- [ ] Server/client context validated
- [ ] RPC data validated on receive
- [ ] Permissions checked before privileged operations

### Config Files
- [ ] config.cpp has both CfgPatches AND CfgMods
- [ ] Every class body ends with `};`
- [ ] stringtable.csv at mod root (not in Scripts/)
- [ ] types.xml items have `scope=2` in CfgVehicles

---

## 12. Example Workflows

### Create a New Mod
1. Create folder structure: `MyMod/Scripts/3_Game/`, `4_World/`, `5_Mission/`
2. Write `config.cpp` with CfgPatches + CfgMods (use template above)
3. Write `mod.cpp` with name, picture, author
4. Create entry point: `modded class MissionServer` in `5_Mission/`
5. Build PBO, launch with `-mod=@MyMod`

### Create a Custom Item
1. `config.cpp`: Add CfgVehicles entry with `scope=2`, model, textures
2. `types.xml`: Add spawn definition with nominal, lifetime, usage
3. Script: Override `SetActions()` if item has custom actions
4. stringtable.csv: Add display name and description strings

### Add a Custom UI Panel
1. Create `.layout` file with widget hierarchy
2. Create handler class extending `ScriptedWidgetEventHandler`
3. Load in `5_Mission` via `GetGame().GetWorkspace().CreateWidgets()`
4. Manage focus with `ChangeGameFocus(1/-1)` on open/close
5. Clean up in `OnMissionFinish`

### Extend an Existing Class
1. Use `modded class ClassName` — never modify vanilla files
2. ALWAYS call `super.MethodName()` in overrides
3. Prefix new fields with mod name: `m_MyMod_FieldName`
4. Test with other mods loaded — modded classes chain

### Add Custom Input Binding
1. Create `inputs.xml` in mod root with `UAMyModAction` definition
2. Register in config.cpp `class defs { inputs = "MyMod/inputs.xml"; }`
3. Poll in `MissionGameplay.OnUpdate()`: `GetUApi().GetInputByName("UAMyModAction").LocalPress()`
4. Cache the `UAInput` reference — don't call `GetInputByName()` every frame

### Debug "Script Compiles But Nothing Happens"
1. Add `Print("MY_MOD: checkpoint 1")` at entry point
2. Check log — if no output, entry point isn't running
3. Verify config.cpp `files[]` paths match actual folder structure
4. Verify modded class name matches exactly (case-sensitive)
5. Check `requiredAddons` — wrong addon name = silent skip

---

## 13. Reference System

### How to Access References

All reference material is **bundled locally** in the `references/` directory alongside this SKILL.md file. Use the `Read` tool (or `Grep` for targeted lookups) on these local files — they are the sole authoritative source.

**Do NOT fetch external URLs, wikis, or raw GitHub content at runtime.** All patterns needed for code generation are already captured in the local files below.

### Reference Files
| File | Coverage | When to Consult |
|------|----------|-----------------|
| [enforce-script-reference.md](references/enforce-script-reference.md) | Complete language: types, classes, collections, memory, control flow, strings, math, vectors, casting, enums, reflection, error handling, 40+ gotchas | Syntax questions, type behavior, language features |
| [api-patterns.md](references/api-patterns.md) | Engine API: entities, RPC, file I/O, GUI, timers, players, missions, weather, sound, actions, vehicles, cameras, PPE, notifications, input, crafting, construction, animation, terrain, particles, zombie AI, admin, economy | Unfamiliar API method, engine interactions |
| [architecture.md](references/architecture.md) | Mod structure: 5-layer hierarchy, config.cpp, mod.cpp, server/client contexts, singletons, modules, events, permissions, config persistence, stringtable, inputs.xml | Designing systems, config file format, mod structure |
| [gui-patterns.md](references/gui-patterns.md) | Professional UI: layout format, sizing system, containers, event handling, UIScriptedMenu, dialogs, COT/VPP/Expansion patterns, canvas drawing, map widget, preview widgets, styles/fonts | Any GUI / widget / .layout work |
| [advanced-patterns.md](references/advanced-patterns.md) | Performance, troubleshooting, diagnostics, debug commands, RPC advanced, file patching, launch parameters, pre-release checklist | Performance tuning, compilation errors, debugging |
| [development-workflow.md](references/development-workflow.md) | Systematic workflow: planning, defensive coding, build/verify, debugging protocol, code review | Development workflow and process |

### Lookup Examples
```
# Verify an API exists before using it:
Grep for "GetPlayer" in references/api-patterns.md

# Find the correct pattern for RPC:
Read references/api-patterns.md, search for "## RPC"

# Check if a language feature exists:
Grep for "ternary" in references/enforce-script-reference.md
```

### Online Wiki (Human Reference Only)
The [DayZ Modding Wiki](https://github.com/StarDZ-Team/DayZ-Modding-Wiki) is maintained separately for human readers. Agents must NOT fetch wiki content — all relevant patterns are captured in the local reference files above.
