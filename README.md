# bx-mustache

**Render logic-less Mustache templates in native BoxLang applications**

Use familiar Mustache syntax with BoxLang data, functions, partials, inheritance, and a small global BIF API.

`bx-mustache` implements the [Mustache v1.4.3 specification](https://github.com/mustache/spec/tree/v1.4.3) as a native BoxLang module. It started as a native conversion of the [Stubble](https://github.com/tonyjunkes/stubble) CFML library and requires BoxLang 1.16.0 or newer.

## Install

Install the module from GitHub while it is being prepared for a ForgeBox release:

```bash
box install git://github.com/tonyjunkes/bx-mustache.git
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

Object lookup supports public fields and zero-argument `getName()` / `isName()` accessors. Arbitrary Java methods are not invoked by template names. BoxLang function values can be used as lambdas. Null intermediate values in dotted paths render as missing values.

## Usage

### Global BIFs

| Function | Purpose |
| --- | --- |
| `mustacheRender( template, view = {}, partials = {} )` | Render with the shared, module-configured renderer. |
| `renderMustache( template, view = {}, partials = {} )` | Alias for `mustacheRender()`. |
| `mustache( cacheEnabled = true, cacheMaxEntries = 200 )` | Create an isolated renderer with its own cache. |
| `mustacheTokenize( template, openDelimiter = "{{", closeDelimiter = "}}" )` | Return the low-level token stream. |
| `mustacheParse( tokens, template )` | Convert a token stream into the template AST. |

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

The cache is enabled with a maximum of 200 parsed templates by default. Renderers created by `mustache()` use the arguments supplied when you create them.

## Contributing

Install CommandBox, then install the development dependencies and run the TestBox suite:

```bash
box install
box run-script test
```

Parsing or rendering changes should include focused regression coverage under [`tests/specs/`](tests/specs/). The suite currently exercises over 300 examples across the official Mustache corpus and project-specific cases. A non-gating performance probe is available at [`tests/benchmarks/MustacheBenchmark.bxs`](tests/benchmarks/MustacheBenchmark.bxs).

## Acknowledgments

`bx-mustache` was heavily based and inspired by the [vast Mustache ecosystem](https://mustache.github.io/) and the many developers who have made it all possible. Thanks!
