# Model & Asset Pipeline (.p3d, model.cfg, .rvmat, .paa)

Everything between "the script is correct" and "the object actually exists in the
world". Enforce Script cannot fix a broken model: if the Geometry LOD is missing,
no amount of `SetActions()` will make an action appear.

**Verification note.** Measured values in this file were read from vanilla DayZ
assets with a MLOD reader (py3d) and from the unpacked vanilla config tree. Paths
are given relative to the work drive (`P:\dz\...`, `P:\scripts\...`) so any modder
with DayZ Tools can re-check them. Claims that could not be measured locally are
labelled **(convention)** — treat them as inference per the Evidence Hierarchy in
SKILL.md §2.

---

## 1. The Four-File Contract

A custom object is never one file. Four artifacts must agree, and **they agree by
string matching, not by compilation** — every mismatch fails silently or with a
generic engine warning.

```
my_object.p3d          named selections:  "camo_body"    (geometry + selections)
        ↕
model.cfg              sections[]      =  {"camo_body"}  (class name == p3d filename)
        ↕
config.cpp             hiddenSelections[] = {"camo_body"} (CfgVehicles entry)
        ↕
data\my_object.rvmat   texture stages  →  my_object_co.paa, _nohq.paa, _smdi.paa
```

| Broken link | Symptom in game |
|---|---|
| `.p3d` selection name ≠ `model.cfg` `sections[]` | Texture swaps / hidden-selection changes do nothing |
| `model.cfg` class name ≠ `.p3d` filename | Engine falls back to `Default`: no skeleton, no sections, attachment proxies do not render |
| `config.cpp` `hiddenSelections[]` ≠ `sections[]` | `SetObjectTexture()` / `hiddenSelectionsTextures[]` has no effect |
| `.rvmat` points at a missing `.paa` | Object renders white / pink, RPT logs a missing-file warning |
| `model=` path wrong in `config.cpp` | Object spawns as an invisible entity that still collides |

The `model.cfg` class name rule is the one that costs the most hours: the class is
looked up **by the `.p3d` file name**, not by the config classname. A weapon whose
model is `sr2m_body_raised.p3d` needs `class sr2m_body_raised` even if the
CfgVehicles class is `A6_SR2M` — inheriting from the base model class is enough:

```cpp
class sr2m_body_raised: sr2m_body {};
```

---

## 2. `.p3d` Formats: MLOD vs ODOL

| Format | First bytes | Editable | Produced by | Consumed by |
|---|---|---|---|---|
| **MLOD** | `4D 4C 4F 44` = `MLOD`, then `P3DM` per LOD block | Yes | Object Builder, exporters | Object Builder, `binarize.exe` |
| **ODOL** | `4F 44 4F 4C` = `ODOL`, then version `0x36` = 54 in DayZ | No | `binarize.exe` (during PBO build) | The game engine |

(Both verified by reading the first 16 bytes of `dz\gear\camping\wooden_log.p3d`
and of its MLOD-converted copy.)

Vanilla `.p3d` files shipped in DayZ PBOs are **ODOL** — Object Builder will not
open them. Binarization has no supported reverse path (third-party ODOL→MLOD
converters exist and are lossy in practice), so treat your MLOD source as the only
source of truth and keep it under version control.

Working on a model programmatically (batch fixes, audits, procedural generation)
means working on MLOD before the build step.

---

## 3. LOD Resolutions

A `.p3d` holds N *levels of detail*, each identified by a float `resolution`. The
engine decides what a LOD is **purely from that number** — the name shown in
Object Builder is derived from it.

### Measured values

Read from two debinarized vanilla models — `dz\structures\residential\houses\house_2b02.p3d`
(a building) and `dz\gear\camping\wooden_log.p3d` (an item), both converted back
to MLOD:

