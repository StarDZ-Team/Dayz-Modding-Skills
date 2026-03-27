# Advanced Patterns & Troubleshooting

Performance optimization, real-world UI patterns, troubleshooting diagnostics, and advanced engine API patterns used by professional DayZ mods (COT, VPP, Expansion, Dabs).

---

## Performance Optimization

DayZ server frame budget at 60 FPS is ~16ms. At 20 FPS (common on loaded servers), it's ~50ms. A single mod should stay under 2ms per frame. These patterns are NOT premature optimization — they are standard practice in every major mod.

### Lazy Loading

Never pre-load data the user might not need:

```c
class ItemDatabase
{
    protected ref map<string, ref ItemData> m_Cache;
    protected bool m_Loaded;

    ItemData GetItem(string className)
    {
        if (!m_Loaded)
        {
            LoadAllItems();
            m_Loaded = true;
        }
        ItemData data;
        m_Cache.Find(className, data);
        return data;
    }
}
```

### Batched Processing (N Per Frame)

Process a fixed batch per frame instead of the entire collection at once:

```c
class LootCleanup
{
    protected ref array<Object> m_DirtyItems;
    protected int m_ProcessIndex;
    static const int BATCH_SIZE = 50;

    void OnUpdate(float dt)
    {
        if (!m_DirtyItems || m_DirtyItems.Count() == 0) return;

        int processed = 0;
        while (m_ProcessIndex < m_DirtyItems.Count() && processed < BATCH_SIZE)
        {
            Object item = m_DirtyItems[m_ProcessIndex];
            if (item) ProcessItem(item);
            m_ProcessIndex++;
            processed++;
        }

        if (m_ProcessIndex >= m_DirtyItems.Count())
        {
            m_DirtyItems.Clear();
            m_ProcessIndex = 0;
        }
    }
}
```

**Batch size guide:** Lightweight ops (null checks, positions) → 100-200/frame. Heavy ops (spawning, pathfinding, file I/O) → 5-10/frame. Default: 50.

### Widget Pooling

Creating/destroying UI widgets is expensive. Pre-create a pool, reuse by showing/hiding:

```c
class WidgetPool
{
    protected ref array<Widget> m_Pool;
    protected Widget m_Parent;
    protected string m_LayoutPath;
    protected int m_ActiveCount;

    void WidgetPool(Widget parent, string layoutPath, int initialSize)
    {
        m_Parent = parent;
        m_LayoutPath = layoutPath;
        m_Pool = new array<Widget>();
        m_ActiveCount = 0;

        for (int i = 0; i < initialSize; i++)
        {
            Widget w = GetGame().GetWorkspace().CreateWidgets(m_LayoutPath, m_Parent);
            w.Show(false);
            m_Pool.Insert(w);
        }
    }

    Widget Acquire()
    {
        if (m_ActiveCount < m_Pool.Count())
        {
            Widget w = m_Pool[m_ActiveCount];
            w.Show(true);
            m_ActiveCount++;
            return w;
        }
        // Pool exhausted — grow it
        Widget newWidget = GetGame().GetWorkspace().CreateWidgets(m_LayoutPath, m_Parent);
        m_Pool.Insert(newWidget);
        m_ActiveCount++;
        return newWidget;
    }

    void ReleaseAll()
    {
        for (int i = 0; i < m_ActiveCount; i++)
            m_Pool[i].Show(false);
        m_ActiveCount = 0;
    }

    void Destroy()
    {
        for (int i = 0; i < m_Pool.Count(); i++)
            if (m_Pool[i]) m_Pool[i].Unlink();
        m_Pool.Clear();
        m_ActiveCount = 0;
    }
}

// Usage: first call creates widgets, subsequent calls reuse them
void RefreshPlayerList(array<string> players)
{
    m_WidgetPool.ReleaseAll();
    for (int i = 0; i < players.Count(); i++)
    {
        Widget row = m_WidgetPool.Acquire();
        TextWidget.Cast(row.FindAnyWidget("NameText")).SetText(players[i]);
    }
}
```

