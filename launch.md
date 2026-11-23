---
title: Launch a chain
---

{% include nav.html %}

# Launch a chain

This page shows how to launch your app's chain from the Hearth web app, keep it running, and check its hosts.

You don't run a node. You buy market coin, pay one or more hosts for some periods, and each host builds your chain's genesis from your spec, stores it, serves it at its URL and makes its blocks.

Each host makes your chain's blocks: zero-work blocks, about one per target block time, so the chain lives without miners. Each host also runs a sponsor, so your visitors can write with a few seconds of their device's work instead of coin. Anyone else may mine your chain too. See [Blocks and mining](mining.md).

## The web app

The web app is one page with five tabs: **Launch a chain**, **Buy coin**, **Hosts**, **Storage** and **Your chains**.

Your account is a key kept in this browser. The app makes it the first time you buy or launch. It holds your coin and signs your orders. Download it from your account page (`#/wallet`) to keep it: anyone with the file can spend from it. You can also load a key file there, from `lattice key generate` or one downloaded earlier.

The page takes these settings in its URL:

| Parameter | What it sets |
|---|---|
| `?node=` | The node it reads from and sends to. Point it at your own node's read URL. |
| `?market=` | The market chain. Default `Nexus/hearth`. |
| `?sponsor=` | A fee sponsor that pays out parent coin for work. None by default; **Earn with this device** shows only when one is given. |
| `?blockSeconds=` | Seconds per market block, used only to show durations. Default `60`. |

## 1. Buy market coin

