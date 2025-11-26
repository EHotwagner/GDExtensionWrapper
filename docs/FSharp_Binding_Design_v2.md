# F# GDExtension Binding Design Document v2

## 1. Executive Summary

This document outlines a comprehensive design for F# bindings to Godot 4 using the GDExtension system. The design draws from extensive research of existing implementations (godot-cpp, godot-rust/gdext) and adapts their patterns to idiomatic F#.

### Key Design Principles
1. **Idiomatic F#**: Leverage F# features like discriminated unions, computation expressions, and type providers
2. **Safety First**: Compile-time guarantees where possible, runtime checks where necessary
3. **Zero-Cost Abstractions**: Minimize overhead through proper use of structs and inline functions
4. **Incremental Adoption**: Support both GDExtension plugin mode and experimental libgodot embedding

---

## 2. Architecture Overview

### 2.1 Project Structure

```
GDExtensionWrapper/
├── src/
│   ├── GDExtensionWrapper.Generator/     # Code generator (build-time tool)
│   │   ├── JsonParser.fs                 # extension_api.json parser
│   │   ├── DomainModels.fs               # Internal representation
│   │   ├── CodeEmitter.fs                # F# code generation
│   │   └── Program.fs                    # Generator entry point
│   │
│   ├── GDExtensionWrapper.Ffi/           # Low-level FFI layer
│   │   ├── GdExtensionInterface.fs       # Function pointer table
│   │   ├── MethodTables.fs               # Builtin/Class method bindings
│   │   ├── StringCache.fs                # StringName interning
│   │   └── Types.fs                      # Core FFI types
│   │
│   ├── GDExtensionWrapper.Core/          # Generated bindings + runtime
│   │   ├── Generated/                    # Auto-generated from extension_api.json
│   │   │   ├── Builtins/                 # Vector3, String, Array, etc.
│   │   │   ├── Classes/                  # Node, Object, Resource, etc.
│   │   │   ├── Enums/                    # Error, Key, PropertyHint, etc.
│   │   │   └── UtilityFunctions.fs       # print, sin, cos, etc.
│   │   ├── Registry/                     # Class registration system
│   │   ├── Meta/                         # Type metadata, GodotConvert
│   │   └── Gd.fs                         # Main Gd<'T> smart pointer
│   │
│   └── GDExtensionWrapper.Macros/        # Compile-time metaprogramming (optional)
│       └── GodotClass.fs                 # [<GodotClass>] attribute processing
│
├── extension_api.json                    # Godot API definition
└── gdextension_interface.h               # C interface header (reference)
```

### 2.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BUILD TIME                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  extension_api.json ──► Generator ──► Generated/*.fs                        │
│                              │                                               │
│                              └──► MethodTables.fs (indices, hashes)          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                              RUNTIME                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Godot ──► godot_gdextension_entry() ──► GDExtensionBinding.init()          │
│                                              │                               │
│                     ┌────────────────────────┼───────────────────────┐       │
│                     ▼                        ▼                       ▼       │
│           Load Interface FPtrs      Load Method Tables      Register Classes │
│                     │                        │                       │       │
│                     └────────────────────────┴───────────────────────┘       │
│                                              │                               │
│                                              ▼                               │
│                                      User Code Runs                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. FFI Layer Design

### 3.1 Interface Function Pointers

The GDExtension interface provides ~200+ function pointers. These are loaded at initialization via `get_proc_address`.

