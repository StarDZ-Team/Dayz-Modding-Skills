# DayZ Mod Development Workflow

Systematic engineering discipline adapted for DayZ's manual verification environment. Based on Superpowers principles: plan before code, verify before claiming, systematic over ad-hoc.

---

## Core Philosophy

DayZ modding has:
- **No automated test framework** — Validation is manual (build → launch → observe)
- **No linter** for Enforce Script — The compiler is your only static checker
- **No CI/CD** — PBO building and testing is local
- **Script logs as feedback** — `SCRIPT (E)` errors and `Print()` output are your test results

This means discipline must come from process, not tooling. Every step below compensates for the lack of automated safety nets.

---

## Phase 1: Brainstorm & Plan

### Before ANY Code

1. **Define the goal clearly** — What should happen? On server, client, or both?
2. **Identify affected systems** — Which engine APIs, which vanilla classes to mod?
3. **Consult references** — Read the wiki chapter or vanilla scripts for the relevant system
4. **Map the architecture:**
   - Which script layers? (3_Game / 4_World / 5_Mission)
   - Which existing classes to `modded`?
   - What new classes are needed?
   - What RPCs are needed for client-server communication?
   - What configs need to be created or loaded?
5. **Present the plan** — For complex features, outline the design to the user before coding

### Design Document Template (Complex Features)

```markdown
## Feature: [Name]

### Goal
[One sentence: what does this feature do?]

### Architecture
- Layer placement: [3_Game / 4_World / 5_Mission for each class]
- New classes: [list with parent class]
- Modded classes: [list with what's being added/changed]
- RPCs: [list with direction: client→server, server→client]
- Configs: [JSON files with path]

### Dependencies
- Mod dependencies: [requiredAddons additions]
- Cross-mod: [#ifdef guards needed]

### Implementation Steps
1. [Step 1 — specific, small, verifiable]
2. [Step 2]
3. ...

### Verification
- Build succeeds
- No SCRIPT (E) errors
- [Specific in-game test steps]
```

---

## Phase 2: Implement with Defensive Coding

### The Defensive Coding Protocol

Since there are no automated tests, the CODE ITSELF must be defensive. Every function is its own safety net.

#### Rule 1: Guard Every Entry Point

```c
void OnPlayerAction(Man man, int actionId)
{
    // Guard: null check
    if (!man) return;

    // Guard: type check
    PlayerBase player;
    if (!Class.CastTo(player, man)) return;

    // Guard: context check
    if (!GetGame().IsServer()) return;

    // Guard: state check
    if (!player.IsAlive()) return;

    // Guard: permission check
    PlayerIdentity identity = player.GetIdentity();
    if (!identity) return;

    // ALL guards passed — safe to execute
    ProcessAction(player, actionId);
}
```

#### Rule 2: Log at Boundaries

Every significant operation should log entry, success, and failure:

```c
void InitializeSystem()
{
    SDZ_Log.Info("MyMod", "Initializing system...");

    if (!LoadConfig())
    {
        SDZ_Log.Error("MyMod", "Failed to load config, system disabled");
        return;
    }

    if (!RegisterRPCs())
    {
        SDZ_Log.Error("MyMod", "Failed to register RPCs");
        return;
    }

    SDZ_Log.Info("MyMod", "System initialized successfully");
}
```

#### Rule 3: Fail Gracefully, Never Crash

```c
// BAD: assumes everything works
void ProcessItems()
{
    array<ItemBase> items = GetAllItems();
    for (int i = 0; i < items.Count(); i++)
    {
        items[i].DoSomething();  // Crash if item is null!
    }
}

// GOOD: defensive iteration
void ProcessItems()
{
    array<ItemBase> items = GetAllItems();
    if (!items) return;

    for (int i = 0; i < items.Count(); i++)
    {
        ItemBase item = items[i];
        if (!item) continue;

        item.DoSomething();
    }
}
```

#### Rule 4: Validate RPC Data

Never trust data received over RPC:

```c
void OnRPC_SetValue(CallType type, ParamsReadContext ctx, PlayerIdentity sender, Object target)
{
    // Validate context
    if (type != CallType.Server) return;
    if (!sender) return;

    // Validate data reads
    Param2<string, float> data = new Param2<string, float>("", 0);
    if (!ctx.Read(data)) return;

    // Validate values
    string key = data.param1;
    float value = data.param2;

    if (key == "") return;
    if (value < 0 || value > 1000) return;  // Sane bounds

    // Validate permissions
    if (!HasPermission(sender.GetPlainId(), "mymod.setvalue")) return;

    // ALL validation passed
    ApplyValue(key, value);
}
```

---

## Phase 3: Build & Verify

### Build Commands

```bash
# Build all PBOs
python dev.py build

# Check latest script log for errors
python dev.py check

# Launch server and monitor logs
python dev.py server

# Real-time log tail
python dev.py watch

# Full cycle: build + server + client
python dev.py full

# Kill processes if stuck
python dev.py kill
```

### Verification Protocol

After every significant change:

1. **Build:** `python dev.py build` — Must succeed with no errors
2. **Check logs:** `python dev.py check` — Must show no `SCRIPT (E)` errors
3. **Runtime test:** `python dev.py server` — Monitor logs for runtime errors
4. **In-game verify:** Test the specific feature that was changed

### What to Look For in Logs

