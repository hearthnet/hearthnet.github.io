---
title: CLI reference
---

{% include nav.html %}

# CLI reference

This page lists every `hearth` subcommand and flag, as `bin/hearth.mjs` defines them.

Install from source (not on npm yet; `npx @hearthnet/hearth` soon):

```
git clone https://github.com/hearthnet/hearth && npm i -g ./hearth
```

```
hearth <sell | status | stop | unpin | sweep | earn | burn | sell-coin> [flags]
```

`hearth`, `hearth help` and `hearth --help` print the usage and exit 0. `hearth <subcommand> --help` (or `-h`) prints that subcommand's usage and exits 0. Any other subcommand prints it and exits 2. An unknown flag is an error. A bad value prints `hearth: <reason>` and exits 2.

## Flags every subcommand takes

Every subcommand finds the node, the seller's key and the market with these flags:

| Flag | Default | Description |
|---|---|---|
| `--root <dir>` | `--root`, else the current directory, else `~/.lattice`: the first with `chains/Nexus/.cookie`; with none, Hearth's own (`~/.hearth/node`, made by `hearth sell`) | The node's root. A `--root` with no node there (no `lattice.json` or `chains/Nexus`) stops with `no node at <root>`. |
| `--node <url>` | none | A node's operator listener. Needs `--token-file`. |
| `--token-file <file>` | none | The operator token for `--node`. |
| `--key <file>` | `<root>/hearth/seller.json`, or `./hearth/seller.json` with `--node` | The seller's key. |
| `--lattice <path>` | Hearth's own | The node's command-line tool, for a node of your own. |
| `--market <path>` | `Nexus/testnet/hearth` | The market chain. The mainnet market, `Nexus/hearth`, still runs the previous market until it is relaunched; this version of Hearth doesn't sell there. |

`--node` needs `--token-file`.

## hearth sell