### Search Debouncing

Delay search until user stops typing (150ms default):

```c
class SearchableList
{
    protected const float DEBOUNCE_DELAY = 0.15;
    protected float m_SearchTimer;
    protected bool m_SearchPending;
    protected string m_PendingQuery;

    void OnSearchTextChanged(string text)
    {
        m_PendingQuery = text;
        m_SearchPending = true;
        m_SearchTimer = 0;
    }

    void Tick(float dt)
    {
        if (!m_SearchPending) return;
        m_SearchTimer += dt;
        if (m_SearchTimer >= DEBOUNCE_DELAY)
        {
            m_SearchPending = false;
            ExecuteSearch(m_PendingQuery);
        }
    }
}
```

### Update Rate Limiting

Not everything needs to run every frame:

```c
class EntityScanner
{
    protected const float SCAN_INTERVAL = 5.0;
    protected float m_ScanTimer;

    void OnUpdate(float dt)
    {
        m_ScanTimer += dt;
        if (m_ScanTimer < SCAN_INTERVAL) return;
        m_ScanTimer = 0;
        ScanEntities(); // Runs every 5 seconds, not every frame
    }
}
```

**Stagger multiple timers** so they don't all fire on the same frame:
```c
m_LootTimer    = 0;     // Fires first
m_VehicleTimer = 1.6;   // Fires 1.6s later
m_WeatherTimer = 3.3;   // Fires 3.3s later
```

### CfgVehicles Scan Cache

Scanning CfgVehicles (thousands of entries) is expensive. Cache the results:

```c
class WeaponRegistry
{
    private static ref array<string> s_AllWeapons;

    static array<string> GetAllWeapons()
    {
        if (s_AllWeapons) return s_AllWeapons;
        s_AllWeapons = new array<string>();

        int cfgCount = GetGame().ConfigGetChildrenCount("CfgVehicles");
        string className;
        for (int i = 0; i < cfgCount; i++)
        {
            GetGame().ConfigGetChildName("CfgVehicles", i, className);
            if (GetGame().IsKindOf(className, "Weapon_Base"))
                s_AllWeapons.Insert(className);
        }
        return s_AllWeapons;
    }

    static void Cleanup() { s_AllWeapons = null; }
}
```

### Vehicle Registry (Registration-Based)

NEVER use `GetObjectsAtPosition3D` with huge radius. Track entities as they are created:

```c
class VehicleRegistry
{
    private static ref array<CarScript> s_Vehicles = new array<CarScript>();

    static void Register(CarScript v)
    {
        if (v && s_Vehicles.Find(v) == -1) s_Vehicles.Insert(v);
    }

    static void Unregister(CarScript v)
    {
        int idx = s_Vehicles.Find(v);
        if (idx >= 0) s_Vehicles.Remove(idx);
    }

    static array<CarScript> GetAll() { return s_Vehicles; }
    static void Cleanup() { s_Vehicles.Clear(); }
}

modded class CarScript
{
    override void EEInit()
    {
        super.EEInit();
        if (GetGame().IsServer()) VehicleRegistry.Register(this);
    }

    override void EEDelete(EntityAI parent)
    {
        if (GetGame().IsServer()) VehicleRegistry.Unregister(this);
        super.EEDelete(parent);
    }
}
```

### Things to NEVER Do

| Anti-Pattern | Impact | Fix |
|-------------|--------|-----|
| `GetObjectsAtPosition3D` with 50km radius in OnUpdate | 50-200ms stall per call | Registration-based registry |
| Destroy/recreate all widgets on list refresh | Frame spikes, visible stutter | Widget pooling |
| Sort large arrays every frame | O(n log n) wasted per frame | Sort on data change only (dirty flag) |
| String concatenation in tight per-frame loops | Garbage allocation every frame | Do on state change, not per frame |
| Entity spawning in tight loop (100 spawns in one frame) | Massive frame spike | Batch across frames (5-10 per frame) |
| `JsonSaveFile()` in OnUpdate | Disk I/O blocks main thread | Auto-save timer (300s) with dirty flag |
| `FileExist()` in a loop for the same path | Redundant disk checks | Check once, cache result |
| `GetGame()` in a tight loop | Repeated function call overhead | Cache: `CGame game = GetGame();` |

