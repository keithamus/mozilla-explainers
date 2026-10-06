# JSON form encoding explainer

Author: [Keith Cirkel]

## User problems to be solved

Most modern HTTP APIs have settled on JSON as the format for request bodies, but
HTML forms can only submit:

- `application/x-www-form-urlencoded`,
- `multipart/form-data`, or
- `text/plain`.

Existing encodings provide neither typed values nor nested structure. To match
APIs described by OpenAPI or JSON Schema (the prevailing systems for building
server side APIs), authors must reshape form data in script or reconstruct it
on the server.

To send a form to a JSON endpoint, authors have to intercept the `submit` event
in script, build a `FormData`, convert it to an object by hand, serialise it,
and send it with `fetch()`. This loses what a declarative form gives for free.
It no longer works without script, and the browser no longer handles
navigation, history, the response document, or back/forward behaviour.
Rebuilding that by hand means rendering the response yourself and restoring
pages on `popstate`, and even then the new page's scripts don't run, there is
no loading indicator or stop button, and scroll and focus have to be reset by
hand. Every site also ends up re-implementing the same conversion slightly
differently, for example in how repeated names are handled.

### Evidence of demand

The obvious conversion, `Object.fromEntries(formData)`, is lossy. `FormData` can
hold several entries with the same name, from a `<select multiple>` or a group
of checkboxes, but an object has one value per key, so `fromEntries()` keeps
only the last:

```html
<input type="checkbox" name="tag" value="magic" checked />
<input type="checkbox" name="tag" value="horns" checked />
```

```js
Object.fromEntries(new FormData(form)); // { tag: "horns" }
```

Code that wants to avoid losing data has to consult a schema, call
`getAll()` for every name, or turn a value into an array when a second entry
appears. Frameworks either recommend lossy `fromEntries`, leave this to
userland, or reinvent it:

- **Next.js**: the official App Router tutorial ([Learn, chapter 11]) recommends
  the lossy idiom without mentioning repeated names: "If you're working with
  forms that have many fields, you may want to consider using the `entries()`
  method with JavaScript's `Object.fromEntries()`."
- **Astro Actions** convert FormData by branching on the Zod schema, calling
  `getAll()` for array types and `get()` for everything else
  ([`formDataToObject`]).
- **API schemas**: frameworks generate OpenAPI and JSON Schema from code.
  [FastAPI] is "based on (and fully compatible with)" OpenAPI and JSON Schema.
- **Nested names**: servers already build nested objects from bracket names in
  urlencoded and multipart bodies: [PHP], Rails through [Rack], and [qs] (~213M
  weekly), which Express uses for `urlencoded({ extended: true })`.
- **Security**: hand-rolled converters are an attack surface. Qwik City's
  `formToObj()` had a critical prototype pollution vulnerability
  ([CVE-2026-25150]) in its nesting code.

The same two checkboxes give different results depending on the approach, and
most approaches change a field's type when only one box is ticked:

| Approach                                                                        | Both ticked         | Only `magic` ticked |
| ------------------------------------------------------------------------------- | ------------------- | ------------------- |
| `Object.fromEntries()`                                                          | `"horns"`           | `"magic"`           |
| `FormData.get()`                                                                | `"magic"`           | `"magic"`           |
| htmx [`json-enc`], [Hono][hono-source] `{ all: true }`, Bun `toJSON()`          | `["magic","horns"]` | `"magic"`           |
| `getAll()` with schema ([superforms][sveltekit-superforms], Astro array fields) | `["magic","horns"]` | `["magic"]`         |

Hono, like PHP, Rails and qs, always returns an array for a name ending in
`[]`.

## Methodology for approaching & evaluating solutions

- Advance the [Declarative Web]. A plain `<form>` should be able to talk to a
  JSON endpoint with no script.
- Do not expand the attack surface. Servers widely assume that a request with
  `Content-Type: application/json` can't come from a cross-origin form, and many
  rely on that for CSRF protection. A solution must keep that assumption true
  unless the server explicitly opts in.
- Reuse existing platform primitives (the `enctype`/`formenctype` attributes,
  the form entry list, `FormData`, CORS) rather than inventing parallel ones.
- Follow the naming conventions servers already parse, so that a browser without
  support falls back to a body those servers decode to the same shape.
- Have exactly one JSON representation of form data on the platform, shared by
  declarative submission and script.
- Layer the solution, following the [Extensible Web Manifesto]. The conversion
  should be a primitive that is useful on its own and that authors can compose
  in script, and the declarative feature should be built on top of it.

## Prior features and proposals

- [W3C HTML JSON form submission] (Working Group Note, 2015) proposed
  `enctype="application/json"` with a path syntax in control names
  (`name="pet[0][species]"`) that builds nested objects and arrays, and
  base64-encoded file contents. It was published as a Note and not pursued
  further. Its path algorithm has costly edge cases. Sparse indices are filled
  with `null`, so `kids[999999999]` builds a billion-element array, and
  conflicting paths are merged into objects with `""` and index keys. Its
  examples show numbers and booleans (`"shiny": true`, `[1, 2, 3]`), but its
  algorithm only ever produces strings. It also forced same-origin requests, so
  it could never reach a cross-origin API, even with the server's consent.
