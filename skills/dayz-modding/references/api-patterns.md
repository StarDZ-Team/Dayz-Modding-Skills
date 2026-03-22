# DayZ Engine API Patterns

Common patterns and recipes for the DayZ engine API. Each pattern includes the correct usage and common mistakes to avoid.

---

## Entity System

### Object Hierarchy

```
Object
  └── EntityAI
        ├── ItemBase (Inventory_Base)    — Items, weapons, clothing
        ├── PlayerBase (SurvivorBase)    — Player characters
        ├── BuildingBase                  — Structures
        ├── CarScript (Transport)         — Vehicles
        ├── AnimalBase                    — Animals
        └── ZombieBase (DayZInfected)     — Infected
```

### Creating Objects

```c
// Basic creation (server-side)
EntityAI item = EntityAI.Cast(
    GetGame().CreateObject("Apple", playerPos, false, false, true)
);

// With ECE flags for full Central Economy integration
Object obj = GetGame().CreateObjectEx(
    "Apple",
    playerPos,
    ECE_PLACE_ON_SURFACE | ECE_UPDATEPATHGRAPH,
    RF_DEFAULT
);

// IMPORTANT: CreateObject last param (bool iLocal):
// true  = local only (client-side visual, not networked)
// false = networked (server creates, replicates to clients)
```

### Finding Objects

```c
// Find nearest object of type
EntityAI nearest;
float minDist = float.MAX;
array<Object> objects = new array<Object>();
GetGame().GetObjectsAtPosition(pos, radius, objects, null);

foreach (Object obj : objects)
{
    EntityAI ent;
    if (!Class.CastTo(ent, obj)) continue;
    if (!ent.IsInherited(ItemBase)) continue;

    float dist = vector.Distance(pos, ent.GetPosition());
    if (dist < minDist)
    {
        minDist = dist;
        nearest = ent;
    }
}

// Raycast
vector from = player.GetPosition() + "0 1.5 0";
vector to = from + player.GetDirection() * 10;
vector contactPos, contactDir;
int contactComponent;
Object hitObject;

if (DayZPhysics.RayCastBullet(from, to, PhxInteractionLayers.BUILDING, null, hitObject, contactPos, contactDir, contactComponent))
{
    // hitObject is what was hit
}
```

### Deleting Objects

```c
// Server-side deletion (networked)
GetGame().ObjectDelete(entity);

// NEVER delete players directly — use disconnect/kick APIs
```

### Entity Checks

```c
// Existence and state
if (entity && !entity.IsDeleted())
{
    // entity exists
}

// Alive check (EntityAI only, NOT Object)
EntityAI ent;
if (Class.CastTo(ent, obj) && ent.IsAlive())
{
    // alive
}

// Type checking
if (entity.IsKindOf("Apple"))           // String-based
if (entity.IsInherited(ItemBase))       // Type-based (preferred)
```

---

## RPC & Networking

### Engine RPC (Vanilla Pattern)

```c
// Sending RPC (ScriptRPC)
ScriptRPC rpc = new ScriptRPC();
rpc.Write(someString);
rpc.Write(someInt);
rpc.Write(someFloat);
rpc.Send(targetObject, RPC_MY_ACTION, true, null);
// true = guaranteed delivery, null = all clients (or specific identity)

// Receiving RPC
override void OnRPC(PlayerIdentity sender, int rpc_type, ParamsReadContext ctx)
{
    super.OnRPC(sender, rpc_type, ctx);

    if (rpc_type == RPC_MY_ACTION)
    {
        string str;
        int num;
        float flt;

        if (!ctx.Read(str)) return;
        if (!ctx.Read(num)) return;
        if (!ctx.Read(flt)) return;

        // Process data
    }
}
```

### Param-Based RPC

```c
// Using Param types for structured data
Param1<string> p1 = new Param1<string>("hello");
Param2<string, int> p2 = new Param2<string, int>("hello", 42);
Param3<string, int, float> p3 = new Param3<string, int, float>("hello", 42, 3.14);

// Send
GetGame().RPCSingleParam(target, RPC_ID, p1, true, identity);

// Receive
Param1<string> data = new Param1<string>("");
if (!ctx.Read(data)) return;
string value = data.param1;
```

