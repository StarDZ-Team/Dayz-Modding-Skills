# Enforce Script Language Reference

Complete reference for Enforce Script — the scripting language used by DayZ's Enfusion engine. This is NOT C/C++/C#.

---

## Type System

### Primitive Types

| Type | Default | Size | Notes |
|------|---------|------|-------|
| `int` | `0` | 32-bit | Signed integer. Float-to-int truncates (no rounding) |
| `float` | `0.0` | 32-bit | IEEE 754 single precision |
| `bool` | `false` | — | `true` / `false` only |
| `string` | `""` | — | VALUE type (copied on assign/pass). Immutable content |
| `vector` | `"0 0 0"` | 3 floats | VALUE type. Literal format: `"x y z"` (spaces, not commas) |
| `typename` | `null` | — | Represents a type at runtime. Used for reflection |
| `void` | — | — | No return value |

### Value vs Reference Types

**Value types** (copied on assignment): `int`, `float`, `bool`, `string`, `vector`
**Reference types** (reference counted): All `class` instances

```c
// Value type behavior - independent copies
string a = "hello";
string b = a;      // b is a COPY
b = "world";       // a is still "hello"

// Reference type behavior - shared object
ref MyClass a = new MyClass();
MyClass b = a;     // b points to SAME object
b.value = 42;      // a.value is also 42
```

### Type Constants

```c
int.MAX;           // 2147483647
int.MIN;           // -2147483648
float.MAX;         // 3.402823e+38
float.MIN;         // 1.175494e-38
float.LOWEST;      // -3.402823e+38
string.Empty;      // ""
vector.Zero;       // "0 0 0"
vector.Up;         // "0 1 0"
vector.Aside;      // "1 0 0"
vector.Forward;    // "0 0 1"
```

---

## Collections

### array<T>

```c
// Declaration
ref array<string> names = new array<string>();
autoptr array<int> nums = {};  // NOT USED — prefer ref

// Core methods
names.Insert("Alice");              // Append → returns index
names.InsertAt("Bob", 0);           // Insert at position
names.Get(0);                       // Access by index (or names[0])
names.Set(0, "Charlie");            // Replace at index
names.Find("Alice");                // Find index (-1 if not found)
names.Remove(0);                    // Remove at index
names.RemoveItem("Bob");            // Remove by value
names.RemoveOrdered(0);             // Remove preserving order
names.Clear();                      // Remove all
names.Count();                      // Size
names.IsValidIndex(5);              // Bounds check
names.Sort();                       // Ascending sort (primitives)
names.Invert();                     // Reverse order
names.ShuffleArray();               // Randomize
names.GetRandomElement();           // Random element
names.Copy(otherArray);             // Deep copy from another array
names.InsertAll(otherArray);        // Append all from another
```

### map<K, V>

```c
ref map<string, int> scores = new map<string, int>();

scores.Insert("player1", 100);     // Add/update key
scores.Get("player1");             // Get value (0/null if missing)
scores.Contains("player1");        // Key exists?
scores.Find("player1", outVal);    // Find → returns bool, fills outVal
scores.Remove("player1");          // Remove key
scores.Count();                    // Size
scores.Clear();                    // Remove all
scores.GetKey(0);                  // Key at internal index
scores.GetElement(0);              // Value at internal index
scores.GetKeyArray(outArray);      // Copy all keys to array
scores.GetValueArray(outArray);    // Copy all values to array
```

### set<T>

```c
ref set<string> tags = new set<string>();

tags.Insert("vip");                // Add (no duplicates)
tags.Find("vip");                  // Index (-1 if missing)
tags.Remove(0);                    // Remove at index
tags.Count();                      // Size
tags.Clear();                      // Remove all
```

---

## Class System

### Declaration & Inheritance

