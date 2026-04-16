# UZON Specification

**Version**: 0.8  
**Status**: Pre-release  
**Date**: 2026-04-16  
**Author**: Suho Kang  
**Repository**: [github.com/uzon-dev](https://github.com/uzon-dev)  
**Website**: [uzon.dev](https://uzon.dev)

---

## Preface

This document is the complete specification of UZON. It serves two audiences:

**Document authors** — people who write `.uzon` files. Read §1 through §8 for the language definition, consult Appendix A for a complete working example, and follow Appendix E for style conventions. The formal grammar (§9) and implementer appendices can be skipped.

**Implementers** — people who build parsers, evaluators, and tooling for UZON. The entire document is relevant. §9 provides the formal EBNF grammar, §11 defines conformance levels and error handling requirements, and Appendix D consolidates rules that are distributed across multiple sections into a quick-reference format.

The specification is organized as follows:

- **§1–§3**: Introduction, lexical structure, and type definitions (struct, tuple, list, enum, union, tagged union, function)
- **§4**: Literal syntax (booleans, numbers, strings, null)
- **§5**: Expressions (binding, arithmetic, comparison, conditionals, type conversion, name resolution, environment variables, field extraction, function calls, standard library)
- **§6**: Type system (annotation, naming, reuse)
- **§7**: File import
- **§8**: Comma and newline rules
- **§9**: Formal grammar (EBNF)
- **§10**: File format metadata
- **§11**: Conformance and error handling
- **Appendix A**: Complete example
- **Appendix B**: Comparison with JSON, TOML, YAML
- **Appendix C**: Keyword and operator quick reference
- **Appendix D**: Implementer's quick reference
- **Appendix E**: Style guide

---

## 1. Introduction

UZON is a typed, human-readable data expression format. Every UZON document evaluates to a single, immutable value with no side effects, no loops, and no recursion. At its simplest, UZON is a data format — bindings associate names with values using the `is` keyword, and nothing more is needed. At its most expressive, UZON supports conditional expressions, name references, environment variable access, arithmetic, struct overrides and extensions, file imports, functions, and a standard library of collection and numeric utilities. Everything in UZON is a value: integers, strings, structs, lists, enums, tagged unions, and functions are all first-class values that can be bound, passed, and composed.

A UZON file is an **anonymous struct**: a sequence of **bindings** at the top level. Each binding associates a name with a value using the `is` keyword. For a comparison with JSON, TOML, and YAML, see Appendix B.

### 1.1 Design Goals

1. **Self-describing**: Every value is unambiguously parseable without external type context.
2. **Typed**: Rich primitive and compound types with named type reuse.
3. **Cross-language**: Clean mapping to any mainstream programming language.
4. **Expressive**: Conditionals, arithmetic, name references, and environment variables enable dynamic configuration.
5. **Human-first**: English-like keywords (`is`, `from`, `called`, `as`, `to`, `named`) for readability.
6. **Minimal**: No redundant syntax; every construct earns its place.

### 1.2 Keyword Role Summary

UZON uses English words as operators. The five type-related keywords have strictly separated roles:

| Keyword | Role                               | English meaning                      | Example                          |
| ------- | ---------------------------------- | ------------------------------------ | -------------------------------- |
| `as`    | **Annotation** (assertion)         | "this value **as** a Type"           | `42 as i32`, `blue as RGB`       |
| `to`    | **Conversion** (transformation)    | "convert this **to** a Type"         | `env.PORT to u16`, `3.14 to i32` |
| `of`    | **Extraction** (field lookup)      | "this binding **is of** that struct" | `port is of config`              |
| `type`  | **Type check** (runtime type test) | "this value **is type** that Type"   | `x is type i32`, `case type x` |
| `from`  | **Definition** (enum/union source) | "a variant **from** these choices"   | `red from red, green, blue`      |

`as` declares what a value already **is** — it does not change the value. If the value does not fit, it is an error. `to` transforms a value into a different representation — truncation, parsing, and format changes are expected. `of` extracts a field using the binding's own name as the key. `from` defines the set of valid variants for an enum, or the set of member types for a union.

### 1.3 Notation Conventions

This specification uses Extended Backus–Naur Form (EBNF) for grammar definitions. In prose:

- `monospace` denotes literal syntax or production names.
- *italic* denotes terms defined elsewhere in this specification.
- **bold** denotes normative keywords (MUST, SHOULD, MAY) per RFC 2119.

---

## 2. Lexical Structure

### 2.1 Encoding

A UZON document **MUST** be encoded in UTF-8. A byte order mark (U+FEFF) at the start **SHOULD** be ignored by parsers but **SHOULD NOT** be emitted by generators.

### 2.2 Comments

UZON supports line comments only. A comment begins with `//` and extends to the end of the line.

```uzon
// This is a comment
x is 42  // Inline comment
```

Block comments (`/* */`) are intentionally omitted.

Comments are treated as whitespace by the parser — they do not act as separators (`sep`) and may appear after any token on a line without affecting parsing. A newline consumed by a comment still counts as a line boundary for NEWLINE_SEP purposes — the comment is stripped but the line break remains.

### 2.3 Identifiers

An identifier is any contiguous sequence of non-whitespace characters that does not create ambiguity with another grammatical construct in its syntactic context. There is no restriction on starting characters or character sets — Unicode letters, digits, emoji, and any other non-whitespace characters are all valid in identifiers, provided the resulting token is not ambiguous with a keyword, operator, delimiter, literal, or other grammar element.

```uzon
message is "Hello, World!"   // ASCII identifier
1st is "first"               // starts with digit — valid
안녕 is "인사말"              // Korean identifier
😀😀 is "two smiles"          // emoji identifier
café is "coffee"             // accented characters
π is 3.14159                 // single Unicode character
hello_world is "hi"          // underscores — valid
```

Tokens that fully match a numeric literal grammar (§4.2, §4.3) are parsed as numbers, not identifiers. Numeric literal matching takes priority over token-boundary splitting — `3.14` is a single float literal, not three tokens `3` `.` `14`. However, if the token does not fully match (e.g., `1st`, `0xZZ`), it is an identifier.

```uzon
42       // integer literal (matches number grammar)
1st      // identifier (does not match number grammar)
0xff     // integer literal (matches hex grammar)
0xZZ     // identifier (does not match hex grammar)
```

Whitespace characters (space, tab, LF, CR) and the following ASCII punctuation **MUST** terminate or cannot appear within an unquoted identifier: `{ } [ ] ( ) , . " ' @ + - * / % ^ < > = ! ? : ; | & $ ~ # \` and `//`. This is not a soft guideline — these characters have fixed structural meaning in the grammar or are reserved for future use, and always act as token boundaries. To use these characters in a name, use a quoted identifier (e.g., `'Content-Type'`).

**Quoted identifiers**: An identifier that starts with `'` and ends with `'` may contain any characters between the quotes — including spaces, operators, and keywords. The enclosing quotes are part of the syntax, not the name: `'key'` and `key` refer to the same name. Only a closing `'` or a newline terminates a quoted identifier.

**Keyword restriction**: Since `'key'` and `key` are the same name, `'is'` equals the keyword `is` — it is still a keyword, not an identifier. Quoted identifiers whose content matches a keyword cannot be used as binding names. To use a keyword as a binding name, use `@` escape (§2.4). Operator characters in quoted identifiers are permitted — `'+' is 1` is valid because `+` is not a keyword.

An unmatched `'` (opening without closing on the same line) is a **lexer error**.

```uzon
'Content-Type' is "application/json"    // contains hyphen — valid
'this is a key' is "value"              // contains spaces — valid
'' is "empty"                           // empty name — valid
'+' is 1                                // operator character — valid
'is' is 5                               // INVALID — 'is' = keyword is
@is is 5                                // valid — @ escapes the keyword

// Quoted and unquoted are the same name:
port is 8080
x is 'port'                        // same as port → 8080

// Keyword names in member access also require @:
meta is { @type is "config" }
x is meta.@type                    // "config" — @ required for keyword field
y is meta.'type'                   // INVALID — 'type' = keyword type
```

This enables lossless round-tripping with JSON and other formats where keys are arbitrary strings, while requiring no changes to the rest of the grammar — `'` is in the token boundary list, so the lexer recognizes it as the start of a quoted identifier context.

### 2.4 Keyword Escape

The `@` prefix allows reserved keywords to be used as identifiers.

```uzon
@is is 3           // binding named "is" with value 3
@from is "hello"   // binding named "from" with value "hello"
@true is 0         // binding named "true" with value 0
```

Any keyword may be escaped with `@`. The `@` is not part of the identifier name itself — `@is` and a hypothetical non-keyword `is` refer to the same name.

### 2.5 Keywords

The following identifiers are reserved:

| Category         | Keywords                                                    |
| ---------------- | ----------------------------------------------------------- |
| Values           | `true`, `false`, `null`                                     |
| Numeric          | `inf`, `nan`                                                |
| State            | `undefined`                                                 |
| Binding          | `is`, `are`                                                 |
| Type system      | `from`, `called`, `as`, `named`, `with`, `union`, `plus`, `type` |
| Conversion       | `to`                                                        |
| Field extraction | `of`                                                        |
| Logic            | `and`, `or`, `not`                                          |
| Control          | `if`, `then`, `else`, `case`, `when`                        |
| Environment      | `env`                                                       |
| Import           | `struct`                                                    |
| Membership       | `in`                                                        |
| Function         | `function`, `returns`, `default`                            |
| Reserved         | `lazy`                                                      |

### 2.6 Operators and Punctuation

```
+   -   *   /   %        // arithmetic
^                        // exponentiation
<   <=  >   >=           // comparison
++                       // concatenation
**                       // repetition
,                        // separator
{   }                    // struct delimiters
(   )                    // tuple delimiters / grouping
[   ]                    // list delimiters
.                        // member access
"                        // string delimiter
'                        // quoted identifier delimiter
@                        // keyword escape
```

---

## 3. Types

### 3.1 Primitive Types

| Type     | Description                                  | Example             |
| -------- | -------------------------------------------- | ------------------- |
| `i{N}`   | Signed integer, N bits (N: 0–65535)          | `i8`, `i32`, `i64`  |
| `u{N}`   | Unsigned integer, N bits (N: 0–65535)        | `u8`, `u16`, `u256` |
| `f16`    | 16-bit float (IEEE 754 half-precision)       |                     |
| `f32`    | 32-bit float (IEEE 754 single-precision)     |                     |
| `f64`    | 64-bit float (IEEE 754 double-precision)     |                     |
| `f80`    | 80-bit float (x86 extended precision)        |                     |
| `f128`   | 128-bit float (IEEE 754 quadruple-precision) |                     |
| `bool`   | Boolean                                      | `true`, `false`     |
| `string` | UTF-8 string                                 | `"hello"`           |
| `null`   | Absence of value                             | `null`              |

**Integer width semantics**: An `iN` or `uN` type can hold `2^N` distinct values. This formula holds uniformly for all `N` in the range, including `N = 0`: `u0` and `i0` are unit types holding exactly one value (`0`). Zero-width integers carry no information and are rarely useful in configuration, but they are permitted for consistency with the general integer type rule.

**`null` vs `undefined`**: `null` is a value — it can be assigned to a binding to indicate "intentionally empty." `undefined` is not a value — it is a **state** that results from querying something that does not exist. The literal `undefined` cannot appear on the right side of `is` in a binding — this is a **type error** (§4.5). However, expressions that produce `undefined` (e.g., `missing`, `env.UNSET`) can — the `undefined` state flows through the dependency graph until it is either resolved by `or else` or reaches a terminal context where it becomes an error.

```uzon
x is null          // valid — x exists, holds no value
y is undefined     // INVALID — literal undefined cannot be assigned

// But expressions producing undefined CAN be bound:
a is missing          // valid — a holds undefined state
b is a or else 1      // valid — b is 1 (undefined resolved by or else)
c is "value: {a}"     // INVALID — undefined reached string interpolation
```

The following table defines how `undefined` interacts with every operator:

| Operator                     | `undefined` Behavior     | Rationale                                                        |
| ---------------------------- | ------------------------ | ---------------------------------------------------------------- |
| `.` (member access)          | Propagates               | Implicit optional chaining                                       |
| `to` (conversion)            | Propagates               | Enables `env.X to u16 or else 8080`                              |
| `with`                       | Runtime error            | Shape preservation requires a concrete struct                    |
| `plus`                    | Runtime error            | Extension requires a concrete struct                             |
| `as` (annotation)            | Propagates               | Type name is still validated; undefined value propagates         |
| `or else`                    | Returns right operand    | Primary undefined resolution mechanism                           |
| `is` / `is not`              | Permitted (returns bool) | Enables `x is undefined` checks                                  |
| `+`, `-`, `*`, `/`, `%`, `^` | Runtime error            | Arithmetic on undefined is meaningless                           |
| `**` (repetition)            | Runtime error            | Repetition requires concrete operands                            |
| `++` (concatenation)         | Runtime error            | Concatenation requires concrete operands                         |
| `<`, `<=`, `>`, `>=`         | Runtime error            | Ordering undefined is meaningless                                |
| `and`, `or`, `not`           | Runtime error            | Requires bool, undefined is not bool                             |
| `in`                         | Runtime error            | Membership requires concrete operands (both left and right)      |
| `is named`                   | Runtime error            | Requires tagged union, not undefined                             |
| `is type` / `is not type`    | Runtime error            | Requires a concrete value to check type                          |
| String interpolation         | Runtime error            | Terminal context — must resolve first                            |
| `case` match value           | Runtime error            | Cannot match against undefined                                   |
| `()` (function call)         | Runtime error            | Calling undefined is meaningless; undefined as argument is error |

### 3.2 Struct

A struct is a collection of named fields. Each field is a binding using `is`.

```uzon
star is { name is "Sirius", magnitude is -1.46, visible is true }

// Multi-line
server is {
    host is "localhost"
    port is 8080
    debug is true
}
```

Struct fields are accessed via dot notation: `star.name`.

**Quoted keys**: Field names may use quoted identifiers (§2.3) to allow characters that are not valid in regular identifiers. This ensures lossless representation of formats like JSON where keys are arbitrary strings.

```uzon
headers is {
    'Content-Type' is "application/json"
    'X-Request-ID' is "abc123"
    '' is "empty key"
    normal_key is "no quotes needed"
}
```

**Key identity**: A key's identity is its name. The identifier `key` and the quoted form `'key'` refer to the same name — they are interchangeable in definitions, access, and duplicate checking. `{ key is 1, 'key' is 2 }` is a duplicate error.

```uzon
config is { port is 8080 }
config.port          // 8080
config.'port'        // 8080 — same key, different notation
```

Duplicate field names within a struct are **forbidden**; parsers **MUST** reject them. More broadly, duplicate bindings or type names within the same scope are **forbidden** — this applies to structs, file-level bindings, and type names declared with `called`.

```uzon
// INVALID — duplicate binding in same scope
x is 1
x is 2   // error: x already declared in this scope

// INVALID — duplicate type name
a is x from x, y, z called Letters
b is p from p, q, r called Letters   // error: Letters already declared
```

#### 3.2.1 Struct Override (`with`)

The `with` keyword creates a new struct by copying an existing struct and overriding specific fields. Fields not mentioned in the override are preserved from the original.

```uzon
base is {
    host is "localhost"
    port is 8080
    debug is false
    log_level is info from trace, debug, info, warn, error called LogLevel
}

development is base with { debug is true }
// { host is "localhost", port is 8080, debug is true, log_level is info }

production is base with { port is 443, log_level is warn as base.LogLevel }
// { host is "localhost", port is 443, debug is false, log_level is warn }
```

`with` produces a new struct — the original is not modified. The override struct may only contain fields that exist in the original; adding new fields that are not in the original is an error. The overridden fields **MUST** be type-compatible with the original fields — changing the type of a field (e.g., from `u16` to `string`) is a type error. An override expression that evaluates to `undefined` is a runtime error — `with` preserves the shape and types of the original struct; to remove a field, define a new struct instead.

**Type compatibility** for `with` overrides is defined as follows: (1) typed values must have the **same type** — `u8` is compatible with `u8`, but not with `u16` or `i32`; (2) an unannotated literal adopts the original field's type per the untyped literal compatibility rule (§5) — `100` overriding a `u8` field adopts `u8` and is range-checked; (3) `null` is bidirectionally compatible with any type (see below); (4) anonymous structs are structurally compatible if they have the **same field names and the same field types** — field order does not matter, extra or missing fields are errors; (5) named types declared with `called` must match by name — two separately `called` types with identical structure are **not** compatible. When the base struct has a named type (via `called`), the result of `with` preserves that type — the override fields are checked against the named type's field definitions.

```uzon
base is { x is 0 as i32, y is 0 as i32 } called Point

base with { x is 10 }              // ✓ — 10 adopts i32 (rule 2)
base with { x is 10 as i32 }       // ✓ — same type (rule 1)
base with { x is 10 as i64 }       // INVALID — i64 ≠ i32 (rule 1)
base with { x is null }            // ✓ — null compatible (rule 3)
```

**Exception for `null`**: `null` receives special treatment in `with` overrides. A field with value `null` (without explicit type annotation) may be overridden by a value of any type. Conversely, any typed field may be overridden with `null` to reset it to an empty state. This bidirectional `null` compatibility enables `null` as both a placeholder (type to be determined) and a reset mechanism (clear a previously set value). When the struct has a named type (via `called`), the named type's field definitions take priority — a `null` field that was originally typed as `i32` in the named type can only be overridden with `i32` or `null`, not with an unrelated type like `string`.

```uzon
base is { api_key is null, port is 8080 as u16 }

prod is base with { api_key is "secret_token" }    // valid — null → string
local is prod with { api_key is null }             // valid — string → null (reset)
bad is base with { port is "8080" }                // INVALID — u16 → string (not null)
```

```uzon
// INVALID — 'tls' does not exist in base
staging is base with { tls is true }   // error: unknown field 'tls'
```

This restriction is intentional: `with` permits only safe copy-and-update of an immutable struct. By disallowing arbitrary field addition, the original struct's shape is preserved and type safety is guaranteed. If a different shape is needed, define a new struct.

**No implicit deep merge**: Overriding a nested struct replaces it entirely — the new value must match the full shape and types of the original nested struct. UZON does not recursively merge nested fields. To deeply update a nested struct, use an intermediate binding:

```uzon
base is { inner is { a is 1, b is 2 } }

// INVALID — { a is 3 } has different shape than { a is 1, b is 2 }
bad is base with { inner is { a is 3 } }

// CORRECT — chain with on the nested struct
good is base with { inner is base.inner with { a is 3 } }
// good.inner = { a is 3, b is 2 }
```

The override block in `with` is a struct literal evaluated in the **calling scope** — not inside the base struct's scope. Identifiers inside the override resolve using the normal lexical scope chain from where `with` appears. Chaining multiple `with` operations (e.g., `a with {...} with {...}`) is not permitted; use an intermediate binding instead.

**Value-copy semantics**: `with` operates on the **evaluated values** of the base struct, not its expressions. The base struct is fully evaluated first, then the override fields replace values in the copy. Expressions in non-overridden fields are not re-evaluated.

```uzon
base is { x is 1, y is x + 10 }
// base is evaluated: { x is 1, y is 11 }

modified is base with { x is 2 }
// modified.y is 11, NOT 12 — y was already evaluated as a value
```

The override block itself is evaluated in the calling scope:

```uzon
y is 10
base is { x is 1 }

z is base with { x is y }   // y → calling scope → 10
// z = { x is 10 }
```

**Error classification for `with`**:

| Condition                                    | Error type    |
| -------------------------------------------- | ------------- |
| New field not in original struct             | Type error    |
| Overridden field type incompatible           | Type error    |
| Override expression evaluates to `undefined` | Runtime error |
| Chaining (`a with { } with { }`)             | Syntax error  |
| Base expression is not a struct              | Type error    |
| Base expression evaluates to `undefined`     | Runtime error |

#### 3.2.2 Struct Extension (`plus`)

The `plus` keyword creates a new struct by copying an existing struct, adding new fields, and optionally overriding existing fields. The extension block **MUST** contain at least one field that does not exist in the base struct — if all fields already exist, use `with` instead. Unlike `with` (which preserves the original struct's shape), `plus` always grows the structure.

```uzon
base is { host is "localhost", port is 8080 } called Base

// plus — override existing field and add new ones
secure is base plus { port is 443, tls is true, cert is "/path" }
// { host is "localhost", port is 443, tls is true, cert is "/path" }

// plus — add only (no override)
tagged is base plus { env is "production" }
// { host is "localhost", port is 8080, env is "production" }

// INVALID — no new fields (use with instead)
alt is base plus { port is 9090 }   // error: plus must add at least one new field
```

`plus` produces a new struct — the original is not modified. Overridden fields **MUST** be type-compatible with the original fields, following the same rules as `with` (§3.2.1): same type required, untyped literals adopt the original field's type, `null` is bidirectionally compatible. The result of `plus` is always a **new type** — even if the base had a named type via `called`, the extended struct does not inherit that type name (it has a different shape). Use `called` on the result to name the new type.

```uzon
base is { x is 0 as i32, y is 0 as i32 } called Point

extended is base plus { z is 0 as i32 } called Point3D
// Point3D has fields x, y, z — it is NOT a Point

extended as Point    // INVALID — different shape (3 fields vs 2)
```

**No chaining**: `a plus { } plus { }` is not permitted — use an intermediate binding. The override block in `plus` is evaluated in the **calling scope**, consistent with `with`. `plus` follows the same value-copy semantics as `with` (§3.2.1) — the base is fully evaluated first, then overrides and additions are applied.

**Error classification for `plus`**:

| Condition                                 | Error type    |
| ----------------------------------------- | ------------- |
| No new fields (all fields exist in base)  | Type error    |
| Overridden field type incompatible        | Type error    |
| Overridden field evaluates to `undefined` | Runtime error |
| New field evaluates to `undefined`        | Runtime error |
| Base expression is not a struct           | Type error    |
| Chaining (`a plus { } plus { }`)    | Syntax error  |
| Base expression evaluates to `undefined`  | Runtime error |

**`plus` vs `with`**:

| Property        | `with`                      | `plus`                 |
| --------------- | --------------------------- | ------------------------- |
| Override fields | ✓                           | ✓ (optional)              |
| Add new fields  | ✗ (error)                   | ✓ (at least one required) |
| Shape preserved | ✓ (same fields as original) | ✗ (always more fields)    |
| Type preserved  | ✓ (named type carried over) | ✗ (always a new type)     |
| Use case        | Safe copy-and-update        | Template-based extension  |

### 3.3 Tuple

A tuple is a fixed-length, **heterogeneous** ordered sequence. Tuples use parentheses `()`. Unlike lists, tuple elements may have different types — this is the fundamental distinction between the two.

```uzon
star is (6.646, "Sirius", true)       // distance (ly), name, visible — valid tuple
pixel is (255, 128, 0)                // same types — also valid

// Nested
orbits is ((1.0, "Mercury"), (5.2, "Jupiter"))

// Empty tuple
unit is ()
```

|               | List `[]`                       | Tuple `()`                              |
| ------------- | ------------------------------- | --------------------------------------- |
| Element types | **Homogeneous** — all same type | **Heterogeneous** — may differ          |
| Length        | Variable                        | Fixed at declaration                    |
| Use case      | Collections of similar items    | Records of related but different values |

Tuples are syntactically distinct from structs (`{}`) and lists (`[]`). Parentheses are shared between tuples and grouping expressions. The presence of a comma distinguishes them: `(expr)` is grouping, `(expr,)` is a 1-element tuple, and `(expr, expr, ...)` is a multi-element tuple. `()` is an empty tuple. The same rules apply in type position: `(i32)` is grouping, `(i32,)` is a 1-element tuple type, `(i32, string)` is a 2-element tuple type, and `()` is the empty tuple type.

Tuple elements are accessed using the same dot notation as lists (§3.4.2):

```uzon
pixel is (255, 128, 0)
pixel.0             // 255
pixel.1             // 128
pixel.first         // 255
pixel.second        // 128
```

### 3.4 List

A list is a variable-length, ordered sequence of **same-typed** elements. Lists use square brackets. All elements in a list **MUST** be the same type — mixing types is an error.

`null` may appear as a list element. When a list contains both `null` and non-`null` elements, the element type is inferred from the non-`null` elements — `null` adopts that type. When **all** elements are `null`, an explicit `as [Type]` annotation is **REQUIRED**, since no type can be inferred.

```uzon
primes is [ 2, 3, 5, 7, 11 ]
planets are "Mercury", "Venus", "Earth", "Mars"
empty is [] as [i32]

[ 1, null, 3 ]                   // valid — null adopts integer type
[ null, "hello", null ]          // valid — null adopts string type
[ null ]                         // INVALID — no type info, as [Type] required
[ null, null ]                   // INVALID — no type info, as [Type] required
[ null ] as [i32]                // valid — null in i32 list
[ null, null ] as [string]       // valid — null in string list

// INVALID — mixed types in a list
bad is [ 1, "hello", true ]   // error: all elements must be the same type
```

**Type annotation** for lists uses `as [Type]`:

```uzon
ids is [ 1, 2, 3 ] as [i32]
tags is [ "a", "b", "c" ] as [string]
```

For heterogeneous collections of fixed length, use a tuple (§3.3) instead.

When an empty list `[]` appears in an expression where its type can be immediately inferred (e.g., concatenation with a typed list), the element type is inferred from context. When assigned directly to a binding without an immediate inference context, an explicit `as [Type]` annotation is **REQUIRED**. The evaluator does not perform cross-binding global type inference.

```uzon
typed_empty is [] as [i32]                     // explicit annotation — required for standalone binding
combined is [] ++ [ 1, 2, 3 ]                  // valid — type inferred immediately from concatenation
deferred is []                                 // INVALID — no immediate context, annotation required
branched is if cond then [] else [ 1, 2, 3 ]   // valid — inferred from the other branch
membership is 5 in []                          // valid — type inferred from left operand (§5.8.1)
both_empty is [] ++ []                         // INVALID — neither side has type information
```

#### 3.4.1 The `are` Keyword

The `are` keyword provides syntactic sugar for list binding, allowing the square brackets to be omitted. The following pairs are equivalent:

```uzon
layouts is [ tiled, monocle, floating ]
layouts are tiled, monocle, floating

ids is [ 1, 2, 3 ] as [i32]
ids are 1, 2, 3 as [i32]

names is [ "alfa", "bravo", "charlie" ] as [string]
names are "alfa", "bravo", "charlie" as [string]
```

`are` reads naturally as English ("layouts are tiled, monocle, floating"), making list declarations more readable. Note that `are` only replaces `is [...]` at the binding level — it cannot be used in nested expressions. Elements in an `are` binding are separated by commas. The binding naturally continues across multiple lines until a new binding (e.g., `name is`) begins.

Each element in an `are` binding is a full expression — including arithmetic, `or else`, `with`, and all other operators. This enables concise lists of computed values and struct variants:

```uzon
base is { x is 1, y is 2 } called Config
configs are base, base with { x is 10 }, base with { y is 20 }
// = [ { x is 1, y is 2 }, { x is 10, y is 2 }, { x is 1, y is 20 } ]
```

**`as` disambiguation in `are` bindings**: A trailing `as` at the end of an `are` binding is the **list-level type annotation** — it is part of the binding syntax, not part of the last element. To annotate the last element individually, use parentheses.

```uzon
ids are 1, 2, 3 as [i32]                        // 3 as [i32] → lifted → list is [ i32 ]
ids are 1, 2, 3 as MyListType                   // lifted → list is MyListType
ids are 1, 2, (3 as i32)                        // parentheses → element-level: 3 is i32
numbers are -1, 1 + 1, 3 * 2                    // full expressions — no parentheses needed
ports are 8080, env.PORT to u16 or else 8081    // or else works naturally
```

**Warning**: The list-level `as` annotation applies regardless of the type form. For example, `ids are 1, 2, 3 as i32` annotates the entire list as `i32` — a list annotated as an integer, which is a type error. Use parentheses to annotate the last element individually: `ids are 1, 2, (3 as i32)`.

**Named list types**: When `called` appears on an `are` binding, it names the resulting list type. Other bindings may reference this type with `as`. Conformance requires the same element type.

```uzon
primary is red from red, green, blue called RGB

colors are red, green, blue as [RGB] called Colors
// Colors is a named type for [RGB]

palette are red, blue as Colors         // valid — same element type
nums are 1, 2, 3 called Numbers         // Numbers is a named type for [ i64 ]
more are 4, 5, 6 as Numbers             // valid — same element type
bad are "a", "b" as Numbers             // INVALID — string ≠ i64
```

#### 3.4.2 Element Access

List and tuple elements are accessed using dot notation. UZON provides two forms:

**Numeric indices** — 0-based, unlimited:

```uzon
scores is [ 97, 85, 92 ]

scores.0        // 97
scores.1        // 85
scores.2        // 92
```

**Named ordinals** — `first` through `tenth`, mapped to indices 0–9:

```uzon
scores.first    // 97  (= .0)
scores.second   // 85  (= .1)
scores.third    // 92  (= .2)
```

The full set of named ordinals:

| Ordinal   | Index |
| --------- | ----- |
| `first`   | 0     |
| `second`  | 1     |
| `third`   | 2     |
| `fourth`  | 3     |
| `fifth`   | 4     |
| `sixth`   | 5     |
| `seventh` | 6     |
| `eighth`  | 7     |
| `ninth`   | 8     |
| `tenth`   | 9     |

Accessing an index beyond the length of the list or tuple evaluates to `undefined`. This is consistent with struct field access — accessing a nonexistent name also yields `undefined` (§5.12). Negative indices are not supported — they are treated as out-of-bounds and evaluate to `undefined`. More generally, if a member accessor does not apply to the value's type (e.g., a numeric index on a struct, or a non-ordinal name on a list), the result is `undefined` — consistent with any other unresolved member access.

**Rationale.** UZON provides two forms of element access for lists and tuples:

- **Numeric indices (0-based, unlimited)** — Most programming languages use 0-based indexing. Since UZON's primary users are programmers, diverging from this convention would introduce unnecessary confusion.

- **Named ordinals (`first` through `tenth`)** — Data is easier to reason about when it reads like language rather than arithmetic. `servers.first` communicates intent more clearly than `servers.0`. Following Common Lisp's precedent of `first` through `tenth`, UZON provides natural-word aliases for the first ten positions. Beyond ten, numeric indices are available — and if positional access goes that deep, the data likely wants named fields in a struct instead.

These ordinals are **not keywords**. They have no special meaning outside of member access on lists and tuples. `first is "Alice"` is a valid binding — the identifier `first` is only interpreted as an index when it follows `.` on a list or tuple value.

```uzon
// first as a regular identifier — no conflict
first is "Mercury"
winner is first            // "Mercury" (struct-level binding)

// first as list accessor — context determines meaning
scores is [ 97, 85, 92 ]
top is scores.first        // 97 (list element access)
```

### 3.5 Enum

An enum defines a set of named variants. The value is one of the variants, and the variant set is declared with `from`. An enum **MUST** have at least two variants — a single-variant enum carries no useful information. Duplicate variant names are a **type error** — each variant must be unique.

```uzon
e is green from red, green, blue
```

Here `e` takes the value `green` from the set `{red, green, blue}`.

**Named enum types** use `called` for reuse:

```uzon
primary is red from red, green, blue called RGB
secondary is blue as RGB   // reuse the RGB type
```

**Variant resolution**: Bare identifiers in expression position resolve as **bindings** via the lexical scope chain (§5.12). Enum variants are resolved only in explicit contexts — there is no ambiguity between bindings and variants:

1. **`from` clauses** — identifiers after `from` are always variants.
2. **`as EnumType`** — a bare identifier immediately before `as EnumType` resolves as a variant of that type.
3. **`case`/`when`** — bare identifiers in `when` clauses resolve as variants of the `case` scrutinee's enum type (§5.10).
4. **Type-context inference** — when the expected type from context is a named enum type, a bare identifier matching a variant of that type resolves as that variant. Type context propagates through: list element type via `as [EnumType]`, `if`/`case` branch unification, `or else` right operand, `in` operator left operand (inferred from the list's element type — this inference applies only to lists, not to tuples or structs whose values are heterogeneously typed), and `is`/`is not` operand (inferred from the other operand).

Outside these contexts, a bare identifier is always a binding reference. If a bare identifier matches variants in multiple visible enum types, `as EnumType` is **required** to disambiguate.

```uzon
layouts is {
    tile is { gap is 12, padding is 9 }
    monocle is { }
    default_layout is tile from tile, monocle called Layout
    // "tile" before "from" = variant, not the struct binding above

    active is tile as Layout
    // "tile" before "as Layout" = variant, not the struct binding
    // Without this rule, "tile" would resolve to the struct { gap is 12, ... }
}
```

**Enum termination**: The `from` clause consumes comma-separated identifiers as variants. Consumption stops when any of the following conditions is met:

1. **Newline** — a NEWLINE_SEP triggers (next line starts with non-keyword identifier + `is`/`are`).
2. **Comma lookahead** — after a `,`, the next two tokens are a non-keyword identifier followed by `is` or `are`. The identifier begins a new binding, not a variant.
3. **`called`** — the `called` keyword explicitly closes the enum and names the type.
4. **End of expression** — a closing delimiter (`}`, `)`, `]`) or EOF.

This comma-lookahead rule reuses the same 2-token pattern as NEWLINE_SEP — the only difference is the trigger (`,` instead of newline). Keywords like `named` and `from` are not treated as binding starts, so they are consumed as variants when they appear after `,`. The exception is `called`, which always closes the enum (rule 3) — use `@called` to include it as a variant name. **Variant name / binding name collision**: When a variant name coincidentally matches the next binding name, the 2-token lookahead terminates the variant list. For example, `size is active from active, size` followed by `size is 10` on the next line — the lookahead sees `size is` and terminates after `active` (1 variant → rejected as < 2). To avoid this, use `called` to explicitly close the enum: `size is active from active, size called SizeMode`. The `called` keyword unambiguously terminates the variant list regardless of what follows.

Similarly, bindings with keyword names on the following line (e.g., `@true is 5`) require `@` escape — without it, the keyword is consumed as a variant because NEWLINE_SEP only triggers for non-keyword identifiers.

```uzon
// Now correctly parsed — size is 10 is a separate binding
{ color is red from red, green, blue, size is 10 }
// color = red from {red, green, blue, size=variant? NO — "size is" triggers termination}
// color = red from {red, green, blue}, size = 10

// Newline termination still works
{
    color is red from red, green, blue
    size is 10
}

// called explicitly terminates
{ color is red from red, green, blue called RGB, size is 10 }

// Keywords as variants — not affected (named is a keyword, not a binding start)
x is a from a, named, b    // three variants: a, named, b
```

```uzon
green is 1
e is green from red, green, blue   // green = variant, not the binding (1)
```

### 3.6 Union

A union holds a value whose type is one of several possible types. The `union` keyword after `from` distinguishes it from an enum.

A union **MUST** have at least two member types — a single-type union is equivalent to a type annotation and carries no useful information.

```uzon
u is 3.14 from union i32, f64, string
```

Here `u` holds the value `3.14`, and its type is one of `i32`, `f64`, or `string`. The parser/runtime determines the active type from the value. Duplicate member types within an untagged union are a **type error** — each type must be unique (e.g., `from union i32, i32, string` is invalid).

Untagged unions can be discriminated using `is type` and `is not type` operators, which check the runtime type of the inner value:

```uzon
port is 8080 from union i32, string

is_numeric is port is type i32        // true
is_text is port is type string        // false
```

For multi-branch dispatch, use `case type`:

```uzon
port is 8080 from union i32, string

display is case type port
    when i32 then "port {port}"
    when string then port
    else "unknown"
```

Using `is type` or `is not type` on a non-union value is permitted — it checks the value's concrete type and returns `bool`. The type expression after `type` **MUST** be a valid type name. All branches in a `case type` expression **MUST** produce the same result type, consistent with `case` and `case named`.

**Union type identity**: Two anonymous union types are the **same type** if and only if they have the same set of member types — declaration order does not matter. `from union i32, string` and `from union string, i32` are the same type. Named union types (via `called`) follow nominal identity — two separately `called` union types with the same member set are different types, consistent with named structs and named enums.

**Untagged union equality**: `is` and `is not` require both operands to be the same union type (same member type set, or same named type). Comparing unions with different member type sets is a **type error**. When two values of the same union type are compared, the evaluator checks their runtime types first — if the runtime types differ, the result is `false` (not a type error). If the runtime types match, deep value comparison is performed using the same rules as §5.2. Comparing an untagged union with a non-union value follows transparency — the inner value is compared directly.

```uzon
a is 42 from union i32, string
b is "hi" from union string, i32     // same type (order irrelevant)
c is 42 from union i32, f64          // different type (different member set)

check1 is a is b                      // false (runtime types differ: i32 vs string)
check2 is a is 42                     // true (transparency — inner value compared)
// a is c                             // INVALID — type error (different union types)
```

Ordered comparison (`<`, `<=`, `>`, `>=`) on untagged union values is a **type error** — union types have no defined ordering.

**Untagged union and `to`**: The `to` operator is governed by the conversion table (§5.11.0), not by general transparency. Only `to string` is permitted for untagged unions — all other target types are type errors. This is consistent with tagged union `to` behavior (§3.7.1).

#### 3.6.1 Enum vs Union Disambiguation

Both enums and unions use the `from` keyword. The distinction is **purely syntactic** — determined by the presence of `union`:

- `from` followed by bare identifiers → **enum**
- `from union` followed by types → **union**

```uzon
e is green from red, green, blue              // enum — bare identifiers
u is 3.14 from union i32, f64, string         // union — from union
tu is 7 named ln from n as i32, ln as i128    // tagged union — name of type
```

Because the parser only checks for the `union` keyword — not whether an identifier is a known type — there is no risk of namespace pollution. User-defined type names cannot accidentally change how an enum is parsed.

### 3.7 Tagged Union

A tagged union pairs a value with an explicit variant tag. It uses `named` to specify which variant is active. A tagged union **MUST** have at least two variants — a single-variant tagged union offers no dispatch capability and carries no useful information. Duplicate variant names are a **type error** — each variant must be unique.

```uzon
tu is 7 named ln from n as i32, ln as i128, f as f80
```

Here:
- The value is `7`
- The active variant is `ln` (of type `i128`)
- The possible variants are: `n` as `i32`, `ln` as `i128`, `f` as `f80`

Syntax:

```
<value> named <active_variant> from <variant₁> as <type₁>, <variant₂> as <type₂>, …
```

#### 3.7.1 Inner Value

When a tagged union is used as a value, it evaluates to its **inner value** transparently for most operators — arithmetic, concatenation, string interpolation, and comparison with non-tagged-union values all see the inner value. The tag is metadata and does not appear in the value itself. However, several operators see the tagged union **wrapper** instead (see exceptions below).

**Transparency exceptions**: The following operators see the tagged union wrapper, not the inner value: `is` / `is not` (compare both tag and inner value — §3.7.2), `is named` / `is not named` (check tag), `case named` (match tag in `case`), `to` (conversion applies to the tagged union as a unit — §5.11.0), and `with`/`plus` (requires a struct, not a tagged union — apply to the inner struct explicitly). All other operators (arithmetic, concatenation, comparison with non-tagged-union values, string interpolation) see the inner value. **Binding** (`is` as assignment) is not an operator — it preserves the tagged union wrapper. To extract the inner value, use a transparent operator context: `tagged_val + 0` for numeric, `tagged_val ++ ""` for string. Member access (`.`) is transparent — `tagged_val.field` accesses the inner struct's fields, `tagged_val.0` accesses tuple elements, and `tagged_val.length` accesses list properties, all without extraction.

*Note on `to` and tagged unions*: Because `to` sees the wrapper, conversions are governed by the tagged union row in the conversion table (§5.11.0). Only `to string` is permitted — it extracts the inner value and converts it to string per §5.11.2. All other target types (`to i32`, `to f64`, etc.) are type errors. To convert a tagged union's inner value to a non-string type, extract it via a transparent operator first (e.g., `inner is tu + 0` for numeric, `inner is tu ++ ""` for string), then apply `to`.

```uzon
health is "all systems go" named ok from
    ok as string, degraded as string, down as null
    called HealthStatus

msg is "Status: {health}"   // "Status: all systems go"
x is health ++ "!"          // "all systems go!"
```

#### 3.7.2 Variant Comparison (`is named` / `is not named`)

To check the active variant of a tagged union, use `is named` or `is not named`:

```uzon
health is named ok          // true
health is named down        // false
health is not named ok      // false
```

**Equality between tagged unions**: When comparing two tagged union values with `is` or `is not`, the evaluator **MUST** compare **both** the variant tag and the inner value. Two tagged unions are equal only if they have the same tag AND the same inner value. The transparent inner-value evaluation (§3.7.1) applies when a tagged union is used in non-comparison contexts (arithmetic, concatenation, interpolation), but not for structural equality checks between two tagged unions.

```uzon
r1 is "error" named err from err as string, ok as string called Result
r2 is "error" as Result named ok
r1 is r2               // false — same inner value but different tags
```

In `case` expressions, use `case named`:

```uzon
summary is case named health
    when ok then "OK: {health}"
    when degraded then "WARN: {health}"
    when down then "DOWN"
    else "unknown"
```

Using `is named` or `is not named` on a value that is not a tagged union is a type error. If the name after `named` is not a valid variant of the tagged union type, the evaluator **MUST** report a type error. In `case named` expressions, **all** `when` clauses **MUST** be validated against the tagged union's variant list, regardless of which clause matches — consistent with the speculative evaluation rule (§5.9).

**Comparing a tagged union with a non-tagged-union value** using `is` or `is not` is a type error — they are different types. Exception: comparing against `null` or `undefined` is always permitted (§5.2), consistent with the universal null/undefined comparison rule. Since `is`/`is not` are transparency exceptions (§3.7.1), the wrapper is not unwrapped for comparison. To compare the inner value, extract it via a transparent operator first (e.g., `inner is tu + 0` for numeric, `inner is tu ++ ""` for string), then compare `inner`.

```uzon
message is "hello"
check is message is named ok   // INVALID — string is not a tagged union
```

### 3.8 Function

A function is a **value** — just as `42` is an integer value, `{ }` is a struct value, and `[1, 2]` is a list value, a function is a function value. It can be bound to a name with `is`, stored in a list, passed as an argument, and annotated with `called` to name its type.

A function takes zero or more typed parameters and evaluates to a single result. The `returns` keyword specifies the result type, and the body is enclosed in `{ }`.

```uzon
isEven is function n as i32 returns bool { n % 2 is 0 }
```

**Syntax:**

```
function [<param₁> as <type₁>, <param₂> as <type₂>, …] returns <type> { <body> }
```

A zero-parameter function is useful for deferred computation — it captures the lexical scope and `env` at definition time but evaluates the body only when called:

```uzon
getPort is function returns u16 {
    env.PORT to u16 or else 8080
}

port is getPort()
```

**Parameter access**: Parameters are accessed as bare identifiers inside the function body — there is no special prefix needed. Parameters are part of the function body's scope layer, so they are visible to nested structs and string interpolation without intermediate bindings.

**Self-exclusion in functions**: The self-exclusion rule (§5.12) applies to function bindings. When a function references `x` inside its body and the function itself is bound to `x`, the lookup skips the function binding and searches parent scopes — the same behavior as `x is x or else 1` for regular bindings. This is not recursion (which is forbidden) — it is scope chain traversal. Self-exclusion is most useful in nested scopes where a binding in an inner scope shares a name with one in an outer scope:

```uzon
base is 10
add is function n as i32 returns i32 { n + base }
// n = bare (parameter), base = already declared (outer binding)

// self-exclusion in nested scope
outer is {
    x is 99
    inner is {
        x is function n as i32 returns i32 { n + x }
        // x → self-exclusion skips inner.x → finds outer.x = 99
    }
}
result is outer.inner.x(1)    // 1 + 99 = 100
```

**Expression boundaries in function bodies**: Because expression continuation (§8) consumes binary operators across lines, a function body's return expression cannot begin with `-`, `+`, or any binary operator on a new line — it would be consumed as a continuation of the previous binding's value. To return a negative value, use an intermediate binding or parentheses:

```uzon
// WRONG — parses as a is 5 - 3 = 2, no return expression
f is function returns i32 {
    a is 5
    -3
}

// CORRECT — use intermediate binding
f is function returns i32 {
    a is 5
    result is -3
    result
}
```

**Multiline string suppression**: Multiline string concatenation (§4.4.2) is suppressed inside function bodies. Each string literal on its own line is a separate expression, not a continuation of a multiline string. This ensures that the final expression (return value) can be a string literal without being merged with a preceding string binding. To concatenate strings in a function body, use `++`.

```uzon
f is function returns string {
    prefix is "hello"       // binding — string value
    "world"                 // return value — NOT joined with "hello"
}
// f() returns "world", not "hello\nworld"
```

**Function body as scope layer**: A function body forms its own scope layer in the scope chain, consistent with structs. Both parameters and intermediate bindings defined with `is` are part of this scope. Function bodies only support `is` bindings — `are` (list shorthand) and `called` (type naming) are not permitted inside function bodies. Use `is` with list literals and `as` with type annotations instead. When a nested struct is defined inside a function body, identifiers inside that struct resolve by walking: struct scope → function body scope (parameters + local bindings) → enclosing scope → file scope.

```uzon
x is 99
f is function n as i32 returns i32 {
    offset is 10
    inner is {
        a is offset         // struct → function body → found (10)
        b is x              // struct → function body → enclosing → found (99)
        c is n              // struct → function body → found (parameter n)
    }
    inner.a + inner.b + inner.c
}
```

**Parameter types are required**: Every parameter **MUST** have an explicit `as` type annotation. Unlike bindings where untyped literals default to `i64`/`f64`, function parameters define an interface — the caller must know what types to provide.

**Default parameter values**: A parameter may specify a default value using the `default` keyword. When a default is provided, the caller may omit that argument. Parameters with defaults **MUST** appear after all required parameters — a required parameter after a defaulted one is a syntax error.

```uzon
greet is function name as string default "world" returns string {
    "Hello, " ++ name
}

greet()          // "Hello, world"
greet("UZON")    // "Hello, UZON"

connect is function host as string, port as u16 default 8080, tls as bool default false returns string {
    if tls then "https://" ++ host ++ ":" ++ port to string
    else "http://" ++ host ++ ":" ++ port to string
}

connect("example.com")                // "http://example.com:8080"
connect("example.com", 443, true)    // "https://example.com:443"
```

The default value expression is evaluated in the **enclosing scope** — not inside the function body. Identifier references in default expressions resolve against the scope where the function is defined.

```uzon
x is 1
f is function n as i32 default x returns i32 {
    x is 2        // this x is a local binding inside the function body
    n             // n is 1 (from default), NOT 2 — default was evaluated outside
}
// f() → 1
```

**Multi-expression body**: A function body may contain multiple expressions. Intermediate bindings use `is`. The last expression in the body is the return value.

```uzon
clamp is function n as i32, lo as i32, hi as i32 returns i32 {
    clamped_lo is if n < lo then lo else n
    if clamped_lo > hi then hi else clamped_lo
}
```

**Anonymous functions**: A function without a binding name is an anonymous function. It is used inline as a value — most commonly as an argument to `std.map`, `std.reduce`, or `std.filter`.

```uzon
evens is std.filter(numbers, function n as i32 returns bool { n % 2 is 0 })
```

**Named function types**: The `called` keyword names a function's type for reuse. The type captures the parameter types and return type — the function's **signature**. Other functions can be checked against this type with `as`.

```uzon
isEven is function n as i32 returns bool { n % 2 is 0 } called IntPredicate

// Another function with the same signature
isPositive is function n as i32 returns bool { n > 0 } as IntPredicate

// Use in a typed list
predicates is [ isEven, isPositive ] as [IntPredicate]
```

Two function values are **type-compatible** when they have the same parameter types (in order) and the same return type. The parameter names do not matter — only types.

**Structural conformance, nominal identity**: Function types follow the same two-level system as all UZON types. An anonymous function can be checked against a named function type with `as` — this is **structural conformance** (the signatures must match). Once a function type is named with `called`, the name becomes the type's identity — two separately `called` types with identical signatures are **not** the same type. This is **nominal identity**, consistent with structs (§3.2.1 rule 5).

```uzon
isEven is function n as i32 returns bool { n % 2 is 0 } called IntPredicate
isPos is function n as i32 returns bool { n > 0 } as IntPredicate   // structural conformance — signatures match

// Separately named types with same signature are NOT compatible
isOdd is function n as i32 returns bool { n % 2 is not 0 } called OddChecker
isOdd as IntPredicate   // INVALID — OddChecker ≠ IntPredicate (nominal)
```

**No recursion**: Functions may call other functions, but the call graph **MUST** be a directed acyclic graph (DAG). A function **MUST NOT** call itself, whether directly (A→A), through mutual recursion (A→B→A), or through any transitive chain (A→B→C→A). This is checked statically — if a cycle exists in the call graph, it is rejected regardless of whether runtime values would prevent the cycle from being traversed. This guarantees that all function evaluation terminates.

```uzon
// VALID — functions calling other functions (no cycle)
double is function n as i32 returns i32 { n * 2 }
addOne is function n as i32 returns i32 { n + 1 }
transform is function n as i32 returns i32 { addOne(double(n)) }

// INVALID — direct recursion (A→A)
factorial is function n as i32 returns i32 {
    if n is 0 then 1 else n * factorial(n - 1)
}

// INVALID — mutual recursion (A→B→A)
f is function n as i32 returns i32 { g(n) }
g is function n as i32 returns i32 { f(n) }

// INVALID — transitive cycle (A→B→C→A)
a is function n as i32 returns i32 { b(n) }
b is function n as i32 returns i32 { c(n) }
c is function n as i32 returns i32 { a(n) }
```

**No side effects**: Functions are pure. They may read their parameters, reference bindings via the scope chain, and access environment variables via `env`. They cannot modify any binding or perform I/O. Since `env` is read-only, accessing it does not break purity — the same environment always produces the same result.

```uzon
greet is function name as string returns string {
    "Hello from " ++ (env.REGION or else "unknown") ++ ": " ++ name
}
```

**Function equality**: Comparing two function values with `is` or `is not` is a **type error**. Functions have no meaningful value equality — two functions with identical bodies are not guaranteed to be equal, and referential equality has no meaning in an immutable evaluation model.

**Error classification for functions**:

| Condition                                    | Error type   |
| -------------------------------------------- | ------------ |
| Parameter without `as` type annotation       | Syntax error |
| Required parameter after defaulted parameter | Syntax error |
| Recursive call (direct or indirect)          | Circular dependency error |
| Comparing functions with `is` / `is not`     | Type error   |
| Ordering functions with `<` / `<=` / `>` / `>=` | Type error   |
| Calling a non-function value                 | Type error   |
| Wrong argument count                         | Type error   |
| Argument type mismatch                       | Type error   |
| Function body evaluates to wrong return type | Type error   |

---

## 4. Literals

### 4.1 Boolean Literals

```uzon
true
false
```

### 4.2 Integer Literals

Integers may be expressed in decimal, hexadecimal, octal, or binary. Underscores may appear between digits as visual separators and are ignored.

```uzon
42
-7
0xff            // hexadecimal
0o77            // octal
0b1010_0001     // binary with separator
1_000_000       // decimal with separator
```

```ebnf
integer     = [ "-" ] , ( dec_int | hex_int | oct_int | bin_int ) ;
dec_int     = DIGIT , { ( "_" , DIGIT ) | DIGIT } ;
hex_int     = "0" , ("x" | "X") , HEX , { ( "_" , HEX ) | HEX } ;
oct_int     = "0" , ("o" | "O") , OCT , { ( "_" , OCT ) | OCT } ;
bin_int     = "0" , ("b" | "B") , BIN , { ( "_" , BIN ) | BIN } ;
```

### 4.3 Float Literals

Floating-point literals follow IEEE 754 semantics. The special values `inf`, `-inf`, and `nan` are supported.

```uzon
3.14
-0.5
1.0e10
2.5E-3
inf
-inf
nan
```

```ebnf
float       = [ "-" ] , ( float_num | "inf" | "nan" ) ;
float_num   = dec_int , "." , dec_int , [ exponent ]
            | dec_int , exponent ;
exponent    = ("e" | "E") , [ "+" | "-" ] , dec_int ;
```

A numeric literal with a decimal point or exponent is a float; otherwise it is an integer. A bare `inf` or `nan` is a float.

**Default numeric types**: An integer literal without a type annotation (`as`) defaults to **i64**. A float literal without annotation defaults to **f64**. Use `as` to constrain to a specific type:

```uzon
x is 42                                  // i64 by default
y is 42 as u8                            // constrained to u8 (overflow checked)
z is 3.14                                // f64 by default
w is 3.14 as f32                         // constrained to f32
big is 99999999999999999                 // valid — fits in i64 (max ≈ 9.2 × 10¹⁸)
huge is 99999999999999999 as i128        // use as for values beyond i64 range
```

### 4.4 String Literals

Strings are enclosed in double quotes.

```uzon
"hello, world"
"she said \"hi\""
"tab:\there"
"unicode: \u{1F600}"
```

**Escape sequences:**

| Escape       | Meaning                                                                                   |
| ------------ | ----------------------------------------------------------------------------------------- |
| `\\`         | Backslash                                                                                 |
| `\"`         | Double quote                                                                              |
| `\n`         | Line feed (U+000A)                                                                        |
| `\r`         | Carriage return (U+000D)                                                                  |
| `\t`         | Horizontal tab (U+0009)                                                                   |
| `\0`         | Null (U+0000)                                                                             |
| `\{`         | Literal left brace (suppress interpolation)                                               |
| `\xHH`       | Byte value (2 hex digits, 0x00–0x7F only)                                                 |
| `\u{HHHHHH}` | Unicode scalar value (1–6 hex digits, U+0000–U+10FFFF excluding surrogates U+D800–U+DFFF) |

`\xHH` is restricted to the ASCII range (0x00–0x7F) because values above 0x7F would produce invalid UTF-8 sequences as standalone bytes. Use `\u{...}` for Unicode scalar values above U+007F. The `\u{...}` value **MUST** be a valid Unicode scalar value — values above U+10FFFF or in the surrogate range (U+D800–U+DFFF) are errors.

#### 4.4.1 String Interpolation

Curly braces `{expression}` inside a string embed the result of an expression. Any valid UZON expression may appear inside the braces.

```uzon
a is 1
b is 2
c is "{a} + {b} = {a + b}"
// c = "1 + 2 = 3"

name is "UZON"
greeting is "Hello, {name}!"
// greeting = "Hello, UZON!"

port is 8080
url is "http://localhost:{port}/api"
// url = "http://localhost:8080/api"
```

The expression inside `{}` is evaluated and converted to its string representation. Nested braces within the expression (e.g., struct literals) are allowed — the parser matches braces to find the closing `}` of the interpolation.

*Implementer note*: When the lexer encounters `{` inside a string, it enters interpolation mode and lexes a full expression. String literals within the interpolation (e.g., `"hello {"world"}"`) are handled naturally — the inner `"` opens a new string token inside the interpolation expression, and the lexer tracks brace depth to find the matching `}` that closes the interpolation. This requires the lexer to maintain a stack of contexts (string → interpolation → string → …).

Identifiers inside `{}` resolve via the normal scope chain (§5.12):

```uzon
name is "UZON"
greeting is "Hello, {name}!"         // "Hello, UZON!"
```

```uzon
greet is function name as string returns string {
    decorated is "★ " ++ name ++ " ★"
    "Hello, {decorated}!"              // valid — decorated is a local binding
}
```

To include a literal `{` in a string, use the `\{` escape sequence. A literal `}` needs no escaping — without a matching `{`, it is unambiguously a literal character:

```uzon
json_example is "the struct is \{key: value}"
bare_close is "a lonely } is fine"
```

#### 4.4.2 Multiline Strings

Multiline strings are formed by placing consecutive string literals on adjacent lines. Each string literal represents one line; they are automatically joined with newline (`\n`) characters. Unlike C's adjacent string concatenation, UZON inserts a newline between each string. The strings **MUST** appear on immediately adjacent physical lines — blank lines or comment lines between strings break the sequence. A comment between multiline string parts is an error; parsers **MUST** reject it. (This is a specific override of the general rule that comments are whitespace — multiline string adjacency is checked before comment stripping.) This strictness is intentional: if comments or blank lines were allowed between parts, a stray blank line could silently split a string into two separate expressions, creating a difficult-to-diagnose bug.

**Multiline string and NEWLINE_SEP**: Multiline strings are naturally supported by the NEWLINE_SEP rule — since a string literal does not begin a new binding (`name is` or `name are`), newlines between adjacent string literals are treated as whitespace, allowing the multiline string to continue.

```uzon
message is "Hello, world!"
            "This is line two."
            "This is line three."
```

This is equivalent to `"Hello, world!\nThis is line two.\nThis is line three."`.

For concatenation **without** newlines, use the `++` operator explicitly:

```uzon
greeting is "Hello, " ++ "world!"   // "Hello, world!" (no newline)
```

**Exception**: Multiline string concatenation is suppressed inside function bodies (§3.8). Each string literal on its own line inside a function body is a separate expression, ensuring the final expression (return value) can be a string literal without being merged with a preceding string binding.

### 4.5 Null

```uzon
a is null        // value intentionally absent — a exists but holds nothing
```

`null` is a literal value that can be assigned to any binding. It represents "present but empty."

**`null` operator compatibility**: `null` may be used as an operand for `is`, `is not` (§5.2), `or else` (§5.7), `in` (§5.8.1), `if`/`case` branch typing (§5.9, §5.10), and `with` override compatibility (§3.2.1). Using `null` with arithmetic (`+`, `-`, `*`, `/`, `%`, `^`), comparison (`<`, `<=`, `>`, `>=`), concatenation (`++`), repetition (`**`), or logical operators (`and`, `or`, `not`) is a type error. `null to string` produces `"null"` (§5.11.2); `null to null` is permitted (identity); `null to` any other type is a type error.

`undefined` is **not** a literal and cannot be assigned. It is a state returned by the runtime when accessing something that does not exist (e.g., an unset environment variable, or an unbound name in the scope chain). The literal `undefined` is primarily useful in comparison expressions (`x is undefined`). Using `undefined` as the right-hand side of a binding (`x is undefined`) is a **type error** — `undefined` is a state, not a value. In most other operator contexts, `undefined` produces an error — see the propagation table in §3.1 for details.

```uzon
// valid — comparing against undefined
port is if env.PORT is undefined then 8080 else env.PORT to u16

// INVALID — cannot assign undefined literal
x is undefined   // error: undefined cannot be bound directly
```

---

## 5. Expressions

UZON is an expression-oriented language. Almost every construct produces a value.

**Untyped literal compatibility**: An integer literal without an explicit `as` annotation defaults to `i64` but is **adoptable** — when combined with a typed operand, the literal adopts the type of the typed operand instead. The same applies to float literals (default `f64`, adoptable). An integer literal may also adopt a float type when combined with a float operand — `delta * 2` where `delta` is `f64` promotes `2` to `f64`. This cross-category promotion applies only from integer literals to float types, never the reverse. This rule applies **everywhere** two operands are required to be the same type — arithmetic, `or else`, `if`/`case` branch unification, `is`/`is not`, `when` values, and any other same-type constraint. Overflow is still checked at the adopted type's range.

```uzon
3 as u8 + 2                    // 2 adopts u8 → valid
env.PORT to u16 or else 8080   // 8080 adopts u16 → valid
if cond then 42 as i32 else 0  // 0 adopts i32 → valid
```

### 5.1 Binding (`is`)

The `is` keyword serves two roles:

**Assignment** — binds a name to a value:

```uzon
x is 42
name is "UZON"
config is { port is 8080 }
```

**Equality test** — when used in expression context (not at binding position), `is` tests equality and produces a `bool`:

```uzon
g is 1 is 0       // 1 == 0 → false, so g is false
h is 1 is 1       // 1 == 1 → true, so h is true
```

**Disambiguation rule**: The first `is` in a binding statement is always assignment. Subsequent `is` within the value expression are equality tests. **Chaining equality `is` is forbidden** — `a is b is c` is a syntax error. Use parentheses or logical operators to make intent explicit.

```uzon
x is 1 is 1             // x = (1 is 1) → x = true  — OK (one equality)
y is 1 is 1 is 1        // INVALID — chained is, use parentheses
y is (1 is 1) is 1      // explicit — (true is 1) → type error
z is 1 is 1 and 2 is 2  // OK — two separate comparisons joined by and
```

### 5.2 Inequality (`is not`)

```uzon
h is 1 is not 0   // 1 != 0 → true
```

`is not` is a single operator meaning "is not equal to".

Both `is` and `is not` require operands to be the **same type**. Comparing values of different types is an error, with one exception: **comparing any value against `null` or `undefined` is always permitted**, regardless of type. This returns `true` or `false` without raising an error.

```uzon
port is 8080
check_1 is port is not null     // true — permitted (integer vs null)
check_2 is port is undefined    // false — permitted (integer vs undefined)

name is "Suho Kang"
check_3 is name is null        // false — permitted (string vs null)

x is null
check_4 is x is null           // true — same type, trivially permitted
```

This exception exists because checking whether a value is present or absent is a fundamental operation in any configuration format, and requiring type-aware null checks would make the most common pattern — `if x is not null then ...` — impractical.

**NaN comparison**: Float equality and ordering follow IEEE 754 semantics. `nan is nan` evaluates to `false`; `nan is not nan` evaluates to `true`. All ordered comparisons (`<`, `<=`, `>`, `>=`) involving `nan` evaluate to `false`. Consequently, `nan` cannot be matched by a `when` clause in a `case` expression — use a dedicated boolean flag to track NaN-ness. `-nan` is permitted and is semantically identical to `nan` — per IEEE 754, NaN carries a sign bit but it has no meaning.

When comparing compound types (structs, lists, tuples) with `is` or `is not`, the evaluator **MUST** perform **deep structural value comparison** — recursively comparing all fields and elements by value. For structs, field order does not matter — `{ a is 1, b is 2 }` and `{ b is 2, a is 1 }` are equal. Since all UZON values are immutable, there is no concept of reference identity; value equality is the only meaningful comparison. Implementations **MAY** optimize this comparison (e.g., via structural sharing or hash-based shortcuts) as long as the result is equivalent to a full recursive comparison. Comparing compound values with **different structures** — tuples of different length or element types, structs with different field names or types, lists with different element types — is a type error, not `false`.

```uzon
a is [ 1, 2, 3 ]
b is [ 1, 2, 3 ]
a is b          // true — same elements

x is { name is "Sirius", magnitude is -1.46 }
y is { name is "Sirius", magnitude is -1.46 }
x is y          // true — same fields and values
```

**Type checking** (`is type` / `is not type`): The `is type` operator checks whether a value's runtime type matches the specified type. It returns `bool`. `is not type` is the negation. These operators work on any value — they are most useful for untagged union discrimination (§3.6), but can be used on any binding.

```uzon
x is 42
x is type i64          // true (default integer type)
x is type string       // false
x is not type f64      // true

u is "hello" from union i32, string
u is type string       // true
u is type i32          // false
```

For `null`, `null is type null` returns `true`. (To check for `undefined`, use `x is undefined` via the standard equality operator — `undefined` is not a type.) For unions (tagged or untagged), `is type` checks the **inner value's type** — this is the primary use case for untagged union discrimination (§3.6). For tagged unions, `is type` checks the inner value's type, while `is named` checks the variant tag.

### 5.3 Arithmetic Operators

| Operator | Meaning        | Example           |
| -------- | -------------- | ----------------- |
| `+`      | Addition       | `3 + 4` → `7`     |
| `-`      | Subtraction    | `10 - 3` → `7`    |
| `*`      | Multiplication | `2 * 5` → `10`    |
| `/`      | Division       | `10 / 3` → `3`    |
| `%`      | Modulo         | `10 % 3` → `1`    |
| `^`      | Exponentiation | `2 ^ 10` → `1024` |

The `^` operator performs exponentiation. Both operands **MUST** be the same numeric type. The result type is the same as the operand type. For integer exponentiation, the exponent **MUST** be non-negative — a negative exponent would produce a fractional result, which is a runtime error. For float exponentiation, negative exponents are permitted and produce fractional results.

**Integer division and modulo**: Integer division (`/`) truncates toward zero (not toward negative infinity). Modulo (`%`) follows truncation semantics: the result has the same sign as the dividend. This matches Zig, Rust, C, and Java behavior. Float modulo follows the same truncation semantics — the result has the same sign as the dividend (equivalent to C's `fmod`).

```uzon
 10 / 3          // 3 (truncated, not 3.333...)
-10 / 3          // -3 (toward zero, not -4)
 10 % 3          // 1
-10 % 3          // -1 (sign follows dividend)
```

Arithmetic operators require both operands to be the **same numeric type**. There is no implicit type coercion — mixing types (e.g., `i32 + f64`) is an error. Use `to` for explicit conversion when needed. The untyped literal compatibility rule (§5) applies — an untyped literal adopts the type of its typed counterpart.

Numeric overflow is a **runtime error**. If an arithmetic operation produces a result that exceeds the range of the operand type, the evaluator **MUST** report an error. There is no wrapping, saturating, or silent truncation.

**Division by zero**: Integer division or modulo by zero is a runtime error. For float division and modulo, IEEE 754 semantics apply: float division of a nonzero value by zero produces `inf` or `-inf`; float division of zero by zero produces `nan`; float modulo by zero produces `nan`.

If either operand of an arithmetic operator is `undefined`, the evaluator **MUST** report a **runtime error**. Unlike `to` (which propagates `undefined`), arithmetic operations require concrete values — silently computing with `undefined` would produce meaningless results. Use `or else` to provide a fallback before performing arithmetic. As a runtime error, `undefined` in arithmetic is suppressed in non-selected branches during speculative evaluation (§5.9) — this enables patterns like `if x is undefined then 0 else x + 1`.

```uzon
3 + 4                                      // valid — both untyped
3 as i32 + 4 as f64                        // INVALID — type mismatch
3 as i32 + (4 to i32)                      // valid — explicit conversion
3 as u8 + 2                                // valid — 2 adopts u8, result is u8
255 as u8 + 1 as u8                        // INVALID — overflow (u8 max is 255)
env.OFFSET + 8000                          // INVALID if OFFSET is undefined
(env.OFFSET or else "0") to i32 + 8000     // valid — fallback ensures a value
```

### 5.4 Comparison Operators

| Operator | Meaning               | Example            |
| -------- | --------------------- | ------------------ |
| `<`      | Less than             | `1 < 2` → `true`   |
| `<=`     | Less than or equal    | `2 <= 2` → `true`  |
| `>`      | Greater than          | `3 > 2` → `true`   |
| `>=`     | Greater than or equal | `1 >= 2` → `false` |

Comparison operators return `bool`. Both operands **MUST** be the same type. They may be used with integers, floats, and strings. String comparison uses **Unicode scalar value (codepoint) order** — no locale-aware collation is performed. Comparing values of different types is an error. Ordered comparison (`<`, `<=`, `>`, `>=`) on booleans, enums, untagged unions, structs, tuples, lists, or functions is a type error — these types have no defined ordering. For tagged unions, ordered comparison with a non-tagged-union value sees the inner value (transparency, §3.7.1) — the inner type must support ordering. Ordered comparison between two tagged union values is a type error — tags have no defined ordering.

### 5.5 Operator Precedence

Operator and suffix precedence (highest to lowest):

| Precedence  | Operator / Suffix                          | Category                         | Assoc. |
| ----------- | ------------------------------------------ | -------------------------------- | ------ |
| 1 (highest) | `.name`, `.0`, `.first`~`.tenth`, `(args)` | Member access / Function call    | left   |
| 2           | `to`                                       | Type conversion                  | —      |
| 3           | `with`, `plus`                          | Struct override / extension      | —      |
| 4           | `as`                                       | Type annotation                  | —      |
| 5           | `from`, `named`                            | Enum / Union / Tagged union      | —      |
| 6           | `called`                                   | Type naming (binding level only) | —      |
| 7           | `^`                                        | Exponentiation                   | right  |
| 8           | unary `-`                                  | Negation                         | right  |
| 9           | `*`, `/`, `%`, `**`                        | Multiplicative / Repetition      | left   |
| 10          | `+`, `-`                                   | Additive                         | left   |
| 11          | `++`                                       | Concatenation                    | left   |
| 12          | `<`, `<=`, `>`, `>=`                       | Comparison                       | —      |
| 13          | `in`                                       | Membership                       | —      |
| 14          | `is`, `is not`, `is named`, `is not named`, `is type`, `is not type` | Equality / Variant / Type check  | —      |
| 15          | `not`                                      | Logical NOT                      | right  |
| 16          | `and`                                      | Logical AND                      | left   |
| 17          | `or`                                       | Logical OR                       | left   |
| 18 (lowest) | `or else`                                  | Undefined coalescing             | left   |

Operators marked "—" in associativity do not chain (at most one per expression level).

Parentheses `()` may be used to override precedence.

```uzon
// Precedence examples:
config.port to u16 or else 8080
// → (((config).port) to u16) or else 8080

1 + 2 to f64             // → 1 + (2 to f64), NOT (1 + 2) to f64
(1 + 2) to f64           // → use parentheses to convert the sum
(n to string)            // → member access on result requires parens: (n to string).length

red from red, blue called RGB
// → ((red) from red, blue) called RGB
```

### 5.6 Logical Operators

```uzon
true and false           // false
true or false            // true
not true                 // false
not false                // true

1 + 1 is not 2 or 2 * 3 is 6   // true
```

`and` and `or` are short-circuit: `or` stops at the first `true`, `and` stops at the first `false`.

### 5.7 Undefined Coalescing (`or else`)

The `or else` operator provides a fallback value when the left operand is `undefined`. If the left operand has a value (including `null`), it is returned as-is. If it is `undefined`, the right operand is used instead.

```uzon
port is env.PORT to u16 or else 8080
host is env.HOST or else "localhost"
debug is env.DEBUG or else "false"
theme is config.theme or else "dark"
```

This replaces the verbose `if ... is undefined then ... else ...` pattern:

```uzon
// These two are equivalent:
port is env.PORT to u16 or else 8080
port is if env.PORT is undefined then 8080 else env.PORT to u16
```

`or else` has the **lowest precedence** of all operators — lower than `or`. It is not a logical operator: its operands may be any type, and it checks specifically for `undefined`, not for `false`. Implementations **MUST** treat `or else` as a single composite operator — it is distinct from `or` followed by `else`. Composite keyword formation happens at the **lexer level**, before parsing begins. Consequently, when `else` appears after `or` (even across a newline), it is **always** consumed as part of `or else` — it is never available as the `else` of an enclosing `if` or `case`. To use `or else` inside an `if`/`case` branch, wrap it in parentheses:

```uzon
// WRONG — or else consumes the else, if has no else branch → error
result is if cond then x or else "fallback"

// CORRECT — the if's else branch is separate from or else
result is if cond then (x or else "fallback") else "other"
// Parentheses are optional here (or else is already a single token),
// but recommended for readability.
``` The left and right operands **MUST** be the same type — if the left operand has a value, it is that type; the right operand must match. As with `is`/`is not` (§5.2), `null` is compatible with any type in this context. Both operands are speculatively evaluated to determine their types (see §5.9 evaluation semantics); runtime errors in the non-taken side are suppressed. When the left operand is statically provable as `undefined` (e.g., referencing a name that does not exist in any scope), the result type is determined by the right operand.

```uzon
// or = logical (bool only), or else = undefined coalescing (any type)
true or false                        // logical OR → true
env.PORT to u16 or else 8080         // coalescing → env.PORT to u16 if defined, else 8080

// null is NOT undefined — or else passes null through
x is null or else "fallback"         // x is null (null is a value, not undefined)
```

### 5.8 Collection Operators

Collection operators apply only to specific types: `in` applies to **lists**, **tuples**, and **structs** (value membership); `++` and `**` apply to **lists** and **strings**. Applying `++` or `**` to tuples or structs is a type error.

#### 5.8.1 Membership (`in`)

Tests whether a value is contained in a collection. The meaning is **value membership** for all collection types — "is this value inside?"

**Lists**: The value and the list elements **MUST** be the same type.

```uzon
"bravo" in [ "alfa", "bravo", "charlie" ]    // true
4 in [ 1, 2, 3 ]                             // false
4 in [ "a", "b", "c" ]                       // INVALID — type mismatch
5 in []                                      // false — empty list type inferred from left operand
```

**Tuples**: Tuples are heterogeneous — each element may have a different type. The evaluator compares the left operand against each element in order. Elements whose type does not match the left operand's type are skipped (treated as non-matching). No type error is raised for mismatched elements.

```uzon
42 in (1, "hello", 42)                      // true — matches third element
"hello" in (1, "hello", true)               // true — matches second element
99 in (1, "hello", true)                    // false — no matching value
```

**Structs**: The evaluator tests whether the value exists among the struct's **field values** — not its keys. As with tuples, fields whose value type does not match the left operand's type are skipped.

```uzon
8080 in { port is 8080, host is "localhost" }   // true — 8080 matches port's value
"localhost" in { port is 8080, host is "localhost" }  // true — matches host's value
"port" in { port is 8080, host is "localhost" }       // false — no field has value "port"
```

To check whether a **key** exists in a struct, use `std.hasKey` (§5.16.2).

**`null` exemption**: As with `is`/`is not` (§5.2), `null` is exempt from the type constraint on either side — `null` may appear as the value being tested or as an element/field value without raising a type error. The result is based on actual containment.

```uzon
null in [ 1, 2, 3 ]                          // false — null permitted, not found
null in [ 1, null, 3 ]                       // true — null is an element
1 in [ 1, null, 3 ]                          // true — 1 is an element
null in (1, "hello", null)                   // true — null is an element
null in { port is null, name is "api" }      // true — port's value is null
```

**Tagged union behavior**: `in` uses the same comparison semantics as `is`/`is not` (§5.2). For tagged union values, this means both the variant tag and the inner value are compared — a bare inner value does not match a tagged union element, and vice versa.

**`undefined` elements**: If an element within a tuple or a field value within a struct is `undefined` (e.g., from an unset `env` variable), the comparison follows `is`/`is not` semantics (§5.2) — comparing any value against `undefined` is permitted and returns `false`. No runtime error is raised for `undefined` elements encountered during the scan. Note that `undefined` as the left operand of `in` is still a runtime error per §3.1 — to check whether a specific element is `undefined`, use `is undefined` on that element directly.

**Function values**: Comparing function values with `is`/`is not` is a type error (§3.8). In heterogeneous collections (tuples, structs), elements whose value is a function are skipped when the left operand is not a function type — no error is raised. Attempting to use a function value as the left operand of `in` when any element is also a function type is a type error, consistent with §3.8.

#### 5.8.2 Concatenation (`++`)

Concatenates strings or lists.

```uzon
"alfa" ++ "bravo"                      // "alfabravo"
[ "a", "b" ] ++ [ "c" ]                // [ "a", "b", "c" ]
```

Both operands must be the same type (string ++ string, or list ++ list). For lists, the element types must also match exactly — `[1, 2] as [i32] ++ [3, 4] as [i16]` is a type error.

#### 5.8.3 Repetition (`**`)

Repeats a string or list a given number of times.

```uzon
"*" ** 3                                 // "***"
[ 0, 1 ] ** 4                            // [ 0, 1, 0, 1, 0, 1, 0, 1 ]
```

The left operand is a string or list; the right operand **MUST** be a non-negative integer (≥ 0, any integer type). Negative values are errors. Float values are not permitted as the right operand. When the count is zero, the result is an empty string (`""`) or an empty list (`[]` with the same element type).

**Precedence note**: `**` shares the same precedence level as `*`, `/`, `%` (Level 9, left-associative). When `**` and `*` appear in the same expression without parentheses, left-to-right grouping applies, which will typically result in a type error due to mismatched operand types. Use parentheses to clarify intent:

```uzon
"*" ** 3 + 1         // ("***") + 1 → type error — use "*" ** (3 + 1) for "****"
2 * "*" ** 3         // (2 * "*") ** 3 → type error — use "*" ** (2 * 3) for "******"
"*" ** 3             // "***" — no ambiguity when used alone
```

### 5.9 Conditional Expression (`if` / `then` / `else`)

```uzon
f is if e is green then 1 else 0
```

Syntax:

```
if <bool_expression> then <expression> else <expression>
```

Both `then` and `else` branches are required. The condition **MUST** evaluate to a `bool` — specifically, the literal values `true` or `false`. There is no truthy/falsy coercion: `0`, `null`, `""`, and any other non-bool values are **not** treated as `false`. Using a non-bool value as a condition is an error.

Both branches **MUST** evaluate to the same type. Returning different types from different branches (e.g., `i32` from `then` and `string` from `else`) is a type error. As with `is`/`is not` (§5.2), `null` is compatible with any type — if one branch evaluates to a specific type and the other to `null`, the expression is valid. This ensures that the result of an `if` expression has a single, unambiguous type — consistent with UZON's strong typing and cross-language mapping goals.

```uzon
if true then "yes" else "no"           // valid
if x > 0 then "pos" else "neg"    // valid — comparison returns bool
if x then "yes" else "no"         // INVALID if x is not bool
if 0 then "yes" else "no"              // INVALID — 0 is not bool
if null then "yes" else "no"           // INVALID — null is not bool
```

This strict rule also applies to `and`, `or`, and `not` — their operands **MUST** be `bool`.

**Evaluation semantics**: Only the selected branch is evaluated to produce the final result. However, implementations **MUST** speculatively evaluate all branches to determine their result types. Runtime errors (division by zero, overflow, invalid conversion) encountered in non-selected branches **MUST** be suppressed — only the result type is retained for type compatibility checking. This speculative evaluation is the mechanism by which UZON enforces type constraints without a separate static type system. The same mechanism applies to `case` (all `when` and `else` branches), `and`/`or` (short-circuit), and `or else` (right side).

```uzon
// safe — the error branch is never evaluated
x is if true then 1 else (1 / 0)   // 1 — division by zero never reached
```

### 5.10 Case Expression (`case` / `when` / `then` / `else`)

A multi-branch conditional that matches a value against several alternatives.

```uzon
p is case 5 % 3
    when 0 then "a"
    when 1 then "b"
    else "c"
// 5 % 3 = 2, no match → p is "c"
```

Syntax:

```
case <expression>          { when <value> then <expression> }  else <expression>
case type <expression>     { when <type> then <expression> }   else <expression>
case named <expression>    { when <tag> then <expression> }    else <expression>
```

The `else` branch is required. At least one `when` clause is also required — a `case` with no `when` is a syntax error. `when` values are tested in order; the first match wins. The `case` expression and all `when` values **MUST** be the same type, with the same exception as `is` (§5.2): `null` is always permitted regardless of type. Note that while `undefined` passes type checking in `when` position, it is always a runtime error at evaluation time (see below).

All `when` and `else` branches **MUST** evaluate to the same type. Returning different types from different branches is a type error. As with `if` (§5.9), `null` is compatible with any type across branches.

**Enum variant resolution in `case`**: When a `case` expression matches against a value of a named enum type, bare identifiers in `when` clauses are implicitly resolved as variants of that enum type, bypassing normal lexical scope lookup. Semantically, `when trace` is equivalent to `when trace as LogLevel` — the type checker infers the enum type from the `case` scrutinee and applies it to bare identifiers. This is consistent with variant resolution in `from` and `as` contexts (§3.5) and avoids the need for explicit `as EnumType` annotations on every `when` value.

```uzon
log is info from trace, debug, info, warn, error called LogLevel

// Bare identifiers resolve as LogLevel variants automatically
msg is case log
    when trace then "verbose"
    when debug then "debugging"
    when warn then "warning"
    else "normal"

// Explicit annotation is also valid but not required
msg is case log
    when trace as LogLevel then "verbose"
    else "normal"
```


**Type dispatch** (`case type`): Branches on the runtime type of a value. Most commonly used for untagged union discrimination, but works on any value:

```uzon
value is 42 from union i32, f64, string

description is case type value
    when i32 then "integer"
    when f64 then "float"
    when string then "text"
    else "other"
// description is "integer"
```

All branches **MUST** produce the same result type. `case type` works on any value, consistent with `is type` (§5.2). For unions (tagged or untagged), it dispatches on the inner value's type — consistent with `is type` (§5.2). For non-union values, the type is statically known, so only one `when` branch can ever match — but this is permitted for consistency and metaprogramming flexibility. When the scrutinee is a union (tagged or untagged), the type expressions after `when` are validated against the union's member types (for tagged unions, the set of distinct inner types across all variants) — using a type that is not a member of the union is a type error. When the scrutinee is a non-union value, any valid type expression is permitted in `when` clauses — non-matching branches simply never execute. Type expressions include compound types (`[i32]`, `(i32, string)`) and named types (via `called`). Anonymous struct shapes are **not** valid type expressions — use a named struct type via `called` to match against a struct shape. Matching uses the same "exactly the same type" rule as §5.2.

```uzon
// case type with compound types
data is [1, 2, 3] from union [i32], [string], i32

label is case type data
    when [i32] then "int list"
    when [string] then "string list"
    when i32 then "single int"
    else "other"
// label is "int list"
```

**Branch narrowing**: Inside a `when` branch, the scrutinee is **narrowed** to the matched type. This enables type-safe operations on the narrowed value:

```uzon
u is 42 from union i32, f64, string
msg is case type u
    when i32 then u + 1           // u narrowed to i32 — arithmetic valid
    when string then u ++ "!"     // u narrowed to string — concatenation valid
    else "other"
```

Narrowing applies to speculative evaluation (§5.9) — non-selected branches are type-checked against the narrowed type, not the original union type. The `else` branch is narrowed to the **remaining types** — those not matched by any `when` clause. If all member types are covered by `when` clauses, the `else` branch has no valid type and any type-dependent operation is an error (though the branch is never reached). This means `u + 1` in the `when i32` branch is always valid, even if `u` is actually a `string` at runtime (the branch is not selected, and the narrowed type `i32` makes `+ 1` well-typed).

**Variant dispatch** (`case named`): When a `case` expression needs to branch on the variant tag of a tagged union, use `case named`:

```uzon
health is "all ok" named ok from ok as string, degraded as string, down as null called Status

report is case named health
    when ok then "OK: {health}"
    when degraded then "WARN: {health}"
    when down then "DOWN"
    else "unknown"
```

**Branch narrowing**: Inside a `when` branch, the scrutinee is narrowed to the matched variant's inner type. This enables type-safe operations:

```uzon
result is 42 named ok from ok as i32, err as string called Result
msg is case named result
    when ok then result + 1         // result narrowed to i32
    when err then result ++ "!"     // result narrowed to string
    else "unknown"
```

Narrowing applies to speculative evaluation (§5.9) — non-selected branches are type-checked against the narrowed variant type, not the original tagged union type. The `else` branch is narrowed to the **remaining variants** — those not matched by any `when` clause. If multiple variants remain, the narrowed type is the union of their inner types.

All branches **MUST** produce the same result type. `case named` is only valid when the scrutinee is a tagged union — using `case named` on any other value is a type error. The variant names after `when` are validated against the tagged union's variant list.

`case` **cannot** directly match `undefined`. While the grammar permits `undefined` in expression position (including after `when`), the evaluator **MUST** reject `when undefined then ...` as a **type error** (§4.5) — consistent with `x is undefined` as a binding. `undefined` is a state, not a value; using the literal `undefined` as a binding value or match pattern is a type error. To handle a potentially undefined value, check with `is undefined` before entering the `case`:

```uzon
// Correct pattern for handling undefined + case
result is if x is undefined then "missing"
          else case x
              when 1 then "one"
              when 2 then "two"
              else "other"
```

```uzon
// valid — all integers
case code
    when 200 then "ok"
    when 404 then "not found"
    else "error"

// INVALID — case is integer, when is string
case code
    when "200" then "ok"       // error: type mismatch
    else "error"
```

*Keyword-as-variant edge case*: If an enum has a variant whose name is a keyword (e.g., `type`, `named`), use `@` escape in `when` clauses: `when @named then ...`. The same applies to binding decomposition: `x is named from ...` decomposes to `x = (named from ...)` where `named` begins a named-clause; use `x is @named from ...` to select the variant `named`.

### 5.11 Type Conversion (`to`)

The `to` operator converts a value to a specified type.

```uzon
port is if env.PORT is undefined then 8000 else env.PORT to u16
```

`to` performs explicit type conversion. Conversions not listed in the permitted conversion table (§5.11.0) are type errors. Numeric conversions that would overflow the target type are runtime errors — there is no silent truncation or wrapping.

**Precedence note**: `to` binds tightly as a postfix operator — it applies to the immediately preceding value, not to an entire expression. Use parentheses when converting the result of an arithmetic expression:

```uzon
1 + 2 to f64         // parsed as: 1 + (2 to f64) — NOT (1 + 2) to f64
(1 + 2) to f64       // explicit: 3.0
```

**`undefined` propagates through `to`**: since `undefined` is a state (not a value), applying `to` to `undefined` does not produce an error — it returns `undefined`. This enables natural chaining with `or else`:

```uzon
// undefined to u16 → undefined → or else 8080 → 8080
port is env.PORT to u16 or else 8080

// Without propagation, this would crash before reaching or else
```

```uzon
256 to u8              // INVALID — overflow (u8 max is 255)
-1 to u32              // INVALID — negative value in unsigned type
3.14 to i32            // valid — truncates decimal (3)
env.PORT to u16        // valid if value is within u16 range, error otherwise
```

Environment variables referenced through `env` are always `string`, so `to` is commonly used to convert them.

#### 5.11.0 Permitted Conversions

The following table defines **all** permitted `to` conversions. Any conversion not listed is a **type error**. The `undefined` row is included for completeness — `undefined` propagates through `to` regardless of target type (§5.11).

| Source → Target       | integer             | float              | `string`                  | `bool`     | `null`     | named enum    |
| --------------------- | ------------------- | ------------------ | ------------------------- | ---------- | ---------- | ------------- |
| integer               | ✓ range-checked     | ✓ widening         | ✓ decimal                 | ✗          | ✗          | ✗             |
| float                 | ✓ truncates decimal | ✓ precision change | ✓ decimal                 | ✗          | ✗          | ✗             |
| `string`              | ✓ parse literal     | ✓ parse literal    | ✓ identity                | ✗          | ✗          | ✓ exact match |
| `bool`                | ✗                   | ✗                  | ✓ `"true"`/`"false"`      | ✓ identity | ✗          | ✗             |
| `null`                | ✗                   | ✗                  | ✓ `"null"`                | ✗          | ✓ identity | ✗             |
| enum                  | ✗                   | ✗                  | ✓ variant name            | ✗          | ✗          | ✗             |
| tagged union          | ✗                   | ✗                  | ✓ inner value `to string` | ✗          | ✗          | ✗             |
| union (untagged)      | ✗                   | ✗                  | ✓ inner value `to string` | ✗          | ✗          | ✗             |
| struct / tuple / list | ✗                   | ✗                  | ✗                         | ✗          | ✗          | ✗             |
| function              | ✗                   | ✗                  | ✗                         | ✗          | ✗          | ✗             |
| `undefined`           | propagates          | propagates         | propagates                | propagates | propagates | propagates    |

**Key rules**:

- **Numeric → numeric**: Same category only. Integer ↔ integer (range-checked, overflow is error). Float ↔ float (precision change). Integer → float (widening). Float → integer (truncates decimal part toward zero). `inf` and `nan` cannot be converted to any integer type — `inf` exceeds all finite ranges (runtime error) and `nan` has no integer representation (runtime error).
- **String → numeric**: Parses the string as a numeric literal (§5.11.1). Leading/trailing whitespace or invalid characters are errors.
- **String → enum**: Exact match against variant names, case-sensitive (§5.11.1a).
- **To `string`**: See §5.11.2 for detailed rules per type. Compound types (struct, tuple, list) and functions cannot be converted.
- **Tagged union / untagged union → `string` only**: The `to` operator is a transparency exception (§3.7.1) — it sees the wrapper, not the inner value. The conversion table defines `to string` as a special case that extracts the inner value and converts it per §5.11.2. All other target types are type errors because `to` cannot implicitly unwrap and convert in one step. To convert a union's inner value to a non-string type, extract it via a transparent operator first (e.g., `inner is tu + 0` for numeric, `inner is tu ++ ""` for string), then apply `to`.
- **To `bool`**: Only identity conversion (`bool to bool`) is permitted. No other type may be converted to `bool`. Use explicit comparison instead (e.g., `env.DEBUG is "true"`).
- **To `null`**: Only identity conversion (`null to null`) is permitted. No other type may be converted to `null`.

#### 5.11.1 String-to-Numeric Conversion

When converting a string to a numeric type via `to`, the string **MUST** contain a valid numeric literal with no surrounding whitespace or extraneous characters. UZON recognizes the same numeric formats in strings as it does in source code, including base prefixes:

```uzon
"8080" to u16            // valid → 8080
"0xff" to u16            // valid → 255 (hex prefix recognized)
"0b1010" to u8           // valid → 10 (binary prefix recognized)
"007" to i32             // valid → 7 (leading zeros permitted in decimal)
"3.14" to f64            // valid → 3.14
"inf" to f64             // valid → inf (float keyword recognized)
"-inf" to f64            // valid → -inf
"nan" to f64             // valid → nan

" 8080 " to u16          // INVALID — leading/trailing whitespace
"12abc" to u16           // INVALID — non-numeric characters
"" to u16                // INVALID — empty string
"infinity" to f64        // INVALID — only "inf" is recognized, not "infinity"
```

#### 5.11.1a String-to-Enum Conversion

When converting a string to a named enum type via `to`, the string **MUST** exactly match one of the enum's variant names. If the string does not match any variant, the evaluator **MUST** report a runtime error.

```uzon
log_level is info from trace, debug, info, warn, error called LogLevel

// String → Enum via to
current is env.LOG_LEVEL to LogLevel or else info as LogLevel

// Equivalent verbose form (not needed with to):
// current is case env.LOG_LEVEL
//     when "trace" then trace as LogLevel
//     when "debug" then debug as LogLevel
//     ...

"info" to LogLevel         // valid → info variant
"TRACE" to LogLevel        // INVALID — case-sensitive, no match
"verbose" to LogLevel      // INVALID — not a variant name
```

This conversion is case-sensitive and performs no normalization — the string must match the variant name exactly as declared.

#### 5.11.2 String Conversion Rules

String interpolation (`{expr}` inside a string) implicitly performs `to string` on the expression result. The following table defines the result for each type. Note that while `undefined` propagates through `to` for other types (§5.11), string interpolation is the **terminal point** where an unhandled `undefined` becomes an error — this prevents silent bugs in configuration values like URLs and paths.

| Type             | `to string` Result                                               | Example                                |
| ---------------- | ---------------------------------------------------------------- | -------------------------------------- |
| `bool`           | `"true"` or `"false"`                                            | `true` → `"true"`                      |
| integer          | Decimal representation (always base 10)                          | `0xff` → `"255"`                       |
| float            | Shortest round-trip decimal (see rules below)                    | `3.14` → `"3.14"`, `1e30` → `"1.0e30"` |
| `inf`, `-inf`    | `"inf"`, `"-inf"`                                                |                                        |
| `nan`, `-nan`    | `"nan"`                                                          |                                        |
| `string`         | The string itself (identity)                                     | `"hi"` → `"hi"`                        |
| `null`           | `"null"`                                                         |                                        |
| `undefined`      | Propagates (returns `undefined`) — see note below                |                                        |
| enum             | Variant name                                                     | `green as RGB` → `"green"`             |
| tagged union     | Inner value `to string` (error if inner type is not convertible) | `"ok" named ok` → `"ok"`               |
| union (untagged) | Inner value `to string` (based on actual type)                   | `3.14 from union i32, f64` → `"3.14"`  |
| struct           | **Error** — compound types cannot be converted                   |                                        |
| tuple            | **Error**                                                        |                                        |
| list             | **Error**                                                        |                                        |
| function         | **Error**                                                        |                                        |

Compound types (struct, tuple, list) and functions **MUST NOT** be converted to string — neither via `to string` nor via interpolation.

**`undefined` and `to string`**: The explicit conversion `undefined to string` **propagates** — it returns `undefined`, consistent with all other `to` conversions (§5.11). This enables chaining: `env.MISSING to string or else "fallback"`. However, string **interpolation** (`{expr}`) is a **terminal context** — if the interpolated expression evaluates to `undefined`, the evaluator **MUST** report a runtime error. Use `or else` to provide a fallback before interpolation.

**Float `to string` rules**: The result is the shortest decimal string *s* such that parsing *s* as a float produces a value identical to the original. Let *k* be the number of significant digits and *n* be the exponent such that the value equals *s* × 10^(*n*−*k*); *k* **MUST** be as small as possible while preserving round-trip identity. The output format is determined by *n*:

- If 0 < *n* ≤ 21: plain decimal notation (e.g., `3.14`, `10000000.0`)
- If −6 < *n* ≤ 0: plain decimal with leading zeros (e.g., `0.000001`)
- Otherwise: scientific notation with one digit before the decimal point (e.g., `1.5e100`, `3.0e-8`)

The result **MUST** always contain a decimal point to distinguish it from an integer.

*Implementer note*: IEEE 754 negative zero (`-0.0`) **MUST** produce `"-0.0"` — the sign is preserved. Algorithms such as Ryū, Grisu3, and Schubfach all produce valid shortest round-trip strings. For a given float value, the shortest round-trip string is unique in almost all cases — conformant implementations **SHOULD** produce identical output regardless of algorithm choice.

```uzon
port is 8080
url is "http://localhost:{port}/api"   // valid — integer to string → "8080"

config is { host is "localhost" }
bad is "config: {config}"              // INVALID — struct to string is an error

// undefined in interpolation is an error — use or else
bad_url is "http://host:{env.PORT}/api"                  // INVALID — env.PORT may be undefined
good_url is "http://host:{env.PORT or else 8080}/api"    // valid — fallback ensures a value
```

### 5.12 Name Resolution

Bare identifiers in expression position are resolved using the **lexical scope chain**: the evaluator first looks in the current scope, then walks up through enclosing scopes, and finally reaches the file's top-level scope. If the name is not found in any scope and no enum variant resolution applies (§3.5), the result is `undefined`.

```uzon
x is 42
y is x           // x resolves to 42
z is x + 1       // x resolves to 42 → z is 43
```

Dot notation for field access works naturally: the first identifier resolves to a binding, and subsequent dots access fields on the resulting value.

```uzon
config is { port is 8080, host is "localhost" }
p is config.port           // config → binding → struct, .port → 8080
```

**Self-exclusion rule**: When evaluating the expression for a binding, the binding's own name is excluded from the current scope lookup. The evaluator skips the binding being defined and searches parent scopes instead. This exclusion is maintained for the **entire** value expression — including all sub-expressions, operators, and nested evaluations within that binding. This prevents trivial circular references and allows inheriting a value with the same name from an enclosing scope.

```uzon
port is 8080

server is {
    port is port            // evaluating this binding → server.port not yet defined → skip → file scope → 8080
    admin_port is port + 1  // server.port already defined → found → 8081
}
```

The self-exclusion rule also enables safe fallback patterns:

```uzon
// x skips this binding → undefined → or else → 1. NOT circular.
x is x or else 1     // x is 1
```

Nested scope example:

```uzon
base_port is 8080

server is {
    port is base_port       // not in server scope → walks up → file scope → 8080
    admin_port is port + 1  // found in server scope → 8081
    
    tls is {
        port is admin_port + 100  // not in tls → walks up to server → 8181
    }
}
```

Forward references are allowed: evaluation order is determined by the dependency graph, not textual order. Circular references are **forbidden** — this includes direct cycles (`a → b → a`), indirect cycles through nested scopes, and transitive cycles through file imports. Cycle detection is **static**: if a cycle exists in the dependency graph, it is rejected regardless of whether runtime values would prevent the cycle from being traversed. This ensures that the validity of a UZON document does not depend on external inputs like environment variables.

```uzon
// INVALID — static cycle, even though neither else-branch would execute
a is if true then 1 else b
b is if true then 2 else a
```

**`undefined` propagates through member access**: accessing a member on an `undefined` value yields `undefined` rather than an error. This provides implicit optional chaining — a missing intermediate path does not crash, and can be caught by `or else` at any point. Member access on `null` is a **type error** — `null` is a value (not a missing state), so accessing a field on it is a programming mistake. Member access on other primitive values (integers, floats, bools, strings, enums) yields `undefined` — primitives have no fields.

```uzon
config is { }
port is config.missing.port                     // undefined (not an error)
safe_port is config.missing.port or else 8080   // 8080
```


### 5.13 Environment Variables (`env`)

The `env` keyword accesses runtime environment variables. All values obtained through `env` have type `string`. If the variable is not set, `env.NAME` evaluates to `undefined`. Environment variable names that contain special characters (hyphens, dots, etc.) can be accessed using quoted identifiers: `env.'MY-VAR'`, `env.'app.config'`. Names that match keywords require `@` escape: `env.@type`, `env.@default` (§2.4). Standalone `env` without member access (i.e., `x is env`) is a type error — `env` must be followed by `.NAME` to access a variable.

```uzon
host is if env.HOST is undefined then "localhost" else env.HOST
port is if env.PORT is undefined then 8080 else env.PORT to u16
debug is env.DEBUG is not undefined
```

**Common trap — boolean flags from `env`**: Because `env.VAR` is always `string`, patterns like `env.DEBUG to bool or else false` look natural but are a **type error**. `string to bool` is not a permitted conversion (§5.11.0) — only `bool to bool` (identity) is allowed. `or else` only recovers from runtime `undefined`; it cannot catch type errors, which are rejected statically. Use explicit comparison instead:

```uzon
// INVALID — string to bool is a type error
debug is env.DEBUG to bool or else false

// CORRECT — explicit comparison
debug is (env.DEBUG or else "false") is "true"

// CORRECT — accept multiple truthy values
_raw is env.DEBUG or else "false"
debug is _raw is "true" or _raw is "1" or _raw is "yes"
```

### 5.14 Field Extraction (`of`)

The `of` keyword in binding position extracts a field from a struct using the binding's own name as the lookup key. `a is of b` is equivalent to `a is b.a`.

```uzon
_startup is struct "./startup"
environ is of _startup              // = _startup.environ
working_directory is of _startup    // = _startup.working_directory
startup_commands is of _startup     // = _startup.startup_commands
```

This eliminates the redundancy of writing the name twice (`environ is _startup.environ`). The pattern is particularly useful when importing bindings from an external file.

`of` in this position is syntactically unambiguous: it can only appear immediately after `is` in a binding — it is not valid in any other expression position.

The expression after `of` **MUST** evaluate to a struct; if it evaluates to a non-struct value (e.g., a number, string, or list), the evaluator **MUST** report an error. If the expression evaluates to `undefined`, `undefined` is propagated — consistent with member access (§5.12). If the extracted field does not exist in the target struct, the result is also `undefined`.

The `of` keyword is syntactically restricted to member access expressions — it cannot include operators like `or else`, arithmetic, or comparisons. To apply a fallback, place `or else` outside the `is of` binding:

```uzon
// INVALID — of is restricted to member_access, cannot include or else
port is of config or else 8080     // syntax error

// CORRECT — use explicit dot notation for fallback
port is config.port or else 8080

// of works with struct paths and imports
timeout is of struct "./config"         // valid: struct import
db_host is of database.primary     // valid: chained member access
x is of missing_struct             // valid: undefined propagated
z is of some_number                // INVALID — not a struct
```

**Caution with chained access**: `of` uses the binding's own name as the lookup key on the **entire** expression that follows. This can be surprising with deep paths:

```uzon
// x is of config.port  →  x is config.port.x
// This looks up "x" INSIDE config.port, NOT config.port
// If you want the value of config.port, use explicit dot notation:
port is config.port
```

### 5.15 Function Call

A function is called using parentheses. The opening `(` **MUST** appear on the same line as the callee expression — if `(` appears on the following line, it is parsed as a new expression (tuple or grouping), not a function call. This prevents ambiguity between function calls and tuple literals across line boundaries. The arguments are positional and **MUST** match the function's parameter types in order. The argument count **MUST** be between the number of required parameters (those without `default`) and the total parameter count. Function application is parsed as a postfix operation at the same precedence as member access (level 1) and may be interleaved with it — see the `call_or_access` production in §9.

```uzon
add is function a as i32, b as i32 returns i32 { a + b }
result is add(1, 2)               // 3
result is add (1, 2)              // 3 — space ok, same line
result is add(
    1, 2                          // ✓ — ( on same line as add
)
```

**Not a function call** — `(` on the next line starts a new expression:

```uzon
// In a function body:
make is function returns (i32, i32) {
    x is compute(10)
    (x, x + 1)                   // return tuple — NOT compute(10)(x, x + 1)
}
```

**Immediately Invoked Function Expression (IIFE)**: A function literal can be called inline if `(` is on the same line as the closing `}`:

```uzon
x is (function n as i32 returns i32 { n + 1 })(10)   // x is 11
```

The grouping parentheses around the function literal are required — without them, `}(10)` would attempt to call the struct/function body directly, which is syntactically valid by `call_or_access` but results in a type error since struct and list literals are not callable. The `call_or_access` production imposes no restriction on callee type at parse time — type errors from calling non-function values are caught during evaluation.

Functions are values — they can be stored in bindings, passed as arguments, and retrieved from lists or structs before calling:

```uzon
isEven is function n as i32 returns bool { n % 2 is 0 } called IntPredicate
isPos is function n as i32 returns bool { n > 0 } as IntPredicate

checks is [ isEven, isPos ] as [IntPredicate]
first_result is checks.first(42)  // true
```

Calling a non-function value is a **type error**. Calling with the wrong number of arguments or mismatched types is a **type error**.

### 5.16 Standard Library (`std`)

UZON provides a set of built-in functions through `std` — an implicit top-level binding injected by the evaluator. `std` is **not** a keyword — it is a binding that can be shadowed by the user (at their own risk).

| Category              | Function      | Signature                       | Returns    |
| --------------------- | ------------- | ------------------------------- | ---------- |
| Collection & String   | `len`         | `(collection_or_string)`        | `i64`      |
|                       | `reverse`     | `(list_or_string)`              | same type  |
| Collection Query      | `get`         | `(collection, index_or_key)`    | element    |
|                       | `hasKey`      | `(struct, key)`                 | `bool`     |
|                       | `keys`        | `(struct)`                      | `[string]` |
|                       | `values`      | `(struct)`                      | tuple      |
| Collection Predicate  | `all`         | `(list, func)`                  | `bool`     |
|                       | `any`         | `(list, func)`                  | `bool`     |
| Collection Transform  | `filter`      | `(list, func)`                  | `[T]`      |
|                       | `map`         | `(list, func)`                  | `[U]`      |
|                       | `reduce`      | `(list, initial, func)`         | `U`        |
|                       | `sort`        | `(list, comparator)`            | `[T]`      |
| Numeric               | `isFinite`    | `(float)`                       | `bool`     |
|                       | `isInf`       | `(float)`                       | `bool`     |
|                       | `isNan`       | `(float)`                       | `bool`     |
| String                | `contains`    | `(string, substring)`           | `bool`     |
|                       | `endsWith`    | `(string, suffix)`              | `bool`     |
|                       | `join`        | `([string], separator)`         | `string`   |
|                       | `lower`       | `(string)`                      | `string`   |
|                       | `replace`     | `(string, target, replacement)` | `string`   |
|                       | `split`       | `(string, delimiter)`           | `[string]` |
|                       | `startsWith`  | `(string, prefix)`              | `bool`     |
|                       | `trim`        | `(string)`                      | `string`   |
|                       | `upper`       | `(string)`                      | `string`   |

#### 5.16.1 Collection & String

Functions that operate on both collections (lists, tuples, structs) and strings.

**`std.len(collection_or_string)`** — Returns the number of elements in a list, tuple, or the number of fields in a struct. When applied to a string, returns the number of Unicode scalar values (codepoints). The result type is `i64`.

```uzon
std.len([ 1, 2, 3 ])                   // 3
std.len((true, "hi", 42))              // 3
std.len({ a is 1, b is 2 })            // 2
std.len("hello")                       // 5
std.len("")                            // 0
```

**`std.reverse(list_or_string)`** — Returns a new list with elements in reverse order, or a new string with characters (Unicode scalar values / codepoints) in reverse order. The argument **MUST** be a list or string. The return type matches the input type — a `[T]` list produces a `[T]` list, a `string` produces a `string`.

```uzon
std.reverse([ 1, 2, 3 ])              // [ 3, 2, 1 ]
std.reverse([ "a", "b", "c" ])        // [ "c", "b", "a" ]
std.reverse("hello")                  // "olleh"
std.reverse("한글")                    // "글한"
std.reverse([] as [i32])              // []
std.reverse("")                       // ""
```

`reverse` is a positional operation — it preserves all values and only changes their order. This is distinct from `std.sort`, which reorders by value comparison.

#### 5.16.2 Collection Query

**`std.get(collection, index_or_key)`** — Retrieves an element from a list or tuple by index (`i64`), or a field from a struct by name (`string`). Returns `undefined` if the index is out of bounds or the key does not exist — consistent with member access (§5.12). For tuples, this provides dynamic index access (whereas `.0`, `.1` etc. are static). Since tuples are heterogeneous, the return type when the index is not a compile-time constant is the union of all distinct element types (if all elements share the same type, the return type is simply that type — no union wrapper is created); the result is always `undefined`-able (the index may be out of bounds). For structs, the same rule applies — the return type is the union of all distinct field value types (or the common type if all fields share one); the result is `undefined`-able (the key may not exist). If the second argument's type does not match the expected type for the collection (`i64` for lists/tuples, `string` for structs), it is a **type error**.

```uzon
std.get([ 10, 20, 30 ], 1)             // 20
std.get({ port is 8080 }, "port")      // 8080
std.get([ 10, 20 ], 5)                 // undefined
```

**`std.hasKey(struct, key)`** — Tests whether a struct contains a field with the given name. Returns `bool`. The first argument **MUST** be a struct, and the key **MUST** be a `string`. For value membership testing (checking whether a value exists inside a list, tuple, or struct), use the `in` operator (§5.8.1).

```uzon
std.hasKey({ name is "UZON" }, "name")    // true
std.hasKey({ name is "UZON" }, "port")    // false
std.hasKey({ port is 8080, host is "localhost" }, "port")  // true
```

**`std.keys(struct)`** — Returns the field names of a struct as a `[string]` list, in declaration order.

```uzon
std.keys({ host is "localhost", port is 8080 })
// [ "host", "port" ]
```

**`std.values(struct)`** — Returns the field values of a struct as a **tuple**, in declaration order. The tuple element types match the field types of the struct — determined by speculative evaluation. This enables extraction of values from mixed-type structs. An empty struct produces an empty tuple `()`.

```uzon
std.values({ a is 1, b is "hi" })             // (1, "hi") — type (i64, string)
std.values({ x is true, y is 3.14 as f64 })   // (true, 3.14) — type (bool, f64)
std.values({ a is 1, b is 2, c is 3 })        // (1, 2, 3) — type (i64, i64, i64)
std.values({})                                // () — empty tuple
```

#### 5.16.3 Collection Predicate

Collection predicate functions test a condition across all elements of a list and return `bool`. They operate on **lists only**, not tuples — tuples are fixed-length and heterogeneously typed, so applying a single predicate function across all elements is not meaningful.

**`std.all(list, func)`** — Returns `true` if the function returns `true` for **every** element in the list. Returns `true` for an empty list (vacuous truth). The function **MUST** return `bool`.

```uzon
numbers are 2, 4, 6, 8
std.all(numbers, function n as i64 returns bool { n % 2 is 0 })
// true — all even

std.all(numbers, function n as i64 returns bool { n > 5 })
// false — 2 and 4 are not > 5

std.all([] as [i64], function n as i64 returns bool { n > 0 })
// true — vacuous truth

// practical use: validate all servers have TLS
std.all(servers, function s as Server returns bool { s.tls })
```

**`std.any(list, func)`** — Returns `true` if the function returns `true` for **at least one** element in the list. Returns `false` for an empty list. The function **MUST** return `bool`.

```uzon
numbers are 1, 3, 5, 8
std.any(numbers, function n as i64 returns bool { n % 2 is 0 })
// true — 8 is even

std.any(numbers, function n as i64 returns bool { n > 10 })
// false — none > 10

std.any([] as [i64], function n as i64 returns bool { n > 0 })
// false — empty list

// practical use: check if any server uses a privileged port
std.any(servers, function s as Server returns bool { s.port < 1024 })
```

#### 5.16.4 Collection Transform

Collection transform functions (`std.filter`, `std.map`, `std.reduce`, `std.sort`) operate on **lists only**, not tuples. Tuples are fixed-length and heterogeneously typed, so filtering or sorting them is not meaningful.

**`std.filter(list, func)`** — Returns a new list containing only the elements for which the function returns `true`. The function **MUST** return `bool`.

```uzon
numbers are 1, 2, 3, 4, 5
evens is std.filter(numbers, function n as i64 returns bool { n % 2 is 0 })
// [ 2, 4 ]
```

**`std.map(list, func)`** — Applies a function to each element of a list, returning a new list of the results.

```uzon
numbers are 1, 2, 3, 4, 5
doubled is std.map(numbers, function n as i64 returns i64 { n * 2 })
// [ 2, 4, 6, 8, 10 ]
```

**`std.reduce(list, initial, func)`** — Reduces a list to a single value by applying a function to an accumulator and each element. The function takes two parameters: the accumulator and the current element. The initial value and accumulator **MUST** be the same type as the function's return type.

```uzon
numbers are 1, 2, 3, 4, 5
total is std.reduce(numbers, 0, function acc as i64, n as i64 returns i64 { acc + n })
// 15
```

**`std.sort(list, comparator)`** — Returns a new list with elements sorted according to the comparator function. The comparator takes two elements and returns `bool` — `true` if the first argument should come before the second. The sort is **stable** — elements that compare equal retain their original order. The comparator **MUST** return `bool`. For stability, the implementation determines equality by checking both `comp(a,b)` and `comp(b,a)` — two elements are equal when both return `false` (strict weak ordering). If the list contains `nan` values, the comparator should handle them explicitly — IEEE 754 comparisons involving `nan` always return `false`, which can produce unpredictable ordering.

```uzon
numbers are 5, 2, 8, 1, 9
ascending is std.sort(numbers, function a as i64, b as i64 returns bool { a < b })
// [ 1, 2, 5, 8, 9 ]

descending is std.sort(numbers, function a as i64, b as i64 returns bool { a > b })
// [ 9, 8, 5, 2, 1 ]

// sort structs by field
_entry is { name is "", score is 0 as i32 } called Entry

entries are
    { name is "Charlie", score is 70 as i32 },
    { name is "Alice", score is 95 as i32 },
    { name is "Bob", score is 85 as i32 } as [Entry]

by_score is std.sort(
    entries,
    function a as Entry, b as Entry returns bool { a.score > b.score }
)
// [ Alice(95), Bob(85), Charlie(70) ]
```

#### 5.16.5 Numeric Utilities

**`std.isFinite(value)`** — Returns `true` if the value is neither `nan` nor `inf`/`-inf`. The argument **MUST** be a float type. This is the most practical check — it answers "can I safely compute with this value?"

```uzon
std.isFinite(3.14)                     // true
std.isFinite(inf)                      // false
std.isFinite(nan)                      // false
std.isFinite(0.0)                      // true
```

**`std.isInf(value)`** — Returns `true` if the value is `inf` or `-inf`. The argument **MUST** be a float type.

```uzon
std.isInf(1.0 / 0.0)                  // true (inf)
std.isInf(-1.0 / 0.0)                 // true (-inf)
std.isInf(3.14)                        // false
std.isInf(nan)                         // false
```

**`std.isNan(value)`** — Returns `true` if the value is `nan`. The argument **MUST** be a float type. This exists because IEEE 754 defines `nan is nan` as `false`, making `nan` impossible to detect with normal equality.

```uzon
x is 0.0 / 0.0                        // nan
std.isNan(x)                           // true
std.isNan(3.14)                        // false
std.isNan(inf)                         // false
```

#### 5.16.6 String Utilities

**`std.contains(string, substring)`** — Returns `true` if the string contains the given substring. Both arguments **MUST** be `string`. An empty substring always returns `true` (every string contains the empty string).

```uzon
std.contains("hello world", "world")       // true
std.contains("hello world", "xyz")         // false
std.contains("/api/v1/users", "api")       // true
std.contains("hello", "")                 // true

// practical use: check for debug flags
has_debug is std.contains(env.FLAGS or else "", "debug")
```

**`std.endsWith(string, suffix)`** — Returns `true` if the string ends with the given suffix. Both arguments **MUST** be `string`. An empty suffix always returns `true`.

```uzon
std.endsWith("config.uzon", ".uzon")      // true
std.endsWith("config.uzon", ".json")      // false
std.endsWith("hello", "")                // true

// practical use: check hostname domain
is_internal is std.endsWith(env.HOSTNAME or else "", ".internal")
```

**`std.join(list, separator)`** — Joins a `[string]` list into a single string, inserting `separator` between each element. The first argument **MUST** be `[string]` and the separator **MUST** be `string`. An empty list produces an empty string. This is the inverse of `std.split`.

```uzon
std.join([ "a", "b", "c" ], ":")           // "a:b:c"
std.join([ "hello" ], ":")                 // "hello"
std.join([] as [string], ":")              // ""
std.join([ "one", "two", "three" ], "--")  // "one--two--three"

// roundtrip with std.split
original is "a:b:c"
roundtrip is std.join(std.split(original, ":"), ":")  // "a:b:c"
```

**`std.lower(string)`** — Returns a new string with all characters converted to lowercase. The argument **MUST** be `string`. Non-alphabetic characters are unchanged.

```uzon
std.lower("Hello, World!")             // "hello, world!"
std.lower("DSV-2049")                  // "dsv-2049"
std.lower("already lower")            // "already lower"

// practical use: case-insensitive env comparison
mode is std.lower(env.MODE or else "PRODUCTION")
```

**`std.replace(string, target, replacement)`** — Replaces all occurrences of `target` in `string` with `replacement`. All three arguments **MUST** be `string`. If `target` does not occur in the input, the original string is returned unchanged. If `target` is an empty string, the original string is returned unchanged.

```uzon
std.replace("a:b:c", ":", "-")             // "a-b-c"
std.replace("hello world", "world", "UZON")  // "hello UZON"
std.replace("aaa", "a", "bb")              // "bbbbbb"
std.replace("hello", "xyz", "!")           // "hello" (no match)

// practical use: normalize paths
unix_path is std.replace(env.WIN_PATH or else "", "\\", "/")
```

**`std.split(string, delimiter)`** — Splits a string by the given delimiter, returning a `[string]` list. Both arguments **MUST** be `string`. If the delimiter does not occur in the input, a single-element list containing the original string is returned. If the input is an empty string, a single-element list containing the empty string is returned. If the delimiter is an empty string, the input is split into individual characters (Unicode scalar values) — `std.split("abc", "")` returns `["a", "b", "c"]`. These rules are checked in the order listed; the first matching rule determines the result. Thus `std.split("", "")` returns `[""]` (empty input rule takes precedence over empty delimiter rule).

```uzon
std.split("a:b:c", ":")                    // [ "a", "b", "c" ]
std.split("hello", ":")                    // [ "hello" ]
std.split("", ":")                         // [ "" ]
std.split("one--two--three", "--")         // [ "one", "two", "three" ]

// practical use with env
paths is std.split(env.PATH or else "", ":")
```

**`std.startsWith(string, prefix)`** — Returns `true` if the string starts with the given prefix. Both arguments **MUST** be `string`. An empty prefix always returns `true`.

```uzon
std.startsWith("/api/v1/users", "/api")    // true
std.startsWith("/api/v1/users", "/web")    // false
std.startsWith("hello", "")              // true

// practical use: route matching
is_api is std.startsWith(path, "/api")
is_admin is std.startsWith(path, "/admin")
```

**`std.trim(string)`** — Removes leading and trailing whitespace (space, tab, LF, CR) from a string. Returns a new string. The argument **MUST** be `string`.

```uzon
std.trim("  hello  ")                      // "hello"
std.trim("\thello\n")                      // "hello"
std.trim("hello")                          // "hello" (no change)

// practical use with env
db_host is std.trim(env.DB_HOST or else "localhost")
```

**`std.upper(string)`** — Returns a new string with all characters converted to uppercase. The argument **MUST** be `string`. Non-alphabetic characters are unchanged.

```uzon
std.upper("hello")                     // "HELLO"
std.upper("dsv-2049")                  // "DSV-2049"
std.upper("ALREADY UPPER")            // "ALREADY UPPER"

// practical use: normalize identifiers
ship_id is std.upper(env.SHIP_ID or else "wanderer")
```

All `std` functions are **pure** — they produce new values without modifying inputs. Their evaluation is **guaranteed to terminate** — lists are finite and the applied functions cannot recurse (§3.8).

---

## 6. Type System

### 6.1 Type Annotation (`as`)

The `as` keyword annotates a value with a type — it is an **assertion**, not a conversion. `as` declares that the value already belongs to the specified type; it does not transform the value. If the value is not compatible with the declared type, the evaluator **MUST** report a type error.

When the value is `undefined`, it propagates through `as` (§3.1) — the result is `undefined`. However, the type name **MUST** still be validated: `undefined as NonexistentType` is a type error because `NonexistentType` does not exist, even though the value would propagate regardless.

For **numeric types**, `as` constrains an untyped literal to a specific width. The value must fit within the target type's range — overflow is an error. Cross-category annotation (e.g., float as integer or string as integer) is a type error; use `to` for conversion instead.

```uzon
x is 42 as i32             // valid — 42 fits in i32
y is 3.14 as f64           // valid — float annotated as f64
z is 255 as u8             // valid — fits in u8
w is 256 as u8             // INVALID — overflow (u8 max is 255)
v is 3.14 as i32           // INVALID — float cannot be asserted as integer; use 3.14 to i32
name is "UZON" as string   // valid — string annotated as string
```

For lists, the type is wrapped in brackets:

```uzon
ids is [ 1, 2, 3 ] as [i32]
names is [ "a", "b", "c" ] as [string]
```

Type annotations are optional. More precisely, `as` can be omitted whenever the type is unambiguous from context. **`as` is fundamentally an ambiguity resolver** — this principle applies across the entire type system.

**Type namespace**: Built-in type names (`i{N}`, `u{N}`, `f16`–`f128`, `bool`, `string`) and user-defined type names (declared with `called`) occupy a separate namespace from bindings. A binding named `string` does not shadow the `string` type — `as string` always refers to the type. To avoid human confusion, using `@` escape for bindings that share a name with a type is **RECOMMENDED** but not required (e.g., `@string is "text"`).

```uzon
// Enum variants are unique to Layout → as Layout can be omitted on individual elements
layout is tiled from tiled, monocle, floating called Layout
layouts is [ tiled, monocle, floating ] as [Layout]

// But if a variant exists in multiple types → as is required
color is red from red, green, blue called RGB
temp is red from red, yellow, purple called Warm
x is red as RGB     // "red" is ambiguous — must specify
y is red as Warm
```

### 6.2 Type Naming (`called`)

The `called` keyword names a type for reuse. It appears at the end of a type-defining expression.

**Restriction**: `called` **MUST** appear only at the top level of a binding's expression — not inside control flow branches (`if`/`case`), parenthesized subexpressions, or nested contexts. This ensures that all type names are statically discoverable regardless of runtime values.

```uzon
// Valid — called at top level of binding
color is red from red, green, blue called RGB

// INVALID — called inside if branch (type existence depends on runtime)
x is if env.PROD is "true" then (80 called ProdPort) else 8080
```

```uzon
// Name an enum type
color is red from red, green, blue called RGB

// Name a struct type
origin is { x is 0, y is 0 } called Point

// Name a union type
value is 42 from union i32, f64, string called NumberOrString

// Name a tagged union type
result is "hello" named ok from ok as string, err as string called Result
```

**Type name scoping**: Type names declared with `called` follow **lexical scoping**, consistent with bindings. A type name is visible within the scope where it is declared and in all child scopes. To reference a type name declared in a nested scope from an outer scope, use dot notation for qualified access.

**Type paths vs value paths**: Type paths (after `as`, `to`, `from union`, etc.) use bare dotted names (`inner.RGB`, `base.LogLevel`). Value paths also use bare identifiers to traverse the scope chain (`inner.x`). Both resolve via their respective namespaces — types statically, values via lexical scope.

When a type path contains multiple segments (e.g., `_shared.Role`), the first segment is resolved as a **binding name** in the current scope. If that binding evaluates to a struct (including an imported file), the remaining segments are resolved as type names within that struct's scope. This enables type references through imports: `_shared.Role` means "the type `Role` declared inside the struct bound to `_shared`."

```uzon
inner is {
    x is red from red, blue called RGB
    y is blue as RGB                      // visible — same scope
}

// Value access
v is inner.x                         // value: red variant

// Type access — bare path
z is blue as inner.RGB                    // type: inner.RGB
w is blue as RGB                          // INVALID — RGB not in outer scope
```

### 6.3 Type Reuse (`as` with named types)

Once a type is named with `called`, other bindings can reference it with `as`:

```uzon
primary is red from red, green, blue called RGB
secondary is blue as RGB
accent is green as RGB

origin is { x is 0, y is 0 } called Point
cursor is { x is 100, y is 200 } as Point

// Tagged union reuse — no need to repeat variant definitions
result is "ok" named ok from ok as string, err as string called Result
another is "fail" as Result named err
// Ordering: as Type MUST precede named variant.
// "fail" named err as Result is a syntax error.
```

When `as` is used with a named type, the value **MUST** conform to that type's structure. Conformance follows the same rules as `with` type compatibility (§3.2.1): field names must match exactly (no extra, no missing), field types must be identical, and field order does not matter. Untyped literals adopt the expected field type and are range-checked — if the named type defines `x is 0 as i32`, then `{ x is 1000000 } as Point` adopts `i32` for `x` and verifies the value fits. Conformance checking is recursive — nested struct fields are checked against their declared types in the named type definition. Using `as` with a tagged union type **requires** `named` to specify the active variant — `as TaggedUnionType` without `named` is a type error, because a tagged union value must always have a variant tag. Using `as` with a named union type checks that the value's type matches at least one of the union's member types — if no member type is compatible, it is a type error.

---

## 7. File Import

### 7.1 Importing Files

Since every UZON file is an anonymous struct, importing a file is done with the `struct` keyword followed by a path. The imported file's bindings become accessible via dot notation.

```uzon
q is struct "./mod1"
r is q.array ++ [ 4, 5 ]
```

The path is a string literal, resolved relative to the importing file's location. The path **MUST** be a non-interpolated string — interpolation in import paths is a syntax error, because import resolution occurs before expression evaluation. If the last component of the path does not contain a `.` character, the evaluator **MUST** automatically append `.uzon` and resolve accordingly. If the path already contains a `.` in the last component (e.g., `"./data.uzon"`, `"./my.config"`), it is used as-is.

```uzon
shared is struct "./shared"         // resolves to ./shared.uzon
utils is struct "./lib/utils"       // resolves to ./lib/utils.uzon
explicit is struct "./data.uzon"    // used as-is
```

**Path canonicalization**: After resolving the relative path against the importing file's directory and appending `.uzon` if needed, the evaluator **MUST** canonicalize the result to a **physical absolute path** before using it for circular import detection or diamond import deduplication. Canonicalization **MUST** include: (1) converting to an absolute path, (2) resolving `.` and `..` segments, and (3) resolving symbolic links to their physical target. Two import paths that refer to the same physical file **MUST** produce the same canonical path, regardless of how the paths are written in source. Implementations that compare raw or partially resolved path strings will fail to detect circular imports and diamond duplicates when different relative paths (e.g., `"./shared"` from one directory and `"../shared"` from a subdirectory) or symbolic links refer to the same file.

```
// Example: both resolve to the same physical file
// project/main.uzon
shared is struct "./shared"           // → /project/shared.uzon

// project/sub/config.uzon
shared is struct "../shared"          // → /project/shared.uzon

// Canonical paths are equal → diamond dedup applies
```

### 7.2 File as Struct

A UZON file evaluates to a struct containing all of its top-level bindings. There is no distinction between an inline struct and an imported file — both are structs.

```uzon
// shared.uzon
array is [ 1, 2, 3 ]
name is "shared config"
```

```uzon
// main.uzon
shared is struct "./shared"
items is shared.array      // [ 1, 2, 3 ]
label is shared.name       // "shared config"

// Equivalent to writing inline:
// shared is { array is [ 1, 2, 3 ], name is "shared config" }
```

Circular file imports are **forbidden**. Cycle detection **MUST** use canonical paths (§7.1) — two imports form a cycle if their canonical paths are equal, regardless of how the relative paths are written in source.

**Diamond imports**: When the same file is imported through multiple paths (e.g., A imports B and C, both of which import D), the evaluator **MUST** deduplicate by canonical file path (§7.1). Only one copy of D exists in the namespace tree. Since name resolution does not cross file boundaries (§7.3), deduplication is always safe — the shared file behaves identically regardless of who imports it.

> **Convention — exporting a single value.** A UZON file is always an anonymous struct (§7.2). This means a file cannot directly be a list, string, or other non-struct value — a bare `[1, 2, 3]` at the top level is **not** valid UZON. To export a single value, wrap it in a binding. Any binding name is valid, but `_` is recommended by convention as it clearly signals that the binding exists only as a wrapper:
>
> ```uzon
> // key_bindings.uzon
> _ is [
>     { keysym is "q", modifiers is { mod4 is true }, event is { pressed is quit } }
> ]
> ```
>
> The importing file then accesses the value explicitly via dot notation:
>
> ```uzon
> key_bindings is struct "./key_bindings"._
> ```

### 7.3 Scope Isolation Across Files

Name resolution **MUST** stop at the file boundary. An imported file does **not** inherit the lexical scope of the file that imports it. Identifiers inside an imported file resolve only within that file's own top-level scope — they never walk up into the importing file.

```uzon
// theme.uzon
bg is dark_mode or else false
// dark_mode → not found in theme.uzon → undefined → bg is false

// main.uzon
dark_mode is true
theme is struct "./theme"
// theme.bg is false — theme.uzon cannot see main.uzon's dark_mode
```

To share values across files, pass them explicitly by defining them in a shared imported file:

```uzon
// shared.uzon
dark_mode is true

// theme.uzon
_shared is struct "./shared"
bg is _shared.dark_mode or else false
// _shared.dark_mode → true → bg is true

// main.uzon
shared is struct "./shared"
theme is struct "./theme"
// Both files reference the same shared.uzon
```

This file-boundary isolation ensures that a file's behavior is identical whether it is imported or used standalone — it never depends on who imports it or in what order. This also makes diamond imports (where multiple files import the same dependency) safe: since the shared file cannot see any importer's scope, deduplication produces consistent results regardless of import order.

Evaluation **MUST** follow this order: (1) parse all files and recursively process all `struct` imports, (2) deduplicate files by canonical path (§7.1), (3) resolve all identifier references within each file's own scope.

---

## 8. Comma and Newline Rules

Commas (`,`) serve as separators between bindings and values. Their requirement depends on syntactic context:

**Required** — when multiple bindings or values appear on the **same line** and the boundary between them would be ambiguous:

```uzon
l is 4, m is false, n is "some string"
```

**Optional** — when the line boundary is unambiguous, such as after a closing delimiter (`}`, `]`) or when each binding is on its own line:

```uzon
o is { a is null, b is 0 },    // trailing comma: optional

config is {
    host is "localhost"        // no comma needed (own line)
    port is 8080
    debug is true
}
```

Within **struct** literals, commas between fields on **different lines** are **optional** — newlines act as separators following the NEWLINE_SEP rule. Within **tuple** and **list** literals, commas between elements are **required** even across lines — newlines inside `()` and `[]` are treated as ordinary whitespace. Trailing commas are always allowed.

**Newline as separator**: A UZON document (and each struct body) consists exclusively of bindings — there are no standalone expressions. A newline acts as a binding separator **only** when the following non-whitespace, non-comment tokens form a new binding: `identifier is` or `identifier are`, where the identifier is a non-keyword token. Keywords like `called`, `named`, `from` on a new line are always expression continuations, never binding starts. Otherwise, the newline is treated as ordinary whitespace and the current expression continues on the next line. Inside list `[…]` and tuple `(…)` literals, newlines are always whitespace — commas are required. Inside struct `{…}` literals, the NEWLINE_SEP rule applies identically to the document level. Inside function bodies `{ … }`, the same rule applies — `name is` starts a new intermediate binding; when the next line does not match `name is`, the parser exits the binding loop and parses the final expression. Blank lines (consecutive newlines with no tokens) are whitespace.

This rule requires at most 2-token lookahead after a newline. Because every top-level construct is a binding, there is no ambiguity — a line that does not start with `name is`, `name are`, a closing delimiter (`}`), or EOF must be a continuation of the previous expression.

**Expression continuation**: When parsing an expression, newlines are treated as ordinary whitespace. A binary operator (`+`, `-`, `*`, `/`, `%`, `^`, `++`, `**`, `and`, `or`, `or else`, `is`, `is not`, `in`, `<`, `<=`, `>`, `>=`), postfix keyword (`to`, `with`, `plus`, `as`, `from`, `named`), or member access (`.`) may appear on the following line and is consumed as part of the current expression. The exception is `(` — a line starting with `(` is **never** a function call continuation; it begins a new expression (tuple or grouping). This rule applies in **all** contexts: at the document level, inside struct `{…}` bodies, inside list `[…]` and tuple `(…)` literals, inside function bodies, and inside string interpolation `{…}`. This ensures that tuple literals and grouping expressions on their own line are never misinterpreted as function arguments. The `called` keyword may also appear after a binding's expression on the following line; since it is a keyword, the NEWLINE_SEP rule ensures it is never mistaken for a new binding start. The NEWLINE_SEP rule applies **only** at the binding/struct-field level — never inside an expression being parsed.

**`are` binding continuation**: In an `are` binding, each element is a full expression. A binary operator on the next line continues the **last element's** expression, not the list. `items are 1, 2` followed by `+ 3` on the next line evaluates as `items are 1, 5` — the `+ 3` continues the expression `2`. Keep each element on a single line or use explicit parentheses to avoid surprises.



```uzon
// Newline followed by "y is" → separator → two bindings
x is 1
y is 2

// Newline NOT followed by a binding → continuation → x is 1 + 2
x is 1
+ 2

// Control flow naturally spans lines
shell is case env.SHELL
    when "bash" then "bash"
    when "zsh" then "zsh"
    else "fish"

// Blank lines are whitespace, not double-separators
x is 1

y is 2
```

---

## 9. Formal Grammar

```ebnf
(* === Document === *)
document        = [ binding , { sep , binding } , [ sep ] ] ;

(* === Common === *)
name            = identifier | keyword_escape ;
                  (* identifier includes quoted identifiers: 'Content-Type', 'this is a key'.
                     See §2.3 for quoted identifier rules. *)

(* === Bindings === *)
binding         = name , ( is_binding | are_binding ) ;
is_binding      = "is" , ( "of" , member_access          (* field extraction: a is of b = a is b.a *)
                         | expression ) ,
                  [ "called" , name ] ;
                  (* called MUST only appear at the top level of a binding.
                     This ensures type names are statically discoverable. *)
are_binding     = "are" , expression , { "," , expression } ,
                  [ "as" , type_expr ] , [ "called" , name ] ;
                (* The optional "as" at the end is the list-level type annotation,
                   NOT part of the last element. The parser resolves this by treating
                   a trailing "as" after the final element as the binding's annotation.
                   For element-level annotation, wrap in parentheses:
                   ids are 1, 2, (3 as i32).
                   Multiline are bindings are supported — continuation follows
                   the NEWLINE_SEP rule. *)

(* === Expressions === *)
expression      = or_else_level ;
or_else_level   = or_level , { "or else" , or_level } ;
                (* "or else" is a single composite operator, distinct from "or".
                   Lexer emits it as one token using 1-token lookahead. *)
or_level        = and_expr , { "or" , and_expr } ;
and_expr        = not_expr , { "and" , not_expr } ;
not_expr        = "not" , not_expr | equality ;
equality        = membership , [ ( "is not named" , variant_name       (* negated variant check *)
                                   | "is not type" , type_expr           (* negated type check *)
                                   | "is named" , variant_name           (* variant check *)
                                   | "is type" , type_expr               (* type check *)
                                   | "is not" , membership              (* inequality *)
                                   | "is" , membership ) ] ;            (* equality *)
                (* No chaining — at most one comparison per expression level.
                   "is named" / "is not named" checks the variant of a tagged union.
                   "is type" / "is not type" checks the runtime type of a value.
                   All six forms are single composite tokens emitted by the lexer. *)
membership      = relational , [ "in" , relational ] ;
relational      = concat_expr , [ ( "<" | "<=" | ">" | ">=" ) , concat_expr ] ;
concat_expr     = addition , { "++" , addition } ;
addition        = multiplication , { ( "+" | "-" ) , multiplication } ;
multiplication  = unary , { ( "*" | "/" | "%" | "**" ) , unary } ;
unary           = [ "-" ] , power_expr ;
power_expr      = type_decl , [ "^" , unary ] ;
                  (* Right-associative: 2 ^ 3 ^ 2 = 2 ^ (3 ^ 2) = 512.
                     Exponentiation (^) is numeric only.
                     Repetition (**) is string/list only. *)

(* === Postfix chain — each level wraps the previous, tightest first === *)
(* Precedence: . > to > with > as > from/named                            *)
(* Note: called is at binding level, not in the expression chain.          *)
type_decl       = type_annot_level , [ from_clause | named_clause ] ;
type_annot_level = struct_override , [ "as" , type_expr ] ;
struct_override = conversion , [ ( "with" | "plus" ) , struct_literal ] ;
                  (* with: override only, no new fields. plus: override + add.
                     At most one per expression level, no chaining.
                     Override/extension block is a struct literal evaluated in
                     the calling scope, not inside the base. *)
conversion      = call_or_access , [ "to" , type_expr ] ;
                  (* to: applies to immediate left — a + b to i32 = a + (b to i32) *)
member_access   = primary , { "." , ( name | integer ) } ;
                  (* When a field name is a keyword, use @ escape:
                     config.@type, obj.@from, settings.@default. *)
                  (* Used by "of" in is_binding — no function calls allowed. *)
call_or_access  = primary , { "." , ( name | integer )
                             | "(" , [ expression , { "," , expression } , [ "," ] ] , ")" } ;
                  (* Member access and function call share the same precedence
                     and are left-associative. funcs.first(42) is valid.
                     Calling a non-function value is a type error.
                     IMPORTANT: The opening "(" of a function call MUST be on
                     the same line as the callee. A "(" on the next line starts
                     a new expression — tuple or grouping — not a call. *)

(* Type system suffixes *)
from_clause     = "from" , ( "union" , type_expr , "," , type_expr ,
                           { "," , type_expr }                            (* union — 2+ types *)
                           | variant_name , "," , variant_name ,
                           { "," , variant_name } ) ;                     (* enum — 2+ variants *)
                (* For enum, the preceding primary MUST be a name (the selected variant).
                   Duplicate variant names within a single `from` clause are a type error — each variant must be unique. Variant consumption terminates when: (1) a comma is followed by a
                   non-keyword identifier + "is"/"are" (2-token lookahead, same pattern
                   as NEWLINE_SEP), (2) "called" appears, (3) a closing delimiter or
                   EOF is reached. Keywords after comma are consumed as variants. *)
variant_name    = name | keyword ;
                (* Bare keywords are valid as enum/tagged-union variant names.
                   e.g., x is a from a, named, b — "named" is a variant here. *)
named_clause    = "named" , variant_name , [ "from" ,
                  variant_name , "as" , type_expr , "," ,
                  variant_name , "as" , type_expr ,
                  { "," , variant_name , "as" , type_expr } ] ;    (* tagged union — 2+ variants *)
                  (* When "from" is omitted, the tagged union type must be
                     specified via "as" in type_annot_level. This enables
                     reuse of a named tagged union type:
                     e.g., "hello" as Result named ok *)

primary         = literal
                | name
                | struct_literal
                | tuple_or_group
                | list
                | if_expr
                | case_expr
                | struct_import
                | function_expr
                | "env"             (* standalone env without .NAME is a type error — §5.13 *)
                | "undefined" ;
                (* "undefined" is syntactically valid in primary position but
                   semantically forbidden as the right side of a binding (§3.1, §4.5).
                   Its primary use is in comparisons: x is undefined. *)
                (* undefined is a first-class state, not a value.
                   It propagates through `.`, `to`, and `as`, but is an error
                   for most other operators — see the propagation table in §3.1.
                   Expressions may produce undefined (missing, env.UNSET, out-of-bounds).
                   This enables patterns like: a is x, b is a or else 1 *)

(* === Literals === *)
literal         = integer | float | multiline_string
                | "true" | "false" | "null" ;

integer         = [ "-" ] , ( hex_int | oct_int | bin_int | dec_int ) ;
dec_int         = DIGIT , { ( "_" , DIGIT ) | DIGIT } ;
hex_int         = "0" , ("x"|"X") , HEX , { ( "_" , HEX ) | HEX } ;
oct_int         = "0" , ("o"|"O") , OCT , { ( "_" , OCT ) | OCT } ;
bin_int         = "0" , ("b"|"B") , BIN , { ( "_" , BIN ) | BIN } ;

float           = [ "-" ] , ( float_num | "inf" | "nan" ) ;
float_num       = dec_int , "." , dec_int , [ exponent ]
                | dec_int , exponent ;
exponent        = ("e"|"E") , ["+"|"-"] , dec_int ;
                (* Lexer note: a minus sign is context-sensitive. If the preceding token
                   is a value token (number, string, identifier, true, false, null, inf, nan,
                   undefined, env, ")", "]", "}"), minus is tokenized as the binary
                   subtraction operator. Otherwise (at expression start, after an operator,
                   after "is", etc.), a minus immediately followed by digits, "inf", or "nan"
                   is a single negative numeric literal.
                   E.g., "3 - 5" and "3-5" are both subtraction; "x is -5" is a negative literal;
                   "x is -inf" is a negative infinity literal. *)

string          = '"' , { str_part } , '"' ;
str_part        = char | interpolation ;
char            = unescaped | escaped ;
unescaped       = (* any Unicode scalar value except '"', '\', '{', and control characters U+0000–U+001F *) ;
interpolation   = "{" , expression , "}" ;
escaped         = "\\" , ( '"' | "\\" | "n" | "r" | "t" | "0" | "{"
                              | "x" , HEX , HEX
                              | "u{" , HEX , { HEX } , "}" ) ;
                  (* \u{...} is limited to 1-6 hex digits representing a valid
                     Unicode scalar value (U+0000-U+10FFFF, excluding surrogates
                     U+D800-U+DFFF). Values outside this range are errors. *)
multiline_string = string , { NEWLINE , [ ws ] , string } ;

(* === Compounds === *)
struct_literal  = "{" , [ field , { sep , field } , [ sep ] ] , "}" ;
field           = binding ;
                  (* struct fields use the same binding rules as the document level,
                     including is, are, is of, type annotations, and enum/union definitions. *)

tuple_or_group  = "(" , [ expression , { "," , expression } , [ "," ] ] , ")" ;
                (* () = empty tuple. (expr) = grouping. (expr,) = 1-tuple.
                   (expr, expr) = 2-tuple. Comma presence distinguishes tuple from grouping. *)

list            = "[" , [ expression , { "," , expression } , [ "," ] ] , "]" ;
                  (* Commas are required between list elements, even across lines.
                     Newlines inside [] are ordinary whitespace. Trailing comma allowed. *)

(* === Control === *)
if_expr         = "if" , expression , "then" , expression , "else" , expression ;
case_expr       = "case" , [ "type" | "named" ] , expression ,
                  "when" , ( type_expr | variant_name | expression ) , "then" , expression ,
                  { "when" , ( type_expr | variant_name | expression ) , "then" , expression } ,
                  "else" , expression ;
                  (* Three forms:
                     case expr when val then ...        — value matching
                     case type expr when T then ...     — type dispatch (any value)
                     case named expr when tag then ...  — variant dispatch (tagged unions)
                     At least one "when" clause is REQUIRED.
                     If no "when" matches, the "else" branch is taken. *)

(* === Import === *)
struct_import   = "struct" , string ;
                (* The string MUST be a static string literal — interpolation
                   is a syntax error (§7.1). *)

(* === Function === *)
function_expr   = "function" , [ param_list ] , "returns" , type_expr , function_body ;
param_list      = param , { "," , param } ;
param           = name , "as" , type_expr , [ "default" , expression ] ;
                  (* Parameters with defaults MUST appear after all required
                     parameters. A required parameter after a defaulted one
                     is a syntax error. *)
function_body   = "{" , [ func_binding , { sep , func_binding } , [ sep ] ] ,
                  expression , "}" ;
                  (* The body contains zero or more intermediate bindings followed
                     by a final expression whose value is the function's result.
                     The separator between the last func_binding and the final
                     expression is optional — the parser distinguishes them using
                     2-token lookahead: if the next tokens are `name is`, it is a
                     func_binding; otherwise, it is the final expression.
                     Parameters are accessed as bare identifiers.
                     Inside a function body, parameters and local bindings are part of the function body scope layer.
                     env inside a function body is permitted (read-only).
                     Recursive calls (direct or mutual) are forbidden — the call
                     graph MUST be a DAG, checked statically. *)
func_binding    = name , "is" , expression ;

(* === Type Expressions === *)
type_expr       = named_type | list_type | tuple_type | "(" , type_expr , ")" | "null" ;
                  (* "null" is a keyword and cannot appear as an identifier in
                     named_type, but is a valid type for tagged union variants
                     (e.g., down as null). *)
named_type      = name , { "." , name } ;
                  (* A type referenced by name — covers primitives (i32, f64, bool,
                     string), and any user-defined type named with `called` (enums,
                     structs, unions, tagged unions, function types). Qualified paths
                     (e.g., inner.RGB, base.LogLevel) use dot notation. *)
list_type       = "[" , type_expr , "]" ;
tuple_type      = "()" | "(" , type_expr , "," , ")" | "(" , type_expr , "," , type_expr , { "," , type_expr } , ")" ;
                  (* 0-element: (). 1-element: (type,) — trailing comma
                     distinguishes from grouping. 2+: (type, type, ...). *)

(* === Misc === *)
keyword_escape  = "@" , keyword ;
keyword         = "is" | "are" | "from" | "called" | "of" | "named" | "as" | "to" | "with" | "union"
                | "plus" | "if" | "then" | "else" | "case" | "when"
                | "and" | "or" | "not" | "in"
                | "env" | "struct"
                | "true" | "false" | "null" | "undefined"
                | "inf" | "nan"
                | "function" | "returns" | "default"
                | "type" | "lazy" ;
                (* reserved for future use *)
sep             = "," | NEWLINE_SEP ;
                  (* NEWLINE_SEP: a newline that acts as a separator.
                     Applies at document level, inside struct literals, and
                     inside function bodies.
                     Inside list [] and tuple () literals, newlines are always
                     whitespace — commas are required between elements.
                     A newline is a separator ONLY when the following
                     non-whitespace, non-comment tokens form a new binding
                     (identifier "is" or identifier "are", where the identifier
                     is NOT a keyword), a closing delimiter ("}"),
                     or EOF. Otherwise, newlines are ordinary whitespace.
                     This means keywords like "called", "named", "from" on a
                     new line are always continuations, never binding starts.
                     Consecutive blank lines are whitespace, not multiple separators.
                     In function bodies, the same 2-token lookahead applies:
                     "name is" starts a new func_binding; anything else after
                     the last func_binding begins the final expression. *)
comment         = "//" , { any_char } , NEWLINE ;
ws              = { " " | "\t" } ;

(* === Lexer Rules (normative) ===
   The following rules govern tokenization:
   - The lexer MUST recognize the following composite operators as single tokens
     using up to 2-token lookahead: "or else", "is not", "is named", "is not named", "is type", "is not type". Composite keyword lookahead **crosses newlines** — `or` followed by a newline and then `else` is still emitted as a single `or else` token. This ensures that splitting a composite keyword across lines does not change semantics.
     When the lexer encounters "is", it checks if the next token is "not" —
     if so, it further checks if the token after that is "named" to distinguish
     "is not named" from "is not". Similarly for "is" + "named" and "or" + "else".
   - BINDING DECOMPOSITION: At binding position, the first "is" is always the
     binding operator. Composite tokens are decomposed: "is not" → binding + "not expr",
     "is named" → binding + "named expr", "is type" → binding + "type expr" (parse error
     since "type" cannot start an expression). "is not named" → binding + "not named expr".
     To use these as comparison operators at binding level, wrap in parentheses:
     y is (x is not named foo).
   - ORIGINAL DECOMPOSITION DETAIL: When a composite "is not" / "is named" / "is not named"
     token appears at binding position (immediately after a name at the start of a
     binding), the parser MUST decompose it: "is" is the binding operator, and the
     remaining tokens ("not", "named", "not named") begin the value expression.
     Example: `x is not true` parses as `x = (not true)`, NOT as `x (is not) true`.
   - Multi-character operators (<=, >=, ++, **, //) use longest-match.
   - A minus sign is context-sensitive: after a value token
     (number, string, identifier, true/false/null/inf/nan, undefined, env, ")", "]", "}"),
     it is the binary subtraction operator. Otherwise, a minus immediately
     followed by digits, "inf", or "nan" is a single negative numeric literal.
   - Keywords are only reserved when they appear as complete tokens,
     not as substrings of identifiers (e.g., "island" is a valid identifier).  *)

(* === Identifier ===
   An identifier is any contiguous sequence of non-whitespace
   characters not containing any of the following token boundaries:
   { } [ ] ( ) , . " ' @ + - * / % ^ < > = ! ? : ; | & $ ~ # \ //
   Tokens matching numeric literal grammar are parsed as numbers.
   Keywords in identifier position must be escaped with @.
   QUOTED IDENTIFIERS: An identifier starting with ' and ending with '
   may contain any characters (including spaces and token boundaries)
   between the quotes. The enclosing quotes are syntax — 'key' and key
   are the same name. If the unquoted name is a keyword, it remains a
   keyword (use @ to escape instead). An unmatched ' (no closing ' on
   the same line) is a lexer error (§2.3).                                  *)
DIGIT           = "0"…"9" ;
HEX             = DIGIT | "A"…"F" | "a"…"f" ;
OCT             = "0"…"7" ;
BIN             = "0" | "1" ;
NEWLINE         = "\n" | "\r\n" ;
```

---

## 10. File Format

| Property  | Value              |
| --------- | ------------------ |
| Extension | `.uzon`            |
| MIME type | `application/uzon` |
| Encoding  | UTF-8              |

A UZON file is an anonymous struct containing zero or more bindings.

---

## 11. Conformance

### 11.1 Conformance Levels

| Level            | Requirements                                                                |
| ---------------- | --------------------------------------------------------------------------- |
| **Parser**       | MUST parse all syntactically valid documents per §9.                        |
| **Evaluator**    | MUST evaluate all expressions per §5. MUST resolve identifiers and `env`.        |
| **Type-checker** | MUST validate `as` annotations, `called`/`as` type reuse, `to` conversions. |
| **Full**         | Parser + Evaluator + Type-checker.                                          |

### 11.2 Error Handling

Parsers **MUST** reject syntactically invalid documents with a clear error message indicating the location of the error. Evaluators **MUST** report errors for circular dependencies and invalid `to` conversions.

#### 11.2.0 Error Location

All error messages **MUST** include the location where the error was detected. The required location fields depend on the input source:

**File input** (parsing a `.uzon` file): The error message **MUST** include the **file name**, **line number**, and **column number**.

**String input** (parsing UZON from a string in a host language): The error message **MUST** include the **line number** and **column number**.

**Imported files** (`struct` imports): When an error originates inside a file imported via `struct`, the error message **MUST** include the **file name**, **line number**, and **column number** of the imported file where the error occurred — not the import site. If the error propagates to the importing file (§11.2 error propagation), implementations **SHOULD** additionally include the import chain to help the user trace the error.

```
// Error in imported file — MUST show the imported file's location
./database.uzon:3:12 — error: 'port' needs a number, but you gave it
a text value ("8080"). Try removing the quotes.

// Import chain — SHOULD be included for propagated errors
  imported from ./main.uzon:7:1 (db is struct "./database")
```

Line numbers are 1-based. Column numbers are 1-based and count Unicode scalar values (not bytes) from the start of the line.

Implementations **SHOULD** produce error messages that are understandable to non-programmers. UZON's human-first philosophy extends to error reporting. When a keyword appears at binding-name position (e.g., `type is 5`), implementations **SHOULD** suggest the `@` escape syntax in the error message (e.g., "did you mean `@type is 5`?") — a user who writes configuration files should be able to understand what went wrong without consulting the language specification.

```
Bad:  TypeError: Expected i32, found string at line 5, col 12.
Good: Line 5: 'port' needs a number, but you gave it a text value ("8080"). 
      Try removing the quotes.
```

**Error propagation**: If a binding's expression produces an error (e.g., division by zero, overflow, invalid conversion), the error propagates to any binding that references it. The evaluator **MUST** report the original error location, not the reference site. Circular dependency detection takes priority over expression evaluation — the evaluator **MUST** detect and report cycles before attempting to evaluate any expressions.

**Error priority**: When multiple error conditions apply, implementations **MUST** prioritize in this order: (1) syntax errors (parsing), (2) circular dependency errors (static analysis), (3) type errors (type checking), (4) runtime errors (evaluation — overflow, division by zero, invalid conversion).

```uzon
a is 1 / 0               // error: division by zero
b is a + 1          // error: propagated from a
```

#### 11.2.1 Required Rejections

This list highlights common rejection requirements. All **MUST**, **MUST NOT**, and error requirements throughout this specification are equally normative — this list is illustrative, not exhaustive.

Conformant implementations **MUST** reject the following:

- Chained `is` equality: `a is b is c` (syntax error — use parentheses)
- Chained `with`: `a with { } with { }` (syntax error — use intermediate binding)
- Chained `plus`: `a plus { } plus { }` (syntax error — use intermediate binding)
- `plus` with no new fields: all fields already exist in base (type error — use `with` instead)
- Type-incompatible override in `plus`: overridden field must match original type (type error)
- Duplicate names in the same scope: `x is 1, x is 2`
- Duplicate type names: `... called T` where `T` is already declared
- Mixed types in a list: `[1, "hello", true]`
- `undefined` as literal binding: `x is undefined` (type error — §4.5)
- `undefined` in string interpolation: `"{env.MISSING}"`
- `undefined` in arithmetic: `env.OFFSET + 1`
- `is named` or `case named` on non-tagged-union values (for untagged unions, use `is type` / `case type` instead)

- New fields in `with` override: `base with { unknown_field is 1 }`
- Type-incompatible `with` override: `base with { port is "8080" }` when `port` is `u16`
- Invalid or overflowing `to` conversions
- Mismatched branch types in `if`/`case`: branches returning different types
- Mismatched types in `or else`: left and right operands of different types (null exempted)
- Collection operators (`++`, `**`) on tuples or structs

- Single-variant enums, single-type unions, or single-variant tagged unions (all require at least two)
- `when undefined then ...` in `case` expressions (type error — §4.5)
- `case` with no `when` clauses (at least one required)
- Standalone `env` without member access (`.name` required)
- `called` inside control flow branches, parenthesized subexpressions, or nested contexts
- Interpolation in `struct` import paths (paths must be static string literals)
- Circular dependencies in data bindings (circular dependency error — §11.2.0)
- Recursive function calls (direct or mutual — call graph must be DAG; circular dependency error — §11.2.0)
- Function parameters without explicit `as` type annotation
- Required parameter after a defaulted parameter
- Comparing function values with `is` or `is not`
- Ordering function values with `<`, `<=`, `>`, or `>=`
- Calling a non-function value
- Function call with wrong argument count or mismatched types

### 11.3 Conformance Test Suite

A conformance test suite is maintained at `github.com/uzon-dev/conformance` containing:

- `parse/valid/` — Documents that parsers and evaluators **MUST** accept. Each file tests one feature.
- `parse/valid/cross/` — Documents that test interactions between multiple features. Each file exercises two or more spec sections simultaneously.
- `parse/valid/starship/` — A multi-file integration test (`shared.uzon`, `crew.uzon`, `starship.uzon`) that exercises all 33 keywords (including 1 reserved) and 17 of 24 `std` functions in a cohesive scenario with diamond imports, struct overrides, extensions, tagged unions, functions, and chained `std` operations.
- `parse/invalid/` — Documents that parsers and evaluators **MUST** reject. Each file tests one error condition.
- `parse/invalid/cross/` — Documents that test error conditions arising from feature interactions.
- `parse/invalid/starship/` — Realistic starship-like documents, each containing exactly one subtle error arising from complex feature interactions — type mismatches through conversion chains, nominal identity conflicts, indirect recursion through `std.map`, undefined propagation through member access into interpolation, null passing through `or else` into concatenation, and similar edge cases.
- `eval/` — Documents with expected evaluation results.
- `roundtrip/` — Documents for testing parse → serialize → parse identity.

Each test file begins with a comment identifying the spec section(s) under test (e.g., `// §3.2.1 + §5.7: with + or else`). Cross-feature tests are the most valuable — they verify behavior at the boundaries where multiple rules interact, which is where implementations are most likely to diverge.

---

## Appendix A: Complete Example

```uzon
// ══════════════════════════════════════════════════════════════════
//   Observatory Configuration — Dune of Celestial Cartography
//   A station on the edge of a desert, mapping stars for travelers
//   who have not yet departed.
// ══════════════════════════════════════════════════════════════════

// ── the observatory ─────────────────────────────────────────────

name is "Star Observatory"
epoch is 7
longitude is -117.8531 as f64
latitude is 33.6694 as f64
operational is env.SHUTDOWN is undefined

// ── the sky ─────────────────────────────────────────────────────

season is autumn from spring, summer, autumn, winter called Season
time_of_day is night from dawn, day, dusk, night called TimeOfDay

visibility is case time_of_day
    when night then "stellar"
    when dusk then "partial"
    when dawn then "partial"
    else "solar only"

seeing_conditions is env.SEEING to f64 or else 1.2

// ── the instruments ─────────────────────────────────────────────

base_telescope is {
    aperture_mm is 200 as u16
    focal_ratio is 8.0 as f64
    focal_length_mm is aperture_mm to f64 * focal_ratio
    tracking is true
    filter is none from none, hydrogen_alpha, oxygen_iii, luminance called Filter
}

primary is base_telescope with { aperture_mm is 400 as u16 }
secondary is base_telescope with { aperture_mm is 80 as u16, tracking is false }
hydrogen is base_telescope with { filter is hydrogen_alpha as base_telescope.Filter }

adaptive is base_telescope plus { ao_enabled is true, guide_star is "Polaris" }

active_scope is if seeing_conditions < 1.0
    then primary
    else secondary

focal_length is active_scope.focal_length_mm

// ── the catalog ─────────────────────────────────────────────────

_catalog is development from development, staging, production called CatalogEnv
catalog_env is env.CATALOG to CatalogEnv or else development as CatalogEnv

catalog is struct "./catalog"
db_host is of struct "./database"

targets are "Andromeda", "Orion Nebula", "Crab Nebula", "Pleiades" as [string]
priority_targets is targets ++ if env.EXTRA_TARGET is undefined
    then [] as [string]
    else [ env.EXTRA_TARGET ]

coordinates are
    (10.6847, 41.2687),
    (83.8221, -5.3911),
    (83.6331, 22.0145),
    (56.8712, 24.1052)

first_target is targets.first
first_ra is coordinates.first.0

// ── the observers ───────────────────────────────────────────────

max_observers is 2 ^ 3 as u8
shift_hours is 24 / max_observers

observer is {
    name is env.OBSERVER or else "unnamed"
    credential is env.CREDENTIAL or else null
    role is astronomer from astronomer, engineer, guest called Role
}

has_credential is observer.credential is not null
is_authorized is has_credential and observer.role is not guest as observer.Role

// ── the mission ─────────────────────────────────────────────────

mission_status is "mapping sector 7G" named active from
    active as string, paused as string, completed as null
    called MissionStatus

status_report is if mission_status is named active
    then "IN PROGRESS: {mission_status}"
    else if mission_status is named paused
    then "PAUSED: {mission_status}"
    else "COMPLETED"

alert_level is case seeing_conditions to i32
    when 0 then "perfect"
    when 1 then "good"
    when 2 then "fair"
    else "poor"

// ── the logbook ─────────────────────────────────────────────────

_log_level is info from trace, debug, info, warn, error called LogLevel

verbosity is case catalog_env
    when production then warn as LogLevel
    when staging then debug as LogLevel
    else trace as LogLevel

is_verbose is verbosity in [ trace, debug ] as [LogLevel]

divider is "─" ** 60

greeting is "{name} · epoch {epoch}"
         "longitude {longitude} · latitude {latitude}"
         "{divider}"
         "observer: {observer.name}"
         "mission: {status_report}"
         "seeing: {seeing_conditions} arcsec"

// ── the computations ────────────────────────────────────────────

angularDistance is function ra1 as f64, dec1 as f64, ra2 as f64, dec2 as f64 returns f64 {
    delta_ra is ra1 - ra2
    delta_dec is dec1 - dec2
    delta_ra ^ 2 + delta_dec ^ 2
}

formatTarget is function name as string, ra as f64, dec as f64 returns string {
    "{name} at ({ra}, {dec})"
} called TargetFormatter

target_labels is std.map(
    targets,
    function t as string returns string { "★ " ++ t }
)

bright_targets is std.filter(
    coordinates,
    function c as (f64, f64) returns bool { c.0 < 80.0 }
)

total_ra is std.reduce(
    coordinates,
    0.0,
    function acc as f64, c as (f64, f64) returns f64 { acc + c.0 }
)

target_count is std.len(targets)
has_andromeda is "Andromeda" in targets
second_target is std.get(targets, 1)

seeing_ok is std.isFinite(seeing_conditions)

currentStatus is function returns string {
    if seeing_ok then "operational" else "degraded"
}

greetObserver is function greeting as string default "Welcome" returns string {
    "{greeting}, {observer.name}!"
}

welcome_message is greetObserver()
custom_message is greetObserver("Good evening")

// ── the archive ─────────────────────────────────────────────────

archive is {
    @type is "observation_log"
    @default is false
    'Content-Type' is "application/uzon"
    format_version is epoch
    writable is is_authorized
}

// ── the constants ───────────────────────────────────────────────

speed_of_light is 299_792_458
planck is 6.626e-34 as f64
golden_ratio is 1.618_033_988 as f64
earth_radius_km is 6_371 as u16
hex_marker is 0xFF
binary_flags is 0b1010_0101
permissions is 0o755
```

## Appendix B: UZON vs JSON vs TOML vs YAML

| Feature                  | UZON | JSON | TOML | YAML |
| ------------------------ | ---- | ---- | ---- | ---- |
| Comments                 | ✓    | ✗    | ✓    | ✓    |
| Trailing commas          | ✓    | ✗    | N/A  | N/A  |
| Typed enums              | ✓    | ✗    | ✗    | ✗    |
| Unions / tagged unions   | ✓    | ✗    | ✗    | ✗    |
| Tuples                   | ✓    | ✗    | ✗    | ✗    |
| Named/reusable types     | ✓    | ✗    | ✗    | ✗    |
| Conditional expressions  | ✓    | ✗    | ✗    | ✗    |
| Arithmetic expressions   | ✓    | ✗    | ✗    | ✗    |
| Functions                | ✓    | ✗    | ✗    | ✗    |
| Standard library         | ✓    | ✗    | ✗    | ✗    |
| Self-references          | ✓    | ✗    | ✗    | ✓    |
| Environment variables    | ✓    | ✗    | ✗    | ✗    |
| File imports             | ✓    | ✗    | ✗    | ✗    |
| Type conversions         | ✓    | ✗    | ✗    | ✗    |
| Case expressions         | ✓    | ✗    | ✗    | ✗    |
| Self-describing          | ✓    | ✓    | ✓    | ✗    |
| Integer bases (hex etc.) | ✓    | ✗    | ✓    | ✓    |
| String/list concat       | ✓    | ✗    | ✗    | ✗    |
| String interpolation     | ✓    | ✗    | ✗    | ✗    |
| Field extraction         | ✓    | ✗    | ✗    | ✗    |

## Appendix C: Keyword and Operator Quick Reference

```
is               assign / equals        x is 5, x is 5 → true
are              list binding sugar     layouts are a, b, c (= is [ a, b, c ])
is not           not equals             x is not 0 → true
as               type annotation        42 as i32, blue as RGB
to               type conversion        env.PORT to u16, 3.14 to i32
of               field extraction       port is of config
from             define enum/union      green from red, green, blue
union            define union types     from union i32, f64, string
called           name a type            ... called RGB
named            tag/dispatch variant   7 named ln from ..., case named x
is named         check variant tag      x is named ok
is not named     negated variant check  x is not named ok
with             struct override        base with { port is 443 }
plus             struct extension       base plus { tls is true }
if/then/else     conditional            if x then a else b
case/when        multi-match            case x when 1 then ... else ...
and              logical AND            a and b
or               logical OR             a or b
or else          undefined coalescing   env.PORT to u16 or else 8080
not              logical NOT            not a
in               membership test        x in [ 1, 2, 3 ], x in (1, "a"), x in struct
env              environment variable   env.PORT
struct           file import            struct "./path"
function         function value         function n as i32 returns bool { ... }
returns          function return type   function n as i32 returns bool { ... }
default          parameter default      function n as i32 default 0 returns i32 { ... }
true             boolean true           true
false            boolean false          false
null             null value             null (intentionally empty)
undefined        missing state          undefined (unbound name, env.UNSET, out-of-bounds)
inf              positive infinity      inf (float keyword)
nan              not a number           nan (float keyword)
type             type check             x is type i32, case type x
lazy             (reserved)             reserved for future use
++               concatenation          "a" ++ "b", [ 1 ] ++ [ 2 ]
**               repetition             "*" ** 3, [ 0 ] ** 4
^                exponentiation         2 ^ 10 → 1024
< <= > >=        comparison             1 < 2, x >= 0
@                keyword escape         @is is 3
```

---

## Appendix D: Implementer's Quick Reference

This appendix consolidates rules that are distributed across multiple sections of the specification. It is non-normative — the referenced sections are authoritative.

### D.1 `null` Behavior Summary

`null` is a **value** representing intentional absence. It is not `undefined`.

| Context                                                  | Behavior                                                          | Reference   |
| -------------------------------------------------------- | ----------------------------------------------------------------- | ----------- |
| Binding                                                  | `x is null` — valid                                               | §3.1        |
| Member access                                            | `null.foo` — **type error** (not undefined)                            | §5.12       |
| `is` / `is not`                                          | Compatible with any type                                          | §5.2        |
| `or else`                                                | Passes through (null is not undefined)                            | §5.7        |
| `if`/`case` branches                                     | Compatible with any type across branches                          | §5.9, §5.10 |
| `with` override                                          | Bidirectional — any→null, null→any                                | §3.2.1      |
| List element                                             | Adopts type from non-null elements; all-null requires `as [Type]` | §3.4        |
| `in` operator                                            | Exempt from type constraint on either side                        | §5.8.1      |
| Arithmetic, comparison, concatenation, repetition, logic | **Type error**                                                    | §4.5, §5.3, §5.4, §5.6, §5.8 |
| `to string`                                              | `"null"`                                                          | §5.11.2     |
| `to null`                                                | Identity (permitted)                                              | §5.11.0     |
| `to` any other type                                      | **Type error**                                                    | §5.11.0     |

### D.2 `undefined` Behavior Summary

`undefined` is a **state** (not a value) resulting from accessing something that does not exist. Errors arising from `undefined` in operator contexts are **runtime errors** — they are suppressed in non-selected branches during speculative evaluation (§5.9), enabling patterns like `if x is undefined then 0 else x + 1`. There are two exceptions where the literal `undefined` produces a **type error** (§4.5) instead, always reported regardless of branch selection: `x is undefined` as a binding value, and `when undefined then ...` as a `case` match pattern.

| Context                                   | Behavior                                           | Reference    |
| ----------------------------------------- | -------------------------------------------------- | ------------ |
| Binding                                   | `x is undefined` — **type error** (literal)        | §4.5         |
| Expressions producing undefined           | unbound name, `env.UNSET`, out-of-bounds — valid | §5.12, §5.13 |
| `.` (member access)                       | Propagates                                         | §5.12        |
| `to` (conversion)                         | Propagates                                         | §5.11        |
| `as` (annotation)                         | Propagates (type name still validated)             | §6.1         |
| `or else`                                 | Returns right operand                              | §5.7         |
| `is` / `is not`                           | Permitted (returns bool)                           | §5.2         |
| Arithmetic (`+`, `-`, `*`, `/`, `%`, `^`) | **Runtime error**                                  | §5.3         |
| Concatenation (`++`), repetition (`**`)   | **Runtime error**                                  | §5.8         |
| Comparison (`<`, `<=`, `>`, `>=`)         | **Runtime error**                                  | §5.4         |
| Logic (`and`, `or`, `not`)                | **Runtime error**                                  | §5.6         |
| `in` (either operand)                     | **Runtime error**                                  | §5.8.1       |
| `with`                                    | **Runtime error**                                  | §3.2.1       |
| `plus`                                 | **Runtime error**                                  | §3.2.2       |
| `is named`                                | **Runtime error**                                  | §3.7.2       |
| `is type` / `is not type`                 | **Runtime error**                                  | §5.2         |
| String interpolation                      | **Runtime error** (terminal context)               | §5.11.2      |
| `case` match value                        | **Runtime error**                                  | §5.10        |
| `when undefined then ...` (literal)       | **Type error** (literal, not matchable)            | §5.10, §4.5  |
| `()` (function call)                      | **Runtime error**                                  | §5.15        |

### D.3 Type Compatibility Rules

"Same type" means **exactly the same** — `i32` ≠ `i64`, `u8` ≠ `i8`, `f32` ≠ `f64`.

| Rule                                        | Where it applies                                                            | Reference                                     |
| ------------------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------- |
| Same numeric type required                  | Arithmetic, comparison                                                      | §5.3, §5.4                                    |
| Untyped literal adopts typed operand's type | Everywhere same-type is required                                            | §5                                            |
| `null` compatible with any type             | `is`/`is not`, `or else`, `if`/`case` branches, `with`, `in`, list elements | §5.2, §5.7, §5.9, §5.10, §3.2.1, §5.8.1, §3.4 |
| Speculative evaluation for type checking    | `if`/`case` branches, `or else`, `and`/`or`                                 | §5.9                                          |
| Structural compatibility for structs        | `with`, `plus`, `as NamedStructType` — field names + types must match    | §3.2.1, §3.2.2, §6.3                          |
| Structural compatibility for unions         | Same member type set = same type (order irrelevant)                         | §3.6                                           |
| Named type compatibility is nominal         | `called` types match by name, not structure                                 | §3.2.1 rule (5)                               |

### D.4 Tagged Union Transparency

Tagged unions are transparent for **most** operators but not all.

| Operator                            | Sees                                     | Reference    |
| ----------------------------------- | ---------------------------------------- | ------------ |
| `.` (member access)                 | Inner value                              | §3.7.1       |
| Arithmetic (`+`, `-`, etc.)         | Inner value                              | §3.7.1       |
| Concatenation (`++`)                | Inner value                              | §3.7.1       |
| String interpolation (`{...}`)      | Inner value                              | §3.7.1       |
| Comparison with non-tagged-union    | Inner value                              | §3.7.1, §5.4 |
| Ordered comparison (two tagged)     | **Type error**                           | §5.4         |
| `is` / `is not` (two tagged unions) | **Wrapper** (tag + inner value)          | §3.7.2       |
| `is` / `is not` (tagged vs plain)   | **Type error** (except `null`/`undefined`) | §3.7.2, §5.2 |
| `in`                                | **Wrapper** (same semantics as `is`)     | §5.8.1       |
| `is named` / `is not named`         | **Wrapper** (checks tag)                 | §3.7.2       |
| `case named`                        | **Wrapper** (checks tag)                 | §3.7.2       |
| `to`                                | **Wrapper** (only `to string` permitted) | §5.11.0      |
| `is type` / `is not type`           | Inner value (checks inner value's type)  | §5.2         |
| `with`                              | **Type error** (requires struct)         | §3.2.1       |
| `plus`                           | **Type error** (requires struct)         | §3.2.2       |

### D.5 Speculative Evaluation

UZON has no separate static type system. Type checking is performed by speculative evaluation:

1. **All** branches in `if`/`case`, both sides of `or else`, and both operands of `and`/`or` are evaluated.
2. **Runtime errors** (division by zero, overflow, invalid conversion) in non-selected branches are **suppressed**.
3. **Type errors** (type mismatch, incompatible operands) are **always reported**, regardless of selection.
4. Only the selected branch's **value** is used as the result.

Reference: §5.9

### D.6 Error Classification

| Category            | Examples                                                                                                                  | When                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| Syntax error        | Missing `else`, chained `is`, chained `plus`, duplicate bindings                                                       | Parsing                                |
| Circular dependency | `a → b → a` in dependency graph                                                                                           | Static analysis (before evaluation)    |
| Type error          | Mismatched branch types, incompatible `with`/`plus` override, `plus` with no new fields, disallowed `to` conversion | Type checking (speculative evaluation) |
| Runtime error       | Division by zero, integer overflow, `inf to i32`, out-of-range `to` conversion                                            | Evaluation                             |

Error priority: syntax → circular → type → runtime (§11.2).

### D.7 Minimum Count Requirements

| Construct             | Minimum | Reference |
| --------------------- | ------- | --------- |
| Enum variants         | 2       | §3.5, §9  |
| Union member types    | 2       | §3.6, §9  |
| Tagged union variants | 2       | §3.7, §9  |
| `case` `when` clauses | 1       | §5.10, §9 |

### D.8 Function Rules

Functions are **values** — they follow the same binding rules as all other values.

| Rule                             | Description                                                                                                             | Reference    |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ------------ |
| Functions are values             | Bound with `is`, stored in lists, passed as arguments                                                                   | §3.8         |
| Zero parameters permitted        | `function returns type { ... }` is valid — useful for deferred computation                                              | §3.8         |
| Parameter access is bare         | Parameters and local bindings use bare identifiers — visible in nested structs and string interpolation     | §3.8, §4.4.1 |
| Parameter types required         | When parameters are present, every parameter **MUST** have explicit `as` type annotation                                | §3.8         |
| Default parameters trailing      | Parameters with `default` **MUST** appear after all required parameters                                                 | §3.8         |
| Last expression is return value  | No `return` keyword — last expression in body is the result                                                             | §3.8         |
| No recursion                     | Call graph **MUST** be a DAG — static cycle detection                                                                   | §3.8         |
| Self-exclusion applies           | `x` inside function `x` skips the function binding, searches parent                                                | §3.8, §5.12  |
| Function body is a scope layer   | Nested struct walks: struct → function body (params + locals) → enclosing → file                            | §3.8         |
| No side effects                  | Pure — may read scope and `env`, cannot modify bindings or perform I/O                                                 | §3.8         |
| No function equality             | Comparing functions with `is`/`is not` or ordering with `<`/`<=`/`>`/`>=` is a type error                               | §3.8         |
| `called` names function types    | Captures parameter types + return type as a named type                                                                  | §3.8         |
| Type compatibility is structural | Same parameter types (in order) + same return type = compatible for `as`                                                | §3.8         |
| Named function types are nominal | Two separately `called` function types with same signature are different                                                | §3.8, §3.2.1 |
| `std` is an implicit binding     | Not a keyword — provides `len`, `reverse`, `get`, `hasKey`, `keys`, `values`, `all`, `any`, `filter`, `map`, `reduce`, `sort`, `isNan`, `isInf`, `isFinite`, `contains`, `endsWith`, `join`, `lower`, `replace`, `split`, `startsWith`, `trim`, `upper` | §5.16        |
| `std` functions are pure         | Produce new values, guaranteed to terminate                                                                             | §5.16        |

---

## Appendix E: Style Guide (Non-normative)

This appendix provides recommended conventions for writing consistent, readable UZON documents. These guidelines are not enforced by parsers — they are recommendations for teams and projects.

### E.1 Naming

UZON uses the following naming conventions:

| Kind                                  | Convention   | Example                              |
| ------------------------------------- | ------------ | ------------------------------------ |
| Type names (`called`)                 | `PascalCase` | `LogLevel`, `Point3D`, `SecureBase`  |
| Binding names                         | `snake_case` | `base_port`, `active_scope`          |
| Function names                        | `camelCase`  | `formatTarget`, `angularDistance`    |
| Functions returning a type (`called`) | `PascalCase` | follows the type convention          |
| Enum variants                         | `snake_case` | `red`, `hydrogen_alpha`, `dark_mode` |

Acronyms and abbreviations lose their casing in identifiers: `XmlParser`, `HttpPort`, `HtmlConfig` — not `XMLParser`, `HTTPPort`, `HTMLConfig`.

**Rationale**: Casing encodes role — you can tell whether an identifier is a type, a value, or a function without looking at its definition. This is especially valuable in UZON where types (`called`), values (`is`), and functions (`function`) coexist. Flattening acronyms prevents inconsistency (`XMLParser` vs `XmlParser` vs `xmlparser`) and ensures identifiers sort predictably.

### E.2 Formatting

**Indentation**: 4 spaces. Tabs are not recommended.

**Rationale**: UZON config files are shared across teams, editors, and environments. Spaces render identically everywhere. 4 spaces provide clear visual nesting without excessive horizontal drift in deeply nested structs.

**Line length**: 80 columns maximum. Exceptions are permitted when a single semantic unit (e.g., a long string literal, a URL, or a deeply nested expression) would be less readable if broken.

**Rationale**: 80 columns ensures readability in side-by-side diffs, terminal windows, and embedded documentation. The exception clause prevents absurd breaks like splitting a URL across two lines.

**Curly braces** (`{ }`): When a struct contains elements, add one space after `{` and one space before `}` for single-line structs. Multi-line structs place `{` at the end of the opening line and `}` on its own line.

**Rationale**: The inner spaces visually separate the delimiters from the content, preventing `{x is 1}` from reading as a dense block. Multi-line structs follow the universal convention of opening brace on the same line to avoid wasting a line on a lone `{`.

```uzon
// Single-line — spaces inside braces
point is { x is 0, y is 0 }

// Multi-line — opening brace on same line, closing on own line
server is {
    host is "localhost"
    port is 8080
}
```

**Square brackets** (`[ ]`): Same rule as curly braces — one space after `[` and one space before `]` for single-line lists with elements. Empty lists have no spaces.

**Rationale**: Consistent with curly brace spacing. `[ 1, 2, 3 ]` is easier to scan than `[1, 2, 3]` — the spaces give the elements room to breathe.

```uzon
ids is [ 1, 2, 3 ]
empty is [] as [i32]

tags are "alfa", "bravo", "charlie"
```

**Trailing commas**: Avoid trailing commas by default. When elements are written on separate lines in struct `{ }` or list `[ ]` literals, either use commas on every line or omit commas on every line — do not mix. This rule excludes constructs where a trailing comma would produce a grammar error — `are` bindings, enum variants (`from`), union members (`from union`), and tagged union variants (`named ... from`) require commas between elements but do not permit a trailing comma.

**Rationale**: Mixing commas and newline separators in the same block creates visual noise and makes it easy to introduce parsing errors when reordering lines. Consistency within each block is the priority.

```uzon
// Good — no trailing comma
point is { x is 0, y is 0 }

// Good — multi-line, no commas (newline separates)
server is {
    host is "localhost"
    port is 8080
}

// Good — multi-line with commas on every line
server is {
    host is "localhost",
    port is 8080,
}

// Bad — mixed commas
server is {
    host is "localhost",
    port is 8080
}

// Good — are binding, commas between elements, no trailing comma
targets are
    "Andromeda",
    "Orion Nebula",
    "Pleiades"
```

**Blank lines**: Use one blank line to separate semantic groups of bindings. Do not use multiple consecutive blank lines.

**Rationale**: A single blank line signals "these are different conceptual groups." Multiple blank lines waste vertical space without adding information.

### E.3 Binding Order

Organize bindings in this order within a file or struct:

1. `struct` imports
2. Everything else, ordered by dependency — definitions before uses

**Rationale**: Imports have no local dependencies and establish the external context, so they come first. For all other bindings — constants, computed values, and functions alike — place each definition before the bindings that reference it. Since UZON evaluates by dependency graph (not textual order), this convention is purely for readability: a reader scanning top-down encounters definitions before their uses. Functions are values like any other and do not need a separate section.

```uzon
// 1. imports
_shared is struct "./shared"
_database is struct "./database"

// 2. definitions before uses
name is "My Service"
port is env.PORT to u16 or else 8080

formatEntry is function key as string,
    value as string returns string {
    "{key}: {value}"
}

base_url is "http://{name}:{port}"
db_host is of _database
summary is formatEntry("host", base_url)
```

### E.4 Prefer `are` for Lists

Use `are` instead of `is [...]` for list bindings. It reads more naturally as English.

**Rationale**: `targets are "Andromeda", "Orion Nebula", "Pleiades"` reads like a sentence. `targets is [ "Andromeda", "Orion Nebula", "Pleiades" ]` reads like code. UZON's human-first philosophy favors the former.

```uzon
// Good
targets are "Andromeda", "Orion Nebula", "Pleiades"

// Acceptable but less idiomatic
targets is [ "Andromeda", "Orion Nebula", "Pleiades" ]
```

### E.5 Internal Bindings

Prefix binding names with `_` when they are not intended for external consumption — they exist only as intermediate values or imports used internally.

**Rationale**: When another file imports this document via `struct`, all bindings are visible. The `_` prefix is a social contract — "this is an implementation detail, not part of the public interface." It is the same convention used in Python and other languages without access modifiers.

```uzon
_shared is struct "./shared"
_base_config is { host is "localhost", port is 8080 }

// public-facing bindings
production is _base_config with { port is 443 }
```

### E.6 String Interpolation vs Concatenation

Use string interpolation when embedding values in text. Use `++` only for pure string-to-string concatenation without surrounding text.

**Rationale**: Interpolation preserves the visual shape of the output string — you can see the template at a glance. Concatenation scatters the template across multiple fragments, making the final string shape hard to predict. However, when the operation is purely mechanical joining (no surrounding text), `++` is more honest — it says "these are separate pieces being joined," not "this is a template with holes."

```uzon
// Good — interpolation for embedded values
greeting is "Hello, {name}! Port: {port}"

// Good — concatenation for pure joining
full_name is first ++ " " ++ last

// Bad — concatenation where interpolation is clearer
greeting is "Hello, " ++ name
    ++ "! Port: " ++ port to string
```

### E.7 Comment Alignment

Inline comments within the same semantic group **SHOULD** be aligned to the same column.

**Rationale**: Aligned comments form a visual column that the eye can scan independently of the code. Unaligned comments create a ragged right edge that forces the reader to hunt for each comment's starting position.

```uzon
aperture_mm is 200 as u16        // primary mirror diameter
focal_ratio is 8.0 as f64        // f/8 Ritchey-Chrétien
tracking is true                 // equatorial mount enabled
```

---

*End of UZON Specification v0.8*
