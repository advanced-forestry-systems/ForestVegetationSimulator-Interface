# FVS-ACD (AcadianGY.R) production version history

Tracks `fvsOL/inst/extdata/AcadianGY.R` on this branch, `acd-production-history`, from the
version last synced to this fork (v12.5.0) through the two revisions pushed here: v12.11 and
v13.1 (draft). Each version below is a single commit on this branch, oldest first, so the diff
between any two versions is reviewable directly from the commit list. Full per-point-release
detail lives in the file's own running header comment block and in holoros/fvs-acd.

## v12.5.0 (baseline on this fork)

Production baseline this branch starts from. Significant rewrite from the v12.3.x line:
standardized lowercase variable names, `make_stand()` / `make_acd_tree()` constructors,
modularized `calc_*()` component functions, tree/stand/ops validation functions, removed
`cyclen` (cycle length fixed at 1, reporting frequency is a user choice), removed the `plyr`
dependency.

## v12.6.0 through v12.10.1 (folded into the v12.11 commit)

Not pushed as separate commits; each is gated and OFF by default, so the cumulative effect on
default projections is nil until enabled. Summarized here because they are the bridge between
12.5.0 and 12.11:

- **12.6.0**: site-driver refinements (`make_stand()` accepts `esi=`), new SDImax-ceiling
  self-thinning mortality (`ops$use.sdimort`), MORTCAL retained but default OFF.
- **12.6.1**: ingrowth refinement, default recruitment switched to the statistically correct
  expected-value hurdle model (`E[ingrowth] = P(occur) * E[count]`), replacing a 0.95 occurrence
  gate that had suppressed recruitment on most plot-years.
- **12.6.4**: regional stand-mortality cut scale (`ops$mort.cut.scale`), a structural fix for the
  Maine mortality deficit.
- **12.6.5**: interim size-form ddbh modifier on the Kuehne 2022 diameter increment base.
- **12.7.0**: fitted ESI dominant-height anchor (`ops$use.esi.domht`), integrated as the
  operational site driver.
- **12.9.0**: regional SDImax multiplier hook (`ops$sdimax.region.mult`).
- **12.10.0**: strategic management modifiers (planted, PCT) as a fixed-untreated ratio of
  class-mean increments, guard-clamped 0.7-1.6. Both classes pinned to the 1.6 ceiling, an
  identifiability failure superseded immediately by 12.10.1.
- **12.10.1**: re-fit of the 12.10.0 management modifiers as a robust additive direct-effect
  release (10 pct trimmed mean of per-stand observed-minus-predicted-untreated increment),
  clamp-free. Fitted BA release planted +3.8, PCT +2.4 m2/ha. Validated out of sample:
  leave-one-installation-out mean abs bias 1.4, leave-one-province 1.5, beats OSM in 76 pct of a
  200-fold managed split-half. This is the frozen baseline v12.11 and v13.1 are held byte-
  identical against with their new levers gated off.

## v12.11

Targets the long-horizon volume bias in `20260718_longinterval_volume_diagnosis.md`:
under-prediction in developing stands, over-prediction in mature stands, the net of three
separate systematic component errors. Framed on Weiskittel et al. (2011) Ch 10 (linking
resolutions): run stand-level growth/mortality/ingrowth equations and constrain or disaggregate
tree-level predictions to them, rather than raw per-tree multipliers. Four gated levers, all OFF
by default, so default projections are byte-identical to frozen 12.10.1:

- **Lever 1, recruitment** (`ops$use.ingrowth.rdtaper`): an RD-conditioned taper on the flat
  BA-anchored ingrowth correction (about 2.5x, roughly density-flat), protecting mature closed
  stands (Baxter own ratio 1.31) from the same multiplier applied to open stands.
- **Lever 2, mortality** (`ops$mort.dens.scale`): management-gated attenuation of the
  self-thinning removal fraction. SILC-calibrated self-thinning over-removes at managed
  high-density structures (CTRN 4.6x observed, NB 2.5x) while naturally regenerated stands
  validate close to observed (Baxter, Maine FIA about 1.07); the correction is gated to managed
  stands only.
- **Lever 3, survivor diameter increment** (`ops$use.ddbh.rein` or `ops$use.ddbh.standcon`, not
  both): corrects excess survivor diameter growth concentrated in large trees (20-40 cm) of
  mature high-QMD stands, the sole driver of the Baxter benchmark's +9 pct volume over-prediction
  (survivor BA obs/pred 0.634).
- **Lever 4, hardwood height** (`ops$use.dht.hw.mod`): conditions height increment on form and
  risk class for six hardwood species, fit from FIA Maine GRM remeasurement. Form coefficient is
  a placeholder pending field NHRI form data.

Source: `holoros/fvs-acd/scripts/AcadianGY_12.11.r`, tag `v12.11-candidate-aug2026`.

## v13.1 (draft)

Adds two new gated switches alongside the v12.11 levers, which are preserved unchanged. With
both switches OFF and no v12.11 lever set, byte-identical to frozen v12.10.1 on a 20-year
young/mature projection (max abs diff 0.000e+00).

- **Switch A, recruitment** (`ops$use.hier.ingrowth`): replaces the v12.11 flat ingrowth
  multiplier with a fitted hierarchical negative-binomial (log link) posterior, carrying density
  (RD) and composition (QMD, BA) dependence the flat multiplier could not, plus region/eco/
  forest-type random effects. Cardinal job 12470707 v5.
- **Switch B, mortality** (`ops$use.hier.mortality`): replaces the v12.11 management-gated
  self-thinning attenuation with a management-free hierarchical binomial (cloglog link) mortality
  posterior, crossed random effects on region, region:eco, and forest type, no management term.
  Enters at the same point the v12.11 gate sat, preserving Reineke self-thinning and the density
  plateau. Targets two v12.11 managed-stand flags traced to the hand-tuned management gate: the
  QC volume over-correction and the SILC basal-area widening.

Both posteriors load at runtime from CSV (`ops$hier.postdir`, default `~/v13_kickoff`), never
hardcoded, memoized per (submodel, postdir). Per-group random intercepts fall back to 0
(population-level) when a group level is unseen or the ranef file is absent.

**Status: draft.** Both switches default OFF; this version ships the capability, not a change in
default projection behavior. Do not enable in production runs pending the refit review in
`20260722_v131_stress_redteam_review.md` and `20260722_v131_harvest_and_mort_repair.md`.

Source: `holoros/fvs-acd/scripts/AcadianGY_13.1_DRAFT.r`.
