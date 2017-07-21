---
layout: default
title: "Parliamentary Consensus II"
date: 2017-07-20
use_math: true
---

# Parliamentary Consensus II (The Protocol)

## Notation

We use $$\mathcal{P}_i$$ to refer to a node with id indexed by $$i$$. If we want to refer to a specific broadcaster node, we'll typically use the notation $$\mathcal{P}_b$$ for clarity.

A **thread** is roughly some set of nodes that someone pays attention to when making decisions. Every node $$\mathcal{P}_i$$ has a **safety net** $$N_i$$, which is the set of all threads $$\mathcal{P}_i$$ listens to. By a convenient abuse of notation, we also use $$N_i$$ to refer to the union of all the threads in $$N_i$$. It should always be clear which usage we're refering to. The **extended safety net** of $$\mathcal{P}_i$$, denoted $$N_i^{\infty}$$, is the transitive closure of its safety net; i.e., the set of all nodes in a thread of any node in a thread of any node in a thread of etc. etc. of any node in $$N_i$$.

To transition from the language of UNLs, one can think of $$N_i$$ as roughly being the set of all subsets of $$\mathsf{UNL}_i$$ consisting of at least $$3t_i+1$$ nodes, where $$t_i$$ is the maximal number of nodes that are allowed to fail in $$\mathsf{UNL}_i$$. Note that a node can certainly listen to a thread without being a member of that thread; this is necessary to give untrusted nodes a way to avoid forking with nodes that might not trust them (and hence wouldn't want to include the untrusted node in any of their threads).

Given a thread $$S$$, let $$f_S=\lfloor(\vert S\vert-1)/3\rfloor$$. A thread $$S$$ is **healthy** if the number of unhealthy nodes in $$S$$ is no greater than $$f_S$$, where a node is **healthy** if it is nonfaulty and all of its threads are healthy. Although this definition appears circular at first, it can be made noncircular by defining a sequence of sets $$F_n$$ for each natural number $$n\in\mathbb{N}$$, where $$F_1$$ is the set of faulty nodes, and $$F_n$$ is the union of $$F_{n-1}$$ with the set of all nodes $$\mathcal{P}_i$$ such that $$\vert S\cap F_{n-1}\vert\geqslant f_S+1$$ for some thread $$S\in N_i$$. Then the set of unhealthy nodes is simply defined to be the union of the $$F_n$$ across all $$n\in\mathbb{N}$$.

We say that $$\mathcal{P}_i$$ and $$\mathcal{P}_j$$ are **directly connected** if there is at least one healthy thread in $$N_i\cap N_j$$. We say that $$\mathcal{P}_i$$ is **strongly connected to** $$\mathcal{P}_j$$ if for every node $$\mathcal{P}_k\in N_i$$, $$\mathcal{P}_j$$ and $$\mathcal{P}_k$$ are directly connected. We say that $$\mathcal{P}_i$$ is **split** if there are two nodes $$\mathcal{P}_j,\mathcal{P}_k\in N_i$$ such that $$\mathcal{P}_j$$ and $$\mathcal{P}_k$$ are *not* directly connected, and $$\mathcal{P}_i$$ is **unsplit** otherwise. We say that $$\mathcal{P}_i$$ is **fully unsplit** if every pair of nodes $$\mathcal{P}_j,\mathcal{P}_k$$ in $$\mathcal{P}_i$$'s *extended* safety net is directly connected. It is easily seen that any node in the extended safety net of a fully unsplit node is itself fully unsplit. Conversely, 

A **candidate** is a node which has some chance of becoming a council member. Each node $$\mathcal{P}_i$$ keeps track of a **ballot** $$B_i$$, consisting of all the nodes $$\mathcal{P}_i$$ knows are candidates. They also keep track of a **tablist** $$\overline{B}_i$$, consisting of all nodes which $$\mathcal{P}_i$$ thinks might be candidates, but $$\mathcal{P}_i$$ might not have received enough confirmations to know that those nodes are actually candidates. For simplicity, we assume that $$B_i\subseteq \overline{B}_i$$. In practice, $$\overline{B}_i$$ can reasonably be kept implicit; it only exists to deal with asynchrony, and every node in $$\overline{B}_i$$ should eventually be added to $$B_i$$.

A node $$\mathcal{P}_i$$ **sees weak support for** a message $$v$$ if *there exists some* thread $$S\in N_i$$ such that at least $$f_S+1$$ nodes in $$S$$ have broadcasted $$v$$. $$\mathcal{P}_i$$ **sees strong support for** $$v$$ if *for every* thread $$S\in N_i$$, at least $$2f_S+1$$ nodes in $$S$$ have broadcasted $$v$$. Seeing strong support for a message is exactly the condition under which a node knows that eventually all directly connected nodes will eventually see weak support for that message; this is the key property underpinning most parts of the parliamentary consensus protocol.

## The Election Protocol

We begin by describing the election protocol. The election protocol is run every round to decide who is in the council.

I think the clearest way to describe the protocol is to first list the properties of the primitives we use, then describe the implementation of the "high level protocol", then finally at the end describe the actual implementation of the primitives. It is necessary to understand what the primitives do in order to understand the high level protocol (obviously), but it might be unintuitive why the primitives are designed as they are without first understanding where they come into play in the high level protocol.

#### Primitives

Electoral consensus utilizes two primitives: validated binary agreement ($$\mathsf{VBA}$$) and reliable broadcast ($$\mathsf{RBC}$$). Validated binary agreement is an asynchronous consensus protocol where the proposals are restricted to binary bits. The "validated" part of the title refers to the fact that nodes have two states, validating and nonvalidating, and $$\mathsf{VBA}$$ can only terminate on $$1$$ if "enough" nodes are validating. Reliable broadcast is a message-passing protocol that allows nodes to broadcast messages throughout the network in such a way that either 1) no healthy node accepts a message, or 2) all healthy nodes will accept the same message, even if the sender was Byzantine.

