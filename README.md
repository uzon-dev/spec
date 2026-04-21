# UZON

**A typed configuration language that reads as data at the simple end and scales to a composed, imported system at the other — in one grammar, one type system, one evaluator.**

- **File Extension**: `.uzon`
- **MIME Type**: `application/uzon`
- **Encoding**: UTF-8
- **Specification**: [SPECIFICATION.md](SPECIFICATION.md)
- **Status**: Pre-release — breaking changes possible before v1.0.

> This README is a non-normative overview intended to motivate reading the full specification. It is a marketing document, not the language definition. For conforming behavior, see [SPECIFICATION.md](SPECIFICATION.md).

---

## A first look

```uzon
name    is "Star Observatory"
epoch   is 7
region  is env.REGION or else "us-west-2"

Mode    is enum dev, staging, prod
mode    is staging as Mode

Telescope is struct {
    aperture_mm     is 0 as u16
    focal_ratio     is 0.0
    focal_length_mm is 0.0
    tracking        is false
}

base    is { aperture_mm is 200 as u16, focal_ratio is 8.0, focal_length_mm is 1600.0, tracking is true } as Telescope
primary is base with { aperture_mm is 400 as u16, focal_length_mm is 3200.0 }

targets are "Andromeda", "Orion Nebula", "Crab Nebula" as [string]

active_scope is if env.SEEING to f64 or else 1.5 < 1.0
    then "primary"
    else "secondary"
```

Every UZON document is a single immutable value with no side effects, no loops, and no recursion. Every well-formed document terminates.

---

## Quick start

Install the UZON library for your language:

| Language                | Install                                                     |
| ----------------------- | ----------------------------------------------------------- |
| Python                  | `pip install uzon`                                          |
| TypeScript / JavaScript | `npm install @uzon/uzon`                                    |
| Rust                    | `cargo add uzon`                                            |
| Go                      | `go get github.com/uzon-dev/uzon-go`                        |
| Zig                     | `zig fetch --save git+https://github.com/uzon-dev/uzon-zig` |

Write a UZON document:

```uzon
// hello.uzon
greeting is "Hello, UZON"
port     is env.PORT to u16 or else 8080
```

Read it from your language of choice. Each binding follows its host language's idioms — Python dicts, Rust `Option` / `Result`, Go `(value, ok)` pairs, Zig allocator-based parsing, TypeScript property access — so the APIs intentionally differ rather than forcing a uniform surface across languages.

**Python**

```python
import uzon

doc = uzon.load("hello.uzon")
print(doc["greeting"])   # Hello, UZON
print(doc["port"])       # 8080
```

**TypeScript / JavaScript**

```typescript
import { parseFile } from "@uzon/uzon";

const doc = parseFile("hello.uzon", { native: true });
console.log(doc.greeting);   // Hello, UZON
console.log(doc.port);       // 8080
```

**Rust**

```rust
use std::path::Path;
use uzon::from_path;

let doc = from_path(Path::new("hello.uzon"))?;
println!("{}", doc["greeting"].as_str().unwrap());   // Hello, UZON
println!("{}", doc["port"].as_i64().unwrap());       // 8080
```

**Go**

```go
import "github.com/uzon-dev/uzon-go"

doc, err := uzon.ParseFile("hello.uzon")
if err != nil { log.Fatal(err) }

greeting, _ := doc.GetPath("greeting").AsString()
port, _     := doc.GetPath("port").AsInt()
fmt.Println(greeting) // Hello, UZON
fmt.Println(port)     // 8080
```

**Zig**

```zig
const std = @import("std");
const uzon = @import("uzon");

var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();

const result = uzon.parseFile(arena.allocator(), "hello.uzon");
switch (result) {
    .value => |doc| {
        std.debug.print("{s}\n", .{doc.struct_val.get("greeting").?.string});    // Hello, UZON
        std.debug.print("{d}\n", .{doc.struct_val.get("port").?.integer.value}); // 8080
    },
    .err => |e| e.dump(),
}
```

---

## 1. Introduction

