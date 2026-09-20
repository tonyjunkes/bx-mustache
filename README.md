# bx-mustache

**Render Mustache templates in native BoxLang applications**

Use familiar Mustache syntax with BoxLang data, functions, partials, inheritance, and a small global BIF API.

`bx-mustache` implements the [Mustache v1.4.3 specification](https://github.com/mustache/spec/tree/v1.4.3) as a native BoxLang module. It started as a native conversion of the [Stubble](https://github.com/tonyjunkes/stubble) CFML library.

## Requirements

- BoxLang 1.17.x or newer

> Older versions may still work, providing they support the functionality leveraged, but this project has been targeting whatever is considered `latest` when updated.

## Installation

Install the latest stable release from ForgeBox:

```bash
box install bx-mustache
```

BoxLang registers the module as `bxMustache` and makes its BIFs available to your application.

## Quickstart

Render a template with `mustacheRender()`:

```boxlang
template = "Hello {{name}}!";
output = mustacheRender( template, { name: "BoxLang" } );

writeOutput( output ); // Hello BoxLang!
```

Pass partials as the third argument:

```boxlang
template = "{{##people}}{{>card}}{{/people}}";
output = mustacheRender(
	template,
	{ people: [ { name: "Ada" }, { name: "Grace" } ] },
	{ card: "<li>{{name}}</li>" }
);

writeOutput( output ); // <li>Ada</li><li>Grace</li>
```

BoxLang uses `#` for interpolation inside quoted `.bx` and `.bxm` strings, so write a literal Mustache section hash as `##`. Templates loaded from `.mustache` files use the normal `{{#people}}` syntax.

## Supported Mustache Features

The renderer covers the official v1.4.3 corpus, including its optional lambdas, inheritance, and dynamic-names modules.

- **Variables:** escaped `{{name}}` and unescaped `{{{html}}}` or `{{&html}}` interpolation.
- **Context lookup:** dotted names, numeric array indexes, implicit iterators, missing values, and parent contexts.
- **Sections:** truthy, falsey, inverted, nested, and array-iteration sections.
- **Partials:** string or BoxLang-function partials, nesting, recursion, indentation, and dynamic names.
- **Templates:** comments, alternate delimiters, and standalone whitespace handling with LF or CRLF line endings.
- **Lambdas:** variable and section lambdas whose returned values can contain Mustache markup.
- **Inheritance:** parent templates and block overrides with `{{<layout}}` and `{{$body}}`.

Templates and partials are strings. Your application owns file loading, so read a template with `fileRead()` before passing its contents to the renderer.

Object lookup supports public fields and zero-argument `getName()` / `isName()` accessors, including generated and inherited BoxLang getters. Private members and BoxLang runtime implementation scopes are excluded. Arbitrary Java methods are not invoked by template names. BoxLang function values can be used as lambdas. Null intermediate values in dotted paths render as missing values.

Block names and matching opening/closing tag names are case-sensitive. Data-key lookup follows the supplied struct's case sensitivity.

### BoxLang extensions and value semantics

- Array indexes in dotted names are one-based: `{{users.2.name}}` selects the second user.
- Function-valued partials are called with no arguments; their return value supplies the template text.
- Variable lambdas normally take no arguments; a lambda declaring a parameter receives the current context. Section lambdas may take no arguments, the raw section text, or the raw text plus a render callback. Their returned text is rendered using Mustache's lambda delimiter rules.
- The closing-tag shorthand `{{/#items}}` is accepted as an extension. Prefer standard `{{/items}}` in portable templates; quoted BoxLang strings require `{{/##items}}` for the shorthand.
- Sections use native scalar coercion: `false`, zero, empty strings, null, empty arrays, and strings such as `"false"`, `"no"`, and `"0"` are falsey. Structs, objects, functions, and nonempty arrays are truthy. Inverted sections use the same decision.

## Usage

### Global BIFs

| Function | Purpose |
| --- | --- |
| `mustacheRender( template, view = {}, partials = {} )` | Render with the shared, module-configured renderer. |
| `renderMustache( template, view = {}, partials = {} )` | Alias for `mustacheRender()`. |
| `mustache( cacheEnabled = true, cacheMaxEntries = 200 )` | Create an isolated renderer with its own cache. |
| `mustacheTokenize( template, openDelimiter = "{{", closeDelimiter = "}}" )` | Return the low-level token stream. |
| `mustacheParse( tokens, template )` | Convert a token stream into the template AST. |

The parser's name splitting is internal. Use `mustacheParse()` or the renderer's `parse()` method to obtain AST nodes and their `nameParts` arrays.

### Renderer class

Import and retain a renderer for direct cache control:

```boxlang
import bxModules.bxMustache.models.Mustache;

renderer = new Mustache();
renderer.configureCache( enabled = true, maxEntries = 100 );
output = renderer.render( "Hello {{name}}!", { name: "Ada" } );

writeOutput( output ); // Hello Ada!
```

The class exposes `render()`, `tokenize()`, `parse()`, `configureCache()`, `clearCache()`, and `getCacheStats()`.

For custom collaborators, `new Mustache( tokenizer = ..., parser = ..., cache = ... )` accepts objects with the corresponding method contracts. Injected parsers use `parse( tokens, template )`; cache implementations can implement `models.ITemplateCache`. Collaborators remain private and have no generated getters or setters. Use the cache-control methods above to configure a renderer.

## Configuration

`mustacheRender()` and `renderMustache()` share an instance-local LRU cache. Configure it in your application's `boxlang.json`:

```json
{
  "modules": {
    "bxMustache": {
      "enabled": true,
      "settings": {
        "cacheEnabled": true,
        "cacheMaxEntries": 200
      }
    }
  }
}
```

The cache is enabled with a maximum of 200 parsed templates by default. Module settings are applied on the shared renderer's first invocation. Renderers created by `mustache()` use the arguments supplied when you create them and do not inherit module settings. `configureCache()` truncates fractional capacities and normalizes values below one to one; disabling the cache clears its entries.

The default renderer caches a compact internal AST whose section bodies refer to source spans. Public `parse()` and `mustacheParse()` results still include raw section text. Injected parsers continue to use `parse( tokens, template )`. Indented partial and block source transformations are reused within each render, with at most 200 prepared variants retained until that render finishes; callable partials still run on every occurrence, and rendered output is never cached.

## Contributing

Install BoxLang and CommandBox, then install the development dependencies and run the native TestBox suite:

```bash
box install
box run-script test
```

The test script uses the `boxlang` executable on your PATH and an isolated home under `tests/.boxlang`, so installed global modules cannot shadow the checkout. The project runner exits unsuccessfully for failures, errors, or an empty suite. CI runs BoxLang latest and snapshot, plus an isolated production install that checks BIF registration, alias sharing, and enabled/disabled module cache settings.

Parsing or rendering changes should include focused regression coverage under [`tests/specs/`](tests/specs/). The nine conformance bundles consume the pinned, unmodified upstream JSON files in [`tests/resources/mustache-spec/`](tests/resources/mustache-spec/) with exact output assertions. [`MustacheConformance.bx`](tests/resources/MustacheConformance.bx) supplies the BoxLang lambda adapters; [`mustache-spec-manifest.json`](tests/resources/mustache-spec-manifest.json) records upstream provenance and verified category counts. Update these together when advancing the spec version. The test suite requires no network access after installing TestBox.

Run the non-gating performance probe with:

```bash
boxlang --bx-home tests/.boxlang --bx-config tests/boxlang.json tests/benchmarks/MustacheBenchmark.bxs
```

## Acknowledgments

`bx-mustache` was heavily based and inspired by the [vast Mustache ecosystem](https://mustache.github.io/) and the many developers who have made it all possible. Thanks!