```fsharp
// GdExtensionInterface.fs
module GdExtensionInterface

open System
open System.Runtime.InteropServices

// Core types (matching gdextension_interface.h)
type GDExtensionBool = byte
type GDExtensionInt = int64
type GDExtensionObjectPtr = nativeint
type GDExtensionTypePtr = nativeint
type GDExtensionMethodBindPtr = nativeint
type GDExtensionClassLibraryPtr = nativeint

// Function pointer delegates
type GetProcAddress = delegate of string -> nativeint

[<Struct>]
type GDExtensionInterface = {
    // Variant operations
    mutable variant_new_nil: nativeint
    mutable variant_new_copy: nativeint
    mutable variant_destroy: nativeint
    mutable variant_call: nativeint
    mutable variant_get_type: nativeint
    
    // String operations
    mutable string_new_with_utf8_chars: nativeint
    mutable string_to_utf8_chars: nativeint
    
    // Object operations
    mutable object_method_bind_call: nativeint
    mutable object_method_bind_ptrcall: nativeint
    mutable object_set_instance: nativeint
    mutable object_get_instance_binding: nativeint
    
    // ClassDB operations
    mutable classdb_construct_object2: nativeint
    mutable classdb_get_method_bind: nativeint
    mutable classdb_register_extension_class4: nativeint
    mutable classdb_register_extension_class_method: nativeint
    mutable classdb_register_extension_class_property: nativeint
    mutable classdb_register_extension_class_signal: nativeint
    
    // And ~180 more...
}

/// Global interface instance (initialized once at startup)
let mutable private _interface: GDExtensionInterface = Unchecked.defaultof<_>
let mutable private _library: GDExtensionClassLibraryPtr = 0n

/// Load all function pointers from Godot
let initialize (getProcAddr: GetProcAddress) (library: GDExtensionClassLibraryPtr) =
    _library <- library
    
    let load name =
        let ptr = getProcAddr.Invoke(name)
        if ptr = 0n then failwithf "Failed to load: %s" name
        ptr
    
    _interface <- {
        variant_new_nil = load "variant_new_nil"
        variant_new_copy = load "variant_new_copy"
        variant_destroy = load "variant_destroy"
        // ... load all function pointers
        classdb_get_method_bind = load "classdb_get_method_bind"
        classdb_register_extension_class4 = load "classdb_register_extension_class4"
        // etc.
    }

/// Access the loaded interface
let inline interface_fn () = _interface
let inline get_library () = _library
```

### 3.2 Method Tables (Lazy Loading Pattern from godot-rust)

Following godot-rust's pattern, method bindings are loaded lazily to reduce startup time:

```fsharp
// MethodTables.fs
module MethodTables

open System.Collections.Concurrent
open System.Runtime.CompilerServices

/// Key for looking up class methods
[<Struct>]
type ClassMethodKey = {
    ClassName: string
    MethodName: string
    Hash: int64
}

/// Lazily-loaded method table
type ClassMethodTable() =
    let cache = ConcurrentDictionary<ClassMethodKey, nativeint>()
    
    member _.GetMethodBind(key: ClassMethodKey) =
        cache.GetOrAdd(key, fun k ->
            // Call into Godot to get the method bind
            let classNamePtr = StringCache.getOrCreate k.ClassName
            let methodNamePtr = StringCache.getOrCreate k.MethodName
            GdExtensionInterface.classdb_get_method_bind(classNamePtr, methodNamePtr, k.Hash)
        )

/// Global method table instances (one per API level for hot-reload support)
let coreMethodTable = ClassMethodTable()
let sceneMethodTable = ClassMethodTable()
let editorMethodTable = ClassMethodTable()
```

### 3.3 String Interning (Critical for Performance)

StringName interning is essential - godot-cpp and godot-rust both do this:

```fsharp
// StringCache.fs
module StringCache

open System.Collections.Concurrent
open System.Runtime.InteropServices

/// Cached StringName pointers
let private cache = ConcurrentDictionary<string, nativeint>()

/// Get or create a StringName pointer
let getOrCreate (s: string) : nativeint =
    cache.GetOrAdd(s, fun str ->
        // Allocate a StringName in Godot's memory
        let ptr = Marshal.AllocHGlobal(24) // StringName size
        use utf8 = System.Text.Encoding.UTF8.GetBytes(str)
        fixed pBytes = utf8 do
            GdExtensionInterface.string_name_new_with_utf8_chars(ptr, pBytes)
        ptr
    )

/// Cleanup on shutdown
let clear () =
    for kvp in cache do
        GdExtensionInterface.string_name_destroy(kvp.Value)
        Marshal.FreeHGlobal(kvp.Value)
    cache.Clear()
```

---

## 4. Type System Design

### 4.1 Builtin Types (Value Types)

Builtin types like Vector3, Color, Transform3D are stack-allocated:

