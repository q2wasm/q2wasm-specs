# Quake II WASM
This is a repo designed to educate and explain the goal of this project, as well as how to contribute and create things based on this specification.

# What Is WASM?
WASM is short for WebAssembly, a fairly new technology which has been making the rounds in web circles. It's goal is to provide an extremely simple, safe and effective format for bytecode, primarily aimed at web development. You can read more about WebAssembly here: https://github.com/WebAssembly/design

It may seem confusing to link a web technology to Quake II, as this project has nothing to do with the web. The interesting component of WebAssembly here is the assembly part. There is a lot of existing tooling to both create and consume WebAssembly, and even as an interpreted language, it is fairly fast. One of the main things that drew us to the format is that existing C/C++ code can be compiled down to WebAssembly (via Clang/LLVM 11+). This means that old mods that were written over 20 years ago can be re-compiled to WebAssembly and "just work" on an engine that accepts the new specification. The format is also platform and architecture agnostic: whether you're running a legacy Win32 engine from 1997 or, perhaps, a newer engine re-compiled for x86_64, or maybe even trying to play Quake II portably on an ARM-based platform, the exact same WebAssembly game mod binary will work everywhere!

# The Problem
Quake II modding moving to native compilation caused a lot of ire at the time it was announced, and while at the time it wasn't considered (John Carmack considered it a non-issue since he expected mod authors to release their sources; unfortunately, a lot of authors never considered releasing their sources), new architectures beyond x86 (and mods that have only ever been compiled for Windows x86) create a giant hole in the history of Quake II modding. A lot of people cannot experience these mods, and in some cases these mods were written for such an old dialect of C or for extremely outdated libraries that recompiling them in the now is a hassle.

Developing new mods for Quake II is also a huge cause of shot-foot syndrome. Where does one even start? How does one consider other platforms? It's a problem with no real good solution *other* than developing a new solution to make this process painless, which is what this specification hopes to solve and cause adoption for.

# The Solution
On the archiving front, work has already begun to secure the future of the past of Quake II modding: at https://github.com/Paril/quake2-source-archive, you can browse legacy source code and even re-compile some of them already to WebAssembly binary. The goal of that project is to recompile old sources through a wrapper that will enable them to be loaded on existing engines, as well as newer engines that may implement the q2wasm specification. The wrapper is very early, and currently compiles a native gamex86.dll which can load WebAssembly mods and execute them. This can be viewed here: https://github.com/q2wasm/quake2-wasm

It's considered a proof of concept, and does not implement any of the additional specification that I'm looking to standardize across Quake II engines going forward. In the future, this DLL will be unnecessary to actually execute these mods, and instead older mods will only need to be compiled with a small wrapper that handles the translation from the old native mod interface to the new one.

There will still be a DLL wrapper, but that will solely be to support engines that have not yet implemented the q2wasm specification. The wrapper will allow these engines to load `game.wasm` binaries.

# Specification Goals
The specifications for a q2wasm-supported engine are currently being written. Since we have the *opportunity* to standardize a new API, we also want to reduce friction for engine maintainers. The specifications are being developed with these key points in mind:
* Extensibility: allowing the `game.wasm` binaries to support existing game API modifications that exist in the wild, without requiring major changes or friction on the engine end. Currently, this grouping only includes vanilla and KMQuake2 flavors, since those are the only two real major APIs that are still in use. Going forward, the idea is to enable the WASMs to communicate effectively to the engine as to what is where, and what the engine and/or game needs to be able to properly execute the game code. This also includes a "capability query" interface that both the engine *and* game code implement, and will allow both ends to optionally communicate blocks of data between each other about capabilities it may implement. While q2wasm itself won't use any of these capabilities, other engines can easily extend the game binary by giving it more imports via this system, and game binaries can provide more exports to the engine that it may use.
* Simplicity: existing engines out there are essentially stable and finalized; it would be hard for us to convince engine authors to merge a large swath of changes to support this. In addition, it is paramount that engine authors *do* implement the bare minimum MVP in order for the project to succeed at all. As a result, the MVP for implementation is designed to be *as simple as possible* to implement into existing Quake II engines. That means:
  * No API *changes*! The existing game API must work with no changes. We can't simply take this opportunity to completely erase and re-write the game imports or exports, even if that would benefit creators going forward.
  * Minimal API *additions*. To properly support the **Extensibility** point, there will be *at least* one new import and one new export. These will be the entry points required to support the extensibility concept. Ideally, these will only need to be stubs for the MVP, but we'll get to that point soon.
  * MVP Pull Requests: To take the pressure off of engine authors, we'll be aiming to provide patches, submitted as pull requests, to the major engines out there to support the new system. We will also listen to the feedback of said authors, and address any concerns they may have. Issues should be opened on this repo about the specification.

