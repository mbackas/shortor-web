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

### Settable params (`CP` / `Cw`) — DONE
Added controls in the Balancing pane under the slot picker (write-through over the
same debounced `C<code><val>` channel as the LQR card; gated `needs-conn`):
- **Probe frequency** — number input (3–240 Hz) → `CP<hz>` on commit. Because it
  invalidates the cal, the UI immediately raises the "re-run Probe Sweep" banner
  (and also reacts to your `[CAL] probe freq=… set — re-run W` echo).
- **Wa threshold** — slider (0–2.5 g/V, per your raised upper limit) → `Cw<g/V>`
  (debounced on drag). The live `wa` chip shows the value cross the new gate.

Neither is echoed in `[SYS]` (as you noted), so the UI is the source of truth. I
also **sync the controls back from the wire** so they're never stale: from
`paired`/`saved` (`f=` → probe, `wa_thr=` → threshold) and from `[CAL] probe
freq=… set` / `[CAL] wa_thr=… set`. Edits while a field is focused aren't clobbered.

To confirm: the wire spellings are `CP<hz>` (e.g. `CP60`) and `Cw<g/V>` (e.g.
`Cw0.42`) — bare value, no space, like `CF`/`CH`/`CT`. Shout if either differs.

### `CO` observer-bandwidth slider + NVS readback — DONE
- **`CO<hz>` slider** added to the haptic feel group (right under "Velocity
  smoothing"), range 5–60 Hz, default 20, sent as `CO<hz>` on drag. It's a
  `PARAMS` entry, so it's covered by the existing snapshot + **"Save to knob"
  (`S`)** affordance — matching its feel-param persistence (live now, NVS only
  on `S`). `CP`/`Cw` stay auto-save-on-send (no Save needed), as you specified.
- **NVS readback — DONE.** On connect the UI now requests a dump (`Z`) and parses
  the `NVS …` line for **`pfl`** → probe-freq control, **`pwa`** → Wa-threshold
  control, **`fc_vo`** → observer slider, so all three seed from the knob's stored
  values instead of UI defaults (focused fields aren't clobbered). The manual
  "Dump NVS" button feeds the same parser.
- Confirm the wire spelling is `CO<hz>` (e.g. `CO20`) and that the NVS-line tokens
  are exactly `pfl` / `pwa` / `fc_vo` (`fc_vo` with the underscore). Shout if any differ.

---

## Update 2026-06-02 — Wi-Fi OTA STA mode — DONE
Implemented the `NS`/`NP`/`NX` + `shortor.local` flow:
- **Creds form** in the Firmware/OTA pane: SSID + password + **Save Wi-Fi**
  (sends `NS<ssid>` then `NP<pass>`) and **Clear** (`NX`). SSID/password are sent
  as the literal rest of the line (spaces preserved, no quoting/trim on the
  password). The browser remembers only the **SSID** (for display); the password
  is never stored client-side. `[WIFI] ssid set` / `pass set` / `creds cleared`
  echoes are logged and keep the UI's STA flag in sync.
- **Dual-host race** on flash: after `Y`, the UI probes `/ota-prep` on **both
  `shortor.local` and the SoftAP IP** in parallel each round and uploads to
  whichever answers first — so STA (creds saved → no network switch) and the
  SoftAP fallback both "just work," and it's robust to `shortor.local` mDNS not
  resolving (e.g. Windows w/o Bonjour). The OTA overlay is adaptive: STA shows
  "joining your Wi-Fi → shortor.local, stay put"; otherwise the SoftAP steps.
- No GATT change consumed; all on the existing CMD characteristic.

Confirm the spellings: `NS<ssid>` / `NP<password>` / `NX`, the OTA path is
`http://shortor.local/firmware` (+ `/ota-prep` for readiness), and the `[WIFI]`
echo strings contain `ssid set` / `pass set` / `creds cleared`. Shout if any differ.
---

## Update 2026-06-02 — synced to the consolidated wire contract

Worked through your rewritten `forUIfromFW.md`. New/changed items now done UI-side:

- **`[TUN]` / `Cu` self-describing registry — DONE.** New "Tuning" pane builds
  itself from the `[TUN]` dump (parsed on the `Z` read): one number input per
  `name=value`, friendly labels for the known set (`rthr`/`tfs`/`fsvm`/`fsvt`/
  `coh`/`ptone`/`sbkl`/`smax`), and **unknown names render under their raw key**
  so future params need no UI change. Edits send live; the existing Save-to-knob
  (`S`) persists. **Confirm the wire format:** I send `Cu <name>=<value>` *with a
  space* (matching your example `Cu rthr=0.7`) — flag if it should be `Cu<name>=…`.
- **Telemetry packet 56 bytes — DONE.** Dispatch now reads `[13]@52` as the RAW
  `shaft_angle` and feeds *that* to the scroll-practice ring (falls back to `[0]`
  on older 52-byte packets). `[0]` still drives the balance/cart visual.
- **`[HB]` keepalive — DONE.** Treated as a no-op: it pets the 1 s link-alive
  watchdog and is **not** logged (so quiet periods don't spam the console or
  trip a false "disconnected").
- NVS tokens: `pfl`/`pwa`/`fc_vo` already seed probe-freq/Wa-thr/CO. `fc_v` (CF)
  is seeded via the params readback already; `fc_vh` (CH) is intentionally
  ignored — the haptic velocity-smoothing slider was removed.

**One mismatch to confirm — `CO`.** Your contract lists `CO` as default **80 Hz,
~20–300**. Per an explicit request the UI slider is currently **log-scaled 5–500,
def 20** (and relabeled "Velocity LPF (Hz)"). The range is a superset of yours and
the real value is seeded from NVS `fc_vo` on connect, so it's functional — but say
the word if you want the UI bounds/default tightened to 20–300 / 80.

---

## Update 2026-06-03 — macOS getDevices()-first connect + usbhap

- **`getDevices()`-first connect — DONE (§5).** The Connect button now tries
  `navigator.bluetooth.getDevices()` first, matches a `Shortor`-prefixed device,
  and connects with **no chooser** — so it works on macOS while the OS holds the
  HID-mouse link (the device is invisible to the `requestDevice()` scan there) and
  is silent on Windows too. The chooser (`requestDevice`, `namePrefix:"Shortor"`)
  is the fallback for the first-ever grant only, and on that path we log the Mac
  hint ("connect right after power-on before it pairs as a mouse, or Forget it
  first"). Falls back gracefully if `getDevices` is undefined. (Load-time
  auto-reconnect already used `getDevices()`.)
- **`usbhap` tunable — DONE.** Appears automatically in the self-describing Tuning
  panel; added a friendly label ("haptic on USB — 1=on (bench)"). It's a 0/1 flag
  rendered as a number input — say if you'd prefer a checkbox for boolean tunables.

Nothing firmware-side needed for either (you noted §5 is a platform issue).

---

## Update 2026-06-03 (pm) — Tuning units/ranges + csat, macOS §5

- **Per-name hint map for the Tuning panel — DONE.** Known tunables now render as
  ranged **sliders with units** instead of bare number inputs (rthr, tfs ms,
  fsvm ×, fsvt ms, coh, ptone V, sbkl notch, smax /frame, usbhap, **csat mm**).
  Out-of-range incoming values auto-expand the slider max so they're still
  reachable. Unknown names keep the generic **number input** (no assumed range),
  so future params still appear. Send + persist (S) unchanged.
- **`csat` — DONE.** "balance pos-err sat", unit mm, slider 0.5–20 (you noted FW
  converts mm→rad by the 27.5 mm wheel R). Confirm the spelling `csat` and that
  the value is in **mm** as labeled.
- **macOS §5 — noted, no code change.** Understood it was a stuck CoreBluetooth
  daemon (reboot fixes), and the 1442+ advertising-name fix makes `namePrefix`
  robust. We already ship `getDevices()`-first (the "optional polish") as the
  default connect path on all platforms, with `requestDevice()` as the first-grant
  fallback — kept.

Unrelated UI tweaks this session: telemetry-loss now tears down a stale GATT and
reconnects immediately (was: sit Disconnected); Position-gain slider max raised to
5.0; and a connect-button gesture regression fixed (getDevices cache read
synchronously so requestDevice keeps its user activation).
