# Pilot 007 — Frozen Protocol
## Service-Level Configuration Sweep for vLLM Inference: Conservative Tier Recipe v1 (Zero-Precision-Risk Layer)

> **Status**: FROZEN upon maintainer sign-off. This document is the authoritative public protocol. Execution begins only after ignition-day pinning (§12) is backfilled as logged addenda.
> **Frozen**: 2026-11-05 (v1.0). **v1.1**: 2026-11-06 — adjudicates the external review round (dispositions in Appendix A). **Window**: 21 days from ignition.
> **License**: MIT (repository-wide).

---

## 1. Objective and Product Definition

**One sentence**: In a frozen measurement environment, sweep system-level vLLM serving knobs in a pre-registered order and produce **Conservative Tier Recipe v1** — the first item on Effiq's tiered optimization menu: no weight modifications, zero precision risk, every number shipped with a confidence interval.

**Product boundary**: The deliverable of this pilot is the **recipe** (an optimization artifact). All measurement machinery exists solely to make the recipe verifiable; measurement itself is never the product.

**Rationale**: The first question a compute-center customer asks is "how much can you extract without changing the model?" The conservative tier is the mandatory first good on the shelf. Serving-configuration knobs are publicly documented; the gap is not knowledge but **measured recipes with evidence**. The measurement pipeline already exists (replay pipeline and judging line from prior pilots), so this is an engineering-measurement task, not research.

**Scope boundary**: No model compression, no weight edits, no kernel changes. PD-disaggregation requires multi-GPU/multi-node and is out of scope (candidate for Pilot 009+).

## 2. Dual-Track Design

| | Track A — primary (domestic accelerator) | Track B — reference (NVIDIA) |
|---|---|---|
| Hardware | Hygon DCU (gfx936, 64 GB), shared cluster | Cloud L40S 46 GB |
| Model | Qwen3.8-27B (rehearsal-line model; revision pinned at ignition) | Qwen2.5-14B-Instruct (revision `cf98f3b3…`, full 40-hex recorded in the Effiq pilot-logs manifest; identical to Pilots 001/002) |
| Engine | Official Hygon vLLM 0.21.0-dtk2604 container (image sha256 pinned at ignition) | vLLM 0.31.0 (identical to Pilots 001/002) |
| Output | Knob-response intelligence for the domestic accelerator + Conservative Tier Recipe (DCU edition) | Conservative Tier Recipe (L40S edition) + dual-track response comparison table |

Both tracks run **one protocol and one measurement-script architecture**. Engine-parameter differences enter as spec-JSON data; the pipeline code contains zero hardware conditional branches.

## 3. Sweep Targets — First Batch: Four Knobs

| Knob | Levels (pre-registered) | Zero-precision-risk basis |
|---|---|---|
| `enable_prefix_caching` | off / on | Cache hits save compute without changing semantics |
| `max_num_seqs` | default / +2 levels / −1 level | Batch size affects scheduling and memory allocation |
| Chunked prefill (`max_num_batched_tokens`) | default / 2 levels | Prefill chunking granularity |
| Speculative decoding | off / on | A lever validated in Pilot 001; this pilot measures its interaction with the other three |

**Level derivation — a frozen rule, not post-freeze freedom.** Track B levels are pinned (`max_num_seqs`: default / two up / one down; `max_num_batched_tokens`: 4096 / 16384 / default — mapping in the spec file). Track A absolute levels are **deterministic outputs of this frozen rule applied to ignition-day probe evidence**, not discretionary choices:

- `max_num_seqs` (Track A): { ⌊0.5×D⌋, D, ⌊1.5×D⌋, ⌊2×D⌋ }, where D = the engine's compiled default, each level clamped to [1, the probe-verified engine maximum].
- `max_num_batched_tokens` (Track A): { 4096, default, 16384 } — identical to Track B; if a probe shows a level unsupported on the Track-A engine, the pre-registered N/A path (§4) applies.
- `enable_prefix_caching`: off / on (both tracks).
- Speculative decoding: on / off where the probe confirms a usable draft head; otherwise the pre-registered N/A disclosure path applies.

