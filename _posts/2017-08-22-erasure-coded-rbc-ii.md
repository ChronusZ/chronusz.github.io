---
layout: default
title: "Erasure Coded RBC II"
date: 2017-08-22
use_math: true
---

# Erasure-Coded Reliable Broadcast V3

A couple weeks ago I posted a note describing an optimization of HoneyBadger's reliable broadcast protocol which was supposed to halve the communication complexity. As I was working out the details though, it turned out that with a malicious sender the worst case communication complexity would only improve over HoneyBadger by a factor of around $$1.6$$. Although this is still a decent improvement, it's a bit annoying that malicious actors can force the rest of the network to do extra work. I realized that this problem can be fixed by changing the order in which the protocol runs, so this note describes the ammended protocol. The ammended protocol achieves a factor-of-two improvement over HoneyBadger both in the honest-sender and the malicious-sender case, and only slightly increases the size of the constant term over the previous protocol.

The modification is surprisingly simple. The previous iteration involved echoing the erasure blocks and readying the hashes. This leads to wasted effort in the case of a malicious sender, because not all honest nodes might send out the same echo. So instead, this protocol *echoes the hashes and readies the erasure blocks*. Since Bracha's algorithm guarantees that no two honest nodes ready different messages, this eliminates the potential for wasted effort. Unfortunately, this makes it so that we need an extra verification message (for which we use the tag $$ACCEPT$$) to guarantee that if any honest node accepts the reliable broadcast then at least $$f+1$$ honest nodes are able to reconstruct the original message. Thus instead of "exchange erasure blocks then exchange hashes", we need "exchange hashes, then exchange erasure blocks, then exchange hashes again".

More explicitly, we describe the new protocol in detail as follows.

- The broadcaster encrypts its message $$M$$ using an $$(n-f,n)$$-erasure-code, constructing a block $$s_i$$ for each node $$\mathcal{P}_i$$. It then constructs a Merkle tree over the set $$\{s_i\}$$, and sends the node $$\mathcal{P}_i$$ the message $$INIT(s_i,b_i,h)$$, where $$b_i$$ is the Merkle branch corresponding to the leaf $$s_i$$, and $$h$$ is the hash of $$M$$.
- When node $$\mathcal{P}_i$$ receives $$INIT(s_i,b_i,h)$$ from the broadcaster, it verifies the branch and sends $$ECHO(h)$$ to every other node.
- Upon receiving $$ECHO(h)$$ messages from $$n-f$$ nodes, where $$h$$ is the same hash we received from the broadcaster, then broadcast $$READY(s_i,b_i,h)$$.
- Upon receiving valid ready messages from $$n-f$$ nodes, then reconstruct the message and broadcast $$ACCEPT(h)$$. At the same time, re-encrypt the message with the same erasure code as above to construct every node's block, and compute the Merkle tree over it. For every node $$\mathcal{P}_j$$ that we *didn't* receive a valid echo from (i.e., an echo whose hash matches ours), encrypt $$\{s_j,b_j\}$$ using an $$(n-2f,n)$$-erasure code, construct a Merkle tree over its blocks, and send $$INITRE(s_j',b_j',h)$$ to $$\mathcal{P}_j$$, where $$s_j'$$ is *our own* block in the encryption of $$s_j$$, $$b_j'$$ is the branch in the Merkle tree corresponding to $$s_j'$$, and $$h$$ is the hash of $$M$$.
- If we receive $$f+1$$ valid initre messages, use them to reconstruct our block $$s_i$$ and the Merkle branch $$b_i$$, then broadcast $$READY(s_i,b_i,h)$$ to every node from which we did not receive an initre message.
- Upon receiving $$n-f$$ valid accept messages for $$h$$, wait until we have reconstructed the message corresponding to $$h$$ if we haven't already. Then accept the message.
