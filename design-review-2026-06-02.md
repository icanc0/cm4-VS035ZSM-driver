# CM4 VS035ZSM Driver — Design Review

**Date:** 2026-06-02
**Reviewer:** kicad-happy (Claude Code)
**Scope:** Pre-fab review with focus on the just-completed MIPI DSI length/skew matching.
**Files:** `cm4-VS035ZSM-driver.kicad_sch`, `.kicad_pcb` (modified 2026-06-01), `.kicad_pro`
**Run:** `analysis/2026-06-01_2356/`

---

## Verdict

**Impedance concern RESOLVED (2026-06-02). No remaining blocker beyond MPNs/sourcing.**

> **Update 2026-06-02:** The earlier "~150 Ω" finding was based on the *stale* 0.2104 mm
> prepreg in the old KiCad stackup. The real fab stackup (**Huaqiu04161H02-1080**) uses a
> **0.077 mm 1080 prepreg** to L2 and is **Coplanar Differential**, for which Huaqiu's
> calculator gives **99.66 Ω** at 3.56 mil W / 5.94 mil gap — exactly the `100Ω_Diff`
> netclass (0.0904 / 0.1509 mm), with the 5.47 mil coplanar gap matching the netclass
> clearance (0.1389 mm). The geometry is correct; the length/skew matching stands. See the
> updated impedance section below.

The length and skew matching is genuinely excellent (see below). Remaining items are MPN
population (SS-001 sourcing blocker), a few cleanups, and a normal switcher EMC note.

---

## MIPI DSI review (your focus)

### Length & skew matching — excellent ✅

Measured from the routed PCB (`net_lengths`):

| Pair | P length | N length | **Intra-pair skew** |
|------|---------:|---------:|--------------------:|
| DSI0_CLK   | 35.710 mm | 35.710 mm | **0.0 µm** |
| DSI0_DATA0 | 38.880 mm | 38.880 mm | **0.0 µm** |
| DSI0_DATA1 | 38.800 mm | 38.800 mm | **0.0 µm** |
| DSI0_DATA2 | 38.880 mm | 38.880 mm | **0.0 µm** |
| DSI0_DATA3 | 38.880 mm | 38.880 mm | **0.0 µm** |

- **Intra-pair (P↔N) skew: 0 µm on every pair.** This is the parameter that matters most
  for D-PHY (it converts directly into differential→common-mode noise), and it's perfect.
- **Data lane-to-lane:** lanes 0/2/3 = 38.880 mm, lane 1 = 38.800 mm → **80 µm** spread.
  Far inside any DSI inter-lane budget.
- **Clock-to-data:** clock is 3.17 mm shorter than the data lanes (35.71 vs 38.88 mm).
  ≈19 ps at ~6 ps/mm. D-PHY is source-synchronous and the receiver deskews per lane, so
  clock-to-data matching is loose — this is comfortably within budget. No action needed.

### Diff-pair impedance — correct on the real stackup ✅ (resolved)

The `100Ω_Diff` netclass routes at **W = 90.4 µm (3.56 mil), gap = 150.9 µm (5.94 mil),
clearance = 138.9 µm (5.47 mil)**. Against the **real Huaqiu04161H02-1080** stackup —
**Coplanar Differential** on L1, referenced to L2 across a **0.077 mm 1080 prepreg** — the
fab's calculator returns:

```
Huaqiu Stack-up2 (Huaqiu04161H02-1080), L1 Coplanar Differential, ref L2:
  target  3.50 / 6.00 / 5.50 mil  (W / gap / coplanar-gap)
  etched  3.56 / 5.94 / 5.47 mil  ->  Zdiff = 99.66 Ω
```

That matches the netclass exactly (W 0.0904, gap 0.1509, clearance 0.1389 = 5.47 mil
coplanar gap). The geometry is correct **for this stackup**.

**Why the earlier estimate was wrong:** my first pass used the stackup *then loaded in the
KiCad file* — a **0.2104 mm prepreg, plain microstrip** — which gives ~150 Ω at 90 µm. The
real fab stack uses a **0.077 mm** prepreg to the reference plane **and** coplanar ground
coupling, both of which call for a narrower trace at 100 Ω. With the correct inputs, 90 µm
is right. Lesson confirmed: the KiCad stackup must match the fab's controlled-impedance sheet
before trusting any impedance number.

**Remaining actions (now bookkeeping, not blockers):**
1. Enter Huaqiu's real dielectrics into Board Setup → Physical Stackup (0.077 mm 1080
   prepreg ×2, 1.294 mm core, εr per their sheet) so KiCad's own tools agree. Note KiCad's
   built-in calculator models plain coupled-microstrip and won't reproduce the coplanar
   number — trust Huaqiu's 99.66 Ω.
2. Confirm the F.Cu GND pour flanks the pairs continuously at the 5.47 mil coplanar gap for
   their full length — the coplanar return depends on it, not just on the L2 plane.

**Stackup thickness sanity (Huaqiu04161H02-1080):** the listed stack sums to 1.471 mm
(0.0115 foil + 0.077 pp + 1.294 core + 0.077 pp + 0.0115 foil), not 1.60 mm. The table lists
only bare dielectric + base foil; the ~0.13 mm balance is outer-copper plating (base ~0.0115
→ finished ~0.035 mm/side), solder mask (~0.02 mm/side), and surface finish, with the 1.6 mm
nominal carrying a ±10 % tolerance. Not an error.

### Reference planes & return paths — clean ✅

- **In1.Cu and In2.Cu carry only the GND pour — no signal routing on either inner layer.**
  So both inner layers are solid ground planes. Data lanes (F.Cu) reference solid GND on
  In1; the clock's B.Cu portion references solid GND on In2.