- [whatwg/html#1667] asked for that Note to be brought into HTML. It was closed.
- [whatwg/xhr#84] considered `FormData.prototype.toJSON()`. It was closed. The
  concerns raised were that `toJSON()` is synchronous while reading `Blob`
  contents is not, and that JSON cannot represent bytes. Anne van Kesteren also
  doubted the value: "Servers can deal with multipart/form-data just fine and
  JSON doesn't offer that much benefit over it."
- [whatwg/url#143] asked for `URLSearchParams.prototype.toObject()`. It was
  declined, partly because of duplicate keys and partly on principle. Domenic
  Denicola: "Part of the whole point of URLSearchParams, ES6 Map, etc. is to
  move _away_ from using objects as maps". He also cited `__proto__` prototype
  pollution bugs.
- Bun ships a non-standard `FormData.toJSON()` ([source][bun-source]). It
  returns a plain string for single values, an array for repeated names, and
  `{}` for files.
- A known workaround abuses `enctype="text/plain"` to produce a body that looks
  like JSON. The `Content-Type` stays `text/plain`, and the trick is best known
  as a CSRF technique.
- [whatwg/html#3577] asks for PUT, PATCH and DELETE in forms. Its spec PR,
  [whatwg/html#11347], sends cross-origin submissions with those methods through
  CORS, relying on [whatwg/fetch#1785] to let navigations preflight "if the
  navigation contains a request that is not safelisted". That covers
  non-safelisted headers as well as methods, so it covers `application/json`
  too. Neither PR has implementer support yet.
- [RFC 10008] defines QUERY, a safe, idempotent method with a body, and
  [whatwg/html#12594] proposes `method="query"` for forms. Whether QUERY is
  CORS-safelisted is unsettled. A QUERY form with `enctype="application/json"`
  would give search forms a structured body that can be retried and revisited
  like GET.

## Proposed solution

The proposal has two parts, described in the order they would ship.

### Part 1: `FormData.prototype.toJSON()` and `FormData.fromJSON()`

`FormData` gains `toJSON()` and static `fromJSON()` methods. Bracket names
describe nested objects and arrays; control types provide value hints.

#### Names

| Name                | Places the value at                           |
| ------------------- | --------------------------------------------- |
| `title`             | key `title`                                   |
| `pet[name]`         | key `name` of object `pet`                    |
| `pet[0][name]`      | key `name` of the first object in array `pet` |
| `pet["0"][name]`    | key `name` of object key `0` of `pet`         |
| `tag[]`             | array `tag`, even with only one value         |
| `q["a]b"]`, `q[""]` | key `a]b`, or the empty key, of `q`           |
| `["x[y]"]`          | top-level key `x[y]`                          |

- An unquoted bracket segment matching `0` or a non-zero ASCII digit followed
  by zero or more ASCII digits identifies an array element. Fields sharing that
  label belong to the same element, so `pet[0][name]` and `pet[0][legs]` build
  one object. Arrays can contain objects, arrays and scalar values at any
  nesting level.
- Labels are positions. At every array, at every nesting level, the distinct
  labels must be exactly `0` to `n - 1`, in any entry order, so `kids[1]=b` and
  `kids[0]=a` give `{"kids":["a","b"]}`. A missing position, as in `kids[0]`
  with `kids[2]`, or a lone large label such as `kids[99999]`, makes conversion
  throw a `TypeError`. Gaps are never closed or padded, and labels are never
  renumbered (see [Caveats](#caveats-shortcomings-and-other-drawbacks-of-design-choices)).
- Other bracket segments are object keys. Quoted segments are always object
  keys, so `pet["0"][name]` gives `{"pet":{"0":{"name":…}}}`. Leading-zero
  names such as `pet[01]` are also object keys.
- `[]` appends a value and is allowed only at the end of a name.
  `p[][a]` stays invalid, as it doesn't say which object owns `a`. Use
  `p[0][a]`, `p[1][a]` and so on for separate objects.
- A key can be written as a JSON string in brackets, which works for any key.
  Canonical names quote keys that are empty, contain `[`, `]`, `"` or a lone
  surrogate, or match an array label after the first key. The first key is
  always an object key, never an index, so a top-level `0` stays bare, while
  a nested numeric key stays quoted, as in `0["1"]`.
- A name used more than once collects its values into an array, so no value is
  lost. A name ending in `[]` is an array even with one value, so its type
  doesn't depend on how many boxes are ticked. Checkbox groups and
  `<select multiple>` should use `[]`.
- Names that aren't valid paths, such as `error[bad`, `[x]` or `a[]b`, are kept
  whole as flat keys.
- If entries sharing a first key contradict each other, for example `mix`
  alongside `mix[key]`, `tag[]` alongside `tag[0]`, or `tag[0]` alongside
  `tag[a]`, none of them nest. Each is kept whole as a flat key, merging with
  any entry that names that key directly. No value is lost, and whether a name
  nests doesn't depend on entry order. This replaces the Note's merge rules.
  A flattened name loses its terminal `[]` meaning too, so a flattened `a[b][]`
  with one value is a string (see [Empty states](#empty-states)).
- Density is checked on every name that parses as a path, before conflict
  flattening, so flattening never rescues a gap, and `t[0]`, `t[2]` alongside
  `t[a]` still throws. This is an admission rule the draft adopts, not
  something order independence needs (see [open question 3][open questions]).
- A name can have at most a limited number of key and index segments (see
  [open question 2][open questions]); a final `[]` doesn't count. Longer names
  stay flat.

Every field has one **canonical name**: bare object keys where possible,
quoted otherwise. Arrays containing only JSON leaves (including empty objects
and arrays) use `[]`; arrays containing a non-empty object or array use dense
labels starting at `[0]` for every element, including scalar elements. Labels
are canonicalised at each nesting level. Thus repeated plain `tag` becomes
`tag[]`, and `kids[1]`, `kids[0]` become `kids[]`, `kids[]` in label order.
Repeated `fromJSON(data.toJSON())` calls then preserve the canonical names.

#### Values

- Values are strings unless their control or the script that added them provides
  a type hint of `"string"`, `"number"` or `"boolean"`.
- `<input type=number>` and `<input type=range>` hint numbers, so the value is
  parsed with the rules for parsing floating-point number values, giving the
  same number as `valueAsNumber`. A value those rules reject gives `null`.
  Besides the empty string, that includes a syntactically valid value outside
  the double range such as `1e400`, for which the rules return an error, not
  infinity. The stored string stays `"1e400"` for `get()` and legacy
  encodings, and [FormValidation](form-validation.md) reports it as bad input,
  not as an empty value.
- A checkbox with no `value` attribute gives `true` instead of `"on"`. An
  eligible unchecked Boolean checkbox gives JSON `false`, without adding a
  publicly visible entry. Empty declared collections give `[]`, as below.
- `File` entries give their file name, as with
  `application/x-www-form-urlencoded` today. Their contents can't be put in the
  JSON (see [caveats]), so uploads are handled in part 2.
- Everything else, including date controls, `<select>` and hidden inputs, stays
  a string. JSON has no date type, and date controls already submit formats that
  servers parse.
- The type comes from the control, not the value. Typing `01234` into a text
  input gives `"01234"`, with its leading zero.
- Types are recorded when the form's entry list is constructed, so
  `new FormData(form)` stores internal type hints, and `get()` still returns
  strings.
- Newlines are not converted to CRLF.

##### Empty states

An eligible unchecked checkbox without a `value` attribute records a
**JSON-only empty-state contribution** of `false`. A valid terminal `[]` name
instead declares a collection: unchecked checkbox/radio groups, a multiple
select with no selected enabled option, and a multiple file input with no
selected files record an empty-list contribution at the collection's path.
Thus `name="tag[]"` gives `{"tag":[]}`, not `{"tag":[[]]}`, when nothing is
selected. Collection handling takes precedence over scalar Boolean handling.

Eligibility uses HTML's normal form-participation rules, except that an
unchecked checkbox or radio can record an empty state. Disabled controls,
including controls disabled through a fieldset, unnamed controls and controls
inside a `datalist` contribute nothing. A selected enabled option with
`value=""` is an actual value, not an empty collection. An explicitly valued
unchecked checkbox without a collection name, an unchecked scalar radio and a
custom element submitting `null` remain absent.

These contributions are captured with the entry list, before `formdata`.
`get()`, `getAll()`, `has()`, iteration and existing encodings do not expose
them. Empty multiple-file inputs with valid terminal `[]` names keep their
native empty `File` placeholder in that public view, but omit it from JSON and
`multipart/json` parts. This exclusion uses native no-selection provenance, not
the file's name or size; script-added files are never mistaken for placeholders.

A Boolean contribution is ignored if an ordinary entry has the same source
name. `false` is a default for a missing field, not one value per unchecked
control. For JSON, false contributions with the same exact source name
coalesce, keeping the first captured, while validation keeps each record.
Contributions under different source names that parse to the same path, such
as `draft` and `["draft"]`, are not merged. An empty-list contribution is
ignored if its target path contains an actual value, a surviving Boolean
contribution or a more-specific empty-list contribution. Duplicate empty-list
targets coalesce. Comparisons use parsed paths, not string prefixes. Surviving
empty states participate in ordinary conflict flattening, which can also
flatten actual entries. Empty-state-only fields follow actual entries in JSON
insertion order.

With checkboxes that have no `value` attribute:

| Checkbox names       | Checked | JSON                      |
| -------------------- | ------- | ------------------------- |
| `draft`, `draft`     | neither | `{"draft":false}`         |
| `draft`, `draft`     | one     | `{"draft":true}`          |
| `draft`, `["draft"]` | neither | `{"draft":[false,false]}` |
| `draft`, `["draft"]` | `draft` | `{"draft":[true,false]}`  |

A terminal `[]` keeps a collection's type stable only while it nests. Here `a`
contradicts `a[b][]`, so both stay flat, and the flattened key holds whatever
the flat name received:

```html
<input name="a" value="x">
<input type="checkbox" name="a[b][]" value="y">
```

| `a[b][]` checkbox | JSON                     |
| ----------------- | ------------------------ |
| checked           | `{"a":"x","a[b][]":"y"}` |
| unchecked         | `{"a":"x","a[b][]":[]}`  |

Another checked box named `a[b][]` with value `z` would make it `["y","z"]`.
The draft doesn't resolve this type change (see [open question 4][open
questions]).

`append()`, `set()` and `delete()` clear contributions with the exact source
name they receive, after successful argument conversion. This applies to both
string and Blob overloads. In particular, `delete("tag[]")` removes the implicit
empty collection even though `has("tag[]")` was false. Mutations do not rescan
the form. Appending `tag[0]` instead suppresses an existing `tag[]` empty state
by occupying its target subtree; it does not rename either source name.

```html
<input type="checkbox" name="draft">
<input type="checkbox" name="tag[]" value="magic">
<select name="categories[]" multiple></select>
```

With no selections, the JSON is
`{"draft":false,"tag":[],"categories":[]}`. The visible entry list is empty.

```html
<input name="title" value="Unicorns" />
<input type="checkbox" name="tag[]" value="magic" checked />
<input type="checkbox" name="tag[]" value="horns" />
<input type="checkbox" name="draft" checked />
<input name="pet[0][name]" value="Hypatia" />
<input type="number" name="pet[0][legs]" value="4" />
<input name="pet[1][name]" value="Billie" />
<input type="number" name="pet[1][legs]" value="" />
```

```json
{
  "title": "Unicorns",
  "tag": ["magic"],
  "draft": true,
  "pet": [
    { "name": "Hypatia", "legs": 4 },
    { "name": "Billie", "legs": null }
  ]
}
```

#### Script and custom elements

Form-associated custom elements submit through
`ElementInternals.setFormValue()`, which takes a `File`, a string, a `FormData`
or `null`. Script can also add entries in the `formdata` event. Both can give
their string values an explicit type hint:

- The string-value overloads of `FormData`'s `append()` and `set()` take an
  optional third argument, a type hint. The `Blob` overloads keep their
  third-argument filename.
- `setFormValue()` takes an optional third argument, a type hint, after the
  existing restoration-state argument. It applies only to a string submission
  value. A `FormData` submission keeps per-entry JSON values, empty-state
  contributions and native no-selection provenance; a `File` or `null` is
  unaffected. Each call replaces the previous hint, defaulting to `"string"`,
  without changing restoration-state handling.
- Omitting the hint, or passing `"string"`, keeps the string. In particular,
  `data.append("page", 2)` still stores `"2"` and gives a JSON string. Passing
  `"number"` parses the stored string with the same rules as a native number
  control. A string they reject, such as `""`, `"abc"` or the overflowing
  `"1e400"`, gives JSON `null`. The rules read a leading number and ignore the
  rest, so `"12px"` gives `12`; a native number control's value sanitisation
  would have emptied that string, and FormValidation reports it as bad input.
- `"boolean"` gives `false` for the empty string and `true` for any other
  string, like JavaScript string truthiness. `"false"` and `"0"` both give
  `true`. Conversion happens after the value becomes a string.
- `get()`, `getAll()`, iteration and existing encodings keep the original
  strings. `toJSON()` and the new encodings use the JSON values;
  [FormValidation](form-validation.md) also inspects them for typed compatibility
  and requiredness.
- `null` still becomes `"null"` in `FormData`. Passing `null` to
  `setFormValue()` still submits nothing, regardless of the hint.

```js
const data = new FormData();
data.set("quantity", "42", "number");
data.set("enabled", "", "boolean");
data.set("draft", "on", "boolean");
data.toJSON(); // { quantity: 42, enabled: false, draft: true }
```

A custom checkbox can choose legacy omission on an unchecked state. Unlike the
native Boolean checkbox's JSON empty state, a null FACE submission stays absent:

```js
internals.setFormValue(
  this.checked ? this.value || "on" : null,
  this.checked ? "checked" : "unchecked",
  "boolean",
);
```

A toggle that wants an explicit `false` submits `""`.

A custom element submits structure by passing a `FormData` with bracket names
and per-entry hints:

```js
class LatLngInput extends HTMLElement {
  static formAssociated = true;
  #internals = this.attachInternals();
  set position({ lat, lng }) {
    const name = this.getAttribute("name");
    const data = new FormData();
    data.append(`${name}[lat]`, lat, "number");
    data.append(`${name}[lng]`, lng, "number");
    this.#internals.setFormValue(data);
  }
}
```

`<lat-lng-input name="home">` at `{ lat: 51.5, lng: -0.12 }` gives
`{"home":{"lat":51.5,"lng":-0.12}}`. Its urlencoded body is unchanged:
`home%5Blat%5D=51.5&home%5Blng%5D=-0.12`.

`JSON.stringify(new FormData(form))` gives a JSON body that keeps every value
for `fetch()`, and a `FormData` nested inside a larger object passed to
`JSON.stringify()` serialises too, the same use case that settled
[`URL.prototype.toJSON()`][whatwg/url#137]. If an indexed array has a gap,
either call throws a `TypeError`, including when the `FormData` is nested (see
[caveats] for how that affects generic serialisers). `toJSON()` sends no
requests, so it needs no CSRF analysis and doesn't depend on any Fetch change.

#### `FormData.fromJSON()`

Server runtimes implement `FormData` too. It is part of the WinterTC [Minimum
Common Web Platform API], and `request.formData()` is how a lot of server code
reads form bodies. `FormData.fromJSON()` gives that code the same view of a JSON
body:

```js
const data = FormData.fromJSON(await request.json());
data.get("pet[0][name]"); // "Hypatia"
data.getAll("tag[]"); // ["magic"]
```

- It returns a new `FormData` holding the canonical names of the value, in object
  key order and array element order.
- Typed values are stored as their string representation, with the type hint.
  So numbers use their JavaScript string conversion and store the type hint of
  "number"; booleans use `""` for `false` and `"true"` for `true`, with a type
  hint of "boolean". Applying their type hints gives the original JSON values
  back. `null` and empty arrays and objects become entries whose string is `""`
  and whose JSON value is `null`, `[]` or `{}`. An empty array is a leaf, so
  `{"tag":[]}` gives one visible entry named `tag`, while `{"tag":["a"]}` gives
  `tag[]`. These entries are public, unlike the JSON-only empty states they
  may have come from, so getters and legacy encodings see `tag=""`. No single
  exact-name validation rule addresses both spellings; see the FormValidation
  open question [One rule set for both representations][FormValidation open
  questions].
- The value is first serialised and parsed as JSON, so `toJSON()` methods run
  and `undefined` members are dropped, as with `JSON.stringify()`.
- It throws a `TypeError` if the value isn't an object at the top level, or if
  a canonical name would exceed the nesting limit, counting key and index
  segments but not a final `[]`. Nested arrays and objects are accepted,
  including mixed arrays. Whatever it accepts,
  `FormData.fromJSON(value).toJSON()` equals the value after that JSON step,
  except that lone surrogates in string values become U+FFFD REPLACEMENT
  CHARACTER. It accepts everything `toJSON()` returns.
- It can't produce `File` entries, because JSON has no files. Reading a
  `multipart/json` body with files needs its file parts too (see [open
  questions]).

Examples and their canonical names:

| Entry names, in order                                           | `toJSON()`                                                    | Canonical names                                       |
| --------------------------------------------------------------- | ------------------------------------------------------------- | ----------------------------------------------------- |
| `kids[1]=b`, `kids[0]=a`, `kids[2]=c`                           | `{"kids":["a","b","c"]}`                                      | `kids[]`, `kids[]`, `kids[]`                          |
| `pet[1][name]=Billie`, `pet[0][name]=Hypatia`, `pet[0][legs]=4` | `{"pet":[{"name":"Hypatia","legs":"4"},{"name":"Billie"}]}`   | `pet[0][name]`, `pet[0][legs]`, `pet[1][name]`        |
| `grid[1][0]=c`, `grid[0][1]=a`, `grid[0][0]=b`                  | `{"grid":[["b","a"],["c"]]}`                                  | `grid[0][]`, `grid[0][]`, `grid[1][]`                 |
| `items[1][name]=Hypatia`, `items[0]=loose`                      | `{"items":["loose",{"name":"Hypatia"}]}`                      | `items[0]`, `items[1][name]`                          |
| `kids[0]=a`, `kids[2]=c`                                        | throws a `TypeError`: position 1 is missing                   | none                                                  |
| `kids[99999]=c`                                                 | throws a `TypeError`: positions 0 to 99998 are missing        | none                                                  |
| `pet[0][tag][1]=x`                                              | throws a `TypeError`: the nested array lacks position 0       | none                                                  |
| `t[0]=x`, `t[2]=y`, `t[a]=z`                                    | throws a `TypeError`; density is checked before flattening    | none                                                  |
| `t[]=x`, `t[1]=y`                                              | throws a `TypeError`; append entries don't supply index 0    | none                                                  |
| `pet["1"][name]=Hypatia`                                        | `{"pet":{"1":{"name":"Hypatia"}}}`                            | `pet["1"][name]`                                      |
| `0=a`, `1["2"]=b`                                               | `{"0":"a","1":{"2":"b"}}`                                     | `0`, `1["2"]`                                         |
| `t[]=x`, `t[0]=y`                                               | `{"t[]":"x","t[0]":"y"}`                                      | `["t[]"]`, `["t[0]"]`                                 |
| `t[0]=x`, `t[a]=y`                                              | `{"t[0]":"x","t[a]":"y"}`                                     | `["t[0]"]`, `["t[a]"]`                                |
| `tag=a`, `tag=b`                                                | `{"tag":["a","b"]}`                                           | `tag[]`, `tag[]`                                      |
| `mix=s`, `mix[key]=k`, `["mix[key]"]=q`                         | `{"mix":"s","mix[key]":["k","q"]}`                            | `mix`, `["mix[key]"][]`, `["mix[key]"][]`             |
| `t[]=x`, `t[a]=y`                                               | `{"t[]":"x","t[a]":"y"}`                                      | `["t[]"]`, `["t[a]"]`                                 |
| `error[good]=1`, `error[bad=2`, `[x]=3`, `a[]b=4`               | `{"error":{"good":"1"},"error[bad":"2","[x]":"3","a[]b":"4"}` | `error[good]`, `["error[bad"]`, `["[x]"]`, `["a[]b"]` |
| `q[""]=e`, `q["a]b"]=f`                                         | `{"q":{"":"e","a]b":"f"}}`                                    | `q[""]`, `q["a]b"]`                                   |
| `__proto__[polluted]=yes`                                       | `{"__proto__":{"polluted":"yes"}}`, an own property           | `__proto__[polluted]`                                 |

### Alternatives considered

- **Flat, always arrays.** Types never vary, even for plain repeated names, but
  there is no nesting and no typing, and every single value is wrapped, as in
  `{"title":["Unicorns"]}`.
- **Arrays chosen by control.** Inferring arrays from control types or shared
  names can conflict with the server's expected shape. `[]` makes the author's
  choice explicit in the submitted name.
- **Last value wins for repeated plain names**, as in PHP and Rack. This matches
  more servers but silently drops data, a bug which this proposal aims to fix.
- **Dot syntax** (`pet.0.name` or `pet[0].name`), as in decode-formdata,
  Conform, ASP.NET Core and Spring. A browser without support falls back to
  urlencoded, which PHP, Rack and qs decode into the same shape only with
  bracket syntax.
- **Sparse positional indices**, as in the 2015 Note. Filling gaps with `null`
  makes an array's size depend on its largest label, and the padding can't be
  told apart from an empty number input's `null`.
- **Compacting labels**: sort them and close gaps. Output stays proportional to
  the entries and removing a row needs no renumbering, but a value can move to
  a position with different rules. If `surname[0]` is required and `surname[1]`
  optional, a lone `surname[1]` would land in the required slot.
- **Dense or object**, where an array level whose labels aren't exactly `0` to
  `n - 1` becomes an object with numeric keys. Omissions survive, but validation
  would need to treat numeric indices and their quoted object-key spellings as
  the same address. The field would be an array or an object depending on what
  was submitted; a schema can accept both with a union. This is the main
  alternative we want feedback on.
- **Dense array indices that stay flat** when not dense. Positions are
  preserved, but the whole group silently changes shape. The draft throws
  instead.
- **Density checked only on arrays that survive flattening.** Indexed names
  that conflict flattening keeps whole would then need no dense labels, so
  `t[0]`, `t[2]` alongside `t[a]` would give three flat keys instead of
  throwing. Either check is independent of entry order. The draft checks every
  parsed indexed path; the choice is [open question 3][open questions].
- **A non-throwing `toJSON()`** that falls back to another shape, such as
  dense or object, while submissions stay strict. Generic serialisers would
  never meet a `TypeError`, but script and submission would no longer share
  one representation (see [caveats]).
- **Rack's `p[][a]` rule**, which starts a new element when a key repeats.
  `p[][a]` then `p[][b]` could be one element or two, so it can't round-trip.
- **Typing inferred from values**, so `"4"` becomes `4`. This breaks values such
  as postcodes and phone numbers, and makes a field's type depend on what the
  user typed.

### Part 2: `enctype` keywords

1. Add an `application/json` keyword to the `enctype` and `formenctype`
   attributes.
2. When a form is submitted with method POST in that state, the body is the
   form's entry list converted exactly as `toJSON()` converts it, serialised as
   UTF-8 JSON. The form's `accept-charset` is ignored. If conversion throws,
   for example on non-dense array labels, no request is sent. How that failure
   is reported is not settled; see [open question 7][open questions].
3. The request's `Content-Type` is `application/json`.
4. `application/json` is not a CORS-safelisted `Content-Type`. When the
   navigation request is cross-origin, the browser sends a CORS preflight first
   and only sends the POST if the server approves it. Navigations include
   credentials, so approval requires `Access-Control-Allow-Origin` naming the
   exact origin and `Access-Control-Allow-Credentials: true`. The response to
   the POST itself needs no CORS headers, because it's rendered as a document
   and never exposed to the submitting page. Same-origin submissions are never
   preflighted.
5. Add a `multipart/json` keyword for forms that upload files. The request's
   `Content-Type` is `multipart/json`, a new media type with the same part
   syntax as `multipart/form-data`:
   - The first part is the JSON part. It has an empty field name and
     `Content-Type: application/json`. Its body is the same JSON as for
     `application/json`, so file fields appear in it by file name.
   - Each `File` entry follows as an ordinary file part containing the file's
     contents, in JSON traversal order (object key order and array element
     order), not necessarily entry-list order. Its field name is the canonical
     name of where its file name sits in the JSON. A repeated plain `photo`
     becomes `photo[]`, numeric labels keep their positions, and contradictory names
     are quoted. These names describe the draft wire shape; exact association
     for mixed string/file arrays needs an additional positional reference
     before a receiver can reconstruct those arrays unambiguously.
   - Non-file entries appear only in the JSON part.
   - `multipart/json` is not a CORS-safelisted `Content-Type`, so cross-origin
     submissions are preflighted exactly as for `application/json`.

Multipart fields are flat strings, and servers rebuild structure and types from
names and schemas. The JSON part has both, so upload endpoints get the same
body shape as `application/json` ones, in a single JSON parse.

The upload goal is streaming and early rejection, not approval before any file
bytes are transmitted. A streaming receiver can validate the JSON part before
consuming file bodies and stop on an error, but browser and network buffers may
already contain file bytes. File size declarations, when provided, are
untrusted; receivers must enforce actual byte limits while reading.

`request.json()` does not parse a multipart body. A multipart parser must expose
the first part as JSON and subsequent parts as file streams. Calling today's
`request.formData()` consumes the whole body before returning; extending its
media-type support would not provide incremental access. The receiver API,
upfront metadata and exact file-reference format remain design questions.

GET submissions (where the data goes in the URL query) and `mailto:` actions are
unchanged.

## Examples

A plain form:

```html
<form method="post" action="/api/posts" enctype="application/json">
  <input name="title" value="Unicorns" />
  <label
    ><input type="checkbox" name="tag[]" value="magic" checked /> Magic</label
  >
  <label
    ><input type="checkbox" name="tag[]" value="horns" checked /> Horns</label
  >
  <textarea name="body">
Line one
Line two</textarea
  >
  <button>Publish</button>
</form>
```

Submitting it sends:

```http
POST /api/posts HTTP/1.1
Content-Type: application/json

{"title":"Unicorns","tag":["magic","horns"],"body":"Line one\nLine two"}
```

A single button can opt in while the rest of the form keeps its existing
encoding:

```html
<form method="post" action="/legacy-endpoint">
  <input name="q" />
  <button>Submit</button>
  <button formaction="/api/search" formenctype="application/json">
    Submit as JSON
  </button>
</form>
```

Script sees the same representation:

```js
const data = new FormData(form);
JSON.stringify(data);
// '{"title":"Unicorns","tag":["magic","horns"],"body":"Line one\\nLine two"}'
```

A cross-origin submission from `https://app.example` to `https://api.example` is
preceded by a preflight:

```http
OPTIONS /api/posts HTTP/1.1
Origin: https://app.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type
```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: content-type
```

If the server does not respond like this, the navigation fails with a network
error and the POST is never sent.

Feature detection works through the `enctype` IDL attribute, which reflects only
known values:

```js
const form = document.createElement("form");
form.enctype = "application/json";
const supported = form.enctype === "application/json";
```

A form that uploads files uses `multipart/json`:

```html
<form method="post" action="/api/posts" enctype="multipart/json">
  <input name="title" value="Unicorns" />
  <input type="file" name="photo" />
  <button>Publish</button>
</form>
```

With `unicorn.png` selected, this sends:

```http
POST /api/posts HTTP/1.1
Content-Type: multipart/json; boundary=----b

------b
Content-Disposition: form-data; name=""
Content-Type: application/json

{"title":"Unicorns","photo":"unicorn.png"}
------b
Content-Disposition: form-data; name="photo"; filename="unicorn.png"
Content-Type: image/png

<file bytes>
------b--
```

## Caveats, shortcomings, and other drawbacks of design choices

- **`application/json` can't upload files.** File entries appear only as file
  names. Uploads need `multipart/json` (or `multipart/form-data`). Putting
  contents in the JSON (for example base64, as the 2015 Note did) would make
  file data about a third larger. A synchronous `FormData.prototype.toJSON()`
  can't read file contents, so including them would also mean giving up the
  shared representation.
- **`multipart/json` is a new media type.** It needs an IANA registration (the
  multipart registry has no `json` subtype). Alternatives considered:
  - Sending `multipart/form-data` would avoid the preflight, but the `enctype`
    keyword would then name a type that is never sent, unlike every existing
    keyword. It would also buy little compatibility. An existing parser would
    accept the body, but the app would see its files and an empty-named field,
    and none of its form fields, until it was changed to read the JSON part.
  - `multipart/form-data+json` would be truthful but misleading, because the
    `+json` suffix ([RFC 6839]) tells servers the whole body is JSON, which it
    isn't.

  `multipart/json` must also never be added to the CORS safelist. That would let
  any page send it with an arbitrary body via `fetch()` with `mode: "no-cors"`,
  and servers that match content types loosely would then parse attacker JSON
  without a preflight, which is the class of bug behind Fastify's
  [CVE-2022-41919].

- **Existing parsers reject `multipart/json`.** busboy, which multer uses,
  accepts only a `multipart/form-data` type and throws
  `Unsupported content type` otherwise. This fails loudly rather than silently
  losing fields, and every server has to change anyway to read the JSON part.
  The fix is a parser update, or treating `multipart/json` as
  `multipart/form-data` before parsing, since the part syntax is the same.
- **Servers find the JSON part by position.** It is always the first part and
  has an empty field name. Script and form-associated custom elements can also
  add empty-named entries. Their strings appear inside the JSON object, while
  files appear as file names in the JSON and as subsequent file parts.
  Parsers reusing `multipart/form-data` code may drop or mishandle an
  empty-named part, which needs surveying. OpenAPI's Encoding Object can
  describe a multipart part typed `application/json`, but parts are keyed by
  name, so describing this one needs a `""` property. How schema tooling handles
  that also needs checking.
- **File parts use canonical names.** Repeated plain `photo` produces `photo[]`
  parts, so lookup by control name can miss files.
- **Bracket and quote names gain meaning.** Only `toJSON()`, `fromJSON()` and
  the new keywords read them. Existing encodings and the other `FormData`
  methods are unchanged. Most existing names with brackets were written for PHP,
  Rails or qs, which mostly give them the same meaning. Other names that happen
  to contain brackets will nest, and invalid ones stay flat. Quoted keys
  (`q["a]b"]`) are new; canonical names use them where a key can't be written bare.
- **Array labels are positions, and must be dense.** At every array level the
  labels must be exactly `0` to `n - 1`; otherwise `toJSON()` throws a
  `TypeError` and an `application/json` or `multipart/json` submission sends
  no request. We chose this because positions can have different rules, for
  example `surname[0]` may be required and `surname[1]` optional. Compacting
  would move `surname[1]` into the required position, and `null` padding
  collides with the `null` of an empty number input. Keep persistent record IDs
  in values, or use quoted keys (`pet["7"]`) for an intentionally sparse object
  keyed by numeric ID. File-part names follow the canonical paths, but those
  names alone cannot identify positions in mixed string/file arrays; their
  reference format is still an open question. The check covers every name that
  parses as an indexed path, before conflict flattening, so `t[0]`, `t[2]`
  alongside `t[a]` throws even though flattening would keep all three whole.
  That is an admission policy, not something order independence needs, since
  checking only the arrays that survive flattening would be independent of
  entry order too. [Open question 3][open questions] asks which to adopt.
- **Dense-only rejects some ordinary forms.** Native submissions leave gaps when
  an optional indexed radio group or an explicit-valued checkbox submits
  nothing, a row is disabled or removed, a custom element submits `null`, or
  script changes the entry list. Boolean checkboxes and declared terminal-`[]`
  groups can retain positions through their existing empty states. A gap makes
  the new encodings fail outright, though legacy encodings are unaffected.
  Renumbering is a fix only when positions have no distinct meaning; it isn't
  a remedy for a skipped fixed question, and the browser never renumbers. The
  dense-or-object alternative keeps omissions at the cost of a schema accepting
  both shapes and different validation-name matching.

  An optional scalar radio group is the common case. Radios named `answer[1]`
  left unanswered between answered `answer[0]` and `answer[2]` leave a gap.
  Authors have two existing options, and neither is a transparent rename.
  Naming the group `answer[1][]` keeps the position, because an unchecked
  terminal-`[]` group records `[]`, but the field becomes a list, `[]` or
  `["yes"]`; it suits only fields where collection semantics and that group
  result are acceptable. Quoted numeric keys for every question,
  `answer["0"]`, `answer["1"]` and so on, give an object that can omit a key.
  Mixing them with unquoted labels at the same level throws if those labels
  aren't dense on their own; otherwise the conflict flattens the group. The
  draft doesn't give unanswered radios a `null` empty state, because `null`
  already means an empty or invalid number input and the two would collide.

  How a failed submission is surfaced is unsettled. We recommend that, before
  stage 2 ships, a conversion failure produce at least an author-visible
  diagnostic such as a console warning, and that a form-level event like
  `invalid` and accessible feedback for the user be considered. Which of these
  to add, and their names and details, are undecided ([open question
  7][open questions]); this draft defines none of them. We want framework,
  server and browser feedback on these omission workflows, the failure
  experience and the dense-only versus dense-or-object tradeoff.
- **A throwing `toJSON()` reaches generic serialisers.** `JSON.stringify()`
  calls `toJSON()` on a `FormData` wherever it sits in the value, so a logger,
  error reporter or persistence layer that serialises an object holding a
  `FormData` with an indexed gap gets a `TypeError` from code that never chose
  this encoding. A non-throwing `toJSON()`, falling back for example to the
  dense-or-object shape, with only `application/json` and `multipart/json`
  submissions strict, would avoid that. It would also split the shared
  representation this proposal is built on. Script would see one shape while
  the server received another, or nothing. This needs a decision, not a
  default ([open question 3][open questions]).
- **Numeric brackets differ from Rack.** `pet[1][name]` builds an array here,
  but an object in Rack. PHP and qs also interpret digit brackets as arrays,
  though their handling of sparse and very large labels differs. Quoted keys
  are new syntax, so numeric object keys need server support in the urlencoded
  fallback.
- **Repeated plain names change type.** Without `[]`, `tag` is a string with one
  entry and an array with two, as in qs and htmx. Collecting keeps every value,
  and a schema expecting a string rejects the array loudly, where last-wins
  would silently drop data. Authors avoid it with `[]`, and conformance checkers
  could warn when a checkbox group or `<select multiple>` name lacks it.
- **Native hints remain control-derived.** Hidden inputs and `<select>` stay
  strings, even for numeric IDs. There is no new markup type-hint attribute.
  Script and custom elements can supply explicit hints; servers still need
  coercion where their schemas expect numbers or booleans from these controls.
- **Numbers have double precision.** As with `valueAsNumber`,
  `<input type=number value=12345678901234567890>` gives `12345678901234567000`,
  so number inputs shouldn't hold IDs that require exact decimal digits.
- **Empty states need a participating control.** JSON includes `false` for an
  eligible unchecked Boolean checkbox and `[]` for an empty declared collection.
  Disabled or absent controls do not supply defaults. Legacy encodings still
  omit unchecked controls; servers accepting both formats must account for that
  difference. Explicitly valued scalar checkboxes still submit selected values
  rather than Boolean state.
- **Script can't create empty states.** Only HTML's entry-list construction
  records JSON-only empty states. A `FormData` passed to `setFormValue()`
  keeps any it already has, but no method creates one. A custom element
  or `formdata` handler can give `false` only as a visible entry (`""` with the
  `"boolean"` hint), which legacy encodings send, and `[]` only through a
  `FormData.fromJSON()` result, where `{"tags":[]}` is a visible `tags=""`
  entry. Whether to add an API for script-created empty states, and its shape,
  is [open question 4][open questions]; none is proposed.
- **Types follow controls or explicit hints.** Changing `type=text` to
  `type=number`, or changing a script-supplied hint, changes the JSON type and
  can break a server contract.
- **Reading and rewriting entries loses JSON information.** For non-file
  entries, `get()`, `getAll()` and iteration expose stored strings, not internal
  JSON values or empty-state contributions. Copying through iteration loses
  both hints and implicit empty states. `data.set(name, data.get(name))` replaces
  a typed entry with an untyped one and clears that name's contribution. Script
  must supply hints again, or, when labels are dense, use
  `FormData.fromJSON(data.toJSON())`, though that round trip replaces files with
  names and canonicalises field names. It also materialises JSON-only empty
  states as ordinary public entries, so `draft: false` becomes a visible
  `draft=""`, and an empty `tag[]` group becomes `tag=""` with JSON value
  `[]`. Getters and legacy encodings then expose these entries, and deletion
  must use their new canonical names.
- **Nesting needs a limit.** Deep names such as `a[b][c][d]...` cost recursion
  in the browser and the server. qs stops nesting after 5 levels by default and
  Rack raises an error at 32. Past some limit, names stay flat, and `fromJSON()`
  throws on deeper values, because it couldn't give them back. The draft counts
  key and index segments and doesn't count a final `[]`, both when parsing
  names and when producing canonical names. Otherwise a name at the limit used
  twice would give an array whose canonical name, with its added `[]`, exceeds
  the limit, and `fromJSON()` would reject what `toJSON()` returned. If the
  limit were 32, for example, a name with 32 keys used twice gives a canonical
  name with those 32 keys followed by `[]`, which stays within it. Flattening
  repeated values at the limit is the alternative ([open question
  2][open questions]).
- **Schema compatibility still needs checking.** Typed, nested output doesn't
  meet every OpenAPI or JSON Schema body. Schemas must account for disabled or
  absent controls, empty number inputs giving `null`, and repeated plain names
  changing from scalars to arrays. Values that remain strings may need coercion,
  for example Ajv's `coerceTypes` [option], which
  Fastify enables by default for every request part, including the body
  ([`default-ajv-options.js`]).
- **Order across names is lost.** Values keep their order within a name, but
  interleaving between different names is not preserved. Servers that need the
  full ordered entry list can keep using `multipart/form-data`.
- **Newlines are not normalised.** The other encodings convert newlines to CRLF,
  while JSON keeps line feeds as they are in the DOM. This matches what
  `fetch()` with `JSON.stringify(new FormData(form))` would send.
- **Fallback doesn't preserve every shape or upload files.** Browsers that
  don't support the keyword fall back to `application/x-www-form-urlencoded`.
  Values become strings, and PHP, Rack and qs differ on digit keys, repeated
  plain names, quoted keys and conflicting paths. Servers accepting both formats
  must handle those differences. For `multipart/json`, the fallback sends file
  names, not file contents. Authors needing uploads in older browsers must use
  `multipart/form-data`, for example as the baseline before feature-detecting
  and selecting `multipart/json`.
- **Cross-origin needs server opt-in.** Cross-origin JSON submissions work only
  if the server answers the preflight with credentialed CORS headers.
  Compared with the other encodings:

  | `Content-Type`                                        | `<form>`          | `fetch()`, `no-cors`      | `fetch()`, `cors` |
  | ----------------------------------------------------- | ----------------- | ------------------------- | ----------------- |
  | urlencoded, `multipart/form-data`, `text/plain`       | sent              | sent                      | sent              |
  | `application/json`                                    | never             | never (header is dropped) | after a preflight |
  | `application/json` or `multipart/json`, with proposal | after a preflight | never                     | after a preflight |

  A 307/308 redirect keeps the body and is preflighted
  again against the new origin. A 303 redirect drops the body and its
  `Content-Type`, so the common POST/redirect/GET pattern works without a
  preflight for the GET.

- **Other request headers in the preflight.** Any other non-safelisted headers
  the browser adds to navigation requests before the preflight (for example
  `Upgrade-Insecure-Requests`, depending on where that spec hooks in) would also
  appear in `Access-Control-Request-Headers`, and servers would need to allow
  them. This, and whether servers need a signal that a preflight is for a
  navigation (for example `Sec-Fetch-Dest`), are questions for
  [whatwg/fetch#1785], and apply equally to PUT, PATCH and DELETE forms.
- **Libraries that detect JSON bodies by `toJSON`.** Some fetch wrappers decide
  to send a body as JSON because `typeof body.toJSON === "function"`. Under Bun,
  old versions of [ofetch], which is Nuxt's `$fetch`, sent FormData bodies as
  `application/json`. That was fixed in [unjs/ofetch#483] (Aug 2025), which
  checks `instanceof FormData` first. Bun's tracker blames the same mechanism
  for broken multipart uploads through node-fetch and axios
  ([oven-sh/bun#21467]). That is the reporter's diagnosis and hasn't been
  confirmed. An audit of current main branches found no other affected library.
  axios, redaxios, wretch and Angular HttpClient check for FormData first. ky,
  got, hono/client and tRPC only serialise JSON when you ask for it explicitly.
  Native fetch, node-fetch and undici never call `toJSON()`. Wrappers that do
  serialise a `FormData` as JSON can now encounter a `TypeError` for indexed
  gaps, including when it is nested in a larger object. The remaining detection
  risk is sites still bundling ofetch from before Aug 2025. Browsers would need
  telemetry or a staged rollout to measure that.

## Open questions

In the order they would block shipping, with who is best placed to answer.
Questions 1 to 4 can change what `toJSON()` returns, so they must be settled
before it ships.

1. **Repeated plain names** (stage 1; framework and server authors): collect
   values into an array, as qs and the Note do, or keep the last, as PHP and
   Rack do?
2. **Nesting limit** (stage 1; browser vendors and server authors): how many
   key and index segments can a name have before it stays flat, and
   `fromJSON()` throws? The draft doesn't count a final `[]`, in parsing and in
   canonical names, so a name at the limit can still append and a repeated
   name at the limit still round-trips. The alternative is to count every
   segment and flatten repeated values at the limit. That keeps one measure for
   all segments, but a name at the limit would change shape when a second value
   appears, and a `[]` name at the limit would stay flat.
3. **Array labels** (stage 1; framework, server and browser authors): is
   rejecting non-dense labels acceptable for repeatable-row and optional-field
   workflows, or is the dense-or-object shape better? The draft checks density
   on every parsed indexed path before conflict flattening, so `t[0]`, `t[2]`
   alongside `t[a]` throws; should only arrays that survive flattening be
   checked, letting that example flatten? Should `toJSON()` stay strict, or
   fall back without throwing while submissions stay strict, at the cost of a
   representation that differs between script and submission? Should
   unanswered radios get an empty state, given that `null` already means empty
   numeric input? Can servers support quoted numeric object keys in the
   urlencoded fallback?
4. **Empty states from script** (stage 1; framework and custom element
   authors): script and custom elements can't create JSON-only empty states,
   and `fromJSON()` turns `[]` into a visible entry such as `tags=""`. Is an
   API for script-created empty states needed, and what shape should it take?
   None is proposed. Relatedly, conflict flattening keeps a terminal-`[]` name
   whole, so its value is a string with one checked box and `[]` with none;
   should that change?
5. **`toJSON` detection** (stage 1; site owners, and browsers through
   telemetry): how many sites still bundle ofetch from before Aug 2025, which
   sends any body with a `toJSON` method as JSON?
6. **Preflight details** (stage 1; browser vendors, in [whatwg/fetch#1785]): are
   browser-added headers such as `Upgrade-Insecure-Requests` excluded from
   `Access-Control-Request-Headers`, and do servers need a signal that a
   preflight is for a navigation?
7. **Conversion failure feedback** (stage 2; browser vendors and accessibility
   experts): a submission whose conversion throws sends no request. We
   recommend that, before stage 2 ships, this produce at least an
   author-visible diagnostic such as a console warning. Should there also be a
   form-level event like `invalid`, and accessible feedback for the user? Their
   names, timing and details are undecided.
8. **Is `multipart/json` worth a new media type?** (stage 2; server authors): is
   a typed, nested JSON part worth a registration and parser changes, over
   rebuilding structure from `multipart/form-data` field names?
9. **Parser support** (stage 2; multipart parser authors): how quickly would
   multipart parsers accept `multipart/json`, and do they and schema tooling
   handle the empty-named JSON part and quoted part names?
10. **`formData()` on JSON bodies** (stage 2; server runtime authors): should
    `Request` and `Response`'s `formData()` accept `application/json` through
    `fromJSON()`, and reconstruct `multipart/json` using its file parts? This
    depends on the unambiguous file-reference format in question 11; replacing
    filenames by canonical-name lookup alone is insufficient.
11. **Incremental multipart consumption** (stage 2; server runtime and parser
    authors): expose the JSON part before file bodies, then bounded file
    streams with cancellation and backpressure. Define upfront file metadata
    and unambiguous references into the JSON before relying on declared-size
    rejection or alternative file scheduling. Canonical append names alone
    cannot locate a file among identical string values in a mixed array.

## Draft specification

The change spans three WHATWG standards.

### XMLHttpRequest

An entry gains an optional **JSON value**: a number, a boolean, null, an empty
list, or an empty map. [Create an entry] gains an optional _jsonValue_ argument,
which `fromJSON()` passes. Its existing value argument is unchanged.

Each ordinary entry also has a **JSON-omission flag**, initially false. HTML
sets it only for a native no-selection multiple-file placeholder with a valid
terminal-append name. The flag affects JSON conversion and `multipart/json`
file-part emission, not public accessors, validation or legacy encodings.

An entry list also has a separate ordered list of **empty-state contributions**,
initially empty. Each has a source name and a JSON value of false or an empty
list. They are not entries, so existing accessors, iteration and legacy encoders
continue to use only the ordinary entry list. Entry-list cloning preserves the
contributions and native no-selection provenance.

**Get active empty states**, given an entry list:

1. Parse each ordinary entry's name as below, using its whole name as a single
   string path on failure. Exclude native no-selection placeholders marked for
   JSON omission. Keep these paths and the original entry references.
2. Retain a false contribution only if no ordinary entry has its exact source
   name. Parse its path with the same invalid-name fallback.
3. For each empty-list contribution, parse its source name and remove the final
   append segment to obtain its target. These names are valid terminal-append
   paths by construction.
4. Suppress an empty-list contribution if an ordinary path or retained false
   contribution is its target or a descendant, or if another empty-list target
   is a strict descendant. Compare path segments, including their types.
5. Return retained contributions in capture order. For JSON conversion only,
   coalesce false contributions with the same exact source name, and
   empty-list contributions with equal targets, keeping the first of each.
   False contributions with different source names are not coalesced, even if
   their paths are equal. Validation retains each active source-name record.

This helper does not mutate the stored snapshot or apply conflict flattening.
Suppressed contributions can become active after differently named values are
removed. Exact-source-name mutations, in contrast, remove contributions.

**Apply a type hint**, given an entry _entry_ and a _typeHint_:

1. If _entry_'s value is not a string, then return.
2. If _typeHint_ is `"number"`:
   1. Let _number_ be the result of the [rules for parsing floating-point number
      values] given _entry_'s value.
   2. Set _entry_'s JSON value to null if _number_ is an error, or to _number_
      otherwise.
3. Otherwise, if _typeHint_ is `"boolean"`, set _entry_'s JSON value to false if
   its value is the empty string, or to true otherwise.

Apply type hints only when creating entries, not when serialising or reading
them. The `"string"` hint leaves the JSON value absent, so the original string
is used.

A **JSON leaf** is a string, a number, a boolean, null, an empty list, or an
empty map.

**Parse a quoted form key**, given a string _name_ and a position variable
_position_ pointing at U+0022 ("):

1. If no prefix of the code points from _position_ matches the JSON `string`
   production, then return failure.
2. Let _literal_ be the shortest such prefix, and advance _position_ past it.
3. Return the result of [parsing a JSON string to an Infra value] given
   _literal_.

The **nesting depth** of a path is its size, not counting a final _append_.
The nesting limit (see [open questions]) bounds the nesting depth both of
parsed names and of canonical names, so the _append_ that canonicalisation adds
to a repeated name at the limit doesn't take it past the limit.

**Parse a form entry name**, given a string _name_:

1. Let _position_ be a position variable for _name_, initially pointing at its
   start.
2. Let _path_ be « ».
3. If the code point at _position_ is U+005B ([):
   1. Advance _position_ by 1.
   2. If the code point at _position_ is not U+0022 ("), then return failure.
   3. Let _key_ be the result of parsing a quoted form key given _name_ and
      _position_. If _key_ is failure, then return failure.
   4. If the code point at _position_ is not U+005D (]), then return failure.
   5. Advance _position_ by 1, and append _key_ to _path_.
4. Otherwise:
   1. Let _key_ be the result of collecting a sequence of code points that are
      not U+005B ([) from _name_ given _position_.
   2. If _key_ is the empty string, then return failure.
   3. Append _key_ to _path_.
5. While _position_ does not point past the end of _name_:
   1. If the code point at _position_ is not U+005B ([), then return failure.
   2. Advance _position_ by 1.
   3. If the code point at _position_ is U+0022 ("):
      1. Let _key_ be the result of parsing a quoted form key given _name_ and
         _position_. If _key_ is failure, then return failure.
      2. If the code point at _position_ is not U+005D (]), then return failure.
      3. Advance _position_ by 1, append _key_ to _path_, and continue.
   4. Let _key_ be the result of collecting a sequence of code points that are
      not U+005D (]) from _name_ given _position_.
   5. If _position_ points past the end of _name_, then return failure.
   6. Advance _position_ by 1.
   7. If _key_ is the empty string, then: if _position_ does not point past the
      end of _name_, return failure; otherwise append _append_ to _path_.
   8. Otherwise, if _key_ matches `0` or a non-zero ASCII digit followed by zero
      or more ASCII digits, append its non-negative integer value to _path_.
      This is a position, not an allocation size.
   9. Otherwise, append _key_ to _path_.
6. If _path_'s nesting depth is greater than the nesting limit, then return
   failure.
7. Return _path_.

An **item** has a _path_, a _value_, a _sourceName_, an optional _entry_ reference,
a _flat_ flag, and a **JSON path**, a list of strings and non-negative integers
that may end with _append_. A contribution item has no ordinary entry reference.

**Build a form JSON node**, given a list of items _items_, an integer _depth_,
and a list _orderedEntries_:

1. If, for every _item_ of _items_, _item_'s path has size _depth_, or has size
   _depth_ + 1 and ends with _append_:
   1. Append each non-null entry reference of _items_, in order, to
      _orderedEntries_.
   2. If _items_ has one item and its path has size _depth_, then return its
      value.
   3. Append _append_ to each item's JSON path.
   4. Return the list of _items_' values.
2. If, for every _item_ of _items_, _item_'s path has size greater than _depth_
   and _item_'s path[_depth_] is a string:
   1. Let _map_ be a new ordered map.
   2. For each _key_ → _group_ of _items_ grouped by path[_depth_], in
      first-seen order:
      1. Append _key_ to the JSON path of each item of _group_.
      2. Let _child_ be the result of building a form JSON node given _group_
         and _depth_ + 1, and _orderedEntries_.
      3. If _child_ is failure, then return failure.
      4. Set _map_[_key_] to _child_.
   3. Return _map_.
3. If, for every _item_ of _items_, _item_'s path has size greater than _depth_
   and _item_'s path[_depth_] is a non-negative integer:
   1. Let _list_ be a new list.
   2. For each _label_ → _group_ of _items_ grouped by path[_depth_], in
      ascending numeric label order:
      1. Let _index_ be _list_'s size.
      2. Append _index_ to the JSON path of each item of _group_.
      3. Let _child_ be the result of building a form JSON node given _group_
         and _depth_ + 1, and _orderedEntries_.
      4. If _child_ is failure, then return failure.
      5. Append _child_ to _list_.
   3. If every child of _list_ is a JSON leaf, then set the JSON path[_depth_]
      of every item of _items_ to _append_.
   4. Return _list_.
4. Return failure.

**Convert an entry list to a JSON value**, given an entry list _entryList_:

1. Let _items_ be « ».
2. For each _entry_ of _entryList_ not marked for JSON omission:
   1. Let _value_ be _entry_'s JSON value if it has one, otherwise _entry_'s
      value's `name` if _entry_'s value is a `File` object, otherwise _entry_'s
      value.
   2. Let _path_ be the result of parsing a form entry name given _entry_'s
      name, and let _flat_ be false.
   3. If _path_ is failure, then set _path_ to « _entry_'s name » and _flat_ to
      true.
   4. Append a new item whose path is _path_, whose value is _value_, whose
      sourceName is _entry_'s name, whose entry is _entry_, and whose flat flag
      is _flat_ to _items_.
3. After all ordinary entries, append an item for each active empty state,
   coalescing false contributions with the same source name and equal
   empty-list targets as above. Use its source name, JSON value and parsed
   target path, with a null entry reference. A false
   contribution uses normal parsing, including the invalid-name fallback.
   An empty-list contribution uses its target path.
   Set its flat flag to true if parsing failed, or false otherwise.
4. Run ? check array density given _items_.
5. While true:
   1. Let _result_ be a new ordered map, let _failed_ be false, and let
      _orderedEntries_ be a new list.
   2. For each _key_ → _group_ of _items_ grouped by path[0], in first-seen
      order:
      1. Set the JSON path of each item of _group_ to « _key_ ».
      2. Let _node_ be the result of building a form JSON node given _group_
         and 1, and _orderedEntries_.
      3. If _node_ is not failure, then set _result_[_key_] to _node_.
      4. Otherwise, set _failed_ to true, and for each item of _group_ whose
         flat flag is false, set its path to « its sourceName » and its flat
         flag to true.
   3. If _failed_ is false:
      1. For each _entry_ of _orderedEntries_ at index _index_, set _entry_'s
         **JSON order** to _index_.
      2. Return _result_.

The loop ends. A group of flat items always builds, because all their paths have
size 1, so every round that fails flattens at least one more item. Each ordinary
entry's **JSON path** is its item's JSON path when the loop ends. Contribution
items affect JSON output but never become file parts.

**Serialise a form key**, given a string _key_ and a boolean _first_: if _key_
is not the empty string, contains no U+005B ([), U+005D (]), U+0022 (") or
surrogate, and either _first_ is true or _key_ does not match an array label,
return _key_. Otherwise, return the result of [serializing a JavaScript value
to a JSON string] given _key_. The first key of a name never parses as an
index, so a bare digit key there, such as a top-level `0`, round-trips; a
later digit key must be quoted to stay an object key.

**Serialise a JSON path**, given a JSON path _path_: let _name_ be the result
of serialising the form key _path_[0] with _first_ true, wrapped in `[` and
`]` if it isn't _path_[0] itself. For each later item, append `[]` if it is
_append_; `[`, its decimal representation, and `]` if it is an integer; and
otherwise `[`, the result of serialising that string as a form key with
_first_ false, and `]`. Return _name_.

**Get the canonical entries**, given a map _value_: let _entries_ be « », run
the following steps given « _key_ » and _child_ for each _key_ → _child_ of
_value_, and return _entries_. Given a JSON path _path_ and a value _node_:

1. If _path_'s nesting depth is greater than the nesting limit, then throw a
   `TypeError`.
2. If _node_ is a JSON leaf, then append (serialise _path_, _node_) to
   _entries_.
3. Otherwise, if _node_ is a list:
   1. If every item of _node_ is a JSON leaf, then for each _item_ of _node_,
      append (serialise _path_ with _append_ appended, _item_) to _entries_.
      The final _append_ doesn't add to the nesting depth.
   2. Otherwise, for each _item_ of _node_ at index _index_, run these steps
      given _path_ with the integer _index_ appended and _item_.
4. Otherwise, for each _key_ → _child_ of _node_, run these steps given _path_
   with _key_ appended and _child_.

**Check array density**, given a list of items _items_:

1. For each _item_ of _items_ whose flat flag is false, and each index _i_ of
   _item_'s path where path[_i_] is a non-negative integer, record path[_i_] as
   a label of the prefix path[0] to path[_i_ - 1], comparing prefix segments
   including their types.
2. For each prefix with recorded labels, let _n_ be the number of distinct
   labels. If any of its labels is _n_ or greater, then throw a `TypeError`.

Distinct labels all below _n_ are exactly `0` to _n_ - 1. This runs before
conflict flattening, over every item parsed as a path, and before any indexed
output list is built. Density is an admission rule for parsed indexed paths, so
adding a contradictory field cannot make an otherwise non-dense indexed path
acceptable. Thus `t[0]`, `t[2]` alongside `t[a]` throws even though flattening
would preserve all three names. This keeps gap acceptance independent of the
presence of conflicting fields; it isn't needed for entry-order independence.
Checking only arrays that survive flattening would be order-independent too.
Which policy to use is an [open question][open questions]. The check needs work
proportional to the total length of the names. Implementations can keep labels
as digit strings, and a label with more digits than _n_ fails without being
converted to an integer or sizing any allocation.

The `toJSON()` method steps are to return the result of [converting an Infra
value to a JSON-compatible JavaScript value] given the result of ? converting
an entry list to a JSON value given this's entry list, rethrowing its
`TypeError`. That conversion creates null-prototype objects with
`CreateDataPropertyOrThrow`, so no name can reach a prototype.

The static `fromJSON(value)` method steps are:

1. Let _map_ be the result of [parsing a JSON string to an Infra value] given ?
   [serializing a JavaScript value to a JSON string] given _value_.
2. If _map_ is not a map, then throw a `TypeError`.
3. Let _formData_ be a new `FormData`.
4. For each (_name_, _leaf_) of the canonical entries given _map_:
   1. If _leaf_ is a string, then let _entry_ be the result of creating an entry
      with _name_ and _leaf_.
   2. Otherwise, if _leaf_ is a number or boolean:
      1. Let _string_ be the empty string if _leaf_ is false, or ToString(_leaf_)
         otherwise.
      2. Let _entry_ be the result of creating an entry with _name_, _string_,
         and _jsonValue_ _leaf_.
   3. Otherwise, let _entry_ be the result of creating an entry with _name_, the
      empty string, and _jsonValue_ _leaf_.
   4. Append _entry_ to _formData_'s entry list.
5. Return _formData_.

For every _map_ that step 4 accepts, converting _formData_'s entry list to a
JSON value gives _map_ back, with lone surrogates in string values replaced,
because the canonical names parse back to the keys they were serialised from,
no two canonical names contradict each other, and the JSON paths match how each
value was reached. Every value `toJSON()` returns is accepted, including an array
from a repeated name at the nesting limit, because the final _append_ of its
canonical name doesn't count towards the nesting depth.

Replace the string-value overloads of `append()` and `set()` with the following
signatures, leaving their `Blob` overloads unchanged:

```webidl
enum FormDataTypeHint { "string", "number", "boolean" };

partial interface FormData {
  undefined append(USVString name, USVString value,
                   optional FormDataTypeHint typeHint = "string");
  undefined set(USVString name, USVString value,
                optional FormDataTypeHint typeHint = "string");
  object toJSON();
  [NewObject] static FormData fromJSON(any value);
};
```

The string-value overloads create an entry with _name_ and _value_, then apply
_typeHint_ to that entry before appending or replacing it. After argument
conversion succeeds, both overloads of `append()` and `set()`, and `delete()`,
remove every empty-state contribution with source name _name_, then perform
their existing entry operations. Newly created entries do not inherit a native
no-selection placeholder's JSON-omission flag.

A `set()` call without a hint replaces any previous JSON value with an ordinary
string entry. Web IDL converts _value_ to `USVString` before these steps, so a
numeric `0` with the `"boolean"` hint gives the string `"0"` and JSON value true.

`object` is a JSON type, so this is a valid `toJSON` operation. A precise record
type isn't possible, because the value is recursive.

### HTML

- In [constructing the entry list], apply a type hint to each newly created
  entry for:
  - an `input` element in the Number or Range state: `"number"`;
  - an `input` element in the Checkbox state with no `value` attribute:
    `"boolean"`. Its existing value is `"on"`, so its JSON value is true.
    Other native controls get no hint.
- During entry-list construction, before the `formdata` event, record empty
  states for participating native controls. Use the ordinary participation
  filters, including inherited disabledness and `datalist` ancestry, but do not
  skip an unchecked checkbox/radio before considering its empty state:
  - If parsing a checkbox/radio name gives a terminal append path, record an
    empty-list contribution when unchecked. This takes precedence over false.
  - Otherwise an unchecked checkbox with no `value` attribute records false.
  - A multiple select with a terminal-append name and no selected non-disabled
    options records an empty-list contribution.
  - A multiple file input with a terminal-append name and no selected files
    records an empty-list contribution. Mark only its native empty `File`
    placeholder for omission from JSON conversion and `multipart/json` parts.
  - Other controls, including custom elements submitting null, gain no implicit
    empty state. Capture source names as the same scalar strings used by entry
    creation, and preserve contributions when cloning the completed entry list.
- Add an optional third argument to `ElementInternals`'s [`setFormValue()`],
  leaving its value and restoration-state arguments unchanged:

  ```webidl
  undefined setFormValue((File or USVString or FormData)? value,
                         optional (File or USVString or FormData)? state,
                         optional FormDataTypeHint typeHint = "string");
  ```

  Each form-associated custom element has a submission type hint, initially
  `"string"`. After the existing FACE check, `setFormValue()` sets that hint
  to _typeHint_ and runs its existing submission-value and restoration-state
  steps unchanged. In the [entry construction algorithm], apply the element's
  submission type hint to the newly created entry for a string submission
  value, before appending it. A null submission still produces no entry, and
  a `File` is unchanged. For a list of entries, append them with their existing
  JSON values and no-selection provenance unchanged, and copy its empty-state
  contributions with their original source names. The element's hint does not
  apply to the list.

- Add the "`application/json`" keyword to the [`enctype` and `formenctype`]
  enumerated attributes. It maps to a new `application/json` state. The missing
  value default and invalid value default are unchanged.
- In [Submit as entity body], add a case for `application/json`:
  1. Let _map_ be the result of running convert an entry list to a JSON value
     given _entry list_. If this throws, end these steps, so no request is sent
     and nothing navigates. How the failure is reported is unresolved; open
     question 7 recommends at least an author-visible diagnostic before this
     ships, and these steps don't mean the failure stays silent.
  2. Let _body_ be the result of running [serialize an Infra value to JSON
     bytes] given _map_.
  3. Let _mimeType_ be `` `application/json` ``.
- Give the [`multipart/form-data` encoding algorithm] an optional byte sequence
  _json_. If _json_ is given, the output starts with an extra part whose field
  name is the empty string, whose body is _json_, and whose `Content-Type` is
  `application/json`. _json_ is not affected by the form's encoding. Existing
  callers, including Fetch's `FormData` body extraction, don't pass it and are
  unaffected.
- Add the "`multipart/json`" keyword and state. In Submit as entity body, add a
  case for `multipart/json`:
  1. Let _map_ be the result of running convert an entry list to a JSON value
     given _entry list_. If this throws, end these steps as for
     `application/json`.
  2. Let _json_ be the result of running serialize an Infra value to JSON bytes
     given _map_.
  3. Let _files_ be a new entry list containing each ordinary entry whose value
     is a `File` and which is not marked for JSON omission, in ascending JSON
     order, with the same value and, as its name, the serialisation of that
     entry's JSON path.
  4. Let _body_ be the result of running the `multipart/form-data` encoding
     algorithm with _files_, _encoding_, and _json_.
  5. Let _mimeType_ be `multipart/json; boundary=` followed by the boundary
     string.
- Register `multipart/json` in HTML's IANA considerations, defined as the
  `multipart/form-data` syntax whose first part is the JSON part.

No change to navigation is needed. HTML already sets the request's
`Content-Type` from the POST resource.

### Fetch

This proposal depends on [whatwg/fetch#1785], which lets navigations run the
existing CORS-preflight check in HTTP fetch. That check sends a preflight only
when the request's method or headers are not CORS-safelisted, so a navigation
with `` `Content-Type: application/json` `` or `` `multipart/json` `` is
preflighted, and the existing enctypes, which use safelisted content types, are
not. Response tainting stays "`basic`", so the navigation response itself is not
CORS-checked. No further Fetch change is needed.

## Incubation and/or standardization destination

WHATWG XMLHttpRequest, Fetch and HTML, in two stages:

1. **XMLHttpRequest with a small HTML change, and Fetch, in parallel.** Neither
   depends on the other.
   - **XMLHttpRequest:** `FormData.prototype.toJSON()`, `FormData.fromJSON()`,
     conversion, type-hint arguments, and JSON-only empty-state snapshots and
     mutation rules, together with HTML's native hints and empty-state capture
     and the type-hint argument of `setFormValue()`. They ship together because
     adding nesting, types or empty-state semantics later changes `toJSON()`.
     This stage lets real use test the shape before HTML submission depends on
     it, while a staged rollout can measure compatibility risk.
   - **Fetch:** [whatwg/fetch#1785], already proposed for PUT, PATCH and DELETE
     forms ([whatwg/html#11347]), lets cross-origin navigations preflight.
2. **HTML:** the `application/json` and `multipart/json` keywords, built on
   both. Their steps use the same conversion as `toJSON()`, and cross-origin
   submissions rely on the preflight from [whatwg/fetch#1785].

Each stage lands with web-platform-tests. For stage 1, the tests cover nesting,
arrays of objects, nested and mixed arrays, shared labels grouping fields,
dense labels in any entry order, `TypeError`s for missing, large and nested
missing labels (including alongside contradictory names, such as `t[0]`,
`t[2]` and `t[a]`), quoted numeric object keys, leading-zero keys, bare
top-level digit keys with quoted nested digit keys in canonical names, the
nesting limit with a final `[]` not counted (a repeated name at the limit
round-trips through `fromJSON()`),
the flat fallback for invalid and contradictory names (including append/index
and object/index conflicts), typed controls, the type-hint arguments of `append()`,
`set()` and `setFormValue()` keeping today's strings for `get()` and existing
encodings, empty-string boolean conversion, invalid and overflowing numbers
(such as `1e400`) giving null while keeping their strings,
replacement clearing old JSON values, restoration-state handling unchanged,
per-entry JSON values surviving FACE submission, script-added values without
hints staying strings, `__proto__` names, `JSON.stringify()` of a `FormData`,
and `fromJSON()`: canonical names, its `TypeError`s, and the round trip both
ways. The navigation preflight tests belong to [whatwg/fetch#1785]. Stage 2
covers the body shape, `formenctype` precedence, feature detection, conversion
failures sending no request, and cross-origin preflight including redirect
handling, for both keywords. For `multipart/json`, the tests would also cover
the `Content-Type`, part order, the JSON part's headers, and canonically named
file parts in JSON traversal order,
including files submitted with dense labels out of numeric order.
Empty-state coverage includes unchecked scalar booleans, empty declared groups,
disabled controls and options, explicit empty option values, native file
placeholders versus script-added files, duplicate and nested fallback targets,
repeated false contributions under one source name coalescing while
differently spelled names for the same path don't, checked and unchecked
terminal-`[]` names that conflict flattening keeps whole,
exact-name mutation clearing, alternate indexed paths suppressing defaults,
and legacy getters and encodings remaining unchanged. Incremental receiver
coverage must check early JSON availability and cancellation without asserting
that no file bytes were sent.

[Keith Cirkel]: https://github.com/keithamus
[sveltekit-superforms]: https://github.com/ciscoheat/sveltekit-superforms/blob/main/src/lib/formData.ts
[Learn, chapter 11]: https://nextjs.org/learn/dashboard-app/mutating-data
[hono-source]: https://github.com/honojs/hono/blob/main/src/utils/body.ts
[`formDataToObject`]: https://github.com/withastro/astro/blob/main/packages/astro/src/actions/runtime/server.ts
[`json-enc`]: https://github.com/bigskysoftware/htmx-extensions/blob/main/src/json-enc/json-enc.js
[FastAPI]: https://fastapi.tiangolo.com/
[CVE-2026-25150]: https://github.com/QwikDev/qwik/security/advisories/GHSA-xqg6-98cw-gxhq
[Declarative Web]: https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb
[Extensible Web Manifesto]: https://extensiblewebmanifesto.org/
[W3C HTML JSON form submission]: https://www.w3.org/TR/html-json-forms/
[whatwg/html#1667]: https://github.com/whatwg/html/issues/1667
[whatwg/xhr#84]: https://github.com/whatwg/xhr/issues/84
[whatwg/url#143]: https://github.com/whatwg/url/issues/143
[whatwg/url#137]: https://github.com/whatwg/url/issues/137
[bun-source]: https://github.com/oven-sh/bun/blob/bun-v1.4.2/src/jsc/bindings/webcore/JSDOMFormData.cpp
[option]: https://github.com/ajv-validator/ajv/blob/master/docs/coercion.md
[`default-ajv-options.js`]: https://github.com/fastify/ajv-compiler/blob/master/lib/default-ajv-options.js
[ofetch]: https://github.com/unjs/ofetch/issues/439
[unjs/ofetch#483]: https://github.com/unjs/ofetch/pull/483
[oven-sh/bun#21467]: https://github.com/oven-sh/bun/issues/21467
[`enctype` and `formenctype`]: https://html.spec.whatwg.org/#attr-fs-enctype
[Submit as entity body]: https://html.spec.whatwg.org/#submit-body
[serialize an Infra value to JSON bytes]: https://infra.spec.whatwg.org/#serialize-an-infra-value-to-json-bytes
[`multipart/form-data` encoding algorithm]: https://html.spec.whatwg.org/#multipart/form-data-encoding-algorithm
[whatwg/html#3577]: https://github.com/whatwg/html/issues/3577
[CVE-2022-41919]: https://nvd.nist.gov/vuln/detail/CVE-2022-41919
[RFC 6839]: https://www.rfc-editor.org/rfc/rfc6839
[whatwg/html#11347]: https://github.com/whatwg/html/pull/11347
[whatwg/fetch#1785]: https://github.com/whatwg/fetch/pull/1785
[RFC 10008]: https://www.rfc-editor.org/rfc/rfc10008
[whatwg/html#12594]: https://github.com/whatwg/html/issues/12594
[caveats]: #caveats-shortcomings-and-other-drawbacks-of-design-choices
[open questions]: #open-questions
[FormValidation open questions]: form-validation.md#open-questions
[PHP]: https://www.php.net/manual/en/language.variables.external.php
[Rack]: https://github.com/rack/rack/blob/main/lib/rack/query_parser.rb
[qs]: https://github.com/ljharb/qs
[Create an entry]: https://xhr.spec.whatwg.org/#create-an-entry
[converting an Infra value to a JSON-compatible JavaScript value]: https://infra.spec.whatwg.org/#convert-an-infra-value-to-a-json-compatible-javascript-value
[constructing the entry list]: https://html.spec.whatwg.org/#constructing-the-form-data-set
[rules for parsing floating-point number values]: https://html.spec.whatwg.org/#rules-for-parsing-floating-point-number-values
[`setFormValue()`]: https://html.spec.whatwg.org/#dom-elementinternals-setformvalue
[entry construction algorithm]: https://html.spec.whatwg.org/#face-entry-construction
[parsing a JSON string to an Infra value]: https://infra.spec.whatwg.org/#parse-a-json-string-to-an-infra-value
[serializing a JavaScript value to a JSON string]: https://infra.spec.whatwg.org/#serialize-a-javascript-value-to-a-json-string
[Minimum Common Web Platform API]: https://min-common-api.proposal.wintertc.org/
