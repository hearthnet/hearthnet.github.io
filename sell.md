---
title: Sell hosting
---

{% include nav.html %}

# Sell hosting

This page shows how to sell hosting from your Lattice node with one command, keep it running, and stop it.

## Why host

To earn Hearth coin, the market's coin. Hosts earn it, then spend it on hosting or storage from any seller, or sell it on Lattice's exchange (see [Sell your coin](#sell-your-coin-hearth-sell-coin)), where buyers buy it on the app's **Buy coin** page. You earn it two ways:

- **Orders.** Buyers pay your seller key for each chain you host, by the day.
- **Block rewards.** The market pays 1,000 coin a block, about one block a minute. Your seller makes market blocks (see [Blocks](#blocks)), and each one it makes pays your sponsor's key. `hearth sweep` moves them to your seller key.

The default price is **10,000 coin per chain per day**, so orders count beside block rewards. The market mints 1,440,000 coin a day (1,000 a block, 1,440 blocks), shared by the sellers making its blocks. With N sellers, each mints about 1,440,000 / N a day:

| Sellers pacing the market | Block rewards per seller per day | A full default offer (10 chains × 10,000) |
|---|---|---|
| 1 | 1,440,000 | 100,000 (7%) |
| 10 | 144,000 | 100,000 (69%) |
| 50 | 28,800 | 100,000 (347%) |

Set your own price with `--price`.

A host does three things for each chain it sells: it hosts the chain on its node, makes the chain's blocks, and runs a sponsor that pays visitors' write fees in return for proof-of-work.

## Prerequisites

- A running Lattice node, **5.0.0 or newer** (`lattice up`). See the [Lattice docs](https://lattice.build/docs/). The seller checks the node's features (its `GET /config` saying which layer lists each chain) and stops beside an older one with "Hearth needs Lattice 5.0.0 or newer (you have X: ...)".
- `lattice-sponsor` on your `PATH`, or `--sponsor-bin <path>`. See the [Lattice docs](https://lattice.build/docs/) for where to get it.
- Node.js 22 or later.
- Hearth itself, from source (it is not on npm yet; `npx @hearthnet/hearth` soon):

```
git clone https://github.com/hearthnet/hearth && npm i -g ./hearth
```

## The command

In the node's root, or with `--root`:

```
hearth sell
```

This sells hosting for up to 10 small chains under `Nexus` (see [Offer terms](#offer-terms)), at 10,000 market coin per chain per day (`--price 10000/day`; give `--price` to choose another), for 30 days, on the market `Nexus/hearth`. It runs in the foreground.

It checks every flag, and that a Lattice node lives at `--root`, before it writes anything. A `--root` with no node stops with `no Lattice node at <root>`. Every subcommand takes `--help`.

**The first sync of Nexus and the market takes a few minutes.** Until the node has synced the market, the seller prints `syncing market: h / H` (its tip against the best header its peers showed) and reads, funds, writes and burns nothing.

## What it does

1. **Finds the node.** It uses `--root`, else the current directory, else `~/.lattice` (where a bare `lattice-node` keeps its storage): the first that holds a running node's `chains/Nexus/.cookie`. With `--node` and `--token-file` it uses a node elsewhere instead.
2. **Checks the node** is Lattice 5.0.0 or newer, by its features.
3. **Makes the seller's key** at `<root>/hearth/seller.json` unless there is one. The file is mode 0600. It prints the address, never the key, and says what you earn and what you can spend it on. It refuses a key file that others can read.
4. **Writes the offer file** from the flags, once, at `<root>/hearth/<offer id>.json`. The offer id comes from the terms, so the same flags find the same file.
5. **Runs the seller** (`host/sell-hosting.mjs`) in the foreground. Beside the node it starts lattice-sponsor (below). Every `--interval` seconds (default 30) it makes one pass:
   1. It adds the market and the offer's parent to its own fragment of the node's configuration, `hostmarket` (`config.d/hostmarket.json`), so the node hosts them.
   2. Once the node has synced the market, it reads the market at its tip and judges its own orders, the same way every reader does.
   3. Once the key holds enough market coin (see [Funding](#funding)), it writes the offer (and the URL record, if `--url`), then the storage offer, then any changed URL.
   4. It creates each owed chain not yet on its parent, once, from the buyer's payload (`POST /chains`), and lists every owed chain in the `hostmarket` fragment.
   5. With `--burn`, it burns what the burned total is missing.
6. **Makes blocks** on the market and on every chain it hosts for its orders (below).
7. **Reports** each pass (see [Status](#status)).

A chain whose paid time ran out is left out of the fragment. The node stops hosting it unless another layer of its configuration lists it; what it stored stays. The seller never restarts the node. A failed pass is logged and tried again on the next one.

## Blocks

The seller makes the blocks of the market and of every chain it hosts for its orders. There is no flag: it always does. Each is a zero-work block, about one hash, made with this machine's processor. A chain is due once its tip is one target block time (`targetBlockTime`) old, from anyone's block, so a chain others keep moving needs nothing from it. A chain the node is still syncing is never paced.

Every block pays the sponsor's key, never the seller's.

One hash can clear several chains in a template. On a node whose template requests take `exclude`, each round leaves out every chain not due (and with none due below it), so those get no early block. A due chain's parents are still carried with it, so a parent with an easier target can get an early block, and its difficulty adjustment (ASERT) hardens it back toward its target block time. On a node without `exclude`, the seller skips a hash that would also clear a harder chain not yet due, but can't keep a hash off an easier one, so that chain can get some early blocks.

See [Blocks and mining](mining.md) for how buyers check the blocks.

## Sponsor

Beside the node the seller runs lattice-sponsor, and starts it again, with backoff, whenever it exits. Its key is `<root>/hearth/sponsor.json` (mode 0600), never the seller's: it owns nothing but its float, which the blocks pay.

The sponsor:

- pays fees for visitors who pay with proof-of-work shares, so they write with no coin;
- sells pins for work at `--storage-rate` bytes per market coin (default 10240), kept until you take them down, up to `--storage-budget` at once (default `256M`; see [Abuse, takedowns and legal](#abuse-takedowns-and-legal));
- listens on `--sponsor-port` (default 8650).

Without `--sponsor-url` it listens on loopback only, and the offer names no sponsor. With `--sponsor-url https://host.example/sponsor` it listens on every interface (put TLS in front of the port), and the offer names that URL. The URL must be `https`, or `http` on loopback.

With `--node` (a remote node) the seller runs no sponsor: lattice-sponsor reaches a node only on its loopback, so run it there.

## What an offer says

Besides its terms, every offer names `blocks <address>` (the sponsor's key, which its blocks pay) and, with a public sponsor, `sponsor <url>`. Buyers use both to check you; see [Check a host](launch.md#check-a-host).

## Rewards: hearth sweep

Block rewards build up in the sponsor's key, on every chain the seller makes blocks of, and fund the fees its visitors pay with work. To move what it holds above a float to the seller's address:

```
hearth sweep [--float 100] [--chain <path>]
```

It moves what the sponsor's key holds on one chain (default the market), keeping the float, the fees it holds for transactions it signed, and any fund deposits. It signs with the sponsor's key, so it refuses while the seller runs, or while any lattice-sponsor answers on `--sponsor-port`, and says how to pause the seller: stop its service (or the process), run `hearth sweep`, and start it again (with `--resume` when you start it by hand). Don't use `hearth stop` for this: it closes your offers.

## Funding

Every write on the market costs a fee in market coin. Before it writes, the seller checks that its key holds three fees: for the offer, the URL record and a storage offer.

You don't have to fund it. The seller's blocks pay the sponsor's key (1,000 market coin a market block), so for its first writes the seller moves four fees (enough for a close too) from the sponsor's key to its own, then starts lattice-sponsor. It moves them before the sponsor starts, so nothing else signs with that key at the same time. You see:

```
moving 4 market coin of the sponsor key's block rewards to the seller's key for its first writes (lattice-sponsor starts once it has landed)
```

It moves them once the sponsor's key holds five fees: the four it moves, and the fee for the move itself. Until then, it says it is waiting for block rewards, and how to skip the wait. If any lattice-sponsor already answers on `--sponsor-port` (one you started yourself could sign with the same key), it moves nothing, and says so.

With `--node` (no sponsor beside it), or once an offer of yours is out, it moves nothing. If the key is short then, it prints:

```
Send at least <n> coin on Nexus/hearth to <address>, from any account that holds some, or earn it with this machine: hearth earn --sponsor <url>; this seller waits and checks again every 30 s. The sponsor's key holds the block rewards: hearth sweep moves them here while no lattice-sponsor signs with it.
```

Send market coin to the address from any account, buy it in the web app (see [Buy market coin](launch.md#1-buy-market-coin)), or earn it:

```
hearth earn [--until <coins>] [--workers 1] [--sponsor <url>]
```

`hearth earn` does work for a fee sponsor's payouts with this machine's processor, naming the seller's address. The sponsor pays the address in coin each time its earnings reach its payout. It stops once the address holds `--until` coins on the market (by default four fees: the seller's first writes and a close). There is no default sponsor: give `--sponsor <url>`, a lattice-sponsor that pays out market coin for work.

Buyers pay you in market coin, to the seller's address.

## Sell your coin: hearth sell-coin

The coin you earn is yours to spend on hosting or storage, or to sell. Buyers of hosting need market coin, and they buy it on the web app's **Buy coin** page from coin that hosts have put on sale. To put yours on sale:

```
hearth sell-coin 300000 --price 3000 [--float 100]
```

This locks 300,000 market coin from your seller key (a month of one chain at the default price) on Lattice's own exchange, for 3,000 `Nexus` coin for all of it. `--price` is what a buyer pays for the whole lot, in the coin of the market's parent chain. Before it signs, it checks the amounts and that your key keeps at least `--float` coin (default 100) for its fees. To sell block rewards, `hearth sweep` them to your seller key first.

The running seller signs with the same key, so, like `hearth sweep`, it refuses while the seller runs or a lattice-sponsor answers on `--sponsor-port`, and says how to pause the seller without closing your offers.

Because a deposit can't be cancelled, a retry never locks a second lot by accident: if your key already has an open deposit of the same amount and price, it prints that deposit and sends nothing. Pass `--again` to lock another identical lot.

It prints the deposit and what a buyer pays:

```
locked 300000 market coin on Nexus/hearth for sale at 3000 Nexus coin: deposit <address>/3000/<nonce>
transaction <cid>
A buyer pays 3000 Nexus coin (and that chain's fee) on Nexus; Lattice credits it straight to <address> there, with nothing to claim, and the buyer takes the coin. ...
```

**Getting paid.** When a buyer pays, Lattice moves the price from the buyer to your address on `Nexus` in the same transaction. You don't claim anything, and nothing needs to run. Your seller key's address is the same on every chain, so the key you sell with is the key that holds the payment.

**No cancel.** Lattice has no cancel or refund for a deposit. The coin stays locked until someone buys it. The only way back is to buy it yourself like any buyer: pay its price to your own address on `Nexus` and take the coin, at a fee on each chain. Price it to sell.

**Pricing.** `hearth status` shows the cheapest coin on sale on the market, in parent coin per market coin, and each of your deposits, with the buyer once one has paid for it.

## Status

Each pass prints, and `hearth status` prints once:

```
market Nexus/hearth at block 12561: the seller's key <address> holds 297; the sponsor's key <address> holds 52000 (its block rewards)
2 orders {"owed":1,"pending":1} and 0 renewals: earned 70000 coin from buyers, 70000 more settling
owes hosting to Nexus/app1 (to market block 14001)
storage pinned by the sponsor: 12.0 MiB of its 256.0 MiB budget, 1.0 MiB held for uploads being paid, 243.0 MiB left
```

`hearth status` adds the exchange:

```
exchange: the cheapest market coin on sale is 300000 for 3000 Nexus coin (0.01 each), of 4 open deposits
on sale: 300000 for 3000 Nexus coin (0.01 each), deposit <address>/3000/<nonce>
the seller's key holds 3000 on Nexus, where buyers of its coin pay it
```

A deposit someone has paid for says `bought by <address>, who paid you on Nexus` until they take the coin. The reads are bounded: the cheapest is of the first 1,000 deposit entries (it says so when there are more), your own are read by your address, and at most 20 of their payments are checked. The price per coin is exact; `~` marks one cut at 8 decimals.

*Earned* is what other buyers paid your key for hosting you owe or owed: settled orders (16 market blocks) judged *owed*, and settled renewals of those chains that bought time (not ones after a close). Void, full and taken orders, and your own orders, count in neither. *Settling* is what buyers paid in orders and renewals that have not settled yet. `hearth status` never waits: if the market isn't synced it says `market not synced (height X of Y)`, and each read has a deadline. With `--node`, the storage line is skipped, since the sponsor runs beside the remote node. The storage line comes from the sponsor's `GET /terms`: what it has pinned, what it holds for uploads still being paid, and what is left of the budget.

## Offers: 30 days, versions, no auto-renew

An offer sells for 30 days from when it is written (or its file's `duration`, in market blocks). It never renews by itself. To sell for another 30 days, run `hearth sell` again: it writes a new version of the offer.

Each write is a **version** of the offer. A version takes effect 10 market blocks after it is written. Each payment is priced by the version in force at the payment's own height, so a new price applies only to orders yet to come. After the first write, the seller writes a new version only when the address its blocks pay or its sponsor changes.

Each set of flags names its own offer. Running again with the same flags finds the same offer. Running with other flags writes a new offer beside the old one, and the seller says the old one stays on sale. The seller keeps hosting the parents of all your offers still on sale. With `--id`, the offer file `<id>.json` is written only once: changing other flags with the same `--id` keeps the old terms.

## Flags

| Flag | Default | What it does |
|---|---|---|
| `--price <coin>/<period>` | `10000/day` | Whole market coin per chain per period. The period is a number of market blocks, or `hour`, `day` or `week`, converted to market blocks at the market's block time. Give `--price` or `--offer`, not both. |
| `--offer <file>` | none | An offer file, instead of `--price`. |
| `--parent <path>` | `Nexus` | The chain the offer's chains go under. |
| `--chains <n>` | `10` | How many chains the offer sells. |
| `--id <id>` | from the terms | The offer's id, with `--price`. Without it: `o-` and 10 hex digits of a hash of the terms. |
| `--market <path>` | `Nexus/hearth` | The market chain. |
| `--root <dir>` | found (see above) | The node's root. |
| `--node <url>` | none | A node's operator listener, for a seller running elsewhere. Needs `--token-file`. |
| `--token-file <file>` | none | An operator token for `--node`. |
| `--key <file>` | `<root>/hearth/seller.json` | The seller's key. Made if missing. With `--node`, the default is `hearth/seller.json` in the current directory. |
| `--url <url>` | none | Your public read URL. Without it you sell key only. |
| `--proof <url>` | `<url's origin>/.well-known/lattice.json` | Where your domain lists your address, if not the default. |
| `--storage-offer <file>` | none | Also write a storage offer from this file. |
| `--sponsor-bin <path>` | `lattice-sponsor` | The lattice-sponsor to run. |
| `--sponsor-port <port>` | `8650` | The port the sponsor listens on. |
| `--sponsor-url <url>` | none | The sponsor's public URL (`https`, or loopback). The offer names it. |
| `--storage-rate <n>` | `10240` | Bytes the sponsor pins per market coin. |
| `--storage-budget <size>` | `256M` | The most the sponsor holds pinned at once (uploads being paid included), in bytes or with `K`, `M`, `G`, `T` (binary). A takedown frees its bytes. |
| `--burn <coin>` | none | Burn until the burned total for your key reaches this amount. |
| `--burn-chain <path>` | the market | Where `--burn` burns. Only with `--burn`. |
| `--fee <n>` | the chain's minimum relay fee, else 1 | The fee for each write and burn. |
| `--interval <s>` | `30` | Seconds between passes. |
| `--once` | off | Make one pass and exit. |
| `--lattice <path>` | `lattice` | The `lattice` command. |
| `--install` | off | Write services for the node and the seller, print how to load them, and exit. |
| `--resume` | off | Write no offer version on start (what the installed service runs). |

### Offer terms

`hearth sell` (with or without `--price`) sells these terms:

| Term | Value |
|---|---|
| `capacity` | 10 chains (`--chains`) |
| `maxBlockSize` | 20,000 bytes |
| `maxStateGrowth` | 10,000 bytes |
| `minTargetBlockTime` | 30,000 ms |
| `maxNumberOfTransactionsPerBlock` | 100 |
| `modules` | `any` |
| `duration` | 30 days of market blocks from when the offer is written |

**Worst case.** A buyer's chain may fill every block. Your node keeps every block and the state they add, and your own sponsor pays the fees. So these terms let one chain add at most 2,880 blocks a day (30 s apart) of 20 kB, each adding 10 kB of state: about **86 MB a day per chain, 864 MB a day for all 10**. Blocks faster than the target harden a chain, so a buyer can't get many more. The offer format has no cap on a chain's total state, so this rate is the bound. The seller prints the worst case for the terms it sells:

```
offer o-...: worst case, each chain sold adds up to 86.4 MB a day to this node's disk (2880 blocks of 20.0 kB, each adding 10.0 kB of state), 864.0 MB a day for all 10; an offer file (--offer) sells other limits
```

The old defaults (5 s blocks, 1 MB blocks, 1 MB of state growth, 50 chains) allowed about 34.6 GB a day per chain.

To set other terms, write an offer file and pass `--offer`. `period` is in market blocks, or `"hour"`, `"day"` or `"week"`; `duration` is in market blocks (30 days when absent):

```json
{ "id": "small", "parent": "Nexus", "price": 10000, "period": 1440, "duration": 43200,
  "capacity": 10, "maxBlockSize": 20000, "maxStateGrowth": 10000, "minTargetBlockTime": 30000,
  "maxNumberOfTransactionsPerBlock": 100, "modules": "any" }
```

`modules` is `"any"` or a list of policy module CIDs. Without `id`, the offer's id is `1`.

## Abuse, takedowns and legal

**Everything you host is served from your IP.** That includes the chains you sell (their state is public, and `modules any` lets a buyer's chain hold any bytes), and the files your sponsor pins for anyone who pays with work. Some of it may be unlawful where you live, or against your provider's terms. One abuse report can close a VPS account. On a home connection the exposure is yours.

**You choose what to serve.** No protocol rule makes you host or serve anything:

- Without `--sponsor-url` your sponsor listens on loopback only, so strangers can't pin files through it. The trade-off: your offer names no sponsor, so your buyers' visitors can't write for work.
- `--storage-budget` caps what your sponsor holds pinned at once (default `256M`, lower than lattice-sponsor's own 1 GiB). An upload that doesn't fit is refused. Lower it, or use a separate disk.
- The offer's limits cap how fast a chain grows (see [Offer terms](#offer-terms)). An offer file can set them lower, or list the policy `modules` you accept instead of `any`.
- `hearth stop --now` stops hosting at once. A chain you stop hosting is dropped from the node's configuration; what it stored stays on disk until you delete it.

**Take a file down** with:

```
hearth unpin <cid>
```

Run it beside the node (in its root, or with `--root`). It asks the sponsor on `--sponsor-port` to take the file down (`POST /takedown`, with the node's cookie as the operator's credential): the sponsor unpins its Volumes on the node, frees their bytes from the budget, and refuses that CID from then on, for storing, work and payment. The seller must be running, since its sponsor serves the route. It needs a lattice-sponsor with the takedown route; an older one answers that it has none. If the node refuses the unpin, the CID is still blocked: run `hearth unpin` again.

**Whoever paid for a file you take down loses the rest of its time. It is never refunded.**

`hearth status` shows how much of the budget is used.

Keep the operator token and the node's cookie to yourself: the seller holds an admin-equivalent credential over the node.

## A domain, or key only

**Key only.** Without `--url` you write no URL record. You claim no domain. The web app lists your offers under your key, marked *Key only*, for buyers to judge by your record. It can't probe the chains you serve, since you name no URL.

**A domain.** With `--url https://host.example`, the seller writes a URL record naming your public read URL. Then serve this file at `https://host.example/.well-known/lattice.json`:

```json
{ "sellers": ["<your seller address>"] }
```

Your offers then carry the domain's name and *Verified*. One domain can list several sellers. If the file is somewhere else on the same hostname, write where with `--proof`:

```
hearth sell --url https://host.example \
  --proof https://host.example:8650/.well-known/lattice.json
```

The web app fetches the file from the buyer's browser, so serve it with `Access-Control-Allow-Origin: *`. If the domain does not list your address, or the file can't be read, your offers are hidden: anyone could name someone else's URL. A URL record stays once written.

Storage offers are offered only from verified hosts.

## Storage offers

Your sponsor already sells pins for work. To also sell storage for market coin, pass `--storage-offer storage.json`:

```json
{ "id": "files", "sponsor": "https://host.example/sponsor", "bytesPerCoin": 10240, "capacity": 10737418240 }
```

| Field | What it is |
|---|---|
| `id` | Required. Not the hosting offer's id. |
| `sponsor` | Your sponsor's `https` URL. |
| `bytesPerCoin` | Bytes kept for good per market coin. |
| `capacity` | Bytes the offer sells in all. |
| `until` or `duration` | When it ends: a market height, or market blocks from now. |

Buyers see it as verified only if your domain lists your address, and your sponsor's `GET /terms` names your address as `payTo` and the offer's `bytesPerCoin` as its rate for the market. The sponsor does the selling.

## A node elsewhere

The seller can run away from the node, through the node's operator listener (`operatorListen` in its configuration), with a token:

```
lattice tokens create seller --role admin --out seller.token --root <root>
hearth sell --node http://10.0.0.5:8090 --token-file seller.token
```

It runs no sponsor there (see [Sponsor](#sponsor)). A token that may write a fragment may set any key in it, so it controls the node nearly as fully as `admin` anyway.

## Run it as a service

```
hearth sell --install
```

`--install` writes services for both the node (`lattice up --root <root>`) and the seller, prints the commands that load them, and exits. It loads nothing itself. The seller's service runs with `--resume`, so a restart writes no new offer version. Running `--install` itself counts as running `hearth sell` again: the service's next start writes one version from the offer file.

**Linux.** systemd user units `~/.config/systemd/user/hearth-node.service` and `hearth.service` (which starts after the node and requires it). Load them with:

```
loginctl enable-linger $USER && systemctl --user daemon-reload && systemctl --user enable --now hearth-node.service hearth.service
```

`enable-linger` keeps your user's services running without a login, and starts them at boot.

**macOS.** launchd agents `~/Library/LaunchAgents/net.hearth.node.plist` and `net.hearth.seller.plist`, logging to `~/Library/Logs/hearth-node.log` and `hearth.log`. The command prints the `launchctl bootstrap` lines that load them. launchd has no ordering, so the seller waits for the node itself. A launch agent runs while you are logged in; the command also says how to make them LaunchDaemons for boot.

The node is started again after a failure exit (which includes `lattice restart`), but not after `lattice down` or an invalid configuration. The seller is started again after a failure, not after it exits 0. If your node already runs under another service, load only the seller's.

Other systems are not supported by `--install`.

## Stop selling

```
hearth stop [--now]
```

`hearth stop` closes every offer of the key still on sale: it writes a new version whose `until` is now, which takes effect 10 market blocks later. Orders placed before then are owed. It waits until those orders have settled and says which chains have paid time left and when the last ends.

The seller keeps running, closing: it writes no offer, and keeps hosting, making blocks for and sponsoring each chain with paid time until that ends. Then it deletes its fragment and exits. With no seller running, `hearth stop` says to start it with `--resume` to serve the paid chains.

`hearth stop --now` stops everything at once: the seller (its installed service, else the process), its block-making and its sponsor, and hosting. It says what buyers lose: the rest of the time they paid for, with no refund.

`hearth sell` opens the offer again.