Probe raw outputs and scheduling records (`sacct`/`squeue`) are hash-archived. The §12 addendum records **rule outputs with their evidence hashes** — the search-space boundary is fixed at freeze time; only its arithmetic is executed at ignition.

## 4. Sweep Design (combination-explosion guard; two-phase)

1. **Phase 1 — direction screening**: OFAT (one-factor-at-a-time) from official defaults, one knob at a time, **3 paired runs per configuration**. Advancement reads the **point estimate only** (screening mean ratio > 1.00) — a deliberately permissive gate.
2. **Phase 2 — confirmation**: only screening-positive knobs advance, **topped up to 6 runs**. **All CI-based decisions occur here**: a knob whose confirmation 95% CI lower bound ≤ 1.00 is eliminated — no combination round.
3. **Combination round**: combine only knobs individually positive in confirmation, **at most 3 combos**, 6 runs each. `combo_1` = the strongest level of each positive knob (one level per knob) — the single headline configuration.
4. A knob unavailable on a track follows the pre-registered **N/A disclosure path**; no in-window code changes.
5. All level tables, scan order, and elimination rules are pinned by this document; no additions or removals within the window.

**Screening-power disclosure**: screening at N=3 is underpowered for small effects *by design*. False positives are caught in confirmation; the false-negative risk (dropping a weak-but-true knob) is a pre-registered cost-discipline tradeoff, disclosed here. Statistical rigor lives in the confirmation and verdict layers (N ≥ 6, 100,000-resample paired bootstrap) — no claim in the release package rests on 3-run evidence.

## 5. Measurement Protocol

- **Workload**: Azure LLM Inference Trace 2024 (conv + code dual service, CC-BY). Slicing and subsampling follow the frozen Pilot-002 procedure (`TRACE_SEED=20261010`, hour-aligned windows, nested seeded subsampling). The plan file is rebuilt offline and reconciled bit-exact against the Pilot-002 anchor chain (terminal link `d9f94896…`); the 1.8 GB raw CSV never touches a cluster. Track A and Track B use identical workload shapes so knob responses are comparable.
- **Metrics**: per-request TTFT / TPOT / E2E, P95/P99 tail latency, SLO goodput (P95 TTFT ≤ 5 s and TPOT ≤ 100 ms, per the Pilot-002 definition).
- **Statistics**: paired bootstrap (100,000 resamples, seed pinned in §9); block-crossed pairing with odd/even first-mover alternation to cancel time drift.
- **Sequestration discipline**: scan scripts physically contain no CI/P95 computation; verdicts occur for the first time in an independent stage-23 script (structural anti-p-hacking).
- **Quality layer**: this pilot does not touch weights, but batching changes numerical paths, so outputs are not bit-deterministic — pre-registered disclosure of this fact. Two-part quality screen:
  - (a) 150-item frozen set (reused from Pilot 001, sha256 `728b2f83…5cf7d`), dual-arm text pairs; report verbatim-match rate plus descriptive differences — zero judge cost;
  - (b) 50-sample blind evaluation. **Single judge: `deepseek-chat-v3-0324`** (certified on both English and Arabic judging lines in prior pilots). Fallback chain, endpoint-failure only, logged and disclosed: `glm-5.3-prime` → `gpt-6-astra-pro`. All judge inferences run at **temperature 0 with pinned generation settings**; the frozen prompt is reused byte-identical from Pilot 002; raw responses are fully archived for third-party re-audit.
  - **Calibration slice** (drift alarm, not a vote): 10 of the 50 samples (deterministic selection — every 5th item of the seeded sample order) are additionally scored by both fallback judges, each item evaluated twice per judge in **position-swapped** order. Inter-judge agreement and position-swap self-consistency are reported **descriptively**. The quality gate reads only the primary judge.
- **Run validity (mechanical rules)**: invalidation or rerun requires contemporaneous platform evidence (cluster side: `sacct`/`squeue` scheduling records plus node-state capture), symmetric in both directions. The intervention definition and intervention-list appendix are published in the same repository as this protocol.
- **Node trust**: any power- or energy-adjacent measurement on Track A must avoid the untrusted-node list (`f11r1n09`, `f11r2n06`, per the current DCU rehearsal record). This pilot's primary metrics are latency/throughput; **power is descriptive only**.
- **Execution split**: on Track A, stage 24 generates text pairs only; after the evidence package is transferred off-cluster, blind judging runs on a cloud CPU pod.

