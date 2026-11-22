---
title: Sell hosting
---

{% include nav.html %}

# Sell hosting

This page shows how to sell hosting from your Lattice node with one command, and how to keep it running or stop it.

You host. You don't make blocks. Your node carries each chain you host in its block templates for the parent, and whoever mines or commits it makes its blocks. If you also mine, you can say so in your offer; see [Mining](mining.md).

## Prerequisites

- A running Lattice node (`lattice up`). See the [Lattice docs](https://lattice.build/docs/).
- Node.js 22 or later.

## The command

In the node's root, or with `--root`:

```
npx @hearthnet/hearth sell --price 100/day
```

This sells hosting for up to 50 chains under `Nexus/testnet`, at 100 market coin per day, on the market `Nexus/testnet/lodger-3129cc`. It runs in the foreground.

## What it does

1. **Finds the node.** It uses `--root`, else the current directory, else `~/.lattice` (where a bare `lattice-node` keeps its storage): the first that holds a running node's `chains/Nexus/.cookie`. With `--node` and `--token-file` it uses a node elsewhere instead.
2. **Makes the seller's key** at `<root>/hearth/seller.json` unless there is one, in the format `lattice key generate` writes. The file is mode 0600. It prints the address, never the key. It refuses a key file that others can read.
3. **Writes the offer file** from the flags, once, at `<root>/hearth/<offer id>.json`. The offer id comes from the terms, so the same flags find the same file.
4. **Runs the seller.** Every `--interval` seconds (default 30) it makes one pass:
   1. It reads the node's loopback port with `lattice config get rpc`.
   2. It adds the market and the offer's parent to its own fragment of the node's configuration, `hostmarket` (`config.d/hostmarket.json`), so the node hosts them.
   3. It reads the market at its tip and judges its own orders, the same way every reader does.
   4. If the offer claims mining, it prints the miner entries for the chains it owes whenever they change.
   5. Once the key holds enough market coin, it writes the offer (and the URL record, if `--url`), then the storage offer, then any changed URL. It writes one of these per pass.
   6. It creates each owed chain not yet on its parent, once, from the buyer's payload (`POST /chains`). It lists every owed chain in the `hostmarket` fragment.
   7. With `--burn`, it burns what the burned total is missing.

A chain whose paid time ran out is left out of the fragment. The node stops hosting it unless another layer of its configuration lists it.

The seller never restarts the node. A failed pass is logged and tried again on the next one.

## Flags

| Flag | Default | What it does |
|---|---|---|
| `--price <coin>/<period>` | none | The price: whole market coin per period. The period is a number of market blocks, or `hour`, `day` or `week`. A unit is converted to market blocks at the market's target block time when the offer is written (at least 1 block). Give `--price` or `--offer`, not both. |
| `--parent <path>` | `Nexus/testnet` | The chain the offer's chains go under. Each order's path must be directly under it. |
| `--chains <n>` | `50` | How many chains the offer sells. |
| `--id <id>` | from the terms | The offer's id, with `--price`. Without it the id is `o-` and 10 hex digits of a hash of the terms. |
| `--offer <file>` | none | An offer file, instead of `--price`. |
| `--market <path>` | `Nexus/testnet/lodger-3129cc` | The market chain. |
| `--key <file>` | `<root>/hearth/seller.json` | The seller's key. Made if missing. With `--node`, the default is `hearth/seller.json` in the current directory. |
| `--root <dir>` | found (see above) | The node's root. The seller uses its loopback API and cookie. |
| `--node <url>` | none | A node's operator listener, for a seller running elsewhere. Needs `--token-file`. |
| `--token-file <file>` | none | An operator token for `--node`. |
| `--url <url>` | none | Your public read URL. Without it you sell key only. |
| `--proof <url>` | `<url's origin>/.well-known/lattice.json` | Where your domain lists your address, if not the default. Must be `https`, on `--url`'s hostname, at `/.well-known/lattice.json`. |
| `--storage-offer <file>` | none | Also write a storage offer from this file. |
| `--mine <rate>` | none | Claim a mining rate on the chains you sell, in hashes per second: `1TH/s`, `500GH/s`, or a plain whole number. Units `kH/s` to `EH/s`. Only with `--price`. |
| `--burn <coin>` | none | Burn until the burned total for your key reaches this amount. |
| `--burn-chain <path>` | the market | Where `--burn` burns. Must be on the path from Nexus to the market. Only with `--burn`. |
| `--fee <n>` | the chain's minimum relay fee, else 1 | The fee for each write and burn. |
| `--interval <s>` | `30` | Seconds between passes. |
| `--once` | off | Make one pass and exit. |
| `--lattice <path>` | `lattice` | The `lattice` command, used to read the node's port. |
| `--install` | off | Write a service that runs the same seller, print how to load it, and exit. |

Amounts (`--price` coin, `--chains`, `--burn`) are whole numbers from 1 to 9007199254740991.

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
| `duration` | 525,600 market blocks from when the offer is written |

To set other terms, write an offer file and pass `--offer`. `period` is in market blocks, or `"hour"`, `"day"` or `"week"`:

```json
{ "id": "small", "parent": "Nexus/testnet", "price": 100, "period": 1440, "duration": 525600,
  "capacity": 50, "maxBlockSize": 1000000, "maxStateGrowth": 1000000, "minTargetBlockTime": 5000,
  "maxNumberOfTransactionsPerBlock": 1000, "modules": "any" }
```

`modules` is `"any"` or a list of policy module CIDs. You may give `until` (a market height) instead of `duration`. Add `"mines"` (hashes per second) to claim mining. Without `id`, the offer's id is `1`.

## Funding

Every write on the market costs a fee in market coin. Before it writes, the seller checks that its key holds three fees: for the offer, the URL record and a storage offer. Until it does, it prints:

```
Send at least <n> coin on Nexus/testnet/lodger-3129cc to <address>, from any account that holds some; this seller waits and checks again every 30 s.
```

Send market coin to the address from any account. You can buy it in the web app; see [Buy market coin](launch.md#1-buy-market-coin).

Buyers pay you in market coin, to the same address.

## A domain, or key only

**Key only.** Without `--url` you write no URL record. You claim no domain. The web app lists your offers under your key, marked *Key only*, for buyers to judge by your record. It can't probe the chains you serve, since you name no URL.

**A domain.** With `--url https://host.example`, the seller writes a URL record naming your public read URL. Then serve this file at `https://host.example/.well-known/lattice.json`:

```json
{ "sellers": ["<your seller address>"] }
```

Your offers then carry the domain's name and *Verified*. One domain can list several sellers.

If the file is somewhere else on the same hostname (another port, for example), write where with `--proof`:

```
npx @hearthnet/hearth sell --price 100/day --url https://host.example \
  --proof https://host.example:8650/.well-known/lattice.json
```

Readers fetch the file without following redirects and give up after 8 seconds. The web app fetches it from the buyer's browser, so serve it with `Access-Control-Allow-Origin: *`.

If the domain does not list your address, or the file can't be read, your offers are hidden. Anyone could name someone else's URL, so a failing URL record is worse than none.

A URL record stays once written. Dropping `--url` later does not make you key only again.

**Storage offers need a domain.** They are offered only from verified hosts.

## Storage offers

If you run a storage sponsor (lattice-sponsor with `--sell-storage`, paid to your seller address with `--pay-to`, at a `--storage-rate` on the market), you can also sell storage. Pass `--storage-offer storage.json`:

```json
{ "id": "files", "sponsor": "https://host.example/sponsor", "bytesPerCoin": 1048576, "capacity": 10737418240, "duration": 525600 }
```

| Field | What it is |
|---|---|
| `id` | Required. Not the hosting offer's id. |
| `sponsor` | Your sponsor's `https` URL. |
| `bytesPerCoin` | Bytes kept for good per market coin. |
| `capacity` | Bytes the offer sells in all. |
| `until` or `duration` | When it ends: a market height, or market blocks from now (default 525,600). |

Buyers see it as verified only if your domain lists your address, and your sponsor's `GET /terms` names your address as `payTo` and the offer's `bytesPerCoin` as its rate for the market. The sponsor does the selling. The seller does nothing for storage orders.

## A node elsewhere

The seller can run away from the node, through the node's operator listener (`operatorListen` in its configuration), with a token:

```
lattice tokens create seller --role admin --out seller.token --root <root>
npx @hearthnet/hearth sell --price 100/day --node http://10.0.0.5:8090 --token-file seller.token
```

Of the node's configuration it uses `GET /config`, `GET /config/fragments/hostmarket`, `PUT /config/fragments/hostmarket` and `POST /chains`. It also reads the market and its chains (`GET /api/...`) and submits transactions (`POST /transactions`). A token that may write a fragment may set any key in it, so it controls the node nearly as fully as `admin` anyway.

## Run it as a service

```
npx @hearthnet/hearth sell --price 100/day --install
```

`--install` writes a service that runs the same seller with the same flags and absolute paths, prints the command that loads it, and exits. It loads nothing itself. Run it with the same flags you sell with.

**Linux.** It writes a systemd user unit, `~/.config/systemd/user/hearth.service`, which restarts the seller 30 seconds after it stops. Load it with:

```
loginctl enable-linger $USER && systemctl --user daemon-reload && systemctl --user enable --now hearth.service
```

`enable-linger` keeps your user's services running without a login, and starts them at boot.

**macOS.** It writes a launchd agent, `~/Library/LaunchAgents/build.lattice.hearth.plist`, which logs to `~/Library/Logs/hearth.log`. Load it with:

```
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/build.lattice.hearth.plist
```

A launch agent runs while you are logged in. To run it from boot, copy it to `/Library/LaunchDaemons`, add `<key>UserName</key><string>$USER</string>`, and load it with:

```
sudo launchctl bootstrap system /Library/LaunchDaemons/build.lattice.hearth.plist
```

Other systems are not supported by `--install`.

## Offers can't be withdrawn

An offer sells until it ends. Each set of flags names its own offer. Running again with the same flags finds the same offer. Running with other flags writes a new offer beside the old one, and the seller says the old one stays on sale.

The seller keeps hosting the parents of all your offers still on sale, so chains owed on an old offer stay hosted.

With `--id`, the offer file `<id>.json` is written only once. Changing other flags with the same `--id` keeps the old terms.

## Stop hosting

Stopping the seller leaves its fragment, so the node keeps hosting what it lists. To stop hosting it, empty the fragment:

```
echo '{}' > empty.json && lattice apply --fragment hostmarket --file empty.json --root <root>
```

or delete it through the API:

```
curl -X DELETE -H "Authorization: Bearer $(cut -d: -f2 <root>/chains/Nexus/.cookie)" \
  http://127.0.0.1:<rpc>/config/fragments/hostmarket
```

Your offers stay on sale until they end. Buyers see the chains you stop serving on the **Hosts** tab, under *Serving now*.
