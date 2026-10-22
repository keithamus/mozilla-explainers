# Promise-backed form submission state explainer

Author: [Keith Cirkel]

## Summary

Register a scripted submission's promises with its form. The form stays
submitting until native submission and every registered promise have settled,
even if one rejects.

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
- Keep cancellation explicit. Tracking work must not send a request or cancel
  the default action.
- Track pending work independently of success or failure.
- Reuse promises without adding result, transport, timeout or cancellation APIs.
- Share the native and scripted lifecycle with [conditional disabling].

## Prior features and proposals

- HTML's [form submission algorithm] validates controls, fires `submit`, builds
  the payload and submits. Listeners can cancel it but cannot register
  replacement asynchronous work.
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

## Motivation for this explainer

[JSON form encoding] and [FormValidation] address payloads and constraints.
Associating promises with the form would let script share native pending state
without making those APIs responsible for transport.

## Proposed solution

Add a promise-taking method to `HTMLFormElement`. This explainer uses
`trackSubmission(promise)` as a working spelling; the name is not settled.

It registers work without cancelling submission, validating, constructing a
request or interpreting the promise's value. Replacing native submission still
requires `preventDefault()`.

### Submission lifetime

Pending work includes the native submission and each registered promise.
Cancelling the default action completes only the native contribution; otherwise
native and registered work both contribute.

The form remains submitting until **all contributions have settled**. Rejection
ends only that promise's contribution; it neither clears other work nor cancels
or retries requests. Fulfilment values and rejection reasons do not affect this
state. An auxiliary failure can therefore be shown while an order POST keeps
the form busy.

This follows `Promise.allSettled()` rather than `Promise.all()`, without adding
a results API. Callers retain their promises and handle results and failures.
The examples register synchronously during `submit`; the allowed registration
window remains open.

### Preserving the submission entry list

[Conditional disabling] must preserve the triggering submission's payload.
Native submission builds it after `submit`, so conditional disabling must wait
until it has been captured. Disabling earlier would omit controls' values.

When script cancels native submission, conditional disabling applies after
`submit` dispatch. A listener can first capture
`new FormData(form, event.submitter)` synchronously. Ordinary disabling by a
listener still affects the payload under HTML's existing rules. This proposal
neither freezes fields nor captures data for script.

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

## Caveats, shortcomings, and other drawbacks of design choices

- Slow or hung work delays re-enabling controls, even after an error. Register
  only work whose lifetime should affect submission.
- Fetch fulfilment does not establish HTTP success. Authors interpret responses.
- Tracking does not restore native navigation, history, focus or rendering.
- Native completion must cover downloads, HTTP 204, cancellation and other
  target contexts, not just document replacement.
- Restoration must distinguish obsolete native work from still-relevant script
  work; that lifecycle remains open.
- Completion guarantees neither exactly-once execution nor server rollback.

## Open questions

1. **Method name and return value.** The working spelling is not a settled API.
2. **Registration window.** Must registration occur during synchronous `submit`
   dispatch, or can script register work outside it? An `async` listener's
   returned promise alone must not implicitly register anything.
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

## Draft specification

Associate native work and registered promises with the form. It is submitting
while any contribution is pending; completing one must leave others intact.

Failed validation creates no native contribution. Cancelling `submit` completes
only the native contribution. Reset changes controls without cancelling work
or settling registered promises. Apply conditional disabling after entry capture.

Native completion, registration windows and overlapping attempts must be
resolved before this becomes normative.

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
