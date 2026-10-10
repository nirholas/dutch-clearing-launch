# DutchClearingLaunch

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/dutch-clearing-launch?style=flat&logo=github)](https://github.com/nirholas/dutch-clearing-launch/stargazers) [![License](https://img.shields.io/github/license/nirholas/dutch-clearing-launch?style=flat)](https://github.com/nirholas/dutch-clearing-launch/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/dutch-clearing-launch?style=flat)](https://github.com/nirholas/dutch-clearing-launch/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/dutch-clearing-launch/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/dutch-clearing-launch/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**A descending-price launch enforced as a floor on what the pool will sell at, so a token opens at the price the first real buyer is willing to pay rather than the price the deployer guessed.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://dutch-clearing-launch.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/DutchClearingLaunchHook.sol`](src/hooks/DutchClearingLaunchHook.sol)
- **Licence:** Apache-2.0

## How it works

A launch has to answer a question nobody can answer in advance: what is this worth. Fixed-price launches answer it by guessing, and are wrong in one of two expensive directions. Guess low and the entire mispricing is captured in the first block by whoever pays the most gas, which is the mechanism people call a snipe.

Guess high and nothing trades until the price falls, except that in an AMM the price only falls when somebody sells, so the first honest buyer eats the whole descent. A Dutch auction answers it by asking. Start above any plausible value and come down until somebody accepts; the clearing price is then discovered rather than declared, and the first buyer pays roughly what the marginal buyer thinks it is worth instead of a number in a config file.

The published way to put a Dutch auction on an AMM is a liquidity bootstrapping pool, which ramps the pool's weights so the quoted price drifts down on its own. That works and it needs a weighted pool, a weight schedule and a curve that is not the one v4 has. This does it with a constraint instead of a curve.

The pool is an ordinary v4 pool. The hook computes a floor price that decays from `startPriceX96` to `floorPriceX96` over `duration`, and rejects any swap that buys the launched token below it. Nothing forces the price down; the schedule simply refuses to sell cheaply yet.

When a buyer accepts, the pool's own price moves above the floor and the constraint stops binding, which is the auction clearing. When the schedule expires it stops binding forever and the pool is a normal pool. Three properties fall out of doing it this way.

Sniping the first block is pointless, because the first block's floor is the start price and there is no discount to capture. Selling is never restricted, at any point in the schedule, so nobody can be trapped in a position by the launch mechanism. And the hook holds nothing, mints nothing and has no privileged role: it can refuse a swap and that is the whole of its power.

The decay is linear in time. A geometric schedule is the more common choice in Dutch auctions and would need either an exponential or a lookup table on-chain; linear is exact in integer arithmetic, and a launch that wants a steep early descent can express it by picking a shorter duration and a lower floor.

## Prior art

Liquidity bootstrapping pools (Balancer LBPs, and the Uni LBP and LBP Hook submissions for v4) run the descent through a weight schedule, which needs a weighted curve. Fair-launch and anti-snipe hooks address the same problem through fees or allowlists. Expressing the auction as a decaying floor price on an otherwise ordinary pool, so the curve is untouched and the constraint simply stops binding once demand arrives, is the contribution here.

## Where it does not help

It sets a floor on the execution price, so it cannot stop somebody buying at a price far above the floor: it prevents a cheap fill, not an expensive one. A launch with no buyers at any price simply runs the schedule out and becomes an ordinary pool at the floor, which is the correct outcome but not a rescue. And because the floor binds only on buys of the launched token, a pool whose currencies are the other way round must be configured with `buyIsZeroForOne` set accordingly or the constraint will guard the wrong side.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    DutchClearingLaunchHook.Config({
        startPriceX96: /* uint160 */ 0,
        floorPriceX96: /* uint160 */ 0,
        startTime: /* uint64 */ 0,
        duration: /* uint32 */ 0,
        buyIsZeroForOne: /* bool */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `startPriceX96` | `uint160` |  |
| `floorPriceX96` | `uint160` |  |
| `startTime` | `uint64` |  |
| `duration` | `uint32` |  |
| `buyIsZeroForOne` | `bool` |  |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `BelowFloor(uint256,uint256)` | The swap would buy below the price the schedule is currently asking. |
| `FloorAboveStart()` | The floor must not be above the start price; a schedule that ascends is not a Dutch auction. |
| `InvalidDuration()` | `duration` was zero, which would make the schedule a single instant. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `StartInThePast()` | The schedule must begin at or after configuration; a descent that already happened is not a schedule. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `afterSwap`

Mask: `0x1040`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # DutchClearingLaunch
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # launch, auction, price-discovery, anti-snipe, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/dutch-clearing-launch
cd dutch-clearing-launch
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

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/dutch-clearing-launch&type=Date)](https://www.star-history.com/#nirholas/dutch-clearing-launch&Date)

<!-- three.ws:growth -->
## Support the project

If dutch-clearing-launch saves you time, **[star it on GitHub](https://github.com/nirholas/dutch-clearing-launch)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=dutch-clearing-launch%3A%20A%20descending-price%20launch%20enforced%20as%20a%20floor%20on%20what%20the%20pool%20will%20sell%20at%2C%20so%20a%20token%20opens%20at&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fdutch-clearing-launch) · [Share on Bluesky](https://bsky.app/intent/compose?text=dutch-clearing-launch%3A%20A%20descending-price%20launch%20enforced%20as%20a%20floor%20on%20what%20the%20pool%20will%20sell%20at%2C%20so%20a%20token%20opens%20at%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fdutch-clearing-launch) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fdutch-clearing-launch) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fdutch-clearing-launch&t=dutch-clearing-launch%3A%20A%20descending-price%20launch%20enforced%20as%20a%20floor%20on%20what%20the%20pool%20will%20sell%20at%2C%20so%20a%20token%20opens%20at) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fdutch-clearing-launch&title=dutch-clearing-launch%3A%20A%20descending-price%20launch%20enforced%20as%20a%20floor%20on%20what%20the%20pool%20will%20sell%20at%2C%20so%20a%20token%20opens%20at)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/dutch-clearing-launch` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/dutch-clearing-launch/issues) or [start a discussion](https://github.com/nirholas/dutch-clearing-launch/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/dutch-clearing-launch)](https://github.com/nirholas/dutch-clearing-launch/graphs/contributors)

<!-- /three.ws:growth -->
