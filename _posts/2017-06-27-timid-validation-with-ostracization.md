---
layout: default
title: "Timid Validation with Ostracization"
date: 2017-06-27
use_math: true
---

# Timid Validation with Ostracization

This note is a small update to the proposed timid validation protocol. Instead of having a global $$f$$ parameter allowing nodes to ignore the dangers of up to $$f$$ nodes, it allows nodes to "ostracize" each other, thereby allowing a node to choose when to ignore the dangers of any other given node. This gives each node the ability to prevent trivial behavior caused by nodes which do not have sufficient overlap, and to protect themselves against Byzantine nodes which lie about their UNL in an attempt to make other nodes behave trivially. Additionally, timid validation with ostracization guarantees that two nonfaulty nodes will never validate different ledgers unless both have ostracized each other.

## Protocol Outline

Once again, we assume every node has access to the global topology. We also assume that every node $$v\in V_G$$ has an associated set $$F(v)\subseteq V_G$$ consisting of all the nodes $$v$$ has ostracized.

At the beginning of validation, each node $$v\in V_G$$ picks a ledger, denoted $$X(v)$$. Each node runs the following algorithm to determine whether or not it should validate:

1. Wait until we have heard the proposal of enough nodes in $$UNL_v$$ to where $$80\%$$ of them agree on a single ledger; if this ever becomes impossible, immediately reject validation and terminate the algorithm. If there was actually at least $$80\%$$ agreement on some ledger $$L$$, keep track of the ledger $$L$$ as a private variable.
2. Let $$\bot$$ denote "unknown". For each node $$u\in V_G\setminus F(v)$$, mark $$u$$ as safe iff for every ledger $$L'\neq L$$, $$0.2\vert UNL_u \vert<\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=L'\vee X(w)=\bot\}$$.
3. If every node in $$V_G\setminus F(v)$$ is marked as safe, fully validate the ledger $$L$$. Otherwise reject validation.

This protocol still boasts all the specs of regular timid validation: a node will never validate if it wouldn't have validated under Ripple validation, and in any graph which is guaranteed safe under Ripple validation, timid validation with ostracization behaves exactly the same as Ripple validation.

Further, assume that two nodes $$v$$ and $$u$$ are both nonfaulty and honest about their UNLs, and $$v$$ does not ostracize $$u$$. Then if there is any chance that $$u$$ will progress past step $$1$$ on a ledger different from $$v$$, $$v$$ will refuse to validate. Thus the only way for two nonfaulty honest nodes to validate different ledgers is if both nodes are simultaneously ostracizing each other.
