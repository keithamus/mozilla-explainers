# Promise-backed form submission state explainer

Author: [Keith Cirkel]

## Summary

Register a scripted submission's promises with its form. The form stays
submitting until native submission and every registered promise have settled,
even if one rejects.

An opt-in hold delays native submission until asynchronous checks finish. This
lets an author check username availability through a separate endpoint, then
continue the original submission without replacing it with `fetch()`.

```js
const form = document.querySelector("form");
form.addEventListener("submit", event => {
  event.preventDefault();
  const body = new FormData(form, event.submitter);
  const work = fetch(form.action, { method: "POST", body })
    .then(response => response.text());
  form.trackSubmission(work);
  work.catch(console.error);
});
```

`trackSubmission()` is a provisional name. The shared pending state drives
[conditional disabling]; cancellation and result handling remain the listener's
job.

## User problems to be solved

A scripted submission needs pending state so authors can disable controls and
show progress until it finishes. Cancelling native submission to use `fetch()`
disconnects that work from the form's lifecycle; the browser cannot infer
which promises belong to it or whether response processing is included.

Several operations may contribute. Re-enabling controls when one finishes
allows another submission while other work is still pending, even if the
first operation failed.

## Methodology for approaching & evaluating solutions

- Let script share submission state, preserving validation, submitter selection
  and the `submit` event.
- Keep ordinary tracking independent of the default action. Only an explicit
  hold may delay or prevent native submission.
- Track pending work independently of success or failure.
- Let asynchronous validation gate native submission without restarting it.
- Reuse promises without adding result, transport, timeout or cancellation APIs.
- Share the native and scripted lifecycle with [conditional disabling].

## Prior features and proposals

- HTML's [form submission algorithm] validates controls, fires `submit`, builds
  the payload and submits. Listeners can cancel it but cannot register
  asynchronous checks that delay its continuation.
- `requestSubmit()` runs validation and fires `submit`; `submit()` bypasses
  both. Neither registers an existing promise.
- An `async` listener's returned promise does not extend event dispatch or
  delay the default action.
- [`Promise.allSettled()`] waits for every input, whether fulfilled or rejected.
- Script can toggle `disabled` or `aria-busy`, but the browser cannot link that
  state to submission.

## Flaws or limitations in existing features/proposals

`fetch()` identifies neither the form nor its submitter. Fulfilment makes the
response available, but body consumption and page updates may still be pending.
A local `finally` handler can restore controls too early; combining promises
works only when one caller owns every operation.

Asynchronous validation currently requires cancelling submission, checking the
result, then submitting again. `requestSubmit()` repeats validation and the
`submit` event, requiring a guard against another check; `submit()` skips both.

## Motivation for this explainer

[JSON form encoding] and [FormValidation] address payloads and constraints.
Associating promises with the form would let script share native pending state
without making those APIs responsible for transport.

## Proposed solution

Add a promise-taking method to `HTMLFormElement`. This explainer uses
`trackSubmission(promise, { hold: false })` as a working spelling; the name is
not settled.

Ordinary tracking registers work without delaying or cancelling submission,
validating, constructing a request or interpreting the promise's value.
Replacing native submission still requires `preventDefault()`.

With `{ hold: true }`, the promise also gates the current native submission.
It must be registered synchronously during that attempt's `submit` dispatch;
registration outside that window throws `InvalidStateError`. Returning a
promise from an event listener does not register a hold.

### Submission lifetime

Pending work includes the native submission and each registered promise.
Cancelling the default action completes only the native contribution; otherwise
native and registered work both contribute.

The form remains submitting until **all contributions have settled**. Rejection
ends that promise's contribution without clearing other pending work or
cancelling requests already sent. For ordinary tracking it has no effect on
native submission; a rejected hold prevents that attempt from proceeding.

The pending state follows `Promise.allSettled()` rather than `Promise.all()`,
without adding a results API. Callers retain their promises and handle results
and failures. Fulfilment values and rejection reasons do not determine pending
state. An auxiliary failure can therefore be shown while an order POST keeps
the form busy.

### Holding native submission

After an uncancelled `submit` dispatch, pause before constructing the payload
if any holds were registered. No submission request or navigation starts while
the attempt is held. Ordinary tracked promises do not delay this continuation.

Continue only after every hold fulfils. First rerun interactive constraint
validation unless validation is bypassed by `novalidate` or the submitter's
`formnovalidate`. This lets a check set a control's custom validity and fulfil
normally, using the browser's validation UI for an unavailable username.
An invalid result ends the attempt without sending its submission request.
Fulfilment values, including `false`, are not interpreted as validation results.

