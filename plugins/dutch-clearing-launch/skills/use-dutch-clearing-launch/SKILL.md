---
name: use-dutch-clearing-launch
description: Build, test, integrate and deploy the DutchClearingLaunchHook Uniswap v4 hook from the dutch-clearing-launch repo. Use when you need to work with this hook's contract, wire it into a pool, or deploy it with a mined CREATE2 salt.
---

# DutchClearingLaunchHook

A descending-price launch enforced as a floor on what the pool will sell at, so a token opens at the price the first real buyer is willing to pay rather than the price the deployer guessed.

Family: Launch. Tags: launch, auction, price-discovery, anti-snipe, oracle-free. License: Apache-2.0. Source: `src/hooks/DutchClearingLaunchHook.sol`. The machine-readable manifest is `hook.json` (ABI, errors, permissions, properties).

## Build and test

Foundry project (`foundry.toml`, solc 0.8.26, evm `cancun`). From a clone of the repo:

```bash
git clone https://github.com/nirholas/dutch-clearing-launch && cd dutch-clearing-launch
git submodule update --init --recursive   # dependencies live in lib/
forge build
forge test
```

Tests are in `test/DutchClearingLaunchHook.t.sol`. Use `FOUNDRY_PROFILE=ci forge test` for the larger fuzz run (5000 runs).

## Hook permissions

Uniswap v4 reads a hook's permissions from the low bits of its address. This hook claims: `afterSwap`, `afterInitialize`. It is not upgradeable and has no custom swap data requirement (`hook.json` `properties`).

## Configuring a pool

The hook holds per-pool terms that are fixed for the pool's lifetime. Configure the key first, then initialize the pool:

```solidity
hook.configure(key, DutchClearingLaunchHook.Config({ /* startPriceX96 (uint160), floorPriceX96 (uint160), startTime (uint64), duration (uint32), buyIsZeroForOne (bool) */ }));
poolManager.initialize(key, startingSqrtPriceX96);
```

Signature: `configure((address,address,uint24,int24,address),(uint160,uint160,uint64,uint32,bool))`. A pool the hook was never configured for cannot be created (`PoolNotConfigured`). Anyone can configure a key that has no pool yet, so if someone front-runs your terms, pick a different `tickSpacing`, which is a different pool id.

## Deploying

A v4 hook must be deployed at an address whose bits match its permissions, so `script/Deploy.s.sol` mines a CREATE2 salt first. The dry run sends nothing:

```bash
export RPC_URL=https://...        # a chain with a Uniswap v4 PoolManager
forge script script/Deploy.s.sol --rpc-url $RPC_URL
```

Broadcasting spends gas from the signer (`PRIVATE_KEY`, or `--account` / `--ledger`) and is irreversible, so show the user the chain, the predicted hook address and the signer, and get an explicit yes before running:

```bash
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Re-running is safe: a hook already at its mined address is reported and skipped. If the miner fails with `EvmError: OutOfGas`, add `--gas-limit 30000000000`. `--verify` needs `ETHERSCAN_API_KEY`. The chain must be in the address book `src/libraries/Chains.sol`. `script/Deploy.s.sol` fails with "no Uniswap v4 PoolManager known for this chain" otherwise.

## Identifying a deployed hook

```solidity
IHookMetadata(hookAddress).hookName();
IHookMetadata(hookAddress).specURI();   // points at hook.json
```

## Errors you may meet

`BelowFloor(uint256,uint256)`, `FloorAboveStart()`, `InvalidDuration()`, `InvalidPool()`, `PoolAlreadyInitialized()`, `PoolNotConfigured()`, `StartInThePast()`.

## Limitation

It sets a floor on the execution price, so it cannot stop somebody buying at a price far above the floor: it prevents a cheap fill, not an expensive one. A launch with no buyers at any price simply runs the schedule out and becomes an ordinary pool at the floor, which is the correct outcome but not a rescue. And because the floor binds only on buys of the launched token, a pool whose currencies are the other way round must be configured with `buyIsZeroForOne` set accordingly or the constraint will guard the wrong side.

More: `docs/integrating.md`, `docs/deploying.md`, and None.