```fsharp
// Generated/Builtins/Vector3.fs
namespace GDExtensionWrapper.Builtins

open System.Runtime.InteropServices
open System.Runtime.CompilerServices

/// Godot Vector3 - a 3D vector with float32 components
[<Struct; StructLayout(LayoutKind.Sequential)>]
type Vector3 =
    val mutable x: float32
    val mutable y: float32
    val mutable z: float32
    
    new(x, y, z) = { x = x; y = y; z = z }
    
    // Constants
    static member Zero = Vector3(0f, 0f, 0f)
    static member One = Vector3(1f, 1f, 1f)
    static member Up = Vector3(0f, 1f, 0f)
    static member Forward = Vector3(0f, 0f, -1f)
    
    // Operators (inline for performance)
    static member inline (+) (a: Vector3, b: Vector3) =
        Vector3(a.x + b.x, a.y + b.y, a.z + b.z)
    
    static member inline (-) (a: Vector3, b: Vector3) =
        Vector3(a.x - b.x, a.y - b.y, a.z - b.z)
    
    static member inline (*) (a: Vector3, s: float32) =
        Vector3(a.x * s, a.y * s, a.z * s)
    
    // Methods that call into Godot
    member this.Normalized() =
        let mutable result = Vector3.Zero
        let mutable self = this
        BuiltinMethodTable.vector3_normalized(&self, &result)
        result
    
    member this.Dot(other: Vector3) =
        let mutable self = this
        let mutable other = other
        BuiltinMethodTable.vector3_dot(&self, &other)
    
    member this.Cross(other: Vector3) =
        let mutable result = Vector3.Zero
        let mutable self = this
        let mutable other = other
        BuiltinMethodTable.vector3_cross(&self, &other, &result)
        result
    
    member this.Length() =
        let mutable self = this
        BuiltinMethodTable.vector3_length(&self)

/// Extension methods for F# idioms
[<AutoOpen>]
module Vector3Extensions =
    type Vector3 with
        member inline this.WithX(x) = Vector3(x, this.y, this.z)
        member inline this.WithY(y) = Vector3(this.x, y, this.z)
        member inline this.WithZ(z) = Vector3(this.x, this.y, z)
```

### 4.2 Godot String (Opaque Handle)

Godot's String type requires special handling:

```fsharp
// Generated/Builtins/GodotString.fs
namespace GDExtensionWrapper.Builtins

open System
open System.Runtime.InteropServices

/// Godot String - an opaque wrapper around Godot's internal String type
[<Struct; StructLayout(LayoutKind.Sequential)>]
type GodotString =
    // Opaque storage matching Godot's String size
    [<MarshalAs(UnmanagedType.ByValArray, SizeConst = 8)>]
    val mutable private opaque: byte[]
    
    /// Create from .NET string
    static member FromString(s: string) =
        let mutable gs = Unchecked.defaultof<GodotString>
        use bytes = System.Text.Encoding.UTF8.GetBytes(s)
        fixed pBytes = bytes do
            fixed pGs = &gs.opaque.[0] do
                GdExtensionInterface.string_new_with_utf8_chars(pGs, pBytes, bytes.Length)
        gs
    
    /// Convert to .NET string
    member this.ToString() =
        fixed pGs = &this.opaque.[0] do
            let len = GdExtensionInterface.string_to_utf8_chars(pGs, 0n, 0)
            let buffer = Array.zeroCreate<byte> (int len)
            fixed pBuf = buffer do
                GdExtensionInterface.string_to_utf8_chars(pGs, pBuf, len) |> ignore
            System.Text.Encoding.UTF8.GetString(buffer)
    
    interface IDisposable with
        member this.Dispose() =
            fixed pGs = &this.opaque.[0] do
                GdExtensionInterface.string_destroy(pGs)

/// Implicit conversions
[<AutoOpen>]
module GodotStringConversions =
    let inline godotString (s: string) = GodotString.FromString(s)
```

### 4.3 Object Types (Reference Semantics with Gd<'T>)

Following godot-rust's `Gd<T>` pattern for type-safe object handles:

