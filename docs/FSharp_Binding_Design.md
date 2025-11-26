# F# Binding Design for Godot 4

## 1. Executive Summary
This document outlines the architectural design for an F# binding to Godot 4. It addresses two distinct use cases:
1.  **LibGodot Embedding (Host):** The .NET application hosts Godot as a library. This is the user's primary interest but requires a **custom build** or fork (e.g., `migeran/libgodot`) as standard Godot 4 does not yet expose a public C API for embedding on desktop.
2.  **GDExtension (Plugin):** The standard supported method where Godot loads the F# assembly.

## 2. Architecture Options

### Option A: LibGodot Embedding (Host)
*   **Architecture:** `.NET App` -> `NativeLibrary.Load` -> `libgodot.dll`
*   **Status:** **Experimental / Custom Build Required**.
*   **Entry Point:** Requires exposing `Main::setup`, `Main::iteration` via `extern "C"`, or using a fork like `migeran/libgodot` which provides `gdextension_create_godot_instance`.
*   **Pros:** Full control of main loop, easy integration into existing .NET apps.
*   **Cons:** Requires maintaining a custom Godot build; API is not stable.

### Option B: GDExtension (Plugin)
*   **Architecture:** `Godot Executable` -> `GDExtension Interface` -> `F# Assembly`
*   **Status:** **Stable / Supported**.
*   **Entry Point:** The F# assembly exports `godot_gdextension_entry` which Godot calls on startup.
*   **Pros:** Works with standard Godot binaries (Steam/Website); simpler distribution.
*   **Cons:** Godot owns the process lifecycle.

## 3. Implementation Details: LibGodot Embedding

### 3.1. Build Requirements
To use Godot as a library, you must compile it from source:
```bash
# Windows
scons platform=windows target=template_debug library_type=shared_library
```
*Note: You may need to patch `platform/windows/godot_windows.cpp` to export the `main` logic if not using a specialized fork.*

### 3.2. Initialization (C API)
If using a custom build that exposes the `Main` class methods via C:
```fsharp
[<DllImport("godot")>]
extern int godot_init(int argc, string[] argv)

[<DllImport("godot")>]
extern bool godot_step()

[<DllImport("godot")>]
extern void godot_shutdown()
```

### 3.3. Native Library Loading
Since `libgodot` is a native DLL, use `NativeLibrary.Load` with a `DllImportResolver` to handle platform differences (e.g., `.dll` vs `.so`).

```fsharp
module NativeLoader =
    let ImportResolver (libraryName: string) (assembly: Assembly) (searchPath: DllImportSearchPath option) : IntPtr =
        if libraryName = "godot" then
            NativeLibrary.Load("path/to/libgodot.dll")
        else
            IntPtr.Zero
```

## 4. Implementation Details: GDExtension Interface

Regardless of the hosting model, interaction with the engine uses the **GDExtension C API**.

### 4.1. `extension_api.json`
This file is the source of truth. The F# generator must parse it to create:
*   **Classes:** `Node`, `RefCounted`, `Vector3`.
*   **Methods:** `get_position()`, `set_position()`.
*   **Enums:** `Error`, `Key`.

### 4.2. `gdextension_interface.h`
The F# runtime must load function pointers from this interface:
*   `get_proc_address`: Bootstraps all other functions.
*   `variant_new_nil`: Creates Variants.
*   `object_method_bind_ptrcall`: Fast method invocation.

### 4.3. Type Marshaling
*   **F# `int`** -> `GDExtensionInt` (int64)
*   **F# `float`** -> `GDExtensionFloat` (double)
*   **F# `string`** -> `GDExtensionStringPtr` (pointer to Godot String)
*   **F# Objects** -> `GDExtensionObjectPtr` (opaque pointer)

## 5. F# Specifics

### 5.1. Memory Management
*   **Ref<T>:** Use `IDisposable` to decrement reference counts for `RefCounted` types.
*   **Godot Objects:** For non-ref-counted objects (Nodes), be careful not to free them if Godot owns them.

### 5.2. Generator Strategy
1.  **Parse** `extension_api.json`.
2.  **Generate** F# records/classes for Godot types.
3.  **Emit** P/Invoke signatures for `gdextension_interface.h`.

## 6. Next Steps
1.  **Decide on Build:** Stick with standard GDExtension (Option B) for initial development, or commit to building a custom `libgodot` (Option A).
2.  **Generate Bindings:** Run the generator against `extension_api.json`.
3.  **Hello World:** Create a simple F# script that prints to the Godot console.