Separate instances of $$\mathsf{VBA}$$ and $$\mathsf{RBC}$$ are created for each candidate; reliable broadcast is used to dessiminate the proposals of candidates throughout the network, and then binary agreement is used to decide whether that reliable broadcast succeeded. A candidate is included in the council iff its binary consensus instance terminates with a value of $$1$$.

Formally, reliable broadcast has the following properties:
- RBC-Validity: If a healthy node $$\mathcal{P}_i$$ accepts some message, $$\mathcal{P}_b$$ must have sent that message to some healthy node.
- RBC-Agreement: No two healthy, directly connected nodes accept different messages.
- RBC-Termination-Agreement: If a healthy node $$\mathcal{P}_i$$ accepts some message, then every other fully unsplit, healthy node directly connected with $$\mathcal{P}_i$$ will eventually ready that same message.
- RBC-Strong-Termination-Agreement: If a healthy node $$\mathcal{P}_i$$ accepts some message, then every other fully unsplit, healthy node *strongly* connected with $$\mathcal{P}_i$$ will eventually *accept* that same message.

Validated Binary agreement has the following properties:
- VBA-Validity: If a healthy node $$\mathcal{P}_i$$ accepts some value, a healthy node must have proposed that value.
- VBA-Agreement: No two healthy, directly connected nodes accept different values.
- VBA-Rejection: If any honest node $$\mathcal{P}_i$$ accepts $$1$$, then for every node $$\mathcal{P}_j$$ directly connected to $$\mathcal{P}_i$$, there is some thread $$S\in N_j$$ such that at least $$f_S+1$$ nodes entered $$\mathsf{VALIDATING}$$ mode.
- VBA-Termination: Every healthy, nonsplit node eventually terminates with probability $$1$$.

#### Multi-valued Consensus (based on Ben-Or's protocol)

At the start of the protocol, all candidates $$\mathcal{P}_b$$ (or people who think they're candidates when they actually aren't; the only issue that is caused by non-candidates proposing is that it increases the network traffic with no benefit) broadcast $$\mathsf{READY}(h)$$ to begin an instance of $$\mathsf{RBC}$$, where $$h$$ is the hash of $$\mathcal{P}_b$$'s proposal. We use $$\mathsf{RBC}_b$$ to refer to the instance of reliable broadcast involving $$\mathcal{P}_b$$, and similarly $$\mathsf{VBA}_b$$ to refer to the instance of binary agreement involving $$\mathcal{P}_b$$.

