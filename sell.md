---
title: Sell hosting
---

{% include nav.html %}

# Sell hosting

This page shows how to sell hosting with one command, keep it running, and stop it.

`hearth sell` sells on the current market, on testnet at `Nexus/testnet/hearth`. The mainnet market, `Nexus/hearth`, still runs the previous market until it is relaunched; this version of Hearth doesn't sell there.

## Why host

To earn Hearth coin, the market's coin. Hosts earn it, then spend it on hosting from any seller, or sell it on the chains' own exchange (see [Sell your coin](#sell-your-coin-hearth-sell-coin)), where buyers buy it on the app's **Buy coin** page. You earn it two ways:

- **Orders.** Buyers pay your seller key for each chain they order, by the period.
- **Block rewards.** The market pays 1,000 coin a block, about one block a minute, to whoever mines it. Your seller mines the market and every chain it hosts (see [Mining](#mining)), and each block it makes pays your sponsor's key. `hearth sweep` moves them to your seller key.

A host does three things for each chain it sells: it hosts the chain on its node, mines it with all of its hash rate, and runs a sponsor beside the node that pays visitors' write fees in return for proof-of-work.

## Prerequisites

- Node.js 22 or later.
- macOS on Apple silicon, or Linux on x86-64.

That's all. Hearth runs the node it sells from: the first time, it downloads a pinned node and sponsor into `~/.hearth/bin` (each checked against a SHA-256 written into Hearth), makes the node in `~/.hearth/node`, and runs it while `hearth sell` sells (its log is `~/.hearth/node/node.log`). Stop the seller and the node stops with it; other `hearth` commands use it while it runs, or start it for as long as they take. To keep selling across restarts, see [Run it as a service](#run-it-as-a-service). It listens on port 4001 for peers and serves its operator API on `127.0.0.1:8080`.

Already run a node? It must be 5.1.0 or newer: the seller hands it each buyer's genesis with its proof of work. Give its root with `--root`, or `--node` with `--token-file` for one elsewhere.

## The command

```
npx @hearthnet/hearth sell
```

This sells hosting at 60 market coin per kB/s of block capacity and 6 per kB of state growth, per day, for every chain on a buyer's path, on the market `Nexus/testnet/hearth`. It offers the hash rate its miner measures. It runs in the foreground.

It checks every flag, and that a node lives at `--root` when you give one, before it writes anything. A `--root` with no node stops with `no node at <root>`. Every subcommand takes `--help`.

**The first sync of Nexus and the market takes a few minutes.** Until the node has synced the market, the seller prints `syncing market: h / H` (its tip against the best header its peers showed) and reads, funds, writes and burns nothing.

## What it does

1. **Finds the node.** It uses `--root`, else the current directory, else `~/.lattice`: the first that holds a running node's `chains/Nexus/.cookie`. With none, it runs its own (see [Prerequisites](#prerequisites)); a node of your own there that is not running stops it with `the node at <dir> is not running`. With `--node` and `--token-file` it uses a node elsewhere instead.
2. **Checks the node** is new enough, by its features.
3. **Makes the seller's key** at `<root>/hearth/seller.json` unless there is one. The file is mode 0600. It prints the address, never the key, and says what you earn and what you can spend it on. It refuses a key file that others can read.
4. **Runs the seller** (`host/sell-hosting.mjs`) in the foreground. Every `--interval` seconds (default 30) it makes one pass:
   1. It adds the market to its own fragment of the node's configuration, `hostmarket` (`config.d/hostmarket.json`), so the node hosts it.
   2. Once the node has synced the market, it starts the miner (see [Mining](#mining)).
   3. Once the key holds enough market coin (see [Funding](#funding)), it starts the sponsor and writes the offer, or changes it in place when its terms differ from what the market says.
   4. It reads the market at its tip and judges its own orders, the same way every reader does. It lists every owed chain, with every chain above it, in the `hostmarket` fragment, and starts each owed chain from the genesis its order carries: it pins the genesis on the node and hands it the genesis block with its proof of work. It tries again each pass until the node takes it. It pushes each paid genesis once and keeps the path hosted; when another genesis takes the path over at your host, it pushes that one too. Which chain wins at the path is Lattice's fork choice.
   5. With `--burn`, it burns what the burned total is missing.
5. **Reports** each pass (see [Status](#status)).

A chain whose paid time ran out is left out of the fragment. The node stops hosting it unless another layer of its configuration lists it; what it stored stays, but for the genesis pins no owed chain needs, which the seller unpins. It unpins only what its own launches pinned (recorded in `launch-pins-<market>.json` beside its key), and nothing while a chain it hosts has an order not judged yet; such a chain stays listed. Doubt never drops a hosted chain. Run one seller per node: a seller knows only its own launches. The seller never restarts the node. A failed pass is logged and tried again on the next one.

## Your offer

Your offer is one record on the market, `offer/<your address>/<id>` (`--id`, default `1`), with five lines:

```
hashrate 5000000      hashes per second you mine every chain you host with
throughput 60         coin per kB/s of block capacity, per period
storage 6             coin per kB of state growth, per period
period 1440           market blocks a period lasts
blocks bafyrei...     the address your blocks pay: your sponsor's key
```

| Line | Flag | Default |
|---|---|---|
| `hashrate` | `--hashrate <hashes/s>` | What your miner measured over its first minute. The offer waits for it. Once written, a measured rate is not rewritten as it wanders. |
| `throughput` | `--throughput <coin>` | `60` |
| `storage` | `--storage <coin>` | `6` |
| `period` | `--period <blocks>` | `1d`: market blocks, or hours, days or weeks (`12h`, `1d`, `2w`) at the market's block time |
| `blocks` | none | Your sponsor's key |

The hash rate isn't priced. It says how hard you mine: each chain you host gets all of it. Buyers compare it with what your blocks deliver (see [Blocks and mining](mining.md#how-a-buyer-checks-the-hash-rate)).

There's no URL, no limit on chain size, no capacity and no parent in an offer. You host any path a buyer orders, at any size; your prices are what make a big or fast chain cost more.

**Prices.** A chain costs, per period, `throughput × maxBlockSize ÷ targetBlockTime` (its block capacity in kB/s) plus `storage × maxStateGrowth ÷ 1000` (its state growth in kB), each rounded up. You run every chain from Nexus down to the one ordered, so a buyer pays for each of them, Nexus included, each priced by its own spec. At the defaults, a small chain (20 kB blocks every 30 s, 10 kB of state growth) costs 40 + 60 = 100 a day itself, and the chains above it add their own prices: under `Nexus/testnet` the whole path comes to about 25,000 a day. See [Price](how-it-works.md#price).

**Changing it.** The seller writes the offer once, and changes it in place whenever the flags' terms or the address your blocks pay differ from what the market says. Run `hearth sell` with new flags to change your prices. An order asserts your offer unchanged, so an order priced at your old terms doesn't land after a change; orders already in a block keep the terms they landed on. The offer stays on sale until you stop it (`hearth stop`): there is no end date and nothing to renew.

## Mining

The seller mines every chain the node hosts, all the time, with this machine's processor. There is no flag: it always does. Each round it asks the node for a template carrying every hosted chain, searches it, and submits each hit. One hash makes a block on every chain whose target it clears, so each chain gets all of your hash rate, and its difficulty settles where the miners on it make its blocks on time. It logs the hash rate it measures each minute.

Every block pays the sponsor's key, never the seller's.

See [Blocks and mining](mining.md) for how buyers check it.

## Sponsor

Beside the node the seller runs the sponsor, and starts it again, with backoff, whenever it exits. Its key is `<root>/hearth/sponsor.json` (mode 0600), never the seller's: it owns nothing but its float, which the blocks pay.

The sponsor:

- pays fees for visitors who pay with proof-of-work shares, so they write with no coin;
- sells pins for work at `--storage-rate` bytes per market coin (default 10240), kept until you take them down, up to `--storage-budget` at once (default `256M`; see [Abuse, takedowns and legal](#abuse-takedowns-and-legal));
- listens on `--sponsor-port` (default 8650).

Sponsoring is a service of its own, beside the market: your offer never names your sponsor. A buyer who wants their visitors to write without coin adds a sponsor's URL to their app themselves.

Without `--sponsor-url` the sponsor listens on loopback only. With `--sponsor-url https://host.example/sponsor` it listens on every interface (put TLS in front of the port). The URL must be `https`, or `http` on loopback. When Hearth makes you reachable (below), the sponsor stays on loopback behind Hearth's proxy.

With `--node` (a remote node) the seller runs no sponsor: a sponsor reaches a node only on its loopback, so run it there.

## Rewards: hearth sweep

Block rewards build up in the sponsor's key, on every chain the seller mines, and fund the fees its visitors pay with work. To move what it holds above a float to the seller's address:

```
hearth sweep [--float 100] [--chain <path>]
```

It moves what the sponsor's key holds on one chain (default the market), keeping the float, the fees it holds for transactions it signed, and any fund deposits. It signs with the sponsor's key, so it refuses while the seller runs, or while any sponsor answers on `--sponsor-port`, and says how to pause the seller: stop its service (or the process), run `hearth sweep`, and start it again (with `--resume` when you start it by hand). Don't use `hearth stop` for this: it closes your offer.

## Funding

Every write on the market costs a fee in market coin. Before it writes its offer, the seller checks that its key holds two fees: for the offer, and for its close.

You don't have to fund it. The seller's blocks pay the sponsor's key, so for its first write the seller moves those two fees from the sponsor's key to its own, then starts the sponsor. It moves them before the sponsor starts, so nothing else signs with that key at the same time. You see:

```
moving 2 market coin of the sponsor key's block rewards to the seller's key for its offer and its close (the sponsor starts once it has landed)
```

It moves them once the sponsor's key holds three fees: the two it moves, and the fee for the move itself. Until then, it says it is waiting for block rewards, and how to skip the wait. If any sponsor already answers on `--sponsor-port` (one you started yourself could sign with the same key), it moves nothing, and says so.

With `--node` (no sponsor beside it), or once your offer is out, it moves nothing. If the key is short then, it says what to send, earn or sweep:

```
Send at least <n> coin on Nexus/testnet/hearth to <address>, from any account that holds some, or earn it with this machine: hearth earn --sponsor <url>; this seller waits and checks again every 30 s.
```

Send market coin to the address from any account, buy it in the web app (see [Buy market coin](launch.md#1-buy-market-coin)), or earn it:

```
hearth earn [--until <coins>] [--workers 1] --sponsor <url>
```

`hearth earn` does work for a fee sponsor's payouts with this machine's processor, naming the seller's address. The sponsor pays the address in coin each time its earnings reach its payout. It stops once the address holds `--until` coins on the market (by default two fees: the offer and its close). There is no default sponsor: give `--sponsor <url>`, a sponsor that pays out market coin for work.

Buyers pay you in market coin, to the seller's address.

## Sell your coin: hearth sell-coin

The coin you earn is yours to spend on hosting, or to sell. Buyers of hosting need market coin, and they buy it on the web app's **Buy coin** page from coin that hosts have put on sale. To put yours on sale:

```
hearth sell-coin 300000 --price 3000 [--float 100]
```

This locks 300,000 market coin from your seller key on the chains' own exchange, for 3,000 coin of the market's parent chain (`Nexus/testnet` for the testnet market) for all of it. `--price` is what a buyer pays for the whole lot. Before it signs, it checks the amounts and that your key keeps at least `--float` coin (default 100) for its fees. To sell block rewards, `hearth sweep` them to your seller key first.

The running seller signs with the same key, so, like `hearth sweep`, it refuses while the seller runs or a sponsor answers on `--sponsor-port`, and says how to pause the seller without closing your offer.

Because a deposit can't be cancelled, a retry never locks a second lot by accident: if your key already has an open deposit of the same amount and price, it prints that deposit and sends nothing. Pass `--again` to lock another identical lot.

It prints the deposit and what a buyer pays:

```
locked 300000 market coin on Nexus/testnet/hearth for sale at 3000 Nexus/testnet coin: deposit <address>/3000/<nonce>
transaction <cid>
A buyer pays 3000 Nexus/testnet coin (and that chain's fee) on Nexus/testnet; the chain credits it straight to <address> there, with nothing to claim, and the buyer takes the coin. ...
```

**Getting paid.** When a buyer pays, the chain moves the price from the buyer to your address on the parent chain in the same transaction. You don't claim anything, and nothing needs to run. Your seller key's address is the same on every chain, so the key you sell with is the key that holds the payment.

**No cancel.** There is no cancel or refund for a deposit. The coin stays locked until someone buys it. The only way back is to buy it yourself like any buyer: pay its price to your own address on the parent chain and take the coin, at a fee on each chain. Price it to sell.

**Pricing.** `hearth status` shows the cheapest coin on sale on the market, in parent coin per market coin, and each of your deposits, with the buyer once one has paid for it.

## Status

Each pass prints, and `hearth status` prints once:

```
market Nexus/testnet/hearth at block 12561: the seller's key <address> holds 297; the sponsor's key <address> holds 52000 (its block rewards)
3 orders and renewals {"owed":2,"renewal":1}: earned 70000 coin from buyers
owes hosting to Nexus/testnet/app1 (to market block 14001)
storage pinned by the sponsor: 12.0 MiB of its 256.0 MiB budget, 1.0 MiB held for uploads being paid, 243.0 MiB left
```

`hearth status` adds the exchange:

```
exchange: the cheapest market coin on sale is 300000 for 3000 Nexus/testnet coin (0.01 each), of 4 open deposits
on sale: 300000 for 3000 Nexus/testnet coin (0.01 each), deposit <address>/3000/<nonce>
the seller's key holds 3000 on Nexus/testnet, where buyers of its coin pay it
```

A deposit someone has paid for says `bought by <address>, who paid you on Nexus/testnet` until they take the coin. The reads are bounded: the cheapest is of the first 1,000 deposit entries (it says so when there are more), your own are read by your address, and at most 20 of their payments are checked.

*Earned* is what other buyers paid your key for hosting you owe or owed: orders judged *owed*, and the renewals that bought time on them. Void orders, and your own orders, count for nothing. `hearth status` never waits: if the market isn't synced it says `market not synced (height X of Y)`, and each read has a deadline. With `--node`, the storage line is skipped, since the sponsor runs beside the remote node. The storage line comes from the sponsor's `GET /terms`: what it has pinned, what it holds for uploads still being paid, and what is left of the budget.

## Flags

| Flag | Default | What it does |
|---|---|---|
| `--throughput <coin>` | `60` | Whole market coin per kB/s of block capacity, per period. |
| `--storage <coin>` | `6` | Whole market coin per kB of state growth, per period. |
| `--period <n>` | `1d` | The period: market blocks, or hours, days or weeks (`12h`, `1d`, `2w`) at the market's block time. |
| `--hashrate <hashes/s>` | measured | The hash rate the offer claims. Without it, the rate the miner measures over its first minute. |
| `--id <id>` | `1` | The offer's id: no slash or space. |
| `--market <path>` | `Nexus/testnet/hearth` | The market chain. |
| `--root <dir>` | found (see above) | The node's root. |
| `--node <url>` | none | A node's operator listener, for a seller running elsewhere. Needs `--token-file`. |
| `--token-file <file>` | none | An operator token for `--node`. |
| `--key <file>` | `<root>/hearth/seller.json` | The seller's key. Made if missing. With `--node`, the default is `hearth/seller.json` in the current directory. |
| `--public` | on Hearth's own node, without `--sponsor-url` or `--key-only` | Make this machine reachable through a Cloudflare quick tunnel (below). A node of your own (`--root`) needs it given, and `publicRead` and `publicSubmit` set. |
| `--key-only` | off | No tunnel. |
| `--sponsor-bin <path>` | one on `PATH`, else Hearth's own | The sponsor to run. |
| `--sponsor-port <port>` | `8650` | The port the sponsor listens on. |
| `--sponsor-url <url>` | none | The sponsor's public URL (`https`, or loopback). It then listens on every interface. |
| `--storage-rate <n>` | `10240` | Bytes the sponsor pins per market coin. |
| `--storage-budget <size>` | `256M` | The most the sponsor holds pinned at once (uploads being paid included), in bytes or with `K`, `M`, `G`, `T` (binary). A takedown frees its bytes. |
| `--burn <coin>` | none | Burn until the burned total for your key reaches this amount. |
| `--burn-chain <path>` | the market | Where `--burn` burns. Only with `--burn`. |
| `--fee <n>` | the chain's minimum relay fee, else 1 | The fee for each write and burn. |
| `--interval <s>` | `30` | Seconds between passes. |
| `--once` | off | Make one pass and exit. |
| `--lattice <path>` | Hearth's own | The node's command-line tool, for a node of your own. |
| `--install` | off | Write services for the node and the seller, print how to load them, and exit. |
| `--resume` | off | Don't reopen an offer `hearth stop` closed (what the installed service runs). |

## Abuse, takedowns and legal

**Everything you host is served from your IP.** That includes the chains you sell (their state is public, and a buyer's chain can hold any bytes its own rules allow), and the files your sponsor pins for anyone who pays with work. Some of it may be unlawful where you live, or against your provider's terms. One abuse report can close a VPS account. On a home connection the exposure is yours.

**Size.** Your offer sets no limit: a buyer chooses their chain's block size, block time and state growth, and you host every chain on its path. A chain may fill every block, and your node keeps every block and the state they add. Price throughput and storage for what that costs you.

**You choose what to serve.** No protocol rule makes you host or serve anything:

- Without `--sponsor-url` (and without Hearth's tunnel) your sponsor listens on loopback only, so strangers can't pin files through it.
- `--storage-budget` caps what your sponsor holds pinned at once (default `256M`, lower than the sponsor's own 1 GiB). An upload that doesn't fit is refused. Lower it, or use a separate disk.
- `hearth stop --now` stops hosting at once. A chain you stop hosting is dropped from the node's configuration; what it stored stays on disk until you delete it.

**Take a file down** with:

```
hearth unpin <cid>
```

Run it beside the node (Hearth's own, or with `--root`). It asks the sponsor on `--sponsor-port` to take the file down (`POST /takedown`, with the node's cookie as the operator's credential): the sponsor unpins its Volumes on the node, frees their bytes from the budget, and refuses that CID from then on, for storing, work and payment. The seller must be running, since its sponsor serves the route. If the node refuses the unpin, the CID is still blocked: run `hearth unpin` again.

**Whoever paid for a file you take down loses the rest of its time. It is never refunded.**

`hearth status` shows how much of the budget is used.

Keep the operator token and the node's cookie to yourself: the seller holds an admin-equivalent credential over the node.

## Reachable by default, or key only

**Reachable by default.** On Hearth's own node, with neither `--sponsor-url` nor `--key-only`, `hearth sell` makes this machine reachable itself. It sets the node's public read port (`publicRead` 8082, taking writes), runs a small proxy that serves that port and your sponsor under `/sponsor`, and opens a Cloudflare quick tunnel to it (cloudflared, pinned and checked like the node; no account). It prints the tunnel's `https://….trycloudflare.com` URL. Nothing on the market names it.

The URL changes each time `hearth sell` starts. Quick tunnels are Cloudflare's free tier, without uptime promises: right for testnets and for trying Hearth. If the tunnel stops, the seller stops too (a service started by `--install` starts both again). With a node of your own, give `--public`; the node must already take public reads and writes on its public read port (Hearth says how). The port Hearth opened is closed again when you later sell without the tunnel; one you set yourself stays.

**Key only.** With `--key-only` (or a node of your own without `--public`) Hearth opens no tunnel. Your offer is the same either way: it names your key, not a URL.

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

`--install` writes services for both the node (Hearth's own, or the one at `--root`) and the seller, prints the commands that load them, and exits. It loads nothing itself. The seller's service runs with `--resume`, so a restart after `hearth stop` doesn't put the offer on sale again. Running `--install` itself counts as running `hearth sell` again: the service's next start sells.

**Linux.** systemd user units `~/.config/systemd/user/hearth-node.service` and `hearth.service` (which starts after the node and requires it). Load them with:

```
loginctl enable-linger $USER && systemctl --user daemon-reload && systemctl --user enable --now hearth-node.service hearth.service
```

`enable-linger` keeps your user's services running without a login, and starts them at boot.

**macOS.** launchd agents `~/Library/LaunchAgents/net.hearth.node.plist` and `net.hearth.seller.plist`, logging to `~/Library/Logs/hearth-node.log` and `hearth.log`. The command prints the `launchctl bootstrap` lines that load them. launchd has no ordering, so the seller waits for the node itself. A launch agent runs while you are logged in; the command also says how to make them LaunchDaemons for boot.

The node is started again after a failure exit (which includes a restart), but not after a clean stop or an invalid configuration. The seller is started again after a failure, not after it exits 0. If your node already runs under another service, load only the seller's.

Other systems are not supported by `--install`.

## Stop selling

```
hearth stop [--now]
```

`hearth stop` deletes your offer, so no order lands on it again. Orders already in a block are owed for their paid time. It waits (up to ten market blocks) for the deletion to land, and says which chains have paid time left and when the last ends.

The seller keeps running, closing: it writes no offer, and keeps hosting, mining and sponsoring each chain with paid time until that ends. Then it deletes its fragment and exits. With no seller running, `hearth stop` keeps only the paid chains in the fragment and says to start the seller with `--resume` to serve them.

`hearth stop --now` stops everything at once: the seller (its installed service, else the process), its miner and its sponsor, and hosting. It says what buyers lose: the rest of the time they paid for, with no refund.

`hearth sell` puts the offer on sale again.