```
SCRIPT (E):     → Compilation error (code won't load)
SCRIPT ERROR:   → Runtime error (null ref, bad cast, etc.)
[StarDZ]        → StarDZ mod log output
WARNING:        → Non-fatal issues (may indicate problems)
```

---

## Phase 4: Systematic Debugging

When something breaks, do NOT guess. Follow this protocol.

### Step 1: Read the Error

```bash
python dev.py check    # Find the error message and file:line
```

Common error patterns:
- `Undefined variable 'X'` → Variable not declared or wrong layer
- `Cannot convert from 'X' to 'Y'` → Wrong type, missing cast
- `Member not found 'X'` → Method doesn't exist on that type
- `Definition 'X' was already defined` → Duplicate class name or redeclaration
- `Attempt to access null reference` → Missing null check

### Step 2: Trace the Cause

1. Read the code at the error location
2. Trace backward through the call chain
3. Check: Is this a layer violation? Lifecycle issue? Null ref? Wrong type?
4. Find working examples of similar code — what's different?

### Step 3: Fix and Verify

1. Make ONE change at a time
2. Add `SDZ_Log` calls to instrument the failing path
3. Rebuild: `python dev.py build`
4. Check: `python dev.py check`
5. Test: `python dev.py server`

### Step 4: Escalate

If 3+ fix attempts fail:

1. **STOP guessing** — Your mental model of the problem is wrong
2. **Re-read the API** — Consult wiki chapter or vanilla scripts
3. **Search vanilla scripts** — `Grep` for how the engine uses this API
4. **Ask the user** — They can test in-game and report what happens
5. **Simplify** — Strip the code to the minimum that reproduces the issue

---

## Phase 5: Code Review Protocol

Before committing any code, run through this checklist:

### Enforce Script Rules
- [ ] No ternary `? :`
- [ ] No `do...while` loops
- [ ] No `try/catch` blocks
- [ ] No backslashes in string literals
- [ ] No variable redeclaration in sibling if/else blocks
- [ ] No `#include` directives
- [ ] `string.Format()` used (not interpolation)

### Type Safety
- [ ] All downcasts use `Class.CastTo()`
- [ ] `GetGame().GetPlayer()` cast to `PlayerBase`
- [ ] `IsAlive()` only called on `EntityAI` or subclass
- [ ] `JsonFileLoader.JsonLoadFile()` not assigned to return value
- [ ] Float-to-int conversions use `Math.Round()` where needed

### Memory Safety
- [ ] No `ref` cycles (A→B→A)
- [ ] No `autoptr` usage (use `ref` instead)
- [ ] Static refs nulled in `DestroyInstance()` or cleanup
- [ ] Singletons destroyed in `OnMissionFinish`
- [ ] ScriptInvoker listeners removed on cleanup

### Architecture
- [ ] Code in correct layer (3_Game / 4_World / 5_Mission)
- [ ] No upward layer references (3_Game using 4_World types)
- [ ] `requiredAddons[]` includes all mod dependencies
- [ ] Server/client context validated with `GetGame().IsServer()`
- [ ] RPC data validated on receive side
- [ ] Permissions checked before privileged operations

### Style
- [ ] `m_` prefix on member variables
- [ ] `s_` prefix on static variables
- [ ] PascalCase for classes and methods
- [ ] Guard clauses before main logic
- [ ] Logging at error paths

---

## Anti-Patterns to Avoid

### 1. Testing by Assumption
**Bad:** "This should work because the logic looks right."
**Good:** Build it, check logs, test in-game. Evidence over assumption.

### 2. Shotgun Debugging
**Bad:** Change 5 things at once hoping one fixes it.
**Good:** One change → rebuild → test. Isolate the variable.

### 3. Ignoring Layer Rules
**Bad:** "I'll just put everything in 5_Mission, it compiles there."
**Good:** Place code in the lowest appropriate layer. It prevents circular dependencies and makes the mod more maintainable.

### 4. Skipping Cleanup
**Bad:** "The server restarts anyway, who cares about memory."
**Good:** Missions restart without process restart. Stale singletons WILL cause bugs in the next mission cycle.

### 5. Trusting RPC Input
**Bad:** Use RPC data directly without validation.
**Good:** Validate type, range, permissions, and context on every RPC receive.

### 6. Copy-Pasting Without Understanding
**Bad:** Copy vanilla code and modify blindly.
**Good:** Read the vanilla code, understand what each line does, then adapt with full understanding. Consult the wiki for API explanations.

---

## When to Consult References

| Situation | Reference |
|-----------|-----------|
| Unfamiliar API method | `docs/wiki/en/06-engine-api/` or Grep `scripts/` |
| GUI / widget work | `docs/DAYZ_GUI_REFERENCE.md` |
| CF module system | `docs/CF_COMPLETE_REFERENCE.md` |
| Compilation error you don't understand | `docs/wiki/en/troubleshooting.md` |
| Designing a new system | `docs/wiki/en/07-patterns/` |
| First time building a mod type | `docs/wiki/en/08-tutorials/` |
| Quick syntax lookup | `docs/wiki/en/cheatsheet.md` |
| Term you don't know | `docs/wiki/en/glossary.md` |

**Rule of thumb:** If you're about to use an API you haven't used in this session, take 30 seconds to read the reference first. It's faster than debugging a wrong assumption.