- There **is** a +5V_IN island (195 mm²) carved into In2.Cu at x∈[133,164]. The clock's
  B.Cu run sits at x∈[122,125] — **clear of it**, so the clock's return path is not broken.
- The clock makes **two layer transitions** (F→B→F, 2 vias per trace — likely to dive under
  an obstacle). Each clock via has a **GND stitching via ~0.8 mm away**, so return current
  has a path. Tighten to <0.5 mm if convenient, but this is acceptable. (Notably, the EMC
  return-path check flagged several *control* nets for missing stitching vias but **not** the
  clock — consistent with the clock being properly stitched.)

### Minor MIPI notes

- **Clock pair naming is inconsistent:** data lanes use `_P`/`_N` (`DSI0_DATA.0_P`) but the
  clock uses `.P`/`.N` (`DSI0_CLK.P`). The analyzer auto-detected the 4 data pairs as MIPI
  diff pairs but **missed the clock** because of the separator. Harmless to the board, but
  rename to `DSI0_CLK_P/_N` so tooling (and future-you) treat it uniformly.
- **No inline ESD on the DSI lanes** (`has_esd = false` on all pairs). For an internal
  board-to-FPC display link this is a normal trade-off — TVS adds capacitance that hurts the
  eye, and RPi's own DSI connector has none. Leave it off unless the flex is user-exposed.

---

## Other findings

### False positives (no action)

- **EMC SU-001 ×3 "adjacent signal layers" (error):** In1.Cu/In2.Cu are *typed* "signal" in
  the layer list but are used as solid GND planes. Functionally these are planes. Optionally
  set their type so DRC/EMC tools stop flagging it, but the board is fine.
- **Schematic VM-001 ×7 "domain crossing without level shifter" (error):** all trace to the
  2×8 header U6 pins being tagged a 5 V domain (I2C `SDA_3V3`/`SCL_3V3`, `LED_PWM`, backlight
  sense) or to the negative rail `-5V7_*` magnitude being compared as a positive logic level.
  None are real missing level shifters.
- **EMC XT-001 "close spacing CLK.N/CLK.P":** that's the intentional intra-pair coupling of
  the 100 Ω pair. Expected.
- **EMC DC-001 "decoupling too far from U6/U4/U5":** U6 is a header, U4/U5 are KF1027B
  switches — not ICs needing tight bypassing.

### Worth a look (not blockers)

- **EMC SW-001/SW-003 — switcher EMI (U1 MCP1603, U2):** harmonics in the 30–88 / 88–216 MHz
  bands and a "large hot loop" on U1. Normal for a buck, but tighten U1's hot loop (input cap
  → VIN/SW/GND) and keep the catch path short. Worth doing while you're re-routing for
  impedance anyway.
- **I2C pull-ups:** the only resistors are R1 (100 k), R2 (5.1 Ω sense), R3/R4 (feedback
  divider → 1.818 V, SPICE-confirmed). There are **no dedicated SDA/SCL pull-ups on this
  board** — confirm they live on the CM4 carrier, or the TPS65132 control bus won't work.
- **+5V_IN plane split — 3 islands (cross PS-002, info):** the +5V_IN pour is fragmented.
  Fine if each island independently reaches the source, but verify the input current path
  isn't necking through a thin bridge.
- **Stale gerbers:** `dfm/gerber/` is dated 2025-05-22, long before this routing. Regenerate
  before ordering — don't fab from those.

### Clean ✅

- **SPICE (ngspice): 7/7 pass** — R3/R4 divider 1.818 V, decoupling impedances low
  (z_min ~13–46 mΩ), inrush low (<0.1 A). No value-computation errors.
- **Thermal: 100/100, 0 findings.** No power part is near its limit.
- **Cross-domain:** only the +5V_IN split (above) and "7 parts in PCB not in schematic"
  (the 4 mounting holes + test point + fiducials — expected).

---

## Verification basis & gaps

- **No datasheets / no MPNs** (24/25 parts have no MPN; SS-001 sourcing blocker, DS-002).
  All pin-level statements here are **consistency checks (schematic↔PCB↔analyzer agree)**,
  **not** datasheet-verified. Before fab, populate MPNs and ideally sync datasheets so the
  TPS65132 (U3), MCP1603 (U1), LP3320 (U2) pinouts can be checked against the real parts —
  especially U3's WQFN-20 pad mapping and the KF1027B switch pinouts.
- **Impedance** is a hand estimate, not a field solve — confirm with KiCad's calculator.
- **Board outline dimensions** did not extract from the PCB (board_outline null) — not
  material to this review.
- **EMC `emc_risk_score` reported 1.0** with provenance 0 % (no datasheets); treat EMC
  findings as topology-level.
- **Lifecycle audit not run** — no MPNs / no distributor credentials.
- No prior review to diff against (first run).

---

## Pre-fab checklist

- [x] ~~Reconcile diff-pair impedance with the real Huaqiu stackup~~ — done; 99.66 Ω confirmed, geometry correct
- [ ] Enter Huaqiu04161H02-1080 dielectrics into KiCad Physical Stackup (0.077 mm pp, 1.294 mm core) so KiCad tools agree
- [ ] Confirm F.Cu GND pour flanks the pairs continuously at the 5.47 mil coplanar gap
- [ ] Confirm I2C pull-ups exist (this board or the carrier)
- [ ] Rename `DSI0_CLK.P/.N` → `_P/_N` for tooling consistency
- [ ] Tighten U1 buck hot loop; optionally move clock stitching vias <0.5 mm
- [ ] Populate MPNs (SS-001 blocker) and sync datasheets; verify U3 WQFN pad map
- [ ] Regenerate gerbers (current ones predate this routing)
