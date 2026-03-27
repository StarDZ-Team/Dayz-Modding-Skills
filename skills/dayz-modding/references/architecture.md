# DayZ Mod Architecture Reference

Patterns, structures, and rules for building maintainable DayZ mods.

---

## Mod Folder Structure

### Single Mod (Client + Server Combined)

```
MyMod/
├── mod.cpp                         # Workshop metadata
├── config.cpp                      # CfgPatches + CfgMods (REQUIRED)
├── Scripts/
│   ├── 3_Game/                     # Enums, constants, configs, RPC IDs
│   │   ├── MyConstants.c
│   │   └── MyConfig.c
│   ├── 4_World/                    # Entities, managers, gameplay logic
│   │   ├── MyEntity.c
│   │   └── MyManager.c
│   └── 5_Mission/                  # Mission hooks, UI, HUD
│       ├── MissionServer.c
│       └── MissionGameplay.c
├── gui/
│   └── layouts/                    # Layout XML files
│       └── my_panel.layout
├── data/                           # Textures, models, configs
│   └── textures/
└── key/                            # Mod signing keys
```

### Split Mod (Client + Server Separate)

For mods with heavy server-only logic (AI, missions, etc.):

```
MyMod/                              # Client mod (loaded by all)
├── config.cpp
├── Scripts/
│   ├── 3_Game/                     # Shared types, RPC defs
│   ├── 4_World/                    # Client-side entities, rendering
│   └── 5_Mission/                  # Client UI, input handling

MyModServer/                        # Server mod (server only)
├── config.cpp                      # requiredAddons[] includes MyMod
├── Scripts/
│   ├── 3_Game/                     # Server-only constants
│   ├── 4_World/                    # Server-only logic, spawning, AI brains
│   └── 5_Mission/                  # Server mission hooks
```

**Why split?**
- Clients don't download server logic (smaller download)
- Server logic is hidden (anti-cheat, proprietary systems)
- Clear separation of concerns

---

## Layer Hierarchy Rules

### Strict Compilation Order

```
3_Game    →    4_World    →    5_Mission
(first)                        (last)
```

**THE RULE:** A layer can ONLY reference types defined in itself or LOWER layers.

| If code is in... | Can reference... | CANNOT reference... |
|-------------------|------------------|---------------------|
| `3_Game` | Engine types, 3_Game types | 4_World, 5_Mission |
| `4_World` | Engine, 3_Game, 4_World types | 5_Mission |
| `5_Mission` | Everything | — |

### Common Layer Violations

```c
// VIOLATION: 3_Game referencing 4_World type
// File: Scripts/3_Game/MyConfig.c
class MyConfig
{
    ItemBase m_DefaultItem;  // ERROR: ItemBase is 4_World!
}

// FIX: Use string reference instead
class MyConfig
{
    string m_DefaultItemType = "Apple";  // Resolve at runtime in 4_World
}
```

### What Goes Where

**3_Game (earliest — no entity access):**
- Enum definitions
- Constants and static values
- RPC identifiers and route strings
- Config data classes (plain data, no entity refs)
- Utility functions that don't touch the world

**4_World (middle — entity access, no mission access):**
- Entity subclasses (items, vehicles, buildings)
- Manager classes (spawning, tracking, persistence)
- Modded entity overrides
- Action definitions
- World interaction logic

**5_Mission (latest — full access):**
- `MissionServer` / `MissionGameplay` overrides
- UI panels, HUD elements
- Input handling
- System initialization and cleanup

---

## Singleton Pattern

The most common pattern in DayZ mods. Every manager, system, and service is typically a singleton.

### Standard Implementation

