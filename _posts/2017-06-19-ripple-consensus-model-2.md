---
layout: default
title: "Ripple Consensus Model 2"
date: 2017-06-19
use_math: true
---

# Convergence of Consensus

## Model

I would like to use a model of consensus which behaves as a randomized model for the actual way that Ripple consensus works. I abstract away some of the parameters in the hope that we might be able to analyze which parameters give the best results, but I hope that plugging in the parameters currently in-use would make the model behave the same as the actual consensus algorithm under only the assumption that nodes behave consistently (i.e., according to some random distribution which does not vary over time) within each round of consensus. Since the time-span of a round of consensus is so short, I think this is a reasonable assumption to make and I believe shouldn't affect the probabilities too much. This does not account for the possibility of (Byzantine or crash) faults, but in the section below on faults I discuss a potential way to include these faults in the analysis. If anyone thinks nodes behaving non-faultily shouldn't be modeled by a random process in the way I propose below, please let me hear your thoughts so I can possibly adjust the model if need be.

Also if anyone notices something about the model that does not match up with how Ripple consensus works, please let me know so that I can fix the model. I find the actual algorithm very confusing so there is a high chance that I messed something up. The only thing that I intentionally changed about the protocol was to make it so that people are voting on individual transactions rather than full ledgers. This is only a simplifying assumption to make the model more feasible. If this strikes anyone as something that could significantly affect the probabilities, again please let me hear your thoughts.

On to the model outline.

As always, let $$G=(V,E)$$ be a directed graph with adjacency matrix $$A$$. For a given node $$v\in V$$, let $$k^v$$ be the set of neighbors of $$v$$, i.e., $$N^v=\{u\in V\vert(u\to v)\in E\}$$. For simplicity we assume that each node in $$V$$ is indexed by a natural number, and we refer to the node and its index interchangeably. We assume that every node is its own neighbor. Let $$D$$ be the diagonal in-degree matrix. Define the **Laplacian operator** $$L$$ by $$L=D-A$$. We call $$X=\{0,1\}^V$$ the **state space** and elements $$x\in X$$ **state vectors**. The components of the state vector $$x$$ are denoted $$x^v$$ for $$v\in V$$. A matrix acts on a state vector $$x$$ by treating $$x$$ as an element of the vector space $$\mathbb{R}^V$$. The result of applying $$D^{-1}L$$ to the state vector $$x$$ is a (real) vector whose $$v$$-th component is the difference between $$x^v$$ and the average value across the neighbors of $$v$$. Although graph theorists like to call elements of $$V$$ vectors, we try to reserve this terminology for referring only to elements of $$X$$ or $$\mathbb{R}^V$$ and instead always refer to elements of $$V$$ as **nodes**.

We ignore relativistic issues and assume the existence of a continuous **absolute time** which is represented by a real number $$t\in[0,\infty)]$$. Each node keeps track of a discrete **internal timestep**, which is represented by a natural number $$n^v\in\mathbb{N}$$. We will often identify $$n^v$$ with the absolute time at which $$n^v$$ is called. Let $$\tau:\mathbb{N}\to[0.5,1)]$$ be a monotone-increasing function. We call $$\tau$$ the **threshold function**. For a given timestep $$n^v$$, $$\tau(n)$$ gives the current threshold of agreement for the node $$v$$; note that although $$\tau$$ takes as input the internal timestep of a node, the function is assumed to be the same for every node. Thus for instance, node $$a$$ may get to its third timestep after $$3.1$$ seconds while node $$b$$ might get to its third timestep after only $$2.8$$ seconds, but the threshold at that step will be the same for both $$a$$ and $$b$$.

For each node $$v\in V$$, let $$dt^v$$ be a probability measure on $$(0,\infty)$$ interpreted as the length of absolute time it takes the node $$v$$ to get to the next internal timestep. Similarly, for each edge $$e\in E$$, let $$dt^e$$ be a probability measure on $$(0,\infty)$$ interpreted as the absolute time it takes the edge $$e$$ to transmit a message. Note that we consider undirected edges as consisting of two distinct edges, so the distribution in one direction may be different from the distribution in the other direction.

Heuristically, I think $$dt^v$$ and $$dt^e$$ should probably be gamma distributions or maybe beta distributions, and probably have pretty small standard deviations I guess. It should be noted that with the randomness of $$dt^e$$, it could happen that a node sends out message $$B$$ after node $$A$$, but an out-neighbor hears the message $$A$$ after node $$B$$. I don't know if this can happen in real life or not; if not I'll try to come up with some way of precluding this behavior. Either way, if the standard deviation of $$dt^e$$ is sufficiently small compared to the mean of $$dt^v$$ where $$v$$ is the input of $$e$$, this will happen only very infrequently.

