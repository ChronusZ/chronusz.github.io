---
layout: default
title: "Comparing the Asymptotic Behavior of ALC and SLC"
date: 2017-06-15
use_math: true
---

# Comparing Asynchronous and Synchronous Linear Consensus

## Introduction

Brad considered in his note a linear approximation of consensus that operated by iteratively updating the value of each node by averaging the values of its neighbors from the previous step. We call this version of consensus **synchronous linear consensus (SLC)**, since the nodes are updating synchronously and the update function is a linear operator on the state vectors.

We contrast SLC with what we call **asynchronous linear consensus (ALC)**; instead of all nodes simultaneously updating their values by averaging with their neighbors, in the ALC model at each step a node is chosen at random according to a probability distribution $$P$$ on $$V$$, and the chosen node updates its value by averaging with its neighbors&#39; values.

It turns out that in the ALC model, despite the lack of determinism, the update function is in fact a linear operator on the *expectation values* of the state vectors, justifying our use of the word linear in its name.

In this note we will compare the asymptotic behavior of SLC to that of ALC and show that both converge to the same stable state. This gives more justification for the use of SLC as a useful approximation of Ripple consensus.

## Notation

Let $$G=(V,E)$$ be a directed graph with adjacency matrix $$A$$. For $$v\in V$$, let $$k^v$$ be the set of neighbors of $$v$$, i.e., $$N^v=\\{u\in V\vert(u\to v)\in E\\}$$. For simplicity we assume that each node in $$V$$ is indexed by a natural number, and we refer to the node and its index interchangeably. We assume that every node is its own neighbor. Let $$D$$ be the diagonal in-degree matrix. Define the averaging operator $$T$$ by $$T=D^{-1}A$$.

Throughout the paper we will use a matrix norm defined as follows: Let $$M$$ be an $$m\times n$$ matrix. Then $$\vert M\vert$$ denotes the $$\ell_\infty$$ entry-wise norm of $$M$$, i.e., $$\vert M\vert=\max\\{\vert M^{ij}\vert:1\leqslant i \leqslant m,1\leqslant j\leqslant n\\}$$. This norm behaves very similarly to the regular absolute value on real numbers, but care should be taken since it doesn't respect products, i.e., $$\vert M\vert\cdot\vert N\vert\neq\vert M\cdot N\vert$$ in general.

## Synchronous Linear Consensus

In the SLC model, we begin with a **state vector** $$x_0$$ indexed by $$V$$, which has either a $$1$$ or $$0$$ in the $$v$$ coordinate according to whether or not $$v$$ agrees with the transaction being considered. We use superscripts to refer to the components of a vector, e.g., $$x_n^v$$ refers to the $$v$$-th component of the vector $$x_n$$. Since our vectors are all indexed by nodes, there shouldn&#39;t be any confusion between component indices and exponents.

SLC operates at each step by applying $$T$$ to the previous state vector $$x_{n-1}$$ to get a new state vector $$x_n$$. Thus the state after the $$n$$-th step is $$x_n=Tx_{n-1}=T^nx_0$$.

## Technical Interlude on Convergence

We now make a quick interlude to discuss the technical convergence of SLC. A graph is called **aperiodic** if the greatest common denominator of the lengths of all its directed cycles is one. Note that since we make the assumption that every node in $$G$$ has a loop, $$G$$ will always be aperiodic (since loops are exactly cycles of length $$1$$). Consider a random walk on $$G$$, where at each step one picks a random neighbor with uniform probability and moves to it. This defines a Markov chain whose stochastic matrix is $$T$$. As shown in Costello[^fn1], the random walk on a strongly connected aperiodic graph always converges. In other words, whenever $$G$$ is strongly connected, $$T^{\star}:=\lim\limits_{n\to \infty} T^n$$ is defined. We will henceforth assume that $$G$$ is always strongly connected and thus that $$T^{\star}$$ is always defined. We call $$T^{\star}$$ the **asymptotic distribution**. It is an operator which when applied to a state vector gives the vector that the state eventually converges to.

(Intuitively, the aperiodicity condition prevents a configuration where everything just cycles around in a predictable way forever, like in a directed cycle graph.)