Hosts are paid in the market chain's own coin, Hearth coin. Hosts earn it, and sell what they don't spend with `hearth sell-coin` (see [Sell your coin](sell.md#sell-your-coin-hearth-sell-coin)). On **Buy coin**, the page lists coin that sellers have locked on the market chain, each lot for a price in the parent chain's coin (Nexus coin). The cheapest per coin comes first.

Press **Buy** on a lot. The page then:

1. pays the seller on the parent chain (a receipt);
2. waits for the payment to be in a block, and sends the same payment again if it is not in a block after two minutes;
3. takes the coin on the market chain (a withdrawal);
4. waits for the coin to land.

The fee for taking the coin comes out of it, so you get the lot less one coin at the minimum fee. If you reload, the purchase picks up where it left off.

You need parent coin to pay. Anyone holding some can send it to your address. Where a fee sponsor pays out parent coin, you can also earn it with **Earn with this device**: your browser does work for the fee sponsor, which pays your address each time your earnings reach its payout.

You can also sell market coin from the same tab: lock an amount for a price in parent coin. It stays locked until someone pays. There is no cancel.

See [How it works](how-it-works.md#buying-coin) for the deposit, receipt and withdrawal.

## 2. Describe your chain

On **Launch a chain**, under **1. Your chain**:

| Field | What it does |
|---|---|
| Name | Lowercase letters, digits and dashes, up to 40, starting with a letter or digit. The chain's path is the offer's parent plus this name, for example `Nexus/my-app`. |
| Size | A preset. **Small app**: 30 s blocks, 20 kB blocks, 10 kB state growth, 100 transactions per block (what a host sells by default). **Busy app (needs a seller with bigger limits)**: 5 s blocks, 1,000 kB blocks, 1 MB state growth, 1,000 transactions per block. No seller's default offer fits it; it fits only offers whose limits are at least this big. |
| Coin you start with | Paid to your account in the genesis. Writes pay fees from it, unless a sponsor pays them. Rounded up to a multiple of 100, the block reward. Default 21,000. |
| Who can write | A rules preset: **Anyone can write** (any key); **Each person edits only their own keys** (every key ends with its writer's address); **Append-only (write once)** (anyone adds a key, nobody changes it after); or **Upload your own .wasm** (your own policy modules). |

Each block pays 100 coin on your chain to whoever makes it. A host's blocks pay its sponsor, which spends it on your visitors' write fees.

The rules are fixed at launch: they are part of the genesis.

## 3. Choose hosts

Under **2. Hosts** the page lists the offers that take your chain, one per host (its cheapest with room). An offer is listed when:

- its host is **Verified** or **Key only** (below);
- it ends more than 16 market blocks from now;
- your chain's size is within its limits;
- it runs your policy modules, or any.

The cheapest per block comes first. You can tick more than one host; each is paid separately. A full offer is shown but can't be chosen, and nor can an offer under another parent than the first you chose.

| Column | What it shows |
|---|---|
| Host | The domain of a verified host, or the start and end of a key-only host's address, marked *Key only*. *Says it makes blocks* and *Says it sponsors writes* when its offer names a `blocks` address and a `sponsor`. |
| Price | Market coin per period. |
| Room | Chains the offer can still sell, of how many. |
| Record | Chains the host has sold, and how many it was *paid, not served*. |

**Verified or key only.** A verified host's domain lists its address at `/.well-known/lattice.json`. A key-only host has written no URL at all: it claims no name, so you judge it by its record. A host whose URL record fails the check is hidden. Anyone could name someone else's URL, so the page never offers it.

A key-only host names no URL, so the page can't probe it or show you where to read your chain. Read your chain from your own node.

**Burns.** The **Hosts** tab shows the coin burned for each host, on each chain from Nexus down to the market, chain by chain. A new host with no record shows its burns in place of a record. A burn costs whoever made it. It says nothing about how a host serves. See [Burns](burns.md).

## 4. Pay and launch

Set **Periods** (default 7). The cost is each offer's price times the periods, summed over the hosts you chose. You need that plus the fee.

Press **Launch**. The page reads the market again first, and stops if an offer you chose is gone, has closed, is full, or has new terms since you chose it: check your hosts and launch again. It refuses a path that is already a chain on the parent, or that someone else has ordered at one of these hosts. It also refuses to pay twice: if your account already has an order for this path at one of these hosts (a payment that landed), it says so. Then it writes one transaction that orders your chain at every chosen host and pays each one, priced by the offer version in force at the market's height.

Your chain's page then shows three steps:

1. **Paid**: the order is on the market chain.
2. **Settled**: 16 market blocks after the order. From then on every reader judges it the same way.
3. **Launched**: the host serves a chain at your path whose genesis is the one you paid for, checked from your browser.

Paid time counts from the order's own block. The host owes the chain from the order's block plus 16 until the paid time runs out or the offer ends, whichever is first.

If the order is void, or the host was full, or the path was taken, the page says why. See [Verdicts](how-it-works.md#settlement-and-verdicts).

## 5. Use it

Once launched, **Use it** shows where to read and write (the host's read URL), the chain's height and latest block, and whether each host you ordered from made a block of it lately (see [Blocks and mining](mining.md#how-a-buyer-checks-the-blocks)).

**Test write** writes one key to your chain and reads it back, checking what it reads against the chain. When your hosts name sponsors, they pay the write fee: the writer's device does a few seconds of work instead. Otherwise the writer needs coin on your chain.

The code to paste into your own page appears here once a release of the SDK is tagged. Test write works now.

## 6. Renew

Anyone can pay for more time. On your chain's page, under **Keep it running**, set the periods and press **Pay for more**. This writes a `renew` order that pays the host, priced by the offer version in force. If the offer's price or period changed since the page loaded, it stops and asks you to reload.

Offers sell for 30 days at a time, and a host extends one by writing it again. Time past the offer's end is never owed. The page warns you when the offer ends before your paid time would. A closed offer sells no more time.

There are no refunds. If a host serves badly, don't renew.

When paid time ends, a host may stop hosting your chain and drop its data. Anyone still hosting it keeps it.

## Check a host

By default the page reads from one public node, `https://rpc.hearthnet.io`. If your app pins its first block or a checkpoint, every value is checked against a block whose headers link back to it. One node alone can still serve old or made-up data; with two or more nodes run by different operators, a value is shown only from a block at least two of them report holding (in lattice-sdk, pass `connect` your app's `genesis` or a `checkpoint`, and a list of nodes). One node can also withhold records. Burns and block reads come from the node's API.

To trust only yourself, run a node that hosts the market and open the page with:

```
?node=<your node's read URL>
```

Then the market, burns and your chains' blocks all come from your node. The **Serving now** probes still go to each host's URL, and the **Listed by its sponsor** check to its sponsor's URL, since that is what they test. That sponsor sees your browser's IP address.

On **Hosts**, each host shows:

| Field | What it means |
|---|---|
| Verified / Key only / Not verified | Whether its domain lists its address. |
| Burned | Coin burned for it, per chain, as read from the node. |
| Serving now | Of the chains it owes, how many it serves with the right genesis, checked from your browser. |
| Listed by its sponsor | The sponsor its offer names (an `https` URL, or loopback) answers `GET /terms` bound to this host: the sponsor's own address is the offer's `blocks` address, and it pays this host. This checks the host is up and consistent, not that it hosts anything. |
| Makes its chains' blocks | Of up to three chains it is paid to run, how many have a block paying its `blocks` address within max(24 hours, 10 target block times). *Unknown* when the chain made more blocks in that window than the 1000 the page reads back. |
| Paid, not served | Orders it took after its offer was full, or for a path it had given to another chain. |
| Void | Orders whose buyer broke the offer's terms. |

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
  [--node <read URL>] [--market Nexus/hearth] [--offer <host>/<id> ...] [--min 1]
```

`--min` is how many verified offers it needs. It refuses to run with fewer.
