# ERC-8211 Reference Implementation

Reference smart contracts for **[ERC-8211: Smart Batching](https://ethereum-magicians.org/t/erc-8211-smart-batching/28135)** — a batch encoding where each parameter declares how to obtain its value at execution time, and what conditions that value must satisfy.

## Why

Today, every parameter in a batched Ethereum transaction (via ERC-4337 or EIP-5792) is frozen at signing time. If anything changes between signing and execution — a swap returns fewer tokens than expected, a balance shifts, a bridge delivers late — the batch reverts. The only existing workaround is deploying a custom smart contract for every flow.

ERC-8211 fixes this with three small primitives:

- **Fetchers** decide how each parameter is obtained: `RAW_BYTES` (literal at signing time), `STATIC_CALL` (resolve via `staticcall` to any contract at execution time), or `BALANCE` (ERC-20 or native balance query).
- **Param types** decide where the resolved value goes: `TARGET` (call destination), `VALUE` (ETH forwarded), or `CALL_DATA` (appended to the calldata being built).
- **Constraints** decide whether the resolved value is acceptable: `EQ`, `GTE`, `LTE`, `IN`. A failed constraint reverts the entire batch.

A `ComposableExecution` entry with no `TARGET` parameter naturally becomes a **predicate entry** — no call is made, but parameters are resolved and constraints validated, producing a pure boolean gate on chain state inside an otherwise normal batch.

## Contracts

| Contract | Role |
|---|---|
| [`ComposableExecutionLib`](contracts/ComposableExecutionLib.sol) | Execution engine. Resolves inputs via fetchers, validates constraints, routes resolved values to `TARGET`/`VALUE`/`CALL_DATA`, executes the call, and captures outputs. |
| [`ComposableExecutionBase`](contracts/ComposableExecutionBase.sol) | Abstract base contract. Smart accounts inherit this to support composable execution natively. |
| [`ComposableExecutionModule`](contracts/ComposableExecutionModule.sol) | ERC-7579 module (executor + fallback) that adds composable execution to any ERC-7579 account without modifying the account implementation. |
| [`Storage`](contracts/Storage.sol) | Namespaced key/value store used to capture return values from one entry and feed them into a later entry within the same batch. |
| [`interfaces/IComposableExecution.sol`](contracts/interfaces/IComposableExecution.sol) | The `IComposableExecution` interface as defined by ERC-8211. |
| [`types/ComposabilityDataTypes.sol`](contracts/types/ComposabilityDataTypes.sol) | All on-chain types: `ComposableExecution`, `InputParam`, `OutputParam`, `Constraint`, and the `*Type` enums. |

The module is account-standard agnostic — it works with ERC-7579, ERC-6900, native smart accounts, and ERC-7702-delegated EOAs.

## Storage slot caveat for `ComposableExecutionModule`

The actual storage slot used in `Storage.sol` depends on both the `account` address **and** the `caller` address. If `ComposableExecutionModule` is invoked via a `call` flow (as a Fallback or Executor module), it ends up at a different slot than when invoked via `delegatecall`. Pick one flow per smart account and stay consistent.

## Audits

Both audits cover the codebase on `main`:

- [`audits/2025-03-Composability-Pashov-Review.pdf`](audits/2025-03-Composability-Pashov-Review.pdf) — Pashov Audit Group review (March 2025).
- [`audits/2025-03-Composability_Zenith-Audit-Report.pdf`](audits/2025-03-Composability_Zenith-Audit-Report.pdf) — Zenith audit report (March 2025).

The [`feat/signed-constraints-and-or-composition`](https://github.com/bcnmy/erc8211-contracts/pull/1) branch adds signed-integer constraints (`GTE_SIGNED`, `LTE_SIGNED`) and OR predicate composition. It is **not yet audited** — the open PR is intentionally left open for community review.

## Spec & links

- ERC-8211 discussion: <https://ethereum-magicians.org/t/erc-8211-smart-batching/28135>
- Project site: <https://erc8211.com>
- Demo: <https://demo.erc8211.com>

## Build & test

```shell
pnpm i
forge build
forge test
```