```fsharp
// Gd.fs
namespace GDExtensionWrapper

open System
open System.Runtime.CompilerServices

/// Type-safe smart pointer to a Godot object
/// Similar to godot-rust's Gd<T>
[<Struct; NoEquality; NoComparison>]
type Gd<'T when 'T :> GodotObject> =
    val internal ptr: nativeint
    
    internal new(ptr: nativeint) = { ptr = ptr }
    
    /// Check if the pointer is valid
    member this.IsValid = this.ptr <> 0n
    
    /// Get the raw pointer (use with caution)
    member this.RawPtr = this.ptr
    
    /// Upcast to a base type
    member this.Upcast<'U when 'U :> GodotObject and 'T :> 'U>() : Gd<'U> =
        Gd<'U>(this.ptr)
    
    /// Try to downcast to a derived type
    member this.TryDowncast<'U when 'U :> 'T>() : Gd<'U> option =
        if this.ptr = 0n then None
        else
            // Check if the actual type is compatible
            let actualClass = GdExtensionInterface.object_get_class_name(this.ptr)
            if ClassRegistry.isSubclassOf<'U>(actualClass) then
                Some(Gd<'U>(this.ptr))
            else
                None

/// Smart pointer for RefCounted objects (automatic reference counting)
[<Struct; NoEquality; NoComparison>]
type GdRef<'T when 'T :> RefCounted> =
    val internal ptr: nativeint
    
    internal new(ptr: nativeint) = 
        if ptr <> 0n then
            RefCounted.reference(ptr) |> ignore
        { ptr = ptr }
    
    member this.IsValid = this.ptr <> 0n
    
    interface IDisposable with
        member this.Dispose() =
            if this.ptr <> 0n then
                if RefCounted.unreference(this.ptr) then
                    // Reference count hit zero, Godot will free it
                    ()

/// Base class for all Godot objects
[<AbstractClass>]
type GodotObject() =
    static member internal ClassName = "Object"

/// Base class for reference-counted objects  
[<AbstractClass>]
type RefCounted() =
    inherit GodotObject()
    static member internal ClassName = "RefCounted"
    
    static member internal reference(ptr: nativeint) : bool =
        // Call RefCounted::reference() via Godot API
        let methodBind = MethodTables.coreMethodTable.GetMethodBind({
            ClassName = "RefCounted"
            MethodName = "reference"
            Hash = 2240911060L
        })
        let mutable result = 0uy
        GdExtensionInterface.object_method_bind_ptrcall(methodBind, ptr, 0n, &result)
        result <> 0uy
    
    static member internal unreference(ptr: nativeint) : bool =
        let methodBind = MethodTables.coreMethodTable.GetMethodBind({
            ClassName = "RefCounted"
            MethodName = "unreference"
            Hash = 2240911060L
        })
        let mutable result = 0uy
        GdExtensionInterface.object_method_bind_ptrcall(methodBind, ptr, 0n, &result)
        result <> 0uy
```

### 4.4 Generated Engine Classes

```fsharp
// Generated/Classes/Node3D.fs
namespace GDExtensionWrapper.Classes

open GDExtensionWrapper
open GDExtensionWrapper.Builtins

/// Node3D provides transformation in 3D space
type Node3D internal (ptr: nativeint) =
    inherit Node(ptr)
    
    static member internal ClassName = "Node3D"
    
    // Property: position
    member this.Position
        with get() =
            let methodBind = MethodTables.sceneMethodTable.GetMethodBind({
                ClassName = "Node3D"
                MethodName = "get_position"
                Hash = 3360562783L
            })
            let mutable result = Vector3.Zero
            GdExtensionInterface.object_method_bind_ptrcall(methodBind, this.ptr, 0n, &result)
            result
        and set(value: Vector3) =
            let methodBind = MethodTables.sceneMethodTable.GetMethodBind({
                ClassName = "Node3D"
                MethodName = "set_position"
                Hash = 3460891852L
            })
            let mutable value = value
            let args = [| &value |> box :?> nativeint |]
            fixed pArgs = args do
                GdExtensionInterface.object_method_bind_ptrcall(methodBind, this.ptr, pArgs, 0n)
    
    // Property: rotation (in radians)
    member this.Rotation
        with get() = this.GetRotation()
        and set(value) = this.SetRotation(value)
    
    // Method: look_at
    member this.LookAt(target: Vector3, ?up: Vector3) =
        let up = defaultArg up Vector3.Up
        let methodBind = MethodTables.sceneMethodTable.GetMethodBind({
            ClassName = "Node3D"
            MethodName = "look_at"
            Hash = 3123400617L
        })
        let mutable target = target
        let mutable up = up
        let args = [| nativeint &&target; nativeint &&up |]
        fixed pArgs = args do
            GdExtensionInterface.object_method_bind_ptrcall(methodBind, this.ptr, pArgs, 0n)
    
    // Virtual method stub (overridable in user classes)
    abstract member _Process: delta: float -> unit
    default _.Process(_) = ()
```

---

## 5. Class Registration System

### 5.1 User-Defined Classes

Following godot-rust's attribute-based approach:

```fsharp
// Example user class
namespace MyGame

open GDExtensionWrapper
open GDExtensionWrapper.Classes
open GDExtensionWrapper.Builtins

/// Custom player class - registered with Godot
[<GodotClass(Base = "CharacterBody3D")>]
type Player() =
    inherit CharacterBody3D()
    
    // Exported property (visible in Inspector)
    [<Export>]
    member val Speed = 5.0f with get, set
    
    // Signal declaration
    [<Signal>]
    member val HealthChanged: Signal<int> = Signal()
    
    // Virtual method override
    override this._PhysicsProcess(delta: float) =
        let input = Input.Singleton
        let direction = Vector3(
            input.GetAxis("move_left", "move_right"),
            0f,
            input.GetAxis("move_forward", "move_back")
        ).Normalized()
        
        this.Velocity <- direction * this.Speed
        this.MoveAndSlide() |> ignore
    
    // Custom method exposed to Godot
    [<Func>]
    member this.TakeDamage(amount: int) =
        this.HealthChanged.Emit(this.Health - amount)
```

### 5.2 Registration Machinery

```fsharp
// Registry/ClassRegistry.fs
module ClassRegistry

open System
open System.Reflection
open System.Runtime.InteropServices

/// Information about a registered class
type ClassRegistrationInfo = {
    ClassName: string
    ParentClassName: string
    CreateInstance: unit -> nativeint
    FreeInstance: nativeint -> unit
    Methods: MethodRegistration list
    Properties: PropertyRegistration list
    Signals: SignalRegistration list
}

/// Register a class with Godot
let registerClass<'T when 'T :> GodotObject> () =
    let ty = typeof<'T>
    let attr = ty.GetCustomAttribute<GodotClassAttribute>()
    let className = ty.Name
    let parentName = attr.Base
    
    // Create the class creation info struct
    let mutable creationInfo = {
        is_virtual = 0uy
        is_abstract = 0uy
        is_exposed = 1uy
        set_func = Marshal.GetFunctionPointerForDelegate(SetPropertyCallback)
        get_func = Marshal.GetFunctionPointerForDelegate(GetPropertyCallback)
        notification_func = Marshal.GetFunctionPointerForDelegate(NotificationCallback)
        create_instance_func = Marshal.GetFunctionPointerForDelegate(CreateInstanceCallback<'T>)
        free_instance_func = Marshal.GetFunctionPointerForDelegate(FreeInstanceCallback<'T>)
        // ... more callbacks
    }
    
    let classNamePtr = StringCache.getOrCreate className
    let parentNamePtr = StringCache.getOrCreate parentName
    
    GdExtensionInterface.classdb_register_extension_class4(
        GdExtensionInterface.get_library(),
        classNamePtr,
        parentNamePtr,
        &creationInfo
    )
    
    // Register methods
    for method in getExportedMethods<'T>() do
        registerMethod className method
    
    // Register properties
    for prop in getExportedProperties<'T>() do
        registerProperty className prop
    
    // Register signals
    for signal in getSignals<'T>() do
        registerSignal className signal
```

---

## 6. Variant System

### 6.1 F# Discriminated Union for Variant

```fsharp
// Meta/Variant.fs
namespace GDExtensionWrapper.Meta

open System
open System.Runtime.InteropServices
open GDExtensionWrapper.Builtins

/// F# representation of Godot's Variant type
[<Struct; StructLayout(LayoutKind.Sequential)>]
type Variant =
    // Opaque storage (24 bytes on 64-bit)
    [<MarshalAs(UnmanagedType.ByValArray, SizeConst = 24)>]
    val mutable private opaque: byte[]
    
    /// Get the variant type
    member this.Type: VariantType =
        fixed p = &this.opaque.[0] do
            GdExtensionInterface.variant_get_type(p) |> enum

/// Active patterns for type-safe variant matching
[<AutoOpen>]
module VariantPatterns =
    let (|VarNil|VarBool|VarInt|VarFloat|VarString|VarVector3|VarObject|VarOther|) (v: Variant) =
        match v.Type with
        | VariantType.Nil -> VarNil
        | VariantType.Bool -> VarBool(v.AsBool())
        | VariantType.Int -> VarInt(v.AsInt())
        | VariantType.Float -> VarFloat(v.AsFloat())
        | VariantType.String -> VarString(v.AsString())
        | VariantType.Vector3 -> VarVector3(v.AsVector3())
        | VariantType.Object -> VarObject(v.AsObject())
        | _ -> VarOther v

/// GodotConvert trait for type conversions
type IGodotConvert<'T> =
    abstract member ToVariant: 'T -> Variant
    abstract member FromVariant: Variant -> 'T

/// Built-in conversions
module GodotConvert =
    let inline toVariant<'T when 'T :> IGodotConvert<'T>> (value: 'T) =
        (Unchecked.defaultof<'T> :> IGodotConvert<'T>).ToVariant(value)
    
    let inline fromVariant<'T when 'T :> IGodotConvert<'T>> (v: Variant) =
        (Unchecked.defaultof<'T> :> IGodotConvert<'T>).FromVariant(v)
```