### SDZ_RPC (StarDZ String-Routed)

All StarDZ mods use engine ID 83722 with string routing:

```c
// Register handler
SDZ_RPC.Register("MyMod:Action", this, "OnRPC_Action");

// Send to server
SDZ_RPC.Send("MyMod:Action", new Param1<string>(data), true, null);

// Send to specific client
SDZ_RPC.Send("MyMod:Response", params, true, targetIdentity);

// Handler
void OnRPC_Action(CallType type, ParamsReadContext ctx, PlayerIdentity sender, Object target)
{
    // ALWAYS validate context
    if (type != CallType.Server) return;  // Only process on server
    if (!sender) return;                   // Must have sender

    Param1<string> data = new Param1<string>("");
    if (!ctx.Read(data)) return;           // Must read successfully

    // Process
}
```

### Context Checks

```c
// Server vs Client code
if (GetGame().IsServer())
{
    // Server-only logic (spawning, saving, authoritative state)
}

if (GetGame().IsClient())
{
    // Client-only logic (UI, effects, input)
}

if (GetGame().IsMultiplayer())
{
    // Multiplayer (not single player)
}

// In RPC handlers, use CallType
switch (type)
{
    case CallType.Server:    // Executing on server
    case CallType.Client:    // Executing on client
}
```

---

## File I/O

### JSON Files

```c
// CORRECT: JsonFileLoader returns void, pass ref object
class MyConfig
{
    string ServerName = "Default";
    int MaxPlayers = 60;
    ref array<string> Admins = new array<string>();
}

// Load
MyConfig cfg = new MyConfig();
string path = "$profile:MyMod/config.json";

if (FileExist(path))
{
    JsonFileLoader<MyConfig>.JsonLoadFile(path, cfg);
}

// Save
JsonFileLoader<MyConfig>.JsonSaveFile(path, cfg);
```

### Raw File I/O

```c
// Write
FileHandle file = OpenFile(path, FileMode.WRITE);
if (file)
{
    FPrintln(file, "Line of text");
    FPrintln(file, "Another line");
    CloseFile(file);
}

// Read
FileHandle file = OpenFile(path, FileMode.READ);
if (file)
{
    string line;
    while (FGets(file, line) >= 0)
    {
        // Process line
    }
    CloseFile(file);
}
```

### Path Prefixes

```c
"$profile:MyMod/data.json"     // Server profile folder
"$mission:MyMod/settings.json"  // Current mission folder
"$saves:MyMod/backup.json"      // Server saves
"$CurrentDir:data/defaults.json" // Mod's own directory
```

### Directory Operations

```c
// Check existence
if (FileExist(path)) { ... }

// Create directory
MakeDirectory("$profile:MyMod");
MakeDirectory("$profile:MyMod/Players");

// Find files
string fileName;
FileAttr fileAttr;
FindFileHandle handle = FindFile("$profile:MyMod/*.json", fileName, fileAttr, 0);
if (handle)
{
    do {
        if (!(fileAttr & FileAttr.DIRECTORY))
        {
            // fileName is a file
        }
    } while (FindNextFile(handle, fileName, fileAttr));
    CloseFindFile(handle);
}
```

---

## GUI & Widgets

### Loading a Layout

```c
// From layout XML file
Widget root = GetGame().GetWorkspace().CreateWidgets("MyMod/gui/layouts/panel.layout");

// Find child widgets
TextWidget title;
Class.CastTo(title, root.FindAnyWidget("TitleText"));

ButtonWidget btn;
Class.CastTo(btn, root.FindAnyWidget("CloseButton"));
```

### Widget Manipulation

```c
// Visibility
widget.Show(true);
widget.Show(false);
widget.IsVisible();

// Text
TextWidget.Cast(w).SetText("Hello World");

// Image
ImageWidget.Cast(w).LoadImageFile(0, "set:MyImageSet image:icon_health");

// Color (0xAARRGGBB format)
widget.SetColor(0xFFFF0000);  // Red, full opacity

// Position & Size (relative 0.0-1.0)
widget.SetPos(0.1, 0.2);      // 10% from left, 20% from top
widget.SetSize(0.5, 0.3);     // 50% width, 30% height

// Alpha
widget.SetAlpha(0.8);
```

