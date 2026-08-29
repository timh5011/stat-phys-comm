# Experimentation Plan

This document outlines the high level objectives and low level details of the computational experiments we will run. Our primary goal is to look for equilibrium statistical mechanic macrostate structure in natural language by using encoder-only transformer LLMs to define equivalence classes by semantic meaning.

## Guiding methodology: start bare-bones, build up as needed

Our default research strategy is to begin with the simplest possible model — even one that is naively simple and that we do not expect to succeed — and add complexity only when the simple version demonstrably fails. Concretely: we first attempt to model a dataset with *exact* standard equilibrium statistical mechanics (additive energy, clean canonical / microcanonical assumptions, no interaction terms). If that fails, we then consider the minimal next addition — e.g. interaction terms or effective-energy corrections — and build on top as needed. Always start as bare-bones as possible; let observed failures, not anticipation, drive added complexity.

## Table of Contents: Pillars of the experiment

* [Characterizing the Encoder Model](#characterizing-the-encoder-model)
  * [STSb evaluation](#stsb-evaluation)
* [Getting the Proper Datasets: Datasets will mimic the microcanonical and canonical ensemble microstates](#getting-the-proper-datasets-datasets-will-mimic-the-microcanonical-and-canonical-ensemble-microstates)
  * [Stance: equilibrium, not non-equilibrium steady state (for now)](#stance-equilibrium-not-non-equilibrium-steady-state-for-now)
  * [What "a real community" means here](#what-a-real-community-means-here)
  * [Microstate and the two-ensemble construction](#microstate-and-the-two-ensemble-construction)
  * [Primary target: the canonical ensemble first](#primary-target-the-canonical-ensemble-first)
  * [Dataset constraints (v1)](#dataset-constraints-v1)
  * [The ensemble source is left open](#the-ensemble-source-is-left-open)
  * [Approximations carried into every dataset](#approximations-carried-into-every-dataset)
  * [Coupling, correlations, and the additivity requirement](#coupling-correlations-and-the-additivity-requirement)
  * [The encoder is an instrument, not part of the physics](#the-encoder-is-an-instrument-not-part-of-the-physics)
  * [Meaning as a causal/predictive equivalence](#meaning-as-a-causalpredictive-equivalence)
* [The Validation Plan: What observables will we look for and how do we determine success?](#the-validation-plan-what-observables-will-we-look-for-and-how-do-we-determine-success)
  * [Partitioning the latent space: defining macrostates and choosing a scale](#partitioning-the-latent-space-defining-macrostates-and-choosing-a-scale)
  * [Encoder loss as an instrument-side resolution floor](#encoder-loss-as-an-instrument-side-resolution-floor)
  * [Primary observable](#primary-observable)
  * [Exploratory diagnostics: equilibrium-likeness checks from counts and timestamps](#exploratory-diagnostics-equilibrium-likeness-checks-from-counts-and-timestamps)
  * [Density-of-states estimation: prior art for the sparsity problem](#density-of-states-estimation-prior-art-for-the-sparsity-problem)
  * [Open problems (deferred, developed separately in the theory)](#open-problems-deferred-developed-separately-in-the-theory)

## Characterizing the Encoder Model

Before testing our hypothesis, we must understand the tools we are using. We must understand the performance of the encoder model we use to define the projection.

We will also be interested in substituting the encoder model for a simple rate distortion method as well as an IB method approach to defining the projection. We do not expect these results to work well, but it is a baseline to compare against.

### STSb evaluation

A first, standard characterization of the encoder is its agreement with human judgments of semantic similarity on the STS Benchmark (STSb). This is the field-standard benchmark for sentence-embedding models, not a bespoke measure, so it lets us confirm our encoder behaves as documented before we rely on it. Implemented in `encode.ipynb`.

The procedure:

1. Take the STSb test split (~1,379 sentence pairs, each with a human similarity score in $[0,1]$).
2. Embed `sentence1` and `sentence2` separately with the encoder.
3. Compute the cosine similarity of each pair's two embeddings.
4. Report the **Spearman rank correlation** between the model's cosine similarities and the human scores.

Spearman (rank) correlation is the convention here: it measures whether the model orders pairs from least to most similar the same way humans do, without requiring the raw cosine values to match the human scale. For `all-MiniLM-L6-v2` the published result is roughly $0.82$; reproducing that number is a sanity check that the embedding/pooling/similarity pipeline is correct, and it establishes a baseline quality for the projection we will build on.

Scope and limits of this measure:

* It is a single scalar over one fixed dataset at one fixed resolution.
* "Semantic similarity" here is operationalized as averaged human annotation, not as any causal or information-theoretic definition of meaning.
* It reports agreement in *ranking* only, and says nothing about the multiplicity / equivalence-class structure that is the object of the validation plan below.

## Getting the Proper Datasets: Datasets will mimic the microcanonical and canonical ensemble microstates

We hope to find a dataset which fulfills the assumptions of the microcanonical and canonical ensemble. This may be difficult to find and thus we may need to loosen this requirement. The purpose of this section is to state, as precisely as we currently can, the assumptions and constraints a dataset must satisfy to be usable — and, just as importantly, to **name every approximation** we are making rather than hand-waving past it.

### Stance: equilibrium, not non-equilibrium steady state (for now)

We commit to **equilibrium** statistical mechanics as the target. Systems that are only in *local* equilibrium remain acceptable data sources, but we set aside the non-equilibrium steady state (NESS) picture for the moment (not ruling it out permanently). This is a deliberate choice to pursue the harder, more falsifiable claim: a NESS is merely *stationary*, whereas equilibrium additionally requires **detailed balance** (no net probability currents between states). Stationarity alone does not distinguish the two.

### What "a real community" means here

We want a dataset that was, at some point, a **genuine real-world community** — a discussion forum, a bounded online community with internal dialogue — as opposed to a pile of unrelated texts. It does **not** need to be a *living*, continuously-updating feed. A static archived snapshot (e.g. a discussion forum from ~2010) is perfectly acceptable, and is in fact preferred, provided it retains the internal temporal and reply structure (below).

### Microstate and the two-ensemble construction

Following the same philosophy as standard statistical mechanics, a **microstate is the description of the entire state of the entire system at the microscopic scale** — as in the Ising model, where a microstate is the configuration of *all* spins, not one. Here the microstate is the **whole system's natural-language state within a small time-slice** (for computational practicality, e.g. everything posted within the same hour). The exact mathematical structure of a microstate is nontrivial and is developed separately in the theory; we leave it unspecified here.

From a single dataset we can build **both** ensembles:

* **Microcanonical:** the *whole forum* is the system.
* **Canonical:** an *embedded subsystem* is the system, and the rest of the community is the heat reservoir. This is the textbook construction — a small subsystem of a microcanonical whole, with the reservoir integrated out. **The subsystem unit is not fixed here:** an individual is only the simplest *example*; the natural unit (a person, a subgroup, a sub-forum) is determined by the dataset, not decided in advance (see [Coupling, correlations, and the additivity requirement](#coupling-correlations-and-the-additivity-requirement)).

### Primary target: the canonical ensemble first

We will design the dataset around the **canonical** (subsystem-and-bath) experiment first, for a specific reason. At the whole-system scale the microcanonical test is **tautology-prone**: a whole-forum microstate is so high-dimensional it never recurs (every observed microstate has count $1$), so the estimated multiplicity of a macrostate equals its observed frequency, and the prediction $P(\text{macrostate})\propto\Omega$ becomes an identity that tests nothing. Making it non-trivial would require either computing $\Omega$ **structurally/combinatorially** (which needs energy/coarse-graining machinery not yet built) or coarse-graining microstates (an $\varepsilon$-ball grouping) so that cells recur. The canonical case avoids this: the system (a small subsystem — an individual is the simplest example, but the unit is dataset-dependent) is small, its language states recur and can be binned, and the distinction between message *types* and *tokens* gives an independent handle on multiplicity. Microcanonical is therefore deferred. (This is a preference for the canonical *construction* on technical grounds — it says nothing about *which* subsystem: that is fixed only once we have a dataset in hand.)

### Dataset constraints (v1)

Each constraint is tagged with the approximation it encodes.

1. **Was a genuine, bounded real-world community.** *(approx: asserts a community boundary exists; the boundary is fuzzy.)*
2. **Retains timestamps and reply/thread structure.** *(Hard requirement — needed for detailed-balance testing and any temporal ordering; not an approximation.)*
3. **Sampled from a stationary window, not the whole lifetime; stationarity is checked, not assumed.** *(approx: "stationary enough." This guards the pooling trap — pooling a community's whole lifetime mixes non-equilibrium distributions / a relaxation trajectory into one blurred pseudo-ensemble.)*
4. **Insular / endogenously driven** — internal dynamics dominate over exogenous news and events. *(approx: treats weak external coupling as a fixed thermal bath.)*
5. **Mature, past its growth phase, with no active shock in the window.** *(approx: asserts relaxation had occurred.)*
6. **Enough volume to populate macrostate cells with real counts.** *(approx: the sparse-sampling floor; sets a minimum size.)*
7. **[Canonical] Persistent, identifiable subsystem units with high volume across the window** — individuals are the simplest example, but the unit could be a definable subgroup. *(Required to build a subsystem-level distribution at all; the natural unit is dataset-dependent, not fixed to a single person.)*
8. **[Canonical] Bath $\gg$ system:** the community dwarfs the chosen subsystem, which is a non-dominant part of it.
9. **Wide temporal span with fine-grained timestamps** — to test stationarity and isolate a relaxed window. *(Upgrades constraint 3.)*
10. **Raw volume sufficient to trace a differentiable $S(U)$ curve, not merely to populate cells.** *(approx: strengthens constraint 6. Because temperature will be defined from the first-principles derivative $\beta = \partial S/\partial U$ — not as a fitted noise parameter — we need several well-filled energy levels to finite-difference across, so total volume, not just one-count-per-cell, is the binding requirement.)*
11. **Dynamic range: the community visits a spread of macrostates rather than parking in one dominant state.** *(approx: needed so there is a $\Delta U$ to difference across. Expected to be satisfied automatically — real communities carry a wide variety of states, and the opposite (a single occupied state) is the unrealistic, hard-to-source case. Doubles as a bonus: a spread of states lets us test the same theory across several regimes within one dataset.)*

### The ensemble source is left open

Whether we treat **one community as a single trajectory** (many time-slices as the samples) or **many comparable communities as parallel draws** is intentionally left open. **No choice is made here, and none is needed to begin the dataset search:** when we look for candidates we will deliberately **cast a wide net for both**, and let the data we can actually obtain — not an up-front commitment — narrow it. Both readings map onto a standard move in statistical mechanics, and each carries its own debt.

**One community as a trajectory — the ergodic substitution.** The forum is one system; its successive time-slices are the states it visits along a trajectory, and using them as ensemble samples invokes the ergodic hypothesis (time-average $=$ ensemble-average). Note this is *thermodynamic* ergodicity — exploration of the state space — not the information-theoretic "ergodic source" assumption; conflating the two is exactly the failure mode flagged earlier. Debts:

* **Ergodic mixing:** does the community actually mix, or stall in a metastable pocket (a subculture that never couples to the rest)? Broken mixing breaks time-avg $=$ ensemble-avg.
* **Within-archive stationarity:** the window must not drift; checked by comparing sub-windows.
* **Autocorrelation tax:** successive slices are not independent, so the effective sample size is $\sim N/\tau$ (with $\tau$ the autocorrelation time), not $N$ — a real, measurable cost that interacts with the raw-volume constraint (10).

**Many communities as parallel draws — homogeneity.** Each community is an independent replica of "the same system" at the same macro-conditions. Debts:

* **Comparability:** differing size / topic / platform / era mean differing macro-conditions, so the ensemble risks being *inhomogeneous* (mixing systems at different "temperatures"); checked via cross-community comparability.
* **Chicken-and-egg with energy:** matching communities on their macro-conditions ideally uses the energy/temperature variables not yet pinned down — so building this ensemble *rigorously* is partly gated by the energy program.
* **Independence (the upside):** parallel communities are more plausibly independent than time-slices — *if* they do not share members or exogenous shocks — which sidesteps the autocorrelation tax.

**How the two readings differ on the observables we care about:**

* **Temporal / dynamical diagnostics** (net-current / detailed-balance-style and fluctuation–dissipation-style checks) are statements about transitions *over time within one system*; there is no time-ordered transition between two independent communities, so these live naturally in the **trajectory** reading.
* **The energy axis for a differentiable $S(U)$** (constraints 10–11) is sourced differently: in the **trajectory** reading the spread in $U$ comes from *equilibrium fluctuations within the window* (narrow for a large subsystem); in the **many-community** reading it comes from *different communities sitting at different energies* (a wider, more controllable axis, but one that presupposes a common, defined energy scale).

**Two practical notes.**

* The readings are **not exclusive**: a natural staging is one community as a trajectory, then a matched family of communities layered on later specifically to widen the energy axis.
* A single forum already contains a **mini-ensemble**: its many *members* (or other definable sub-units) are parallel canonical subsystems, giving some of the "many independent draws" benefit without sourcing many forums — a third flavor between the two poles.

### Approximations carried into every dataset

* Natural language is treated as an **ergodic stationary source** (the standard machine-learning assumption; taken as approximately true).
* We are careful **not to conflate two distinct notions of "ergodic":** the information-theoretic ergodic source (AEP / typical sets) is *not* the same as thermodynamic ergodicity (exploration of the energy shell). Conflating them is precisely the "analogy vs. first principles" failure mode criticized in the report.
* **The encoder is a proxy for a causal/predictive notion of meaning — not a different thing.** Meaning here is environment-relative (the causal role a message plays within the community's distribution), and the only "action" is more talk (the forward flow of discourse, which is in the data). The encoder's distributional similarity is a proxy for the resulting predictive equivalence; its faithfulness rests on adaptation to the community and on predictive sufficiency. See "Meaning as a causal/predictive equivalence" below.
* **The canonical construction needs additivity of energy across the system/bath split.** The Boltzmann derivation relies on $E_\mathrm{tot} = E_S + E_B$, i.e. a negligible interaction energy $H_\mathrm{int}$ between the chosen subsystem and the rest of the community. Whether our partition satisfies this is not yet known — it depends on the dataset and on how we split it. See "Coupling, correlations, and the additivity requirement" below.
* **Equilibrium may not hold at any instant.** The system may instead be *relaxing toward* equilibrium, which would call for a theory of kinetics rather than equilibrium mechanics. This is noted and deferred.

### Coupling, correlations, and the additivity requirement

The subsystem/bath split need not be a single person — it can be any subset much smaller than the rest of the community; the right unit and scale depend on the dataset. What the canonical derivation actually requires is that energy be **additive** across the split, $E_\mathrm{tot} = E_S + E_B$, which holds when the interaction energy $H_\mathrm{int}$ across the boundary is negligible relative to the internal energies. It is the additivity that must be maintained, not the absence of interactions.

Interactions *within* the system are not a problem and are handled routinely in standard statistical mechanics:

* **Ising model:** the spin–spin coupling is not assumed negligible — it lives inside $H_S$ and is the entire model. The canonical ensemble treats the whole lattice as the system; only the lattice–bath interface is assumed weak, justified by surface/volume ($H_\mathrm{int}/H_S \sim N^{-1/3} \to 0$ for a large system).
* **Van der Waals / virial expansion:** interparticle interactions are absorbed into the equation of state — starting from the non-interacting ideal gas and correcting order by order in density (e.g. $B_2(T) = -\tfrac{1}{2}\int (e^{-\beta u(r)} - 1)\, d^3r$).

The genuine caveat is boundary coupling for a *small* subsystem (all surface, no bulk — like taking a single spin as the system): there the bare-energy Boltzmann form need not hold. But even then, given equilibrium, the subsystem's marginal is Boltzmann in an **effective energy** (the Hamiltonian of mean force; in mean-field Ising, an effective field $h_\mathrm{eff} = h + Jz\langle s\rangle$ that absorbs the neighbors' influence). So strong coupling changes *which* energy appears, not *whether* the distribution is exponential. Consistent with the start-simple methodology above, we begin by assuming additive energy (no interaction term) and add an interaction / effective-energy correction only if the bare version fails.

### The encoder is an instrument, not part of the physics

The physics of interest concerns causal semantic macrostates that exist **independently of any LLM**. Encoder models are only the best tool we currently have for *measuring* those macrostates in real-world text — a deliberately imperfect instrument, not part of the ontology. They are a reasonable instrument because their training is predictive (masked-token prediction for encoder-only models, plus contrastive similarity training for sentence encoders), and **good prediction is equivalent to good compression**: minimizing predictive log-loss is minimizing code length (source coding / MDL). A model that compresses the source well must have captured its statistical structure, which is exactly what we want to measure. This instrument-independence is consistent with the Platonic Representation Hypothesis cited in the report — the relevant structure is a property of the data, not of any particular model.

### Meaning as a causal/predictive equivalence

Two premises of the theory sharpen what the encoder is for. (i) Semantic meaning is **not** inherent to a sentence in a vacuum; it is an emergent property of a message's **causal role within a specific environment**, and that environment is the distribution modeling the community. (ii) The only "action" a receiver takes is to **emit further messages** — the causal chain is A's message → induces B's message → induces C's, entirely within the medium of communication, so the behavioral readout is just the forward discourse, already present in the dataset.

Under these premises the target notion is **predictive equivalence**: two messages are equivalent iff they induce the same distribution over the community's future messages. This is the **causal states** notion of computational mechanics (classes of pasts with the same conditional distribution over futures), and equivalently the **information bottleneck with the future as the relevance variable** ($\min I(X;T) - \beta I(T;Y)$ with $Y$ = subsequent messages — already our framework). So the encoder is not the wrong *kind* of tool; its distributional similarity is a **proxy** for this predictive map.

Whether the proxy is faithful rests on two conditions — named approximations, not guarantees:

1. **Adaptation to the community.** A general-purpose encoder fixes "same meaning" from its own training corpus — a different environment than the target community. To honor premise (i), the projection should eventually be adapted to the community's distribution rather than generic.
2. **Predictive sufficiency (the load-bearing hypothesis).** For the coarse-grained macrostate process to be *fully* described by the encoder's classes, those classes must be predictively sufficient — the macrostate future must depend only on the current macrostate, not on the fine syntax integrated out (a lumpability / Markov-closure condition). Causal states are constructed to have this property; an arbitrary encoder's classes need not. This is testable, and is really *the* central empirical claim rather than an assumption to wave through.

Consistent with the start-simple methodology, we begin with a general-purpose encoder that at least broadly agrees on semantic similarity and build toward community-adapted / predictive maps only as needed. Two related issues are noted and deferred: encoder sensitivity to *style vs. meaning* (an encoder-choice question), and *receiver-relative* meaning (different members projecting differently — eventually a per-subsystem projection / symmetry group).

#### Equivalence is defined at a macroscopic scale — we observe, we do not model microdynamics

An essential qualification: **semantic equivalence always means equivalence at a chosen macroscopic scale**, exactly as in the passage from statistical mechanics to thermodynamics. We adopt predictive equivalence as the *definition* of "same meaning," but we do **not** intend to reconstruct the microscopic causal chain of messages. Coarse-graining that microscopic detail away is the entire point of doing statistical mechanics — the same reason thermodynamics exists. The methodology is therefore **observational and count-based**: we record which macrostates occur (and, later, which co-occur), without attempting to explain *why* those correlations arise.

This clarifies how we borrow from the two nearest frameworks — both worth keeping in view, and their key difference is loss:

* **Computational mechanics — *lossless*.** Its causal states are the minimal *sufficient statistic* of the past for the future: they retain **all** predictive information. This grounds the *definition* of meaning (predictive-equivalence classes) and is the fine / exact limit. We take the concept, not the program — we do not intend to reconstruct the process's dynamical model (the $\varepsilon$-machine).
* **Information bottleneck — *lossy*.** It trades compression $I(X;T)$ against relevance $I(T;Y)$ via $\beta$, deliberately discarding predictive information to compress further. Its lossiness is a **feature** here: finite $\beta$ *is* a macroscopic scale, and sweeping $\beta$ sweeps the coarse-graining scale (the $\beta \to \infty$ limit recovers the lossless, sufficient-statistic case). This is the operational tool for semantic equivalence at a chosen scale.

#### First-attempt procedure (high level)

**This is the round-one test** — the first, bare-bones experiment, which produces the [Primary observable](#primary-observable) (the Boltzmann distribution). It is meant to mimic the logic of standard statistical mechanics → thermodynamics as literally as possible:

1. Obtain a dataset that fulfills the canonical-ensemble assumptions; **specify the system and the bath**.
2. By observation alone, **enumerate the microstates**.
   1. Use the encoder to project microstates onto macrostates.
   2. Define a metric / similarity kernel on microstates via the macrostate they fall into — using the number of **bits** (from the semantic-similarity metric) by which macrostates differ.
   3. Use that to define the **Hamiltonian**, and hence the Boltzmann factors.

We are simply observing which states appear in concert with which others; we are not explaining why those correlations exist.

#### Points requiring further care

* **A metric is not yet a Hamiltonian.** A pairwise "bits of difference" kernel gives distances *between* states, not an absolute energy *per* state. To get Boltzmann factors $e^{-\beta E_i}$ we must fix an **origin / reference** (e.g. the equilibrium / consensus macrostate) so that $E_i$ = bits of semantic difference from that reference. The choice of reference is a modeling decision.
* **The similarity → bits conversion is not automatic.** Cosine similarity is not natively an information measure; turning it into "bits" needs an explicit construction (a $-\log$ / description-length reading). This is the same bridge flagged for the energy problem.
* **Adaptation vs. the circularity guard pull against each other.** Defining energy from a *pretrained* encoder's geometry keeps it independent of this community's frequencies (non-circular). But adapting the encoder to the community (recommended for faithfulness) makes its geometry partly a function of those frequencies — reintroducing circularity risk. These two desiderata are in tension.
* **Marginal occupation vs. co-occurrence are different experiments.** The bare-bones Boltzmann test is the *marginal* occupation distribution of one subsystem's macrostates against energy (non-interacting). "Which states pop up in parallel with which" is *joint / co-occurrence* structure — i.e. the interaction term, to be added only if the non-interacting version fails.
* **Counting suffices for the equilibrium distribution; dynamics is a separate, harder ask.** The observational, count-based approach is exactly right for the equilibrium Boltzmann / multiplicity measurements. The dynamical observables (detailed balance, FDT) require transitions and the Markov-closure scale criterion, and are more demanding — consistent with treating them as a later stage.

## The Validation Plan: What observables will we look for and how do we determine success?

Our primary form of validating our results will be attempting to reproduce the expected statistics which various statistical mechanic ensembles predict. For example, the Boltzmann distribution.

I can also measure the multiplicity of macrostates which is the cardinality of the semantic equivalance classes.

The encoder will project the dataset into a latent space. We will then partition the latent space according to a scale.

### Partitioning the latent space: defining macrostates and choosing a scale

The encoder maps each microstate to a vector in $\mathbb{R}^d$ ($d = 384$ for MiniLM), giving a *cloud* of points, not macrostates. A macrostate is a **cell** of this space, so we need a map $\mathbb{R}^d \to \{1, \dots, K\}$. Everything downstream is defined only once this map exists: the macrostate distribution $P(\text{macro})$ (normalized cell counts), the multiplicity $\Omega$ (distinct microstates per cell), and the transition matrix $W(\text{macro} \to \text{macro})$ (the input to detailed balance and FDT). No decision has been made yet; this records the options.

**Scale is intrinsic, not a nuisance.** There is likely no single "correct" scale. Too fine → cells of size one (the microcanonical tautology / sparse-sampling wall); too coarse → everything collapses to one blob. The interesting structure lives in between, and because the framework is RG / coarse-graining, scale is the *axis we flow along*, not a knob to eliminate. The deliverable is therefore observables **as a function of scale**, with trustworthy structure signalled by a stable range (a plateau) rather than a magic value.

**Candidate partition methods** (scale knob in parentheses):

* **$\varepsilon$-ball / grid quantization** ($\varepsilon$ or cell size) — simplest, but $d = 384$ makes uniform grids infeasible (curse of dimensionality).
* **$k$-means / spherical $k$-means** ($K$) — easy, gives counts directly; must pick $K$, assumes isotropic blobs, re-cluster per scale.
* **Hierarchical / agglomerative** (dendrogram cut height) — the whole scale family is one object; natural fit for the RG framing.
* **Density-based (HDBSCAN)** (min cluster size) — natural modes, arbitrary shapes, no forced $K$; labels sparse points as "noise."
* **Information Bottleneck / rate–distortion** ($\beta$ or rate $R$) — clusters to preserve predictive information about the future $Y$; the theory-faithful method (matches meaning = predictive equivalence), but heavier. The geometric methods above are proxies for it.

**Pitfalls.** Use the **cosine / angular metric** (these encoders are trained for it), not raw Euclidean. Avoid density-distorting dimensionality reduction (UMAP / t-SNE) if cell counts will be read as probabilities; PCA is safer. Start with **hard** assignment (one microstate → one macrostate); soft assignment complicates $\Omega$.

**Choosing / validating a scale.** Three criteria turn "what scale?" into measurable questions:

1. **Sampling floor** — cells must hold enough microstates to estimate $P$ and $W$ reliably; sets the finest *data-admissible* scale (ties to dataset constraint 6).
2. **Stability** — the partition should be robust to small data perturbations and to encoder choice.
3. **Predictive sufficiency / Markov-closure** (the principled criterion) — the good scale is one where the macrostate process is approximately Markov: the next macrostate depends only on the current one, not on the fine syntax integrated out. Test by comparing $P(\text{next} \mid \text{current})$ with $P(\text{next} \mid \text{current} + \text{more past})$, and pick the **coarsest partition that still closes**. This is the discrete, empirical version of finding the causal states, and reuses the predictive-sufficiency hypothesis already flagged.

**Bare-bones first pass.** Cosine metric, native space, hierarchical clustering or $k$-means swept over scale, every observable reported vs. scale; sampling floor as the lower bound and the Markov test as the scale selector. Upgrade geometric clustering to IB / predictive clustering only if the simple version demonstrably falls short.

### Encoder loss as an instrument-side resolution floor

*Open idea, and the conclusion we reached on it.*

**The idea, as raised.** Can an encoder's loss (a cross-entropy) define an intrinsic amount of coarse-graining it is capable of — i.e., does a higher average loss mean the encoder can only resolve more macroscopic detail? Framed deliberately as a property of the *instrument*, not the dataset: it tells us whether a given scale is even *testable* with the encoder we are using, and is explicitly **not** meant to drive theory or to connect to the Landauer energy–information relation (which is about the system, not the tool).

**What holds up.** Cross-entropy is a genuine information quantity — by the prediction ↔ compression equivalence it is bits per token, the code length the model achieves — so lower loss means finer resolvable distinctions. The framing as an *instrument resolution* question is the correct, modest one.

**Three corrections.**

1. For a *sentence* encoder (e.g. `all-MiniLM`) the training loss is **contrastive (InfoNCE-type), not token cross-entropy**. Usefully, InfoNCE is a **lower bound on mutual information** ($I \ge \log N - L_\mathrm{contrastive}$), so it directly bounds how much paired-item structure the encoder captures. Token cross-entropy is the base encoder's pretraining objective.
2. **Raw loss is not an absolute scale.** It depends on dataset, tokenizer, vocabulary, and log base (comparable only on a fixed evaluation), and it decomposes as $H(p,q) = H(p) + D_\mathrm{KL}(p\,\|\,q)$ — irreducible source entropy plus the model's excess. Only the **KL gap to the optimal predictor** is attributable to the encoder, so the meaningful quantity is the gap, not the raw number.
3. **The bridge from bits-per-token to a latent-partition scale is not automatic.** A more direct handle on instrument resolution is the **embedding noise floor**: how reliably the encoder separates known-same from known-different pairs (exactly what the STSb calibration already measures). Below that separation the encoder is not actually resolving distinctions.

**Conclusion.** Keep the idea, reframed as an **instrument-side resolution floor**: do not trust a partition finer than the encoder can resolve (as measured by its loss gap / MI bound, or more directly its empirical noise floor). This complements the **data-side sampling floor** above — the fine end of scale is now bounded from two independent sides, and the trustworthy scale range is where *both* are satisfied. It stays strictly an experimental-feasibility tool and is kept out of the physics / Landauer story.

### Primary observable

The primary observable, in line with the canonical-first decision above, is to **reproduce the Boltzmann distribution for the chosen subsystem's microstates**: enumerate the subsystem's language microstates across timestamps and test whether its occupation statistics follow $P_i \propto e^{-\beta E_i}$ for the reservoir temperature. The subsystem/bath split is **not fixed in advance** — an individual is only the simplest example; the natural unit (a person, a subgroup, a sub-forum) is determined by the dataset (see [Coupling, correlations, and the additivity requirement](#coupling-correlations-and-the-additivity-requirement)). The concrete round-one recipe that produces this observable is the [First-attempt procedure (high level)](#first-attempt-procedure-high-level) above.

### Exploratory diagnostics: equilibrium-likeness checks from counts and timestamps

*These are exploratory diagnostics — **not** the formal round-one test, and not a substitute for the energy-based program.* A useful batch of checks requires only **counts and timestamps** (no energy function), so they can be run early as exploratory data analysis (EDA) to see whether a candidate system even *looks* equilibrium-like. They inform dataset selection and sanity-checking; they do **not** stand in for the formal, energy-based validation (the [Primary observable](#primary-observable)), which is developed separately — energy is under active development, not set aside.

* **Stationarity** — does the macrostate distribution drift across sub-windows?
* **Detailed balance** — are there net probability currents / cycles in the macrostate-to-macrostate transitions? Nonzero currents point to a NESS rather than equilibrium.
* **Fluctuation–dissipation** — does the response to a natural shock match spontaneous fluctuations? A violation likewise points to a NESS.

Being *dynamical* (detailed balance and FDT read transitions over time), these sit naturally in the one-community-as-trajectory reading of the ensemble-source fork above. Treat them as reconnaissance: they can flag a system as clearly-not-equilibrium early, but passing them is necessary, not sufficient, and is not itself the validation target.

### Density-of-states estimation: prior art for the sparsity problem

Because temperature will be defined from first-principles derivatives ($\beta = \partial S/\partial U$, and similar) rather than as a fitted noise parameter, the energy stage needs a **well-sampled $S(U)$ curve we can finite-difference** — and the tails of that curve go sparse first (rare energies are visited rarely). This is a long-standing problem in computational physics; the standard machinery is worth keeping on file, with one caveat that decides how much of it actually transfers to our setting.

* **Wang–Landau sampling.** A Monte Carlo method that estimates the density of states $g(E)$ *directly* by doing a random walk in energy with acceptance $\propto 1/g(E)$, which flattens the energy histogram and forces the walk to visit rare energies as often as common ones; $g(E)$ is refined on the fly by a shrinking modification factor. Payoff: a single estimate of $g(E)$ (hence $S(E) = k_B \ln g(E)$) across the *full* energy range, from which any thermodynamic quantity — and its derivatives — follows at any temperature.
* **Multicanonical sampling (MUCA).** The close predecessor: sample from a reweighted "multicanonical" ensemble engineered to give a flat energy histogram, so the simulation crosses energy barriers and visits the whole range; canonical averages at any $T$ are then recovered by reweighting. Same goal as Wang–Landau (beat the sparse tails / rarely-visited states), reached with a fixed bias rather than an adaptive one.
* **WHAM (Weighted Histogram Analysis Method).** A *post-processing* estimator, not a sampler: it optimally (minimum-variance) stitches together histograms collected under different conditions (different temperatures, or biased windows) into a single best estimate of $g(E)$ / the free-energy profile, correcting for each run's sampling weight. This is the one closest in spirit to what we might actually do — merge counts from multiple time-windows or communities sitting at different energies into one $S(U)$ estimate.

**The caveat that decides which of these transfers.** Wang–Landau and MUCA are *active-sampling* algorithms: they assume you can propose moves in energy space and evaluate an energy function to bias the walk. Our data is **observational** — we take the samples the community actually produced and cannot reweight its real dynamics — and we have no energy function yet, so neither transfers directly. What *does* transfer is the **analysis idea**: WHAM-style optimal combination of overlapping histograms is a way to build a low-variance $S(U)$ from counts we already have, and the flat-histogram philosophy is the conceptual reference for how a smooth density of states is recovered despite sparse tails. Reserved for the energy stage; listed here so we don't reinvent it.

### Open problems (deferred, developed separately in the theory)

The following are known gaps, flagged here so the validation plan does not silently assume them away. They are not solved in this document.

* **The energy function / energy levels** of natural-language microstates.
* **A circularity guard for energy:** the energy must be defined *independently* of this community's measured frequencies. Otherwise "reproducing Boltzmann" is vacuous, since *any* distribution can be written as $P_i \propto e^{-\beta E_i}$ by setting $E_i = -\beta^{-1}\ln P_i$.
* **The temperature parameter** for the reservoir in the canonical case.
* **A conserved quantity** to anchor the ensembles (the microcanonical energy shell / the canonical exchange). This may be logically prior to asking whether the system is at equilibrium at all.