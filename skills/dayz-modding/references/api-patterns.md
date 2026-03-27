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

---

## Vehicle System

### Class Hierarchy
```
Transport > Car > CarScript (scripted vehicles)
Transport > Boat > BoatScript
```

### Vehicle Fluids
```c
// CarFluid enum: FUEL, OIL, BRAKE, COOLANT
float fuel = car.GetFluidFraction(CarFluid.FUEL);    // 0.0-1.0
float capacity = car.GetFluidCapacity(CarFluid.FUEL); // Liters
car.Fill(CarFluid.FUEL, 50.0);                        // Add 50L
car.Leak(CarFluid.OIL, 5.0);                          // Remove 5L
```

### Engine Control
```c
car.EngineStart();
car.EngineStop();
bool running = car.EngineIsOn();
float rpm = car.EngineGetRPM();
float speed = car.GetSpeedometer();                    // km/h
```

### Crew Management
```c
int seats = car.CrewSize();
Man driver = car.CrewMember(0);                        // 0 = driver
car.CrewGetOut(0);                                      // Eject seat 0
// Player checks
bool inVehicle = player.IsInVehicle();
Transport vehicle = player.GetDrivingVehicle();
```

### Spawn Ready Vehicle
```c
CarScript car = CarScript.Cast(GetGame().CreateObject("OffroadHatchback", pos, false, false, true));
if (car)
{
    car.Fill(CarFluid.FUEL, car.GetFluidCapacity(CarFluid.FUEL));
    car.Fill(CarFluid.OIL, car.GetFluidCapacity(CarFluid.OIL));
    car.Fill(CarFluid.BRAKE, car.GetFluidCapacity(CarFluid.BRAKE));
    car.Fill(CarFluid.COOLANT, car.GetFluidCapacity(CarFluid.COOLANT));
    car.OnDebugSpawn();  // Attaches all parts (wheels, doors, battery, etc.)
}
```

### CarScript Callbacks
```c
modded class CarScript
{
    override void OnEngineStart() { super.OnEngineStart(); }
    override void OnEngineStop() { super.OnEngineStop(); }
    override void OnContact(string zoneName, vector localPos, IEntity other, Contact data) { super.OnContact(zoneName, localPos, other, data); }
    override void OnFluidChanged(CarFluid fluid, float newValue, float oldValue) { super.OnFluidChanged(fluid, newValue, oldValue); }
}
```

---

## Camera System

```c
// Global camera accessors
vector camPos = GetCurrentCameraPosition();
vector camDir = GetCurrentCameraDirection();

// World-to-screen conversion
vector screenPos = GetGame().GetScreenPos(worldPos);        // Absolute pixels
vector screenRel = GetGame().GetScreenPosRelative(worldPos); // 0.0-1.0 relative
// screenPos[2] > 0 means in front of camera

// Camera type detection
int camType = player.GetCurrentCameraType();
// DayZPlayerCameras: DAYZCAMERA_1ST, DAYZCAMERA_3RD_ERC, DAYZCAMERA_IRONSIGHTS, DAYZCAMERA_OPTICS, DAYZCAMERA_3RD_VEHICLE

// Free debug camera
FreeDebugCamera.SetActive(true);
FreeDebugCamera cam = GetFreeDebugCamera();
cam.SetPosition(pos);

// DOF control
GetGame().GetWorld().SetDOF(focusDist, focusLength, nearBlur, blurAmount, offset);
```

---

## Post-Process Effects (PPE)

```c
// Get a requester from the bank
PPERequester req = PPERequesterBank.GetRequester(PPERequesterBank.REQ_INVENTORYBLUR);

// Start/Stop
req.Start();
req.Stop();

// Custom PPE requester
class MyPPERequester extends PPERequester
{
    override protected void OnStart(Param par = null)
    {
        super.OnStart(par);
        // Vignette effect
        SetTargetValueFloat(PostProcessEffectType.Glow, PPEGlow.PARAM_VIGNETTE, false, 0.8, PPEManager.L_0_STATIC, PPOperators.SET);
        // Color grading
        SetTargetValueFloat(PostProcessEffectType.ColorGrading, PPEColorGrading.PARAM_SATURATION, false, 0.3, PPEManager.L_0_STATIC, PPOperators.SET);
    }
}

// PPOperators: SET, ADD, ADD_RELATIVE, HIGHEST, LOWEST, MULTIPLY, OVERRIDE
// Common requesters: REQ_INVENTORYBLUR, REQ_CAMERANV, REQ_DEATHEFFECTS, REQ_BLOODLOSS, REQ_FLASHBANGEFFECTS
// Priority layers: L_0_STATIC (0), L_1_VALUES (1), L_2_SCRIPTS (2), L_3_EFFECTS (3), L_4_OVERLAY (4), L_LAST (100)
```