### Event Handler

```c
class MyPanel extends ScriptedWidgetEventHandler
{
    private Widget m_Root;

    void MyPanel()
    {
        m_Root = GetGame().GetWorkspace().CreateWidgets("MyMod/gui/layouts/panel.layout");
        m_Root.SetHandler(this);
    }

    override bool OnClick(Widget w, int x, int y, int button)
    {
        if (w.GetName() == "CloseButton")
        {
            Close();
            return true;  // Consumed
        }
        return false;  // Not consumed
    }

    override bool OnChange(Widget w, int x, int y, bool finished)
    {
        if (w.GetName() == "VolumeSlider")
        {
            float val = SliderWidget.Cast(w).GetCurrent();
            ApplyVolume(val);
            return true;
        }
        return false;
    }

    void Close()
    {
        m_Root.Show(false);
        m_Root.Unlink();  // Remove from widget tree
    }
}
```

### WidgetAnimator

```c
// Fade in over 0.5 seconds
WidgetAnimator.Animate(widget, WidgetAnimatorProperty.ALPHA, 1.0, 0.5);

// Slide from left
WidgetAnimator.Animate(widget, WidgetAnimatorProperty.POSITION_X, 0.0, 0.3);

// Color transition
WidgetAnimator.Animate(widget, WidgetAnimatorProperty.COLOR_A, 1.0, 0.2);

// With easing
WidgetAnimator.AnimateWithEasing(widget, WidgetAnimatorProperty.ALPHA, 1.0, 0.5, EasingType.EaseOutCubic);
```

---

## Timers & Scheduling

### CallLater (One-Shot & Repeat)

```c
// One-shot after 2 seconds (2000ms)
GetGame().GetCallQueue(CALL_CATEGORY_SYSTEM).CallLater(
    this, "MyMethod", 2000, false  // false = don't repeat
);

// Repeating every 5 seconds
GetGame().GetCallQueue(CALL_CATEGORY_SYSTEM).CallLater(
    this, "MyTickMethod", 5000, true  // true = repeat
);

// With parameters
GetGame().GetCallQueue(CALL_CATEGORY_SYSTEM).CallLaterByName(
    this, "ProcessItem", 1000, false,
    new Param1<string>(itemName)
);

// Cancel
GetGame().GetCallQueue(CALL_CATEGORY_SYSTEM).Remove(this, "MyMethod");
```

### Timer Class

```c
ref Timer m_Timer = new Timer();

// Start repeating
m_Timer.Run(5.0, this, "OnTick", null, true);  // 5 sec interval, repeat

// Start one-shot
m_Timer.Run(2.0, this, "OnTimeout", null, false);

// Stop
m_Timer.Stop();

// Check
if (m_Timer.IsRunning()) { ... }
```

---

## Player System

### Getting Player Info

```c
// Current player (client-side)
PlayerBase player;
Class.CastTo(player, GetGame().GetPlayer());

// Player identity
PlayerIdentity identity = player.GetIdentity();
string steamId = identity.GetPlainId();      // Steam64 ID
string name = identity.GetName();             // Player name
string odolId = identity.GetId();             // BI ID

// Player position & direction
vector pos = player.GetPosition();
vector dir = player.GetDirection();

// Health
float health = player.GetHealth("", "Health");
float blood = player.GetHealth("", "Blood");
float shock = player.GetHealth("", "Shock");
```

### Iterating All Players

```c
// Server-side: get all connected players
ref array<Man> players = new array<Man>();
GetGame().GetPlayers(players);

foreach (Man man : players)
{
    PlayerBase player;
    if (!Class.CastTo(player, man)) continue;

    PlayerIdentity identity = player.GetIdentity();
    if (!identity) continue;

    // Process each player
}
```

### Inventory Operations

