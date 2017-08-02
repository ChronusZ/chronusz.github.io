---
layout: default
title: "Zero Knowledge Blind Set Membership"
date: 2017-08-02
use_math: true
---

# Zero-Knowledge Blind Set Membership

This note gives a protocol for doing *zero-knowledge blind set membership queries*, wherein party $$A$$ has a hidden set $$S$$, and party $$B$$ has a hidden number $$x$$, and the goal is for $$B$$ to learn whether or not $$x\in S$$ without learning anything else about $$S$$ in such a way that $$A$$ neither learns what $$x$$ is nor whether or not $$x\in S$$.

The protocol is basically a butchering of the ideas behind RSA. I'm about $$50\%$$ confident it works, but I'm not experienced enough with cryptography to be able to prove that it works. The general idea is that $$B$$ gives $$A$$ an encrypted "puzzle" that can be solved "under the encryption" with knowledge of the set $$S$$, but for which only $$B$$ knows how to decrypt the answer.

## Protocol

For the rest of this note, Alice will be the party holding the private set $$S$$ and Bob will be the party asking whether or not the private element $$x$$ is contained in $$S$$.

Let $$k$$ be an approximate bit-length security parameter. $$k$$ should be chosen as though it's the bit-length of an RSA scheme (i.e., the security of this protocol with security parameter $$k$$ is similar an RSA scheme with $$k$$ bits). I think Bob should be able to choose $$k$$, but the fee should increase (as a polynomial of degree $$2-3$$ is probably reasonable) with the size of $$k$$, since larger values of $$k$$ mean the prover needs to do more work as well. There should also be a minimal value of $$k$$, like $$1024$$.

Bob chooses two primes $$p,q$$ such that $$N:=p\cdot q\approx 2^k$$. Bob also chooses an additional prime $$r$$ randomly between $$2^k$$ and $$2^{2k}$$. These boundaries are not critical; the idea is that $$r$$ should be hard to discover at random but should be at least around the size of $$N$$ in order to get similar safety to RSA. Note however that the bit-size of the messages exchanged between Alice and Bob is around $$\log_2(N\cdot r)$$, so choosing $$r$$ too big can make the messages fairly big as well.

Bob computes the totient $$\lambda(N)=\mathrm{lcm}(p-1,q-1)$$. Then Bob picks random numbers $$v<n$$ and $$s<n$$, and sends $$N':=N\cdot r$$ and $$w:=v^s \mod N'$$ to Alice.

Alice picks a random number $$t<N'$$ and sends $$g:=w^t \mod N'$$ to Bob.

Bob sends to Alice $$s:=\sqrt{x} \mod \lambda(N)$$ and $$h:=g^{\lambda(N)}\cdot v \mod N'$$. (The idea is that $$h=v^{k\lambda(N)+1} \mod N'$$, but $$k$$ is large and random, neither party knows $$k$$, and Alice doesn't even know $$v$$).

Alice then computes for every $$i\leqslant \vert S\vert$$ the number $$e_i:=h^{s^{2i}} \mod N'$$. Let $$c_i$$ denote the $$i-th$$ coefficient in the characteristic polynomial $$\chi_S$$ of $$S$$ (i.e., the polynomial of degree $$\vert S\vert$$ whose roots are the elements of $$S$$). Then Alice computes

\begin{aligned}
e:=\prod_{i=0}^{\vert S\vert}e_i^{c_i}\mod N'\hspace{1em}(=h^{\chi_S(s^2)}=v^{\chi_S(x)+f(\lambda(N))}\mod N'),
\end{aligned}

where $$f$$ is some large polynomial with integer coefficients. Alice sends $$e$$ back to Bob.

Since $$x\in S$$ iff $$\chi_S(x)=0$$, if $$x\in S$$ then $$e$$ is a power of $$v^{\lambda(N)}$$; since $$\lambda(N)$$ is large, there is at most neglible chance that $$\chi_S(x)$$ is a multiple of $$\lambda(N)$$, so with at most neglible chance of failure, $$e$$ is a power of $$v^{\lambda(N)}$$ iff $$x\in S$$. By definition of Carmichael's totient, $$e=1 \mod N$$ iff $$e$$ is a power of $$v^{\lambda(N)}$$.

Thus Bob checks if $$e=1 \mod N$$. With at most neglible chance of failure, this tells Bob whether or not $$x\in S$$.