---

## Notification System

```c
// Server to specific player
NotificationSystem.SendNotificationToPlayerExtended(player, 10, "Title", "Message", "set:dayz_gui image:icon_info");

// Server broadcast to all
NotificationSystem.SendNotificationToPlayerIdentityExtended(null, 10, "Broadcast", "Server message", "");

// Client-side local notification
NotificationSystem.AddNotificationExtended(5, "Local", "Client-only message", "set:dayz_gui image:icon_info");
```

---

## Input System

### Polling Input in OnUpdate
```c
modded class MissionGameplay
{
    private UAInput m_MyAction;  // Cache the input reference

    override void OnInit()
    {
        super.OnInit();
        m_MyAction = GetUApi().GetInputByName("UAMyModAction");
    }

    override void OnUpdate(float timeslice)
    {
        super.OnUpdate(timeslice);
        if (!m_MyAction) return;
        if (!GetGame().GetPlayer()) return;
        if (GetGame().GetUIManager().GetMenu()) return;  // Skip if menu open

        if (m_MyAction.LocalPress())
            OnMyActionPressed();
    }
}
```

### Input State Methods
```c
UAInput input = GetUApi().GetInputByName("UAMyAction");
input.LocalPress();       // True for ONE frame when key first pressed
input.LocalRelease();     // True for ONE frame when key released
input.LocalHold();        // True while key held (after hold threshold)
input.LocalHoldBegin();   // True for one frame when hold threshold reached
input.LocalClick();       // True for one frame on quick press+release
input.LocalDoubleClick(); // True for one frame on double-click
input.LocalValue();       // Float 0.0-1.0 (for analog inputs)
```

### Input Suppression
```c
input.Supress();                             // Suppress current frame (note: single 's')
input.ForceDisable(true);                    // Disable until re-enabled
GetGame().GetMission().AddActiveInputExcludes({"menu"}); // Block gameplay inputs
GetGame().GetMission().RemoveActiveInputExcludes({"menu"}, true); // Restore
```

---

## Crafting System

### Recipe-Based Crafting
```c
class MyRecipe extends RecipeBase
{
    override void Init()
    {
        m_Name = "#STR_craft_stone_knife";
        m_IsInstaRecipe = false;            // Uses progress bar
        m_AnimationLength = 2.0;            // Actual time = 2.0 * 4.0 = 8.0 seconds
        m_Specialty = -0.01;                // Negative = precise, positive = rough

        // Ingredient 0: sharp stone
        InsertIngredient(0, "SmallStone");
        m_IngredientAddHealth[0] = -20;     // Damage stone by 20
        m_IngredientDestroy[0] = false;

        // Ingredient 1: stick
        InsertIngredient(1, "WoodenStick");
        m_IngredientDestroy[1] = true;      // Consume stick

        // Result
        AddResult("StoneKnife");
        m_ResultToInventory[0] = -2;        // -2 = place on ground, -1 = player inventory
    }

    override bool CanDo(ItemBase ingredients[], PlayerBase player)
    {
        return true;  // Additional conditions
    }

    override void Do(ItemBase ingredients[], PlayerBase player, array<ItemBase> results, float specialty_weight)
    {
        // Post-craft logic
    }
}
```

---

## Animation System

### Movement State
```c
HumanMovementState state = new HumanMovementState();
player.GetMovementState(state);

// Stance: STANCEIDX_ERECT (0), CROUCH (1), PRONE (2), RAISEDERECT (3), RAISEDCROUCH (4), RAISEDPRONE (5)
int stance = state.m_iStanceIdx;

// Movement: IDLE (0), WALK (1), RUN (2), SPRINT (3)
int movement = state.m_iMovement;
```

### Object Animations
```c
// Animation phase (0.0-1.0) — used for doors, gates, hatches
entity.SetAnimationPhase("doors_driver", 1.0);  // Open
entity.SetAnimationPhase("doors_driver", 0.0);  // Close
float phase = entity.GetAnimationPhase("doors_driver");
```