UZON is a typed, human-centric configuration language. Every UZON document — a single `.uzon` file — evaluates to a single immutable value: an anonymous struct containing the file's top-level bindings. Evaluation is pure (no side effects), total (no recursion, no loops — every well-formed document terminates), and self-describing (every value carries its type). Everything in UZON is a value: primitives (booleans, integers, floats, strings, `null`), containers (structs, tuples, lists), sum types (enums, unions, tagged unions), and functions are all first-class values that can be bound, passed, and composed. The language is deliberately minimal — every construct earns its place — yet expressive enough to span conditionals, arithmetic, function abstraction, environment reads, and cross-file composition within a single grammar.

The following subsections describe UZON's distinguishing properties — natural-language readability, scale-continuity as projects grow, modular decomposition across files, direct correspondence with typed programming languages, deterministic evaluation, and an unambiguous syntactic surface.

### 1.1 Natural-language readability

UZON uses English words as its operators — `is`, `are`, `as`, `to`, `of`, `from`, `with`, `plus`, `or else`, `when`, `then`, `else` — so a UZON document reads as the sentences that describe it.

Flat declarations read as plain statements of fact:

```uzon
package      is "hanriver"
version      is "0.3.1"
contributors are "Alice", "Bob", "Carol"
released     is true
```

Typed values, conversions, and fallbacks compose as directed prose:

```uzon
Mode is enum dev, staging, prod

port   is env.PORT to u16 or else 8080
region is env.REGION or else "us-east-1"
mode   is dev as Mode
```

Multi-branch dispatch follows the shape of an English conditional:

```uzon
severity is case exit_code
    when 0 then "ok"
    when 1 then "warning"
    when 2 then "error"
    else        "unknown"
```

### 1.2 Scale-continuity

Real configuration rarely stays small. A service begins as a handful of keys and grows into a composed system of typed schemas, environment-derived overrides, per-deployment overlays, and imported shared modules. JSON and TOML offer no way to express any of this inside the format: comments, environment reads, conditionals, arithmetic, and schema are all delegated to an external stack — templating engines, schema validators, shell-script preludes, per-tool DSLs. The moment configuration needs a branch or a typed default, the author leaves the format.

A UZON document may grow from a handful of bindings to a typed, composed, imported system without leaving the language. Every capability a growing configuration requires — comments, environment variables, conditionals, arithmetic, validation, composition, and abstraction — lives in the same grammar, type system, and evaluator:

| Capability              | UZON mechanism                                                                 | Typical external stack (JSON/YAML/TOML)            |
| ----------------------- | ------------------------------------------------------------------------------ | -------------------------------------------------- |
| Comments                | `//` line comments                                                             | not in JSON; stripped pre-parse                    |
| Environment variables   | `env.VAR`, composable with `or else` and `to`                                  | shell `envsubst`, Helm, Jinja, `sed`               |
| Conditional evaluation  | `if` / `then` / `else`; `case` with `when` / `type` / `named`                  | Go templates, Jsonnet, Jinja                       |
| Arithmetic / string ops | native operators, collection operators, standard library                       | external template language                         |
| Schema validation       | `as T` annotation, named type declarations                                     | JSON Schema, OpenAPI, Pydantic, ajv                |
| Composition / overlays  | `with`, `plus`, file import                                                    | Kustomize, Helm values hierarchy, merge scripts    |
| Abstraction             | first-class functions                                                          | not available; copy-paste across files             |

The document that begins as three lines of bindings and the document that ends as ten thousand lines of typed, composed, imported definitions are the same language processed by the same tool.

### 1.3 Modular decomposition

Most data formats have no notion of imports. TOML has no way to reference another file. JSON's `$ref` is a schema-layer convention, not part of the format. YAML anchors work only within a single document. In practice, configuration is split across files by concatenation — templating tools paste fragments together before the format ever sees them, and the resulting assembled artifact is what gets parsed.

Any binding, any type declaration, and any struct value can be moved into a separate `.uzon` file, and each fragment is already a complete UZON document. Because a file evaluates to an anonymous struct, a consumer imports the file as a struct value and references its contents through dot notation. Field extraction via `is of` accepts any member-access expression — including an inline file import — so a single field can be bound in one line without an intermediate name:

