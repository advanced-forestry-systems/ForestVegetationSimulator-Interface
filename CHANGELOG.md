# FVS-ACD (AcadianGY.R) production version history

Tracks `fvsOL/inst/extdata/AcadianGY.R` on this branch, `acd-production-history`, from the
version last synced to this fork (v12.5.0) through the revisions pushed here: v12.11, v13.1
(draft), v13.2 and v13.3 (draft). Each version below is a single commit on this branch, oldest first, so the diff
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

## v13.2

The v13.1 hierarchical engine as it was actually run on Cardinal through August 2026, replacing
the August 11 draft above. Retagged from AcadianV13.1 to AcadianV13.2 on September 8, 2026 because
the AcadianV13.1 tag had come to name four distinct files in the source repository. Both v13.1
switches stay OFF by default, so default projections remain byte-identical to frozen v12.10.1.
Changes relative to the draft, oldest first:

- **July 25, Switch A enters multiplicatively.** The ingrowth posterior modifies the engine's
  native `ingrowth.ph` rather than replacing it, since `exp(b0)` sits on the FIA microplot recruit
  basis (about 51 recruits ha⁻¹ yr⁻¹, roughly four times the engine's native 13) and as a level
  inflated PEF ingrowth about 14 fold. The intercept and offset are excluded so the modifier is
  1.0 at training-mean covariates.
- **July 28, aggregate-neutrality normalizer.** A log-link modifier centred on the training mean
  is pointwise 1.0 but population-mean `exp(sigma²/2)`. N is measured once on the fitting rows,
  shipped beside the posterior as `<prefix>_norm.csv`, memoized with it, and never recomputed from
  the stands being projected. Missing file means N = 1.
- **August 13, ingrowth density shutoff on the hierarchical path.** `ops$ingrowth.rd.ceiling`
  (default 1.00) ramps `ingrowth.ph` linearly to zero as relative density approaches the
  ceiling, mirroring the mortality posterior's `sdm_keepc` clamp. Fixes the multi-minute
  density-loop stalls of Cardinal job 13451233. The flat-multiplier branch is untouched.
- **August 27, vectorization fix.** `ddbh_hw_mod` is scalar and was being handed whole columns,
  so under R 4.2 and later any stand carrying a hardwood with non-NA form and risk errored out of
  `AcadianGYOneStand` and was silently dropped by the benchmark runner's per-stand `tryCatch`.
  Now applied element-wise with `purrr::pmap_dbl`, the convention the other modifiers use. No
  modifier's numerical behaviour changes.
- **Candidate, `ops$use.ingrowth.matgate`** (default FALSE). A size-structure gate on
  density-released ingrowth on the non-hierarchical path, `g(QMD) = 1/(1+exp(k(QMD − qc)))` with
  `qc` 12.7 cm and `k` 0.6, so a mid-rotation crash refills its density headroom with survivor
  volume rather than volume-free recruits.

Source: `holoros/fvs-acd/scripts/AcadianGY_13.1.r` on branch `engines-13.1-13.3-20260908`,
4,897 lines, md5 `6ec1823690ab6756b11e4f6fc684a325`.

## v13.3 (draft)

Hardwood form and risk gap-fill on Lever 4's existing `ops$use.hw.mod`, per the August 26, 2026
"fill gaps only" call. Nothing already validated for RO, SM, YB, RM, PB or QA growth, or for QA,
RM, RO, YB or PB survival, is touched, and `dht_hw_mod` (height) is untouched. With `use.hw.mod`
OFF the engine is byte-identical to v13.2 behaviour. Everything in v13.2 is preserved.

- **Growth, American beech (AB) and white ash (WA).** The six species `ddbh_hw_mod` modifier is
  a ratio in which every species-shared term cancels, so it is provably identical across species
  and could not be extended by adding rows. AB and WA get a new non-cancelling modifier,
  `ddbh_hw_mod_fia`, fit August 26, 2026 on the Maine FIA GRM form and risk crosswalk (glmmTMB,
  Gamma log link, n = 134,449, RM reference), keeping the native three level risk. AB's form and
  risk-R2 terms are not individually significant and are disclosed in the coefficient comments.
- **Survival, sugar maple (SM), amended August 27.** Carried by `tsurv_hw_mod_fia`, a cloglog
  hazard-ratio form `M = clamp(S_ref^(HR − 1), 0.60, 1.00)` fit on the same crosswalk
  (n = 15,881), behind a new sub-flag `ops$use.hw.surv.fia` nested under `use.hw.mod` and
  defaulting to FALSE, so behaviour with `use.hw.mod` alone is unchanged.

**Status: draft.** Ships the capability, not a change in default projection behaviour.

Source: `holoros/fvs-acd/scripts/AcadianGY_13.3.r` on branch `engines-13.1-13.3-20260908`,
5,120 lines, md5 `1910df0e5586410f922852efb6293aa0`.

## v12.11.1, a maintenance release NOT on this branch

Three mortality shutoff repairs to v12.11 made September 8, 2026 in response to the report that
under some conditions mortality stops and projections become growth only. M1, `ba.growth.top30`
takes `na.rm` and `sg` is coalesced to the all-species mean, so a species code absent from
`sp.attr` (common on raw FIA input, whose species list is wider than the Acadian table) can no
longer propagate NA through `cum.sdi.ward` into `stand.pmort` and silently void a plot's
mortality. M2, `ops$mort.expected.value` (default FALSE) switches the stand occurrence gate to
the hurdle expectation `P(occur) × E[amount]`. M3, the tree allocation gains `na.rm`, a guard for
the case where every tree survival saturates at 1.0 and the ordering weights sum to zero, and a
coalesce on `mort.mult`. No fitted coefficient changes, and every repair either removes a silent
NA-to-zero path or is gated OFF.

It is not committed here because this branch carries one file and v13.2 and v13.3 do not yet
contain M1 to M3, so committing v12.11.1 after v13.3 would present a downgrade as the newest
revision. It is under review as `holoros/fvs-acd` pull request #7 (`scripts/AcadianGY_12.11.r`,
4,661 lines, md5 `127cffc33ec35607e3b138a7acf12384`) and will be folded into the next v13 point
release, which will then land here in sequence.