## 6. Pre-Registered Acceptance Criteria

- **Primary criterion**: best combination (`combo_1`) vs official defaults — geometric-mean speedup 95% CI lower bound ≥ **1.15×**, and P95 tail latency non-inferior (≤ 105% of default).
  - Rationale: first measurement on a domestic cluster, threshold set at 1.15×. Below Pilot 001's 1.20× **deliberately** — the conservative tier carries zero precision risk; its commercial value is "free speedup," and 1.15× is a sellable recipe.
  - **1.15× is a threshold, not a target**: the sweep runs to the measured ceiling. The descriptive section publishes every candidate configuration's point estimate and interval **including negative results** — no hidden defeats.
- **FAIL semantics** (negative-lab-report semantics): FAIL = under the pre-registered procedure, budget, and statistical discipline, the sweep found no configuration meeting the threshold. It is neither proof of a vLLM deficiency nor a denial of Effiq capability.
- **FAIL is an asset**: an all-negative sweep is publishable intelligence — evidence that "vLLM defaults are already near-optimal for this workload family."

## 7. Governance and Publication

- **Public surface — two touchpoints only**: the frozen protocol, and the verdict. Zero intermediate disclosure, zero ceremonial addenda. Foreseeable branches are pre-written into this document (internalized branch tree); executing an in-protocol branch is not an amendment. Only a genuine surprise not covered by the protocol triggers an amendment, and it must be logged.
- Repository: `effiq/pilot007`, MIT from creation. Repository language: English.
- **Publication**: release package (VERDICT + REPRODUCE) within ≤ 7 days of results. FAIL titles never carry model names.
- **Track-A disclosure policy**: Track A's verdict (PASS/FAIL) will be disclosed; detailed Track-A measurements may remain proprietary telemetry pending a separate publication decision recorded in the repository. (Decision of record: maintainer, 2026-11-06. This retention is pre-declared here and the verdict sign is committed to publication regardless of outcome — pre-declared retention is not post-hoc selection.)
- **Novelty gate**: this pilot introduces no new mechanism (system-level knob measurement is dense prior art). Publication follows the standard dual-track review, pre-publication scanner, and MANIFEST discipline; the gate is checked once before this protocol is published.

## 8. Execution Mechanics — Self-Chaining Blocks

- Block cadence: 60-minute job blocks, soft rails `MAX_SECONDS=3300`, `STOP_NEW_BLOCKS_S=2400`; checkpoint granularity = single arm-block; idempotent resume.
- **Self-chaining**: each job submits the next block before walltime expiry (ignition-day probe verifies cluster permission; if denied, block length rises to 4–6 h with manual re-submission).
- Four guardrails: stop on COMPLETE / STOP file / hard cap on block count (≤ window + budget envelope) / chain-break alert on a zero-progress block.

## 9. Pinned Statistical Parameters (no changes within the window)

| Parameter | Value | Note |
|---|---|---|
| THRESH (primary CI lower bound) | 1.15 | threshold, not target |
| BOOT_N (bootstrap resamples) | 100,000 | same as Pilot 002 |
| BOOT_SEED | 20262007 | pinned for this pilot |
| N_BASE / N_MAX (confirmation runs) | 6 / 10 | extended from Pilot 002 |
| N_SCREEN (screening runs) | 3 | permissive gate, point estimate only (§4) |
| ADVANCE_RULE | screening mean ratio > 1.00 | CI decisions at confirmation only |
| ELIM_RULE (confirmation) | 95% CI lower bound ≤ 1.00 | N ≥ 6 |
| MAX_COMBOS | 3 | |
| P95_TOL (tail-latency non-inferiority) | 1.05 | same as Pilot 002 |
| TRACE_SEED (workload slicing) | 20261010 | frozen Pilot-002 procedure |
| FROZEN_SHA (150-item frozen set) | `728b2f83…5cf7d` | reused from Pilot 001 |
| BLIND_N / BLIND_SEED / SAMPLE_SEED | 50 / 202611043 / 202611044 | blind evaluation |
| CALIB_N (calibration slice) | 10 (every 5th item) | drift alarm, descriptive only |
| Judge decoding | temperature 0, pinned settings | frozen prompt from Pilot 002 |
| Block cadence / soft rails | 60 min / `MAX_SECONDS=3300`, `STOP_NEW_BLOCKS_S=2400` | self-chaining per §8 |