```c
class MyManager
{
    // Private static ref — the single instance
    private static ref MyManager s_Instance;

    // Private members
    private ref map<string, ref MyData> m_DataStore;
    private bool m_Initialized;

    // Private constructor prevents external instantiation
    private void MyManager()
    {
        m_DataStore = new map<string, ref MyData>();
        m_Initialized = false;
    }

    // Lazy initialization
    static MyManager GetInstance()
    {
        if (!s_Instance)
            s_Instance = new MyManager();
        return s_Instance;
    }

    // CRITICAL: Must be called in OnMissionFinish
    static void DestroyInstance()
    {
        if (s_Instance)
        {
            s_Instance.Cleanup();
            s_Instance = null;
        }
    }

    private void Cleanup()
    {
        m_DataStore.Clear();
        m_Initialized = false;
    }

    void Init()
    {
        if (m_Initialized) return;
        // Load configs, register RPCs, etc.
        m_Initialized = true;
    }
}
```

### Lifecycle Integration

```c
modded class MissionServer
{
    override void OnInit()
    {
        super.OnInit();
        MyManager.GetInstance().Init();
    }

    override void OnMissionFinish()
    {
        MyManager.DestroyInstance();  // BEFORE super!
        super.OnMissionFinish();
    }
}
```

### WHY Cleanup Matters

DayZ missions can restart WITHOUT restarting the process. Static variables survive across mission restarts. If you don't null static refs:
- Old data persists into the new mission
- Event handlers fire on destroyed objects
- RPC handlers reference stale state
- Memory leaks accumulate

---

## Module System (CF Pattern)

CommunityFramework provides an auto-instantiation module system:

```c
class MyModule extends CF_ModuleGame  // or CF_ModuleWorld
{
    override void OnInit()
    {
        super.OnInit();
        // Auto-called by CF on startup
    }

    override void OnMissionStart()
    {
        super.OnMissionStart();
        // Mission is active
    }

    override void OnMissionFinish()
    {
        // Cleanup
        super.OnMissionFinish();
    }

    override void OnUpdate(float dt)
    {
        // Per-frame update (if registered)
    }
}
```

CF modules are auto-discovered and instantiated. No manual registration needed.

---

## Event-Driven Architecture

### Custom Event Bus

```c
// Event definition (3_Game)
class MyEvent
{
    string PlayerUID;
    int ActionType;

    void MyEvent(string uid, int action)
    {
        PlayerUID = uid;
        ActionType = action;
    }
}

// Publisher
class MySystem
{
    ref ScriptInvoker m_OnAction = new ScriptInvoker();

    void DoAction(string uid, int type)
    {
        // ... logic ...
        m_OnAction.Invoke(new MyEvent(uid, type));
    }
}

// Subscriber
class MyUI
{
    void Init()
    {
        MySystem.GetInstance().m_OnAction.Insert(OnActionEvent);
    }

    void OnActionEvent(MyEvent evt)
    {
        UpdateDisplay(evt.PlayerUID, evt.ActionType);
    }

    void Cleanup()
    {
        MySystem.GetInstance().m_OnAction.Remove(OnActionEvent);
    }
}
```

### ScriptInvoker Usage

```c
// Declaration
ref ScriptInvoker m_OnChanged = new ScriptInvoker();

// Subscribe
m_OnChanged.Insert(MyCallback);       // Add listener
m_OnChanged.Remove(MyCallback);       // Remove listener

// Fire
m_OnChanged.Invoke(arg1, arg2);       // Call all listeners

// IMPORTANT: Always Remove listeners on cleanup to prevent stale refs
```

---

## Config Persistence Pattern

### JSON-Based Config

