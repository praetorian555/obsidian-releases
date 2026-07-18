# Obsidian

Obsidian is a reflection code generator for C++20/23. You annotate enums, classes, and properties with marker macros, run the `obsidian` tool over your headers, and get generated C++ headers providing both compile-time and runtime reflection — no RTTI, no runtime parsing, no hand-maintained registration tables.

What the generated reflection gives you:

- **Compile-time facades** — `Obs::Enum<T>` and `Obs::Class<T>` with names, scopes, descriptions, properties, and value conversions
- **Runtime collections** — look up any reflected type by name or type id, enumerate all types, construct/destroy/copy instances through a `void*` interface
- **Attributes** — free-form `key=value` metadata on enums, enumerators, classes, and properties, queryable at compile time and runtime
- **Stable type ids** — 32-bit hashes of qualified type names, safe to serialize and send over the wire
- **Checked property access** — typed reads and writes that fail cleanly on type mismatch instead of corrupting memory
- **Deep copy and equality** — generated per-type operations that walk reflected properties, including for move-only and non-comparable member types
- **Container support** — type-erased size/resize/element access for recognized container properties
- **Inheritance** — single-base reflected chains with flattened property lists and parent links
- **Doc comments** — `///` and `/** */` comments on reflected entities become their `description` strings

## Getting Obsidian

Each release ships a zip with one self-contained install per platform:

```
obsidian/
├── win32/    # Windows x64
│   ├── obsidian.exe
│   ├── libclang.dll
│   ├── include/obs/    # obs.hpp (markers), reflection-core.hpp (fixed machinery)
│   ├── README.md       # this document
│   └── LICENSE, LLVM-LICENSE
└── linux/    # Linux x64
    ├── obsidian
    ├── libclang.so
    ├── include/obs/
    ├── README.md
    └── LICENSE, LLVM-LICENSE
```

The tool runs standalone — libclang ships next to the executable. Your project needs:

