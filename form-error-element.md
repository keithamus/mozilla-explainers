# The `<error>` element explainer

Author: [Keith Cirkel]

## Summary

Place validation errors next to a field, a group of fields or a whole form,
using an `<error>` element instead of a validation bubble. CSS controls when
the error is visible. Child content can provide links or other rich UI.

```html
<form>
  <label for="email">Email (required)</label>
  <error for="email"></error>
  <input id="email" name="email" type="email" required>
  <button>Sign up</button>
</form>
```

The proposed default presentation reserves space for errors and uses
`:user-invalid` to show them. Authors can override these styles:

```css
error {
  display: block;
  visibility: hidden;
}

:user-invalid::error {
  visibility: visible;
}
```

## User problems to be solved

Browsers report HTML [constraint validation] problems in browser-managed UI,
commonly a temporary bubble. Authors cannot place it in their layout or style
it with the page's CSS.

Persistent in-page messages require script to read [`validationMessage`],
control visibility, maintain an accessible association and update the message
as validity changes. Authors should be able to reuse the browser's localised
messages and choose their presentation with CSS, without rebuilding the
accessible association.

## Methodology for approaching & evaluating solutions

Follow Mozilla's [Web Vision] with a declarative feature that:

1. Displays existing validation messages using ordinary page layout and CSS.
2. Keeps validity separate from presentation, including author-set group errors.
3. Uses `:user-invalid` for default visibility, with author-controlled styling.
4. Preserves native reporting for individual controls in unsupported browsers.

## Prior features and proposals

**Validation bubbles.** [`reportValidity()`] and form submission run
[interactively validate the constraints], which reports problems to the user in
a UA-defined way. [`checkValidity()`] runs [statically validate the
constraints], which fires `invalid` events but does not report.

**[`setCustomValidity()`].** Sets a custom error on a control. For a control
eligible for validation, a non-empty message makes it invalid and
`validationMessage` returns the custom string until it is cleared. A
[`fieldset`] already exposes this API, but is not eligible for validation;
its custom error does not block submission. A [`form`] has `checkValidity()`
and `reportValidity()` but no own custom error or `validationMessage`.

