# Coincident-route trials on current `dev` (`0b8f9f34`)

Material for the #248 / #260 discussion, produced as requested: several diagrams
generated with the **current version** and inspected, rather than another
implementation change.

Everything here was rendered with the public CLI:

```sh
node archify/bin/archify.mjs deliver architecture <spec>.json --quality showcase --json
```

Screenshots are 1440x900, dark theme, `Classic` preset, `Still` motion, default
zoom, taken through the Viewer's own CDP path with fonts settled — so the pairs
below are comparable.

## Results at a glance

| # | Case | `showcase` result | Reading observation |
| --- | --- | --- | --- |
| 01 | Original ambiguous pair (`labelAt`) | **passes, 0 diagnostics** | Two edges render as **one line** — see below |
| 02 | Dense mesh, 3 bidirectional pairs | **fails**: 2x `composition/ambiguous-corridor`, 1x `composition/micro-segment`, 2x `composition/label-route-clearance` | Undrawn here; the rejections are what the profile reports |
| 03 | Ordinary diagram, no overlap | passes, 0 diagnostics | Control case, reads normally |
| 04 | Comparison A — shared route | passes, 0 diagnostics | Single double-headed line, two labels |
| 05 | Comparison B — same content, separated | passes, 0 diagnostics | Two clearly distinct routes |
| 06 | Dense but valid, 1 intentional shared pair | passes, 0 diagnostics | Shared corridor reads fine; weak argument for warning |

## The head-line finding: case 01 is still not detected

The original reproduction from this issue is **unchanged on current `dev`**, and
`showcase` still passes it with no diagnostics:

```json
{ "id": "reads",    "from": "graph", "to": "engine", "label": "lists",    "labelAt": [355, 150] },
{ "id": "declares", "from": "engine", "to": "graph", "label": "declares", "labelAt": [355, 260] }
```

Both edges emit the **same path**:

```text
reads    : M 280 201 L 430 201
declares : M 430 201 L 280 201
```

The rendered SVG contains two arrow markers for two edges — they overlap into a
single double-headed line, and the two labels sit above and below it with
nothing tying either label to a direction. `deliver ... --quality showcase`
returns `exit 0`, `ok: true`, zero diagnostics.

I could not find any input where the current detector fires on this shape. The
one knob that decides it is **`labelAt`**:

| same two nodes, differing only in how the label is placed | routes emitted | `showcase` |
| --- | --- | --- |
| `labelAt: [355, 150]` / `[355, 260]` (case 01) | identical, one line | **pass** |
| `fromSide`/`toSide`, no `labelAt` | two distinct polylines, 14px apart | **pass** |

So on current `dev` the coincidence is a property of pinned-label routing, and
`composition/*` has no rule that sees it. That seems worth separating from the
"warn on every coincident pair" question: this is not a case of a shared line
being a deliberate presentation choice — nothing in the input asked for one.

## What the dense cases say about the warning tradeoff

**Case 02** is the denser diagram with several bidirectional pairs and explicit
sides. The current profile already rejects it with
`composition/ambiguous-corridor` (twice, on 158px shared corridors) plus
`composition/micro-segment` and `composition/label-route-clearance`. That is
relevant to the concern in the thread: the profile is not silent about shared
corridors in dense diagrams — it already treats some of them as blocking errors,
and this input needed rework before it could be drawn at all.

**Case 06** is the same dense topology reduced to **one** intentional shared
pair, spaced so it passes. Looking at it: the shared corridor between
`API Gateway` and `Orders` reads without ambiguity, because the two labels
(`submit`, `status`) sit on either side of it and the arrowheads point opposite
ways. I would not want a warning here — this is the "sharing improves
readability" case, and it is a fair argument against warning on every pair.

The distinction I would draw from these two, offered as an observation rather
than a rule: what hurts in case 01 is not that a line is shared, it is that the
**two labels cannot be assigned to a direction**. In case 06 they can. That
would suggest keying any future diagnostic on label/direction attribution
rather than on geometric coincidence — but that is exactly the kind of claim
that needs more cases than I have, so I am not proposing it as settled.

## Comparison A / B — does separating help or add clutter?

Same two nodes, same labels, same viewBox, same theme; only routing differs.

| | A — shared (case 04) | B — separated (case 05) |
| --- | --- | --- |
| routes | `M 280 201 L 430 201` and `M 430 201 L 280 201` | one loop above, one below |
| labels | `lists` above, `declares` below, both floating free | `lists` on the top loop, `declares` on the bottom loop |
| reading | cannot tell which label belongs to which direction | each label sits on its own route |
| `showcase` | pass | pass |
| extra repair needed | none | none — both forms are authored, neither is auto-repaired |

On this two-node case separating clearly helps and adds no clutter. But the pair
is the easy end of the range, and B needed explicit `via`/side geometry that an
author would have to write by hand. In case 06, separating all six relationships
would be the cluttered outcome rather than the readable one. So "separation
helps" looks true for an isolated ambiguous pair and false as a general rule —
which is, I think, the tradeoff you were asking to settle before adopting
anything.

## Files

- `specs/` — the eight JSON inputs, all runnable as-is
- `artifacts/` — delivered HTML plus each `*.delivery.json` receipt
- `screenshots/` — 1440x900 captures, plus the two `with-labelat` /
  `explicit-sides` probes that isolate the `labelAt` knob

## What I am not claiming

- Not a proposal to implement anything; `#260` stays parked per your request.
- Not a claim that warning is right or wrong. I have five cases, not a corpus.
- No detector was built and no implementation was changed for this. The two
  probes above were run with `render`/`validate` on the public CLI only.
- Case 02 was deliberately left un-repaired, so it has a `delivery-pending`
  receipt rather than a delivered artifact: its value here is the diagnostic
  set, not the picture.

## How to reproduce

Each spec delivers with the public CLI; no build step or local patch is needed:

```sh
node archify/bin/archify.mjs deliver architecture specs/01-original-ambiguous.json --quality showcase --json
```

`meta.output` in each spec is a bare filename, so the artifact lands next to the
spec. `artifacts/` holds what these specs produced on `dev` @ `0b8f9f34`, with
the matching receipt beside each one. `screenshots/` was captured at 1440x900,
dark theme, `Classic` preset, `Still` motion, default zoom, via the Viewer's own
CDP path with fonts settled.
