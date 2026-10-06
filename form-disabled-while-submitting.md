# Disabled while submitting explainer

Author: [Keith Cirkel]

## User problems to be solved

A user who presses a submit button and sees nothing change will often press it
again. Slow networks, slow servers and script handlers that wait on `fetch()`
all leave a gap between the press and any visible result. During that gap the
form looks idle and stays fully interactive, so a second press starts a second
submission. At best that is harmless. At worst it means a duplicate order or
payment, or a second request that races the first and leaves the page in a
confusing state.

Authors fix this by disabling the submit button, or the fields as well so the
user cannot edit values while they are being saved. Doing that correctly takes
more script than it appears to:

1. **Controls must be disabled at the right moment.** Disabling them in a
   `click` handler runs before [form submission][form submission algorithm]
   has collected the form's entries. A [disabled][concept-fe-disabled]
   control is skipped, so its `name` and `value` disappear from the payload.
   Servers that branch on which button was pressed ("Save draft" versus
   "Publish") then receive the wrong request. Disabling a fieldset can omit
   the whole group of fields.
2. **Controls must be re-enabled at the right moment.** Script has to know
   when the work has finished, including failures, and restore the previous
   state without overwriting a `disabled` that something else set.
3. **Native completion is not exposed to page script.** A native submission may
   replace the document, return no content, start a download, or target another
   browsing context. There is no form completion event covering these cases.
4. **Associated controls need the same treatment.** Controls outside the form
   that use the [`form` attribute], several submit buttons, and a "Cancel"
   button that must stay usable all need individual handling.

## Methodology for approaching & evaluating solutions

Follow the Mozilla [Web Vision]'s preference for declarative features. The
native case should work without script, and script should extend the same
model. Preserve the submitted payload and existing unconditional disabling.
Evaluate the compatibility cost of extending a boolean attribute explicitly;
this spelling does not provide a usable fallback in unsupported browsers.

## Prior features and proposals

- The [`disabled` attribute][attr-fe-disabled] on form controls and fieldsets,
  toggled from script.
- React's [`useFormStatus`], which exposes pending state for a parent form so a
  component can render a disabled submit button.
- The companion [form submission state explainer][submission lifetime], which
  lets script register work with the form's pending lifetime.

## Flaws or limitations in existing features/proposals

Script-driven disabling can omit controls from the payload and must be
repeated per page or component. It does nothing for forms without script.
Framework helpers can render their state into the DOM, but do not establish a
shared native lifetime for work registered by independent callers.

## Motivation for this explainer

The browser already knows when a native submission starts, what the submitter
was, and when the payload has been built. It is in the right place to apply
a busy state after the payload is captured and to clear it when the form's
[submission lifetime] ends. A keyword on the existing attribute lets authors
request disabled interaction while work is pending, without writing a script
to toggle controls for a native submission.

## Proposed solution

Add a keyword value to the `disabled` attribute on `<button>`, `<input>`,
`<select>`, `<textarea>` and `<fieldset>`, and add `disabled` to `<form>`:

```html
<button disabled="while-submitting">Pay</button>
```

A control with `disabled="while-submitting"` is enabled while its form owner
is idle, and disabled while its form owner is submitting. This includes text
inputs, checkboxes and input-based submitters. The form owner is resolved as
usual, so a control associated through the [`form` attribute] follows the form
it names, not its tree position. Without a form owner, the keyword alone does
not disable it.

On a `<fieldset>`, the keyword applies the existing fieldset disabling rules
while its form owner is submitting, including the first-`legend` exception.
Its descendant controls need not opt in individually.

On a `<form>`, the keyword disables all controls owned by that form while it
is submitting, including controls outside its subtree. Controls owned by
another form are unaffected. Unlike a fieldset, it has no `legend` exception.

When a form is submitting is defined in the shared
[form submission state explainer][submission lifetime]. In short, the form is
submitting while any native submission or any promise registered with the
provisional `form.trackSubmission(promise)` method is pending, and stops once
all of them have settled. This explainer does not repeat that algorithm.

The ordering matters for the payload. When submission begins, the browser
builds the payload with all eligible controls included, as described in
[preserving the submission entry list], and only then applies the disabled
state. Conditionally disabling an input, fieldset or form must not remove its
values from the submission that caused the state change.

Any other value of `disabled`, including the empty string, means
unconditionally disabled. This preserves existing behaviour on controls and
fieldsets, and gives the new form attribute an unconditional form-wide mode.
Ordinary disabling takes precedence over the keyword. A control inside a
disabled fieldset or owned by a disabled form stays disabled even if its own
keyword condition is false.

Controls without the keyword are unaffected unless disabled through a fieldset
or their form. A "Cancel" or "Back" button can remain usable by leaving it
outside the disabled group and keeping the form itself enabled. Nothing in
this proposal cancels a submission; any cancel behaviour is the author's own
code.

An attempt that fails interactive validation adds no new pending work. It
does not conditionally disable an idle form's controls; existing pending work,
if any, is unaffected.

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

The text input, checkbox and submit input become disabled after their entries
have been captured. The request includes `displayName=Keith`, `updates=on`
and `action=Save`; the user cannot change the fields while it is pending.

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
capturing their entries. No control needs its own conditional attribute.