```c
class Animal
{
    protected string m_Name;
    protected int m_Health;

    void Animal(string name)        // Constructor (same name as class)
    {
        m_Name = name;
        m_Health = 100;
    }

    void ~Animal()                  // Destructor
    {
        // cleanup
    }

    string GetName() { return m_Name; }
    void TakeDamage(int dmg) { m_Health = m_Health - dmg; }
}

class Dog extends Animal
{
    private string m_Breed;

    void Dog(string name, string breed)
    {
        // super constructor called automatically if no-arg
        // for parameterized: must match parent constructor signature
        m_Name = name;
        m_Breed = breed;
    }

    override void TakeDamage(int dmg)
    {
        super.TakeDamage(dmg / 2); // Dogs are tough
    }
}
```

### Access Modifiers

| Modifier | Access |
|----------|--------|
| (none) | Public (default) |
| `private` | Same class only |
| `protected` | Same class + subclasses |

### Static Members

```c
class Counter
{
    private static int s_Count = 0;

    static int GetCount() { return s_Count; }
    static void Increment() { s_Count++; }
}

// Usage: Counter.Increment(); Counter.GetCount();
```

### Modded Classes

Extend existing classes WITHOUT modifying original source. The engine chains all `modded` declarations.

```c
modded class PlayerBase
{
    protected float m_Stamina;

    override void Init()
    {
        super.Init();          // ALWAYS call super
        m_Stamina = 100.0;
    }

    float GetStamina() { return m_Stamina; }

    override void OnDamageEvent(TotalDamageResult dmg)
    {
        super.OnDamageEvent(dmg);
        m_Stamina = m_Stamina - 10.0;
    }
}
```

**Rules:**
- Always call `super.MethodName()` in overrides
- Multiple mods can `modded` the same class — they chain in load order
- Cannot add new constructors, only override existing methods or add new ones
- Cannot change a method's signature

---

## Memory Management

### Reference Counting (ARC)

Objects are destroyed when their strong reference count reaches zero.

```c
// ref = strong reference (keeps object alive)
ref MyClass obj = new MyClass();    // refcount = 1
ref MyClass other = obj;            // refcount = 2
other = null;                       // refcount = 1
obj = null;                         // refcount = 0 → DESTROYED

// raw pointer = weak reference (does NOT keep alive)
MyClass weak = obj;                 // does NOT increment refcount
// If all ref holders release, weak becomes dangling
```

### Preventing Leaks

```c
// LEAK - circular reference
class Parent
{
    ref Child m_Child;    // strong ref to child
}
class Child
{
    ref Parent m_Parent;  // strong ref to parent — LEAK!
}

// FIXED - one side uses raw pointer
class Child
{
    Parent m_Parent;      // raw (weak) ref to parent — no leak
}
```

### Cleanup Pattern

```c
class MyManager
{
    private static ref MyManager s_Instance;
    private ref array<ref MyObject> m_Objects = new array<ref MyObject>();

    static void DestroyInstance()
    {
        if (s_Instance)
        {
            s_Instance.Cleanup();
            s_Instance = null;
        }
    }

    void Cleanup()
    {
        m_Objects.Clear();  // releases all ref'd objects
    }
}
```

---

## Control Flow

```c
// if / else if / else
if (health <= 0)
    Die();
else if (health < 25)
    Limp();
else
    Run();

// for loop
for (int i = 0; i < items.Count(); i++)
{
    ProcessItem(items[i]);
}

// foreach (value only)
foreach (ItemBase item : items)
{
    item.Update();
}

// foreach (index + value)
foreach (int idx, ItemBase item : items)
{
    Print(string.Format("Item %1: %2", idx, item.GetType()));
}

// foreach on map (key + value)
foreach (string key, int value : myMap)
{
    Print(key + " = " + value.ToString());
}

// while (replaces do-while with break)
while (true)
{
    DoWork();
    if (ShouldStop()) break;
}

// switch (DOES fall through — always add break!)
switch (action)
{
    case 0:
        HandleIdle();
        break;           // Without break, falls through to case 1!
    case 1:
        HandleRun();
        break;
    default:
        HandleUnknown();
        break;
}
```

---

## String Operations