```c
// Config class (3_Game — plain data, no entity refs)
class MyModConfig
{
    // Fields with defaults
    bool Enabled = true;
    float SpawnRadius = 150.0;
    int MaxItems = 50;
    string WelcomeMessage = "Welcome!";
    ref array<string> AdminUIDs = new array<string>();

    // Nested config
    ref MySubConfig Advanced = new MySubConfig();
}

class MySubConfig
{
    bool DebugMode = false;
    float TickRate = 5.0;
}

// Manager (4_World)
class MyConfigManager
{
    private static ref MyModConfig s_Config;
    private static const string CONFIG_PATH = "$profile:MyMod/config.json";

    static MyModConfig Get()
    {
        if (!s_Config)
            Load();
        return s_Config;
    }

    static void Load()
    {
        s_Config = new MyModConfig();
        MakeDirectory("$profile:MyMod");

        if (FileExist(CONFIG_PATH))
        {
            JsonFileLoader<MyModConfig>.JsonLoadFile(CONFIG_PATH, s_Config);
        }
        else
        {
            Save();  // Create default config
        }
    }

    static void Save()
    {
        JsonFileLoader<MyModConfig>.JsonSaveFile(CONFIG_PATH, s_Config);
    }
}
```

---

## Permission System Pattern

### Hierarchical Dot-Separated Permissions

```c
class PermissionManager
{
    private ref map<string, ref array<string>> m_PlayerPerms;

    bool HasPermission(string uid, string permission)
    {
        if (!m_PlayerPerms.Contains(uid)) return false;

        array<string> perms = m_PlayerPerms.Get(uid);

        // Wildcard superadmin
        if (perms.Find("*") != -1) return true;

        // Exact match
        if (perms.Find(permission) != -1) return true;

        // Hierarchical check: "admin.players" grants "admin.players.kick"
        array<string> parts = new array<string>();
        permission.Split(".", parts);

        string check = "";
        for (int i = 0; i < parts.Count() - 1; i++)
        {
            if (i > 0) check += ".";
            check += parts[i];
            if (perms.Find(check) != -1) return true;
        }

        return false;
    }
}
```

---

## Cross-Mod Feature Detection

```c
// In config.cpp, define a preprocessor symbol
class CfgPatches
{
    class MyMod_Scripts
    {
        // ...
    };
};

// Other mods check for it
#ifdef MYMOD
    // MyMod is loaded, use its API
    MyModAPI.DoSomething();
#else
    // Fallback behavior
    Print("MyMod not loaded, using defaults");
#endif
```

### StarDZ Convention

```c
#ifdef STARDZ_CORE
    SDZ_Log.Info("MyMod", "Core detected, using unified logging");
#else
    Print("[MyMod] Core not loaded, using Print()");
#endif
```

---

## CommunityFramework (CF) Module System

Many DayZ mods use CF (CommunityFramework) for module lifecycle management. CF auto-discovers and instantiates modules, provides lifecycle hooks, and handles networked variable synchronization.

### CF Module Declaration

```c
// Inherit from CF_ModuleGame (available in 3_Game) or CF_ModuleWorld (4_World)
[CF_RegisterModule(MyModule)]  // Auto-registration attribute
class MyModule extends CF_ModuleWorld
{
    // Lifecycle hooks — called automatically by CF
    override void OnInit()
    {
        super.OnInit();
        EnableUpdate();           // Opt-in to OnUpdate calls
        EnableMissionStart();     // Opt-in to OnMissionStart
        EnableMissionFinish();    // Opt-in to OnMissionFinish
        EnableInvokeConnect();    // Opt-in to player connect events
        EnableInvokeDisconnect(); // Opt-in to player disconnect events
    }

    override void OnMissionStart()
    {
        super.OnMissionStart();
        // Server/client ready
    }

    override void OnUpdate(float dt)
    {
        // Per-frame update (only if EnableUpdate() was called)
    }

    override void OnMissionFinish()
    {
        // Cleanup
        super.OnMissionFinish();
    }

    override void OnInvokeConnect(PlayerBase player, PlayerIdentity identity)
    {
        // Player connected
    }

    override void OnInvokeDisconnect(PlayerBase player)
    {
        // Player disconnecting
    }
}
```

### Retrieving a CF Module

```c
// Get a module instance from anywhere
MyModule mod = CF_Modules<MyModule>.Get();
if (mod)
{
    mod.DoSomething();
}
```

### CF vs Manual Singletons