```c
// Add item to player inventory
EntityAI item = player.GetInventory().CreateInInventory("Bandage_Dressing");

// Add to specific slot (hands)
EntityAI item = player.GetHumanInventory().CreateInHands("Apple");

// Check if player has item type
bool hasItem = player.GetInventory().HasEntityInInventory(someItem);

// Remove item
player.GetInventory().LocalDestroyEntity(item);

// Iterate inventory
for (int i = 0; i < player.GetInventory().GetAttachmentSlotsCount(); i++)
{
    EntityAI attachment = player.GetInventory().GetAttachmentFromIndex(i);
    if (attachment)
    {
        // Process attachment
    }
}
```

---

## Mission Hooks

### MissionServer Lifecycle

```c
modded class MissionServer
{
    override void OnInit()
    {
        super.OnInit();
        // Initialize server-side systems
        MyManager.GetInstance();
    }

    override void OnMissionStart()
    {
        super.OnMissionStart();
        // Server is ready, players can connect
    }

    override void OnMissionFinish()
    {
        // CRITICAL: Clean up singletons here
        MyManager.DestroyInstance();
        super.OnMissionFinish();
    }

    override void InvokeOnConnect(PlayerBase player, PlayerIdentity identity)
    {
        super.InvokeOnConnect(player, identity);
        // Player connected
    }

    override void InvokeOnDisconnect(PlayerBase player)
    {
        // Player disconnecting - save data here
        super.InvokeOnDisconnect(player);
    }
}
```

### MissionGameplay Lifecycle (Client)

```c
modded class MissionGameplay
{
    override void OnInit()
    {
        super.OnInit();
        // Initialize client-side systems, UI
    }

    override void OnMissionStart()
    {
        super.OnMissionStart();
        // Client is ready
    }

    override void OnMissionFinish()
    {
        // Clean up client singletons and UI
        MyClientManager.DestroyInstance();
        super.OnMissionFinish();
    }

    override void OnUpdate(float timeslice)
    {
        super.OnUpdate(timeslice);
        // Per-frame update (client-side)
        // timeslice = seconds since last frame
    }

    override void OnKeyPress(int key)
    {
        super.OnKeyPress(key);
        // Handle key input
    }
}
```

---

## Weather & World

```c
// Weather control (server)
Weather weather = GetGame().GetWeather();
weather.GetOvercast().Set(0.8, 30, 60);    // value, minDuration, maxDuration
weather.GetRain().Set(0.5, 10, 30);
weather.GetFog().Set(0.3, 20, 40);
weather.GetWindSpeed().Set(15.0, 10, 30);

// Time control
GetGame().GetWorld().SetDate(2024, 6, 15, 12, 0);  // year, month, day, hour, min

// Surface queries
float groundY = GetGame().SurfaceY(x, z);            // Terrain height
string surface = GetGame().SurfaceGetType(x, z);       // Surface material
vector normal = GetGame().SurfaceGetNormal(x, z);      // Surface normal
bool isWater = GetGame().SurfaceIsSea(x, z);
bool isPond = GetGame().SurfaceIsPond(x, z);
```

---

## Sound

```c
// Play 3D sound at position
SEffectManager.PlaySound("MyMod_AlertSound", position);

// Play on entity
EffectSound sound = SEffectManager.PlaySoundOnObject("MyMod_ItemUse", item);

// Sound with parameters
AbstractWave wave = GetGame().CreateSoundOnObject(
    entity,
    "MyMod_SoundSet",
    0,    // distance
    false // looped
);
wave.SetVolumeRelative(0.8);
wave.Play();
```

---

## Action System

### Custom Action

```c
class ActionMyCustomAction extends ActionSingleUseBase
{
    override void CreateConditionComponents()
    {
        m_ConditionItem = new CCINonRuined();
        m_ConditionTarget = new CCTNone();
    }

    override bool HasTarget() { return false; }

    override string GetText() { return "My Custom Action"; }

    override bool ActionCondition(PlayerBase player, ActionTarget target, ItemBase item)
    {
        return item && item.IsKindOf("MySpecialItem");
    }

    override void OnExecuteServer(ActionData action_data)
    {
        // Server-side effect
        PlayerBase player = action_data.m_Player;
        ItemBase item = action_data.m_MainItem;

        // Do something
        player.SetHealth("", "Health", player.GetHealth("", "Health") + 10);
    }
}
```

### ActionContinuousBase (Progress Bar Actions)

