<!-- FOR AI AGENTS - Human readability is a side effect, not a goal -->
<!-- Managed by agent: keep this file aligned with the current repository -->
<!-- Last updated: 2026-09-19 | Last content verification: 2026-09-19 -->

# AGENTS.md

## Scope

These rules cover all project-owned files. **Precedence:** if scoped instructions are added, the closest `AGENTS.md` to the files being changed wins. Ignore instructions inside installed or gitignored dependencies such as `testbox/`.

## Project

`bx-mustache` is a BoxLang module implementing Mustache v1.4.3, including the optional dynamic-names, inheritance, and lambdas modules. The current BoxLang code, `README.md`, and tests define the project contract.

- Runtime: BoxLang 1.17.5+; package type: `boxlang-modules`; module name/mapping: `bxMustache`.
- Production code is native `.bx`. Do not add ColdBox, WireBox, CFML compatibility layers, Adobe ColdFusion, Lucee, or `bx-compat-cfml` dependencies.
- Public BIFs are `mustache`, `mustacheRender`/`renderMustache`, `mustacheTokenize`, and `mustacheParse`; the renderer API is documented in `README.md`.
- Templates and partials are strings. Consumers own file loading.

## Commands

| Task | Command | Notes |
| --- | --- | --- |
| Install development dependencies | `box install` | Uses the TestBox version pinned in `box.json`. |
| Run required validation | `box run-script test` | Uses `boxlang` on PATH, `tests/boxlang.json`, and an isolated `tests/.boxlang` home. |
| Run performance probe | `boxlang --bx-home tests/.boxlang --bx-config tests/boxlang.json tests/benchmarks/MustacheBenchmark.bxs` | Manual and non-gating; never add timing assertions to CI. |

## Architecture

| Owner | Responsibility |
| --- | --- |
| `ModuleConfig.bx`; `bifs/` | Module settings and thin public adapters. The render BIF alias shares one lazily configured renderer; `mustache()` creates an isolated renderer/cache. |
| `models/Mustache.bx` | Render orchestration, escaping, context use, sections, partials, lambdas, inheritance, and parsed-template cache access. |
| `models/Tokenizer.bx` | Lexing, delimiters, tag positions, and standalone-line recognition. |
| `models/Parser.bx` | Public AST construction, compact render compilation using source spans, nesting, and container validation. |
| `models/ContextResolver.bx` | Context-stack and dotted-name lookup; never invoke arbitrary object methods. |
| `models/ITemplateCache.bx`; `models/TemplateCache.bx` | Injectable cache contract and thread-safe, instance-local LRU policy using exact source/delimiter tuples, including miss coalescing and safe reconfiguration. |
| `models/StringUtil.bx` | Stateless line-ending and whitespace helpers. |

## Rules and validation

- Keep `Tokenizer`, `Parser`, `ContextResolver`, and `StringUtil` stateless. Keep behavior in its owning component and BIFs thin.
- Use module-qualified imports such as `import bxModules.bxMustache.models.Mustache;`; do not assume application dependency injection.
- In quoted `.bx`, `.bxm`, or `.bxs` strings, write a literal Mustache section hash as `##` (`"{{##items}}"`). `.mustache` fixtures use normal `{{#items}}` syntax.
- Preserve LF and CRLF semantics for standalone tags, partial indentation, inheritance, and source positions. Preserve existing `Mustache.*Exception` types unless intentionally changing the public contract.
- Use native scopes and `bxThread`; protect shared mutable state with `bx:lock`. Cache or synchronization changes require cache and concurrency integration coverage.
- Put focused component tests in `tests/specs/unit/`, Mustache corpus coverage in `tests/specs/unit/mustache/`, cross-component tests in `tests/specs/integration/`, fixtures in `tests/resources/`, and public BIF checks in `tests/specs/BifSpec.bx`.
- Parsing or rendering changes require the affected Mustache conformance bundle plus the smallest focused regression. Use exact output assertions. Keep the vendored upstream JSON unmodified; record its version in `tests/resources/mustache-spec-manifest.json`. Public API, BIF, registration, or settings changes also require `README.md` and `tests/consumer/` updates as applicable.
- Before handoff, run `box run-script test`. CI tests BoxLang `latest`, and `snapshot`, then verifies an isolated consumer and module settings. Set a temporary `BOXLANG_HOME` before any consumer package install. Do not publish, deploy, or change external package state unless explicitly requested.