| LOD | Nominal | **Value actually stored** | Present in sampled models |
|---|---|---|---|
| Visual (LOD 0..N) | `1.0`, `2.0`, `3.0`, `4.0` | same | yes (4 visual LODs in both) |
| Geometry | `1e13` | `9999999827968.0` | yes |
| Memory | `1e15` | `999999986991104.0` | yes |
| Roadway | `3e15` | `3000000028082176.0` | yes (house only) |
| Paths | `4e15` | `3999999947964416.0` | yes (house only) |
| HitPoints | `5e15` | `5000000136282112.0` | yes (house only) |
| ViewGeometry | `6e15` | `6000000056164352.0` | yes |
| FireGeometry | `7e15` | `6999999976046592.0` | yes |

> **Trap for tooling.** `resolution` is stored as a 32-bit float. `lod.resolution
> == 1e13` is **false** for a real Geometry LOD — the stored value is
> `9999999827968.0`. Always classify with a relative tolerance:
> `abs(res - nominal) / nominal < 1e-3`.

Not present in the two sampled models, listed by convention: LandContact `2e15`,
ShadowVolume `10000` / `11000`, ViewCargo `8e15`, ViewCargoGeometry `9e15`
**(convention)**.

### What each LOD does — and the symptom when it is missing

| LOD | Purpose | Missing / broken ⇒ |
|---|---|---|
| Visual | What you see | Invisible object that still blocks movement |
| **Geometry** | Physical collision, object mass | Player and vehicles walk straight through it |
| **ViewGeometry** | Cursor raycast, action targeting | **Actions never appear** — the #1 "my script doesn't work" that is not a script bug |
| **FireGeometry** | Bullet / projectile hits | Bullets pass through; object cannot be damaged by shooting |
| Memory | Named points (no faces needed) | Doors do not open, attachments spawn at origin, no inventory icon framing |
| LandContact | How the object rests on terrain | Object floats or sinks |
| Roadway | Walkable surfaces on the object | Player cannot stand on stairs / floors |
| Paths | AI navigation through the object | Infected cannot path indoors |
| HitPoints | Per-part damage zones | `dmgZones` never register hits |

**Debugging order when an object "does not work":** Visual (do I see it?) →
Geometry (do I collide?) → ViewGeometry (does the cursor find it?) → FireGeometry
(do bullets stop?). Each is a different LOD; they fail independently, and a model
can be perfect in one and broken in the next.

---

## 4. Named Selections & Components

Selections are named vertex/face subsets. Two different naming systems coexist:

### `componentNN` — collision components

Geometry, ViewGeometry and FireGeometry LODs are split into **closed convex
components** named `component01`, `component02`, … The engine treats each as one
convex collision volume.

Measured in `house_2b02`: Geometry = 127 components, ViewGeometry = 113,
FireGeometry = 472. `wooden_log` (a single item) = 1 component per collision LOD.

Rules:
- Each component must be **closed** (watertight) and **convex**. A concave shape
  must be split into several components, not left as one.
- Numbering is zero-padded to 2 digits and continues past 99 (`component100` is
  valid — verified in vanilla).
- Object Builder can generate the split automatically (Structure → Convexity)
  instead of naming them by hand **(convention)**.

### Semantic selections — texture/material targets and animation targets

Any other name is yours: `camo_body`, `plank_front`, `main_rotor`, `door_1`.
These are what `sections[]`, `hiddenSelections[]` and `model.cfg` animations
reference. Verified vanilla example — `WoodenCrate` in
`P:\dz\gear\camping\config.cpp:10074`:

```cpp
class WoodenCrate: Container_Base
{
    scope=2;
    model="\DZ\gear\camping\wooden_case.p3d";
    hiddenSelections[]={ "camoGround" };
    hiddenSelectionsTextures[]={ "\dz\gear\camping\data\wooden_case_co.paa" };
};
```

---

## 5. Memory LOD — the Named Points That Drive Behaviour

The Memory LOD carries points with **no faces**. Names are a contract with the
engine. Measured in vanilla:

### On an item (`wooden_log`)

| Selection | Meaning |
|---|---|
| `invview` | Camera framing for the inventory icon |
| `boundingbox_min` / `boundingbox_max` | Placement / snapping volume |
| `ce_center`, `ce_radius` | Central Economy placement volume for the item |

