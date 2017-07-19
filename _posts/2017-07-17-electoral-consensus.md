---
layout: default
title: "Electoral Consensus"
date: 2017-07-17
use_math: true
---

## Introduction

This note proposes an overhaul of Ripple consensus that achieves safety with (what I think is) the theoretical minimal connectivity requirements. The protocol is heavily designed after HoneyBadgerBFT, but it modifies a few key parts to make it work in incomplete networks with unknown participants. We call this protocol Electoral Consensus.

Before describing the protocol (which is unfortunately much more complicated than timid validation or conformist validation), we describe some of its characteristics. Loosely speaking, electoral consensus operates each round by electing a commonly agreed upon group of nodes called the **council** in such a way that every node is able to hear a proposal from every node in the council, and every node hears the same proposals, even if an elected councilor is Byzantine. After the council is elected for the round, then each node knows that they see the same information as every other node, and a deterministic algorithm can be employed to then fuse that information to construct the fully-validated ledger. Not every node can be chosen to be a council member; one needs loosely speaking to have reasonably good trust within the network, i.e., a fair number of nodes have you in their UNL. We refer to nodes which are capable of being chosen as councilors by calling them "candidates". Thus this algorithm explicitly segregates validators into reasonably-trusted candidates and not-as-well-trusted electors. The way I imagine it, for optimal decentralization one wants as many candidates as possible without making too many of the candidates potentially faulty or malicious (malicious candidates cannot compromise fork-safety, but adding too many of them could give them excessive influence over the transactions validated), so nodes that are honest electors will eventually gain enough trust in the network to become candidates. This gives a more concrete realization of the "layered trust" which is implicit in the current Ripple consensus algorithm as well (untrusted nodes don't have any real influence over consensus under either protocol).

#### Pros

Currently, even with stubborn history, in the presence of $$t_i$$ Byzantine faulty nodes in one's UNL, in order for $$\mathcal{P}_i$$ and $$\mathcal{P}_j$$ to guarantee future-fork safety, they need a UNL overlap of at least $$\min\{t_i,t_j\}+\max\{t_i+\lceil 0.5\vert \mathsf{UNL}_j\vert\rceil,t_j+\lceil 0.5\vert \mathsf{UNL}_i\vert\rceil\}$$. In other words, with a fault tolerance of $$20\%$$, nodes need around $$90\%$$ overlap of UNLs. When the overlap required is this large, one is effecitvely restricted to the current set up of a central completely connected graph with random "leaf validators" listening only to themselves and the central part, at which point the whole concept of being a public blockchain seems to lose most of its meaning. Electoral consensus is designed to minimize the overlap needed to guarantee fork-safety, without resorting to theoretical tools which are dangerous in practice, like forced "message passing" schemes which require nodes to pass along any message they receive, opening up honest nodes to DDoSing.

Each node $$\mathcal{P}_i$$ has a personally designated fault tolerance $$t_i$$, which is the maximal number of faulty nodes allowed in $$UNL_i$$. Electoral consensus gives complete fork safety between two nodes $$\mathcal{P}_i,\mathcal{P}_j$$ whenever their UNL overlap is at least $$t_i+t_j+min\{t_i,t_j\}+1$$. This is the same condition under which Ripple validation guarantees safety against *immediate* forks, but in the case of electoral consensus, this gives safety against *all* forks. In addition, electoral consensus "guarantees" termination even in the midst of total asynchrony, so that every nonfaulty node fully validates a ledger every round; the quotation marks will be explained later because "guarantees termination" is sort of a lie (for several reasons), but it's true in essence.

#### Cons

Despite the excellent safety properties of electoral consensus, it is important to emphasize its drawbacks. The main drawback is performance. Electoral consensus can be run on its own from the start as a complete replacement for Ripple consensus; however, in order to make this work it is probably not sufficient for the councilors to simply broadcast the hash of their transaction set, since it might not be possible to recover the original set if no other nodes proposed an identical set (since e.g. the councilor might crash after broadcasting). Thus one would probably need to propose the entire transaction set; I came up with a way of doing a compression operation based on erasure coding in directed graphs (not described in this note; if you're curious I'd be happy to explain later) that can be used to cut down message size (in such a way that increased overlap allows higher compression), but this might still be too big to get any significant throughput.

Alternatively, electoral consensus can be built on top of Ripple consensus as a validation system. In this case, we already expect the transaction sets to be mostly the same, so it should be unproblematic to just send the hash. The issue with doing things this though is that electoral consensus is significantly slower than Ripple validation or even timid/conformist validation. This will likely add at least several seconds to the time it takes for a ledger to be fully validated. This added latency occurs even when the network is a complete graph. Thus either way, the performance of electoral consensus is relatively poor compared to the current protocols. Note though that just as with the current validation algorithm, there is no need to necessarily wait for electoral consensus to finish before starting work on the next ledger.

In a conversation with David, Stefan, Haoxing, and Joseph, I was tentatively convinced that the latter idea seems more reasonable, so the current note will describe a protocol which operates optimally under the assumption that most nodes can instantly recognize most transaction sets by their hash, so that less communication is involved requesting new transaction sets than propogating each entire (or even compressed portions, using erasure coding) transaction set throughout the network.

#### Note Summary

Electoral consensus can be broken down on a high level into two parts: electing candidates for council, and applying for candidate status.

Each round of consensus, it is assumed that all nonforking nodes are in agreement about a set $$C$$ of candidates and a "global candidate fault-tolerance" $$f$$, defined as the number of candidates which are allowed to fail in a given round. Increasing $$f$$ guarantees faster termination and (in the event that $$f$$ candidates *actually* fail) prevents termination from being endlessly blocked, but decreases the average council size. In practice, the value of $$f$$ does not need to be globally agreed upon; the "actual" value will tend towards the maximum of all the $$f$$ values among the "trusted" honest nodes. In fact, $$f$$ does not even need to stay constant within a round; for instance, one could add time-outs which increase $$f$$ when consensus is taking too long to terminate in order to avoid termination being blocked by failed candidates while still trying to maximize council size during normal operations. Note that setting $$f$$ too low does not cause any issues for fork-safety; it only can cause issues for termination.

In parallel to the regular consensus process, there is also a secondary process which is used to allow nodes to slowly come to consensus on the set $$C$$. It is important for fork-safety that all honest nodes agree on $$C$$, since disagreement over $$C$$ can cause nodes to terminate without hearing reliable broadcast from all the council members. There is no real need to make changes to $$C$$ quickly though; adding new nodes should be a slow process anyway, and even if a candidate is clearly malicious, there is no danger to fork-safety from letting it remain a candidate. Further, if enough nodes remove a malicious candidate from their UNL, then that candidate will lose its ability to join the council immediately anyway.

We describe the election protocol first, and then the protocol for applying for candidacy second.

## Connectivity Properties

The correctness of the protocol naturally requires certain assumptions about the graph connectivity. For complete transparency, we define below several connectivity properties that we use as assumptions, and then whenever we describe a property of an algorithm, we reference the particular assumption needed to satisfy that individual algorithmic property (note: not all graph properties defined are as weak as they can be; in particular, properties involving "information flow" are generally quite difficult to describe exactly. Instead we use the "weakest sufficient property that can be described relatively easily"). It is probably best to skip over these properties and only refer back to them when necessary.

Let $$\mathcal{P}_i,\mathcal{P}_j$$ be two nodes. We say $$\mathcal{P}_i$$ and $$\mathcal{P}_j$$ are **directly connected** if $$\vert\mathsf{UNL}_i\cap\mathsf{UNL}_j\vert\geqslant t_i+t_j+\min\{t_i,t_j\}+1$$. We say $$\mathcal{P}_i$$ **weakly induces** $$\mathcal{P}_j$$ if $$\vert\mathsf{UNL}_i\cap\mathsf{UNL}_j\vert\geqslant t_j+\min\{t_i,t_j\}+1$$.

Direct connectivity is the condition on overlap under which if $$\mathcal{P}_i$$ receives a message from $$\vert\mathsf{UNL}_i\vert-t_i$$ nodes in $$\mathsf{UNL}_i$$, then $$\mathcal{P}_j$$ receives that same message from at least $$t_j+1$$ nodes in $$\mathsf{UNL}_j$$. This is useful both for "blocking" (e.g., if $$\mathcal{P}_i$$ terminates on some value $$X$$, then $$\mathcal{P}_j$$ cannot terminate on any value $$Y\neq X$$) and for "direct induction" (e.g., if $$\mathcal{P}_i$$ terminates on some value $$X$$, then $$\mathcal{P}_j$$ will eventually broadcast support for $$X$$). Weak induction guarantees that if $$\mathcal{P}_i$$ receives a message from every honest node in $$\mathsf{UNL}_i$$, then $$\mathcal{P}_j$$ receives that same message from at least $$t_j+1$$ nodes in $$\mathsf{UNL}_j$$.

We say $$\mathcal{P}_i$$ is **self inductive** if $$\mathcal{P}_i$$ is directly connected with every honest node in $$\mathsf{UNL}_i$$. Given two nodes $$\mathcal{P}_i,\mathcal{P}_j$$, we say $$\mathcal{P}_i$$ **strongly induces termination in** $$\mathcal{P}_j$$ if $$\mathcal{P}_i$$ is directly connected to every honest node $$\mathcal{P}_k\in\mathsf{UNL}_j$$. We say $$\mathcal{P}_i$$ **weakly induces termination in** if one of the following conditions hold: 1) $$\mathcal{P}_i$$ strongly induces termination in $$\mathcal{P}_j$$; 2) $$\mathcal{P}_i$$ is self inductive, and $$\mathcal{P}_i$$ weakly induces every honest node $$\mathcal{P}_k\in\mathsf{UNL}_j$$; or 3) $$\mathcal{P}_i$$ weakly induces termination in some node $$\mathcal{P}_k$$ which weakly induces termination in $$\mathcal{P}_j$$.

