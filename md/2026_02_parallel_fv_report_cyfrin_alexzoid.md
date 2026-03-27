# Formal Verification Report: Parallel Protocol

- Repository: https://github.com/Cyfrin/audit-2026-02-parallel_3.1
- Latest Commit Hash: [2781b67](https://github.com/Cyfrin/audit-2026-02-parallel_3.1/commit/2781b6799d7e7cb9946cc681f6ae173cf9fbcd50)
- Date: February 2026
- Author: [@alexzoid](https://x.com/alexzoid) ([@cyfrin](https://x.com/cyfrin) private formal verification engagement)
- Certora Prover version: 8.8.0

---

## Table of Contents

1. [About Parallel Protocol](#about-parallel-protocol)
2. [Formal Verification Methodology](#formal-verification-methodology)
   - [Types of Properties](#types-of-properties)
   - [Verification Process](#verification-process)
   - [Project Structure](#project-structure)
   - [Assumptions](#assumptions)
3. [Verification Properties](#verification-properties)
   - [Valid State](#valid-state)
   - [Variable Transitions](#variable-transitions)
   - [State Transitions](#state-transitions)
   - [High-Level](#high-level)
   - [Unit Tests](#unit-tests)
   - [ERC-4626 Conformance](#erc-4626-conformance)
4. [Real Issues Found](#real-issues-found)
5. [Mutation Testing](#mutation-testing)
   - [What is Mutation Testing](#what-is-mutation-testing)
   - [VS-P-15: normalizedStablesSumConservation](#vs-p-15-normalizedstablessumconservation)
6. [Setup and Execution](#setup-and-execution)
   - [Common Setup (Steps 1–4)](#common-setup-steps-14)
   - [Remote Execution](#remote-execution)
   - [Local Execution](#local-execution)
   - [Required source code modifications](#required-source-code-modifications)
   - [Running Verification](#running-verification)

---

## About Parallel Protocol

Parallel is a decentralized stablecoin protocol (fork of Angle Protocol's Transmuter) that lets users mint over-collateralized stablecoins (TokenP) by depositing correlated collateral assets. The core engine ("Parallelizer") uses an EIP-2535 Diamond proxy supporting 1:1 minting/burning with exposure-based piecewise-linear fees, proportional basket redemption, and surplus distribution.

Three formal verification targets were in scope:

1. **Parallelizer** — The Diamond-proxy stablecoin engine with four facets verified independently:
   - **Swapper** — Mint TokenP by depositing collateral; burn TokenP to withdraw collateral
   - **Redeemer** — Proportional basket redemption and normalizer updates
   - **Surplus** — Convert collateral surplus to TokenP and distribute to payees
   - **Setter** — Governance functions: add/revoke collateral, pause, set fees, manage trusted roles

2. **Savings** — ERC-4626 yield vault where users deposit TokenP to earn interest accrued via governance-set rates

3. **FlashParallelToken** — ERC-3156 flash lender that temporarily mints TokenP, collects basis-point fees, and burns the principal

Key operations:
- `swapExactInput` / `swapExactOutput` — mint TokenP by depositing collateral or burn TokenP to withdraw collateral, with exposure-based piecewise-linear fees
- `redeem` / `redeemWithForfeit` — proportional basket redemption of TokenP for underlying collateral assets
- `updateNormalizer` — rebase normalizedStables accounting to reflect external yield or loss events
- `processSurplus` / `release` — convert excess collateral to TokenP and distribute to registered payees
- `deposit` / `mint` / `withdraw` / `redeem` (Savings) — ERC-4626 yield vault where TokenP holders earn governance-set interest rates
- `flashLoan` — ERC-3156 flash mint/burn of TokenP with configurable basis-point fees

<div style="page-break-before: always;"></div>

---

## Formal Verification Methodology

Certora Formal Verification (FV) provides mathematical proofs of smart contract correctness by verifying code against a formal specification. Unlike testing and fuzzing which examine specific execution paths, Certora FV examines all possible states and execution paths.

The process involves crafting properties in CVL (Certora Verification Language) and submitting them alongside compiled Solidity smart contracts to a remote prover. The prover transforms the contract bytecode and rules into a mathematical model and determines the validity of rules.

### Types of Properties

Properties are categorized following the [official Certora methodology](https://github.com/Certora/Tutorials/blob/master/06.Lesson_ThinkingProperties/Categorizing_Properties.pdf). Valid State, Variable Transitions and State Transitions properties are **parametric** — they are automatically verified against every external function in the contract, including functions added after the specification is completed. High-Level properties target specific function sequences.

**Valid State** — System-wide invariants that MUST always hold true. These properties define the fundamental constraints of the protocol, such as accounting consistency and structural integrity. Once proven, these invariants serve as trusted assumptions in other properties via `requireInvariant`, reducing verification complexity.

**Variable Transitions** — Properties that verify specific storage variables change only under expected conditions. The process captures a variable value before instruction execution, runs the instruction, then asserts the variable changed only as permitted or remained unchanged.

**State Transitions** — Properties that verify the correctness of transitions between valid states. Building upon the valid state invariants, these properties ensure the protocol's state machine operates correctly and that state changes are both authorized and sequentially valid.

**High-Level** — Complex multi-step properties verifying business logic integrity. Unlike parametric properties, these target specific function sequences to validate end-to-end protocol behavior.

**Unit Tests** — Properties that verify basic behavior of individual functions: revert conditions, direct effects on state, and non-effects on unrelated state. Unlike parametric properties, each unit test targets a specific function call.

**ERC-4626 Conformance** — Properties verifying the Savings vault's compliance with the ERC-4626 tokenized vault standard, including asset/totalAssets correctness, conversion functions, preview accuracy, deposit/mint/withdraw/redeem semantics, and allowance management.

### Verification Process

1. **Setup phase**: Define ghost variables, storage hooks, and helper definitions to model contract state in CVL. Establish verification harnesses and configurations for three independent targets (Parallelizer with four facets, Savings, FlashParallelToken). This phase addresses several prover limitations:
   - Diamond proxy storage ([`parallelizer_storage.spec`](./specs/setup/parallelizer_storage.spec)): The Parallelizer uses EIP-2535 Diamond storage with a complex `TransmuterStorage` struct containing nested arrays and mappings. Ghost variables and storage hooks mirror all relevant slots — collateral list, fee curves, normalizedStables, payees, and shares — enabling CVL rules to reason about the full protocol state.
   - External dependency summaries ([`parallelizer.spec`](./specs/setup/parallelizer.spec), [`collateral_manager.spec`](./specs/setup/collateral_manager.spec)): LibOracle, LibManager, LibWhitelist, and LibDiamond contain complex external interactions (price feeds, yield strategies, access control) that cause solver timeouts. These are replaced with ghost-backed CVL functions or `NONDET` summaries that preserve essential behavioral properties while eliminating implementation complexity.
   - ERC-20 model ([`erc20.spec`](./specs/setup/erc20/erc20.spec), [`token_p.spec`](./specs/setup/erc20/token_p.spec), [`safe_erc20.spec`](./specs/setup/erc20/safe_erc20.spec)): Real ERC-20 contracts cause timeouts due to implementation complexity and unbounded address space. A lightweight ghost-variable-based model summarizes all external ERC-20 calls (`balanceOf`, `transfer`, `transferFrom`, `approve`, `mint`, `burn`) over a bounded account set, enabling tractable solvency invariants.
   - Math utilities ([`math.spec`](./specs/setup/math.spec)): PercentageMathLib's `percentMul` uses intermediate multiplication that overflows in the solver. It is replaced with the exact CVL expression `(x * pct + 5000) / 10000`.
   - OpenZeppelin infrastructure ([`parallelizer.spec`](./specs/setup/parallelizer.spec)): Initializable, UUPSUpgradeable, and ReentrancyGuardTransient functions are summarized with `NONDET DELETE` or constrained ghost variables to exclude infrastructure code from verification while preserving reentrancy guard semantics.
2. **Crafting Properties**: Write invariants and rules in CVL, starting with valid state invariants (which become trusted preconditions for other rules), then variable/state transitions, and finally high-level and unit test properties.

### Project Structure

```
certora/
├── confs/                                  # Prover configuration files
│   ├── FlashParallelToken/                 # Flash loan token specs
│   │   ├── high_level.conf
│   │   ├── state_transitions.conf
│   │   ├── unit_tests.conf
│   │   ├── valid_state.conf
│   │   └── variable_transitions.conf
│   ├── Parallelizer/                       # Diamond proxy specs
│   │   ├── Redeemer/                       # Redeemer facet
│   │   │   ├── high_level.conf
│   │   │   ├── state_transitions.conf
│   │   │   ├── unit_tests.conf
│   │   │   ├── valid_state.conf
│   │   │   └── variable_transitions.conf
│   │   ├── Setter/                         # Setter facet (Governor + Guardian)
│   │   │   ├── high_level.conf
│   │   │   ├── state_transitions.conf
│   │   │   ├── unit_tests.conf
│   │   │   ├── valid_state.conf
│   │   │   └── variable_transitions.conf
│   │   ├── Surplus/                        # Surplus facet
│   │   │   ├── high_level.conf
│   │   │   ├── state_transitions.conf
│   │   │   ├── unit_tests.conf
│   │   │   ├── valid_state.conf
│   │   │   └── variable_transitions.conf
│   │   └── Swapper/                        # Swapper facet
│   │       ├── high_level.conf
│   │       ├── state_transitions.conf
│   │       ├── unit_tests.conf
│   │       ├── valid_state.conf
│   │       └── variable_transitions.conf
│   └── Savings/                            # ERC4626 vault specs
│       ├── erc4626.conf
│       ├── high_level.conf
│       ├── state_transitions.conf
│       ├── unit_tests.conf
│       ├── valid_state.conf
│       └── variable_transitions.conf
│
├── harnesses/                              # Verification harnesses
│   ├── FlashParallelTokenHarness.sol       # Flash loan interest accrual wrapper
│   ├── HelperCVL.sol                       # Assertion wrapper for CVL helpers
│   ├── ParallelizerLayout.sol              # Diamond storage layout mapping
│   ├── RedeemerHarness.sol                 # Redeemer facet + layout harness
│   ├── SavingsHarness.sol                  # Savings + layout getters harness
│   ├── SavingsLayout.sol                   # ERC4626/ERC20 OZ storage mapping
│   ├── SetterHarness.sol                   # Governor + Guardian setter harness
│   ├── SurplusHarness.sol                  # Surplus + Swapper facet harness
│   └── SwapperHarness.sol                  # Swapper facet harness
│
└── specs/                                  # CVL specifications
    ├── flashParallelToken/                 # FlashParallelToken properties
    │   ├── flashParallelToken_high_level.spec
    │   ├── flashParallelToken_state_transitions.spec
    │   ├── flashParallelToken_unit_tests.spec
    │   ├── flashParallelToken_valid_state.spec
    │   └── flashParallelToken_variable_transitions.spec
    ├── parallelizer/                       # Parallelizer properties (shared + per-facet)
    │   ├── parallelizer_state_transitions.spec   # Cross-facet state transitions
    │   ├── parallelizer_valid_state.spec         # Cross-facet valid state invariants
    │   ├── parallelizer_variable_transitions.spec # Cross-facet variable transitions
    │   ├── redeemer/                       # Redeemer facet properties
    │   │   ├── redeemer_high_level.spec
    │   │   ├── redeemer_state_transitions.spec
    │   │   ├── redeemer_unit_tests.spec
    │   │   ├── redeemer_valid_state.spec
    │   │   └── redeemer_variable_transitions.spec
    │   ├── setter/                         # Setter facet properties
    │   │   ├── setter_high_level.spec
    │   │   ├── setter_state_transitions.spec
    │   │   ├── setter_unit_tests.spec
    │   │   ├── setter_valid_state.spec
    │   │   └── setter_variable_transitions.spec
    │   ├── surplus/                        # Surplus facet properties
    │   │   ├── surplus_high_level.spec
    │   │   ├── surplus_state_transitions.spec
    │   │   ├── surplus_unit_tests.spec
    │   │   ├── surplus_valid_state.spec
    │   │   └── surplus_variable_transitions.spec
    │   └── swapper/                        # Swapper facet properties
    │       ├── swapper_high_level.spec
    │       ├── swapper_state_transitions.spec
    │       ├── swapper_unit_tests.spec
    │       ├── swapper_valid_state.spec
    │       └── swapper_variable_transitions.spec
    ├── savings/                            # Savings properties
    │   ├── savings_erc4626.spec            # ERC4626 compliance rules
    │   ├── savings_high_level.spec
    │   ├── savings_state_transitions.spec
    │   ├── savings_unit_tests.spec
    │   ├── savings_valid_state.spec
    │   └── savings_variable_transitions.spec
    └── setup/                              # Shared setup: ghosts, hooks, summaries
        ├── boundaries.spec                 # Loop/ERC20 user count bounds
        ├── collateral_manager.spec         # Per-collateral manager ghost summaries
        ├── env.spec                        # Environment constraints (timestamp, sender, value)
        ├── erc20/                          # ERC20 CVL models
        │   ├── erc20.spec                  # Ghost-based ERC20 token model
        │   ├── safe_erc20.spec             # SafeERC20 wrapper summaries
        │   └── token_p.spec                # TokenP mint/burn CVL summaries
        ├── flashParallelToken.spec         # FlashParallelToken setup and ghosts
        ├── helper.spec                     # CVL helper (assertOnFailure)
        ├── math.spec                       # Math library summaries (mulDiv, sqrt)
        ├── parallelizer.spec               # Diamond setup, oracle/manager/whitelist summaries
        ├── parallelizer_boundaries.spec    # Parallelizer verification bounds
        ├── parallelizer_redeemer.spec      # Redeemer facet common spec imports
        ├── parallelizer_setter.spec        # Setter facet common spec imports
        ├── parallelizer_storage.spec       # Storage layout ghosts and Sload/Sstore hooks
        ├── parallelizer_surplus.spec       # Surplus facet setup
        ├── parallelizer_swapper.spec       # Swapper facet setup
        └── savings.spec                    # Savings (ERC4626) setup and ghosts
```

### Assumptions

Formal verification requires assumptions about the code and its environment to address prover timeouts, tool limitations, and state consistency. However, incorrect assumptions can mask real bugs by excluding reachable states from analysis. To maintain transparency, all assumptions are categorized into four groups: **Safe** (real-world constraints that don't reduce security coverage), **Proved** (formally verified invariants reused as preconditions), **Trusted** (initialization state and admin-configured parameters assumed correct because initialization logic is excluded from verification), and **Unsafe** (scope reductions necessary for tractability that may exclude valid scenarios).

#### Safe Assumptions

These reflect real-world constraints that don't impact security guarantees. In the codebase, every `require` statement that constitutes a safe assumption is annotated with a `"SAFE: ..."` message string.

Environment Constraints ([`env.spec`](./specs/setup/env.spec)):
- The contract does not handle native ETH, so transactions carry no value (`msg.value == 0`)
- The sender is never the zero address and never the contract itself
- Block timestamps fall within realistic bounds (`[65535, max_uint32)`)

Address Separation ([`env.spec`](./specs/setup/env.spec), [`parallelizer.spec`](./specs/setup/parallelizer.spec)):
- TokenP is not the Parallelizer and not a collateral address
- For managed collateral, the diamond balance is zero (all funds held by external strategy)

Math Summaries ([`math.spec`](./specs/setup/math.spec)):
- PercentageMathLib's `percentMul` replaced with exact CVL arithmetic: `(x * pct + 5000) / 10000`

ERC-20 Model ([`erc20.spec`](./specs/setup/erc20/erc20.spec)):
- Total supply equals the sum of all individual balances (standard ERC-20 solvency invariant)

#### Proved Assumptions

These properties have been formally verified as valid state invariants and are used as trusted preconditions (via `requireInvariant`) in state transition and high-level rules. See [Valid State](#valid-state) for detailed descriptions and verification statuses.

Parallelizer Structural Invariants ([`parallelizer_valid_state.spec`](./specs/parallelizer/parallelizer_valid_state.spec)):
- Collateral list contains no zero addresses and no duplicates, and list membership is linked to non-zero decimals (VS-P-01, VS-P-02, VS-P-03)
- Non-listed addresses have zero collateral data; stored decimals match ERC-20 decimals for active collaterals (VS-P-24, VS-P-25)
- Boolean flags (isRedemptionLive, isTrusted, isSellerTrusted, isManaged, isMintLive, isBurnLive, onlyWhitelisted) are always 0 or 1 (VS-P-08 through VS-P-14)
- Normalizer stays within [BASE_18, BASE_36]; slippage tolerance ≤ BASE_9; surplus buffer ratio is 0 or ≥ BASE_9; lastReleasedAt ≤ block.timestamp (VS-P-04 through VS-P-07)

Parallelizer Accounting Invariants ([`parallelizer_valid_state.spec`](./specs/parallelizer/parallelizer_valid_state.spec)):
- Global normalizedStables equals the sum of per-collateral normalizedStables (VS-P-15)
- Total shares equals the sum of per-payee shares; payees are distinct with positive shares; non-payee addresses have zero shares (VS-P-21, VS-P-22, VS-P-23, VS-P-26)

Parallelizer Fee Curve Invariants ([`parallelizer_valid_state.spec`](./specs/parallelizer/parallelizer_valid_state.spec)):
- Mint/burn/redemption fee curve X and Y arrays have matching lengths (VS-P-16, VS-P-17, VS-P-18)
- Mint fee X-array is strictly increasing with X[0] == 0; burn fee X-array is strictly decreasing with X[0] == BASE_9 (VS-P-19, VS-P-20, VS-P-30, VS-P-31)
- Y-values are bounded: mint ≤ BASE_12, burn ≤ BASE_9, redemption ∈ [0, BASE_9]; redemption X bounded by BASE_9 (VS-P-27, VS-P-28, VS-P-29, VS-P-32)

Savings Invariants ([`savings_valid_state.spec`](./specs/savings/savings_valid_state.spec)):
- Paused flag and trusted updater flag are always 0 or 1 (VS-SA-01, VS-SA-02)
- lastUpdate never exceeds block.timestamp (VS-SA-03)
- The vault never approves its own shares (VS-SA-04)

FlashParallelToken Invariants ([`flashParallelToken_valid_state.spec`](./specs/flashParallelToken/flashParallelToken_valid_state.spec)):
- Fee rate never exceeds 100% (VS-FP-01)

> Note: VS-P-14 (`collateralOnlyWhitelistedFlag`) has expected violations in `setWhitelistStatus()` due to a [real bug](#3-low-settersgovernorsetwhiteliststatus-allows-values-other-than-0-and-1). It is not used as a `requireInvariant` in other rules.

#### Unsafe Assumptions

These reduce verification scope to make the problem tractable for the prover. In the codebase, every `require` statement that constitutes an unsafe assumption is annotated with an `"UNSAFE: ..."` message string.

External Dependency Summaries ([`parallelizer.spec`](./specs/setup/parallelizer.spec), [`collateral_manager.spec`](./specs/setup/collateral_manager.spec)):
- LibOracle: Replaced with ghost-backed CVL functions. Real oracle calls are not modeled, so price manipulation scenarios are excluded.
- LibManager: Ghost-backed CVL functions model invest/release/totalAssets with per-collateral ghost state. Real strategy interactions are replaced, so strategy failure scenarios are not modeled.
- LibWhitelist: `checkWhitelist` summarized as `NONDET` — may return true or false. Whitelist bypass scenarios are not modeled.
- LibDiamond: `checkCanCall` summarized as `NONDET`; `isGovernor` modeled via persistent ghost mapping. Real access control enforcement is not verified.
- AccessManaged (OpenZeppelin): `canCall` and `consumeScheduledOp` summarized as `NONDET`. Access manager restrictions are not verified.
- Initializable/UUPSUpgradeable: Functions `NONDET DELETE`-d. Initialization and upgrade logic excluded from verification.

Bounded Model ([`boundaries.spec`](./specs/setup/boundaries.spec), [`parallelizer_boundaries.spec`](./specs/setup/parallelizer_boundaries.spec)):
Bounding the number of collaterals, payees, and accounts serves two purposes: it prevents prover timeouts on unbounded quantifiers, and it makes it possible to reason about total supply as a concrete sum of all individual values — enabling protocol-wide conservation invariants such as "global normalizedStables equals the sum of per-collateral normalizedStables."

- Loop iterations capped at 2 across all configurations
- Collateral list length bounded at 2 for universal quantifier tractability
- Payees list length bounded at 2 for universal quantifier tractability
- ERC-20 accounts tracked for at most 5 addresses per token

#### Trusted Assumptions

These assume that initialization has been completed and that admin-configured parameters fall within reasonable operational bounds. Since `initialize()` and upgrade logic are excluded from verification (summarized as `NONDET DELETE`), the post-initialization state is taken as a precondition. In the codebase, every `require` statement that constitutes a trusted assumption is annotated with a `"TRUSTED: ..."` message string.

Initialization State ([`parallelizer.spec`](./specs/setup/parallelizer.spec)):
- `accessManager != 0` — set by `initialize()`; all access control checks depend on this being non-zero
- `statusReentrant ∈ {1, 2}` — set by the first `nonReentrant` call; the reentrancy guard uses transient storage (EIP-1153) which the prover does not natively model, so the guard state is constrained to its two valid values via ghost variables

Oracle Value Bounds ([`parallelizer.spec`](./specs/setup/parallelizer.spec)):
- All oracle ghost functions (`readMint`, `readBurn`, `getBurnOracle`, `readRedemption`) return values in `(0, max_uint128]`
- Burn ratio and minRatio bounded to `(0, BASE_18]`
- These bounds assume the admin has configured a functioning oracle that returns non-zero, non-overflowing prices

Savings Rate Bounds ([`savings.spec`](./specs/setup/savings.spec)):
- Savings rate bounded to `[10^12, 10^21]` — prevents solver overflow in `_computeUpdatedAssets` rate exponentiation; the admin is trusted to set rates within this range (fuzz tests use `[10^14, 10^20]`)

<div style="page-break-before: always;"></div>

---

## Verification Properties

Links to specific CVL spec files are provided for each property, with status indicators.

- ✅ Verified
- ❌ Violated

### Valid State

Valid State properties define the fundamental invariants that must always hold true throughout the protocol's lifecycle. These are parametric — each property is automatically verified against every external function in the contract, including functions added in the future.

#### Parallelizer

The 32 Parallelizer invariants are verified independently across all four facets (Swapper, Redeemer, Setter, Surplus).

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VS-P-01](./specs/parallelizer/parallelizer_valid_state.spec#L58) | `collateralListNoZeroValues` | Collateral list entries are non-zero addresses<br>`forall i: i < listLen => collateralList[i] != 0` | ✅ | |
| [VS-P-02](./specs/parallelizer/parallelizer_valid_state.spec#L66) | `collateralListNoDuplicates` | Collateral list entries are distinct<br>`forall i, j: i < listLen AND j < listLen AND i != j => collateralList[i] != collateralList[j]` | ✅ | |
| [VS-P-03](./specs/parallelizer/parallelizer_valid_state.spec#L74) | `collateralListDecimalsNonZero` | Collateral list linked to mapping (decimals non-zero)<br>`forall i: i < listLen <=> decimals[collateralList[i]] != 0` | ✅ | |
| [VS-P-04](./specs/parallelizer/parallelizer_valid_state.spec#L81) | `normalizerBounded` | Normalizer stays within [BASE_18, BASE_36]<br>`BASE_18 <= norm <= BASE_36` | ✅ | |
| [VS-P-05](./specs/parallelizer/parallelizer_valid_state.spec#L87) | `slippageToleranceBounded` | Slippage tolerance per collateral ≤ BASE_9<br>`forall a: slippageTolerance[a] <= BASE_9` | ✅ | |
| [VS-P-06](./specs/parallelizer/parallelizer_valid_state.spec#L94) | `surplusBufferRatioBounded` | Surplus buffer ratio is 0 or ≥ BASE_9<br>`surplusBufferRatio == 0 OR surplusBufferRatio >= BASE_9` | ✅ | |
| [VS-P-07](./specs/parallelizer/parallelizer_valid_state.spec#L100) | `lastReleasedAtBounded` | lastReleasedAt ≤ block.timestamp<br>`lastReleasedAt <= block.timestamp` | ✅ | |
| [VS-P-08](./specs/parallelizer/parallelizer_valid_state.spec#L106) | `isRedemptionLiveFlag` | isRedemptionLive is boolean {0, 1}<br>`isRedemptionLive in {0, 1}` | ✅ | |
| [VS-P-09](./specs/parallelizer/parallelizer_valid_state.spec#L112) | `isTrustedFlag` | isTrusted mapping values are boolean {0, 1}<br>`forall a: isTrusted[a] in {0, 1}` | ✅ | |
| [VS-P-10](./specs/parallelizer/parallelizer_valid_state.spec#L118) | `isSellerTrustedFlag` | isSellerTrusted mapping values are boolean {0, 1}<br>`forall a: isSellerTrusted[a] in {0, 1}` | ✅ | |
| [VS-P-11](./specs/parallelizer/parallelizer_valid_state.spec#L125) | `collateralIsManagedFlag` | isManaged is boolean {0, 1}<br>`forall t: isManaged[t] in {0, 1}` | ✅ | |
| [VS-P-12](./specs/parallelizer/parallelizer_valid_state.spec#L132) | `collateralIsMintLiveFlag` | isMintLive is boolean {0, 1}<br>`forall t: isMintLive[t] in {0, 1}` | ✅ | |
| [VS-P-13](./specs/parallelizer/parallelizer_valid_state.spec#L139) | `collateralIsBurnLiveFlag` | isBurnLive is boolean {0, 1}<br>`forall t: isBurnLive[t] in {0, 1}` | ✅ | |
| [VS-P-14](./specs/parallelizer/parallelizer_valid_state.spec#L146) | `collateralOnlyWhitelistedFlag` | onlyWhitelisted is boolean {0, 1}<br>`forall t: onlyWhitelisted[t] in {0, 1}` | ❌ | Violated in `setWhitelistStatus()` — see [Real Issue #3](#real-issue-3) |
| [VS-P-15](./specs/parallelizer/parallelizer_valid_state.spec#L153) | `normalizedStablesSumConservation` | Global normalizedStables == sum of per-collateral normalizedStables<br>`NS == sum(collatNS[t] for t in collateralList)` | ✅ | |
| [VS-P-16](./specs/parallelizer/parallelizer_valid_state.spec#L161) | `feeMintLengthMatching` | Mint fee curve X and Y arrays have same length<br>`forall t: len(xFeeMint[t]) == len(yFeeMint[t])` | ✅ | |
| [VS-P-17](./specs/parallelizer/parallelizer_valid_state.spec#L168) | `feeBurnLengthMatching` | Burn fee curve X and Y arrays have same length<br>`forall t: len(xFeeBurn[t]) == len(yFeeBurn[t])` | ✅ | |
| [VS-P-18](./specs/parallelizer/parallelizer_valid_state.spec#L175) | `redemptionCurveLengthMatching` | Redemption curve X and Y arrays have same length<br>`len(xRedemptionCurve) == len(yRedemptionCurve)` | ✅ | |
| [VS-P-19](./specs/parallelizer/parallelizer_valid_state.spec#L181) | `feeMintXOrdering` | Mint fee curve X-array strictly increasing<br>`forall t: len(xFeeMint[t]) == 2 => xFeeMint[t][0] < xFeeMint[t][1]` | ✅ | |
| [VS-P-20](./specs/parallelizer/parallelizer_valid_state.spec#L189) | `feeBurnXOrdering` | Burn fee curve X-array strictly decreasing<br>`forall t: len(xFeeBurn[t]) == 2 => xFeeBurn[t][0] > xFeeBurn[t][1]` | ✅ | |
| [VS-P-21](./specs/parallelizer/parallelizer_valid_state.spec#L197) | `totalSharesSumConservation` | totalShares == sum of shares over payees<br>`totalShares == sum(shares[payees[i]] for i < payeesLen)` | ✅ | |
| [VS-P-22](./specs/parallelizer/parallelizer_valid_state.spec#L205) | `payeesNoDuplicates` | Payees array has no duplicates<br>`forall i, j: i < payeesLen AND j < payeesLen AND i != j => payees[i] != payees[j]` | ✅ | |
| [VS-P-23](./specs/parallelizer/parallelizer_valid_state.spec#L213) | `payeeSharesPositive` | Every active payee has positive shares<br>`forall i: i < payeesLen => shares[payees[i]] > 0` | ✅ | |
| [VS-P-24](./specs/parallelizer/parallelizer_valid_state.spec#L220) | `collateralDecimalsMatchErc20` | Stored decimals matches ERC20 decimals for active collaterals<br>`forall i: i < listLen => decimals[collateralList[i]] == erc20Decimals[collateralList[i]]` | ✅ | |
| [VS-P-25](./specs/parallelizer/parallelizer_valid_state.spec#L227) | `collateralDataZeroOutsideList` | Non-listed addresses have zero collateral data<br>`forall a: a not in collateralList <=> allCollatFields[a] == 0` | ✅ | |
| [VS-P-26](./specs/parallelizer/parallelizer_valid_state.spec#L246) | `sharesZeroOutsidePayees` | Non-payee addresses have zero shares<br>`forall a: a not in payees => shares[a] == 0` | ✅ | |
| [VS-P-27](./specs/parallelizer/parallelizer_valid_state.spec#L255) | `feeMintYBounded` | Mint fee Y-values weakly increasing and bounded by BASE_12<br>`forall t: yFeeMint[t][0] <= yFeeMint[t][1] <= BASE_12` | ✅ | |
| [VS-P-28](./specs/parallelizer/parallelizer_valid_state.spec#L266) | `feeBurnYBounded` | Burn fee Y-values weakly increasing and bounded by BASE_9<br>`forall t: yFeeBurn[t][0] <= yFeeBurn[t][1] <= BASE_9` | ✅ | |
| [VS-P-29](./specs/parallelizer/parallelizer_valid_state.spec#L277) | `redemptionCurveYBounded` | Redemption curve Y-values bounded in [0, BASE_9]<br>`forall i: i < len(yRedemptionCurve) => 0 <= yRedemptionCurve[i] <= BASE_9` | ✅ | |
| [VS-P-30](./specs/parallelizer/parallelizer_valid_state.spec#L285) | `feeMintXAnchor` | xFeeMint[0] == 0<br>`forall t: len(xFeeMint[t]) > 0 => xFeeMint[t][0] == 0` | ✅ | |
| [VS-P-31](./specs/parallelizer/parallelizer_valid_state.spec#L293) | `feeBurnXAnchor` | xFeeBurn[0] == BASE_9<br>`forall t: len(xFeeBurn[t]) > 0 => xFeeBurn[t][0] == BASE_9` | ✅ | |
| [VS-P-32](./specs/parallelizer/parallelizer_valid_state.spec#L301) | `redemptionCurveXBounded` | Redemption curve X bounded by BASE_9<br>`forall i: i < len(xRedemptionCurve) => xRedemptionCurve[i] <= BASE_9` | ✅ | |

#### Savings

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VS-SA-01](./specs/savings/savings_valid_state.spec#L23) | `pausedFlagIsZeroOrOne` | Paused flag is always 0 or 1<br>`paused == 0 OR paused == 1` | ✅ | |
| [VS-SA-02](./specs/savings/savings_valid_state.spec#L29) | `trustedUpdaterFlagIsZeroOrOne` | Trusted updater mapping values are always 0 or 1<br>`forall a: isTrustedUpdater[a] == 0 OR isTrustedUpdater[a] == 1` | ✅ | |
| [VS-SA-03](./specs/savings/savings_valid_state.spec#L36) | `lastUpdateNeverExceedsBlockTime` | lastUpdate never exceeds current block timestamp<br>`lastUpdate <= block.timestamp` | ✅ | |
| [VS-SA-04](./specs/savings/savings_valid_state.spec#L42) | `vaultNeverApprovesOwnShares` | The vault itself never has share allowances granted<br>`forall spender: allowance[vault][spender] == 0` | ✅ | |

#### FlashParallelToken

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VS-FP-01](./specs/flashParallelToken/flashParallelToken_valid_state.spec#L20) | `feeRateNeverExceedsMax` | Fee rate never exceeds 100% (PERCENTAGE_FACTOR)<br>`forall token: feesRate[token] <= PERCENTAGE_FACTOR` | ✅ | |

### Variable Transitions

Variable Transition properties verify that specific storage variables change only under expected conditions (monotonicity, set-once semantics, or authorized-function restrictions). These are parametric — each property is automatically verified against every external function in the contract, including functions added in the future.

#### Parallelizer

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VT-P-01](./specs/parallelizer/parallelizer_variable_transitions.spec#L12) | `tokenPNeverChanges` | tokenP is immutable after initialization<br>`forall f: tokenP' == tokenP` | ✅ | |
| [VT-P-02](./specs/parallelizer/parallelizer_variable_transitions.spec#L33) | `lastReleasedAtOnlyIncreases` | lastReleasedAt only increases<br>`forall f: lastReleasedAt' >= lastReleasedAt` | ✅ | |
| [VT-P-03](./specs/parallelizer/parallelizer_variable_transitions.spec#L56) | `collateralDecimalsZeroNonZeroTransition` | Collateral decimals only transition between zero and non-zero<br>`forall f, t: decimals[t]' != decimals[t] => (decimals[t] == 0 OR decimals[t]' == 0)` | ✅ | |

#### Savings

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VT-SA-01](./specs/savings/savings_variable_transitions.spec#L7) | `assetAddressNeverChanges` | Asset address is immutable once initialized<br>`forall f: asset != 0 => asset' == asset` | ✅ | |
| [VT-SA-02](./specs/savings/savings_variable_transitions.spec#L23) | `lastUpdateOnlyAdvancesForward` | lastUpdate timestamp only advances forward<br>`forall f: lastUpdate' >= lastUpdate` | ✅ | |
| [VT-SA-03](./specs/savings/savings_variable_transitions.spec#L39) | `rateOnlyChangesViaSetRate` | Rate only changes via setRate<br>`forall f: rate' != rate => f in {setRate}` | ✅ | |
| [VT-SA-04](./specs/savings/savings_variable_transitions.spec#L55) | `pausedOnlyChangesViaTogglePause` | Paused only changes via togglePause<br>`forall f: paused' != paused => f in {togglePause}` | ✅ | |

#### FlashParallelToken

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VT-FP-01](./specs/flashParallelToken/flashParallelToken_variable_transitions.spec#L11) | `feeRecipientCannotBeResetToZero` | Fee recipient cannot be set back to zero once assigned<br>`forall f: feeRecipient != 0 => feeRecipient' != 0` | ✅ | |

### State Transitions

State Transition properties verify the correctness of transitions between valid states. These are parametric — each property is automatically verified against every external function in the contract, including functions added in the future.

#### Parallelizer

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [ST-P-01](./specs/parallelizer/parallelizer_state_transitions.spec#L9) | `renormalizationResetsNormalizerToBase27` | When both normalizer and normalizedStables change, normalizer resets to BASE_27<br>`forall f: norm' != norm AND NS' != NS => norm' == BASE_27` | ✅ | |
| [ST-P-02](./specs/parallelizer/parallelizer_state_transitions.spec#L31) | `perCollateralNsChangeImpliesGlobalNsChange` | Per-collateral normalizedStables change implies global change<br>`forall f, t: collatNS[t]' != collatNS[t] => NS' != NS` | ✅ | |
| [ST-P-03](./specs/parallelizer/parallelizer_state_transitions.spec#L50) | `globalNsChangeImpliesSomeCollatNsChange` | Global normalizedStables change implies at least one per-collateral change<br>`forall f: NS' != NS => exists t in collateralList: collatNS[t]' != collatNS[t]` | ✅ | |
| [ST-P-04](./specs/parallelizer/parallelizer_state_transitions.spec#L77) | `multipleCollateralNsChangesImplyRenormalization` | Multiple per-collateral changes imply renormalization<br>`forall f: collatNS[t0]' != collatNS[t0] AND collatNS[t1]' != collatNS[t1] => norm' == BASE_27` | ✅ | |
| [ST-P-05](./specs/parallelizer/parallelizer_state_transitions.spec#L101) | `collateralDecimalsAndListAtomicity` | Decimals set implies list grew; cleared implies list shrank<br>`forall f, t: (decimals[t]==0 AND decimals[t]'!=0 => listLen'>listLen) AND (decimals[t]!=0 AND decimals[t]'==0 => listLen'<listLen)` | ✅ | |
| [ST-P-06](./specs/parallelizer/parallelizer_state_transitions.spec#L122) | `collateralListGrowthPreservesTokenPSupply` | Adding a collateral does not change tokenP totalSupply<br>`forall f: listLen' > listLen => supply' == supply` | ✅ | |
| [ST-P-07](./specs/parallelizer/parallelizer_state_transitions.spec#L148) | `tokenPSupplyChangeImpliesProtocolStateChange` | TokenP supply change implies NS, normalizer, or surplus release changed<br>`forall f: supply' != supply => NS' != NS OR norm' != norm OR lastReleasedAt' == block.timestamp OR (supply - supply') * BASE_27 < NS` | ❌ | Violated at dust amounts due to integer division rounding in `swapExactInput()` and `swapExactOutput()` |
| [ST-P-08](./specs/parallelizer/parallelizer_state_transitions.spec#L176) | `stablecoinCapEnforcedOnNsIncrease` | Per-collateral normalizedStables increase respects stablecoinCap<br>`forall f, t: collatNS[t]' > collatNS[t] AND cap[t] > 0 => collatNS[t]' * norm' / BASE_27 <= cap[t]` | ❌ | Violated at dust amounts due to integer division rounding in `redeem()`, `redeemWithForfeit()` and `updateNormalizer()`. Violated in `adjustStablecoins` — see [Real Issue #4](#real-issue-4) |
| [ST-P-09](./specs/parallelizer/parallelizer_state_transitions.spec#L198) | `tokenPSupplyIncreaseImpliesNsIncrease` | TokenP supply increase implies normalizedStables increase<br>`forall f: supply' > supply => NS' > NS` | ✅ | |
| [ST-P-10](./specs/parallelizer/parallelizer_state_transitions.spec#L220) | `diamondTokenPBalanceChangeImpliesSurplusOrRecoveryPath` | Diamond tokenP balance change implies surplus path or recovery function<br>`forall f: bal[tokenP][diamond]' != bal[tokenP][diamond] => NS' != NS OR lastReleasedAt' == block.timestamp OR f.selector == recoverERC20.selector` | ✅ | |
| [ST-P-11](./specs/parallelizer/parallelizer_state_transitions.spec#L245) | `accountingChangePreservesConfiguration` | Accounting change preserves configuration parameters<br>`forall f: NS' != NS OR norm' != norm => config' == config` | ✅ | |
| [ST-P-12](./specs/parallelizer/parallelizer_state_transitions.spec#L295) | `lastReleasedAtOnlyUpdatesToTimestamp` | lastReleasedAt only updates to block.timestamp<br>`forall f: lastReleasedAt' != lastReleasedAt => lastReleasedAt' == block.timestamp` | ✅ | |
| [ST-P-13](./specs/parallelizer/parallelizer_state_transitions.spec#L315) | `collateralListShrinkImpliesZeroNs` | Removed collateral must have had zero normalizedStables<br>`forall f, t: listLen' < listLen AND decimals[t]!=0 AND decimals[t]'==0 => collatNS[t] == 0` | ✅ | |

#### Savings

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [ST-SA-01](./specs/savings/savings_state_transitions.spec#L6) | `stateChangeAlwaysRefreshesTimestamp` | Any state change forces lastUpdate to block.timestamp<br>`forall f: (rate' != rate OR supply' != supply) => lastUpdate' == block.timestamp` | ✅ | |
| [ST-SA-02](./specs/savings/savings_state_transitions.spec#L27) | `pausedContractCannotChangeShareSupply` | Share total supply cannot change while paused<br>`forall f: paused == 1 => supply' == supply` | ✅ | |
| [ST-SA-03](./specs/savings/savings_state_transitions.spec#L47) | `rateCannotIncreaseAboveMaxRate` | Rate can only increase up to maxRate<br>`forall f: rate' > rate => rate' <= maxRate'` | ✅ | |

#### FlashParallelToken

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [ST-FP-01](./specs/flashParallelToken/flashParallelToken_state_transitions.spec#L11) | `tokensOnlyWithdrawnToFeeRecipient` | Tokens can only leave the contract to the fee recipient<br>`forall f: bal[t][contract]' < bal[t][contract] => bal[t][feeRecipient]' - bal[t][feeRecipient] == bal[t][contract] - bal[t][contract]'` | ✅ | |
| [ST-FP-02](./specs/flashParallelToken/flashParallelToken_state_transitions.spec#L38) | `feeRecipientAndTokenSettingsChangeSeparately` | Fee recipient and token settings cannot change in the same call<br>`forall f: feeRecipient' != feeRecipient => (feesRate[t]' == feesRate[t] AND maxBorrowable[t]' == maxBorrowable[t] AND isActive[t]' == isActive[t])` | ✅ | |
| [ST-FP-03](./specs/flashParallelToken/flashParallelToken_state_transitions.spec#L61) | `tokenSettingsDoNotAffectOtherTokens` | Changing one token's settings does not affect other tokens<br>`forall f, tA != tB: config[tA]' != config[tA] => config[tB]' == config[tB]` | ✅ | |
| [ST-FP-04](./specs/flashParallelToken/flashParallelToken_state_transitions.spec#L93) | `configChangesDoNotMoveTokens` | Config changes do not move tokens or change supply<br>`forall f: configChanged => bal[t][acct]' == bal[t][acct] AND supply[t]' == supply[t]` | ✅ | |

### High-Level

High-Level properties verify multi-step business logic, user isolation across core operations, and monotonicity of core operations.

#### Parallelizer — Swapper

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [HL-SW-01](./specs/parallelizer/swapper/swapper_high_level.spec#L15) | `mintExactInputSupplyDelta` | Mint via swapExactInput increases TokenP totalSupply by exactly amountOut<br>`amountIn > 0 AND amountOut > 0 => supply' == supply + amountOut` | ✅ | |
| [HL-SW-02](./specs/parallelizer/swapper/swapper_high_level.spec#L38) | `burnExactInputSupplyDelta` | Burn via swapExactInput decreases TokenP totalSupply by exactly amountIn<br>`amountIn > 0 AND amountOut > 0 => supply' == supply - amountIn` | ✅ | |
| [HL-SW-03](./specs/parallelizer/swapper/swapper_high_level.spec#L61) | `mintExactOutputSupplyDelta` | Mint via swapExactOutput increases TokenP totalSupply by exactly amountOut<br>`amountIn > 0 AND amountOut > 0 => supply' == supply + amountOut` | ✅ | |
| [HL-SW-04](./specs/parallelizer/swapper/swapper_high_level.spec#L84) | `burnExactOutputSupplyDelta` | Burn via swapExactOutput decreases TokenP totalSupply by returned amountIn<br>`amountIn > 0 AND amountOut > 0 => supply' == supply - amountIn` | ✅ | |
| [HL-SW-05](./specs/parallelizer/swapper/swapper_high_level.spec#L113) | `mintCollateralTransfer` | Mint transfers exactly amountIn of collateral from sender to diamond<br>`amountIn > 0 AND amountOut > 0 => bal[collat][sender]' == bal[collat][sender] - amountIn AND bal[collat][diamond]' == bal[collat][diamond] + amountIn` | ✅ | |
| [HL-SW-06](./specs/parallelizer/swapper/swapper_high_level.spec#L141) | `burnCollateralTransfer` | Burn transfers exactly amountOut of collateral from diamond to recipient<br>`amountIn > 0 AND amountOut > 0 => bal[collat][diamond]' == bal[collat][diamond] - amountOut AND bal[collat][to]' == bal[collat][to] + amountOut` | ✅ | |
| [HL-SW-07](./specs/parallelizer/swapper/swapper_high_level.spec#L173) | `swapOnlyAffectsTargetCollateral` | Swap only affects target collateral's normalizedStables<br>`NOT (collatNS[c0]' != collatNS[c0] AND collatNS[c1]' != collatNS[c1])` | ✅ | |
| [HL-SW-08](./specs/parallelizer/swapper/swapper_high_level.spec#L204) | `zeroAmountInNoOp` | Zero amountIn produces zero output and no state changes<br>`amountIn == 0 => amountOut == 0 AND NS' == NS AND supply' == supply` | ✅ | |
| [HL-SW-09](./specs/parallelizer/swapper/swapper_high_level.spec#L235) | `mintExactInputIncreasesNs` | Mint via swapExactInput strictly increases normalizedStables<br>`amountIn > 0 AND amountOut > 0 => collatNS[t]' > collatNS[t] AND NS' > NS` | ✅ | |
| [HL-SW-10](./specs/parallelizer/swapper/swapper_high_level.spec#L260) | `burnExactInputDecreasesNs` | Burn via swapExactInput does not increase normalizedStables<br>`amountIn > 0 AND amountOut > 0 => collatNS[t]' <= collatNS[t] AND NS' <= NS` | ✅ | |
| [HL-SW-11](./specs/parallelizer/swapper/swapper_high_level.spec#L285) | `mintExactOutputIncreasesNs` | Mint via swapExactOutput strictly increases normalizedStables<br>`amountIn > 0 AND amountOut > 0 => collatNS[t]' > collatNS[t] AND NS' > NS` | ✅ | |
| [HL-SW-12](./specs/parallelizer/swapper/swapper_high_level.spec#L310) | `burnExactOutputDecreasesNs` | Burn via swapExactOutput does not increase normalizedStables<br>`amountIn > 0 AND amountOut > 0 => collatNS[t]' <= collatNS[t] AND NS' <= NS` | ✅ | |
| [HL-SW-13](./specs/parallelizer/swapper/swapper_high_level.spec#L339) | `swapExactInputSlippageGuarantee` | amountOut ≥ amountOutMin on success<br>`!REVERTS => amountOut >= amountOutMin` | ✅ | |
| [HL-SW-14](./specs/parallelizer/swapper/swapper_high_level.spec#L358) | `swapExactOutputSlippageGuarantee` | amountIn ≤ amountInMax on success<br>`!REVERTS => amountIn <= amountInMax` | ✅ | |
| [HL-SW-15](./specs/parallelizer/swapper/swapper_high_level.spec#L381) | `zeroAmountOutExactOutputNoOp` | swapExactOutput with zero amountOut is a no-op<br>`amountOut == 0 => amountIn == 0 AND NS' == NS AND supply' == supply` | ✅ | |
| [HL-SW-16](./specs/parallelizer/swapper/swapper_high_level.spec#L413) | `quoteInMatchesMintExactInput` | quoteIn matches swapExactInput for mint direction<br>`quoteIn(amountIn, collat, tokenP) == swapExactInput(amountIn, collat, tokenP)` | ✅ | |
| [HL-SW-17](./specs/parallelizer/swapper/swapper_high_level.spec#L433) | `quoteInMatchesBurnExactInput` | quoteIn matches swapExactInput for burn direction<br>`quoteIn(amountIn, tokenP, collat) == swapExactInput(amountIn, tokenP, collat)` | ✅ | |
| [HL-SW-18](./specs/parallelizer/swapper/swapper_high_level.spec#L457) | `quoteInSameForAnyCaller` | quoteIn returns same value regardless of caller<br>`forall sender1, sender2: quoteIn(sender1, ...) == quoteIn(sender2, ...)` | ✅ | |
| [HL-SW-19](./specs/parallelizer/swapper/swapper_high_level.spec#L481) | `quoteOutSameForAnyCaller` | quoteOut returns same value regardless of caller<br>`forall sender1, sender2: quoteOut(sender1, ...) == quoteOut(sender2, ...)` | ✅ | |

#### Parallelizer — Redeemer

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [HL-RD-01](./specs/parallelizer/redeemer/redeemer_high_level.spec#L15) | `updateNormalizerIncreaseGrowsStablecoinsIssued` | increase=true does not decrease stablecoinsIssued<br>`updateNormalizer(amount, true) => stablecoinsIssued' >= stablecoinsIssued` | ✅ | |
| [HL-RD-02](./specs/parallelizer/redeemer/redeemer_high_level.spec#L34) | `updateNormalizerDecreaseShrinksStablecoinsIssued` | increase=false does not increase stablecoinsIssued<br>`updateNormalizer(amount, false) => stablecoinsIssued' <= stablecoinsIssued` | ✅ | |
| [HL-RD-03](./specs/parallelizer/redeemer/redeemer_high_level.spec#L52) | `updateNormalizerIncreaseRaisesNormalizer` | Without renorm, increase=true raises normalizer<br>`updateNormalizer(amount, true) AND NS' = NS => norm' >= norm` | ✅ | |
| [HL-RD-04](./specs/parallelizer/redeemer/redeemer_high_level.spec#L72) | `updateNormalizerDecreaseLowersNormalizer` | Without renorm, increase=false lowers normalizer<br>`updateNormalizer(amount, false) AND NS' = NS => norm' <= norm` | ✅ | |
| [HL-RD-05](./specs/parallelizer/redeemer/redeemer_high_level.spec#L92) | `redeemDecreasesStablecoinsIssued` | After redeem, stablecoinsIssued ≤ before<br>`redeem(amount, ...) => stablecoinsIssued' <= stablecoinsIssued` | ✅ | |
| [HL-RD-06](./specs/parallelizer/redeemer/redeemer_high_level.spec#L117) | `largerRedeemDecreasesStablecoinsIssuedMore` | Larger redeem decreases stablecoinsIssued at least as much<br>`amount1 >= amount2 => stablecoinsIssued'(redeem(amount1)) <= stablecoinsIssued'(redeem(amount2))` | ✅ | |
| [HL-RD-07](./specs/parallelizer/redeemer/redeemer_high_level.spec#L143) | `largerUpdateNormalizerIncreasesMore` | Larger amount produces larger stablecoinsIssued increase<br>`amount1 >= amount2 => stablecoinsIssued'(updateNormalizer(amount1, true)) >= stablecoinsIssued'(updateNormalizer(amount2, true))` | ✅ | |
| [HL-RD-08](./specs/parallelizer/redeemer/redeemer_high_level.spec#L168) | `updateNormalizerIncreaseBoundedByAmount` | Increase in stablecoinsIssued bounded by amount<br>`updateNormalizer(amount, true) => stablecoinsIssued' - stablecoinsIssued <= amount` | ✅ | |
| [HL-RD-09](./specs/parallelizer/redeemer/redeemer_high_level.spec#L187) | `updateNormalizerDecreaseBoundedByAmount` | Decrease in stablecoinsIssued bounded by amount + renorm rounding<br>`updateNormalizer(amount, false) => stablecoinsIssued - stablecoinsIssued' <= amount` | ✅ | |
| [HL-RD-10](./specs/parallelizer/redeemer/redeemer_high_level.spec#L209) | `redeemWithoutRenormPreservesNormalizedStables` | If normalizer ≠ BASE_27 after redeem, normalizedStables unchanged<br>`redeem(amount, ...) AND norm' != BASE_27 => NS' = NS` | ✅ | |

#### Parallelizer — Surplus

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [HL-S-01](./specs/parallelizer/surplus/surplus_high_level.spec#L12) | `processSurplusIncreasesTokenPSupply` | processSurplus increases TokenP totalSupply by exactly issuedAmount<br>`supply' == supply + issuedAmount` | ✅ | |
| [HL-S-02](./specs/parallelizer/surplus/surplus_high_level.spec#L36) | `processSurplusCreditsTokenPToDiamond` | processSurplus credits exactly issuedAmount to diamond<br>`bal[TokenP][diamond]' == bal[TokenP][diamond] + issuedAmount` | ✅ | |
| [HL-S-03](./specs/parallelizer/surplus/surplus_high_level.spec#L60) | `releaseDistributesAllTokenP` | release does not increase diamond TokenP balance<br>`bal[TokenP][diamond]' <= bal[TokenP][diamond]` | ✅ | |
| [HL-S-04](./specs/parallelizer/surplus/surplus_high_level.spec#L81) | `processSurplusIncreasesNormalizedStables` | processSurplus does not decrease global normalizedStables<br>`NS' >= NS` | ✅ | |
| [HL-S-05](./specs/parallelizer/surplus/surplus_high_level.spec#L105) | `processSurplusIncreasesPerCollateralNs` | processSurplus does not decrease per-collateral normalizedStables<br>`collatNS[collateral]' >= collatNS[collateral]` | ✅ | |
| [HL-S-06](./specs/parallelizer/surplus/surplus_high_level.spec#L133) | `processSurplusPositiveIssuedImpliesNsIncrease` | Positive issuedAmount implies strict normalizedStables increase<br>`issuedAmount > 0 => NS' > NS` | ✅ | |
| [HL-S-07](./specs/parallelizer/surplus/surplus_high_level.spec#L159) | `surplusLifecycleProcessThenRelease` | processSurplus then release: balance does not increase<br>`bal[TokenP][diamond] == 0 => bal[TokenP][diamond]_afterRelease <= bal[TokenP][diamond]_afterProcess` | ✅ | |
| [HL-S-08](./specs/parallelizer/surplus/surplus_high_level.spec#L193) | `processSurplusCapsToMaxCollateral` | collateralSurplus capped to maxCollateralAmount<br>`maxCollateralAmount > 0 => collateralSurplus <= maxCollateralAmount` | ✅ | |
| [HL-S-09](./specs/parallelizer/surplus/surplus_high_level.spec#L215) | `processSurplusReachableWithHighBufferRatio` | Reachable when surplusBufferRatio > 100%<br>`satisfy(surplusBufferRatio > BASE_9 AND collateralSurplus > 0 AND issuedAmount > 0)` | ❌ | Violated — see [Real Issue #2](#real-issue-2) |
| [HL-S-10](./specs/parallelizer/surplus/surplus_high_level.spec#L236) | `processSurplusManagedCollateralIsReachable` | Reachable for managed collateral<br>`satisfy(isManaged[collateral] > 0 AND collateralSurplus > 0 AND issuedAmount > 0)` | ❌ | Violated — see [Real Issue #1](#real-issue-1) |

#### Parallelizer — Setter

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [HL-SET-01](./specs/parallelizer/setter/setter_high_level.spec#L11) | `togglePauseMintRoundTrip` | togglePause(Mint) twice restores isMintLive<br>`togglePauseMint; togglePauseMint => isMintLive' == isMintLive` | ✅ | |
| [HL-SET-02](./specs/parallelizer/setter/setter_high_level.spec#L25) | `togglePauseBurnRoundTrip` | togglePause(Burn) twice restores isBurnLive<br>`togglePauseBurn; togglePauseBurn => isBurnLive' == isBurnLive` | ✅ | |
| [HL-SET-03](./specs/parallelizer/setter/setter_high_level.spec#L39) | `togglePauseRedeemRoundTrip` | togglePause(Redeem) twice restores isRedemptionLive<br>`togglePauseRedeem; togglePauseRedeem => isRedemptionLive' == isRedemptionLive` | ✅ | |
| [HL-SET-04](./specs/parallelizer/setter/setter_high_level.spec#L53) | `toggleTrustedUpdaterRoundTrip` | toggleTrusted(Updater) twice restores isTrusted<br>`toggleTrustedUpdater(a); toggleTrustedUpdater(a) => isTrusted'[a] == isTrusted[a]` | ✅ | |
| [HL-SET-05](./specs/parallelizer/setter/setter_high_level.spec#L67) | `toggleTrustedSellerRoundTrip` | toggleTrusted(Seller) twice restores isSellerTrusted<br>`toggleTrustedSeller(a); toggleTrustedSeller(a) => isSellerTrusted'[a] == isSellerTrusted[a]` | ✅ | |
| [HL-SET-06](./specs/parallelizer/setter/setter_high_level.spec#L82) | `adjustStablecoinsRoundTrip` | Increase then decrease restores normalizedStables<br>`adjustStablecoins(t, amt, true); adjustStablecoins(t, amt, false) => NS' == NS AND collatNS'[t] == collatNS[t]` | ✅ | |
| [HL-SET-07](./specs/parallelizer/setter/setter_high_level.spec#L103) | `adjustStablecoinsIncreaseMonotonicity` | increase=true does not decrease normalizedStables<br>`amt > 0 => adjustStablecoins(t, amt, true) => NS' >= NS AND collatNS'[t] >= collatNS[t]` | ✅ | |
| [HL-SET-08](./specs/parallelizer/setter/setter_high_level.spec#L118) | `adjustStablecoinsDecreaseMonotonicity` | increase=false does not increase normalizedStables<br>`amt > 0 => adjustStablecoins(t, amt, false) => NS' <= NS AND collatNS'[t] <= collatNS[t]` | ✅ | |
| [HL-SET-09](./specs/parallelizer/setter/setter_high_level.spec#L137) | `adjustStablecoinsGlobalLocalDeltaConservation` | Per-collateral delta equals global delta<br>`adjustStablecoins(t, amt, inc) => NS' - NS == collatNS'[t] - collatNS[t]` | ✅ | |

#### Savings

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [HL-SA-01](./specs/savings/savings_high_level.spec#L13) | `depositThenRedeemCannotProfit` | Depositing then redeeming yields at most the original assets<br>`redeem(deposit(assets)) <= assets` | ✅ | |
| [HL-SA-02](./specs/savings/savings_high_level.spec#L26) | `mintThenRedeemCannotProfit` | Minting then redeeming yields at most the paid assets<br>`redeem(shares) <= mint(shares)` | ✅ | |
| [HL-SA-03](./specs/savings/savings_high_level.spec#L39) | `depositThenWithdrawCannotProfit` | Depositing then withdrawing burns at least the minted shares<br>`withdraw(assets).sharesBurned >= deposit(assets).sharesMinted` | ✅ | |
| [HL-SA-04](./specs/savings/savings_high_level.spec#L52) | `withdrawThenDepositCannotProfit` | Withdrawing then depositing mints at most the burned shares<br>`deposit(assets).sharesMinted <= withdraw(assets).sharesBurned` | ✅ | |
| [HL-SA-05](./specs/savings/savings_high_level.spec#L65) | `redeemThenMintCannotProfit` | Redeeming then minting costs at least the received assets<br>`mint(shares) >= redeem(shares)` | ✅ | |
| [HL-SA-06](./specs/savings/savings_high_level.spec#L78) | `mintThenWithdrawCannotProfit` | Minting then withdrawing burns at least the minted shares<br>`withdraw(mint(shares)).sharesBurned >= shares` | ✅ | |
| [HL-SA-07](./specs/savings/savings_high_level.spec#L98) | `moreAssetsDepositedMeansMoreShares` | Deposit monotonicity: more assets yield at least as many shares<br>`assets1 >= assets2 => deposit(assets1).shares >= deposit(assets2).shares` | ✅ | |
| [HL-SA-08](./specs/savings/savings_high_level.spec#L113) | `moreSharesRedeemedMeansMoreAssets` | Redeem monotonicity: more shares yield at least as many assets<br>`shares1 >= shares2 => redeem(shares1).assets >= redeem(shares2).assets` | ✅ | |
| [HL-SA-09](./specs/savings/savings_high_level.spec#L128) | `moreSharesMintedCostsMoreAssets` | Mint monotonicity: more shares cost at least as many assets<br>`shares1 >= shares2 => mint(shares1).assets >= mint(shares2).assets` | ✅ | |
| [HL-SA-10](./specs/savings/savings_high_level.spec#L143) | `moreAssetsWithdrawnBurnsMoreShares` | Withdraw monotonicity: more assets burn at least as many shares<br>`assets1 >= assets2 => withdraw(assets1).shares >= withdraw(assets2).shares` | ✅ | |
| [HL-SA-11](./specs/savings/savings_high_level.spec#L162) | `depositDoesNotDiluteShares` | Deposit does not decrease share-to-asset exchange rate<br>`convertToAssets(1e18)' >= convertToAssets(1e18) after deposit` | ✅ | |
| [HL-SA-12](./specs/savings/savings_high_level.spec#L176) | `mintDoesNotDiluteShares` | Mint does not decrease share-to-asset exchange rate<br>`convertToAssets(1e18)' >= convertToAssets(1e18) after mint` | ✅ | |
| [HL-SA-13](./specs/savings/savings_high_level.spec#L190) | `redeemDoesNotDiluteShares` | Redeem does not decrease share-to-asset exchange rate<br>`convertToAssets(1e18)' >= convertToAssets(1e18) after redeem` | ✅ | |
| [HL-SA-14](./specs/savings/savings_high_level.spec#L204) | `withdrawDoesNotDiluteShares` | Withdraw does not decrease share-to-asset exchange rate<br>`convertToAssets(1e18)' >= convertToAssets(1e18) after withdraw` | ✅ | |
| [HL-SA-15](./specs/savings/savings_high_level.spec#L222) | `depositIsAdditive` | Splitting a deposit produces total shares within 1 of a single deposit<br>`|deposit(a+b).shares - (deposit(a).shares + deposit(b).shares)| <= 1` | ✅ | |

#### FlashParallelToken

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [HL-FP-01](./specs/flashParallelToken/flashParallelToken_high_level.spec#L7) | `flashLoanDoesNotChangeTotalSupply` | Flash loan mint and burn cancel out, total supply unchanged<br>`supply[t]' == supply[t] after flashLoan` | ✅ | |
| [HL-FP-02](./specs/flashParallelToken/flashParallelToken_high_level.spec#L22) | `flashLoanCollectsExactFee` | Contract balance increases by exactly the expected fee<br>`bal[t][contract]' == bal[t][contract] + (amount * feesRate[t] + 5000) / 10000 after flashLoan` | ✅ | |
| [HL-FP-03](./specs/flashParallelToken/flashParallelToken_high_level.spec#L38) | `borrowerCannotProfitFromFlashLoan` | Borrower's balance cannot increase after a flash loan<br>`bal[t][receiver]' <= bal[t][receiver] after flashLoan` | ✅ | |
| [HL-FP-04](./specs/flashParallelToken/flashParallelToken_high_level.spec#L53) | `flashLoanFeeNeverExceedsBorrowedAmount` | Fee charged never exceeds the borrowed amount<br>`bal[t][contract]' - bal[t][contract] <= amount after flashLoan` | ✅ | |

### Unit Tests

Unit Test properties verify basic behavior of individual functions: revert conditions, direct effects, and non-effects on unrelated state.

#### Parallelizer — Swapper

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-SW-01](./specs/parallelizer/swapper/swapper_unit_tests.spec#L11) | `swapExactInputRevertsAfterDeadline` | Reverts when block.timestamp > deadline<br>`block.timestamp > deadline => REVERTS` | ✅ | |
| [UT-SW-02](./specs/parallelizer/swapper/swapper_unit_tests.spec#L26) | `swapExactOutputRevertsAfterDeadline` | Reverts when block.timestamp > deadline<br>`block.timestamp > deadline => REVERTS` | ✅ | |
| [UT-SW-03](./specs/parallelizer/swapper/swapper_unit_tests.spec#L45) | `swapExactInputMintRevertsWhenPaused` | Mint reverts when isMintLive == 0<br>`isMintLive[collat] == 0 => REVERTS` | ✅ | |
| [UT-SW-04](./specs/parallelizer/swapper/swapper_unit_tests.spec#L61) | `swapExactInputBurnRevertsWhenPaused` | Burn reverts when isBurnLive == 0<br>`isBurnLive[collat] == 0 => REVERTS` | ✅ | |
| [UT-SW-05](./specs/parallelizer/swapper/swapper_unit_tests.spec#L77) | `swapExactOutputMintRevertsWhenPaused` | Mint reverts when isMintLive == 0<br>`isMintLive[collat] == 0 => REVERTS` | ✅ | |
| [UT-SW-06](./specs/parallelizer/swapper/swapper_unit_tests.spec#L93) | `swapExactOutputBurnRevertsWhenPaused` | Burn reverts when isBurnLive == 0<br>`isBurnLive[collat] == 0 => REVERTS` | ✅ | |
| [UT-SW-07](./specs/parallelizer/swapper/swapper_unit_tests.spec#L113) | `mintExactInputIsReachable` | Mint via swapExactInput is reachable with positive output<br>`satisfy(swapExactInput(amountIn, collat, tokenP).amountOut > 0)` | ✅ | |
| [UT-SW-08](./specs/parallelizer/swapper/swapper_unit_tests.spec#L131) | `burnExactInputIsReachable` | Burn via swapExactInput is reachable with positive output<br>`satisfy(swapExactInput(amountIn, tokenP, collat).amountOut > 0)` | ✅ | |
| [UT-SW-09](./specs/parallelizer/swapper/swapper_unit_tests.spec#L149) | `mintExactOutputIsReachable` | Mint via swapExactOutput is reachable with positive input<br>`satisfy(swapExactOutput(amountOut, collat, tokenP).amountIn > 0)` | ✅ | |
| [UT-SW-10](./specs/parallelizer/swapper/swapper_unit_tests.spec#L167) | `burnExactOutputIsReachable` | Burn via swapExactOutput is reachable with positive input<br>`satisfy(swapExactOutput(amountOut, tokenP, collat).amountIn > 0)` | ✅ | |

#### Parallelizer — Redeemer

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-RD-01](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L11) | `redeemBurnsExactAmount` | Sender's TokenP balance decreases by exactly amount<br>`redeem(amount, ...) => bal[tokenP][sender] - bal'[tokenP][sender] = amount` | ✅ | |
| [UT-RD-02](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L32) | `redeemDecreasesTokenPTotalSupply` | TokenP total supply decreases by exactly amount<br>`redeem(amount, ...) => supply[tokenP] - supply'[tokenP] = amount` | ✅ | |
| [UT-RD-03](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L53) | `redeemWithForfeitBurnsExactAmount` | Sender's balance decreases by exactly amount<br>`redeemWithForfeit(amount, ...) => bal[tokenP][sender] - bal'[tokenP][sender] = amount` | ✅ | |
| [UT-RD-04](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L75) | `redeemWithForfeitDecreasesTokenPTotalSupply` | Total supply decreases by exactly amount<br>`redeemWithForfeit(amount, ...) => supply[tokenP] - supply'[tokenP] = amount` | ✅ | |
| [UT-RD-05](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L97) | `updateNormalizerReturnMatchesGhost` | Return value matches post-call normalizer<br>`retVal = updateNormalizer(amount, increase) => retVal = norm'` | ✅ | |
| [UT-RD-06](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L116) | `redeemRevertsAfterDeadline` | Reverts when block.timestamp > deadline<br>`block.timestamp > deadline => redeem(amount, ..., deadline, ...) REVERTS` | ✅ | |
| [UT-RD-07](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L132) | `redeemRevertsWhenPaused` | Reverts when isRedemptionLive == 0<br>`isRedemptionLive = 0 => redeem(amount, ...) REVERTS` | ✅ | |
| [UT-RD-08](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L151) | `redeemWithForfeitRevertsAfterDeadline` | Reverts when block.timestamp > deadline<br>`block.timestamp > deadline => redeemWithForfeit(amount, ..., deadline, ...) REVERTS` | ✅ | |
| [UT-RD-09](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L169) | `redeemWithForfeitRevertsWhenPaused` | Reverts when isRedemptionLive == 0<br>`isRedemptionLive = 0 => redeemWithForfeit(amount, ...) REVERTS` | ✅ | |
| [UT-RD-10](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L192) | `updateNormalizerDoesNotChangeTokenPSupply` | Does not change TokenP total supply<br>`updateNormalizer(amount, increase) => supply'[tokenP] = supply[tokenP]` | ✅ | |
| [UT-RD-11](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L215) | `redeemIsolatesThirdPartyTokenPBalance` | Does not change third-party TokenP balances<br>`third != sender => redeem(amount, ...) => bal'[tokenP][third] = bal[tokenP][third]` | ✅ | |
| [UT-RD-12](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L243) | `redeemIsReachable` | Redeem is reachable with positive burn<br>`satisfy(redeem(amount, ...) AND bal[tokenP][sender] - bal'[tokenP][sender] > 0)` | ✅ | |
| [UT-RD-13](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L263) | `redeemWithForfeitIsReachable` | RedeemWithForfeit is reachable with positive burn<br>`satisfy(redeemWithForfeit(amount, ...) AND bal[tokenP][sender] - bal'[tokenP][sender] > 0)` | ✅ | |
| [UT-RD-14](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L284) | `updateNormalizerIsReachable` | UpdateNormalizer is reachable with normalizer change<br>`satisfy(updateNormalizer(amount, increase) AND norm' != norm)` | ✅ | |
| [UT-RD-15](./specs/parallelizer/redeemer/redeemer_unit_tests.spec#L302) | `quoteRedemptionCurveIsReachable` | QuoteRedemptionCurve is reachable with positive amount<br>`satisfy(!REVERTS AND amount > 0)` | ✅ | |

#### Parallelizer — Setter

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-SET-01](./specs/parallelizer/setter/setter_unit_tests.spec#L10) | `togglePauseMintIntegrity` | Toggles isMintLive (1 - current)<br>`togglePauseMint(t) => isMintLive'[t] == 1 - isMintLive[t]` | ✅ | |
| [UT-SET-02](./specs/parallelizer/setter/setter_unit_tests.spec#L22) | `togglePauseBurnIntegrity` | Toggles isBurnLive (1 - current)<br>`togglePauseBurn(t) => isBurnLive'[t] == 1 - isBurnLive[t]` | ✅ | |
| [UT-SET-03](./specs/parallelizer/setter/setter_unit_tests.spec#L34) | `togglePauseRedeemIntegrity` | Toggles isRedemptionLive (1 - current)<br>`togglePauseRedeem() => isRedemptionLive' == 1 - isRedemptionLive` | ✅ | |
| [UT-SET-04](./specs/parallelizer/setter/setter_unit_tests.spec#L47) | `toggleTrustedUpdaterIntegrity` | Toggles isTrusted (1 - current)<br>`toggleTrustedUpdater(a) => isTrusted'[a] == 1 - isTrusted[a]` | ✅ | |
| [UT-SET-05](./specs/parallelizer/setter/setter_unit_tests.spec#L60) | `toggleTrustedSellerIntegrity` | Toggles isSellerTrusted (1 - current)<br>`toggleTrustedSeller(a) => isSellerTrusted'[a] == 1 - isSellerTrusted[a]` | ✅ | |
| [UT-SET-06](./specs/parallelizer/setter/setter_unit_tests.spec#L73) | `updateSlippageToleranceIntegrity` | Sets slippageTolerance to param<br>`updateSlippageTolerance(t, val) => slippageTolerance'[t] == val` | ✅ | |
| [UT-SET-07](./specs/parallelizer/setter/setter_unit_tests.spec#L83) | `updateSurplusBufferRatioIntegrity` | Sets surplusBufferRatio to param<br>`updateSurplusBufferRatio(val) => surplusBufferRatio' == val` | ✅ | |
| [UT-SET-08](./specs/parallelizer/setter/setter_unit_tests.spec#L94) | `setStablecoinCapIntegrity` | Sets stablecoinCap to param<br>`setStablecoinCap(t, cap) => stablecoinCap'[t] == cap` | ✅ | |
| [UT-SET-09](./specs/parallelizer/setter/setter_unit_tests.spec#L104) | `setAccessManagerIntegrity` | Sets accessManager to param<br>`setAccessManager(addr) => accessManager' == addr` | ✅ | |
| [UT-SET-10](./specs/parallelizer/setter/setter_unit_tests.spec#L119) | `togglePauseMintRevertsNotCollateral` | Reverts when decimals == 0<br>`decimals[t] == 0 => togglePauseMint(t) REVERTS` | ✅ | |
| [UT-SET-11](./specs/parallelizer/setter/setter_unit_tests.spec#L129) | `togglePauseBurnRevertsNotCollateral` | Reverts when decimals == 0<br>`decimals[t] == 0 => togglePauseBurn(t) REVERTS` | ✅ | |
| [UT-SET-12](./specs/parallelizer/setter/setter_unit_tests.spec#L139) | `adjustStablecoinsRevertsNotCollateral` | Reverts when decimals == 0<br>`decimals[t] == 0 => adjustStablecoins(t, amt, inc) REVERTS` | ✅ | |
| [UT-SET-13](./specs/parallelizer/setter/setter_unit_tests.spec#L150) | `updateSlippageToleranceRevertsInvalidRate` | Reverts when tolerance > BASE_9<br>`val > BASE_9 => updateSlippageTolerance(t, val) REVERTS` | ✅ | |
| [UT-SET-14](./specs/parallelizer/setter/setter_unit_tests.spec#L161) | `updateSurplusBufferRatioRevertsInvalidParam` | Reverts when ratio < BASE_9<br>`val < BASE_9 => updateSurplusBufferRatio(val) REVERTS` | ✅ | |
| [UT-SET-15](./specs/parallelizer/setter/setter_unit_tests.spec#L171) | `setStablecoinCapRevertsNotCollateral` | Reverts when decimals == 0<br>`decimals[t] == 0 => setStablecoinCap(t, cap) REVERTS` | ✅ | |
| [UT-SET-16](./specs/parallelizer/setter/setter_unit_tests.spec#L182) | `revokeCollateralRevertsNotCollateral` | Reverts when decimals == 0<br>`decimals[t] == 0 => revokeCollateral(t) REVERTS` | ✅ | |
| [UT-SET-17](./specs/parallelizer/setter/setter_unit_tests.spec#L193) | `revokeCollateralRevertsCollateralBacked` | Reverts when normalizedStables > 0<br>`decimals[t] != 0 AND collatNS[t] > 0 => revokeCollateral(t) REVERTS` | ✅ | |
| [UT-SET-18](./specs/parallelizer/setter/setter_unit_tests.spec#L204) | `updateOracleRevertsNotTrusted` | Reverts when caller is not seller trusted<br>`isSellerTrusted[msg.sender] == 0 => updateOracle(t) REVERTS` | ✅ | |
| [UT-SET-19](./specs/parallelizer/setter/setter_unit_tests.spec#L218) | `togglePauseMintDoesNotChangeBurnLive` | Does not change isBurnLive<br>`togglePauseMint(t) => isBurnLive'[t] == isBurnLive[t]` | ✅ | |
| [UT-SET-20](./specs/parallelizer/setter/setter_unit_tests.spec#L230) | `togglePauseBurnDoesNotChangeMintLive` | Does not change isMintLive<br>`togglePauseBurn(t) => isMintLive'[t] == isMintLive[t]` | ✅ | |
| [UT-SET-21](./specs/parallelizer/setter/setter_unit_tests.spec#L242) | `togglePauseMintDoesNotChangeRedemptionLive` | Does not change isRedemptionLive<br>`togglePauseMint(t) => isRedemptionLive' == isRedemptionLive` | ✅ | |
| [UT-SET-22](./specs/parallelizer/setter/setter_unit_tests.spec#L254) | `togglePauseRedeemDoesNotChangeMintLive` | Does not change isMintLive for any collateral<br>`forall t: togglePauseRedeem() => isMintLive'[t] == isMintLive[t]` | ✅ | |
| [UT-SET-23](./specs/parallelizer/setter/setter_unit_tests.spec#L266) | `toggleTrustedUpdaterDoesNotChangeSellerTrusted` | Does not change isSellerTrusted<br>`toggleTrustedUpdater(a) => isSellerTrusted'[a] == isSellerTrusted[a]` | ✅ | |
| [UT-SET-24](./specs/parallelizer/setter/setter_unit_tests.spec#L279) | `toggleTrustedSellerDoesNotChangeTrusted` | Does not change isTrusted<br>`toggleTrustedSeller(a) => isTrusted'[a] == isTrusted[a]` | ✅ | |
| [UT-SET-25](./specs/parallelizer/setter/setter_unit_tests.spec#L292) | `adjustStablecoinsIsolatesOtherCollateral` | Does not change other collateral's normalizedStables<br>`t != other => adjustStablecoins(t, amt, inc) => collatNS'[other] == collatNS[other]` | ✅ | |
| [UT-SET-26](./specs/parallelizer/setter/setter_unit_tests.spec#L306) | `updateSlippageToleranceIsolatesOtherCollateral` | Does not change other collateral's tolerance<br>`t != other => updateSlippageTolerance(t, val) => slippageTolerance'[other] == slippageTolerance[other]` | ✅ | |
| [UT-SET-27](./specs/parallelizer/setter/setter_unit_tests.spec#L320) | `setStablecoinCapIsolatesOtherCollateral` | Does not change other collateral's cap<br>`t != other => setStablecoinCap(t, cap) => stablecoinCap'[other] == stablecoinCap[other]` | ✅ | |
| [UT-SET-28](./specs/parallelizer/setter/setter_unit_tests.spec#L338) | `togglePauseMintIsReachable` | Reachable with state change<br>`satisfy(isMintLive'[t] != isMintLive[t])` | ✅ | |
| [UT-SET-29](./specs/parallelizer/setter/setter_unit_tests.spec#L350) | `adjustStablecoinsIsReachable` | Reachable with global NS increase<br>`satisfy(NS' > NS)` | ✅ | |
| [UT-SET-30](./specs/parallelizer/setter/setter_unit_tests.spec#L362) | `updateSlippageToleranceIsReachable` | Reachable with positive tolerance<br>`satisfy(slippageTolerance'[t] > 0)` | ✅ | |
| [UT-SET-31](./specs/parallelizer/setter/setter_unit_tests.spec#L372) | `updateSurplusBufferRatioIsReachable` | Reachable with valid ratio<br>`satisfy(surplusBufferRatio' >= BASE_9)` | ✅ | |
| [UT-SET-32](./specs/parallelizer/setter/setter_unit_tests.spec#L383) | `setStablecoinCapIsReachable` | Reachable with positive cap<br>`satisfy(stablecoinCap'[t] > 0)` | ✅ | |

#### Parallelizer — Surplus

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-SUR-01](./specs/parallelizer/surplus/surplus_unit_tests.spec#L10) | `releaseUpdatesLastReleasedAt` | lastReleasedAt == block.timestamp after release<br>`lastReleasedAt' == block.timestamp` | ✅ | |
| [UT-SUR-02](./specs/parallelizer/surplus/surplus_unit_tests.spec#L29) | `processSurplusRevertsWhenBufferRatioZero` | Reverts when surplusBufferRatio == 0<br>`surplusBufferRatio == 0 => REVERTS` | ✅ | |
| [UT-SUR-03](./specs/parallelizer/surplus/surplus_unit_tests.spec#L46) | `releaseRevertsWhenIncomeZero` | Reverts when diamond TokenP balance == 0<br>`bal[TokenP][diamond] == 0 => REVERTS` | ✅ | |
| [UT-SUR-04](./specs/parallelizer/surplus/surplus_unit_tests.spec#L60) | `releaseRevertsWhenNoPayees` | Reverts when payees array is empty<br>`payeesLength == 0 => REVERTS` | ✅ | |
| [UT-SUR-05](./specs/parallelizer/surplus/surplus_unit_tests.spec#L77) | `processSurplusDoesNotChangeLastReleasedAt` | Does not change lastReleasedAt<br>`lastReleasedAt' == lastReleasedAt` | ✅ | |
| [UT-SUR-06](./specs/parallelizer/surplus/surplus_unit_tests.spec#L94) | `processSurplusDoesNotChangeTotalShares` | Does not change totalShares<br>`totalShares' == totalShares` | ✅ | |
| [UT-SUR-07](./specs/parallelizer/surplus/surplus_unit_tests.spec#L111) | `processSurplusDoesNotChangePayeesLength` | Does not change payees.length<br>`payeesLength' == payeesLength` | ✅ | |
| [UT-SUR-08](./specs/parallelizer/surplus/surplus_unit_tests.spec#L128) | `processSurplusDoesNotChangeSurplusBufferRatio` | Does not change surplusBufferRatio<br>`surplusBufferRatio' == surplusBufferRatio` | ✅ | |
| [UT-SUR-09](./specs/parallelizer/surplus/surplus_unit_tests.spec#L149) | `releaseDoesNotChangeNormalizedStables` | Does not change global normalizedStables<br>`NS' == NS` | ✅ | |
| [UT-SUR-10](./specs/parallelizer/surplus/surplus_unit_tests.spec#L162) | `releaseDoesNotChangePerCollateralNs` | Does not change per-collateral normalizedStables<br>`forall t: collatNS[t]' == collatNS[t]` | ✅ | |
| [UT-SUR-11](./specs/parallelizer/surplus/surplus_unit_tests.spec#L175) | `releaseDoesNotChangeSurplusBufferRatio` | Does not change surplusBufferRatio<br>`surplusBufferRatio' == surplusBufferRatio` | ✅ | |
| [UT-SUR-12](./specs/parallelizer/surplus/surplus_unit_tests.spec#L188) | `releaseDoesNotChangeTotalShares` | Does not change totalShares<br>`totalShares' == totalShares` | ✅ | |
| [UT-SUR-13](./specs/parallelizer/surplus/surplus_unit_tests.spec#L205) | `processSurplusIsReachable` | Reachable with positive issuedAmount<br>`satisfy(issuedAmount > 0)` | ✅ | |
| [UT-SUR-14](./specs/parallelizer/surplus/surplus_unit_tests.spec#L223) | `processSurplusZeroIssuedIsReachable` | Reachable with zero issuedAmount<br>`satisfy(issuedAmount == 0)` | ✅ | |
| [UT-SUR-15](./specs/parallelizer/surplus/surplus_unit_tests.spec#L241) | `releaseIsReachable` | Release is reachable<br>`satisfy(lastReleasedAt' == block.timestamp)` | ✅ | |

#### Savings

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-SA-01](./specs/savings/savings_unit_tests.spec#L10) | `depositIncreasesReceiverShares` | Deposit increases receiver's share balance by returned shares<br>`bal[shares][receiver]' == bal[shares][receiver] + deposit(assets).shares` | ✅ | |
| [UT-SA-02](./specs/savings/savings_unit_tests.spec#L25) | `mintIncreasesReceiverShares` | Mint increases receiver's share balance by requested shares<br>`bal[shares][receiver]' == bal[shares][receiver] + shares` | ✅ | |
| [UT-SA-03](./specs/savings/savings_unit_tests.spec#L40) | `depositTransfersAssetsFromSender` | Deposit transfers exactly deposited assets from sender<br>`bal[asset][sender]' == bal[asset][sender] - assets` | ✅ | |
| [UT-SA-04](./specs/savings/savings_unit_tests.spec#L55) | `withdrawDecreasesOwnerShares` | Withdraw decreases owner's share balance by returned shares<br>`bal[shares][owner]' == bal[shares][owner] - withdraw(assets).shares` | ✅ | |
| [UT-SA-05](./specs/savings/savings_unit_tests.spec#L70) | `redeemTransfersAssetsToReceiver` | Redeem transfers returned assets to receiver<br>`bal[asset][receiver]' == bal[asset][receiver] + redeem(shares).assets` | ✅ | |
| [UT-SA-06](./specs/savings/savings_unit_tests.spec#L91) | `depositIncreasesShareSupply` | Deposit increases share totalSupply by minted shares<br>`supply' == supply + deposit(assets).shares` | ✅ | |
| [UT-SA-07](./specs/savings/savings_unit_tests.spec#L106) | `redeemDecreasesShareSupply` | Redeem decreases share totalSupply by burned shares<br>`supply' == supply - shares` | ✅ | |
| [UT-SA-08](./specs/savings/savings_unit_tests.spec#L125) | `depositRefreshesLastUpdate` | Deposit sets lastUpdate to block.timestamp<br>`lastUpdate' == block.timestamp after deposit` | ✅ | |
| [UT-SA-09](./specs/savings/savings_unit_tests.spec#L138) | `mintRefreshesLastUpdate` | Mint sets lastUpdate to block.timestamp<br>`lastUpdate' == block.timestamp after mint` | ✅ | |
| [UT-SA-10](./specs/savings/savings_unit_tests.spec#L151) | `withdrawRefreshesLastUpdate` | Withdraw sets lastUpdate to block.timestamp<br>`lastUpdate' == block.timestamp after withdraw` | ✅ | |
| [UT-SA-11](./specs/savings/savings_unit_tests.spec#L164) | `redeemRefreshesLastUpdate` | Redeem sets lastUpdate to block.timestamp<br>`lastUpdate' == block.timestamp after redeem` | ✅ | |
| [UT-SA-12](./specs/savings/savings_unit_tests.spec#L177) | `depositIncreasesVaultAssets` | Deposit does not decrease vault's TokenP balance<br>`bal[asset][vault]' >= bal[asset][vault] after deposit` | ✅ | |
| [UT-SA-13](./specs/savings/savings_unit_tests.spec#L196) | `togglePauseFlipsFlag` | togglePause flips the paused flag<br>`paused' == 1 - paused after togglePause` | ✅ | |
| [UT-SA-14](./specs/savings/savings_unit_tests.spec#L211) | `setRateRefreshesLastUpdate` | setRate sets lastUpdate to block.timestamp<br>`lastUpdate' == block.timestamp after setRate` | ✅ | |
| [UT-SA-15](./specs/savings/savings_unit_tests.spec#L224) | `setRateStoresNewValue` | setRate stores the new rate value<br>`rate' == newRate after setRate(newRate)` | ✅ | |
| [UT-SA-16](./specs/savings/savings_unit_tests.spec#L237) | `setMaxRateStoresNewValue` | setMaxRate stores the new maxRate value<br>`maxRate' == newMaxRate after setMaxRate(newMaxRate)` | ✅ | |
| [UT-SA-17](./specs/savings/savings_unit_tests.spec#L250) | `toggleTrustedFlipsFlag` | toggleTrusted flips the trusted flag<br>`isTrustedUpdater[addr]' == 1 - isTrustedUpdater[addr] after toggleTrusted(addr)` | ✅ | |
| [UT-SA-18](./specs/savings/savings_unit_tests.spec#L269) | `depositZeroGivesZeroShares` | deposit(0) returns 0 shares<br>`deposit(0).shares == 0` | ✅ | |
| [UT-SA-19](./specs/savings/savings_unit_tests.spec#L280) | `redeemZeroGivesZeroAssets` | redeem(0) returns 0 assets<br>`redeem(0).assets == 0` | ✅ | |
| [UT-SA-20](./specs/savings/savings_unit_tests.spec#L291) | `mintZeroGivesZeroAssets` | mint(0) returns 0 assets<br>`mint(0).assets == 0` | ✅ | |
| [UT-SA-21](./specs/savings/savings_unit_tests.spec#L302) | `withdrawZeroGivesZeroShares` | withdraw(0) returns 0 shares<br>`withdraw(0).shares == 0` | ✅ | |
| [UT-SA-22](./specs/savings/savings_unit_tests.spec#L317) | `depositRevertsWhenPaused` | Deposit reverts when paused<br>`paused == 1 => deposit REVERTS` | ✅ | |
| [UT-SA-23](./specs/savings/savings_unit_tests.spec#L329) | `mintRevertsWhenPaused` | Mint reverts when paused<br>`paused == 1 => mint REVERTS` | ✅ | |
| [UT-SA-24](./specs/savings/savings_unit_tests.spec#L341) | `withdrawRevertsWhenPaused` | Withdraw reverts when paused<br>`paused == 1 => withdraw REVERTS` | ✅ | |
| [UT-SA-25](./specs/savings/savings_unit_tests.spec#L353) | `redeemRevertsWhenPaused` | Redeem reverts when paused<br>`isRedemptionLive = 0 => redeem(amount, ...) REVERTS` | ✅ | |
| [UT-SA-26](./specs/savings/savings_unit_tests.spec#L365) | `setRateRejectsRateAboveMax` | setRate reverts when newRate > maxRate<br>`newRate > maxRate => setRate(newRate) REVERTS` | ✅ | |
| [UT-SA-27](./specs/savings/savings_unit_tests.spec#L381) | `depositDoesNotChangeRate` | Deposit does not change rate<br>`rate' == rate after deposit` | ✅ | |
| [UT-SA-28](./specs/savings/savings_unit_tests.spec#L396) | `depositDoesNotChangePaused` | Deposit does not change paused<br>`paused' == paused after deposit` | ✅ | |
| [UT-SA-29](./specs/savings/savings_unit_tests.spec#L411) | `setRateDoesNotChangePaused` | setRate does not change paused<br>`paused' == paused after setRate` | ✅ | |
| [UT-SA-30](./specs/savings/savings_unit_tests.spec#L426) | `togglePauseDoesNotChangeRate` | togglePause does not change rate<br>`rate' == rate after togglePause` | ✅ | |
| [UT-SA-31](./specs/savings/savings_unit_tests.spec#L441) | `togglePauseDoesNotChangeLastUpdate` | togglePause does not change lastUpdate<br>`lastUpdate' == lastUpdate after togglePause` | ✅ | |
| [UT-SA-32](./specs/savings/savings_unit_tests.spec#L460) | `togglePauseDoesNotChangeTotalSupply` | togglePause does not affect share totalSupply<br>`supply' == supply after togglePause` | ✅ | |
| [UT-SA-33](./specs/savings/savings_unit_tests.spec#L475) | `setRateDoesNotChangeTotalSupply` | setRate does not change share totalSupply<br>`supply' == supply after setRate` | ✅ | |
| [UT-SA-34](./specs/savings/savings_unit_tests.spec#L490) | `setMaxRateDoesNotChangeRate` | setMaxRate does not change rate<br>`rate' == rate after setMaxRate` | ✅ | |
| [UT-SA-35](./specs/savings/savings_unit_tests.spec#L509) | `depositDoesNotAffectThirdPartyShares` | Deposit does not change third party's share balance<br>`third != receiver => bal[shares][third]' == bal[shares][third] after deposit` | ✅ | |
| [UT-SA-36](./specs/savings/savings_unit_tests.spec#L525) | `redeemDoesNotAffectThirdPartyShares` | Redeem does not change third party's share balance<br>`third != owner => bal[shares][third]' == bal[shares][third] after redeem` | ✅ | |
| [UT-SA-37](./specs/savings/savings_unit_tests.spec#L541) | `depositDoesNotAffectThirdPartyAssets` | Deposit does not change third party's TokenP balance<br>`third != sender AND third != vault => bal[asset][third]' == bal[asset][third] after deposit` | ✅ | |
| [UT-SA-38](./specs/savings/savings_unit_tests.spec#L557) | `withdrawDoesNotAffectThirdPartyAssets` | Withdraw does not change third party's TokenP balance<br>`third != receiver AND third != vault => bal[asset][third]' == bal[asset][third] after withdraw` | ✅ | |
| [UT-SA-39](./specs/savings/savings_unit_tests.spec#L577) | `toggleTrustedDoesNotAffectOtherAddresses` | Does not change other addresses' trusted status<br>`other != addr => isTrustedUpdater[other]' == isTrustedUpdater[other] after toggleTrusted(addr)` | ✅ | |
| [UT-SA-40](./specs/savings/savings_unit_tests.spec#L597) | `estimatedAPRSameForAnyCaller` | estimatedAPR returns same result regardless of caller<br>`sender1 != sender2 => estimatedAPR(e1) == estimatedAPR(e2)` | ✅ | |
| [UT-SA-41](./specs/savings/savings_unit_tests.spec#L613) | `totalAssetsSameForAnyCaller` | totalAssets returns same result regardless of caller<br>`sender1 != sender2 => totalAssets(e1) == totalAssets(e2)` | ✅ | |
| [UT-SA-42](./specs/savings/savings_unit_tests.spec#L633) | `estimatedAPRNeverReverts` | estimatedAPR never reverts under valid state<br>`!REVERTS(estimatedAPR())` | ✅ | |
| [UT-SA-43](./specs/savings/savings_unit_tests.spec#L648) | `depositIsReachable` | Deposit can succeed with positive shares<br>`satisfy(deposit(assets).shares > 0)` | ✅ | |
| [UT-SA-44](./specs/savings/savings_unit_tests.spec#L658) | `withdrawIsReachable` | Withdraw can succeed with positive shares burned<br>`satisfy(withdraw(assets).shares > 0)` | ✅ | |
| [UT-SA-45](./specs/savings/savings_unit_tests.spec#L668) | `mintIsReachable` | Mint can succeed with positive assets paid<br>`satisfy(mint(shares).assets > 0)` | ✅ | |
| [UT-SA-46](./specs/savings/savings_unit_tests.spec#L678) | `redeemIsReachable` | Redeem can succeed with positive assets returned<br>`satisfy(redeem(amount, ...) AND bal[tokenP][sender] - bal'[tokenP][sender] > 0)` | ✅ | |

#### FlashParallelToken

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-FPT-01](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L10) | `setFeeRecipientStoresNewValue` | Correctly stores the new recipient<br>`feeRecipient' == newRecipient after setFlashLoanFeeRecipient(newRecipient)` | ✅ | |
| [UT-FPT-02](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L24) | `setParametersStoresAllFields` | Correctly stores fee rate, max borrowable, and active flag<br>`feesRate[t]' == feesRate AND maxBorrowable[t]' == maxBorrowable AND isActive[t]' == isActive after setFlashLoanParameters(t, feesRate, maxBorrowable, isActive)` | ✅ | |
| [UT-FPT-03](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L43) | `toggleFlipsActiveFlag` | Flips the isActive flag<br>`isActive[t]' != isActive[t] after toggleActiveToken(t)` | ✅ | |
| [UT-FPT-04](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L58) | `maxFlashLoanReturnsCorrectLimit` | Returns maxBorrowable when active, zero when inactive<br>`maxFlashLoan(t) == (isActive[t] ? maxBorrowable[t] : 0)` | ✅ | |
| [UT-FPT-05](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L74) | `flashFeeReturnsCorrectAmount` | Returns amount * feeRate with half-up rounding<br>`flashFee(t, amount) == (amount * feesRate[t] + 5000) / 10000` | ✅ | |
| [UT-FPT-06](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L92) | `setFeeRecipientRejectsZeroAddress` | Reverts on zero address<br>`newRecipient == 0 => REVERTS after setFlashLoanFeeRecipient(newRecipient)` | ✅ | |
| [UT-FPT-07](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L104) | `setParametersRejectsFeeRateAboveMax` | Reverts when fee rate exceeds 100%<br>`feesRate > PERCENTAGE_FACTOR => REVERTS after setFlashLoanParameters(t, feesRate, ...)` | ✅ | |
| [UT-FPT-08](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L116) | `flashFeeRejectsInactiveToken` | Reverts for inactive tokens<br>`!isActive[t] => REVERTS after flashFee(t, amount)` | ✅ | |
| [UT-FPT-09](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L130) | `flashLoanRejectsInactiveToken` | Reverts for inactive tokens<br>`!isActive[t] => REVERTS after flashLoan(receiver, t, amount, data)` | ✅ | |
| [UT-FPT-10](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L144) | `flashLoanRejectsAmountAboveMax` | Reverts when amount exceeds max borrowable<br>`amount > maxBorrowable[t] => REVERTS after flashLoan(receiver, t, amount, data)` | ✅ | |
| [UT-FPT-11](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L162) | `setFeeRecipientDoesNotChangeTokenSettings` | Does not change any token settings<br>`config[t]' == config[t] after setFlashLoanFeeRecipient(newRecipient)` | ✅ | |
| [UT-FPT-12](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L185) | `setParametersDoesNotChangeFeeRecipient` | Does not change the fee recipient<br>`feeRecipient' == feeRecipient after setFlashLoanParameters(t, feesRate, maxBorrowable, isActive)` | ✅ | |
| [UT-FPT-13](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L202) | `setParametersDoesNotAffectOtherTokens` | Does not affect other tokens<br>`forall o != t: config[o]' == config[o] after setFlashLoanParameters(t, ...)` | ✅ | |
| [UT-FPT-14](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L228) | `toggleDoesNotChangeFeeRecipient` | Does not change the fee recipient<br>`feeRecipient' == feeRecipient after toggleActiveToken(t)` | ✅ | |
| [UT-FPT-15](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L243) | `toggleDoesNotChangeFeeRateOrMaxBorrowable` | Does not change fee rate or max borrowable<br>`feesRate[t]' == feesRate[t] AND maxBorrowable[t]' == maxBorrowable[t] after toggleActiveToken(t)` | ✅ | |
| [UT-FPT-16](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L262) | `toggleDoesNotAffectOtherTokens` | Does not affect other tokens<br>`forall o != t: config[o]' == config[o] after toggleActiveToken(t)` | ✅ | |
| [UT-FPT-17](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L286) | `flashLoanDoesNotChangeAnyConfig` | Does not modify fee recipient or token settings<br>`feeRecipient' == feeRecipient AND config[t]' == config[t] after flashLoan(receiver, t, amount, data)` | ✅ | |
| [UT-FPT-18](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L317) | `maxFlashLoanSameForAnyCaller` | Returns same result regardless of caller<br>`forall e1, e2: maxFlashLoan(e1, t) == maxFlashLoan(e2, t) at same state` | ✅ | |
| [UT-FPT-19](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L333) | `flashFeeSameForAnyCaller` | Returns same result regardless of caller<br>`forall e1, e2: flashFee(e1, t, amount) == flashFee(e2, t, amount) at same state` | ✅ | |
| [UT-FPT-20](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L353) | `maxFlashLoanNeverReverts` | Never reverts<br>`!REVERTS after maxFlashLoan(t)` | ✅ | |
| [UT-FPT-21](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L367) | `setFeeRecipientIsReachable` | Can succeed<br>`satisfy(feeRecipient' == newRecipient)` | ✅ | |
| [UT-FPT-22](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L378) | `setParametersIsReachable` | Can succeed with non-trivial params<br>`satisfy(feesRate > 0 AND maxBorrowable > 0 AND isActive)` | ✅ | |
| [UT-FPT-23](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L389) | `toggleActiveTokenIsReachable` | Can succeed and activate a token<br>`satisfy(isActive[t]' == true)` | ✅ | |
| [UT-FPT-24](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L400) | `flashLoanIsReachable` | Can succeed with non-zero borrow<br>`satisfy(amount > 0) after flashLoan(receiver, t, amount, data)` | ✅ | |
| [UT-FPT-25](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L411) | `flashFeeIsReachable` | Can return a non-zero fee<br>`satisfy(flashFee(t, amount) > 0)` | ✅ | |
| [UT-FPT-26](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L427) | `accrueInterestTransfersExactBalance` | Fee recipient receives exactly the contract's balance<br>`bal[t][contract]' == 0 AND bal[t][feeRecipient]' == bal[t][feeRecipient] + bal[t][contract] after accrueInterest(t)` | ✅ | |
| [UT-FPT-27](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L449) | `accrueInterestDoesNotChangeConfig` | Does not change any config<br>`feeRecipient' == feeRecipient AND config[t]' == config[t] after accrueInterest(token)` | ✅ | |
| [UT-FPT-28](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L471) | `accrueInterestDoesNotChangeTotalSupply` | Does not change total supply<br>`supply[t]' == supply[t] after accrueInterest(t)` | ✅ | |
| [UT-FPT-29](./specs/flashParallelToken/flashParallelToken_unit_tests.spec#L484) | `accrueInterestIsReachable` | Can succeed with non-zero balance<br>`satisfy(bal[t][contract] > 0) after accrueInterest(t)` | ✅ | |

### ERC-4626 Conformance

Properties verifying conformance to the ERC-4626 tokenized vault standard (Savings).

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [ERC4626-01](./specs/savings/savings_erc4626.spec#L10) | `assetReturnsConfiguredToken` | asset() returns the configured ERC-20 token<br>`asset() == configuredAsset` | ✅ | |
| [ERC4626-02](./specs/savings/savings_erc4626.spec#L18) | `assetNeverReverts` | asset() never reverts<br>`!REVERTS(asset())` | ✅ | |
| [ERC4626-03](./specs/savings/savings_erc4626.spec#L26) | `totalAssetsIncludesAccruedInterest` | totalAssets ≥ vault balance (includes accrued interest)<br>`totalAssets() >= bal[asset][vault]` | ✅ | |
| [ERC4626-04](./specs/savings/savings_erc4626.spec#L38) | `totalAssetsNeverReverts` | totalAssets() never reverts<br>`!REVERTS(totalAssets())` | ✅ | |
| [ERC4626-05](./specs/savings/savings_erc4626.spec#L50) | `convertToSharesSameForAnyCaller` | Caller-agnostic<br>`convertToShares(e1, assets) == convertToShares(e2, assets) for any e1, e2` | ✅ | |
| [ERC4626-06](./specs/savings/savings_erc4626.spec#L64) | `convertToSharesNeverReverts` | Never reverts with reasonable input<br>`assets < max_uint96 AND supply > 0 => !REVERTS(convertToShares(assets))` | ✅ | |
| [ERC4626-07](./specs/savings/savings_erc4626.spec#L75) | `convertToSharesRoundsDown` | Rounds toward zero<br>`convertToAssets(convertToShares(assets)) <= assets` | ✅ | |
| [ERC4626-08](./specs/savings/savings_erc4626.spec#L89) | `convertToAssetsSameForAnyCaller` | Caller-agnostic<br>`convertToAssets(e1, shares) == convertToAssets(e2, shares) for any e1, e2` | ✅ | |
| [ERC4626-09](./specs/savings/savings_erc4626.spec#L103) | `convertToAssetsNeverReverts` | Never reverts with reasonable input<br>`shares < max_uint96 => !REVERTS(convertToAssets(shares))` | ✅ | |
| [ERC4626-10](./specs/savings/savings_erc4626.spec#L113) | `convertToAssetsRoundsDown` | Rounds toward zero<br>`convertToShares(convertToAssets(shares)) <= shares` | ✅ | |
| [ERC4626-11](./specs/savings/savings_erc4626.spec#L130) | `maxDepositNeverReverts` | maxDeposit() never reverts<br>`!REVERTS(maxDeposit(receiver))` | ✅ | |
| [ERC4626-12](./specs/savings/savings_erc4626.spec#L138) | `maxMintNeverReverts` | maxMint() never reverts<br>`!REVERTS(maxMint(receiver))` | ✅ | |
| [ERC4626-13](./specs/savings/savings_erc4626.spec#L146) | `depositAboveMaxReverts` | Deposit above maxDeposit limit reverts<br>`maxDeposit != max_uint256 AND assets > maxDeposit => deposit REVERTS` | ✅ | |
| [ERC4626-14](./specs/savings/savings_erc4626.spec#L161) | `mintAboveMaxReverts` | Mint above maxMint limit reverts<br>`maxMint != max_uint256 AND shares > maxMint => mint REVERTS` | ✅ | |
| [ERC4626-15](./specs/savings/savings_erc4626.spec#L176) | `zeroMaxMintBlocksMinting` | Zero maxMint disables minting<br>`maxMint == 0 AND shares > 0 => mint REVERTS` | ✅ | |
| [ERC4626-16](./specs/savings/savings_erc4626.spec#L191) | `withdrawAboveMaxReverts` | Withdraw above maxWithdraw limit reverts<br>`assets > maxWithdraw(owner) => withdraw REVERTS` | ✅ | |
| [ERC4626-17](./specs/savings/savings_erc4626.spec#L206) | `maxWithdrawNeverReverts` | maxWithdraw() never reverts<br>`!REVERTS(maxWithdraw(owner))` | ✅ | |
| [ERC4626-18](./specs/savings/savings_erc4626.spec#L214) | `maxRedeemNeverReverts` | maxRedeem() never reverts<br>`!REVERTS(maxRedeem(owner))` | ✅ | |
| [ERC4626-19](./specs/savings/savings_erc4626.spec#L222) | `redeemAboveMaxReverts` | Redeem above maxRedeem limit reverts<br>`shares > maxRedeem(owner) => redeem REVERTS` | ✅ | |
| [ERC4626-20](./specs/savings/savings_erc4626.spec#L237) | `zeroMaxRedeemBlocksRedemption` | Zero maxRedeem disables redemption<br>`maxRedeem == 0 AND shares > 0 => redeem REVERTS` | ✅ | |
| [ERC4626-21](./specs/savings/savings_erc4626.spec#L258) | `previewDepositNeverOverestimates` | Actual shares ≥ previewed shares<br>`deposit(assets).shares >= previewDeposit(assets)` | ✅ | |
| [ERC4626-22](./specs/savings/savings_erc4626.spec#L271) | `previewMintNeverUnderestimates` | Actual assets ≤ previewed assets<br>`mint(shares).assets <= previewMint(shares)` | ✅ | |
| [ERC4626-23](./specs/savings/savings_erc4626.spec#L284) | `previewWithdrawNeverUnderestimates` | Actual shares ≤ previewed shares<br>`withdraw(assets).shares <= previewWithdraw(assets)` | ✅ | |
| [ERC4626-24](./specs/savings/savings_erc4626.spec#L297) | `previewRedeemNeverOverestimates` | Actual assets ≥ previewed assets<br>`redeem(shares).assets >= previewRedeem(shares)` | ✅ | |
| [ERC4626-25](./specs/savings/savings_erc4626.spec#L312) | `previewDepositIncludesFees` | Fee-inclusive (shares ≤ ideal)<br>`previewDeposit(assets) <= convertToShares(assets)` | ✅ | |
| [ERC4626-26](./specs/savings/savings_erc4626.spec#L325) | `previewMintIncludesFees` | Fee-inclusive (assets ≥ ideal)<br>`previewMint(shares) >= convertToAssets(shares)` | ✅ | |
| [ERC4626-27](./specs/savings/savings_erc4626.spec#L338) | `previewWithdrawIncludesFees` | Fee-inclusive (shares ≥ ideal)<br>`previewWithdraw(assets) >= convertToShares(assets)` | ✅ | |
| [ERC4626-28](./specs/savings/savings_erc4626.spec#L351) | `previewRedeemIncludesFees` | Fee-inclusive (assets ≤ ideal)<br>`previewRedeem(shares) <= convertToAssets(shares)` | ✅ | |
| [ERC4626-29](./specs/savings/savings_erc4626.spec#L366) | `previewDepositSameForAnyCaller` | Caller-agnostic<br>`previewDeposit(e1, assets) == previewDeposit(e2, assets) for any e1, e2` | ✅ | |
| [ERC4626-30](./specs/savings/savings_erc4626.spec#L380) | `previewMintSameForAnyCaller` | Caller-agnostic<br>`previewMint(e1, shares) == previewMint(e2, shares) for any e1, e2` | ✅ | |
| [ERC4626-31](./specs/savings/savings_erc4626.spec#L394) | `previewWithdrawSameForAnyCaller` | Caller-agnostic<br>`previewWithdraw(e1, assets) == previewWithdraw(e2, assets) for any e1, e2` | ✅ | |
| [ERC4626-32](./specs/savings/savings_erc4626.spec#L408) | `previewRedeemSameForAnyCaller` | Caller-agnostic<br>`previewRedeem(e1, shares) == previewRedeem(e2, shares) for any e1, e2` | ✅ | |
| [ERC4626-33](./specs/savings/savings_erc4626.spec#L424) | `previewDepositRevertsOnlyIfDepositReverts` | Revert correlation<br>`REVERTS(previewDeposit(assets)) => REVERTS(deposit(assets))` | ✅ | |
| [ERC4626-34](./specs/savings/savings_erc4626.spec#L441) | `previewMintRevertsOnlyIfMintReverts` | Revert correlation<br>`REVERTS(previewMint(shares)) => REVERTS(mint(shares))` | ✅ | |
| [ERC4626-35](./specs/savings/savings_erc4626.spec#L458) | `previewWithdrawRevertsOnlyIfWithdrawReverts` | Revert correlation<br>`REVERTS(previewWithdraw(assets)) => REVERTS(withdraw(assets))` | ✅ | |
| [ERC4626-36](./specs/savings/savings_erc4626.spec#L475) | `previewRedeemRevertsOnlyIfRedeemReverts` | Revert correlation<br>`REVERTS(previewRedeem(shares)) => REVERTS(redeem(shares))` | ✅ | |
| [ERC4626-37](./specs/savings/savings_erc4626.spec#L497) | `depositTransfersAndMintsCorrectly` | Transfers exact assets, mints correct shares<br>`bal[asset][sender]' == bal[asset][sender] - assets AND bal[shares][receiver]' == bal[shares][receiver] + shares AND supply' == supply + shares` | ✅ | |
| [ERC4626-38](./specs/savings/savings_erc4626.spec#L519) | `depositRevertsIfAssetsNotFullyTransferred` | Reverts if assets not fully transferred<br>`bal[asset][sender]' != bal[asset][sender] - assets => deposit REVERTS` | ✅ | |
| [ERC4626-39](./specs/savings/savings_erc4626.spec#L535) | `depositRespectsAssetAllowance` | Respects ERC20 allowance<br>`allowance[asset][sender][vault] != max_uint256 AND allowance < assets => deposit REVERTS` | ✅ | |
| [ERC4626-40](./specs/savings/savings_erc4626.spec#L550) | `mintTransfersAndMintsCorrectly` | Transfers computed assets, mints exact shares<br>`bal[asset][sender]' == bal[asset][sender] - assets AND bal[shares][receiver]' == bal[shares][receiver] + shares AND supply' == supply + shares` | ✅ | |
| [ERC4626-41](./specs/savings/savings_erc4626.spec#L572) | `mintRevertsIfSharesNotFullyMinted` | Reverts if shares not fully minted<br>`bal[shares][receiver]' != bal[shares][receiver] + shares => mint REVERTS` | ✅ | |
| [ERC4626-42](./specs/savings/savings_erc4626.spec#L590) | `mintDecreasesAllowanceByExactCost` | Decreases allowance by exact assets spent<br>`allowance[asset][sender][vault] - allowance[asset][sender][vault]' == mint(shares).assets` | ✅ | |
| [ERC4626-43](./specs/savings/savings_erc4626.spec#L613) | `withdrawBurnsAndTransfersCorrectly` | Burns correct shares, transfers exact assets<br>`bal[shares][owner]' == bal[shares][owner] - sharesBurned AND bal[asset][receiver]' == bal[asset][receiver] + assets AND supply' == supply - sharesBurned` | ✅ | |
| [ERC4626-44](./specs/savings/savings_erc4626.spec#L634) | `withdrawRevertsIfAssetsNotFullyReceived` | Reverts if assets not fully received<br>`receiver != vault AND bal[asset][receiver]' != bal[asset][receiver] + assets => withdraw REVERTS` | ✅ | |
| [ERC4626-45](./specs/savings/savings_erc4626.spec#L652) | `redeemBurnsAndTransfersCorrectly` | Burns exact shares, transfers computed assets<br>`bal[shares][owner]' == bal[shares][owner] - shares AND bal[asset][receiver]' == bal[asset][receiver] + assetsOut AND supply' == supply - shares` | ✅ | |
| [ERC4626-46](./specs/savings/savings_erc4626.spec#L675) | `redeemRevertsIfSharesNotFullyBurned` | Reverts if shares not fully burned<br>`bal[shares][owner]' != bal[shares][owner] - shares => redeem REVERTS` | ✅ | |
| [ERC4626-47](./specs/savings/savings_erc4626.spec#L696) | `withdrawRespectsShareAllowance` | Respects share allowance<br>`sender != owner AND allowance[shares][owner][sender] != max_uint256 AND allowance < sharesBurned => withdraw REVERTS` | ✅ | |
| [ERC4626-48](./specs/savings/savings_erc4626.spec#L716) | `redeemRespectsShareAllowance` | Respects share allowance<br>`sender != owner AND allowance[shares][owner][sender] != max_uint256 AND allowance < shares => redeem REVERTS` | ✅ | |
| [ERC4626-49](./specs/savings/savings_erc4626.spec#L733) | `withdrawDecreasesAllowanceByBurnedShares` | Decreases allowance by burned shares<br>`allowance[shares][owner][sender] - allowance[shares][owner][sender]' == withdraw(assets).sharesBurned` | ✅ | |
| [ERC4626-50](./specs/savings/savings_erc4626.spec#L752) | `redeemDecreasesAllowanceByRedeemedShares` | Decreases allowance by redeemed shares<br>`allowance[shares][owner][sender] - allowance[shares][owner][sender]' == shares` | ✅ | |

<div style="page-break-before: always;"></div>

---

## Real Issues Found

Four real bugs were identified by the audit team during the security engagement. For some of these bugs, the formal verification suite was able to confirm the fixes by verifying that previously-violated rules now pass.

<a id="real-issue-1"></a>

### 1. [HIGH] `Surplus::processSurplus` always reverts for managed collateral [#5](https://github.com/Cyfrin/audit-2026-02-parallel_3.1/issues/5)

`LibSurplus::_computeCollateralSurplus` reads the collateral balance from `LibManager::totalAssets` for managed collateral, which returns the balance held by the external strategy. It then computes a `collateralSurplus`. However, `Surplus::processSurplus` self-calls `swapExactInput(collateralSurplus, ...)`, and since this is an external self-call, `msg.sender` is the diamond — which holds zero balance for managed collateral. The transfer always reverts.

This is a permanent DoS on surplus processing for all managed collateral. Strategy yield cannot be captured as distributable TokenP.

❌ Violated:

```cvl
// HL-S-10: processSurplus reachable for managed collateral
// FORMULA: satisfy(isManaged[collateral] > 0 AND collateralSurplus > 0 AND issuedAmount > 0)
rule processSurplusManagedCollateralIsReachable(
    env e,
    address collateral,
    uint256 maxCollateralAmount
) {
    setupValidStateParallelizer(e);

    require(ghostCollateralIsManaged8[collateral] > 0,
        "SAFE: test managed collateral path");

    uint256 collateralSurplus;
    uint256 stableSurplus;
    uint256 issuedAmount;
    collateralSurplus, stableSurplus, issuedAmount = processSurplus(e, collateral, maxCollateralAmount);

    satisfy(collateralSurplus > 0 && issuedAmount > 0,
        "processSurplus must be reachable for managed collateral");
}
```

✅ Passed (after the fix): 

Before the self-swap, withdraw the surplus from the strategy:

```diff
+if (collatInfo.isManaged > 0) {
+    LibManager.release(collateral, address(this), collateralSurplus, collatInfo.managerData.config);
+}
```

<a id="real-issue-2"></a>

### 2. [MEDIUM] `LibSurplus::_computeCollateralSurplus` doesn't account for `surplusBufferRatio > 100%` [#29](https://github.com/Cyfrin/audit-2026-02-parallel_3.1/issues/29)

`_computeCollateralSurplus` treats as surplus everything above 100% collateral ratio, but then uses `surplusBufferRatio` for the final check. When `surplusBufferRatio > 100%`, `processSurplus()` always reverts because the surplus computed at 100% CR is less than what's needed to maintain the buffer ratio. The `surplusBufferRatio` parameter effectively has no meaningful purpose.

❌ Violated:

```cvl
// HL-S-09: processSurplus reachable when surplusBufferRatio > 100%
// FORMULA: satisfy(surplusBufferRatio > BASE_9 AND collateralSurplus > 0 AND issuedAmount > 0)
rule processSurplusReachableWithHighBufferRatio(
    env e,
    address collateral,
    uint256 maxCollateralAmount
) {
    setupValidStateParallelizer(e);

    require(ghostSurplusBufferRatio64 > BASE_9(),
        "SAFE: buffer ratio > 100%");

    uint256 collateralSurplus;
    uint256 stableSurplus;
    uint256 issuedAmount;
    collateralSurplus, stableSurplus, issuedAmount = processSurplus(e, collateral, maxCollateralAmount);

    satisfy(collateralSurplus > 0 && issuedAmount > 0,
        "processSurplus reachable with surplusBufferRatio > 100%");
}
```

✅ Passed (after the fix):

Use `surplusBufferRatio` to compute the maximum backable amount before checking for surplus:

```diff
-if (totalCollateralValue <= stablesBacked) revert ZeroSurplusAmount();
-stableSurplus = totalCollateralValue - stablesBacked;
+uint256 maxBackable = (totalCollateralValue * BASE_9) / ts.surplusBufferRatio;
+if (maxBackable <= stablesBacked) revert ZeroSurplusAmount();
+stableSurplus = maxBackable - stablesBacked;
```

<a id="real-issue-3"></a>

### 3. [LOW] `SettersGovernor::setWhitelistStatus` allows values other than 0 and 1 [#16](https://github.com/Cyfrin/audit-2026-02-parallel_3.1/issues/16)

The `toggleWhitelist` function ensures the `whitelistStatus` field stays boolean via `1 - current`. But `setWhitelistStatus` accepts arbitrary `uint8` values (2–255). If a governor passes any of these, it causes a DoS on any function that calls `LibWhitelist::checkWhitelist`.

❌ Violated:

```cvl
// VS-P-14: collateral onlyWhitelisted is boolean
// FORMULA: forall t: onlyWhitelisted[t] in {0, 1}
invariant collateralOnlyWhitelistedFlag(env e)
    forall address token.
        ghostCollateralOnlyWhitelisted8[token] == 0 || ghostCollateralOnlyWhitelisted8[token] == 1
filtered { f -> !EXCLUDED_FUNCTION(f) } { preserved with (env eFunc) { SETUP_P(e, eFunc); } }
```

✅ Passed (after the fix):

Reject non-boolean values at the entry point:

```diff
  function setWhitelistStatus(address collateral, uint8 whitelistStatus, bytes memory whitelistData) internal {
+   if (whitelistStatus > 1) revert InvalidWhitelistStatus();
    Collateral storage collatInfo = s.transmuterStorage().collaterals[collateral];
```

<a id="real-issue-4"></a>

### 4. [LOW] `collatInfo.stablecoinCap` hardcap can be bypassed via `SettersGovernor::adjustStablecoins` [#18](https://github.com/Cyfrin/audit-2026-02-parallel_3.1/issues/18)

The `stablecoinCap` is correctly enforced during normal user mint operations in the Swapper facet. However, `adjustStablecoins` can arbitrarily increase `normalizedStables` without checking against `stablecoinCap`, allowing the system to enter a state where a collateral backs more stablecoins than its configured limit.

❌ Violated:

```cvl
// ST-P-08: Per-collateral normalizedStables increase respects stablecoinCap
// FORMULA: forall f, t: collatNS[t]' > collatNS[t] AND cap[t] > 0
//          => collatNS[t]' * norm' / BASE_27 <= cap[t]
rule stablecoinCapEnforcedOnNsIncrease(env e, method f, calldataarg args, address token)
    filtered { f -> !EXCLUDED_FUNCTION(f) } {

    setupValidStateParallelizer(e);

    mathint collatNsBefore = ghostCollateralNormalizedStables216[token];
    mathint cap = ghostCollateralStablecoinCap256[token];

    f(e, args);

    mathint collatNsAfter = ghostCollateralNormalizedStables216[token];
    mathint normalizerAfter = ghostNormalizer128;

    assert(collatNsAfter > collatNsBefore && cap > 0
        => collatNsAfter * normalizerAfter / BASE_27() <= cap,
        "Per-collateral normalizedStables increase must respect stablecoinCap");
}
```

✅ Passed (after the fix):

Enforce the cap inside `adjustStablecoins` when increasing:

```diff
  function adjustStablecoins(address collateral, uint128 amount, bool increase) internal {
    ...
    if (increase) {
      collatInfo.normalizedStables = collatInfo.normalizedStables + uint216(normalizedAmount);
      ts.normalizedStables = ts.normalizedStables + normalizedAmount;
+     if (collatInfo.normalizedStables * ts.normalizer / BASE_27 > collatInfo.stablecoinCap) {
+       revert AboveCap();
+     }
    } else {...}
  }
```

<div style="page-break-before: always;"></div>

---

## Mutation Testing

Valid state invariants are powerful — a single well-crafted invariant can catch a wide range of bugs across multiple functions. To demonstrate this, we use `normalizedStablesSumConservation` (VS-P-15) as a case study, showing through mutation testing that this one invariant detects 18 distinct bugs across `Swapper._swap()`, `LibSetters.adjustStablecoins()`, and `Redeemer._updateNormalizer()`.

### What is Mutation Testing

Mutation testing validates the strength of a formal specification by injecting small syntactic changes (mutations) into the source code and verifying that the spec catches them. Each mutation represents a potential bug — a removed assignment, a flipped operator, an off-by-one error.

The [certoraMutate](https://docs.certora.com/en/latest/docs/gambit/mutation-verifier.html) tool automates this process: it applies each mutant to the source, runs the prover against the specified rule, and reports whether the invariant caught the introduced defect.

### VS-P-15: normalizedStablesSumConservation

The `normalizedStablesSumConservation` invariant asserts that the global `normalizedStables` always equals the sum of per-collateral `normalizedStables` values. This strict equality makes the invariant maximally sensitive to any accounting discrepancy in the protocol's stablecoin bookkeeping.

```cvl
// VS-P-15: global normalizedStables == sum of per-collateral normalizedStables
// FORMULA: NS == sum(collatNS[t] for t in collateralList)
invariant normalizedStablesSumConservation(env e)
    ghostNormalizedStables128
        == ghostCollateralNormalizedStables216[ghostCollateralList[0]]
        + ghostCollateralNormalizedStables216[ghostCollateralList[1]]
filtered { f -> !EXCLUDED_FUNCTION(f) } { preserved with (env eFunc) { SETUP_P(e, eFunc); } }
```

**Result: 18/18 mutations caught** — the invariant catches every injected bug across all three contracts.

#### Swapper._swap() mutations (8/8 caught)

```bash
certoraMutate certora/confs/Parallelizer/Swapper/valid_state_mutations_normalizedStablesSumConservation.conf
```

![Swapper mutation testing](./mutations/normalizedStablesSumConservation/Swapper/Swapper.png)

**[❌ #1](./mutations/normalizedStablesSumConservation/Swapper/1.sol)** Remove global NS update in mint.
```diff
  // mint path
- ts.normalizedStables = ts.normalizedStables + changeAmount;
+ // removed
```

**[❌ #2](./mutations/normalizedStablesSumConservation/Swapper/2.sol)** Remove per-collateral NS update in mint.
```diff
  // mint path
- collatInfo.normalizedStables = collatInfo.normalizedStables + uint216(changeAmount);
+ // removed
```

**[❌ #3](./mutations/normalizedStablesSumConservation/Swapper/3.sol)** Flip + to − for global NS in mint.
```diff
  // mint path
- ts.normalizedStables = ts.normalizedStables + changeAmount;
+ ts.normalizedStables = ts.normalizedStables - changeAmount;
```

**[❌ #4](./mutations/normalizedStablesSumConservation/Swapper/4.sol)** Flip + to − for per-collateral NS in mint.
```diff
  // mint path
- collatInfo.normalizedStables = collatInfo.normalizedStables + uint216(changeAmount);
+ collatInfo.normalizedStables = collatInfo.normalizedStables - uint216(changeAmount);
```

**[❌ #5](./mutations/normalizedStablesSumConservation/Swapper/5.sol)** Off-by-one for global NS in mint.
```diff
  // mint path
- ts.normalizedStables = ts.normalizedStables + changeAmount;
+ ts.normalizedStables = ts.normalizedStables + changeAmount + 1;
```

**[❌ #6](./mutations/normalizedStablesSumConservation/Swapper/6.sol)** Remove global NS update in burn.
```diff
  // burn path
- ts.normalizedStables = ts.normalizedStables - changeAmount;
+ // removed
```

**[❌ #7](./mutations/normalizedStablesSumConservation/Swapper/7.sol)** Remove per-collateral NS update in burn.
```diff
  // burn path
- collatInfo.normalizedStables = collatInfo.normalizedStables - uint216(changeAmount);
+ // removed
```

**[❌ #8](./mutations/normalizedStablesSumConservation/Swapper/8.sol)** Flip − to + for global NS in burn.
```diff
  // burn path
- ts.normalizedStables = ts.normalizedStables - changeAmount;
+ ts.normalizedStables = ts.normalizedStables + changeAmount;
```

![Swapper patch 8 detail](./mutations/normalizedStablesSumConservation/Swapper/Swapper_patch8.png)

#### LibSetters.adjustStablecoins() mutations (6/6 caught)

```bash
certoraMutate certora/confs/Parallelizer/Setter/valid_state_mutations_normalizedStablesSumConservation.conf
```

**[❌ #1](./mutations/normalizedStablesSumConservation/LibSetters/1.sol)** Remove global NS update in increase.
```diff
  if (increase) {
    collatInfo.normalizedStables = collatInfo.normalizedStables + uint216(normalizedAmount);
-   ts.normalizedStables = ts.normalizedStables + normalizedAmount;
+   // removed
```

**[❌ #2](./mutations/normalizedStablesSumConservation/LibSetters/2.sol)** Remove per-collateral NS update in increase.
```diff
  if (increase) {
-   collatInfo.normalizedStables = collatInfo.normalizedStables + uint216(normalizedAmount);
+   // removed
    ts.normalizedStables = ts.normalizedStables + normalizedAmount;
```

**[❌ #3](./mutations/normalizedStablesSumConservation/LibSetters/3.sol)** Flip + to − for global NS in increase.
```diff
  if (increase) {
    collatInfo.normalizedStables = collatInfo.normalizedStables + uint216(normalizedAmount);
-   ts.normalizedStables = ts.normalizedStables + normalizedAmount;
+   ts.normalizedStables = ts.normalizedStables - normalizedAmount;
```

**[❌ #4](./mutations/normalizedStablesSumConservation/LibSetters/4.sol)** Remove global NS update in decrease.
```diff
  } else {
    collatInfo.normalizedStables = collatInfo.normalizedStables - uint216(normalizedAmount);
-   ts.normalizedStables = ts.normalizedStables - normalizedAmount;
+   // removed
```

**[❌ #5](./mutations/normalizedStablesSumConservation/LibSetters/5.sol)** Remove per-collateral NS update in decrease.
```diff
  } else {
-   collatInfo.normalizedStables = collatInfo.normalizedStables - uint216(normalizedAmount);
+   // removed
    ts.normalizedStables = ts.normalizedStables - normalizedAmount;
```

**[❌ #6](./mutations/normalizedStablesSumConservation/LibSetters/6.sol)** Flip − to + for global NS in decrease.
```diff
  } else {
    collatInfo.normalizedStables = collatInfo.normalizedStables - uint216(normalizedAmount);
-   ts.normalizedStables = ts.normalizedStables - normalizedAmount;
+   ts.normalizedStables = ts.normalizedStables + normalizedAmount;
```

#### Redeemer._updateNormalizer() mutations (4/4 caught)

```bash
certoraMutate certora/confs/Parallelizer/Redeemer/valid_state_mutations_normalizedStablesSumConservation.conf
```

**[❌ #1](./mutations/normalizedStablesSumConservation/Redeemer/1.sol)** Remove global NS assignment after renormalization.
```diff
    }
-   ts.normalizedStables = newNormalizedStables;
+   // removed
```

**[❌ #2](./mutations/normalizedStablesSumConservation/Redeemer/2.sol)** Remove per-collateral NS update in renormalization loop.
```diff
    newNormalizedStables += newCollateralNormalizedStable;
-   ts.collaterals[collateralListMem[i]].normalizedStables = uint216(newCollateralNormalizedStable);
+   // removed
```

**[❌ #3](./mutations/normalizedStablesSumConservation/Redeemer/3.sol)** Remove accumulator update in renormalization loop.
```diff
-   newNormalizedStables += newCollateralNormalizedStable;
+   // removed
    ts.collaterals[collateralListMem[i]].normalizedStables = uint216(newCollateralNormalizedStable);
```

**[❌ #4](./mutations/normalizedStablesSumConservation/Redeemer/4.sol)** Double accumulator in renormalization.
```diff
-   newNormalizedStables += newCollateralNormalizedStable;
+   newNormalizedStables += 2 * newCollateralNormalizedStable;
```

<div style="page-break-before: always;"></div>

---

## Setup and Execution

The Certora Prover can be run either remotely (using Certora's cloud infrastructure) or locally (building from source). Both modes share the same initial setup steps.

### Common Setup (Steps 1–4)

The instructions below are for Ubuntu 24.04. For step-by-step installation details refer to this setup [tutorial](https://alexzoid.com/first-steps-with-certora-fv-catching-a-real-bug#heading-setup).

1. Install Java (tested with JDK 21)

```bash
sudo apt update
sudo apt install default-jre
java -version
```

2. Install [pipx](https://pipx.pypa.io/) — installs Python CLI tools in isolated environments, avoiding dependency conflicts

```bash
sudo apt install pipx
pipx ensurepath
```

3. Install Certora CLI. To match a specific prover version, pin it explicitly (e.g. `certora-cli==8.8.0`)

```bash
pipx install certora-cli
```

4. Install solc-select and the Solidity compiler version required by the project

```bash
pipx install solc-select
solc-select install 0.8.28
solc-select use 0.8.28
```

### Remote Execution

5. Set up Certora key. You can get a free key through the Certora [Discord](https://discord.gg/certora) or on their website. Once you have it, export it:

```bash
echo "export CERTORAKEY=<your_certora_api_key>" >> ~/.bashrc
```

> **Note:** If a local prover is installed (see below), it takes priority. To force remote execution, add the `--server production` flag:
> ```bash
> certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf --server production
> ```

### Local Execution

Follow the full build instructions in the [CertoraProver repository (v8.8.0)](https://github.com/Certora/CertoraProver/tree/8.8.0). Once the local prover is installed it takes priority over the remote cloud by default. Tested on Ubuntu 24.04.

1. Install prerequisites

```bash
# JDK 19+
sudo apt install openjdk-21-jdk

# SMT solvers (z3 and cvc5 are required, others are optional)
# Download binaries and place them in PATH:
#   z3:   https://github.com/Z3Prover/z3/releases
#   cvc5: https://github.com/cvc5/cvc5/releases

# LLVM tools
sudo apt install llvm

# Rust 1.81.0+
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
cargo install rustfilt

# Graphviz (optional, for visual reports)
sudo apt install graphviz
```

2. Set up build output directory

```bash
export CERTORA="$HOME/CertoraProver/target/installed/"
mkdir -p "$CERTORA"
export PATH="$CERTORA:$PATH"
```

3. Clone and build

```bash
git clone --recurse-submodules https://github.com/Certora/CertoraProver.git
cd CertoraProver
git checkout tags/8.8.0
./gradlew assemble
```

4. Verify installation with test example

```bash
certoraRun.py -h
cd Public/TestEVM/Counter
certoraRun counter.conf
```

### Required source code modifications

The Certora Prover does not fully support arrays whose elements pack into a single storage slot (e.g. `uint64[]` where two elements share one 256-bit word). The Parallel codebase stores fee-curve breakpoints as `uint64[]` and `int64[]` arrays, which triggers incorrect INDEX hook behavior during verification. To work around this limitation, all packed fee arrays were widened to `uint256[]` / `int256[]`, and explicit `int64(...)` downcasts were added where narrower types are required at use sites. All changes are marked with `// CERTORA:` comments and are semantically neutral — values stay within their original 64-bit range.

1. **Storage.sol** — storage type widening (lines 110–113, 128–131)
   - `Collateral.xFeeMint`: `uint64[]` → `uint256[]`
   - `Collateral.yFeeMint`: `int64[]` → `int256[]`
   - `Collateral.xFeeBurn`: `uint64[]` → `uint256[]`
   - `Collateral.yFeeBurn`: `int64[]` → `int256[]`
   - `ParallelizerStorage.xRedemptionCurve`: `uint64[]` → `uint256[]`
   - `ParallelizerStorage.yRedemptionCurve`: `int64[]` → `int256[]`

2. **LibHelpers.sol** — function signature widening + downcasts (lines 41, 80–84)
   - `findLowerBound` param `array`: `uint64[]` → `uint256[]`
   - `piecewiseLinear` params `xArray`/`yArray`: `uint64[]`/`int64[]` → `uint256[]`/`int256[]`
   - Three return-value downcasts added: `int64(yArray[...])` (lines 82, 83, 84)

3. **LibSetters.sol** — function signatures, events, local variables (lines 30, 34, 200, 216, 290, 330, 337)
   - Events `FeesSet`, `RedemptionCurveParamsSet`: params `uint64[]`/`int64[]` → `uint256[]`/`int256[]`
   - Functions `setFees`, `setRedemptionCurveParams`, `checkFees`: params `uint64[]`/`int64[]` → `uint256[]`/`int256[]`
   - Local variables `burnFees`, `mintFees` inside `checkFees`: `int64[]` → `int256[]`

4. **Swapper.sol** — downcasts at fee lookup (lines 335, 485)
   - Two `int64(...)` downcasts added on `collatInfo.yFeeMint[0]` / `collatInfo.yFeeBurn[0]` and `[n-1]` accesses

5. **Redeemer.sol** — local variable widening + downcast (lines 178, 182, 193)
   - `yRedemptionCurveMem`: `int64[]` → `int256[]`
   - `xRedemptionCurveMem`: `uint64[]` → `uint256[]`
   - Downcast `int64(yRedemptionCurveMem[...])` at penalty-factor calculation

6. **Getters.sol** — return type widening (lines 52, 62, 72)
   - `getCollateralMintFees` returns: `uint64[]`/`int64[]` → `uint256[]`/`int256[]`
   - `getCollateralBurnFees` returns: `uint64[]`/`int64[]` → `uint256[]`/`int256[]`
   - `getRedemptionFees` returns: `uint64[]`/`int64[]` → `uint256[]`/`int256[]`

7. **IGetters.sol** — interface return type widening (lines 33, 37, 41)
   - Same three getters as above

8. **ISetters.sol** — interface parameter widening (lines 85, 88)
   - `setFees`, `setRedemptionCurveParams`: params `uint64[]`/`int64[]` → `uint256[]`/`int256[]`

9. **SettersGuardian.sol** — parameter widening (lines 23, 28)
   - `setFees`, `setRedemptionCurveParams`: params `uint64[]`/`int64[]` → `uint256[]`/`int256[]`

### Running Verification

Build the project first:

```bash
cd Parallel-Tokens && npm install --legacy-peer-deps && forge build && cd ..
cd Parallel-Parallelizer && npm install --legacy-peer-deps && forge build && cd ..
```

#### Quick Runs

`variable_transitions` and `unit_tests` are lightweight and can be run as full conf files:

```bash
certoraRun certora/confs/FlashParallelToken/variable_transitions.conf
certoraRun certora/confs/FlashParallelToken/unit_tests.conf
certoraRun certora/confs/Parallelizer/Swapper/variable_transitions.conf
certoraRun certora/confs/Parallelizer/Swapper/unit_tests.conf
certoraRun certora/confs/Parallelizer/Redeemer/variable_transitions.conf
certoraRun certora/confs/Parallelizer/Redeemer/unit_tests.conf
certoraRun certora/confs/Parallelizer/Setter/variable_transitions.conf
certoraRun certora/confs/Parallelizer/Setter/unit_tests.conf
certoraRun certora/confs/Parallelizer/Surplus/variable_transitions.conf
certoraRun certora/confs/Parallelizer/Surplus/unit_tests.conf
certoraRun certora/confs/Savings/variable_transitions.conf
certoraRun certora/confs/Savings/unit_tests.conf
certoraRun certora/confs/Savings/erc4626.conf
```

> **Note:** `valid_state`, `state_transitions`, and `high_level` may time out when running all rules at once. Run them per-rule using [`--rule`](https://docs.certora.com/en/latest/docs/prover/cli/options.html#rule) as shown below.

#### FlashParallelToken

```bash
# Valid State (1 invariant)
certoraRun certora/confs/FlashParallelToken/valid_state.conf \
  --rule feeRateNeverExceedsMax

# State Transitions (4 rules)
certoraRun certora/confs/FlashParallelToken/state_transitions.conf \
  --rule tokensOnlyWithdrawnToFeeRecipient
certoraRun certora/confs/FlashParallelToken/state_transitions.conf \
  --rule feeRecipientAndTokenSettingsChangeSeparately
certoraRun certora/confs/FlashParallelToken/state_transitions.conf \
  --rule tokenSettingsDoNotAffectOtherTokens
certoraRun certora/confs/FlashParallelToken/state_transitions.conf \
  --rule configChangesDoNotMoveTokens

# High-Level (4 rules)
certoraRun certora/confs/FlashParallelToken/high_level.conf \
  --rule flashLoanDoesNotChangeTotalSupply
certoraRun certora/confs/FlashParallelToken/high_level.conf \
  --rule flashLoanCollectsExactFee
certoraRun certora/confs/FlashParallelToken/high_level.conf \
  --rule borrowerCannotProfitFromFlashLoan
certoraRun certora/confs/FlashParallelToken/high_level.conf \
  --rule flashLoanFeeNeverExceedsBorrowedAmount
```

#### Parallelizer — Valid State (32 invariants, verified across all 4 facets)

```bash
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralListNoZeroValues
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralListNoDuplicates
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralListDecimalsNonZero
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule normalizerBounded
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule slippageToleranceBounded
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule surplusBufferRatioBounded
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule lastReleasedAtBounded
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule isRedemptionLiveFlag
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule isTrustedFlag
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule isSellerTrustedFlag
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralIsManagedFlag
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralIsMintLiveFlag
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralIsBurnLiveFlag
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralOnlyWhitelistedFlag
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule normalizedStablesSumConservation
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule feeMintLengthMatching
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule feeBurnLengthMatching
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule redemptionCurveLengthMatching
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule feeMintXOrdering
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule feeBurnXOrdering
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule totalSharesSumConservation
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule payeesNoDuplicates
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule payeeSharesPositive
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralDecimalsMatchErc20
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule collateralDataZeroOutsideList
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule sharesZeroOutsidePayees
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule feeMintYBounded
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule feeBurnYBounded
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule redemptionCurveYBounded
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule feeMintXAnchor
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule feeBurnXAnchor
certoraRun certora/confs/Parallelizer/Swapper/valid_state.conf \
  --rule redemptionCurveXBounded
```

> **Note:** The same 32 invariants are verified across all four facets. Replace `Swapper` with `Redeemer`, `Setter`, or `Surplus` to run against other facets.

#### Parallelizer — State Transitions (13 rules, verified across all 4 facets)

```bash
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule renormalizationResetsNormalizerToBase27
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule perCollateralNsChangeImpliesGlobalNsChange
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule globalNsChangeImpliesSomeCollatNsChange
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule multipleCollateralNsChangesImplyRenormalization
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule collateralDecimalsAndListAtomicity
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule collateralListGrowthPreservesTokenPSupply
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule tokenPSupplyChangeImpliesProtocolStateChange
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule stablecoinCapEnforcedOnNsIncrease
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule tokenPSupplyIncreaseImpliesNsIncrease
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule diamondTokenPBalanceChangeImpliesSurplusOrRecoveryPath
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule accountingChangePreservesConfiguration
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule lastReleasedAtOnlyUpdatesToTimestamp
certoraRun certora/confs/Parallelizer/Swapper/state_transitions.conf \
  --rule collateralListShrinkImpliesZeroNs
```

> **Note:** Replace `Swapper` with `Redeemer`, `Setter`, or `Surplus` to run against other facets.

#### Parallelizer — Swapper High-Level (19 rules)

```bash
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule mintExactInputSupplyDelta
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule burnExactInputSupplyDelta
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule mintExactOutputSupplyDelta
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule burnExactOutputSupplyDelta
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule mintCollateralTransfer
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule burnCollateralTransfer
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule swapOnlyAffectsTargetCollateral
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule zeroAmountInNoOp
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule mintExactInputIncreasesNs
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule burnExactInputDecreasesNs
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule mintExactOutputIncreasesNs
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule burnExactOutputDecreasesNs
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule swapExactInputSlippageGuarantee
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule swapExactOutputSlippageGuarantee
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule zeroAmountOutExactOutputNoOp
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule quoteInMatchesMintExactInput
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule quoteInMatchesBurnExactInput
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule quoteInSameForAnyCaller
certoraRun certora/confs/Parallelizer/Swapper/high_level.conf \
  --rule quoteOutSameForAnyCaller
```

#### Parallelizer — Redeemer High-Level (10 rules)

```bash
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule updateNormalizerIncreaseGrowsStablecoinsIssued
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule updateNormalizerDecreaseShrinksStablecoinsIssued
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule updateNormalizerIncreaseRaisesNormalizer
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule updateNormalizerDecreaseLowersNormalizer
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule redeemDecreasesStablecoinsIssued
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule largerRedeemDecreasesStablecoinsIssuedMore
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule largerUpdateNormalizerIncreasesMore
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule updateNormalizerIncreaseBoundedByAmount
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule updateNormalizerDecreaseBoundedByAmount
certoraRun certora/confs/Parallelizer/Redeemer/high_level.conf \
  --rule redeemWithoutRenormPreservesNormalizedStables
```

#### Parallelizer — Setter High-Level (9 rules)

```bash
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule togglePauseMintRoundTrip
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule togglePauseBurnRoundTrip
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule togglePauseRedeemRoundTrip
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule toggleTrustedUpdaterRoundTrip
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule toggleTrustedSellerRoundTrip
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule adjustStablecoinsRoundTrip
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule adjustStablecoinsIncreaseMonotonicity
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule adjustStablecoinsDecreaseMonotonicity
certoraRun certora/confs/Parallelizer/Setter/high_level.conf \
  --rule adjustStablecoinsGlobalLocalDeltaConservation
```

#### Parallelizer — Surplus High-Level (10 rules)

```bash
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule processSurplusIncreasesTokenPSupply
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule processSurplusCreditsTokenPToDiamond
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule releaseDistributesAllTokenP
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule processSurplusIncreasesNormalizedStables
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule processSurplusIncreasesPerCollateralNs
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule processSurplusPositiveIssuedImpliesNsIncrease
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule surplusLifecycleProcessThenRelease
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule processSurplusCapsToMaxCollateral
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule processSurplusReachableWithHighBufferRatio
certoraRun certora/confs/Parallelizer/Surplus/high_level.conf \
  --rule processSurplusManagedCollateralIsReachable
```

#### Savings

```bash
# Valid State (4 invariants)
certoraRun certora/confs/Savings/valid_state.conf \
  --rule pausedFlagIsZeroOrOne
certoraRun certora/confs/Savings/valid_state.conf \
  --rule trustedUpdaterFlagIsZeroOrOne
certoraRun certora/confs/Savings/valid_state.conf \
  --rule lastUpdateNeverExceedsBlockTime
certoraRun certora/confs/Savings/valid_state.conf \
  --rule vaultNeverApprovesOwnShares

# State Transitions (3 rules)
certoraRun certora/confs/Savings/state_transitions.conf \
  --rule stateChangeAlwaysRefreshesTimestamp
certoraRun certora/confs/Savings/state_transitions.conf \
  --rule pausedContractCannotChangeShareSupply
certoraRun certora/confs/Savings/state_transitions.conf \
  --rule rateCannotIncreaseAboveMaxRate

# High-Level (15 rules)
certoraRun certora/confs/Savings/high_level.conf \
  --rule depositThenRedeemCannotProfit
certoraRun certora/confs/Savings/high_level.conf \
  --rule mintThenRedeemCannotProfit
certoraRun certora/confs/Savings/high_level.conf \
  --rule depositThenWithdrawCannotProfit
certoraRun certora/confs/Savings/high_level.conf \
  --rule withdrawThenDepositCannotProfit
certoraRun certora/confs/Savings/high_level.conf \
  --rule redeemThenMintCannotProfit
certoraRun certora/confs/Savings/high_level.conf \
  --rule mintThenWithdrawCannotProfit
certoraRun certora/confs/Savings/high_level.conf \
  --rule moreAssetsDepositedMeansMoreShares
certoraRun certora/confs/Savings/high_level.conf \
  --rule moreSharesRedeemedMeansMoreAssets
certoraRun certora/confs/Savings/high_level.conf \
  --rule moreSharesMintedCostsMoreAssets
certoraRun certora/confs/Savings/high_level.conf \
  --rule moreAssetsWithdrawnBurnsMoreShares
certoraRun certora/confs/Savings/high_level.conf \
  --rule depositDoesNotDiluteShares
certoraRun certora/confs/Savings/high_level.conf \
  --rule mintDoesNotDiluteShares
certoraRun certora/confs/Savings/high_level.conf \
  --rule redeemDoesNotDiluteShares
certoraRun certora/confs/Savings/high_level.conf \
  --rule withdrawDoesNotDiluteShares
certoraRun certora/confs/Savings/high_level.conf \
  --rule depositIsAdditive
```