For a given absolute time $$t$$, let $$x_t$$ denote the current **absolute state vector** or just the **absolute state**. The absolute state is updated synchronously whenever a node goes through its timestep protocol. Note however that the absolute state is a purely formal construction; because of asynchrony, a node will in general not be aware of the current values of the absolute state, even for its immediate neighbors. For a given node $$v$$, we define the **local state** $$l_n^v\in\{0,1\}^{N^v}$$ to be the most recently received values of the neighbors of $$v$$ at timestep $$n^v$$. The components of $$l_n^v$$ are denoted $$l_n^{v,u}$$ for $$u\in N^v$$.

The algorithm behaves for a given node $$v$$ at absolute time $$t$$ and internal time $$n^v$$ as follows:

1. For every neighbor $$u\in N^v$$, check the most recent message which has arrived from $$u$$, and update $$l_n^v$$ accordingly.
2. Compute $$S=\frac{1}{\vert N^v\vert }\sum_{u\in N^v} l_n^{v,u}$$. If $$S>\tau(t^v)$$, set $$x^v=1$$. If $$S\leqslant\tau(n^v)$$, set $$x^v=0$$.
3. For each edge $$e\in E$$ pointing out from $$v$$, pick a number $$t^e$$ according to $$dt^e$$ and broadcast the new value of $$x^v$$ so that it arrives at $$t+t^e$$.
4. Pick a number $$t^v$$ according to $$dt^v$$, and wait until time $$t+t^v$$.

We assume the algorithm is allowed to run forever. Obviously this is not the case in practice; we want to be able to bound the runtime of the algorithm by some number $$T$$ and know the algorithm will have come to consensus by time $$T$$ with probability at least $$1-\varepsilon$$ for some very small $$\varepsilon$$. Through deeper analysis of the model I hope to construct a way to explicitly compute the probability that a given graph will converge on a given input within time $$T$$, but for now we ignore the bound.

## Basic plan of attack

Here I outline the basic idea I have for analyzing the model.

We start by defining the **conflict** $$\eta(x)$$ of a state vector $$x\in X$$ as
$$$\begin{aligned}
\eta(x):=\vert\vert Lx\vert\vert_1=\sum_{v\in V} \vert L^v\cdot x\vert ,
\end{aligned}$$$
where $$L^v$$ is the $$v$$-th row vector of the Laplacian matrix, and $$\cdot$$ is the usual dot product.

The conflict quantifies the total amount that the values of nodes differ from the values of their neighbors. It is easily seen to have the property that $$\eta(x)=0$$ iff $$x$$ is constant on each connected component. Up to rescaling of rows, the Laplacian matrix is the unique matrix for which $$\vert\vert Lx\vert\vert_1$$ has this property.

**Theorem 1**. For any fixed $$d,\varepsilon>0$$, suppose that for every time $$t$$, either $$\eta(x_t)=0$$ or the probability $$P(\eta(x_{t+d})<\eta(x_{t}))$$ is at least $$\varepsilon$$. Then the conflict will eventually drop to zero with probability $$1$$, so each connected component is guaranteed to achieve consensus eventually.

*Proof*: Since there are only a finite number of of possible absolute states, there are only a finite number of possible values of $$\eta$$. Hence there exists an upper bound $$\Omega$$ on the values of $$\eta$$ and a lower bound $$\delta$$ on the distances between distinct values of $$\eta$$. Thus if $$\eta(x_{t+d})<\eta(x_{t})$$, then in fact $$\eta(x_{t+d})<\eta(x_{t})-\delta$$. Suppose starting at time $$t$$ if we repeatedly get lucky and decrease the conflict during every interval $$[t,t+d]$$. Then after $$k=\lceil\frac{\Omega}{\delta}\rceil$$ steps, if $$\eta(x_{t+kd})$$ were still positive we would have
$$$\begin{aligned}
\eta(x_{t+kd})<\eta(x_t)-k\delta=\eta(x_t)-\left\lceil\frac{\Omega}{\delta}\right\rceil\delta\mathop{\leqslant}\limits^{!}\eta(x_t)-\left\lceil\frac{\eta(x_t)}{\delta}\right\rceil\delta<0,
\end{aligned}$$$
where $$(!)$$ uses the fact that $$\Omega$$ is an upper bound for $$\eta$$. But this contradicts positive-definiteness of $$\eta$$, so $$\eta(x_{t+kd})=0$$. But the probability of getting a lucky round at any time $$t$$ is bounded below by $$\varepsilon>0$$ by hypothesis, so at each time $$t$$ there is a probability of at least $$\varepsilon^k>0$$ that the conflict will be zero at time $$t+kd$$. Since the quantity $$\varepsilon^k$$ is positive and independent of $$t$$, by the infinite monkey theorem eventually the conflict will drop to zero with probability $$1$$.
