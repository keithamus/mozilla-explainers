# FormValidation explainer

Author: [Keith Cirkel]

## Summary

Run HTML's form constraints against `FormData`, in the browser or on a server
without a DOM. Define rules as data or extract them with
`new FormValidation(form)`, then serialise them for reuse.

```js
const rules = new FormValidation({
  email: { type: "email", required: true },
  username: { required: true, pattern: "[a-z0-9_]+", maxLength: 20 },
});
const restored = new FormValidation(rules.toJSON());
const { valid, fields } = restored.validate(formData);
```

Results use HTML's validity flags. Rules match exact entry names; sharing rules
across JSON name conversion remains an [open question](#open-questions).

## User problems to be solved

HTML [constraint validation] enforces rules on form controls, not on data.
Servers must repeat those rules because clients can bypass validation, but
server runtimes have `FormData` without a DOM. General validators differ from
HTML on patterns, step bases, lengths and email syntax, so the two checks can
disagree.

Script-built, modified or restored `FormData` has the same problem. Authors
must create hidden controls to borrow their validity or reimplement the rules.

### Evidence of demand

Libraries already map between server schemas and HTML constraints, generating
attributes from a schema:

- **Conform** (`@conform-to/zod`, ~297k weekly downloads) ships
  [`getZodConstraint`], which "returns an object containing the validation
  attributes for each field by introspecting the zod schema". Its docs list the
  gaps: "The `i` (case-insensitive) flag is not supported. No `pattern` is
  generated for case-insensitive regexes or when the serialized regex is not a
  valid HTML `pattern` under the browser's `v` flag."
- **sveltekit-superforms** (~294k weekly) exposes [`$constraints`][superforms
  constraints], schema-derived attributes authors spread onto inputs to "use the
  built-in browser validation". It warns that for `date` fields "both the
  underlying data and the constraint needs to be in `yyyy-mm-dd` format", and
  recommends overriding the generated `min` by hand.
- **Zod** ships a second email regex, [`html5Email`][Zod regexes], described as
  "Equivalent to the HTML5 input[type=email] validation implemented by
  browsers", because its default `z.email()` accepts a different set of
  addresses. The copy comes from MDN rather than the HTML Standard, so it goes
  stale if HTML changes the grammar, which [whatwg/html#4562] proposes for
  internationalised addresses.
- **validator.js** (~29M weekly) [`isEmail`][validator isEmail] accepts quoted
  and UTF-8 local parts that HTML rejects.

Browser and server checks can disagree:

| Field                          | Value              | Browser | Typical server check                    |
| ------------------------------ | ------------------ | ------- | --------------------------------------- |
| `pattern="[0-9]+"`             | `12a`              | invalid | valid: JSON Schema `pattern`, Zod regex |
| `type=email`                   | `admin@localhost`  | valid   | invalid: Zod `z.email()`                |
| `type=number min=0.25 step=.5` | `1`                | invalid | valid: JSON Schema `multipleOf: 0.5`    |
| `<textarea maxlength=9>`       | `abcd`, LF, `efgh` | valid   | invalid: `z.string().max(9)`            |

- JSON Schema [`pattern`][JSON Schema pattern] notes that "regular expressions
  are not implicitly anchored". Zod's `.regex()` is also unanchored; HTML's
  [`pattern`] must match the whole value.
- Zod's email regex requires a dot and a two-letter top-level domain; HTML's
  [valid email address] doesn't.
- HTML's [step base][`step`] is `min` when present, allowing 0.25, 0.75, 1.25,
  etc. JSON Schema [`multipleOf`][JSON Schema multipleOf] counts from zero.
- [`textarea`] [`maxlength`] measures the API value, "normalized so that
  line breaks use U+000A LINE FEED (LF) characters". Legacy encodings
  [convert them to CRLF][name-value pairs], turning 9 code units into 10 in the
  example. [JSON form encoding] keeps LF.

Weekly download figures are from api.npmjs.org for 2026-09-22 to 2026-09-28.

## Methodology for approaching & evaluating solutions

- Share HTML's constraint algorithms, IDL names and [`ValidityState`] flags.
- Run wherever `FormData` runs, including Window, workers and WinterTC's
  [Minimum Common Web Platform API] runtimes, which already have `FormData`
  and `URL`.
- Serialise rules as JSON for use on servers, at build time and in pages.
- Add no constraints. Libraries can build messages, cross-field rules and
  adapters on this primitive, following the [Extensible Web Manifesto].
- Validate entries independently of focus, edit history or current controls.
- Match exact entry names; JSON canonicalisation must not silently retarget
  rules.

## Prior features and proposals

- **Constraint validation** needs elements. [`ValidityState`] is
  `[Exposed=Window]` with no constructor.
- **[`ElementInternals.setValidity()`]** accepts flags computed by a
  [form-associated custom element][form-associated custom elements].
- **JSON Schema and OpenAPI** use different semantics and model objects, not
  repeated entries and `File` values.
- **[Standard Schema]** lets tools call Zod, Valibot, ArkType and roughly 30
  other libraries through `~standard.validate()`, without defining rule semantics.
- **Schema-to-attribute helpers**, including Conform's [`getZodConstraint`] and
  superforms' [`$constraints`][superforms constraints], generate HTML attributes.
- **Server DOMs**, such as jsdom, can parse forms and call `checkValidity()`.

## Flaws or limitations in existing features/proposals

The existing API requires elements rather than received entries. Shipping a DOM
to the server is heavy. Element validity also depends on interaction:
[`maxlength`] and `minlength` apply only when the "value was last changed by a
user edit (as opposed to a change made by a script)", so prefilled values pass
even in the browser.

Schema-to-attribute helpers translate one way and lose information. Neither
they nor Standard Schema ensure that server and browser rules agree.

## Motivation for this explainer

Define HTML's constraints over entries so one rule set can drive markup and
server checks. Names must match the
[submitted representation](#entry-names-and-json-canonicalisation).

## Proposed solution

A new `FormValidation` interface, exposed in every global. It holds constraints
keyed by exact entry name, and checks a `FormData` against them. It does not
parse or canonicalise names.

### Constructing from a dictionary

The constructor takes a record from entry name to constraints. Every member is
optional, and each applies to the same types as the attribute of the same name:

| Member                     | Applies to                                    |
| -------------------------- | --------------------------------------------- |
| `type`                     | all; defaults to `"text"`                     |
| `required`                 | all except `range` and `color`                |
| `pattern`                  | text-like types                               |
| `minLength`, `maxLength`   | text-like types and `textarea`                |
| `min`, `max`, `step`       | `number`, `range` and the date and time types |
| `multiple`                 | `email` and `file`                            |

Text-like types are `text`, `search`, `tel`, `url`, `email` and `password`. The
date and time types are `date`, `month`, `week`, `time` and `datetime-local`.

`type` takes the value of the control's `type` IDL attribute, meaning the input
types that can be validated, plus `"textarea"`, `"select-one"` and
`"select-multiple"`, which are what `HTMLTextAreaElement.type` and
`HTMLSelectElement.type` return. Using IDL names means a rule can be spread
straight onto an `<input>` in JSX and most templating libraries, as Conform's
and superforms' generated objects already are.

`min`, `max` and `step` are strings in the attribute syntax for the type, so a
date rule reads `{ type: "date", min: "2026-01-01" }`. Numbers convert to
strings through WebIDL as usual.

Unlike markup, the dictionary rejects invalid rules. Unknown types,
inapplicable members and unparseable `min`, `max` or `step` throw `TypeError`.
A `pattern` that cannot compile with the `v` flag throws `SyntaxError`.

### Entry names and JSON canonicalisation

Rule keys address the representation being validated. A `FormData` constructed
from a form keeps the controls' original names. The [JSON form encoding]
proposal's `FormData.fromJSON()` instead produces canonical names:

| Original entries                         | Canonical names                            |
| ---------------------------------------- | ------------------------------------------ |
| `pet[0][name]` and `pet[1][name]`        | Unchanged                                  |
| `pet[0]["name"]`                         | `pet[0][name]`                             |
| `tag[0]`, containing a scalar            | `tag[]`                                    |
| Two entries named `tag`                   | Two entries named `tag[]`                  |
| `error[bad`, an invalid path              | `["error[bad"]`                            |
| `mix` alongside `mix[key]`                | `mix` and `["mix[key]"]`                   |
| `pet[5][name]`, with no other pet        | None: conversion throws a `TypeError`      |

Rules for reconstructed JSON must use those canonical names. A rule for
`tag[]` applies to every entry with that name, including a singleton array.
`[]` is part of the exact name, not a wildcard, so `pet[][name]` does not match
`pet[0][name]`. Quoted numeric object keys remain distinct from array indices.

Canonicalisation needs the complete structure. Array elements determine whether
names use `[]`, leaf arrays merge names, and unruled entries can cause conflicting
paths to stay flat. Gapped indices fail conversion. Neither constructor
canonicalises rule keys independently, which could retarget them.

With current exact-name matching, source-name rules check original entries and
canonical-name rules check `FormData.fromJSON()` entries. Sharing one rule set
across that round trip remains [unresolved](#open-questions). Quoted numeric
keys can represent sparse records as JSON objects rather than arrays.

### Constructing from a form

`new FormValidation(form)` reads an `HTMLFormElement` and builds the same
record. For each of the form's listed elements, in tree order:

1. Skip the element if it is [barred from constraint validation] (disabled,
   readonly, `type=hidden`, buttons, inside a `datalist`, and so on), has an
   empty name, or is a [form-associated custom element][form-associated custom
   elements].
2. Record its exact `name`, its `type` and every constraint attribute that
   applies to the type and is valid, exactly as the element's own validity would
   use it. Invalid attributes are left out, as HTML ignores them.
3. Merge each [radio button group][radio] into one field, which is `required` if
   any radio in the group is, matching HTML.
4. If another control already produced constraints for the same name, and they
   differ, throw a `"NotSupportedError"` `DOMException`. Identical constraints
   merge, which covers the common cases: checkbox groups and repeated rows such
   as `name="items[]"`.

This snapshots source names and constraints, not successful entries: unchecked
required controls still contribute rules. Later form changes do not update it,
and it does not translate names for JSON submission. Workers expose the
interface but have no form to pass to this overload.

### Validating

`validate(formData)` returns a result for every field in the record:

```js
{
  valid: false,
  fields: {
    email: { valid: false, valueMissing: true, typeMismatch: false, ... },
    username: { valid: true, valueMissing: false, ... },
  },
}
```

Each field has [`ValidityState`] flags except `customError`, which has no data
equivalent. Result keys are exactly the rule keys, including those without
entries; unruled entries are ignored.

Rules match ordinary entries by stored name and active JSON-only empty-state
contributions by original source name. The [JSON form encoding] activation
rules apply; contributions remain invisible to public getters and legacy
encodings. Validation neither converts to JSON nor canonicalises names.

Checks retain the raw string or `File` and optional internal JSON value.
Untyped entries use HTML's string/file semantics; typed values follow this
table without arbitrary coercion. "Present" means able to satisfy `required`,
not necessarily valid:

| Internal JSON value | Compatible rule types | Presence and checks |
| ------------------- | --------------------- | ------------------- |
| None | Existing string/`File` compatibility | Existing HTML checks and presence rules |
| Number | `number`, `range` | Present, including zero, if any nonempty raw value is a valid numeric string; check numeric bounds and step |
| `true` | `checkbox` | Present |
| `false` | `checkbox` | Empty |
| `null` | `number`, `range` | Empty numeric input if the raw value is `""`; no bounds or step check |
| Empty list `[]` | `checkbox`, `radio`, `select-multiple`, `file` with `multiple: true` | Empty group; no selected value |
| Empty map `{}` | None | Incompatible |

An incompatible value sets `badInput` and cannot satisfy `required`, even if its
stored string is nonempty. Every matching entry is checked, so a compatible
value cannot cancel an incompatible sibling's failure. Empty typed states skip
the nonempty-value checks.

For `number` and `range`, a **valid numeric string** must be a
[valid floating-point number] whose
[parsing][rules for parsing floating-point number values] returns a finite
number. `1e400` has valid syntax but overflows. Nonempty invalid raw strings set
`badInput` regardless of hints, skipping bounds and step checks rather than
using a partial parse. Date and time types use their own parsers.

Invalid nonempty strings satisfy `required` only when untyped. Typed numbers
and `null` with invalid raw strings do not. Empty strings are not invalid merely
because parsing fails. Hinted `null` represents empty numeric input only over
raw `""`; reconstruction always gives it that raw string, losing the
distinction between invalid and empty source input.

The remaining checks are HTML's, with three differences that follow from
validating data instead of an element:

- **Lengths always apply.** `tooLong` and `tooShort` don't depend on how the
  value was last changed.
- **Form encoding is undone first.** CRLF and lone CR become LF before any
  check, reversing what the encoders do. The value is otherwise checked as
  received, not sanitised.
- **`badInput` includes incompatible representations.** A typed value outside
  the compatibility table is invalid. For untyped entries, the flag is also set
  when:
  - a `File` is sent for a non-`file` field, or a string for a `file` field,
  - a nonempty `number` or `range` string isn't a valid numeric string,
  - a `date`, `month`, `week`, `time` or `datetime-local` value isn't a valid
    string for its type,
  - a `color` value would be changed by the colour state's value sanitisation
    algorithm, or
  - a `text`, `search`, `tel` or `password` value contains a line break.

If `required` applies and is true, `valueMissing` is true when no compatible
matching value is present:

- `checkbox`: an untyped string entry, including `""`, or typed `true`.
  Typed `false` and an empty-group contribution do not count.
- `radio`: an untyped string entry, including `""`; an empty group does not
  count.
- `file`: a `File` with a non-empty name. The native no-selection placeholder
  remains available to validation but does not count as present.
- `number` and `range`: a typed number whose raw string is a valid numeric
  string, or an untyped nonempty string. Typed `null` is not present.
- `select-multiple`: a compatible string entry, including a selected option
  with `value=""`. An empty-list marker does not count as a selected option.
- Everything else: a compatible nonempty string.

Otherwise `valueMissing` is false. A false or empty contribution does not veto
a present sibling: this is field-level aggregation, not a requirement that every
checkbox sharing a name be checked.

### Serialising

`toJSON()` returns the flat rule record with its keys unchanged, so the rules
can be stored or sent with `JSON.stringify()` and rebuilt with the constructor.
It does not apply `FormData`'s entry-name canonicalisation or nest the rules:

```js
const copy = new FormValidation(JSON.parse(JSON.stringify(rules)));
```

### Alternatives considered

- **Expose `ValidityState` everywhere and give it a constructor.** This changes
  where the flags live, not how they're computed; rules would still be read from
  an element.
- **Static helpers**, such as a function that tests an email address. These
  cover the grammars but not `required`, step bases, `multiple`, repeated names
  or files, and give no single object to share between client and server.
- **Adopt JSON Schema.** Its keywords would need an HTML profile redefining
  `pattern`, `multipleOf` and `format` to match the browser, making it a new
  vocabulary with confusing names. It also describes JSON documents, not entry
  lists.
- **Implement Standard Schema.** Its `~standard` property is a userland
  convention that a platform object shouldn't carry. An adapter around
  `FormValidation` is a few lines of library code.
- **A live object bound to the form.** It couldn't be serialised or used on a
  server, the main reasons for the API.
- **Return a list of issues with messages.** Validation messages are localised
  by the browser, and a server runtime has no strings to localise. Returning
  `ValidityState` flags lets authors reuse the code they already write to map
  `element.validity` to messages.

## Examples

A form with ordinary constraints:

```html
<form id="signup" method="post" action="/signup">
  <input name="email" type="email" required>
  <input name="username" required pattern="[a-z0-9_]+" maxlength="20">
  <input name="age" type="number" min="13" step="1">
  <textarea name="bio" maxlength="160"></textarea>
  <label><input name="plan" type="radio" value="free" required> Free</label>
  <label><input name="plan" type="radio" value="pro"> Pro</label>
  <button>Sign up</button>
</form>
```

Reading it in the browser:

```js
const rules = new FormValidation(document.getElementById("signup"));
JSON.stringify(rules);
```

```json
{
  "email": { "type": "email", "required": true },
  "username": {
    "type": "text",
    "required": true,
    "pattern": "[a-z0-9_]+",
    "maxLength": 20
  },
  "age": { "type": "number", "min": "13", "step": "1" },
  "bio": { "type": "textarea", "maxLength": 160 },
  "plan": { "type": "radio", "required": true }
}
```

Checking a submission in any WinterTC runtime, with the rules in a shared
module:

```js
import signup from "./signup-rules.json" with { type: "json" };

const rules = new FormValidation(signup);

export default {
  async fetch(request) {
    const { valid, fields } = rules.validate(await request.formData());
    if (!valid) {
      return Response.json(fields, { status: 422 });
    }
    // ...
  },
};
```

A client that bypasses the browser and posts
`email=me&username=Keith&age=12.5`, with no `plan`, gets:

| Field      | Flags set                        |
| ---------- | -------------------------------- |
| `email`    | `typeMismatch`                   |
| `username` | `patternMismatch`                |
| `age`      | `rangeUnderflow`, `stepMismatch` |
| `bio`      | none                             |
| `plan`     | `valueMissing`                   |

Rendering the form from the same rules, so markup and server can't drift:

```jsx
<input name="email" {...signup.email} />
<input name="username" {...signup.username} />
```

Checking an entry list built in script, here a multi-step form whose steps live
in separate `<form>` elements:

```js
const data = new FormData();
for (const step of document.querySelectorAll("form.step")) {
  for (const [name, value] of new FormData(step)) data.append(name, value);
}
if (rules.validate(data).valid) {
  await fetch("/signup", { method: "POST", body: data });
}
```

Checking an `application/json` submission uses trusted rules written for the
canonical names, rather than a snapshot of the original control names:

```js
const rules = new FormValidation({
  "pet[0][legs]": { type: "number", required: true, min: "0" },
  "tag[]": { minLength: 3 },
});
const data = FormData.fromJSON(await request.json());
const { valid, fields } = rules.validate(data);
```

For `{"pet":[{"legs":-1}],"tag":["ox"]}`, `valid` is false, with
`fields["pet[0][legs]"].rangeUnderflow` and `fields["tag[]"].tooShort` true.
The tag remains an array despite holding one value. A missing `pet[0][legs]`
instead produces `valueMissing` under the same rule key.

A source control named `pet[0][legs]` keeps that name, so its form-derived rule
also matches the reconstructed entry. A source control named `tag[0]` does not:
its form-derived rule validates the original entry, but the reconstructed entry
is named `tag[]`. An error for a canonical name identifies the JSON field, not
necessarily a control with that name; mapping it back to the control needs the
submitting page's knowledge of its control names.

Typed presence differs from stored-string presence:

```js
const consentRules = new FormValidation({
  consent: { type: "checkbox", required: true },
});
const typed = FormData.fromJSON({ consent: false });
const untyped = new FormData();
untyped.append("consent", "");
const typedResult = consentRules.validate(typed);
const untypedResult = consentRules.validate(untyped);
```

`typedResult.fields.consent.valueMissing` is true. The untyped empty string
represents a checked checkbox with an explicit empty value, so `untypedResult`
is valid. An eligible native unchecked Boolean checkbox has no visible entry
but supplies a JSON-only false contribution, so it fails `required` too.

For an empty `<select multiple name="tag[]">`, a form-derived required rule for
`tag[]` has type `select-multiple`, sees an empty-group contribution and reports
missing without `badInput`. Reconstructing `{"tag":[]}` instead creates the
canonical marker named `tag`, which an exact-name rule for `tag` can inspect. A
rule for `tag[]` has no matching entry: a required one reports missing, while an
optional one doesn't inspect the differently named marker. This API is not a
JSON shape validator.

Typed `0` is present under a required numeric rule, and typed `null` over an
empty raw string is missing. Overflow shows why both syntax and parsing matter.
For `{ type: "number" }`:

| Candidate | `required: true` | `required: false` |
| --------- | ---------------- | ----------------- |
| Untyped `1e400` | `badInput` only; the string counts as present | `badInput` |
| `"number"` hint on `1e400` (internal JSON `null`) | `badInput` and `valueMissing` | `badInput` |
| Reconstructed `null` (raw `""`) | `valueMissing` only | valid |

A typed number under a required text rule sets both `badInput` and
`valueMissing`. For `"consent[]": { type: "checkbox", required: true }`,
reconstructing `{"consent":[true,{}]}` sets `badInput` but not `valueMissing`,
since the valid `true` supplies presence but can't hide the incompatible
object.

## Caveats, shortcomings, and other drawbacks of design choices

- **Stricter lengths.** Prefilled values can pass browser length checks but
  fail data validation, which ignores edit history.
- **`wrap="hard"` textareas.** Wrapping inserts line breaks after browser length
  validation. The server cannot distinguish them from typed breaks and may
  report `tooLong`.
- **No messages.** Authors map flags to text.
- **Element-only validity.** Custom validity and custom-element checks have no
  declarative rules to extract and must be repeated in script.
- **Stale snapshots.** After controls change, rebuild the snapshot or use a
  dictionary describing the final form.
- **Only HTML constraints.** Cross-field checks, checkbox-group minimums, file
  size/`accept` checks and allowed option values remain script or library work.
- **Unknown names.** Servers must decide which unruled fields to accept.
- **Renamed entries.** Source-name rules can miss reconstructed entries,
  producing missing-value failures or unchecked optional fields. Use rules for
  the input representation, not an independently canonicalised source record.
- **Lossy reconstruction.** Filenames cannot recreate files; leaf-array labels,
  quoted keys and flattened conflicts lose their original spellings. Empty and
  invalid hinted numbers, including `1e400`, both become `null`. Reconstruction
  treats that as empty input, passing optional numeric rules and failing
  required ones, rather than reporting the original `badInput`.
- **Indexed gaps.** JSON conversion rejects missing array positions and sends
  no `application/json` or `multipart/json` request. Validation does not convert:
  with only `pet[1][name]`, a required `pet[0][name]` reports missing and an
  optional rule passes. An unanswered optional indexed radio group or a disabled
  or removed row can make an otherwise valid form unsendable as JSON. See the
  encoding [caveats][JSON form encoding caveats] for terminal `[]` or quoted-key
  alternatives. That explainer recommends an author-visible diagnostic before
  stage 2 ships; its form remains open there, not a policy of this proposal.

## Open questions

1. **Name.** `FormValidation` describes what the object does. `FormConstraints`
   describes what it holds, and may read better as a dictionary of rules.
2. **Same-name controls with different constraints.** The form constructor
   throws. Alternatives are per-control rules matched by position, which breaks
   for controls that submit zero or several entries, or matched by `value`,
   which only works for checkboxes. Throwing keeps the behaviour obvious until
   there is evidence these forms exist in practice.
3. **Step base from the `value` attribute.** HTML uses the `value` content
   attribute as the [step base][`step`] when `min` is absent. The dictionary has
   no member for it, so the form constructor can't represent
   `<input type=number value=0.5 step=1>`. A `stepBase` member would fix this
   at the cost of a constraint name with no attribute equivalent.
4. **Allowed values.** The form constructor could record checkbox, radio and
   `option` values, letting a server reject values no control could send. HTML
   has no such constraint, and options are often added in script, so this is
   left out.
5. **Custom validators.** Accepting functions would break serialisation, and
   composing `validate()` with script already covers them.
6. **Converted results.** `validate()` could also return converted values
   (`valueAsNumber`, `valueAsDate`), which overlaps with the typed values in the
   [JSON form encoding] explainer but is separate from matching canonical
   names.
7. **One rule set for both representations.** The same trusted rules must give
   the same rule targets and validity flags for original and reconstructed
   `FormData`. Dense indices preserve positions, but leaf-array labels, quoted
   keys and flattened conflicts still change entry names. For example,
   `{"tag":[]}` reconstructs as a marker named `tag`, while a nonempty `tag`
   array reconstructs as entries named `tag[]`, so no single exact-name rule
   inspects both (see [Examples]). The current exact-name matching does not
   meet this requirement, and how to share the structural interpretation
   without losing raw-value checks remains open.

## Draft specification

### IDL

```webidl
enum FormFieldType {
  "text", "search", "tel", "url", "email", "password",
  "date", "month", "week", "time", "datetime-local",
  "number", "range", "color", "checkbox", "radio", "file",
  "textarea", "select-one", "select-multiple"
};

dictionary FormFieldConstraints {
  FormFieldType type = "text";
  boolean required;
  boolean multiple;
  DOMString pattern;
  DOMString min;
  DOMString max;
  DOMString step;
  [EnforceRange] unsigned long minLength;
  [EnforceRange] unsigned long maxLength;
};

dictionary FormFieldValidity {
  boolean valueMissing = false;
  boolean typeMismatch = false;
  boolean patternMismatch = false;
  boolean tooLong = false;
  boolean tooShort = false;
  boolean rangeUnderflow = false;
  boolean rangeOverflow = false;
  boolean stepMismatch = false;
  boolean badInput = false;
  boolean valid = true;
};

dictionary FormValidationResult {
  required boolean valid;
  required record<USVString, FormFieldValidity> fields;
};

[Exposed=*]
interface FormValidation {
  constructor(optional (HTMLFormElement or
                        record<USVString, FormFieldConstraints>) init = {});
  FormValidationResult validate(FormData data);
  record<USVString, FormFieldConstraints> toJSON();
};
```

### HTML

Refactor each "suffering from" definition in [constraint validation] and the
input type sections so the condition is stated over a set of constraints and a
value, and have element validity call it with the element's constraints and
value. Element validity keeps its extra conditions, such as the user-edit
condition on lengths. `FormValidation` calls the same definitions, so the two
can't diverge. This also gives [whatwg/html#4562] a single place to change
email validity.

Each `FormValidation` has a **rule map**, an ordered map from entry-name strings
to constraint sets. Names are compared exactly, without parsing bracket paths,
merging canonical names or flattening conflicts.

The **`new FormValidation(init)`** constructor steps are:

1. If _init_ is an `HTMLFormElement`, set this's rule map to the result of
   reading the form's constraints from _init_, using the steps in [Constructing
   from a form].
2. Otherwise, for each _name_ and _constraints_ of _init_:
   1. If any member of _constraints_ other than `type` is present, isn't false,
      and doesn't apply to _constraints_["type"], throw a `TypeError`.
   2. If `min`, `max` or `step` is present and parsing it with the type's
      algorithm to convert a string to a number returns an error (and, for
      `step`, it isn't an ASCII case-insensitive match for `"any"`), throw a
      `TypeError`.
   3. If `pattern` is present and compiling it as a [`pattern`] returns an
      error, throw a `SyntaxError`.
   4. Set this's rule map[_name_] to _constraints_.

The **`validate(data)`** method steps are:

1. Let _fields_ be an empty ordered map.
2. For each _name_ and _constraints_ of this's rule map:
   1. Let _entries_ be the entries of _data_'s entry list whose stored name is
      _name_. Do not convert the entry list to JSON or replace its names with
      canonical names.
   2. Let _result_ be a `FormFieldValidity` with every flag false.
   3. Append each active empty-state contribution whose original source name
      is _name_ to _entries_, using the encoding proposal's activation rules.
      Keep ordinary entries, including native no-selection file placeholders.
   4. Let _present_ be false.
   5. For each candidate of _entries_:
      1. Retain its raw string or `File` and optional JSON value. A contribution
         supplies its typed false or empty-list value. Normalise CRLF and lone
         CR to LF in a local raw string, without modifying the candidate.
      2. If its representation is incompatible under the table in [Validating],
         set _result_["badInput"] to true and continue.
      3. If the rule type is `number` or `range`, the candidate is ordinary, and
         its raw string is nonempty and not a valid numeric string under
         [Validating] (a valid floating-point number that parses to a finite
         value), set _result_["badInput"] to true. If it is untyped, set
         _present_ to true under the existing nonempty string rule; a typed
         candidate, including a hinted `null`, does not set _present_ and is
         not treated as an empty state. Continue without applying bounds or
         step checks.
      4. If it satisfies the corresponding presence predicate in [Validating],
         set _present_ to true.
      5. If it is a compatible typed empty state, continue.
      6. Apply the other applicable HTML checks, without the user-edit
         condition, OR-ing each failure into its field flag. For a typed
         number, bounds and step use that number; untyped values use the
         ordinary raw-value algorithms.
   6. Set _result_["valueMissing"] to true if `required` applies and is true
      and _present_ is false, or to false otherwise.
   7. Set _result_["valid"] to true if every other flag is false.
   8. Set _fields_[_name_] to _result_.
3. Return a `FormValidationResult` whose `valid` is true if every value of
   _fields_ has `valid` true, and whose `fields` is _fields_.

The **`toJSON()`** method steps return this's rule map.

### WinterTC

Add `FormValidation` to the [Minimum Common Web Platform API], alongside
`FormData`, with the `HTMLFormElement` overload not applicable.

## Incubation and/or standardization destination

WHATWG HTML, as an addition to constraint validation, through the [WHATWG
Stages] process starting with an issue on whatwg/html. Defining
`FormValidation` next to the algorithms it shares with element validity keeps
them in step.

Once it is in HTML, propose it to WinterTC's [proposal-minimum-common-api] so
server runtimes ship it alongside `FormData`.

[Keith Cirkel]: https://github.com/keithamus
[constraint validation]:
  https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#constraints
[`ValidityState`]:
  https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#validitystate
[`ElementInternals.setValidity()`]:
  https://html.spec.whatwg.org/multipage/custom-elements.html#dom-elementinternals-setvalidity
[form-associated custom elements]:
  https://html.spec.whatwg.org/multipage/custom-elements.html#form-associated-custom-element
[barred from constraint validation]:
  https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#barred-from-constraint-validation
[radio]:
  https://html.spec.whatwg.org/multipage/input.html#radio-button-state-(type=radio)
[`maxlength`]:
  https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#limiting-user-input-length:-the-maxlength-attribute
[`textarea`]:
  https://html.spec.whatwg.org/multipage/form-elements.html#the-textarea-element
[name-value pairs]:
  https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#convert-to-a-list-of-name-value-pairs
[`pattern`]:
  https://html.spec.whatwg.org/multipage/input.html#the-pattern-attribute
[`step`]: https://html.spec.whatwg.org/multipage/input.html#the-step-attribute
[valid email address]:
  https://html.spec.whatwg.org/multipage/input.html#valid-e-mail-address
[whatwg/html#4562]: https://github.com/whatwg/html/issues/4562
[Minimum Common Web Platform API]: https://min-common-api.proposal.wintertc.org/
[proposal-minimum-common-api]:
  https://github.com/WinterTC55/proposal-minimum-common-api
[`getZodConstraint`]: https://conform.guide/api/zod/getZodConstraint
[superforms constraints]:
  https://superforms.rocks/concepts/client-validation#constraints
[Zod regexes]:
  https://github.com/colinhacks/zod/blob/main/packages/zod/src/v4/core/regexes.ts
[validator isEmail]:
  https://github.com/validatorjs/validator.js/blob/master/src/lib/isEmail.js
[JSON Schema pattern]:
  https://json-schema.org/draft/2020-12/json-schema-validation#section-6.3.3
[JSON Schema multipleOf]:
  https://json-schema.org/draft/2020-12/json-schema-validation#section-6.2.1
[Standard Schema]: https://github.com/standard-schema/standard-schema
[Extensible Web Manifesto]: https://extensiblewebmanifesto.org/
[JSON form encoding]: form-json-encoding.md
[JSON form encoding caveats]:
  form-json-encoding.md#caveats-shortcomings-and-other-drawbacks-of-design-choices
[Constructing from a form]: #constructing-from-a-form
[Validating]: #validating
[Examples]: #examples
[valid floating-point number]:
  https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#valid-floating-point-number
[rules for parsing floating-point number values]:
  https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#rules-for-parsing-floating-point-number-values
[WHATWG Stages]: https://whatwg.org/stages
