---
layout: default
title: "Conformist Consensus"
date: 2017-07-02
use_math: true
---

# Conformist Consensus

## Introduction

Conformist consensus is another proposed modification to the Ripple consensus protocol. It consists of two main parts: a modification of the validation protocol which we refer to as **conformist validation**, and a modification of the "wrong ledger mode" protocol, which we refer to as **stubborn correction**.

Stubborn correction is a slight modification to the condition under which a node switches to the wrong ledger mode. It makes nodes stubborn to change their ledger unless they have proof that another ledger is the most popular. This ensures that if a ledger is seen as the most popular ledger by all nodes, it can never become any less popular. Because of the stubbornness it adds, stubborn correction also allows a further robustness optimization that increases the speed with which nodes come to unanimous agreement on the most recent ledger. This optimization was unavailable with the previous protocol since it would have lead to unsafe flip-flopping and the creation of "noise" due to asynchrony.

Conformist validation increases the safety of timid validation by ensuring that a node will not fully validate a ledger unless it has proof that every other node sees that ledger as the most popular ledger. This still preserves the proveable "instant safety" of timid validation by preventing two nodes from forking in the same round, but when combined with stubborn correction it further provides proveable safety against nodes forking even when separated by different rounds. By a pleasant accident of design, it also actually allows an *increase* in the rate of forward progress over both timid validation and Ripple validation in graphs which are highly connected, with no loss in safety.

## Stubborn Correction

Imagine a scenario with a fully connected graph of $$100$$ nodes. Suppose $$80$$ nodes vote for ledger $$A$$ at the beginning of validation, and $$20$$ nodes vote for ledger $$B$$. If $$v$$ is a node which hears from all the nodes which voted $$A$$, then $$v$$ will fully validate. Now suppose an asteroid hits Earth, throwing the network into chaos, and $$70$$ nodes, all of which voted $$A$$ in the previous round, lose all communication with each other. The only votes they see are $$20$$ votes for $$B$$ and $$11$$ votes for $$A$$. None of these nodes will fully validate, so there is still no problem yet. But in the next round of consensus, these $$70$$ nodes will go into wrong ledger mode and try to continue on ledger $$B$$. If communication is restored within a couple of seconds, the nodes will realize this switch was wrong and go back to $$A$$ and everything is fine. But if the communication remains down throughout the whole round, then the $$70$$ nodes will enter validation voting for ledgers which succeed $$B$$ rather than $$A$$. Now the successors of $$A$$ are in a super-minority, with only $$10$$ nodes voting for a successor to $$A$$ and $$90$$ nodes voting for a successor to $$B$$. A ledger which was in super-majority has become super-minority.

This is of course an absurdly contrived situation, but it shows that with sufficiently bad luck it is possible to fork even in the optimal set up. It would be nice if this could be made impossible with little extra effort. Stubborn correction modifies the conditions under which a node will enter wrong ledger mode in order to rule out situations like this.

Note that the only issue in the situation above was that the nodes failed to accurately learn the most popular ledger and hence switched to a worse one. Stubborn correction makes it so that nodes will not go into wrong ledger mode unless they have *proof* that doing so is actually the good thing to do.

Let $$v\in V_G$$ be a node, and for each node $$u\in UNL_v$$, let $$X(u)$$ be the ledger $$v$$ thinks $$u$$ voted for in the previous validation round. If $$v$$ has not yet heard from $$u$$ since the start of the previous validation round, then set $$X(u)=\bot$$. Then $$v$$ will enter wrong ledger mode and attempt to switch to working on the ledger $$L\neq X(v)$$ iff for every $$L'\neq L$$,
\begin{aligned}
\\#\\{u\in UNL_v:X(u)=L\\}>\\#\\{u\in UNL_v:X(u)=L'\vee X(u)=\bot\\}.
\end{aligned}

Basically, we assume that every node in our UNL which we have not heard from voted for $$L'$$ and ask if $$L$$ is still the most popular ledger. If this is true for every $$L'$$, then we have proof that $$L$$ is indeed the most popular ledger in our UNL.

This basically works, but it runs into an issue for example if $$v$$ is working on ledger $$A$$ which only has $$20\%$$ support, while there are ledgers $$B$$ and $$C$$ both with $$40\%$$ support; in this case $$v$$ would not change its vote even though it obviously is working on the wrong ledger. In this situation $$v$$ should just pick either $$B$$ or $$C$$ and run with it. This should be a deterministic choice though; in the event that several nodes are in this same situation, we would want them all to default to the same ledger rather than choosing randomly. Hence we make the following minor technical adjustment to the above rule:

