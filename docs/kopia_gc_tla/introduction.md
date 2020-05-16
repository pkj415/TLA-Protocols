---
layout: default
title: Introduction
parent: TLA+ for Kopia's GC
nav_order: 1
---

[Kopia](https://kopia.io/) is a backup/restore tool which allows creating snapshots of filesystem contents and uploading them to some specified remote cloud storage (it supports S3, GCS, and a few more).

As of [this commit](https://github.com/kopia/kopia/tree/9c3d419bc396c0446f21c68813ca195196843eab), Kopia does non-quiescent periodic garbage collection (abbreviated GC) to free up data corresponding to deleted snapshots which is known to break safety conditions (data of a snapshot is deleted even when the snapshot isn't deleted). Let us call the strategy of garbage collection followed in the specified commit as vanilla garbage collection. The TLA+ specification in the master branch of [this repository](https://github.com/pkj415/KopiaTLA) corresponds to vanilla GC and is written to confirm and identify the behaviour where safety is broken. There is a new design of GC (called GC2) documented [here](https://docs.google.com/document/d/15Q6UkbQ6Qz7OxZ15TTyQ3su7HYFvAWMwDEbxvtKDpvA/edit?usp=sharing) by Julio Lopez. The document also explains why vanilla GC breaks safety (I will explain it here as well). The specification in [gc2 branch](https://github.com/pkj415/KopiaTLA/tree/gc2) corresponds to the "GC with metadata" design as presented in the GC2 design document (we will call this gc2 unless explicitly mentioned). However, GC2 doesn't guarantee safety either and the specification for gc2 demonstrates this. Later sections explain the various optimizations and tricks used to abstract out relevant parts of the implementation to be specified. The final section explains the behaviour that violates safety in GC2.

The architecture of Kopia is explained well [here](https://kopia.io/docs/architecture/). I will explain the important pieces to keep in mind for our purposes of specifying behaviours relevant to GC in Kopia.
