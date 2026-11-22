---
title: CLI reference
---

{% include nav.html %}

# CLI reference

This page lists every `hearth` subcommand and flag, as `bin/hearth.mjs` defines them.

```
npx @hearthnet/hearth <sell | burn> [flags]
```

`hearth`, `hearth help` and `hearth --help` print the usage and exit 0. Any other subcommand prints it and exits 2. An unknown flag is an error. A bad value prints `hearth: <reason>` and exits 2.

## hearth sell

Sells hosting from a running Lattice node, in the foreground. See [Sell hosting](sell.md) for what it does.

```
hearth sell --price <coin>/<period> [--parent Nexus/testnet] [--chains 50]
            [--url https://host.example [--proof <url>]] [--market Nexus/testnet/lodger-3129cc]
            [--root <node root> | --node <url> --token-file <file>] [--key <file>] [--offer offer.json]
            [--id <offer id>] [--storage-offer storage.json] [--lattice lattice] [--interval 30] [--fee <n>] [--once]
            [--mine <rate, e.g. 1TH/s>] [--burn <coin> [--burn-chain <path>]] [--install]
```

| Flag | Default | Description |
|---|---|---|
| `--price <coin>/<period>` | none | Whole market coin per period; the period in market blocks or `hour`, `day`, `week`. Exactly one of `--price` and `--offer`. |
| `--offer <file>` | none | An offer file instead of `--price`. |
| `--parent <path>` | `Nexus/testnet` | With `--price`: the parent the offer's chains go under. |
| `--chains <n>` | `50` | With `--price`: how many chains the offer sells. |
| `--id <id>` | `o-` + 10 hex digits of a hash of the terms | With `--price`: the offer's id. |
| `--mine <rate>` | none | With `--price`: claim this many hashes per second (`1000`, `500GH/s`, `1TH/s`; `kH/s` to `EH/s`). |
| `--market <path>` | `Nexus/testnet/lodger-3129cc` | The market chain. |
| `--root <dir>` | `--root`, else the current directory, else `~/.lattice`: the first with `chains/Nexus/.cookie` | The node's root. |
| `--node <url>` | none | A node's operator listener. Needs `--token-file`. |
| `--token-file <file>` | none | The operator token for `--node`. |
| `--key <file>` | `<root>/hearth/seller.json`, or `./hearth/seller.json` with `--node` | The seller's key. Made, mode 0600, if missing. |
| `--url <url>` | none | Your public read URL. Without it, you sell key only. |
| `--proof <url>` | `<url's origin>/.well-known/lattice.json` | Where your domain lists your address: `https`, same hostname, path `/.well-known/lattice.json`. |
| `--storage-offer <file>` | none | Also write a storage offer from this file. |
| `--burn <coin>` | none | Burn, once, what your burned total lacks of this amount. |
| `--burn-chain <path>` | the market | With `--burn`: the chain to burn on, from Nexus down to the market. |
| `--fee <n>` | the chain's `minRelayFee`, else 1 | The fee for each transaction. |
| `--interval <s>` | `30` | Seconds between passes. |
| `--once` | off | One pass, then exit. |
| `--lattice <path>` | `lattice` | The `lattice` command, used to read the node's port (`lattice config get rpc`). |
| `--install` | off | Write a systemd user unit (Linux) or launchd agent (macOS) that runs this command, print the command that loads it, and exit. |

Rules between flags:

- `--node` needs `--token-file`.
- `--mine` needs `--price`. With `--offer`, put `"mines"` in the file.
- `--burn-chain` needs `--burn`.
- `--id`, `--parent`, `--chains` and `--mine` apply only with `--price`.

Whole amounts (`--price` coin and blocks, `--chains`, `--burn`, `--mine`'s number) run from 1 to 9007199254740991.

## hearth burn

Burns coin for the seller's key: a plain transfer to its burn address on one chain. See [Burns](burns.md).

```
hearth burn <coin> --chain <path> [--market Nexus/testnet/lodger-3129cc]
            [--root <node root> | --node <url> --token-file <file>] [--key <file>] [--lattice lattice] [--fee <n>]
```

| Argument or flag | Default | Description |
|---|---|---|
| `<coin>` | required | Whole coin to burn, 1 to 9007199254740991. |
| `--chain <path>` | required | The chain to burn on. Must be on the path from Nexus down to `--market`. |
| `--market <path>` | `Nexus/testnet/lodger-3129cc` | The market, which sets which chains count. |
| `--root <dir>` | as for `sell` | The node's root. |
| `--node <url>` | none | A node's operator listener. Needs `--token-file`. |
| `--token-file <file>` | none | The operator token for `--node`. |
| `--key <file>` | `<root>/hearth/seller.json`, or `./hearth/seller.json` with `--node` | The key to burn from. It must exist. |
| `--lattice <path>` | `lattice` | The `lattice` command, used to read the node's port. |
| `--fee <n>` | the chain's `minRelayFee`, else 1 | The transaction's fee. |

It prints the amount, the burn address, the transaction, and the burned total once it is in a block:

```
burned 300 on Nexus/testnet for <address>
to <burn address>
transaction <cid>
burned total on Nexus/testnet: 300 once it is in a block (was 0)
```

Each run burns again. To burn only up to a total, use `hearth sell --burn`.
