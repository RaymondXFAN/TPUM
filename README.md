# TPUM — Temporal Privacy-Unit Mismatch in Differentially Private Federated Human-Activity Recognition

> **One-liner.** A nominal ε = 8 can cost 135.  
> TPUM audits how the *privacy-unit accounting convention* — not the noise, not the model,  
> not the data — decides what a differential-privacy budget actually buys in federated  
> smart-home human-activity recognition.

---

## Abstract

TPUM is an empirical audit of how the choice of privacy unit changes what a nominal  
differential-privacy budget actually buys in federated human-activity recognition (HAR)  
from smart-home sensor streams. We work on 80 CASAS homes, with NEX, Ordonez and  
Mi4A-HomeData used as cross-dataset controls. We first calibrate the temporal correlation  
length τ of the activity-label stream **in continuous time** — CASAS gives τ = 2919.4 s  
(95 % CI [2437, 3546]) — and show that converting τ into the window unit requires dividing  
by the **realised** inter-window spacing of the cache, not by the nominal stride, because  
the window builder silently drops unlabelled and zero-event windows (for CASAS stride 1800  
the realised spacing is 2775.9 s, not 1800 s; for stride 600 it is 925.4 s, not 600 s).  
Feeding this calibrated τ into a subsampled-Gaussian RDP accountant, we run a three-arm  
controlled experiment — arm A: stride 1800 with calibrated τ = 1.0517; arm B: stride 600  
with calibrated τ = 3.1547; arm C: stride 600 with the naive "one window = one privacy  
unit" convention τ = 1.0 — at ε ∈ {5, 8} × 3 seeds, and then **recompute every arm's true ε  
using one shared accountant, one seed, and one protocol**. All three arms report ε ≈ 8, but  
arm C's true cost is 135.26, a **16.55× overspend**, and that overspend is exactly what buys  
its +62.5 % utility advantage. Because arm C's utility is statistically indistinguishable  
from that of the correctly calibrated arm A (paired *t*, *p* = 0.671), **utility alone  
cannot tell the two systems apart** — which is the paper's central claim.

---

## Key results

### Three arms, one accountant (the headline table)

Same accountant (`fed_train.py` `RDPAccountant` + `rdp_vector`, temporal mode), same  
MC seed = 0, `n_rep = 200`, `B = 128`, `R = 50`, `m = 10`, `δ = 1e-5`.

| Arm | stride | cache           | τ (accounting)                         | σ (ε8 / ε5)   | ε **claimed** | ε **true**             | macro-F1 @ε8    | macro-F1 @ε5    |
| --- | ------ | --------------- | -------------------------------------- | ------------- | ------------- | ---------------------- | --------------- | --------------- |
| A   | 1800   | `cache_g4`      | **1.0517** (calibrated, 2919.4/2775.9) | 2.989 / 4.468 | 8.001 / 5.000 | **8.1734 / 5.1138**    | 0.1110 ± 0.0206 | 0.1017 ± 0.0141 |
| B   | 600    | `cache_g4_s600` | **3.1547** (calibrated, 2919.4/925.4)  | 5.314 / 7.657 | 7.999 / 5.000 | **8.2295 / 5.1418**    | 0.0703 ± 0.0250 | 0.0623 ± 0.0221 |
| C   | 600    | `cache_g4_s600` | **1.0** (naive: one window = one unit) | 1.684 / 2.427 | 8.002 / 5.001 | **135.2567 / 31.5077** | 0.1142 ± 0.0093 | 0.1054 ± 0.0120 |

**Overspend ratios — always state the denominator** (reviewers recompute these):

| Ratio              | ε8                             | ε5                       |
| ------------------ | ------------------------------ | ------------------------ |
| C_true / A_true    | **16.55×** (135.2567 / 8.1734) | 6.16× (31.5077 / 5.1138) |
| C_true / B_true    | **16.44×** (135.2567 / 8.2295) | 6.13×                    |
| C_true / nominal 8 | **16.91×**                     | 6.30×                    |

### The three claims

1. **A vs B — the physics of the unit.** Holding true ε ≈ 8 fixed and only changing the  
   sampling stride, going 1800 → 600 drops utility from 0.1110 to 0.0703 (−36.7 %).  
   Mechanism: one privacy unit spans 3.1547 windows on the stride-600 cache, so the same  
   unit is sampled ~3× and the noise must be paid ~3×. Paired *p* = 0.0140.
