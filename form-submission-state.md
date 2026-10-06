# Promise-backed form submission state explainer

Author: [Keith Cirkel]

## User problems to be solved

A form submitted with script still needs to tell users that their submission is
in progress. Authors disable buttons, display progress, and restore the controls
when the operation finishes. Each application has to maintain this state itself.

A submission can involve several asynchronous operations. Re-enabling a button
when one operation finishes can let the user submit again while another request
is still running. An error in an auxiliary operation does not mean that a
state-changing request has stopped.

Cancelling the browser's default submission to use `fetch()` also disconnects
that work from the form's native submission lifecycle. The browser cannot infer
which promise represents the submission, or whether processing the response is
part of it.

## Methodology for approaching & evaluating solutions

- Let script participate in the form's submission state without replacing
  validation, submitter selection, or the `submit` event.
- Keep cancellation explicit. Tracking a promise must not send a request or
  cancel the browser's default action.
- Track pending work independently of its outcome. Reporting an error must not
  require pretending that other pending work has finished.
- Reuse promises rather than introduce a submission-result format, transport,
  timeout policy, or cancellation mechanism.
- Share the lifecycle with [conditional disabling], rather than define separate
  native and scripted meanings of submitting.

## Prior features and proposals

- HTML's [form submission algorithm] validates controls, fires `submit`, builds
  the payload, and performs the native submission. A listener can cancel the
  default action, but cannot register replacement asynchronous work with it.
- `requestSubmit()` initiates a submission through validation and the `submit`
  event. `submit()` bypasses those steps. Neither registers an application's
  existing promise.
- An event listener can be `async`, but its returned promise does not extend
  event dispatch or make the default action wait.
- [`Promise.allSettled()`] observes when every input promise has settled,
  independently of whether it fulfilled or rejected.
- Authors can implement pending state in script and use it to set `disabled` or
  `aria-busy`. The application maintains that state, and the browser has no way
  to link it to the form's submission lifecycle.

## Flaws or limitations in existing features/proposals

A `fetch()` request does not identify a form, its submitter, or the point at
which the application has finished applying its response. Fetch fulfilment
itself only makes the response available; consuming the body and updating the
page can still be pending.

A local `finally` handler can restore a button while work registered elsewhere
is still pending. Combining promises in application code works when one caller
owns every operation, but not when independent callers contribute work.

## Motivation for this explainer

The [JSON form encoding] and [FormValidation] explainers address payloads and
constraints. They do not tell the browser how long a script-handled submission
lasts. Associating promises with the form lets native submission and script use
the same pending state, without making either encoding or validation responsible
for transport.

## Proposed solution

Add a promise-taking method to `HTMLFormElement`. This explainer uses
`trackSubmission(promise)` as a working spelling; the name is not settled.

The method registers work with the form's submission state. It does not call
`preventDefault()`, perform validation, construct a request, or interpret the
promise's value. Authors that replace native submission must still cancel the
`submit` event explicitly.

### Submission lifetime

A form's pending work can include:

- the browser's native submission; and
- each promise registered through the method.

Cancelling the native default action treats the native contribution as
completed. Registered promises remain pending even when that action is
cancelled. If the native action proceeds, both native and registered work
contribute to the lifetime.

The form remains submitting until **all contributions have settled**. A rejected
promise completes only its own contribution. It does not clear other pending
work, cancel requests, or cause the browser to retry anything. The promise's
fulfilment value and rejection reason have no effect on this bookkeeping.

For example, if an auxiliary request rejects while an order POST is pending,
the form remains submitting until the POST also finishes. The application can
show the auxiliary error immediately; presenting an error does not have to wait
for busy state to clear.

The lifetime follows `Promise.allSettled()` rather than `Promise.all()`. It does
not expose the array of settlement results as a new form API. Callers already
have their own promises and remain responsible for presenting results and
handling failures.

The examples register work synchronously during `submit` dispatch. The allowed
registration window, and whether registration outside an attempt is useful,
remain open questions.

### Preserving the submission entry list

[Conditional disabling] must not change the payload of the submission that
caused it. In native HTML submission, the payload is built after the `submit`
event. Disabling a control before then would omit its `name` and `value`;
disabling a fieldset or form could omit a whole group of values.

The proposed ordering therefore defers conditional disabling until the current
native payload has been captured. If script cancels native submission,
conditional disabling takes effect after `submit` dispatch; script can capture
`new FormData(form, event.submitter)` synchronously in its listener first.

This does not change ordinary disabling performed explicitly by a listener,
which continues to affect the payload under HTML's existing rules. It also does
not freeze fields or automatically capture data for a scripted request.

### Errors and cancellation

Tracking work is separate from handling its result. Applications must not have
to suppress a rejection to keep the form submitting while other work continues.
How rejections are reported once the browser observes the promise still needs
specifying, but registering work must not silently swallow failures.

A never-settling promise keeps its contribution pending. The method does not add
a timeout. Authors can cancel the underlying operation where it supports
cancellation and register a promise that reflects the resulting completion.
Settling a promise alone does not prove that every side effect it started has
stopped.

## Examples

### Replacing native submission

This example uses the proposed method and [conditional disabling]. The promise
includes consuming the response and updating the page. The endpoint accepts
ordinary multipart form data; JSON encoding is not required.

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

The explicit `preventDefault()` prevents a second, native request. Without it,
tracking the promise would leave native submission enabled as well.

### Choosing a composite lifetime

An author can register one composite promise for work they own:

```js
form.trackSubmission(Promise.all([upload, save]));
```

If that composite rejects early, its contribution ends even if its constituent
operations continue. Any independently registered promises or native submission
still keep the form submitting. To track both operations through settlement,
register them separately or register `Promise.allSettled([upload, save])`.

## Caveats, shortcomings, and other drawbacks of design choices

- Slow or hung registered work can delay re-enabling buttons after an error.
  Only work whose lifetime should affect submission belongs in this set.
- A fulfilled `fetch()` promise does not mean an HTTP success response. Authors
  decide what constitutes success and how to display a response.
- This method does not restore the navigation, history, focus, or response
  rendering behaviour lost when an application replaces native submission.
- Native completion is not equivalent to unloading the source document.
  Downloads, HTTP 204 responses, cancellation, and submissions targeting another
  browsing context need completion paths that release the contribution.
- Page replacement and restoration must not leave a restored form busy with
  obsolete native work. They must not accidentally clear still-relevant script
  work either; that lifecycle needs further definition.
- Ending pending state does not establish that a server did not process a
  request. The API provides neither exactly-once execution nor server rollback.

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

Extend HTML's submission algorithms with native-work bookkeeping, and associate
registered promises with the form. Derive submitting state from whether any
contribution remains pending. Marking any contribution completed, fulfilled or
rejected must leave all other pending contributions intact.

A validation failure before `submit` creates no new native pending contribution.
Cancelling `submit` completes its native contribution without removing
registered promises. A reset changes controls; it does not cancel requests or
settle registered promises.

Apply conditional-disabled effects only after the entry-list boundary described
above. Native submission remains possible without any registered promises;
script-only submission remains pending without a native contribution.

The native completion hooks, registration window, and overlapping-attempt rules
must be resolved before this outline becomes a normative algorithm.

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