Resume the original attempt without redispatching `submit` or calling
`requestSubmit()`. Keep its submitter and apply that submitter's submission
overrides through the native algorithm. Recheck whether the form can navigate
after waiting.

Any rejected hold prevents the attempt from continuing, including when a
validation request fails. The native contribution ends, but every remaining
promise still contributes until it settles. `preventDefault()` also ends the
native contribution; no later settlement may revive a cancelled attempt.

A hold begins pending state when registered. [Conditional disabling] applies
after `submit` dispatch, before waiting for holds. Controls covered by the
keyword cannot be edited by the user during the hold; uncovered controls and
script can still change values. Authors must reject stale checks rather than
submit a new value using an earlier value's result. The precise boundary
between validation, edits and payload capture remains an open question.

### Preserving the submission entry list

[Conditional disabling] applies after `submit` dispatch, including while holds
are pending. If `submit()` bypasses dispatch, apply it at the corresponding
point before native entry construction. It prevents interaction but must not
remove controls from the triggering native submission's payload or post-hold
constraint validation. Those operations ignore disabling caused solely by
`while-submitting`, including disabling inherited from a conditional fieldset
or form.

Ordinary disabling still applies. A control independently disabled by script
or an ordinary disabled fieldset remains excluded under HTML's existing rules.
Other eligibility rules, such as which submitter contributes an entry, are
unchanged. This is not a snapshot; capture current values after holds and
repeated validation, without temporarily re-enabling controls.

When script cancels native submission, conditional disabling also applies after
`submit` dispatch. A listener can first capture
`new FormData(form, event.submitter)` synchronously. The exception above is for
the triggering native attempt, not arbitrary later `FormData` construction or
validation API calls, which still observe effective disabled state.

Repeated validation must report errors on conditionally disabled controls.
It does not enable or focus a disabled control while another contribution
keeps the form submitting. Once the submission lifetime ends, conditional
disabling ends and the user can correct the fields.

### Errors and cancellation

Registration must not silently swallow rejections or require applications to
suppress them while other work continues. Rejection reporting remains open.

A never-settling promise keeps its contribution pending; there is no timeout.
Authors can cancel supported operations and register promises reflecting their
completion, but settlement alone does not prove that side effects have stopped.

## Examples

### Replacing native submission

This uses the proposed method and [conditional disabling] with ordinary
multipart data, not JSON. The promise covers response consumption and page updates.

```html
<form id="profile" action="/profile" method="post">
  <label for="display-name">Display name</label>
  <input id="display-name" name="displayName" required>
  <button name="intent" value="save" disabled="while-submitting">Save</button>
</form>
<p id="save-status" role="status"></p>

<script type="module">
  const form = document.querySelector("#profile");
  const status = document.querySelector("#save-status");

  form.addEventListener("submit", event => {
    event.preventDefault();
    const data = new FormData(form, event.submitter);
    status.textContent = "Saving.";

    const work = (async () => {
      const response = await fetch(form.action, {
        method: "POST",
        body: data,
      });
      if (!response.ok) {
        throw new Error(`Request failed (${response.status})`);
      }
      status.textContent = await response.text();
    })();

    form.trackSubmission(work);
    work.catch(() => {
      status.textContent = "Could not save. Try again.";
    });
  });
</script>
```

### Choosing a composite lifetime

An author can register one composite promise for work they own:

```js
form.trackSubmission(Promise.all([upload, save]));
```

If the composite rejects early, its contribution ends while its operations may
continue. Other registered or native work remains pending independently. To
track both operations through settlement, register them separately or use
`Promise.allSettled([upload, save])`.

For holds, register checks separately if each rejection must prevent
submission. `Promise.allSettled()` always fulfils, so using it as a hold would
not propagate a failed check to the gate.

### Checking a username before native submission

This example assumes a form with a named `username` input and a
`#username-status` status element. The validation endpoint returns JSON with a
`message` string, empty when the username is available.