---

## 7. Initialization & Entry Point

### 7.1 GDExtension Entry Point

```fsharp
// Init/EntryPoint.fs
module GDExtensionWrapper.Init.EntryPoint

open System
open System.Runtime.InteropServices
open GDExtensionWrapper

/// The main entry point called by Godot
[<UnmanagedCallersOnly(EntryPoint = "godot_gdextension_entry")>]
let godotGdextensionEntry
    (getProcAddress: nativeint)
    (library: nativeint)
    (initialization: nativeint) : byte =
    
    try
        // Convert function pointer to delegate
        let getProcAddr = Marshal.GetDelegateForFunctionPointer<GetProcAddress>(getProcAddress)
        
        // Initialize the interface
        GdExtensionInterface.initialize(getProcAddr, library)
        
        // Initialize method tables
        MethodTables.initialize()
        
        // Set up initialization callbacks
        let initStruct = Marshal.PtrToStructure<GDExtensionInitialization>(initialization)
        initStruct.minimum_initialization_level <- InitializationLevel.Scene
        initStruct.initialize <- initializeCallback
        initStruct.deinitialize <- deinitializeCallback
        Marshal.StructureToPtr(initStruct, initialization, false)
        
        1uy // Success
    with ex ->
        Console.Error.WriteLine($"GDExtension init failed: {ex}")
        0uy // Failure

let private initializeCallback (userData: nativeint) (level: InitializationLevel) =
    match level with
    | InitializationLevel.Core ->
        // Initialize core types
        ()
    | InitializationLevel.Servers ->
        // Initialize server types
        ()
    | InitializationLevel.Scene ->
        // Register user classes
        ClassRegistry.registerAllClasses()
    | InitializationLevel.Editor ->
        // Register editor plugins
        ()

let private deinitializeCallback (userData: nativeint) (level: InitializationLevel) =
    match level with
    | InitializationLevel.Scene ->
        ClassRegistry.unregisterAllClasses()
    | _ -> ()
```

---

## 8. Code Generator Design

### 8.1 Generator Pipeline

```fsharp
// Generator/Program.fs
module Generator.Program

open System.IO
open System.Text.Json

[<EntryPoint>]
let main args =
    let apiPath = args.[0]
    let outputPath = args.[1]
    
    // 1. Parse extension_api.json
    let json = File.ReadAllText(apiPath)
    let api = JsonSerializer.Deserialize<ExtensionApi>(json, jsonOptions)
    
    // 2. Build context (inheritance tree, type mappings)
    let ctx = Context.build api
    
    // 3. Generate builtin types (Vector3, String, etc.)
    for builtin in api.builtin_classes do
        let code = BuiltinGenerator.generate ctx builtin
        File.WriteAllText(Path.Combine(outputPath, "Builtins", $"{builtin.name}.fs"), code)
    
    // 4. Generate engine classes (Node, Resource, etc.)
    for cls in api.classes do
        let code = ClassGenerator.generate ctx cls
        File.WriteAllText(Path.Combine(outputPath, "Classes", $"{cls.name}.fs"), code)
    
    // 5. Generate enums
    let enumCode = EnumGenerator.generateAll api.global_enums
    File.WriteAllText(Path.Combine(outputPath, "Enums.fs"), enumCode)
    
    // 6. Generate utility functions
    let utilCode = UtilityGenerator.generate api.utility_functions
    File.WriteAllText(Path.Combine(outputPath, "UtilityFunctions.fs"), utilCode)
    
    // 7. Generate method tables
    let tableCode = MethodTableGenerator.generate ctx api
    File.WriteAllText(Path.Combine(outputPath, "..", "Ffi", "GeneratedMethodTables.fs"), tableCode)
    
    0
```

### 8.2 Type Mapping

