# JSON form encoding explainer

Author: [Keith Cirkel]

## Summary

Submit typed, nested JSON from an HTML form without replacing native submission
with script.

```html
<form method="post" action="/pets" enctype="application/json">
  <label>Name <input name="pet[name]" value="Hypatia" /></label>
  <label>Legs <input name="pet[legs]" type="number" value="4" /></label>
  <button>Save</button>
</form>
```

The request body is:

```json
{ "pet": { "name": "Hypatia", "legs": 4 } }
```

`FormData` exposes the same conversion through `data.toJSON()` and
`FormData.fromJSON(value)`. Use `enctype="multipart/json"` to include file
contents alongside the JSON. Conversion throws a `TypeError` when name conflicts
arise. Cross-origin submissions require server opt-in through CORS preflight.

## User problems to be solved

HTML forms support `application/x-www-form-urlencoded`, `multipart/form-data`
and `text/plain`, but many APIs expect typed, nested JSON described by OpenAPI
or JSON Schema.

Authors must intercept `submit`, convert `FormData` to an object, serialise it
and send it with `fetch()`. This requires script and replaces native navigation,
history, response rendering and back/forward handling. Restoring those also
means handling `popstate`, page scripts, loading and cancellation UI, scroll
and focus. Sites separately reimplement conversion rules such as repeated names.

### Evidence of demand

`Object.fromEntries(formData)` keeps only the last value for each name, losing
repeated checkbox or `<select multiple>` values:

```html
<input type="checkbox" name="tag" value="magic" checked />
<input type="checkbox" name="tag" value="horns" checked />
```

```js
Object.fromEntries(new FormData(form)); // { tag: "horns" }
```

Preserving values requires a schema, `getAll()` or collecting repeated entries.
Framework approaches differ:

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

The same checkboxes produce different values and types:

| Approach                                                                        | Both ticked         | Only `magic` ticked |
| ------------------------------------------------------------------------------- | ------------------- | ------------------- |
| `Object.fromEntries()`                                                          | `"horns"`           | `"magic"`           |
| `FormData.get()`                                                                | `"magic"`           | `"magic"`           |
| htmx [`json-enc`], [Hono][hono-source] `{ all: true }`, Bun `toJSON()`          | `["magic","horns"]` | `"magic"`           |
| `getAll()` with schema ([superforms][sveltekit-superforms], Astro array fields) | `["magic","horns"]` | `["magic"]`         |

Hono, like PHP, Rails and qs, always returns an array for a name ending in
`[]`.

## Methodology for approaching & evaluating solutions

- Support script-free JSON submission, following the [Declarative Web].
- Preserve CSRF protection based on cross-origin forms being unable to send
  `Content-Type: application/json`; require explicit server opt-in.
- Reuse `enctype`/`formenctype`, form entries, `FormData` and CORS.
- Follow server naming conventions for compatible fallback.
- Share one JSON representation between script and declarative submission.
- Fail loudly on ambiguous or conflicting names, rather than guess a shape.
- Expose conversion first, then build declarative submission on it, following
  the [Extensible Web Manifesto].

## Prior features and proposals

