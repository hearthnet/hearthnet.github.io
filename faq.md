---
title: FAQ
---

{% include nav.html %}

# FAQ

This page answers common questions about what Hearth can and can't do.

## Limits

### Can my chain run SQL or server code?

No. A chain stores key-value records. Its rules are WASM policy modules that accept or refuse each transaction. There is no database server and no code of yours running on the host.

### Can I change my chain's rules later?

No. The rules are part of the chain's spec, which is fixed in its genesis at launch.

### Is my data private?

No. Everything on a chain is public. Anyone can read it from any node that hosts it.

### Is this on mainnet?

No. Hearth runs on testnet only.

### Can I get a refund?

No. There are no refunds. Your recourse is renewal: if a host serves badly, don't renew, and order from another host. Check a host's record on the **Hosts** tab before you pay.

### What if the host doesn't launch my chain?

The page shows it: the order stays *owed* but not *Launched*, and the host's *Serving now* count shows it. If the host was full or the path was taken, the order counts against the host as *paid, not served*. You don't get the coin back.

### Does the host make my chain's blocks?

No. Hosts only host. Your chain's blocks come from whoever mines it or commits it on its parent, for its block reward. That can be a miner, the host's own miner, or you.

### Who gets my chain's mining rewards?

Whoever makes its blocks.

### Can someone take my chain's name?

Before you pay, the web app refuses a path that is already a chain, or that someone else has ordered at the host. If two orders at one host name the same path with different payloads, the earlier one wins and the later one is *taken*. Nothing protects a chain from a heavier rival chain at the same path.

### How long until my chain is up?

An order settles 16 market blocks after it is written. The host then creates the chain on its next pass. Paid time counts from the order's own block.

## Burns

### Why would a host burn coin?

A new host has no record. Burning coin shows it has something at stake. See [Burns](burns.md).

### Does a burn mean a host serves well?

No. A burn costs whoever made it. It says nothing about how a host serves.

### Can I burn for a host?

Yes. Anyone can. Send a plain transfer to the host's burn address on any chain from Nexus down to the market.

### Why aren't burns added up?

Each chain's coin is its own. Nexus coin, testnet coin and market coin aren't worth the same, so the web app shows each chain's burn separately.

### Does `hearth sell --burn` burn every time it runs?

No. It burns only what the burned total is missing, up to the amount. Once the total is reached, by anyone, it burns nothing.

## Mining

### What does a host's `mines` claim promise?

Nothing on the market enforces it. It is a claim, not a contract. See [Mining](mining.md).

### How do I check it?

On the **Hosts** tab, the web app shows the hashrate your chains' blocks credit the host, over the last day or the last 100 blocks, whichever has fewer. Open it with `?node=<your node's read URL>` to read the blocks from your own node.

### What if a host mines less than it claims?

Don't renew. That is the only recourse.

### Can a host fake the credited hashrate?

Not on your own node. Each block proves its work, `2^256 / (target + 1)` hashes on average, and names who it credits; your node checks both. A node you don't run can misreport block data, so read from your own.
