# Struct nativization test (Chivalry 2 / ArgonSDK)

Can a Chivalry 2 mod ship its own **user-defined structs** (and enums) in its PAK? Yes, but only if each one is excluded from Blueprint nativization when you cook. This repository holds a small A/B test that shows it.

## The problem

ArgonSDK (UE 4.25.4) ships with Blueprint nativization switched on: `BlueprintNativizationMethod=Exclusive` in `Config/DefaultGame.ini`, listing some of the game's Blueprints. "Exclusive" sounds like only the listed Blueprints are converted, but in UE 4.25 that filter applies to Blueprint classes only. **Every user-defined enum and struct is converted to C++ during the cook unless it is excluded.** The converted type is left out of the cooked content, while your Blueprints still reference it, and the cook still reports `Success - 0 error(s)`. The generated C++ never ships in a mod, so the mod loses those types.

Engine source (UE 4.25, `Engine/Source/Developer/BlueprintNativeCodeGen/Private/BlueprintNativeCodeGenModule.cpp`, `IsTargetedForReplacement`):

- the same decision covers UserDefinedStruct, UserDefinedEnum and Blueprint classes;
- `ExcludedAssets` is checked for all three;
- the Exclusive "only selected" rule and `ExcludedFolderPaths` apply to Blueprint classes only;
- an excluded asset can still be converted if it has a circular reference with a converted asset.

A cook-by-the-book run nativizes whenever the project setting is not `Disabled` (`CookOnTheFlyServer.cpp`), with no extra command-line flag.

## The fix

Add one line per struct or enum to `Config/DefaultEditor.ini` of the SDK copy you cook with, then recook:

```ini
[BlueprintNativizationSettings]
+ExcludedAssets=/Game/Mods/<YourMod>/<Path>/<StructOrEnumName>
```

Check that each type's `.uasset` / `.uexp` is in the PAK listing (`UnrealPak <pak> -List`). A matching hash or a successful cook does not prove the types are there.

## The test

Two parts, both in `Content/` for the package path `/Game/Mods/AgMods/StructNativizationTest`.

### 1. Package A/B (does the type reach the PAK?)

| Asset | Excluded for the cook | Result |
| --- | --- | --- |
| `Types/S_SNT_Excluded` (struct) | yes | cooked and in the PAK |
| `Types/S_SNT_Control` (struct) | no | converted to C++, **missing** from the PAK |
| `Types/E_SNT_Excluded` (enum) | yes | cooked and in the PAK |
| `Types/E_SNT_Control` (enum) | no | converted to C++, **missing** (proves nativization ran) |
| `BP_SNT_Holder` | n/a | in the PAK, with one variable of each type; still references both control types |

### 2. In-game probe (does the struct work at runtime?)

`FFA_StructNativizationTest` is a small FFA room. `Blueprints/BP_SNT_Probe` stands at the north wall next to a grey reference cube. On BeginPlay it:

1. makes an `S_SNT_Excluded` value with its bool member set to true,
2. stores it in a variable of that struct type,
3. reads the variable back, breaks the struct and branches on the bool.

True: the panel shows **STRUCT OK** and the green cube grows into a 6 m pillar. False: **STRUCT READ FALSE**. If BeginPlay never runs, the panel keeps "BeginPlay did not run"; if the probe class fails to load, the probe is missing. The probe uses only the excluded struct.

Result in Chivalry 2 (Unchained, 2026-11-11): **STRUCT OK** with the pillar grown.

Evidence level: **package** for the A/B and **in game** for the excluded struct (ArgonSDK, UE 4.25.4). This shows that the struct type loads from the mod PAK and carries a value written at runtime through a variable. Not tested: struct default values saved in assets or set on placed actors, arrays of structs, data tables, replication and SaveGame.

### Reproduce

1. Copy `Content/` into your SDK so it appears as `/Game/Mods/AgMods/StructNativizationTest` (a copy or a directory junction).
2. Add the two `_Excluded` paths to `[BlueprintNativizationSettings]` in `Config/DefaultEditor.ini`:
   ```ini
   +ExcludedAssets=/Game/Mods/AgMods/StructNativizationTest/Types/S_SNT_Excluded
   +ExcludedAssets=/Game/Mods/AgMods/StructNativizationTest/Types/E_SNT_Excluded
   ```
3. Cook the probe map and the folder: `UE4Editor-Cmd <ProjectRoot>/TBL.uproject -run=cook -targetplatform=WindowsNoEditor -map=/Game/Mods/AgMods/StructNativizationTest/FFA_StructNativizationTest -COOKDIR=/Game/Mods/AgMods/StructNativizationTest`.
4. Package A/B: in `<ProjectRoot>/Saved/Cooked/WindowsNoEditor/TBL/Content/Mods/AgMods/StructNativizationTest` the control types are absent; their generated C++ is under `<ProjectRoot>/Intermediate/Plugins/NativizedAssets/`.
5. In game: **before packing, delete the cooked `BP_SNT_Holder.uasset` / `.uexp`** (it references the missing control types and does not belong in a playable PAK). Pack the rest under `../../../TBL/Content/Mods/AgMods/StructNativizationTest/`, install it like any mod PAK, start an FFA match on `FFA_StructNativizationTest` and look at the north wall.

Remove the two ini lines again afterwards if you don't want them in your project. `Blueprints/BP_MazeStartup` is the match-startup helper from [MirrorMaze](https://github.com/SageFlare/MirrorMaze) (zeroes the start wait and starts respawn waves).

## More

The [Chivalry 2 Mod Wiki](https://github.com/SageFlare/Chivalry2-Mod-Wiki) covers this under Blueprint authoring, along with other ArgonSDK and Unchained modding notes.

License: GPL-3.0 (see [LICENSE](LICENSE)).