### Script-driven submission

The complete [scripted example](form-submission-state.md#replacing-native-submission)
captures `new FormData(form, event.submitter)`, explicitly calls
`preventDefault()`, and registers its request and response processing through
the provisional `trackSubmission()` method. Cancelling the default action
completes the native contribution, while the registered promise keeps the form
submitting.

Several promises can be tracked for one form. The opted-in controls and groups
stay disabled until every one has settled, and a rejected promise does not end
the busy state early.

### Styling

```css
[disabled="while-submitting" i]:disabled {
  cursor: progress;
}
```

`:disabled` matches opted-in controls, fieldsets and forms while submitting,
and controls disabled through a fieldset or form. It stops matching once they
are no longer disabled.

## Caveats, shortcomings, and other drawbacks of design choices

**Old browsers disable opted-in controls permanently.** A browser without this
feature reads `disabled="while-submitting"` as a plain boolean `disabled` on
controls and fieldsets. On a form, it ignores the attribute instead, leaving
the controls usable without protection. Authors who need to support those
browsers must feature-detect and remove or rewrite the attribute, which means
script. We considered a separate attribute, which would fall back to enabled
controls, and rejected it in favour of extending `disabled`.

**Existing content may already use the value.** Pages that today write
`disabled="while-submitting"` (or generate it from a template) get permanently
disabled controls now and would get conditionally enabled controls after this
change. Adding meaning to `disabled` on forms also affects existing uses of
that attribute. Compatibility needs investigation before shipping.

**No exactly-once guarantee.** Disabling controls prevents interaction with
those controls. It does not stop [implicit submission] through controls that
remain enabled, calls to `form.submit()` or
`requestSubmit()`, another tab, a retrying proxy, or a user reloading. Servers
that must not process a request twice still need idempotency keys or similar.

**Completion of native submissions.** A navigating submission normally ends
when the document is replaced, so the disabled state does not need clearing.
When the navigation does not replace the document (a `204 No Content`
response, a download, a submission targeting another browsing context, or a
navigation that is cancelled or fails), the submission lifetime must still
end and the conditionally disabled controls must re-enable. The shared
explainer records these completion hooks as open questions.

**Back/forward cache restoration.** A restored page must not retain stale
native submission state. Unconditionally resetting every form to idle would
also clear still-pending registered promises, so telling obsolete work apart
from them is a shared lifecycle question.

**Disabling a form is not a submission lock.** It disables its controls, but
does not prevent programmatic submissions or cancel work automatically.
Authors who need a usable Cancel button should disable a fieldset or individual
controls instead.

## Open questions

1. **Method name.** `trackSubmission(promise)` is a working name used across
   these explainers. The final name is not settled.
2. **`.disabled` reflection.** On existing elements, `.disabled` reflects the
   attribute as a boolean, so it returns `true` for
   `disabled="while-submitting"` even while the element is usable. Should it
   keep reflecting the attribute, or report the current state? Keeping
   reflection means script must use `matches(":disabled")` to learn the
   current state. Setting `.disabled = false` would also remove the keyword.
   Should the new form attribute have the same boolean reflection?
3. **`:disabled` and accessibility.** We propose that `:disabled` and the
   accessibility tree follow the current state, not the attribute. Whether
   assistive technology should also announce that the form is busy (for
   example through `aria-busy` on the form) is open.
4. **A `:submitting` pseudo-class.** A selector for the form's state would let
   authors style content independently of disabling it. It is not part of this
   proposal and is left open.
5. **Feature detection.** Is there a clean way to detect support, so authors
   can strip the attribute in old browsers, without parsing the attribute
   value through a new IDL attribute?

## Draft specification

Proposed changes to the HTML Standard, as an outline:

1. In [the `disabled` attribute][attr-fe-disabled], define the keyword
   `while-submitting`, matched ASCII case-insensitively, for `button`, `input`,
   `select`, `textarea` and `fieldset`. Add `disabled` to `form`, accepting the
   same keyword and treating other values as unconditional disabling.
2. In the definition of [disabled][concept-fe-disabled] for form controls,
   make the keyword itself disable a control only while it has a submitting
   form owner, as defined in [submission lifetime]. Apply the same condition
   to fieldsets using their existing descendant and first-`legend` rules.
   A disabled form disables every control it owns, including external
   controls. Non-keyword disabling and inherited disabling apply independently
   of a control's own keyword condition.
3. Make `:disabled` and the accessibility tree reflect the effective state of
   controls, fieldsets and forms.
4. In the [form submission algorithm], after the entry list is constructed
   with all eligible controls, mark the form as having a pending native
   submission as described in [preserving the submission entry list]. The
   disabled state then applies through step 2 without further changes here.
5. Failed validation creates no new native contribution. Cancelling `submit`
   completes its native contribution without clearing registered promises.
6. Resolve the shared native completion and restoration hooks before defining
   their effect on conditional disabling normatively.

The submission lifetime, `trackSubmission()` and the native contribution rules
are specified by the shared explainer, not here.

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
