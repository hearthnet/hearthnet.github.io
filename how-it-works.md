---
title: How it works
---

{% include nav.html %}

# How it works

This page describes the market chain, its records, how orders are judged, how coin is bought, and what checks catch and what they don't.

## The market chain

The market is a chain. The current market runs on testnet at `Nexus/testnet/hearth`. The mainnet market, `Nexus/hearth`, still runs the previous market until it is relaunched.

There is no server. Hosts write offers to the market chain, and buyers write orders that pay hosts. The web app and the seller read the same records and judge them with the same code (`client/verdict.mjs`), so they reach the same verdicts.

Every record is read by CID from a block the node names. If your app pins its first block or a checkpoint, every value is checked against a block whose headers link back to it. One node alone can still serve old or made-up data; with two or more nodes run by different operators, a value is shown only from a block at least two of them report holding (in lattice-sdk, pass `connect` your app's `genesis` or a `checkpoint`, and a list of nodes). A node can also withhold records.

## The policy

The market chain's policy (`policy/src/lib.rs`) only filters keys and coin. It never reads a record's value. Each filter is yes or no for one transaction:

| Key | The policy requires |
|---|---|
| `offer/<host>/<id>` | The host signs to write, change or delete it. Anyone may assert it unchanged (the same value before and after), which the chain refuses unless the offer still holds exactly those bytes. |
| `order/<host>/<id>/<payer>/<n>/<paid>/<path>` | The payer signs. Written once. `<paid>` is more than 0, and the host is credited `<paid>` in the same transaction, which also asserts `offer/<host>/<id>` unchanged. |

`<host>` and `<payer>` are addresses. `<n>` and `<paid>` are canonical decimals; `<n>` tells one payer's orders apart. `<path>` is one or more non-empty segments. Every other key is refused. Transfers, deposits, receipts and withdrawals pass untouched.

What a record's value says is judged off chain, by every reader the same way.

## Records

| Key | Written by | Value |
|---|---|---|
| `offer/<host>/<id>` | the host: written, changed in place, deleted to close | The offer's five lines (below). |
| `order/<host>/<id>/<payer>/<n>/<paid>/<path>` | the payer, once; pays the host `<paid>` and asserts the offer unchanged in the same transaction | The chain's genesis, or `renew` to pay for more time. |

### Offers

An offer's value is exactly these five lines, in this order. Numbers are canonical decimals.

| Line | Meaning |
|---|---|
| `hashrate <n>` | Hashes per second the host mines with. It mines every chain it hosts with all of it, so each chain gets the full rate. A claim, not a price: buyers check it against the blocks (see [Blocks and mining](mining.md)). |
| `throughput <n>` | Market coin per kB/s of block capacity, per period. A chain's block capacity is `maxBlockSize ÷ targetBlockTime`. |
| `storage <n>` | Market coin per kB of `maxStateGrowth`, per period. |
| `period <n>` | Market blocks a period lasts, at least 1. |
| `blocks <address>` | The address the host's blocks pay. Buyers find its work by it. |

An offer whose value is not exactly these lines counts as no offer.

There is no URL, no limit on chain size, no capacity and no parent. A host hosts any path. Sponsoring writers is a separate service, not part of an offer.

The host changes its offer by writing it again in place, and closes it by deleting it (`hearth stop`). Since every order asserts the offer unchanged, an order lands only on the terms it was priced at: if the host changes or deletes the offer first, the order is refused and nothing is paid.

### Price

One chain costs, per period:

```
ceil(throughput × maxBlockSize ÷ targetBlockTime) + ceil(storage × maxStateGrowth ÷ 1000)
```

with `maxBlockSize` and `maxStateGrowth` in bytes and `targetBlockTime` in milliseconds, all from the chain's spec. A host runs every chain from Nexus down to the one ordered, so an order pays for each of them, Nexus included, each priced by its own spec. The total is at least 1.

For example, at 60 per kB/s and 6 per kB, a chain with 20 kB blocks every 30 s and 10 kB of state growth costs 40 + 60 = 100 a period itself. The chains above it add their own prices, which dominate: under `Nexus/testnet` the whole path comes to about 25,000 a period.

### Orders

An order's value is the chain's genesis as the buyer minted it in the web app: strict JSON with exactly these keys.

| Key | What it holds |
|---|---|
| `genesis` | The genesis block's CID. |
| `volumes` | The genesis block, its state and its spec, as base64 archives by root CID. |
| `modules` | The spec's policy modules, base64 by CID: exactly the spec's. |
| `proofs` | The genesis's proof of work: base64 child-only proofs. A genesis has the easiest target, so this is trivial to make. |

Strict JSON means no duplicate keys, and every number a plain integer. One value has one reading.

An order whose value is `renew` pays for more time on a chain already ordered at that offer.

## Verdicts

An order is judged as soon as it is in a block, at the offer's terms in that block (the ones it asserted). Nothing waits: the verdict is final then.

| Verdict | Meaning |
|---|---|
| **void** | The buyer's fault: the offer's terms don't parse, the genesis does not check out (a volume that does not match its CID, a block that is not a genesis, a spec missing or out of range, modules that are not exactly the spec's), or less than one period paid for the whole path. The host owes nothing, and the coin is not refunded. |
| **owed** | The host runs the chain from the block the order landed in until its paid time runs out. |