At the beginning of the protocol, the node $$\mathcal{P}_i$$ creates an instance of $$\mathsf{VBA}_b$$ and $$\mathsf{RBC}_b$$ for every node $$\mathcal{P}_b\in\overline{B}_i$$. We assume that $$B_i$$ and $$\overline{B}_i$$ can become larger in the middle of the round at any time up until $$\mathcal{P}_i$$ broadcasts $$\mathsf{ACCEPT}$$ in step $$3$$. If $$\overline{B}_i$$ increases in the middle of the round, new instances of $$\mathsf{VBA}$$ and $$\mathsf{RBC}$$ are created and run implicitly. If a node $$\mathcal{P}_b$$ is not in $$\overline{B}_i$$, then $$\mathcal{P}_i$$ refuses to broadcast messages (i.e., refuses to participate in reliable broadcast or binary agreement) pertaining to $$\mathcal{P}_b$$.

Every node $$\mathcal{P}_i$$ (including the candidates themselves) then operates according to the following protocol. Let $$R$$ be the minimum council size we want; $$R$$ doesn't need to be globally agreed upon, but the "actual" value will tend towards the minimum among the "trusted" nodes. Setting $$R$$ too high can block termination if there are fewer than $$R$$ nonfaulty candidates. In practice, $$R$$ can decrease in the middle of a round of consensus, so if one sees consensus stalling they can decrease $$R$$ to aid termination. This won't cause any issues for fork-safety, but lowering $$R$$ too much lowers decentralization.


1. Upon readying a hash value from $$\mathsf{RBC}_b$$, check if we know the transaction set that it references. If not, ask our peers if they know to try to find it out. After learning the transaction set which gave rise to the hash, set our state in $$\mathsf{VBA}_b$$ to $$\mathsf{VALIDATING}$$.
2. Upon accepting a hash value from $$\mathsf{RBC}_b$$, if $$\mathcal{P}_b$$ is in $$B_i$$ and we are already in $$\mathsf{VALIDATING}$$ mode for $$\mathsf{VBA}_b$$ (i.e., we know the transaction set leading to the hash), then vote $$1$$ in $$\mathsf{VBA}_b$$.
2. Once at least $$R$$ instances of $$\mathsf{VBA}$$ have terminated (possibly not on $$1$$), thereafter automatically vote $$0$$ for every other instance of $$\mathsf{VBA}$$. Additionally, for any instance of $$\mathsf{VBA}$$
3. Once all instances of $$\mathsf{VBA}$$ relating to all nodes in $$\overline{B}_i$$ have terminated, let $$T$$ be the set of nodes in $$\overline{B}_i$$ which terminated on $$1$$.
4. Wait until we have readied some message under reliable broadcast from every node in $$T$$. If we ready a hash that we haven't heard from, ask our peers to try to find it out; by VBA-Rejection, there must have been at least $$f_S+1$$ nonfaulty nodes in one of our threads which was in $$\mathsf{VALIDATING}$$ mode on the relevant $$\mathsf{VBA}$$ instance, which, due to the criterion for entering $$\mathsf{VALIDATING}$$ mode in step $$1$$, means they must have the transaction set. Thus we can get the transaction set from at least one of them. Because we have the hash, a Byzantine node cannot trick us into taking the wrong transaction set.
5. Apply a deterministic algorithm to the proposals we now know of to construct the final block. This deterministic algorithm could be a union, or an intersection, or picking out the transactions with super-majority support, or whatever; as long as all the honest nodes that don't want to fork with each other agree on the same algorithm anything works.


## Amending the Ballot

We now describe the ballot amendment protocol, which allows nodes to join or be removed from the ballot. In the name of decentralization, it helps if the ballot is as large as possible. However, including too many faulty nodes on the ballot can make termination take longer at no benefit to decentralization, since termination time grows logarithmically with the number of candidates and faulty nodes will never be chosen for the council. Thus, allowing a dynamic council is necessary for optimal long-term operations.

The most important factor to being a candidate is being able to successfully do reliable broadcast. Thus this protocol effectively just checks if the applicant is able to do and at least somewhat regularly doing reliable broadcast, and then checks to make sure that every other node they care about not forking with has also already seen that the applicant can do reliable broadcast.