- a **C++20 (or C++23) compiler** for the generated headers,
- the **[Opal](https://github.com/praetorian555/opal)** utility library — generated code uses `Opal::DynamicArray` for its collections and allocators for object construction, so targets that include generated headers must link `opal`.

## Quick Start

**1. Annotate** your types (the macros come from `include/obs/obs.hpp` and expand to nothing — they are markers for the parser):

```cpp
#include "obs/obs.hpp"

namespace MyGame
{

/// Player character data.
OBS_CLASS("serializable")
struct Character
{
    OBS_PROP("min=0", "max=100")
    int32_t health = 100;

    OBS_PROP()
    float speed = 5.0f;
};

} // namespace MyGame
```

**2. Generate** reflection data:

```bash
obsidian input-files=character.hpp output-dir=generated inc-dirs=include std=c++20
```

**3. Use** it:

```cpp
#include "reflection.hpp"

Obs::Class<MyGame::Character>::GetDescription();   // "Player character data."

MyGame::Character player;
int32_t hp = 0;
Obs::ClassCollection::Read(&hp, &player, "Character", "health");   // hp == 100
```

## Annotating Types

Mark types with `OBS_ENUM()`, `OBS_CLASS()`, and `OBS_PROP()` (plus `OBS_ITEM()` for individual enumerators). Attach attributes to any marker as comma-separated `"key=value"` string arguments; omitting the value defaults it to `"1"`.

```cpp
#include "obs/obs.hpp"

namespace MyGame
{

/// A character class.
OBS_ENUM("flags")
enum class CharacterClass : int8_t
{
    /** The warrior. */
    OBS_ITEM("dsl=warrior")     // attributes on an individual enumerator
    Warrior = 0,
    Mage,
    Rogue,
};

/// Player character data.
OBS_CLASS("serializable")
struct Character
{
    OBS_PROP("min=0", "max=100")
    int32_t health = 100;

    OBS_PROP()
    float speed = 5.0f;

    OBS_PROP()
    CharacterClass char_class = CharacterClass::Warrior;
};

} // namespace MyGame
```

A marker must be placed on the declaration's line or on the line directly above it, and it cannot be hidden inside another macro. Obsidian verifies every marker it sees: a marker that does not end up attached to a reflected declaration (wrong placement, an unsupported declaration, or an `OBS_PROP` inside a class that has no `OBS_CLASS`) fails the run with an error pointing at the marker's location, instead of silently dropping the type.

### Private members and accessors

Reflecting a non-public member requires an `OBS_BODY()` marker anywhere inside the class — it expands to friend declarations granting the generated code access (a non-public property without it fails the run). Three property attribute keys are reserved to control access:

- `name=x` — the reflected property name, when it should differ from the member name (`m_gap` reflected as `gap`).
- `set=Method` — reflected writes (`TrySet`, the `write` transfer, `ClassCollection::Write`) call `Method(value)` instead of assigning the field, so setter side effects fire.
- `get=Method` — reflected reads call `Method()` instead of reading the field.

`GetPtr`/`TryGetAs` remain raw access into the object by design, and the generated per-type copy and equality transfer fields directly — accessors are for the value-transfer paths. The tool validates that `get`/`set` name existing member functions.

```cpp
OBS_CLASS()
class Slider
{
public:
    OBS_BODY()

    void SetValue(float v);   // clamps and invalidates
    float GetValue() const;

private:
    OBS_PROP("name=value", "set=SetValue", "get=GetValue")
    float m_value = 0.0f;
};
```

## Running the Tool

Process specific files:

```bash
obsidian input-files=my_types.hpp,some-folder/other-header.hpp output-dir=generated \
    compile-options=-DMY_DEFINE,-Wall inc-dirs=include/this/dir std=c++20
```

Process all headers in a directory (recursively finds `.h` and `.hpp` files):

```bash
obsidian input-dirs=my-lib/include,dependency/include output-dir=generated \
    inc-dirs=include/this/dir
```

### Command-Line Arguments

| Argument                 | Required | Description                                                                                |
|--------------------------|----------|--------------------------------------------------------------------------------------------|
| `version`                | No       | Print version and exit                                                                     |
| `help`                   | No       | Print help                                                                                 |
| `input-files=<paths>`    | Yes*     | Comma-separated paths to input header files                                                |
| `input-dirs=<paths>`     | Yes*     | Comma-separated paths to directories with input headers (recursive)                        |
| `output-dir=<path>`      | Yes      | Output directory for generated headers (must exist)                                        |
| `std=<version>`          | No       | C++ standard version (default: `c++20`). Supported: `c++20`, `c++23`                      |
| `compile-options=<opts>` | No       | Comma-separated Clang compile options                                                      |
| `inc-dirs=<dirs>`        | No       | Comma-separated list of include directories (automatically prefixed with `-I`)             |
| `log-level=<level>`      | No       | Control verbosity of logs. Supported: `verbose`, `info`, `error` (default: `error`)        |
| `type-validation=<mode>` | No       | Validation of reflected property types. Supported: `permissive`, `strict` (default: `permissive`) |
| `type-whitelist=<value>` | No       | Allowed property types for strict validation: a built-in profile name (`opal`) or a path to a whitelist file. Required when `type-validation=strict` |
| `dump-ast=true`          | No       | Dump the extracted AST metadata                                                            |

\*You must specify either `input-files` or `input-dirs` but not both.

### Type Validation

Obsidian can validate the types of reflected properties:

- **Permissive (default):** every property type is accepted. Properties with raw pointer or reference types log a warning (visible at `log-level=info` or higher), since such properties cannot be safely serialized.
- **Strict (`type-validation=strict`):** every property type must be valid, otherwise the run fails with an error per offending property. A type is valid if it is:
  - a fundamental type (`bool`, integers, floating-point types),
  - an enum or class reflected in the same run (nested reflected structs recurse naturally),
  - listed in the whitelist by its qualified name,
  - a fixed-size array or template instantiation of valid types — template arguments are validated recursively, and a whitelisted template name (e.g. `Opal::DynamicArray`) allows every instantiation whose arguments are themselves valid.

  Raw pointers and references are always rejected in strict mode, regardless of the whitelist. Strict mode also requires every concrete reflected class to be default-constructible (in permissive mode this only logs a warning).

Strict mode requires the `type-whitelist` argument. It accepts either a built-in profile name or a path to a text file with one qualified type name per line (blank lines and lines starting with `#` are ignored):

```
# my-whitelist.txt
Opal::DynamicArray
MyEngine::Vec3
```

The only built-in profile so far is `opal`, which allows `Opal::String` (and thus `Opal::StringUtf8`, which resolves to it), `Opal::DynamicArray`, `Opal::InPlaceArray` and `Opal::Optional`. Note that whitelist matching happens on *canonical* type names: a type alias resolves to the underlying type, so for example allowing `std::string` requires whitelisting `std::basic_string` (plus `std::char_traits` and `std::allocator`, its template arguments).

## CMake Integration

Point a cache variable at an unpacked install (the `win32/` or `linux/` directory of a release) and run the tool as a custom target before your code compiles. To avoid checking the install into your repo, fetch the release zip at configure time:

```cmake
include(FetchContent)

set(MY_OBSIDIAN_DIR "" CACHE PATH "Path to an obsidian install (empty: download the pinned release)")
if (NOT MY_OBSIDIAN_DIR)
    FetchContent_Declare(obsidian_release
            URL https://github.com/praetorian555/obsidian-releases/releases/download/v0.9.3/obsidian-0.9.3.zip
            DOWNLOAD_EXTRACT_TIMESTAMP TRUE)
    FetchContent_MakeAvailable(obsidian_release)
    if (WIN32)
        set(MY_OBSIDIAN_DIR ${obsidian_release_SOURCE_DIR}/win32)
    else ()
        set(MY_OBSIDIAN_DIR ${obsidian_release_SOURCE_DIR}/linux)
    endif ()
endif ()

add_executable(my_app src/main.cpp)

# The clang parse needs the same include roots my_app compiles with, plus the obs headers.
add_custom_target(generate_reflection
        COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/generated
        COMMAND ${MY_OBSIDIAN_DIR}/obsidian${CMAKE_EXECUTABLE_SUFFIX}
            input-files=${PROJECT_SOURCE_DIR}/include/my_types.hpp
            output-dir=${PROJECT_BINARY_DIR}/generated
            "inc-dirs=$<JOIN:$<TARGET_PROPERTY:my_app,INCLUDE_DIRECTORIES>,,>,${MY_OBSIDIAN_DIR}/include"
            std=c++20
        COMMENT "Generating obsidian reflection data"
        VERBATIM)

add_dependencies(my_app generate_reflection)
target_include_directories(my_app PRIVATE
        ${PROJECT_BINARY_DIR}/generated
        ${MY_OBSIDIAN_DIR}/include)
```

If your target has compile definitions the reflected headers depend on, forward them the same way through `compile-options=` (prefix each with `-D`).

## Output Layout

The tool writes into `output-dir`:

```
reflection.hpp             # umbrella: includes every generated file + the runtime collections
<stem>.<hash8>.refl.hpp    # one per source header defining reflected types
reflection-manifest.json   # tool version, umbrella name, [{source, generated}] mappings
```

Most consumers just include `reflection.hpp`. A translation unit that only needs the compile-time API of one header's types (`Obs::Class<T>`, `Obs::Enum<T>`, checked access) can include that header's generated file instead — it pulls in the generated files of its dependencies itself — which keeps rebuilds after a header edit from touching every reflection-using TU. The `<hash8>` is derived from the source header's path, so the name is stable across runs; the manifest maps source headers to generated files for build-system integration (the file list, or a script resolving the generated name). Outputs are only rewritten when their content changes, so timestamps stay stable for incremental builds, and generated files whose source header disappeared are deleted on the next run.

The fixed machinery (the `Property`/`ClassEntry` structs, transfer templates, container adapters) lives in `include/obs/reflection-core.hpp`, which ships with the tool next to `obs/obs.hpp`. Generated files include it and refuse to compile (`#error`) against a core header from a different tool version — always regenerate after updating the tool.

Reflected types may reference reflected types from other headers freely (bases, property types, container elements) — the generated files include each other along those edges. The one restriction: two headers whose reflected types mutually depend on each other are a hard error naming the cycle. That situation can only arise through a container of a forward-declared element (`Opal::DynamicArray<B>` compiles with `B` incomplete); move the involved types into one header or break the forward-declared dependency.

## Using the Generated Reflection

Include the generated `reflection.hpp` in your code. It can be safely included from multiple translation units.

### Compile-time enum reflection

```cpp
#include "reflection.hpp"

using namespace MyGame;

// Enum metadata
Obs::Enum<CharacterClass>::GetName();               // "CharacterClass"
Obs::Enum<CharacterClass>::GetScope();              // "MyGame"
Obs::Enum<CharacterClass>::GetScopedName();         // "MyGame::CharacterClass"
Obs::Enum<CharacterClass>::GetDescription();        // "A character class."

// Value conversions
Obs::Enum<CharacterClass>::GetValueName(CharacterClass::Warrior);   // "Warrior"
Obs::Enum<CharacterClass>::GetValueDescription(CharacterClass::Warrior); // "The warrior."
Obs::Enum<CharacterClass>::GetValue("Mage");        // CharacterClass::Mage
Obs::Enum<CharacterClass>::GetValue(2);             // CharacterClass::Rogue
Obs::Enum<CharacterClass>::GetUnderlyingValue(CharacterClass::Warrior); // 0
```

### Compile-time class reflection

Properties can be both POD types (e.g., `int`, `float`, `enum`) and non-POD types (e.g., `std::string`). For POD types, read and write operations use raw memory copies. For non-POD types, copy assignment is used, so the type must be copy-assignable.

```cpp
// Class metadata
Obs::Class<Character>::GetName();        // "Character"
Obs::Class<Character>::GetScopedName();  // "MyGame::Character"
Obs::Class<Character>::GetDescription(); // "Player character data."

// Iterate properties
for (const auto& prop : Obs::Class<Character>::Get())
{
    printf("%s: type=%s, offset=%d, size=%d\n",
           prop.name, prop.type_name, prop.offset, prop.size);
}

// Read and write by property name
Character player;
int hp = 0;
Obs::Class<Character>::Read(&hp, &player, "health");    // hp == 100

int new_hp = 50;
Obs::Class<Character>::Write(&new_hp, &player, "health"); // player.health == 50
```

### Attributes

Attributes are available on enums, individual enumerators, classes, properties, and their runtime counterparts (`EnumEntry`, `EnumItem`, `ClassEntry`, `Property`). Each type provides `HasAttribute` and `GetAttributeValue` member functions.

```cpp
// Compile-time enum attributes
Obs::Enum<CharacterClass>::HasAttribute("flags");          // true
Obs::Enum<CharacterClass>::GetAttributeValue("flags");     // "1"
Obs::Enum<CharacterClass>::HasAttribute("other");          // false
Obs::Enum<CharacterClass>::GetAttributeValue("other");     // nullptr

// Compile-time class attributes
Obs::Class<Character>::HasAttribute("serializable");       // true
Obs::Class<Character>::GetAttributeValue("serializable");  // "1"

// Property attributes (via iteration)
for (const auto& prop : Obs::Class<Character>::Get())
{
    if (prop.HasAttribute("min"))
    {
        printf("%s min=%s max=%s\n", prop.name,
               prop.GetAttributeValue("min"),    // "0"
               prop.GetAttributeValue("max"));   // "100"
    }
}

// Runtime entry attributes
const Obs::EnumEntry* entry = nullptr;
Obs::EnumCollection::GetEnum("CharacterClass", entry);
entry->HasAttribute("flags");       // true
entry->GetAttributeValue("flags");  // "1"
```

### Per-enumerator attributes

`OBS_ITEM(...)` on an enumerator (its line or the line directly above; a doc comment above the marker still attaches) stores attributes on that enumerator's `EnumItem`. `Enum<T>` adds per-value queries and a reverse lookup by attribute value — the primitive for mapping external spellings (a DSL keyword, a config token) to enum values without a parallel table. A misplaced `OBS_ITEM` (detached, or inside an unreflected enum) fails the run like any other orphaned marker.

```cpp
Obs::Enum<CharacterClass>::HasValueAttribute(CharacterClass::Warrior, "dsl");        // true
Obs::Enum<CharacterClass>::GetValueAttributeValue(CharacterClass::Warrior, "dsl");   // "warrior"
Obs::Enum<CharacterClass>::GetValue("dsl", "warrior");   // CharacterClass::Warrior (k_end if no match)
Obs::Enum<CharacterClass>::GetItem(CharacterClass::Mage); // the EnumItem, or null for unknown values

for (const Obs::EnumItem& item : *entry->items)           // runtime: items carry the same attributes
{
    if (item.HasAttribute("dsl")) { /* ... */ }
}
```

### Object construction

Obsidian can construct and destroy reflected class instances through an allocator. The compile-time API returns a typed pointer — destroy those with `Opal::Delete`. The runtime API returns `void*`, and since the caller may not know the static type, destruction goes through reflection too: `ClassEntry::destroy` or `ClassCollection::Destroy` (by name or type id). Objects are default-constructed (using their in-class member initializers).

Abstract classes and classes without a public default constructor can still be reflected (metadata, properties, attributes all work), but they cannot be constructed: `Obs::Class<T>::Create` is not generated for them, their `ClassEntry::create` is `nullptr`, and `ClassCollection::Construct` returns `nullptr`. Likewise, a class without a public destructor gets `ClassEntry::destroy == nullptr` and `ClassCollection::Destroy` returns false for it.

```cpp
Opal::MallocAllocator allocator;

// Compile-time: returns a typed pointer
Character* player = Obs::Class<Character>::Create(&allocator);
// player->health == 100, player->speed == 5.0f, etc.

// Runtime: look up by name, returns void*
void* obj = Obs::ClassCollection::Construct("Character", &allocator);

// Returns nullptr for unknown types
void* bad = Obs::ClassCollection::Construct("NonExistent", &allocator);
// bad == nullptr

// Can also construct via ClassEntry directly
const Obs::ClassEntry* entry = nullptr;
Obs::ClassCollection::GetClassEntry("Character", entry);
void* obj2 = entry->create(&allocator);
// entry->size and entry->alignment are also available

// Cleanup: typed pointers via Opal::Delete, void* through reflection
Opal::Delete(&allocator, player);
Obs::ClassCollection::Destroy(obj, "Character", &allocator);
entry->destroy(obj2, &allocator);
```

### Object copy

Every reflected class gets a generated deep copy that walks its reflected properties: plain values are assigned, container properties are cloned, and nested reflected types recurse through their own generated copy. This works even for types whose container members delete the C++ copy assignment, and it is what makes such types usable as properties of other reflected types. Unreflected members are not copied — reflection copy transfers reflected state only. The `"readonly"` attribute guards `TrySet`, not whole-object copy.

```cpp
Character dst;

// Compile-time: typed references
Obs::Class<Character>::Copy(dst, src);

// Runtime: by name, type id, or ClassEntry; dst and src must be live objects of the class
Obs::ClassCollection::Copy(&dst, &src, "Character");
Obs::ClassCollection::Copy(&dst, &src, Obs::Class<Character>::GetTypeId());
entry->copy(&dst, &src);
```

### Runtime reflection (string-based lookup)

Collections in the generated API are `Opal::DynamicArray` containers — use `GetSize()`, `IsEmpty()`, `operator[]`, or range-for to work with them. The runtime entries are views over the compile-time layer: `EnumEntry::items`, `ClassEntry::properties`, and the entry `attributes` are pointers to the arrays owned by the `Obs::Enum<T>` / `Obs::Class<T>` specializations, so the two layers share one copy of the data (dereference them: `entry.properties->GetSize()`, `(*entry.properties)[0]`).

```cpp
// Look up enum by name at runtime
const Obs::EnumEntry* entry = nullptr;
Obs::EnumCollection::GetEnum("CharacterClass", entry);
// entry->name, entry->items, entry->underlying_type_size, ...

// Convert enum value by string
CharacterClass cls;
Obs::EnumCollection::GetValue(&cls, "CharacterClass", "Mage");

// Look up class by name at runtime
const Obs::ClassEntry* class_entry = nullptr;
Obs::ClassCollection::GetClassEntry("Character", class_entry);

// Read/write properties by class and property name strings
int hp = 0;
Character player;
Obs::ClassCollection::Read(&hp, &player, "Character", "health");
Obs::ClassCollection::Write(&hp, &player, "Character", "health");
```

Name lookups accept either the short name (`"Character"`) or the qualified name (`"MyGame::Character"`). An exact qualified-name match always wins; a short name resolves only while exactly one reflected type has it. A short name shared by several types (the same class name in two namespaces) is ambiguous and the lookup fails — use the qualified name or a type id.

### Checked property access

`Read`/`Write` copy bytes through a `void*` and trust the caller to pass a buffer of the right type. For type-safe access, every property carries the type id of its declared type, and `Property` offers checked accessors: the requested type must match the property type exactly (no conversions), so a mismatch is a clean failure instead of memory corruption. `TrySet` also rejects properties marked with the `"readonly"` attribute. `TypeIdOf<T>` maps a C++ type to its type id — it is specialized for fundamental types, for every reflected type, and for every template instantiation appearing as a reflected property type (container instantiations, `std::string`, consumer wrapper types); requesting an unsupported type (pointers, unreflected non-template classes) is a compile error.

```cpp
const Obs::Property* prop = nullptr;
Obs::ClassCollection::GetProperty(*entry, "health", prop);

// Level 0: raw pointer into the instance (instance + offset)
void* raw = prop->GetPtr(&player);

// Level 1: checked typed access
int32_t* health = prop->TryGetAs<int32_t>(&player);   // nullptr if the property is not an int32_t
prop->TrySet(&player, 150);                           // false on type mismatch or "readonly" attribute
```

### Type-erased value ops

For callers whose types are only known at runtime (editor property grids, serializers), `ValueOpsCollection` dispatches deep copy and deep equality by type id. Entries exist for fundamentals, reflected enums and classes, and template instantiations appearing as reflected property types. Both operations are deep: containers work element-wise, reflected classes property-wise — so equality works even for types without `operator==`, as long as every reflected property is comparable (a `DynamicArray<Point>` compares element-wise through reflected property equality). When a type is not comparable (an unreflected property type without `operator==`), its `equals` is null and `AreEqual` returns false without writing the result; when a type cannot be deep-copied (see opaque instantiations below), its `copy` is null and `CopyValue` returns false.

```cpp
// Copy a property value between two objects of the same class, knowing only the type id
Obs::ValueOpsCollection::CopyValue(prop->GetPtr(&dst_obj), prop->GetPtr(&src_obj), prop->type_id);

// Diff two objects property by property
bool equal = false;
if (Obs::ValueOpsCollection::AreEqual(equal, prop->GetPtr(&a), prop->GetPtr(&b), prop->type_id) && !equal)
{
    // property differs - write it out
}

// Or grab the ops table directly
const Obs::ValueOps* ops = Obs::ValueOpsCollection::GetValueOps(prop->type_id);  // null for unknown ids
```

### Type IDs

Every reflected type gets a stable 32-bit identity: the FNV-1a hash of its canonical qualified name (fully qualified, no leading `::`). Type ids are computed at compile time, never change across builds or platforms as long as the type keeps its name, and are safe to store on disk or send over the wire — use them instead of names for serialization. The tool fails the build if two reflected types hash to the same id, and rejects types in anonymous namespaces (they have no stable qualified name). A default-constructed `Obs::TypeID` is the "no type" value (`IsValid()` returns false).

```cpp
// Compile time - usable in constexpr contexts, switch tables, static_asserts
constexpr Obs::TypeID id = Obs::Class<Character>::GetTypeId();
static_assert(Obs::Class<Character>::GetTypeId() == Obs::TypeID("MyGame::Character"));

// Runtime - look up and construct by type id (e.g. an id read from a save file)
const Obs::ClassEntry* entry = nullptr;
Obs::ClassCollection::GetClassEntry(id, entry);            // entry->type_id, entry->parent_type_id
void* obj = Obs::ClassCollection::Construct(id, &allocator);

const Obs::EnumEntry* enum_entry = nullptr;
Obs::EnumCollection::GetEnum(Obs::Enum<CharacterClass>::GetTypeId(), enum_entry);
```

### Container properties

Properties typed as `Opal::DynamicArray<T>`, `Opal::InPlaceArray<T, N>`, or `Opal::Optional<T>` are recognized as containers. Their `Property::container_ops` points at a type-erased vtable (null for non-container properties) with `size`, `resize`, `element_at`, `clear`, `copy` (deep clone), and `equals`, plus the container kind and the element's type id — enough to serialize or edit a container without knowing its element type at compile time. Fixed-size arrays have null `resize`/`clear`; `Opal::Optional<T>` acts as a 0-or-1 container (`resize(1)` emplaces a default value); `equals` is null when the element type has no `operator==`. Container instantiations also get `TypeIdOf` specializations, so `TryGetAs<Opal::DynamicArray<float>>` works. The element type must be a fundamental, enum, or class — containers of containers, pointers, or references reflect as opaque values without ops (with a warning). Because value transfer goes through `Clone()` + move-assign, Opal's move-only containers work with `Read`/`Write` too.

```cpp
const Obs::Property* prop = nullptr;
Obs::ClassCollection::GetProperty(*entry, "waypoints", prop);   // Opal::DynamicArray<Vec3> waypoints;

const Obs::ContainerOps* ops = prop->container_ops;
void* container = prop->GetPtr(&character);
ops->resize(container, count);                                  // resize-then-fill; never store element pointers
for (uint64_t i = 0; i < count; i++)
{
    Deserialize(ops->element_at(container, i), ops->element_type_id);
}
```

### Opaque template instantiations

Template-instantiation property types that are not recognized containers (`std::string`, framework wrapper types like a `Property<T>` binding) still reflect: they get a type id and a `TypeIdOf` specialization, so `TryGetAs<Wrapper<float>>` returns a typed pointer and the caller drives the wrapper's own API — no tool-side knowledge of the type needed. What the generated code cannot do is transfer such a value: if the type is not copy-assignable, not clonable, and not reflected, the property's `read`/`write`, the class's `copy`, and the value-ops `copy`/`equals` are null (decided at compile time in the generated header), and the corresponding calls return false instead of failing to compile. A write routed through a `set=` accessor stays available — it passes the value through without copying it. The tool warns once per such instantiation.

```cpp
// class Widget { OBS_PROP() Framework::Property<float> gap; ... };
Framework::Property<float>* gap = prop->TryGetAs<Framework::Property<float>>(&widget);  // checked, typed
gap->Set(4.0f);                                     // the wrapper's own API does the mutation
prop->TrySet(&widget, other_gap);                   // false: the wrapper cannot be copied
```

### Inheritance

A reflected class may derive from at most one other reflected class (chains of any depth are fine). The derived class exposes all inherited reflected properties as its own — property lists are flattened base-first — and records its parent: `Obs::Class<T>::GetParentScopedName()` / `ClassEntry::parent_scoped_name` (null when there is no reflected base) and `Obs::Class<T>::GetParentTypeId()` / `ClassEntry::parent_type_id` (invalid when there is no reflected base). Unreflected bases are ignored. Three layouts are hard errors: virtual inheritance, more than one reflected base, and an unreflected class sitting between two reflected ones in the hierarchy.

Reflected property names must be unique across a class and its reflected base chain — member names and `name=` overrides share one namespace. A derived property shadowing an inherited reflected name (or two properties of one class colliding through a `name=` override) fails the run: rename the member or give it a distinct reflected name.

### Type enumeration

Iterate over every reflected type — useful for tooling, validation tests, or building lookup tables.

```cpp
for (const Obs::EnumEntry& entry : Obs::EnumCollection::GetAllEnums())
{
    printf("enum %s (%zu items)\n", entry.full_name, entry.items->GetSize());
}

for (const Obs::ClassEntry& entry : Obs::ClassCollection::GetAllClasses())
{
    printf("class %s (%zu properties)\n", entry.scoped_name, entry.properties->GetSize());
    if (entry.HasAttribute("serializable"))
    {
        // ...
    }
}
```

## Caching

The tool skips regeneration when nothing changed since the last run: the manifest and every generated file it lists still exist, the arguments and tool version are unchanged, and no input header (or whitelist file, when one is used) was added, removed, or modified. The cache lives in `obs.cache` in the working directory of the run — if a run ever reports nothing changed when you know something did, delete that file to force regeneration.

## AI Disclosure

Portions of this software were developed with the assistance of AI tools (Claude by Anthropic). All AI-generated code was reviewed and approved by human contributors.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
