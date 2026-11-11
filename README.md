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

`Content/` holds the assets, for the package path `/Game/Mods/AgMods/StructNativizationTest`:

| Asset | Excluded for the cook | Result |
| --- | --- | --- |
| `Types/S_SNT_Excluded` (struct) | yes | cooked and in the PAK |
| `Types/S_SNT_Control` (struct) | no | converted to C++, **missing** from the PAK |
| `Types/E_SNT_Excluded` (enum) | yes | cooked and in the PAK |
| `Types/E_SNT_Control` (enum) | no | converted to C++, **missing** (proves nativization ran) |
| `BP_SNT_Holder` | n/a | in the PAK, with one variable of each type; still references both control types |

Tested with ArgonSDK on UE 4.25.4 (2026-11-11). Evidence level: **package** (cooked output and PAK listing) and **in game** for the excluded struct: a probe Blueprint in a test map wrote a value into `S_SNT_Excluded` at runtime, read it back through a variable of that type and branched on it correctly in Chivalry 2 (2026-11-11). Not tested: struct default values saved in assets, replication, SaveGame.

### Reproduce

1. Copy `Content/` into your SDK so it appears as `/Game/Mods/AgMods/StructNativizationTest` (a copy or a directory junction).
2. Add the two `_Excluded` paths to `[BlueprintNativizationSettings]` in `Config/DefaultEditor.ini`:
   ```ini
   +ExcludedAssets=/Game/Mods/AgMods/StructNativizationTest/Types/S_SNT_Excluded
   +ExcludedAssets=/Game/Mods/AgMods/StructNativizationTest/Types/E_SNT_Excluded
   ```
3. Cook (`UE4Editor-Cmd <ProjectRoot>/TBL.uproject -run=cook -targetplatform=WindowsNoEditor -map=<any small map> -COOKDIR=/Game/Mods/AgMods/StructNativizationTest`) and look in `<ProjectRoot>/Saved/Cooked/WindowsNoEditor/TBL/Content/Mods/AgMods/StructNativizationTest`.
4. The control types are absent; their generated C++ is under `<ProjectRoot>/Intermediate/Plugins/NativizedAssets/`.

Remove the two lines again afterwards if you don't want them in your project.

## More

The [Chivalry 2 Mod Wiki](https://github.com/SageFlare/Chivalry2-Mod-Wiki) covers this under Blueprint authoring, along with other ArgonSDK and Unchained modding notes.

License: GPL-3.0 (see [LICENSE](LICENSE)).
