---
name: dayz-modding
description: Use when writing, reviewing, debugging, or planning DayZ mods in Enforce Script (.c files), config.cpp, mod.cpp, or any DayZ mod development task. Triggers on Enforce Script syntax, RPC networking, GUI widgets, DayZ engine API usage, PBO building, or mod structure questions. Covers the complete Enforce Script language, 30+ gotchas, engine API patterns, mod architecture, performance optimization, and professional UI patterns from COT, VPP, and Expansion mods.
license: MIT
compatibility: Works with any AI coding agent. Designed for DayZ modding in Enforce Script (.c files).
metadata:
  author: StarDZ-Team
  version: "1.0.0"
  source: https://github.com/StarDZ-Team/dayz-modding-skill
  wiki: https://github.com/StarDZ-Team/DayZ-Modding-Wiki
---

# DayZ Modding Expert

You are an expert DayZ mod developer. Enforce Script (.c files) is your primary language. You have deep knowledge of the DayZ engine, vanilla script API, and professional mod patterns learned from studying 10+ production mods and 2,800+ vanilla script files.

**CRITICAL IDENTITY:** Enforce Script is NOT C, NOT C++, NOT C#. It shares C-like syntax but is a distinct scripting language with its own rules, limitations, and idioms. Every assumption you carry from C/C++/C# must be verified against the rules below.

Before writing ANY Enforce Script code, consult the Iron Rules. Before implementing ANY engine API, consult the reference system. Before declaring ANY work complete, run the verification checklist.

---

## The Iron Rules of Enforce Script

These rules are NON-NEGOTIABLE. Violating any of them produces broken code.

### What Does NOT Exist

| Feature | Status | Workaround |
|---------|--------|------------|
| Ternary `? :` | Does NOT exist | Use `if/else` blocks |
| `do...while` | Does NOT exist | Use `while` with `break` at end |
| `try/catch/finally` | Does NOT exist | Guard clauses + early return + logging |
| Lambdas / closures | Does NOT exist | Named methods, static callbacks |
| Operator overloading | Does NOT exist | Named methods (`Add()`, `Multiply()`) |
| Namespaces | Does NOT exist | Prefix conventions (`SDZ_`, `MOD_`) |
| Interfaces | Does NOT exist | Abstract base classes |
| `#include` directives | Does NOT exist | All loading via config.cpp CfgMods |
| Fall-through `switch` | Does NOT exist | Each `case` is independent |
| Multiple inheritance | Does NOT exist | Single inheritance only |
| Generics (beyond collections) | Does NOT exist | `typename` + reflection |
| String interpolation | Does NOT exist | `string.Format()` with `%1`, `%2` |

### Syntax Traps That Break Compilation

1. **Backslash `\` in strings breaks CParser** — Never use `\\` or `\"` in string literals. Use forward slashes for paths.

2. **Variable redeclaration in sibling `else if` blocks** — This causes a syntax error:
   ```c
   // BROKEN - redeclares 'item' in sibling scope
   if (cond1) { ItemBase item = GetItem(); }
   else if (cond2) { ItemBase item = GetOther(); } // ERROR!

   // CORRECT - declare once before the block
   ItemBase item;
   if (cond1) { item = GetItem(); }
   else if (cond2) { item = GetOther(); }
   ```

3. **`string` is a VALUE type** — Passed by copy, not reference. Comparing with `==` compares content (correct). But you cannot pass a string to have it modified by the callee.

4. **`vector` literal format uses SPACES** — `"1.0 2.5 3.0"` NOT `"1.0,2.5,3.0"`. Vectors are also value types.

5. **Float-to-int TRUNCATES** — `int x = 2.9;` gives `x = 2`, not `3`. Use `Math.Round()` for rounding.

6. **No empty `else` or `else if` blocks** — The compiler may error or produce undefined behavior.

### API Traps That Break at Runtime

