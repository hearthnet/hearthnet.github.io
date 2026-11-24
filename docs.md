---
title: Hearth docs
---

{% include nav.html %}

![Hearth](assets/lockup-horizontal-color.svg){: width="320"}

This site explains how to use Hearth, as a buyer or as a host.

Hearth is a hosting market for app chains. You hire anyone to host your app's chain and pay them by the day, or by whatever period their offer names. Every read is checked in your browser, or on your own node.

There is no Hearth server. The market is itself a chain. The web app reads the market from any node that hosts it and checks every record against its CID.

You mint your chain's first block (its genesis) in your browser, and the order that pays a host carries it. The host starts your chain from that block and mines it. Every host mines every chain it hosts, all the time, with all of its hash rate; its offer says how much that is, and the web app shows what its blocks delivered. Anyone else may mine a chain too, for its block reward. See [Blocks and mining](mining.md).

The current market runs on testnet, at `Nexus/testnet/hearth`. The mainnet market, `Nexus/hearth`, still runs the previous market until it is relaunched.

## Who it's for

**App developers** who want a chain for their app without running a node.

- [Launch a chain](launch.md): describe your chain, pick hosts, pay, renew, and check the hosts.
- [FAQ](faq.md): what Hearth does not do.

**Hosts** who have a machine that stays online and want to be paid to host chains on it. A host earns Hearth coin (the market's coin) from buyers' orders and from the blocks it makes, and spends it on hosting from any seller.

- [Sell hosting](sell.md): one command, `hearth sell`.
- [Burns](burns.md): put coin at stake so new buyers can judge you.
- [Blocks and mining](mining.md): how a host mines, where the rewards go, and how buyers check its hash rate.
- [CLI reference](cli.md): every subcommand and flag.

**Everyone**

- [How it works](how-it-works.md): the market chain, its records, prices, verdicts, buying coin and the trust model.

## Source

The code is at [github.com/hearthnet/hearth](https://github.com/hearthnet/hearth). The package is `@hearthnet/hearth` and needs Node.js 22 or later. It is not on npm yet; install it from source:

```
git clone https://github.com/hearthnet/hearth && npm i -g ./hearth
```

(`npx @hearthnet/hearth` soon.)