At any point in time, a node $$\mathcal{P}_b$$ which is not a candidate (or one that is a candidate, although there's no reason to in that case) can begin an instance of reliable broadcast, sending a message like $$"app:b"$$. Nodes can choose to ignore this reliable broadcast, but if they can be reasonably confident that the broadcaster is legitimate (e.g., if the sender can verify that it's a certain bank, and that bank is not overly affiliated with any other candidates) then they should try to participate in the reliable broadcast.

Let $$r$$ be a natural number such that we think every reasonable candidate should complete reliable broadcast *at least* once every $$r$$ rounds of consensus. There is no requirement on $$r$$ for safety, but setting it too low might overly strain honest candidates and setting it too high might make slow down consensus by allowing lame nodes into the ballot.

Starting at the beginning of every round of consensus, the following protocol is run in parallel; correct nodes are required to wait until this algorithm terminates before moving on to the next round of the main multi-valued consensus protocol. This is required to ensure safety. Changes to $$\overline{B}_i$$ and $$B_i$$ go into effect the following round. All messages should be signed with the id of the current round of consensus to avoid messages overlapping due to asynchrony.

1. Let initially $$\overline{B}_i$$ be the set of all nodes we have accepted reliable broadcast from within the past $$r$$ rounds. Broadcast $$\mathsf{BALLOT}(\overline{B}_i)$$.
2. Upon accepting a reliable broadcast from a node, add that node to $$\overline{B}_i$$.
3. For every thread $$S\in N_i$$, wait until there exists some subset $$T\subseteq S$$, such that $$\vert T\vert=2f_S+1$$, every node in $$T$$ has sent out a ballot message from round $$r$$, and $$\overline{B}_i$$ contains the ballot suggestion of every node in $$T$$.
4. Let $$B_i$$ be the set of all nodes strongly supported in the ballot messages we received.

The above ammendment protocol satisfies two properties:
- Safety: If a healthy node adds $$\mathcal{P}_b$$ to its ballot, then every other directly connected healthy node will add $$\mathcal{P}_b$$ to its tablist before terminating the protocol.
- Termination: If a node is fully unsplit, then eventually it will terminate.

Proof of safety: Suppose a healthy node $$\mathcal{P}_i$$ adds a node $$\mathcal{P}_b$$ to $$B_i$$. Then for every node $$\mathcal{P}_j$$ directly connected with $$\mathcal{P}_i$$, there must be some thread $$S\in N_j$$ such that at least $$f_S+1$$ honest nodes in $$S$$ broadcasted support for $$\mathcal{P}_i$$ in their ballot messages. Thus $$\mathcal{P}_j$$ won't move past step $$3$$ until adding $$\mathcal{P}_b$$ to $$\overline{B}_j$$. □

Proof of termination: Suppose $$\mathcal{P}_i$$ is healthy and fully unsplit. Then if $$\mathcal{P}_j\in N_i^{\infty}$$ is a healthy node which broadcasts support for a node $$\mathcal{P}_b$$, then $$\mathcal{P}_j$$ must have accepted a reliable broadcast from $$\mathcal{P}_b$$, so by RBC-Strong-Termination-Agreement, eventually $$\mathcal{P}_i$$ will also accept a reliable broadcast from $$\mathcal{P}_b$$ and add $$\mathcal{P}_b$$ to $$\overline{B}_i$$. Thus eventually $$\overline{B}_i$$ will contain the ballot suggestion of every node in its extended safety net; hence in particular it will contain the ballot suggestion of every node in its safety net. Since $$\mathcal{P}_i$$ is healthy by assumption, at least $$2f_S+1$$ nodes in every thread $$S\in N_i$$ is healthy, so eventually $$\mathcal{P}_i$$ will satisfy the predicate in step $$3$$ and terminate. □

## Implementing the Primitives

Here we describe finally how the primitives themselves work. There are actually three primitives that are needed: binary proposal ($$\mathsf{BPP}$$), validated binary agreement ($$\mathsf{VBA}$$), and reliable broadcast ($$\mathsf{RBC}$$). $$\mathsf{RBB}$$ is used to construct $$\mathsf{VBA}$$, and $$\mathsf{VBA}$$ and $$\mathsf{RBC}$$ are used in conjunction to construct the overlying multi-valued consensus protocol.