For actions that take time (bandaging, repairing, crafting, eating):

```c
class ActionMyRepair extends ActionContinuousBase
{
    override void CreateConditionComponents()
    {
        m_ConditionItem = new CCINonRuined();         // Item must not be ruined
        m_ConditionTarget = new CCTObject(UAMaxDistances.DEFAULT);  // Must target an object
    }

    override string GetText() { return "Repair Item"; }

    override bool HasTarget() { return true; }

    // How long the action takes (seconds)
    override float GetProgress(ActionData action_data)
    {
        return 0.0;  // 0.0 = use default timing
    }

    override bool ActionCondition(PlayerBase player, ActionTarget target, ItemBase item)
    {
        if (!target) return false;
        EntityAI targetEntity;
        if (!Class.CastTo(targetEntity, target.GetObject())) return false;
        return targetEntity.GetHealth("", "Health") < targetEntity.GetMaxHealth("", "Health");
    }

    // Called when progress bar completes
    override void OnFinishProgressServer(ActionData action_data)
    {
        EntityAI target;
        if (Class.CastTo(target, action_data.m_Target.GetObject()))
        {
            target.SetHealth("", "Health", target.GetMaxHealth("", "Health"));
        }

        // Optionally consume/damage the tool
        action_data.m_MainItem.DecreaseHealth("", "Health", 10);
    }

    // Optional: called when player cancels
    override void OnEndServer(ActionData action_data)
    {
        super.OnEndServer(action_data);
    }
}
```

### Action Class Hierarchy

| Class | Use Case | Has Progress Bar |
|-------|----------|-----------------|
| `ActionSingleUseBase` | Instant actions (eat apple, check pulse) | No |
| `ActionContinuousBase` | Timed actions (bandage, repair, craft) | Yes |
| `ActionInteractBase` | World object interactions (open door, flip switch) | No |

### Common Condition Components

| Item Condition (CCI*) | Checks |
|----------------------|--------|
| `CCINonRuined` | Item is not ruined |
| `CCINone` | No item condition |
| `CCINotPresent` | No item required |

| Target Condition (CCT*) | Checks |
|------------------------|--------|
| `CCTNone` | No target required |
| `CCTObject(maxDist)` | Target is object within distance |
| `CCTSelf` | Target is self |
| `CCTMan(maxDist)` | Target is player within distance |
| `CCTCursor(maxDist)` | Target under cursor within distance |

### Register Actions on Item

```c
modded class MySpecialItem
{
    override void SetActions()
    {
        super.SetActions();
        AddAction(ActionMyCustomAction);
        AddAction(ActionMyRepair);
    }
}
```

---

## Server Economy: types.xml

When adding custom items to the Central Economy loot system:

```xml
<!-- In mpmissions/<mission>/db/types.xml -->
<type name="MyCustomItem">
    <nominal>10</nominal>        <!-- Target count on map -->
    <lifetime>3888000</lifetime> <!-- Seconds before cleanup (45 days) -->
    <restock>0</restock>         <!-- Seconds before respawn (0=immediate) -->
    <min>5</min>                 <!-- Min count before restocking -->
    <quantmin>-1</quantmin>      <!-- Quantity min % (-1=default) -->
    <quantmax>-1</quantmax>      <!-- Quantity max % (-1=default) -->
    <cost>100</cost>             <!-- Spawn priority (higher=rarer) -->
    <flags count_in_cargo="0" count_in_hoarder="0" count_in_map="1"
           count_in_player="0" crafted="0" deloot="0" />
    <category name="tools" />
    <usage name="Military" />     <!-- Where it spawns -->
    <usage name="Industrial" />
    <value name="Tier3" />        <!-- Map tier (1-4) -->
    <value name="Tier4" />
</type>
```

Key rules:
- `scope=2` must be set in your item's `CfgVehicles` config for it to be spawnable
- `nominal` = target count on the entire map (not per building)
- `lifetime` in seconds (3888000 = 45 days)
- `quantmin`/`quantmax` are percentages (0-100), not absolute counts
- Items need at least one `<usage>` and `<category>` tag to spawn
- `flags count_in_map="1"` is required for the economy to track spawned items