### Performance Checklist

- [ ] No `GetObjectsAtPosition3D` with radius > 100m in per-frame code
- [ ] All expensive scans (CfgVehicles, entity searches) are cached
- [ ] UI lists use widget pooling, not destroy/recreate
- [ ] Search inputs use debouncing (150ms+)
- [ ] OnUpdate operations throttled by timer or batch size
- [ ] Large collections processed in batches (50 items/frame default)
- [ ] Entity spawning batched across frames
- [ ] String concatenation not done per-frame in tight loops
- [ ] Multiple periodic systems have staggered timers
- [ ] Entity tracking uses registration, not world scanning

---

## Profiling

### Measuring Frame Time

```c
void OnUpdate(float dt)
{
    float startTime = GetGame().GetTickTime();
    // ... your logic ...
    float elapsed = GetGame().GetTickTime() - startTime;
    if (elapsed > 0.005)  // More than 5ms
    {
        Print("[MyMod] PERF WARNING: OnUpdate took " + elapsed.ToString() + "s");
    }
}
```

### Log Indicators
- `SCRIPT (W): Exceeded X ms` — Script exceeded engine time budget
- Long pauses in log timestamps — Something blocked the main thread
- Target: single mod should stay under 2ms/frame

---

## Troubleshooting Quick Reference

### Diagnostic Flowchart: "My Mod Doesn't Work"

1. **Check script log** for `SCRIPT (E)` errors → Fix the FIRST error (errors cascade)
2. **Is mod in launcher?** If not → `mod.cpp` missing or has syntax errors
3. **Does log mention your CfgPatches?** If not → Check `config.cpp` syntax, `requiredAddons[]`, `-mod=` parameter
4. **Do scripts compile?** Look for compile errors in RPT → Fix syntax errors
5. **Is there an entry point?** Need `modded class MissionServer/MissionGameplay`, registered module, or plugin
6. **Still nothing?** Add `Print("MY_MOD: Init reached");` to confirm execution

### Diagnostic Flowchart: "Works Offline, Fails on Dedicated Server"

1. **Is mod installed on server?** Check `-mod=` includes your mod, PBO is in `@Mod/Addons/`
2. **Client-only code on server?** `GetGame().GetPlayer()` returns null during server init → Add `IsServer()`/`IsClient()` guards
3. **RPCs working?** Add `Print()` on both send/receive → Check IDs match, target entity exists on both sides
4. **Data syncing?** Verify `SetSynchDirty()` called after changes → Check read/write param order matches
5. **Timing issues?** Listen servers hide race conditions → Add null checks and readiness guards

### Diagnostic Flowchart: "My UI Is Broken"

1. **`CreateWidgets()` returns null?** → Layout path wrong or file missing (check forward slashes)
2. **Widgets exist but invisible?** → Check sizes (>0, no negatives), `Show(true)`, alpha not 0
3. **Visible but not clickable?** → Check widget `priority` (z-order), verify `ScriptClass`, confirm handler set
4. **Input stuck after closing?** → `ChangeGameFocus()` calls imbalanced (every +1 needs a -1)

### Common Error Messages & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Null pointer access` | Accessing null variable | Add null check before use |
| `Cannot convert type 'X' to 'Y'` | Direct cast | Use `Class.CastTo()` |
| `Undefined variable 'X'` | Typo, wrong scope, or wrong layer | Check spelling, check layer hierarchy |
| `Method 'X' not found` | Wrong class or needs cast | Cast to specific type first |
| `Redeclaration of variable 'X'` | Same var name in sibling if/else | Declare before if/else chain |
| `Stack overflow` | Infinite recursion | Add base case or depth check |
| `Index out of range` | Bad array index | Check `Count()` or `IsValidIndex()` |
| `Error: Serializer X mismatch` | Save/load order mismatch | Ensure same types in same order + version check |
| Config parse error at startup | Missing `;` or `};` in config.cpp | Check every class body ends with `};` |
| `Cannot register cfg class X` | Duplicate CfgPatches name | Rename with unique mod prefix |
| `Addon requires addon X` | Missing `requiredAddons[]` | Add exact CfgPatches class name (case-sensitive) |
| No script log entries at all | CfgMods path is wrong | Engine silently ignores wrong paths — verify folder structure |

