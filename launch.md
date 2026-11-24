---
title: Launch a chain
---

{% include nav.html %}

# Launch a chain

This page shows how to launch your app's chain from the Hearth web app, keep it running, and check its hosts.

You don't run a node. You describe your chain, the page mints its first block (its genesis) in your browser, and you pay one or more hosts for some periods. The order carries the genesis; each host starts your chain from it, stores it and mines it.

The web app uses the current market, on testnet at `Nexus/testnet/hearth`. The mainnet market, `Nexus/hearth`, still runs the previous market until it is relaunched, and the page doesn't read its offers.

## The web app

The web app is one page with four tabs: **Launch an app**, **Buy coin**, **Hosts** and **Your apps**.

Your account is a key kept in this browser. The app makes it the first time you buy or launch. It holds your coin and signs your orders. Download it from your account page (`#/wallet`) to keep it: anyone with the file can spend from it. You can also load a key file there, from `lattice key generate` or one downloaded earlier.

The page takes these settings in its URL:

| Parameter | What it sets |
|---|---|
| `?node=` | The node it reads from and sends to. Point it at your own node's read URL. Without it, the page finds one (below). |
| `?entry=` | A node hosting Nexus that the page asks for the market's nodes, and every other chain's. Default `https://rpc.lattice.build`. |
| `?market=` | The market chain. Default `Nexus/testnet/hearth`. |
| `?genesis=` | The market's genesis, for a `?market=` the page doesn't know. A node serving another chain at that path is never read as the market. |
| `?sponsor=` | A fee sponsor that pays out parent coin for work. None by default; **Earn with this device** shows only when one is given. |
| `?blockSeconds=` | Seconds per market block, used only to show durations. Default `60`. |

## 1. Buy market coin

