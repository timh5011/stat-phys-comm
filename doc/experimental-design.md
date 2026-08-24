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

We hope to find a dataset which fulfills the assumptions of the microcanonical and canonical ensemble. This may be difficult to find and thus we may need to loosen this requirement and look for datasets which mimic a non-equilibrium steady state system.

## The validation plan; what observables will we look for and how do we determine success?

Our primary form of validating our results will be measuring the multiplicity of macrostates which is the cardinality of the semantic equivalance classes.

The encoder will project the dataset into a latent space. We will then partition the latent space according to a scale.