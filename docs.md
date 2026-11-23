---
title: Hearth docs
---

{% include nav.html %}

![Hearth](assets/lockup-horizontal-color.svg){: width="320"}

This site explains how to use Hearth, as a buyer or as a host.

Hearth is a hosting market for app chains. You hire anyone to host your app's chain and pay them by the day, or by whatever period their offer names. Every read is checked in your browser, or on your own node.

There is no Hearth server. The market is itself a chain, `Nexus/hearth`, on [Lattice](https://lattice.build/docs/), the chain Hearth runs on. The web app reads the market from any node that hosts it and checks every record against its CID.

Hosts also make their chains' blocks. Every host makes zero-work blocks (about one hash each) on the market and on every chain it hosts, one per target block time, so a chain lives without miners. Each host also runs a sponsor, so your visitors can write and store with a few seconds of proof-of-work instead of coin. Anyone else may mine a chain too, for its block reward. See [Blocks and mining](mining.md).

Hearth runs on Lattice mainnet.

## Who it's for

**App developers** who want a chain for their app without running a node.

- [Launch a chain](launch.md): pick a host, pay, renew, and check the host.
- [FAQ](faq.md): what Hearth does not do.

**Hosts** who run a Lattice node and want to be paid to host chains on it. A host earns Hearth coin (the market's coin) from buyers' orders and from the blocks it makes, and spends it on hosting or storage from any seller.

- [Sell hosting](sell.md): one command, `hearth sell`.
- [Burns](burns.md): put coin at stake so new buyers can judge you.
- [Blocks and mining](mining.md): the blocks a host makes, where their rewards go, and how buyers check them.
- [CLI reference](cli.md): every subcommand and flag.

**Everyone**

- [How it works](how-it-works.md): the market chain, its records, verdicts, buying coin and the trust model.

## Source

The code is at [github.com/hearthnet/hearth](https://github.com/hearthnet/hearth). The package is `@hearthnet/hearth` and needs Node.js 22 or later. It is not on npm yet; install it from source:

```
git clone https://github.com/hearthnet/hearth && npm i -g ./hearth
```

(`npx @hearthnet/hearth` soon.)