#### Binary Proposal (based on Mostefaouil's protocol)

Binary proposal allows nodes to share their proposals for binary consensus with each other and "accept" $$0$$, $$1$$, or both, without accidentally picking up a bit that only faulty nodes proposed, with certain guarantees on the agreement of acceptance. More specifically, in $$\mathsf{BPP}$$ each node $$\mathcal{P}_i$$ broadcasts a binary digit "proposal" $$v_i$$, and creates a (private) set $$\mathsf{values}_i$$, with the following properties:

- BPP-Validity: If any healthy node $$\mathcal{P}_i$$ adds a bit $$v$$ to $$\mathsf{values}_i$$, then some healthy node proposed $$v$$.
- BPP-Agreement: If any healthy node $$\mathcal{P}_i$$ adds a bit $$v$$ to $$\mathsf{values}_i$$, then every other healthy node $$\mathcal{P}_j$$ strongly connected to $$\mathcal{P}_i$$ will eventually add $$v$$ to $$\mathsf{values}_j$$.
- BPP-Progression: If $$\mathcal{P}_i$$ is unsplit, then eventually $$\mathsf{values}_i$$ is nonempty.

The fact that we restrict proposals to binary values is important for guaranteeing BPP-Progession holds, as can be seen in the proof below.

The protocol run by node $$\mathcal{P}_i$$ is as follows:


1. Broadcast $$\mathsf{BPP}(v_i)$$.
2. If we see weak support for $$\mathsf{BPP}(v)$$, where $$v$$ is any bit we have not yet broadcasted, broadcast $$\mathsf{BPP}(v)$$.
3. If we see strong support for $$\mathsf{BPP}(v)$$, where $$v$$ is any bit, add $$v$$ to $$\mathsf{values}_i$$.


Proof of BPP-Validity:
If a healthy node $$\mathcal{P}_i$$ adds a bit $$v$$ to $$\mathsf{values}_i$$, then it must have seen strong support for $$\mathsf{BPP}(v)$$; in particular, a healthy node in $$N_i$$ must have broadcasted $$\mathsf{BPP}(v)$$. Now suppose a healthy node $$\mathcal{P}_i$$ broadcasts $$\mathsf{BPP}(v)$$. Then $$\mathcal{P}_i$$ saw weak support for $$\mathsf{BPP}(v)$$, which means at least one healthy node in $$\mathcal{P}_i$$'s safety net must have broadcasted $$\mathsf{BPP}(v)$$. But since there are only a finite number of nodes, by infinite descent there must have been some healthy node $$\mathcal{P}_k$$ such that no healthy node in $$N_k$$ broadcasted $$\mathsf{BPP}(v)$$ before $$\mathcal{P}_k$$. Thus $$\mathcal{P}_k$$ must have proposed $$v$$, so $$v$$ was proposed by a healthy node as claimed. □

Proof of BPP-Agreement:
If $$\mathcal{P}_i$$ is healthy and adds $$v$$ to $$\mathsf{values}_i$$, then $$\mathcal{P}_i$$ must have seen strong support for $$\mathsf{BPP}(v)$$. Since $$\mathcal{P}_j$$ is strongly connected to $$\mathcal{P}_i$$, by definition every healthy node in $$N_j$$ is directly connected to $$\mathcal{P}_i$$, so every healthy node in $$N_j$$ eventually sees weak support for $$\mathsf{BPP}(v)$$ and thus broadcasts $$\mathsf{BPP}(v)$$. Thus $$\mathcal{P}_j$$ eventually sees strong support for $$\mathsf{BPP}(v)$$ and adds $$v$$ to $$\mathsf{values}_j$$. □

