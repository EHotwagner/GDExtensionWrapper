# F# GDExtension Binding Design Document

## 1. Executive Summary
This document outlines the design for a high-performance, idiomatic F# binding for the Godot Engine using the GDExtension interface. The goal is to allow developers to write Godot game logic, nodes, and plugins entirely in F#, leveraging the language's strong type system, functional paradigms, and .NET ecosystem.

## 2. Architecture Overview

The project consists of two main components:
1.  **The Generator (`GDExtensionWrapper.Generator`)**: A console application that parses `extension_api.json` and emits F# source code.
2.  **The Runtime (`GDExtensionWrapper.Runtime`)**: A library containing the core interop logic, base types, and the generated code.

### Build Flow
1.  Godot Engine -> `extension_api.json`
2.  `extension_api.json` -> **Generator** -> `Generated.fs`
3.  `Generated.fs` + **Runtime Core** -> **F# Compiler** -> `MyGame.dll` (NativeAOT Shared Library)
4.  `MyGame.dll` -> Godot Engine (loaded via `.gdextension` file)

## 3. Technical Approach & Libraries

### 3.1. Interop Strategy
To communicate with Godot's C API, we will use **NativeAOT** (Ahead-of-Time compilation) to produce a native shared library that exports the required C symbols (`godot_gdextension_entry`).

*   **`System.Runtime.InteropServices`**: For `NativeLibrary`, `UnmanagedCallersOnly`, and marshalling.
*   **`FSharp.NativeInterop`**: For low-level pointer arithmetic (`NativePtr`) where performance is critical.
*   **P/Invoke**: We will define function pointer delegates for every GDExtension interface function.

### 3.2. Libraries
*   **Parsing**: `System.Text.Json` for reading `extension_api.json`.
*   **Build**: standard `.fsproj` with `<PublishAot>true</PublishAot>`.

## 4. API Structure & Type Mapping

### 4.1. The `Variant` Type
The `Variant` is the core dynamic type in Godot. In F#, we have two main approaches: a Discriminated Union (DU) or a Struct with Active Patterns.

**Decision**: **Struct with Active Patterns**.
A full DU would require allocating a new .NET object every time a Variant crosses the boundary, which is too slow for a game engine. We will wrap the native C struct and provide idiomatic pattern matching.

```fsharp
[<Struct>]
type Variant =
    val private handle: nativeint // Opaque pointer or struct data
    
    // Active Pattern for idiomatic matching
    static member (|Int|Float|String|Nil|...) (v: Variant) =
        match v.Type with
        | VariantType.Int -> Int (v.AsInt64())
        | VariantType.Float -> Float (v.AsDouble())
        // ...
```

### 4.2. Godot Objects (Nodes, Resources)
Godot objects are reference types. We will map them to F# classes.

*   **Base Class**: `GodotObject`
    *   Holds the `IntPtr` handle to the native object.
    *   Implements `IDisposable` (optional, mostly for RefCounted).
    *   Overrides `GetHashCode` and `Equals` based on the handle.
*   **Inheritance**: The generator will replicate the Godot hierarchy.
    *   `Node` inherits `GodotObject`
    *   `Node3D` inherits `Node`
*   **Methods**: Generated as member methods.
    *   `member this.SetPosition(pos: Vector3) = ...`
    *   Internally calls `godot_icall_...` helpers.

### 4.3. Enums
Mapped directly to F# enums.
```fsharp
type Error =
    | Ok = 0
    | Failed = 1
```

### 4.4. Signals
Mapped to F# `IEvent<_>` or standard .NET `event`.
```fsharp
member this.Ready = new Event<unit>()
```

## 5. Memory Management & Lifecycles

This is the most critical part of the design. We must bridge .NET's Garbage Collector (GC) with Godot's manual/reference-counted memory model.

### 5.1. Object Identity
When Godot passes an `Object*` to F#, we must ensure we return the *same* F# wrapper instance if it already exists.
*   **Cache**: A `Dictionary<IntPtr, WeakReference<GodotObject>>` mapping native pointers to F# wrappers.

### 5.2. Script Instances (Extending Godot Classes)
When a user defines a new class in F# (`type MyPlayer() = inherit Node3D()`), we need to attach this to a Godot object.
1.  **GDExtension Class Creation**: We register the F# type as a GDExtension class.
2.  **Instance Binding**: When Godot instantiates this class, it calls our `create_instance` callback.
3.  **GCHandle**: We allocate a `GCHandle.Alloc(fsharpObj, GCHandleType.Normal)` to prevent the GC from collecting the F# object while Godot is using it.
4.  **Cleanup**: When Godot destroys the object (notification `OBJECT_PREDELETE` or `free_instance`), we `GCHandle.Free()` the handle, allowing the GC to collect the F# object.

### 5.3. RefCounted Objects
For `RefCounted` types (like `Resource`), we must hook into `unreference`.
*   If F# holds a strong reference, the RefCount should be > 0.
*   If F# drops the reference, we decrement.

## 6. Implementation Roadmap

### Phase 1: The Generator Core
1.  Define F# record types representing the `extension_api.json` schema.
2.  Implement JSON parsing.
3.  Implement a basic text emitter (StringBuilder or a templating engine).

### Phase 2: The Runtime Core (NativeAOT)
1.  Create the `GDExtensionWrapper.Runtime` project.
2.  Implement `godot_gdextension_entry` using `[<UnmanagedCallersOnly>]`.
3.  Load the basic function pointers (`get_proc_address`, `print_error`, etc.).

### Phase 3: Binding Generation
1.  **Enums**: Generate `Enums.fs`.
2.  **Builtins**: Generate `Vector3.fs`, `String.fs` (wrapping native calls).
3.  **Classes**: Generate `Classes.fs` with method stubs.

### Phase 4: User Scripting Support
1.  Implement the `ClassDB` registration logic.
2.  Implement the `GCHandle` lifecycle management.
3.  Test with a simple "Hello World" rotating cube.

## 7. Example User Code

```fsharp
namespace MyGame

open Godot

type RotatingCube() =
    inherit Node3D()

    override this._Process(delta) =
        this.RotateY(delta * 2.0)
```