### Log File Locations

| Log | Location |
|-----|----------|
| Client script log | `%localappdata%\DayZ\` (most recent `.RPT` file) |
| Server script log | `<server_root>\profiles\` (most recent `.RPT` file) |
| Server admin log | `<server_root>\profiles\` (`.ADM` file) |
| Crash dumps | Same directories (`.mdmp` files) |

**Reading logs:** Search for `SCRIPT (E)` first. Fix the FIRST error (they cascade). Search for your mod name to filter relevant entries.

---

## Debug Commands Quick Reference

Use in DayZDiag debug console:

```c
// Spawn
GetGame().CreateObject("AKM", GetGame().GetPlayer().GetPosition());

// Spawn assembled vehicle
EntityAI car = EntityAI.Cast(GetGame().CreateObject("OffroadHatchback", GetGame().GetPlayer().GetPosition()));
if (car) car.OnDebugSpawn();

// Teleport
GetGame().GetPlayer().SetPosition("6543 0 2114".ToVector());

// Heal
GetGame().GetPlayer().SetHealth("", "", 5000);
GetGame().GetPlayer().SetHealth("", "Blood", 5000);

// Time
GetGame().GetWorld().SetDate(2024, 9, 15, 12, 0);  // Noon
GetGame().GetWorld().SetDate(2024, 9, 15, 2, 0);    // Night

// Weather
GetGame().GetWeather().GetOvercast().Set(0,0,0);     // Clear
GetGame().GetWeather().GetRain().Set(1,0,0);          // Rain

// Info
Print(GetGame().GetPlayer().GetPosition());
Print("IsServer: " + GetGame().IsServer().ToString());
```

**Chernarus locations:** Elektro `"10570 0 2354"`, Cherno `"6649 0 2594"`, NWAF `"4494 0 10365"`, Tisy `"1693 0 13575"`, Berezino `"12121 0 9216"`

### Launch Parameters

| Parameter | Purpose |
|-----------|---------|
| `-filePatching` | Load unpacked files (requires DayZDiag) |
| `-scriptDebug=true` | Enable script debug features |
| `-doLogs` | Enable detailed logging |
| `-adminLog` | Enable admin log |
| `-freezeCheck` | Detect and log script freezes |
| `-noPause` | Server doesn't pause when empty |
| `-mod=@Mod1;@Mod2` | Load mods (semicolon-separated) |
| `-serverMod=@Mod` | Server-only mods (not sent to clients) |

---

## Input System

### Custom Keybindings

```c
// Register custom input action
GetUApi().RegisterInput("UAMyModAction", "My Action", "keyboard", "kU");

// Check input state in OnUpdate
if (GetUApi().GetInputByName("UAMyModAction").LocalPress())
{
    // Key was just pressed
    OnMyAction();
}

if (GetUApi().GetInputByName("UAMyModAction").LocalHold())
{
    // Key is being held
}
```

### inputs.xml

Custom inputs must also be declared in `inputs.xml` in your mod root:

```xml
<?xml version="1.0" encoding="utf-8"?>
<modactions>
    <actions>
        <input name="UAMyModAction" loc="My Action" type="pressed" default="keyboard:kU" />
    </actions>
</modactions>
```

---

## Advanced Entity API

### Bone Positions & Transforms

```c
// Get bone position in world space
vector bonePos = entity.GetBonePositionWS(entity.GetBoneIndexByName("Head"));