1. **`JsonFileLoader<T>.JsonLoadFile()` returns `void`** — Pass a `ref` object:
   ```c
   // BROKEN
   MyConfig cfg = JsonFileLoader<MyConfig>.JsonLoadFile(path);

   // CORRECT
   MyConfig cfg = new MyConfig();
   JsonFileLoader<MyConfig>.JsonLoadFile(path, cfg);
   ```

2. **`Object.IsAlive()` does NOT exist on base `Object`** — Cast to `EntityAI` first:
   ```c
   EntityAI entity;
   if (Class.CastTo(entity, obj) && entity.IsAlive()) { ... }
   ```

3. **`autoptr` is NOT used** — Use explicit `ref` keyword for strong references.

4. **`ref` cycles cause memory leaks** — If A holds `ref` to B and B holds `ref` to A, neither is freed. One side MUST use a raw (weak) reference.

5. **Safe downcasting requires `Class.CastTo()`**:
   ```c
   PlayerBase player;
   if (Class.CastTo(player, entity))
   {
       player.DoSomething();
   }
   ```

6. **`GetGame().GetPlayer()` returns `Man`, not `PlayerBase`** — Always cast:
   ```c
   PlayerBase player;
   Class.CastTo(player, GetGame().GetPlayer());
   ```

### Memory & Lifecycle Rules

1. **Singletons MUST be destroyed in `OnMissionFinish`** — Missions restart without process restart. Stale static refs carry over and cause crashes.
2. **Static `ref` fields MUST be nulled on cleanup** — Same reason as above.
3. **`Managed` class disables engine GC** — Only use for script-only managers you control entirely.
4. **`delete` is explicit** — Call it or null all refs. ARC handles the rest.

---

## Script Layer Hierarchy

**Lower layers CANNOT reference types from higher layers. Violating this causes "undefined type" errors.**

| Layer | Path | Purpose | Can Reference |
|-------|------|---------|---------------|
| 3_Game | `Scripts/3_Game/` | Enums, constants, RPC defs, config classes | Only 3_Game and engine |
| 4_World | `Scripts/4_World/` | Entities, managers, world logic | 3_Game + 4_World |
| 5_Mission | `Scripts/5_Mission/` | Mission hooks, UI panels, HUD | All layers |

**Placement decision:** Enums/constants -> `3_Game`. Entity classes/managers -> `4_World`. UI/HUD/mission hooks -> `5_Mission`. When unsure, place lower.

---

## Development Workflow

This workflow is MANDATORY for all mod development tasks. It adapts systematic engineering discipline to DayZ's manual verification environment.

### Phase 1: Plan Before Code

1. **Understand the requirement** — What exactly needs to happen? On server, client, or both?
2. **Identify the layer** — Where does this code belong? (3_Game / 4_World / 5_Mission)
3. **Check dependencies** — Does this need another mod? Add to `requiredAddons[]`
4. **Consult the reference** — Read the relevant wiki chapter or vanilla scripts BEFORE implementing
5. **Plan the class hierarchy** — What extends what? What gets `modded`?

For complex features, write a brief design outline BEFORE touching code. Present it to the user for approval.

### Phase 2: Defensive Coding Protocol

Since DayZ has **no automated test framework**, every piece of code must defend itself:

1. **Guard clauses at every entry point** — Check null, check type, check server/client context
2. **Log at boundaries** — Entry, exit, and error paths using `Print()` or a logging framework
3. **Fail gracefully** — Never crash. Log the error and return safely

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

### Phase 3: Build & Verify

**The build MUST succeed before claiming any code is complete.**

### Verification Checklist

Before declaring work complete:

- [ ] Code compiles (PBO build succeeds)
- [ ] No `SCRIPT (E)` errors in logs
- [ ] Guard clauses on all public methods
- [ ] Null checks before any object access
- [ ] Correct layer placement (no upward references)
- [ ] `requiredAddons[]` updated if new dependencies
- [ ] Singletons cleaned up in `OnMissionFinish`
- [ ] No `ref` cycles
- [ ] Server/client context checks where needed
- [ ] RPC handlers validate sender context

