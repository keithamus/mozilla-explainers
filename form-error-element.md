# The `<error>` element explainer

Author: [Keith Cirkel]

## User problems to be solved

Browsers report HTML [constraint validation] problems through browser-managed
UI, commonly a temporary bubble. Authors cannot place that message in their
page layout or style it with the page's CSS.

Authors who want persistent messages next to fields must read
[`validationMessage`], decide when to reveal it, and maintain an accessible
association with the control. They also need to update messages as validity
changes and clear reporting state on reset. Suppressing native reporting does
not provide any of that presentation.

The browser already supplies localised messages. Authors should be able to
reuse them in the page rather than write and translate replacements merely to
control placement.

## Methodology for approaching & evaluating solutions

We follow Mozilla's [Web Vision], which favours [declarative][Web Vision]
features that work by default and degrade well. A good solution should:

1. Show the browser's constraint validation messages in the page, with no script.
2. Keep validity in one place. The message element only displays validity.
3. Reveal messages at the points where the browser already validates, rather
   than adding new timing rules authors must learn.
4. Fall back to today's behaviour in browsers without support.
5. Be styled with ordinary CSS, without generated boxes or moving nodes.

## Prior features and proposals

**Validation bubbles.** [`reportValidity()`] and form submission run
[interactively validate the constraints], which reports problems to the user in
a UA-defined way. [`checkValidity()`] runs [statically validate the
constraints], which fires `invalid` events but does not report.

**[`setCustomValidity()`].** Sets a custom error on a control. The control
becomes invalid and `validationMessage` returns the custom string until it is
set to the empty string.