```uzon
timeout is of struct "./config"      // binds `timeout` from ./config.uzon
```

Named type declarations retain nominal identity across file boundaries — a type declared once is the same type wherever it is referenced through its declaring file. A ten-thousand-line system is realized as a hundred hundred-line files whose dependencies are expressed in the same language as their contents.

| Need                                   | UZON mechanism                                                      |
| -------------------------------------- | ------------------------------------------------------------------- |
| Import a file as a struct value        | `cfg is struct "./cfg"`                                             |
| Access a specific binding or type      | dot notation on the import: `cfg.port`, `cfg.Point`                 |
| Extract one field into the local scope | `port is of cfg` — or, inline, `port is of struct "./cfg"`          |
| Export a single non-struct value       | wrap in a binding (conventionally `_`); read via `struct "./f"._`   |
| Share a named type across files        | nominal identity preserved across imports                           |
| Validate a fragment in isolation       | any fragment is itself a complete UZON document                     |

#### Typical decomposition patterns

**1. Base configuration with targeted overrides.** A base file holds defaults; deployment- or environment-specific files override individual fields with `with` (changing existing fields) or `plus` (adding new ones).

```uzon
// base.uzon
host is "localhost"
port is 8080
tls  is false

// prod.uzon
config is struct "./base" with { host is "api.example.com", tls is true }

// dev.uzon — add a field the base does not declare
config is struct "./base" plus { verbose is true }
```

**2. Shared struct prototypes.** A common file provides prototype struct values that consumers override with `with` or `plus`. Every consumer starts from the same field layout and defaults, so shape drift across files is eliminated without declaring a named type.

```uzon
// defaults.uzon
endpoint is { host is "" as string, port is 0 as u16 }

// servers.uzon
defaults is struct "./defaults"
primary   is defaults.endpoint with { host is "a.local", port is 443 }
secondary is defaults.endpoint with { host is "b.local", port is 443 }
```

**3. Function library.** Validators, clampers, and format helpers are defined once as function values and invoked from each consumer. Importing the file does not execute the function bodies — evaluation happens only at the call site.

```uzon
// math_utils.uzon
clamp is function v as i32, lo as i32, hi as i32 returns i32 {
    if v < lo then lo else if v > hi then hi else v
}

// config.uzon
math is struct "./math_utils"
brightness is math.clamp(env.BRIGHTNESS to i32 or else 50, 0, 100)
```

**4. Per-domain decomposition.** A large system is split by concern — networking, security, UI, storage — with each domain in its own file. The entry file imports each as a subsystem binding, and the resulting document reads as a directory of responsibilities rather than a monolith.

```uzon
// main.uzon
network  is struct "./network"
security is struct "./security"
ui       is struct "./ui"
storage  is struct "./storage"
```

**5. Centralized environment-variable contract.** All `env` reads, their typed coercions, and their defaults are collected in one file. Consumers extract only the fields they need, and every environment-derived value in the system has exactly one canonical definition.

```uzon
// env_contract.uzon
port   is env.PORT to u16 or else 8080
region is env.REGION or else "us-east-1"
tls    is env.TLS is "true"

// server.uzon
env_config is struct "./env_contract"
port   is of env_config
region is of env_config
tls    is of env_config
```

### 1.4 Direct correspondence with typed programming languages

JSON and YAML have no concept of types beyond a small fixed set (object, array, string, number, boolean, null). Every richer construct — enums, tagged unions, sized integers, tuples, non-null discipline — is reconstructed ad-hoc in each consumer language through string conventions, comment pragmas, or external schemas. The same JSON document becomes a different shape in every consumer, and the schema lives outside the data.

UZON's value categories correspond directly to the data constructs of mainstream typed programming languages. An evaluated document embeds into a consumer's native type system without structural rewriting, and the type information carried by the document is the type information the consumer receives.

