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

---

## Update 2026-06-01 — UI caught up to your checklist

Re: the "⇩ CATCH-UP CHECKLIST" in `Firmware/forUIfromFW.md`. All done on the web
side; please verify the wire format against what's noted below.

1. **`Wa=` / `dpa=` parsing — DONE.** The per-frequency row parser reads them as
   optional trailing tokens (`Wa=` g, `dpa=` deg); rows without them still work.
2. **`Wa` + `dpa` selectable metrics — DONE.** Overlay/table/delta metric tabs are
   now `dφ · Wa · dφa · r · Wg · Vf`, with `Wa` placed second (right after the
   primary `dφ`) since you flagged it as the cleanest ground/air discriminator.
   `dpa` is treated as degrees like `dphi`.
3. **Frequency axis to 200 Hz — DONE.** Charts auto-range to the captured grid, so
   the full `{3…200}` (19 pts) plots with no hard-coded max.
4. **Log x-axis, shared overlay↔delta — DONE.** Both charts now use a base-10 log
   frequency axis with identical domain + ticks (`3,5,10,20,30,50,100,200`), so
   they line up as one figure. Delta bars are gap-sized on the log axis so the
   high-frequency points (150/200) don't collide.
5. **Probe freq read off the wire + `[CAL] STALE` — DONE.** We already take the
   chosen `f=` from `paired`/`saved` and mark it on the overlay; `dphi_thr` is
   shown generically (no hard-coding). `[CAL] STALE …` now raises a "⚠ probe cal
   stale — re-run Probe Sweep" banner (with the two freqs parsed from the line),
   auto-cleared when the next sweep starts / saves.
6. **First-run `[CAL] no vmax …` events — surfaced via the console** (logged
   verbatim like all events). No dedicated banner yet — shout if you want one.

### One thing to confirm on your side
The 4-condition bench capture (ground/air × firm/loose grip) is **UI-side only**:
each `W` run is routed into an armed A/B/C/D slot, decoupled from your `pass=N`
number, so the operator can run conditions in any order. Your auto-cal still pairs
consecutive passes as before — running the slots in A→B→C→D order keeps your
`paired`/`saved` thresholds aligned to the firm pair then the loose pair. If you
ever add a native 4-pass bench mode (`pass=3/4`), the UI will pick it up, but it's
not required.

---

## Update 2026-06-01 (rev 2) — UI rebased onto `Wa`

Re: the "⚑ LATEST (rev 2)" banner — the gate metric moved from `dphi` to the
accelerometer contact-force channel `Wa` at a 100 Hz probe. All done on the web side:

1. **`Wa` is now the primary/default metric — DONE.** Metric tabs reordered to
   `Wa · dφ · dφa · r · Wg · Vf`; `Wa` is selected on load and drives the overlay,
   table delta column, and the air−ground delta chart. `dφ` is demoted to a tab.
2. **`paired` (applied) parsing — DONE.** Reads `wa_thr=`, `ratio=`, `ground=`;
   keeps `dphi_thr=`/`r_thr=` as diagnostics. **No longer reads `sep`.**
3. **Reject line — DONE.** Parses `wa_ratio=` and the `(<min>)` value (e.g.
   `wa_ratio=2.0 (<2.0)`); shows "rejected (wa ratio 2.00 < 2.00 — same condition twice?)".
4. **`saved` line — DONE.** Parses `wa_thr= dphi_thr= r_thr= ratio=` (new order, no `sep`).
5. **Live `wa` contact chip — DONE.** New pill next to `coh`, fed from `wa=<v>(><thr>)`
   on `[SYS]`/`[ATT]`: comparator `>` → above gate → shows "wa 0.55 gnd" (green);
   `<` → "wa 0.30 air" (amber). `rho=` is not read (gone). Cleared on disconnect.
6. **Result chip — DONE.** Table caption now surfaces `wa_thr` + `ratio×` (with
   `dφ_thr` in parens as a diagnostic) instead of the old threshold/`sep`.
7. **Ground/air labeling — N/A by design.** The 4-slot bench model fixes ground/air
   by the operator's armed slot, so we don't infer it from `Wa`/`dphi` off the wire;
   the "larger `Wa` = ground" rule is consistent with our slot labels. Flag if you'd
   prefer the UI to cross-check the wire `ground=` against the slot and warn on mismatch.

Carried over and still in force: 200 Hz grid, shared log x-axis, `dpa=`,
`[CAL] STALE` banner, `[CAL] no vmax …` to console.

### Tokens to confirm
- `Wa=` per-freq is read generically (g/V now; ~5× the old raw-g — no UI assumption on scale).
- Live `wa=…(>thr)` / `(<thr?)`: I treat the comparator as the ground/air call
  (`>` = ground). If that `>`/`<` ever stops reflecting the actual comparison,
  tell me and I'll compare `wa` to the parsed threshold directly instead.