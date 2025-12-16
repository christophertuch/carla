# CARLA 0.10.0 - macOS Build Attempt

⚠️ **Status: Build Blocked** - Cannot link with Epic's prebuilt UE5.5 on macOS

## Quick Summary

This repository contains a **partially successful** attempt to build CARLA 0.10.0 on Apple Silicon (M1) with macOS 26.0 and Unreal Engine 5.5.

### What Works ✅
- C++ client/server libraries built successfully
- All 193 source files compile without errors  
- Example client runs correctly

### What Doesn't Work ❌
- Plugin linking fails due to missing RTTI typeinfo symbols in prebuilt UE5.5
- Unreal Editor cannot load CARLA modules

## Documentation

📄 **[MACOS_BUILD_STATUS.md](MACOS_BUILD_STATUS.md)** - Complete build report with technical details  
📝 **[CHANGES.md](CHANGES.md)** - List of all code modifications for UE5.5 compatibility

## Quick Start

### Prerequisites
```bash
# Install Xcode 26.0.1
# Install UE5.5 from Epic Games Launcher
# Install CMake, Ninja, Python 3.14
```

### Build LibCarla
```bash
cd Build
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release ..
ninja
```

### Test Example Client
```bash
cd Build/Examples
./carla-example-client
```

## Repository Contents

```
Build/
├── LibCarla/
│   ├── libcarla-client.a (56 MB) ✅
│   └── libcarla-server.a (17 MB) ✅
└── Examples/
    └── carla-example-client (23 MB) ✅

Unreal/CarlaUnreal/
├── Content/Carla/ (80 GB) ✅
└── Plugins/ (Modified for UE5.5) ✅
```

## Why It Doesn't Fully Work

Epic's prebuilt UE5.5 for macOS doesn't export RTTI typeinfo symbols that CARLA plugins need during linking. This is a fundamental limitation of the "installed" engine distribution.

### Solutions
1. **Build UE5.5 from source** (~2 weeks compilation)
2. **Use Linux** (officially supported)
3. **Wait for official macOS support**

## Environment

- **Platform**: M1 MacBook, macOS 26.0
- **Compiler**: AppleClang 17.0.0, Xcode 26.0.1
- **Engine**: UE5.5 (prebuilt from Epic Games Launcher)
- **CARLA**: 0.10.0 (ue5-dev branch)

## Modified Files

All modifications are documented in [CHANGES.md](CHANGES.md):
- 11 source files updated for UE5.5 API compatibility
- 2 build configuration files modified
- All changes successfully compile

## License

CARLA is licensed under MIT License. See [LICENSE](LICENSE) for details.

## Resources

- [CARLA Documentation](https://carla.readthedocs.io/)
- [CARLA GitHub](https://github.com/carla-simulator/carla)
- [UE5 Build Guide](https://docs.unrealengine.com/5.5/en-US/building-unreal-engine-from-source/)

---

**Last Updated**: December 16, 2025