We say $$\mathcal{P}_b$$ **weakly broadcasts to** the node $$\mathcal{P}_i$$ if one of the following conditions holds: 1) $$\mathcal{P}_i\in\mathsf{BUNL}_b$$; 2) $$\mathcal{P}_b$$ weakly broadcasts to at least $$2t_i+1$$ nodes in $$\mathsf{UNL}_i$$. Thus for instance, if $$\vert\mathsf{UNL}_i\cap\mathsf{BUNL}_b\vert\geqslant 2t_i+1$$, then $$\mathcal{P}_b$$ weakly broadcasts to $$\mathcal{P}_i$$.

We say $$\mathcal{P}_b$$ **strongly broadcasts to** $$\mathcal{P}_i$$ if $$\mathcal{P}_b$$ weakly broadcasts to every honest node in $$\mathsf{UNL}_i$$. Finally, $$\mathcal{P}_b$$ **is a candidate for** $$\mathcal{P}_i$$ if $$\mathcal{P}_b$$ strongly broadcasts to every honest node in $$\mathsf{UNL}_i$$. Given some node $$\mathcal{P}_i$$, define the sets $$\mathsf{UNL}_i^n$$ inductively by setting $$\mathsf{UNL}_i^1=\mathsf{UNL}_i$$, and letting $$\mathsf{UNL}_i^n$$ be the set of nodes in the UNL of any node in $$\mathsf{UNL}_i^{n-1}$$. Then it is easily seen that $$\mathcal{P}_b$$ is a candidate for $$\mathcal{P}_i$$ iff $$\mathcal{P}_b$$ weakly broadcasts to every node in $$\mathsf{UNL}_i^2$$.

