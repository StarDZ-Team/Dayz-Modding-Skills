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