## 10. Known Limitations (honest list)

1. Configuration optima drift with workload shape — this pilot's output is "the optimal interval for this workload family, plus the selection method," not a universal configuration. Customer workloads still require measured pricing.
2. vLLM upgrades may stale knob responses — recipes are always released pinned to a version; re-testing after staleness is a separate pilot.
3. Speculative decoding is likely N/A on both tracks (Pilot 001 demonstrated no usable registered head for the 14B model; Track-A EAGLE availability awaits ignition-day probes). If N/A on both, the sweep becomes three knobs — disclosed as-is, no padding.
4. Flag-semantics drift: same-named flags in Hygon vLLM 0.21.0 vs vLLM 0.31.0 may differ in meaning; each flag is verified against live `--help` output before ignition, with the verification record attached as an addendum.
5. Image/environment drift: the spec pins image sha256 and model revision; a zero-drift assertion runs on ignition day.
6. Cluster load tides — shared-cluster queueing and neighbor noise: block-crossing and first-mover alternation absorb time drift; run timestamps and node names are fully logged.
7. Boot time dominates cost — compressed by the two-phase design; the upper-bound case still fits the window and is stated honestly.

## 11. Reproducibility Anchors

- Frozen set and Pilot-002 anchor-chain manifest: extracted from the Effiq pilot-logs repository and hash-verified at ignition.
- Judge prompt: byte-identical reuse from Pilot 002 (hash recorded at ignition); judge decoding at temperature 0 with pinned generation settings.
- All run logs, scheduling records, and raw judge responses are archived for third-party re-audit.

## 12. Ignition-Day Pin List (LOCK_AT_* — backfilled before the window opens)

| Item | When pinned |
|---|---|
| Track-A container image sha256 | measured on ignition day |
| Qwen3.8-27B model revision (40-hex) | ignition day |
| Track-A knob absolute levels | **computed by the frozen rule of §3** from probe evidence; probe outputs hash-archived, values recorded as rule outputs in the addendum |
| Three probe results (EAGLE / ngram offline bound / FP8 smoke) | same window as stage 20, into addendum |
| Pilot-002 anchor-chain manifest + 150-item frozen set | extracted from pilot-logs, hash-verified |
| Flag-semantics verification record | live `--help` check before ignition |

---

## Appendix A — External Review Dispositions

External review round (Google Gemini, 2026-11-05/06): verdict **NO-SHIP**, 2 blockers / 1 major / 1 minor. Dispositions:

1. **[BLOCKER] Track-A levels unpinned at freeze (§3/§12)** → **Fixed**: levels are now deterministic outputs of a frozen derivation rule applied to hash-archived probe evidence (§3). No post-freeze discretion over the search space.
2. **[BLOCKER] Track-A telemetry retention (§7)** → **No change (maintainer decision of record, 2026-11-06)**: retention is pre-declared in this protocol and the verdict sign is committed to publication regardless of outcome; pre-declared retention is not post-hoc selection.
3. **[MAJOR] Single-judge drift risk (§5)** → **Fixed**: temperature-0 pinning stated explicitly; calibration slice (10 samples × fallback judges × position swapping) added as a descriptive drift alarm. The quality gate remains single-judge by design — the quality layer is a descriptive screen, while the pilot's claims rest on objective latency/throughput metrology.
4. **[MINOR] 3-run screening power (§4)** → **Clarified, runs unchanged**: screening is a permissive point-estimate gate; all CI-based decisions occur at confirmation (N ≥ 6). The false-negative tradeoff is now disclosed explicitly in §4.

---

*This protocol was frozen on 2026-11-05 and amended once on 2026-11-06 (v1.1, external-review adjudication). The only permitted post-v1.1 changes are the §12 backfills and any logged amendments under §7.*