| Feature | CF Module | Manual Singleton |
|---------|-----------|-----------------|
| Auto-instantiation | Yes (via attribute) | No (manual GetInstance) |
| Lifecycle hooks | Built-in (OnInit, OnUpdate, etc.) | Manual (modded MissionServer) |
| Cleanup | Automatic on mission finish | Must call DestroyInstance manually |
| Event opt-in | EnableUpdate(), EnableInvokeConnect() | Manual timer/hook setup |
| Dependency | Requires CF mod loaded | No external dependency |

**Use CF modules** when: your mod already depends on CF, you need lifecycle hooks, or you want automatic cleanup.

**Use manual singletons** when: you want zero dependencies, your mod is standalone, or you need full control over instantiation timing.

### Networked Variables (CF + Vanilla)

For variables that auto-sync from server to client:

```c
modded class ItemBase
{
    protected int m_CustomState;

    void ItemBase()
    {
        // Register in constructor
        RegisterNetSyncVariableInt("m_CustomState", 0, 10);  // min, max
    }

    void SetCustomState(int state)
    {
        m_CustomState = state;
        SetSynchDirty();  // CRITICAL: tells engine to broadcast
    }

    override void OnVariablesSynchronized()
    {
        super.OnVariablesSynchronized();
        // Called on CLIENT when synced vars change
        UpdateVisuals(m_CustomState);
    }
}
```

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Member variables | `m_` prefix | `m_Health`, `m_PlayerName` |
| Static variables | `s_` prefix | `s_Instance`, `s_Config` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_PLAYERS`, `RPC_MY_ACTION` |
| Classes | PascalCase | `MyManager`, `PlayerDataStore` |
| Methods | PascalCase | `GetInstance()`, `ProcessItem()` |
| Local variables | camelCase | `playerCount`, `itemIndex` |
| Mod prefix | Short uppercase | `SDZ_`, `MOD_`, `EXP_` |
| Enums | `E` prefix + PascalCase | `EWeatherState`, `EPermLevel` |
| RPC routes | `ModName:Action` | `"StarDZ:PlayerReady"` |

---

## Full 5-Layer Hierarchy

The engine has 5 script layers, not 3. Layers 1-2 are rarely needed but important to know:

| Layer | Config Name | Path | Purpose |
|-------|------------|------|---------|
| 1_Core | `engineScriptModule` | `Scripts/1_Core/` | Fundamental constants, logging infrastructure, preprocessor defines |
| 2_GameLib | `gameLibScriptModule` | `Scripts/2_GameLib/` | Game library utilities (rare — used by DabsFramework for MVC) |
| 3_Game | `gameScriptModule` | `Scripts/3_Game/` | Enums, constants, RPC defs, config classes |
| 4_World | `worldScriptModule` | `Scripts/4_World/` | Entities, managers, world logic |
| 5_Mission | `missionScriptModule` | `Scripts/5_Mission/` | Mission hooks, UI panels, HUD |

**Compilation order:** Engine compiles ALL mods' scripts for each layer before moving to the next. Within a layer, mods compile in `requiredAddons` dependency order. Unrelated mods compile in ASCII alphabetical order of CfgMods class name.

**Runtime init sequence:** Engine boots → 1_Core → 2_GameLib → 3_Game (CfgMods entry functions) → 4_World (entities available) → Mission loads → 5_Mission (OnInit fires)

### Layer Placement Decision Guide
- Extends EntityAI? → `4_World`
- References MissionServer or UI widgets? → `5_Mission`
- Pure data/config/enum/RPC? → `3_Game`
- Fundamental constant with zero game deps? → `1_Core`
- None of the above? → `3_Game` (default)

### Cross-Layer Workaround: Cast Through Base Types
When 3_Game code needs to handle an entity that's `PlayerBase` at runtime, use `Man` (defined in 3_Game) and cast later in 4_World:
```c
// In 3_Game — use Man, not PlayerBase
void HandlePlayer(Man man) { /* store reference */ }

