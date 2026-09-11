# Build, Sign, Deploy & Test

From a folder of sources to a mod that loads on a dedicated server. This is where
correct code still fails to ship.

**Verification note.** Tool paths, CLI shapes and file layouts below were checked
against a real DayZ Tools install and real deployed Workshop mods on Windows.
Items that could not be verified locally are labelled **(convention)**.

---

## 1. The `P:\` Work Drive

Bohemia's tools (Workbench, Object Builder, AddonBuilder, Buldozer, binarize) do
not accept an arbitrary source path. They resolve asset references against a
drive **mounted as `P:\`**. Every in-PBO path (`dz\gear\...`, `MyMod\data\...`) is
implicitly rooted there.

Mount it with `<DayZ Tools>\Bin\WorkDrive\WorkDrive.exe`, or point `P:\` at a
normal folder yourself (a `subst` or a directory junction both work). What matters
is that `P:\` exists and contains:

```
P:\dz\            unpacked vanilla data (Extract Game Data from the Tools launcher)
P:\scripts\       unpacked vanilla scripts — 1_core, 2_gamelib, 3_game, 4_world, 5_mission
P:\MyMod\         your mod source
```

`P:\scripts\` is the single best reference in DayZ modding: it is the actual
vanilla script source, and it is where every API signature claim should be
verified (`P:\scripts\3_game\entities\entityai.c`, etc.).

### Mod naming rule

The mod folder name doubles as a C-style identifier in `CfgPatches`:
`[A-Za-z][A-Za-z0-9_]{0,63}`. **Hyphens do not parse.** `My-Mod` is invalid; use
`My_Mod` or `MyMod`.

---

## 2. Tool Inventory

Everything ships with DayZ Tools (Steam, free with DayZ). Verified paths, relative
to the install root:

| Tool | Path | Use |
|---|---|---|
| **AddonBuilder** | `Bin\AddonBuilder\AddonBuilder.exe` | Folder → PBO, with binarization |
| **FileBank** | `Bin\PboUtils\FileBank.exe` | Folder → PBO, **no** binarization |
| **BankRev** | `Bin\PboUtils\BankRev.exe` | PBO → folder (unpack) |
| **binarize** | `Bin\Binarize\binarize.exe` | MLOD → ODOL, config.cpp → config.bin |
| **CfgConvert** | `Bin\CfgConvert\CfgConvert.exe` | config.cpp ⇄ config.bin, standalone |
| **ImageToPAA** | `Bin\ImageToPAA\ImageToPAA.exe` | PNG/TGA → PAA |
| **TexView** | `Bin\ImageToPAA\TexView.exe` | PAA viewer/editor |
| **Object Builder** | `Bin\ObjectBuilder\ObjectBuilder.exe` | `.p3d` editing |
| **Workbench** | `Bin\Workbench\workbenchApp.exe` | Script editor/debugger, particle editor, animation |
| **DSCreateKey** | `Bin\DsUtils\DSCreateKey.exe` | Generate signing key pair |
| **DSSignFile** | `Bin\DsUtils\DSSignFile.exe` | Sign a PBO |
| **DSCheckSignatures** | `Bin\DsUtils\DSCheckSignatures.exe` | Verify signatures |
| **Publisher** | `Bin\Publisher\Publisher.exe` | Steam Workshop upload |
| **CeEditor** | `Bin\CeEditor\CeEditor.exe` | Central Economy editor |
| **TerrainBuilder** | `Bin\TerrainBuilder\terrainBuilder.exe` | Maps |
| **NavMeshGenerator** | `Bin\NavMeshGenerator\NavMeshGenerator_x64.exe` | AI navmesh for custom buildings |

Resolve the install root from `DAYZ_TOOLS_PATH` if set, otherwise the Steam
library path, before hardcoding anything.

---

## 3. Source Layout and `$PBOPREFIX$`

```
P:\MyMod\
├── $PBOPREFIX$          <-- one line: MyMod
├── config.cpp
├── mod.cpp              <-- launcher metadata, NOT packed into the PBO
├── stringtable.csv
├── data\                <-- .paa, .rvmat
├── models\              <-- .p3d, model.cfg
└── Scripts\
    ├── 3_Game\
    ├── 4_World\
    └── 5_Mission\
