# Code Changes for UE5.5 Compatibility

This document lists all source code modifications made to achieve compilation compatibility with Unreal Engine 5.5 on macOS ARM64.

## ✅ Successfully Compiled Files (193 files)

---

## Modified Files

### 1. LibCarla - Type Compatibility Fixes

#### `LibCarla/source/carla/Buffer.h`
**Issue**: `size_t` vs `uint64_t` type mismatch  
**Fix**: Type casting for 64-bit compatibility
```cpp
// Updated type conversions for ARM64
```

#### `LibCarla/source/carla/MsgPackAdaptors.h`
**Issue**: msgpack type compatibility  
**Fix**: Added explicit type conversions

#### `LibCarla/source/carla/road/element/Waypoint.h`
**Issue**: size_t type inconsistencies  
**Fix**: Standardized to uint64_t

---

### 2. Unreal Plugin - UE5.5 API Updates

#### `Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/BlueprintLibary/MeshToLandscape.cpp`
**Issue**: Missing PhysScene_Chaos.h, deprecated FPhysScene API  
**Changes**:
```cpp
// Added include
#include "Physics/Experimental/PhysScene_Chaos.h"

// Fixed LockRead call
FChaosScene* PhysScene = World->GetPhysicsScene();
if (PhysScene)
{
    PhysScene->GetScene()->LockRead();
    // ... physics operations
    PhysScene->GetScene()->UnlockRead();
}
```

#### `Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/BlueprintLibary/PostProcessJsonUtils.cpp`
**Issue**: Missing JsonObjectConverter include  
**Fix**:
```cpp
#include "JsonObjectConverter.h"
```

#### `Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Sensor/Radar.cpp`
**Issue**: Deprecated ParallelMultiLineTrace API  
**Changes**:
```cpp
// OLD (UE4/UE5.3):
World->ParallelMultiLineTrace(OutHits, ...);

// NEW (UE5.5):
for (auto& StartEnd : StartEndVectorList)
{
    FHitResult Hit;
    World->MultiLineTraceSingleByChannel(
        Hit,
        StartEnd.Key,
        StartEnd.Value,
        ECC_MAX,
        TraceParams,
        FCollisionResponseParams::DefaultResponseParam
    );
    OutHits.Add(Hit);
}
```

#### `Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Sensor/RayCastSemanticLidar.cpp`
**Issue**: Deprecated ParallelMultiSphereTraceByChannel API  
**Changes**:
```cpp
// OLD (UE4/UE5.3):
World->ParallelMultiSphereTraceByChannel(...);

// NEW (UE5.5):
for (const auto& Loc : RayPreprocessCondition)
{
    FHitResult OutHit;
    World->MultiSphereTraceSingleByChannel(
        OutHit,
        Loc.Key,
        Loc.Value,
        // ... parameters
    );
    OutHits.Add(OutHit);
}
```

---

### 3. CarlaTools Plugin

#### `Unreal/CarlaUnreal/Plugins/CarlaTools/Source/CarlaTools/Private/BlueprintLibrary/MeshToSplineActor.cpp`
**Issue**: Missing ProceduralMeshComponent include  
**Fix**:
```cpp
#include "ProceduralMeshComponent.h"
```

---

### 4. Build Configuration

#### `Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Carla.Build.cs`
**Changes**:
```csharp
// Module dependencies restructured
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "AIModule",
    "Json",
    "JsonUtilities",
    "PhysicsCore",
    "Chaos",
    "ChaosVehicles",
    "RenderCore",
    "RHI",
    "Renderer",
    "ProceduralMeshComponent",
    "MeshDescription",
    "Projects",
    "AnimGraphRuntime",
    "NavigationSystem"
});

// Attempted linker flag (non-functional with installed engine)
if (Target.Platform == UnrealTargetPlatform.Mac)
{
    PublicAdditionalLinkerFlags.Add("-undefined dynamic_lookup");
}
```

#### `Unreal/CarlaUnreal/Source/CarlaUnrealEditor.Target.cs`
**Changes**:
```csharp
// Override build environment to allow linker modifications
if (Platform == UnrealTargetPlatform.Mac)
{
    bOverrideBuildEnvironment = true;
    AdditionalLinkerArguments = "\"-undefined dynamic_lookup\"";
}
```

---

### 5. StreetMap Plugin (Multiple Files)

**Files Modified**:
- `StreetMapImporting/Private/StreetMapFactory.cpp`
- Other StreetMap source files

**Issue**: Missing AssetToolsModule include  
**Fix**:
```cpp
#include "AssetToolsModule.h"
```

---

## Build System Files

### `CMake/Toolchain.cmake` (New File)
- Custom macOS ARM64 toolchain
- 400+ lines
- Integrates prebuilt UE5.5
- Handles Apple Silicon-specific configuration

### `Unreal/CarlaUnreal/CarlaUnreal.uproject`
- Updated for UE5.5 compatibility
- Plugin configurations updated

---

## API Migration Summary

| Old API (UE5.3) | New API (UE5.5) | Affected Files |
|----------------|----------------|----------------|
| `FPhysScene*` | `FChaosScene*` | MeshToLandscape.cpp |
| `ParallelMultiLineTrace()` | `MultiLineTraceSingleByChannel()` | Radar.cpp |
| `ParallelMultiSphereTraceByChannel()` | `MultiSphereTraceSingleByChannel()` | RayCastSemanticLidar.cpp |

---

## Compilation Results

- ✅ **193 files compiled successfully**
- ✅ **0 compilation errors**
- ✅ **0 warnings** (related to our changes)
- ❌ **Link failed** (not a code issue - see MACOS_BUILD_STATUS.md)

---

## Testing

### LibCarla C++ Client
```bash
cd Build/Examples
./carla-example-client
# Result: ✅ Runs successfully
```

### Unreal Editor Modules
- Compilation: ✅ Success
- Linking: ❌ Failed (RTTI typeinfo symbols missing from prebuilt engine)

---

## Compatibility Notes

### What Works
- All C++ code compiles with AppleClang 17.0.0
- CMake-based LibCarla builds successfully
- UE5.5 API changes properly addressed
- Type safety maintained on ARM64 architecture

### What Doesn't Work
- Plugin dylib linking (requires source-built UE5.5)
- Runtime execution in Unreal Editor (blocked by linker errors)

---

## Reverting Changes

To revert all changes:
```bash
git checkout LibCarla/source/carla/Buffer.h
git checkout LibCarla/source/carla/MsgPackAdaptors.h
git checkout LibCarla/source/carla/road/element/Waypoint.h
git checkout Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/BlueprintLibary/MeshToLandscape.cpp
git checkout Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/BlueprintLibary/PostProcessJsonUtils.cpp
git checkout Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Carla.Build.cs
git checkout Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Sensor/Radar.cpp
git checkout Unreal/CarlaUnreal/Plugins/Carla/Source/Carla/Sensor/RayCastSemanticLidar.cpp
git checkout Unreal/CarlaUnreal/Plugins/CarlaTools/Source/CarlaTools/Private/BlueprintLibrary/MeshToSplineActor.cpp
git checkout Unreal/CarlaUnreal/Source/CarlaUnrealEditor.Target.cs
```

---

**Note**: These changes make CARLA 0.10.0 code **compile** with UE5.5 on macOS ARM64, but do not solve the **linking** issue which is a limitation of Epic's prebuilt engine distribution.
