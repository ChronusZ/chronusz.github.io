---
layout: default
title: "Timid Validation"
date: 2017-06-26
use_math: true
---

# Timid Validation

Here we outline a proposal for a modified validation algorithm called **timid validation**. Timid validation is so-called because each node only validates when it knows that there is no chance another safe node will validate a different ledger. The algorithm makes use of knowledge of the global topology of the graph. The algorithm allows a safety parameter $$f$$, thought of as the number of allowed "faulty" nodes, which can be either Byzantine nodes lying about their UNL or nodes which are not well connected. When $$f=0$$, the protocol never forks on any graph, although low connectivity can make the nodes never validate. For higher values of $$f$$, some graphs become forkable but the algorithm is less likely to make nodes refuse to validate. And even with arbitrarily high $$f$$, timid validation will never fork on a graph for which Ripple validation is safe. Further, in any graph where Ripple validation is guaranteed to be safe, timid validation will behave exactly the same as Ripple validation, so under usual operations nothing is lost by switching over to timid validation except for a miniscule increase in computation time.

Timid validation is advantageous over more naive safety-improvement protocols like double validation (wherein one simply doubles their UNL and runs Ripple validation) because it does not induce an increased network cost over regular Ripple validation; although each individual computer needs to do more computations to determine whether or not to validate, there is no extra need to communicate with other nodes. With the speed of modern computers the extra computation time of timid validation is negligible compared to the time it takes to communicate over the network and learn the states of more nodes.

## Protocol Outline

We assume that every node has access to the global topology. However, by the way the algorithm is deigned, even if nodes lie about their UNL this cannot make the network fork (although if $$f$$ is low it can make the protocol refuse to ever validate in some graphs).

At the beginning of validation, each node $$v\in V_G$$ picks a ledger, denoted $$X(v)$$. Each node runs the following algorithm to determine whether or not it should validate:

1. Look at the ledger of each node in $$UNL_v$$. If less than $$80\%$$ of the ledgers agree with $$v$$, immediately reject validation and terminate the algorithm.
2. For each node $$u\in V_G$$, switch on the value of $$X(u)$$:

    a. If $$X(u)=X(v)$$, mark $$u$$ as safe.
    
    b. If $$X(u)\neq X(v)$$, mark $$u$$ as safe iff $$0.2\vert UNL_u \vert<\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=X(u)\}$$.
    
    c. If $$X(u)$$ is unknown, mark $$u$$ as safe iff for EVERY ledger $$L$$, $$0.2\vert UNL_u \vert<\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=L\}$$.
3. Count up all the nodes which have not been marked safe. If this number is greater than or equal to $$f$$, reject validation and terminate the algorithm. Otherwise accept validation and terminate the program.

Note that by step $$1$$, a node will never validate if it wouldn't have validated under Ripple validation. Thus timid validation is strictly safer than Ripple validation.

Step $$2$$ of the algorithm is a "worst-case analysis"; it marks a node as safe if and only if there is no possibility for that node to validate a different ledger than the one we are proposing.

Recall that a graph is safe under Ripple consensus iff for every pair of nodes $$v,u\in V_G$$, $$\vert UNL_v\cap UNL_u\vert>\lfloor 0.2\vert UNL_v\vert\rfloor + \lfloor 0.2\vert UNL_u\vert\rfloor$$. In fact, since $$\vert UNL_v\cap UNL_u\vert$$ is an integer, we have $$\vert UNL_v\cap UNL_u\vert-0.2\vert UNL_v\vert>\lfloor 0.2\vert UNL_u\vert\rfloor$$.

If a node $$v$$ validates under Ripple consensus, then for any node $$u$$ and any ledger $$L$$ contradicting $$X(v)$$, we must have that $$\#\{w\in UNL_v\cap UNL_u\vert X(w)=L\}<0.2\vert UNL_v\vert$$. Thus
\begin{aligned}
\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=L\}&>\vert UNL_v\cap UNL_u\vert - 0.2\vert UNL_v\vert \\\
&>\lfloor 0.2\vert UNL_u\vert\rfloor
\end{aligned}
and since $$\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=L\}$$ is an integer, $$0.2\vert UNL_u \vert<\vert UNL_v\cap UNL_u\vert - \#\{w\in UNL_v\cap UNL_u\vert X(w)=L\}$$ thus always holds. Thus in the case of graphs where Ripple consensus guarantees safety, timid consensus will behave in exactly the same way.

## Heuristics

We have yet to do much testing of the algorithm, so our heuristics are currently limited. Unlike with Ripple validation, we do not have a closed formula for determining when a graph is forkable under timid validation.

We tested timid validation on all undirected graphs of at most eight nodes with $$f=0$$ and as expected no graphs fork.

When $$f=1$$, we tested graphs of nodes $$5,6,7,$$ and $$8$$. Our results showed the following:

**$$5$$ nodes**:
Total graphs: $$21$$

Graphs which are safe under Ripple validation: $$15$$

Graphs which are safe under timid validation: $$18$$

**$$6$$ nodes**:
Total graphs: $$112$$

Graphs which are safe under Ripple validation: $$56$$

Graphs which are safe under timid validation: $$97$$

**$$7$$ nodes**:
Total graphs: $$853$$

Graphs which are safe under Ripple validation: $$265$$

Graphs which are safe under timid validation: $$719$$

**$$8$$ nodes**:
Total graphs: $$11117$$

Graphs which are safe under Ripple validation: $$1255$$

Graphs which are safe under timid validation: $$7672$$

Although more testing is of course necessary to make an unbiased judgement about how much safer timid validation is compared to Ripple validation, these numbers do seem to suggest a quite reasonable improvement in safety.
