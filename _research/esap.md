---
title: "ESAP Gen AI × Bio: An End-to-End In Silico Pipeline for Kinase Lead Discovery"
date: 2026.07
layout: single
categories: research
sidebar:
  nav: "main"
mathjax: true
---

# ESAP Gen AI × Bio — An End-to-End In Silico Pipeline for Kinase Lead Discovery — Project Explainer

This file explains the three-stage project completed for the Engineering Summer Academy at Penn (ESAP) 2026, Gen AI × Bio.

- **Title:** *ESAP Gen AI × Bio: An End-to-End In Silico Pipeline for Kinase Lead Discovery*
- **Author:** Jayden Jung (Brea Olinda High School)
- **Code:** [github.com/jaydenjung-bohs09/ESAP](https://github.com/jaydenjung-bohs09/ESAP)

---

## 1) One-paragraph summary

Machine-learning drug discovery is usually taught as three separate exercises: predict a property, generate a molecule, validate a structure. This project runs all three as one pipeline and holds them to a single rule — **the model that generates must never be the model that grades**. Stage 1 builds a sequence-to-property regressor and asks what a protein representation actually has to encode in order to beat a mean-predicting baseline. Stage 2 takes a frozen, unconditional generative model (GenMol) and steers it toward stronger JAK2 binders using physics-based QuickVina2 docking as the only feedback signal, comparing a genetic algorithm against a population-wise exponential-tilting sampler under a matched oracle budget. Stage 3 drops the generator entirely and validates a known JAK2 ligand against experimental PDB evidence and an independent Boltz-2 co-folding prediction. Every number reported downstream comes from a scorer that had no hand in producing the molecule it scores.

---

## 2) Motivation

A generative model that is trained, tuned and then evaluated against its own learned scoring function will always look excellent. The number it reports measures the scorer, not the molecule. This is the failure mode the project is organized around, and it is why the pipeline separates three roles that are often collapsed into one:

- the **representation** that turns a biological sequence into numbers,
- the **generator** that proposes molecules, and
- the **oracle** that decides whether a proposal is any good.

Keeping the third strictly outside the first two is the whole design. It costs sample efficiency and it costs runtime, but it is the only arrangement in which a reported improvement means anything.

---

## 3) Problem definition

### Stage 1 — sequence to property

Given a protein sequence $$s = s_1 s_2 \cdots s_L$$ over the 20-letter amino-acid alphabet $$\mathcal{A}$$, predict a continuous target $$y \in \mathbb{R}$$. The difficulty is structural rather than statistical: $$L$$ varies from protein to protein, while a Random Forest requires a fixed-length input. The design question is what a fixed-length encoding $$\phi(s)$$ has to preserve.

### Stage 2 — goal-directed lead optimization

Given a seed molecule $$x_0$$ that is a known binder and a black-box scoring function, find molecules $$x$$ that bind more strongly while remaining drug-like, synthesizable, and recognizably related to the seed. The generator is frozen and unconditional: it has no notion of binding affinity anywhere in its forward pass.

### Stage 3 — independent structural validation

Given a protein target and a candidate ligand, ask two questions using evidence the optimizer never saw: does experimental precedent exist in the Protein Data Bank, and does an independent structure predictor place the ligand in the pocket with confidence?

---

## 4) Proposed method

### 4.1 Two protein representations

**One-hot composition.** Each coordinate is the fraction of one amino acid in the sequence:

$$
\phi_{\text{OH}}(s)_a = \frac{1}{L}\sum_{i=1}^{L} \mathbb{1}[s_i = a], \qquad a \in \mathcal{A}, \qquad \sum_{a \in \mathcal{A}} \phi_{\text{OH}}(s)_a = 1.
$$

This treats amino acids as 20 unrelated identities. Leucine and isoleucine are as different from each other as leucine is from aspartate.

**VHSE average.** Each residue instead maps to eight *Vectors of Hydrophobic, Steric and Electronic* descriptors $$v(\cdot) \in \mathbb{R}^8$$, averaged over the sequence:

$$
\phi_{\text{VHSE}}(s) = \frac{1}{L}\sum_{i=1}^{L} v(s_i) \in \mathbb{R}^{8}.
$$

Individual VHSE dimensions are less interpretable than "fraction of alanine," but the representation compresses correlated physicochemical properties into eight dimensions and, crucially, places chemically similar residues near one another.

Both encodings destroy length information by averaging, so $$L$$ is appended as one extra feature, giving design matrices with 21 and 9 columns respectively. Both are fed to the same Random Forest with the same hyperparameters, so the comparison isolates the representation.

### 4.2 Steering a frozen generator

The generator is GenMol, a discrete-diffusion model over SAFE molecular strings, used entirely unconditionally. The objective is QuickVina2 docking against a fixed JAK2 pocket, converted into a reward that always points the same way:

$$
r(x) = \max\big(0,\ -\mathrm{DS}(x)\big),
$$

where $$\mathrm{DS}(x)$$ is the docking score and a failed docking returns $$99.9$$, hence reward $$0$$.

**Why the guidance lives outside the sampler.** The three standard options for steering a diffusion model all fail here, and they fail for the same reason. Classifier guidance needs a differentiable predictor that can score noisy, half-masked sequences. Classifier-free guidance needs retraining with the property as a conditioning input, which ties the model to one objective. Per-step exponential tilting keeps the model frozen but still needs a reward estimate at every denoising step. All three require scoring an incomplete molecule, and **you cannot dock half a molecule**.

So the model stays frozen and the steering is lifted out of the sampler up to the level of a *population* of finished molecules. The oracle is only ever called on complete molecules and only has to rank them, which means it can be any black box at all — including a physics simulator with no gradients. What is given up is per-step control and sample efficiency; what is gained is the freedom to swap in any scoring function unchanged.

### 4.3 Search A — genetic algorithm

Each round performs four steps:

1. **Attach (crossover).** Two fragments are drawn from the pool and joined with the reaction SMARTS `[*:1]-[1*].[1*]-[*:2]>>[*:1]-[*:2]`. This step is blind to the objective.
2. **Remask (mutation).** One fragment of the product is masked and regenerated by GenMol. This is what lets the search reach fragments that were never in the starting vocabulary.
3. **Score.** Dock the candidate and compute QED, synthetic accessibility, and Tanimoto similarity to the seed.
4. **Select and enrich.** A candidate is kept only if it clears every gate at once:

$$
r(x) > r(x_0), \qquad \mathrm{QED}(x) \ge 0.6, \qquad \widetilde{\mathrm{SA}}(x) = \frac{10 - \mathrm{SA}(x)}{9} \ge \frac{6}{9}, \qquad T(x, x_0) \ge \delta,
$$

where $$\widetilde{\mathrm{SA}} \ge 6/9$$ is exactly the constraint $$\mathrm{SA} \le 4$$ rewritten on a flipped 0-to-1 scale, and $$T$$ is Tanimoto similarity over Morgan fingerprints of radius 2 and 2048 bits:

$$
T(x, x_0) = \frac{|F(x) \cap F(x_0)|}{|F(x) \cup F(x_0)|}.
$$

Surviving leads are cut back into fragments and returned to the pool. A fragment appearing in many leads is added many times, so it is more likely to be drawn next round, and the pool drifts toward substructures that build good binders. That feedback loop — selection plus fragment reuse — is what turns a blind generator into a goal-directed one.

### 4.4 Search B — population-wise exponential tilting

The contrasting baseline targets the frozen model tilted by the reward,

$$
\pi(x) \propto p_\theta(x)\,\exp\!\left(\frac{r(x)}{\lambda}\right),
$$

realized by sampling-importance-resampling over **complete** molecules rather than inside the diffusion trajectory. Each round mutates every member of the population using the same remasking call the genetic algorithm uses, scores the batch, and resamples with weights

$$
w_i = \frac{\exp\big(r(x_i)/\lambda\big)}{\sum_j \exp\big(r(x_j)/\lambda\big)},
$$

where any molecule violating a constraint is assigned $$r = -\infty$$ and therefore zero weight. The temperature $$\lambda$$ sets how aggressive the selection pressure is: small $$\lambda$$ is greedy and collapses the population onto a few high scorers, large $$\lambda$$ barely tilts and drifts toward unbiased sampling.

Generator, oracle, constraints and oracle budget are all held fixed between the two searches, so the only difference is soft temperature-weighted resampling of whole molecules versus a fragment pool with hard accept-or-reject.

### 4.5 Independent validation

The demonstration pair is human kinase JAK2 (UniProt `O60674`) with tofacitinib, a marketed small-molecule JAK inhibitor. Validation proceeds along two independent lines:

1. **Experimental precedent.** Query RCSB PDB by UniProt accession, retrieve every non-polymer entity, and filter to drug-like candidates with an operational heuristic: not in a known cofactor, solvent, buffer or ion exclusion list; formula weight at least 150 Da; and containing carbon. The heuristic is audited in both directions — what it retained *and* what it discarded — because the PDB has no field meaning "drug."
2. **Structure prediction.** Submit the JAK2 sequence together with the tofacitinib SMILES to Boltz-2 for co-folding with affinity prediction, then read back ipTM, complex pLDDT, predicted binding affinity and binding probability.

Boltz-2 is chosen precisely because it appears nowhere upstream in the pipeline.

---

## 5) Experimental settings

- Environment: Google Colab Pro, GPU for generation, CPU for docking
- Generator: GenMol V1, `nvidia/NV-GenMol-89M-v1`, 89M parameters, frozen
- Docking: QuickVina2 at `exhaustiveness = 1`, the packaged JAK2 receptor and binding box
- Seed molecule: `OCCCCc2nc1ccccc1c4ncnc3[nH]cc2c34`, a known JAK2 active with reference docking score 7.7
- Constraints: $$\mathrm{QED} \ge 0.6$$, $$\mathrm{SA} \le 4$$, $$\delta = 0.4$$
- Budget: approximately 60 oracle calls per search method, against the reference implementation's 1,000
- Regression: 300 proteins, 80/20 random split, Random Forest with 200 trees, fixed seed

---

## 6) Evaluation metrics

Stage 1 reports mean absolute error, root mean squared error, and the coefficient of determination:

$$
\mathrm{MAE} = \frac{1}{n}\sum_{i=1}^{n} |y_i - \hat{y}_i|, \qquad
\mathrm{RMSE} = \sqrt{\frac{1}{n}\sum_{i=1}^{n} (y_i - \hat{y}_i)^2}, \qquad
R^2 = 1 - \frac{\sum_i (y_i - \hat{y}_i)^2}{\sum_i (y_i - \bar{y})^2},
$$

each against a baseline that always predicts the training mean.

Stage 2 reports best constrained docking reward and the number of distinct leads, plus a molecular-quality panel: validity, uniqueness, internal diversity

$$
\mathrm{IntDiv} = 1 - \frac{2}{n(n-1)}\sum_{i<j} T(x_i, x_j),
$$

Bemis–Murcko scaffold counts, mean MW, cLogP, QED and SA, Lipinski compliance, and the fraction free of PAINS structural alerts.

Stage 3 reports ipTM, complex pLDDT, predicted binding affinity, and binding probability.

---

## 7) Key results and interpretation

### Stage 1 — representation matters more than dimension

| Model | MAE ↓ | RMSE ↓ | R² ↑ |
| :--- | :--- | :--- | :--- |
| Baseline (always predict training mean) | 2.329 | 2.913 | −0.016 |
| One-hot composition + length | 2.057 | 2.567 | 0.212 |
| **VHSE average + length** | **1.973** | **2.540** | **0.228** |

Both learned models clear the baseline, and the 8-dimensional VHSE encoding edges out the 20-dimensional one-hot composition on every metric. Fewer features win because those features carry physicochemical meaning rather than identity alone — which is the point of the comparison, not an incidental detail.

### Stage 2 — two search mechanisms, matched budget

| Method | Seed affinity | Best lead affinity | Δ vs. seed | Distinct leads |
| :--- | :--- | :--- | :--- | :--- |
| Genetic algorithm | 7.800 | 8.800 | +1.000 | 7 |
| **Exponential tilting** ($$\lambda = 1.0$$) | 7.800 | **9.500** | **+1.700** | **9** |

### Stage 2 — quality against an unguided generator

| Method | Validity | Uniqueness | IntDiv | Scaffolds | QED | SA ↓ | Lipinski | No-PAINS |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Unguided generator | 1.000 | 0.983 | 0.677 | 39 | 0.510 | 3.39 | 0.898 | 0.983 |
| GA (guided) | 1.000 | 0.983 | **0.771** | 38 | **0.517** | **3.28** | 0.932 | 0.949 |
| Exponential tilting (guided) | 1.000 | 0.900 | 0.696 | 37 | 0.509 | 3.57 | **1.000** | 0.963 |

The interesting result here is what guidance did *not* cost. A common worry about goal-directed generation is mode collapse — the search finds one good scaffold and grinds on it. The genetic algorithm is instead *more* internally diverse than the unguided generator at the same sample count (0.771 against 0.677), because fragment recombination reaches combinations that blind sampling does not. Exponential tilting pays for its higher peak affinity in uniqueness (0.900 against 0.983), which is the expected signature of a population concentrating on high scorers.

### Stage 3 — corroboration, not proof

| Evidence | Result |
| :--- | :--- |
| PDB entries mapped to UniProt `O60674` | 165 |
| Non-polymer entity records retrieved | 246 |
| Unique chemical components | 146 |
| Candidate drug-like ligands after filtering | 131 |
| Tofacitinib in the PDB | component `MI1`, entry `3FUP` |

| Boltz-2 metric (best of 5 models) | Value |
| :--- | :--- |
| Confidence score | 0.866 |
| ipTM | 0.912 |
| Complex pLDDT | 0.854 |
| Predicted binding affinity | −10.62 kcal/mol |
| Binding probability | 0.850 |

An ipTM of 0.912 on a 1,132-residue kinase indicates a confidently placed ligand, and the predicted affinity is consistent with tofacitinib being a genuine JAK2 inhibitor. One oracle agreeing with one line of experimental precedent is corroboration, however, and not proof.

---

## 8) How to reproduce the core experiments

1. Run the Stage 1 notebook end to end; it needs only a CPU runtime and no external credentials.
2. For Stage 2, select a GPU runtime, let the install cell restart the runtime once so the pinned `numpy` takes effect, then rerun it. Set `TARGET = 'jak2'` and `START_IDX = 0` to reproduce the reported task.
3. Run the genetic algorithm and the tilting baseline with identical `sim_thr`, `qed_thr`, `sa_thr` and a matched oracle budget, then compare with the paired comparison cell.
4. For Stage 3, supply a Tamarind Bio API key at runtime through `getpass()` — never in a committed cell — and submit the Boltz-2 job once, since it is quota-limited.
5. Read `metrics.csv` for ipTM and predicted affinity, and inspect `best.pdb` interactively with `py3Dmol`.

---

## 9) Limitations

Stated plainly, because the numbers above are easy to over-read.

1. **The Stage 1 target is synthetic.** The stability score is generated from a known formula, so $$R^2 \approx 0.23$$ measures how well an encoding recovers a designed signal, not real protein stability. Random splitting also permits sequence-similarity leakage between train and test.
2. **Docking scores are not binding affinities.** QuickVina2 at `exhaustiveness = 1` is a fast ranking signal, and docking quality depends far more on receptor preparation and box placement than on anything inside the search loop.
3. **The search comparison is illustrative.** One seed, one temperature, roughly 60 oracle calls against the reference budget of 1,000. The genetic algorithm also holds a structural advantage the tilting loop lacks — crossover between different good molecules — so this is not a clean test of the selection rule in isolation.
4. **Computational predictions are hypotheses.** Predicted structures and affinities are not experimental proof of binding, potency, selectivity, or biological activity. Strong in silico validation requires agreement among several independent lines of evidence.

---

## 10) Next steps

- Run Stage 2 at the full reference budget and sweep the tilting temperature $$\lambda \in \{0.3,\ 1.0,\ 3.0\}$$.
- Add crossover to the tilting proposal, or remove it from the genetic algorithm, to isolate the selection rule from the recombination advantage.
- Replace the synthetic Stage 1 target with a measured property and use a sequence-identity-aware split.
- Close the loop by routing Stage 2 leads through the Stage 3 Boltz-2 oracle and reporting agreement between docking rank and co-folding confidence.

---

## 11) Provenance and references

The Stage 1 and Stage 3 notebooks are built on workbook scaffolds provided by the ESAP 2026 Gen AI × Bio course; the analyses, parameter choices, extensions and all reported results are this project's own. The Stage 2 notebook reproduces and extends Section 5.4 of the GenMol paper using the receptor files, seed actives and docking binary that ship with the upstream repository.

- Lee et al., *GenMol: A Drug Discovery Foundation Model*, [arXiv:2501.06158](https://arxiv.org/abs/2501.06158), ICML 2025
- [NVIDIA-BioNeMo/genmol](https://github.com/NVIDIA-BioNeMo/genmol), reference implementation and packaged docking targets
- Boltz-2, protein–ligand co-folding with affinity prediction
- Alhossary et al., *QuickVina 2*, fast and accurate protein–ligand docking
- Mei et al., VHSE descriptors, vectors of hydrophobic, steric and electronic properties
- [RCSB PDB](https://www.rcsb.org/) and [UniProt](https://www.uniprot.org/) search and data APIs