2. **B vs C — the accounting convention.** Same data, same stride, only the accounting  
   convention differs. Arm C spends 16.4× more privacy than B while claiming the same  
   ε = 8, and that is what buys its +62.5 % utility. The naive convention is **not  
   conservative here; it is a severe overspend**.
3. **C vs A — the closing argument.** Arm C (0.1142) and arm A (0.1110) are statistically  
   indistinguishable (paired *p* = 0.671), yet their true privacy costs differ by 16.55×.  
   **Utility cannot discriminate between these two systems.**

### Statistical protocol (this matters — do not use pooled-t here)

Arm B has sd = 0.0216 and arm C has sd = 0.0107 — a 2× variance ratio, so a pooled  
two-sample *t*-test is invalid; Welch is required. More importantly, B and C share the same  
data, the same `default_rng(seed)` client sampling, and differ only in τ, so the **correct  
test is paired**:

| Test                              | *p*                                                   |
| --------------------------------- | ----------------------------------------------------- |
| Welch, ε = 8 only (n = 3)         | 0.0790 (n.s.)                                         |
| Welch, ε = 5 only (n = 3)         | 0.0572 (n.s.)                                         |
| Welch, pooled over both ε (n = 6) | 0.00273, Cohen's *d* = 2.55                           |
| **Paired, 6 pairs**               | **0.01580, 95 % CI [+0.0123, +0.0747]** ← report this |
| Sign test                         | 6/6 differences positive, *p* = 0.031                 |

### Honest negative results (do not hide these)

- **No configuration beats the prior-matched random baseline of 0.1176**  
  (uniform random = 0.0709). 3 of 18 runs fall below uniform random, all in arm B:  
  0.0431, 0.0407, 0.0614.
- **The constant-predictor macro-F1 of 0.4443 on NEX is an artifact of the reporting  
  rule, not a capability.** With `min-support-macro = 1000` only Eat (6750) and OutOfHome  
  (1068) enter the macro average; OutOfHome has F1 = 0 and Eat has F1 = 0.888, so  
  macro-F1 ≡ F1(Eat)/2 = 0.444. The identity holds on all 6/6 checkable configurations.
- **On NEX, OutOfHome (1068 test windows, the second largest class) has recall = 0 in  
  7/7 configurations**, including the strongest intervention available — non-DP, no noise,  
  `class-weight-cap = 100`. The collapse on NEX is therefore a **class-separability  
  limit**, not a hyper-parameter, sampling, or DP artifact.
- NEX "8/8 window configurations have τ_win < 1" holds **only at the point estimate**:  
  the (1800, 300) configuration has a CI upper bound of 1.3265 > 1. State this caveat.

---

## Repository layout

```
.
├── fed_train.py            # Trainer + DP accountant (monolithic, ~345 KB). The single
│                           #   source of truth for RDPAccountant / rdp_vector.
├── sigma_solve.py          # sigma <-> eps solver. Re-uses fed_train's REAL accountant.
│                           #   NOTE: positional-arg CLI, NOT argparse.
├── _acct_t1b.py            # Three-arm same-protocol true-eps recomputation (CPU only).
├── n2_unit_calib.sh        # N2 three-arm calibration driver (the headline experiment).
├── n4_v2.sh                # NEX rescue Stage 1 driver.
├── waiting_batch.py        # Concurrent batch runner (respects a free-memory guard).
├── finalize_tau_all.py     # tau calibration aggregation across datasets.
├── build_partition.py      # Federated client partitioning.
├── instance_duration.py    # Activity-duration statistics (motivates the window study).
├── convert_mi4a.py         # Mi4A-HomeData format conversion.
├── g4_eps_v2.py            # eps prediction utilities.
├── g4_predict_eps.py       # eps prediction utilities (earlier variant).
├── ablation_*.py / .sh     # Ablation tiers.
├── fig_s45.py              # Figure generator (matplotlib) -> figs/fig44..fig48.
├── n2_stats.py             # Statistics: Welch / paired / Cohen's d / sign test.
│                           #   Student-t CDF implemented from scratch
│                           #   (regularised incomplete beta + Lentz continued fraction).
├── view_cloud_results.sh   # One-shot cloud result viewer (modes: gpu|n2|n3|n4|all).
├── figs/                   # fig44..fig48 (300 dpi PNG).
└── nex_tau_win_by_stride.csv
```

### Where the two most important numbers come from