```fsharp
// Generator/TypeMapping.fs
module Generator.TypeMapping

/// Map Godot type names to F# types
let mapType (godotType: string) =
    match godotType with
    // Primitives
    | "int" | "Int" -> "int64"
    | "float" | "Float" -> "float"
    | "bool" | "Bool" -> "bool"
    
    // Builtins
    | "String" -> "GodotString"
    | "StringName" -> "StringName"
    | "Vector2" -> "Vector2"
    | "Vector3" -> "Vector3"
    | "Vector4" -> "Vector4"
    | "Color" -> "Color"
    | "Transform2D" -> "Transform2D"
    | "Transform3D" -> "Transform3D"
    | "Basis" -> "Basis"
    | "Quaternion" -> "Quaternion"
    | "AABB" -> "Aabb"
    | "Plane" -> "Plane"
    | "Rect2" -> "Rect2"
    | "Rect2i" -> "Rect2i"
    
    // Arrays
    | "Array" -> "GodotArray"
    | "PackedByteArray" -> "PackedByteArray"
    | "PackedInt32Array" -> "PackedInt32Array"
    | "PackedFloat32Array" -> "PackedFloat32Array"
    | "PackedStringArray" -> "PackedStringArray"
    | "PackedVector2Array" -> "PackedVector2Array"
    | "PackedVector3Array" -> "PackedVector3Array"
    
    // Dictionary
    | "Dictionary" -> "GodotDictionary"
    
    // Variant
    | "Variant" -> "Variant"
    
    // Enums (format: "enum::EnumName" or "bitfield::EnumName")
    | s when s.StartsWith("enum::") -> s.Substring(6)
    | s when s.StartsWith("bitfield::") -> s.Substring(10)
    
    // Typed arrays (format: "typedarray::ClassName")
    | s when s.StartsWith("typedarray::") -> 
        let inner = s.Substring(12)
        $"TypedArray<{mapType inner}>"
    
    // Object types
    | s -> s // Assume it's a class name
```

---

## 9. LibGodot Embedding (Experimental)

### 9.1 Building Godot as a Library

```bash
# Build Godot as a shared library (requires source modifications)
scons platform=windows target=template_debug library_type=shared_library

# With custom patches for exposing Main class
scons platform=windows custom_modules=../my_embed_module
```

### 9.2 Embedding API (If Available)

```fsharp
// Embedding/LibGodot.fs
module GDExtensionWrapper.Embedding.LibGodot

open System
open System.Runtime.InteropServices

/// Platform-specific library names
let private libraryName =
    if OperatingSystem.IsWindows() then "godot.windows.editor.x86_64.dll"
    elif OperatingSystem.IsLinux() then "libgodot.linux.editor.x86_64.so"
    elif OperatingSystem.IsMacOS() then "libgodot.macos.editor.universal.dylib"
    else failwith "Unsupported platform"

/// DLL import resolver for cross-platform loading
let private resolver _ _ _ =
    NativeLibrary.Load(libraryName)

/// Register the resolver
do NativeLibrary.SetDllImportResolver(typeof<LibGodot>.Assembly, resolver)

// Hypothetical C API (requires custom Godot build)
[<DllImport("godot", CallingConvention = CallingConvention.Cdecl)>]
extern int godot_create_instance(int argc, string[] argv)

[<DllImport("godot", CallingConvention = CallingConvention.Cdecl)>]
extern bool godot_iteration()

[<DllImport("godot", CallingConvention = CallingConvention.Cdecl)>]
extern void godot_destroy_instance()

/// High-level embedding API
type GodotInstance private () =
    static let mutable instance: GodotInstance option = None
    
    /// Create and run Godot
    static member Create(args: string[]) =
        if instance.IsSome then
            failwith "Godot instance already created"
        
        let argc = args.Length
        let result = godot_create_instance(argc, args)
        if result <> 0 then
            failwithf "Failed to create Godot instance: %d" result
        
        let inst = GodotInstance()
        instance <- Some inst
        inst
    
    /// Run one iteration of the main loop
    member _.Step() : bool =
        godot_iteration()
    
    /// Run until exit
    member this.Run() =
        while this.Step() do ()
    
    interface IDisposable with
        member _.Dispose() =
            godot_destroy_instance()
            instance <- None
```

---

## 10. Comparison with Other Bindings