| UZON         | Rust                 | TypeScript           | Python                  | Zig            |
| ------------ | -------------------- | -------------------- | ----------------------- | -------------- |
| struct       | `struct`             | `interface` / object | `dataclass`             | `struct`       |
| tuple        | tuple                | `[T, U]`             | `tuple`                 | tuple          |
| list         | `Vec<T>`             | `T[]`                | `list[T]`               | `[]T`          |
| enum         | unit-variant `enum`  | string literal union | `Enum`                  | `enum`         |
| union        | wrapper `enum`       | `T \| U`             | `Union[T, U]`           | `union`        |
| tagged union | data-carrying `enum` | discriminated union  | `Union` + `Literal` tag | tagged `union` |
| function     | `fn` / closure       | function             | `Callable`              | `fn`           |
| null         | `Option::None`       | `null`               | `None`                  | `null` / `?T`  |

### 1.5 Deterministic, total evaluation

Configuration languages that admit recursion admit non-termination. Jsonnet is Turing-complete; so is Nix. A malformed or malicious input can hang the evaluator, and evaluation cost is unpredictable. CUE avoids recursion but accepts unbounded unification, which is powerful but not always obvious in its cost. At the other end, formats like JSON and YAML are total by having no evaluator at all — the cost is that every dynamic property belongs to an external tool, and those tools often re-introduce the termination problem they sidestepped.

UZON treats evaluation as a pure, total, deterministic function from a document — together with its imports and environment variables — to a single immutable value. Recursion and loops are forbidden outright; the combined dependency graph of binding references and function calls **must** be acyclic, and cycle detection is static, so every well-formed document terminates. Evaluation produces no side effects: the only external inputs are file imports and `env` reads, both explicit in source. Struct fields keep their declared order in the evaluated value — there is no merge-reordering step. Within a scope, bindings may reference one another regardless of textual position; the evaluator resolves references through the dependency DAG and rejects cycles before any value is computed.

| Guarantee                         | Mechanism                                                          | Affected formats / languages (for contrast) |
| --------------------------------- | ------------------------------------------------------------------ | ------------------------------------------- |
| Termination                       | no recursion, no loops; dependency DAG enforced statically         | Jsonnet, Nix (Turing-complete)              |
| Purity                            | inputs limited to file imports and `env` reads                     | Dhall imports can fetch remote URLs         |
| Field-order preservation          | struct fields retain declaration order in the evaluated value      | CUE unification may reorder                 |
| Order-independent authoring       | forward references resolved through the dependency DAG             | TOML's sectioned layout constrains order    |
| Single specified evaluation model | no lazy/strict distinction exposed to the author                   | Jsonnet's lazy semantics leak into errors   |

### 1.6 Unambiguous surface

JSON limits integers to 2⁵³ in practice — larger integers silently lose precision in most consumers. YAML 1.1 infamously parses `no`, `off`, and the country code `NO` as the boolean `false` (the "Norway problem"); YAML 1.2 fixed this but most deployed parsers still use 1.1 defaults, and implementations diverge. TOML requires two syntactically distinct constructs for arrays of tables (`[[x]]` header form vs `[{...}]` inline form) with different layout rules. JSON and YAML have no way to say "this is a `u48`" — every sized integer is a string convention.

UZON's syntactic surface is unambiguous by design. Every literal is self-describing: `yes` is an identifier or enum variant, `"yes"` is a string, and `true` is a boolean — no context turns one into another. Numeric types are explicit and width-specific: signed `i{N}` and unsigned `u{N}` for any bit width N in 0–65535, plus IEEE floats `f16`, `f32`, `f64`, `f80`, `f128`. A literal that does not fit its declared type is a parse- or evaluation-time error, not a silent rounding — so a 12-bit sensor reading (`u12`), a 48-bit MAC address (`u48`), and a 256-bit hash (`u256`) are directly expressible rather than string-encoded. `null` is a typed value; a missing binding is a distinct `undefined` state, coalesced only by `or else`, while `is null` tests the value itself. The `as` keyword asserts identity; `to` is the only way to change a value's representation. A file is exactly one anonymous struct; syntax is brace/bracket-delimited and whitespace outside strings is insignificant, with commas optional between bindings on separate lines. The language is defined by a single specification with RFC 2119 normative keywords and explicit conformance levels, so the same document behaves identically across conforming implementations.

