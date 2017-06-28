---
layout: default
title: "Timid Validation with Optimistic Waiting"
date: 2017-06-28
use_math: true
---

# Timid Validation with Optimistic Waiting

This note is yet another small update to the proposed timid validation protocol which came out of a discussion with Brad. It adds an "optimistic waiting" step for when a node has passed step $$1$$ and then failed step $$2$$, but it has not yet received the votes of everyone on its UNL and there might be some way to pass step two given this extra information. In doing so the algorithm gains a (perhaps slight) boost in the likelihood of validating. We are still unsure whether or not adding this extra step is a good idea, but hopefully simulations can give us a better idea of how helpful this extra step might be.

The update can be easily patched onto either timid validation with or without ostracization, but throughout this note I'll assume we're using ostracization because I think it's superior. It should hopefully be clear which minor changes one would need to make to add these new changes to timid validation without ostracization.

## Protocol Outline

Once again, we assume every node has access to the global topology. We also assume that every node $$v\in V_G$$ has an associated set $$F(v)\subseteq V_G$$ consisting of all the nodes $$v$$ has ostracized. We also add an extra time parameter $$D$$ (perhaps $$1$$ second or so), which is the length of time a node is willing to wait in order to confirm an optimistic judgement.

At the beginning of validation, each node $$v\in V_G$$ picks a ledger, denoted $$X(v)$$. Each node runs the following algorithm to determine whether or not it should validate:

1. Wait until we have heard the proposal of enough nodes in $$UNL_v$$ to where $$80\%$$ of them agree on a single ledger; if this ever becomes impossible, immediately reject validation and terminate the algorithm. If there was actually at least $$80\%$$ agreement on some ledger $$L$$, keep track of the ledger $$L$$ as a private variable.
2. Let $$\bot$$ denote "unknown". For each node $$u\in V_G\setminus F(v)$$, mark $$u$$ as safe iff for every ledger $$L'\neq L$$, $$0.2\vert UNL_u \vert<\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=L'\vee X(w)=\bot\}$$.
3. If every node in $$V_G\setminus F(v)$$ is marked as safe, fully validate the ledger $$L$$ and terminate the algorithm.
4. For each node $$u\in V_G\setminus F(v)$$, mark $$u$$ as *potentially safe* iff for every ledger $$L'\neq L$$, $$0.2\vert UNL_u \vert<\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=L'\}$$.
5. If every node in $$V_G\setminus F(v)$$ is marked as *potentially safe*, wait for time $$D$$. Otherwise immediately reject validation and terminate the algorithm.
6. Repeat step $$2$$ with any new information we might have received about the votes of our neighbors.
7. If every node in $$V_G\setminus F(v)$$ is then marked as *actually safe*, fully validate ledger $$L$$. Otherwise reject validation.

Step $$4$$ is the "optimistic step". It determines whether there is any chance that given new information about the votes not yet received from your neighbors you might have passed step $$2$$. If so we wait a little while longer to see if our optimistic judgement was correct.
