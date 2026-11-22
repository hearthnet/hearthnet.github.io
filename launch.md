---
title: Launch a chain
---

{% include nav.html %}

# Launch a chain

This page shows how to launch your app's chain from the Hearth web app, keep it running, and check its host.

You don't run a node. You buy market coin, pay a host for some periods, and the host builds your chain's genesis from your spec, stores it and serves it at its URL.

The host makes no blocks. Your chain's blocks come from whoever mines it or commits it on its parent, for its block reward. That can be the host's own miner (see [Mining](mining.md)), another miner, or you.

## The web app

The web app is one page with five tabs: **Launch a chain**, **Buy coin**, **Hosts**, **Storage** and **Your chains**.

Your account is a key kept in this browser. The app makes it the first time you buy or launch. It holds your coin and signs your orders. Download it from your account page (`#/wallet`) to keep it: anyone with the file can spend from it. You can also load a key file there, from `lattice key generate` or one downloaded earlier.

The page takes these settings in its URL:

| Parameter | What it sets |
|---|---|
| `?node=` | The node it reads from and sends to. Point it at your own node's read URL. |
| `?market=` | The market chain. Default `Nexus/testnet/lodger-3129cc`. |
| `?sponsor=` | The fee sponsor used to earn parent coin on testnet. |
| `?blockSeconds=` | Seconds per market block, used only to show durations. Default `60`. |

## 1. Buy market coin

Hosts are paid in the market chain's own coin. On **Buy coin**, the page lists coin that sellers have locked on the market chain, each lot for a price in the parent chain's coin (`Nexus/testnet` coin). The cheapest per coin comes first.

Press **Buy** on a lot. The page then:

1. pays the seller on the parent chain (a receipt);
2. waits for the payment to be in a block, and sends the same payment again if it is not in a block after two minutes;
3. takes the coin on the market chain (a withdrawal);
4. waits for the coin to land.

The fee for taking the coin comes out of it, so you get the lot less one coin at the minimum fee. If you reload, the purchase picks up where it left off.

You need parent coin to pay. Anyone holding some can send it to your address. On testnet you can also earn it with **Earn with this device**: your browser does work for the fee sponsor, which pays your address each time your earnings reach its payout.

You can also sell market coin from the same tab: lock an amount for a price in parent coin. It stays locked until someone pays. There is no cancel.