**[OpenUI #802].** During `<selectlist>` work, the Open UI CG resolved that
validation errors are "a good problem to solve, but it's independent of the
`<selectlist>`. We should solve this generally." Suggestions included an
author-placed `<error for=IDREF_of_input>`, extending `<output>`, and a
`::error` pseudo-element for styling the bubble with anchor positioning and
`content`.

**[whatwg/html #9878].** Proposes a cancelable `report` event so controls,
especially form-associated custom elements, can distinguish interactive
reporting from `checkValidity()`, since both fire `invalid`. Comments also
propose per-constraint messages, a target container attribute and form attributes
controlling reporting triggers.

## Flaws or limitations in existing features/proposals

- Native bubbles do not participate in page layout.
- Cancelling `invalid` or adding a `report` event still leaves presentation to
  script.
- Per-constraint elements need message selection and precedence rules.
- A generated bubble is not an author-placed DOM element.

## Motivation for this explainer

An element could display an existing validation message without duplicating
constraints in presentation code. CSS would control visibility;
`checkValidity()` would remain non-reporting. Group and form errors need
validation API changes too.

## Proposed solution

1. **Binding.** `for` identifies one validation target by ID: an `input`,
   `textarea`, `select`, form-associated custom element, `fieldset` or `form`.
   Multiple associations and shadow boundaries remain open.
2. **Content.** `::validation-message` contains the target's current validation
   message, localising built-in text to the error's language. Author child
   nodes remain intact and appear alongside that message.
3. **Custom messages.** `setCustomValidity()` or
   `ElementInternals.setValidity()` supplies custom or server errors. Child
   content and CSS do not change validity. Group and form errors use the
   extensions below; no declarative cross-field constraints are added.
4. **Presentation.** Default styles use `visibility: hidden` and
   `:user-invalid::error` to reserve space and show reported errors. Authors
   can change these styles. There is no separate reveal flag or rendering gate.
5. **Reporting.** `checkValidity()` remains non-reporting. `reportValidity()`
   and validated submission would use the associated error in place of the
   native bubble for that target. Hidden replacements and cancellation still
   need rules.
6. **State.** Messages update with validity independently of visibility.
   Reporting-state integration with `:user-invalid`, reset and the proposed
   container APIs needs specification.
7. **Accessibility.** The browser provides the error relationship. Accessible
   invalid state, announcements and container discovery remain requirements to
   solve with ARIA and HTML-AAM editors.

### Fieldset errors

An error can belong to a group rather than either individual field, for example
when a repeated password does not match the first. Bind `<error>` to the
`fieldset` and use `fieldset.setCustomValidity(message)`.

This requires making enabled fieldsets eligible for constraint validation.
Their own `checkValidity()` and `reportValidity()` would check their custom
error, and `validationMessage` would expose it. Form validation would include
each fieldset whose form owner is that form, alongside the individual controls.
A group error would block validated submission and fire `invalid` at the
fieldset. Disabled fieldsets would not participate.

The fieldset's message is its own custom error, not a selected descendant's
message. Its validation methods would not recursively validate its descendants;
form validation would still check each eligible control once. Setting or
clearing a group error does not change the controls' validity.

This changes existing fieldset behaviour even without an `<error>` element.
Compatibility needs investigation before adopting the validation change.

### Form errors

Add `HTMLFormElement.setCustomValidity(message)` and a read-only
`validationMessage` for the form's own custom error. This covers a rejection
that belongs to no individual field, such as an unavailable booking.

`form.checkValidity()`, `form.reportValidity()` and validated submission would
check both the form's own error and its eligible controls, including fieldsets.
A non-empty form error would make validation fail and fire `invalid` at the
form, even if every control is valid. Clearing it would leave field errors
unchanged. The existing form `:valid` and `:invalid` states would also account
for its own error.

`<error for="form-id">` would show only that form error. It would not become an
error summary or select the first invalid field's message. A form with invalid
controls but no own error would have an empty `validationMessage`.

Existing validation bypasses, including `novalidate`, `formnovalidate` and
`form.submit()`, would still bypass validation. Reset would not clear custom
validity, matching the existing control API. Authors must clear stale group
and form errors before retrying.

### CSS-driven presentation

The opening stylesheet describes the default for individual controls. Authors
can use other selectors to show errors earlier or later, use `display: none`
to collapse their space, or choose an out-of-flow presentation.

With `visibility: hidden`, generated text and author children still participate
in layout. There is no internal rule suppressing their boxes until reporting.
Revealing unchanged content therefore does not change its size. Different
messages can still change the size, and a valid target has no generated message.
Authors needing a stable region can reserve space with `min-block-size`.
Arbitrary rich content cannot be guaranteed shift-free without reserving its
size.

Ordinary CSS visibility and focus behaviour apply to descendants. Hidden
content is not normally exposed in the accessibility tree, but the native error
relationship must also define when its text can be announced.

Calling `checkValidity()` does not itself request reporting or reveal errors.
Setting custom validity changes validity, not reporting state; author styles
can still react to that change. How explicit reporting updates `:user-invalid`
for all eligible targets remains to be specified.

The default selector is not yet sufficient for fieldsets and forms. Container
invalidity can reflect a descendant, while their `<error>` represents only
their own custom error. A required field must not reveal an unrelated
"Passwords do not match" message. Distinguishing the container's own reported
error without another reveal mechanism needs a solution.

### Child content and generated messages

`<error>` accepts flow content, including links and other rich markup. Updating
or showing an error does not replace child nodes, change `textContent` or
parse a custom validity string as HTML.

The proposed `error::validation-message` pseudo-element contains only the
target's plain-text message. It is generated before the author children, after
`::before`, and inherits from `<error>`. This follows [`::placeholder`]'s
separation of browser-provided text from its containing element; the exact
property set needs CSS WG agreement.

Generated text and child content appear together when the error is visible.
Authors can style or hide `::validation-message` independently when child
content provides a complete replacement. Hiding it does not clear the error or
unblock submission. Replacement content must describe the current failure,
including built-in constraints if the target has them.

The accessible error relationship should describe the visible error without
duplicating hidden generated text. Rich children must also retain their normal
semantics and navigation; exposing only a flattened description is insufficient
for links or buttons. This does not require a new error-specific focus API.

`::content` would be too broad a name for just the generated validation text.
`::validation-message` is a working name, distinct from the element-backed
`::error` selector.

### Message language and privacy

Built-in messages must use the `<error>` element's resolved language, including
inherited `lang`, following the [localised `<time>` proposal]. Changing that
language updates the generated message, including while hidden. Formatting and
fallback for missing, unknown or unsupported languages must not depend on the
user's browser or operating system locale or expose that preference.

Custom messages remain unchanged; authors choose their language. The existing
control `validationMessage` getter is unchanged except for the proposed
fieldset eligibility change.

### The `::error` pseudo-element

`input::error` selects the input's associated `<error>`, or nothing if none
exists. It generates no box and changes neither tree position nor inheritance.
Control state can drive its styles, as in the opening example.

`::error` applies to every eligible `for` target, including `fieldset` and
`form`. These containers can be invalid because of their controls, but their
generated error message is only their own custom error. The CSS distinction
needed to show authored container errors remains open.

## Examples

### Default message, custom styling

```html
<form>
  <label for="name">Name (required)</label>
  <error for="name"></error>
  <input id="name" name="name" required minlength="2">
  <button>Send</button>
</form>
```

Under the proposed defaults, submitting an empty field shows the browser's
localised required-field message. The generated text updates with validity
while CSS controls visibility. `checkValidity()` alone does not reveal it.

### Repeated fields and rich content

The group owns the mismatch error. Individual fields retain their required-field
errors and native reporting.

```html
<form>
  <fieldset id="passwords">
    <legend>Choose a password</legend>
    <error for="passwords" class="password-mismatch">
      <strong>The passwords do not match.</strong>
      <a href="#password">Return to the first password</a>.
    </error>
    <label>Password (required) <input id="password" type="password" required></label>
    <label>Repeat password (required) <input id="confirmation" type="password" required></label>
  </fieldset>
  <button>Create account</button>
</form>

<style>
  error.password-mismatch::validation-message { display: none; }
</style>

<script type="module">
  const group = document.querySelector("#passwords");
  const password = document.querySelector("#password");
  const confirmation = document.querySelector("#confirmation");

  const update = () => {
    const mismatch = password.value && confirmation.value &&
      password.value !== confirmation.value;
    group.setCustomValidity(mismatch ? "The passwords do not match." : "");
  };

  password.addEventListener("input", update);
  confirmation.addEventListener("input", update);
  update();
</script>
```

This example sets the group's own error without changing either input's
validity. The container-state distinction described above still needs a
solution before the default CSS can show this message reliably.

With that reporting behaviour resolved, the custom string would remain
available for native reporting when no `<error>` is bound. This binding hides
the generated string in favour of rich child content. Removing the CSS rule
would show the generated message before the children. The equality check
remains author script, not a new constraint on `<input>`.

### A form-wide rejection

```html
<form id="booking">
  <error for="booking">
    <a href="/availability">Check other available dates</a>.
  </error>
  <button>Book</button>
</form>
```

After a server rejection, script could call
`form.setCustomValidity("This booking is no longer available.")` followed by
`form.reportValidity()`. The intended result is to show the message and link,
even if every individual field is valid. This depends on resolving form
reporting state and container error presentation.

### Server-side error

The endpoint returns a JSON `message` for a rejected username. Script sets it
as a custom error and clears it on the next edit to allow resubmission.

```html
<form id="signup" action="/signup" method="post">
  <label for="username">Username (required)</label>
  <error for="username"></error>
  <input id="username" name="username" required>
  <button>Sign up</button>
</form>
<p id="signup-status" role="status"></p>

<script type="module">
  const form = document.querySelector("#signup");
  const input = document.querySelector("#username");
  const status = document.querySelector("#signup-status");

  input.addEventListener("input", () => input.setCustomValidity(""));

  form.addEventListener("submit", event => {
    event.preventDefault();
    const data = new FormData(form, event.submitter);
    const submittedValue = input.value;

    const work = (async () => {
      const response = await fetch(form.action, { method: "POST", body: data });
      if (!response.ok) {
        const { message } = await response.json();
        if (input.value === submittedValue) {
          input.setCustomValidity(message);
          input.reportValidity();
        }
      } else {
        status.textContent = "Signed up.";
      }
    })();

    form.trackSubmission(work);
    work.catch(() => {
      status.textContent = "Could not submit. Try again.";
    });
  });
</script>
```

The provisional method in [form submission state] tracks the request. Comparing
values avoids applying a stale response after an edit.

`<error>` does not clear custom validity. A stale error blocks resubmission.
Response formats remain author-defined, and script is still needed to set
custom errors; server-rendered custom-error state is not solved here.
[FormValidation] separately covers server-side HTML constraints, not business
rules such as username availability.

## Caveats, shortcomings, and other drawbacks of design choices

- Unsupported browsers render `<error>` as an unknown inline element. Empty
  elements leave native bubbles unchanged; authored fallback text is visible.
  Fieldset custom errors do not block submission in existing browsers, and the
  proposed form API needs feature detection and a scripted fallback.
- Generated messages remain single strings, not structured lists of failed
  constraints. Rich child content does not make those strings HTML.
- The element does not apply external results, including `FormValidation`, to
  controls.

## Issues needing solutions

The presentation model uses existing CSS, but the following platform behaviour
is not settled:

1. **Container error state.** Define how CSS distinguishes a fieldset or form's
   own reported custom error from invalid descendants. The default
   `:user-invalid::error` rule must not reveal a group mismatch merely because
   an individual field is missing.
2. **Reporting-state integration.** Specify how `reportValidity()`, validated
   submission, correction and reset interact with `:user-invalid`, including
   the proposed fieldset and form APIs. Keep `checkValidity()` non-reporting;
   do not add a separate `<error>` reveal flag.
3. **Accessible mapping.** Distinguish failing constraints from having a
   reported error in the exposed invalid state. Define when the native error
   relationship is active and how it respects visibility and author ARIA.
   Hiding `<error>` alone does not define the target's accessible state.
4. **Announcements.** Coordinate focus-based reporting and message updates so
   failed submission, asynchronous errors and corrections are discoverable
   without duplicate or competing announcements. Do not assume every error
   is an assertive live region or every correction needs a success message.
5. **Container discovery and focus.** Define how users navigating individual
   controls discover a fieldset error without falsely marking each control
   invalid. Specify where form-wide errors are reported without focusing an
   arbitrary field. Automatic summaries remain out of scope.
6. **Native reporting and cancellation.** Decide whether an associated but
   author-hidden error suppresses the native bubble, and what cancelling
   `invalid` does during interactive reporting. Mere connectedness does not
   establish that a replacement provides a usable message. This also
   interacts with the `report` event proposal in #9878.
7. **Validation API compatibility.** Investigate existing uses of fieldset
   custom errors before making them block submission. Confirm the form API's
   shape and interaction with named properties.

## Further open questions

1. **Multiple associations.** Several `<error>` elements for one control,
   or one for several controls (such as a radio group or a date range).
2. **Shadow DOM.** Whether `for` can cross shadow boundaries, and how
   `::error` reaches an `<error>` in a different tree from the control.
3. **Form-associated custom elements.** How the validation anchor passed to
   `setValidity()` relates to the `<error>` position, and whether custom
   elements need a hook to render their own message.
4. **CSS integration.** Specify how `::error` selectors participate in the
   cascade alongside selectors that match the error element directly, without
   changing its inheritance or tree position.
5. **Generated content and rich children.** Agree the generated-message
   pseudo-element's supported properties and placement, and avoid validation
   cycles from controls inside an error.
6. **Names.** `<error>`, `::error` and `::validation-message` are working names.

## Draft specification

1. Define `<error>` with flow content, `for` and an IDL reflector. Initially
   resolve IDs in the same tree to one of the target types listed above;
   multiplicity and shadow boundaries remain open.
2. Make enabled fieldsets eligible for their own custom constraint validation.
   Include form-owned fieldsets in form validation without recursively
   validating their descendants through the fieldset.
3. Add form custom validity storage, `setCustomValidity()` and
   `validationMessage`. Include the form's own error in its validation methods,
   validated submission, `invalid` dispatch and validity pseudo-classes.
4. Define default presentation using `visibility: hidden` and `:user-invalid`.
   Preserve author control over visibility. Specify reporting-state integration
   and the container-state distinction before extending this default to
   fieldsets and forms; these are unresolved requirements, not new reveal flags.
5. Keep `checkValidity()` non-reporting. Integrate associated errors with
   interactive reporting, subject to the unresolved hidden-replacement,
   cancellation and container-focus rules above.
6. Preserve author children and generate the target's current message as plain
   text in `::validation-message` before them. Generate and lay out this content
   independently of reporting state, including when `visibility: hidden`.
7. Localise built-in text using the element's resolved language and update on
   language changes. Fallback must not depend on user locale; custom messages
   remain unchanged.
8. Define the accessible relationship, invalid state and announcement behaviour
   with ARIA and HTML-AAM. Preserve rich child semantics and avoid duplicate
   announcements. The requirements above need solutions before specification
   text can settle this behaviour.
9. Integrate form reset with reporting state, including for the form itself
   and its fieldsets. Do not clear custom validity.
10. Define `::error` on all eligible target types to select the associated
    element without generating a box or changing its position or inheritance.
11. Define `::validation-message` on `<error>`, including its generated position,
    inheritance and supported styling properties.

## Incubation and/or standardization destination

WHATWG HTML, starting at [Stage 0][WHATWG Stages], with the `::error` and
`::validation-message` selectors taken to the CSS WG. The Open UI resolution on
[#802][OpenUI #802] makes Open UI a venue for research on the design.

[Keith Cirkel]: https://github.com/keithamus
[Web Vision]: https://www.mozilla.org/en-US/about/webvision/full/#thedeclarativeweb
[constraint validation]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#constraints
[`validationMessage`]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#dom-cva-validationmessage
[`setCustomValidity()`]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#dom-cva-setcustomvalidity
[`checkValidity()`]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#dom-cva-checkvalidity
[`reportValidity()`]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#dom-cva-reportvalidity
[statically validate the constraints]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#statically-validate-the-constraints
[interactively validate the constraints]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#interactively-validate-the-constraints
[OpenUI #802]: https://github.com/openui/open-ui/issues/802
[whatwg/html #9878]: https://github.com/whatwg/html/issues/9878
[FormValidation]: form-validation.md
[form submission state]: form-submission-state.md
[WHATWG Stages]: https://whatwg.org/stages
[localised `<time>` proposal]: https://github.com/whatwg/html/issues/12591
[`fieldset`]: https://html.spec.whatwg.org/multipage/form-elements.html#the-fieldset-element
[`form`]: https://html.spec.whatwg.org/multipage/forms.html#htmlformelement
[`::placeholder`]: https://drafts.csswg.org/css-pseudo-4/#placeholder-pseudo