```c
string s = "Hello World";

s.Length();                              // 11
s.Substring(0, 5);                      // "Hello"
s.IndexOf("World");                     // 6 (-1 if not found)
s.Contains("World");                    // true
s.Replace("World", "DayZ");            // Modifies s IN PLACE, returns int count
s.ToLower();                           // Modifies s IN PLACE, returns int length
s.ToUpper();                           // Modifies s IN PLACE, returns int length
s.Trim();                              // Remove whitespace
s.TrimInPlace();                       // In-place trim
s.Split(" ", outArray);                // Split into array

// Conversion
string.Format("HP: %1/%2", current, max);  // "HP: 75/100"
(42).ToString();                            // "42"
"42".ToInt();                              // 42
"3.14".ToFloat();                          // 3.14
"1.0 2.0 3.0".ToVector();                 // vector(1,2,3)

// Concatenation
string full = "Hello" + " " + "World";
```

---

## Math Operations

```c
// Constants
Math.PI;                    // 3.14159265...
Math.DEG2RAD;               // PI / 180
Math.RAD2DEG;               // 180 / PI

// Random
Math.RandomInt(0, 100);     // [0, 100) exclusive upper
Math.RandomFloat01();       // [0.0, 1.0]
Math.RandomFloat(1.0, 5.0); // [1.0, 5.0]

// Rounding
Math.Round(2.7);            // 3.0
Math.Floor(2.9);            // 2.0
Math.Ceil(2.1);             // 3.0

// Utility
Math.AbsFloat(-5.0);       // 5.0
Math.AbsInt(-5);            // 5
Math.Clamp(val, min, max);  // Clamp to range
Math.Min(a, b);
Math.Max(a, b);
Math.Lerp(a, b, t);        // Linear interpolation
Math.InverseLerp(a, b, v); // Reverse lerp
Math.Pow(base, exp);
Math.Sqrt(x);
Math.Log2(x);

// Trigonometry (all in RADIANS)
Math.Sin(rad);
Math.Cos(rad);
Math.Tan(rad);
Math.Asin(x);
Math.Acos(x);
Math.Atan2(y, x);          // Angle from components
```

---

## Vector Operations

```c
vector pos = "100.0 20.0 50.0";
vector dir = "0.0 0.0 1.0";

// Properties
pos[0];                     // X component (100.0)
pos[1];                     // Y component (20.0)
pos[2];                     // Z component (50.0)

// Methods
vector.Distance(a, b);     // Distance between two points
pos.Length();               // Magnitude
pos.LengthSq();            // Squared magnitude (faster)
pos.Normalized();           // Unit vector
vector.Lerp(a, b, t);      // Linear interpolation
vector.Direction(from, to); // Direction vector

// Arithmetic
vector sum = pos + dir;
vector scaled = pos * 2.0;
float dot = vector.Dot(a, b);

// Common patterns
float dist = vector.Distance(playerPos, targetPos);
vector moveDir = vector.Direction(playerPos, targetPos).Normalized();
vector newPos = playerPos + moveDir * speed * deltaTime;
```

---

## Casting & Reflection

```c
// Safe downcast (ALWAYS use this)
PlayerBase player;
if (Class.CastTo(player, someEntity))
{
    // player is valid and correct type
}

// Inline cast (shorter but less readable)
PlayerBase player = PlayerBase.Cast(someEntity);
if (player)
{
    // valid
}

// Type checking
if (entity.IsInherited(PlayerBase))
{
    // entity is a PlayerBase or subclass
}

// Runtime type name
string typeName = entity.ClassName();     // "PlayerBase"
typename type = entity.Type();            // typename object

// Create by typename
typename t = MyClass;
Class instance = t.Spawn();               // Creates new instance
```

---

## Enums & Preprocessor

### Enums

```c
enum EWeatherState
{
    CLEAR,          // 0
    CLOUDY,         // 1
    RAINY,          // 2
    STORMY          // 3
}

// Usage
EWeatherState weather = EWeatherState.RAINY;
int val = weather;                    // 2
string name = typename.EnumToString(EWeatherState, weather); // "RAINY"
```

### Bitflags