See [How it works](how-it-works.md#buying-coin) for the deposit, receipt and withdrawal.

## 2. Describe your chain

On **Launch a chain**:

| Field | What it does |
|---|---|
| Name | Lowercase letters, digits and dashes, up to 40, starting with a letter or digit. The chain's path is the offer's parent plus this name, for example `Nexus/testnet/my-app`. |
| Size | **Small app**: 10 s blocks, 100 kB blocks, 100 kB state growth, 100 transactions per block. **Busy app**: 5 s blocks, 1,000 kB blocks, 1 MB state growth, 1,000 transactions per block. |
| Coin you start with | Paid to your account in the genesis. It pays the fees for writes on your chain. Rounded up to a multiple of 100, the block reward. Default 21,000. |
| Rules (policy modules) | WASM policy modules for your chain. Leave it empty for a chain anyone can write any key to. |

The rules are fixed at launch: they are part of the genesis.

## 3. Choose a host

The page lists the offers that take your chain. An offer is listed when:

- its host is **Verified** or **Key only** (below);
- it ends more than 16 market blocks from now;
- your chain's size is within its limits;
- it runs your policy modules, or any.

The cheapest per block comes first. A full offer is shown but can't be chosen.

| Column | What it shows |
|---|---|
| Host | The domain of a verified host, or the start and end of a key-only host's address, marked *Key only*. |
| Price | Market coin per period. |
| Room | Chains the offer can still sell, of how many. |
| Record | Chains the host has sold, and how many it was *paid, not served*. |

**Verified or key only.** A verified host's domain lists its address at `/.well-known/lattice.json`. A key-only host has written no URL at all: it claims no name, so you judge it by its record. A host whose URL record fails the check is hidden. Anyone could name someone else's URL, so the page never offers it.

A key-only host names no URL, so the page can't probe it or show you where to read your chain. Read your chain from your own node.

**Burns.** The **Hosts** tab shows the coin burned for each host, on each chain from Nexus down to the market, chain by chain. A new host with no record shows its burns in place of a record. A burn costs whoever made it. It says nothing about how a host serves. See [Burns](burns.md).

**Mining claims.** An offer may claim a hashrate on the chains it sells. The **Hosts** tab shows the claim, and on your own chains the hashrate their blocks credit the host. See [Mining](mining.md).

## 4. Pay and launch

Set **Periods** (default 7). The cost is the offer's price times the periods. You need that plus the fee.

Press **Launch**. The page checks the name first: it refuses a path that is already a chain on the parent, or that someone else has ordered at this host. Then it writes your order and pays the host in one transaction.

Your chain's page then shows three steps:

1. **Paid**: the order is on the market chain.
2. **Settled**: 16 market blocks after the order. From then on every reader judges it the same way.
3. **Launched**: the host serves a chain at your path whose genesis is the one you paid for, checked from your browser.

Paid time counts from the order's own block. The host owes the chain from the order's block plus 16 until the paid time runs out or the offer ends, whichever is first.

If the order is void, or the host was full, or the path was taken, the page says why. See [Verdicts](how-it-works.md#settlement-and-verdicts).

Once launched, the page shows where to read and write: the host's URL, with `?chainPath=<your path>`.

## 5. Renew

Anyone can pay for more time. On your chain's page, under **Keep it running**, set the periods and press **Pay for more**. This writes a `renew` order that pays the host.

Time past the offer's end is never owed. The page warns you when the offer ends before your paid time would.

There are no refunds. If a host serves badly, don't renew.

## Check hosts on your own node

By default the page reads from a public testnet node. That node can't forge a record: every market record is checked against its CID. But it can withhold records or serve an old tip. Burns and credited hashrate come from the node's API, not from records checked by CID.

To trust only yourself, run a node that hosts the market and open the page with:

```
?node=<your node's read URL>
```

Then the market, burns and your chains' blocks all come from your node. The **Serving now** probes still go to each host's URL, since that is what they test.

On **Hosts**, each host shows:

| Field | What it means |
|---|---|
| Verified / Key only / Not verified | Whether its domain lists its address. |
| Burned | Coin burned for it, per chain, as read from the node. |
| Serving now | Of the chains it owes, how many it serves with the right genesis, checked from your browser. |
| Paid, not served | Orders it took after its offer was full, or for a path it had given to another chain. |
| Void | Orders whose buyer broke the offer's terms. |
| Mines | Each offer's mining claim, if any. |
| Your chains | On chains you bought there, the hashrate their blocks credit the host, against its claim. |

"Serving now" measures availability now. It does not prove the host stores the chain, and it is not a record of past uptime.

## Storage offers

Some hosts also sell storage: they keep files for good, paid once in market coin. The **Storage** tab lists these offers:

| Column | What it shows |
|---|---|
| Host | Its domain, and whether it is verified. |
| Price | Market coin per MiB, for good. |
| Room | MiB left of the offer's capacity. |
| Files kept | Files ordered there and owed. |
| Checked now | Up to five of its newest files, and how many its node holds itself. |

Storage offers need a verified domain. The host's storage sponsor must also be paid to the host's address, at the offer's rate.

To store a file, upload it to each host's sponsor, then order it at every host in one transaction. Each order pays its host. Then claim each payment at its sponsor, which pins the file. Nothing is ever unpinned. Store a file at several hosts so it outlives any one of them.

The source repo has `tools/store.mjs`, which does all of this for one file of up to 1 MiB, at every verified storage offer:

```
node tools/store.mjs --key buyer.json --file photo.png \
  [--node <read URL>] [--market Nexus/testnet/lodger-3129cc] [--offer <host>/<id> ...] [--min 1]
```

`--min` is how many verified offers it needs. It refuses to run with fewer.