A node $$\mathcal{P}_i$$ is a called a **terminator** if $$\vert\mathsf{UNL}_i\cap\mathsf{UNL}_j\vert\geqslant\frac{\vert\mathsf{UNL}_i\vert+t_i+1}{2}+t_j$$ for every honest node $$\mathcal{P}_j\in\mathsf{UNL_i}$$. Being a terminator is the condition under which, if at least half of the honest nodes in $$\mathsf{UNL}_i$$ send some message, then every honest node $$\mathcal{P}_j\in\mathsf{UNL_i}$$ will eventually receive that message from at least $$t_j+1$$ nodes in $$\mathsf{UNL}_j$$.

**To summarize loosely the way graph properties are used: nodes automatically have pairwise fork safety if they are directly connected. In order to guarantee termination, there need to be terminators which have particularly good UNL connectivity properties, and candidates which have particularly good broadcasting properties. If all honest nodes in a closed network are directly connected, then only a single terminator is needed. As long as one doesn't overestimate the number of nonfaulty candidates, there is no real requirement on the number of candidates needed for termination; however, naturally having more candidates improves fault tolerance.**

## The Election Protocol

We begin by describing the election protocol. The election protocol is run every round to decide who is in the council.

I think the clearest way to describe the protocol is to first list the properties of the primitives we use, then describe the implementation of the "high level protocol", then finally at the end describe the actual implementation of the primitives. It is necessary to understand what the primitives do in order to understand the high level protocol (obviously), but it might be unintuitive why the primitives are designed as they are without first understanding where they come into play in the high level protocol.

