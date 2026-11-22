---
title: Burns
---

{% include nav.html %}

# Burns

This page explains how a host burns coin to show it has something at stake, and how buyers read those burns.

## Why sellers burn

A new host has no record. Buyers can't judge it by chains it has served. It can show it has something at stake by burning coin: sending it to an address nobody can spend from, named for the host.

A burn costs whoever made it. It says nothing about how a host serves. It is one more thing a buyer can weigh, beside the host's record and its domain.

## The burn address

Each host has its own burn address:

```
burnAddress(host) = cidV1DagCbor(encodeDagCbor({ domain: "lattice burn v1", host }))
```

`host` is the host's address. An account's address is the CID of `{ key }`, so no key has this address, and coin sent there is gone.

The burned total on a chain is the burn address's balance on that chain, 0 if it has never been used.

## Which chains count

Burns count on each chain from Nexus down to the market:

| Chain | Coin |
|---|---|
| `Nexus` | Nexus coin |
| `Nexus/testnet` | testnet coin |
| `Nexus/testnet/lodger-3129cc` | market coin |

Each chain's coin is its own. The web app shows burns per chain and never adds them together.

## Burning

From the seller's key, burn now with `hearth burn`:

```
hearth burn 300 --chain Nexus/testnet
```

It sends a transfer of 300 from the seller's key to its burn address on that chain. It prints the transaction and the burned total. It refuses a chain off the path from Nexus to the market. It reads the key from `<root>/hearth/seller.json`, or `--key`, and does not make one.

Or let the seller top up while it sells:

```
hearth sell --price 100/day --burn 5000
```

`--burn <coin>` burns until the burned total for your key reaches `<coin>`. It sends only what the total is missing, and nothing once the total is reached, whoever burned it. Running it again never burns again. `--burn-chain <path>` picks the chain; the default is the market.

The seller reads the key's nonce and the burned total at the same block. Until the burn is in a block, each pass sends the same transaction (same nonce, same amount), which can land only once. A burn waiting for coin never holds up hosting: it runs last in each pass.

See the [CLI reference](cli.md#hearth-burn) for every flag.

## Anyone can burn for a host

A burn is a plain transfer to the host's burn address. Anyone can send one from any account, with any wallet that can send a transfer, on any of the chains above. The host's burned total counts it the same way.

## How buyers read burns

The **Hosts** tab reads each host's burn address balance on each chain from the node it reads from:

```
GET /api/state/account/<burn address>?chainPath=<path>
```

A host with no record shows its burns in place of a record. A host with a record shows them under its name. Each line names the node it was read from.

The balance comes from the node's API, not from a record checked by CID. To be sure of it, open the web app with `?node=<your node's read URL>` and read it from your own node.