```c
enum EPermFlags
{
    NONE   = 0,
    READ   = 1,
    WRITE  = 2,
    EXEC   = 4,
    ALL    = 7     // READ | WRITE | EXEC
}

// Usage
int perms = EPermFlags.READ | EPermFlags.WRITE;
bool canWrite = (perms & EPermFlags.WRITE) != 0;
```

### Preprocessor

```c
#ifdef SERVER
    // Server-only code
#endif

#ifndef STARDZ_CORE
    // Fallback when StarDZ Core not loaded
#endif

#ifdef DEVELOPER
    Print("Debug info: " + data);
#endif
```

---

## Error Handling Pattern

Since try/catch does NOT exist, use the guard clause pattern:

```c
bool LoadConfig(string path)
{
    // Guard: file exists?
    if (!FileExist(path))
    {
        SDZ_Log.Error("MyMod", "Config file not found: " + path);
        return false;
    }

    // Guard: valid config object?
    MyConfig cfg = new MyConfig();
    JsonFileLoader<MyConfig>.JsonLoadFile(path, cfg);

    if (!cfg)
    {
        SDZ_Log.Error("MyMod", "Failed to parse config: " + path);
        return false;
    }

    // Guard: valid values?
    if (cfg.Radius <= 0)
    {
        SDZ_Log.Warn("MyMod", "Invalid radius, using default");
        cfg.Radius = 100.0;
    }

    // All guards passed — safe to use
    ApplyConfig(cfg);
    return true;
}
```

---

## Global Functions

```c
// Object creation
GetGame().CreateObject(className, position, createFlags, iLocal);
GetGame().CreateObjectEx(className, position, createFlags, type);
GetGame().ObjectDelete(object);

// Game queries
GetGame().GetPlayer();              // Returns Man (cast to PlayerBase!)
GetGame().IsServer();
GetGame().IsClient();
GetGame().IsMultiplayer();
GetGame().GetWorld();
GetGame().GetMission();

// Time
GetGame().GetTime();                // Milliseconds since start
GetGame().GetTickTime();            // Current frame time

// Printing
Print("Debug message");             // Console output
PrintString("Also console");
ErrorEx("Error message");           // Error with stack trace
```

---

## Additional Gotchas & Edge Cases

These are traps that go beyond the Iron Rules in SKILL.md. Each has caused real bugs in production mods.

### Collection Traps

1. **`array.Remove(index)` is UNORDERED** — It swaps the target element with the last element, then removes the last. Array order is NOT preserved. Use `RemoveOrdered(index)` when order matters.

2. **Static arrays are fixed-size** — `int arr[5]` is compile-time fixed. You cannot resize it. Use `array<int>` for dynamic collections.

3. **Iterating while modifying** — Removing elements during a `foreach` loop corrupts iteration. Collect indices to remove, then remove in reverse order after the loop:
   ```c
   // CORRECT: collect then remove
   array<int> toRemove = new array<int>();
   foreach (int idx, EntityAI ent : m_Entities)
   {
       if (!ent) toRemove.Insert(idx);
   }
   for (int i = toRemove.Count() - 1; i >= 0; i--)
   {
       m_Entities.RemoveOrdered(toRemove[i]);
   }
   ```

### String Traps

4. **`ToLower()` and `ToUpper()` modify in place and return `int` (length)** — They do NOT return a new string:
   ```c
   string name = "Hello";
   name.ToLower();         // name is now "hello" (modified in place!)
   // NOT: string lower = name.ToLower(); // Returns int, not string!
   ```

5. **`TrimInPlace()` also modifies in place** — Same behavior as `ToLower()`.

6. **`Replace()` modifies in place and returns `int` (replacement count)** — Does NOT return a new string:
   ```c
   string s = "Hello World";
   int count = s.Replace("World", "DayZ"); // s is now "Hello DayZ", count is 1
   // NOT: string result = s.Replace(...); // Returns int, not string!
   ```

### Class & Type Traps

6. **No nested class declarations** — You cannot define a class inside another class. All classes are top-level.

7. **No enum validation** — Enums can be set to any integer value. `EWeatherState state = 999;` compiles and runs. There is no runtime check.