In what follows we assume implicitly that for all honest nodes $$\mathcal{P}_i$$, there are fewer than $$t_i$$ faulty nodes in $$\mathsf{UNL}_i$$ and $$\vert\mathsf{UNL}_i\vert\geqslant 2t_i+1$$. The latter condition is necessary for most of the protocols to simply logically make any sense (e.g., seeing enough support for a proposal to terminate implies *at least* that some honest node should have advocated that proposal). In reality the UNL size will be forced to be larger by other conditions anyway.

#### Primitives

Electoral consensus utilizes two primitives: binary agreement ($$\mathsf{BA}$$) and reliable broadcast ($$\mathsf{RBC}$$). Binary agreement is an asynchronous consensus protocol where the proposals are restricted to binary bits. Reliable broadcast is a message-passing protocol that allows nodes to broadcast messages throughout the network in such a way that either 1) no honest node receives a message, or 2) all honest nodes will receive the same message, even if the sender was Byzantine.

Separate instances of $$\mathsf{BA}$$ and $$\mathsf{RBC}$$ are created for each candidate; reliable broadcast is used to dessiminate the proposals of candidates throughout the network, and then binary agreement is used to decide whether that reliable broadcast succeeded. A candidate is included in the council iff its binary consensus instance terminates with a value of $$1$$.

Formally, reliable broadcast has the following properties:
- RBC-Validity: If an honest node $$\mathcal{P}_i$$ accepts some message, $$\mathcal{P}_b$$ must have sent that message to some honest node.
- RBC-Agreement: No two honest, directly connected nodes accept different messages.
- RBC-Termination-Agreement: If any honest node $$\mathcal{P}_i$$ accepts some message, eventually every other honest node directly connected with $$\mathcal{P}_i$$ will eventually ready that same message.
- RBC-Strong-Termination-Agreement: If an honest node $$\mathcal{P}_i$$ accepts some message and $$\mathcal{P}_i$$ strongly induces termination in an honest node $$\mathcal{P}_j$$, then $$\mathcal{P}_j$$ will eventually accept that same message.
- RBC-Termination: If the sender is honest and is a candidate for any honest node $$\mathcal{P}_i$$, eventually $$\mathcal{P}_i$$ will accept the message sent.

Binary agreement has the following properties:
- BA-Validity: If an honest node $$\mathcal{P}_i$$ accepts some value, an honest node must have proposed that value.
- BA-Agreement: No two honest, directly connected nodes accept different values.
- BA-Termination-Agreement: If an honest node $$\mathcal{P}_i$$ terminates and $$\mathcal{P}_i$$ weakly induces termination in an honest node $$\mathcal{P}_j$$, then $$\mathcal{P}_j$$ will eventually terminate.
- BA-Termination: If there exists a terminator $$\mathcal{P}_j$$ which weakly induces termination in an honest node $$\mathcal{P}_i$$ for some honest node $$\mathcal{P}_i$$, then $$\mathcal{P}_j$$ will eventually terminate.