// Get full transform matrix
vector transform[4];
entity.GetTransform(transform);
// transform[0] = right, transform[1] = up, transform[2] = forward, transform[3] = position

// Set transform
entity.SetTransform(transform);
```

### Entity Hierarchy

```c
// Parent-child relationships
entity.AddChild(childEntity, -1);     // Attach child
entity.RemoveChild(childEntity);       // Detach child
Object parent = entity.GetParent();    // Get parent
Object sibling = entity.GetSibling();  // Get next sibling in parent's children
```

### Health & Damage Zones

```c
// Get health by zone and type
float health = entity.GetHealth("Zone_Head", "Health");
float armor = entity.GetHealth("Zone_Chest", "Armor");

// Set health
entity.SetHealth("", "Health", 100);  // Global health
entity.SetHealth("Zone_Head", "Health", 50);  // Zone-specific

// Apply damage
entity.ProcessDirectDamage(
    DT_CLOSE_COMBAT,   // damage type
    source,              // who dealt it
    "Zone_Chest",        // damage zone
    "MeleeFist",         // ammo type (determines damage values)
    "0 0 0",             // model position
    1.0                  // damage coef
);
```

### ECE Creation Flags

```c
// Common flags for CreateObjectEx
ECE_PLACE_ON_SURFACE     // Place on terrain surface
ECE_UPDATEPATHGRAPH      // Update AI navmesh
ECE_CREATEPHYSICS        // Create physics body
ECE_ROTATIONFLAGS        // Apply rotation flags
ECE_EQUIP                // Equip the item
ECE_IN_INVENTORY         // Create inside inventory
ECE_LOCAL                // Local-only (not networked)

// Example: spawn item on ground with physics
GetGame().CreateObjectEx("Apple", pos, ECE_PLACE_ON_SURFACE | ECE_UPDATEPATHGRAPH, RF_DEFAULT);
```

---

## Advanced RPC Patterns

### Full Client-Server Roundtrip

```c
// CLIENT: Send request to server
void RequestAction(string itemId)
{
    ScriptRPC rpc = new ScriptRPC();
    rpc.Write(itemId);
    rpc.Send(null, RPC_REQUEST_ACTION, true, null);
}

// SERVER: Receive, validate, respond
override void OnRPC(PlayerIdentity sender, int rpc_type, ParamsReadContext ctx)
{
    super.OnRPC(sender, rpc_type, ctx);

    if (rpc_type == RPC_REQUEST_ACTION)
    {
        // 1. Validate sender
        if (!sender) return;

        // 2. Read data
        string itemId;
        if (!ctx.Read(itemId)) return;

        // 3. Validate permissions
        if (!HasPermission(sender.GetPlainId(), "mymod.action"))
        {
            // Send error response
            ScriptRPC errRpc = new ScriptRPC();
            errRpc.Write("Permission denied");
            errRpc.Send(null, RPC_ACTION_ERROR, true, sender);
            return;
        }

        // 4. Process
        bool success = DoAction(itemId);

        // 5. Send response to requesting client
        ScriptRPC respRpc = new ScriptRPC();
        respRpc.Write(success);
        respRpc.Write(itemId);
        respRpc.Send(null, RPC_ACTION_RESPONSE, true, sender);
    }
}

// CLIENT: Handle response
// (in OnRPC on client side)
if (rpc_type == RPC_ACTION_RESPONSE)
{
    bool success;
    string itemId;
    if (!ctx.Read(success)) return;
    if (!ctx.Read(itemId)) return;

    if (success)
        ShowNotification("Action completed: " + itemId);
    else
        ShowNotification("Action failed: " + itemId);
}
```

### Three RPC Approaches Compared

| Approach | Pros | Cons | Use When |
|----------|------|------|----------|
| **Engine ScriptRPC** | Direct, no dependencies, full control | Manual read/write, numeric IDs, verbose | Standalone mods, vanilla modding |
| **CF RPCManager** | String-based routing, auto-serialization | Requires CF dependency | Mod already uses CF |
| **SDZ_RPC (StarDZ)** | String-routed, single engine ID 83722, clean API | StarDZ Core dependency | StarDZ ecosystem mods |

### RPC Throttling

Never send RPCs every frame. Use an accumulator:

```c
class RPCThrottle
{
    protected float m_SendTimer;
    protected const float SEND_INTERVAL = 0.5;  // Max 2 RPCs/second
    protected bool m_Dirty;

