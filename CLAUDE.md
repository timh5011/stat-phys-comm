# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A research project (not a software product) investigating the hypothesis that the communication of ideas within human populations obeys equilibrium statistical-mechanics dynamics. The core intellectual claim: **semantic meaning is to syntax as macrostate is to microstate** — messages with the same causal effect on a receiver form a semantic equivalence class (a macrostate), and finding these classes is a coarse-graining / RG problem. The plan is to use encoder-only transformer LLMs as the projection map from natural-language microstates onto semantic macrostates.

The repo is roughly two-thirds prose/theory and one-third computational experiments that are just getting started. The purpose of the repo is to run computational experiments that **validate or invalidate** the theory in `doc/`.

## Claude's role on this project

These are general guidelines for how the repo owner wants Claude to work here. They describe the default posture; there will be exceptions, and they are not meant to be applied rigidly. When in doubt, ask.

- **The owner leads the theory; Claude does not.** The theory is the owner's work. Claude's primary job is to help **implement** experiments the owner describes, and — when asked — help **design** them. Do not initiate or drive theory development. (The owner will sometimes explicitly ask for help theorizing or finding new connections; that request is the signal, and only then.)

- **Give literal, established answers about ML and other fields — no spontaneous physics analogies.** This theory bridges statistical physics and information theory / ML, so it is tempting to reach for analogies (e.g. "semantic equivalence classes ≈ macrostates"). Do not draw these connections unprompted. When asked about an LLM architecture, an ML method, or any scientific topic, give a straight, widely-accepted, literal characterization of what the thing *actually is* and how the relevant research/industry community currently understands it. The owner is trying to understand the independent fields on their own terms. The analogies that matter live in the theory docs and are the owner's to make.

- **Distinguish established fact from speculation.** Be explicit about what is well-proven and accepted versus what is uncertain, contested, or speculative. Never present speculation as fact.

- **Implement and design; do not conclude.** Claude runs and builds experiments but should not draw conclusions from them. Be **conservative about claiming success or real findings** — the owner does not expect novel findings for a long time. Early experiments are simple, learning-oriented, and expected to produce nothing novel.

- **Go slow. Do exactly what is asked — nothing more, nothing less.** Start simple; add complexity incrementally. Do not jump ahead, over-build, or try to do everything in the first pass. Suggestions for additional or future experiments are welcome, but **describe them separately — do not implement them unprompted.**

## Layout

- `doc/report.md` — **the canonical theory reference: the most up-to-date and comprehensive statement of the theory** (and known to be incomplete). **Read this first.** The `> blockquotes` at the start of each section are the thesis of that section. The other two docs are supplemental to this one.
- `doc/theory-notes.md` — *supplemental.* A long (2400+ line) set of notes on adjacent topics from a conversation with Claude (scale-dependent symmetries, Wilsonian RG, Ginzburg-Landau, Landauer's principle, information bottleneck, rate-distortion, the Platonic Representation Hypothesis). Grep it for a specific concept rather than reading top-to-bottom. Not authoritative — `report.md` is.
- `doc/original/main.tex` (and `main.pdf`) — *supplemental.* The **outdated original draft** of `report.md`, written before the owner had studied information theory properly ("Statistical Mechanics and the Dynamics of Communication within Populations", Tim Healy, Spring 2025). Superseded by `report.md`.
- `encode.ipynb` — the sole computational experiment so far: sentence embeddings via `sentence-transformers` (`all-MiniLM-L6-v2`) on the `sentence-transformers/stsb` (STS-Benchmark) dataset, measuring cosine similarity between paraphrase pairs. This is the empirical probe for "do semantically equivalent sentences collapse to the same region of latent space."

## Working with the code

There is no build system, test suite, requirements file, or package config. The notebook depends on: `numpy`, `matplotlib`, `pandas`, `sentence-transformers`, `scikit-learn`, and `datasets` (Hugging Face). Running `encode.ipynb` downloads the MiniLM model weights and the STSB dataset from Hugging Face on first run.

## Conventions

- **Math notation must be KaTeX-compatible** in the markdown docs (they are rendered as part of a portfolio). Use `\mathrm{...}` rather than `\operatorname{...}` — this was a deliberate fix (commit `4042027`). Prefer `$...$` / `$$...$$`.
- The physics and the computation are tightly coupled — a change to the experimental method should be reflected in the "Computational Method" / "Research Statement" sections of `doc/report.md`, and vice versa. When editing theory, preserve the microstate/macrostate/coarse-graining vocabulary already established in the docs.
- LaTeX build artifacts under `doc/original/` are gitignored (`.aux`, `.log`, `main.pdf`, etc.).