8. **Default parameter expressions only accept literals or NULL** — You cannot write `void Func(int x = GetValue())`. Defaults must be compile-time constants.

9. **Constructor name must match class name exactly** — Including case. If the constructor name differs, it becomes a regular method.

### Lifecycle Traps

10. **No destructor guarantee on server shutdown** — If the server process is killed (crash, SIGKILL), destructors may never run. Do not rely on destructors for critical saves. Use periodic auto-save instead.

11. **No RAII (scope-based resource management)** — Objects are NOT auto-cleaned when variables go out of scope (unless the variable is the last `ref`). You must explicitly manage lifetime.

12. **`Managed` class prevents engine from tracking the object** — The engine's garbage collector ignores `Managed` subclasses. Only use for objects whose lifecycle you fully control in script.

### Compilation Traps

13. **Multiline method call with comments fails** — Splitting a chained call across lines with comments between can confuse the parser. Keep method chains on one line or use intermediate variables.

14. **File order within a layer is alphabetical** — The engine loads `.c` files in alphabetical order within each layer folder. If class B depends on class A, ensure A's file sorts before B's (or put both in the same file).

15. **`proto` methods without bodies make a class abstract** — You cannot `new` a class that has unimplemented `proto` methods. Implement all proto methods or use a concrete subclass.

### Network Traps

16. **`SetSynchDirty()` must be called after changing synced variables** — Without it, the engine does not broadcast changes to clients. This is the #1 cause of "data not syncing" bugs.

17. **Net Sync Variable registration must happen in the constructor** — Call `RegisterNetSyncVariableInt()`, `RegisterNetSyncVariableFloat()`, `RegisterNetSyncVariableBool()` in the entity constructor. Override `OnVariablesSynchronized()` to react on the client side.

18. **Read/write order in RPC MUST match exactly** — If the sender writes `string, int, float`, the receiver must read `string, int, float` in that exact order. A single mismatch corrupts all subsequent reads.

---

## Additional Language Features

### auto Keyword
```c
auto count = 10;           // Inferred as int
auto name = "Hello";       // Inferred as string
auto player = GetGame().GetPlayer(); // Inferred as Man
```

### const Keyword
```c
const int MAX_PLAYERS = 60;
const float GRAVITY = 9.81;
const string MOD_NAME = "MyMod";
// const only works on value types. No const for reference types/objects.
```

### Truthiness Rules
What evaluates to `false`: `0` (int), `0.0` (float), `""` (empty string), `null` (reference). Everything else is `true`. Short-circuit evaluation applies: `&&` stops if left is false, `||` stops if left is true.

### null vs NULL
Both `null` and `NULL` work. `nullptr` does NOT exist. Idiomatic null check: `if (!obj)`.

### Implicit Type Conversions
| From | To | Behavior |
|------|-----|----------|
| `int` | `float` | Safe (no loss) |
| `float` | `int` | Truncates (use `Math.Round()`) |
| `int`/`float` | `bool` | Non-zero = `true` |
| `bool` | `int` | `true` = 1, `false` = 0 |

---

## Classes: Additional Details

### Constructor Overloading
Multiple constructors with different parameter lists are supported on regular classes:
```c
class MyClass
{
    void MyClass() { /* no-arg */ }
    void MyClass(string name) { /* one-arg */ }
    void MyClass(string name, int id) { /* two-arg */ }
}
```

### Colon Syntax for Inheritance
`class Truck : BaseVehicle` is equivalent to `class Truck extends BaseVehicle`.

### sealed Keyword
Prevents a class from being inherited: `sealed class FinalClass { }`.

### Parameter Modifiers
| Modifier | Semantics |
|----------|-----------|
| `out` | Write-only — caller reads after call returns |
| `inout` | Read+write — caller sees modifications |
| `notnull` | Compiler enforces non-null at call site |

### Modded Class Special Rules
- **Can access `private` members** of the original class (special modded keyword rule)
- **Can override `const` values** from the original class
- **Field naming**: Use mod-specific prefixes (`m_MyMod_Points`) to avoid collisions with other mods
- **CfgPatches class names become `#ifdef`-testable symbols** automatically