# The Specification
The specification itself has two parts: WASM and native. The WASM part documents the communication between the native engine and the WASM module. The native part documents the communication between native engine and native game binary. For debugging and development purposes, engines should support both, but in theory they only really need to support the WASM specification if they aren't interested in needing to handle developer debugging.

# Shared concepts
The API version identifier for capability-enabled binaries is `87`.

There are only two new functions being introduced to the legacy game ABI - one import and one export.
The import is: `game_capability_t QueryEngineCapability(const char *)`
The export is: `game_capability_t QueryGameCapability(const char *)`

## Capabilities
The main new concept being introduced with this API is a capability. It is two-way communication to allow both ends to ask the other end what extensions or capabilities it supports. The functions return a `game_capability_t`, which is a platform-dependent type: on WASM, it is a 32-bit integer that will only ever be `0` or `1`. On native, it is either NULL or a function pointer.

Capabilities can *only* be queried at very specific points of the game module negotation: this is to prevent misuse of the concept, and allows both ends to make optimizations related to capability querying as they know the life time of these functions.

For the engine, `QueryGameCapability` may only be called *between* the calls to `GetGameAPI` and the game library's `Init` export. Any calls to `QueryGameCapability` after the call to `Init` will simply return 0 or NULL, and may print a warning in the console for the developer to be made aware of.

For the game library, `QueryEngineCapability` may only be called *during* the call to `GetGameAPI`. Any calls to `QueryEngineCapability` after `GetGameAPI` returns will simply return 0 or NULL, and may print a warning in the console for the developer to be made aware of.

## WASM specification
WASM binaries will not support the legacy game API. The only game API version it should return is `87` (or something built on top of it).

The imports sent from engine to game have some changes in order to simplify the API and improve performance.

Due to varargs being implementation-dependent, any function that took "formatted" values in the Quake II API now take fully-formatted strings (and the `f` removed, to better signify their usage):
- `void bprintf(int32_t, const char *, ...)` -> `void bprint(int32_t, const char *)`
- `void dprintf(const char *, ...)` -> `void dprint(const char *)`
- `void cprintf(edict_t *, int32_t, const char *, ...)` -> `void cprint(edict_t *, int32_t, const char *)`
- `void centerprintf(edict_t *, const char *, ...)` -> `void centerprint(edict_t *, const char *)`
- `void error(const char *, ...)` -> `void error(const char *)`

To speed up certain function calls (since otherwise memory validation has to occur), any imports that used `float[3]` as a parameter now take three separate floats in their place.

- `void positioned_sound(float[3], edict_t *, int32_t, int32_t, float, float, float)` -> `void positioned_sound(float, float, float, edict_t *, int32_t, int32_t, float, float, float)` (note that in this function, passing NULL to the origin had a secondary benefit of causing automated origin calculation, effectively acting like `sound`. Since this is no longer possible with the new prototype, an IEEE 754 infinity value should instead be sent to the `_x` component which will be interpreted as an automated position generation. Otherwise, you may also redirect the operation to `sound` if the intention was to do automated origin calculation.)
- `void trace(float[3], float[3], float[3], float[3], edict_t *, int32_t, trace_t *)` -> `void trace(float, float, float, float, float, float, float, float, float, float, float, float, edict_t *, int32_t, trace_t *)` (note that trace_t writes to a WASM-allocated pointer rather than returning the result. This is due to multi-value not currently being widely emitted by compilers.)
- `int32_t pointcontents(float[3])` -> `int32_t pointcontents(float, float, float)`
- `void WriteDir(float[3])` -> `void WriteDir(float, float, float)`
- `void WritePosition(float[3])` -> `void WritePosition(float, float, float)`
- `void multicast(float[3], qboolean)` -> `void multicast(float, float, float, qboolean)`
- `int32_t BoxEdicts(float[3], float[3], edict_t *, int32_t, int32_t)` -> `int32_t BoxEdicts(float, float, float, float, float, float, edict_t *, int32_t, int32_t)`
- `qboolean inPHS(float[3], float[3])` -> `qboolean inPHS(float, float, float, float, float, float)`
- `qboolean inPVS(float[3], float[3])` -> `qboolean inPVS(float, float, float, float, float, float)`

In addition, six new functions are exported by the game module.
- These functions replace the functionality of the returned `game_export_t` pointer, and are required in order to curb around compiler optimizations + to signify this data back to the engine.
  - `edict_t *GetEdicts(void)` -> returns the current location, in the WASM heap, of the entity table
  - `int32_t *GetNumEdicts(void)` -> returns the current location, in the WASM heap, of the integer holding the number of entities
  - `int32_t GetMaxEdicts(void)` -> returns the maximum number of entities that the game library supports
  - `int32_t GetEdictSize(void)` -> returns the size of the entity structure that is pointed to by GetEdicts
