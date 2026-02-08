# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

IW4x is a C++ DLL mod for Call of Duty: Modern Warfare 2 (2009) that hooks into the game executable to extend its functionality. It provides dedicated server support, modding capabilities, and enhanced multiplayer features. The output is `iw4x.dll` which loads into the game process.

## Build Commands

```bash
# Generate Visual Studio 2022 solution (also initializes git submodules)
generate.bat

# Build from command line
msbuild /m /p:Configuration=Release /p:Platform=Win32 build/iw4x.sln
msbuild /m /p:Configuration=Debug /p:Platform=Win32 build/iw4x.sln

# Premake options
tools/premake5 vs2022 --copy-to=<MW2_PATH>    # Copy DLL to game dir after build
tools/premake5 vs2022 --copy-pdb               # Also copy PDB files
tools/premake5 vs2022 --disable-binary-check    # Skip exe integrity checks
```

Build produces `build/bin/Win32/<Config>/iw4x.dll`. Target platform is Win32 (x86) only. Language standard is C++20.

## Architecture

### Entry Point

`DllMain.cpp` hooks the game's C runtime startup (address `0x6BAC0F`) to run `Main::EntryPoint()`, which initializes all IW4x systems before the game starts. An optional binary integrity check validates the game executable.

### Component System

All major features are implemented as components extending `Components::Component` (defined in `src/Components/Loader.hpp`). Components are registered in a specific order in `Loader::Initialize()` (`src/Components/Loader.cpp`) — registration order matters because some components depend on others being initialized first.

Retrieve a component instance with `Loader::GetInstance<T>()`. There are ~80 component modules in `src/Components/Modules/`.

The first group registered (Singleton, Auth, Command, Dvar, Exception, Network, Logger, etc.) are high-priority foundation components. ConfigStrings must come before ModelCache and Weapons.

### Function Hooking

`Utils::Hook` provides inline function patching to intercept and extend native game functions. This is the core mechanism for modifying game behavior. Game function pointers are declared in `src/Game/Functions.hpp`.

### Game Integration Layer (`src/Game/`)

- **Structs.hpp**: Complete game data structure definitions (~266KB). Contains asset type enums, entity definitions, engine structures.
- **Functions.hpp/cpp**: Declarations and addresses of original game functions.
- **Dvars.hpp/cpp**: Dynamic variable (configuration) system.
- **Script.hpp/cpp**: Game script execution system.
- **Engine/**: Memory management (Hunk allocator), threading primitives (FastCriticalSection, ScopedCriticalSection).
- **Scripting/**: GSC function wrappers and stack isolation.

### Scheduler System

`src/Components/Modules/Scheduler.hpp` provides timed and pipeline-based callback execution across multiple pipelines: ASYNC, RENDERER, SERVER, CLIENT, MAIN, QUIT. Supports once/loop/OnGameInitialized patterns.

### Asset System

43 asset types defined in `Game::XAssetType` enum. `AssetHandler` manages loading/saving/relocating. Individual asset type handlers are in `src/Components/Modules/AssetInterfaces/`.

### Protocol Buffers

`.proto` files in `src/Proto/` define serialization formats for auth, friends, nodes, parties, sessions, IPC, and RCon. Compiled via `tools/protoc.exe` during build.

### Dependencies

All external dependencies are git submodules in `deps/`, configured via Lua scripts in `deps/premake/*.lua`. Key dependencies: zlib, libtomcrypt, protobuf, mongoose (HTTP), discord-rpc, rapidjson, udis86 (x86 disassembler), pdcurses, DirectX 9 SDK, GSL.

## Code Style

From CODESTYLE.md and .editorconfig:

- **Indentation**: Tabs, not spaces
- **Line endings**: CRLF (Windows)
- **Braces**: Allman style (opening brace on its own line)
- **Naming**: `camelCase` for instance members/methods, `PascalCase` for static members/functions, classes, and types
- **Pre-compiled header**: `STDInclude.hpp` is force-included in all translation units — do not add `#include "STDInclude.hpp"` manually
- Structure validation macros: `AssertSize()`, `AssertOffset()` used throughout `Structs.hpp`
