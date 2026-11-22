---
title: Hearth docs
---

{% include nav.html %}

![Hearth](assets/lockup-horizontal-color.svg){: width="320"}

This site explains how to use Hearth, as a buyer or as a host.

Hearth is a hosting market for app chains. You hire anyone to host your app's chain and pay them by the day, or by whatever period their offer names. Every read is checked in your browser, or on your own node.

There is no Hearth server. The market is itself a chain, `Nexus/testnet/lodger-3129cc`, on [Lattice](https://lattice.build/docs/), the chain Hearth runs on. The web app reads the market from any node that hosts it and checks every record against its CID.

Hosts only host. A chain's blocks come from whoever mines it or commits it on its parent. A host may also claim to mine the chains it sells; see [Mining](mining.md).

Hearth runs on testnet only.

## Who it's for

**App developers** who want a chain for their app without running a node.

- [Launch a chain](launch.md): pick a host, pay, renew, and check the host.
- [FAQ](faq.md): what Hearth does not do.

**Hosts** who run a Lattice node and want to be paid to host chains on it.

- [Sell hosting](sell.md): one command, `npx @hearthnet/hearth sell --price 100/day`.
- [Burns](burns.md): put coin at stake so new buyers can judge you.
- [Mining](mining.md): claim a hashrate on the chains you sell.
- [CLI reference](cli.md): every subcommand and flag.

**Everyone**

- [How it works](how-it-works.md): the market chain, its records, verdicts, buying coin and the trust model.

## Source

The code is at [github.com/hearthnet/hearth](https://github.com/hearthnet/hearth). The package is `@hearthnet/hearth` and needs Node.js 22 or later.
