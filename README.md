# Multi Collateral USDV

[comment]: <> ([![Build Status]&#40;https://travis-ci.com/makerdao/dss.svg?branch=master&#41;]&#40;https://travis-ci.com/makerdao/dss&#41;)

[comment]: <> ([![codecov]&#40;https://codecov.io/gh/makerdao/dss/branch/master/graph/badge.svg&#41;]&#40;https://codecov.io/gh/makerdao/dss&#41;)

This repository contains the core smart contract code for Multi
Collateral USDV. This is a high level description of the system, assuming
familiarity with the basic economic mechanics as described in the
whitepaper.

## Additional Documentation

[comment]: <> (`dss` is also documented in the [wiki]&#40;https://github.com/makerdao/dss/wiki&#41; and in )
[DEVELOPING.md](https://github.com/velerofinance/dss/blob/master/DEVELOPING.md)

## Design Considerations

- Token agnostic
  - system doesn't care about the implementation of external tokens
  - can operate entirely independently of other systems, provided an authority assigns
    initial collateral to users in the system and provides price data.

- Verifiable
  - designed from the bottom up to be amenable to formal verification
  - the core cdp and balance database makes *no* external calls and
    contains *no* precision loss (i.e. no division)

- Modular
  - multi contract core system is made to be very adaptable to changing
    requirements.
  - allows for implementations of e.g. auctions, liquidation, CDP risk
    conditions, to be altered on a live system.
  - allows for the addition of novel collateral types (e.g. whitelisting)


## Collateral, Adapters and Wrappers

Collateral is the foundation of USDV and USDV creation is not possible
without it. There are many potential candidates for collateral, whether
native velas, ERC20 tokens, other fungible token standards like ERC777,
non-fungible tokens, or any number of other financial instruments.

Token wrappers are one solution to the need to standardise collateral
behaviour in USDV. Inconsistent decimals and transfer semantics are
reasons for wrapping. For example, the WVLX token is an ERC20 wrapper
around native velas.

In MCD, we abstract all of these different token behaviours away behind
*Adapters*.

Adapters manipulate a single core system function: `slip`, which
modifies user collateral balances.

Adapters should be very small and well defined contracts. Adapters are
very powerful and should be carefully vetted by VDGT holders. Some
examples are given in `join.sol`. Note that the adapter is the only
connection between a given collateral type and the concrete on-chain
token that it represents.

There can be a multitude of adapters for each collateral type, for
different requirements. For example, VLX collateral could have an
adapter for native velas and *also* for WVLX.


## The USDV Token

The fundamental state of a USDV balance is given by the balance in the
core (`vat.dai`, sometimes referred to as `D`).

Given this, there are a number of ways to implement the USDV that is used
outside of the system, with different trade offs.

*Fundamentally, "USDV" is any token that is directly fungible with the
core.*

In the Kovan deployment, "USDV" is represented by an ERC20 DSToken.
After interacting with CDPs and auctions, users must `exit` from the
system to gain a balance of this token, which can then be used in Oasis
etc.

It is possible to have multiple fungible USDV tokens, allowing for the
adoption of new token standards. This needs careful consideration from a
UX perspective, with the notion of a canonical token address becoming
increasingly restrictive. In the future, cross-chain communication and
scalable sidechains will likely lead to a proliferation of multiple USDV
tokens. Users of the core could `exit` into a Plasma sidechain, an
Velas shard, or a different blockchain entirely via e.g. the Cosmos
Hub.


## Price Feeds

Price feeds are a crucial part of the USDV system. The code here assumes
that there are working price feeds and that their values are being
pushed to the contracts.

Specifically, the price that is required is the highest acceptable
quantity of CDP USDV debt per unit of collateral.


## Liquidation and Auctions

An important difference between SCD and MCD is the switch from fixed
price sell offs to auctions as the means of liquidating collateral.

The auctions implemented here are simple and expect liquidations to
occur in *fixed size lots* (say 10,000 VLX).


## Settlement

Another important difference between SCD and MCD is in the handling of
System Debt. System Debt is debt that has been taken from risky CDPs.
In SCD this is covered by diluting the collateral pool via the PETH
mechanism. In MCD this is covered by dilution of an external token,
namely VDGT.

As in collateral liquidation, this dilution occurs by an auction
(`flop`), using a fixed-size lot.

In order to reduce the collateral intensity of large CDP liquidations,
VDGT dilution is delayed by a configurable period (e.g 1 week).

Similarly, System Surplus is handled by an auction (`flap`), which sells
off USDV surplus in return for the highest bidder in VDGT.


## Authentication

The contracts here use a very simple multi-owner authentication system,
where a contract totally trusts multiple other contracts to call its
functions and configure it.

It is expected that modification of this state will be via an interface
that is used by the Governance layer.