- [W3C HTML JSON form submission] (Working Group Note, 2015) proposed
  `enctype="application/json"`, bracket paths such as `pet[0][species]` and
  base64 files. It was not pursued. Its algorithm pads sparse arrays with
  `null` (`kids[999999999]` creates a billion entries), merges conflicts using
  `""` and index keys, and produces strings despite typed examples. It also
  forbids cross-origin requests even with server consent.
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
- Labels must be dense, meaning every array at every nesting level has labels
  exactly `0` to `n - 1`, so `kids[1]=b` and `kids[0]=a` give
  `{"kids":["a","b"]}`. A control that gives `null` still occupies its
  position. A missing position (`kids[0]` with `kids[2]`) or a lone large label
  such as `kids[99999]` throws a `TypeError`. Gaps are never closed or padded,
  and labels are never renumbered (see
  [Caveats](#caveats-shortcomings-and-other-drawbacks-of-design-choices)).
- Other bracket segments are object keys. After the first key, a bare segment
  of only ASCII digits must be an index, so `[01]` throws; write `["01"]`.
- Any key, the first included, can also be written as a JSON string in
  brackets, with any valid JSON escapes. This is needed for an empty key, a key
  containing `[`, `]` or `"`, or a nested key of only ASCII digits, so
  `pet["0"]` and `pet["01"]` become `{"pet":{"0":"..."}}` and
  `{"pet":{"01":"..."}}` respectively. Otherwise quoting is optional, since a
  quoted and a bare spelling that decode to the same key are the same path. So
  `pet[name]`, `pet["name"]` and `pet["n\u0061me"]` all address key `name` of
  `pet`, and `["title"]` is `title`. Invalid JSON string contents throw.
- Repeated names and conflicts are detected on parsed paths, not spellings, so
  controls named `pet[name]` and `pet["name"]` are a repeated name, and throw.
- `[]` appends a value and is allowed only at the end of a name. `p[][a]`
  throws, as it doesn't say which object owns `a`. Use `p[0][a]`, `p[1][a]` and
  so on for separate objects.
- Any other name, such as `error[bad`, `[x]`, `a]`, `a[b]c`, `a["b"]c` or
  `a[]b`, throws.
- Conflicting shapes throw: a value with something nested under it (`mix` with
  `mix[key]`, or `a` with `a[]` or `a[0]`), or a container addressed by more
  than one of keys, labels and `[]` (`t[0]` with `t[a]`, `t[]` with `t[0]`).
- A path not ending in `[]` takes at most one value, so a repeated plain `tag`
  throws. A name ending in `[]` is always an array even with one value.
  `<select multiple>` must use `[]`.
- A name can have at most a limited number of key and index segments (see
  [open question 1][open questions]); a final `[]` doesn't count. Longer names
  throw.

For a form's entry list, values are counted per control, not per submitted
value, so whether conversion throws depends only on the form's controls and
submitter, not on what the user chose:

- Each named control outside a `datalist` gives exactly one value per name it
  submits under: its value, or a JSON-only empty state (see [Empty
  states](#empty-states)).
- A radio group gives one value: the checked radio's, or `null`.
- A button gives a value only when it is the submitter, so buttons can share a
  name.
- A `<select multiple>` or `<input type=file multiple>` whose name doesn't end
  in `[]` throws, however many items are selected.
- Two other controls sharing a name that doesn't end in `[]` throw, whatever
  their state, so Rails' hidden input before a same-named checkbox throws, as
  do same-named alternatives toggled with `disabled`.

Entries added by script, through `append()`, `set()`, the `formdata` event or a
`FormData` passed to `setFormValue()`, count too, so for them the check depends
on the values.

Canonicalisation rewrites non-canonical names. Quoted keys are unquoted where
possible, and arrays containing only JSON leaves (including empty objects and
arrays) use `[]`. Arrays containing a non-empty object or array use dense
labels starting at `[0]` for every element. Labels are canonicalised at each
nesting level, so `kids[1], kids[0]` become `kids[], kids[]` in label order.
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
  enabled, unchecked one gives JSON `false`, without adding an entry. Empty
  declared collections give `[]`, and other controls without a value give
  `null`, as below.
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

A control that submits no value still influences JSON:

- An enabled, unchecked checkbox without a `value` attribute gives `false`.
- A name ending in `[]` declares a collection. Its checked checkboxes or
  radios, selected enabled options and selected files are its values; with
  none it gives `[]`, so `name="tag[]"` gives `{"tag":[]}`, not
  `{"tag":[[]]}`. Disabled or unchecked members add nothing.
- Any other control without a value gives `null`: an unchecked checkbox with a
  `value` attribute, a radio group with nothing checked, a disabled control
  (including through a fieldset) and a custom element submitting `null`.

`null` means "no value", as it does for an empty number input. Unnamed
controls, controls inside a `datalist` and buttons other than the submitter
give nothing. A selected enabled option with `value=""` is an actual value,
not an empty collection.

These states are captured with the entry list, before `formdata`, and follow
actual entries in JSON insertion order. `get()`, `getAll()`, `has()`,
iteration and existing encodings do not expose them. Empty multiple-file
inputs with valid terminal `[]` names keep their native empty `File`
placeholder in that public view, but omit it from JSON and `multipart/json`
parts.

`append()`, `set()` and `delete()` clear empty states with the exact source
name they receive, after successful argument conversion. This applies to both
string and Blob overloads. `delete("tag[]")` removes the implicit empty
collection even though `has("tag[]")` was false. Mutations do not rescan the
form.

```html
<input type="checkbox" name="draft" />
<input type="checkbox" name="tag[]" value="magic" />
<select name="categories[]" multiple></select>
<input type="radio" name="size" value="s" />
<input type="radio" name="size" value="l" />
```

With no selections, the JSON is
`{"draft":false,"tag":[],"categories":[],"size":null}`. The visible entry list
is empty.

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
- Omitting the hint, or passing `"string"`, keeps the string, so
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
  `setFormValue()` still submits no visible entry, regardless of the hint, and
  gives JSON `null`.

```js
const data = new FormData();
data.set("quantity", "42", "number");
data.set("enabled", "", "boolean");
data.set("draft", "on", "boolean");
data.toJSON(); // { quantity: 42, enabled: false, draft: true }
```

A custom checkbox that submits `null` when unchecked gives JSON `null`, as a
native checkbox with a `value` attribute does:

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
`{"home":{"lat":51.5,"lng":-0.12}}`. Its urlencoded body stays
`home%5Blat%5D=51.5&home%5Blng%5D=-0.12`.

`JSON.stringify(new FormData(form))` gives a `fetch()` body that keeps every
value. A `FormData` nested in a larger object passed to `JSON.stringify()`
serialises too, the same use case that settled
[`URL.prototype.toJSON()`][whatwg/url#137]. If conversion fails, for example
on a gap or a conflict, either call throws a `TypeError`, including when the
`FormData` is nested (see [caveats] for how that affects generic serialisers).
`toJSON()` sends no requests, so it needs no CSRF analysis and doesn't depend
on any Fetch change.

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
- Typed values are stored as strings with a type hint. A number uses its
  JavaScript string conversion and the `"number"` hint; a boolean uses `""` for
  `false` and `"true"` for `true`, with the `"boolean"` hint. Applying the
  hints gives the original JSON values back. `null` and empty arrays and
  objects become entries whose string is `""` and whose JSON value is `null`,
  `[]` or `{}`. An empty array is a leaf, so `{"tag":[]}` gives one visible
  entry named `tag`, while `{"tag":["a"]}` gives `tag[]`. These entries are
  public, unlike the JSON-only empty states they may have come from, so getters
  and legacy encodings see `tag=""`. No single exact-name validation rule
  addresses both spellings; see the FormValidation open question [One rule set
  for both representations][FormValidation open questions].
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
  question 9][open questions]).

Examples and their canonical names:

| Entry names, in order                                           | `toJSON()`                                                  | Canonical names                                |
| --------------------------------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------- |
| `kids[1]=b`, `kids[0]=a`, `kids[2]=c`                           | `{"kids":["a","b","c"]}`                                    | `kids[]`, `kids[]`, `kids[]`                   |
| `pet[1][name]=Billie`, `pet[0][name]=Hypatia`, `pet[0][legs]=4` | `{"pet":[{"name":"Hypatia","legs":"4"},{"name":"Billie"}]}` | `pet[0][name]`, `pet[0][legs]`, `pet[1][name]` |
| `grid[1][0]=c`, `grid[0][1]=a`, `grid[0][0]=b`                  | `{"grid":[["b","a"],["c"]]}`                                | `grid[0][]`, `grid[0][]`, `grid[1][]`          |
| `items[1][name]=Hypatia`, `items[0]=loose`                      | `{"items":["loose",{"name":"Hypatia"}]}`                    | `items[0]`, `items[1][name]`                   |
| `pet["1"][name]=Hypatia`, `pet["01"]=x`                         | `{"pet":{"1":{"name":"Hypatia"},"01":"x"}}`                 | `pet["1"][name]`, `pet["01"]`                  |
| `0=a`, `1["2"]=b`                                               | `{"0":"a","1":{"2":"b"}}`                                   | `0`, `1["2"]`                                  |
| `q[""]=e`, `q["a]b"]=f`                                         | `{"q":{"":"e","a]b":"f"}}`                                  | `q[""]`, `q["a]b"]`                            |
| `__proto__[polluted]=yes`                                       | `{"__proto__":{"polluted":"yes"}}`, an own property         | `__proto__[polluted]`                          |
| `kids[0]=a`, `kids[2]=c`                                        | throws a `TypeError`: position 1 is missing                 | none                                           |
| `kids[99999]=c`                                                 | throws a `TypeError`: positions 0 to 99998 are missing      | none                                           |
| `pet[0][tag][1]=x`                                              | throws a `TypeError`: the nested array lacks position 0     | none                                           |
| `tag=a`, `tag=b`                                                | throws a `TypeError`: `tag` gets two values; use `tag[]`    | none                                           |
| `mix=s`, `mix[key]=k`                                           | throws a `TypeError`: `mix` is both a value and an object   | none                                           |
| `t[0]=x`, `t[a]=y`                                              | throws a `TypeError`: `t` has both an index and a key       | none                                           |
| `t[]=x`, `t[0]=y`                                               | throws a `TypeError`: `t` has both `[]` and an index        | none                                           |
| `error[bad=1`, `[x]=2` or `a[]b=3`                              | throws a `TypeError`: not a valid name                      | none                                           |
| `kids[01]=x`                                                    | throws a `TypeError`: a nested digit key must be quoted     | none                                           |
| `pet["name"]=x`, `pet["legs"]=4`                                | `{"pet":{"name":"x","legs":"4"}}`                           | `pet[name]`, `pet[legs]`                       |
| `["title"]=Unicorns`, `q["\u0041"]=a`                           | `{"title":"Unicorns","q":{"A":"a"}}`                        | `title`, `q[A]`                                |
| `pet[name]=x`, `pet["name"]=y`                                  | throws a `TypeError`: both name the same path; use `[]`     | none                                           |

### Alternatives considered

- **Flat, always arrays.** Types never vary, even for plain repeated names, but
  there is no nesting and no typing, and every single value is wrapped, as in
  `{"title":["Unicorns"]}`.
- **Arrays chosen by control.** Inferring arrays from control types or shared
  names can conflict with the server's expected shape. `[]` makes the author's
  choice explicit in the submitted name.
- **Collecting repeated plain names into an array**, as qs, htmx and the Note
  do. No value is lost, but the field's type depends on how many values were
  submitted: one ticked box gives a string, two give an array. Throwing makes
  the author choose `[]`.
- **Last value wins for repeated plain names**, as in PHP and Rack. This matches
  more servers but silently drops data, a bug which this proposal aims to fix.
- **Dot syntax** (`pet.0.name` or `pet[0].name`), as in decode-formdata,
  Conform, ASP.NET Core and Spring. A browser without support falls back to
  urlencoded, which PHP, Rack and qs decode into the same shape only with
  bracket syntax, and PHP rewrites `.` in top-level names to `_`.
- **JavaScript-style quoted keys** (`pet["name"]`, with bare brackets only for
  indices). Unambiguous, but in the urlencoded fallback PHP, Rack and qs keep
  the quotes as part of the key, so bare keys stay the ordinary spelling and
  quotes are accepted as an alias.
- **Flattening conflicts**, keeping contradictory names such as `mix` and
  `mix[key]`, and names that aren't valid paths, whole as flat keys. No value
  is lost and the result doesn't depend on entry order, but a stray name
  silently changes the shape of another field, and a flattened `tag[]` loses
  its array type. Conflicts throw instead.
- **Sparse positional indices**, as in the 2015 Note. Filling gaps with `null`
  makes an array's size depend on its largest label, and the padding can't be
  told apart from a control's `null`.
- **Compacting labels**: sort them and close gaps. Output stays proportional to
  the entries and removing a row needs no renumbering, but a value can move to
  a position with different rules. If `surname[0]` is required and `surname[1]`
  optional, a lone `surname[1]` would land in the required slot.
- **Dense or object**, where an array level whose labels aren't exactly `0` to
  `n - 1` becomes an object with numeric keys. Omissions survive, but validation
  would need to treat numeric indices and their quoted object-key spellings as
  the same address, and the field would be an array or an object depending on
  what was submitted. Framework feedback preferred throwing.
- **Dense array indices that stay flat** when not dense. Positions are
  preserved, but the whole group silently changes shape. The draft throws
  instead.
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
   for example on a conflict, a repeated name or a missing array position, no
   request is sent. How that failure is reported is not settled; see [open
   question 6][open questions].
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
     order), not necessarily entry-list order. Its field name is an
     [RFC 6901] JSON Pointer to its file name in the JSON, such as `/photo`,
     `/photo/1` or `/pet/0/photo`, so each part identifies exactly one
     location, even in an array mixing strings and files.
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
media-type support would not provide incremental access. The receiver API and
upfront metadata remain design questions.

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
Line two</textarea>
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
Content-Disposition: form-data; name="/photo"; filename="unicorn.png"
Content-Type: image/png

<file bytes>
------b--
```

## Caveats, shortcomings, and other drawbacks of design choices

- **`application/json` can't upload files.** File entries appear only as file
  names. Uploads need `multipart/json` (or `multipart/form-data`). Base64 in
  the JSON, as the 2015 Note did, would make file data about a third larger,
  and a synchronous `toJSON()` can't read file contents, so including them
  would give up the shared representation.
- **`multipart/json` is a new media type.** It needs an IANA registration, as
  the multipart registry has no `json` subtype. Alternatives considered:
  - Sending `multipart/form-data` would avoid the preflight, but the `enctype`
    keyword would then name a type that is never sent, unlike every existing
    keyword. It would buy little compatibility either. An existing parser would
    accept the body, but the app would see its files and an empty-named field,
    and none of its form fields, until it was changed to read the JSON part.
  - `multipart/form-data+json` would mislead, because the `+json` suffix
    ([RFC 6839]) tells servers the whole body is JSON, which it isn't.

  `multipart/json` must also never be added to the CORS safelist. Any page
  could then send it with an arbitrary body via `fetch()` with
  `mode: "no-cors"`, and servers that match content types loosely would parse
  attacker JSON without a preflight, the class of bug behind Fastify's
  [CVE-2022-41919].

- **Existing parsers reject `multipart/json`.** busboy, which multer uses,
  accepts only `multipart/form-data` and throws `Unsupported content type`
  otherwise. This fails loudly rather than silently losing fields, and every
  server has to change anyway to read the JSON part, either by updating the
  parser or by treating `multipart/json` as `multipart/form-data` before
  parsing, since the part syntax is the same.
- **Servers find the JSON part by position.** It is always the first part and
  has an empty field name. Script and form-associated custom elements can also
  add empty-named entries. Their strings appear inside the JSON object, while
  files appear as file names in the JSON and as subsequent file parts.
  Parsers reusing `multipart/form-data` code may drop or mishandle an
  empty-named part, which needs surveying. OpenAPI's Encoding Object can
  describe a multipart part typed `application/json`, but parts are keyed by
  name, so describing this one needs a `""` property. How schema tooling handles
  that also needs checking.
- **File parts are named by JSON Pointer.** A file from `pet[0][photo]` arrives
  in a part named `/pet/0/photo`, so lookup by control name misses it.
- **Names gain meaning.** Only the new methods and keywords parse brackets;
  other `FormData` methods and encodings are unchanged. PHP, Rails and qs use
  similar paths, but incidental brackets can now nest, and names that aren't
  valid paths make conversion throw. Quoted keys such as `q["a]b"]` are new.
  Any key may be quoted, so `pet["name"]` is `pet[name]` to conversion, but
  `get()`, `form.elements`, exact-name validation rules and the urlencoded
  fallback, where PHP and qs keep the quotes, see the spelling as written.
- **Dense array positions.** Every array level needs labels `0` to `n - 1`;
  gaps throw `TypeError` and abort JSON submission before sending. Compaction
  could move an optional `surname[1]` into required `surname[0]`; padding would
  conflate gaps with numeric `null`. Store persistent IDs in values or use
  quoted keys (`pet["7"]`) for sparse objects.
- **Removed rows leave gaps.** Unanswered radios, unchecked checkboxes,
  disabled controls and custom elements submitting `null` keep their positions
  with `null` or `false`, so a gap needs a missing control, such as a row
  removed by script. Renumbering is safe only where positions have no distinct
  meaning, and the browser never does it.
- **Generic serialisers can throw.** `JSON.stringify()` calls nested
  `FormData.toJSON()` too, so a gap, a conflict or a repeated name can break
  logging, error reporting or persistence that never chose this encoding.
  `JSON.stringify()` already throws for BigInt values and cycles, and today
  `JSON.stringify(formData)` returns `{}`, so its output is unlikely to be
  relied on. Breakage needs incidental stringifying of a non-empty `FormData`
  whose names fail these rules. Before shipping, a use counter for
  `JSON.stringify()` reaching a non-empty `FormData` would measure that. A
  non-throwing `toJSON()` would split the representation shared with
  submission.
- **Numeric brackets differ from Rack.** `pet[1][name]` builds an array here,
  but an object in Rack. PHP and qs also interpret digit brackets as arrays,
  though their handling of sparse and very large labels differs. Quoted keys
  are new syntax, so numeric object keys need server support in the urlencoded
  fallback.
- **Repeated names throw.** A name not ending in `[]` takes one value, so two
  controls named `tag`, or a `<select multiple name="tag">`, make conversion
  throw whatever is selected, though qs, htmx and PHP accept them. Rails'
  hidden input before a same-named checkbox throws too, as do same-named
  alternatives where script enables one at a time, such as a country `<select>`
  and an "Other" text field. Use `[]` for collections, or distinct names. For a
  form not modified by script the error is visible in the markup, so
  conformance checkers can flag it.
- **Hints are control-derived.** Hidden inputs and selects remain strings.
  There is no markup hint attribute; script and custom elements can supply
  hints. Servers may still need coercion for numeric IDs or booleans.
- **Numbers have double precision.** As with `valueAsNumber`,
  `<input type=number value=12345678901234567890>` gives `12345678901234567000`,
  so number inputs shouldn't hold IDs that require exact decimal digits.
- **Empty states need controls.** Enabled, unchecked checkboxes without a
  `value` attribute supply `false`, empty declared collections supply `[]`, and
  other controls without a value supply `null`. Absent controls supply nothing.
  Legacy encodings omit all of these, so servers must handle the difference.
- **Disabled controls give `null`.** Disabled means "not submitted" in legacy
  encodings, but in JSON a disabled control gives `null`, or nothing inside a
  `[]` collection, so the shape of the JSON depends only on the form's
  controls. Disabling a field no longer removes its key, a server can't tell a
  disabled field from an empty one, and same-named alternatives can't be
  switched with `disabled` ([open question 2][open questions]).
- **Script cannot create JSON-only empty states.** Only HTML entry construction
  creates them, though `setFormValue()` preserves existing ones in `FormData`.
  Script can supply visible `false` as `""` with a `"boolean"` hint, sent by
  legacy encodings, or visible `[]` through `FormData.fromJSON({tags: []})`,
  which creates `tags=""`. An API remains [open question 3][open questions].
- **Types can change contracts.** Changing a control's type or a script hint
  changes the JSON type.
- **Read/write copies lose JSON information.** Non-file getters and iteration
  expose strings, omitting hints and implicit empty states.
  `data.set(name, data.get(name))` also clears that name's contribution.
  Reapply hints or use `FormData.fromJSON(data.toJSON())` for dense labels.
  That round trip loses file contents and canonicalises names. It
  materialises `draft: false` as `draft=""` and an empty `tag[]` group as
  `tag=""` with JSON value `[]`; deletion must use those canonical names.
- **Nesting needs a limit.** Deep names cost browser and server recursion.
  qs defaults to 5 levels; Rack errors at 32. Over-limit names throw, as does
  `fromJSON()` for deeper values. A terminal `[]` is not counted, so a
  collection at the limit can still append.
- **Schema compatibility.** Typed, nested output does not match every OpenAPI
  or JSON Schema body. Schemas must allow omissions and `null` for empty,
  unanswered and disabled fields. Strings may need coercion, such as Ajv's
  `coerceTypes` [option], enabled by Fastify for all request parts
  ([`default-ajv-options.js`]).
- **Order across names is lost.** Values remain ordered within a name.
  Use `multipart/form-data` when full interleaving matters.
- **Newlines stay unchanged.** Legacy encodings convert them to CRLF; JSON
  preserves DOM line feeds, matching `JSON.stringify(new FormData(form))`.
- **Fallback loses types and uploads.** Unsupported keywords fall back to
  `application/x-www-form-urlencoded`. Servers must handle strings, parser
  differences in numeric or quoted keys, and the repeats and conflicts the
  browser would have rejected. `multipart/json` fallback sends filenames only.
  For older-browser uploads, use `multipart/form-data` as the baseline and
  feature-detect the new encoding.
- **Cross-origin needs server opt-in.** Cross-origin JSON submissions work only
  if the server answers the preflight with credentialed CORS headers.
  Compared with the other encodings:

  | `Content-Type`                                        | `<form>`          | `fetch()`, `no-cors`      | `fetch()`, `cors` |
  | ----------------------------------------------------- | ----------------- | ------------------------- | ----------------- |
  | urlencoded, `multipart/form-data`, `text/plain`       | sent              | sent                      | sent              |
  | `application/json`                                    | never             | never (header is dropped) | after a preflight |
  | `application/json` or `multipart/json`, with proposal | after a preflight | never                     | after a preflight |

  A 307/308 redirect keeps the body and is preflighted again against the new
  origin. A 303 redirect drops the body and its `Content-Type`, so the common
  POST/redirect/GET pattern works without a preflight for the GET.

- **Other request headers in the preflight.** Any other non-safelisted headers
  the browser adds to navigation requests before the preflight (for example
  `Upgrade-Insecure-Requests`, depending on where that spec hooks in) would also
  appear in `Access-Control-Request-Headers`, and servers would need to allow
  them. This, and whether servers need a signal that a preflight is for a
  navigation (for example `Sec-Fetch-Dest`), are questions for
  [whatwg/fetch#1785], and apply equally to PUT, PATCH and DELETE forms.
- **`toJSON`-detecting libraries.** Old [ofetch] (Nuxt's `$fetch`) sent Bun
  `FormData` as JSON because it had `toJSON`. [unjs/ofetch#483] fixed detection
  order in Aug 2025. The same cause was alleged for node-fetch and axios upload
  failures in [oven-sh/bun#21467], but remains unconfirmed. A main-branch audit
  found no other affected library: axios, redaxios, wretch and Angular check
  `FormData` first; ky, got, hono/client and tRPC require explicit JSON; native
  fetch, node-fetch and undici never call `toJSON()`. Wrappers that do can also
  encounter conversion errors, even with nested `FormData`. Measuring sites
  still bundling old ofetch needs browser telemetry or a staged rollout.

## Open questions

In the order they would block shipping, with who is best placed to answer.
Questions 1 to 3 can change what `toJSON()` returns, so they must be settled
before it ships.

1. **Nesting limit** (stage 1; browser vendors and server authors): how many
   key and index segments can a name have before conversion and `fromJSON()`
   throw? A final `[]` is not counted.
2. **Disabled controls and fallback keys** (stage 1; framework, server and
   browser authors): is `null` for disabled controls the right tradeoff,
   keeping the JSON shape a static property of the form although disabled
   means "not submitted" in legacy encodings, and same-named alternatives
   toggled with `disabled` throw? Can servers support quoted numeric object
   keys in the urlencoded fallback?
3. **Empty states from script** (stage 1; framework and custom element
   authors): script and custom elements can't create JSON-only empty states,
   and `fromJSON()` turns `[]` into a visible entry such as `tags=""`. Is an
   API for script-created empty states needed, and what shape should it take?
   None is proposed.
4. **`toJSON` detection** (stage 1; site owners, and browsers through
   telemetry): how many sites still bundle ofetch from before Aug 2025, which
   sends any body with a `toJSON` method as JSON?
5. **Preflight details** (stage 1; browser vendors, in [whatwg/fetch#1785]): are
   browser-added headers such as `Upgrade-Insecure-Requests` excluded from
   `Access-Control-Request-Headers`, and do servers need a signal that a
   preflight is for a navigation?
6. **Conversion failure feedback** (stage 2; browser vendors and accessibility
   experts): a submission whose conversion throws sends no request. For a form
   not modified by script, every conversion error is a static property of its
   controls and submitter, so it can be detected before the user submits.
   These are author bugs, so we recommend a console diagnostic and a
   form-level event like `invalid` before stage 2 ships, not user-facing
   validation UI. A pseudo-class matching forms that can't convert is an option
   we want feedback on. This draft defines none of these, and their names,
   timing and details are undecided.
7. **Is `multipart/json` worth a new media type?** (stage 2; server authors): is
   a typed, nested JSON part worth a registration and parser changes, over
   rebuilding structure from `multipart/form-data` field names?
8. **Parser support** (stage 2; multipart parser authors): how quickly would
   multipart parsers accept `multipart/json`, and do they and schema tooling
   handle the empty-named JSON part and JSON Pointer part names?
9. **`formData()` on JSON bodies** (stage 2; server runtime authors): should
   `Request` and `Response`'s `formData()` accept `application/json` through
   `fromJSON()`, and reconstruct `multipart/json` by putting each file part at
   the location its JSON Pointer names?
10. **Incremental multipart consumption** (stage 2; server runtime and parser
    authors): expose the JSON part before file bodies, then bounded file
    streams with cancellation and backpressure. Define upfront file metadata
    before relying on declared-size rejection or alternative file scheduling.

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
initially empty. Each has a source name and a JSON value of false, null, an
empty list, or _multiple_. They are not entries, so existing accessors,
iteration and legacy encoders continue to use only the ordinary entry list.
Entry-list cloning preserves the contributions and native no-selection
provenance.

The **active empty states** of an entry list are its contributions, except an
empty-list contribution whose source name is also the name of an ordinary entry
not marked for JSON omission.

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

A **form path** is a list of strings (keys) and non-negative integers (indices)
that may end with _append_. Its **nesting depth** is its size, not counting a
final _append_. The nesting limit (see [open question 1][open questions])
bounds the nesting depth both of parsed names and of canonical names.

A string _key_ **needs quoting**, given a boolean _first_, if it is the empty
string, contains U+005B ([), U+005D (]), U+0022 (") or a surrogate, or _first_
is false and _key_ consists only of ASCII digits. This governs only
serialising canonical names; parsing accepts any quoted key.

**Parse a quoted form key**, given a string _name_ and a position variable
_position_ pointing at U+0022 ("):

1. If no prefix of the code points from _position_ matches the JSON `string`
   production, then return failure.
2. Let _literal_ be the shortest such prefix, and advance _position_ past it.
3. Return the result of [parsing a JSON string to an Infra value] given
   _literal_.

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
   2. If _key_ is the empty string or contains U+005D (]) or U+0022 ("), then
      return failure.
   3. Append _key_ to _path_.
5. While _position_ does not point past the end of _name_:
   1. If _path_ ends with _append_, or the code point at _position_ is not
      U+005B ([), then return failure.
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
   7. If _key_ is the empty string, then append _append_ to _path_.
   8. Otherwise, if _key_ matches `0` or a non-zero ASCII digit followed by zero
      or more ASCII digits, append its non-negative integer value to _path_.
      This is a position, not an allocation size.
   9. Otherwise, if _key_ needs quoting given false, then return failure.
   10. Otherwise, append _key_ to _path_.
6. If _path_'s nesting depth is greater than the nesting limit, then return
   failure.
7. Return _path_.

An **item** has a form _path_, a _value_, an optional _entry_ reference, an
_empty_ flag, and a **JSON location**, a list of strings and non-negative
integers. A contribution item has no entry reference.

**Build a form JSON node**, given a list of items _items_, an integer _depth_,
and a list _orderedEntries_:

1. If every _item_ of _items_ has a path of size _depth_:
   1. If _items_ has more than one item, then throw a `TypeError`.
   2. Append _items_[0]'s entry reference, if any, to _orderedEntries_, and
      return its value.
2. If every _item_ of _items_ has a path of size _depth_ + 1 that ends with
   _append_:
   1. Let _list_ be a new list.
   2. For each _item_ of _items_ whose empty flag is false, in order, append
      _list_'s size to _item_'s JSON location, append _item_'s value to _list_,
      and append _item_'s entry reference, if any, to _orderedEntries_.
   3. Return _list_.
3. If every _item_ of _items_ has a path of size greater than _depth_ whose
   item at _depth_ is a string:
   1. Let _map_ be a new ordered map.
   2. For each _key_ → _group_ of _items_ grouped by path[_depth_], in
      first-seen order:
      1. Append _key_ to the JSON location of each item of _group_.
      2. Set _map_[_key_] to the result of ? building a form JSON node given
         _group_, _depth_ + 1 and _orderedEntries_.
   3. Return _map_.
4. If every _item_ of _items_ has a path of size greater than _depth_ whose
   item at _depth_ is a non-negative integer:
   1. Let _list_ be a new list.
   2. For each _label_ → _group_ of _items_ grouped by path[_depth_], in
      ascending numeric label order:
      1. Append _label_ to the JSON location of each item of _group_.
      2. Append the result of ? building a form JSON node given _group_,
         _depth_ + 1 and _orderedEntries_ to _list_.
   3. Return _list_.
5. Throw a `TypeError`.

Step 1 rejects a repeated path, and step 5 a leaf with something nested under
it, or a container addressed by more than one of keys, indices and _append_.

**Convert an entry list to a JSON value**, given an entry list _entryList_:

1. Let _items_ be « ».
2. For each _entry_ of _entryList_ not marked for JSON omission:
   1. Let _value_ be _entry_'s JSON value if it has one, otherwise _entry_'s
      value's `name` if _entry_'s value is a `File` object, otherwise _entry_'s
      value.
   2. Let _path_ be the result of parsing a form entry name given _entry_'s
      name. If _path_ is failure, then throw a `TypeError`.
   3. Append a new item whose path is _path_, whose value is _value_, whose
      entry is _entry_, and whose empty flag is false to _items_.
3. For each _contribution_ of _entryList_'s active empty states:
   1. If _contribution_'s JSON value is _multiple_, then throw a `TypeError`.
   2. Let _path_ be the result of parsing a form entry name given
      _contribution_'s source name. If _path_ is failure, then throw a
      `TypeError`.
   3. Append a new item whose path is _path_, whose value is _contribution_'s
      JSON value, and whose empty flag is true if that value is an empty list,
      or false otherwise, to _items_.
4. Run ? check array density given _items_.
5. Let _result_ be a new ordered map, and let _orderedEntries_ be a new list.
6. For each _key_ → _group_ of _items_ grouped by path[0], in first-seen order:
   1. Set the JSON location of each item of _group_ to « _key_ ».
   2. Set _result_[_key_] to the result of ? building a form JSON node given
      _group_, 1 and _orderedEntries_.
7. For each _entry_ of _orderedEntries_ at index _index_, set _entry_'s **JSON
   order** to _index_.
8. Return _result_.

Each ordinary entry's **JSON location** is its item's JSON location when
conversion returns. Every item reaches the output exactly once, at its own
location, except an empty item, which only keeps its collection's `[]`.
Contribution items affect JSON output but never become file parts.

The **JSON Pointer** of a JSON location is the concatenation, for each of its
items, of `/` followed by the item's decimal representation if it is an
integer, or otherwise by the string with each `~` replaced by `~0` and then
each `/` replaced by `~1`, as in [RFC 6901].

**Serialise a form key**, given a string _key_ and a boolean _first_: if _key_
needs quoting given _first_, return the result of [serializing a JavaScript
value to a JSON string] given _key_; otherwise, return _key_. A bare digit key
such as a top-level `0` round-trips, while a later digit key is quoted to stay
an object key.

**Serialise a form path**, given a form path _path_: let _name_ be the result
of serialising the form key _path_[0] with _first_ true, wrapped in `[` and
`]` if it isn't _path_[0] itself. For each later item, append `[]` if it is
_append_; `[`, its decimal representation, and `]` if it is an integer; and
otherwise `[`, the result of serialising that string as a form key with
_first_ false, and `]`. Return _name_.

**Get the canonical entries**, given a map _value_: let _entries_ be « », run
the following steps given « _key_ » and _child_ for each _key_ → _child_ of
_value_, and return _entries_. Given a form path _path_ and a value _node_:

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

1. For each _item_ of _items_, and each index _i_ of _item_'s path where
   path[_i_] is a non-negative integer, record path[_i_] as a label of the
   prefix path[0] to path[_i_ - 1], comparing prefix segments including their
   types.
2. For each prefix with recorded labels, let _n_ be the number of distinct
   labels. If any of its labels is _n_ or greater, then throw a `TypeError`.

Distinct labels all below _n_ are exactly `0` to _n_ - 1. The check runs
before any indexed output list is built, and needs work proportional to the
total length of the names. Implementations can keep labels as digit strings,
and a label with more digits than _n_ fails without being converted to an
integer or sizing any allocation.

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
no two of them conflict or repeat except with a final `[]`, and array labels
are dense. Every value `toJSON()` returns is accepted, including a `[]`
collection at the nesting limit, because the final _append_ of its canonical
name doesn't count towards the nesting depth.

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
- During entry-list construction, before the `formdata` event, record
  contributions for each submittable element that has a non-empty name, is not
  a submit button or an `object` element, and has no `datalist` ancestor,
  including disabled elements (also through a fieldset) and unchecked
  checkboxes and radio buttons. Treat each radio button group as one element.
  For each name the element would submit under:
  - If it is a `select` element with `multiple`, or an `input` element in the
    File Upload state with `multiple`, and the name does not end with `[]`,
    record a _multiple_ contribution.
  - Otherwise, if it adds no entries under the name and the name ends with
    `[]`, record an empty-list contribution. For a multiple file input with no
    selected files, mark only its native empty `File` placeholder for omission
    from JSON conversion and `multipart/json` parts.
  - Otherwise, if it adds no entries under the name and is not a
    form-associated custom element whose submission value is a list of
    entries, record false for an enabled `input` element in the Checkbox state
    with no `value` attribute, or null otherwise.

  Every element thus adds exactly one entry or contribution per name, except
  a `[]` collection and a custom element submitting a list. Capture source
  names as the same scalar strings used by entry creation, and preserve
  contributions when cloning the completed entry list.

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
  value, before appending it. A null submission still produces no entry, only
  the contribution above, and a `File` is unchanged. For a list of
  entries, append them with their existing JSON values and no-selection
  provenance unchanged, and copy its empty-state contributions with their
  original source names. The element's hint does not apply to the list.

- Add the "`application/json`" keyword to the [`enctype` and `formenctype`]
  enumerated attributes. It maps to a new `application/json` state. The missing
  value default and invalid value default are unchanged.
- In [Submit as entity body], add a case for `application/json`:
  1. Let _map_ be the result of running convert an entry list to a JSON value
     given _entry list_. If this throws, end these steps, so no request is sent
     and nothing navigates. How the failure is reported is unresolved (open
     question 6), and it is not meant to stay silent.
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
     order, with the same value and, as its name, the JSON Pointer of that
     entry's JSON location.
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

Each stage lands with web-platform-tests. Stage 1 covers:

- Names and structure: nesting, arrays of objects, nested and mixed arrays,
  shared labels, dense labels in any entry order, quoted numeric object keys,
  bare top-level digit keys with quoted nested ones, the nesting limit with an
  uncounted final `[]` (a collection at the limit round-trips), `__proto__`
  names, and `TypeError`s for missing, large and nested missing labels,
  invalid names, invalid JSON strings in quoted keys, quoted and escaped
  aliases parsing to the same path as bare keys (and repeating it), digit-only
  nested keys such as `[01]`, leaf/nesting, index/key, append/index and
  append/key conflicts, repeated names without `[]`, and over-limit names.
- Values: typed controls, the type-hint arguments of `append()`, `set()` and
  `setFormValue()` keeping today's strings for `get()` and existing encodings,
  empty-string boolean conversion, invalid and overflowing numbers (such as
  `1e400`) giving null while keeping their strings, replacement clearing old
  JSON values, unchanged restoration-state handling, per-entry JSON values
  surviving FACE submission, and unhinted script-added values staying strings.
- `JSON.stringify()` of a `FormData`, and `fromJSON()` canonical names,
  `TypeError`s and round trip both ways.

The navigation preflight tests belong to [whatwg/fetch#1785]. Stage 2 covers
the body shape, `formenctype` precedence, feature detection, conversion
failures sending no request, and cross-origin preflight including redirects,
for both keywords. For `multipart/json` it also covers the `Content-Type`, part
order, the JSON part's headers, and file parts named by JSON Pointer in JSON
traversal order, including mixed string and file arrays and files with dense
labels out of numeric order.

Empty-state coverage includes unchecked scalar booleans, `null` for
explicit-valued checkboxes, unanswered radio groups, disabled controls
(directly and through a fieldset) and custom elements submitting null, empty
declared groups, disabled options, explicit empty option values, native file
placeholders versus script-added files, repeated-name errors decided per
control regardless of state (including a hidden input sharing a checkbox's name
and multiple selects without `[]`), exact-name mutation clearing, and unchanged
legacy getters and encodings.

Incremental receiver coverage must check early JSON availability and
cancellation without asserting that no file bytes were sent.

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
[RFC 6901]: https://www.rfc-editor.org/rfc/rfc6901
