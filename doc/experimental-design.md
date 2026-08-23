# Experimentation Plan

This document outlines the high level objectives and low level details of the computational experiments we will run. Our primary goal is to look for equilibrium statistical mechanic macrostate structure in natural language by using encoder-only transformer LLMs to define equivalence classes by semantic meaning.

## Table of Contents: Pillars of the experiment

* [Characterizing the encoder model](#characterizing-the-encoder-model)
* [Getting the proper datasets; datasets will mimic the microcanonical and canonical ensemble](#getting-the-proper-datasets-datasets-will-mimic-the-microcanonical-and-canonical-ensemble)
* [The validation plan; what observables will we look for and how do we determine success?](#the-validation-plan-what-observables-will-we-look-for-and-how-do-we-determine-success)

## Characterizing the encoder model

Before testing our hypothesis, we must understand the tools we are using. We must understand the performance of the encoder model we use to define the projection.

We will also be interested in substituting the encoder model for a simple rate distortion method as well as an IB method approach to defining the projection. We do not expect these results to work well, but it is a baseline to compare against.

## Getting the proper datasets; datasets will mimic the microcanonical and canonical ensemble

We hope to find a dataset which fulfills the assumptions of the microcanonical and canonical ensemble. This may be difficult to find and thus we may need to loosen this requirement and look for datasets which mimic a non-equilibrium steady state system.

## The validation plan; what observables will we look for and how do we determine success?

Our primary form of validating our results will be measuring the multiplicity of macrostates which is the cardinality of the semantic equivalance classes.