```

`$PBOPREFIX$` is a plain text file, no extension, whose entire content is the
prefix every file inside the PBO gets. For a mod folder `MyMod` it contains:

```
MyMod
```

This is what makes `MyMod\data\thing_co.paa` resolvable at runtime. Get it wrong
and every texture, model and script path in your configs breaks at once.

---

## 4. Building the PBO

### AddonBuilder (binarizing build)

```cmd
AddonBuilder.exe P:\MyMod P:\Mods\@MyMod\Addons -prefix=MyMod -temp=P:\temp\MyMod -clear
```

| Argument | Meaning |
|---|---|
| arg 1 | Source folder |
| arg 2 | Output folder (the PBO lands here) |
| `-prefix=` | PBO prefix (mirrors `$PBOPREFIX$`) |
| `-temp=` | Staging folder AddonBuilder copies sources into |
| `-clear` | Wipe the temp folder before staging |

**Two verified traps, both of which produce "my fix did not take effect":**

1. **Without `-clear`, the temp sync is incremental — and it can serve stale
   sources.** With `-clear` the log says *"Clearing temp folder"*; without it,
   *"Syncing folders"*, and a changed `.c` may simply not be re-copied. Observed
   in production: one script file kept a two-month-old copy in `P:\temp\<Mod>\`,
   so every build packed old code while the source on disk was correct and
   verified. The canonical symptom is **a compile error that persists unchanged
   after you edit and re-verify the source.** Always `-clear`, or delete
   `P:\temp\<Mod>` before building.

2. **AddonBuilder clears `-temp` *before* copying, and Windows paths are
   case-insensitive.** Staging your sources anywhere under `P:\temp\*` (or
   `P:\TEMP\*`) while passing `-temp=P:\temp\<X>` makes AddonBuilder delete your
   staging directory and then fail to copy. Never put sources under `P:\temp`.

Note also that AddonBuilder can exit **0 on some failure paths**. The log is the
source of truth, not the exit code.

### FileBank (pack-only, no binarization)

For script-only mods, packing without binarizing is faster and avoids binarize
failures entirely:

```cmd
CfgConvert.exe -bin -dst config.bin config.cpp
FileBank.exe -property prefix=MyMod -property "product=dayz ugc" P:\MyMod_staging
```

Produces `MyMod_staging.pbo` next to the folder. Convert `config.cpp` → `config.bin`
first and remove `config.cpp` and `mod.cpp` from the staging folder (`mod.cpp`
belongs in `@MyMod\`, not inside the PBO).

FileBank does **not** compile `.c` files — the first real compile check is the RPT
at server start.

> `CfgConvert -dst` fails **silently (exit 0, no output)** when arguments are
> passed with nested quoting through some process launchers. Run it with the
> working directory set to the config folder and relative unquoted paths, then
> validate the output by timestamp **and** location — not by "a file with that
> name exists".

### What binarization actually does

- `.p3d` MLOD → ODOL
- `config.cpp` → `config.bin`
- Validates every asset reference. A missing texture that "worked" in an unpacked
  test becomes a hard build error here.

Binarization failures are usually *reference* failures (a path that does not
resolve under `P:\`) rather than syntax failures.

---

## 5. Deployed Mod Layout

Verified against real Workshop mods in `<DayZ>\!Workshop\`:

```
@MyMod\
├── meta.cpp
├── mod.cpp
├── Addons\
│   ├── MyMod.pbo
│   └── MyMod.pbo.MyKeyName.bisign
└── Keys\
    └── MyKeyName.bikey