Define the function $$\chi(L,L')$$ to output value $$1$$ if the ID of $$L$$ is greater than the ID of $$L'$$, and $$0$$ otherwise. Then $$v$$ will enter wrong ledger mode and attempt to switch to working on the ledger $$L\neq X(v)$$ iff for every $$L'\neq L$$,
\begin{aligned}
\\#\\{u\in UNL_v:X(u)=L\\}+\chi(L,L')>\\#\\{u\in UNL_v:X(u)=L'\vee X(u)=\bot\\}.
\end{aligned}

Thus in the event that several ledgers are all tied for popularity, the algorithm will just favor the ledger with the lowest ID.

The above concludes the safety modification that stubborn correction proposes. We could stop there having gained proveable safety, but having made the above change, there is a rather nice optimization that we can make without losing safety. This optimization helps to speed up the rate at which the network unanimously adopts the most popular ledger.

Note that because of the way nodes resist change unless they know that they are moving towards the most popular ledger, support for the most popular ledger increases monotonically. Thus, if a node pays attention not just to the ledgers its neighbors proposed for validation but in fact reacts to its neighbors switching to wrong ledger mode, this can only increase the chance of that node switching to the most popular ledger. This would have resulted in terrible behaviour under the original protocol; asychrony could have made less popular ledgers propagate throughout the network simply due to differences in communication speed. But due to the safety modification above, this issue is no longer a problem.

With this optimization in mind, the proposed protocol for entering wrong ledger mode under stubborn correction is as follows. Every timer entry during consensus, nodes run

1. For each node $$u\in UNL_v$$, check if $$u$$ went into wrong ledger mode since the end of the last timer entry. If so update the value of $$X(u)$$ accordingly.
2. Check if there is some ledger $$L\neq X(v)$$ a peer is working on such that for every $$L'\neq L$$, $$\#\{u\in UNL_v:X(u)=L\}+\chi(L,L')>\#\{u\in UNL_v:X(u)=L'\vee X(u)=\bot\}.$$ If so, enter wrong ledger mode and switch to $$L$$, and broadcast that we have done so. Otherwise do nothing until the next timer entry.


## Conformist Validation

Conformist validation is actually very similar to timid validation; the main change is just to replace "$$80\%$$ support" with "most popular" everywhere. One could easily add an "optimistic waiting" optimization layer on top of conformist validation (it would work just like adding optimistic waiting to timid validation), but we avoid doing so here to maximize clarity. Unfortunately, performance ostracization leads to issues with future-safety, as we highlight in the analysis.

Let $$v\in V_G$$ be any node. Let $$C(v)\subseteq V_G$$ be the set of nodes $$v$$ wishes to avoid forking with ("C" for "cares about"...?). We assume that $$v$$ has knowledge of the UNL of every node in $$C(v)$$. Note that we no longer speak of ostracized nodes in this setting. Using $$C(v)$$ instead of $$V_G\setminus F(v)$$ is simply a linguistic choice which hopefully will cause less cognitive dissonance than referring to a "global topology" which is not "globally" agreed upon.

To clarify, $$C(v)$$ has no impact on Byzantine safety. It *only* says which nodes $$v$$ checks fork-safety with. Byzantine safety is entirely determined by UNLs: a Byzantine node which is not in your UNL can't make you fork with another node, and a Byzantine node which is in your UNL will affect your fork-safety the same regardless of whether you "care about" it or not. To avoid thinking the protocol is safe in situations where it isn't, it is best to always require that $$UNL_v\subseteq C(v)$$ (but there's of course no need to require that $$UNL_v = C(v)$$).

For robustness, we also add a parameter allowing some fraction of anyone's UNL to be Byzantine and still get proveable safety. Thus nodes which are not known to be Byzantine still won't affect the protocol. This allowance comes at the cost of forward progress: the more nodes one wants to allow to be Byzantine, the harder it is to fully validate a ledger.

To keep things as abstract as possible, let $$f:\mathbb{N}\to\mathbb{N}_0$$ be an arbitrary monotone-increasing function. The values of $$f$$ are interpreted as "in a UNL of size $$n\in\mathbb{N}$$, we allow at most $$f(n)$$ arbitrarily chosen nodes in the UNL to be Byzantine". In general $$f$$ will probably only ever be (the floor of) a linear function with a slope between $$0$$ and $$0.5$$, but it may be helpful for pedagogy to be able to recognize immediately where $$f$$ shows up.

For each $$u\in UNL_v$$, let $$X(u)$$ denote the ledger $$u$$ is validating, or $$\bot$$ if this is unknown. $$v$$ runs the following algorithm to determine whether or not it should validate:

1. Check if either there exists some ledger $$L$$ such that for every $$L'\neq L$$, $$\vert S\vert+\chi(L,L')>\#\{u\in UNL_v:X(u)=L'\vee X(u)=\bot\}+2f(\vert UNL_v\vert).$$ If so, store the ledger $$L$$ and the set $$S=\#\{u\in UNL_v:X(u)=L\}$$ as private variables, and proceed to step $$2$$. If we hear from all of our neighbors and there is no ledger satisfying the above condition, or enough time passes and still the above condition is not satisfied, reject validation and terminate the algorithm.
2. For each node $$u\in C(v)$$, mark $$u$$ as **safe** iff for every ledger $$L'\neq L$$, $$\vert UNL_u\cap S\vert+\chi(L,L')>\vert UNL_u\setminus UNL_v\vert + \#\{w\in UNL_v\cap UNL_u : X(w)=L'\vee X(w)=\bot\}+2f(\vert UNL_u\vert)$$.
3. If every node in $$C(v)$$ has been marked safe, fully validate the ledger $$L$$. Otherwise reject validation (or do optimistic waiting if so desired).

