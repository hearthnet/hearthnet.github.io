---
title: Blocks and mining
---

{% include nav.html %}

# Blocks and mining

This page explains who makes a hosted chain's blocks, where the rewards go, and how a buyer checks that a host makes them.

## Hosts make zero-work blocks

Every seller makes blocks on the market and on every chain it hosts for its orders. Each is a zero-work block: about one hash, made with the seller's processor. There is no flag; the seller always does it.

The seller paces each chain: a chain is due once its tip is one target block time (`targetBlockTime`, from the chain's spec) old, from anyone's block. Then the seller asks its node for a template, searches nonces up to that chain's target, and submits the hit. So a chain gets about one block per target block time with no miner at all. A chain that other miners or other hosts keep moving needs nothing from the seller. A chain the node is still syncing is never paced, and after a miss a chain waits its whole target block time.

**Early blocks.** A template carries every chain the node hosts, and one hash makes a block on each chain whose target it clears. The seller skips a hash that would also clear a harder chain not yet due. It can't keep a hash off an easier chain not yet due, since any hash that clears the due chain clears that one too. So an easier chain can get some early blocks today. ASERT difficulty adjustment hardens it back toward its target block time. A per-chain `exclude` in template requests, in a coming Lattice node release, will fix this.

## Rewards go to the sponsor's key

Every block the seller makes pays its sponsor's key (`<root>/hearth/sponsor.json`), on every chain in the template, never the seller's key. That key funds the sponsor: it pays the write fees of visitors who pay with proof-of-work instead of coin.

To move rewards above a float to the seller's address, run `hearth sweep` with the seller stopped. See [Rewards](sell.md#rewards-hearth-sweep).

Each hosting offer names the address its blocks pay, as its line `blocks <address>`.

## Anyone can mine

Anyone else may mine a hosted chain too, for its block reward. A Lattice miner merged-mines the chains its node hosts. A buyer who wants the rewards of their own chain can mine it.

## How a buyer checks the blocks

The web app checks, from the buyer's browser, that a host makes its chains' blocks (`client/service.mjs`):

- **serving**: the offer's `blocks` address is credited on the chain within the last max(24 hours, 10 target block times). The blocks are read from the node the page reads from (`?node=`), by CID along their parent links, from the tip back. The read stops at 1000 blocks: on a chain fast enough to have more in that window, a check that finds no credit before the cap says *unknown*, not that the host is not serving.
- **live**, separately: the chain has a block, from anyone, within its last three target block times.

Every clock comparison allows two minutes of slack, since block timestamps are the miner's clock.

The **Hosts** tab shows *Makes its chains' blocks* for up to three chains a host is paid to run. On your own chain's page, under **Use it**, the app says for each host you ordered from whether it made a block in the window, and whether the chain has a recent block.

Open the web app with `?node=<your node's read URL>` to read the blocks from your own node. Your node checks every block's work and recipient.

## Old mining claims

Older offers could claim a hashrate with a line `mines <hashes per second>`. That line still parses, and the **Hosts** tab still shows it, but nothing writes it any more. For a host with such a claim, on chains you bought there, the page shows the hashrate the chain's blocks credit the host over the last day or 100 blocks, beside the claim. A claim was never enforced: verdicts never read it.
