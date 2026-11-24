---
title: Blocks and mining
---

{% include nav.html %}

# Blocks and mining

This page explains who makes a hosted chain's blocks, where the rewards go, and how a buyer checks that a host mines what it claims.

## Hosts mine every chain they host

Every seller mines every chain its node hosts, all the time, with this machine's processor. There is no flag; the seller always does it.

Each round it asks its node for a template that carries every hosted chain, searches nonces, and submits each hit. One hash makes a block on every chain whose target it clears, so every chain gets all of the seller's hashes. Each chain's difficulty (ASERT) settles where the miners on it make its blocks on time.

The seller measures its hash rate each minute and logs it (`miner: <n> hashes per second`). That is the rate its offer claims: the offer's `hashrate` line is `--hashrate` when you give it, else the rate measured over the miner's first minute. The offer waits for that first minute.

## Rewards go to the sponsor's key

Every block the seller makes pays its sponsor's key (`<root>/hearth/sponsor.json`), on every chain in the template, never the seller's key. That key funds the sponsor: it pays the write fees of visitors who pay with proof-of-work instead of coin.

To move rewards above a float to the seller's address, run `hearth sweep` with the seller stopped. See [Rewards](sell.md#rewards-hearth-sweep).

Each offer names the address its blocks pay, as its line `blocks <address>`. That is how buyers find a host's work.

## Anyone can mine

Anyone else may mine a hosted chain too, for its block reward. A buyer who wants the rewards of their own chain can mine it.

## How a buyer checks the hash rate

An offer's `hashrate` is only a claim. Each block proves its work and names who it credits, so the web app measures what a host's blocks delivered (`client/mining.mjs`): it sums the expected hashes behind each block (2^256 ÷ (target + 1)) credited to the offer's `blocks` address, over the window.

- On **Hosts** and on **Launch**, *delivered* is the lowest hash rate the host's `blocks` address was credited with over the last day (at most 200 blocks), across up to three chains it is paid to host now. Each chain is read by CID from a node serving its genesis.
- On your app's page, under **Use it**, each host you ordered from shows the hash rate it was credited with on your chain over the last day, beside what it claims.

A host that delivers less than it claims shows it there; don't renew with it.

The blocks come from whichever node the page finds serving the chain's genesis. A node checks every block's work and recipient before it serves it; to be sure of them, run your own node that hosts the chain and read its blocks there.
