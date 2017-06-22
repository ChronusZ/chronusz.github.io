---
layout: default
title: "Validation Notation"
date: 2017-06-22
use_math: true
---

# Validation Style Guide

## Introduction

This is a general guide for various notations relating to Ripple validation. Due to the subtle nature of arguments about validation, it is important to be able to communicate effectively to avoid unnecessary confusion. Most of this notation is obvious (indeed, it was chosen to be as obvious as possible!), but there are some key points highlighting language which can lead to confusions and should thus be avoided. The first few sections mostly just go over basic concerns about definitions already in place. Later we introduce some new language that might be convenient for talking about validation.

This document is still majorly under construction. If you disagree about some notation or want to add some definitions, feel free to make a change to the document by editting the page on [my GitHub](https://github.com/ChronusZ/chronusz.github.io/tree/master/_posts).


## Notations for the Validation Network Structure

There are two useful equivalent notations for the **validation network structure**: the **digraph structure**, the **UNL structure**. Depending on the example being looked at, different notations are more or less useful. There is also a third notation called the the **clique structure**, which is significantly less expressive than the other two structures but can be very useful for efficiently describing certain examples.

#### Digraph Structure

The digraph structure simply defines the validation network to be a directed graph $$G=(V,E)$$. The elements of $$V$$ are referred to as **nodes** and the elements of $$E$$ are referred to as **edges**. For consistent notation, it's probably best to avoid referring to nodes as vectors like graph theorists typically do. An edge $$e$$ with pointing from $$u$$ to $$v$$ can be denoted $$u\to v$$ or $$(u,v)$$ or just $$uv$$. I think $$u\to v$$ is the best since there can be no confusion about the direction the edge points. In situations where LaTeX is unavailable like Slack, one can still write $$u$$->$$v$$ as shorthand.

Nodes should be denoted using the lowercase letters $$u,v,w$$ or by natural numbers (possibly including zero if you swing that way). Edges should be denoted by $$e$$, perhaps adding a subscript natural number if one needs to refer to multiple edges. I think it's best to reserve the letters $$G,V,E$$ for referring to a digraph structure to avoid confusion, thus we can simply refer to $$E$$ without context and it should be clear that we are referring to a set of edges. If there is confusion though, we can write $$V(G)$$ or $$E(G)$$ to refer to the nodes and edges, and this should never cause confusion I think.

Since it seems that most people on RippleD and on the research team think in terms of the **trust direction** where $$u\to v$$ means $$u$$ **trusts** $$v$$ or $$u$$ **listens to** $$v$$, we should be consistent and always use the trust direction instead of the **broadcast direction** where $$u\to v$$ means $$u$$ **broadcasts to** $$v$$. The broadcast direction is used more often in the literature though, so when writing for communication with the outside it's probably better to use the broadcast direction.

The **trusted neighbors** or **out-neighbors** of a node $$v$$ are the nodes $$u\in V$$ such that $$(v\to u)\in E$$. We can denote the set of trusted neighbors of $$v$$ by $$N_v$$ or in situations without LaTeX available, by $$N(v)$$. The **broadcast neighbors** or **in-neighbors** of a node $$v$$ are the nodes $$u\in V$$ such that $$(u\to v)\in E$$. We can denote the set of broadcast neighbors of $$v$$ by $$B_v$$ or $$B(v)$$. Using the explicit network terms "trusted" and "broadcast" is probably better than using the graph-theoretic terms "out" and "in", since the graph-theoretic terms are easier to mix up.

#### UNL Structure

The digraph structure is most convenient for graphs which have low similarity between two vertices. The UNL structure however is much more efficient when the set of nodes can be partitioned into large symmetric subsets. A **UNL** is defined simply to be a set of nodes. A node $$v$$ **listens to** or **trusts** a UNL $$X$$ if $$X=N_v$$.

The uppercase letters $$X,Y,Z$$ should be used to refer to UNLs. The notation $$UNL_v$$ or $$UNL(v)$$ can be used to refer to the UNL that the node $$v$$ listens to, but should not be used as a label for the UNL itself unless $$v$$ is the only node listening to $$UNL_v$$. For instance, the whitepaper counterexample in the German paper names its UNLs $$UNL_v$$ and $$UNL_u$$ despite the fact that several nodes are listening to each of them. This confused Haoxing and me for a long time.

#### Clique Structure

Lastly we have the clique structure. The clique structure is very effective for defining many types of simplistic example graphs we are interested in, but it lacks the ability to express directed edges, and for more complicated graphs it can overcomplicate things. In general it is best used in combination with UNL notation or digraph notation, by first defining the cliques and then specifying the extra connections between cliques.

Just like a UNL, a **clique** is simply defined as a set of nodes. The interpretation of the set is different though. A node $$v$$ is **a member of a clique** $$Q$$ if $$v\in Q$$. Being a member of the clique $$Q$$ further necessitates that $$Q\subseteq N_v$$. Thus every node in a clique listens to every other node in that clique. We can treat cliques as though they are single nodes in order to use the digraph and UNL notations "in-batch": for example, a clique $$Q$$ trusts a node $$u$$ if every node $$v\in Q$$ trusts $$u$$, and $$Q$$ listens to a UNL $$X$$ if every node $$v\in Q$$ listens to $$X$$ (note that this requires that $$Q\subseteq X$$).

The uppercase letters $$Q,R,S$$ should be used to refer to cliques. Note that whereas a node can necessarily only listen to a single UNL, a node can be a member of multiple cliques. Thus it doesn't make sense to refer to *the* clique that a given node is a member of.

The following visualization of cliques works well in my opinion:

<p align="center"><img src="https://raw.githubusercontent.com/ChronusZ/chronusz.github.io/master/images/clique_blob.png"></p>

## Notations for the Validation Process

Validation begins with each node **proposing** a **ledger**. When discussing validation, I don't think it's useful to explicitly consider the ledgers as consisting of a set of transactions. Instead they can simply be considered as elements in an arbitrary set. We can use the uppercase letters $$A,B,C,D$$ to refer to ledgers, or we can denote them simply by natural numbers. Obviously if the nodes are being labelled by natural numbers the ledgers should be labelled using letters.

For each node $$v\in V$$, the ledger initially proposed by $$v$$ is denoted $$p(v)$$. At the end of validation, each node either **accepts** a ledger or **sits**, meaning it doesn't accept any ledger. The ledger accepted by $$v$$ is denoted by $$l(v)$$ ('p' for proposal, 'l' for... ledger?). If $$v$$ sits, then $$l(v)=\bot$$.

Two nodes $$u$$ and $$v$$ are said to **align** if whenever both nodes accept some ledger, the ledger accepted by them is the same. Thus if $$u$$ and $$v$$ align, it could be the case that $$l(u)=A$$ and $$l(v)=\bot$$, but it can never be the case that $$l(u)=A$$ and $$l(v)=B$$ for some ledgers $$A\neq B$$. We use the notation $$v\sim u$$ to denote that $$v$$ and $$u$$ are aligned. This is a reflexive, symmetric relation, but ***it is not transitive*** as the following example shows:

<p align="center"><img src="https://raw.githubusercontent.com/ChronusZ/chronusz.github.io/master/images/nontransitive_alignment.png"></p>

For this graph, it is easily seen that $$u\sim w$$ and $$v\sim w$$ by the condition given in the German paper. However, with the ledgers proposed as above, $$u$$ accepts ledger $$B$$ whereas $$v$$ accepts ledger $$A$$, so $$u\nsim v$$.

## Notations for Math Concepts

The notations in this section are not explicitly related to validation, but rather they set in stone certain concepts in mathematics that are useful for talking about validation, but for which there is not total agreement even among mathematicians about how they are defined.

#### Linear Algebra

An $$m\times n$$ matrix refers to a matrix with $$m$$ rows and $$n$$ columns. If $$M$$ is a matrix, $$M_{ij}$$ or $$M(i,j)$$ refers to the component in row $$i$$ and column $$j$$.

#### Graph Theory

Let $$G$$ be a graph with $$n$$ nodes. We assume the nodes are totally ordered $$v_1,...,v_n$$. The **adjacency matrix** of $$G$$ is the $$n\times n$$ matrix $$A_G$$ (When LaTeX is unavailable, $$A(G)$$ or just $$A$$ if there's no chance of confusion with ledger labels---which there shouldn't ever be) which has $$A_G(i,j)=1$$ when $$(v_i\to v_j)\in E(G)$$ and $$A_G(i,j)=0$$ otherwise. Since all our graphs have a loop at every node, $$A_G(i,i)=1$$ for every $$i\leqslant n$$.

The **trust matrix** or **out-degree matrix** of $$G$$ is the $$n\times n$$ diagonal matrix $$D_G$$ with $$D_G(i,i)=\vert UNL(v_i)\vert$$. In other words, the $$i$$-th diagonal component is the **out-degree** of $$v_i$$. "Trust matrix" is completely nonstandard terminology, but it's slightly more clear than out-degree matrix.

The **Laplacian matrix** or **Laplacian operator** of $$G$$ is the $$n\times n$$ matrix $$L_G=D-A$$. Note that this is different from the Laplacian that many people consider, which uses the total-degree matrix rather than the out-degree matrix.