```js
const form = document.querySelector("form");
const username = form.elements.namedItem("username");
const status = document.querySelector("#username-status");

username.addEventListener("input", () => {
  username.setCustomValidity("");
  status.textContent = "";
});

form.addEventListener("submit", () => {
  const value = username.value;
  status.textContent = "Checking username.";

  const check = (async () => {
    const query = new URLSearchParams({ username: value });
    const response = await fetch(`/validate-username?${query}`);
    if (!response.ok) {
      throw new Error(`Request failed (${response.status})`);
    }
    const { message } = await response.json();
    if (username.value !== value) {
      throw new Error("Username changed. Submit again.");
    }
    username.setCustomValidity(message);
    status.textContent = "";
  })();

  form.trackSubmission(check, { hold: true });
  check.catch(error => {
    status.textContent = error.message;
  });
});
```

No `preventDefault()` is needed. An available username permits native
submission; an unavailable username is reported by repeated constraint
validation. A failed request or a changed username rejects the hold. Clearing
custom validity on input allows a corrected value to reach the next `submit`
event instead of being blocked by the previous error.

The example checks for edits before applying the response. Edits after that
check and overlapping attempts still need the lifecycle rules discussed below.
The submission endpoint must validate again, since availability can change
between the check and submission.

## Caveats, shortcomings, and other drawbacks of design choices

- Slow or hung work delays re-enabling controls, even after an error. Register
  only work whose lifetime should affect submission.
- Fetch fulfilment does not establish HTTP success. Authors interpret responses.
- Ordinary tracking does not restore native navigation, history, focus or
  rendering after `preventDefault()`. Holds preserve native submission instead.
- Native completion must cover downloads, HTTP 204, cancellation and other
  target contexts, not just document replacement.
- Restoration must distinguish obsolete native work from still-relevant script
  work; that lifecycle remains open.
- Completion guarantees neither exactly-once execution nor server rollback.
- A hold can postpone native submission indefinitely. There is no automatic
  timeout or retry, so conditionally disabled controls can remain unavailable.
- Holds require review of delayed submissions to other targets, including
  interactions with transient user activation.

## Open questions

1. **Method name and return value.** The working spelling is not a settled API.
2. **Ordinary registration window.** Must ordinary tracking occur during
   synchronous `submit` dispatch, or can script register work outside it?
   Holds are restricted to synchronous dispatch for the current attempt.
3. **Overlapping attempts.** How should contributions be associated with attempts
   if a form is submitted again while work remains pending? A completion from
   an older attempt must not clear independently pending work from a newer one.
4. **Native completion.** Which existing navigation and download hooks release
   native contributions, including failed, cancelled, and no-navigation cases?
5. **Observability.** Is conditional disabling sufficient, or is a CSS selector
   or DOM getter for the shared state also needed?
6. **Promise rejection reporting.** How can browser observation preserve useful
   failure reporting without duplicating an application's own handling?
7. **Document lifecycle.** How do detachment, adoption, page replacement and
   back/forward-cache restoration affect registered work?
8. **Edits during a hold.** How should validation results be tied to the values
   eventually submitted, including edits after a check fulfils? Reset does not
   cancel work, so authors also need to guard against results made stale by reset.
9. **Delayed native actions.** How do holds interact with target selection,
   transient user activation and `method="dialog"`?

## Draft specification

Associate native work and registered promises with the form. It is submitting
while any contribution is pending; completing one must leave others intact.

Failed initial validation creates no native contribution. Cancelling `submit`
completes only the native contribution. Reset changes controls without
cancelling work or settling registered promises.

Accept holds only during synchronous `submit` dispatch and associate them with
that attempt. After uncancelled dispatch, wait for every hold to fulfil before
repeating applicable interactive validation and continuing native submission
without another `submit` event. A rejected hold or failed repeated validation
ends the native contribution without clearing pending promises.

Apply conditional disabling after the submission-event phase, before waiting
for holds or constructing entries, including when `submit()` skips that phase.
For the triggering native attempt's repeated validation and entry construction,
ignore only disabling caused by `while-submitting`; ordinary disabling and
other eligibility rules remain in force.

Native completion, ordinary registration windows, overlapping attempts and the
validation-to-payload boundary must be resolved before this becomes normative.

## Incubation and/or standardization destination

WHATWG HTML, alongside the form submission algorithms, through the [WHATWG
Stages] process. Develop the shared lifecycle with [conditional disabling].

[Keith Cirkel]: https://github.com/keithamus
[conditional disabling]: form-disabled-while-submitting.md
[JSON form encoding]: form-json-encoding.md
[FormValidation]: form-validation.md
[form submission algorithm]: https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#form-submission-algorithm
[`Promise.allSettled()`]: https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise.allsettled
[WHATWG Stages]: https://whatwg.org/stages