### Emotes
```c
// Trigger emote programmatically
EmoteManager mgr = player.GetEmoteManager();
if (mgr) mgr.CreateEmoteCBFromMenu(EmoteConstants.ID_EMOTE_DANCE);
```

---

## Terrain & World Queries

```c
// Surface height
float groundY = GetGame().SurfaceY(x, z);              // Raw terrain
float roadY = GetGame().SurfaceRoadY(x, z);             // Including roads

// Surface type (material name)
string surfaceType;
GetGame().SurfaceGetType(x, z, surfaceType);

// Surface normal (slope direction)
vector normal = GetGame().SurfaceGetNormal(x, z);

// Water queries
bool isSea = GetGame().SurfaceIsSea(x, z);
bool isPond = GetGame().SurfaceIsPond(x, z);

// Raycasting
vector from = player.GetPosition() + "0 1.5 0";
vector to = from + player.GetDirection() * 50;
vector contactPos, contactDir;
int contactComponent;
set<Object> hitObjects = new set<Object>();

if (DayZPhysics.RaycastRV(from, to, contactPos, contactDir, contactComponent, hitObjects, null, player, false, false, ObjIntersectView, 0.0, CollisionFlags.NEARESTCONTACT))
{
    // contactPos = hit point, hitObjects[0] = what was hit
}

// ObjIntersect: Fire, View, Geom, IFire, None
// CollisionFlags: FIRSTCONTACT, NEARESTCONTACT, ONLYSTATIC, ONLYDYNAMIC, ALLOBJECTS
```

---

## Particle Effects

```c
// High-level API — Particle static methods (legacy, simple)
Particle p = Particle.PlayOnObject(ParticleList.CAMP_FIRE_START, entity, "0 0 0");
Particle p2 = Particle.PlayInWorld(ParticleList.CAMP_SMALL_SMOKE, worldPos);

// Modern API — ParticleManager (recommended, pool-based)
ParticleManager pm = ParticleManager.GetInstance();
if (pm)
{
    ParticleSource ps = pm.PlayOnObject(ParticleList.CAMP_SMALL_FIRE, entity, "0 0.5 0");
    ParticleSource ps2 = pm.PlayInWorld(ParticleList.SMOKE_GENERIC_WRECK, worldPos);
}

// Stop
p.StopParticle();
p.StopParticle(StopParticleFlags.IMMEDIATE);  // Instant stop

// Runtime parameter tuning
p.SetParameter(-1, EmitorParam.VELOCITY, 5.0);
p.SetParameter(-1, EmitorParam.SIZE, 2.0);
p.SetParameter(-1, EmitorParam.BIRTH_RATE, 50.0);
p.ScaleParticleParamFromOriginal(EmitorParam.SIZE, 1.5);

// Common ParticleList IDs: CAMP_FIRE_START, CAMP_SMALL_SMOKE, CAMP_SMALL_FIRE,
// BLEEDING_SOURCE, EXPLOSION_LANDMINE, BONFIRE_FIRE, SMOKE_GENERIC_WRECK
```

---

## Zombie AI System

### Class Hierarchy
```
DayZCreature > DayZCreatureAI > DayZInfected > ZombieBase > ZombieMaleBase/ZombieFemaleBase
```

### Mind States
```c
// Via input controller
int mindState = zombie.GetInputController().GetMindState();
// MINDSTATE_CALM (0), DISTURBED (1), ALERTED (2), CHASE (3), FIGHT (4)
```

### Modding Zombie Behavior
```c
modded class ZombieBase
{
    override bool ModCommandHandlerBefore(float dt, int currentCommand, bool isHandled)
    {
        // Execute BEFORE vanilla command processing
        return super.ModCommandHandlerBefore(dt, currentCommand, isHandled);
    }

    override bool ModCommandHandlerInside(float dt, int currentCommand, bool isHandled)
    {
        // Execute DURING vanilla command processing
        return super.ModCommandHandlerInside(dt, currentCommand, isHandled);
    }

    override bool ModCommandHandlerAfter(float dt, int currentCommand, bool isHandled)
    {
        // Execute AFTER vanilla command processing
        return super.ModCommandHandlerAfter(dt, currentCommand, isHandled);
    }
}
```

---

## Admin & Server

