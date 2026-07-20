# IT Assets Counter Question — Design

## Context

`index.html` is a single-page, JS-driven questionnaire (typeform-style). Questions
live in the `questions` array; each entry declares a `type` that `buildQuestion()`
renders and `collect()`/`restore()` know how to read/write. Section 5 ("Technical
Assets") currently ends with Q22, an `assets`-type question with six free-text
boxes (End Point Devices, Applications, Networks, Physical Assets, Database &
Locations, Servers).

## Goals

1. Add a new question, Q22b, asking **"How many IT assets are included in the
   scope?"** with a numeric stepper per category: Endpoint Devices, Applications,
   Networks, Database, Cloud.
2. Fix Q22's "Physical Assets" placeholder, which currently reads "Tables,
   Chairs, TV, Projectors" (furniture, not IT/security-relevant) to instead read
   "Laptops, Firewalls, Desktops, External Hard Drives, Server Rooms, Physical
   Cabling".

## Placement

Q22b is inserted immediately after Q22 in the `questions` array (before
`thankyou`), staying in section 5 ("Technical Assets"). This mirrors the
existing Q21/Q21b pattern (a follow-up sub-question in the same section).
`C_END`/`TOTAL` are derived from `questions.length`, so no other index math
needs to change.

## New question type: `counter`

None of the existing types (`text`, `textarea`, `email`, `tel`, `number`,
`date`, `single-choice`, `multi-choice`, `assets`) support a labeled numeric
stepper per item, so this adds a new `type: 'counter'` with an `items` list:

```js
{ id:'q22b', num:'22b', sec:5, label:'IT Assets in Scope (Count)',
  text:'How many IT assets are included in the scope?',
  type:'counter', req:true,
  items:[
    { k:'endpoints',    label:'Endpoint Devices' },
    { k:'applications', label:'Applications' },
    { k:'networks',     label:'Networks' },
    { k:'databases',    label:'Database' },
    { k:'cloud',        label:'Cloud' },
  ]}
```

### Rendering (`buildCounter`)

Each item renders as a row: a lettered badge (A/B/C/D/E, reusing the visual
style of `.choice-key-badge` used on choice questions), the item label, and a
stepper (`− [count] +`) starting at 0. Decrement is clamped at 0 (no negative
counts). New CSS (`.counter-item`, `.counter-stepper`, `.counter-btn`,
`.counter-value`) reuses existing design tokens (`--teal`, `--border`,
`--section-bg`, `--radius`) — no new colors/fonts. A row gets a teal-tinted
border once its count is > 0, mirroring `.choice-item.selected`.

### Data flow

- `collect(q)`: for `type==='counter'`, reads each item's stepper value into
  `ans[q.id] = { endpoints: n, applications: n, ... }` (numbers).
- `restore(q)`: for `type==='counter'`, writes stored numbers back into each
  stepper's displayed value.
- `submitForm()`: for `type==='counter'`, appends one Formspree field per item
  as `"${q.label} – ${item.label}"`, e.g. `IT Assets in Scope (Count) –
  Endpoint Devices`, mirroring how Q22's `assets` fields are submitted today.

### Validation

This form currently has **no enforced required-field validation anywhere** —
`req:true`/`req:false` is set on question objects but never read by `goNext()`.
Q22b is the first question to actually enforce `req`. When `req:true` and
`type==='counter'`, `goNext()` will check whether the sum of all item counts is
> 0 before advancing; if not, it shows an inline red message ("Please enter at
least one asset count") below the steppers and does not advance. This check is
scoped to `type==='counter'` only — no other question's behavior changes.

### Keyboard/Enter behavior

The existing keydown handler advances on Enter for every type except
`textarea` and `assets`. `counter` is added to that exclusion list so Enter
inside a stepper's number field doesn't prematurely submit the row (consistent
with how multi-field question types are already excluded).

## Q22 placeholder fix

In `buildAssets()`, the `physical` field's `ph` changes from
`'Tables, Chairs, TV, Projectors'` to `'Laptops, Firewalls, Desktops, External
Hard Drives, Server Rooms, Physical Cabling'`. This is a placeholder-text-only
change — the field's `id`, label, and submission key are unchanged.

## Out of scope

- No changes to Q22's field set (still 6 free-text boxes) or to any other
  question.
- No validation added to any question type other than the new `counter` type.
- No stepper max-value cap — counts are unbounded positive integers.
