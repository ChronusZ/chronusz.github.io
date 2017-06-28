---
layout: default
title: "Timid Validation Framework"
date: 2017-06-27
use_math: true
---

# Timid Validation Framework

## Introduction

This is a loose description of the general framework we hope to be implemented for timid validation. Since timid validation changes the information each operator uses to make their decisions, it does not suffice to simply consider the "high-view" protocol. Instead we need some consideration as to what framework is needed in order to let the operators actually learn this extra information.

## Determining the Global Topology

This section outlines the framework we have in mind for how a node will learn about the "global topology" for running timid validation.

The basic idea is to run a program in parallel of Ripple consensus which looks through the network and records the UNL of (at least a large portion of) every node in the network. After speaking with David about our ideas, this is the section which seems to be most contentious, so I will highlight here the differences between my proposal and what I think David's proposal is.

### Ethan's Proposal

This proposal is designed with maximum safety in mind, and makes it impossible for two nodes to disagree on the topology. In exchange it sacrifices facility of network topology changes and increases the time it takes for a computer to begin running the protocol.

#### Synchronous Updates

There is some chosen timestep $$X$$ for updating the topology. $$X$$ should be long enough that a precise consensus algorithm in the CS sense can effectively always terminate within time $$X$$, but should be short enough that a node will be okay waiting up to $$2X$$ time to implement any desired changes to its UNL.

We imagine a consensus algorithm (in the computer science sense, and allowing Byzantine faults) running parallel alongside Ripple Consensus every timestep. At the beginning of each timestep, a node proposes the UNL it plans to switch to at the end of the timestep. The nodes then all come to consensus about what everyone's proposed UNL is, and at the end of the timestep they all make the switch.

It is very important that all nodes are synchronously updating their topology in order to allow each node to correctly assess the danger of forking with any other node. There may be a fringe case where asynchronous updates can lead to two nodes both thinking the other can never get past step $$1$$ when in actuality they can both *can* get past step $$1$$, thus leading to a situation where they *actually do* validate different ledgers under timid validation. This situation should never come up anywhere close to ordinary operations, but it seems to Ethan that decreasing facility of updating one's UNL is a minor cost for preventing forks from ever occuring in any situation.

#### Knowledge of the Global Topology

The consensus algorithm should aim to learn the entire topology of whatever strongly-connected component a node is in. This way a node will never fork with any other node in it's strongly-connected component. Since nodes in other components by definition cannot ever communicate with you, this ensures total safety with every node you have communication with.

### David's Proposal

This proposal is designed to maximize the network flexibility. It allows nodes to make immediate changes to their UNL to remove faulty nodes and decreases the time it takes for a computer to begin running the protocol. In exchange it may allow in a fringe situation for two safe nodes to validate different ledgers by misjuding each other's danger.

#### Asynchronous Updates

A node is able to change its UNL whenever it wants to. An algorithm runs parallel to Ripple Consensus and attempts to asynchronously crawl through the network and learn the UNL of every node one cares about.

Because of the fact that nodes learn about each other's UNL asynchronously, there may be a situation wherein two nodes even who are in each other's UNL might not correctly know each other's UNL at a given time. Under these circumstances, the two nodes might wrongfully misjudge each other's danger, and thus both validate on different ledgers. However, this event is quite unlikely, and if the two nodes have sufficient overlap to where Ripple validation would have been safe, they will still not fork even if this does happen.

#### Knowledge of a Portion of the Global Topology

Since each node crawls through the network on its own, a node can choose which portion of the network it wants to pay attention to. Timid validation would then only check the safety condition with these nodes; a node outside of this portion might validate a different ledger than you, but you don't care about that node so it doesn't matter. This notion is quite flexible and doesn't necessarily contradict the idea of knowing the UNL of everyone in your strongly-connected component (just make this the portion you care about). However, David seems to think that it's more okay for this to be small than Ethan does.

## Ostracization

