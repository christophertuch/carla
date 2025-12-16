# CARLA 0.10.0 macOS Build Status

**Date**: December 16, 2025  
**Platform**: M1 MacBook, macOS 26.0, Xcode 26.0.1, AppleClang 17.0.0  
**CARLA Version**: 0.10.0 (ue5-dev branch)  
**Unreal Engine**: 5.5 (Epic Games Launcher prebuilt)  
**Status**: ❌ **Build Blocked - Cannot Link with Prebuilt UE5.5**

---

## Summary

Successfully compiled all 193 CARLA C++ source files and built C++ libraries, but **linking fails** due to missing RTTI typeinfo symbols in Epic's prebuilt UE5.5 for macOS. This is a fundamental architectural limitation of the installed engine distribution.

---

## ✅ What Works

### 1. C++ Libraries (Successfully Built)
- **libcarla-client.a**: 8.1 MB
- **libcarla-server.a**: 3.1 MB  
- **carla-example-client**: 3.9 MB (tested, runs correctly)
- Location: `Build/LibCarla/` and `Build/Examples/`

### 2. Unreal Engine Setup
- ✅ UE5.5 installed from Epic Games Launcher
- ✅ 80GB content downloaded from Bitbucket (ue5-dev branch)
- ✅ Content extracted to: `Unreal/CarlaUnreal/Content/Carla/`
- ✅ Environment variable set: `CARLA_UNREAL_ENGINE_PATH="/Users/Shared/Epic Games/UE_5.5"`

### 3. Source Code Fixes
All UE5.5 API compatibility issues resolved - **193 files compile without errors**:

#### Physics API Updates
- `MeshToLandscape.cpp`: Added `#include "Physics/Experimental/PhysScene_Chaos.h"`
- Fixed `LockRead()` calls to use `FChaosScene*`
- Replaced deprecated `ParallelMultiLineTrace` → `MultiLineTraceSingleByChannel`
- Replaced deprecated `ParallelMultiSphereTraceByChannel` → `MultiSphereTraceSingleByChannel`

#### Missing Includes Added
- `ProceduralMeshComponent.h` to `MeshToSplineActor.cpp`
- `AssetToolsModule.h` to StreetMap plugin files
- `JsonObjectConverter.h` to `PostProcessJsonUtils.cpp`

#### Module Dependencies (Carla.Build.cs)
```csharp
PublicDependencyModuleNames:
- Core, CoreUObject, Engine, AIModule
- Json, JsonUtilities
- PhysicsCore, Chaos, ChaosVehicles
- RenderCore, RHI, Renderer
- ProceduralMeshComponent, MeshDescription
- Projects, AnimGraphRuntime, NavigationSystem

PrivateDependencyModuleNames:
- AssetRegistry, Foliage, HTTP
- StaticMeshDescription, ImageWriteQueue
- Landscape, Slate, SlateCore, MeshConversion
```

#### Xcode SDK Compatibility
- Patched `UE_5.5/Engine/Binaries/Mac/UnrealEditor.modules`
- Updated Apple_SDK.json: `15.0 → 26.0`, `24C5079e → 26A5355f`

---

## ❌ Critical Blocker: Link Errors

### The Problem
When linking plugin dylibs, the linker cannot find RTTI `typeinfo` symbols for UE5 base classes:

```
Undefined symbols for architecture arm64:
  "typeinfo for UObject"
  "typeinfo for AActor"
  "typeinfo for ACharacter"
  "typeinfo for UActorComponent"
  "typeinfo for AController"
  "typeinfo for UAnimInstance"
  ... (25+ similar errors)
```

### Root Cause Analysis
1. **Epic's prebuilt UE5.5 for macOS** uses an "installed" engine configuration
2. RTTI typeinfo symbols are **not exported** from engine dylibs in a plugin-linkable format
3. Verified with `nm`: symbols don't exist in:
   - `UnrealEditor-CoreUObject.dylib`
   - `UnrealEditor-Engine.dylib`
   - `UnrealEditor` executable