### DayZ Class Hierarchy
```
Class
  └── Managed (script-only, no engine GC)
  └── IEntity
        └── Object
              └── Entity
                    └── EntityAI
                          ├── ItemBase (Inventory_Base)
                          │     ├── Weapon_Base
                          │     ├── Magazine_Base
                          │     └── Clothing_Base
                          ├── Man → DayZPlayer → PlayerBase → SurvivorBase
                          ├── Transport → Car → CarScript
                          ├── DayZCreatureAI
                          │     ├── DayZInfected → ZombieBase
                          │     └── DayZAnimal → AnimalBase
                          ├── Building → BuildingBase
                          └── BaseBuildingBase (player constructions)
```

---

## Functions & Methods: Additional Details

### Method Overloading is NOT Supported
Two methods with the same name cause a compile error. Workarounds:
- Different names: `Send()`, `SendToPlayer()`, `SendToAll()`
- `Ex()` suffix convention: `ExplosionEffects()` → `ExplosionEffectsEx()` (DayZ vanilla pattern)
- Default parameters: `void Send(string data, bool guaranteed = true)`

### Standalone (Global) Functions
Functions can exist outside any class. Rare but used in vanilla for utility helpers.

### event Keyword
Marks methods as engine event handlers called at specific moments:
```c
event void OnPossess();
protected event void GetTransform(inout vector transform[4]);
```

### proto Modifier Variants
| Modifier | Meaning |
|----------|---------|
| `proto native` | Implemented in engine C++ code |
| `proto native owned` | Returns a value the caller owns (memory) |
| `proto volatile` | Function may yield/sleep or call back into script |
| `proto native external` | Binding for cross-module calls |

### thread Keyword (Coroutines)
NOT OS threads — coroutine-like constructs:
```c
// Declaration
thread void LongOperation()
{
    Print("Step 1");
    Sleep(2000);        // Wait 2000ms — MUST be in threaded context
    Print("Step 2");
}

// Start
thread LongOperation();

// Kill
KillThread(owner, "LongOperation");
```
`Sleep(int ms)` is an engine intrinsic (no proto declaration). Calling outside a threaded context crashes.

### CallLater API
```c
// One-shot after 2 seconds
GetGame().GetCallQueue(CALL_CATEGORY_SYSTEM).CallLater(this, "MyMethod", 2000, false);

// Repeating every 5 seconds
GetGame().GetCallQueue(CALL_CATEGORY_SYSTEM).CallLater(this, "MyMethod", 5000, true);

// Cancel
GetGame().GetCallQueue(CALL_CATEGORY_SYSTEM).Remove(this, "MyMethod");

// Categories: CALL_CATEGORY_SYSTEM (0), CALL_CATEGORY_GUI (1), CALL_CATEGORY_GAMEPLAY (2)
```

---

## Collections: Additional Details

### Pre-Defined Typedefs
```c
TStringArray     = array<string>
TFloatArray      = array<float>
TIntArray        = array<int>
TBoolArray       = array<bool>
TVectorArray     = array<vector>
TStringIntMap    = map<string, int>
TStringStringMap = map<string, string>
TIntStringMap    = map<int, string>
TStringFloatMap  = map<string, float>
```

### array Additional Methods
```c
names.Reserve(100);             // Pre-allocate capacity (no Count change)
names.Resize(50);               // Change element count (fill with defaults or truncate)
names.Sort(true);               // true = descending order
names.GetRandomIndex();         // Random valid index
names.Debug();                  // Print all elements to script log
```

### map.Insert() vs map.Set() — CRITICAL
- `map.Insert(key, val)` — Does NOT update if key exists (returns false, value unchanged)
- `map.Set(key, val)` — Insert-or-update (always succeeds)
```c
scores.Insert("player1", 100);  // Added
scores.Insert("player1", 200);  // IGNORED! Still 100
scores.Set("player1", 200);     // Updated to 200
```

