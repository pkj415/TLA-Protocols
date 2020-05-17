---
layout: default
title: TLA+ for PBFT
nav_order: 4
---

PBFT is a byzantine fault tolerant consensus protocol used to realise a replicated state machine [^1]. The specification for it is in progress and can be found [here](https://github.com/pkj415/PBFT-TLA).

I will shortly be adding a section on how we can traverse all behaviours in which byzantine behaviour can occur to verify that the algorithm indeed maintains safety in all of them.

[^1]: Practical Byzantine Fault Tolerance - [http://pmg.csail.mit.edu/papers/osdi99.pdf](http://pmg.csail.mit.edu/papers/osdi99.pdf)