| Feature | godot-cpp (C++) | godot-rust (gdext) | F# (This Design) |
|---------|-----------------|--------------------|--------------------|
| **Generator Input** | extension_api.json | extension_api.json | extension_api.json |
| **Method Loading** | Lazy (hash-based) | Lazy (hash-based) | Lazy (hash-based) |
| **String Interning** | Yes (StringCache) | Yes (StringCache) | Yes (StringCache) |
| **Smart Pointers** | Wrapped class | Gd<T>, GdRef<T> | Gd<'T>, GdRef<'T> |
| **Class Registration** | Macros (C++) | Proc macros (Rust) | Attributes + Reflection |
| **Memory Safety** | Manual | Enforced (borrow checker) | Runtime checks + IDisposable |
| **Variant Handling** | Implicit conversions | From/Into traits | Active patterns + IGodotConvert |

---

## 11. Testing Strategy

### 11.1 Unit Tests (No Godot)

```fsharp
// Tests/BuiltinTests.fs
module Tests.BuiltinTests

open Xunit
open GDExtensionWrapper.Builtins

[<Fact>]
let ``Vector3 addition works`` () =
    let a = Vector3(1f, 2f, 3f)
    let b = Vector3(4f, 5f, 6f)
    let c = a + b
    Assert.Equal(5f, c.x)
    Assert.Equal(7f, c.y)
    Assert.Equal(9f, c.z)

[<Fact>]
let ``Vector3 dot product`` () =
    let a = Vector3(1f, 0f, 0f)
    let b = Vector3(0f, 1f, 0f)
    Assert.Equal(0f, a.Dot(b))
```

### 11.2 Integration Tests (With Godot)

```fsharp
// Tests/IntegrationTests.fs
module Tests.IntegrationTests

open Xunit
open GDExtensionWrapper
open GDExtensionWrapper.Classes

[<Fact>]
let ``Can create Node and add to scene`` () =
    use node = Node3D.New()
    Assert.True(node.IsValid)
    
    node.Position <- Vector3(1f, 2f, 3f)
    Assert.Equal(Vector3(1f, 2f, 3f), node.Position)
```

---

## 12. Roadmap

### Phase 1: Core FFI (Current)
- [ ] Parse `extension_api.json`
- [ ] Generate FFI function pointer bindings
- [ ] Implement StringCache
- [ ] Implement lazy method tables

### Phase 2: Builtin Types
- [ ] Generate all builtin types (Vector2/3/4, Color, etc.)
- [ ] Implement operators and methods
- [ ] Add F# extension methods for idiomatic usage

### Phase 3: Engine Classes
- [ ] Generate all engine classes
- [ ] Implement Gd<T> smart pointer
- [ ] Handle RefCounted correctly

### Phase 4: Class Registration
- [ ] Implement [<GodotClass>] attribute
- [ ] Method registration
- [ ] Property export
- [ ] Signal system

### Phase 5: Polish
- [ ] Documentation generation
- [ ] Error messages
- [ ] Performance optimization
- [ ] Hot reload support

---

## 13. References

1. **godot-cpp**: https://github.com/godotengine/godot-cpp
   - Python binding generator (`binding_generator.py`)
   - `Wrapped` base class pattern
   - Method table caching

2. **godot-rust (gdext)**: https://github.com/godot-rust/gdext
   - Rust proc-macros for class registration
   - `Gd<T>` smart pointer design
   - Lazy method table loading
   - Context-based code generation

3. **GDExtension C Interface**: `gdextension_interface.h`
   - All function pointer types
   - Initialization protocol
   - Class registration API

4. **extension_api.json**: Godot's API definition
   - Class hierarchy
   - Method signatures and hashes
   - Builtin type sizes

---

## 14. Key Insights from Research

### From godot-cpp:
- **binding_generator.py** is a ~2300 line Python script that generates all C++ bindings
- Uses a `Wrapped` base class pattern where all engine classes inherit from it
- Method bindings are cached in static `_MethodBindings` structs per class
- Operators are generated with proper C++ overloading

### From godot-rust:
- Uses Rust's proc-macro system for `#[derive(GodotClass)]` and `#[godot_api]`
- `Gd<T>` is a smart pointer that handles both ref-counted and manually-managed objects
- Method tables are lazily loaded using `ConcurrentDictionary` equivalent
- Has separate "API levels" (Core, Servers, Scene, Editor) for hot-reload support
- Uses `Context` struct during codegen to track inheritance, singletons, etc.

### Critical Implementation Details:
1. **Method Hashes**: Every method in `extension_api.json` has a hash. This must be used when calling `classdb_get_method_bind`
2. **StringName Caching**: Creating StringNames is expensive - must cache them
3. **Initialization Order**: Core → Servers → Scene → Editor
4. **Reference Counting**: RefCounted objects need `reference()` called on construction and `unreference()` on drop