### array<ref T> vs array<T> — Ownership
- `array<ref MyClass>` — Strong references: array keeps objects alive
- `array<MyClass>` — Weak references: objects may be destroyed while in array
```c
ref array<ref MyObj> owned = new array<ref MyObj>();   // Objects live as long as array holds them
ref array<MyObj> weak = new array<MyObj>();             // Objects may become null if destroyed elsewhere
```

### Static Arrays
- Fixed-size: `int arr[5]` — cannot resize
- Passed by reference to functions (unlike primitives)
- Out-of-bounds = undefined behavior (no runtime check)
- Can initialize: `float dmg[3] = {10.5, 25.0, 50.0};`
- `foreach` works: `foreach (int val : arr) { }`

---

## Memory: Additional Details

### Managed vs Non-Managed Weak Reference Safety
- **Managed classes**: Weak references auto-set to `null` when object deleted (safe)
- **Non-Managed classes**: Weak references become dangling pointers on delete (crash!)

### delete Keyword
Manually destroys an object immediately regardless of refcount. All references set to null (on Managed classes):
```c
MyClass obj = new MyClass();
delete obj;  // Destroyed immediately, obj becomes null
```

### autoptr Explanation
`autoptr` is a scoped strong reference, identical to `ref` but intended for local variables. Since local variables are already implicitly strong references, `autoptr` is redundant in practice. Use `ref` instead.

---

## Enums: Additional Details

### Enum Inheritance
```c
enum EBaseColor { RED, GREEN, BLUE }
enum EExtendedColor : EBaseColor { YELLOW, CYAN }
// YELLOW = 3, CYAN = 4 (continues from parent)
// Parent values accessible: EExtendedColor.RED works
```

### COUNT Sentinel Pattern
```c
enum EMode { EASY, MEDIUM, HARD, COUNT }
for (int i = 0; i < EMode.COUNT; i++) { /* iterate all modes */ }
```

### Common Engine Defines
| Define | When Active |
|--------|------------|
| `SERVER` | Dedicated server AND listen server |
| `DEVELOPER` | DayZDiag executable |
| `DIAG_DEVELOPER` | DayZDiag executable |
| `PLATFORM_WINDOWS` | Windows build |
| `PLATFORM_XBOX` | Xbox build |
| `PLATFORM_PS4` | PlayStation build |
| `BUILD_EXPERIMENTAL` | Experimental branch |

### Custom Defines via config.cpp
```cpp
class CfgMods { class MyMod { ... class defs { defines[] = { "MYMOD_ENABLED" }; }; }; };
```
Since DayZ 1.21, the CfgMods class name is auto-registered as a `#define`.

### #define Only Creates Existence Flags
No value substitution: `#define MAX_HP 100` does NOT work. Use `const int MAX_HP = 100;` instead.

### #else Directive
```c
#ifdef SERVER
    // Server code
#else
    // Client code
#endif
```

### Bitflag Removal
```c
flags = flags & ~EPermFlags.WRITE;  // Remove WRITE flag
```

---

## Reflection: Additional Details

### EnScript.GetClassVar / SetClassVar
Read/write member variables by name at runtime:
```c
int result = EnScript.GetClassVar(obj, "m_Health", 0, outValue);  // 1=success, 0=failure
int result = EnScript.SetClassVar(obj, "m_Health", 0, newValue);  // index=0 for non-arrays
```
Fails silently on wrong variable names.

### typename Reflection Methods
```c
typename t = MyClass;
int count = t.GetVariableCount();
for (int i = 0; i < count; i++)
{
    string name = t.GetVariableName(i);
    typename type = t.GetVariableType(i);
}
```

### IsKindOf vs IsInherited — Different Hierarchies
- `IsKindOf(string)` — Checks **config.cpp** class hierarchy. Defined on `Object` only. Slower.
- `IsInherited(typename)` — Checks **script class** hierarchy. Defined on `Class`. Faster.
```c
entity.IsKindOf("Apple");              // Checks config.cpp CfgVehicles hierarchy
entity.IsInherited(ItemBase);          // Checks Enforce Script class hierarchy
```

---

## Error Handling: Additional Details

