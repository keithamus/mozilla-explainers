# Disabled while submitting explainer

Author: [Keith Cirkel]

## Summary

Disable a button while its form is submitting, without script to toggle it.
Its name and value still reach the server.

```html
<form method="post" action="/checkout">
  <button name="action" value="pay" disabled="while-submitting">Pay</button>
</form>
```

The same keyword works on fields, fieldsets and whole forms. Controls re-enable
when native submission and any [registered promises][submission lifetime] have
finished.

## User problems to be solved

A form that looks idle during a slow submission invites another press, risking
duplicate requests or orders. Authors disable buttons, and sometimes fields,
but must handle:

1. **Timing.** Disabling in `click` happens before [form submission][form
   submission algorithm] collects entries. [Disabled][concept-fe-disabled]
   controls are skipped, losing the submitter's `name` and `value` or a whole
   fieldset's values.
2. **Restoration.** Controls must re-enable after completion or failure without
   overwriting an independently set `disabled`.
3. **Native completion.** No form completion event covers navigation, downloads,
   no-content responses and submissions targeting other contexts.
4. **Associated controls.** External controls using the [`form` attribute],
   multiple submitters and buttons that must stay usable need individual
   handling.

## Methodology for approaching & evaluating solutions

Follow the Mozilla [Web Vision]'s preference for declarative features. Support
native submission without script and extend the same model to script. Preserve
payloads and unconditional disabling, and assess compatibility with a boolean
attribute whose new value has no usable fallback.

## Prior features and proposals

- The [`disabled` attribute][attr-fe-disabled] on form controls and fieldsets,
  toggled from script.
- React's [`useFormStatus`], which exposes pending state for a parent form so a
  component can render a disabled submit button.
- The companion [form submission state explainer][submission lifetime], which
  lets script register work with the form's pending lifetime.

## Flaws or limitations in existing features/proposals

Script-driven disabling risks omitting values and must be repeated per
component. Framework helpers provide no shared native lifetime for independent
callers, and neither approach works without script.

## Motivation for this explainer

The browser can disable controls for the [submission lifetime] without omitting
their values from native submission. A keyword would make this declarative.

## Proposed solution

Add a keyword value to the `disabled` attribute on `<button>`, `<input>`,
`<select>`, `<textarea>` and `<fieldset>`, and add `disabled` to `<form>`.

A control with `disabled="while-submitting"` is disabled only while its form
owner is submitting, including when associated through the [`form` attribute].
Without a form owner, the keyword alone does not disable it.

While its form owner is submitting, a `<fieldset>` applies the existing
descendant disabling rules, including the first-`legend` exception. A `<form>`
with the keyword disables every owned control while submitting, including
external controls, without a `legend` exception. Controls owned by another
form are unaffected.

The shared [form submission state explainer][submission lifetime] defines
submitting as having any pending native submission or promise registered with
the provisional `form.trackSubmission(promise)` method.

Conditional disabling applies after `submit` dispatch, before waiting for any
[submission holds][submission hold]. If `submit()` bypasses dispatch, apply it
before native entry construction. Controls remain disabled through the hold
and subsequent submission, until every pending contribution has settled.

For the triggering native attempt, conditional disabling prevents interaction
without excluding controls from post-hold constraint validation or the payload.
Those operations ignore disabling caused solely by `while-submitting`,
including through a conditional fieldset or form. Ordinary disabling and other
eligibility rules still apply. Values are captured after holds and repeated
validation, not frozen when controls disable. See
[preserving the submission entry list] for native and scripted ordering.

Other `disabled` values, including the empty string, mean unconditional
disabling on controls, fieldsets and forms. Ordinary or inherited disabling
takes precedence over a control's own keyword condition.

Controls without the keyword remain usable unless disabled through their
fieldset or form. To keep Cancel usable, leave it outside the disabled group
and keep the form enabled. Cancellation itself still requires author code.

Failed initial interactive validation adds no pending work, leaving an idle
form's controls enabled and any existing work unaffected. Failed validation
after a hold ends only that attempt's native contribution.

## Examples

### Native submission, no script

```html
<form method="post" action="/checkout">
  <label>Card number <input name="card" required></label>
  <button name="action" value="pay" disabled="while-submitting">Pay</button>
  <button type="reset">Clear</button>
</form>
```

After a valid press of "Pay", the request includes `action=pay` and the Pay
button becomes disabled. Clear stays enabled, but resetting the form does not
cancel the pending request.

### Buttons outside the form

```html
<form id="post" method="post" action="/posts"></form>
<textarea form="post" name="body"></textarea>
<footer>
  <button form="post" name="state" value="draft" disabled="while-submitting">
    Save draft
  </button>
  <button form="post" name="state" value="live" disabled="while-submitting">
    Publish
  </button>
</footer>
```

Both buttons are disabled for the whole submission, whichever was pressed, and
the payload includes the `state` of the pressed button.

### Inputs and checkboxes

```html
<form method="post" action="/preferences">
  <label>
    Display name
    <input name="displayName" value="Keith" disabled="while-submitting">
  </label>
  <label>
    <input type="checkbox" name="updates" checked disabled="while-submitting">
    Email me updates
  </label>
  <input type="submit" name="action" value="Save" disabled="while-submitting">
</form>
```