```

`meta.cpp` is Workshop metadata, generated by the Publisher — real example:

```cpp
protocol = 1;
publishedid = 3033197104;
name = "28addons";
timestamp = 5250891811916693254;
```

`mod.cpp` is launcher-facing and is **not** Enforce Script — plain key/value, no
class braces:

```cpp
name = "My Mod";
picture = "MyMod\mod_logo.edds";   // .edds/.paa/.tga only — PNG/JPG silently ignored
tooltip = "Shown in the launcher";
author = "Author Name";
```

---

## 6. Signing

Servers with `verifySignatures = 2` (the default, and the only supported value)
reject any PBO without a valid `.bisign` whose `.bikey` the server trusts.

```cmd
DSCreateKey.exe MyKeyName
```

produces two files:

| File | Goes where | Secrecy |
|---|---|---|
| `MyKeyName.biprivatekey` | Stays on your machine | **Never distribute** |
| `MyKeyName.bikey` | `@MyMod\Keys\`, and the server's `keys\` folder | Public |

```cmd
DSSignFile.exe MyKeyName.biprivatekey P:\Mods\@MyMod\Addons\MyMod.pbo
```

produces `MyMod.pbo.MyKeyName.bisign` — the naming pattern is
`<pbo>.<KeyName>.bisign`, verified in deployed mods.

Re-sign after **every** rebuild. A stale `.bisign` against a new PBO is a
signature mismatch, not a missing signature, and the kick message is not obvious.

---

## 7. Launching and Testing Locally

### Use the diag binary

`DayZDiag_x64.exe` (shipped with DayZ Tools / the game) runs as **either** client
or server depending on whether you pass `-server`. Retail binaries
(`DayZ_x64.exe`, `DayZServer_x64.exe`) block past the loading screen when
file patching is on, so iteration means diag on both sides.

**Server:**

```cmd
DayZDiag_x64.exe -server -config=serverDZ.cfg -profiles=<profileDir> ^
    -mission=<ABSOLUTE path to mission folder> -mod=@Mod1;@Mod2 ^
    -filePatching -port=2302
```

**Client:**

```cmd
DayZDiag_x64.exe -profiles=<clientProfileDir> -mod=@Mod1;@Mod2 ^
    -connect=127.0.0.1 -port=2302 -filePatching