#### Multi-valued Consensus

At the start of the protocol, all candidates $$\mathcal{P}_b$$ (or people who think they're candidates when they actually aren't; the only issue that is caused by non-candidates proposing is that it increases the network traffic with no benefit) broadcast $$\mathsf{READY}(h)$$ to begin an instance of $$\mathsf{RBC}$$, where $$h$$ is the hash of $$\mathcal{P}_b$$'s proposal. We use $$\mathsf{RBC}_b$$ to refer to the instance of reliable broadcast involving $$\mathcal{P}_b$$, and similarly $$\mathsf{BA}_b$$ to refer to the instance of binary agreement involving $$\mathcal{P}_b$$.

At the beginning of the protocol, the node $$\mathcal{P}_i$$ creates an instance of $$\mathsf{BA}_b$$ and $$\mathsf{RBC}_b$$ for every node $$\mathcal{P}_b\in\overline{C}_i\setminus X_i$$. We assume that $$C_i$$ and $$\overline{C}_i$$ can become larger in the middle of the round at any time up until $$\mathcal{P}_i$$ broadcasts $$\mathsf{ACCEPT}$$ in step $$3$$. If $$\overline{C}_i$$ increases in the middle of the round, new instances of $$\mathsf{BA}$$ and $$\mathsf{RBC}$$ are created and run implicitly. If a node $$\mathcal{P}_b$$ is not in $$\overline{C}_i\setminus X_i$$, then $$\mathcal{P}_i$$ refuses to broadcast messages (i.e., refuses to participate in reliable broadcast or binary agreement) pertaining to $$\mathcal{P}_b$$.

Every node $$\mathcal{P}_i$$ (including the candidates themselves) then operates according to the following protocol. Let $$R_i$$ be the minimum council size we want under "normal" operations.

1. Upon accepting a hash value from $$\mathsf{RBC}_b$$, check if we know the transaction set that it references. If not, ask our peers if they know to try to find it out. After learning the transaction set which gave rise to the hash, *if $$\mathcal{P}_b$$ is in $$C_i$$*, then vote $$1$$ in $$\mathsf{BA}_b$$.
2. Once at least $$R_i$$ instances of $$\mathsf{BA}$$ have terminated (possibly not on $$1$$), thereafter automatically vote $$0$$ for every other instance of $$\mathsf{BA}$$.
3. Once all instances of $$\mathsf{BA}$$ relating to all nodes in $$\overline{C}_i$$ have terminated, broadcast $$\mathsf{ACCEPT}(h)$$, where $$h$$ is the hash of $$\overline{C}_i$$. Thereafter, do not update $$\overline{C}_i$$ during this round of consensus.
4. Wait until there exists some subset $$S\subseteq\mathsf{UNL}_i$$ such that $$\vert S\vert=\vert\mathsf{UNL}_i\vert-t_i$$, every node in $$S$$ has broadcasted an accept message, and every instance of $$\mathsf{BA}_b$$ has terminated for any node $$\mathcal{P}_b$$ in $$\overline{C}_j$$ of the accept message of any node $$\mathcal{P}_j$$ in $$S$$. Let $$T$$ be the set of nodes for which $$\mathsf{BA}$$ terminated with value $$1$$.
5. Wait until we have readied some message under reliable broadcast from every node in $$T$$. If we ready a hash that we haven't heard from, ask our peers to try to find it out; by BA-Rejection, there must have been some honest node in our UNL which voted yes on the relevant $$\mathsf{BA}$$ instance, which, due to the criterion for voting yes in step $$1$$, means they must have the transaction set, so we can get the transaction set from them. Because we have the hash, a Byzantine node in our UNL cannot trick us into taking the wrong transaction set.
6. Apply a deterministic algorithm to the proposals we now know of to construct the final block. This deterministic algorithm could be a union, or an intersection, or picking out the transactions with super-majority support, or whatever; as long as all the honest nodes agree on the same algorithm anything works.



## Applying for Candidacy