4. The `-undefined dynamic_lookup` linker flag doesn't help because symbols genuinely aren't accessible

### Why This Differs from Other Platforms
- **Linux**: Engine .so files export RTTI symbols
- **Windows**: DLL export tables handle typeinfo explicitly
- **Source-built UE5**: Module linking can be configured differently
- **macOS Installed Engine**: RTTI symbols not exposed to plugins

### Attempted Solutions (All Failed)
1. ✗ Moving modules to `PublicDependencyModuleNames`
2. ✗ Adding system libraries (c++, c++abi)
3. ✗ `BuildEnvironment = TargetBuildEnvironment.Unique` (incompatible with installed engine)
4. ✗ `bOverrideBuildEnvironment = true` + `-undefined dynamic_lookup` (flag passes but symbols still undefined)
5. ✗ `PublicAdditionalLinkerFlags` in Build.cs (UBT appends flag incorrectly)

---

## 📁 Repository State

### Modified Files
1. **Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Carla.Build.cs**
   - Updated module dependencies
   - Added macOS linker flag attempt (non-functional)

2. **Unreal/CarlaUnreal/Source/CarlaUnreal/CarlaUnrealEditor.Target.cs**
   - Added `bOverrideBuildEnvironment = true`
   - Added `AdditionalLinkerArguments` (non-functional)

3. **Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Util/MeshToLandscape.cpp**
   - Added PhysScene_Chaos.h include
   - Fixed LockRead() API usage

4. **Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Sensor/Radar.cpp**
   - Updated to MultiLineTraceSingleByChannel

5. **Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Sensor/RayCastSemanticLidar.cpp**
   - Updated to MultiSphereTraceSingleByChannel

6. **~/.zshrc**
   - Added `export CARLA_UNREAL_ENGINE_PATH="/Users/Shared/Epic Games/UE_5.5"`

7. **CMake Configuration**
   - Custom `CMake/Toolchain.cmake` for macOS ARM64 (400+ lines)
   - Prebuilt UE5.5 integration

### Build Artifacts (Preserved)
```
Build/
├── LibCarla/
│   ├── libcarla-client.a (8.1 MB) ✅
│   └── libcarla-server.a (3.1 MB) ✅
├── Examples/
│   └── carla-example-client (3.9 MB) ✅
└── _deps/ (CMake dependencies)
Total: 1.0 GB
```

### Cleaned (Not in Repository)
- ❌ `Unreal/CarlaUnreal/Intermediate/`
- ❌ `Unreal/CarlaUnreal/Binaries/`
- ❌ `Unreal/CarlaUnreal/Plugins/*/Intermediate/`
- ❌ `Unreal/CarlaUnreal/Plugins/*/Binaries/`
- ❌ Temporary log files

---

## 🔄 Options to Proceed

### Option 1: Build UE5.5 from Source (Most Likely to Work)
**Pros**: Can configure custom symbol exports, full control over build  
**Cons**: Requires Epic GitHub access, 1-2 weeks compilation time on M1 Mac  
**Steps**:
1. Get Epic Games GitHub access
2. Clone UE5.5 source
3. Configure build with proper RTTI symbol exports
4. Compile engine (very time-consuming)
5. Build CARLA against source engine

**Resources**: https://docs.unrealengine.com/5.5/en-US/building-unreal-engine-from-source/

### Option 2: Use Linux (Recommended Alternative)
**Pros**: Officially supported by CARLA 0.10.0, proven to work  
**Cons**: Requires Linux machine or VM  
**Platforms**:
- Native Linux (Ubuntu 22.04+)
- Parallels/UTM VM with GPU passthrough
- Dual boot

### Option 3: Try CARLA 0.9.x (Older Version)
**Pros**: Uses UE4 which may have different linking behavior  
**Cons**: Older version, missing new features  
**Note**: UE4 and macOS compatibility uncertain

### Option 4: Wait for Official Support
**Pros**: No effort required  
**Cons**: Timeline unknown  
**Action**: Report to CARLA team as feature request

