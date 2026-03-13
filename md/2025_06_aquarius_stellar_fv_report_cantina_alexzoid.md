# Formal Verification Report: Aquarius Fees Collector

- Competition: [Aquarius Audit + Certora Formal Verification](https://cantina.xyz/competitions/990ce947-05da-443e-b397-be38a65f0bff)
- Repository: https://github.com/alexzoid-eth/aquarius-cantina-fv
- Scope: [fees_collector](https://github.com/alexzoid-eth/aquarius-cantina-fv/tree/main/fees_collector), [access_control](https://github.com/alexzoid-eth/aquarius-cantina-fv/tree/main/access_control)
- Date: June 2025
- Author: [@alexzoid](https://x.com/alexzoid)
- Certora Sunbeam (Soroban) Prover version: 7.26.0

---

## About Aquarius Protocol

Aquarius is a decentralized protocol governed by DAO voting with AQUA tokens, built on the Stellar blockchain. It provides automated market maker (AMM) functionality through Soroban smart contracts, enabling liquidity provision, token swaps, and fee collection across multiple pool types including stableswap and standard liquidity pools. The protocol features role-based access control with time-delayed administrative operations and emergency mode capabilities.

## About the Fees Collector Module

The fees collector module manages protocol fee collection and distribution with a comprehensive role-based access control system. It implements a multi-role administration model with six distinct roles (Admin, Emergency Admin, Rewards Admin, Operations Admin, Pause Admin, and Emergency Pause Admin). Administrative changes to critical roles (Admin and Emergency Admin) require a two-step, time-delayed transfer process: commit followed by apply after a mandatory delay period. The module also supports upgradeable contracts with similar time-delayed commit-apply patterns, and includes an emergency mode that allows immediate upgrades when activated by the Emergency Admin.

## Competition Scope

For this formal verification competition, the following crates are in scope:

- [`fees_collector/src/contract.rs`](https://github.com/alexzoid-eth/aquarius-cantina-fv/blob/main/fees_collector/src/contract.rs) - Main contract entry points for fee collection and administration
- [`fees_collector/src/interface.rs`](https://github.com/alexzoid-eth/aquarius-cantina-fv/blob/main/fees_collector/src/interface.rs) - Contract interface definitions
- [`fees_collector/src/storage.rs`](https://github.com/alexzoid-eth/aquarius-cantina-fv/blob/main/fees_collector/src/storage.rs) - Storage helpers
- [`access_control/src/emergency.rs`](https://github.com/alexzoid-eth/aquarius-cantina-fv/blob/main/access_control/src/emergency.rs) - Emergency mode management
- [`access_control/src/management.rs`](https://github.com/alexzoid-eth/aquarius-cantina-fv/blob/main/access_control/src/management.rs) - Role management operations
- [`access_control/src/storage.rs`](https://github.com/alexzoid-eth/aquarius-cantina-fv/blob/main/access_control/src/storage.rs) - Storage key mappings for roles
- [`access_control/src/transfer.rs`](https://github.com/alexzoid-eth/aquarius-cantina-fv/blob/main/access_control/src/transfer.rs) - Ownership transfer with time delays
- [`upgrade/src/lib.rs`](https://github.com/alexzoid-eth/aquarius-cantina-fv/blob/main/upgrade/src/lib.rs) - Contract upgrade logic

---

# Table of Contents
- [Formal Verification Methodology](#formal-verification-methodology)
  - [Types of Properties](#types-of-properties)
    - [Parametric Rules](#parametric-rules)
    - [Rules](#rules)
  - [Verification Process](#verification-process)
    - [Setup](#setup)
    - [Crafting Properties](#crafting-properties)
  - [Assumptions](#assumptions)
    - [Safe Assumptions](#safe-assumptions)
    - [Unsafe Assumptions](#unsafe-assumptions)

- [Verification Properties](#verification-properties)
  - [Sanity](#sanity)
  - [State Transitions](#state-transitions)
  - [Permissions](#permissions)
  - [Integrity](#integrity)
  - [High Level](#high-level)

- [Manual Mutations Testing](#manual-mutations-testing)
  - [Access Control - Emergency](#access-control---emergency)
  - [Access Control - Management](#access-control---management)
  - [Access Control - Storage](#access-control---storage)
  - [Access Control - Transfer](#access-control---transfer)
  - [Fees Collector - Contract](#fees-collector---contract)
  - [Upgrade - Library](#upgrade---library)

- [Violated Properties](#violated-properties)
  - [Issue #1 - Upgrade Deadline Lifecycle](#issue-1---upgrade-deadline-lifecycle)
  - [Issue #2 - Admin Deadline Clear](#issue-2---admin-deadline-clear)

- [Setup and Execution Instructions](#setup-and-execution-instructions)
  - [Certora Prover Installation](#certora-prover-installation)
  - [Verification Execution](#verification-execution)

---

## Formal Verification Methodology

Certora Formal Verification (FV) provides mathematical proofs of smart contract correctness by verifying code against a formal specification. It complements techniques like testing and fuzzing, which can only sometimes detect bugs based on predefined properties. In contrast, Certora FV examines all possible states and execution paths in a contract.

Simply put, the formal verification process involves crafting properties (similar to writing tests) in native RUST language and submitting them alongside compiled programs to a remote prover. This prover essentially transforms the program bytecode and rules into a mathematical model and determines the validity of rules.

### Types of Properties

When constructing properties in formal verification, we mainly deal with two types: **Parametric Rules** and **Rules**.

#### Parametric Rules
- Properties verified across ALL external functions automatically. Implemented with the parametric infrastructure that generates one property check per external function.
- Process:
  1. Initialize ghost storage from rule parameters.
  2. Assume realistic timestamp.
  3. Log all storage variables.
  4. Execute external function.
  5. Log all storage variables again.
  6. Assert the property holds.
- Example: "Admin roles can only change through the time-delayed transfer process."
- Use Case: Ensures **State Transition** and **Permission** properties - critical invariants that MUST never be violated across all functions.

#### Rules
- Flexible checks for specific behaviors or conditions.
- Structure:
  1. Setup: Initialize contract state (e.g., "set admin role").
  2. Execution: Simulate contract behavior by calling external functions.
  3. Verification:
     - Use `cvlr_assert!()` to check if a condition is **always true** (e.g., "emergency mode toggle works correctly").
     - Use `cvlr_satisfy!()` to verify a condition is **reachable** (e.g., "immediate upgrade is reachable in emergency mode").
- Example: "A committed upgrade can be reverted, clearing the deadline."
- Use Case: Verifies **Integrity** and **High Level** properties, from simple getter/setter consistency to complex multi-step workflows.

### Verification Process
The process is divided into two stages: **Setup** and **Crafting Properties**.

#### Setup
This stage prepares the contract and prover for verification. Use conditional source code compilation with `#[cfg(feature = "certora")]`.
- Mirror storage read/write operations into ghost state variables via the `ghost_state` crate to reduce complexity.
- Create harness functions in `access_control_harness.rs` and `upgrade_harness.rs` to expose internal library functions for verification.
- Prove parametric properties as a foundation for mutation testing.

#### Crafting Properties
This stage defines and implements the properties:
- Write properties in **plain English** for clarity.
- Categorize properties by purpose. Parametric style (`sanity.rs`, `state_transition.rs`, `permissions.rs`) properties are verified across all external functions. Non-parametric (`integrity.rs`, `high_level.rs`) as regular rules targeting specific behaviors.
- Use ghost state mirroring for efficient storage tracking across function calls.

### Assumptions

Assumptions simplify the verification process and are classified as **Safe** or **Unsafe**. Safe assumptions are backed by valid logic or required by the environment. Unsafe assumptions are made to reduce complexity, potentially limiting coverage.

#### Safe Assumptions
- Block timestamps are always non-zero (`e.ledger().timestamp() > 0`)
- Ghost state is initialized from actual persistent storage before each verification

#### Unsafe Assumptions

##### Loop Unrolling
- Vector iterations limited to 2 iterations (`loop_iter = 2` in configs)

##### Optimistic Loop
- Optimistic loop handling enabled (`optimistic_loop = true` in configs)

---

## Verification Properties

The verification properties are categorized into the following types:

1. **Sanity (SA)**: Basic reachability of function execution paths
2. **State Transitions (ST)**: Time-delayed transitions for roles and upgrades
3. **Permissions (PER)**: Role-based access control for state changes
4. **Integrity (INT)**: Getter/setter consistency and storage key mappings
5. **High Level (HL)**: Multi-step workflows and emergency mode behaviors

Each job status linked to a corresponding run in the dashboard with a specific status:

- ✅ completed successfully
- ⚠️ reached global timeout
- ❌ violated

### Sanity

Basic checks ensuring contract functions remain accessible and operational.

All sanity properties are stored in [sanity.rs](src/certora_specs/sanity.rs) and [⚠️ passed verification](https://prover.certora.com/output/7749274/935201ea7ed6463f8c9503145b540a50?anonymousKey=3c77f80063f4f08b8f026c438deb20c143d4a512) (with `sanity_apply_transfer_ownership` reaching global timeout).

| Source | Rule | Description | Caught mutations |
|--------|------|-------------|------------------|
| SA-01 | sanity | All external functions remain callable | - |

### State Transitions

These properties verify that state changes occur correctly during contract operations, ensuring time-delayed transitions for critical role and upgrade operations.

All state transition properties are stored in [state_transition.rs](src/certora_specs/state_transition.rs) and passed verification across multiple runs (links per property below).

| Source | Rule | Description | Prover | Caught mutations |
|--------|------|-------------|--------|------------------|
| ST-01 | state_transition_admin_role | Admin role can only change through time-delayed transfer process | [✅](https://prover.certora.com/output/7749274/1a72d2120f0344e29c1ff24443a7786d?anonymousKey=abf9ebd6657d336ac2ee4692ce03e76fd94e89a0) [✅](https://prover.certora.com/output/7749274/39af0d12b1f74dadb919769f53bf73ef?anonymousKey=04df1ee3773650275ded813059b938dc5814c590) | [❌](https://prover.certora.com/output/7749274/75182fc17ea94fb789d8ca13c583269d?anonymousKey=d6746cf364031822387e5f20e26cfa06313b4fe1)[management_0](mutations/access_control/management/management_0.rs) |
| ST-02 | state_transition_emergency_admin_role | Emergency admin role can only change through time-delayed transfer process | [✅](https://prover.certora.com/output/7749274/5060b622906843f38278fcca19d0f43d?anonymousKey=1dcf8c3f79e805b5aa6d729294628b2bc6d32682) [✅](https://prover.certora.com/output/7749274/ae4bd72c16124488a00f4c944313b36c?anonymousKey=0b0879cc8b365a6b04f126995ea8acd526e4d77c) | [❌](https://prover.certora.com/output/7749274/68c114b1246d4000b897522b5499a631?anonymousKey=7e228f18f6367c1533daf017ab0efcff23ee822e)[management_0](mutations/access_control/management/management_0.rs), [❌](https://prover.certora.com/output/7749274/90c4bcb913784db79d9a75fff7bf0d8e?anonymousKey=cec707fe9ca0ca1cac581461dd5a990c302194ec)[transfer_4](mutations/access_control/transfer/transfer_4.rs) |
| ST-03 | state_transition_admin_init_once | Initial admin can only be set once | [✅](https://prover.certora.com/output/7749274/58dad281a5554407a0181be4b6e556b0?anonymousKey=854492adbd9ffec8d69e9fc9bab6d54ce0d885e2) [✅](https://prover.certora.com/output/7749274/f3c55eaa1e6d4fa8926d93ea487f4eb4?anonymousKey=50c0f7960fec0eb28192b04b74bf16478121b9aa) | [❌](https://prover.certora.com/output/7749274/3841a0e0093b4677bf95d1a79b5b803f?anonymousKey=feb73f6c344019fc751348b3c8fecab31ac26c51)[management_0](mutations/access_control/management/management_0.rs) |
| ST-04 | state_transition_admin_never_zero | Roles can never be removed once set | [✅](https://prover.certora.com/output/7749274/616b1a5de7f44bfa8d9ce56c4baf12e1?anonymousKey=1ed5eeb86ec4cba8f8e956384044b7f342240ae8) | - |
| ST-05 | state_transition_admin_deadline_lifecycle | Admin deadline transitions: 0→non-zero (commit) and non-zero→0 (apply/revert) | [✅](https://prover.certora.com/output/7749274/afbc167421da419c8211965775fb20e8?anonymousKey=b421c97138ce2c2f13db2baf5d58758e1e350a0a) [✅](https://prover.certora.com/output/7749274/c2b69b414a9740819cc9b2f3d62a8cb7?anonymousKey=8dce4331e1d5fb502668c635f7005d0a144dda1a) [✅](https://prover.certora.com/output/7749274/87bdf589092243da854101c040dcf210?anonymousKey=75678d33b3936b7c9488b22b838f51d839f03c7a) | [❌](https://prover.certora.com/output/7749274/ed51b583f8484cf6aaadd51cf7c9af77?anonymousKey=a48b4faa759edaf150715115263f551d21c07ad2)[transfer_5](mutations/access_control/transfer/transfer_5.rs) |
| ST-06 | state_transition_admin_deadline_clear | Future admin cleared when deadline goes to 0 | ❌ [Issue #2](#issue-2---admin-deadline-clear) | - |
| ST-07 | state_transition_em_admin_deadline_lifecycle | Emergency admin deadline lifecycle transitions | [✅](https://prover.certora.com/output/7749274/54414a49d9ed47eb9f88e0eb42b3ecf5?anonymousKey=ab68785322f5a4e4f760b0e45520e1239f83ca1b) [✅](https://prover.certora.com/output/7749274/1c83b1eb12764e069818bc8c7bbb2370?anonymousKey=70f7f4b6cdfe203d36a291aaf07407a9471f91bf) | - |
| ST-08 | state_transition_em_admin_deadline_clear | Future emergency admin cleared when deadline goes to 0 | ❌ [Issue #2](#issue-2---admin-deadline-clear) | - |
| ST-09 | state_transition_future_admin_consistency | If deadline doesn't change, future admin must remain same | [✅](https://prover.certora.com/output/7749274/f64c0c11b83d41e9b96711e4717c0f5f?anonymousKey=5cdbf920c9e9b6d54c5093b5306df8959da56df1) | [❌](https://prover.certora.com/output/7749274/01ad2ca874ec42e3bdd36f3b24b6c7f4?anonymousKey=48b9092c5eb73272ce12f0d9909e27e32fe992e2)[transfer_5](mutations/access_control/transfer/transfer_5.rs) |
| ST-10 | state_transition_future_em_admin_consistency | If deadline doesn't change, future emergency admin must remain same | [✅](https://prover.certora.com/output/7749274/614d338ad435444db3abf87f6e764c7a?anonymousKey=281c5c5831a6c312bb3a22854c9adb506a3e8a35) | [❌](https://prover.certora.com/output/7749274/dbbfb79761e843cfbd7fb4f9fef663f5?anonymousKey=f82428c73ae03ccae8d9a68919af006d2e397888)[transfer_5](mutations/access_control/transfer/transfer_5.rs) |
| ST-11 | state_transition_emergency_mode_behavior | Emergency mode allows immediate upgrades but not immediate transfers | [✅](https://prover.certora.com/output/7749274/d6c64cde635843eebe982d33fc72b689?anonymousKey=ba296699ebd6c57a0a8f13ad4060ad404d62c2f7) | - |
| ST-12 | state_transition_upgrade_deadline_lifecycle | Upgrade deadline transitions: 0→non-zero (commit) and non-zero→0 (apply/revert) | [✅](https://prover.certora.com/output/7749274/54c77957988f495a9c9918c5a9937902?anonymousKey=595070b8053d8aab84f597c5a5fb0b81e4786397) ❌ [Issue #1](#issue-1---upgrade-deadline-lifecycle) | - |
| ST-13 | state_transition_future_wasm_consistency | If deadline doesn't change, future wasm must remain same | [✅](https://prover.certora.com/output/7749274/6ea3579129c94d03b508174907c57955?anonymousKey=13d1cb585d124886288fb823ac01b9af50c5fb0b) | - |

### Permissions

These properties verify that only authorized roles can modify specific state variables.

All permission properties are stored in [permissions.rs](src/certora_specs/permissions.rs) and passed verification across multiple runs.

| Source | Rule | Description | Prover | Caught mutations |
|--------|------|-------------|--------|------------------|
| PER-01 | permissions_admin_role_changes | Only current admin can change admin role | [✅](https://prover.certora.com/output/7749274/b9f0d90903e3493db3dc9ee11fc72118?anonymousKey=41308dc4e7f8b4cec2eaaa3d16af9295ce112990) | - |
| PER-02 | permissions_other_admins_role_changes | Only admin can change other admin roles | [✅](https://prover.certora.com/output/7749274/b9f0d90903e3493db3dc9ee11fc72118?anonymousKey=41308dc4e7f8b4cec2eaaa3d16af9295ce112990) | - |
| PER-03 | permissions_admin_transfer_deadline_changes | Only admin can change admin transfer deadlines | [✅](https://prover.certora.com/output/7749274/4a65b16dca874cda807032eec88388ca?anonymousKey=a943bfda33a96899865b3b168dfc3d955e875989) [✅](https://prover.certora.com/output/7749274/0a5e63f383874437a3fc643669f1cf74?anonymousKey=9392d7c990043b3b5a09c63700445e88c8366632) | [❌](https://prover.certora.com/output/7749274/7b23b306ddd74c61ac3305c48a342eb3?anonymousKey=82409b3302211fc1ae2249ce33e35e8def812753)[contract_3](mutations/fees_collector/contract/contract_3.rs), [❌](https://prover.certora.com/output/7749274/e98bbb1256974e63a2a5d168ec38c3cb?anonymousKey=10f01c80490ce7a3d1cd871ee4a86fe84b3a93b2)[contract_5](mutations/fees_collector/contract/contract_5.rs) |
| PER-04 | permissions_em_admin_transfer_deadline_changes | Only admin can change emergency admin transfer deadlines | [⚠️](https://prover.certora.com/output/7749274/21162d47b14348e6b8eddc803d496cb3?anonymousKey=f36f324bee087db116a91192170a05053cb3f3d4) | [❌](https://prover.certora.com/output/7749274/5b265f2ff9d14251a246b43343006d9f?anonymousKey=b48ddaa164aa9a3907d8f5b6c4c80f392bf1cfe7)[contract_3](mutations/fees_collector/contract/contract_3.rs), [❌](https://prover.certora.com/output/7749274/96ccf224b69d47a18992fa11b1b4bb1a?anonymousKey=22c11a9159380301d380ce58061c5f4505a07033)[contract_6](mutations/fees_collector/contract/contract_6.rs) |
| PER-05 | permissions_future_admin_changes | Only admin can change future admin; consistent with deadline transitions | [✅](https://prover.certora.com/output/7749274/6fb9a90f80264399964e0d2a6a64075f?anonymousKey=bf91a54d9cdaed356a97e93fd8b46e562ed79835) [✅](https://prover.certora.com/output/7749274/a49a75f6148a44d38456660e27e41410?anonymousKey=aa1aba31f3d89a776b740295209b5820c5a0b817) | [❌](https://prover.certora.com/output/7749274/78f0841322fb44ca8aafe06abde4e411?anonymousKey=16ce8ffe5be3486eb36aca218ef950fda5fde070)[contract_5](mutations/fees_collector/contract/contract_5.rs) |
| PER-06 | permissions_future_em_admin_changes | Only admin can change future emergency admin; consistent with deadline transitions | [✅](https://prover.certora.com/output/7749274/c8f8dfab573a4933a74d2ecc7f524ea2?anonymousKey=d08f677828bdbe24ec70bef1cb118bc13dce0d89) [✅](https://prover.certora.com/output/7749274/5b30616bada9404a86b9d9f92609d6aa?anonymousKey=f8bde2f455aed905b4bbfc60ea90257c18e05dd9) | [❌](https://prover.certora.com/output/7749274/a0db3b896a5a4955a2cd731663ba4084?anonymousKey=d92f7bc26a91ddfbb4d810e6d08ac97f0a7d9098)[contract_5](mutations/fees_collector/contract/contract_5.rs) |
| PER-07 | permissions_emergency_mode_changes | Only emergency admin can change emergency mode | [✅](https://prover.certora.com/output/7749274/f4582b2d82ca4d1c8f634616a3ea1179?anonymousKey=5dc1cfd0ba472b03d5bf1e00824ef1358042dd47) | - |
| PER-08 | permissions_upgrade_deadline_changes | Only admin can change upgrade deadlines | [✅](https://prover.certora.com/output/7749274/f4582b2d82ca4d1c8f634616a3ea1179?anonymousKey=5dc1cfd0ba472b03d5bf1e00824ef1358042dd47) | [❌](https://prover.certora.com/output/7749274/7261a86b49d7452e9ef834d2f837eb89?anonymousKey=c6f50d183b83a6ac73865e31e77bf9f5cf0bd9bd)[contract_0](mutations/fees_collector/contract/contract_0.rs), [❌](https://prover.certora.com/output/7749274/ecdad43c17fc44d2b466f677c1fd1134?anonymousKey=2353e6f923c8a29fc0d134111d32304f7745ab91)[contract_3](mutations/fees_collector/contract/contract_3.rs) |
| PER-09 | permissions_future_wasm_changes | Only admin can change future wasm; consistent with deadline transitions | [✅](https://prover.certora.com/output/7749274/f4582b2d82ca4d1c8f634616a3ea1179?anonymousKey=5dc1cfd0ba472b03d5bf1e00824ef1358042dd47) | [❌](https://prover.certora.com/output/7749274/7261a86b49d7452e9ef834d2f837eb89?anonymousKey=c6f50d183b83a6ac73865e31e77bf9f5cf0bd9bd)[contract_0](mutations/fees_collector/contract/contract_0.rs) |

### Integrity

Properties ensuring data consistency and correctness throughout operations.

All integrity properties are stored in [integrity.rs](src/certora_specs/integrity.rs) and passed verification: [✅ main](https://prover.certora.com/output/7749274/be638612e3a847cdb949622c96e87f4f?anonymousKey=3fae0c4fb2378bb0fe7d667e44ddcdfd094c9982), [✅ address_has_role](https://prover.certora.com/output/7749274/9c89b85d0be7486189e6db8394670553?anonymousKey=bde38bc6578879c06d747de0d4b77ffc800e8c91), [✅ role_address_safe](https://prover.certora.com/output/7749274/0f6f2dc36c2b45af94cf734d3c30c51f?anonymousKey=942d7188077065d3e43f946fd86d3be7098e1bf8), [✅ role_address_single](https://prover.certora.com/output/7749274/9edc3c21836f47ea91566024360b02f2?anonymousKey=c29e230cfc5d0883b110b51b617b765b20740b82) successfully.

#### Emergency Mode Integrity

| Source | Rule | Description | Caught mutations |
|--------|------|-------------|------------------|
| INT-01 | integrity_emergency_mode | Setting emergency mode correctly stores the value | [❌](https://prover.certora.com/jobStatus/7749274/b8df9855008748bf82cb9f1317c47c28?anonymousKey=946c0f4b224684e4b620c704855663387148f5e7)[emergency_0](mutations/access_control/emergency/emergency_0.rs), [❌](https://prover.certora.com/jobStatus/7749274/8763492e1a004c67ab575c6463f95377?anonymousKey=1f0627171dd16ddabcccdf5c8cd5fba0578ac142)[contract_4](mutations/fees_collector/contract/contract_4.rs), [❌](https://prover.certora.com/jobStatus/7749274/c0ef460b23f247f3814d73c482965584?anonymousKey=2f10e16e3f2d1da1ff404fdcbf8dcadf44854633)[contract_2](mutations/fees_collector/contract/contract_2.rs) |

#### Admin and Role Integrity

| Source | Rule | Description | Caught mutations |
|--------|------|-------------|------------------|
| INT-02 | integrity_init_admin | Initial admin address is correctly stored | - |
| INT-03 | integrity_future_address | Future address set by commit_transfer_ownership is stored correctly | [❌](https://prover.certora.com/jobStatus/7749274/3736d8020d7244119ce686986ccf41ae?anonymousKey=03eb923ba87c6a9da24aadc07fb977c0f3f8e1a5)[storage_2](mutations/access_control/storage/storage_2.rs), [❌](https://prover.certora.com/jobStatus/7749274/72b8a9624cc044738686ee96c1aa18a3?anonymousKey=f522c806d9a16bc2d5b085fc0753d7e1f70224ad)[transfer_1](mutations/access_control/transfer/transfer_1.rs), [❌](https://prover.certora.com/jobStatus/7749274/7575a8f6c45f4d7897611ee8c73bd232?anonymousKey=132df700e9dd533bb6260bb5d92ec0e26e4f39c9)[transfer_5](mutations/access_control/transfer/transfer_5.rs) |
| INT-04 | integrity_commit_transfer_deadline | Commit sets deadline to timestamp + ADMIN_ACTIONS_DELAY | [❌](https://prover.certora.com/jobStatus/7749274/3736d8020d7244119ce686986ccf41ae?anonymousKey=03eb923ba87c6a9da24aadc07fb977c0f3f8e1a5)[storage_2](mutations/access_control/storage/storage_2.rs), [❌](https://prover.certora.com/jobStatus/7749274/72b8a9624cc044738686ee96c1aa18a3?anonymousKey=f522c806d9a16bc2d5b085fc0753d7e1f70224ad)[transfer_1](mutations/access_control/transfer/transfer_1.rs), [❌](https://prover.certora.com/jobStatus/7749274/7575a8f6c45f4d7897611ee8c73bd232?anonymousKey=132df700e9dd533bb6260bb5d92ec0e26e4f39c9)[transfer_5](mutations/access_control/transfer/transfer_5.rs) |
| INT-05 | integrity_role_address_single | Single role address set and retrieved correctly | [❌](https://prover.certora.com/output/7749274/84fd014af2a34170b38161be0de08c28?anonymousKey=1f9b0adef6ec514917bf61618428bc08f3ff790c)[management_1](mutations/access_control/management/management_1.rs) |
| INT-06 | integrity_role_addresses_multiple | Multiple role addresses (vector) set and retrieved correctly | - |
| INT-07 | integrity_role_address_safe | Safe role address getter returns correct value | [❌](https://prover.certora.com/output/7749274/3962ffc322fc489282d6f9271dfd429d?anonymousKey=22dac610e546c64c23af7f8ccacabbda2853295f)[management_1](mutations/access_control/management/management_1.rs) |
| INT-08 | integrity_init_all_roles | All roles initialized correctly | [❌](https://prover.certora.com/jobStatus/7749274/ab8f1973a022499ebf5375a1513dca34?anonymousKey=2bba36489922d9018541b2ba43f66a41fd3f71e7)[storage_0](mutations/access_control/storage/storage_0.rs) |
| INT-09 | integrity_privileged_addresses | Privileged addresses set and retrieved correctly | [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd)[management_1](mutations/access_control/management/management_1.rs), [❌](https://prover.certora.com/jobStatus/7749274/ab8f1973a022499ebf5375a1513dca34?anonymousKey=2bba36489922d9018541b2ba43f66a41fd3f71e7)[storage_0](mutations/access_control/storage/storage_0.rs) |
| INT-10 | integrity_address_has_role | address_has_role returns true when address has role | [❌](https://prover.certora.com/output/7749274/b776f390107841069db26309f840dd6e?anonymousKey=2e6cc79433dcff96e6d467f3f70ee96043c3f7ee)[management_1](mutations/access_control/management/management_1.rs) |
| INT-11 | integrity_assert_address_has_role | assert_address_has_role doesn't panic when address has role | - |
| INT-12 | integrity_assert_address_has_role_negative | assert_address_has_role panics when address doesn't have role | [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd)[management_1](mutations/access_control/management/management_1.rs) |

#### Upgrade Integrity

| Source | Rule | Description | Caught mutations |
|--------|------|-------------|------------------|
| INT-13 | integrity_commit_upgrade_future_wasm | Commit upgrade correctly stores future wasm hash | [❌](https://prover.certora.com/jobStatus/7749274/671c16a65c4144f692bc0114bd549969?anonymousKey=1a3dd0d5c8f119462c25a1f9395906c2c0b84d1f)[contract_7](mutations/fees_collector/contract/contract_7.rs) |
| INT-14 | integrity_commit_upgrade_deadline | Commit upgrade sets deadline to timestamp + ADMIN_ACTIONS_DELAY | [❌](https://prover.certora.com/jobStatus/7749274/671c16a65c4144f692bc0114bd549969?anonymousKey=1a3dd0d5c8f119462c25a1f9395906c2c0b84d1f)[contract_7](mutations/fees_collector/contract/contract_7.rs), [❌](https://prover.certora.com/jobStatus/7749274/0fd2ef56b4c54571a20a15cab5b9775b?anonymousKey=5b1c9cd7d823fb6584d7e4b399b844e811f6ac9a)[lib_1](mutations/upgrade/lib/lib_1.rs), [❌](https://prover.certora.com/jobStatus/7749274/8b7b801b98544e9d8d3de7468619daa6?anonymousKey=60ff68e03fc8ef0c46695c8a668a301f4d5f5a11)[lib_0](mutations/upgrade/lib/lib_0.rs) |
| INT-15 | integrity_upgrade_deadline_direct | Upgrade deadline put and get are consistent | - |
| INT-16 | integrity_future_wasm_direct | Future wasm put and get are consistent | - |

#### Storage Key Mapping Integrity

| Source | Rule | Description | Caught mutations |
|--------|------|-------------|------------------|
| INT-17 | integrity_get_key_admin | Storage key mapping for Admin role (1 → 1) | - |
| INT-18 | integrity_get_key_emergency_admin | Storage key mapping for EmergencyAdmin role (2 → 2) | - |
| INT-19 | integrity_get_key_rewards_admin | Storage key mapping for RewardsAdmin role (3 → 3) | [❌](https://prover.certora.com/jobStatus/7749274/ab8f1973a022499ebf5375a1513dca34?anonymousKey=2bba36489922d9018541b2ba43f66a41fd3f71e7)[storage_0](mutations/access_control/storage/storage_0.rs) |
| INT-20 | integrity_get_key_operations_admin | Storage key mapping for OperationsAdmin role (4 → 4) | - |
| INT-21 | integrity_get_key_pause_admin | Storage key mapping for PauseAdmin role (5 → 5) | - |
| INT-22 | integrity_get_key_emergency_pause_admin | Storage key mapping for EmergencyPauseAdmin role (6 → 6) | - |
| INT-23 | integrity_get_future_key_admin | Future storage key mapping for Admin role (1 → 7) | - |
| INT-24 | integrity_get_future_key_emergency_admin | Future storage key mapping for EmergencyAdmin role (2 → 8) | [❌](https://prover.certora.com/jobStatus/7749274/64900911e0764ae4a35dc0688be10bd2?anonymousKey=1ae981872bc7b820750154588bd9de6d933e7366)[storage_1](mutations/access_control/storage/storage_1.rs) |
| INT-25 | integrity_get_future_deadline_key_admin | Future deadline key mapping for Admin role (1 → 9) | [❌](https://prover.certora.com/jobStatus/7749274/3736d8020d7244119ce686986ccf41ae?anonymousKey=03eb923ba87c6a9da24aadc07fb977c0f3f8e1a5)[storage_2](mutations/access_control/storage/storage_2.rs) |
| INT-26 | integrity_get_future_deadline_key_emergency_admin | Future deadline key mapping for EmergencyAdmin role (2 → 10) | - |

#### Role Requirement Integrity

| Source | Rule | Description | Caught mutations |
|--------|------|-------------|------------------|
| INT-27 | integrity_require_rewards_admin_with_rewards_admin | require_rewards_admin_or_owner accepts rewards_admin | - |
| INT-28 | integrity_require_rewards_admin_with_admin | require_rewards_admin_or_owner accepts admin | - |
| INT-29 | integrity_require_rewards_admin_negative | require_rewards_admin_or_owner rejects unauthorized | [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd)[management_1](mutations/access_control/management/management_1.rs) |
| INT-30 | integrity_require_operations_admin_with_operations_admin | require_operations_admin_owner accepts operations_admin | - |
| INT-31 | integrity_require_operations_admin_with_admin | require_operations_admin_owner accepts admin | - |
| INT-32 | integrity_require_operations_admin_negative | require_operations_admin_owner rejects unauthorized | [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd)[management_1](mutations/access_control/management/management_1.rs) |
| INT-33 | integrity_require_pause_emergency_with_pause_admin | require_pause_emergency_admin accepts pause_admin | - |
| INT-34 | integrity_require_pause_emergency_with_emergency_pause_admin | require_pause_emergency_admin accepts emergency_pause_admin | - |
| INT-35 | integrity_require_pause_emergency_with_admin | require_pause_emergency_admin accepts admin | - |
| INT-36 | integrity_require_pause_admin_with_pause_admin | require_pause_admin_or_owner accepts pause_admin | - |
| INT-37 | integrity_require_pause_admin_with_admin | require_pause_admin_or_owner accepts admin | - |
| INT-38 | integrity_require_pause_admin_negative | require_pause_admin_or_owner rejects unauthorized | [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd)[management_1](mutations/access_control/management/management_1.rs) |

### High Level

Complex business logic and protocol-specific properties.

All high-level properties are stored in [high_level.rs](src/certora_specs/high_level.rs) and [✅ passed verification](https://prover.certora.com/output/7749274/fae72ac3587f4bf0afe1e6b7a99c2be3?anonymousKey=533dae6c370b84fbe0b387ae6d72329b965d6027) successfully.

| Source | Rule | Description | Caught mutations |
|--------|------|-------------|------------------|
| HL-01 | high_level_revert_ownership_transfer | Ownership transfer can be reverted before deadline, leaving admin unchanged | - |
| HL-02 | high_level_upgrade_immediate_in_emergency | Upgrades can be applied immediately only in emergency mode | [❌](https://prover.certora.com/output/7749274/06b66b02ea064146a8694b19c35ff3ce?anonymousKey=74b8a3cc4c36608318eec5e1458f812a85e0d059)[contract_7](mutations/fees_collector/contract/contract_7.rs) |
| HL-03 | high_level_upgrade_reachable | Reachability: immediate upgrade path is reachable in emergency mode | [❌](https://prover.certora.com/output/7749274/51781f505db246ca80eb2dc7b7a9a7a8?anonymousKey=5d4a75fca9bde39d4680d50458c251a7aac75e50)[management_1](mutations/access_control/management/management_1.rs) |
| HL-04 | high_level_revert_upgrade | Upgrade can be reverted, clearing the deadline | [❌](https://prover.certora.com/output/7749274/9dcb2dd41c7d4b02bea27ae778251ff4?anonymousKey=84baa3164f9a1ae0c395425b297ed65a8b85ac7a)[contract_8](mutations/fees_collector/contract/contract_8.rs) |
| HL-05 | high_level_revert_upgrade_reachable | Reachability: upgrade revert path is reachable | [❌](https://prover.certora.com/output/7749274/51781f505db246ca80eb2dc7b7a9a7a8?anonymousKey=5d4a75fca9bde39d4680d50458c251a7aac75e50)[management_1](mutations/access_control/management/management_1.rs), [❌](https://prover.certora.com/output/7749274/9dcb2dd41c7d4b02bea27ae778251ff4?anonymousKey=84baa3164f9a1ae0c395425b297ed65a8b85ac7a)[contract_8](mutations/fees_collector/contract/contract_8.rs) |
| HL-06 | high_level_emergency_mode_toggle | Emergency mode can be toggled on and off by emergency admin | [❌](https://prover.certora.com/output/7749274/a9dc793e9dab42eebaa17fd4ce29e9d9?anonymousKey=0356652bac92bc45d29abfdd539eeb475a7220b4)[emergency_0](mutations/access_control/emergency/emergency_0.rs), [❌](https://prover.certora.com/output/7749274/a0c74bafc16140e0bfd5bab32a0be74c?anonymousKey=a414b865a6b730268e3e0696a0e4eee30e1aa49d)[contract_4](mutations/fees_collector/contract/contract_4.rs), [❌](https://prover.certora.com/output/7749274/da0c92ac1c3a408abdb88c6903668240?anonymousKey=21d45ff10b0993f811c0348a53d267cd38f3402f)[contract_2](mutations/fees_collector/contract/contract_2.rs) |
| HL-07 | high_level_concurrent_transfers_blocked | Concurrent ownership transfers are blocked through mutual exclusion | - |
| HL-08 | high_level_deadline_expiry_prevents_apply | Applying ownership transfer before deadline panics | - |

---

## Manual Mutations Testing

This section documents the manual mutations applied to the fees collector contract and its dependencies. Each mutation introduces a single-line code defect to test the specifications' ability to detect bugs. 24 mutations total: 1 emergency + 2 management + 3 storage + 6 transfer + 9 contract + 3 upgrade library.

**Mutation coverage summary:**
- **Caught**: 19/24 (79%)
- **Not caught**: 5/24 (transfer_0, transfer_2, transfer_3, contract_1, lib_2)

### Access Control - Emergency

#### [mutations/access_control/emergency/emergency_0.rs](mutations/access_control/emergency/emergency_0.rs)

Skips setting/clearing emergency mode in `set_emergency_mode()`. Vulnerability: system cannot enter emergency mode, perpetuating any risk by breaking the mitigation mechanism.

```rust
    // e.storage().instance().set(&DataKey::EmergencyMode, value); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/a9dc793e9dab42eebaa17fd4ce29e9d9?anonymousKey=0356652bac92bc45d29abfdd539eeb475a7220b4) **HL-06**: [high_level_emergency_mode_toggle](src/certora_specs/high_level.rs#L99) - Emergency mode can be toggled on and off
- [❌](https://prover.certora.com/jobStatus/7749274/b8df9855008748bf82cb9f1317c47c28?anonymousKey=946c0f4b224684e4b620c704855663387148f5e7) **INT-01**: [integrity_emergency_mode](src/certora_specs/integrity.rs#L21) - Setting emergency mode correctly stores the value

### Access Control - Management

#### [mutations/access_control/management/management_0.rs](mutations/access_control/management/management_0.rs)

Negated test of `is_transfer_delayed()` in `set_role_address()`. Vulnerability: allows immediate change of critical roles that should require a time delay.

```rust
    if addr.is_some() && !role.is_transfer_delayed() {  // MUTANT (negation)
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/75182fc17ea94fb789d8ca13c583269d?anonymousKey=d6746cf364031822387e5f20e26cfa06313b4fe1) **ST-01**: [state_transition_admin_role](src/certora_specs/state_transition.rs#L65) - Admin role can only change through time-delayed transfer
- [❌](https://prover.certora.com/output/7749274/68c114b1246d4000b897522b5499a631?anonymousKey=7e228f18f6367c1533daf017ab0efcff23ee822e) **ST-02**: [state_transition_emergency_admin_role](src/certora_specs/state_transition.rs#L92) - Emergency admin role requires time-delayed transfer
- [❌](https://prover.certora.com/output/7749274/3841a0e0093b4677bf95d1a79b5b803f?anonymousKey=feb73f6c344019fc751348b3c8fecab31ac26c51) **ST-03**: [state_transition_admin_init_once](src/certora_specs/state_transition.rs#L118) - Initial admin can only be set once

#### [mutations/access_control/management/management_1.rs](mutations/access_control/management/management_1.rs)

Skips setting role in `set_role_addresses()`. Vulnerability: role addresses can never actually be stored.

```rust
    // self.0.storage().instance().set(&key, address); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/51781f505db246ca80eb2dc7b7a9a7a8?anonymousKey=5d4a75fca9bde39d4680d50458c251a7aac75e50) **HL-03**: [high_level_upgrade_reachable](src/certora_specs/high_level.rs#L58) - Immediate upgrade path is reachable in emergency mode
- [❌](https://prover.certora.com/output/7749274/51781f505db246ca80eb2dc7b7a9a7a8?anonymousKey=5d4a75fca9bde39d4680d50458c251a7aac75e50) **HL-05**: [high_level_revert_upgrade_reachable](src/certora_specs/high_level.rs#L82) - Upgrade revert path is reachable
- [❌](https://prover.certora.com/output/7749274/b776f390107841069db26309f840dd6e?anonymousKey=2e6cc79433dcff96e6d467f3f70ee96043c3f7ee) **INT-10**: [integrity_address_has_role](src/certora_specs/integrity.rs#L147) - address_has_role returns true when address has role
- [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd) **INT-12**: [integrity_assert_address_has_role_negative](src/certora_specs/integrity.rs#L162) - assert_address_has_role panics when address doesn't have role
- [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd) **INT-09**: [integrity_privileged_addresses](src/certora_specs/integrity.rs#L130) - Privileged addresses set and retrieved correctly
- [❌](https://prover.certora.com/output/7749274/84fd014af2a34170b38161be0de08c28?anonymousKey=1f9b0adef6ec514917bf61618428bc08f3ff790c) **INT-05**: [integrity_role_address_single](src/certora_specs/integrity.rs#L56) - Single role address set and retrieved correctly
- [❌](https://prover.certora.com/output/7749274/3962ffc322fc489282d6f9271dfd429d?anonymousKey=22dac610e546c64c23af7f8ccacabbda2853295f) **INT-07**: [integrity_role_address_safe](src/certora_specs/integrity.rs#L100) - Safe role address getter returns correct value
- [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd) **INT-29**: [integrity_require_rewards_admin_negative](src/certora_specs/integrity.rs#L250) - require_rewards_admin_or_owner rejects unauthorized
- [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd) **INT-32**: [integrity_require_operations_admin_negative](src/certora_specs/integrity.rs#L279) - require_operations_admin_owner rejects unauthorized
- [❌](https://prover.certora.com/jobStatus/7749274/9b69ad8a0969455ca6d884ed997a2831?anonymousKey=befa20fa7864ab0fedb0c62e191b90a44dfc2acd) **INT-38**: [integrity_require_pause_admin_negative](src/certora_specs/integrity.rs#L334) - require_pause_admin_or_owner rejects unauthorized

### Access Control - Storage

#### [mutations/access_control/storage/storage_0.rs](mutations/access_control/storage/storage_0.rs)

In `get_key()`, gives wrong key (OperationsAdmin rather than Operator) for the RewardsAdmin role. Vulnerability: extra permissions granted to RewardsAdmin and OperationsAdmin due to address storage collision.

```rust
    Role::RewardsAdmin => DataKey::OperationsAdmin, // MUTANT: changed from DataKey::Operator
```

Caught by:
- [❌](https://prover.certora.com/jobStatus/7749274/ab8f1973a022499ebf5375a1513dca34?anonymousKey=2bba36489922d9018541b2ba43f66a41fd3f71e7) **INT-19**: [integrity_get_key_rewards_admin](src/certora_specs/integrity.rs#L185) - Storage key mapping for RewardsAdmin role
- [❌](https://prover.certora.com/jobStatus/7749274/ab8f1973a022499ebf5375a1513dca34?anonymousKey=2bba36489922d9018541b2ba43f66a41fd3f71e7) **INT-08**: [integrity_init_all_roles](src/certora_specs/integrity.rs#L107) - All roles initialized correctly
- [❌](https://prover.certora.com/jobStatus/7749274/ab8f1973a022499ebf5375a1513dca34?anonymousKey=2bba36489922d9018541b2ba43f66a41fd3f71e7) **INT-09**: [integrity_privileged_addresses](src/certora_specs/integrity.rs#L130) - Privileged addresses set and retrieved correctly

#### [mutations/access_control/storage/storage_1.rs](mutations/access_control/storage/storage_1.rs)

In `get_future_key()`, gives wrong key (FutureAdmin rather than FutureEmergencyAdmin) for the EmergencyAdmin role. Vulnerability: full Admin permissions may be granted to EmergencyAdmin role due to address storage collision.

```rust
    Role::EmergencyAdmin => DataKey::FutureAdmin, // MUTANT: changed from DataKey::FutureEmergencyAdmin
```

Caught by:
- [❌](https://prover.certora.com/jobStatus/7749274/64900911e0764ae4a35dc0688be10bd2?anonymousKey=1ae981872bc7b820750154588bd9de6d933e7366) **INT-24**: [integrity_get_future_key_emergency_admin](src/certora_specs/integrity.rs#L215) - Future storage key mapping for EmergencyAdmin role

#### [mutations/access_control/storage/storage_2.rs](mutations/access_control/storage/storage_2.rs)

In `get_future_deadline_key()`, gives wrong key (FutureAdmin rather than TransferOwnershipDeadline) for the Admin role. Vulnerability: Admin role transfer cannot be initiated, will revert with type error when checking existing deadline.

```rust
    Role::Admin => DataKey::FutureAdmin, // MUTANT: changed from DataKey::TransferOwnershipDeadline
```

Caught by:
- [❌](https://prover.certora.com/jobStatus/7749274/3736d8020d7244119ce686986ccf41ae?anonymousKey=03eb923ba87c6a9da24aadc07fb977c0f3f8e1a5) **INT-25**: [integrity_get_future_deadline_key_admin](src/certora_specs/integrity.rs#L221) - Future deadline key mapping for Admin role
- [❌](https://prover.certora.com/jobStatus/7749274/3736d8020d7244119ce686986ccf41ae?anonymousKey=03eb923ba87c6a9da24aadc07fb977c0f3f8e1a5) **INT-03**: [integrity_future_address](src/certora_specs/integrity.rs#L41) - Future address set by commit_transfer_ownership
- [❌](https://prover.certora.com/jobStatus/7749274/3736d8020d7244119ce686986ccf41ae?anonymousKey=03eb923ba87c6a9da24aadc07fb977c0f3f8e1a5) **INT-04**: [integrity_commit_transfer_deadline](src/certora_specs/integrity.rs#L48) - Commit sets deadline correctly

### Access Control - Transfer

#### [mutations/access_control/transfer/transfer_0.rs](mutations/access_control/transfer/transfer_0.rs)

Skips ownership transfer in `apply_transfer_ownership()`. Vulnerability: roles cannot be changed, violating design principle.

```rust
    // storage.set(&self.get_key(role), &future_address); MUTANT
```

**Not caught** by any rule.

#### [mutations/access_control/transfer/transfer_1.rs](mutations/access_control/transfer/transfer_1.rs)

Ignores stored deadline in `get_transfer_ownership_deadline()` and returns 0. Vulnerability: `apply_transfer_ownership()` always reverts so roles cannot be changed.

```rust
    let value = 0; // MUTANT: self.0.storage().instance().get(&key).unwrap_or(0);
```

Caught by:
- [❌](https://prover.certora.com/jobStatus/7749274/72b8a9624cc044738686ee96c1aa18a3?anonymousKey=f522c806d9a16bc2d5b085fc0753d7e1f70224ad) **INT-04**: [integrity_commit_transfer_deadline](src/certora_specs/integrity.rs#L48) - Commit sets deadline correctly
- [❌](https://prover.certora.com/jobStatus/7749274/72b8a9624cc044738686ee96c1aa18a3?anonymousKey=f522c806d9a16bc2d5b085fc0753d7e1f70224ad) **INT-03**: [integrity_future_address](src/certora_specs/integrity.rs#L41) - Future address stored correctly

#### [mutations/access_control/transfer/transfer_2.rs](mutations/access_control/transfer/transfer_2.rs)

Ignores requested deadline in `put_transfer_ownership_deadline()` and stores 0. Vulnerability: `apply_transfer_ownership()` always reverts so roles cannot be changed.

```rust
    self.0.storage().instance().set(&key, &0); // MUTANT: replaced value by 0
```

**Not caught** by any rule (only timeouts).

#### [mutations/access_control/transfer/transfer_3.rs](mutations/access_control/transfer/transfer_3.rs)

Fails to reset deadline in `revert_transfer_ownership()`. Vulnerability: transfer of Admin and EmergencyAdmin cannot be canceled.

```rust
    //self.put_transfer_ownership_deadline(role, 0); // MUTANT
```

**Not caught** by any rule.

#### [mutations/access_control/transfer/transfer_4.rs](mutations/access_control/transfer/transfer_4.rs)

Skips reset of deadline in `apply_transfer_ownership()`. Vulnerability: first transfer of Admin or EmergencyAdmin role blocks all future transfers of that role.

```rust
    // self.put_transfer_ownership_deadline(role, 0); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/90c4bcb913784db79d9a75fff7bf0d8e?anonymousKey=cec707fe9ca0ca1cac581461dd5a990c302194ec) **ST-02**: [state_transition_emergency_admin_role](src/certora_specs/state_transition.rs#L92) - Emergency admin role requires time-delayed transfer

#### [mutations/access_control/transfer/transfer_5.rs](mutations/access_control/transfer/transfer_5.rs)

`commit_transfer_ownership()` always sets a deadline in the past. Vulnerability: role changes that should be delayed can be applied immediately after being initiated.

```rust
    let deadline = self.0.ledger().timestamp() - ADMIN_ACTIONS_DELAY;  // MUTANT (+ to -)
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/ed51b583f8484cf6aaadd51cf7c9af77?anonymousKey=a48b4faa759edaf150715115263f551d21c07ad2) **ST-05**: [state_transition_admin_deadline_lifecycle](src/certora_specs/state_transition.rs#L164) - Admin deadline transitions
- [❌](https://prover.certora.com/output/7749274/01ad2ca874ec42e3bdd36f3b24b6c7f4?anonymousKey=48b9092c5eb73272ce12f0d9909e27e32fe992e2) **ST-09**: [state_transition_future_admin_consistency](src/certora_specs/state_transition.rs#L250) - Future admin value consistency
- [❌](https://prover.certora.com/output/7749274/dbbfb79761e843cfbd7fb4f9fef663f5?anonymousKey=f82428c73ae03ccae8d9a68919af006d2e397888) **ST-10**: [state_transition_future_em_admin_consistency](src/certora_specs/state_transition.rs#L270) - Future emergency admin value consistency
- [❌](https://prover.certora.com/jobStatus/7749274/7575a8f6c45f4d7897611ee8c73bd232?anonymousKey=132df700e9dd533bb6260bb5d92ec0e26e4f39c9) **INT-04**: [integrity_commit_transfer_deadline](src/certora_specs/integrity.rs#L48) - Commit sets deadline correctly
- [❌](https://prover.certora.com/jobStatus/7749274/7575a8f6c45f4d7897611ee8c73bd232?anonymousKey=132df700e9dd533bb6260bb5d92ec0e26e4f39c9) **INT-03**: [integrity_future_address](src/certora_specs/integrity.rs#L41) - Future address stored correctly

### Fees Collector - Contract

#### [mutations/fees_collector/contract/contract_0.rs](mutations/fees_collector/contract/contract_0.rs) (public)

Removed authorization check in `commit_upgrade()` restricting to Admin role. Vulnerability: anyone can initiate a software update with arbitrary code.

```rust
    // AccessControl::new(&e).assert_address_has_role(&admin, &Role::Admin); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/7261a86b49d7452e9ef834d2f837eb89?anonymousKey=c6f50d183b83a6ac73865e31e77bf9f5cf0bd9bd) **PER-08**: [permissions_upgrade_deadline_changes](src/certora_specs/permissions.rs#L220) - Only admin can change upgrade deadlines
- [❌](https://prover.certora.com/output/7749274/7261a86b49d7452e9ef834d2f837eb89?anonymousKey=c6f50d183b83a6ac73865e31e77bf9f5cf0bd9bd) **PER-09**: [permissions_future_wasm_changes](src/certora_specs/permissions.rs#L241) - Only admin can change future wasm

#### [mutations/fees_collector/contract/contract_1.rs](mutations/fees_collector/contract/contract_1.rs) (public)

Removed authorization check in `apply_transfer_ownership()` restricting to Admin role. Vulnerability: anyone can complete the transfer of Admin and EmergencyAdmin roles.

```rust
    // access_control.assert_address_has_role(&admin, &Role::Admin); MUTANT
```

**Not caught** by any rule (only timeout in `sanity_apply_transfer_ownership`).

#### [mutations/fees_collector/contract/contract_2.rs](mutations/fees_collector/contract/contract_2.rs) (public)

`get_emergency_mode()` always returns false. Vulnerability: `fees_collector.get_emergency_mode()` unreliable.

```rust
    false // MUTANT: always returns false, changed from `get_emergency_mode(&e)`
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/da0c92ac1c3a408abdb88c6903668240?anonymousKey=21d45ff10b0993f811c0348a53d267cd38f3402f) **HL-06**: [high_level_emergency_mode_toggle](src/certora_specs/high_level.rs#L99) - Emergency mode can be toggled
- [❌](https://prover.certora.com/jobStatus/7749274/c0ef460b23f247f3814d73c482965584?anonymousKey=2f10e16e3f2d1da1ff404fdcbf8dcadf44854633) **INT-01**: [integrity_emergency_mode](src/certora_specs/integrity.rs#L21) - Setting emergency mode correctly stores the value

#### [mutations/fees_collector/contract/contract_3.rs](mutations/fees_collector/contract/contract_3.rs)

Removed authorization check in `revert_upgrade()` restricting to Admin role. Vulnerability: anyone can cancel a software update.

```rust
    // AccessControl::new(&e).assert_address_has_role(&admin, &Role::Admin); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/7b23b306ddd74c61ac3305c48a342eb3?anonymousKey=82409b3302211fc1ae2249ce33e35e8def812753) **PER-03**: [permissions_admin_transfer_deadline_changes](src/certora_specs/permissions.rs#L74) - Only admin can change admin transfer deadlines
- [❌](https://prover.certora.com/output/7749274/ecdad43c17fc44d2b466f677c1fd1134?anonymousKey=2353e6f923c8a29fc0d134111d32304f7745ab91) **PER-08**: [permissions_upgrade_deadline_changes](src/certora_specs/permissions.rs#L220) - Only admin can change upgrade deadlines
- [❌](https://prover.certora.com/output/7749274/5b265f2ff9d14251a246b43343006d9f?anonymousKey=b48ddaa164aa9a3907d8f5b6c4c80f392bf1cfe7) **PER-04**: [permissions_em_admin_transfer_deadline_changes](src/certora_specs/permissions.rs#L95) - Only admin can change emergency admin transfer deadlines

#### [mutations/fees_collector/contract/contract_4.rs](mutations/fees_collector/contract/contract_4.rs)

Skips setting/clearing emergency mode in `set_emergency_mode()`. Vulnerability: system cannot enter emergency mode, perpetuating any risk by breaking the mitigation mechanism.

```rust
    // set_emergency_mode(&e, &value); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/a0c74bafc16140e0bfd5bab32a0be74c?anonymousKey=a414b865a6b730268e3e0696a0e4eee30e1aa49d) **HL-06**: [high_level_emergency_mode_toggle](src/certora_specs/high_level.rs#L99) - Emergency mode can be toggled
- [❌](https://prover.certora.com/jobStatus/7749274/8763492e1a004c67ab575c6463f95377?anonymousKey=1f0627171dd16ddabcccdf5c8cd5fba0578ac142) **INT-01**: [integrity_emergency_mode](src/certora_specs/integrity.rs#L21) - Setting emergency mode correctly stores the value

#### [mutations/fees_collector/contract/contract_5.rs](mutations/fees_collector/contract/contract_5.rs)

Disables `revert_transfer_ownership()`. Vulnerability: transfer of Admin and EmergencyAdmin cannot be canceled.

```rust
    // access_control.revert_transfer_ownership(&role); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/e98bbb1256974e63a2a5d168ec38c3cb?anonymousKey=10f01c80490ce7a3d1cd871ee4a86fe84b3a93b2) **PER-03**: [permissions_admin_transfer_deadline_changes](src/certora_specs/permissions.rs#L74) - Only admin can change admin transfer deadlines
- [❌](https://prover.certora.com/output/7749274/78f0841322fb44ca8aafe06abde4e411?anonymousKey=16ce8ffe5be3486eb36aca218ef950fda5fde070) **PER-05**: [permissions_future_admin_changes](src/certora_specs/permissions.rs#L120) - Only admin can change future admin
- [❌](https://prover.certora.com/output/7749274/a0db3b896a5a4955a2cd731663ba4084?anonymousKey=d92f7bc26a91ddfbb4d810e6d08ac97f0a7d9098) **PER-06**: [permissions_future_em_admin_changes](src/certora_specs/permissions.rs#L157) - Only admin can change future emergency admin

#### [mutations/fees_collector/contract/contract_6.rs](mutations/fees_collector/contract/contract_6.rs)

Removed authorization check in `commit_transfer_ownership()` restricting to Admin role. Vulnerability: anyone can initiate transfer of Admin and EmergencyAdmin roles to an arbitrary address.

```rust
    // access_control.assert_address_has_role(&admin, &Role::Admin); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/96ccf224b69d47a18992fa11b1b4bb1a?anonymousKey=22c11a9159380301d380ce58061c5f4505a07033) **PER-04**: [permissions_em_admin_transfer_deadline_changes](src/certora_specs/permissions.rs#L95) - Only admin can change emergency admin transfer deadlines

#### [mutations/fees_collector/contract/contract_7.rs](mutations/fees_collector/contract/contract_7.rs)

Fails to store code hash in `commit_upgrade()`. Vulnerability: upgrades can never be completed.

```rust
    // commit_upgrade(&e, &new_wasm_hash); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/06b66b02ea064146a8694b19c35ff3ce?anonymousKey=74b8a3cc4c36608318eec5e1458f812a85e0d059) **HL-02**: [high_level_upgrade_immediate_in_emergency](src/certora_specs/high_level.rs#L47) - Upgrades can be applied immediately only in emergency mode
- [❌](https://prover.certora.com/jobStatus/7749274/671c16a65c4144f692bc0114bd549969?anonymousKey=1a3dd0d5c8f119462c25a1f9395906c2c0b84d1f) **INT-13**: [integrity_commit_upgrade_future_wasm](src/certora_specs/integrity.rs#L71) - Commit upgrade correctly stores future wasm hash
- [❌](https://prover.certora.com/jobStatus/7749274/671c16a65c4144f692bc0114bd549969?anonymousKey=1a3dd0d5c8f119462c25a1f9395906c2c0b84d1f) **INT-14**: [integrity_commit_upgrade_deadline](src/certora_specs/integrity.rs#L78) - Commit upgrade sets deadline correctly

#### [mutations/fees_collector/contract/contract_8.rs](mutations/fees_collector/contract/contract_8.rs)

Disables `revert_upgrade()`. Vulnerability: upgrades can never be canceled.

```rust
    // revert_upgrade(&e); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/output/7749274/9dcb2dd41c7d4b02bea27ae778251ff4?anonymousKey=84baa3164f9a1ae0c395425b297ed65a8b85ac7a) **HL-04**: [high_level_revert_upgrade](src/certora_specs/high_level.rs#L69) - Upgrade can be reverted, clearing the deadline
- [❌](https://prover.certora.com/output/7749274/9dcb2dd41c7d4b02bea27ae778251ff4?anonymousKey=84baa3164f9a1ae0c395425b297ed65a8b85ac7a) **HL-05**: [high_level_revert_upgrade_reachable](src/certora_specs/high_level.rs#L82) - Upgrade revert path is reachable

### Upgrade - Library

#### [mutations/upgrade/lib/lib_0.rs](mutations/upgrade/lib/lib_0.rs)

Upgrades always have deadline 0 in `commit_upgrade()`. Vulnerability: upgrades can only be completed in emergency mode.

```rust
    let deadline = 0; // MUTANT: e.ledger().timestamp() + UPGRADE_DELAY;
```

Caught by:
- [❌](https://prover.certora.com/jobStatus/7749274/8b7b801b98544e9d8d3de7468619daa6?anonymousKey=60ff68e03fc8ef0c46695c8a668a301f4d5f5a11) **INT-14**: [integrity_commit_upgrade_deadline](src/certora_specs/integrity.rs#L78) - Commit upgrade sets deadline correctly

#### [mutations/upgrade/lib/lib_1.rs](mutations/upgrade/lib/lib_1.rs)

Skips setting deadline in `commit_upgrade()`. Vulnerability: upgrades can only be completed in emergency mode.

```rust
    // put_upgrade_deadline(e, &deadline); MUTANT
```

Caught by:
- [❌](https://prover.certora.com/jobStatus/7749274/0fd2ef56b4c54571a20a15cab5b9775b?anonymousKey=5b1c9cd7d823fb6584d7e4b399b844e811f6ac9a) **INT-14**: [integrity_commit_upgrade_deadline](src/certora_specs/integrity.rs#L78) - Commit upgrade sets deadline correctly

#### [mutations/upgrade/lib/lib_2.rs](mutations/upgrade/lib/lib_2.rs)

Skips reset of deadline in `apply_upgrade()`. Vulnerability: any past upgrade can be applied again.

```rust
    // put_upgrade_deadline(e, &0); MUTANT
```

**Not caught** by any rule (only timeout in `sanity_apply_transfer_ownership`).

---

## Violated Properties

These properties were intentionally violated during verification, revealing actual issues in the contract code.

### Issue #1 - Upgrade Deadline Lifecycle

**GitHub Issue**: [alexzoid-eth/aquarius-cantina-fv#1](https://github.com/alexzoid-eth/aquarius-cantina-fv/issues/1)

**Finding**: The `state_transition_upgrade_deadline_lifecycle` property is violated for `apply_upgrade` and `revert_upgrade` functions.

**Affected Rules** (from [fees_collector_state_transition_upgrade_deadlines_violated.conf](confs/fees_collector_state_transition_upgrade_deadlines_violated.conf)):
- `state_transition_upgrade_deadline_lifecycle_apply_upgrade`
- `state_transition_upgrade_deadline_lifecycle_revert_upgrade`

**Description**: The upgrade deadline lifecycle check in `check_upgrade_deadline_lifecycle()` validates that when a deadline transitions from non-zero to zero (apply or revert), the timestamp must have passed the deadline in normal mode. The violation indicates that the `apply_upgrade` and `revert_upgrade` functions do not correctly enforce the upgrade deadline lifecycle constraints.

```rust
fn check_upgrade_deadline_lifecycle(
    e: &Env,
    deadline_before: u64,
    deadline_after: u64,
    _future_before: Option<impl Clone>,
    future_after: Option<impl Clone>,
    emergency_mode_before: bool,
    _emergency_mode_after: bool,
) -> bool {
    // ...
    } else if deadline_before != 0 && deadline_after == 0 {
        // Transition from non-zero to 0 (apply or revert)
        if emergency_mode_before {
            future_after.is_none()
        } else {
            e.ledger().timestamp() >= deadline_before && future_after.is_none()
        }
    }
    // ...
}
```

### Issue #2 - Admin Deadline Clear

**GitHub Issue**: [alexzoid-eth/aquarius-cantina-fv#2](https://github.com/alexzoid-eth/aquarius-cantina-fv/issues/2)

**Finding**: The `state_transition_admin_deadline_clear` and `state_transition_em_admin_deadline_clear` properties are violated for `apply_transfer_ownership` and `revert_transfer_ownership` functions.

**Affected Rules** (from [fees_collector_state_transition_admin_transfer_deadlines_violated.conf](confs/fees_collector_state_transition_admin_transfer_deadlines_violated.conf)):
- `state_transition_admin_deadline_clear_apply_transfer_ownership`
- `state_transition_admin_deadline_clear_revert_transfer_ownership`
- `state_transition_em_admin_deadline_clear_apply_transfer_ownership`
- `state_transition_em_admin_deadline_clear_revert_transfer_ownership`

**Description**: When the admin transfer deadline transitions from non-zero to zero (during apply or revert), the `future_admin` (or `future_em_admin`) should be cleared to `None`. The violation shows that the contract does not properly clear the future address when completing or reverting an ownership transfer. This was noted in the `check_admin_deadline_lifecycle()` function where the relevant check was commented out due to this issue.

```rust
fn check_admin_deadline_lifecycle(
    e: &Env,
    deadline_before: u64,
    deadline_after: u64,
    future_after: Option<impl Clone>,
) -> bool {
    if deadline_before == 0 && deadline_after != 0 {
        deadline_after == e.ledger().timestamp() + ADMIN_ACTIONS_DELAY && future_after.is_some()
    // } else if deadline_before != 0 && deadline_after == 0 {
        // Transition from non-zero to 0 (apply or revert)
        // @note This executable path is violated due
        //  to https://github.com/alexzoid-eth/aquarius-cantina-fv/issues/2
        // future_after.is_none()
    } else if deadline_before != 0 && deadline_after != 0 {
        deadline_before == deadline_after
    } else {
        true
    }
}
```

---

## Setup and Execution Instructions

### Certora Prover Installation

For step-by-step installation steps refer to the Certora [documentation](https://docs.certora.com/en/latest/docs/sunbeam/installation.html).

### Verification Execution

1. Build the fees_collector contract with Certora features:
```bash
cd fees_collector
just build
```

2. Run a specific verification:
```bash
cd fees_collector/confs
certoraSorobanProver <config_file>.conf
```

3. Run all verifications:
```bash
cd fees_collector/confs
./run_all.sh
```

4. Run mutation testing for a specific category:
```bash
cd fees_collector/confs
./run_all_mutations.sh
```