**[OpenUI #802].** Opened during `<selectlist>` work to ask whether validation
errors are part of a control's anatomy. The Open UI CG resolved that it is "a
good problem to solve, but it's independent of the `<selectlist>`. We should
solve this generally." Comments suggested a free fallback message plus an
author-placed element, with `<error for=IDREF_of_input>` given as an example,
and extending `<output>` as an alternative. A later comment sketched an
`::error` pseudo-element representing the bubble itself, styled through
anchor positioning and `content`.

**[whatwg/html #9878].** Asks for a hook that tells a control, especially a
form-associated custom element, when the browser is about to report its
validity, since `invalid` fires for both `checkValidity()` and
`reportValidity()`. It proposes a cancelable `report` event inside
interactively validate the constraints. Later comments propose declarative
per-constraint message elements, a target attribute pointing to a container,
and form attributes controlling which interactions trigger reporting.

## Flaws or limitations in existing features/proposals

- Browser-managed reporting cannot participate in the author's page layout.
- Cancelling `invalid` hides the bubble but leaves the author to rebuild
  display, association, updating and clearing.
- A `report` event, as in #9878, tells script when to show a message but still
  needs script to show it.
- Per-constraint message elements need separate message selection and
  precedence rules when several constraints fail.
- A pseudo-element representing a generated bubble does not provide an
  author-placed DOM element for the message.

## Motivation for this explainer

Browsers already compute a message and know when validation is requested. An
element that uses the associated control's existing validity state and message
selection adds placement and styling without defining constraints again.
Revealing on every explicit validation call also gives `checkValidity()` an
in-page presentation effect when the author opts in with `<error>`.

## Proposed solution

An `<error>` element with a `for` attribute naming a control by ID.

```html
<label for="email">Email</label>
<input id="email" type="email" required>
<error for="email"></error>
```

Behaviour:

1. **Binding.** `for` names the associated control by ID. The initial proposal
   associates each error element with one control; multiple associations and
   shadow boundaries remain open questions.
2. **Content.** When revealed, the element uses the same validity state and
   message selection as [`validationMessage`]. Built-in messages must use the
   `<error>` element's resolved language, as described below.
3. **Custom messages.** Custom or server text comes only from
   `setCustomValidity()` (or `ElementInternals.setValidity()` for custom
   elements). Text authored inside `<error>` does not create, assert, or change
   validity. There is no per-constraint message syntax and no new constraint.
4. **Quiet until validation.** The element shows nothing until constraint
   validation is explicitly requested for its control. Every such request
   reveals it: `checkValidity()`, `reportValidity()`, and form submission, on
   either the control or its form.
5. **Replacing the bubble.** When a control has a connected associated
   `<error>`, interactive validation shows the message there instead of in a
   bubble. HTML's existing focusing and scrolling of the invalid control is
   unchanged.
6. **Lifecycle (proposed, not settled).** Once revealed, the message updates
   live as validity changes, hides when the control becomes valid, and returns
   to the quiet state when the form is reset.
7. **Accessibility (proposed).** The association would expose the element as
   the control's error message to accessibility APIs, as `aria-errormessage`
   does. Role and announcement timing need agreement with the ARIA and
   HTML-AAM editors.

### Message language and privacy

Built-in messages must be localised using the `<error>` element's resolved
language, including an inherited `lang`, rather than the browser or operating
system's locale. This follows the [localised `<time>` proposal], which uses
the element's language for formatting.

Changing the resolved language must update a revealed built-in message.
Fallback for a missing, unknown or unsupported language must not depend on the
user's locale either. The rendered message must not expose that preference.
Custom messages from `setCustomValidity()` or `ElementInternals.setValidity()`
remain unchanged; authors are responsible for their language. This proposal
does not change the existing `validationMessage` getter.

### The `::error` pseudo-element

`input::error` selects the `<error>` element associated with that input. It
matches a real element instead of generating an error box. It does not
reparent the element or change inheritance, so the `<error>` stays in its tree
position and inherits from its own parent. If a control has no associated
`<error>`, `::error` matches nothing.

This lets state on the control drive the message's styles:

```css
input:invalid::error { color: red; }
input:user-invalid::error { font-weight: bold; }
```

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

This example assumes an endpoint that returns a JSON `message` when a username
is rejected. Script stores that message as a custom error. An input listener
clears it on the next edit so the next submission can reach the server.

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

The example uses the provisional method in [form submission state] to track
the request without changing the error element's role. The value comparison
avoids applying an old response to a field edited while the request was pending.

The `<error>` element does not clear custom errors itself. Leaving a stale
`setCustomValidity()` string in place keeps the control invalid and blocks the
next submission. The response shape is the author's; this proposal defines no
transport format. Pages without script cannot set a custom error, so a
server-rendered page has no way to put the control into a custom-error state.
This proposal does not solve that case. [FormValidation] separately proposes
running HTML's built-in constraints on the server; it does not supply business
rules such as username availability.

## Caveats, shortcomings, and other drawbacks of design choices

- Older browsers render `<error>` as an unknown inline element. Since it is
  empty until revealed, it shows nothing and the bubble appears as today.
  Authors who put fallback text inside it will see that text in old browsers.
- Revealing on `checkValidity()` changes what that method does from the
  user's point of view. A script calling it to test validity will now display
  messages for controls that have an `<error>`. Today it only fires `invalid`.
- `validationMessage` is one string and may describe one or more failed
  constraints. The element adds no structured list of individual failures.
- The element displays the control's validation message, localised as described
  above. It does not map the results of a `FormValidation` run, or any other
  external result, onto controls.

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

Proposed changes to the HTML Standard:

1. Define `<error>` with a `for` attribute and an IDL reflector. As an initial
   proposal, resolve the ID in the same tree to an eligible control; exact
   eligibility, multiplicity and shadow-boundary rules remain to be agreed.
2. Give each `<error>` element a *revealed* flag, initially false.
3. Extend control-level and form-level constraint validation, including
   `checkValidity()` and `reportValidity()`, to reveal associated errors.
   Whether cancelling `invalid` suppresses this reveal remains open.
4. In interactively validate the constraints, when reporting a problem for a
   control with a connected associated `<error>`, do not show the UA's own
   report for that control.
5. While revealed, render the associated control's validation message, using
   the `<error>` element's resolved language for built-in messages. Update it
   when that language changes. Language fallback must be independent of the
   user's browser or operating system locale. Preserve custom messages
   unchanged, and render nothing when there is no message.
6. On form reset, clear the revealed flag of every `<error>` associated with
   the form's controls.
7. In CSS, define `::error` on `input`, `textarea`, `select` and
   form-associated custom elements as selecting the associated `<error>`
   element, with no generated box and no change to the element's tree
   position or inheritance.

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
