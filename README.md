# ReputationFee

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/reputation-fee?style=flat&logo=github)](https://github.com/nirholas/reputation-fee/stargazers) [![License](https://img.shields.io/github/license/nirholas/reputation-fee?style=flat)](https://github.com/nirholas/reputation-fee/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/reputation-fee?style=flat)](https://github.com/nirholas/reputation-fee/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/reputation-fee/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/reputation-fee/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**Charges a counterparty according to how it has behaved, using an on-chain reputation registry, with no signature to present and nothing for a router to forward.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://reputation-fee.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/ReputationFeeHook.sol`](src/hooks/ReputationFeeHook.sol)
- **Licence:** Apache-2.0

## How it works

Pools price everyone identically because they cannot tell anyone apart, and the cost of that falls on the people who are cheapest to trade with. A venue that could distinguish its flow would quote the retail order and the arbitrageur differently, which is what every venue that can identify its counterparties actually does. The obstacle in v4 is that a hook does not see the trader.

`beforeSwap` receives the router, not the person who called it, so any scheme keyed on the trader's address needs the trader to sign something and the router to forward it, which means most routers cannot trade the pool at all. This takes the other route. register(myRouter); Thereafter the hook resolves `sender` to that agent with one storage read, asks the registry for its score, and prices the swap between `maxFee` at a score of zero and `minFee` at a perfect score.

No signature, no `hookData`, nothing for a router to support. Flow through an unregistered router is simply unknown and pays `maxFee`, which is the correct default: an unidentified counterparty is priced as the worst one. Spoofing is not possible because the mapping runs from router to agent and only the agent can write its own entry.

Registering a router somebody else also uses means paying for their flow with your reputation, which is a mistake you can only make about yourself. The registry is fixed at deployment and the fee bounds are fixed before the pool exists. There is no admin and no way to re-point the pool at a friendlier scorer once liquidity has arrived.

## Prior art

Identity-gated hooks are common (KYC, Civic, VioletID, World ID, PureFi) and they gate: pass or be refused. Loyalty and fidelity hooks discount by volume or tenure, which is a proxy for behaviour rather than a judgement of it. Pricing continuously off an external reputation score, and resolving the trader through a registered router so no signature or hookData is needed, is the contribution here.

## Where it does not help

The pool trusts the registry absolutely: a registry that can be bought is a fee schedule that can be bought, and nothing here detects that. Registration is also per router, so an agent that trades through a router it has not registered pays the unknown rate until it registers, and an agent using a shared public router either cannot register it or is subsidising everyone else who uses it.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    ReputationFeeHook.Config({
        minFee: /* uint24 */ 0,
        maxFee: /* uint24 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `minFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `maxFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidRegistry()` | A registry that reports a zero scale cannot be normalised against. |
| `MaxBelowMin()` | `maxFee` must be at least `minFee`; a schedule that rewards a bad score is a configuration error. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # ReputationFee
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # agent, reputation, dynamic-fee, erc8004, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/reputation-fee
cd reputation-fee
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/reputation-fee&type=Date)](https://www.star-history.com/#nirholas/reputation-fee&Date)

<!-- three.ws:growth -->
## Support the project

If reputation-fee saves you time, **[star it on GitHub](https://github.com/nirholas/reputation-fee)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=reputation-fee%3A%20Charges%20a%20counterparty%20according%20to%20how%20it%20has%20behaved%2C%20using%20an%20on-chain%20reputation%20registry%2C%20with&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Freputation-fee) · [Share on Bluesky](https://bsky.app/intent/compose?text=reputation-fee%3A%20Charges%20a%20counterparty%20according%20to%20how%20it%20has%20behaved%2C%20using%20an%20on-chain%20reputation%20registry%2C%20with%20https%3A%2F%2Fgithub.com%2Fnirholas%2Freputation-fee) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Freputation-fee) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Freputation-fee&t=reputation-fee%3A%20Charges%20a%20counterparty%20according%20to%20how%20it%20has%20behaved%2C%20using%20an%20on-chain%20reputation%20registry%2C%20with) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Freputation-fee&title=reputation-fee%3A%20Charges%20a%20counterparty%20according%20to%20how%20it%20has%20behaved%2C%20using%20an%20on-chain%20reputation%20registry%2C%20with)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/reputation-fee` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/reputation-fee/issues) or [start a discussion](https://github.com/nirholas/reputation-fee/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/reputation-fee)](https://github.com/nirholas/reputation-fee/graphs/contributors)

<!-- /three.ws:growth -->