We now describe how nodes can apply to be candidates. The most important factor to being a candidate is being able to successfully do reliable broadcast. Thus this protocol effectively just runs a distributed agreement protocol checking if the applicant is able to do reliable broadcast, and then checks to make sure that every other node has also already seen that the applicant can do reliable broadcast.

At any point in time, a node $$\mathcal{P}_b$$ can begin an instance of reliable broadcast, sending a message along the lines of "candidate application|$$b$$".

Then a node $$\mathcal{P}_i$$ runs the following protocol:

1. Upon receiving a "candidate application|$$b$$" message for $$\mathcal{P}_b$$ from enough neighbors to where the RBC protocol would tell us to broadcast something, check first if $$\mathcal{P}_b$$ has applied for candidacy in the past $$65536$$ rounds; if so dispose of the message to avoid spam. Otherwise instantiate and run the RBC protocol.
2. Upon accepting the message under RBC, broadcast $$\mathsf{NOMINATE}(b)$$, and add $$\mathcal{P}_b$$ to $$\overline{C}$$.
3. Wait until at least $$\vert\mathsf{UNL}_i\vert-t_i$$ nodes in $$\mathsf{UNL}_i$$ have broadcasted $$\mathsf{NOMINATE}(b)$$, then add $$\mathcal{P}_b$$ to $$C$$.

The above protocol guarantees that if any node adds $$\mathcal{P}_b$$ to $$C$$, then every other directly connected node $$\mathcal{P}_j$$ will see at least $$t_j+1$$ nodes with $$\mathcal{P}_b$$ in $$\overline{C}$$.

## Removing Candidates

Although there are no safety issues caused by allowing too many nodes into one's candidate set or candidate watchlist, paying attention to too many "dead" (i.e., actively malicious, constantly crashed, or no longer connected enough to reliably broadcast) nodes can make termination take longer than need be. Thus we want a protocol for removing nodes from their candidate set/watchlist.

At any time, a node $$\mathcal{P}_i$$ can safely remove a node $$\mathcal{P}_b$$ from $$C_i$$; the only thing this does is makes it harder for $$\mathcal{P}_b$$ to be included in the council. However, more care needs to be taken to remove a node from $$\overline{C}_i$$; if one stops watching a node while another node is still voting to include that node in the council, that node could be included in the council without one knowing, leading to a fork.

If a node wants to exclude a node $$\mathcal{P}_b$$ from candidacy, it broadcasts the message $$\mathsf{REMOVE}(b)$$. Then any honest node $$\mathcal{P}_i$$ (implicitly) runs the following protocol.