    void MarkDirty() { m_Dirty = true; }

    void OnUpdate(float dt)
    {
        if (!m_Dirty) return;
        m_SendTimer += dt;
        if (m_SendTimer < SEND_INTERVAL) return;

        m_SendTimer = 0;
        m_Dirty = false;
        SendUpdate();
    }
}
```

---

## RPC: Additional Details

### Serialization Contract
Read/Write order must EXACTLY match, type for type. Supported types:

| Type | Size | Notes |
|------|------|-------|
| `int` | 4 bytes | 32-bit signed |
| `float` | 4 bytes | IEEE 754 |
| `bool` | 1 byte | |
| `string` | Variable | Length-prefixed UTF-8 |
| `vector` | 12 bytes | Three floats |

Arrays can be written/read directly. For complex objects, flatten to primitives.

### guaranteed Parameter
- `true` (reliable): Config changes, permissions, teleports — dropped packet = desync
- `false` (unreliable): Rapid position updates, visual effects — lower overhead, no retransmit

### RPC Payload Size Limit
Practical limit ~64 KB per RPC. For large data (config sync, player lists), paginate across multiple RPCs.

### CF Named RPC Pattern
```c
// Register
GetRPCManager().AddRPC("MyMod", "RPC_SpawnItem", this, SingleplayerExecutionType.Server);

// Send
GetRPCManager().SendRPC("MyMod", "RPC_SpawnItem", new Param1<string>("AK74"), true);

// Handler receives: (CallType type, ParamsReadContext ctx, PlayerIdentity sender, Object target)
void RPC_SpawnItem(CallType type, ParamsReadContext ctx, PlayerIdentity sender, Object target)
{
    if (type != CallType.Server) return;
    // ...
}
```

### Stale RPC Handler Cleanup
RPC handlers registered by modules destroyed on mission end will crash on next dispatch. Unregister individually or clear the entire registry in OnMissionFinish.

---

## Performance: Additional Patterns

### Expansion's Linked-List Entity Tracking
O(1) insert/remove without array allocation — each entity stores `m_Next` and `m_Prev`:
```c
class TrackedEntity extends EntityAI
{
    static TrackedEntity s_Head;
    TrackedEntity m_Next;
    TrackedEntity m_Prev;

    void Register()
    {
        m_Next = s_Head;
        if (s_Head) s_Head.m_Prev = this;
        s_Head = this;
    }

    void Unregister()
    {
        if (m_Prev) m_Prev.m_Next = m_Next;
        else s_Head = m_Next;
        if (m_Next) m_Next.m_Prev = m_Prev;
        m_Next = null;
        m_Prev = null;
    }
}
```

### String Operation Cache
Pre-compute expensive string transforms once, not on every search:
```c
class SearchableItem
{
    string DisplayName;
    string SearchName;  // Lowercased for search

    void Init(string name)
    {
        DisplayName = name;
        SearchName = name;
        SearchName.ToLower();  // Once in constructor
    }
}
```

### Position Cache
Cache `GetPosition()` and refresh periodically instead of every proximity check:
```c
protected vector m_CachedPos;
protected float m_PosTimer;