### On a building (`house_2b02`)

| Selection | Points | Meaning |
|---|---|---|
| `doors1` … `doors6` | 1 each | The door part identifier |
| `doors1_axis` … | **2 each** | Hinge axis — two points define the rotation line |
| `doors1_action` … | 1 each | Where the "Open" action is offered |
| `pointfloor` | 132 | Modeller-side floor loot hints |
| `pointtable`, `pointwardrobes`, `pointstove` | 27 / 11 / 4 | Same, per furniture class |
| `sound_rainobjectinner2metal1_1` … | 1 each | Rain impact sound emitters, name encodes surface type |

> The door triple `doorsN` + `doorsN_axis` (2 points) + `doorsN_action` is the
> complete model-side contract for an animated door. The config side is
> §7 below. Missing `_axis` ⇒ the door rotates around the model origin and swings
> through the wall.

> `pointfloor` and friends are **not** what the Central Economy reads at runtime —
> loot positions live in `mapgroupproto.xml` (`<container name="lootFloor">` with
> explicit `<point pos= range= height=>`). The counts do not match (132 memory
> points vs 8 `lootFloor` entries for the same house), so a custom building needs
> a `mapgroupproto.xml` group; adding memory points alone will not spawn loot.
> See `central-economy.md` §7. **(inferred from file contents; both files verified)**

---

## 6. Proxies

A proxy embeds another `.p3d` inside this one at a placed triangle. In the MLOD
the proxy appears as a selection whose name is the proxied path:

```
proxy:\dz\structures\furniture\cases\case_cans_b\case_cans_b.001
proxy:\dz\structures\furniture\chairs\ch_mod_c\ch_mod_c.004
```

(measured in `house_2b02` Visual LOD 1.0)

Rules:
- The suffix `.001`, `.002`, … is the **proxy index**, one per placed instance.
- The path has no `.p3d` extension and uses backslashes.
- The proxied file must exist at build time or binarization fails.
- Proxies define *where* attachments go on characters and vehicles — a wheel, a
  weapon attachment, a clothing slot. The proxy triangle's orientation determines
  the attached item's pose.

---

## 7. `model.cfg` — Skeletons, Sections, Animations

`model.cfg` lives next to the `.p3d` (or in a shared folder) and is compiled into
the PBO. It has exactly two top-level classes.

### CfgSkeletons

```cpp
class CfgSkeletons
{
    class Default
    {
        isDiscrete=1;
        skeletonInherit="";
        skeletonBones[]={};
    };
    class my_object: Default
    {
        skeletonBones[]=
        {
            "magazine","",    // pairs: bone name, parent bone name ("" = root)
            "bolt",""
        };
    };
};
```

A rigid, non-animated object still needs a skeleton entry — an empty one is fine.
`skeletonBones[]` is a flat array of **pairs**: bone, parent.

### CfgModels

```cpp
class CfgModels
{
    class Default
    {
        sectionsInherit="";
        sections[]={};
        skeletonName="";
    };
    class my_object: Default          // MUST match my_object.p3d
    {
        skeletonName="my_object";
        sections[]={ "camo_body", "zbytek" };
    };
};
```

- `sections[]` lists the selections the engine may re-texture or re-material at
  runtime. It must be a superset of `hiddenSelections[]` in `config.cpp`.
- `zbytek` ("remainder" in Czech) is the vanilla convention for "everything not
  otherwise assigned" — it appears throughout BI models.
- Damage-model fields (`htMin`, `htMax`, `afMax`, `mfMax`, `mFact`, `tBody`)
  belong here on destructible objects.

### class Animations — the piece that makes `SetAnimationPhase()` work

**This is the most common silent failure in DayZ modding**: the script calls
`SetAnimationPhase("door_1", 1)`, the code is correct, and nothing moves — because
no animation source with that name exists.

Three things must line up:

