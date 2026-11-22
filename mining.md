---
title: Mining
---

{% include nav.html %}

# Mining

This page explains how a host can claim to mine the chains it sells, and how a buyer checks that claim on their own node.

## Why mining needs hosting

A chain's blocks come from whoever mines it or commits it on its parent. Hosting doesn't make blocks.

A Lattice miner merged-mines the chains its node hosts: the node carries each hosted chain's next block in its templates for the parent. A host's node already hosts the chains it sells, so a host that runs a miner can mine them too. That is why mining fits in a hosting offer, as an optional extra.

## The claim

A host that mines can say so in its offer:

```
npx @hearthnet/hearth sell --price 100/day --mine 1TH/s
```

`--mine` takes hashes per second: `1TH/s`, `500GH/s`, or a plain whole number. Units run from `kH/s` to `EH/s`. With an offer file, put `"mines"` in the file instead.

It adds a last line `mines <hashes per second>` to the offer. An offer without it reads exactly as before: the same flags without `--mine` name the same offer, with the same id.

## The miner configuration

The seller runs no miner. With `--mine`, it prints the miner entries that credit the seller's address on the chains it sells. It prints them again whenever the chains it owes change.

Add the entries to the `recipients` of your miner's `<root>/miner.json`, beside your own:

```json
{ "recipients": { "Nexus": "<your address>", "Nexus/testnet/app-1": "<seller address>" } }
```

Each entry is one exact chain path. There are no wildcards, so each owed chain needs its own entry. A chain left out of `recipients` burns its block reward and fees. The miner reads the file again before every round.

## How credited hashrate is computed

Every block names its `rewardRecipient` and proves, on average, this many hashes of work:

```
work = 2^256 / (target + 1)
```

The buyer's node checks every block's work, target and recipient.

On the **Hosts** tab, for each chain you bought at a host with this browser's account, and that it still owes, the web app reads the chain's recent blocks from its node. It sums the work of the blocks crediting the host, and divides by the window's length:

```
hashrate = sum of work crediting the host / seconds in the window
```

The window is the last day or the last 100 blocks, whichever has fewer blocks:

- If the last day has fewer than 100 blocks, the window is the last 24 hours, up to now.
- Otherwise it is the newest 100 blocks, from the oldest one's timestamp to the newest one's. The oldest block's work is left out, since it was done before the window opened.

A block with no recipient (its reward burned) counts toward the chain's total only.

The page shows the credited rate next to the host's claim, from its newest offer that makes one: for example `Nexus/testnet/app-1: 950 GH/s credited of 1 TH/s claimed`.

Block timestamps are set by whoever mines. Over a short window the rate is noisy. Read it over time, not from one look.

Open the web app with `?node=<your node's read URL>` to read the blocks from your own node. The block data comes from the node's API, so read it from a node you trust.

## A claim, not a contract

Nothing on the market enforces `mines`. Verdicts never read it, and no payment depends on it. If a host mines less than it claims, the only recourse is not to renew.