void OnUpdate(float dt)
{
    m_PosTimer += dt;
    if (m_PosTimer >= 1.0)
    {
        m_PosTimer = 0;
        m_CachedPos = GetPosition();
    }
}
```

### Frame-Count Throttling
Alternative to timers for per-N-frame operations:
```c
m_FrameCounter++;
if (m_FrameCounter % 10 != 0) return;  // Every 10th frame
```

### Custom Sort
Built-in `.Sort()` only works for basic types. For custom sort on complex objects, use insertion sort for small arrays (<100 elements). Never sort per frame — only on data change.

---

## File Patching Workflow

`-filePatching` with `DayZDiag_x64.exe` loads loose files from the P: drive instead of PBOs — fastest iteration.

### What Works
| File Type | Hot-Reload? |
|-----------|------------|
| `.c` scripts | Yes |
| `.layout` files | Yes |
| `.paa` textures | Yes |
| `.ogg` audio | Yes |

### What Does NOT Work
| File Type | Requires |
|-----------|----------|
| `config.cpp` changes | PBO rebuild |
| New `.c` files (not existing) | PBO rebuild |
| New model/texture references | PBO rebuild |
| `model.cfg` changes | PBO rebuild + binarize |

---

## Diagnostic Flowcharts

### "My Mod Doesn't Work At All"
1. Check script log for `SCRIPT (E)` errors → Fix FIRST error (they cascade)
2. Is mod in launcher/`-mod=` parameter? → Check `mod.cpp` exists and has no syntax errors
3. Does log mention your CfgPatches? → Check `config.cpp` syntax, `requiredAddons[]`
4. Do scripts compile? → Look for compile errors in `.RPT` file
5. Is there an entry point? → Need `modded class MissionServer/MissionGameplay` or registered module
6. Still nothing? → Add `Print("MY_MOD: Init reached");` to confirm execution

### "Works Offline, Fails on Dedicated Server"
1. Is mod installed on server? → Check `-mod=`, PBO in `@Mod/Addons/`
2. Client-only code on server? → `GetGame().GetPlayer()` returns null on server. Add `IsServer()`/`IsClient()` guards
3. RPCs working? → Add `Print()` on send/receive. Check IDs match, target exists both sides
4. Data syncing? → Verify `SetSynchDirty()` after changes. Check read/write order matches
5. Identity null? → `player.GetIdentity()` is null in offline mode. Test on dedicated.

### "My UI Is Broken"
1. `CreateWidgets()` returns null? → Layout path wrong or file missing (use forward slashes, no error logged)
2. Widgets exist but invisible? → Check sizes (>0), `Show(true)`, alpha not 0, widget not behind parent
3. Visible but not clickable? → Check `priority` (z-order), verify `scriptclass`, confirm handler set
4. Input stuck after closing? → `ChangeGameFocus()` calls imbalanced (every +1 needs -1)

---

## Launch Parameters Reference

| Parameter | Purpose |
|-----------|---------|
| `-filePatching` | Load unpacked files (requires DayZDiag) |
| `-scriptDebug=true` | Enable script debug features |
| `-doLogs` | Enable detailed logging |
| `-adminLog` | Enable admin log |
| `-freezeCheck` | Detect and log script freezes |
| `-noPause` | Server doesn't pause when empty |
| `-noSound` | Disable sound (faster loading) |
| `-profiles=<path>` | Set profile directory for logs |
| `-mod=@Mod1;@Mod2` | Load mods (semicolon-separated) |
| `-serverMod=@Mod` | Server-only mods (not sent to clients) |

---

## Preprocessor Debug Guards

```c
#ifdef DEVELOPER
    // Active in DayZDiag, stripped in retail build. Zero cost in release.
    Print("[DEBUG] " + data);
#endif

#ifdef DIAG_DEVELOPER
    // Also DayZDiag only, used for diagnostic features
#endif
```

---

## Pre-Release Checklist

- [ ] Remove or guard all `Print()` debug statements with `#ifdef DEVELOPER`
- [ ] Test with a clean profile (no saved configs)
- [ ] Test mod loading order (after and before known dependency mods)
- [ ] Verify no memory leaks (check singleton cleanup, ScriptInvoker Remove)
- [ ] Verify stringtable.csv has all referenced keys
- [ ] Test on DEDICATED server (not just listen server)
- [ ] Check `.RPT` for warnings (not just script log)
- [ ] PBO is signed with matching `.bikey`