// In 4_World — cast to PlayerBase
PlayerBase player;
if (Class.CastTo(player, man)) { player.DoSomething(); }
```

---

## config.cpp Deep Dive

### CfgPatches Field Reference
```cpp
class CfgPatches
{
    class MyMod_Scripts      // This name becomes a #ifdef symbol
    {
        units[] = {};        // Populates editor/spawner lists
        weapons[] = {};      // Populates weapon lists
        requiredVersion = 0.1;
        requiredAddons[] = { "DZ_Data", "DZ_Scripts" }; // Dependencies (transitive)
    };
};
```

**Common requiredAddons entries:**
| Addon | When Needed |
|-------|-------------|
| `"DZ_Data"` | Always (base data) |
| `"DZ_Scripts"` | Always (base scripts) |
| `"DZ_Weapons_Firearms"` | Weapon mods |
| `"JM_CF_Scripts"` | CommunityFramework dependency |
| `"DF_Scripts"` | DabsFramework dependency |

### CfgMods Script Modules
```cpp
class CfgMods
{
    class MyMod
    {
        type = "mod";              // "mod" = client+server, "servermod" = server only
        dependencies[] = { "Game", "World", "Mission" };
        class defs
        {
            class gameScriptModule   { value = ""; files[] = { "MyMod/Scripts/3_Game" }; };
            class worldScriptModule  { value = ""; files[] = { "MyMod/Scripts/4_World" }; };
            class missionScriptModule { value = ""; files[] = { "MyMod/Scripts/5_Mission" }; };
            // value = "" means default entry point. Can specify custom function name.
            // files[] entries are directories — engine recursively compiles all .c files within
            // Multiple directories allowed (COT "Common folder" pattern)
            defines[] = { "MYMOD_ENABLED" }; // Custom preprocessor symbols
        };
    };
};
```

### CfgVehicles for Custom Items
```cpp
class CfgVehicles
{
    class Inventory_Base;  // Forward declaration
    class MyItem : Inventory_Base
    {
        scope = 2;         // 0=hidden, 1=internal, 2=spawnable (required for CE)
        displayName = "$STR_MYMOD_ITEM_NAME";
        descriptionShort = "$STR_MYMOD_ITEM_DESC";
        model = "MyMod\data\my_item.p3d";
        weight = 500;      // Grams
        itemSize[] = {1, 2}; // {columns, rows} NOT {width, height}
        hiddenSelections[] = { "zbytek" };
        hiddenSelectionsTextures[] = { "MyMod\data\textures\item_co.paa" };
    };
};
```

### CfgSoundSets & CfgSoundShaders
```cpp
class CfgSoundShaders
{
    class MyMod_SFXShader
    {
        samples[] = { {"MyMod\sounds\alert", 1} }; // Path WITHOUT .ogg extension!
        volume = 0.8;
        range = 50;      // Meters to silence
        radius = 5;      // Meters at full volume
    };
};
class CfgSoundSets
{
    class MyMod_SFXSoundSet
    {
        soundShaders[] = { "MyMod_SFXShader" };
        spatial = 1;     // 0=2D UI, 1=3D world (must be mono .ogg)
    };
};
```

---

## Server / Client Architecture

### Three Execution Contexts

| Context | `IsServer()` | `IsClient()` | `IsDedicatedServer()` | `GetPlayer()` |
|---------|-------------|-------------|----------------------|----------------|
| Dedicated Server | `true` | `false` | `true` | `null` |
| Client | `false` | `true` | `false` | valid `Man` |
| Listen Server | `true` | `true` | `false` | valid `Man` |

### Preprocessor vs Runtime Guards
- `#ifdef SERVER` — Compile-time. SERVER is defined on dedicated AND listen server.
- `GetGame().IsServer()` — Runtime. Use for logic that runs differently per context.
- `#ifndef SERVER` — Only needed when code body references client-only types (widgets, UI).

