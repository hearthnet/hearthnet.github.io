---
title: Sell hosting
---

{% include nav.html %}

# Sell hosting

This page shows how to sell hosting from your Lattice node with one command, keep it running, and stop it.

A host does three things for each chain it sells: it hosts the chain on its node, makes the chain's blocks, and runs a sponsor that pays visitors' write fees in return for proof-of-work.

## Prerequisites

- A running Lattice node, **4.0.1 or newer** (`lattice up`). See the [Lattice docs](https://lattice.build/docs/). The seller checks the node's features (its `GET /config` saying which layer lists each chain) and stops beside an older one with "Hearth needs Lattice 4.0.1 or newer (you have X: ...)".
- `lattice-sponsor` on your `PATH`, or `--sponsor-bin <path>`. See the [Lattice docs](https://lattice.build/docs/) for where to get it.
- Node.js 22 or later.
- Hearth itself, from source (it is not on npm yet; `npx @hearthnet/hearth` soon):

```
git clone https://github.com/hearthnet/hearth && npm i -g ./hearth
```

## The command

In the node's root, or with `--root`:

```
hearth sell --price 100/day
```

This sells hosting for up to 50 chains under `Nexus/testnet`, at 100 market coin per day, for 30 days, on the market `Nexus/testnet/lodger-3129cc`. It runs in the foreground.

**The first sync of testnet takes about 1–2 hours and ~1.5 GB.** Until the node has synced the market, the seller prints `syncing market: h / H` (its tip against the best header its peers showed) and reads, funds, writes and burns nothing.

## What it does

1. **Finds the node.** It uses `--root`, else the current directory, else `~/.lattice` (where a bare `lattice-node` keeps its storage): the first that holds a running node's `chains/Nexus/.cookie`. With `--node` and `--token-file` it uses a node elsewhere instead.
2. **Checks the node** is Lattice 4.0.1 or newer, by its features.
3. **Makes the seller's key** at `<root>/hearth/seller.json` unless there is one. The file is mode 0600. It prints the address, never the key. It refuses a key file that others can read.
4. **Writes the offer file** from the flags, once, at `<root>/hearth/<offer id>.json`. The offer id comes from the terms, so the same flags find the same file.
5. **Runs the seller** (`host/sell-hosting.mjs`) in the foreground. Beside the node it starts lattice-sponsor (below). Every `--interval` seconds (default 30) it makes one pass:
   1. It adds the market and the offer's parent to its own fragment of the node's configuration, `hostmarket` (`config.d/hostmarket.json`), so the node hosts them.
   2. Once the node has synced the market, it reads the market at its tip and judges its own orders, the same way every reader does.
   3. Once the key holds enough market coin, it writes the offer (and the URL record, if `--url`), then the storage offer, then any changed URL.
   4. It creates each owed chain not yet on its parent, once, from the buyer's payload (`POST /chains`), and lists every owed chain in the `hostmarket` fragment.
   5. With `--burn`, it burns what the burned total is missing.
6. **Makes blocks** on the market and on every chain it hosts for its orders (below).

A chain whose paid time ran out is left out of the fragment. The node stops hosting it unless another layer of its configuration lists it; what it stored stays. The seller never restarts the node. A failed pass is logged and tried again on the next one.

## Blocks

The seller makes the blocks of the market and of every chain it hosts for its orders. There is no flag: it always does. Each is a zero-work block, about one hash, made with this machine's processor. A chain is due once its tip is one target block time (`targetBlockTime`) old, from anyone's block, so a chain others keep moving needs nothing from it. A chain the node is still syncing is never paced.

Every block pays the sponsor's key, never the seller's.

One caveat today: one hash can clear several chains in a template. The seller skips a hash that would also clear a harder chain not yet due, but it can't keep a hash off an easier chain not yet due, so that chain can get some early blocks. ASERT difficulty adjustment hardens it back toward its target block time. A per-chain `exclude` in template requests, in a coming Lattice node release, will fix this.

See [Blocks and mining](mining.md) for how buyers check the blocks.

## Sponsor

Beside the node the seller runs lattice-sponsor, and starts it again, with backoff, whenever it exits. Its key is `<root>/hearth/sponsor.json` (mode 0600), never the seller's: it owns nothing but its float, which the blocks pay.

The sponsor:

- pays fees for visitors who pay with proof-of-work shares, so they write with no coin;
- sells permanent pins for work at `--storage-rate` bytes per market coin (default 10240);
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

It moves what the sponsor's key holds on one chain (default the market), keeping the float, the fees it holds for transactions it signed, and any fund deposits. It signs with the sponsor's key, so it refuses while the seller runs, or while any lattice-sponsor answers on `--sponsor-port`. Stop them first.

## Funding

Every write on the market costs a fee in market coin. Before it writes, the seller checks that its key holds three fees: for the offer, the URL record and a storage offer. Until it does, it prints:

```
Send at least <n> coin on Nexus/testnet/lodger-3129cc to <address>, from any account that holds some, or earn it with this machine: hearth earn; this seller waits and checks again every 30 s.
```

Send market coin to the address from any account, buy it in the web app (see [Buy market coin](launch.md#1-buy-market-coin)), or earn it:

```
hearth earn [--until <coins>] [--workers 1] [--sponsor <url>]
```

`hearth earn` does work for a fee sponsor's payouts with this machine's processor, naming the seller's address. The sponsor pays the address in coin each time its earnings reach its payout. It stops once the address holds `--until` coins on the market (by default four fees: the seller's first writes and a close). The default sponsor is `https://lattice-testnet-apps-iad.fly.dev:8650/sponsor`.

Buyers pay you in market coin, to the seller's address.

## Offers: 30 days, versions, no auto-renew

An offer sells for 30 days from when it is written (or its file's `duration`, in market blocks). It never renews by itself. To sell for another 30 days, run `hearth sell` again: it writes a new version of the offer.

Each write is a **version** of the offer. A version takes effect 10 market blocks after it is written. Each payment is priced by the version in force at the payment's own height, so a new price applies only to orders yet to come. After the first write, the seller writes a new version only when the address its blocks pay or its sponsor changes.

Each set of flags names its own offer. Running again with the same flags finds the same offer. Running with other flags writes a new offer beside the old one, and the seller says the old one stays on sale. The seller keeps hosting the parents of all your offers still on sale. With `--id`, the offer file `<id>.json` is written only once: changing other flags with the same `--id` keeps the old terms.

## Flags

| Flag | Default | What it does |
|---|---|---|
| `--price <coin>/<period>` | none | Whole market coin per period. The period is a number of market blocks, or `hour`, `day` or `week`, converted to market blocks at the market's block time. Give `--price` or `--offer`, not both. |
| `--offer <file>` | none | An offer file, instead of `--price`. |
| `--parent <path>` | `Nexus/testnet` | The chain the offer's chains go under. |
| `--chains <n>` | `50` | How many chains the offer sells. |
| `--id <id>` | from the terms | The offer's id, with `--price`. Without it: `o-` and 10 hex digits of a hash of the terms. |
| `--market <path>` | `Nexus/testnet/lodger-3129cc` | The market chain. |
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
| `--burn <coin>` | none | Burn until the burned total for your key reaches this amount. |
| `--burn-chain <path>` | the market | Where `--burn` burns. Only with `--burn`. |
| `--fee <n>` | the chain's minimum relay fee, else 1 | The fee for each write and burn. |
| `--interval <s>` | `30` | Seconds between passes. |
| `--once` | off | Make one pass and exit. |
| `--lattice <path>` | `lattice` | The `lattice` command. |
| `--install` | off | Write services for the node and the seller, print how to load them, and exit. |
| `--resume` | off | Write no offer version on start (what the installed service runs). |

### Offer terms

`--price` alone sells these terms:

| Term | Value |
|---|---|
| `capacity` | 50 chains (`--chains`) |
| `maxBlockSize` | 1,000,000 bytes |
| `maxStateGrowth` | 1,000,000 bytes |
| `minTargetBlockTime` | 5,000 ms |
| `maxNumberOfTransactionsPerBlock` | 1,000 |
| `modules` | `any` |
| `duration` | 30 days of market blocks from when the offer is written |

To set other terms, write an offer file and pass `--offer`. `period` is in market blocks, or `"hour"`, `"day"` or `"week"`; `duration` is in market blocks (30 days when absent):

```json
{ "id": "small", "parent": "Nexus/testnet", "price": 100, "period": 1440, "duration": 43200,
  "capacity": 50, "maxBlockSize": 1000000, "maxStateGrowth": 1000000, "minTargetBlockTime": 5000,
  "maxNumberOfTransactionsPerBlock": 1000, "modules": "any" }
```

`modules` is `"any"` or a list of policy module CIDs. Without `id`, the offer's id is `1`.

## A domain, or key only

**Key only.** Without `--url` you write no URL record. You claim no domain. The web app lists your offers under your key, marked *Key only*, for buyers to judge by your record. It can't probe the chains you serve, since you name no URL.

**A domain.** With `--url https://host.example`, the seller writes a URL record naming your public read URL. Then serve this file at `https://host.example/.well-known/lattice.json`:

```json
{ "sellers": ["<your seller address>"] }
```

Your offers then carry the domain's name and *Verified*. One domain can list several sellers. If the file is somewhere else on the same hostname, write where with `--proof`:

```
hearth sell --price 100/day --url https://host.example \
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
hearth sell --price 100/day --node http://10.0.0.5:8090 --token-file seller.token
```

It runs no sponsor there (see [Sponsor](#sponsor)). A token that may write a fragment may set any key in it, so it controls the node nearly as fully as `admin` anyway.

## Run it as a service

```
hearth sell --price 100/day --install
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