The text input, checkbox and submit input become disabled after `submit`
dispatch. The request still includes `displayName=Keith`, `updates=on` and
`action=Save`; the user cannot change the fields while it is pending.

### A group of controls

```html
<form method="post" action="/shipping">
  <fieldset disabled="while-submitting">
    <legend>Shipping address</legend>
    <label>Address <input name="address" required></label>
    <label>Postcode <input name="postcode" required></label>
    <button>Save</button>
  </fieldset>
  <button type="button">Cancel</button>
</form>
```

The fieldset disables its fields and Save button during submission. Cancel
stays usable. As with an ordinary disabled fieldset, controls inside its first
`legend` would be exempt.

### A whole form

```html
<form id="profile" method="post" action="/profile" disabled="while-submitting">
  <label>Name <input name="name" required></label>
  <button>Save</button>
</form>
<label>
  <input type="checkbox" form="profile" name="public" checked>
  Public profile
</label>
```

The form disables all its controls, including the checkbox outside it, after
`submit` dispatch. Their values still reach the server. No control needs its
own conditional attribute.

### Script-driven submission

The [scripted example](form-submission-state.md#replacing-native-submission)
captures `new FormData(form, event.submitter)`, cancels native submission and
registers the request and response processing with `trackSubmission()`.
Controls stay disabled until every contribution settles, even after a rejection.

### Styling

```css
[disabled="while-submitting" i]:disabled {
  cursor: progress;
}
```

`:disabled` matches effectively disabled controls, fieldsets and forms,
including controls disabled through a fieldset or form.

## Caveats, shortcomings, and other drawbacks of design choices

**Unsupported browsers.** They treat the keyword as unconditional disabling on
controls and fieldsets, but ignore it on forms. Authors must feature-detect and
remove or rewrite it in script. A separate attribute would fall back to enabled
controls; we rejected it in favour of extending `disabled`.

**Compatibility.** Existing `disabled="while-submitting"` controls would change
from permanently to conditionally disabled. Both this and giving `disabled`
meaning on forms need compatibility investigation before shipping.

**No submission lock.** Disabling prevents interaction, not [implicit
submission] through enabled controls, programmatic submission, retries or
requests from other tabs. Servers still need idempotency keys or similar
protection against duplicates.

**Native completion and restoration.** No-navigation responses, downloads,
other targets and cancelled or failed navigation must release disabled state.
Restored pages must discard obsolete native work without clearing still-pending
promises. These hooks remain open in the shared explainer.

## Open questions

1. **Method name.** `trackSubmission(promise)` remains provisional.
2. **`.disabled` reflection.** Boolean reflection returns `true` for the keyword
   even while the control is usable; `matches(":disabled")` gives its effective
   state. Should reflection change, and should forms use the same model?
   Setting `.disabled = false` would remove the keyword.
3. **Accessibility.** `:disabled` and the accessibility tree would follow the
   effective state. Should assistive technology also announce form busy state,
   for example through `aria-busy`?
4. **`:submitting`.** Should a separate pseudo-class expose pending state for
   styling? It is not part of this proposal.
5. **Feature detection.** Can support be detected without a new IDL attribute
   for parsing the value?

## Draft specification

1. Define ASCII case-insensitive `while-submitting` in
   [the `disabled` attribute][attr-fe-disabled] on `button`, `input`, `select`,
   `textarea` and `fieldset`. Add `disabled` to `form` with the same keyword;
   other values disable unconditionally.
2. Extend [disabled][concept-fe-disabled] to apply the keyword while the form
   owner is submitting, retaining fieldset rules. Forms with the keyword disable
   all owned controls while submitting. Ordinary and inherited disabling apply
   independently.
3. Make `:disabled` and accessibility reflect effective state on controls,
   fieldsets and forms.
4. In the [form submission algorithm], apply conditional disabling after the
   submission-event phase, before waiting for holds or constructing entries,
   including when `submit()` skips that phase. For the triggering native
   attempt's repeated validation and entry construction, ignore disabling
   caused solely by `while-submitting`, as defined in
   [preserving the submission entry list]. Ordinary disabling still applies.

The shared explainer defines pending contributions, submission holds, failed
validation and cancellation. Native completion and restoration hooks must be
resolved before their effect on disabling becomes normative.

## Incubation and/or standardization destination

This is a change to the [HTML Standard], through [WHATWG Stages], developed
alongside [form submission state](form-submission-state.md). Related work:
[form error element](form-error-element.md),
[form validation](form-validation.md) and
[form JSON encoding](form-json-encoding.md).

[Keith Cirkel]: https://github.com/keithamus
[Web Vision]: https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb
[form submission algorithm]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#form-submission-algorithm
[concept-fe-disabled]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#concept-fe-disabled
[attr-fe-disabled]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#attr-fe-disabled
[`form` attribute]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#attr-fae-form
[implicit submission]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#implicit-submission
[HTML Standard]: https://html.spec.whatwg.org/multipage/
[WHATWG Stages]: https://whatwg.org/stages
[`useFormStatus`]: https://react.dev/reference/react-dom/hooks/useFormStatus
[submission lifetime]: form-submission-state.md#submission-lifetime
[preserving the submission entry list]: form-submission-state.md#preserving-the-submission-entry-list
[submission hold]: form-submission-state.md#holding-native-submission