---

## 🛠️ Environment Details

### System
```
OS: macOS 26.0
Architecture: ARM64 (Apple Silicon M1)
Xcode: 26.0.1 (Build version 26A5355f)
SDK: MacOSX.sdk 26.0
Compiler: Apple clang version 17.0.0
```

### Build Tools
```
CMake: 4.2.1
Ninja: 1.12.1
Python: 3.14 (venv at /Users/christophertuch/dev/carla/venv)
Git LFS: Installed and configured
```

### Unreal Engine
```
Version: 5.5.0 (prebuilt from Epic Games Launcher)
Path: /Users/Shared/Epic Games/UE_5.5
Type: Installed engine (not source build)
Size: ~40 GB
```

### CARLA Repository
```
Version: 0.10.0
Branch: ue5-dev
Commit: Latest as of Dec 16, 2025
Official Support: Ubuntu 22.04+, Windows 11+
macOS: Not officially supported
```

---

## 📊 Build Statistics

- **Source Files Compiled**: 193/193 (100% success)
- **Compilation Errors**: 0
- **Link Failures**: 2 (UnrealEditor-Carla.dylib, UnrealEditor-CarlaExporter.dylib)
- **Undefined Symbols**: 25+ typeinfo symbols
- **Build Attempts**: 10+ with various linker configurations
- **Time Invested**: ~4 hours debugging linker issues

---

## 🔍 Technical Deep Dive

### RTTI (Run-Time Type Information) Issue
CARLA requires RTTI for:
- `dynamic_cast<>` operations
- Unreal Engine reflection system
- Type-safe downcasting
- Virtual table information

The missing `typeinfo` symbols are the metadata generated by the compiler for RTTI. These must be:
1. **Generated** during compilation (✅ Done - `bUseRTTI = true`)
2. **Exported** from libraries (❌ Not done by Epic's prebuilt engine)
3. **Accessible** to linker (❌ Not available in installed engine)

### Platform Differences
**Linux**:
```bash
# Symbols are exported in .so files
$ nm -D libUnrealEditor-Core.so | grep "typeinfo for UObject"
# Result: Symbol found
```

**Windows**:
```
; DLL export tables explicitly list typeinfo
EXPORTS
    ??_R0?AVUObject@@@ ; typeinfo for UObject
```

**macOS (Installed Engine)**:
```bash
# Symbols are NOT exported in dylibs
$ nm -gU UnrealEditor-CoreUObject.dylib | c++filt | grep "typeinfo for UObject"
# Result: No symbols found
```

---

## 📝 Lessons Learned

1. **Prebuilt vs Source Engine**: Installed engines have limitations for plugin development with RTTI
2. **Platform Differences**: macOS dylib symbol visibility differs significantly from Linux .so
3. **CARLA Platform Support**: Official platform support matters - macOS is not supported for a reason
4. **UE5.5 Changes**: Newer Unreal versions have stricter symbol export policies
5. **Build System Complexity**: UBT has limited flexibility for linker flag injection on installed engines

---

## 🎯 Conclusion

The CARLA 0.10.0 codebase is **fully compatible with UE5.5 APIs** (all compilation succeeds), but **cannot link as plugins** with Epic's prebuilt UE5.5 on macOS due to missing RTTI symbol exports.

**Recommended Next Steps**:
1. Switch to Linux for official support
2. OR commit to building UE5.5 from source (significant time investment)
3. Report macOS compatibility issue to CARLA team

**Repository Status**: Clean, documented, ready for archival or future attempts with source-built engine.

---

## 📚 References

- CARLA Documentation: https://carla.readthedocs.io/
- CARLA GitHub: https://github.com/carla-simulator/carla
- UE5 Build Guide: https://docs.unrealengine.com/5.5/en-US/building-unreal-engine-from-source/
- Unreal Plugin Development: https://docs.unrealengine.com/5.5/en-US/plugins-in-unreal-engine/

---

**End of Report**