1. Upon receiving $$\mathsf{REMOVE}(b)$$ from $$t_i+1$$ neighbors, if we have not accepted a reliable broadcast from $$b$$ within the past $$16$$ (or however many seems reasonable) rounds, broadcast $$\mathsf{REMOVE}(b)$$, and if $$\mathcal{P}_b\in C_i$$, then remove $$\mathcal{P}_b$$ from $$C_i$$.
2. Upon receiving $$\mathsf{REMOVE}(b)$$ from $$\vert\mathsf{UNL}_i\vert-t_i$$ neighbors or $$\mathsf{FULLREMOVE}(b)$$ from $$t_i+1$$ neighbors, broadcast $$\mathsf{FULLREMOVE}(b)$$, and (if we haven't already, even if we *have* accepted a reliable broadcast from $$b$$ within the past $$16$$ rounds) remove $$\mathcal{P}_b$$ from $$C_i$$ and add $$\mathcal{P}_b$$ to $$X_i$$.
3. Upon receiving $$\mathsf{FULLREMOVE}(b)$$ from $$\vert\mathsf{UNL}_i\vert-t_i$$ neighbors, remove $$\mathcal{P}_b$$ from $$\overline{C}_i$$.

The above protocol guarantees that if any node adds $$\mathcal{P}_b$$ to $$C$$, then every other directly connected node $$\mathcal{P}_j$$ will see at least $$t_j+1$$ nodes with $$\mathcal{P}_b$$ in $$\overline{C}$$.

## The Implementation of Primitives

Now we finally describe the concrete implementations of the primitives binary agreement and reliable broadcast. Binary agreement first requires *its own* primitive called **binary proposal**, or $$\mathsf{BPP}$$.

#### Binary Proposal

Binary proposal allows nodes to share their proposals for binary consensus with each other and "accept" $$0$$, $$1$$, or both, without accidentally picking up a bit that only faulty nodes proposed, with certain guarantees on the agreement of acceptance. More specifically, in $$\mathsf{BPP}$$ each node $$\mathcal{P}_i$$ broadcasts a binary digit "proposal" $$v_i$$, and creates a (private) set $$\mathsf{values}_i$$, with the following properties:

- BPP-Validity: If any honest node $$\mathcal{P}_i$$ adds a bit $$v$$ to $$\mathsf{values}_i$$, then some honest node (possibly not in $$\mathcal{P}_i$$'s UNL) proposed $$v$$.
- BPP-Agreement: If any honest node $$\mathcal{P}_i$$ adds a bit $$v$$ to $$\mathsf{values}_i$$, then every other honest node $$\mathcal{P}_j$$ will eventually add $$v$$ to $$\mathsf{values}_j$$.
- BPP-Progression: Eventually, $$\mathsf{values}_i$$ is nonempty for every honest node $$\mathcal{P}_i$$.

The protocol run by node $$\mathcal{P}_i$$ is as follows:

1. Broadcast $$\mathsf{BPP}(v_i)$$.
2. If we receive $$\mathsf{BPP}(v)$$ from at least $$t_i+1$$ nodes for the bit $$v\neq v_i$$, broadcast $$\mathsf{BPP}(v)$$.
3. If we receive $$\mathsf{BPP}(v)$$ from at least $$\vert\mathsf{UNL}_i\vert-t_i$$ nodes for any bit $$v$$, add $$v$$ to $$\mathsf{values}_i$$.

#### Reliable Broadcast

Reliable broadcast gives nodes a way to broadcast messages to the network, and have the other nodes accept some message, such that one can guarantee that either $$1)$$ no honest node accepts any message, or $$2)$$ every honest node accepts the same message, even if the sender was Byzantine. This limits the damage that a Byzantine potential councilor can do, such that a Byzantine potential councilor is no worse than a crashed potential councilor.

More specifically, we call a node a **candidate** if it has "enough" nodes listening to it in order to "effectively" propogate its messages throughout the network. This is defined formally in the "Proofs" section; it seems to be a reasonably light condition. If $$\mathcal{P}_b$$ is the sender, $$\mathsf{RBC}$$ has the following properties:
- RBC-Validity: If an honest node $$\mathcal{P}_i$$ accepts some message, $$\mathcal{P}_b$$ must have sent that message to some honest node.
- RBC-Agreement: No two honest nodes accept different messages.
- RBC-Termination-Agreement: If any honest node accepts some message, eventually every other honest node will ready that same message.
- RBC-Termination: If the sender is honest and is a candidate, every node will accept the message sent.

$$\mathsf{RBC}$$ effectively proceeds in three steps. Each step has a distinct purpose. In step $$1$$, nodes echo to each other in an attempt to flood the message throughout the network without accidentally picking up a fake message along the way. This maximizes the broadcast capabilities of nodes. In step $$2$$, nodes ready a message if they know no other node will ready a different message in the event the sender was faulty and broadcasted multiple messages. This guarantees agreement on the content of the message among honest nodes. Finally in step 3, nodes accept the message and terminate if they know every other honest node will eventually ready the same message. Since we only care about all nodes knowing what the message was, this is sufficient for our purposes (a true reliable broadcast protocol would actually require nodes to only accept if they know every other node will eventually accept; in a sparse graph this condition becomes needlessly strict. In slightly denser graphs, this protocol does guarantee that if any honest node accepts, eventually all other honest nodes will also accept).