### Responsibility Split
| Responsibility | Where |
|---------------|-------|
| Spawn/delete entities | Server |
| Apply damage | Server |
| Validate actions | Server |
| Check permissions | Server |
| Show UI | Client |
| Read player input | Client |
| Play sounds/effects | Client |
| Send authoritative state | Server → Client (RPC) |
| Send player requests | Client → Server (RPC) |

### Listen-Server Gotchas
- Both `IsServer()` and `IsClient()` return true
- Code that works on listen server may break on dedicated
- Always test on dedicated server before publishing

---

## mod.cpp Reference

mod.cpp is NOT Enforce Script. It's a simple key-value file read by the DayZ launcher.
```
name = "My Mod";
picture = "MyMod/mod_logo.edds";       // Only .edds, .paa, .tga work (PNG/JPG ignored)
logo = "MyMod/mod_logo.edds";
logoSmall = "MyMod/mod_logo_small.edds";
tooltip = "My Mod Description";
overview = "Detailed description shown in launcher";
author = "Author Name";
action = "https://github.com/mymod";   // Website URL
type = "mod";                           // Metadata only — -mod= vs -servermod= controls loading
```

---

## stringtable.csv Localization

CSV with 15 columns. Must be at mod root (next to mod.cpp), NOT inside Scripts/.

```csv
"Language","original","english","czech","german","russian","polish","hungarian","italian","spanish","french","chinese","japanese","portuguese","chinesesimp"
"STR_MYMOD_WELCOME","Welcome","Welcome","","Willkommen","","","","","","","","","Bem-vindo",""
```