```
model.cfg   class Animations { class door_1_open { source="door_1_open"; ... } }
config.cpp  class AnimationSources { class door_1_open { source="user"; ... } }
script      SetAnimationPhase("door_1_open", 1.0);
```

Rotation example (verified in-game, helicopter door and rotor):

```cpp
class Animations
{
    class main_rotor_spin
    {
        type = "rotation";  source = "main_rotor_spin";
        selection = "main_rotor";  axis = "main_rotor_axis";
        memory = 1;  sourceAddress = "loop";
        minValue = 0;  maxValue = 1;  angle0 = 0;  angle1 = "rad 360";
    };
    class door_1_open
    {
        type = "rotation";  source = "door_1_open";
        selection = "door_1";  axis = "door_1_axis";
        memory = 1;  sourceAddress = "clamp";
        minValue = 0;  maxValue = 1;  angle0 = 0;  angle1 = "rad -95";
    };
}
```

Hide example (verified in-game, magazine visibility on a weapon):

```cpp
class magazine_hide
{
    type = "hide";  source = "magazineshow";  selection = "magazine";
    minValue = 0.0;  maxValue = 1.0;  hideValue = 0.5;
};
```

| Field | Meaning |
|---|---|
| `type` | `"rotation"`, `"translation"`, `"hide"`, `"rotationX/Y/Z"`, `"translationX/Y/Z"` |
| `source` | Name of the animation source (matches `AnimationSources` / `SetAnimationPhase`) |
| `selection` | The `.p3d` named selection that moves |
| `axis` | Memory-LOD selection holding **2 points** that define the rotation line |
| `memory` | `1` = `axis` refers to the Memory LOD |
| `sourceAddress` | `"clamp"` (stop at limits), `"loop"` (wrap), `"mirror"` |
| `angle0` / `angle1` | Rotation at `minValue` / `maxValue`; `"rad -95"` accepted |
| `hideValue` | For `type="hide"`: selection is hidden when source ≥ this |

The `config.cpp` side, verified in `P:\dz\vehicles\wheeled\config.cpp:351`:

```cpp
class AnimationSources
{
    class DoorsDriver
    {
        source="user";      // driven by script via SetAnimationPhase
        initPhase=0;
        animPeriod=0.5;     // seconds for a full 0→1 transition
    };
    class AnimHitWheel_1_1
    {
        source="Hit";                  // driven by the damage system
        hitpoint="HitWheel_1_1";
    };
};
```

`source="user"` = script-driven. `source="Hit"` = bound to a damage zone. A
source declared in `model.cfg` but **not** in `config.cpp` `AnimationSources`
cannot be driven from script.

---

## 8. Textures: `.paa` and the Suffix Convention