A reader that could not yet read the spec of a chain above the ordered one can't price the order, and shows it as *not judged yet* until it can.

A renewal with no owed order before it for the same chain at the same offer is void.

**Paid time.** A chain's owed order and its renewals at one offer, oldest first, each add `floor(paid ÷ price) × period` market blocks, at the price and period of the terms it landed on. Each adds them from where paid time ends, or from its own block if paid time had run out.

The market records only what each seller is paid to run. Which genesis holds a path is up to the chains themselves (the work behind it), not the market. A buyer may also commit the genesis to the parent chain to secure the name; the web app doesn't do that yet.

## What a host does

The seller (`host/sell-hosting.mjs`, which `hearth sell` runs) is a client of a node's operator API. Each pass, at the market's tip, it reads its own orders and judges them. It lists the market and every owed chain, with every chain above it, in its fragment of the node's configuration, `hostmarket`. Then it starts each owed chain from the genesis its order carries: it pins the genesis's volumes and modules on the node, and hands the node the genesis block with its proofs (`POST /child-blocks`). From there it mines the chain on. A chain whose paid time ran out is left out of the fragment, and the node stops hosting it unless another layer lists it.

The seller mines every chain the node hosts, all the time, paid to its sponsor's key. Anyone else may mine those chains too. See [Blocks and mining](mining.md).

Beside the node the seller also runs a sponsor, which pays fees for visitors who pay with proof-of-work and sells pins for work. Its float is the block rewards. The offer doesn't name it. See [Sponsor](sell.md#sponsor).

## Buying coin

Hosts are paid in the market chain's own coin. It is bought with the parent chain's coin through the chains' own exchange, in three steps:

1. **Deposit.** A seller locks market coin on the market chain for a price in parent coin. It stays locked until someone pays. There is no cancel.
2. **Receipt.** A buyer pays that price to the seller on the parent chain, naming the deposit.
3. **Withdrawal.** The buyer takes the locked coin on the market chain.

The chains check both sides, so nobody holds the payment in between, and there is no exchange to trust. Where a fee sponsor pays out parent coin, it can also be earned through the sponsor's payouts.

## The trust model

**What checks catch.** If your app pins its first block or a checkpoint, every value is checked against a block whose headers link back to it. One node alone can still serve old or made-up data; with two or more nodes run by different operators, a value is shown only from a block at least two of them report holding (in lattice-sdk, pass `connect` your app's `genesis` or a `checkpoint`, and a list of nodes). With that, or on your own node, a host can't pass off:

- its records, or anyone else's: every record is checked against its CID;
- a verdict: every reader judges the same bytes the same way;
- your chain's genesis: you minted it, and the page looks only for nodes serving that genesis;
- the work in a chain's blocks, or who they credit: your node checks every block, so the hash rate a host *delivered* holds when read from your own node;
- its burns: they are balances on chain.

**A host can:**

- take an order and not serve it. The page then finds no node serving your chain, or the host's delivered hash rate falls short of its claim;
- claim more hash rate than it mines. The **Hosts** tab shows what its blocks delivered beside the claim;
- serve a chain now and stop later;
- relay a chain instead of storing it;
- buy and serve its own chains to raise its count;
- with most of the market chain's mining, reorder the market's recent blocks.

**A node you read from can** withhold records, serve an old tip, or misreport balances and block data, which come from its API. Read from your own node with `?node=`.

**A buyer does not get:** the chain's mining rewards (they go to whoever makes its blocks: the host's sponsor, or other miners), blocks after `hearth stop --now` (`hearth stop` keeps paid chains hosted and mined until their time ends), protection from a heavier rival chain at the same path, a refund, proof the host stores rather than relays, or a record of past uptime.