- **τ = 2919.4 s** — `finalize_tau_all.py` + the τ scan under `tau_scan/tau/`.  
  Only a τ that is **convergent in the sampling step dt** is trustworthy: on CASAS, moving  
  dt from 300 s to 5 s moves τ by 0.2 %. By contrast, exponential-fitting τ is an  
  illusion (a true exponential needs τ_half-life = τ_expfit · ln 2; the observed ratios are  
  2.59 / 24.98 / 1.14 / 0.73 for CASAS / NEX / Ordonez / Mi4A instead of 1.00), and the  
  AR(1)-implied τ is invalid because the label channel is not AR(1) (lag-30 autocorrelation  
  0.141 vs. 0.022 predicted).
- **ε_true = 135.26** — `_acct_t1b.py`, or read `outputs/_acct_t1b.json` directly.

---

## Method pipeline

```
raw sensor events
      │
      ├─ build windows (win-sec, stride-sec, label-mode=single)
      │      ⚠ the builder DROPS unlabelled / zero-event windows
      │        => realised spacing ≠ nominal stride
      │
      ├─ calibrate tau (continuous time, dt-convergent)
      │      tau_win = tau_sec / REALISED_SPACING        <-- not / stride
      │
      ├─ solve sigma for a target eps      sigma_solve.py
      │      s_eff = sigma / tau ; eps depends only on s_eff
      │      => matching eps  <=>  sigma ∝ tau   (verified, |Δeps| ≈ 0)
      │
      ├─ federated DP-SGD training         fed_train.py --dp dpsgd --dp-accountant temporal
      │      ⚠ tau is a bookkeeping parameter ONLY; it never enters the noise path
      │
      └─ recompute TRUE eps for every arm with ONE accountant   _acct_t1b.py
```

---

## Datasets

| Dataset              | Role                  | Notes                                                                                                                                   |
| -------------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **CASAS** (80 homes) | Main line             | τ = 2919.4 s, 95 % CI [2437, 3546]. Majority class 30.5 %.                                                                              |
| **NEX**              | Cross-dataset control | τ = 840.4 s, 95 % CI [529, 1533]. Majority class 79.9 %. Test windows = 8444. Collapses to a constant predictor — see negative results. |
| **Ordonez**          | Cross-dataset control | τ ≈ 4299.2 s (provisional).                                                                                                             |
| **Mi4A-HomeData**    | Cross-dataset control | τ ≈ 6376 s (provisional).                                                                                                               |

Cross-dataset τ spread is 7.6×, part of which comes from alphabet skew  
(shannon-entropy baseline H: NEX 0.723 vs CASAS 0.269), so **report both the raw τ and the  
entropy-normalised view**.

---

## Requirements

### Hardware used

| Item | Spec                                                                               |
| ---- | ---------------------------------------------------------------------------------- |
| GPU  | 1 × NVIDIA GeForce RTX 4090 D, 24 564 MiB (training only)                          |
| CPU  | 128 cores (cache building and ε accounting are **CPU-only**)                       |
| RAM  | 503 GB (peak observed per training run ≈ 12.9 GiB; two concurrent runs ≈ 22.9 GiB) |
| Disk | 250 GB; the full experiment tree is ≈ 45 GB excluding raw datasets                 |

### Software

| Package      | Version (as run)        |
| ------------ | ----------------------- |
| OS           | Ubuntu 22.04.5 LTS      |
| Python       | 3.12.3                  |
| PyTorch      | 2.8.0+cu128 (CUDA 12.8) |
| NumPy        | 2.3.2                   |
| pandas       | 3.0.6                   |
| scikit-learn | 1.9.1                   |
| matplotlib   | 3.10.5                  |


Minimal install:

```bash
conda create -n tpum python=3.12
conda activate tpum
pip install torch==2.8.0 --index-url https://download.pytorch.org/whl/cu128
pip install numpy pandas scikit-learn matplotlib
```

> **Xiaohu Fan:** the accountant path is pure NumPy — you do **not** need a GPU to  
> reproduce the headline table. Everything from `sigma_solve.py` and `_acct_t1b.py` runs  
> on a laptop in minutes. Only the three-arm training itself needs the card.

---

## How to run