- These functions are required to support PMove. PMove communicated function pointers between the engine and the module, which is not legal in WASM, so instead these exports called in the stead of those function pointers.
  - `void PmoveTrace(pmove_t *pm, float start_x, float start_y, float start_z, float mins_x, float mins_y, float mins_z, float maxs_x, float maxs_y, float maxs_z, float end_x, float end_y, float end_z, trace_t *out)`
  - `int32_t PmovePointContents(pmove_t *pm, float p_x, float p_y, float p_z)`

GetEdicts() must always point to a pointer that contains at least `GetMaxEdicts() * GetEdictSize()` in size.
GetNumEdicts() must always point to a pointer that contains an int32.
For PmoveTrace, the `pmove_t *pm` will point to the original `pmove_t` that was sent to `Pmove`.
For PmoveTrace and PmovePointContents, the pointers will be guaranteed on the engine side to be pointing to valid memory.

For `TagMalloc`, `TagFree` and `FreeTags`: since WASM cannot interact with memory outside of its sandbox, you should instead call the malloc & free functions of your WASM runtime to return and handle memory addresses from the former two functions. Tagging should still occur as necessary.

For functions dealing with `cvar_t`: while you will get a pointer back, this pointer is owned by the native environment and should be synced with your cvar subsystem.

For `trace_t` and the `csurface_t` pointer returned by it: to be compatible with native systems, you should allocate a copy of the valid `csurface_t` elements to be passed over to the WASM runtime. Keep in mind, too, that you will need some way to convert between the native and WASM pointers, since `gi.Pmove` transfers a `trace_t` structure between native and WASM!

For all functions that the WASM expects to import from the engine (including extensions), they must be declared as imports with their name under the module `q2`.

For WASM binaries, the following steps should occur:
- Register your engine exports under the module name `q2`. For instance, for `AddCommandString`, you would export the function `AddCommandString` under module `q2` with the signature `($)`. This registration step should include *all* supported functions by your engine, including extensions.
- Check for an exported function named `_initialize`. This function is exported by WASI-enabled modules in Reactor mode, and is the main method of compiling libraries with the WASI C API for WASM. If it exists, WASI should be initialized with the mod's directory being available to it, and then `_initialize` should be called to initialize WASI. If WASI is not found, a mod may still function, but it may either need a filesystem capability extension or to not support savegames.
- Check for an exported function named `GetGameAPI`. If this function does not exist, raise an error, as this is not a valid Quake II module. The function is declared as: `int32_t GetGameAPI(int32_t apiversion);`. It will take in your API version, and if it decides it is compatible, will return its own API version for you to negotiate with. If the return value is anything else than a supported API version, you should raise an error.
- Query the WASM binary for every function exported by the supplied API version. If any are missing, you should raise an error as they are considered required by the implementation. 

## Native specification
This specification supports enabling a single game DLL to support both the legacy game ABI (version `3`) and the newer, capability-enhanced API. The specification was designed to support both.

### Engine-side
For API version `87`, the following additional structures must be prepared in your headers:

```c
struct game_import_87_t
{
    game_import_t   gi;

    int32_t apiversion;
    
    game_capability_t (*QueryEngineCapability)(const char *cap);
};

struct game_export_87_t
{
  game_export_t ge;
  
  game_capability_t (*QueryGameCapability)(const char *cap);
};
```

As one can see, both of these structures begin with the legacy structure: this is to allow pointers of this new type to be passed to/from one another to work properly for legacy modules.

First, you must negotiate with the game DLL to figure out which level of API it supports. On the engine side, you first call `GetGameAPI` as you normally would to retrieve the exports from the game library. You should pass it a version 3 import structure as you normally would.

If the game library reports a version other than `3`, you should either raise an error or handle as appropriate. A capability-enhanced library will always report a legacy library interface first.

Following this, construct a game_import_87_t to be passed over to the game library. Copy the version 3 import structure into `gi`; assign the `apiversion` value to 87; assign the `QueryEngineCapability` pointer as appropriate; and, finally, set the value of `gi.DebugGraph` to NULL. This is a special step which informs the game library that you are, indeed, passing it a version 87 import structure, and that it may access the extraneous bytes at the end of the structure.

Now, call `GetGameAPI` again with your game_import_87_t pointer. If the native library supports the new API, it will respond with a `game_export_87_t` pointer whose `ge.apiversion` is `87`. If `ge.apiversion` is `3`, however, the API does not support version 87 and is a legacy module.

At this point, restore the value of `gi.DebugGraph` to its prior function pointer (there may be mods out there that call into it, even though it's stubbed on the engine side usually). You've successfully verified that the module either is, or is not, version 87, and can now query the game for additional capabilities using the `QueryGameCapability` function. Note that during this call to `GetGameAPI` with an `apiversion` of `87`, the game library may call into `QueryEngineCapability` to query the engines' own capabilities.