At the start of an RBC instance, the sender $$\mathcal{P}_b$$ broadcasts $$\mathsf{READY}(proposal)$$. $$\mathcal{P}_b$$ accepts its own message when at least $$\vert\mathsf{UNL}_b\vert-t_b$$ nodes in $$\mathsf{UNL}_b$$ broadcast $$\mathsf{READY}(proposal)$$. Every other node $$\mathcal{P}_i$$ operates according to the following protocol.

1. Upon receiving $$\mathsf{READY}(v)$$ directly from $$\mathcal{P}_b$$, or upon receiving $$\mathsf{ECHO}(v)$$ from at least $$t_i+1$$ nodes, if we have not broadcasted an echo message yet, broadcast $$\mathsf{ECHO}(v)$$.
2. Upon receiving $$\mathsf{ECHO}(v)$$ from at least $$\vert\mathsf{UNL}_i\vert-t_i$$ nodes, or upon receiving $$\mathsf{READY}(v)$$ from at least $$t_i+1$$ nodes, if we have not broadcasted a ready message yet, broadcast $$\mathsf{READY}(v)$$.
3. Upon receiving $$\mathsf{READY}(v)$$ from at least $$\vert\mathsf{UNL}_i\vert-t_i$$ nodes, accept $$v$$.

#### Binary Agreement

Binary agreement allows nodes to come to consensus on a single bit. The importance of limiting the vote size to a single bit is needed for efficiency (a protocol which works like the one described below would require an estimated round count proportional to the number of possible values); more importantly though, limiting the possible votes is needed in order to ensure that "enough" votes are in agreement in order to guarantee termination.

Binary agreement has the following properties:
	- BA-Validity: If an honest node $$\mathcal{P}_i$$ accepts some value, an honest node must have proposed that value.
	- BA-Rejection: If every honest node $$\mathcal{P}_j\in\mathsf{UNL}_i$$ proposes $$0$$ in the UNL of an honest node $$\mathcal{P}_i$$, then no honest node will accept $$1$$.
	- BA-Agreement: No two honest nodes accept different values.
	- BA-Termination: All honest nodes terminate.

BA proceeds in rounds; theoretically it could go on for an arbitrarily large number of rounds (which is of course necessary by FLP), but the expected termination time is four rounds. A node begins on round $$1$$ and broadcasts the message $$\mathsf{PROP}(v_i)$$ with its vote $$v_i$$. On the $$r$$-th round, the node $$\mathcal{P}_i$$ operates according to the following protocol. $$est_i$$ is the value $$\mathcal{P}_i$$ proposes at the start of the round, and $$\mathsf{values}_i$$ starts out empty. We assume nodes can run the algorithm without having proposed a value at the beginning.

1. Begin an instance of $$\mathsf{BPP}$$ proposing value $$est_i$$.
2. If we add a bit $$v$$ to $$\mathsf{values}_i$$ and have not yet broadcasted an aux message: If $$v=0$$, then broadcast $$\mathsf{AUX}(v,r)$$. If $$v=1$$, then only if at least $$\vert\mathsf{UNL}_i\vert-t_i$$ nodes in $$\mathsf{UNL}_i$$ sent out $$\mathsf{PROP}(1)$$ do we broadcast $$\mathsf{AUX}(v,r)$$.
3. Wait until there exists some subset $$S\subseteq\mathsf{UNL}_i$$ such that $$\vert S\vert=\vert\mathsf{UNL}_i\vert-t_i$$, every node in S has sent out an aux message from round $$r$$, and $$\mathsf{values}_i$$ contains the aux value of every node in $$S$$.
4. Let $$s=r\% 2$$ (see also the section on "Regarding the Common Coin"). If $$\vert \mathsf{values}_i\vert=2$$, set $$est_i=s$$ and go back to start round $$r+1$$. If $$\vert \mathsf{values}_i\vert=1$$ and $$s$$ is the value in $$\mathsf{values}_i$$, accept s and terminate the protocol. If $$\vert\mathsf{values}_i\vert=1$$ but $$s$$ is not the value in $$\mathsf{values}_i$$, set $$est_i$$ to the value in $$\mathsf{values}_i$$ and go back to start round $$r+1$$.