Proof of BPP-Progression:
Given a healthy thread $$S$$, since each node can only vote $$0$$ or $$1$$ and $$S$$ has at least $$2f_S+1$$ healthy nodes, there must be some bit which was voted by at least $$f_S+1$$ healthy nodes in $$S$$. Call this bit the **primary vote** of $$S$$, notated $$v(S)$$. If an honest node listens to $$S$$, then that node will eventually broadcast $$\mathsf{BPP}(v(S))$$ by the predicate in step $$2$$. If $$\mathcal{P}_j$$ is healthy and every thread $$S\in N_j$$ has the same primary vote $$v$$, then for every node $$\mathcal{P}_k$$ directly connected to $$\mathcal{P}_j$$, $$\mathcal{P}_k$$ must see weak support for $$v$$ and will eventually broadcast $$\mathsf{BPP}(v)$$. In particular, if $$\mathcal{P}_i$$ is unsplit and there exists some healthy node $$\mathcal{P}_j\in N_i$$ for which all of its threads have the same primary vote $$v$$, then eventually every healthy node in $$N_i$$ will broadcast support for $$v$$, leading $$\mathcal{P}_i$$ to eventually see strong support for $$v$$ and add it to $$\mathsf{values}_i$$.

If on the other hand, for every healthy node $$\mathcal{P}_j\in N_i$$, there exist threads $$S,S'\in N_j$$ such that $$v(S)\neq v(S')$$, then every healthy node in $$N_i$$ will eventually broadcast support for both bits, leading $$\mathcal{P}_i$$ to eventually add both bits to $$\mathsf{values}_i$$. Either way, $$\mathsf{values}_i$$ is eventually nonempty. □

#### Reliable Broadcast (based on Bracha's protocol)

Reliable broadcast gives nodes a way to broadcast messages to the network, and have the other nodes accept some message, such that one can guarantee that either $$1)$$ no honest node accepts any message, or $$2)$$ every honest node accepts the same message, even if the sender was Byzantine. This limits the damage that a Byzantine potential councilor can do, such that a Byzantine potential councilor is no worse than a crashed potential councilor.

If $$\mathcal{P}_b$$ is the sender, $$\mathsf{RBC}$$ has the following properties:
- RBC-Validity: If a healthy node $$\mathcal{P}_i$$ accepts some message, $$\mathcal{P}_b$$ must have sent that message to some healthy node.
- RBC-Agreement: No two healthy, directly connected nodes accept different messages.
- RBC-Termination-Agreement: If a healthy node $$\mathcal{P}_i$$ accepts some message, then every other fully unsplit, healthy node directly connected with $$\mathcal{P}_i$$ will eventually ready that same message.
- RBC-Strong-Termination-Agreement: If a healthy node $$\mathcal{P}_i$$ accepts some message, then every other fully unsplit, healthy node *strongly* connected with $$\mathcal{P}_i$$ will eventually *accept* that same message.

$$\mathsf{RBC}$$ effectively proceeds in three steps. Each step has a distinct purpose. In step $$1$$, nodes echo to each other in an attempt to flood the message throughout the network without accidentally picking up a fake message along the way. This maximizes the broadcast capabilities of nodes. In step $$2$$, nodes ready a message if they know no other node will ready a different message in the event the sender was faulty and broadcasted multiple messages. This guarantees agreement on the content of the message among honest nodes. Finally in step 3, nodes accept the message and terminate if they know every other honest node will eventually ready the same message. Since we only care about all nodes knowing what the message was, this is sufficient for our purposes (a true reliable broadcast protocol would actually require nodes to only accept if they know every other node will eventually accept; in a sparse graph this condition becomes needlessly strict. In slightly denser graphs, this protocol does guarantee that if any honest node accepts, eventually all other honest nodes will also accept).

At the start of an RBC instance, the sender $$\mathcal{P}_b$$ broadcasts $$\mathsf{READY}(proposal)$$. $$\mathcal{P}_b$$ accepts its own message when it sees strong support for $$\mathsf{READY}(proposal)$$. Every other node $$\mathcal{P}_i$$ operates according to the following protocol.


1. Upon receiving $$\mathsf{READY}(v)$$ directly from $$\mathcal{P}_b$$, or upon seeing weak support for $$\mathsf{ECHO}(v)$$, if we have not broadcasted an echo message yet, broadcast $$\mathsf{ECHO}(v)$$.
2. Upon seeing strong support for $$\mathsf{ECHO}(v)$$, or upon seeing weak support for $$\mathsf{READY}(v)$$, if we have not broadcasted a ready message yet, broadcast $$\mathsf{READY}(v)$$.
3. Upon seeing strong support for $$\mathsf{READY}(v)$$, accept $$v$$.