DayZ does not load PNG or JPG at runtime. Textures are `.paa` (DXT-compressed),
converted with `ImageToPAA.exe` (CLI) or `TexView.exe` (GUI), both in
`<DayZ Tools>\Bin\ImageToPAA\`.

```
ImageToPAA.exe source.tga target_co.paa
```

**Dimensions must be powers of two** (512, 1024, 2048…). Non-PoT input is either
rejected or silently rescaled.

### Suffixes are semantic, not decorative

The converter and the shader pick behaviour from the suffix. Census over the
unpacked vanilla tree (`P:\dz`, `.paa` files):

| Suffix | Count | Meaning |
|---|---|---|
| `_lco` | 6152 | Color, "lossless"/uncompressed variant |
| `_co` | 5998 | **Color / albedo** |
| `_nohq` | 3767 | **Normal map**, high quality |
| `_lca` | 3072 | Color+alpha, uncompressed |
| `_smdi` | 2602 | **Specular map** (specular / mask / detail intensity) |
| `_ca` | 1529 | Color with alpha (transparency) |
| `_mc` | 1173 | Macro / multi-color mask |
| `_as` | 1052 | Ambient shadow (baked AO) |
| `_mask` | 922 | Layer mask (multi-material blending) |
| `_ads` | 431 | Alpha-decal shadow |
| `_dt` | 135 | Detail texture |

Minimum viable set for a new object: `_co` + `_nohq` + `_smdi`.
Transparency (glass, foliage, netting) requires `_ca`, not `_co`.

---

## 9. `.rvmat` — Materials

A `.rvmat` is a config-format file that binds shader parameters and texture
stages. Verified structure — `P:\dz\gear\camping\data\barbed_wire.rvmat`:

```cpp
ambient[]={1,1,1,1};
diffuse[]={1,1,1,1};
forcedDiffuse[]={0,0,0,0};
emmisive[]={0,0,0,1};        // note the vanilla spelling: two m's
specular[]={3,3,3};
specularPower=60;
PixelShaderID="Super";
VertexShaderID="Super";
class Stage1 { texture="dz\gear\camping\data\barbed_wire_nohq.paa"; uvSource="tex"; class uvTransform { aside[]={1,0,0}; up[]={0,1,0}; dir[]={0,0,1}; pos[]={0,0,0}; }; };
class Stage2 { texture="#(argb,8,8,3)color(0.5,0.5,0.5,1,DT)";  ... };
class Stage3 { texture="#(argb,8,8,3)color(0,0,0,0,MC)";        ... };
class Stage4 { texture="#(argb,8,8,3)color(1,1,1,1,AS)";        ... };
class Stage5 { texture="#(argb,8,8,3)color(1,0.0,0.5,1,SMDI)";  ... };
class Stage6 { texture="#(ai,64,64,1)fresnel(1.05,0.67)";       ... };
class Stage7 { texture="dz\data\data\env_land_co.paa";          ... };
```

### Stage order for the `Super` shader — memorise this

| Stage | Map | Typical value |
|---|---|---|
| 1 | Normal (`_nohq`) | real texture |
| 2 | Detail (`DT`) | procedural placeholder or `_dt.paa` |
| 3 | Macro (`MC`) | procedural placeholder or `_mc.paa` |
| 4 | Ambient shadow (`AS`) | procedural placeholder or `_as.paa` |
| 5 | Specular (`SMDI`) | real texture or placeholder |
| 6 | Fresnel | `#(ai,64,64,1)fresnel(a,b)` |
| 7 | Environment map | `dz\data\data\env_land_co.paa` |

The `#(argb,8,8,3)color(r,g,b,a,TYPE)` form is a **procedural placeholder**: an
inline 8×8 constant texture. Use it for any stage you are not authoring — do not
delete the stage, because stage *position* is what the shader reads.

The `_co` (color) texture is **not** in the `.rvmat` for hidden-selection objects:
it comes from `hiddenSelectionsTextures[]` in `config.cpp`. For non-hidden-selection
objects it is Stage0/implicit from the model's material assignment.

### Damage material swaps

`healthLevels[]` in `config.cpp` swaps the `.rvmat` as the object degrades —
verified in `P:\dz\gear\camping\config.cpp` (`WoodenCrate`):

```cpp
class DamageSystem
{
    class GlobalHealth
    {
        class Health
        {
            hitpoints=400;
            healthLevels[]=
            {
                { 1.0, { "DZ\gear\camping\data\wooden_case.rvmat" } },
                { 0.7, { "DZ\gear\camping\data\wooden_case.rvmat" } },
                { 0.5, { "DZ\gear\camping\data\wooden_case_damage.rvmat" } },
                { 0.3, { "DZ\gear\camping\data\wooden_case_damage.rvmat" } },
                { 0.0, { "DZ\gear\camping\data\wooden_case_destruct.rvmat" } }
            };
        };
    };
};
```

Use `healthLevels[]`. `healthLevelValues[]` is legacy and breaks the damage system
silently.

---

## 10. Importing From Blender / Other DCC: the Winding Trap

