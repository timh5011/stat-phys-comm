# Experimentation Plan

This document outlines the high level objectives and low level details of the computational experiments we will run. Our primary goal is to look for equilibrium statistical mechanic macrostate structure in natural language by using encoder-only transformer LLMs to define equivalence classes by semantic meaning.

## Table of Contents: Pillars of the experiment

* [Characterizing the encoder model](#characterizing-the-encoder-model)
* [Getting the proper datasets; datasets will mimic the microcanonical and canonical ensemble](#getting-the-proper-datasets-datasets-will-mimic-the-microcanonical-and-canonical-ensemble)
* [The validation plan; what observables will we look for and how do we determine success?](#the-validation-plan-what-observables-will-we-look-for-and-how-do-we-determine-success)

## Characterizing the encoder model

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

## Getting the proper datasets; datasets will mimic the microcanonical and canonical ensemble

We hope to find a dataset which fulfills the assumptions of the microcanonical and canonical ensemble. This may be difficult to find and thus we may need to loosen this requirement. The purpose of this section is to state, as precisely as we currently can, the assumptions and constraints a dataset must satisfy to be usable — and, just as importantly, to **name every approximation** we are making rather than hand-waving past it.

### Stance: equilibrium, not non-equilibrium steady state (for now)

We commit to **equilibrium** statistical mechanics as the target. Systems that are only in *local* equilibrium remain acceptable data sources, but we set aside the non-equilibrium steady state (NESS) picture for the moment (not ruling it out permanently). This is a deliberate choice to pursue the harder, more falsifiable claim: a NESS is merely *stationary*, whereas equilibrium additionally requires **detailed balance** (no net probability currents between states). Stationarity alone does not distinguish the two.

### What "a real community" means here

We want a dataset that was, at some point, a **genuine real-world community** — a discussion forum, a bounded online community with internal dialogue — as opposed to a pile of unrelated texts. It does **not** need to be a *living*, continuously-updating feed. A static archived snapshot (e.g. a discussion forum from ~2010) is perfectly acceptable, and is in fact preferred, provided it retains the internal temporal and reply structure (below).

### Microstate and the two-ensemble construction

Following the same philosophy as standard statistical mechanics, a **microstate is the description of the entire state of the entire system at the microscopic scale** — as in the Ising model, where a microstate is the configuration of *all* spins, not one. Here the microstate is the **whole system's natural-language state within a small time-slice** (for computational practicality, e.g. everything posted within the same hour). The exact mathematical structure of a microstate is nontrivial and is developed separately in the theory; we leave it unspecified here.

From a single dataset we can build **both** ensembles:

* **Microcanonical:** the *whole forum* is the system.
* **Canonical:** *one embedded person* is the system, and the rest of the community is the heat reservoir. This is the textbook construction — a small subsystem of a microcanonical whole, with the reservoir integrated out.

### Primary target: the canonical ensemble first

We will design the dataset around the **canonical** (person-as-system) experiment first, for a specific reason. At the whole-system scale the microcanonical test is **tautology-prone**: a whole-forum microstate is so high-dimensional it never recurs (every observed microstate has count $1$), so the estimated multiplicity of a macrostate equals its observed frequency, and the prediction $P(\text{macrostate})\propto\Omega$ becomes an identity that tests nothing. Making it non-trivial would require either computing $\Omega$ **structurally/combinatorially** (which needs energy/coarse-graining machinery not yet built) or coarse-graining microstates (an $\varepsilon$-ball grouping) so that cells recur. The canonical case avoids this: the system (one person) is small, its language states recur and can be binned, and the distinction between message *types* and *tokens* gives an independent handle on multiplicity. Microcanonical is therefore deferred.

### Dataset constraints (v1)

Each constraint is tagged with the approximation it encodes.

1. **Was a genuine, bounded real-world community.** *(approx: asserts a community boundary exists; the boundary is fuzzy.)*
2. **Retains timestamps and reply/thread structure.** *(Hard requirement — needed for detailed-balance testing and any temporal ordering; not an approximation.)*
3. **Sampled from a stationary window, not the whole lifetime; stationarity is checked, not assumed.** *(approx: "stationary enough." This guards the pooling trap — pooling a community's whole lifetime mixes non-equilibrium distributions / a relaxation trajectory into one blurred pseudo-ensemble.)*
4. **Insular / endogenously driven** — internal dynamics dominate over exogenous news and events. *(approx: treats weak external coupling as a fixed thermal bath.)*
5. **Mature, past its growth phase, with no active shock in the window.** *(approx: asserts relaxation had occurred.)*
6. **Enough volume to populate macrostate cells with real counts.** *(approx: the sparse-sampling floor; sets a minimum size.)*
7. **[Canonical] Persistent, identifiable individuals with high post volume across the window.** *(Required to build a person-level distribution at all.)*
8. **[Canonical] Bath $\gg$ system:** the community dwarfs any single person, and the chosen person is a non-dominant voice.
9. **Wide temporal span with fine-grained timestamps** — to test stationarity and isolate a relaxed window. *(Upgrades constraint 3.)*

### The ensemble source is left open

Whether we treat **one community as a single trajectory** (many time-slices as the samples) or **many comparable communities as parallel draws** is intentionally left open; both remain in scope. Each carries its own debt: the one-community reading relies on the **ergodic substitution** (time-average $=$ ensemble-average), while the many-community reading relies on **homogeneity** (that the communities are the "same system" at the same macro-conditions). These demand *different* stationarity checks — within-archive drift versus cross-community comparability.

### Approximations carried into every dataset

* Natural language is treated as an **ergodic stationary source** (the standard machine-learning assumption; taken as approximately true).
* We are careful **not to conflate two distinct notions of "ergodic":** the information-theoretic ergodic source (AEP / typical sets) is *not* the same as thermodynamic ergodicity (exploration of the energy shell). Conflating them is precisely the "analogy vs. first principles" failure mode criticized in the report.
* **The instrument does not measure the theory's definition of meaning.** The encoder measures *distributional* semantics; the theory defines meaning *causally* (invariance of a receiver's future behavior under exchange of messages). These are different equivalence relations — a named approximation the entire projection rests on.
* **The canonical weak-coupling assumption is violated.** The chosen person is embedded in, reacts to, and helps shape the "bath," so system and bath are strongly coupled and correlated — a more severe violation than the usual idealization, and an open risk to any Boltzmann result.
* **Equilibrium may not hold at any instant.** The system may instead be *relaxing toward* equilibrium, which would call for a theory of kinetics rather than equilibrium mechanics. This is noted and deferred.

## The validation plan; what observables will we look for and how do we determine success?

Our primary form of validating our results will be attempting to reproduce the expected statistics which various statistical mechanic ensembles predict. For example, the Boltzmann distribution.

I can also measure the multiplicity of macrostates which is the cardinality of the semantic equivalance classes.

The encoder will project the dataset into a latent space. We will then partition the latent space according to a scale.

### Primary observable

The primary observable, in line with the canonical-first decision above, is to **reproduce the Boltzmann distribution for a single person's microstates**: enumerate one person's language microstates across timestamps, and test whether their occupation statistics follow $P_i \propto e^{-\beta E_i}$ for the reservoir temperature.

### Staging: what can be tested before the energy problem is solved

The energy function and temperature are hard, unsolved pieces (below). It is worth noting that **not every test needs them.** A useful staging falls out:

* **Energy-free tests (run first):** stationarity (does the macrostate distribution drift across sub-windows?), **detailed balance** (are there net probability currents / cycles in the macrostate-to-macrostate transitions?), and **fluctuation–dissipation** (does the response to a natural shock match spontaneous fluctuations?). These need only **counts and timestamps.** Nonzero currents or an FDT violation would indicate a NESS rather than equilibrium.
* **Energy-dependent test (run later):** the Boltzmann distribution itself, which requires both an energy function and a temperature.

This lets us establish whether the system is even *equilibrium-like* before committing to the energy machinery.

### Open problems (deferred, developed separately in the theory)

The following are known gaps, flagged here so the validation plan does not silently assume them away. They are not solved in this document.

* **The energy function / energy levels** of natural-language microstates.
* **A circularity guard for energy:** the energy must be defined *independently* of this community's measured frequencies. Otherwise "reproducing Boltzmann" is vacuous, since *any* distribution can be written as $P_i \propto e^{-\beta E_i}$ by setting $E_i = -\beta^{-1}\ln P_i$.
* **The temperature parameter** for the reservoir in the canonical case.
* **A conserved quantity** to anchor the ensembles (the microcanonical energy shell / the canonical exchange). This may be logically prior to asking whether the system is at equilibrium at all.