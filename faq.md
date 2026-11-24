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

To earn Hearth coin, the market's coin, which pays for hosting from any seller. A host earns it two ways: buyers' orders pay its seller key, and the market blocks it mines pay its sponsor's key (1,000 coin a block). It spends the coin on hosting, or sells it on the chains' own exchange with `hearth sell-coin`. See [Why host](sell.md#why-host).

### Where do I get Hearth coin?

From hosts. They earn it and sell it on the chains' own exchange, and you buy it on the web app's **Buy coin** page with parent-chain coin. The chains check both sides, so nobody holds your payment in between. See [Buy market coin](launch.md#1-buy-market-coin).

### Can a host cancel coin it put on sale?

No. There is no cancel or refund for a deposit. It stays on sale until someone buys it; the host can only buy it back like any buyer. See [Sell your coin](sell.md#sell-your-coin-hearth-sell-coin).

### Is this on mainnet?

The current market runs on testnet, at `Nexus/testnet/hearth`. The mainnet market, `Nexus/hearth`, still runs the previous market until it is relaunched.

### Can I get a refund?

No. There are no refunds. Your recourse is renewal: if a host serves badly, don't renew, and order from another host. Check a host's record on the **Hosts** tab before you pay.

### What if the host doesn't launch my chain?

The page shows it: your app's page finds no node serving your chain, and the host's *delivered* hash rate on it stays at nothing. The order is still *owed*, and you don't get the coin back. Don't renew with that host. Ordering from two hosts at once keeps your app up if one fails.

### Does the host make my chain's blocks?

Yes. Every host mines every chain it hosts, all the time, with all of its hash rate: the rate its offer claims. Anyone else may mine it too, you included. See [Blocks and mining](mining.md).

### Who gets my chain's mining rewards?

Whoever makes its blocks. A host's blocks pay its sponsor's key, which spends them on the write fees of visitors who pay with work.

### Do my visitors need coin to write?

Not if you add a fee sponsor to your app. Sponsoring is a separate service, not part of a host's offer: add a sponsor's URL as `sponsors` in your app's code. The sponsor pays the write fee, and the visitor's device does a few seconds of proof-of-work instead. Without a sponsor, the writer needs coin on your chain; your account holds the coin your app started with.

### Can a host change its price after I pay?

Not for what you've paid. Each order asserts the offer unchanged, so it lands only on the terms you saw; if the host changes them first, the order is refused and nothing is paid. Paid time keeps the terms it landed on. A renewal pays the offer's terms at the time you renew.

### Can someone take my chain's name?

Before you pay, the web app refuses a path that is already a chain under its parent, or that someone else has paid hosts for. The market records only what each host is paid to run; it doesn't decide which genesis holds a path. A buyer may also commit the genesis to the parent chain to secure the name; the web app doesn't do that yet. Nothing protects a chain from a heavier rival chain at the same path.

### How long until my chain is up?

An order is judged as soon as it is in a block (the market makes about one a minute). The host starts your chain from the genesis your order carries on its next pass (every 30 seconds by default). Paid time counts from the order's own block.

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

### How do I check that a host mines what it claims?

On the **Hosts** tab, each offer shows the hash rate it *claims* and what its blocks *delivered*: the lowest hash rate its `blocks` address was credited with over the last day (at most 200 blocks), across up to three chains it is paid to host. Your app's page shows the same for each of your hosts on your own chain. See [Blocks and mining](mining.md#how-a-buyer-checks-the-hash-rate).

### Can a host fake its blocks?

Not on your own node. Each block proves its work and names who it credits; your node checks both. Elsewhere: if your app pins its first block or a checkpoint, every value is checked against a block whose headers link back to it. One node alone can still serve old or made-up data; with two or more nodes run by different operators, a value is shown only from a block at least two of them report holding (in lattice-sdk, pass `connect` your app's `genesis` or a `checkpoint`, and a list of nodes). Your own node is the simplest check.

### What if a host mines less than it claims, or stops?

Don't renew. That is the only recourse.
