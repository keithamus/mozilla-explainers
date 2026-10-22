# The `<error>` element explainer

Author: [Keith Cirkel]

## Summary

Place the browser's validation message next to a field, using an `<error>`
element instead of a validation bubble. It shows the message when validation
is requested, without script.

```html
<form>
  <label for="email">Email</label>
  <input id="email" name="email" type="email" required>
  <error for="email"></error>
  <button>Sign up</button>
</form>
```

Style that element directly, or select it through the control's state:

```css
input:invalid::error { color: red; }
input:user-invalid::error { font-weight: bold; }
```

## User problems to be solved

Browsers report HTML [constraint validation] problems in browser-managed UI,
commonly a temporary bubble. Authors cannot place it in their layout or style
it with the page's CSS.

Persistent in-page messages require script to read [`validationMessage`],
control its visibility, maintain an accessible association, update it as
validity changes and clear reporting state on reset. Authors should be able to
reuse the browser's localised messages without rebuilding that presentation.

## Methodology for approaching & evaluating solutions

Follow Mozilla's [Web Vision] with a declarative feature that:

1. Displays existing validation messages using ordinary page layout and CSS.
2. Leaves validity and message selection with the control.
3. Reveals messages on explicit validation requests.
4. Falls back to native reporting in unsupported browsers.

## Prior features and proposals

**Validation bubbles.** [`reportValidity()`] and form submission run
[interactively validate the constraints], which reports problems to the user in
a UA-defined way. [`checkValidity()`] runs [statically validate the
constraints], which fires `invalid` events but does not report.

**[`setCustomValidity()`].** Sets a custom error on a control. The control
becomes invalid and `validationMessage` returns the custom string until it is
set to the empty string.

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

An element could display the control's existing message without redefining
constraints. Opting in would also give `checkValidity()` in-page presentation.

## Proposed solution

1. **Binding.** `for` identifies one control by ID. Multiple associations and
   shadow boundaries remain open.
2. **Content.** Revealed errors use the control's validity and message selection
   as [`validationMessage`] does, localising built-in text to the error's language.
3. **Custom messages.** Only `setCustomValidity()` or
   `ElementInternals.setValidity()` supplies custom or server errors. Child
   text does not change validity; no new constraints or per-constraint syntax
   are added.
4. **Reveal.** The element shows nothing until explicit validation, including
   control- or form-level `checkValidity()`, `reportValidity()` and submission.
5. **Reporting.** A connected associated `<error>` replaces the native bubble,
   retaining HTML's focus and scroll behaviour.
6. **Lifecycle (proposed, not settled).** Revealed messages update with validity,
   hide when valid and return to unrevealed state on reset.
7. **Accessibility (proposed).** The association would expose the error-message
   relationship as `aria-errormessage` does. Role and announcements need agreement
   with ARIA and HTML-AAM editors.

### Message language and privacy

Built-in messages must use the `<error>` element's resolved language, including
inherited `lang`, following the [localised `<time>` proposal]. Changing that
language updates a revealed message. Formatting and fallback for missing,
unknown or unsupported languages must not depend on the user's browser or
operating system locale or expose that preference.

Custom messages remain unchanged; authors choose their language. The existing
`validationMessage` getter is unchanged.

### The `::error` pseudo-element

`input::error` selects the input's associated `<error>`, or nothing if none
exists. It generates no box and changes neither tree position nor inheritance.
Control state can drive its styles, as in the opening example.

`::error` applies to the same set of controls as `for` binding: `input`,
`textarea`, `select`, and form-associated custom elements.

## Examples

### Default message, custom styling

```html
<form>
  <label for="name">Name</label>
  <input id="name" name="name" required minlength="2">
  <error for="name"></error>
  <button>Send</button>
</form>
```

Submitting an empty field shows the browser's localised required-field message.
Under the proposed live-update lifecycle, typing one character changes it to
the `minlength` message and a second character hides it.

### Server-side error

The endpoint returns a JSON `message` for a rejected username. Script sets it
as a custom error and clears it on the next edit to allow resubmission.

```html
<form id="signup" action="/signup" method="post">
  <label for="username">Username</label>
  <input id="username" name="username" required>
  <error for="username"></error>
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
- `checkValidity()` would reveal associated errors, where today it only fires
  `invalid`. Scripts using it to test validity would change visible behaviour.
- Messages remain single strings, not structured lists of failed constraints.
- The element does not apply external results, including `FormValidation`, to
  controls.

## Open questions

1. **`invalid` cancellation.** Should cancelling `invalid` keep the `<error>`
   quiet, as it suppresses the bubble today, or does `<error>` presence take
   over? This interacts with the `report` event idea in #9878.
2. **Multiple associations.** Several `<error>` elements for one control,
   or one for several controls (such as a radio group or a date range).
3. **Shadow DOM.** Whether `for` can cross shadow boundaries, and how
   `::error` reaches an `<error>` in a different tree from the control.
4. **Form-associated custom elements.** How the validation anchor passed to
   `setValidity()` relates to the `<error>` position, and whether custom
   elements need a hook to render their own message.
5. **Lifecycle details.** Whether live updates and reset clearing are right
   defaults, and whether authors need a way to opt into interaction-driven
   reveal before submission.
6. **Author content.** What happens to child content of `<error>` when it is
   revealed: replaced, shown alongside the message, or exposed through a slot.
7. **Accessibility mapping.** Role, error-message relationship, and
   announcement timing.
8. **Name.** `<error>` and `::error` are working names.
9. **CSS integration.** Specify how `::error` selectors participate in the
   cascade alongside selectors that match the error element directly, without
   changing its inheritance or tree position.

## Draft specification

1. Define `<error>` with `for` and an IDL reflector. Initially resolve IDs in
   the same tree; eligibility, multiplicity and shadow boundaries remain open.
2. Add a *revealed* flag, initially false.
3. Reveal on control- or form-level validation, including `checkValidity()` and
   `reportValidity()`. The effect of cancelling `invalid` remains open.
4. When a control has a connected associated `<error>`, use it instead of native
   interactive reporting for that control.
5. While revealed, display the control's validation message, or nothing if
   absent. Localise built-in text using the element's resolved language and
   update on language changes. Fallback must not depend on user locale; custom
   messages remain unchanged.
6. Clear associated revealed flags on form reset.
7. Define `::error` on `input`, `textarea`, `select` and form-associated custom
   elements to select the associated element without generating a box or
   changing its position or inheritance.

## Incubation and/or standardization destination

WHATWG HTML, starting at [Stage 0][WHATWG Stages], with the `::error` selector
taken to the CSS WG. The Open UI resolution on [#802][OpenUI #802] makes Open
UI a venue for research on the design.

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