```bash
# 1) Build the window caches (CPU only — do not waste a GPU slot on this)
python fed_train.py --algo fedavg --rounds 0 --eval-mode holdout --homes 80 \
       --device cpu --rebuild-cache \
       --data-root <CASAS labeled_data> --cache-dir cache_g4 \
       --batch-size 128 --win-sec 1800 --stride-sec 1800 \
       --out-dir _cachewarm --log-dir _cachewarm_logs
# repeat with --stride-sec 600 --cache-dir cache_g4_s600

# 2) Read the REALISED window spacing out of the cache, then set
#       tau_win = 2919.4 / realised_spacing
#    (cache_g4 -> 2775.9 s => tau = 1.0517 ; cache_g4_s600 -> 925.4 s => tau = 3.1547)

# 3) Solve sigma for a target eps
#    ⚠ positional CLI, NOT argparse:
python sigma_solve.py cache_g4_s600 '{"armB": 3.1547, "armC": 1.0}'
#    hard-coded inside: B=128, ROUNDS=50, M=10, DELTA=1e-5, N_REP=200
#    targets solved: 8.0, 5.0, 3.0 -> outputs/sigma_solved.json

# 4) Run the three arms (18 runs: 3 arms x 2 eps x 3 seeds)
bash n2_unit_calib.sh

# 5) Recompute every arm's TRUE eps under one protocol (CPU, minutes)
python _acct_t1b.py          # or read outputs/_acct_t1b.json directly

# 6) Statistics
python n2_stats.py           # Welch / paired / Cohen's d / sign test

# 7) Figures
python fig_s45.py            # -> figs/fig44..fig48 at 300 dpi
```

### Viewing results

```bash
bash view_cloud_results.sh n4     # NEX rescue, with automatic RESCUED/PARTIAL/NOT_RESCUED verdict
bash view_cloud_results.sh n2     # N2 18-run table
bash view_cloud_results.sh gpu    # machine / process status only
bash view_cloud_results.sh all
```

Per-run outputs live under `n2_out/<tag>/`:

| File                              | Contents                                                        |
| --------------------------------- | --------------------------------------------------------------- |
| `fed_fedavg_holdout_runs.csv`     | per-round top-1, macro-F1, ε, rounds                            |
| `fed_fedavg_holdout_perclass.csv` | per-class F1 / recall / support / in-macro flag                 |
| `fed_fedavg_holdout.json`         | final summary incl. `epsilon_final`                             |
| `preds_holdout.csv`               | raw predictions (confusion matrices are reproducible from this) |

---

## Reproducibility notes and known traps

> These cost me real hours. Read them before you touch anything. — Xiaohu Fan

1. **`fed_train.py:4930-4934` clamps τ.** Any `--tau < 1.0` is warned about and forced to  
   1.0. NEX was passed `--tau 0.121186` and all 24/24 runs realised `tau = 1.0`. If you are  
   solving σ to match observed training behaviour, use **1.0**, not 0.121.
2. **τ is pure bookkeeping; it never enters the training path.** Verified by grep: τ appears  
   only at `fed_train.py:1790` (`s_eff = sigma / tau`), L1815 `add_round_max`, L1823,  
   L4793 (argparse) and L5358 (summary). Changing τ does not change the model — it changes  
   the ε label. This is why a fixed-σ run can be re-labelled offline at zero GPU cost.
3. **ε depends only on σ/τ ⇒ matching ε ⟺ σ ∝ τ** (numerically verified, |Δε| ≈ 0).  
   That is why arms B and C have identical claimed ε but σ values differing by 3.15×.  
   **Corollary people get wrong:** B and C are two *independent* training runs, not one run  
   with two labels.
4. **`sigma_solve.py` takes positional arguments, not argparse flags.** A call written as  
   `--target-eps ... --tau ... --json-out ...` will silently fail — I lost a whole batch  
   this way.
5. **`pkill -f PATTERN` kills the shell that issued it** (exit 128), because the invoking  
   shell's own command line contains the pattern. The bracket trick `[n]4_v2` does **not**  
   protect the parent shell. Correct fix: get PIDs with `pgrep` and explicitly skip `$$`  
   and `$PPID`.
6. **Inline `setsid nohup ... &` over plink SIGTERMs the local launcher.** Write a launcher  
   script, upload it, then invoke it in the foreground.
7. **NEX is CPU-bound, not GPU-bound.** At concurrency 6 a single round went from 28 s to  
   629 s, so a 3600 s budget covered only 7 rounds. Concurrency must be ≤ 2.
8. **The memory guard must be smaller than the card.** A default `--free-mb 40000` against a  
   24 564 MiB card means only one job ever starts. Use `--free-mb 3000`.