Blender is **Z-up**; DayZ is **Y-up**. Exporting through the axis conversion
reverses the effective *winding order* (the order of a face's corners). The engine
decides which side of a face is "outside" from the winding — **not** from the
declared normal. So an imported model can have perfect normals and still be wrong.

### Symptoms — they appear one at a time, per LOD

| Symptom | LOD with reversed winding |
|---|---|
| Object looks hollow; texture only visible from inside | Visual |
| Bullets pass through | FireGeometry |
| Actions never appear / cursor does not detect it | ViewGeometry |
| Player walks through the object | Geometry |

Because each LOD is independent, "the texture is fine but bullets pass through"
is a completely normal presentation of this single root cause.

### Fix

Reverse the corner order of every face, in **every** LOD:

```python
import py3d
with open(path, 'rb') as f:
    model = py3d.P3D(f)
for lod in model.lods:            # ALL LODs, not just Visual
    for face in lod.faces:
        face.vertices.reverse()
with open(path, 'wb') as f:
    model.write(f)
```

- Apply **uniformly**. Mixed winding is worse than fully-reversed winding: parts
  of the mesh render from outside, parts from inside, and the result is not
  reproducible between viewing angles.
- Do **not** use the old "swap vertices\[1] and vertices\[2]" trick — it only works
  for triangles. Quads and n-gons need a full `reverse()`.
- `reverse()` does not touch the face-normal pool. In the MLOD format the normals
  are a **global pool indexed per vertex** (`vertex.normal_index`), not a per-face
  array; reversing corner order leaves each corner pointing at the same pool entry,
  which is correct.
- The operation is its own inverse — running it twice returns to the original.
  Verify before re-running.

---

## 11. `.p3d` Named Properties

Set in Object Builder via *Edit → Named Properties*. They live in the model, not
in any config.

| Property | Typical value | Effect |
|---|---|---|
| `autocenter` | `0` | Do not recentre the model on load — required for held items so the grip point stays put |
| `mass` | kg | Physics mass; belongs on the Geometry LOD |
| `class` | `house`, `car`, … | Engine behaviour category |
| `mapType` | `building`, `vehicle`, … | Icon shown on the in-game map |
| `damage` | selection name | Marks a damage zone **(convention)** |

---

## 12. Pre-Build Checklist for a Custom Object

- [ ] `.p3d` has Visual + Geometry + ViewGeometry + FireGeometry (+ Memory if it
      animates, has attachments, or needs an inventory icon)
- [ ] Collision LODs split into closed convex `componentNN` selections
- [ ] Geometry LOD has a `mass` named property
- [ ] Winding verified per LOD if the model came from a DCC tool
- [ ] `model.cfg` class name **equals the `.p3d` filename**
- [ ] `sections[]` ⊇ `hiddenSelections[]`
- [ ] Every `SetAnimationPhase()` name exists in `model.cfg` `class Animations`
      **and** in `config.cpp` `class AnimationSources`
- [ ] Every rotation animation has a 2-point `_axis` selection in the Memory LOD
- [ ] Textures are power-of-two `.paa` with correct suffixes
- [ ] `.rvmat` stage order intact; unused stages left as procedural placeholders
- [ ] All texture/material paths in `config.cpp` and `.rvmat` use the in-PBO path
      (`ModName\data\...`), not a local disk path
- [ ] `healthLevels[]` (not `healthLevelValues[]`) if the object is destructible

---

## Symptom → Cause Quick Table

| Symptom | First thing to check |
|---|---|
| Object invisible but blocks movement | Visual LOD missing/empty, or `model=` path wrong |
| Object visible, player walks through | Geometry LOD missing or non-convex components |
| No action prompt on a correct script | **ViewGeometry LOD** |
| Bullets pass through | **FireGeometry LOD** |
| Object white / pink | `.rvmat` or `.paa` path broken (check RPT, not script.log) |
| Texture only from inside | Reversed winding, Visual LOD |
| `SetAnimationPhase` does nothing | Missing `class Animations` entry or missing `AnimationSources` |
| Door swings through the wall | Missing/1-point `doorsN_axis` in Memory LOD |
| Attachment spawns at object origin | Missing proxy or missing Memory point |
| Retexture via `SetObjectTexture` ignored | Selection not listed in `sections[]` / `hiddenSelections[]` |
| Object floats above terrain | LandContact LOD missing |
