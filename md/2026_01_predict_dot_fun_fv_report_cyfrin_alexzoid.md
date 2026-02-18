# Formal Verification Report: predict.fun YieldBearingConditionalTokens

- Repository: https://github.com/Cyfrin/audit-2026-01-predict-dot-fun
- Latest Commit Hash: [ec9b1f8](https://github.com/Cyfrin/audit-2026-01-predict-dot-fun/commit/ec9b1f8889fdf9945e1d121b4c04657af657b7b5)
- Date: January 2026
- Author: [@alexzoid](https://x.com/alexzoid) ([@cyfrin](https://x.com/cyfrin) private formal verification engagement) 
- Certora Prover version: 8.6.3

---

## Table of Contents

1. [About predict.fun](#about-predictfun)
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
4. [Mutation Testing](#mutation-testing)
   - [What is Mutation Testing](#what-is-mutation-testing)
   - [VS-11: depositedAmountEqualsPositions](#vs-11-depositedamountequalspositions)
5. [Setup and Execution](#setup-and-execution)
   - [Common Setup (Steps 1–4)](#common-setup-steps-14)
   - [Remote Execution](#remote-execution)
   - [Local Execution](#local-execution)
   - [Running Verification](#running-verification)
   - [Running Mutation Testing](#running-mutation-testing)
   - [Source Code Modifications](#source-code-modifications)

---

## About predict.fun

predict.fun is a prediction market protocol built on the Gnosis Conditional Token Framework (CTF). Users stake collateral to mint conditional outcome tokens representing positions in binary prediction markets. The protocol extends the standard CTF with Venus Protocol integration for yield generation on idle collateral.

`YieldBearingConditionalTokens` is the core contract combining conditional token logic with Venus yield-bearing functionality. It inherits `WhitelistedERC1155` (transfer-controlled ERC-1155) and `Venus` (yield management via vTokens). This contract and its inherited contracts constitute the verification scope.

Key operations:
- `prepareCondition` / `reportPayouts` — lifecycle management for prediction conditions
- `splitPosition` / `mergePositions` / `redeemPositions` — conditional token operations backed by collateral
- `connectVTokenToUnderlying` / `enableUnderlying` / `disableUnderlying` / `claimYield` — Venus yield integration for idle collateral

When yield is enabled for an underlying token, the contract deposits collateral into Venus (minting vTokens) and redeems it on demand. The `depositedAmount` mapping tracks principal amounts, while yield accrues via vToken exchange rate appreciation.

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

### Verification Process

1. **Setup phase**: Define ghost variables, storage hooks, and helper definitions to model contract state in CVL. Establish the verification harness and configuration. This phase also addresses several prover limitations:
   - Bitwise operations ([`bitops.spec`](./specs/setup/bitops.spec)): The prover overapproximates bitwise ops, producing spurious counterexamples. All inline bitwise expressions were extracted into internal helper functions and replaced with precise CVL lookup-table summaries (see [Source Code Modifications](#source-code-modifications)).
   - ERC-1155 internal arrays ([`erc1155.spec`](./specs/setup/openzeppelin/erc1155.spec)): The prover's pointer analysis crashes on `ERC1155._asSingletonArrays()`. The internal `_burn`, `_mint`, and `_safeTransferFrom` functions are summarized in CVL, bypassing the problematic array construction entirely and tracking balances via ghost variables.
   - CTHelpers hash functions ([`ct_helpers_lib.spec`](./specs/setup/ct_helpers_lib.spec)): Keccak256-based ID computations (`getConditionId`, `getCollectionId`, `getPositionId`) are opaque to the prover. They are replaced with ghost mappings constrained by axioms that capture essential hash properties (non-zero results, injectivity, commutativity).
   - ERC-20 model ([`erc20.spec`](./specs/setup/erc20.spec)): Real ERC-20 contracts cause timeouts due to implementation complexity and unbounded address space. A lightweight ghost-variable-based model summarizes all external ERC-20 calls (`balanceOf`, `transfer`, `transferFrom`, `approve`) over a bounded account set, enabling tractable solvency invariants.
   - Venus vToken model ([`venus.spec`](./specs/setup/venus.spec)): Real Venus vToken contracts introduce non-linear arithmetic via variable exchange rates, causing solver timeouts. All external vToken calls (`mint`, `redeem`, `redeemUnderlying`, `balanceOfUnderlying`, `exchangeRateCurrent`) are summarized in CVL with a fixed 1:1 exchange rate. Ghost mappings with injectivity and anti-circularity axioms model the vToken-to-underlying relationship.
2. **Crafting Properties**: Write invariants and rules in CVL, starting with valid state invariants (which become trusted preconditions for other rules), then state/variable transitions, and finally high-level properties.

### Project Structure

```
certora/
├── confs/                                # Prover configuration files
│   └── yield_ctf/
│       ├── valid_state.conf
│       ├── variable_transitions.conf
│       ├── state_transitions.conf
│       ├── unit_tests.conf
│       ├── high_level.conf
│       └── valid_state_mutations_depositedAmountEqualsPositions.conf
│
├── harnesses/                            # Verification harnesses
│   ├── YieldBearingConditionalTokensHarness.sol
│   └── HelperCVL.sol
│
├── mutations/                            # Mutation testing files
│   ├── add_mutation.sh                   # Helper script to create new mutations
│   └── depositedAmountEqualsPositions/   # VS-11 mutation test suite (18 mutants)
│       ├── 1.sol
│       ├── ...
│       ├── 18.sol
│       ├── mutation-testing.png
│       └── mutation-testing_patch_8.png
│
└── specs/                                # CVL specifications
    ├── setup/                            # Shared setup: ghost variables, hooks, summaries
    │   ├── yield_ctf.spec                # Main harness setup, environment constraints
    │   ├── bitops.spec                   # Bitwise operation summaries (lookup tables)
    │   ├── ct_helpers_lib.spec           # CTHelpers hash function summaries (ghost mappings)
    │   ├── erc20.spec                    # Lightweight ERC-20 model (ghost-based)
    │   ├── venus.spec                    # Venus vToken model (fixed 1:1 exchange rate)
    │   ├── helper.spec                   # General helper definitions
    │   ├── math.spec                     # Math utility definitions
    │   ├── safe_erc20.spec               # SafeERC20 summaries
    │   └── openzeppelin/
    │       ├── erc1155.spec              # ERC-1155 internal function summaries
    │       ├── access_control.spec       # AccessControl summaries
    │       └── utils_array.spec          # Array utility summaries
    │
    └── yield_ctf/                        # Property specifications
        ├── yield_ctf_valid_state.spec    # Valid State invariants
        ├── yield_ctf_variable_transitions.spec  # Variable Transition rules
        ├── yield_ctf_state_transitions.spec     # State Transition rules
        ├── yield_ctf_unit_tests.spec     # Unit Test rules
        └── yield_ctf_high_level.spec     # High-Level rules
```

### Assumptions

Formal verification requires assumptions about the code and its environment to address prover timeouts, tool limitations, and state consistency. However, incorrect assumptions can mask real bugs by excluding reachable states from analysis. To maintain transparency, all assumptions are categorized into three groups: **Safe** (real-world constraints that don't reduce security coverage), **Proved** (formally verified invariants reused as preconditions), and **Unsafe** (scope reductions necessary for tractability that may exclude valid scenarios).

#### Safe Assumptions

These reflect real-world constraints that don't impact security guarantees. In the codebase, every `require` statement that constitutes a safe assumption is annotated with a `"SAFE: ..."` message string.

Environment Constraints ([`yield_ctf.spec`](./specs/setup/yield_ctf.spec)):
- The contract does not handle native ETH, so transactions carry no value
- The sender is never the zero address and never the contract itself
- Block timestamps fall within realistic bounds (up to year 2106)
- The block number is non-zero

Address Separation ([`yield_ctf.spec`](./specs/setup/yield_ctf.spec)):
- The underlying token address is non-zero and distinct from the CTF contract
- The oracle address is non-zero and distinct from the CTF contract
- User account addresses are non-zero, distinct from the CTF contract, and distinct from each other
- The CTF contract is neither a vToken nor an underlying token
- The caller is never one of the token contracts (underlying or vToken)

ERC-20 Model ([`erc20.spec`](./specs/setup/erc20.spec)):
- Total supply equals the sum of all individual balances (standard ERC-20 solvency invariant)
- Token decimals are between 6 and 18 (realistic range for deployed tokens)
- The called token contract is never the zero address

Venus vToken Model ([`venus.spec`](./specs/setup/venus.spec)):
- A vToken address always differs from its own underlying token address
- Different vTokens map to different underlying tokens (injectivity)
- If a token is the underlying of some vToken, it is not itself a vToken (no circular wrapping)

#### Proved Assumptions

These properties have been formally verified as valid state invariants and are used as trusted preconditions (via `requireInvariant`) in state transition and high-level rules. See [Valid State](#valid-state) for detailed descriptions and prover run links.

Conditional Token Invariants ([`yield_ctf_valid_state.spec`](./specs/yield_ctf/yield_ctf_valid_state.spec)):
- Condition ID zero is never prepared (VS-01)
- A prepared condition cannot have a zero oracle (VS-02)
- A prepared condition always has at least 2 outcomes (VS-03)
- The payout denominator equals the sum of payout numerators (VS-04)
- The payout numerators array length matches the outcome slot count used when creating the condition (VS-05)
- ERC-1155 token balances can only be non-zero for prepared conditions (VS-06)
- ERC-1155 positions can only exist for collateral tokens that have a registered vToken (VS-07)
- Before resolution, total supplies of both outcome positions are equal (VS-08)
- Before resolution, outstanding positions are fully backed by collateral (VS-09)
- After resolution, the weighted sum of positions is fully backed by collateral (VS-10)
- Before resolution with yield enabled, the deposited amount exactly equals outstanding positions (VS-11)
- Before resolution with yield disabled, the underlying token balance covers outstanding positions (VS-12)

Venus Integration Invariants ([`yield_ctf_valid_state.spec`](./specs/yield_ctf/yield_ctf_valid_state.spec)):
- The zero address has zero values in all underlying-related mappings (VS-13)
- Non-zero deposited amounts require the underlying to be enabled and have a registered vToken (VS-14)
- A token cannot simultaneously be registered as both an underlying and a vToken (VS-16)
- If an underlying maps to a vToken, then that vToken maps back to the same underlying (VS-17)
- If a vToken maps to an underlying and that underlying has a registered vToken, the mapping is consistent (VS-18)
- Deposited amounts are fully backed by the corresponding vToken value (VS-19)

> Note: VS-11 (`depositedAmountEqualsPositions`) uses strict equality (`==`) rather than `>=`, even though a weaker `>=` would be sound — a donate-then-enable flow causes `enableUnderlying` to deposit the entire contract balance (including donations) into Venus, inflating `depositedAmount` above outstanding positions. The strict equality is intentional: it catches any discrepancy, no matter how small, making the invariant maximally sensitive. To handle the `enableUnderlying` case, a dedicated `preserved` block assumes no untracked balance exists before enabling.

> Note: VS-15 (`enabledUnderlyingHasVToken`) is violated in `enableUnderlying()` — a non-connected token can be enabled. This has no meaningful security impact and is not used as a `requireInvariant`.

#### Unsafe Assumptions

These reduce verification scope to make the problem tractable for the prover. In the codebase, every `require` statement that constitutes an unsafe assumption is annotated with an `"UNSAFE: ..."` message string.

Scope Restrictions ([`yield_ctf.spec`](./specs/setup/yield_ctf.spec)):
Bounding the number of tokens, accounts, conditions, and outcomes serves two purposes: it prevents prover timeouts on unbounded quantifiers, and it makes it possible to reason about total supply as a concrete sum of all user balances — enabling protocol-wide invariants such as "the deposited amount exactly equals outstanding positions."

- Only a single underlying collateral token is supported
- Only a single oracle address is used for condition creation
- Only a single question identifier is used
- The maximum outcome slot count is capped at 2 (binary markets only)
- ERC-1155 balances are tracked for at most 2 user accounts
- Only a single condition can be in a prepared state at a time
- Only initial (non-nested) positions are verified — parent collection is always zero
- Conditions, collateral tokens, and accounts outside the supported set are assumed to hold zero ERC-1155 balances
- Payout numerators do not exist for unsupported conditions
- Payout numerators array length never exceeds the maximum supported outcome slot count

Bit Operations and Index Sets ([`bitops.spec`](./specs/setup/bitops.spec)):
By default the Certora Prover [overapproximates](https://docs.certora.com/en/latest/docs/user-guide/glossary.html#term-overapproximation) bitwise operations rather than building [precise models](https://docs.certora.com/en/latest/docs/prover/cli/options.html#precise-bitwise-ops), which produces spurious counterexamples or causes the prover to crash. To work around this, all bitwise operations in the Solidity source were extracted into separate internal functions and replaced with CVL summaries. Every such code change is marked with an `ORIGINAL:` comment showing the original expression. Inputs are bounded to the valid range for 2-outcome conditions to keep the CVL replacements sound.

- Index sets and free index sets are restricted to values at most 3 (valid for 2-outcome conditions)
- Bit position access is limited to indices below the maximum supported outcome slot count

Condition and Collection Helpers ([`ct_helpers_lib.spec`](./specs/setup/ct_helpers_lib.spec)):
The CTF helper library uses hash-based lookups (condition IDs, collection IDs) that are opaque to the prover. Restricting inputs to a single supported condition and zero parent collection makes these lookups deterministic and allows the prover to relate positions back to their underlying condition.

- Only a single condition with supported parameters (oracle, question ID, outcome slot count) is used
- Collections are derived only from the zero parent (no nested conditional positions)

Amount Bounds — Overflow Prevention ([`erc20.spec`](./specs/setup/erc20.spec), [`erc1155.spec`](./specs/setup/openzeppelin/erc1155.spec), [`yield_ctf.spec`](./specs/setup/yield_ctf.spec)):
Ghost variables use `mathint` (unbounded integers), so without explicit bounds the prover must consider astronomically large values that cannot occur in practice. Capping amounts well below `uint256` eliminates spurious overflow counterexamples while still covering all realistic token supplies.

- ERC-20 balances are bounded by max uint128 to prevent arithmetic overflows
- ERC-20 allowances are bounded by max uint128 to prevent arithmetic overflows
- ERC-1155 balances are bounded by max uint112 to prevent arithmetic overflows
- Deposited amounts are bounded by max uint128 to prevent arithmetic overflows

Account Restrictions ([`erc20.spec`](./specs/setup/erc20.spec), [`yield_ctf.spec`](./specs/setup/yield_ctf.spec)):
Restricting accounts to a finite set allows total supply to be expressed as a concrete sum and ensures storage hooks only fire for known addresses, preventing the prover from reasoning about an unbounded address space.

- All ERC-20 operations restrict accounts to a predefined set of up to 5 addresses per token
- ERC-1155 storage hooks for balances and operator approvals restrict accounts to the 2 supported addresses

Venus Exchange Rate ([`venus.spec`](./specs/setup/venus.spec)):
The Venus protocol's variable exchange rate introduces non-linear arithmetic that is intractable for the SMT solver. Fixing it at 1:1 makes vToken amounts equal to underlying amounts, reducing the problem to linear arithmetic while preserving the deposit/redeem control flow.

- The vToken exchange rate is fixed at 1:1 — minting, redeeming, and balance-of-underlying all use a 1e18 exchange rate

Prover Configuration:
- Loop unrolling is capped at 2 iterations across all configurations

<div style="page-break-before: always;"></div>

---

## Verification Properties

Links to specific Certora Prover runs are provided for each property, with status indicators.

- ✅ Verified successfully
- ⚠️ Timeout 
- ❌ Violated (indicates a potential issue) 

### Valid State

Valid State properties define the fundamental invariants that must always hold true throughout the protocol's lifecycle. These are parametric — each property is automatically verified against every external function in the contract, including functions added in the future.

#### Conditional Token Invariants

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VS-01](./specs/yield_ctf/yield_ctf_valid_state.spec#L50-L52) | `conditionIdZeroNotPrepared` | ConditionId 0 is never prepared | [✅](https://prover.certora.com/output/52567/ca749d748eba4c00b8a7764d2ae9cc4b/?anonymousKey=a7864574de5338b1e5e5d8491f10ef12e64ab03e) | |
| [VS-02](./specs/yield_ctf/yield_ctf_valid_state.spec#L55-L61) | `preparedConditionNotWithZeroOracle` | Prepared condition cannot be created by zero oracle | [✅](https://prover.certora.com/output/52567/a7f5d97f91bb49988a204a43f84d07d9/?anonymousKey=c7c069a2f14fc73f415dd28a14ce33dda4834dc6) | |
| [VS-03](./specs/yield_ctf/yield_ctf_valid_state.spec#L64-L68) | `preparedConditionHasMinTwoOutcomes` | Prepared condition has at least 2 outcomes | [✅](https://prover.certora.com/output/52567/1c75aa852dfb48c087c9e64dcde31113/?anonymousKey=e5ce7bb09d9db67d1014818d92925b3292ff678f) | |
| [VS-04](./specs/yield_ctf/yield_ctf_valid_state.spec#L72-L76) | `payoutDenominatorEqualsSumNumerators` | Payout denominator equals sum of payout numerators | [✅](https://prover.certora.com/output/52567/3d659ae3a6524c09b78c7a41b0ef48bc/?anonymousKey=688b42ef0a20072d35e06a8ce5307c5905ab5618) | |
| [VS-05](./specs/yield_ctf/yield_ctf_valid_state.spec#L79-L85) | `conditionIdMatchesOutcomeSlotCount` | Prepared conditionId length matches outcomeSlotCount in hash | [✅](https://prover.certora.com/output/52567/d29189c32498496d95214b6ac6862295/?anonymousKey=e640a315fbe3f8217ba94ee443a31353b6b7d436) | |
| [VS-06](./specs/yield_ctf/yield_ctf_valid_state.spec#L88-L100) | `erc1155ExistsImpliesConditionPrepared` | ERC-1155 tokens exist only for prepared conditions | [✅](https://prover.certora.com/output/52567/24e14f2b292443fa918a7da55ea6f1db/?anonymousKey=a69eeedd3077c6aa2b998b86b4e11498676a7892) | |
| [VS-07](./specs/yield_ctf/yield_ctf_valid_state.spec#L103-L115) | `erc1155RequiresRegisteredVToken` | ERC-1155 positions exist only for collateral with registered vToken | [✅](https://prover.certora.com/output/52567/82afd523fbf24cc59fca029746158fe5/?anonymousKey=aa01f9e37e1b357c7fdb4f6ab193e2c27ca47ef7) | |
| [VS-08](./specs/yield_ctf/yield_ctf_valid_state.spec#L120-L124) | `positionSuppliesEqualBeforeResolution` | Total supplies of both outcomes are equal before resolution | [✅](https://prover.certora.com/output/52567/37f08ee20d9245eb9cab361bbc05cd6a/?anonymousKey=dcf65a44ffc730897570c3bb5e520a4cda43e674) | |
| [VS-09](./specs/yield_ctf/yield_ctf_valid_state.spec#L129-L133) | `erc1155PositionsFullyBackedUnresolved` | ERC-1155 positions total supply backed by collateral (before resolution) | [✅](https://prover.certora.com/output/52567/16c66e8e12d8476bb0215022c861643a/?anonymousKey=2fe787f45fa219cd24d557b3545ee175bd02f569) | |
| [VS-10](./specs/yield_ctf/yield_ctf_valid_state.spec#L138-L144) | `erc1155PositionsFullyBackedResolved` | ERC-1155 positions total supply backed by collateral (after resolution) | [✅](https://prover.certora.com/output/52567/510cedef50f44965a894205cdad405f0/?anonymousKey=80e6652b81422895ec5bcd634488a33cba944e24) | [✅](https://prover.certora.com/output/52567/86d58481925f4aeb8c8d66173a3bd699/?anonymousKey=5e540dc6b479fcf2f0d8d390e3007d553351e27a) `redeemPositions()` passed [✅](https://prover.certora.com/output/52567/59ea2b789ba641229505d20ff6ee8bc4/?anonymousKey=d1fdb1efe38fc147fb43c5fa69f55e5fe7bc6ed3) `splitPosition()` passed [⚠️](https://prover.certora.com/output/52567/187333e81ab34932a2d08102f3e407bf/?anonymousKey=a8a266bda0c6b554134883013d1e89b8d069ac2e) `mergePositions()` timeout |
| [VS-11](./specs/yield_ctf/yield_ctf_valid_state.spec#L148-L166) | `depositedAmountEqualsPositions` | Before resolution, deposited amount tracks outstanding positions exactly | [✅](https://prover.certora.com/output/52567/905193582c2a495f954e6f4b290ebd86/?anonymousKey=500e5a48c66221af4a2a46cc704fb0fbc32763dd) | [Mutation tested](https://mutation-testing.certora.com/?id=30591291-b949-4ecc-8754-ac7446213654&anonymousKey=2690c41f-ed4b-428a-b568-3e95fca56d5d) — INFO: `preserved` block for `enableUnderlying` requires unsafe assumption (balance == outstanding positions) |
| [VS-12](./specs/yield_ctf/yield_ctf_valid_state.spec#L171-L175) | `underlyingBalanceBacksPositions` | Before resolution with yield disabled, underlying balance covers positions | [✅](https://prover.certora.com/output/52567/8a4d69066acd429f85e53c160033a183/?anonymousKey=e691450e987efdf42f7410975c057b8c7b51fc2c) | |

#### Venus Integration Invariants

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VS-13](./specs/yield_ctf/yield_ctf_valid_state.spec#L182-L186) | `zeroUnderlyingHasZeroValues` | Zero address underlying has zero values in all mappings | [✅](https://prover.certora.com/output/1585051/010233a2582841209838cd6f36a6eddf/?anonymousKey=6549bd49cca2aac600d05f720c35c32d87228248) | |
| [VS-14](./specs/yield_ctf/yield_ctf_valid_state.spec#L189-L193) | `depositsRequireEnabledAndVToken` | Deposits require underlying to be enabled and have a vToken | [✅](https://prover.certora.com/output/1585051/6767fafd160a4e39aeb4905725be205d/?anonymousKey=d6d1726ba8a8890d0d816e9592c09db54675231f) | |
| [VS-15](./specs/yield_ctf/yield_ctf_valid_state.spec#L197-L201) | `enabledUnderlyingHasVToken` | Enabled underlying must have a vToken | [❌](https://prover.certora.com/output/1585051/1e97d24f7d5b47b98e57477247b110fc/?anonymousKey=d18823a136aba25a31cb3b4a0336bce82c3f5db4) | Violated in `enableUnderlying()`: non-connected token can be enabled — no meaningful impact |
| [VS-16](./specs/yield_ctf/yield_ctf_valid_state.spec#L204-L208) | `underlyingNotVToken` | Token cannot be both underlying and vToken simultaneously | [✅](https://prover.certora.com/output/1585051/f3ad622660c24583b5de3ad33d724d76/?anonymousKey=c70f99b717c555b74529697b0411d6bceec6ff5a) | |
| [VS-17](./specs/yield_ctf/yield_ctf_valid_state.spec#L211-L215) | `vTokenMappingConsistency` | If underlying maps to vToken, then vToken maps back to underlying | [✅](https://prover.certora.com/output/1585051/1d1da595b3c64eba8cb1d069d1d29965/?anonymousKey=d6391b2a57c0263147ca0c0bb0fda33ad1445a24) | |
| [VS-18](./specs/yield_ctf/yield_ctf_valid_state.spec#L218-L223) | `vTokenReverseMappingConsistency` | If vToken maps to underlying AND underlying is registered, mapping is consistent | [✅](https://prover.certora.com/output/1585051/45c5ea64db574e57a8657b89760457d2/?anonymousKey=b575db73f5b1a791ce35a065fe501e9c91e8951d) | |
| [VS-19](./specs/yield_ctf/yield_ctf_valid_state.spec#L226-L230) | `depositsFullyBacked` | Deposited amount must be fully backed by vToken value | [✅](https://prover.certora.com/output/1585051/1842bd892c7748a2a6f4d7ad426da053/?anonymousKey=5e28bfc2e97e3f91fe414fb3ff6bd9a4f63896c4) | |

### Variable Transitions

Variable Transition properties verify that specific storage variables change only under expected conditions (monotonicity — set once and never changed). These are parametric — each property is automatically verified against every external function in the contract, including functions added in the future.

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VT-01](./specs/yield_ctf/yield_ctf_variable_transitions.spec#L6-L20) | `transitionUnderlyingToVTokenSetOnce` | underlyingToVToken can only be set once (from zero to non-zero) | [✅](https://prover.certora.com/output/1585051/ce5ee7266b424363914e0ff952af0dff/?anonymousKey=1129d9e841addc9b96ff5fe5279c14ab9dc81be1) | |
| [VT-02](./specs/yield_ctf/yield_ctf_variable_transitions.spec#L22-L36) | `transitionPayoutDenominatorSetOnce` | payoutDenominator can only be set once (from zero to non-zero) | [✅](https://prover.certora.com/output/1585051/ce5ee7266b424363914e0ff952af0dff/?anonymousKey=1129d9e841addc9b96ff5fe5279c14ab9dc81be1) | |
| [VT-03](./specs/yield_ctf/yield_ctf_variable_transitions.spec#L38-L52) | `transitionPayoutNumeratorsSetOnce` | payoutNumerators can only be set once (from zero to non-zero) | [✅](https://prover.certora.com/output/1585051/ce5ee7266b424363914e0ff952af0dff/?anonymousKey=1129d9e841addc9b96ff5fe5279c14ab9dc81be1) | |
| [VT-04](./specs/yield_ctf/yield_ctf_variable_transitions.spec#L54-L68) | `transitionPayoutNumeratorsLengthSetOnce` | payoutNumerators length can only be set once (from zero to non-zero) | [✅](https://prover.certora.com/output/1585051/ce5ee7266b424363914e0ff952af0dff/?anonymousKey=1129d9e841addc9b96ff5fe5279c14ab9dc81be1) | |

### State Transitions

State Transition properties verify the correctness of transitions between valid states. These are parametric — each property is automatically verified against every external function in the contract, including functions added in the future.

#### Venus Deposit/Withdrawal Correlation

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [ST-01](./specs/yield_ctf/yield_ctf_state_transitions.spec#L3-L21) | `depositedAmountIncreasesTogetherWithVTokenBalance` | depositedAmount increase must correlate with vToken balance increase | [✅](https://prover.certora.com/output/1585051/c7f64de3b38b4e86b5b8b59c72ba0885/?anonymousKey=9c646eaaf041c272788cc9ed43c01c6e69f6222e) | |
| [ST-02](./specs/yield_ctf/yield_ctf_state_transitions.spec#L23-L41) | `depositedAmountDecreasesTogetherWithVTokenBalance` | depositedAmount decrease must correlate with vToken balance decrease | [✅](https://prover.certora.com/output/1585051/36f55db5831144b3a4d1f2a94fc79527/?anonymousKey=23887421724b3ec7e80e7769aa1943500d647e76) | |
| [ST-03](./specs/yield_ctf/yield_ctf_state_transitions.spec#L43-L62) | `vTokenBalanceDecreaseOnlyViaBurnOnRedeem` | CTF vToken balance can only decrease via redeem | [✅](https://prover.certora.com/output/52567/e0cd3727e4cc46598218ea420ed59f4d/?anonymousKey=9fd8e44520ee8bd38cee8996f128130d632f2579) | |

#### ERC-1155 Transfer Isolation

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [ST-04](./specs/yield_ctf/yield_ctf_state_transitions.spec#L64-L101) | `erc1155TransferDoesNotAffectTotalBalance` | ERC-1155 transfer does not affect total balance | [✅](https://prover.certora.com/output/1585051/7f11aa8b080e4319b0c57e45c00128d6/?anonymousKey=4b237dfa48989c629eebee07104db6336cafadb0) | |
| [ST-05](./specs/yield_ctf/yield_ctf_state_transitions.spec#L103-L137) | `erc1155TransferDoesNotAffectVTokenBalance` | ERC-1155 transfer does not affect vToken balance on CTF | [✅](https://prover.certora.com/output/1585051/1d244142dd2c4597ba4bbe7423904a5e/?anonymousKey=3e0e21ef4b89f8b1e0640e4e68f95917b7c9edea) | |
| [ST-06](./specs/yield_ctf/yield_ctf_state_transitions.spec#L139-L165) | `erc1155TransferDoesNotAffectUnderlyingBalance` | ERC-1155 transfer does not affect underlying balance on CTF | [✅](https://prover.certora.com/output/1585051/70c9ecf2ba274b0d9a187a57ac0c5feb/?anonymousKey=1f13bf89acf248ad841545d0e11e0987711a63fb) | |
| [ST-07](./specs/yield_ctf/yield_ctf_state_transitions.spec#L167-L193) | `erc1155TransferDoesNotAffectDepositedAmount` | ERC-1155 transfer does not affect depositedAmount | [✅](https://prover.certora.com/output/1585051/e45bda1164974a8c8184b530672a6d7f/?anonymousKey=b6b6b79cea2c376a94ea35590209456a763b6a2d) | |

#### Minting, Access Control, and Resolution

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [ST-08](./specs/yield_ctf/yield_ctf_state_transitions.spec#L195-L246) | `erc1155MintRequiresBurnAnotherOrCollateralIn` | ERC-1155 mint requires either another position burn or collateral transfer in | [✅](https://prover.certora.com/output/1585051/a7d5572537d54ca7b2826947ddd0dc7a/?anonymousKey=5bba2a48c7b339ed8f6e049615fe8ae93a20a8dd) | |
| [ST-09](./specs/yield_ctf/yield_ctf_state_transitions.spec#L248-L284) | `erc1155TransferRequiresWhitelistedParticipant` | ERC-1155 transfer with transferControl requires whitelisted participant | [✅](https://prover.certora.com/output/1585051/7068ec7fd98d4edb98714ad57326bcaf/?anonymousKey=152adf5d6c41c5a0c10ca4503bac8b30cb29906f) | |
| [ST-10](./specs/yield_ctf/yield_ctf_state_transitions.spec#L286-L305) | `reportPayoutsSetsAtLeastOneNumerator` | reportPayouts must set at least one numerator when setting denominator | [✅](https://prover.certora.com/output/1585051/8de524b6a7094a479ae9aa2325388bdd/?anonymousKey=692f3920c43d9e3dc91d28d08c012896405733d4) | |
| [ST-11](./specs/yield_ctf/yield_ctf_state_transitions.spec#L308-L341) | `underlyingDecreaseRequiresYieldManagerRole` | Underlying balance decrease requires YIELD_MANAGER_ROLE when positions unchanged | [✅](https://prover.certora.com/output/1585051/a8235aa7414f41c5ab08a9e4290d6c92/?anonymousKey=fce3af35afe1251e303698ceb1137fb8f4c7db07) | |

### High-Level

High-Level properties verify multi-step business logic, user isolation across core operations, and monotonicity of core operations.

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [HL-01](./specs/yield_ctf/yield_ctf_high_level.spec#L3-L18) | `splitMergePreservesCollateral` | Split + Merge keeps collateral unchanged | [✅](https://prover.certora.com/output/52567/90f2cce328c043f8a509ca92f7acc840/?anonymousKey=88bc9548b5d8c7e399b4d62b4ee695da259fecd4) | |
| [HL-02](./specs/yield_ctf/yield_ctf_high_level.spec#L20-L33) | `splitThenMergePositionsRestored` | Split + Merge preserves ERC-1155 position balances | [✅](https://prover.certora.com/output/52567/fdfdc665ac0e47939336b323dadd25a1/?anonymousKey=90029cb2e3a08cada51948320c33a1e713c5092f) | |
| [HL-03](./specs/yield_ctf/yield_ctf_high_level.spec#L35-L50) | `splitDoesNotAffectOtherUsersPositions` | Split does not affect other users' ERC-1155 positions | [✅](https://prover.certora.com/output/52567/1971d52d627e429988f8dae64aca81da/?anonymousKey=bdf783b04b4676751db734c84521e322bba55799) | |
| [HL-04](./specs/yield_ctf/yield_ctf_high_level.spec#L52-L67) | `mergeDoesNotAffectOtherUsersPositions` | Merge does not affect other users' ERC-1155 positions | [✅](https://prover.certora.com/output/52567/51fc4f9aaf9a44e7a1f3117c34614bf3/?anonymousKey=459b35ec118b84e08317fc975e94f2bd3a1f50bd) | |
| [HL-05](./specs/yield_ctf/yield_ctf_high_level.spec#L69-L83) | `redeemDoesNotAffectOtherUsers` | Redeem does not affect other users' positions | [✅](https://prover.certora.com/output/52567/90ff40c5599645088f69e2b60e1b6a3b/?anonymousKey=fd8706a52e8dc7efaeee9d6eb92e6b0740d40fe3) | |
| [HL-06](./specs/yield_ctf/yield_ctf_high_level.spec#L85-L97) | `redeemBurnsCallerTokens` | Redeem does not increase caller's position balance | [✅](https://prover.certora.com/output/52567/d39fff44fafd444fac9ece2a0ca57a4e/?anonymousKey=4ef2b24ea3727d8d64cc4c0cdbae9afb827505fa) | |
| [HL-07](./specs/yield_ctf/yield_ctf_high_level.spec#L99-L111) | `redeemDoesNotIncreaseCollateral` | Redeem does not increase collateral backing | [✅](https://prover.certora.com/output/52567/84fb6d17c57f41eeac45e4979fa7801a/?anonymousKey=967298da1e754ae41400138f57f18e8a69a1572c) | |

### Unit Tests

Unit Test properties verify basic behavior of individual functions: revert conditions, direct effects, and non-effects on unrelated state.

#### Revert Checks

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-01](./specs/yield_ctf/yield_ctf_unit_tests.spec#L7-L18) | `reportPayoutsImmutability` | reportPayouts reverts for already resolved condition | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |
| [UT-02](./specs/yield_ctf/yield_ctf_unit_tests.spec#L20-L31) | `prepareConditionIdempotencyGuard` | prepareCondition reverts for already prepared condition | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |
| [UT-03](./specs/yield_ctf/yield_ctf_unit_tests.spec#L33-L43) | `redeemRevertsIfNotResolved` | redeemPositions reverts if condition not resolved | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |
| [UT-04](./specs/yield_ctf/yield_ctf_unit_tests.spec#L45-L56) | `splitRevertsWhenGatedAndUnauthorized` | splitPosition reverts when gated and caller lacks role | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |
| [UT-05](./specs/yield_ctf/yield_ctf_unit_tests.spec#L58-L69) | `mergeRevertsWhenGatedAndUnauthorized` | mergePositions reverts when gated and caller lacks role | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |
| [UT-06](./specs/yield_ctf/yield_ctf_unit_tests.spec#L71-L81) | `claimYieldOnlyYieldManager` | claimYield reverts when caller lacks YIELD_MANAGER_ROLE | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |

#### Direct Effects

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-07](./specs/yield_ctf/yield_ctf_unit_tests.spec#L87-L95) | `reportPayoutsSetsResolved` | reportPayouts sets condition as resolved | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |
| [UT-08](./specs/yield_ctf/yield_ctf_unit_tests.spec#L97-L106) | `prepareConditionSetsOutcomeSlots` | prepareCondition initializes outcome slots | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |

#### Non-Effects

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [UT-09](./specs/yield_ctf/yield_ctf_unit_tests.spec#L112-L122) | `reportPayoutsDoesNotChangeCollateral` | reportPayouts does not change collateral backing | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |
| [UT-10](./specs/yield_ctf/yield_ctf_unit_tests.spec#L124-L134) | `reportPayoutsDoesNotAffectPositions` | reportPayouts does not affect ERC-1155 balances | [✅](https://prover.certora.com/output/52567/0a841c11feb9461db25e4696cef30a8c/?anonymousKey=2b5805b3f6b312513a1115f3723aa094a5c1a015) | |

<div style="page-break-before: always;"></div>

---

## Mutation Testing

Valid state invariants are powerful — a single well-crafted invariant can catch a wide range of bugs across multiple functions. To demonstrate this, we use `depositedAmountEqualsPositions` (VS-11) as a case study, showing through mutation testing that this one invariant detects 18 distinct bugs in `splitPosition` and `mergePositions`.

### What is Mutation Testing

Mutation testing validates the strength of a formal specification by injecting small syntactic changes (mutations) into the source code and verifying that the spec catches them. Each mutation represents a potential bug — a removed function call, a flipped condition, a swapped parameter. 

The [certoraMutate](https://docs.certora.com/en/latest/docs/gambit/mutation-verifier.html) tool automates this process: it applies each mutant to the source, runs the prover against the specified rule, and reports whether the invariant caught the introduced defect.

### VS-11: depositedAmountEqualsPositions

The `depositedAmountEqualsPositions` invariant asserts that when yield is enabled and the condition is unresolved, the Venus deposited amount exactly equals the total outstanding ERC-1155 positions. This strict equality makes the invariant maximally sensitive to any accounting discrepancy.

**Result: [18/18 mutations caught ❌](https://mutation-testing.certora.com/?id=30591291-b949-4ecc-8754-ac7446213654&anonymousKey=2690c41f-ed4b-428a-b568-3e95fca56d5d)** — the invariant catches every injected bug.

![mutation-testing](./mutations/depositedAmountEqualsPositions/mutation-testing.png)

#### splitPosition mutations

**#1** [❌](https://prover.certora.com/output/52567/ed2b2e7c397c4b95852da8e06470c98c/?anonymousKey=ed7ca5d0835db64e85ebbf6430afdc9176746d09) Remove Venus deposit after collateral transfer.

```diff
 if (underlyingIsEnabled[address(collateralToken)]) {
-    _mintVToken(address(collateralToken), amount);
+    // removed
 }
```

**#2** [❌](https://prover.certora.com/output/52567/c48c321b88764947887509904d2d8a4c/?anonymousKey=673b36febb97c01ddb4fc87978330ec92246827d) Flip full-partition check from `==` to `!=`.

```diff
-if (freeIndexSet == 0) {
+if (freeIndexSet != 0) {
```

**#3** [❌](https://prover.certora.com/output/52567/3fa6b1908ef94ce8898c484b2a39d570/?anonymousKey=5aebaadab879653b924699e13bc453b0ab412f1e) Flip parent collection check from `==` to `!=`.

```diff
-if (parentCollectionId == bytes32(0)) {
+if (parentCollectionId != bytes32(0)) {
```

**#4** [❌](https://prover.certora.com/output/52567/acd7cee3a782451c823a78c2354f0865/?anonymousKey=7ea59ac5bb2e9871908e46291c3878e2baecf530) Remove entire collateral handling block (transfer, Venus deposit, and partial-split burn).

```diff
-if (freeIndexSet == 0) {
-    if (parentCollectionId == bytes32(0)) {
-        require(collateralToken.transferFrom(msg.sender, address(this), amount), ...);
-        if (underlyingIsEnabled[address(collateralToken)]) {
-            _mintVToken(address(collateralToken), amount);
-        }
-    } else {
-        _burn(msg.sender, CTHelpers.getPositionId(collateralToken, parentCollectionId), amount);
-    }
-} else {
-    _burn(msg.sender, CTHelpers.getPositionId(collateralToken, ...), amount);
-}
+// removed
```

**#5** [❌](https://prover.certora.com/output/52567/f348a8cb30fc43fb8bafecc8d7adafdb/?anonymousKey=7826373cc3f0d585c61b342256a554cdc9b5f7f9) Remove position minting call.

```diff
-_mintBatch(msg.sender, positionIds, amounts, "");
+// removed
```

**#11** [❌](https://prover.certora.com/output/52567/87a4ee11b916488ab9c5458ed2ba2a9e/?anonymousKey=ffff85e1b8e4790525f2cfa52170429f0c7765dc) Negate yield-enabled check before Venus deposit.

```diff
-if (underlyingIsEnabled[address(collateralToken)]) {
+if (!underlyingIsEnabled[address(collateralToken)]) {
```

**#12** [❌](https://prover.certora.com/output/52567/70b91ae67d684399a8c6313142366a6a/?anonymousKey=4e78e058ae5edf0f686f5b30e1c6378466658d14) Off-by-one in Venus deposit amount.

```diff
-_mintVToken(address(collateralToken), amount);
+_mintVToken(address(collateralToken), amount - 1);
```

**#13** [❌](https://prover.certora.com/output/52567/0a6ee45f4d5041b7a255965a6d79669c/?anonymousKey=f4638a9c962b204613f039e7c4c1bdb4d98818ea) Zero out mint amounts for all positions.

```diff
-amounts[i] = amount;
+amounts[i] = 0;
```

#### mergePositions mutations

**#6** [❌](https://prover.certora.com/output/52567/1fa41845d97c4bd8bfdf68ae0ccaedf4/?anonymousKey=9746109f94fd19abbf0da920844dbdfa6ed046d7) Skip Venus redeem, transfer raw amount directly.

```diff
 if (underlyingIsEnabled[address(collateralToken)]) {
-    uint256 amountRedeemed = _redeemUnderlying(address(collateralToken), amount);
-    require(collateralToken.transfer(msg.sender, amountRedeemed), ...);
+    require(collateralToken.transfer(msg.sender, amount), ...);
 }
```

**#7** [❌](https://prover.certora.com/output/52567/3b54a49d26534c1eb851b9915debbb8b/?anonymousKey=f2d5ebc03d4274b759f913dcf78157835507b4c8) Remove position burn batch call.

```diff
-_burnBatch(msg.sender, positionIds, amounts);
+// removed
```

**#8** [❌](https://prover.certora.com/output/52567/ba86ca1569e44f1abd0d71c049273335/?anonymousKey=8b0c273c477c91c2799cef2ef06d17ce91d6c7da) Flip full-partition check from `==` to `!=`.

```diff
-if (freeIndexSet == 0) {
+if (freeIndexSet != 0) {
```

![mutation-testing_patch_8](./mutations/depositedAmountEqualsPositions/mutation-testing_patch_8.png)

**#9** [❌](https://prover.certora.com/output/52567/fca93dd8251249cbb2aff2fb7863b1f1/?anonymousKey=d9e9082418f9d005352308bf56809c23f54376f9) Negate yield-enabled check before Venus redeem.

```diff
-if (underlyingIsEnabled[address(collateralToken)]) {
+if (!underlyingIsEnabled[address(collateralToken)]) {
```

**#10** [❌](https://prover.certora.com/output/52567/39ac2bc1b1c6463caf949e4827ae85f5/?anonymousKey=c2faef09426b8b1468bf6bd877fd2a7c8103a69a) Flip parent collection check from `==` to `!=`.

```diff
-if (parentCollectionId == bytes32(0)) {
+if (parentCollectionId != bytes32(0)) {
```

**#14** [❌](https://prover.certora.com/output/52567/2e5670925b5d4afc851c619582aad318/?anonymousKey=94ba14692d26761f1b6e506f1d5d2900f2eee442) Off-by-one over-redeem from Venus.

```diff
-uint256 amountRedeemed = _redeemUnderlying(address(collateralToken), amount);
+uint256 amountRedeemed = _redeemUnderlying(address(collateralToken), amount + 1);
```

**#15** [❌](https://prover.certora.com/output/52567/1444dc9ff8984f56badbe7f98412ebe0/?anonymousKey=2bd9309f1f7240c02c391ee3e1a33ddd8bc4ee28) Zero Venus redeem amount.

```diff
-uint256 amountRedeemed = _redeemUnderlying(address(collateralToken), amount);
+uint256 amountRedeemed = _redeemUnderlying(address(collateralToken), 0);
```

**#16** [❌](https://prover.certora.com/output/52567/ab203d3e762142b3b14d91962413a99c/?anonymousKey=e975233ddb280f8b9d19ea0fcdbd13343a34707c) Zero out burn amounts for all positions.

```diff
-amounts[i] = amount;
+amounts[i] = 0;
```

**#17** [❌](https://prover.certora.com/output/52567/d6376a3a7e02462c99a754dd39743989/?anonymousKey=0a8529999821037c224aacc05d03da63da965506) Double burn amounts for all positions.

```diff
-amounts[i] = amount;
+amounts[i] = amount * 2;
```

**#18** [❌](https://prover.certora.com/output/52567/34b97c1ce1174ee5839e6b82511cc7d8/?anonymousKey=be5ae3c865464cf6f2cc26cf47a08875f183f19b) Wrong mapping key in yield-enabled check.

```diff
-if (underlyingIsEnabled[address(collateralToken)]) {
+if (underlyingIsEnabled[address(this)]) {
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

3. Install Certora CLI. To match a specific prover version, pin it explicitly (e.g. `certora-cli==8.6.3`)

```bash
pipx install certora-cli
```

4. Install solc-select and the Solidity compiler version required by the project

```bash
pipx install solc-select
solc-select install 0.8.24
solc-select use 0.8.24
```

### Remote Execution

5. Set up Certora key. You can get a free key through the Certora [Discord](https://discord.gg/certora) or on their website. Once you have it, export it:

```bash
echo "export CERTORAKEY=<your_certora_api_key>" >> ~/.bashrc
```

> **Note:** If a local prover is installed (see below), it takes priority. To force remote execution, add the `--server production` flag:
> ```bash
> certoraRun certora/confs/yield_ctf/valid_state.conf --server production
> ```

### Local Execution

Follow the full build instructions in the [CertoraProver repository (v8.6.3)](https://github.com/Certora/CertoraProver/tree/8.6.3). Once the local prover is installed it takes priority over the remote cloud by default. Tested on Ubuntu 24.04.

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
git checkout tags/8.6.3
./gradlew assemble
```

4. Verify installation with test example

```bash
certoraRun.py -h
cd Public/TestEVM/Counter
certoraRun counter.conf
```

### Running Verification

#### Quick Runs

`variable_transitions` and `unit_tests` are lightweight and can be run as full conf files:

```bash
certoraRun certora/confs/yield_ctf/variable_transitions.conf
certoraRun certora/confs/yield_ctf/unit_tests.conf
```

> **Note:** `valid_state`, `state_transitions`, and `high_level` may time out when running all rules at once. Run them per-rule using [`--rule`](https://docs.certora.com/en/latest/docs/prover/cli/options.html#rule), [`--method`](https://docs.certora.com/en/latest/docs/prover/cli/options.html#method), and [`--exclude_method`](https://docs.certora.com/en/latest/docs/prover/cli/options.html#exclude-method) as shown below.

#### Valid State

```bash
certoraRun certora/confs/yield_ctf/valid_state.conf --rule conditionIdZeroNotPrepared
certoraRun certora/confs/yield_ctf/valid_state.conf --rule preparedConditionNotWithZeroOracle
certoraRun certora/confs/yield_ctf/valid_state.conf --rule preparedConditionHasMinTwoOutcomes
certoraRun certora/confs/yield_ctf/valid_state.conf --rule payoutDenominatorEqualsSumNumerators
certoraRun certora/confs/yield_ctf/valid_state.conf --rule conditionIdMatchesOutcomeSlotCount
certoraRun certora/confs/yield_ctf/valid_state.conf --rule erc1155ExistsImpliesConditionPrepared
certoraRun certora/confs/yield_ctf/valid_state.conf --rule erc1155RequiresRegisteredVToken
certoraRun certora/confs/yield_ctf/valid_state.conf --rule positionSuppliesEqualBeforeResolution
certoraRun certora/confs/yield_ctf/valid_state.conf --rule erc1155PositionsFullyBackedUnresolved
certoraRun certora/confs/yield_ctf/valid_state.conf --rule depositedAmountEqualsPositions
certoraRun certora/confs/yield_ctf/valid_state.conf --rule underlyingBalanceBacksPositions
certoraRun certora/confs/yield_ctf/valid_state.conf --rule zeroUnderlyingHasZeroValues
certoraRun certora/confs/yield_ctf/valid_state.conf --rule depositsRequireEnabledAndVToken
certoraRun certora/confs/yield_ctf/valid_state.conf --rule enabledUnderlyingHasVToken
certoraRun certora/confs/yield_ctf/valid_state.conf --rule underlyingNotVToken
certoraRun certora/confs/yield_ctf/valid_state.conf --rule vTokenMappingConsistency
certoraRun certora/confs/yield_ctf/valid_state.conf --rule vTokenReverseMappingConsistency
certoraRun certora/confs/yield_ctf/valid_state.conf --rule depositsFullyBacked
```

`erc1155PositionsFullyBackedResolved` requires per-method splits to avoid timeouts:

```bash
certoraRun certora/confs/yield_ctf/valid_state.conf --rule erc1155PositionsFullyBackedResolved --method "splitPosition(address,bytes32,bytes32,uint256[],uint256)"
certoraRun certora/confs/yield_ctf/valid_state.conf --rule erc1155PositionsFullyBackedResolved --method "mergePositions(address,bytes32,bytes32,uint256[],uint256)"
certoraRun certora/confs/yield_ctf/valid_state.conf --rule erc1155PositionsFullyBackedResolved --method "redeemPositions(address,bytes32,bytes32,uint256[])"
certoraRun certora/confs/yield_ctf/valid_state.conf --rule erc1155PositionsFullyBackedResolved --exclude_method "splitPosition(address,bytes32,bytes32,uint256[],uint256)" --exclude_method "mergePositions(address,bytes32,bytes32,uint256[],uint256)" --exclude_method "redeemPositions(address,bytes32,bytes32,uint256[])"
```

#### State Transitions

```bash
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule depositedAmountIncreasesTogetherWithVTokenBalance
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule depositedAmountDecreasesTogetherWithVTokenBalance
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule vTokenBalanceDecreaseOnlyViaBurnOnRedeem
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule erc1155TransferDoesNotAffectTotalBalance
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule erc1155TransferDoesNotAffectVTokenBalance
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule erc1155TransferDoesNotAffectUnderlyingBalance
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule erc1155TransferDoesNotAffectDepositedAmount
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule erc1155MintRequiresBurnAnotherOrCollateralIn
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule erc1155TransferRequiresWhitelistedParticipant
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule reportPayoutsSetsAtLeastOneNumerator
certoraRun certora/confs/yield_ctf/state_transitions.conf --rule underlyingDecreaseRequiresYieldManagerRole
```

#### High-Level

```bash
certoraRun certora/confs/yield_ctf/high_level.conf --rule splitMergePreservesCollateral
certoraRun certora/confs/yield_ctf/high_level.conf --rule splitThenMergePositionsRestored
certoraRun certora/confs/yield_ctf/high_level.conf --rule splitDoesNotAffectOtherUsersPositions
certoraRun certora/confs/yield_ctf/high_level.conf --rule mergeDoesNotAffectOtherUsersPositions
certoraRun certora/confs/yield_ctf/high_level.conf --rule redeemDoesNotAffectOtherUsers
certoraRun certora/confs/yield_ctf/high_level.conf --rule redeemBurnsCallerTokens
certoraRun certora/confs/yield_ctf/high_level.conf --rule redeemDoesNotIncreaseCollateral
```

### Running Mutation Testing

Run the mutation test suite for `depositedAmountEqualsPositions` (VS-11):

```bash
certoraMutate certora/confs/yield_ctf/valid_state_mutations_depositedAmountEqualsPositions.conf
```

#### Creating Mutations for Other Invariants

1. **Create the mutations directory**: `mkdir -p certora/mutations/<invariantName>`
2. **Introduce a mutation** into the source contract (e.g., comment out a line, flip a condition)
3. **Run the helper script** to snapshot the mutation:
   ```bash
   ./certora/mutations/add_mutation.sh <invariantName> contracts/YieldBearing/YieldBearingConditionalTokens.sol
   ```
   This auto-numbers the mutation file (e.g., `1.sol`, `2.sol`, ...) and embeds the diff block. The original source is automatically restored via `git restore`.
4. **Create a conf file** (copy an existing one and add the `mutations` section pointing to your new directory)
5. **Run** `certoraMutate` with your new conf file

### Source Code Modifications

The Certora Prover [overapproximates](https://docs.certora.com/en/latest/docs/user-guide/glossary.html#term-overapproximation) bitwise operations by default, which produces spurious counterexamples or causes the prover to crash. To work around this, all inline bitwise expressions in `splitPosition`, `mergePositions`, and `redeemPositions` were extracted into internal helper functions. These functions are then replaced with precise CVL summaries defined in [`bitops.spec`](./specs/setup/bitops.spec).

Each replacement site in the Solidity source is marked with an `// ORIGINAL:` comment showing the original expression, making it straightforward to audit the modifications.

#### Helper Functions

The following internal functions were added to the `BitOps Certora Internal Functions` block in `YieldBearingConditionalTokens.sol`. Extracting bitwise expressions into separate functions allows them to be summarized with precise CVL implementations in [`bitops.spec`](./specs/setup/bitops.spec).

#### Replacement Sites

All 10 replacement sites across the 3 functions:

`splitPosition`

```solidity
// ORIGINAL: uint fullIndexSet = (1 << outcomeSlotCount) - 1;
uint fullIndexSet = _createFullIndexSet(outcomeSlotCount);

// ORIGINAL: require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
require(_isSubset(indexSet, freeIndexSet), "partition not disjoint");

// ORIGINAL: freeIndexSet ^= indexSet;
freeIndexSet = _removeBits(freeIndexSet, indexSet);

// ORIGINAL: CTHelpers.getCollectionId(parentCollectionId, conditionId, fullIndexSet ^ freeIndexSet)
CTHelpers.getCollectionId(parentCollectionId, conditionId, _complement(fullIndexSet, freeIndexSet))
```

`mergePositions`

```solidity
// ORIGINAL: uint fullIndexSet = (1 << outcomeSlotCount) - 1;
uint fullIndexSet = _createFullIndexSet(outcomeSlotCount);

// ORIGINAL: require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
require(_isSubset(indexSet, freeIndexSet), "partition not disjoint");

// ORIGINAL: freeIndexSet ^= indexSet;
freeIndexSet = _removeBits(freeIndexSet, indexSet);

// ORIGINAL: CTHelpers.getCollectionId(parentCollectionId, conditionId, fullIndexSet ^ freeIndexSet)
CTHelpers.getCollectionId(parentCollectionId, conditionId, _complement(fullIndexSet, freeIndexSet))
```

`redeemPositions`

```solidity
// ORIGINAL: uint fullIndexSet = (1 << outcomeSlotCount) - 1;
uint fullIndexSet = _createFullIndexSet(outcomeSlotCount);

// ORIGINAL: if (indexSet & (1 << j) != 0) {
if (_isBitSet(indexSet, j)) {
```