Hosts are paid in the market chain's own coin, Hearth coin. Hosts earn it, and sell what they don't spend with `hearth sell-coin` (see [Sell your coin](sell.md#sell-your-coin-hearth-sell-coin)). On **Buy coin**, the page lists coin that sellers have locked on the market chain, each lot for a price in the parent chain's coin (for the testnet market, `Nexus/testnet` coin). The cheapest per coin comes first.

Press **Buy** on a lot. The page then:

1. pays the seller on the parent chain (a receipt);
2. waits for the payment to be in a block, and sends the same payment again if it is not in a block after two minutes;
3. takes the coin on the market chain (a withdrawal);
4. waits for the coin to land.

The fee for taking the coin comes out of it, so you get the lot less the fee. If you reload, the purchase picks up where it left off.

You don't have to buy first: when you launch short of Hearth coin, **Launch** buys it for you from the cheapest sale that covers what you lack, then pays the hosts. What's left over stays in your account.

You need parent coin to pay. Anyone holding some can send it to your address. Where a fee sponsor pays out parent coin, you can also earn it with **Earn with this device**: your browser does work for the fee sponsor, which pays your address each time your earnings reach its payout.

You can also sell Hearth coin from the same tab: lock an amount for a price in parent coin. It stays locked until someone pays. There is no cancel.

See [How it works](how-it-works.md#buying-coin) for the deposit, receipt and withdrawal.

## 2. Describe your app

On **Launch an app**, under **1. Your app**:

| Field | What it does |
|---|---|
| App name | Lowercase letters, digits and dashes, up to 40, starting with a letter or digit. The chain's path is the chain it goes under plus this name, for example `Nexus/testnet/my-app`. |
| Who can write | A rules preset: **Anyone can write** (any key); **Each person edits only their own keys** (every key ends with its writer's address); **Append-only (write once)** (anyone adds a key, nobody changes it after); or **Upload your own .wasm** (your own policy modules). |

Under **Advanced**:

| Field | What it does |
|---|---|
| Under the chain | The chain yours goes under. Default: the market's parent (`Nexus/testnet`). Hosts run every chain from Nexus down to yours, and the price covers each. |
| Size | A preset. **Small app**: 30 s blocks, 20 kB blocks, 10 kB of new state a block, 100 transactions a block. **Busy app**: 5 s blocks, 1,000 kB blocks, 1 MB of new state a block, 1,000 transactions a block. Or set each number yourself. |
| Coin your app starts with | Paid to your account in the genesis. Writes pay fees from it. Rounded up to a multiple of 100, the block reward. Default 21,000. |

Each block pays 100 coin on your chain to whoever mines it.

The rules are fixed at launch: they are part of the genesis. To change them, launch a new app.

Hosts price block capacity and state growth, so a bigger or faster chain costs more. See [Price](how-it-works.md#price).

## 3. Choose hosts

Under **2. Hosts** the page lists every open offer, one per host (its cheapest), priced for your chain and every chain above it. The cheapest per market block comes first. When two hosts are open, the page picks two for you: if one stops, the other keeps your app online. You can tick others. Each is paid separately.

| Column | What it shows |
|---|---|
| Host | The host's address. |
| Price | Market coin per period, for all the chains on the path (Nexus included), each priced by its own spec. |
| Hash rate | The rate the host *claims* in its offer, and what its blocks *delivered* (see [Check a host](#check-a-host)). |
| Record | Chains the host has sold, and how many it hosts now. |

**Burns.** The **Hosts** tab shows the coin burned for each host, on each chain from Nexus down to the market, chain by chain. A burn costs whoever made it. It says nothing about how a host serves. See [Burns](burns.md).

## 4. Pay and launch

Set **Periods** (default 7). The cost is each chosen offer's price times the periods, summed over the hosts you chose. You need that plus the fee.

Press **Launch**. The page:

1. buys Hearth coin first, if you're short (above);
2. checks the name: it refuses a path that is already a chain under the parent, or that someone else has paid hosts for;
3. mints your chain's genesis in this browser, on the parent chain's state now, with the genesis's proof of work (trivial: a genesis has the easiest target);
4. stops if an offer you chose has closed or has new terms since you chose it: check your hosts and launch again. It also refuses to pay twice: if your account already has an order for this path at one of these hosts, it says so;
5. writes one transaction that orders your chain at every chosen host and pays each one. Each order carries the genesis and asserts the host's offer unchanged, so if a host changes its terms before the payment lands, the market refuses it and nothing is paid.

The order is judged as soon as it is in a block:

- **Owed**: the host runs your chain from that market block until your paid time runs out. Your app's page shows the market blocks it is owed for.
- **Void**: with the reason (the genesis did not check out, or less than one period was paid for the whole path). The host owes nothing, and the payment is not refunded.

The page shows *Not judged yet* while it can't read the spec of a chain above yours.

See [Verdicts](how-it-works.md#verdicts).

**Your chain's name.** A buyer may also commit the genesis to the parent chain to secure the name. The web app doesn't do that yet. Nothing protects a chain from a heavier rival chain at the same path.

## 5. Use it

Your app's page (under **Your apps**) shows, once a node serves your genesis:

- the nodes serving it, found through the network, and its height;
- the hosts you ordered it from, and for each the hash rate its blocks were credited over the last day on your chain, beside what it claims;
- code to paste into any web page. It writes one key and reads it back, checking what it reads against a pinned block (your genesis, or a recent block once the chain is older). With more than one node, it reads a block only once two of them hold it.

**Test write** does the same from the page. Writes pay their fee in your chain's coin from the writing key: your account holds the coin your app started with. A fee sponsor, a separate service, can pay instead: add its URL as `sponsors` in the code.

Until a node serves your chain, the page says so and checks again by itself. The host starts it from the genesis your order carries on its next pass.

## 6. Renew

Anyone can pay for more time. On your app's page, under **Keep it running**, set the periods and press **Pay for more**. This writes a `renew` order that pays the host at its offer's terms now. If the offer's terms changed since the page loaded, it stops and asks you to reload. A closed offer sells no more time.

Each renewal adds time from where paid time ends, or from its own block if paid time had run out.

There are no refunds. If a host serves badly, don't renew.

When paid time ends, a host may stop hosting your chain and drop its data. Anyone still hosting it keeps it.

## Check a host

By default the page finds the market's nodes through the network: it asks a Nexus node (`?entry=`) for the nodes the market's hosts declare, keeps those that take transactions and whose market has the market's genesis, and uses the fastest; with none, it reads from the entry itself. It reads from and sends to that one node. One node can serve old or made-up data, or withhold records. Burns come from the node's API.

To trust only yourself, run a node that hosts the market and open the page with:

```
?node=<your node's read URL>
```

On **Hosts**, each host shows:

| Field | What it means |
|---|---|
| Chains sold | Its owed orders, and how many it hosts now. |
| Void orders | Orders whose buyer's order did not hold up. |
| Burned for it | Coin burned for it, per chain, as read from the node. |
| Your chain | Each offer's price for the chain set on **Launch**, per period. |
| Rates | The offer's throughput and storage prices. |
| Hash rate | *Claims*: the offer's `hashrate`, the rate it says it mines every chain it hosts with. *Delivered*: the lowest hash rate its `blocks` address was credited with over the last day (at most 200 blocks), across up to three chains it is paid to host now, read by CID from a node serving each chain's genesis. |
| Blocks pay | The offer's `blocks` address. |

A host that mines less than it claims shows it here; don't renew with it. *Delivered* measures the last day, not past uptime, and it does not prove the host stores the chain. See [Blocks and mining](mining.md).
