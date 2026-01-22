# Self‑Declarative Pipelines: A Palantir Foundry Perspective (with Foundry‑Native Examples)

## Collab

## TL;DR
Spark 4.1’s Spark Declarative Pipelines (SDP) popularize a dataset‑centric approach—declare the datasets and queries; let the engine plan and run the graph. In Palantir Foundry, you build the same kind of pipelines using Transforms (@transform, @transform_df, @incremental) that materialize datasets and are orchestrated by Pipelines with scheduling, lineage, and governance built‑in. 

## Overview
Data engineering is moving from imperative, job‑centric orchestration to dataset‑centric systems. SDP formalizes this in core Spark: you specify what datasets should exist and how they’re derived; Spark plans the graph, retries, and incremental updates for you (SDP guide). Foundry embodies the same philosophy: you declare Transforms that read inputs and write outputs (datasets); Pipelines schedule execution; and Lineage tracks the DAG across your enterprise (Transforms, Lineage, Pipelines).
