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

### 4.3. Initialization API
We need a new module to handle the engine startup.

```fsharp
module GodotEngine =
    let Initialize (args: string[]) =
        // Load libgodot
        // Call main entry point
        ()
```

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
