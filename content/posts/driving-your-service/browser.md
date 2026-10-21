+++
title = 'Driving your service through a browser'
linkTitle = 'Browser'
weight = 30
tool = 'browser'
date = 2026-10-21T06:00:00+02:00
summary = "The browser client uses your web UI in a real Chromium, finding fields and buttons by the labels a person reads, and checking what the page shows."
tags = ['clients', 'browser', 'ui']
+++

{{< client-flow kind="browser" >}}

## The suite

```yaml
containers:
  chrome: { preset: chromium }        # a headless Chromium

resources:
  alice-browser: { type: browser, container: chrome, options: { base_url: http://localhost:3000 } }
```

```gherkin
Scenario: A todo added on the page shows up in the list
  Given "alice-browser" opens "/"
  When "alice-browser" fills "New todo" with "Water the tomatoes"
  And "alice-browser" clicks "Add"
  Then "alice-browser" sees "Water the tomatoes" within "3s"

Scenario: The page says why a todo was refused
  Given "alice-browser" opens "/"
  When "alice-browser" clicks "Add"
  Then "alice-browser" sees "title is required" within "3s"
```

The page this runs against has nothing special for tests:

```html
<form><label>New todo <input name="title"></label> <button>Add</button></form>
```

## How it finds things: by name, like a person

`fills "New todo"` and `clicks "Add"` take no CSS selectors. tomato looks
for the element whose **accessible name** is that text: the label of a field,
the text of a button or link, an `aria-label`. That's what a person reads and
a screen reader announces. So:

- **A test survives a redesign** that keeps the words.
- **A field without a label can't be found**, which is a bug a screen-reader
  user would hit first.
- **Exactly one element must match.** Two buttons called "Delete" fail the
  step and list both.

## What it can do

| To | Step |
|---|---|
| go somewhere | `opens "/path"` |
| fill in | `fills "Email" with "…"`, `types "…" at the end of "…"`, `presses "Enter"` |
| choose | `clicks "Add"`, `checks "Remember me"`, `chooses "Express"`, `selects "Austria" from "Country"` |
| check the page now | `sees "…"`, `does not see "…"`, `sees "Pay" is disabled`, `url is "/done"` |
| wait for a change | `sees "…" within "3s"`, `no longer sees "Saving…" within "5s"` |
| keep a picture | `saves screenshot "checkout"`, and one is saved anyway when a step fails |

## Notes

- **After a click, wait.** Pages update asynchronously, so check an outcome
  with `sees "…" within "3s"`, not `sees "…"`.
- **Two resources are two people.** `alice-browser` and `bob-browser` each get their own
  cookies and storage, in one Chromium: collaboration needs no special steps.
- **`localhost` just works.** The page in the container reaches your service
  on your machine through a tunnel tomato opens, so cookies, CORS and secure
  context behave as they would for a real user.