RBC-Validity follows by the same general argument as BPP-Validity. RBC-Agreement follows from the fact that no healthy thread can strongly support two contradictory statements.

Proof of RBC-Termination-Agreement:
Suppose $$\mathcal{P}_j$$ is healthy and fully unsplit. Suppose a healthy node in $$N_j^{\infty}$$ broadcasts $$\mathsf{READY}(v)$$ for some message $$v$$. Let $$\mathcal{P}_k$$ be the first healthy node in $$N_j^{\infty}$$ to broadcast a ready message for $$v$$. Since $$N_k\subseteq N_j^{\infty}$$ by definition, $$\mathcal{P}_k$$ must have broadcasted ready before any healthy node in $$N_k$$, so in particular $$\mathcal{P}_k$$ could not have broadcasted ready due to seeing weak support for $$\mathsf{READY}(v)$$. Thus $$\mathcal{P}_k$$ must have broadcasted ready due to seeing strong support for $$\mathsf{ECHO}(v)$$. But any two healthy nodes $$N_j^{\infty}$$ are by definition directly connected, and thus cannot both see strong support for contradictory messages. Thus no two healthy nodes in $$N_j^{\infty}$$ can ready contradictory messages.

Now if a healthy node $$\mathcal{P}_i$$ directly connected with $$\mathcal{P}_j$$ accepts a message $$v$$, then $$\mathcal{P}_j$$ will eventually see weak support for $$\mathsf{READY}(v)$$. Since $$\mathcal{P}_j$$ is healthy by assumption, there must have been some honest node in one of its threads which broadcasted $$\mathsf{READY}(v)$$; by the previous paragraph, $$\mathcal{P}_j$$ cannot have already broadcasted $$\mathsf{READY}(w)$$ for any message $$w\neq v$$. Thus eventually $$\mathcal{P}_j$$ will broadcast $$\mathsf{READY}(v)$$ by the predicate in step $$2$$. □

RBC-Strong-Termination-Agreement follows from RBC-Termination-Agreement by definition of strong connectivity and the fact that any node in the safety net of a fully unsplit node is itself fully unsplit: since thus eventually every healthy node in $$N_j$$ will ready the message, causing $$\mathcal{P}_j$$ to see strong support for that message.

#### Validated Binary Agreement (based on Mostefaouil's protocol)

Validated binary agreement allows nodes to come to consensus on a single bit. The importance of limiting the vote size to a single bit is needed for efficiency (a protocol which works like the one described below would require an estimated round count proportional to the number of possible values); more importantly though, limiting the possible votes is needed in order to ensure that "enough" votes are in agreement in order to guarantee eventual termination.

Validated binary agreement has the following properties:
- VBA-Validity: If a healthy node $$\mathcal{P}_i$$ accepts some value, a healthy node must have proposed that value.
- VBA-Agreement: No two healthy, directly connected nodes accept different values.
- VBA-Rejection: If any honest node $$\mathcal{P}_i$$ accepts $$1$$, then for every node $$\mathcal{P}_j$$ directly connected to $$\mathcal{P}_i$$, there is some thread $$S\in N_j$$ such that at least $$f_S+1$$ nodes entered $$\mathsf{VALIDATING}$$ mode.
- VBA-Termination: Every healthy, unsplit node eventually terminates with probability $$1$$.

$$\mathsf{VBA}$$ proceeds in rounds; theoretically it could go on for an arbitrarily large or potentially even infinite number of rounds (which is of course necessary by FLP), but the expected termination time is at most four rounds, and the probability of having terminated by the end of each round after that rapidly approaches $$1$$. A node begins on round $$1$$ and broadcasts the message $$\mathsf{PROP}(v_i)$$ with its vote $$v_i$$. On the $$r$$-th round, the node $$\mathcal{P}_i$$ operates according to the following protocol. $$est_i$$ is the value $$\mathcal{P}_i$$ proposes at the start of the round, and $$\mathsf{values}_i$$ starts out empty. We assume nodes can run the algorithm without having proposed a value at the beginning.


