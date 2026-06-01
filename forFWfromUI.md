# Firmware requests from the web UI

Notes from the web app to the firmware, generated while building the **Probe
Sweep** ("free-spin detection") UI. The UI already works with the event strings
you specified — **nothing here is required to ship.** These are enhancements,
ordered by value.

## Context: what the UI consumes today

The web app sends `W\n` on the existing CMD characteristic
(`e69a0003-1234-5678-9abc-def012345678`) and parses these event-characteristic
notifications:

```
SWEEP f=5.0 r=2.43 Vf=0.0123 Wg=0.0850 cpl=6.91 dphi=-178   (one per frequency)
SWEEP pass 1 captured — now run 'W' in the OTHER condition (ground↔air)
SWEEP paired: PROBE_F=5.0 Hz threshold=2.43 (ratio 1.93x) — applied + saving
PROBE auto-cal: PROBE_F=5.0 Hz threshold=2.43 (air/ground ratio 1.93x) — saved
```

It groups the per-frequency rows into two passes (closed by `SWEEP pass …` and
`SWEEP paired …`), aligns them by `f`, and plots:
- **Overlay graph** — the selected metric (r / Wg / cpl / dφ) for pass A vs pass B
  vs frequency, with the gap shaded and the peak-separation frequency flagged.
- **Delta graph** — every metric's normalized air↔ground delta at once.

---

## 1. (Recommended) Tag each pass with its detected condition

**Why:** the UI currently labels the first pass **A = ground** and the second
**B = air**, purely from the on-screen instruction order. The firmware actually
*knows* which pass is which (it computes the air/ground ratio). If the operator
runs **air first**, every label and the sign of every delta is inverted, and the
"pass A (ground)" legend is simply wrong.

**Ask:** include the detected condition in the close lines, with a stable,
machine-readable token. Suggested forms (either works — pick one and keep it
fixed):

```
SWEEP pass 1 captured cond=ground — now run 'W' in the OTHER condition
SWEEP paired: PROBE_F=5.0 Hz threshold=2.43 ratio=1.93x base=ground — applied + saving
```

i.e. a `cond=ground|air` token on each `pass N captured` line (and/or a
`base=ground` token on the `paired` line telling the UI which pass was ground).

The UI will then label the passes by true condition regardless of run order.
*(This needs a one-line follow-up in the web app to parse the token — flag me
once the token format is final and I'll wire it.)*

## 2. (Optional) Keep the frequency grid identical across both passes

**Why:** the UI joins the two passes on an **exact** `f` match. If pass 1 reports
`f=5.0` and pass 2 reports `f=5.01` (or one pass drops/adds a frequency), those
points won't pair — the overlay shows orphan points and the delta bar for that
frequency disappears.

**Ask:** emit the same set of `f` values, formatted identically, in both passes.
If they can't always match, that's fine — the UI degrades gracefully (it just
can't pair the odd ones) — but identical grids give the cleanest plots.

## 3. (Optional) Machine-readable tokens on the result lines

**Why:** `PROBE_F=`, `threshold=`, and `ratio=` are the values that actually get
saved. The UI could mark the **chosen** `PROBE_F` on the graphs (distinct from
the peak-separation frequency it computes itself) and surface the saved
threshold/ratio in a result chip.

**Ask:** keep `PROBE_F=<hz>`, `threshold=<v>`, and a `ratio=<x>` (or
`ratio 1.93x`) token present and stable on the `SWEEP paired …` / `PROBE
auto-cal …` lines. The current format already satisfies this — just please don't
change the token spellings, and prefer `ratio=1.93` (bare number) over
`(ratio 1.93x)` if you ever revise it, so parsing doesn't depend on the `x`/parens.

## 4. (Optional, nice-to-have) Explicit pass-begin marker

**Why:** run boundaries are currently inferred (a data row arriving after the
previous pass was closed starts a new pass). This is robust given the close
lines, but an explicit header would make it bulletproof against re-runs and
dropped close lines.

**Ask:** emit `SWEEP begin pass=N cond=<ground|air>` before each pass's first
`SWEEP f=` row. Purely belt-and-suspenders; skip if inconvenient.

---

### Not needed
- No new BLE characteristic, no read/GATT changes — all results stay on the
  existing event-notify channel.
- No units required; the UI labels axes generically. (`dphi` is treated as
  degrees in the UI's display only.)
