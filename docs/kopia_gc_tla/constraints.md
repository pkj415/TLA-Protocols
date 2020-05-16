---
layout: default
title: Constraints for the model checker
parent: TLA+ for Kopia's GC
nav_order: 7
---

We use a couple of contraints to restrict the number of behaviours that TLC captures. In other words, content IDs in the actual implementation might be, say, a 16-bit value allowing 2^16 content IDs, but we surely don't want to run TLC to capture all behaviours with these content IDs. Also, it isn't feasible. For this, we keep only a set of possible content IDs in our world, say {1, 2, 3}.

We use the following constant to restrict the set of behaviours captured -
1. **NumContentIDs** - content IDs from 0 to NumContents-1 are allowed.
2. **NumSnapshots** - Maximum number of total snapshot process records in the system i.e., no more than NumSnapshots snapshot processes can be triggered
3. **NumGCs** - Maximum number of total GC process records in the system
4. **NumDataBlobCompactions** - Analogous to above
5. **NumIndexBlobCompactions** - Analogous to above
6. **MaxLogicalTime** - We don't want to allow a behaviour where only the logical timestamp keeps incrementing without any other step being taken ever. So, we restrict the maximum allowed time stamp in the system.

Apart from this, we have one other constant - MaxSnapshotTime (this is not a constraint to restrict behaviours specifically). This is the time within which a snapshot process can complete. If it doesn't it can never complete to generate a snapshot metadata entry (i.e., a snapshot).