1. Begin an instance of binary proposal voting $$est_i$$.
2. If we add a bit $$v$$ to $$\mathsf{values}_i$$ and have not yet broadcasted an aux message: If $$v=0$$, then broadcast $$\mathsf{AUX}(v,r)$$. If $$v=1$$, then broadcast $$\mathsf{AUX}(v,r)$$ only if we're in $$\mathsf{VALIDATING}$$ mode.
3. For every thread $$S\in N_i$$, wait until there exists some subset $$T\subseteq S$$, such that $$\vert T\vert=2f_S+1$$, every node in $$T$$ has sent out an aux message from round $$r$$, and $$\mathsf{values}_i$$ contains the aux value of every node in $$S$$.
4. Let $$s=r\% 2$$ (see also the section on "Regarding the Common Coin"). If $$\vert \mathsf{values}_i\vert=2$$, set $$est_i=s$$ and go back to start round $$r+1$$. If $$\vert \mathsf{values}_i\vert=1$$ and $$s$$ is the value in $$\mathsf{values}_i$$, accept $$s$$ and terminate the protocol. If $$\vert\mathsf{values}_i\vert=1$$ but $$s$$ is not the value in $$\mathsf{values}_i$$, set $$est_i$$ to the value in $$\mathsf{values}_i$$ and go back to start round $$r+1$$.


I'll prove the VBA properties in another note; they require more effort to prove than the BPP and RBC properties, and trying to cram the proofs into this already-huge note would just be overkill. Also I'm not sure VBA is in its final form; my intuition tells me I can make it work better in incomplete graphs with a bit more work.


## Regarding the Common Coin

The binary agreement protocol by Mostefaouil (which VBA is mostly a copy of) uses a "common coin", which is basically a threshold signature scheme used as a source of random bits that all honest nodes agree upon, but for which the Byzantine nodes are unable to predict until at least one honest node has requested the coin's value. The neccessity of such a common coin comes from the assumption that the "adversary" (i.e., the entity that is assumed to be controlling all of the Byzantine nodes in order to wreak optimum havok on the network) controls when messages are delivered between honest nodes. Without the coin, the adversary can make messages be delivered in an unoptimal order to prevent the algorithm from ever terminating. The common coin makes it so that the adversary has no way to know when to deliver messages in order to prevent termination.

I personally think the assumption that the adversary can control message delivery times is an absurd think be concerned about in this case. Even if the adversary can predict the value of the common coin, this only allows the adversary to block termination and is not a threat to fork-safety. And if the adversary chooses when messages are delivered between honest nodes, then it can just as easily prevent those messages from ever being delivered, which would be a far more effective way of preventing termination anyway. Thus the common coin seems pointless to me. Further, computing the value of the common coin from what I can tell is a significant part of the latency cost of Mostefaouil's original algorithm, so avoiding the use of a common coin should give a good performance boost.

Regardless, we still need a way to get coin values for each round which are agreed upon by all honest nodes. An alternative is to simply use the round number mod $$2$$. This has the benefit of being very simple, and (since round numbers start at $$1$$) if all honest nodes propose $$1$$ then consensus will always terminate on the first round. Since the multi-valued algorithm needs to wait until a certain number of $$\mathsf{BA}$$ have terminated on $$1$$ before nodes can start voting $$0$$, this speeds up the time it takes for nodes to move onto the next step. It also biases the council size towards being larger, which seems like a good thing.

I may be in the minority though in thinking that letting the adversary control network delays to prevent termination is not an issue. If this is the case it would be good to have a common coin protocol anyway. Unfortunately, my aversion to common coins is not only based on the fact that I think they're unneccessary, but also I have no idea if it's possible to make a common coin protocol in the relevant domain (i.e., when there can be an arbitrary number of Byzantine nodes so long as the number of Byzantine nodes in any healthy thread is less than one-third). Traditional common coins work by using a threshold signature as a source of random bits. However, it's unclear to me how to make threshold schemes work with an unbounded number of faults. A less-than-satisfactory option that nonetheless might work would be to only give the "core" validators threshold shares, under the assumption that fewer than $$f$$ core validators are faulty for some prescribed $$f$$ which is less than half of the core validators, and every other node listens to the core validators in order to learn the coin values. Then we could just do a regular threshold signature algorithm with no trouble.

If anyone has some other ideas for how a common coin could work more satisfactorily, I'd like to hear it, if for nothing other than to know that it's possible.
