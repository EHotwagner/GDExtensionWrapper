# F# GDExtension Binding Design Document (Embedded / libgodot)

## 1. Executive Summary
This document outlines the design for an F# binding that allows **embedding Godot 4 as a library** (`libgodot`) into a standalone .NET application.
Unlike the standard GDExtension approach (where Godot loads a DLL), here the **F# application is the host** and loads the Godot engine as a dependency. This allows for standard .NET (JIT) usage, rich debugging, and full control over the application lifecycle.

## 2. Architecture Overview

The project consists of:
1.  **The Generator (`GDExtensionWrapper.Generator`)**: Parses `extension_api.json` and emits F# source code.
2.  **The Runtime (`GDExtensionWrapper.Runtime`)**:
    *   Loads `libgodot` (shared library).
    *   Initializes the engine via the C API.
    *   Provides the generated API to the user.

### Execution Flow
1.  **F# App Start**: `dotnet run` -> `Program.fs`.
2.  **Load Engine**: `NativeLibrary.Load("godot")`.
3.  **Init Engine**: Call `godot_main` (or equivalent initialization sequence).
4.  **Game Loop**: The F# app drives the loop or yields control to Godot.
5.  **Interaction**: F# code uses generated bindings to manipulate the SceneTree.

## 3. Technical Approach & Libraries

### 3.1. Interop Strategy
Since we are the host, we do **not** need NativeAOT. We can use standard .NET 8+.

*   **`System.Runtime.InteropServices`**:
    *   `NativeLibrary.Load`: To dynamically load the custom-built `libgodot`.
    *   `[<DllImport>]` / Function Pointers: To call into the GDExtension Interface.
*   **`extension_api.json`**: Still the source of truth. We generate bindings against the API exposed by `libgodot`.

### 3.2. Libraries
*   **Parsing**: `System.Text.Json`.
*   **Host App**: Standard F# Console App (`.fsproj`).

## 4. API Structure & Type Mapping

### 4.1. The `Variant` Type
(Same as before: Struct with Active Patterns)
```fsharp
[<Struct>]
type Variant =
    val private handle: nativeint
    member this.Type = ... 

[<AutoOpen>]
module VariantPatterns =
    let (|Int|Float|String|Nil|Other|) (v: Variant) = ...
```

### 4.2. Godot Objects
(Same as before: Wrapper classes holding pointers)

### 4.3. Initialization API (Migeran libgodot)
The `migeran/libgodot` fork exposes a specific C API for initialization.

**C Signature:**
```c
GDExtensionObjectPtr gdextension_create_godot_instance(int p_argc, char *p_argv[], GDExtensionInitializationFunction p_init_func);
```

**F# Binding:**
```fsharp
module GodotEngine =
    [<DllImport("godot", CallingConvention = CallingConvention.Cdecl)>]
    extern nativeint gdextension_create_godot_instance(int argc, string[] argv, IntPtr init_func)

    let Initialize (args: string[]) =
        // 1. Define our GDExtension initialization callback
        let initCallback = ... 
        
        // 2. Create the instance
        let instanceHandle = gdextension_create_godot_instance(args.Length, args, initCallback)
        
        // 3. Wrap the returned handle in a GodotInstance object
        let instance = new GodotInstance(instanceHandle)
        
        // 4. Start the engine
        instance.Start()
        instance
```

**The `GodotInstance` Class**:
This specific build of Godot adds a `GodotInstance` class to the API. We must ensure we generate bindings for it.
*   **Methods**: `start()`, `iteration()`, `shutdown()`, `is_started()`.
*   **Usage**: The host application calls `instance.Iteration()` in its own loop to drive the engine.

### 4.4. API Source (`extension_api.json`)
**Crucial**: We must generate `extension_api.json` from the **libgodot build itself**, not a standard Godot build.
*   The `GodotInstance` class and `DisplayServerEmbedded` classes are only present in this custom build.
*   Command: `./godot_server --dump-extension-api` (or similar, depending on the build artifact).

## 5. Memory Management & Lifecycles

### 5.1. Object Identity
We still need to map `Object*` to F# wrappers.
*   **Cache**: `Dictionary<IntPtr, WeakReference<GodotObject>>`.

### 5.2. Threading
Godot is not thread-safe. All API calls must happen on the thread that initialized the engine (usually the main thread).
*   **SynchronizationContext**: We might need to install a custom context to marshal async/await calls back to the main loop.

## 6. Implementation Roadmap

### Phase 1: The Generator Core
1.  Define F# record types representing the `extension_api.json` schema.
2.  Implement JSON parsing.

### Phase 2: The Runtime Core (Embedding)
1.  **Build libgodot**: Compile Godot from source as a shared library.
2.  **Loader**: Implement `NativeLibrary.Load` logic.
3.  **Init**: Implement the C# -> C initialization call.

### Phase 3: Binding Generation
1.  **Enums**: Generate `Enums.fs`.
2.  **Builtins**: Generate `Vector3.fs`, `String.fs`.
3.  **Classes**: Generate `Classes.fs`.

### Phase 4: User Scripting Support
1.  Test with a simple "Hello World" that initializes Godot and prints to the console using `GD.Print`.

## 7. Example User Code

```fsharp
open Godot

[<EntryPoint>]
let main args =
    // Start Godot
    GodotEngine.Initialize(args)
    
    // Use Godot API
    let node = new Node3D()
    node.SetPosition(Vector3(1.0, 2.0, 3.0))
    GD.Print(node.GetPosition())
    
    0
```
