---
layout: default
title: First try at specification
parent: TLA+ for Kopia's GC
nav_order: 5
---

Based on the basic abstraction of the the entities involved and the processes, we can start with an initial choice of data structures for the state variables and the steps of the spec. We will refine the state variables and steps as we go along. Note that I will show the refinement in state variables explicitly but will not show what the steps look like syntactically in each refinement of the specification. I will explain in plain english what each step does and when it is enabled. This will help keep the focus on important parts without getting lost in the syntactic details of the specification.

Also, I have constructed this process of refinement for better explaining the ideas. The ideas didn't evolve in the same order when writing the specifications. Also, the variable names vary as compared to the specifications found in the repository.