### ErrorEx() with Severity Levels
```c
ErrorEx("Something happened", ErrorExSeverity.INFO);      // Info in .RPT log
ErrorEx("Potential issue", ErrorExSeverity.WARNING);       // Warning in .RPT
ErrorEx("Critical failure", ErrorExSeverity.ERROR);        // Error in .RPT with stack trace
```
Does NOT stop execution. Writes to `.RPT` log file.

### DumpStackString()
Capture current call stack as a string:
```c
string stack;
DumpStackString(stack);
Print(stack);
```

### JsonFileLoader Modern API
Newer `LoadFile()` returns `bool` (vs legacy `JsonLoadFile()` returning `void`):
```c
MyConfig cfg = new MyConfig();
if (!JsonFileLoader<MyConfig>.LoadFile(path, cfg))
{
    Print("Failed to load config");
}
```

---

## Math: Additional Functions

```c
// Inclusive random (upper bound included)
Math.RandomIntInclusive(1, 10);     // [1, 10] inclusive both ends
Math.RandomFloatInclusive(0.0, 1.0);

// Utility
Math.RandomBool();                  // Random true/false
Math.Randomize(-1);                 // Seed RNG (-1 = system time)
Math.SignFloat(f);                  // Returns -1.0, 0.0, or 1.0
Math.SignInt(i);                    // Returns -1, 0, or 1
Math.SqrFloat(f);                  // Square (not sqrt)
Math.ModFloat(a, b);               // Float modulo
Math.WrapInt(val, min, max);       // Wrap into range
Math.IsInRange(val, min, max);     // Range check

// Angles (degrees)
Math.NormalizeAngle(deg);           // Normalize to [0, 360)
Math.DiffAngle(a, b);              // Shortest path between angles

// Smooth interpolation
Math.SmoothCD(current, target, velocity[], smoothTime, maxSpeed, dt);
// velocity MUST be a float array (modified in place)

// Constants
Math.PI2;                           // 2 * PI
Math.PI_HALF;                       // PI / 2
Math.EULER;                         // Euler's number
```

---

## String: Additional Methods

```c
s.IndexOfFrom(5, "World");         // Find from index 5 onward
s.LastIndexOf("o");                 // Find last occurrence
s.Get(0);                           // Get char at index (returns string)
s.Set(0, "h");                      // Set char at index

// Static join
string.Join(", ", myArray);         // Join array with separator

// Number formatting
int num = 42;
num.ToStringLen(5);                 // "00042" (zero-padded)
```

---

## Vector: Additional Methods

```c
// Constructor function (alternative to string literal)
vector pos = Vector(100.0, 20.0, 50.0);

// Squared distance (faster, no sqrt)
float distSq = vector.DistanceSq(a, b);

// In-place normalize (modifies original)
pos.Normalize();                    // vs Normalized() which returns new

// Rotation
vector rotated = vector.RotateAroundZeroDeg(vec, axis, angleDeg);

// Random directions
vector dir = vector.RandomDir();     // Random unit vector 3D
vector dir2d = vector.RandomDir2D(); // Random unit vector XZ plane

// DayZ coordinate system
// [0] = X = East(+) / West(-)
// [1] = Y = Up(+) / Down(-) (altitude)
// [2] = Z = North(+) / South(-)

// Math3D for matrix operations
vector angles;
Math3D.MatrixToAngles(transform, angles);
Math3D.YawPitchRollMatrix(angles, outMatrix);
Math3D.MatrixIdentity4(outMatrix);
```

---

## Operator Precedence (Highest to Lowest)

| Precedence | Operators |
|-----------|-----------|
| 1 | `()` `[]` `.` `::` |
| 2 | `!` `~` `-` (unary) `++` `--` |
| 3 | `*` `/` `%` |
| 4 | `+` `-` |
| 5 | `<<` `>>` |
| 6 | `<` `<=` `>` `>=` |
| 7 | `==` `!=` |
| 8 | `&` |
| 9 | `^` |
| 10 | `|` |
| 11 | `&&` |
| 12 | `||` |
| 13 | `=` `+=` `-=` `*=` `/=` `%=` `&=` `|=` `^=` `<<=` `>>=` |
