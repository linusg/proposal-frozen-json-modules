# Frozen JSON modules

## Status

Champion(s):

- Linus Groh

Author(s):

- Linus Groh

Stage: 0

## Motivation

Whether JSON modules should be frozen or not has been
[discussed at length](#related-discussions), but they ultimately ended up
producing mutable objects and arrays, just like `JSON.parse()`.

This proposal doesn't challenge the status quo, but offers a way to opt into
frozen JSON modules where they are needed. In particular, it builds on top of
the [JSON.parse Options](https://github.com/tc39/proposal-json-parseimmutable)
proposal, which adds the ability to produce deeply frozen objects using
`JSON.parse(text, { freeze: true })`. Import attributes provide an obvious way
to make this available to both statically and dynamically imported JSON modules.

Frozen JSON modules are useful for data that is not meant to change, such as
configuration, translations, test fixtures, or lookup tables.

Freezing JSON modules manually has several drawbacks:

- Either the first importer has to freeze a JSON module before anyone else sees
  it, or a JavaScript wrapper module has to re-export a frozen copy and every
  importer has to remember to use it.
- Deep freezing requires a recursive helper that walks the whole object graph
  after it has been created. As the JSON.parse Options proposal points out, a
  native implementation can be faster, and static analysis can benefit from
  knowing that the result is always deeply frozen.
- Workarounds such as importing the module with `type: "text"`
  ([Import Text](https://github.com/tc39/proposal-import-text), Stage 3) and
  then passing it to `JSON.parse(text, { freeze: true })` are clunky and
  inefficient compared to importing JSON directly.

## Proposal

```js
// object.json: { "foo": 123 }
import object from "./object.json" with { type: "json", freeze: true };
const { default: object } = await import("./object.json", { with: { type: "json", freeze: true } });

Object.isFrozen(object); // true
Object.getPrototypeOf(object) === null; // true
```

```js
// array.json: [1, 2, 3]
import array from "./array.json" with { type: "json", freeze: true };
const { default: array } = await import("./array.json", { with: { type: "json", freeze: true } });

Object.isFrozen(array); // true
Object.getPrototypeOf(array) === Array.prototype; // true
array.push(4); // TypeError
```

### Boolean import attribute values

Import attribute values are currently restricted to strings. Rather than
settling for `freeze: "true"`, this proposal also adds syntactic support for
boolean values. The
[Import Attributes](https://github.com/tc39/proposal-import-attributes) proposal
addressed this in its FAQ:

> **Should more than just strings be supported as attribute values?**
>
> We could permit import attributes to have more complex values than simply
> strings, for example:
>
> ```js
> import value from "module" with { attr: { key1: "value1", key2: [1, 2, 3] } };
> ```
>
> This would allow import attributes to scale to support a larger variety of
> metadata.
>
> We propose to omit this generalization in the initial proposal, as a key/value
> list of strings already affords significant flexibility to start, but we're
> open to a follow-on proposal providing this kind of generalization.

Implementer feedback on the Import Attributes proposal raised concerns about
BigInt _keys_, since detecting duplicate keys would require allocating
garbage-collected BigInts in the parser. Support for BigInt and Number keys was
subsequently dropped
([2023-09 notes](https://github.com/tc39/notes/blob/main/meetings/2023-09/september-26.md#import-attributes-implementer-feedback),
[tc39/proposal-import-attributes#145](https://github.com/tc39/proposal-import-attributes/issues/145)).

Parsing the `true` and `false` literals requires no allocations, and they are
the only syntax this proposal adds.

## Related proposals

- **[Import Attributes](https://github.com/tc39/proposal-import-attributes)**
  (Stage 4)

  Introduced the `with { ... }` syntax, but settled on only supporting string
  literal values.

- **[JSON Modules](https://github.com/tc39/proposal-json-modules)** (Stage 4)

  Introduced `with { type: "json" }`, but settled on mutable objects.

- **[JSON.parse Options](https://github.com/tc39/proposal-json-parseimmutable)**
  (Stage 2.7)

  Introduces `JSON.parse(text, { freeze: true, preferNullPrototype: true })`.

- **[Import Bytes](https://github.com/tc39/proposal-import-bytes)** (Stage 2.7)

  Introduces `with { type: "bytes" }`, returning a `Uint8Array` backed by an
  **immutable `ArrayBuffer`** with no import attribute to change it.

## Related discussions

- **"JSON modules"**
  ([whatwg/html#4315](https://github.com/whatwg/html/issues/4315), 2019-01)

  First proposed JSON modules for the web, citing Node's support. Deep freezing
  was suggested, but rejected for consistency with JS modules and other shared
  mutable objects on the web platform; code that wants a frozen copy should run
  first and freeze it.

- **"import ConstJson from './config.json' as const;"**
  ([microsoft/TypeScript#32063](https://github.com/microsoft/TypeScript/issues/32063),
  2019-06)

  Request for readonly literal types on JSON imports using
  `with { type: "json", const: true }`, still open.

- **"Should JSON modules be frozen?"**
  ([tc39/proposal-json-modules#3](https://github.com/tc39/proposal-json-modules/issues/3),
  2020-05)

  Reiterates some prior arguments and settles on mutable objects for JSON
  modules, following the precedent of ECMAScript being mutable by default and
  Node.js `require()`.

- **"Should JSON modules be frozen?"**
  ([tc39/proposal-json-modules#1](https://github.com/tc39/proposal-json-modules/issues/1),
  2020-08)

  More in-depth discussion with arguments for both sides. Also mentions import
  attributes as a possible follow-up to make both behaviors available.

- **"undeniable global communication channel"**
  ([tc39/proposal-import-bytes#2](https://github.com/tc39/proposal-import-bytes/issues/2),
  2025-07)

  Concern over `with { type: "bytes" }` producing a `Uint8Array` backed by a
  mutable `ArrayBuffer`, which, unlike JSON modules, can't be frozen after
  import. Resolved by switching to an immutable `ArrayBuffer`.

## Open design questions

- Should JSON modules also support the `preferNullPrototype` attribute? In
  `JSON.parse` it defaults to the value of `freeze`, so `freeze: true` already
  produces null-prototype objects, and a separate attribute would only be needed
  to opt out. Full parity with `JSON.parse` options isn't possible anyway, since
  options like a reviver function can't be expressed as import attributes. This
  highly depends on the outcome of
  [tc39/proposal-json-parseimmutable#24](https://github.com/tc39/proposal-json-parseimmutable/issues/24).
- With Import Bytes at Stage 2.7 it is too late to change it, but should there
  be a follow-up proposal for `with { type: "bytes", immutable: false }`? Or do
  we accept that JSON is mutable by default with a way to opt into freezing, and
  bytes immutable by default with no way of opting out?
- `HostGetSupportedImportAttributes` returns a flat list of keys, so a host that
  supports `freeze` accepts it on any import, not only on JSON modules. ECMA-262
  can't require `type: "json"` alongside `freeze` either, since hosts may
  support JSON modules imported without `type`. Should hosts get a hook that
  sees the whole module request, including its specifier, so they can reject
  attributes that don't apply to the requested module type?