#### Analysis

For expository reasons, assume temporarily that $$f(n)=0$$ for all $$n\in\mathbb{N}$$. Further below we will discuss the general case.

Some explanation may be necessary to explain why conformist validation can change "$$80\%$$ support" to "most popular" in step $$1$$ and not run into issues with safety.

The safety of timid validation comes from the fact that a node will mark another node as dangerous if it has any chance of seeing a super-majority for a different ledger. But this is exactly the condition under which a node moves past step $$1$$ of the timid protocol; hence this could be rephrased as a node is perceived as dangerous if it has any chance of moving past step $$1$$ with a contradictory ledger. Abstracted away to this form, the similarity with conformist validation becomes more clear. In conformist validation, a node is perceived as dangerous if it has any chance of seeing a contradictory ledger as most popular, and it only moves past step $$1$$ if it knows a given ledger is the most popular. Thus in a large sense timid validation and conformist validation are effectively the same protocol; however, replacing "$$80\%$$ support" with "most popular" behaves much better for future safety, because it ensures that a node will not validate a minority ledger simply due to having a bad UNL.

We say a graph $$G$$ **satisfies conformity** if for every pair of nodes $$u,v\in V_G$$,
\begin{aligned}
\vert UNL_u\cap UNL_v \vert > \max\\{\lfloor 0.2\vert UNL_v\vert\rfloor + \lfloor 0.5\vert UNL_u\vert\rfloor,\lfloor 0.5\vert UNL_v\vert\rfloor + \lfloor 0.2\vert UNL_u\vert\rfloor\\}.
\end{aligned}

Satisfying conformity is exactly the necessary-and-sufficient condition on a graph which ensures that even in a worst-case scenario, whenever a node validates a ledger $$L$$ under Ripple validation, every other node is guaranteed to see $$L$$ as the most popular ledger. It is easily shown that in a graph which satisfies conformity, for any node which sees super-majority support for a ledger $$L$$, step $$2$$ will deem every other node safe automatically. Further, as soon as that node has heard from at most $$71\%$$ of its neighbors in this situation, it will pass step $$1$$. Thus that node will fully validate $$L$$ under conformist validation as well (and to do so requires less communication than Ripple validation as well!). Thus in graphs which satisfy conformity, every node will validate under conformist validation at least as often as they would have under Ripple validation. Note however that conformist validation can actually make nodes validate **more** often in graphs with very high connectivity. For instance, in a fully connected graph on five nodes where the nodes vote AABBB, any node which hears from all of the nodes voting $$B$$ before giving up in step $$1$$ will fully validate $$B$$; the nodes which voted $$B$$ themselves will fully validate as soon as they hear from *any* $$3$$ of the other nodes! Thus not only is conformist validation an excellent safety guarantee, it also helps increase the rate of forward progression in "normal" network setups.

On the other end of the spectrum, it is easily deduced that a node $$v$$ will halt (i.e., never fully validate even if it sees unanimous support for a ledger in its UNL) iff there is another node $$u\in C(v)$$ such that
\begin{aligned}
\vert UNL_u\cap UNL_v \vert \leqslant 0.5(\vert UNL_u\vert-1).
\end{aligned}