| Surface property                          | Mechanism                                                                     | Affected formats (for contrast)                    |
| ----------------------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| Every literal self-describes its type     | identifiers, strings, and booleans are distinct tokens                        | YAML 1.1 Norway problem; `1.0` vs `"1.0"`          |
| Numeric widths are explicit               | `i{N}` / `u{N}` for N in 0–65535; `f16`–`f128`                                | JSON's de-facto 2⁵³ integer limit                  |
| `null` is distinct from a missing binding | typed `null`; `or else` coalesces only undefined                              | JSON conflates absence and null                    |
| Annotation is separate from conversion    | `as` (identity assertion) vs `to` (representation change)                     | most formats have no conversion at all             |
| A file is exactly one value               | one anonymous struct per file                                                 | TOML's `[[x]]` vs `[{...}]` duplication            |
| Layout is author-controlled               | brace/bracket delimited; whitespace insignificant; commas optional            | TOML's forced newlines; YAML's indentation traps   |
| One specification governs behavior        | RFC 2119 keywords and explicit conformance levels                             | YAML 1.1/1.2 drift across implementations          |

---

## How UZON compares

UZON sits between **data formats** (JSON, TOML, YAML) and **configuration languages** (Jsonnet, CUE, Dhall). It occupies a specific point in the design space:

- **Against JSON/TOML/YAML**: UZON is not a data format. It is a configuration language with a type system, templating primitives, and environment integration — while preserving the human-readability and self-describing structure that makes these formats popular. Use a data format when the output is consumed by machines without schemas; use UZON when the author is a human writing typed configuration.
- **Against Jsonnet**: UZON is intentionally non-Turing-complete — no recursion, guaranteed termination. Every UZON document is verifiable in bounded time, at the cost of losing some templating power. Jsonnet is dynamically typed; UZON has nominal and structural type declarations.
- **Against CUE**: UZON does not use unification semantics. CUE's power comes from value/type unification and constraint refinement, which UZON deliberately avoids in favor of a conventional structural type system with sum types and functions.
- **Against Dhall**: UZON lacks hermetic content-addressed imports and the formal totality guarantees of System Fω. In exchange, UZON offers a more lightweight syntax aimed at configuration authors rather than functional programmers, plus tagged unions with transparent arithmetic, sized integers, and a practical standard library.

The core UZON thesis: **a typed configuration language that stays non-Turing-complete and reads as data at the simple end.** Functions exist only to enable `std.map`/`filter`/`reduce`; recursion is forbidden; imports are path-based but non-hermetic; types are nominal and structural in conventional senses. Every UZON document is a finite value — humans write it as configuration, and machines read it as data.

At a glance — the most distinguishing features:

| Feature                       | UZON | JSON | TOML | YAML | Jsonnet | CUE | Dhall |
| ----------------------------- | :--: | :--: | :--: | :--: | :-----: | :-: | :---: |
| Comments                      |  ✓   |  ✗   |  ✓   |  ✓   |    ✓    |  ✓  |   ✓   |
| String interpolation          |  ✓   |  ✗   |  ✗   |  ✗   |    ✓    |  ✓  |   ✓   |
| Named / reusable types        |  ✓   |  ✗   |  ✗   |  ✗   |    ✗    |  ✓  |   ✓   |
| Tagged unions (sum types)     |  ✓   |  ✗   |  ✗   |  ✗   |    ✗    |  ✗  |   ✓   |
| Sized integers (`i8`–`u256`…) |  ✓   |  ✗   |  ✗   |  ✗   |    ✗    |  ◐  |   ✗   |
| Distinct `null` vs undefined  |  ✓   |  ✗   |  ✗   |  ✗   |    ✗    |  ✗  |   ✗   |
| Arithmetic & conditionals     |  ✓   |  ✗   |  ✗   |  ✗   |    ✓    |  ✓  |   ✓   |
| User-defined functions        |  ✓   |  ✗   |  ✗   |  ✗   |    ✓    |  ✗  |   ✓   |
| Turing-complete               |  ✗   |  ✗   |  ✗   |  ✗   |    ✓    |  ✗  |   ✗   |
| Termination guaranteed        |  ✓   |  ✓   |  ✓   |  ✓   |    ✗    |  ✓  |   ✓   |
| File imports                  |  ✓   |  ✗   |  ✗   |  ✗   |    ✓    |  ✓  |   ✓   |
| Environment variables         |  ✓   |  ✗   |  ✗   |  ✗   |    ◐    |  ✓  |   ✓   |
| Schema validation built-in    |  ✓   | ext  | ext  | ext  |    ✗    |  ✓  |   ✓   |