Sells hosting, in the foreground: writes your offer, mines every chain the node hosts (Hearth's own node unless you give one), hosts and starts the chains it is paid for, and runs the sponsor beside the node. See [Sell hosting](sell.md).

```
hearth sell [--throughput 6000] [--storage 600] [--period 1d] [--hashrate <hashes/s>] [--id 1]
            [--public | --key-only] [--market Nexus/testnet/hearth]
            [--root <node root> | --node <url> --token-file <file>] [--key <file>] [--lattice <path>]
            [--interval 30] [--fee <n>] [--once]
            [--sponsor-bin <path>] [--sponsor-port 8650] [--sponsor-url <https url>]
            [--storage-rate 10240] [--storage-budget 256M]
            [--burn <coin> [--burn-chain <path>]] [--install] [--resume]
```

| Flag | Default | Description |
|---|---|---|
| `--throughput <coin>` | `6000` | Whole market coin per kB/s of block capacity (`maxBlockSize ÷ targetBlockTime`), per period, for each chain on a buyer's path. |
| `--storage <coin>` | `600` | Whole market coin per kB of `maxStateGrowth`, per period, for each chain on a buyer's path. |
| `--period <n>` | `1d` | Market blocks, or hours, days or weeks (`12h`, `1d`, `2w`) at the market's block time. |
| `--hashrate <hashes/s>` | measured | The hash rate the offer claims. Without it, the rate the miner measures over its first minute; the offer waits for it. |
| `--id <id>` | `1` | The offer's id, with no slash or space. |
| `--public` | on Hearth's own node, without `--sponsor-url` or `--key-only` | Serve the node's public read port and the sponsor through a local proxy and a Cloudflare quick tunnel, and print its URL. See [Reachable by default](sell.md#reachable-by-default-or-key-only). |
| `--key-only` | off | No tunnel. |
| `--sponsor-bin <path>` | one on `PATH`, else Hearth's own | The sponsor to run beside the node. |
| `--sponsor-port <port>` | `8650` | Where the sponsor listens. |
| `--sponsor-url <url>` | none | The sponsor's public URL, `https` or loopback; the sponsor then listens on every interface. Without it, loopback only. The offer never names it. |
| `--storage-rate <n>` | `10240` | Bytes the sponsor pins per market coin. |
| `--storage-budget <size>` | `256M` | The most the sponsor holds pinned at once: bytes, or with `K`, `M`, `G`, `T` (binary). A takedown frees its bytes. |
| `--burn <coin>` | none | Burn, once, what your burned total lacks of this amount. |
| `--burn-chain <path>` | the market | With `--burn`: the chain to burn on, from Nexus down to the market. |
| `--fee <n>` | the chain's `minRelayFee`, else 1 | The fee for each transaction. |
| `--interval <s>` | `30` | Seconds between passes. |
| `--once` | off | One pass, then exit. |
| `--install` | off | Write systemd user units (Linux) or launchd agents (macOS) for the node and the seller, print the commands that load them, and exit. |
| `--resume` | off | Don't reopen an offer that `hearth stop` closed. The installed service runs with it. |

Every flag is checked before anything is written (the seller's key, the node's configuration).

Rules between flags:

- `--burn-chain` needs `--burn`.
- `--public` and `--key-only` don't go together, and neither goes with `--sponsor-url`.
- `--public` needs the node beside it: not with `--node`.
- `--price` is gone: it prints `--price is gone: an offer prices throughput and storage (--throughput, --storage, per --period)` and exits 2.

The offer stays on sale, changed in place when your flags change it, until `hearth stop`.

## hearth status

Prints once, without waiting (each read has a deadline; an unsynced market prints `market not synced (height X of Y)`), what the seller prints each pass: both keys' balances on the market, the seller's orders by verdict and its renewals, the coin they paid, the chains it owes hosting to and until which market block, the storage its sponsor has pinned against the budget, the market's cheapest coin on sale (parent coin per market coin), each of the seller key's open deposits and who paid for it, the key's balance on the market's parent, and whether a seller runs beside the node.

```
hearth status [--market Nexus/testnet/hearth] [--root <node root> | --node <url> --token-file <file>] [--key <file>]
              [--lattice <path>] [--sponsor-port 8650]
```

| Flag | Default | Description |
|---|---|---|
| `--sponsor-port <port>` | `8650` | Where the sponsor reports its storage (`GET /terms`). |

## hearth unpin

Takes a file down on the sponsor beside the node: `POST /takedown` with the node's cookie. The sponsor unpins it on the node and refuses its CID from then on. Whoever paid for it loses the rest of its time, with no refund. Run it beside the node; the seller (and so its sponsor) must be running. It needs a lattice-sponsor with the takedown route. See [Abuse, takedowns and legal](sell.md#abuse-takedowns-and-legal).

```
hearth unpin <cid> [--root <node root>] [--sponsor-port 8650]
```

| Flag | Default | Description |
|---|---|---|
| `<cid>` | none | The root CID of the file. |
| `--sponsor-port <port>` | `8650` | Where the sponsor listens. |

## hearth stop

Deletes the key's offer, so no order lands on it again; orders already in a block are owed. It waits (up to ten market blocks) for the deletion to land. The seller keeps hosting, mining and sponsoring each chain with paid time until that ends, then exits.

```
hearth stop [--now] [--market Nexus/testnet/hearth]
            [--root <node root> | --node <url> --token-file <file>] [--key <file>] [--lattice <path>] [--fee <n>]
```

| Flag | Default | Description |
|---|---|---|
| `--now` | off | Stop the seller (its installed service, else the process), its miner and its sponsor, and hosting, at once. Buyers lose the rest of their paid time, with no refund. |
| `--fee <n>` | the market's `minRelayFee`, else 1 | The fee for the deletion. |

## hearth sweep

Moves what the sponsor's key (`sponsor.json` beside the seller's key) holds above the float on one chain to the seller's address. It keeps the float, the fees the sponsor holds for transactions it signed, and its fund deposits. It refuses while the seller runs or a sponsor answers on `--sponsor-port`, and says how to pause the seller without closing its offer. The seller funds its own first writes from the sponsor's key, so you need not sweep to start selling.

```
hearth sweep [--float 100] [--chain <path>] [--sponsor-port 8650] [--market Nexus/testnet/hearth]
             [--root <node root> | --node <url> --token-file <file>] [--key <file>] [--lattice <path>] [--fee <n>]
```

| Flag | Default | Description |
|---|---|---|
| `--float <coin>` | `100` | What the sponsor's key keeps. |
| `--chain <path>` | the market | The chain to sweep, from Nexus down to the market. |
| `--sponsor-port <port>` | `8650` | Where to check that no sponsor is running. |
| `--fee <n>` | the chain's `minRelayFee`, else 1 | The transfer's fee. |

## hearth earn

Earns market coin for the seller's address with this machine's processor, through a fee sponsor's payouts. Makes the seller's key if there is none.

```
hearth earn [--until <coins>] [--workers 1] --sponsor <url>
            [--market Nexus/testnet/hearth] [--root <node root> | --node <url> --token-file <file>] [--key <file>] [--lattice <path>]
```

| Flag | Default | Description |
|---|---|---|
| `--until <coins>` | two of the market's `minRelayFee` (the offer and its close) | Stop once the address holds this much on the market. |
| `--workers <n>` | `1` | Threads doing work. |
| `--sponsor <url>` | none (required) | A fee sponsor that pays out market coin for work. |

## hearth burn

Burns coin for the seller's key: a plain transfer to its burn address on one chain. See [Burns](burns.md).

```
hearth burn <coin> --chain <path> [--market Nexus/testnet/hearth]
            [--root <node root> | --node <url> --token-file <file>] [--key <file>] [--lattice <path>] [--fee <n>]
```

| Argument or flag | Default | Description |
|---|---|---|
| `<coin>` | required | Whole coin to burn, 1 to 9007199254740991. |
| `--chain <path>` | required | The chain to burn on. Must be on the path from Nexus down to `--market`. |
| `--fee <n>` | the chain's `minRelayFee`, else 1 | The transaction's fee. |

The key must exist. It prints the amount, the burn address, the transaction, and the burned total once it is in a block:

```
burned 300 on Nexus for <address>
to <burn address>
transaction <cid>
burned total on Nexus: 300 once it is in a block (was 0)
```

Each run burns again. To burn only up to a total, use `hearth sell --burn`.

## hearth sell-coin

Puts the seller key's market coin on sale on the chains' own exchange: a deposit on the market that a buyer takes by paying `--price` on the market's parent chain. See [Sell your coin](sell.md#sell-your-coin-hearth-sell-coin).

```
hearth sell-coin <coin> --price <parent coin> [--float 100] [--again] [--sponsor-port 8650] [--market Nexus/testnet/hearth]
                 [--root <node root> | --node <url> --token-file <file>] [--key <file>] [--lattice <path>] [--fee <n>]
```

| Argument or flag | Default | Description |
|---|---|---|
| `<coin>` | required | Whole market coin to lock, 1 to 9007199254740991. |
| `--price <parent coin>` | required | Whole parent-chain coin a buyer pays for all of it. |
| `--float <coin>` | `100` | What the key keeps for fees: it refuses to lock more than its balance less the float and the fee. |
| `--again` | off | Lock it even if the key already has an open deposit of the same amount and price. Without it, that deposit is printed and nothing is sent. |
| `--sponsor-port <port>` | `8650` | It refuses while a sponsor answers here. |
| `--fee <n>` | the market's `minRelayFee`, else 1 | The transaction's fee. |

The key must exist. Every amount is checked before the node is asked anything. Like `hearth sweep`, it refuses while the seller runs (its pid file) or a sponsor answers on `--sponsor-port`. It waits for the market to sync; each read then has a 15 s deadline and the whole a 120 s one. The deposit's nonce is the key's account nonce. It prints the deposit, the transaction, and what a buyer pays. The buyer's payment lands in the key's address on the parent: there is nothing to claim. A deposit can't be cancelled: there is no cancel or refund, so it stays locked until someone buys it (the seller included, paying its price to itself).
