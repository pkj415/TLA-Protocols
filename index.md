---
layout: page
title: TLA+ for ViewStamped Replication, PBFT and Kopia's Garbage collection protocol
---

This project page describes my work on checking correctness of distributed protocols and concurrent systems by specifying behaviours in [TLA+](https://lamport.azurewebsites.net/tla/tla.html) and using the TLC model checker. This is done for two well-known distributed protocols for consensus and for a non-quiescent garbage collection process [^1] in a filesystem backup & restore tool (Kopia). Only safety is verified for the two consensus protocols - ViewStamped Replication (VR) and Practical Byzantine Fault Tolerance (PBFT), since it has been proven that liveness cannot be guaranteed by any consensus protocol [^4]. VR remains safe in face of non-byzantine failures (upto n/2 failures in a network of n processes) while PBFT remains safe even in case of byzantine failures (upto n/3 failures in a network of n processes). The TLA+ spec (short for specification) for VR proves safety of the protocol as specified in the VR paper[^2] (with the exclusion of the reconfiguration protocol). The spec for PBFT is in progress; the section on PBFT's spec describes how safety will be verified. Finally, model checking is used to identify safety violations in the design of Kopia's garbage collection (abbreviated GC) protocol to refine the design and conclude on a safe protocol. Liveness of the protocol is not touched upon as it is trivial to argue to about liveness of the GC protocol.

A side note - I will try to not be sloppy in terminology as much as possible to avoid ambiguity. Also, most of the terminology is derived from the TLA+.

It is difficult to reason about correctness (safety and liveness) of distributed protocols. Before we start off with how to make this process easier for certain protocols, a short brief about certain definitions -

1. **State machine -** Every distributed protocol can be represented by a state machine which is a sequence of states and steps. The state machine starts at an initial state and every state is followed by a step and a corresponding next state until a terminal state is reached.
2. **State -** A state represents all variables in the world of the protocol (in case of a distributed protocol, it is a combination of all the local states of processes).
3. **Step -** A step is a transition in the state machine which enables the state machine to move from one state to another when some preconditions hold true. We call a step "enabled" when these preconditions hold on a state.
4. **Behaviour -** A behaviour of a protocol refers to a particular sequence of states and steps which follow the specified protocol.
5. **Many to one relationship of behaviours with runs of the protocol -** When a distributed protocol is represented as a state machine, there is a total order on the sequence of actions performed by the different entities in every possbile behaviour. Many behaviours might correspond to a single run[^5] of a distributed protocol in case there are concurrent events in the run (an event corresponds to a step in the behaviour). n concurrent events can be ordered in n! different ways in a behaviour and hence the many to one relationship.

[^1]: By non-quiesent garbage collection I mean that the garbage collection process runs in the background while other system processes function as usual without any hinderance.
[^2]: Viewstamped Replication Revisited - [http://pmg.csail.mit.edu/papers/vr-revisited.pdf](http://pmg.csail.mit.edu/papers/vr-revisited.pdf)
[^3]: Practical Byzantine Fault Tolerance - [http://pmg.csail.mit.edu/papers/osdi99.pdf](http://pmg.csail.mit.edu/papers/osdi99.pdf)
[^4]: The FLP Impossibility rule - [https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf)
[^5]: Consistent global states of distributed systems - [https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/chapt4.pdf](https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/chapt4.pdf)

For arguing about safety, we will consider all possible behaviours in the system and hence all possible runs. Distributed protocols are hard to reason about because at each state, multiple steps might be enabled and the system might transition into any one of the possible next states based on these steps. And multiple steps being enabled at a step represent the possibilty that the system might go into any state in reality and we would have to reason about correctness on all possible next states and sequence of steps post them to reason about the protocol as a whole.

Also, I will show only the states and steps relevant to any discussion. You can assume there might be many more possible next steps at states that are shown and any possible step to be taken in case I don't show a green arrow (which means the next step isn't relevant to the discussion). I will point out places with exceptions to this rule, if required.

I have given a short background of terms, but I encourage you to learn about TLA+ to get the most out of this document. This [video lecture series](https://lamport.azurewebsites.net/video/videos.html) by Leslie Lamport is a great place to start (and actually a great place to end, it is sufficient to understand everything we are going to use. Also there is just 4 hours of video content in total, though I assume it will surely take more than 4 hours to grasp the details).

Getting back to our goal of making the process of reasoning about correctness easier, a model checking tool is used to verify that correctness is maintained in all possible behaviours (in a scoped manner, see the second point) of the protocol. The model checker enumerates all possible behaviours and checks safety properties for all of them. For this, the model checker is to be instructed about -

1. What behaviours are allowed by the protocol that is to be checked. This requires specifying the protocol in some form at an abstract level without implementation details. I have used TLA+ to specify behaviours. It might seem vague when I say "abstract level" but I found it difficult to define this precisely and maybe there is no definition to this. For now, just keep in mind that we intend to allow all behaviours without adornments. By adornments I mean to the steps and state variables which are redundant (steps which are such, that not having them won't change the number of behaviours the model checker checks and state variables which are exacly correlated with another state and removing them won't change the number of possible distinct states). Usually lower level implementation details correpond to these adornments.

2. What are the correctness conditions. Correctness conditions are specified using temporal logic expressions.

3. What scope/extent of behaviours are to be checked. Let's take an example - assume any distributed protocol (say Paxos) that can be run on any number of nodes/processes (>= 3 in number). Proving correctness of such a distributed protocol would require us to run the model checker on the protocol for all scenarios i.e., with number of nodes ranging from 3 to infinity. Obviously, this is not possible and hence it is common to resort to a minimal set of constraints to verify that the protocol is atleast correct with those constraints. Note that we can't prove it for all behaviours allowed by the protocol for all sets of inputs, but we can prove it (which I have termed "verify") for a minimal set of inputs (i.e., with some constraints on inputs). In the case of simple Paxos, the constraint would be on just the number of nodes. As we discuss further about specifications of different protocols, I will point out the constraints specifically.

The following sections describe in detail, the use of TLA+ and TLC for the two distributed protocol and the garbage collection protocol -

- [Viewstamped Replication](pages/vr_tla.html)
- [Garbage collection in Kopia](pages/kopia_gc_tla.html)
- [Practical Byzantine Fault Tolerance](pages/pbft_tla.html)