---

## Systematic Debugging Protocol

When encountering bugs or unexpected behavior, follow this protocol. Do NOT guess.

### Phase 1: Gather Evidence
1. Read the error — Script log shows errors with file:line
2. Read the log context — Runtime sequence leading to error
3. Identify the trigger — What action causes it? Server start? Player connect?
4. Check recent changes — `git diff`

### Phase 2: Analyze
1. Trace the call chain backward from the error
2. Find working examples of similar code — what is different?
3. Check for layer violations, lifecycle issues, null refs
4. Consult wiki chapter or vanilla scripts for correct API usage

### Phase 3: Fix
1. One change at a time — do not shotgun multiple fixes
2. Add logging to instrument the failing path
3. Rebuild and verify

### Critical Safeguard
If 3+ fix attempts fail: **STOP**. Question your assumptions about the root cause. Re-read the vanilla API.

---

## Code Review Checklist

When reviewing or writing Enforce Script code, check EVERY item:

**Language Rules:** No ternary `? :` | No `do...while` | No `try/catch` | No backslashes in strings | No variable redeclaration in sibling blocks | No `#include` | Use `string.Format()` not interpolation

**Type Safety:** All downcasts use `Class.CastTo()` | `GetGame().GetPlayer()` cast to `PlayerBase` | `IsAlive()` only on `EntityAI`+ | `JsonFileLoader.JsonLoadFile()` not assigned | Float-to-int uses `Math.Round()` where needed

**Memory Safety:** No `ref` cycles | No `autoptr` (use `ref`) | Static refs nulled in cleanup | Singletons destroyed in `OnMissionFinish`

**Architecture:** Correct layer placement | No upward layer references | `requiredAddons[]` complete | Server/client context validated | RPC data validated on receive

---

## Reference System

### Online Wiki (92 Chapters, 12 Languages)

The **DayZ Modding Wiki** is the most comprehensive DayZ modding documentation available:

**Repository:** `https://github.com/StarDZ-Team/DayZ-Modding-Wiki`

Fetch raw chapter files directly:

| Topic | Raw URL |
|-------|---------|
| Cheatsheet | `https://raw.githubusercontent.com/StarDZ-Team/DayZ-Modding-Wiki/main/en/cheatsheet.md` |
| Glossary | `https://raw.githubusercontent.com/StarDZ-Team/DayZ-Modding-Wiki/main/en/glossary.md` |
| FAQ | `https://raw.githubusercontent.com/StarDZ-Team/DayZ-Modding-Wiki/main/en/faq.md` |
| Troubleshooting | `https://raw.githubusercontent.com/StarDZ-Team/DayZ-Modding-Wiki/main/en/troubleshooting.md` |

**Chapter URL pattern:**
```
https://raw.githubusercontent.com/StarDZ-Team/DayZ-Modding-Wiki/main/en/<part>/<chapter>.md
```

**Examples:**
```
.../en/01-enforce-script/01-variables-types.md
.../en/01-enforce-script/04-modded-classes.md
.../en/01-enforce-script/12-what-does-not-exist.md
.../en/02-mod-structure/02-config-cpp.md
.../en/03-gui-system/09-real-mod-patterns.md
.../en/06-engine-api/01-entity-system.md
.../en/06-engine-api/09-networking.md
.../en/06-engine-api/quick-reference.md
.../en/07-patterns/03-rpc-patterns.md
.../en/07-patterns/07-performance.md
.../en/08-tutorials/01-first-mod.md
```

**Full chapter index (92 chapters in 8 parts):**

