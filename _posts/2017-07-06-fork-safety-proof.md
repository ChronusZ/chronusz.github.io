---
layout: default
title: "Fork Safety Proof"
date: 2017-07-06
use_math: true
---

# Fork-Safety Condition on Ripple Validation

This note formally proves the necessary and sufficient condition on (immediate) fork-safety for Ripple Validation.

**Proposition.** Let $$G=(V,E)$$ be a graph. Then Ripple Validation is safe against (immediate) forks if and only if, for every pair of nodes $$u,v\in V$$,
$$$\begin{aligned}
\vert UNL_u\cap UNL_v\vert > \lfloor 0.2\vert UNL_u\vert \rfloor + \lfloor 0.2\vert UNL_v\vert \rfloor.
\end{aligned}$$$

*Proof*. First assume for showing necessesity that
\begin{aligned}
\vert UNL_u\cap UNL_v\vert \leqslant \lfloor 0.2\vert UNL_u\vert \rfloor + \lfloor 0.2\vert UNL_v\vert \rfloor.
\end{aligned}

Then there exist integers $$0\leqslant i\leqslant \lfloor 0.2\vert UNL_u\vert \rfloor$$ and $$0\leqslant j\leqslant \lfloor 0.2\vert UNL_v\vert \rfloor$$, where $$\vert UNL_u\cap UNL_v\vert=i+j$$. By definition of Ripple Validation, $$u$$ will fully validate a ledger $$A$$ iff at most $$\lfloor 0.2\vert UNL_u\vert \rfloor$$ nodes in $$UNL_u$$ vote against $$A$$. Thus in particular, even if $$i$$ nodes in $$UNL_u$$ vote for a different ledger, $$u$$ can still fully validate $$A$$. Similarly, $$v$$ will fully validate a ledger $$B$$ even if $$j$$ nodes in $$UNL_v$$ vote for a different ledger.

Partition $$UNL_u\cap UNL_v$$ into two arbitrary subsets $$S_A$$ and $$S_B$$, such that $$\vert S_A\vert=j$$ and $$\vert S_B\vert=i$$. Then suppose $$UNL_u\setminus S_B$$ all vote for ledger $$A$$ and $$UNL_v\setminus S_A$$ all vote for ledger $$B$$. Then $$u$$ will see a super-majority support for $$A$$ and fully validate $$A$$ while $$v$$ sees a super-majority support for $$B$$ and fully validate $$B$$, leading to a fork.

Sufficiency follows from a pigeonhole argument. Suppose $$u\in V$$ sees enough support to fully validate the ledger $$A$$, and suppose for every $$v\in V$$
$$$\begin{aligned}
\vert UNL_u\cap UNL_v\vert > \lfloor 0.2\vert UNL_u\vert \rfloor + \lfloor 0.2\vert UNL_v\vert \rfloor.
\end{aligned}$$$

It suffices to show that $$v$$ cannot see super-majority support for a different ledger than $$A$$. Let $$i$$ be the number of nodes in $$UNL_u$$ that don't vote $$A$$. Since $$u$$ sees super-majority support for $$A$$, $$i\leqslant\lfloor 0.2\vert UNL_u\vert \rfloor$$. Thus if $$S\subseteq V$$ is the set of all nodes which vote for $$A$$, then
\begin{aligned}
\vert UNL_v\cap S\vert &\geqslant \vert UNL_u\cap UNL_v\cap S\vert\\\
&= \vert UNL_u\cap UNL_v\vert - \vert (UNL_u\cap UNL_v)\setminus S\vert\\\
&> (\lfloor 0.2\vert UNL_u\vert \rfloor + \lfloor 0.2\vert UNL_v\vert \rfloor) - i\\\
&= \lfloor 0.2\vert UNL_v\vert \rfloor + (\lfloor 0.2\vert UNL_u\vert \rfloor - i)\\\
&\geqslant \lfloor 0.2\vert UNL_v\vert \rfloor.
\end{aligned}

Thus $$v$$ cannot see a super-majority for any ledger other than $$A$$. ⬜