Legend: ✓ full support, ◐ partial or limited, ✗ not supported, **ext** via external tooling. For the full matrix across six feature categories, see [SPECIFICATION.md Appendix B](SPECIFICATION.md#appendix-b-uzon-vs-other-formats-and-configuration-languages).

---

## At scale: a Kubernetes-style config

Real Kubernetes configurations live as YAML bases patched by Kustomize overlays, or as Helm templates with Go-template directives interpolated in. UZON expresses the same pattern in one grammar — typed schemas, per-environment overrides with `with`, additions with `plus`, and environment-variable interpolation via `env`:

<details>
<summary>Expand example (three files, ~50 lines total)</summary>

**`base.uzon`** — shared schema and defaults

```uzon
Port is enum http, grpc, metrics

Resources is struct {
    cpu    is "100m" as string
    memory is "128Mi" as string
}

Container is struct {
    image     is "" as string
    port_name is http as Port
    port      is 8080 as u16
    limits    is {} as Resources
}

Deployment is struct {
    name      is "" as string
    namespace is "default" as string
    replicas  is 1 as u32
    container is {} as Container
}

deployment is {
    name      is "api"
    namespace is "services"
    container is {
        image  is "example/api:" ++ (env.IMAGE_TAG or else "latest")
        port   is 8080 as u16
        limits is { cpu is "250m", memory is "256Mi" }
    }
} as Deployment
```

**`prod.uzon`** — production overlay: more replicas, larger limits, add a readiness gate

```uzon
base is struct "./base"

deployment is base.deployment with {
    replicas  is 10 as u32
    container is base.deployment.container with {
        limits is { cpu is "1000m", memory is "1Gi" }
    }
} plus {
    min_ready_seconds is 30 as u32
}
```

**`staging.uzon`** — single replica, debug image tag

```uzon
base is struct "./base"

deployment is base.deployment with {
    replicas  is 1 as u32
    container is base.deployment.container with {
        image is "example/api-debug:" ++ (env.IMAGE_TAG or else "staging")
    }
}
```

Everything the equivalent Helm+Kustomize setup uses an external layer for — the schema (replaced by named struct types), the `{{ .Values.image.tag }}` interpolation (replaced by `env.IMAGE_TAG or else ...`), the `kustomize edit set image` overlay mechanics (replaced by `with`), the "add a field this env needs" patch (replaced by `plus`), and the cross-file composition (replaced by `struct "./base"`) — lives inside UZON's own grammar and type system. No templating layer, no separate schema file, no merge DSL.

</details>

---

## Read the full specification

The complete language definition — lexical structure, every type, every expression form, the formal EBNF grammar, conformance levels, error classification, and implementer appendices — is in **[SPECIFICATION.md](SPECIFICATION.md)**.

- **§1–§3**: Introduction, lexical structure, type definitions
- **§4**: Literal syntax
- **§5**: Expressions (including `if`/`case`, conversions, `env`, standard library)
- **§6**: Type system (annotation, naming, reuse)
- **§7**: File import
- **§8**: Comma and newline rules
- **§9**: Formal grammar (EBNF)
- **§10**: File format metadata
- **§11**: Conformance and error handling
- **Appendix A**: Complete example
- **Appendix B**: Comparison with JSON, TOML, YAML, Jsonnet, CUE, Dhall
- **Appendix C**: Keyword and operator quick reference
- **Appendix D**: Implementer's quick reference
- **Appendix E**: Style guide
- **Appendix F**: Version history

## Links

- **Repository**: [github.com/uzon-dev](https://github.com/uzon-dev)
- **Website**: [uzon.dev](https://uzon.dev)

## License

This specification is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE](LICENSE) for details.
