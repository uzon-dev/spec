# UZON Language Specification

**A typed, human-readable data expression format.**

UZON is a configuration and data language that combines the simplicity of JSON with the expressiveness of a typed language. Every UZON document evaluates to a single, immutable value with no side effects, no loops, and no recursion.

- **Specification Version**: 0.10 (Pre-release)
- **File Extension**: `.uzon`
- **MIME Type**: `application/uzon`
- **Encoding**: UTF-8

## Quick Example

```uzon
name is "Star Observatory"
epoch is 7
longitude is -117.8531 as f64
latitude is 33.6694 as f64

Season is enum spring, summer, autumn, winter
season is autumn as Season

base_telescope is {
    aperture_mm is 200 as u16
    focal_ratio is 8.0 as f64
    focal_length_mm is aperture_mm to f64 * focal_ratio
    tracking is true
}

primary is base_telescope with { aperture_mm is 400 as u16 }

targets are "Andromeda", "Orion Nebula", "Crab Nebula", "Pleiades" as [string]

seeing_conditions is env.SEEING to f64 or else 1.5

active_scope is if seeing_conditions < 1.0
    then "primary"
    else "secondary"
```

## Key Features

| Feature                     | Description                                                                  |
| --------------------------- | ---------------------------------------------------------------------------- |
| **Human-first syntax**      | English-like keywords (`is`, `from`, `called`, `as`, `to`, `named`, `enum`)  |
| **Rich type system**        | Primitives, structs, tuples, lists, enums, unions, tagged unions, functions  |
| **Named & reusable types**  | Declare via `enum`/`struct`/`union` or `called`, reuse with `as`             |
| **Conditional expressions** | `if`/`then`/`else` and `case`/`when`                                         |
| **Arithmetic & logic**      | `+`, `-`, `*`, `/`, `%`, `^`, `and`, `or`, `not`                             |
| **Lexical name resolution** | Bare identifiers resolve via the scope chain — no prefixes needed            |
| **Environment variables**   | `env.PORT`, `env.HOME` with `or else` fallback                               |
| **Struct composition**      | `with` for overrides, `plus` for extensions                                  |
| **File imports**            | `struct "./path"` to compose documents                                       |
| **Functions**               | First-class functions with typed parameters and defaults                     |
| **Standard library**        | `std.map`, `std.filter`, `std.reduce`, `std.sort`, `std.len`, and more       |
| **String interpolation**    | `"hello, {name}!"`                                                           |
| **Undefined coalescing**    | `env.PORT to u16 or else 8080`                                               |
| **Comments**                | Line comments with `//`                                                      |

## UZON vs Other Formats

| Feature                  | UZON | JSON | TOML | YAML |
| ------------------------ | :--: | :--: | :--: | :--: |
| Comments                 | Yes  |  No  | Yes  | Yes  |
| Typed enums              | Yes  |  No  |  No  |  No  |
| Unions / tagged unions   | Yes  |  No  |  No  |  No  |
| Named/reusable types     | Yes  |  No  |  No  |  No  |
| Conditional expressions  | Yes  |  No  |  No  |  No  |
| Arithmetic expressions   | Yes  |  No  |  No  |  No  |
| Functions                | Yes  |  No  |  No  |  No  |
| Name references          | Yes  |  No  |  No  | Yes  |
| Environment variables    | Yes  |  No  |  No  |  No  |
| File imports             | Yes  |  No  |  No  |  No  |
| String interpolation     | Yes  |  No  |  No  |  No  |
| Self-describing          | Yes  | Yes  | Yes  |  No  |

## Specification

The full language specification is in [SPECIFICATION.md](SPECIFICATION.md). It covers:

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

## Design Goals

1. **Self-describing** — Every value is unambiguously parseable without external type context.
2. **Typed** — Rich primitive and compound types with named type reuse.
3. **Cross-language** — Clean mapping to any mainstream programming language.
4. **Expressive** — Conditionals, arithmetic, name references, and environment variables enable dynamic configuration.
5. **Human-first** — English-like keywords (`is`, `from`, `called`, `as`, `to`, `named`, `enum`, `tagged`, `struct`, `union`) for readability.
6. **Minimal** — Concise syntax; every construct earns its place.

## Links

- **Repository**: [github.com/uzon-dev](https://github.com/uzon-dev)
- **Website**: [uzon.dev](https://uzon.dev)

## License

This specification is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE](LICENSE) for details.