(The $$1$$ comes from the whole $$\chi(L,L')$$ business and should basically be ignored). Thus just like timid validation, there is about a $$20\%$$ margin on the overlap of UNLs between behaving the same as (or better than, in the case of conformist validation) Ripple validation and always halting.

It turns out that implementing a naive performance ostracization scheme based on UNL size is unsafe for conformist validation. The reason for this is a bit subtle. Technically speaking, it is not so much problematic for conformist validation as it is a problem for future safety. In fact, it is at least as much a problem for the future safety of timid validation as it is for conformist validation; the only difference is that timid validation never attempts to address future safety in the first place, so one can't really call this a "problem" for timid validation.

Performance ostracization works well for timid validation because timid validation only cares about preventing other nodes from validating conflicting ledgers in the same round. By inspection it is easily seen that if a node marks another node as dangerous that node will also mark the first as dangerous, assuming both are nonfaulty. Thus one of them can ostracize the other to make forward progress easier, and there is no issue for safety as long as performance ostracization is symmetric. It may seem as though this analysis can be easily adapted to conformist validation as well, since it is still true that if one node marks another as dangerous that node will also mark the first as dangerous. However, in this situation it is not fine for future safety for one node to fully validate knowing the other won't due to an asymmetric property of performance ostracization. For safety against immediate forks, it is perfectly acceptable for one node to simply chug ahead, secure in the knowledge that everyone else won't validate if they detect danger. But this can't ensure that the node chugging ahead will always be validating the most popular ledger. Thus every node needs to check the information of every other node to make sure that it knows it is sufficiently in the majority before proceeding.

Again though, the issue is subtle; suppose $$v$$ is a node in a reasonably well-connected network, and suppose there is a node $$u$$ which doesn't broadcast to any other nodes $$v$$ cares about. Then if $$v$$ doesn't care about forking with $$u$$, $$u$$ cannot cause any issues for forking with nodes $$v$$ does care about. Further, as long as a majority of the nodes in the UNL of $$u$$ are nodes which $$v$$ cares about, then $$v$$ can ignore $$u$$ and have no risk of forking even with $$u$$ itself.

I have not thought extensively about exactly under what other conditions it might be safe to performance ostracize a node while still safeguarding against future forks, but I imagine they must be pretty strict to retain proveable safety. Thankfully, in a network of validators which is generally suspicious of newcomers, the above condition is probably sufficient to prevent halting due to newcomers with low connectivity; simply don't add new nodes to your UNL unless they have sufficient connectivity to prevent halting. If a node $$w$$ in your network adds them too early when they still have low connectivity, $$w$$ is probably malicious so you can ostracize it.

###### Allowing Byzantine Nodes in the UNL

We now discuss the general case of arbitrary $$f$$.

We say a graph $$G$$ **satisfies conformity with f** if for every pair of nodes $$u,v\in V_G$$,
\begin{aligned}
\vert UNL_u\cap UNL_v \vert > \max\\{\lfloor 0.2\vert UNL_v\vert\rfloor + \lfloor 0.5\vert UNL_u\vert\rfloor+f(\vert UNL_u\vert),\lfloor 0.5\vert UNL_v\vert\rfloor + \lfloor 0.2\vert UNL_u\vert\rfloor+f(\vert UNL_v\vert)\\}.
\end{aligned}

This is the condition on which conformist validation will always fully validate at least as easily as Ripple validation does. Note however that just as above, for reasonable $$f$$ functions, on graphs which satisfy conformity with $$f$$ conformist validation will usually have strictly better forward progress than Ripple validation.

To give an idea of the forward progress of conformist validation with faults, take $$f(n)=\lfloor (n-1)/5\rfloor$$. This is the same fault tolerance proposed for Ripple validation in the whitepaper. In this case for a graph to satisfy conformity with $$f$$, there needs to be a roughly $$90\%$$ overlap on UNLs. In complete graphs, step $$2$$ is completely irrelevant; thus in this case a node will fully validate as soon as it sees $$71\%$$ support for any ledger, even if the other $$29\%$$ have crashed. This quorum becomes even better in the event that the competing ledgers are all contentious; if a node sees e.g. $$61\%$$ support for ledger $$A$$ and $$5\%$$ support each for ledgers $$B,C,D$$, then that node will also fully validate regardless of the state of the other $$24\%$$. Thus in this case the protocol still has quite good forward progress.

Forward progress becomes worse as $$f$$ gets bigger. In the extreme case, when $$f(n)>\lfloor n/2\rfloor$$ nodes will always halt. When $$f(n)>\lfloor n/3\rfloor$$, nodes will not halt, but forward progress will always be worse than Ripple validation even when the graph is complete.

## Conclusion

Conformist consensus further refines the modification train which began with timid validation, attempting to prevent all possibility of forking in exchange for possibly making nodes halt in bad topologies. Conformist consensus requires nothing beyond the requirements of timid validation, but it provides proveable safety against any chance of a fork ever occuring, even split between rounds. Further, in networks with high connectivity it actually makes validating ledgers *easier* than either timid validation or Ripple validation, with no risk to safety.

On a more theoretical level, stubborn correction gives one of the first proveable properties on the output of the establish phase of consenus: if every node in a graph sees ledger $$L$$ as the most popular ledger in round $$n$$, then every node in the graph will see (a successor of) $$L$$ as the most popular ledger in round $$n+1$$. It would be nice to have more proveable properties about the output of establishing, so that one does not have to fear the possibility of arbitrary behavior when making design choices.