### Player Management
```c
// Get all players
ref array<Man> players = new array<Man>();
GetGame().GetPlayers(players);

// Player identity details
PlayerIdentity id = player.GetIdentity();
string steamId = id.GetPlainId();     // Steam64 ID
string beGuid = id.GetId();           // BattlEye GUID
string name = id.GetName();

// Network stats
int ping = id.GetPingAct();
int bandwidth = id.GetBandwidthAvg();

// Kick player
GetGame().DisconnectPlayer(identity);

// Admin log
GetGame().AdminLog("Player " + name + " performed action X");
```

### Chat
```c
// Send chat message (server-side)
GetGame().ChatMP(player, "Admin message", "colorImportant");
// Color classes: "colorStatusChannel", "colorImportant", "colorAction", "colorFriendly"
```

### Time Control
```c
int year, month, day, hour, min;
GetGame().GetWorld().GetDate(year, month, day, hour, min);
GetGame().GetWorld().SetDate(2024, 6, 15, 12, 0);        // Set to noon
bool night = GetGame().GetWorld().IsNight();
```

---

## Construction System

### Base Building Classes
```c
// BaseBuildingBase > Fence, Watchtower, ShelterSite
// Construction parts managed via Construction class

// Check if part can be built
Construction construction = building.GetConstruction();
if (construction.CanBuildPart("wall_base_down", player, true))
{
    construction.BuildPartServer("wall_base_down", AT_BUILD_WOOD);
}

// Visual state via animation phases
building.SetAnimationPhase("wall_base_down", 0.0);  // Built (visible)
building.SetAnimationPhase("wall_base_down", 1.0);  // Unbuilt (hidden)

// Physics
building.AddProxyPhysics("wall_base_down");     // Enable collision
building.RemoveProxyPhysics("wall_base_down");  // Disable collision
```

---

## Sound System

### Config-Based Sounds
```cpp
// In config.cpp
class CfgSoundShaders
{
    class MyMod_AlertShader
    {
        samples[] = { {"MyMod\sounds\alert", 1} };  // Path WITHOUT .ogg extension
        volume = 1.0;
        range = 50;       // Meters — silence beyond this
        radius = 5;       // Meters — full volume within this
    };
};

class CfgSoundSets
{
    class MyMod_AlertSoundSet
    {
        soundShaders[] = { "MyMod_AlertShader" };
        spatial = 1;      // 1=3D world sound, 0=2D UI sound
        // 3D sounds MUST be mono .ogg files
    };
};
```

### Script API
```c
// Play at position (recommended for one-shots)
EffectSound sound = SEffectManager.PlaySound("MyMod_AlertSoundSet", worldPos);
sound.SetAutodestroy(true);

// Play on entity
EffectSound snd;
player.PlaySoundSet(snd, "MyMod_AlertSoundSet", 0, 0);
player.StopSoundSet(snd);

// UI sound (2D, no spatialization)
AbstractWave wave = GetGame().CreateSoundOnObject(null, "MyMod_UISoundSet", 0, false);
wave.SetVolumeRelative(0.5);
wave.Play();
```

---

## Central Economy Files Reference

| File | Purpose |
|------|---------|
| `types.xml` | Item spawn definitions (nominal, lifetime, locations) |
| `events.xml` | Dynamic event spawns (vehicles, helicrashes, airdrops) |
| `globals.xml` | Economy parameters (animal/zombie counts, cleanup timers) |
| `cfgspawnabletypes.xml` | Pre-attached items and cargo on spawned entities |
| `cfgrandompresets.xml` | Reusable loot pool definitions |
| `cfgeconomycore.xml` | Root class registration, CE folder config |
| `cfglimitsdefinition.xml` | Valid category/usage/value flag definitions |
| `cfgplayerspawnpoints.xml` | Player spawn locations |

### globals.xml Key Parameters
```xml
<var name="AnimalMaxCount" type="0" value="200"/>
<var name="ZombieMaxCount" type="0" value="1000"/>
<var name="CleanupLifetimeDeadPlayer" type="0" value="3600"/>
<var name="CleanupLifetimeDeadInfected" type="0" value="330"/>
<var name="FlagRefreshFrequency" type="0" value="432000"/>
<var name="LootDamageMin" type="1" value="0.0"/>
<var name="LootDamageMax" type="1" value="0.82"/>
<var name="TimeLogin" type="0" value="15"/>
<var name="TimeLogout" type="0" value="15"/>
```
