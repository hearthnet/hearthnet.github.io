---
title: How it works
---

{% include nav.html %}

# How it works

This page describes the market chain, its records, how orders are judged, how coin is bought, and what checks catch and what they don't.

## The market chain

The market is a Lattice chain, `Nexus/hearth`.

There is no server. Hosts write offers to the market chain, and buyers write orders that pay hosts. The web app and the seller read the same records and judge them with the same code (`client/verdict.mjs`), so they reach the same verdicts.

Every record is read by CID from a block the node names. If your app pins its first block or a checkpoint, every value is checked against a block whose headers link back to it. One node alone can still serve old or made-up data; with two or more nodes run by different operators, a value is shown only from a block at least two of them report holding (in lattice-sdk, pass `connect` your app's `genesis` or a `checkpoint`, and a list of nodes). A node can also withhold records.

## The policy

The market chain's policy (`policy/src/lib.rs`) only filters keys and coin. It never reads a record's value. Each filter is yes or no for one transaction:

| Key | The policy requires |
|---|---|
| `offer/<host>/<id>/<height>` | The host signs. Written once. `<height>` is within the last 10 blocks. |
| `url/<host>` | The host signs. |
| `order/<host>/<id>/<payer>/<height>/<paid>/<path>` | The payer signs. Written once. `<height>` is within the last 10 blocks. `<paid>` is more than 0, and the host is credited `<paid>` in the same transaction. |

`<host>` and `<payer>` are addresses. `<height>` and `<paid>` are canonical decimals. `<path>` is one or more non-empty segments. Every other key is refused. Transfers, deposits, receipts and withdrawals pass untouched.

What a record's value says is judged off chain, by every reader the same way.

## Records

| Key | Written by | Value |
|---|---|---|
| `offer/<host>/<id>/<height>` | the host, once per height: each is a version of the offer | A hosting offer, or a storage offer (below). |
| `url/<host>` | the host | Its public read URL, optionally followed by a line `proof <url>`. |
| `order/<host>/<id>/<payer>/<height>/<paid>/<path>` | anyone, once; pays the host `<paid>` in the same transaction | The chain's payload, or `renew` to pay for more time, or `bytes <n>` for a storage order. |

### Hosting offers

An offer's value is these lines, in order. Numbers are canonical decimals. The offer's `created` height is the one in its key.

| Line | Meaning |
|---|---|
| `parent <path>` | The chain its chains go under. |
| `price <coin>` | Market coin per period. |
| `period <blocks>` | The period, in market blocks. |
| `until <height>` | The market height the offer ends at. |
| `capacity <n>` | How many chains it sells. |
| `maxBlockSize <bytes>` | The largest `maxBlockSize` a chain's spec may have. |
| `maxStateGrowth <bytes>` | The largest `maxStateGrowth`. |
| `minTargetBlockTime <ms>` | The smallest `targetBlockTime`. |
| `maxNumberOfTransactionsPerBlock <n>` | The largest `maxNumberOfTransactionsPerBlock`. |
| `modules any` or `modules <cid>,<cid>,…` | The policy modules it runs. |
| `blocks <address>` | Optional. The address the host's blocks pay: its sponsor's key. |
| `sponsor <url>` | Optional. The host's fee and storage sponsor (lattice-sponsor), `https` or loopback. |
| `mines <hashes per second>` | Optional, read from old offers only. A mining claim; nothing writes it any more. See [Blocks and mining](mining.md#old-mining-claims). |

An offer whose value is not exactly these lines counts as no offer. `price`, `period` and `capacity` can't be 0. Older offers that end with `blocks mine` or `blocks commit` still parse, without a `blocks` address.

### Offer versions

Each `offer/<host>/<id>/<height>` record is a version of the offer `<host>/<id>`. A version takes effect 10 market blocks after the height it is written at. Since a key may name any of the last 10 heights, a host that sees an order could still write a version keyed below it, but never one in force at it.

An order goes by the newest version in force at its own height, so a host can change its terms only for orders yet to come. Each payment, order or renewal, is priced by the version in force at its height. An order where no version is in force yet is void.

A version whose `until` is its own height closes the offer (`hearth stop`). An offer ends at the latest `until` any version names, so a close never cuts short time paid before it.

`hearth sell` writes an offer to sell for 30 days. Nothing renews it: the host runs `hearth sell` again to write a new version.

### URL records

`url/<host>` holds the host's public read URL. A second line `proof <url>` says where the host's domain lists its sellers. Without it, that is `<read URL's origin>/.well-known/lattice.json`. The proof must be `https`, on the read URL's hostname, at the path `/.well-known/lattice.json`. The file is:

```json
{ "sellers": ["<address>", "..."] }
```

Readers trust a host only when its domain lists its address, as Nostr's NIP-05 does for names. A host with no URL record is *key only*: it names no domain, so it impersonates no one. A host whose URL record fails the check is hidden.

### Orders

An order's value is the chain's payload: one JSON object with exactly the keys `spec`, `premineTo`, `modules` and `genesisTransactions`.

- `spec` is a chain spec's nine fields, in range, with integers only.
- `premineTo` names an account exactly when the spec has a premine, and is `null` otherwise.
- `modules` maps each policy module's CID to its bytes, in base64.
- `genesisTransactions` is a list of signed transactions.

The payload is strict JSON: no duplicate keys, and every number a plain integer. One payload has one reading.

An order whose value is `renew` pays for more time on a chain already ordered.

### Storage offers and orders

A storage offer is an `offer/` record whose value is these lines:

| Line | Meaning |
|---|---|
| `kind storage` | Marks a storage offer. |
| `sponsor <https url>` | The host's storage sponsor. |
| `bytesPerCoin <n>` | Bytes kept for good per market coin. |
| `capacity <bytes>` | Bytes it sells in all. |
| `until <height>` | When it ends. |

A storage order is an `order/` record at a storage offer. Its path is the file's root CID, and its value is `bytes <n>`, the size the sponsor counted. It pays the host `ceil(n / bytesPerCoin)`. A buyer can order a file at several hosts in one transaction.

A storage offer is trusted only when the host's domain lists its address, and its sponsor's `GET /terms` has `payTo` set to the host's address and the offer's `bytesPerCoin` as its rate for the market.

## Settlement and verdicts

An order is judged once it is 16 market blocks old: the 10-block window for its height, plus six confirmations. By then no order or offer can land ahead of it. Until then it is *pending*.

Settled orders are judged oldest first, from the order's own bytes and the offer:

| Verdict | Meaning |
|---|---|
| **void** | The buyer broke the offer's terms: no such offer, no version in force at its height yet, a close in force there, created outside the offer's time, paid less than one period, a path not directly under the offer's parent, a payload that does not parse, a spec over the offer's limits, a module the offer does not run, or modules that are not exactly the spec's. |
| **full** | The offer had already sold its chains. |
| **taken** | The path holds another genesis, or an earlier order at this host named the path with another payload. |
| **owed** | The host serves the chain from 16 blocks after the order until its paid time runs out or the offer ends, whichever is first. |

Full and taken are not the buyer's fault, and the host can cause both. Readers count them against the host as *paid, not served*.

**Paid time.** A chain's orders and renewals at one offer, oldest first, each add `floor(paid / price)` periods, at the price and period in force at the payment's own height. Each adds them from where paid time ends, or from its own height if paid time had run out.

**Storage orders** are judged apart:

| Verdict | Meaning |
|---|---|
| **void** | Created outside the offer's time, a path that is not one root CID, a value that is not `bytes <n>`, or paid for fewer bytes than it names. |
| **full** | The offer's earlier owed orders already fill its capacity. |
| **owed** | Kept for good. A storage sponsor never unpins. |

A root ordered again at the same offer counts its bytes once.

## What a host does

The seller (`host/sell-hosting.mjs`, which `hearth sell` runs) is a client of a node's operator API. Each pass it reads its own orders and judges them. For each owed chain not yet on its parent, it creates the chain once from the payload. It lists every owed chain in its fragment of the node's configuration, `hostmarket`. A chain whose paid time ran out is left out, and the node stops hosting it unless another layer lists it.

The seller also makes zero-work blocks, about one hash each, on the market and on every chain it hosts for its orders, one per target block time, paid to its sponsor's key. Anyone else may mine those chains too. See [Blocks and mining](mining.md).

Beside the node the seller runs lattice-sponsor, which pays fees for visitors who pay with proof-of-work and sells permanent pins for work. Its float is the block rewards. See [Sponsor](sell.md#sponsor).

## Buying coin

Hosts are paid in the market chain's own coin. It is bought with the parent chain's coin through Lattice's own exchange, in three steps:

1. **Deposit.** A seller locks market coin on the market chain for a price in parent coin. It stays locked until someone pays. There is no cancel.
2. **Receipt.** A buyer pays that price to the seller on the parent chain, naming the deposit.
3. **Withdrawal.** The buyer takes the locked coin on the market chain.

Lattice checks both sides, so nobody holds the payment in between, and there is no exchange to trust. Where a fee sponsor pays out parent coin, it can also be earned through the sponsor's payouts.

## The trust model

**What checks catch.** If your app pins its first block or a checkpoint, every value is checked against a block whose headers link back to it. One node alone can still serve old or made-up data; with two or more nodes run by different operators, a value is shown only from a block at least two of them report holding (in lattice-sdk, pass `connect` your app's `genesis` or a `checkpoint`, and a list of nodes). With that, or on your own node, a host can't pass off:

- its records, or anyone else's: every record is checked against its CID;
- a verdict: every reader judges the same bytes the same way;
- the genesis of a chain it serves: the page checks it against the payload the buyer paid for;
- the work in a chain's blocks, or who they credit: your node checks every block, so *Makes its chains' blocks* holds when read from your own node;
- its burns: they are balances on chain.

**A host can:**

- take an order and not serve it. It shows as *paid, not served* only when the offer was full or the path was taken; otherwise it shows in *Serving now*;
- serve a chain now and stop later. *Serving now* measures availability now, not past uptime;
- relay a chain instead of storing it;
- pass *Listed by its sponsor* without hosting anything: its own sponsor answers for its own address;
- buy and serve its own chains to raise its count;
- with most of the market chain's mining, reorder the market past the 16-block settlement.

**A node you read from can** withhold records, serve an old tip, or misreport balances and block data, which come from its API. Read from your own node with `?node=`.

**A buyer does not get:** the chain's mining rewards (they go to whoever makes its blocks: the host's sponsor, or other miners), blocks after `hearth stop --now` (`hearth stop` keeps paid chains hosted and their blocks made until their time ends), protection from a heavier rival chain at the same path, a refund, proof the host stores rather than relays, or a record of past uptime.