### Referencing Strings
| Context | Syntax | Example |
|---------|--------|---------|
| Layout files | `#STR_KEY` | `text "#STR_MYMOD_WELCOME"` |
| Script `Widget` | `"#STR_KEY"` | `textWidget.SetText("#STR_MYMOD_WELCOME")` |
| inputs.xml `loc` | `STR_KEY` (NO #) | `loc="STR_MYMOD_ACTION"` |

Fallback chain: Player's language → english → original → raw key name.

---

## Singleton Improvements

### Re-Entrancy Protection
If `GetInstance()` is called during construction (e.g., registering with another system), the instance is still null:
```c
static MyManager GetInstance()
{
    if (!s_Instance)
    {
        s_Instance = new MyManager();
        s_Instance.Initialize();  // Safe: s_Instance already assigned
    }
    return s_Instance;
}
```

### Static-Only Class Alternative
When a class holds no instance state (all static methods/fields), skip GetInstance() entirely:
```c
class MyLog
{
    static void Info(string tag, string msg) { Print("[" + tag + "] " + msg); }
    static void Error(string tag, string msg) { Print("[" + tag + "] ERROR: " + msg); }
}
```
Use for logging, RPC dispatchers, event buses. No null-check overhead.

### Centralized Shutdown
```c
static void ShutdownAll()
{
    MyManager.DestroyInstance();
    MyConfig.DestroyInstance();
    MyRPC.Cleanup();
    // One method in OnMissionFinish — prevents forgetting individual calls
}
```

### Listen-Server Guard
Server-only singletons must guard construction on listen servers:
```c
static MyServerManager GetInstance()
{
    if (!s_Instance && GetGame().IsServer())
        s_Instance = new MyServerManager();
    return s_Instance;
}
```

---

## Config Versioning & Migration

```c
class MyModConfig
{
    int ConfigVersion = 1;
    // ... fields ...
    static const int CURRENT_VERSION = 3;
}

static void Load()
{
    s_Config = new MyModConfig();
    if (FileExist(CONFIG_PATH))
    {
        JsonFileLoader<MyModConfig>.JsonLoadFile(CONFIG_PATH, s_Config);

        // Sequential migration
        if (s_Config.ConfigVersion < 2)
        {
            // v1→v2: renamed field, added new field with default
            s_Config.ConfigVersion = 2;
        }
        if (s_Config.ConfigVersion < 3)
        {
            // v2→v3: changed array format
            s_Config.ConfigVersion = 3;
        }
        Save();  // Re-save with migrated data
    }
    else
    {
        s_Config.ConfigVersion = MyModConfig.CURRENT_VERSION;
        Save();  // Create default
    }
}
```

---

## JsonFileLoader Serialization Rules

- Serializes ALL `public` fields only
- `private`, `protected`, `static` fields are excluded
- Supported types: `int`, `float`, `bool`, `string`, `ref array<T>`, `ref map<K,V>`, nested objects
- Use access modifiers to control what goes to disk
- `MakeDirectory()` is NOT recursive — create each level explicitly

---

## Auto-Save Pattern

```c
class MyPersistenceManager
{
    private const float AUTOSAVE_INTERVAL = 300.0; // 5 minutes
    private float m_SaveTimer;
    private bool m_Dirty;

    void MarkDirty() { m_Dirty = true; }

    void OnUpdate(float dt)
    {
        if (!m_Dirty) return;
        m_SaveTimer += dt;
        if (m_SaveTimer < AUTOSAVE_INTERVAL) return;

        m_SaveTimer = 0;
        m_Dirty = false;
        Save();
    }

    // Also save on critical operations and OnMissionFinish
}
```

---

## Event Bus Pattern

Centralized named event channels instead of per-object ScriptInvokers:

```c
class MyEventBus
{
    static ref ScriptInvoker OnPlayerConnected = new ScriptInvoker();
    static ref ScriptInvoker OnPlayerDisconnected = new ScriptInvoker();
    static ref ScriptInvoker OnConfigChanged = new ScriptInvoker();

    static void Cleanup()
    {
        OnPlayerConnected = null;
        OnPlayerDisconnected = null;
        OnConfigChanged = null;
    }
}

// Subscribe
MyEventBus.OnPlayerConnected.Insert(OnPlayerJoin);

// Fire
MyEventBus.OnPlayerConnected.Invoke(player);

// ALWAYS Remove in cleanup — stale subscribers crash or waste cycles
MyEventBus.OnPlayerConnected.Remove(OnPlayerJoin);
```

### ScriptInvoker Safety Rules
- Every `Insert()` MUST have a matching `Remove()`
- Double `Insert()` = handler called twice per `Invoke()` (one `Remove()` only removes one)
- No compile-time type safety on parameters — document expected signature
- On `Managed` classes: stale refs become null (wasted iteration)
- On non-Managed classes: stale refs become dangling pointers (crash!)
- Null-check the invoker before `Remove()` in destructors (Cleanup may have already run)

---

## Permission Patterns

### VPP Group-Based Permissions
```c
// Groups: "SuperAdmin" → ["*"], "Moderator" → ["admin.kick", "admin.ban"]
// Players: "76561198..." → "SuperAdmin"
// Changes to a group propagate to all members
```

### CF Three-State Permissions (ALLOW/DENY/INHERIT)
Each node in the permission tree can be:
- `ALLOW (2)` — Granted
- `DENY (1)` — Explicitly denied (overrides parent ALLOW)
- `INHERIT (0)` — Inherits from parent

Enables: "Grant `Admin.*` but deny `Admin.Teleport`".

### Permission Security Rules
1. **Never trust the client** — Server validates all permissions
2. **Default deny** — If not explicitly granted, denied
3. **Fail closed** — If check fails (null identity, corrupted data), deny
4. **Define as constants** — `static const string PERM_KICK = "MyMod.Admin.Kick";` (prevents typo bugs)
5. **Sync summary only** — Send only the player's own permissions to client for UI gating, never the full file

---

## Module Lifecycle Rules

1. `OnInit` comes before `OnMissionStart` — never load configs or spawn entities in OnInit
2. Register RPCs in `OnInit` (clients may connect before OnMissionStart completes)
3. `OnUpdate` always receives delta time — never assume fixed frame rate
4. `OnMissionFinish` must clean up everything — every ref collection cleared, every subscription removed
5. Modules should not depend on each other's initialization order — use lazy access
