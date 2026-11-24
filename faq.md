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

### Why would anyone host?

To earn Hearth coin, the market's coin, which pays for hosting and storage from any seller. A host earns it two ways: buyers' orders pay its seller key, and the market blocks it makes pay its sponsor's key (1,000 coin a block). It spends the coin on hosting or storage, or sells it on the chains' own exchange with `hearth sell-coin`. See [Why host](sell.md#why-host).

### Where do I get Hearth coin?

From hosts. They earn it and sell it on the chains' own exchange, and you buy it on the web app's **Buy coin** page with parent-chain coin. The chains check both sides, so nobody holds your payment in between. See [Buy market coin](launch.md#1-buy-market-coin).

### Can a host cancel coin it put on sale?

No. There is no cancel or refund for a deposit. It stays on sale until someone buys it; the host can only buy it back like any buyer. See [Sell your coin](sell.md#sell-your-coin-hearth-sell-coin).

### Is this on mainnet?

Yes. Hearth runs on mainnet.

### Can I get a refund?

No. There are no refunds. Your recourse is renewal: if a host serves badly, don't renew, and order from another host. Check a host's record on the **Hosts** tab before you pay.

### What if the host doesn't launch my chain?

The page shows it: the order stays *owed* but not *Launched*, and the host's *Serving now* count shows it. If the host was full or the path was taken, the order counts against the host as *paid, not served*. You don't get the coin back.

### Does the host make my chain's blocks?

Yes. Every host makes zero-work blocks, about one per target block time, on each chain it hosts, so your chain lives without miners. Anyone else may mine it too, you included. See [Blocks and mining](mining.md).

### Who gets my chain's mining rewards?

Whoever makes its blocks. A host's blocks pay its sponsor's key, which spends them on your visitors' write fees.

### Do my visitors need coin to write?

No, if your hosts run a public sponsor. The sponsor pays the write fee, and the visitor's device does a few seconds of proof-of-work instead. Without a sponsor, the writer needs coin on your chain.

### Can a host change its price after I pay?

No. Each payment is priced by the offer version in force at its own height, and a new version takes effect only 10 market blocks after it is written.

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

Each chain's coin is its own. Nexus coin and market coin aren't worth the same, so the web app shows each chain's burn separately.

### Does `hearth sell --burn` burn every time it runs?

No. It burns only what the burned total is missing, up to the amount. Once the total is reached, by anyone, it burns nothing.

## Blocks

### How do I check that a host makes my chain's blocks?

On the **Hosts** tab and on your chain's page, the web app looks for a block paying the host's `blocks` address within max(24 hours, 10 target block times). Open it with `?node=<your node's read URL>` to read the blocks from your own node. See [Blocks and mining](mining.md#how-a-buyer-checks-the-blocks).

### Why does a check say "unknown"?

The page reads back at most 1000 blocks. On a chain fast enough to have more in the window, a check that finds no block from the host before that cap says *unknown*, not that the host is not serving.

### Can a host fake its blocks?

Not on your own node. Each block proves its work and names who it credits; your node checks both. Elsewhere: if your app pins its first block or a checkpoint, every value is checked against a block whose headers link back to it. One node alone can still serve old or made-up data; with two or more nodes run by different operators, a value is shown only from a block at least two of them report holding (in lattice-sdk, pass `connect` your app's `genesis` or a `checkpoint`, and a list of nodes). Your own node is the simplest check.

### What if a host stops making blocks?

Don't renew. That is the only recourse.