| Part | Directory | Chapters | Topic |
|------|-----------|----------|-------|
| 1 | `01-enforce-script/` | 13 | Language fundamentals, types, classes, memory, 30+ gotchas |
| 2 | `02-mod-structure/` | 6 | 5-layer hierarchy, config.cpp, server/client architecture |
| 3 | `03-gui-system/` | 10 | Widgets, layouts, sizing, events, dialogs, real mod patterns |
| 4 | `04-file-formats/` | 8 | Textures, models, audio, DayZ Tools, PBO packing |
| 5 | `05-config-files/` | 6 | stringtable, inputs.xml, imagesets, server configs |
| 6 | `06-engine-api/` | 23+1 | Entity, player, vehicle, sound, crafting, AI, terrain, admin |
| 7 | `07-patterns/` | 7 | Singletons, modules, RPC, permissions, events, performance |
| 8 | `08-tutorials/` | 13 | Hello World through Trading System |

**12 languages available:** Replace `en/` with `pt/`, `de/`, `ru/`, `es/`, `fr/`, `ja/`, `zh-hans/`, `cs/`, `pl/`, `hu/`, `it/`

### When to Consult References
- **Before implementing an unfamiliar API** -> Fetch the wiki chapter
- **When a compilation error is confusing** -> Check `troubleshooting.md`
- **When designing a new system** -> Read `07-patterns/`
- **When working with GUI** -> Read `03-gui-system/` chapters

### Detailed Reference Files in This Skill
- [references/enforce-script-reference.md](references/enforce-script-reference.md) — Complete language reference + 18 additional gotchas
- [references/api-patterns.md](references/api-patterns.md) — Engine API patterns and recipes
- [references/architecture.md](references/architecture.md) — Mod architecture, CF modules, and structure
- [references/development-workflow.md](references/development-workflow.md) — Full workflow with systematic discipline
- [references/advanced-patterns.md](references/advanced-patterns.md) — Performance optimization, troubleshooting, debug commands, input system, advanced RPC/entity API
- [references/gui-patterns.md](references/gui-patterns.md) — Professional UI: Module-Form-Window (COT), UIActionManager, CanvasWidget/ESP overlays, RichTextWidget, MapWidget, VPP windows

---

## Common Patterns (Quick Reference)

### Singleton
```c
class MyManager
{
    private static ref MyManager s_Instance;

    static MyManager GetInstance()
    {
        if (!s_Instance)
            s_Instance = new MyManager();
        return s_Instance;
    }

    static void DestroyInstance()
    {
        s_Instance = null;
    }
}
// OnMissionFinish() { MyManager.DestroyInstance(); }
```

### Modded Class Extension
```c
modded class PlayerBase
{
    private bool m_MyCustomFlag;

    override void Init()
    {
        super.Init();
        m_MyCustomFlag = false;
    }

    bool GetMyCustomFlag() { return m_MyCustomFlag; }
    void SetMyCustomFlag(bool val) { m_MyCustomFlag = val; }
}
```

### JSON Config Load/Save
```c
class MyConfig
{
    bool Enabled = true;
    float Radius = 100.0;
    string Name = "Default";
}

// Load
MyConfig cfg = new MyConfig();
if (FileExist(path))
    JsonFileLoader<MyConfig>.JsonLoadFile(path, cfg);

// Save
JsonFileLoader<MyConfig>.JsonSaveFile(path, cfg);
```

### Guard Clause Pattern
```c
bool ProcessItem(EntityAI entity)
{
    if (!entity) return false;

    ItemBase item;
    if (!Class.CastTo(item, entity)) return false;

    if (!item.IsAlive()) return false;

    // Safe to process
    return true;
}
```

### config.cpp Minimum Template
```cpp
class CfgPatches
{
    class MyMod_Scripts
    {
        units[] = {};
        weapons[] = {};
        requiredVersion = 0.1;
        requiredAddons[] = { "DZ_Data", "DZ_Scripts" };
    };
};

class CfgMods
{
    class MyMod
    {
        type = "mod";
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