9. **Cache corruption from a badly timed kill is real.** Killing a process during the  
   per-home `.npy` write pass leaves a half-written cache; every later run then dies with  
   `FileNotFoundError`. Recovery: isolate the bad cache directory and rebuild single-threaded  
   with `--rebuild-cache`.
10. **Two batch drivers writing the same output directory will silently overwrite each  
    other's tags.** One owner per output directory.

---

## Limitations and open items

| # | Open item                                                                                                                                    | Status                                                                                                                                                                                                                                                                            |
| - | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | Decomposing σ 2.989 → 5.314 (1.78×) into "windows per unit ×3" vs. subsampling amplification (N_i 115,588 → 346,632 changes q_i and steps_i) | Current design cannot separate them; needs an ablation holding N_i fixed. Stated as such in the paper.                                                                                                                                                                            |
| 2 | The +2.2 % (A) / +2.9 % (B, C) self-check offset                                                                                             | Confirmed common-mode: `n_rep` 40 vs 200 differs by 0.4 %, seed 0→42 shifts A and B identically, and the offset sits inside the ±3–6 % seed-to-seed band the driver script itself declares. **Relative ratios are unaffected.** Do not write "self-check error 0.000 %" anywhere. |
| 3 | Why ε8 overspends 16.44× but ε5 only 6.13×                                                                                                   | Explained by curvature: ε(s_eff) is convex and steep at small s_eff — the local log-log slope runs from −5.9 (s_eff ≈ 0.5) to −1.26 (s_eff ≈ 2.4). The ε8 starting point sits on the steep segment, ε5 on the flat one.                                                           |
| 4 | `temporal_corr` covered 79 homes, not 80                                                                                                     | An artifact of `--homes 80` truncating at `tm038` in lexicographic order. Should be re-run.                                                                                                                                                                                       |
| 5 | NEX Stage 1 is entirely non-DP                                                                                                               | Zero information about NEX privacy–utility trade-off; the root cause of OutOfHome's recall = 0 (feature separability vs. label quality) is undiagnosed and must be written as "a separability limit", not as a conclusion.                                                        |

> **Xiaohu Fan:** item 3 is the one a reviewer is most likely to poke at, and it is also  
> the one where we genuinely have an answer. Do not let the drafting pass soften it into  
> "further work".

---

## Figures

| File                                 | What it shows                                                                                                                                                                       |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `figs/fig44_arms_macroF1.png`        | Three arms × two ε levels, grouped bars with error bars, prior baseline (0.1176) and uniform-random (0.0709) reference lines; the three arm-B runs below uniform random are marked. |
| `figs/fig45_paired_slope.png`        | B→C paired slopes, 6/6 rising — the paired differences are all positive.                                                                                                            |
| `figs/fig46_utility_vs_eps.png`      | Utility vs **nominal** ε (carries a cross-warning: equal nominal ε ≠ equal privacy spend).                                                                                          |
| `figs/fig47_utility_vs_true_eps.png` | **The visual hammer.** Utility vs **true** ε on a log axis — C is flung far right while sitting level with A.                                                                       |
| `figs/fig48_tau_win.png`             | NEX τ_win across window/stride configurations, from `nex_tau_win_by_stride.csv`.                                                                                                    |

---

## Citation

```bibtex
@misc{fan2026tpum,
  author       = {Xiaohu Fan},
  title        = {TPUM: Temporal Privacy-Unit Mismatch in Differentially Private
                  Federated Human-Activity Recognition},
  year         = {2026},
  note         = {A nominal epsilon = 8 can cost 135: auditing the privacy-unit
                  accounting convention in federated smart-home HAR}
}
```

---

## Author

**Xiaohu Fan** — CS PhD, Associate Professor.  
Works on differential privacy and federated learning for IoT / smart-home sensing, with a  
standing bias toward publishing negative results when they are the honest answer.

> **A note from me, Xiaohu Fan, on how to read this repository.** The headline number is  
> not "our method is better" — it is "the accounting convention you quietly picked decides  
> your real privacy spend, and utility will not tell you that you got it wrong." Every  
> negative result in the table above is in the paper on purpose. If you are reviewing this  
> and you find a place where I softened a bad number into a vague sentence, please flag it;  
> that is the failure mode I am most worried about, far more than the results themselves.  
> The NEX collapse in particular is real and unresolved — it is a separability limit of that  
> dataset, not something a hyper-parameter will fix, and I would rather say so than ship a  
> rescue story that does not survive contact with the per-class numbers.