This section outlines the framework we have in mind for how a node decides when to ostracize another node.

There are two types of ostracization we consider: faulty ostracization and perfomance ostracization.

### Faulty Ostracization

Our ideas here are quite hazy. Basically we imagine Ripple will monitor the network and see which nodes appear to be behaving suspiciously. In this case we will recommend that every other node ostracizes those nodes. Alternatively a node can of course independently decide when a node seems to be behaving suspiciously and then ostracize that node, but we should recommend that nodes do not do this unless they are very confident that node is unsafe, since ostracizing that node based on opinion might cause them to fork with it.

### Performance Ostracization

This is when a node ostracizes another node not necessarily because it thinks that node is faulty, but simply because it hopes that doing so will decrease the chance of it's progress being halted.

In looking for a guideline under which a node should decide to ostracize another node for performance, we maintain three tenets in order of decreasing importance:

1) Two nodes which correctly follow the guideline should never simultaneously ostracize each other.

2) If we need to choose between a well-connected node ostracizing a poorly-connected node or visa-versa, we should opt for the former.

3) As long as it can't contradict the first tenet, a node should ostracize as many nodes as possible.

The first tenet is of course important for safety. The only situation under which two nodes can't fork despite both ostracizing each other is when they have sufficient UNL overlap such that they would be aligned under Ripple validation; but in this case ostracizing each other doesn't do anything anyway (since they'll both always think the other is safe regardless), so in this situation ostracizing each other doesn't help anything. Thus there is no situation where both nodes ostracizing each other is both safe and useful, so it should always be avoided.

The second tenet is important because it preserves the incentive to be well-connected. If a node is poorly-connected, then that's its own fault and it should be the one halting, not everyone else.

The third tenet is just an optimization consideration; ostracizing a node increases the chance that you'll validate (or at least it doesn't decease the chance), so if it doesn't cause any safety concerns then one should ostracize as many nodes as possible to allow oneself to progress more easily.

These tenets lead us to the following amusingly simple guideline: **Ostracize a node whenever it has a UNL size strictly smaller than yours**. This is easily seen to satisfy tenet $$1$$, and it seems to be the best choice for tenets $$2$$ and $$3$$.

Note that if we choose David's proposal for learning the global topology, then it might be best to do something like **Ostracize a node whenever it has a UNL size smaller than e.g. $$90\%$$ of yours**. This is worse for tenet $$3$$, but it lowers the likelyhood that two nodes will ostracize each other due to asynchronous knowledge of the network topology, and it still is reasonable at satisfying tenet $$2$$.

## Timid Validation

For completeness we recall here the protocol outline for timid validation with ostracization. Let $$V_G$$ refer to the set of nodes of the graph, and for each node $$v\in V_G$$ let $$F(v)\subseteq V_G$$ refer to the set of nodes $$v$$ has ostracized.

At the beginning of validation, each node $$v\in V_G$$ picks a ledger, denoted $$X(v)$$. Each node runs the following algorithm to determine whether or not it should validate:

1. Look at the ledger of each node in $$UNL_v$$. If less than $$80\%$$ of the ledgers agree with $$v$$, immediately reject validation and terminate the algorithm.
2. For each node $$u\in V_G\setminus F(v)$$, switch on the value of $$X(u)$$ (Let $$\bot$$ denote "unknown"):

    a. If $$X(u)=X(v)$$, mark $$u$$ as safe.
    
    b. If $$X(u)\neq X(v)$$ and $$X(u)\neq\bot$$, mark $$u$$ as safe iff $$0.2\vert UNL_u \vert<\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=X(u)\vee X(w)=\bot\}$$.
    
    c. If $$X(u)=\bot$$, mark $$u$$ as safe iff for EVERY ledger $$L$$, $$0.2\vert UNL_u \vert<\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=L\vee X(w)=\bot\}$$.
3. If every node in $$V_G\setminus F(v)$$ is marked as safe, accept validation. Otherwise reject validation.