The familiar definition of convergence of an infinite sequence says that $$T^n$$ **converges** to $$T^{\star}$$ iff for all $\varepsilon>0$ there exists some $$N\in\mathbb{N}$$ such that for all $$n>N$$, $$\vert T^n-T^{\star}\vert$$ (here we're using the matrix norm defined in the notation section). There is another unfortunately convoluted definition that we will find useful near the end of this note: we say that $$T^n$$ **binomially converges** to $$T^{\star}$$ iff for all $\varepsilon>0$ there exists some $$N\in\mathbb{N}$$ such that for any $$0<p<1$$, $$\lim\limits_{n\to\infty}\sum_{i=N+1}^n\binom{n}{i}(1-p)^{n-i}p^i\vert T^i-T^{\star}\vert<\varepsilon$$. If $$T^n$$ converges to $$T^{\star}$$, then using the $$N$$ given by definition of convergence, we have
\begin{aligned}
\lim\limits_{n\to\infty}\sum_{i=N+1}^n\binom{n}{i}(1-p)^{n-i}p^i\vert T^i-T^{\star}\vert&<\lim\limits_{n\to\infty}\sum_{i=N+1}^n\binom{n}{i}(1-p)^{n-i}p^i\varepsilon\\\
&=\varepsilon\lim\limits_{n\to\infty}\sum_{i=N+1}^n\binom{n}{i}(1-p)^{n-i}p^i\\\
&\leqslant\varepsilon,
\end{aligned}

where the last inequality follows from the fact that the binomial distribution is a probability distribution, i.e., $$\sum_{i=1}^n\binom{n}{i}(1-p)^{n-i}p^i=1$$ for all $$n\in\mathbb{N}$$ and $$0<p<1$$. Thus $$T^n$$ binomially converges whenever it converges. I think binomial convergence is in fact equivalent to regular convergence, but I&#39;m not certain and it doesn&#39;t matter for this note anyway since nothing described in this paper makes sense unless $$T^n$$ is assumed to converge regularly.

## Asynchronous Linear Consensus

In the ALC model, just like in SLC we begin with a state vector $$x_0$$. Let $$P$$ be a probability distribution on the vertices of $$V$$, i.e., a function $$P:V\to [0,1]$$ such that $$\sum_{v\in V} P(v)=1$$. We must assume that $$P(v)\neq 0$$ for every $$v\in V$$, but we make no assumption on the uniformity of $$P$$.

At each step of ALC, we choose an **activated** node $$v$$ randomly according to the distribution $$P$$. The new state vector is then defined to be equal to the old state vector in every index other than $$v$$, and the new value in the $$v$$ coordinate is the average across the old values of all of its neighbors. Since ALC chooses the nodes randomly, the state vectors are not determined. Instead we define the $$n$$-th **state vector** as a vector-valued random variable $$X_n$$ whose $$v$$-th component is the value of $$v$$ after the $$n$$-th step of ALC. Thus if $$v$$ was the node activated at step $$n$$, we define $$X_n^u=X_{n-1}^u$$ for $$u\neq v$$ and $$X_n^v=\frac{1}{\vert N^v\vert}\sum_{u\in N^v}X_{n-1}^u$$.

Since the $$X_n$$ are random, it doesn&#39;t make sense to talk about how they converge as $$n$$ goes to infinity; however, we can consider how their expected values $$E[X_n]$$ converge. This is the avenue we will explore in discussing the asymptotic behavior of ALC. Concretely, we define the **expected state vector** $$x_n$$ as the vector whose $$v$$-th component is $$E[X_n^v]$$. Even though the $$X_n$$ are non-deterministic, the $$x_n$$ are deterministic and so we can consider how they converge.

## The Expectation Transition Function of ALC

We can compute the expected state vectors recursively as follows. For a given node $$v\in V$$, there is a $$P(v)$$ chance that $$v$$ activates, and a $$1-P(v)$$ chance that $$v$$ will stay the same. By linearity of expectation, we thus have
\begin{aligned}
x_n^v&=E[X_n^v]\\\
&=E\left[(1-P(v))X_{n-1}^v+P(v)\left(\frac{1}{\vert N^v\vert}\sum_{u\in N^v} X_{n-1}^u\right)\right]\\\
&=(1-P(v))E[X_{n-1}^v]+P(v)\left(\frac{1}{\vert N^v\vert}\sum_{u\in N^v} E[X_{n-1}^u]\right)\\\
&=(1-P(v))x_{n-1}^v+P(v)\left(\frac{1}{\vert N^v\vert}\sum_{u\in N^v} x_{n-1}^u\right)\phantom{----} (eq.1)
\end{aligned}

This recursion formula defines a linear transition operator $$\tilde{T}$$ on the expected state vectors. This explains our usage of the word linear in the term ALC: even though ALC can behave in each instance nonlinearly, the expected evolution is a linear difference equation.

By a convenient abuse of notation, we consider $$P$$ as a diagonal matrix with $$P^{vv}=P(v)$$ and let $$1$$ denote the identity matrix. We compute from the recursive formula
\begin{aligned}
\tilde{T}&=(1-P)+PT.
\end{aligned}

We call $$\tilde{T}^{\star}:=\lim\limits_{n\to\infty}\tilde{T}^n$$ the **expected asymptotic distribution**. It is an operator which when applied to a state vector gives the vector that the state is expected to eventually converge to on average. We can now explicitly state what we hope to show: $$\tilde{T}^{\star}=T^{\star}$$. If we can show this, this says exactly that running ALC for long enough on any input will on average give the same output as SLC would.

## Outline of Attack

Since the formal argument is rather long and convoluted, in this section I briefly outline the way we go about proving that $$\tilde{T}^{\star}=T^{\star}$$.

We will first break up $$\tilde{T}^n$$ as an indexed sum in terms of powers of $$T$$ up to $$n$$. In this sum the terms are weighted by a binomial distribution, so since the powers of $$T$$ binomially converge to $$T^{\star}$$, there is a finite number $N$ such that, ignoring the first $$N$$ terms, we can replace all the powers of $$T$$ in the sum with $$T^{\star}$$ and in doing so only change the asymptotic value of the sum by an arbitrarily small amount.

Thus the question is reduced to showing that as $$n$$ increases, the contribution of small powers to the sum becomes negligible. But this basically just follows from the shape of the binomial distribution: the binomial distribution puts the most weight on the middle of the curve and shallows out the edges. Thus if we ignore the terms with small powers, the asymptotic distribution doesn&#39;t change, so by the previous paragraph we can just replace all the powers of $$T$$ in the sum by $$T^{\star}$$, and asymptotically this sum will converge to the same distribution as $$\tilde{T}^n$$. But then the sum itself is immediately seen to converge to $$T^{\star}$$ since the weights of the binomial distribution add up to $$1$$, and thus we&#39;re done.

## Computations

By the binomial theorem, we have
\begin{aligned}
\tilde{T}^n&=((1-P)+PT)^n\\\
&=\sum_{i=1}^{n}\binom{n}{i}(1-P)^{n-i}(PT)^i
\end{aligned}

This is the indexed sum we referred to in the previous section. Since this is defined in terms of powers of $$T$$, it gives a convenient setting for comparing $$\tilde{T}^{\star}$$ and $$T^{\star}$$. Note that $$\binom{n}{i}(1-P)^{n-i}P^i$$ is coordinate-wise just a binomial distribution.

Now let $$k\in\mathbb{N}$$ be an arbitrary natural number and consider the $$k$$-th partial sum of $$(eq.1)$$,
\begin{aligned}
\sum_{i=1}^{k}\binom{n}{i}(1-P)^{n-i}(PT)^i
\end{aligned}

as a function of $$n$$. We want to show that for constant $$k$$, this partial sum goes to zero as $$n$$ increases. To this end, let $$c_k$$ be defined as
\begin{aligned}
c_n=\max\\{\left\vert\frac{P^i}{i!(1-P)^i}T^i\right\vert:1\leqslant i\leqslant k\\}.
\end{aligned}

Further let $$p_k(n)$$ be defined as
\begin{aligned}
p_k(n)=\sum_{i=1}^{k}\frac{n!}{(n-i)!}
\end{aligned}

Thus,
\begin{aligned}
\left\vert\sum_{i=1}^{k}\binom{n}{i}(1-P)^{n-i}P^iT^i\right\vert&\leqslant\sum_{i=1}^{k}\left\vert\binom{n}{i}(1-P)^{n-i}P^iT^i\right\vert\\\
&\leqslant\sum_{i=1}^{k}c_k\frac{n!}{(n-i)!}(1-P)^n\\\
&=kc_kp_k(n)(1-P)^n.
\end{aligned}

But $$kc_k$$ is constant with $$n$$ and $$p_k$$ is a polynomial of degree $$k$$ in $$n$$. Since we assumed that $$P(v)>0$$ for all $$v$$, we have $$0<1-P(v)<1$$, so $$(1-P)^n$$ is a decreasing exponential in $$n$$. Since non-constant exponentials are always asymptotically dominant over polynomials, $$kc_kp_k(n)(1-P)^n$$ tends asymptotically to $$0$$, so we have that
\begin{aligned}
\lim\limits_{n\to\infty}\left\vert\sum_{i=1}^{k}\binom{n}{i}(1-P)^{n-i}P^iT^i\right\vert=0.
\end{aligned}

But by definition of the norm $$\vert\cdot\vert$$, this says exactly that the partial sum converges to the zero matrix. Thus, for any $$k\in\mathbb{N}$$ we can cut out the first $$k$$ terms of the sum in $$(eq.1)$$ without changing the asymptotic behavior of the sum, i.e.,
\begin{aligned}
\tilde{T}^{\star}=\lim\limits_{n\to\infty}\sum_{i=k+1}^{n}\binom{n}{i}(1-P)^{n-i}P^iT^i
\end{aligned}

Note that by replacing $$T$$ with the identity matrix and noting that $$\tilde{1}=1-P+P1=1$$, the same argument also gives the formula
\begin{aligned}
1=\lim\limits_{n\to\infty}\sum_{i=k+1}^{n}\binom{n}{i}(1-P)^{n-i}P^i\phantom{----} (eq.2)
\end{aligned}

which we will need later as well.

Given any $$\varepsilon>0$$, we can now show that $$\vert T^{\star}-\tilde{T}^{\star}\vert<\varepsilon$$, from which it immediately follows that $$\vert T^{\star}-\tilde{T}^{\star}\vert=0$$ which implies $$T^{\star}=\tilde{T}^{\star}$$. Since $$T^{\star}=\lim\limits_{n\to\infty}T^n$$, by definition of limits, for any $$\varepsilon>0$$ there exists $$N\in\mathbb{N}$$ such that for any $$n>N$$, $$\vert T^{\star}-T^n\vert<\varepsilon$$.

Here is where we need binomial convergence of $$T^n$$. Let $$N$$ be large enough such that $$\lim\limits_{n\to\infty}\sum_{i=N+1}^{n}\binom{n}{i}(1-P)^{n-i}P^i\left\vert T^i-T^{\star}\right\vert<\varepsilon$$. Then
\begin{aligned}
\vert\tilde{T}^{\star}-T^{\star}\vert&=\lim\limits_{n\to\infty}\left\vert\left(\sum_{i=N+1}^{n}\binom{n}{i}(1-P)^{n-i}P^iT^i\right)-T^{\star}\right\vert\\\
&\mathop{=}\limits^{(eq.2)}\lim\limits_{n\to\infty}\left\vert\sum_{i=N+1}^{n}\binom{n}{i}(1-P)^{n-i}P^i\left(T^i-T^{\star}\right)\right\vert\\\
&\leqslant\lim\limits_{n\to\infty}\sum_{i=N+1}^{n}\binom{n}{i}(1-P)^{n-i}P^i\left\vert T^i-T^{\star}\right\vert\\\
&<\varepsilon.
\end{aligned}

Since $$\varepsilon$$ was chosen arbitrarily, we have that the absolute difference between $$\tilde{T}^{\star}$$ and $$T^{\star}$$ is smaller than every positive real number, so they must in fact be the same matrix. In other words, the asymptotic behavior of SLC is identical to the expected asymptotic behavior of ALC.

Note that $$(!)$$ in the above equation follows from the fact that $$\binom{n}{i}(1-P)^{n-i}P^i$$ is coordinate-wise a probability distribution (indeed, it is the binomial distribution), and hence is at most $$1$$ for every value of $$i$$.

# Conclusion

ALC is still only an abstract approximation of consensus, but it seems a priori to more closely mimic the behavior of Ripple consensus than SLC does. However, fo making far-reaching mathematical statements, SLC is much more useful in practice, being a deterministic linear difference equation. Having showed that ALC converges on average to the same distribution as SLC, this gives reasonable evidence for the validity of arguments based on SLC.

A question that we haven&#39;t touched upon at all which still deserves exploration is a comparison of the rate of convergence of ALC compared to SLC. Also, I ran a few simulations beforehand comparing ALC to SLC, and found that the standard deviation of the stable-state of ALC can be quite high; in the undirected triangle graph with initial state vector $$(1,1,0)$$ for example, we got a coordinate-wise standard deviation of around $$0.24$$. A few tests on higher-order complete graphs seemed to signify that the standard deviation decreases with the number of nodes (or perhaps degree) and with how many of the initial states agree. However, even in a complete graph with $$10$$ nodes and an initial state vector of $$(1,1,1,1,1,1,1,1,1,0)$$ the standard deviation was above $$0.09$$, which seems high to me. I don't know how important this actually is, but it may be worth looking into further.

# Bibliography

[^fn1]: Costello, K. $$2005$$, "Random Walks on Directed Graphs".