```

`-mission=` **must be absolute**. If it is missing or relative the engine looks
under the binary's `mpmissions\`, finds nothing, and starts with an empty mission —
the tell in the log is *"Mission script has no main function. PlayerConnect will
stay disabled"*.

| Flag | Effect |
|---|---|
| `-mod=` | Client-and-server mods (scripts run on both sides) |
| `-servermod=` | Server-only mods; never sent to clients |
| `-filePatching` | Load loose files from disk instead of from the PBO |
| `-scriptDebug=true` | Verbose script diagnostics |
| `-profiles=` | Where logs, BattlEye and cached data go — point it at a per-run folder |
| `-window -x=1920 -y=1080` | Windowed client, useful for iteration |
| `-newErrorsAreWarnings=1` | Downgrade some fatal script errors **(convention)** |

Mod paths with spaces break `-mod` parsing on some binaries. Use junctions without
spaces (`mklink /J`) or paths relative to the working directory.

### File patching: what it does and does not reload

| Changed file | Rebuild needed? |
|---|---|
| `.c` script | No — restart the mission/server is enough |
| `.layout` | No |
| `.paa`, `.ogg` | No |
| **`config.cpp`** | **Yes — always rebuild the PBO** |
| `model.cfg` | Yes |
| `.p3d` | Yes |

This asymmetry produces a classic false diagnosis: you edit `config.cpp`, restart,
nothing changes, and you start debugging the script.

> File patching does **not** override a `.c` that is already inside the PBO with a
> loose copy on the work drive in every configuration. If a fix verified on disk
> "does not take", suspect the stale-temp problem in §4 before suspecting the game.

### `serverDZ.cfg` keys that matter for modding

```cpp
verifySignatures = 2;      // reject unsigned/mismatched PBOs (only 2 is supported)
allowFilePatching = 1;     // REQUIRED if any client connects with -filePatching
forceSameBuild = 1;        // client .exe revision must match the server's
instanceId = 1;            // selects the storage_<id> persistence folder
storageAutoFix = 1;        // replace corrupted persistence files with empty ones
class Missions
{
    class DayZ
    {
        template = "dayzOffline.chernarusplus";   // <MissionName>.<TerrainName>
    };
};
```

Set `allowFilePatching = 0` for production; keep it at `1` on the dev box.

---

## 8. BattlEye Kick Codes

| Code | Cause | Fix |
|---|---|---|
| `0x00020005` | Client connected with `-filePatching`, server has `allowFilePatching = 0` | Set `allowFilePatching = 1;` in `serverDZ.cfg` |
| `0x00010002` | Mod signature mismatch | Rebuild the PBO **and re-sign it**; check the `.bikey` is in the server `keys\` folder |
| *Public Variable Restriction* | A BattlEye filter caught unexpected network traffic | Whitelist in `publicvariable.txt` |

A signature kick after a successful build almost always means "you rebuilt and
forgot to re-sign".

---

## 9. Logs: Which File Answers Which Question

Under the `-profiles=` directory:

| File | Contains | Read it when |
|---|---|---|
| `script_<timestamp>.log` | Script compile errors, `Print()` output, script runtime errors | Your code misbehaves |
| `<name>_<timestamp>.RPT` | Engine-side errors: missing models/textures, malformed configs, addon load order, access violations | The mod does not load, or an asset is missing |
| `crash_<date>_<time>.log` | **Handled** exceptions — not hard segfaults | After an unexplained exit |

Rules that save hours:

- A missing texture, a bad `model=` path or a skipped `requiredAddons` entry show
  up **only in the RPT**, never in `script.log`. Reading only the script log is
  the most common triage mistake.
- Script errors **cascade**. Fix the *first* `SCRIPT (E)` and rebuild before
  reading the rest.
- `Print()` goes to `script_*.log`, not to the RPT.
- A misspelled `requiredAddons` entry silently skips the whole PBO — the evidence
  is an addon-not-found line in the RPT.

---

## 10. Publishing to Steam Workshop

`<DayZ Tools>\Bin\Publisher\Publisher.exe` handles upload and updates. It expects
the deployed `@MyMod\` layout from §5 and writes `meta.cpp` with the assigned
`publishedid`. **Keep `meta.cpp`** — losing it means the next upload creates a
second Workshop item instead of updating yours.

Ship both `Addons\*.pbo`, the matching `.bisign` files, and `Keys\*.bikey`.
Server owners copy the `.bikey` into their server's `keys\` folder.

---

## 11. Build → Test Loop Checklist

- [ ] `P:\` mounted; `P:\dz` and `P:\scripts` present
- [ ] `$PBOPREFIX$` matches the folder name and `-prefix=`
- [ ] `P:\temp\<Mod>` deleted, or `-clear` passed
- [ ] Build log read (not just the exit code)
- [ ] PBO re-signed after the rebuild
- [ ] `.bikey` present in `@Mod\Keys\` and in the server `keys\` folder
- [ ] Server `allowFilePatching = 1` if the client uses `-filePatching`
- [ ] `-mission=` is an absolute path
- [ ] After launch: RPT first (does it load?), then `script.log` (does it run?)

---

## Symptom → Cause Quick Table

| Symptom | First thing to check |
|---|---|
| Mod does not appear at all | RPT: CfgPatches loaded? `requiredAddons` spelled correctly? |
| Kicked at connect, code `0x00020005` | `allowFilePatching` on the server |
| Kicked at connect, signature error | Re-sign the PBO; `.bikey` in server `keys\` |
| Fix verified on disk has no effect | Stale AddonBuilder `-temp`; rebuild with `-clear` |
| `config.cpp` change ignored | Config changes need a PBO rebuild — file patching does not cover them |
| Textures missing in game, no script error | RPT, not script.log; check `$PBOPREFIX$` and in-PBO paths |
| Server starts with no players allowed in | *"Mission script has no main function"* → `-mission=` path wrong |
| Build "succeeded" but PBO is stale/empty | AddonBuilder can exit 0 on failure — read the log |
