# Formal Verification Report: Centrifuge Core

- Date: September 30th, 2026
- Audit Repo: https://github.com/alexzoid-eth/2026-08-centrifuge-fv-ex
- Client Repo: https://github.com/centrifuge/protocol-internal ([74e16461](https://github.com/centrifuge/protocol-internal/tree/74e16461ca39aadc63ab4f0096d907237ac4d51a/src/core))
- Author: [AlexZoid](https://x.com/alexzoid)
- Certora Prover version: 8.19.2

<!-- Audit Commit: a3d51a0de -->

---

## Table of Contents

1. [Verification Scope](#verification-scope)
2. [Methodology](#methodology)
   - [Setup Scenes](#setup-scenes)
   - [Types of Properties](#types-of-properties)
   - [Types of Assumptions](#types-of-assumptions)
3. [Properties](#properties)
   - Hub side
     - [Accounting](#accounting)
     - [Holdings](#holdings)
     - [ShareClassManager](#shareclassmanager)
     - [HubRegistry](#hubregistry)
     - [Hub_LinkedHubCore](#hub_linkedhubcore)
     - [HubHandler_LinkedHubCore](#hubhandler_linkedhubcore)
   - Spoke side
     - [Spoke](#spoke)
     - [SpokeRegistry](#spokeregistry)
     - [SnapshotQueue](#snapshotqueue)
     - [Escrow](#escrow)
     - [Spoke_LinkedSpokeCore](#spoke_linkedspokecore)
     - [SpokeHandler_LinkedSpokeRegistry](#spokehandler_linkedspokeregistry)
     - [SpokeHandler_LinkedRegistryFactory](#spokehandler_linkedregistryfactory)
     - [EscrowFactory](#escrowfactory)
   - Messaging
     - [MultiAdapter](#multiadapter)
     - [Gateway](#gateway)
     - [MessageDispatcher](#messagedispatcher)
     - [MessageProcessor](#messageprocessor)
     - [Gateway_LinkedMessageProcessor](#gateway_linkedmessageprocessor)
4. [Mutation Testing](#mutation-testing)
5. [Reproducing the Results](#reproducing-the-results)
   - [Prerequisites](#prerequisites)
   - [Remote Execution](#remote-execution)
   - [Local Execution](#local-execution)
   - [Running the Suites](#running-the-suites)
6. [Resources](#resources)

---

## Verification Scope

Centrifuge is a real-world-asset tokenization protocol built as a hub and spoke: one hub chain carries each pool's accounting and administration, spoke chains the custody and user-facing balance-sheet operations; a quorum-gated messaging layer keeps them convergent. The contracts verified, all in [`src/core/`](../src/core/), fall into three groups.

**Hub side**

1. **Accounting** ([`hub/Accounting.sol`](../src/core/hub/Accounting.sol)): the double-entry ledger of per-pool charts of accounts, mutable only inside a transient session that refuses to close unless debits equal credits.

2. **Holdings** ([`hub/Holdings.sol`](../src/core/hub/Holdings.sol)): the hub's mirror of spoke asset positions per pool, share class and asset, with carrying value in pool currency against cumulative increase and decrease counters.

3. **ShareClassManager** ([`hub/ShareClassManager.sol`](../src/core/hub/ShareClassManager.sol)): the share class registry and hub-side conservation ledger, where the per-network issuance and revocation counters must net to the class's total supply, a share transfer between networks moving only the network legs.

4. **HubRegistry** ([`hub/HubRegistry.sol`](../src/core/hub/HubRegistry.sol)): the registry of pools, assets, managers and policies, plus the timelocked authorization ledger the hub access-control model rests on.

5. **Hub** ([`hub/Hub.sol`](../src/core/hub/Hub.sol)): the pool-manager orchestrator over the four contracts above, which must post a paired debit and credit for every accounting mutation.

6. **HubHandler** ([`hub/HubHandler.sol`](../src/core/hub/HubHandler.sol)): the hub-side entry point for cross-chain messages, applying spoke deltas to Holdings and ShareClassManager, reconciling the burn-and-mint share bridge.

**Spoke side**

7. **Spoke** ([`spoke/Spoke.sol`](../src/core/spoke/Spoke.sol)): the balance-sheet orchestrator for users and managers, where every escrow operation must mirror into the snapshot queue.

8. **SpokeRegistry** ([`spoke/SpokeRegistry.sol`](../src/core/spoke/SpokeRegistry.sol)): the spoke's source of truth for pools, share classes, roles, policies, vaults and prices, over an asset-id bijection.

9. **SpokeHandler** ([`spoke/SpokeHandler.sol`](../src/core/spoke/SpokeHandler.sol)): the spoke-side message applier, deploying escrows, tokens and vaults and routing mutations into the registry.

10. **SnapshotQueue** ([`spoke/SnapshotQueue.sol`](../src/core/spoke/SnapshotQueue.sol)): the accumulator of share and asset deltas pending submission to the hub, carrying the consistency flag the hub trusts.

11. **Escrow** ([`spoke/Escrow.sol`](../src/core/spoke/Escrow.sol)): the per-pool custody contract, with total and reserved bookkeeping where a withdrawal never pays the reserved part.

12. **EscrowFactory** ([`spoke/factories/EscrowFactory.sol`](../src/core/spoke/factories/EscrowFactory.sol)): the CREATE2 deployer and address oracle for those escrows, where a pool's escrow address is the same for every caller and unchanged by any call the factory takes.

**Messaging**

13. **MultiAdapter** ([`messaging/MultiAdapter.sol`](../src/core/messaging/MultiAdapter.sol)): the quorum layer over bridge adapters, where an inbound payload becomes truth only at threshold votes per chain, pool and session.

14. **MessageDispatcher** ([`messaging/MessageDispatcher.sol`](../src/core/messaging/MessageDispatcher.sol)): the outbound end of the messaging layer, taking a local destination straight into its handler and a remote one into the gateway, where a caller without the ward bit moves no wiring and reaches no neighbour.

15. **Gateway** ([`messaging/Gateway.sol`](../src/core/messaging/Gateway.sol)): the transport between the messaging layer and its adapters, batching and paying for outbound messages and splitting an inbound batch into messages for the processor, where a batch carries one pool and a message that fails in processing is recorded for a retry.

16. **MessageProcessor** ([`messaging/MessageProcessor.sol`](../src/core/messaging/MessageProcessor.sol)): the inbound end of the messaging layer, decoding each message and routing it to the one handler its type names, where a message of no known type is refused.

<div style="page-break-before: always;"></div>

---

## Methodology

Certora Formal Verification (FV) provides mathematical proofs of smart contract correctness by verifying code against a formal specification. Unlike testing and fuzzing which examine specific execution paths, Certora FV examines all possible states and execution paths.

The process involves crafting properties in CVL (Certora Verification Language) and submitting them alongside compiled Solidity smart contracts to the prover. The prover transforms the contract bytecode and rules into a mathematical model and determines the validity of rules.

### Setup Scenes

A scene is the set of contracts the prover compiles for one verification run. Each is present as **real** bytecode or as a **CVL model**, a stand-in written in the specification language, and every target is real in the scene that verifies it.

A scene splits into a setup and the rules that run over it. For each verified contract the setup ships the specification another scene imports to attach it. Where a scene models the contract instead, the setup also ships the model that stands in for it. It links the compiled contracts, resolves every outgoing call to a model, a compiled neighbour or out of scope, and rewrites in CVL the internal functions the prover cannot analyse.

![Solo scenes](./assets/verification-scenes-solo.svg)

A **solo** scene compiles one target and models every neighbour in CVL; it carries whatever valid state that target's storage admits, and every property that needs no second contract.

![Joint scenes](./assets/verification-scenes-joint.svg)

A **joint** scene compiles several contracts and links them, is usually named `A_LinkedB`, and takes each participant's valid state, where a rule needs it, either as a fact already proven on that contract's own scene or by re-proving it inductively here.

A figure draws each scene's target and its main neighbours. A configuration may compile more, such as the real MessageDispatcher in MessageProcessor's Joint configuration; the `files` list of each conf names every contract its run compiles.

### Types of Properties

The official Certora [methodology](https://github.com/Certora/Tutorials/blob/master/06.Lesson_ThinkingProperties/Categorizing_Properties.pdf) defines several property categories. A **parametric** category is checked against every external function of the scene, including ones added after the specification; a rule inside such a category may still name the single call it is about.

- **Valid State** (parametric): system-wide invariants that hold in every reachable state; once proven, other properties reuse them as preconditions through `requireInvariant`.
- **State Transitions** (parametric): the moves between valid states, how a valid state may change and in what order.
- **Variable Transitions** (parametric): the permitted before/after relation on a single state variable across any call: monotonic counters, write-once fields, values that only move within a bound.
- **High Level**: properties of one or several specific function calls (share conservation, a deposit and redeem round trip).
- **Reverts**: an external call reverts exactly on its documented preconditions.
- **Reachability**: every rule's states are reachable, so none passes vacuously.
- **Access Control** (parametric): every privileged entry point enforces its ward or role gate.

### Types of Assumptions

Each `require` limits which states the prover checks, and its tag says why.

- **SAFE**: excludes no reachable state (`msg.sender != 0`).
- **SCOPE**: narrows a rule to the logic it checks (`receiver != escrowAddress`).
- **UNSAFE**: drops reachable states, for a prover limit or timeout (a loop unrolled to a fixed count).
- **TRUSTED**: guaranteed by a trusted party outside the scene (a valid setup left by the initializer logic).
- **PROVED**: not an assumption: a fact proven elsewhere, reused as a precondition (an invariant via `requireInvariant`).
- **ASSERT**: used in models: a `require` reproducing a revert of the real code, a path the prover drops by default anyway (`InsufficientBalance` in the token model).

<div style="page-break-before: always;"></div>

---

## Properties

Every verdict below comes from the prover version named in the header.

- **Properties**: every invariant and rule the suite states
- **Proved** ✅: verified by the prover
- **Timeout** ⏱️: the prover ran the rule and did not decide it within the solver budget the confs carry, or within the 300-minute wall each rule of this edition's run was given
- **Mutations** 🎯: rules with a mutant the prover killed: the rule holds on the real code and fails on a planted fault.
- **Violations** ❌: rules that fail

| Contract | Properties | Proved ✅ | Timeout ⏱️ | Mutations 🎯 | Violations ❌ |
|---|---|---|---|---|---|
| [Accounting](#accounting) | 77 | 77 | 0 | 77 | 0 |
| [Holdings](#holdings) | 114 | 114 | 0 | 114 | 0 |
| [ShareClassManager](#shareclassmanager) | 91 | 91 | 0 | 91 | 0 |
| [HubRegistry](#hubregistry) | 110 | 110 | 0 | 110 | 0 |
| [Hub_LinkedHubCore](#hub_linkedhubcore) | 147 | 147 | 0 | 147 | 0 |
| [HubHandler_LinkedHubCore](#hubhandler_linkedhubcore) | 112 | 112 | 0 | 112 | 0 |
| [Spoke](#spoke) | 124 | 124 | 0 | 124 | 0 |
| [SpokeRegistry](#spokeregistry) | 171 | 171 | 0 | 171 | 0 |
| [SnapshotQueue](#snapshotqueue) | 77 | 77 | 0 | 77 | 0 |
| [Escrow](#escrow) | 63 | 63 | 0 | 63 | 0 |
| [Spoke_LinkedSpokeCore](#spoke_linkedspokecore) | 52 | 52 | 0 | 52 | 0 |
| [SpokeHandler_LinkedSpokeRegistry](#spokehandler_linkedspokeregistry) | 99 | 99 | 0 | 99 | 0 |
| [SpokeHandler_LinkedRegistryFactory](#spokehandler_linkedregistryfactory) | 4 | 4 | 0 | 4 | 0 |
| [EscrowFactory](#escrowfactory) | 13 | 13 | 0 | 13 | 0 |
| [MultiAdapter](#multiadapter) | 135 | 135 | 0 | 135 | 0 |
| [Gateway](#gateway) | 65 | 65 | 0 | 65 | 0 |
| [MessageDispatcher](#messagedispatcher) | 98 | 98 | 0 | 98 | 0 |
| [MessageProcessor](#messageprocessor) | 30 | 30 | 0 | 30 | 0 |
| [Gateway_LinkedMessageProcessor](#gateway_linkedmessageprocessor) | 7 | 7 | 0 | 7 | 0 |
| **Total** | **1589** | **1589** | **0** | **1589** | **0** |

### Accounting

- Single pool
  - One symbolic pool
  - The chart of accounts bounded to 3 symbolic accounts
  - Loops running up to 3 iterations
- Multi pool
  - Two symbolic pools
  - The chart of accounts bounded to 3 symbolic accounts
  - Loops running up to 3 iterations
- Metadata
  - Any symbolic pool and account
  - Loops running up to 16 iterations
  - The account metadata writer kept on the surface, its free form bytes read under the 512 byte hashing bound

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [AC-VS-01](./specs/core/hub/Accounting/single/Accounting_single_pool_valid_state.spec#L11-L15) `absentAccountIsEmpty`<br>A never-created account has no recorded debits or credits. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-602) [🎯](./mutations/Accounting/README.md#accounting-603) [🎯](./mutations/Accounting/README.md#accounting-604) |
| [AC-VS-02](./specs/core/hub/Accounting/single/Accounting_single_pool_valid_state.spec#L17-L20) `nullAccountNeverCreated`<br>The empty account id never exists, so no entry can be posted to it. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-605) |
| [AC-VS-03](./specs/core/hub/Accounting/single/Accounting_single_pool_valid_state.spec#L22-L27) `lastUpdatedBetweenEnvFloorAndBlock`<br>An account's last update time is never in the future. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-606) [🎯](./mutations/Accounting/README.md#accounting-607) |
| [AC-VS-04](./specs/core/hub/Accounting/single/Accounting_single_pool_valid_state.spec#L29-L33) `postedHistoryImpliesAnIssuedJournalId`<br>An account has postings only after at least one journal id was issued. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-617) |
| [AC-VS-05](./specs/core/hub/Accounting/single/Accounting_single_pool_valid_state.spec#L35-L39) `issuedJournalIdsLieAtOrBelowTheHighWaterMark`<br>No journal id a pool has issued is above the latest id the pool has minted. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9001) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [AC-ST-01](./specs/core/hub/Accounting/single/Accounting_single_pool_transitions.spec#L70-L83) `stAcAccountOrientationIsPermanent`<br>An account's debit-normal or credit-normal side never changes once created. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-629) |
| [AC-ST-02](./specs/core/hub/Accounting/Accounting_multi_pool_properties.spec#L5-L32) `stAcAccountRowsArePerPool`<br>A call changing an account in one pool leaves another pool's accounts untouched. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-622) |
| [AC-ST-03](./specs/core/hub/Accounting/Accounting_multi_pool_properties.spec#L34-L49) `stAcJournalNumberingIsPerPool`<br>A call advancing one pool's journal ids leaves another pool's ids unchanged. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-623) |
| [AC-ST-04](./specs/core/hub/Accounting/Accounting_multi_pool_properties.spec#L51-L64) `stAcAPoolsJournalIdIsFixedForTheTransaction`<br>Once a pool has a journal id in a transaction, no call for any pool changes it. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9045) |
| [AC-ST-05](./specs/core/hub/Accounting/single/Accounting_single_pool_transitions.spec#L85-L102) `stAcLedgerMovesOnlyThroughAJournal`<br>Ledger totals change only when a journal is posted. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4006) |
| [AC-ST-06](./specs/core/hub/Accounting/single/Accounting_single_pool_transitions.spec#L104-L118) `stAcMintedJournalIdWasNeverIssuedBefore`<br>A journal a call opens gets an id its pool never issued before. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9044) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [AC-VT-01](./specs/core/hub/Accounting/single/Accounting_single_pool_transitions.spec#L5-L19) `vtAcAccountDebitNeverDecreases`<br>An account's total debits never decrease, so posted history cannot be erased. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-608) [🎯](./mutations/Accounting/README.md#accounting-4000) [🎯](./mutations/Accounting/README.md#accounting-4014) |
| [AC-VT-02](./specs/core/hub/Accounting/single/Accounting_single_pool_transitions.spec#L21-L35) `vtAcAccountCreditNeverDecreases`<br>An account's total credits never decrease, so posted history cannot be erased. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-608) [🎯](./mutations/Accounting/README.md#accounting-4000) [🎯](./mutations/Accounting/README.md#accounting-4015) |
| [AC-VT-03](./specs/core/hub/Accounting/single/Accounting_single_pool_transitions.spec#L37-L51) `vtAcAccountStampNeverRegresses`<br>An account's last update time never goes backwards. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4001) |
| [AC-VT-04](./specs/core/hub/Accounting/single/Accounting_single_pool_transitions.spec#L53-L66) `vtAcJournalCounterNeverRewinds`<br>A pool's journal numbering never goes backwards. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-7107) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [AC-HL-01](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L3-L19) `hlAcSessionClosesOnlyOnEqualPostedValues`<br>A journal can be closed only when its debits and credits balance. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4003) |
| [AC-HL-02](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L21-L36) `hlAcJournalPairDebitsExactlyItsNamedRow`<br>Posting a journal pair debits only the named account, by the stated amount. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4004) [🎯](./mutations/Accounting/README.md#accounting-4302) |
| [AC-HL-03](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L38-L53) `hlAcJournalPairCreditsExactlyItsNamedRow`<br>Posting a journal pair credits only the named account, by the stated amount. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-619) [🎯](./mutations/Accounting/README.md#accounting-4005) |
| [AC-HL-04](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L55-L75) `hlAcSecondJournalOfAPoolReusesItsJournalId`<br>A pool's second journal in one transaction reuses the first journal's id. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4027) |
| [AC-HL-05](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L77-L92) `hlAcAccountValueReportsTheRowsNetMagnitude`<br>An account's reported balance is the difference between its debits and credits. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4008) |
| [AC-HL-06](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L94-L110) `hlAcAccountValueSignsByTheRowsOrientation`<br>An account reads positive exactly when its normal side is at least the other. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4009) |
| [AC-HL-07](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L112-L128) `hlAcReopenedJournalStartsFromEmptyPostedSides`<br>A reopened journal starts with zero running debit and credit sums. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4010) |
| [AC-HL-08](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L130-L141) `hlAcMintedJournalIdCarriesItsPool`<br>A new journal id falls in its pool's own range, so pools' ids never collide. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-8604) |
| [AC-HL-09](./specs/core/hub/Accounting/single/Accounting_single_pool_high_level.spec#L143-L153) `hlAcUnlockOpensTheSessionOnTheNamedPool`<br>Opening a journal unlocks accounting for exactly the requested pool. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-8605) |
| [AC-HL-10](./specs/core/hub/Accounting/Accounting_metadata_properties.spec#L5-L35) `hlAcSetAccountMetadataWritesOnlyItsAccountsMetadata`<br>SetAccountMetadata writes only the named account's metadata; every other field reads the same. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9004) |
| [AC-HL-11](./specs/core/hub/Accounting/Accounting_multi_pool_properties.spec#L68-L83) `hlAcSecondPoolOfATransactionOpensUnderItsOwnId`<br>A second pool opening a journal in one transaction gets an id in its own range. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-8606) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [AC-RV-01](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L3-L15) `rvAcUnlockRefusesANonWard`<br>Opening a journal fails for a caller that is not an admin. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-7106) |
| [AC-RV-02](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L17-L29) `rvAcUnlockRefusesASecondOpenWhileAJournalStands`<br>Opening a journal fails while another journal is still open. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4012) |
| [AC-RV-03](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L31-L48) `rvAcCreateAccountRefusesANonWardTheNullIdOrARowThatStands`<br>Creating an account fails for a non-admin, an empty id, or an existing account. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4013) |
| [AC-RV-04](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L50-L65) `rvAcAddDebitRefusesARowTheLedgerNeverOpened`<br>A debit fails on an account that was never created. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-7103) |
| [AC-RV-05](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L67-L82) `rvAcAddCreditRefusesARowTheLedgerNeverOpened`<br>A credit fails on an account that was never created. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-7104) |
| [AC-RV-06](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L84-L103) `rvAcLockRevertsIffThePostedSidesDiffer`<br>Closing a journal as admin fails exactly when its debits and credits differ. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-616) [🎯](./mutations/Accounting/README.md#accounting-4016) |
| [AC-RV-07](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L105-L119) `rvAcAddJournalOffSessionRefusesANonWardOrAnyEntry`<br>Outside an open journal, a non-admin post fails and so does any post with entries. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4017) |
| [AC-RV-08](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L121-L134) `rvAcAccountValueRefusesARowTheLedgerNeverOpened`<br>Reading the value of a never-created account fails rather than returning zero. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4018) |
| [AC-RV-09](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L136-L154) `rvAcAddDebitRefusesAnOutsiderInsideAnOpenJournal`<br>A debit by a non-admin fails even while an admin has a journal open. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4020) |
| [AC-RV-10](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L156-L174) `rvAcAddCreditRefusesAnOutsiderInsideAnOpenJournal`<br>A credit by a non-admin fails even while an admin has a journal open. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4021) |
| [AC-RV-11](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L176-L193) `rvAcLockRefusesAnOutsiderOnAnOpenJournal`<br>Closing a journal fails for a non-admin even while one is open. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4022) |
| [AC-RV-12](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L195-L208) `rvAcAccountValueAlwaysAnswersForARowThatStands`<br>Reading an existing account's value always succeeds, positive or negative. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-7105) |
| [AC-RV-13](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L210-L219) `rvAcAddDebitIsRefusedOffSession`<br>A debit always fails while no journal is open. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-8600) |
| [AC-RV-14](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L221-L230) `rvAcAddCreditIsRefusedOffSession`<br>A credit always fails while no journal is open. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-8601) |
| [AC-RV-15](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L232-L241) `rvAcLockIsRefusedOffSession`<br>Closing a journal always fails when none is open. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-8602) |
| [AC-RV-16](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L243-L254) `rvAcUnlockRefusesTheNullPool`<br>Opening a journal fails for the empty pool id. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-8603) |
| [AC-RV-17](./specs/core/hub/Accounting/single/Accounting_single_pool_reverts.spec#L256-L267) `rvAcLockIsRefusedAfterAClosedJournal`<br>Closing fails when no journal is open, whatever an earlier journal left behind. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9003) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [AC-RC-01](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L3-L19) `rcAcUnlockOpensAJournalIsReachable`<br>An admin can open a journal on a pool, getting a fresh journal id. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9041) |
| [AC-RC-02](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L21-L40) `rcAcEmptyJournalOverAnUntouchedLedgerIsReachable`<br>An empty journal can open and close, still using up a journal id. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9041) |
| [AC-RC-03](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L42-L64) `rcAcBalancedPairPostsAndClosesIsReachable`<br>A journal can open, post a matching debit and credit to two accounts, and close. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4029) |
| [AC-RC-04](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L66-L84) `rcAcTwoDebitsCloseAgainstOneCreditIsReachable`<br>A journal with two non-zero debits can close against one credit of their sum. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4031) |
| [AC-RC-05](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L86-L103) `rcAcZeroValueDebitRestampsTheRowIsReachable`<br>A zero-value debit can refresh an account's timestamp without changing totals. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9043) |
| [AC-RC-06](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L105-L122) `rcAcZeroValueCreditRestampsTheRowIsReachable`<br>A zero-value credit can refresh an account's timestamp without changing totals. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9042) |
| [AC-RC-07](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L124-L139) `rcAcDebitFillsARowToTheCeilingIsReachable`<br>A single debit can fill an empty account's debit side to the maximum value. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4029) |
| [AC-RC-08](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L141-L156) `rcAcCreditFillsARowToTheCeilingIsReachable`<br>A single credit can fill an empty account's credit side to the maximum value. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4030) |
| [AC-RC-09](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L158-L179) `rcAcCeilingPairPostsAndClosesIsReachable`<br>A journal posting the maximum value on both sides can still close. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4029) |
| [AC-RC-10](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L181-L195) `rcAcSaturatedDebitRowStillTakesACreditIsReachable`<br>An account with debits at the maximum value can still take a credit. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4030) |
| [AC-RC-11](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L197-L211) `rcAcDebitOvertakesACreditHeavyRowIsReachable`<br>A debit can turn a credit-heavy account into a debit-heavy one. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4034) |
| [AC-RC-12](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L213-L230) `rcAcCreateDebitNormalAccountIsReachable`<br>A debit-normal account can be opened where none existed. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4028) |
| [AC-RC-13](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L232-L249) `rcAcCreateCreditNormalAccountIsReachable`<br>A credit-normal account can be opened with empty debit and credit sides. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4028) |
| [AC-RC-14](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L251-L271) `rcAcSecondRowOpensBesideALiveOneIsReachable`<br>A second account can be opened without disturbing an existing one. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4028) |
| [AC-RC-15](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L273-L290) `rcAcFirstRowOfAnEmptyChartIsReachable`<br>A pool's very first account can be opened. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4028) |
| [AC-RC-16](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L292-L317) `rcAcJournalBatchReachesTheWholeChartIsReachable`<br>One journal can debit two accounts, credit a third and still close balanced. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4029) |
| [AC-RC-17](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L319-L340) `rcAcEmptyJournalBatchIsReachable`<br>An empty journal can go through and still use up a journal id. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9041) |
| [AC-RC-18](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L342-L363) `rcAcZeroValueJournalBatchIsReachable`<br>A zero-value journal can close and still refresh an account's timestamp. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-9043) |
| [AC-RC-19](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L365-L385) `rcAcRowsCreatedAndJournalledInOneTransactionIsReachable`<br>Accounts can be created and posted to within one transaction. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4028) |
| [AC-RC-20](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L387-L402) `rcAcFreshRowReadsBackThePostedValueIsReachable`<br>A newly created account can report exactly the value just posted to it. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4028) |
| [AC-RC-21](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L404-L419) `rcAcDebitNormalRowReportsANegativeValueIsReachable`<br>A debit-normal account credited more than debited can report a negative value. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4028) |
| [AC-RC-22](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L421-L436) `rcAcRowDebitedAndCreditedAlikeReadsZeroIsReachable`<br>An account debited and credited by the same amount can report a zero value. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4028) |
| [AC-RC-23](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L438-L452) `rcAcOpenSessionReportsItsRunningSidesIsReachable`<br>An open journal can report its running debit total as posted so far. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4029) |
| [AC-RC-24](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L454-L471) `rcAcAuthorityGrantRoundTripIsReachable`<br>An admin can grant admin rights and revoke them again in one transaction. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-7100) |
| [AC-RC-25](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L473-L485) `rcAcWardRevokesItselfIsReachable`<br>An admin can revoke its own admin rights on the ledger. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-7101) |
| [AC-RC-26](./specs/core/hub/Accounting/single/Accounting_single_pool_reachability.spec#L487-L501) `rcAcPassingJournalPairOnABalancedPairIsReachable`<br>The hub can post a non-zero balanced debit and credit to two existing accounts. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-7102) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [AC-AC-01](./specs/core/hub/Accounting/single/Accounting_single_pool_access_control.spec#L3-L19) `acAcDebitHistoryMovesOnlyForAWard`<br>Only an admin's call can change an account's booked debits. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4023) |
| [AC-AC-02](./specs/core/hub/Accounting/single/Accounting_single_pool_access_control.spec#L21-L37) `acAcCreditHistoryMovesOnlyForAWard`<br>Only an admin's call can change an account's booked credits. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4023) |
| [AC-AC-03](./specs/core/hub/Accounting/single/Accounting_single_pool_access_control.spec#L39-L55) `acAcAccountStampMovesOnlyForAWard`<br>Only an admin's call can change an account's last-updated time. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4023) |
| [AC-AC-04](./specs/core/hub/Accounting/single/Accounting_single_pool_access_control.spec#L57-L71) `acAcAccountOpensOnlyForAWard`<br>Only an admin can open a new account in the ledger. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4023) |
| [AC-AC-05](./specs/core/hub/Accounting/single/Accounting_single_pool_access_control.spec#L73-L88) `acAcAccountOrientationIsSetOnlyByAWard`<br>Only an admin can set whether an account is debit-normal or credit-normal. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4023) |
| [AC-AC-06](./specs/core/hub/Accounting/single/Accounting_single_pool_access_control.spec#L90-L104) `acAcPermissionsMoveOnlyForAWard`<br>Only an admin can grant or revoke admin rights on Accounting. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4025) |
| [AC-AC-07](./specs/core/hub/Accounting/single/Accounting_single_pool_access_control.spec#L106-L122) `acAcJournalNumberingAdvancesOnlyForAWard`<br>Only an admin's call can advance a pool's journal numbering. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4024) |
| [AC-AC-08](./specs/core/hub/Accounting/single/Accounting_single_pool_access_control.spec#L124-L137) `acAcSessionIsLeftOpenOnlyForAWard`<br>Only an admin can leave accounting unlocked for posting. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-4024) |

### Holdings

- Every configuration
  - Loops running up to 4 iterations
- Single pool
  - One symbolic pool
  - One symbolic share class
  - Holding rows bounded to 2 symbolic assets
  - Both deficit counters pinned to one symbolic network
- Multi pool
  - Two symbolic pools
  - One share class per pool
  - Holding rows bounded to 2 symbolic assets per share class
  - Both deficit counters pinned to one symbolic network
- Multi share class
  - Two share classes in the pinned pool
  - Holding rows bounded to 2 symbolic assets per share class
  - Both deficit counters pinned to one symbolic network
- Multi network
  - One symbolic pool
  - One symbolic share class
  - Holding rows bounded to 2 symbolic assets
  - Two symbolic networks; a third reaches no mirrored bucket
- Census
  - Any one symbolic pool, and every share class, asset and network of it
  - The network a deposit or withdrawal is reported under taken to be the asset's own network

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [HO-VS-01](./specs/core/hub/Holdings/single/Holdings_single_pool_valid_state.spec#L15-L18) `snapshotHookImpliesPoolExists`<br>A snapshot hook is attached only to a pool the hub registry knows. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-802) [🎯](./mutations/Holdings/README.md#holdings-803) |
| [HO-VS-02](./specs/core/hub/Holdings/single/Holdings_single_pool_valid_state.spec#L20-L23) `networkDeficitCountMatchesTheShortRows`<br>With one network, a pool's deficit count equals its holdings in deficit. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-804) [🎯](./mutations/Holdings/README.md#holdings-805) [🎯](./mutations/Holdings/README.md#holdings-806) [🎯](./mutations/Holdings/README.md#holdings-807) |
| [HO-VS-03](./specs/core/hub/Holdings/single/Holdings_single_pool_valid_state.spec#L25-L28) `uninitializedHoldingHasNoValue`<br>A holding that was never opened has no value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9048) |
| [HO-VS-04](./specs/core/hub/Holdings/single/Holdings_single_pool_valid_state.spec#L30-L35) `emptyHoldingCarriesNoValue`<br>A holding with nothing left in it has zero value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-810) [🎯](./mutations/Holdings/README.md#holdings-811) |
| [HO-VS-05](./specs/core/hub/Holdings/single/Holdings_single_pool_valid_state.spec#L37-L40) `snapshotFlagImpliesAStartedNonce`<br>A network marked in sync has had at least one sync update. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-814) [🎯](./mutations/Holdings/README.md#holdings-815) |
| [HO-VS-06](./specs/core/hub/Holdings/single/Holdings_single_pool_valid_state.spec#L42-L45) `accountIdImpliesInitialized`<br>Journal accounts are set only on holdings that have been opened. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-812) [🎯](./mutations/Holdings/README.md#holdings-813) |
| [HO-VS-07](./specs/core/hub/Holdings/Holdings_multi_share_class_properties.spec#L12-L15) `networkDeficitCountSpansEveryShareClass`<br>A pool's deficit count on a network includes shortfalls from every share class. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-825) |
| [HO-VS-08](./specs/core/hub/Holdings/single/Holdings_single_pool_valid_state.spec#L47-L50) `deficitCountMatchesTheClassShortRows`<br>With one network, a share class's deficit count equals its holdings in deficit. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8000) |
| [HO-VS-09](./specs/core/hub/Holdings/Holdings_multi_share_class_properties.spec#L17-L21) `deficitCountIsKeptPerShareClass`<br>Each share class's deficit count covers only its own holdings in shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8001) |
| [HO-VS-10](./specs/core/hub/Holdings/Holdings_multi_network_properties.spec#L6-L12) `deficitCountsSumToTheShortRowsAcrossTheNetworks`<br>Summed over networks, the deficit counts match the holdings in shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8007) |
| [HO-VS-11](./specs/core/hub/Holdings/Holdings_census_properties.spec#L6-L9) `networkDeficitCountIsItsNetworksShortRowCensus`<br>A pool's deficit count on a network is the number of its holdings there in shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9049) |
| [HO-VS-12](./specs/core/hub/Holdings/Holdings_census_properties.spec#L11-L14) `deficitCountIsItsClassNetworksShortRowCensus`<br>A share class's deficit count on a network is the number of its holdings there in shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9058) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [HO-ST-01](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L120-L133) `stHoInflowAndOutflowNeverMoveTogether`<br>No single call changes both a holding's deposit and withdrawal totals. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-816) |
| [HO-ST-02](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L135-L164) `stHoOneCallTouchesOneHolding`<br>A call changing one holding leaves the share class's other holdings unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-817) |
| [HO-ST-03](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L166-L179) `stHoInflowNeverLowersValue`<br>An inflow never lowers a holding's value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-818) |
| [HO-ST-04](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L181-L194) `stHoOutflowNeverRaisesValue`<br>An outflow never raises a holding's value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-819) |
| [HO-ST-05](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L196-L214) `stHoOpeningAHoldingFundsNothing`<br>Opening a holding books no assets and no value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-820) |
| [HO-ST-06](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L216-L229) `stHoSnapshotFlagMovesWithItsNonce`<br>A network's sync flag changes only as its sequence number advances by one. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-821) |
| [HO-ST-07](./specs/core/hub/Holdings/Holdings_multi_pool_properties.spec#L6-L33) `stHoHoldingRowIsPerPool`<br>A call that changes one pool's holding leaves other pools' holdings unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-822) |
| [HO-ST-08](./specs/core/hub/Holdings/Holdings_multi_pool_properties.spec#L35-L50) `stHoAccountWiringIsPerPool`<br>Setting one pool's journal accounts leaves other pools' accounts unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-822) |
| [HO-ST-09](./specs/core/hub/Holdings/Holdings_multi_pool_properties.spec#L52-L71) `stHoSnapshotRowIsPerPool`<br>A pool's snapshot change never touches another pool's snapshot on that network. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-823) |
| [HO-ST-10](./specs/core/hub/Holdings/Holdings_multi_pool_properties.spec#L73-L94) `stHoDeficitCountAndHookIgnoreASiblingsFlow`<br>A deposit or withdrawal never changes another pool's deficit counts or NAV hook. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-824) [🎯](./mutations/Holdings/README.md#holdings-8105) |
| [HO-ST-11](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L231-L248) `stHoSnapshotRowIsPerNetwork`<br>A sync update to one network leaves every other network's sync state unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8100) |
| [HO-ST-12](./specs/core/hub/Holdings/Holdings_multi_share_class_properties.spec#L25-L44) `stHoFlowRowIsPerShareClass`<br>A deposit or withdrawal on one share class leaves another's amounts unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8101) |
| [HO-ST-13](./specs/core/hub/Holdings/Holdings_multi_share_class_properties.spec#L46-L79) `stHoHoldingRowIsPerShareClass`<br>A call changing any part of one class's holding leaves the other class's holdings unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9082) |
| [HO-ST-14](./specs/core/hub/Holdings/Holdings_multi_network_properties.spec#L16-L34) `stHoDeficitCountsOfAtMostOneNetworkMovePerCall`<br>A call moves the deficit counts of at most one network. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9012) |
| [HO-ST-15](./specs/core/hub/Holdings/Holdings_multi_share_class_properties.spec#L81-L99) `stHoSnapshotRowIsPerShareClass`<br>A call changing one class's snapshot on a network leaves another class's snapshot there unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9013) |
| [HO-ST-16](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L250-L265) `stHoDeficitCountersMoveOnlyThroughTheFlowEntries`<br>A pool's deficit counts change only through deposits and withdrawals. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8006) |
| [HO-ST-17](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L267-L280) `stHoSyncNotificationOnlyUnderAnInSyncMarker`<br>Any call notifies the NAV hook at most once, and only for a network marked in sync. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9081) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [HO-VT-01](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L5-L18) `vtHoDepositHistoryNeverShrinks`<br>A holding's lifetime deposit total never decreases. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4117) |
| [HO-VT-02](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L20-L33) `vtHoWithdrawalHistoryNeverShrinks`<br>A holding's lifetime withdrawal total never decreases. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4117) |
| [HO-VT-03](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L35-L48) `vtHoOpenedHoldingIsNeverClosed`<br>A holding the pool has opened can never be closed again. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4118) |
| [HO-VT-04](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L50-L64) `vtHoSnapshotNonceStepsByAtMostOne`<br>A network's sync sequence number advances by at most one per call. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4119) |
| [HO-VT-05](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L66-L82) `vtHoAccountWiringMovesOnlyThroughItsTwoWriters`<br>A holding's journal accounts change only through initialize and setAccountId. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4115) |
| [HO-VT-06](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L84-L99) `vtHoSnapshotHookMovesOnlyThroughItsSetter`<br>A pool's NAV hook changes only through setSnapshotHook. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4116) |
| [HO-VT-07](./specs/core/hub/Holdings/single/Holdings_single_pool_transitions.spec#L101-L116) `vtHoSnapshotSequenceMovesOnlyThroughItsPublisher`<br>A network's snapshot nonce changes only through setSnapshot. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9032) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [HO-HL-01](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L3-L18) `hlHoDepositHistoryRecordsTheInflowAndSurvivesTheOutflow`<br>A deposit is recorded in full and stays so after an equal withdrawal. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4108) [🎯](./mutations/Holdings/README.md#holdings-4111) |
| [HO-HL-02](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L20-L35) `hlHoWithdrawalHistoryRecordsTheWholeOutflow`<br>A withdrawal following an equal deposit is recorded at its full amount. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4111) |
| [HO-HL-03](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L37-L51) `hlHoIncreaseReturnsTheValueItBooked`<br>A deposit reports exactly the value it adds to the holding. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4112) |
| [HO-HL-04](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L53-L67) `hlHoDecreaseReturnsTheValueItRemoved`<br>A withdrawal reports exactly the value it removes from the holding. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4113) |
| [HO-HL-05](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L69-L84) `hlHoUpdateReportsTheRevaluationItWrote`<br>A revaluation reports exactly the gain or loss it applies to the holding. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4114) |
| [HO-HL-06](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L86-L99) `hlHoPublishingASyncMarkerAlwaysAdvancesTheSequence`<br>Each accepted snapshot update advances the network's snapshot nonce by one. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4110) |
| [HO-HL-07](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L101-L118) `hlHoSyncNotificationCarriesTheMarkersOwnKeys`<br>A snapshot update notifies the NAV hook, with its class and network, exactly when in sync. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8003) |
| [HO-HL-08](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L120-L139) `hlHoReplayNotifiesUnderTheMarkersOwnKeys`<br>A snapshot replay notifies the NAV hook, with its class and network, exactly when in sync. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8102) |
| [HO-HL-09](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L141-L160) `hlHoDecreaseRemovesValueProRataToTheRealizedAmount`<br>A withdrawal removes value in proportion to the amount it takes out, rounded down. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9080) |
| [HO-HL-10](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L162-L184) `hlHoIncreaseQuotesOnlyTheRealizedAmount`<br>A deposit books one quote from the holding's source on the part past any shortfall, else zero. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9470) |
| [HO-HL-11](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L186-L207) `hlHoUpdateMarksTheBookToTheQuoteOfItsAmount`<br>A revaluation carries one quote of the unchanged amount from the holding's own source. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9472) |
| [HO-HL-12](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L209-L220) `hlHoDecreaseNeverConsultsTheOracle`<br>A withdrawal never asks a price source for a quote. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9473) |
| [HO-HL-13](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L222-L235) `hlHoPriceSourceSwapLeavesTheCarryingValueStanding`<br>Switching a holding's price source takes no quote and leaves its value unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9475) |
| [HO-HL-14](./specs/core/hub/Holdings/Holdings_multi_network_properties.spec#L38-L62) `hlHoEnteringADeficitBooksTheNamedNetworkOnly`<br>A new shortfall raises the deficit counts of the withdrawal's network only. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8002) |
| [HO-HL-15](./specs/core/hub/Holdings/Holdings_multi_network_properties.spec#L64-L88) `hlHoLeavingADeficitReleasesTheNamedNetworkOnly`<br>An ended shortfall lowers the deficit counts of the deposit's network only. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8004) |
| [HO-HL-16](./specs/core/hub/Holdings/Holdings_multi_pool_properties.spec#L98-L115) `hlHoSetSnapshotBumpsTheNamedPoolsNonce`<br>Setting a snapshot bumps its pool's nonce and moves no other pool's snapshot on that network. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9084) |
| [HO-HL-17](./specs/core/hub/Holdings/Holdings_multi_share_class_properties.spec#L103-L120) `hlHoSetSnapshotBumpsTheNamedClassesNonce`<br>Setting a snapshot bumps its class's nonce and moves no other class's snapshot on that network. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9083) |
| [HO-HL-18](./specs/core/hub/Holdings/single/Holdings_single_pool_high_level.spec#L237-L254) `hlHoHoldingViewReportsTheFlooredNetOfWideTotals`<br>A holding reports its amount as inflow minus outflow, or zero when overdrawn. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8418) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [HO-RV-01](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L3-L18) `rvHoInitializeRefusesANonWardANullSourceOrAnOpenRow`<br>Opening a holding fails for a non-admin, with no price source, or if already open. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4101) |
| [HO-RV-02](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L20-L34) `rvHoSetAccountIdRefusesANonWardOrAnUnopenedHolding`<br>Setting a journal account fails for a non-admin or on an unopened holding. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4102) |
| [HO-RV-03](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L36-L51) `rvHoUpdateValuationRefusesANonWardANullSourceOrAnUnopenedHolding`<br>Switching a price source fails for a non-admin, a zero source, or an unopened holding. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4103) |
| [HO-RV-04](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L53-L66) `rvHoSetSnapshotHookRefusesANonWardOrAnUnknownPool`<br>Attaching a snapshot hook fails for a non-admin or for a pool unknown to the hub. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4104) |
| [HO-RV-05](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L68-L82) `rvHoSetSnapshotRefusesANonWardOrAnOutOfOrderNonce`<br>A sync update fails for a non-admin or with an out-of-order sequence number. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4105) |
| [HO-RV-06](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L84-L98) `rvHoCallOnSyncSnapshotIsNeverRefusedToAWard`<br>Holdings itself never rejects an admin resending a network's sync notification. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4106) |
| [HO-RV-07](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L100-L114) `rvHoCallOnTransferSnapshotIsNeverRefusedToAWard`<br>With no snapshot hook set, an admin can always notify a cross-network transfer. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4107) |
| [HO-RV-08](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L116-L128) `rvHoIncreaseRefusesANonWard`<br>Booking an inflow to a holding fails for a non-admin. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7207) |
| [HO-RV-09](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L130-L147) `rvHoIncreaseBeforeTheRowIsOpenedIsNeverRefusedToAWard`<br>With one network, an admin's inflow to an unopened holding fails only on overflow. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4100) |
| [HO-RV-10](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L149-L165) `rvHoDecreaseNeverStallsOnAnOversizedOutflow`<br>An outflow larger than the holding never fails for an admin, barring overflow. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4100) |
| [HO-RV-11](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L167-L179) `rvHoValuationRefusesAnUnopenedHolding`<br>An unopened holding has no readable price source, not even the zero address. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4109) |
| [HO-RV-12](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L181-L193) `rvHoCallOnSyncSnapshotRefusesANonWard`<br>Resending a network's sync notification fails for a non-admin. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7208) |
| [HO-RV-13](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L195-L207) `rvHoCallOnTransferSnapshotRefusesANonWard`<br>Notifying a cross-network share transfer fails for a non-admin. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7209) |
| [HO-RV-14](./specs/core/hub/Holdings/Holdings_multi_network_properties.spec#L92-L115) `rvHoLeavingADeficitIsRefusedWhenTheNamedNetworkCountsNone`<br>A deposit ending a shortfall fails if its network has no shortfall counted. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8008) |
| [HO-RV-15](./specs/core/hub/Holdings/Holdings_multi_share_class_properties.spec#L124-L137) `rvHoIncreaseBeforeOpeningIsNeverRefusedOverASiblingClassShortfall`<br>An admin's inflow to an unopened holding fails only on overflow, even if another class is short. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8103) |
| [HO-RV-16](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L209-L225) `rvHoUpdateRefusesOnlyANonWardAnUnopenedRowOrAnOversizedNet`<br>A revaluation fails exactly for a non-admin, an unopened holding, or a net amount past uint128. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4100) [🎯](./mutations/Holdings/README.md#holdings-6617) [🎯](./mutations/Holdings/README.md#holdings-6618) [🎯](./mutations/Holdings/README.md#holdings-8415) |
| [HO-RV-17](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L227-L243) `rvHoAmountViewsRefuseOnlyANetPastUint128`<br>Reading a holding's amount, alone or with its value, fails exactly when its net exceeds uint128. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4100) [🎯](./mutations/Holdings/README.md#holdings-8416) [🎯](./mutations/Holdings/README.md#holdings-8417) |
| [HO-RV-18](./specs/core/hub/Holdings/single/Holdings_single_pool_reverts.spec#L245-L264) `rvHoIncreaseOnAnOpenHoldingFailsOnlyOnAnOverflowingTotalOrValue`<br>With one network, an admin's inflow to an opened holding fails only on a total or value overflow. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-4100) [🎯](./mutations/Holdings/README.md#holdings-8811) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [HO-RC-01](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L3-L22) `rcHoOpeningThePoolsFirstHoldingIsReachable`<br>A pool can open the first holding of a share class. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6600) |
| [HO-RC-02](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L24-L39) `rcHoOpeningWiringAllFourLedgerAccountsIsReachable`<br>Opening a holding can set all four of its journal accounts in one call. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6601) |
| [HO-RC-03](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L41-L57) `rcHoOpeningASecondHoldingIsReachable`<br>A second holding can be opened beside an existing one, leaving it unchanged. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6602) |
| [HO-RC-04](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L59-L75) `rcHoBookingADepositIsReachable`<br>A deposit can be booked on an open holding and add value to it. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6603) |
| [HO-RC-05](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L77-L95) `rcHoZeroDepositIsReachable`<br>An empty deposit on an open holding can go through without changing it. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6605) |
| [HO-RC-06](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L97-L115) `rcHoDepositPricedAtNothingIsReachable`<br>A deposit the price source values at zero can still grow the holding's amount. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6606) |
| [HO-RC-07](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L117-L135) `rcHoBookingADepositBeforeTheRowIsOpenedIsReachable`<br>A deposit can be booked on a holding not yet opened, adding no value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6607) |
| [HO-RC-08](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L137-L157) `rcHoClearingADeficitExactlyIsReachable`<br>A deposit can exactly offset an earlier over-withdrawal, clearing the shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6608) [🎯](./mutations/Holdings/README.md#holdings-8070) |
| [HO-RC-09](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L159-L175) `rcHoBookingAWithdrawalIsReachable`<br>A withdrawal can be booked on a funded holding and remove value from it. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6609) |
| [HO-RC-10](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L177-L200) `rcHoWithdrawingPastThePositionIsReachable`<br>A withdrawal can exceed the holding, leaving it at zero and in shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6610) [🎯](./mutations/Holdings/README.md#holdings-8070) |
| [HO-RC-11](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L202-L221) `rcHoClosingThePositionExactlyIsReachable`<br>A withdrawal can close a holding at exactly zero without a shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6611) |
| [HO-RC-12](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L223-L242) `rcHoDeepeningADeficitIsReachable`<br>A further withdrawal can deepen a shortfall without counting the holding twice. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6612) |
| [HO-RC-13](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L244-L262) `rcHoDeficitCountCarryingASecondShortRowIsReachable`<br>Two holdings of one share class can be in shortfall at the same time. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6613) [🎯](./mutations/Holdings/README.md#holdings-8070) |
| [HO-RC-14](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L264-L283) `rcHoBookingAWithdrawalBeforeTheRowIsOpenedIsReachable`<br>A withdrawal can be booked on a holding not yet opened, removing no value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6614) |
| [HO-RC-15](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L285-L300) `rcHoRevaluationUpwardIsReachable`<br>A revaluation can raise a holding's value and report the gain. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6615) |
| [HO-RC-16](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L302-L318) `rcHoRevaluationDownwardIsReachable`<br>A revaluation can lower a holding's value and report the loss. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6616) |
| [HO-RC-17](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L320-L336) `rcHoMarkingANetworkInSyncIsReachable`<br>A network can be marked in sync, advancing its sequence number by one. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6619) |
| [HO-RC-18](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L338-L354) `rcHoMarkingANetworkOutOfSyncIsReachable`<br>A network marked in sync can be marked out of sync again. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6620) |
| [HO-RC-19](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L356-L376) `rcHoSyncAndDesyncRoundTripIsReachable`<br>A network can be marked in sync and back out of sync in one transaction. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6621) |
| [HO-RC-20](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L378-L390) `rcHoAttachingNavAutomationIsReachable`<br>A pool without a snapshot hook can have one attached. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6623) |
| [HO-RC-21](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L392-L409) `rcHoAutomationAttachAndDetachRoundTripIsReachable`<br>A pool's snapshot hook can be attached and removed again. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6624) |
| [HO-RC-22](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L411-L425) `rcHoRewiringALedgerAccountIsReachable`<br>A holding's journal account can be replaced with a different one. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6625) |
| [HO-RC-23](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L427-L448) `rcHoSwappingThePriceSourceIsReachable`<br>A holding can switch price source without changing its amount or value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6626) |
| [HO-RC-24](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L450-L467) `rcHoWardGrantAndRevokeRoundTripIsReachable`<br>Admin rights on Holdings can be granted and revoked in one transaction. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6627) |
| [HO-RC-25](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L469-L488) `rcHoOpeningAndFullyClosingAPositionIsReachable`<br>A holding can be opened, funded and emptied, and still show its lifetime totals. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6628) |
| [HO-RC-26](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L490-L513) `rcHoEnteringAndLeavingADeficitIsReachable`<br>A holding can enter and leave a deficit, and both deficit counts clear again. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-6629) [🎯](./mutations/Holdings/README.md#holdings-8070) |
| [HO-RC-27](./specs/core/hub/Holdings/Holdings_multi_share_class_properties.spec#L141-L156) `rcHoTwoShareClassesShortOnOneNetworkIsReachable`<br>Two share classes of a pool can both be in shortfall on one network at once. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8005) |
| [HO-RC-28](./specs/core/hub/Holdings/Holdings_multi_network_properties.spec#L119-L153) `rcHoShortfallAttributionCanOutliveItsNetworkIsReachable`<br>A shortfall can stay counted on a network after a deposit elsewhere clears it. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8009) |
| [HO-RC-29](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L515-L537) `rcHoDecreaseBooksOffTheQuoteJustTaken`<br>An amount just deposited can be withdrawn at a value other than the quote it was booked at. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9474) |
| [HO-RC-30](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L539-L554) `rcHoNettingInflowPricesOnlyItsPositivePart`<br>A deposit into a holding in shortfall can be priced only on the part past the shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9471) |
| [HO-RC-31](./specs/core/hub/Holdings/single/Holdings_single_pool_reachability.spec#L556-L574) `rcHoStaleValueSurvivesAPriceSourceSwapOnALivePosition`<br>A funded holding can switch price source and keep its old value, with no quote taken. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9476) |
| [HO-RC-32](./specs/core/hub/Holdings/Holdings_census_properties.spec#L18-L38) `rcHoCensusReachesThreeShortRowsOnOneNetwork`<br>A pool can count three holdings of two share classes in shortfall on one network. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9522) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [HO-AC-01](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L3-L17) `acHoWardRightsMoveOnlyForAWard`<br>Only an admin can grant or revoke admin rights on Holdings. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7206) |
| [HO-AC-02](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L19-L34) `acHoDepositHistoryMovesOnlyForAWard`<br>Only an admin can change a holding's recorded deposits. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7200) |
| [HO-AC-03](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L36-L51) `acHoWithdrawalHistoryMovesOnlyForAWard`<br>Only an admin can change a holding's recorded withdrawals. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7201) |
| [HO-AC-04](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L53-L67) `acHoCarryingValueMovesOnlyForAWard`<br>Only an admin can change a holding's carrying value. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7200) |
| [HO-AC-05](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L69-L84) `acHoPriceSourceMovesOnlyForAWard`<br>Only an admin can set or change the price source a holding is valued with. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7203) |
| [HO-AC-06](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L86-L101) `acHoJournalWiringMovesOnlyForAWard`<br>Only an admin can set or change a holding's journal accounts. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7202) |
| [HO-AC-07](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L103-L117) `acHoSnapshotHookMovesOnlyForAWard`<br>Only an admin can set or change a pool's NAV hook. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7204) |
| [HO-AC-08](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L119-L133) `acHoSyncMarkerMovesOnlyForAWard`<br>Only an admin can mark a network's assets and shares as in sync or not. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7205) |
| [HO-AC-09](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L135-L149) `acHoSnapshotNonceMovesOnlyForAWard`<br>Only an admin can advance a network's snapshot nonce. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7205) |
| [HO-AC-10](./specs/core/hub/Holdings/single/Holdings_single_pool_access_control.spec#L151-L168) `acHoDeficitCountMovesOnlyForAWard`<br>Only an admin can change a pool's deficit counts. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-7200) [🎯](./mutations/Holdings/README.md#holdings-8104) |

### ShareClassManager

- Single pool
  - One symbolic pool
  - One symbolic share class
- Multi pool
  - Two symbolic pools
  - One share class per pool, its first
- Multi share class
  - Two share classes in the pinned pool
  - Networks bounded to 3 symbolic networks
- Unpinned
  - Any symbolic pool, share class and network

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [SC-VS-01](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L21-L24) `shareClassExistsIffCounted`<br>A pool's first share class exists exactly when the pool has created a class. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-401) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-402) |
| [SC-VS-02](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L26-L29) `shareClassExistsIffSaltSet`<br>A share class exists exactly when its salt is set. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-402) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-403) |
| [SC-VS-03](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L31-L34) `uncreatedClassNetworkIssuancesZero`<br>A share class never created has no issuance on any network. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-407) |
| [SC-VS-04](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L36-L39) `uncreatedClassNetworkRevocationsZero`<br>A share class never created has no revocations on any network. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-407) |
| [SC-VS-05](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L41-L44) `uncreatedClassPriceZero`<br>A share class never created has no share price. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-408) |
| [SC-VS-06](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L46-L49) `uncreatedClassPriceStampZero`<br>A share class never created has no price timestamp. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-408) |
| [SC-VS-07](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L51-L54) `existingClassPoolRegistered`<br>A share class only exists inside a pool the hub registry knows. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-406) |
| [SC-VS-08](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L56-L59) `totalIssuanceIsNetworkNetSum`<br>A class's total supply is the sum of every network's net issuance. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-411) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-449) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-491) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6908) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9055) |
| [SC-VS-09](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L61-L64) `priceStampNotFuture`<br>Price timestamps never run ahead of the chain clock. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-409) |
| [SC-VS-10](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L66-L70) `saltPrefixIsPool`<br>Every share class salt starts with its pool's id. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-405) |
| [SC-VS-11](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L72-L75) `classSaltConsumed`<br>A share class's salt is always marked as used. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-404) |
| [SC-VS-12](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L24-L28) `classSaltSetImpliesRegisteredPerClass`<br>Only a registered share class can have a salt. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-418) |
| [SC-VS-13](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L30-L34) `registeredClassIndexWithinCount`<br>A registered share class never has an index beyond the pool's class count. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-419) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-420) |
| [SC-VS-14](./specs/core/hub/ShareClassManager/ShareClassManager_multi_pool_properties.spec#L16-L22) `poolSaltPrefixIsOwnPool`<br>A pool's share class salt always begins with that pool's own id. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-421) |
| [SC-VS-15](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L36-L45) `classSaltsDistinct`<br>Two share classes of a pool never have the same salt. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8951) |
| [SC-VS-16](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L47-L53) `classSaltConsumedPerClass`<br>A share class's salt is always recorded as used. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-422) |
| [SC-VS-17](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L55-L59) `classWithinCountIsRegistered`<br>Every index up to the pool's class count belongs to a registered share class. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-418) |
| [SC-VS-18](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L61-L65) `registeredClassCarriesASalt`<br>Every registered share class has a salt. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-423) |
| [SC-VS-19](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L67-L71) `classTotalIssuanceIsNetworkNetSum`<br>A share class's total supply is its net issuance summed over all networks. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-424) |
| [SC-VS-20](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L73-L84) `classUncreatedRowsZero`<br>A share class that was never created has no supply, price, or network shares. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-425) |
| [SC-VS-21](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L77-L80) `nullSaltNeverConsumed`<br>The empty salt is never marked used. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-405) |
| [SC-VS-22](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_valid_state.spec#L82-L85) `negativeNetworkCountMatchesTheShortNetworks`<br>The negative-network count equals the networks that revoked more than they issued. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8011) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8012) |
| [SC-VS-23](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L86-L90) `classNegativeNetworkCountMatchesItsShortNetworks`<br>A share class correctly counts its networks that revoked more than they issued. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8013) |
| [SC-VS-24](./specs/core/hub/ShareClassManager/ShareClassManager_unpinned_properties.spec#L5-L10) `recordedClassIdSitsUnderItsPoolWithinCount`<br>A registered share class id carries its own pool and an index from one to the pool's class count. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9051) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9052) |
| [SC-VS-25](./specs/core/hub/ShareClassManager/ShareClassManager_unpinned_properties.spec#L12-L22) `classIdIsOnRecordUnderOnePoolOnly`<br>A share class id is registered under at most one pool. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9052) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SC-ST-01](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_transitions.spec#L67-L82) `stScCreationClaimsAFreeIdentity`<br>A new share class takes an unused salt and marks it used. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-487) |
| [SC-ST-02](./specs/core/hub/ShareClassManager/ShareClassManager_multi_pool_properties.spec#L26-L64) `stScClassRowIsPerPool`<br>A change to one pool's share class leaves another pool's share class untouched. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-415) |
| [SC-ST-03](./specs/core/hub/ShareClassManager/ShareClassManager_multi_pool_properties.spec#L66-L85) `stScNetworkLedgerIsPerPool`<br>A pool's share changes on a network never touch another pool's shares there. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-415) |
| [SC-ST-04](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L94-L131) `stScClassRowIsPerClass`<br>A change to one share class leaves the pool's other share class untouched. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-416) |
| [SC-ST-05](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L133-L152) `stScNetworkLedgerIsPerClass`<br>A class's share changes on a network never touch another class's shares there. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-416) |
| [SC-ST-06](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L154-L166) `stScClassCountStepsByOne`<br>A pool's share class count never decreases and grows by at most one per call. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-417) |
| [SC-ST-07](./specs/core/hub/ShareClassManager/ShareClassManager_multi_share_class_properties.spec#L168-L181) `stScClassSaltIsWriteOnce`<br>A share class's salt never changes once set. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-481) |
| [SC-ST-08](./specs/core/hub/ShareClassManager/ShareClassManager_unpinned_properties.spec#L26-L37) `stScRecordedClassStaysRecorded`<br>A registered share class stays registered under its pool through every call. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9053) |
| [SC-ST-09](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_transitions.spec#L84-L106) `stScClassIdentityMovesOnlyOnCreation`<br>Salts, the class count and class records change only when a share class is added. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5008) |
| [SC-ST-10](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_transitions.spec#L108-L126) `stScSharePriceMovesOnlyOnRepricing`<br>A share class's price and its timestamp change only through a price update. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5008) |
| [SC-ST-11](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_transitions.spec#L128-L153) `stScClassTotalsMoveOnlyOnASupplyUpdateNetworkLegsAlsoOnATransfer`<br>Class totals change only on a share update, network supply also on a transfer. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-414) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8043) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8420) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SC-VT-01](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_transitions.spec#L5-L18) `vtScNetworkIssuedTotalNeverDecreases`<br>A network's cumulative issued shares never decrease. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5004) |
| [SC-VT-02](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_transitions.spec#L20-L33) `vtScNetworkRevokedTotalNeverDecreases`<br>A network's cumulative revoked shares never decrease. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5004) |
| [SC-VT-03](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_transitions.spec#L35-L48) `vtScClassIssuedTotalNeverDecreases`<br>A class's cumulative issued shares never decrease. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8017) |
| [SC-VT-04](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_transitions.spec#L50-L63) `vtScClassRevokedTotalNeverDecreases`<br>A class's cumulative revoked shares never decrease. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8018) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [SC-HL-01](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L3-L18) `hlScCrossChainTransferConservesClassSupply`<br>Equal revocation and issuance on two networks leave the class supply unchanged. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5005) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6916) |
| [SC-HL-02](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L20-L30) `hlScCreationHandsOutThePreviewedId`<br>Adding a share class gives it the id previewed just before. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5009) |
| [SC-HL-03](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L32-L45) `hlScRepricingStoresTheGivenPriceAndStamp`<br>A price update stores exactly the given price and timestamp. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5010) |
| [SC-HL-04](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L47-L60) `hlScCreationMatchesTheIndexedPreview`<br>A new share class's id matches the preview for its position in the pool. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5009) |
| [SC-HL-05](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L62-L94) `hlScSupplyUpdateBooksOneLegOfTheNetworkAndTheClass`<br>A share update adds its amount only to the matching network and class totals. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-412) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5006) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5007) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6907) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6909) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6910) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6915) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6916) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8019) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8045) |
| [SC-HL-06](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L96-L108) `hlScSupplyViewsReportIssuedMinusRevoked`<br>Reported supply, in total and per network, is shares issued minus revoked. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8040) |
| [SC-HL-07](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L110-L118) `hlScShareClassIdIsInjectiveInPoolAndIndex`<br>Two different pool and index pairs never get the same share class id. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9050) |
| [SC-HL-08](./specs/core/hub/ShareClassManager/ShareClassManager_unpinned_properties.spec#L41-L57) `hlScCreationMintsTheNextFreshIndex`<br>Adding a share class bumps the pool's count and registers the next index's id, unregistered before. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9054) |
| [SC-HL-09](./specs/core/hub/ShareClassManager/ShareClassManager_unpinned_properties.spec#L59-L68) `hlScClassIdIsThePoolOverTheIndex`<br>A share class id is its pool in the high eight bytes over its index, zero only when both are zero. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9942) |
| [SC-HL-10](./specs/core/hub/ShareClassManager/ShareClassManager_unpinned_properties.spec#L70-L114) `hlScUpdateMetadataWritesOnlyItsClassesStrings`<br>UpdateMetadata rewrites only the named class's name and symbol; every other field reads the same. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9005) |
| [SC-HL-11](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L120-L136) `hlScAnUpdateMovesItsLegByItsAmount`<br>A supply update moves only its class's issued or revoked sum over every network, by its amount. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9057) |
| [SC-HL-12](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_high_level.spec#L138-L165) `hlScTransferSharesMovesOnlyTheTwoNetworkLegs`<br>A share transfer only revokes on the origin network and issues on the target. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8419) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8421) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [SC-RV-01](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L3-L24) `rvScAddShareClassRefusesACreationThatDoesNotCheckOut`<br>Adding a share class fails for a non-admin, unknown pool, bad or used salt, or bad metadata. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5012) |
| [SC-RV-02](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L26-L42) `rvScUpdateSharePriceRefusesANonWardAnUnknownClassOrAFutureStamp`<br>Setting a share price fails for a non-admin, an unknown class, or a future timestamp. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5013) |
| [SC-RV-03](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L44-L58) `rvScUpdateSharesRefusesANonWardOrAnUnknownClass`<br>A share supply update fails for a non-admin or an unknown share class. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5014) |
| [SC-RV-04](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L60-L74) `rvScIssuanceRefusesANetworkInRevocationDeficit`<br>A network's issuance query fails when its revocations exceed its issuances. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6918) |
| [SC-RV-05](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L76-L87) `rvScPreviewNextShareClassIdRefusesAPoolWhoseClassCounterIsFull`<br>Previewing the next share class id fails once a pool has used up all ids. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5016) |
| [SC-RV-06](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L89-L101) `rvScRelyRefusesANonWard`<br>Granting admin rights fails for a caller that is not an admin. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5011) |
| [SC-RV-07](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L103-L115) `rvScDenyRefusesANonWard`<br>Revoking admin rights fails for a caller that is not an admin. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5011) |
| [SC-RV-08](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L117-L138) `rvScUpdateSharesNeverRefusesAWardWhileTheCountersHaveRoom`<br>An admin's share update on a known class succeeds unless a counter overflows. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8014) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8423) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8424) |
| [SC-RV-09](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L140-L154) `rvScTotalIssuanceRefusesAClassInRevocationDeficit`<br>A class's total supply query fails when its revocations exceed its issuances. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8015) |
| [SC-RV-10](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L156-L178) `rvScRawCountersAlwaysAnswerWithTheLedger`<br>The issued and revoked totals of a class and of each network can always be read. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8200) |
| [SC-RV-11](./specs/core/hub/ShareClassManager/ShareClassManager_unpinned_properties.spec#L118-L133) `rvScUpdateMetadataRefusesANonWardAnUnknownClassOrBadLengths`<br>A metadata update fails for a non-admin, an unknown class, or an empty or too-long name or symbol. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8201) |
| [SC-RV-12](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L180-L193) `rvScTransferSharesRefusesANonWardOrAnUnknownClass`<br>A share transfer fails for a non-admin or an unknown share class. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8422) |
| [SC-RV-13](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L195-L214) `rvScTransferSharesNeverRefusesAWardWhileTheLegsHaveRoom`<br>An admin's share transfer on a known class succeeds unless a counter overflows. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8423) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8424) |
| [SC-RV-14](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L216-L228) `rvScIssuanceRefusesANetworkNetPastUint128`<br>A network's issuance query fails when its net issuance does not fit a uint128. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8425) |
| [SC-RV-15](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L230-L242) `rvScTotalIssuanceRefusesAClassNetPastUint128`<br>A class's total supply query fails when its net does not fit a uint128. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8426) |
| [SC-RV-16](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L244-L259) `rvScIssuanceAnswersANetworkNeitherInDeficitNorPastUint128`<br>A network's issuance query never fails unless it is in deficit or its net exceeds uint128. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5015) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6911) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8427) |
| [SC-RV-17](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reverts.spec#L261-L276) `rvScTotalIssuanceAnswersAClassNeitherInDeficitNorPastUint128`<br>A class's total supply query never fails unless it is in deficit or its net exceeds uint128. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8016) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8428) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [SC-RC-01](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L3-L24) `rcScFirstClassCreationIsReachable`<br>A pool can create its first share class, using up its salt. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5017) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6900) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6901) |
| [SC-RC-02](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L26-L39) `rcScPricePublishedAtTheCurrentBlockIsReachable`<br>A price computed in the current block can be published. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6902) |
| [SC-RC-03](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L41-L55) `rcScBackdatedPriceKeepsItsOwnStamp`<br>A price computed earlier can be published with its original timestamp. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6903) |
| [SC-RC-04](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L57-L73) `rcScOlderPriceReplacesTheStoredOne`<br>An older price can replace a more recent one. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6904) |
| [SC-RC-05](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L75-L88) `rcScPriceCanBeWrittenDownToZero`<br>A share class with a positive price can be marked down to zero. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6905) |
| [SC-RC-06](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L90-L104) `rcScClassSupplyCanReachItsStoredCeiling`<br>One issuance can take an empty share class to the largest readable supply. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-9065) |
| [SC-RC-07](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L106-L121) `rcScNetworkRevocationsCanExceedItsIssuances`<br>A network can revoke more than it issued while the class supply stays positive. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6912) |
| [SC-RC-08](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L123-L139) `rcScRelyThenDenyRoundTripIsReachable`<br>Admin rights can be granted and revoked again in one transaction. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6913) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-6914) |
| [SC-RC-09](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L141-L159) `rcScNegativeNetworkCountOpensAndCloses`<br>A network can go negative by revoking first and recover once its issuance arrives. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8041) |
| [SC-RC-10](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L161-L175) `rcScRefusedClassSupplyIsReachable`<br>A class's total supply query can fail when revocations exceed issuances. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8042) |
| [SC-RC-11](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_reachability.spec#L177-L193) `rcScNetworkNetPastUint128IsReachable`<br>An issuance can push a network past uint128, so its issuance query fails. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8429) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [SC-AC-01](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L3-L17) `acScOnlyAWardMovesTheWardSet`<br>Only an admin can grant or revoke admin rights on the share class manager. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5003) |
| [SC-AC-02](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L19-L33) `acScOnlyAWardMovesTheClassCounter`<br>Only an admin can change a pool's share class count. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5000) |
| [SC-AC-03](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L35-L50) `acScOnlyAWardMovesTheClassRoster`<br>Only an admin can register or remove a share class. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5000) |
| [SC-AC-04](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L52-L66) `acScOnlyAWardMovesTheSaltRegistry`<br>Only an admin can change whether a salt is marked used. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5000) |
| [SC-AC-05](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L68-L82) `acScOnlyAWardMovesTheClassSalt`<br>Only an admin can change a share class's salt. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5000) |
| [SC-AC-06](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L84-L101) `acScOnlyAWardMovesTheClassSupply`<br>Only an admin can change a share class's total issued or revoked shares. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5001) |
| [SC-AC-07](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L103-L117) `acScOnlyAWardMovesANetworkIssuedTotal`<br>Only an admin can change the shares recorded as issued on a network. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5001) |
| [SC-AC-08](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L119-L133) `acScOnlyAWardMovesANetworkRevokedTotal`<br>Only an admin can change the shares recorded as revoked on a network. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5001) |
| [SC-AC-09](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L135-L149) `acScOnlyAWardMovesTheSharePrice`<br>Only an admin can change a share class's price. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5002) |
| [SC-AC-10](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L151-L165) `acScOnlyAWardMovesThePriceTimestamp`<br>Only an admin can change the timestamp of a share class's price. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5002) |
| [SC-AC-11](./specs/core/hub/ShareClassManager/single/ShareClassManager_single_pool_access_control.spec#L167-L182) `acScOnlyAWardMovesTheNegativeNetworkCount`<br>Only an admin can change how many networks have revoked more shares than issued. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-5001) |

### HubRegistry

- Single pool
  - One symbolic pool
- Multi pool
  - Two symbolic pools
- Metadata
  - Any symbolic pool
  - Loops running up to 16 iterations
  - The verbatim read back claimed over writes of one or more whole 32 byte words

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [HR-VS-01](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L25-L28) `assetDecimalsCapped`<br>No asset has more than 18 decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1002) |
| [HR-VS-02](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L30-L33) `unregisteredAssetHasZeroDecimals`<br>An unregistered asset has zero decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1003) |
| [HR-VS-03](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L35-L38) `nullAssetNeverRegistered`<br>The empty asset id is never registered. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1004) |
| [HR-VS-04](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L40-L43) `poolCurrencyRegistered`<br>The pool's denomination currency is always a registered asset. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1005) |
| [HR-VS-05](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L45-L48) `managerImpliesPoolExists`<br>Nobody holds manager rights on a pool before that pool is created. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1006) |
| [HR-VS-06](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L50-L53) `zeroAddressNeverManager`<br>The zero address is never a pool manager. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1007) [🎯](./mutations/HubRegistry/README.md#hubregistry-1018) |
| [HR-VS-07](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L55-L58) `policyImpliesPoolExists`<br>A policy is only ever installed on a created pool. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1008) |
| [HR-VS-08](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L60-L63) `nonceImpliesPoolExists`<br>The policy nonce only ever advances on a created pool. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1008) |
| [HR-VS-09](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L65-L68) `policyImpliesNonzeroNonce`<br>An installed policy always has a non-zero policy nonce. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1009) |
| [HR-VS-10](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L70-L73) `bridgingHookImpliesPoolExists`<br>A bridging hook is only ever wired to a created pool. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1010) |
| [HR-VS-11](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L75-L78) `requestManagerImpliesPoolExists`<br>A hub request manager, on any network, is only ever wired to a created pool. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1011) [🎯](./mutations/HubRegistry/README.md#hubregistry-1042) |
| [HR-VS-12](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L80-L84) `pendingAuthAboveEnvFloor`<br>A pending authorization never matures before the earliest possible block time. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1012) |
| [HR-VS-13](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L86-L90) `pendingNeverUnderNullPolicy`<br>No authorization is ever scheduled while a pool has no policy installed. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1013) |
| [HR-VS-14](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L92-L100) `pendingNeverUnderZeroNonce`<br>No authorization is pending under policy nonce zero. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1014) |
| [HR-VS-15](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L102-L106) `pendingNeverAheadOfNonce`<br>No authorization is pending under a future policy nonce. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1015) |
| [HR-VS-16](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L108-L115) `pendingUnderCurrentNonceMatchesPolicy`<br>At the current nonce, all pending authorizations belong to the installed policy. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1009) [🎯](./mutations/HubRegistry/README.md#hubregistry-1013) |
| [HR-VS-17](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_valid_state.spec#L117-L120) `nullAuthIdNeverPending`<br>The empty authorization id is never pending. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1041) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [HR-ST-01](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L112-L125) `stHrPolicySwapOpensANewTenure`<br>Changing a pool's policy always advances its policy nonce by one. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1027) |
| [HR-ST-02](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L127-L142) `stHrNewTenureStartsWithNoSchedule`<br>A newly installed policy starts with no pending authorizations. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1039) |
| [HR-ST-03](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L144-L162) `stHrSchedulingMaturesAheadAndInTheLiveTenure`<br>A newly scheduled authorization matures in the future, under the current policy. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1028) |
| [HR-ST-04](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L164-L182) `stHrDischargeLeavesTheTenureStanding`<br>A pending authorization is cleared only under the current policy, without changing that policy. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1029) |
| [HR-ST-05](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L184-L196) `stHrPendingMovesOnlyOffOrOntoZero`<br>A pending authorization is created or cleared, never rescheduled in place. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1031) |
| [HR-ST-06](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L198-L209) `stHrCurrencyNeverClears`<br>A created pool never loses its currency, so it cannot stop existing. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1034) |
| [HR-ST-07](./specs/core/hub/HubRegistry/HubRegistry_multi_pool_properties.spec#L6-L33) `stHrPoolRowIsPerPool`<br>No call changes the currency, policy or bridging hook of two pools at once. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1023) |
| [HR-ST-08](./specs/core/hub/HubRegistry/HubRegistry_multi_pool_properties.spec#L35-L50) `stHrManagerRightsArePerPool`<br>No call changes manager rights on two pools at once. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1023) |
| [HR-ST-09](./specs/core/hub/HubRegistry/HubRegistry_multi_pool_properties.spec#L52-L67) `stHrRequestManagerIsPerPool`<br>No call changes the request manager of two pools on one network at once. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1024) |
| [HR-ST-10](./specs/core/hub/HubRegistry/HubRegistry_multi_pool_properties.spec#L69-L85) `stHrPendingAuthIsPerPool`<br>No call changes pending authorizations of two pools at once. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1025) |
| [HR-ST-11](./specs/core/hub/HubRegistry/HubRegistry_multi_pool_properties.spec#L87-L101) `stHrOperatorSeatNeedsItsOwnPoolsCurrency`<br>Manager rights are granted only on a registered pool or when creating it. | ✅ | [🎯](./mutations/Hub/README.md#hub-9034) |
| [HR-ST-12](./specs/core/hub/HubRegistry/HubRegistry_metadata_properties.spec#L5-L18) `stHrOnlySetMetadataMovesMetadata`<br>No registry method but setMetadata moves any pool's metadata. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9543) |
| [HR-ST-13](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L211-L227) `stHrRegisteredAssetStaysAndASetPrecisionIsFinal`<br>An asset stays registered, nonzero decimals never change, and only registration sets decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1032) [🎯](./mutations/HubRegistry/README.md#hubregistry-8410) [🎯](./mutations/HubRegistry/README.md#hubregistry-8809) |
| [HR-ST-14](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L229-L247) `stHrPoolCurrencySetPrecisionIsStable`<br>A pool's currency decimals never change once nonzero, and only asset registration sets them. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-1032) [🎯](./mutations/HubRegistry/README.md#hubregistry-1033) [🎯](./mutations/HubRegistry/README.md#hubregistry-8410) [🎯](./mutations/HubRegistry/README.md#hubregistry-8810) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [HR-VT-01](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L5-L22) `vtHrManagerRosterMovesOneAccountAtATime`<br>A call changes the manager rights of at most one account. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4902) |
| [HR-VT-02](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L24-L38) `vtHrTenureCounterAdvancesByAtMostOne`<br>Each call leaves a pool's policy nonce unchanged or advances it by one. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4903) |
| [HR-VT-03](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L40-L57) `vtHrRequestManagerMovesOneNetworkAtATime`<br>A call changes the hub request manager of at most one network. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4904) |
| [HR-VT-04](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L59-L76) `vtHrAuthLedgerMovesOneEntryAtATime`<br>A call changes at most one pending authorization. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4905) |
| [HR-VT-05](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L78-L92) `vtHrAuthLedgerMovesOnlyThroughItsThreeWriters`<br>Pending authorizations change only by scheduling, canceling or consuming. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4909) |
| [HR-VT-06](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_transitions.spec#L94-L108) `vtHrManagerRightsMoveOnlyThroughTheirTwoWriters`<br>Pool manager rights change only through pool creation or a manager update. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4910) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [HR-HL-01](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L5-L19) `hlHrAuthorizationIsSpentOnlyOnce`<br>An authorization can be consumed at most once. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4906) [🎯](./mutations/HubRegistry/README.md#hubregistry-8621) |
| [HR-HL-02](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L21-L35) `hlHrConsumeSpendsOnlyALiveEntry`<br>Consuming succeeds only for an authorization that is actually scheduled. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4906) [🎯](./mutations/HubRegistry/README.md#hubregistry-8623) |
| [HR-HL-03](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L37-L51) `hlHrConsumeWaitsOutTheVetoDelay`<br>An authorization cannot be consumed before its delay has passed. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4907) [🎯](./mutations/HubRegistry/README.md#hubregistry-6417) |
| [HR-HL-04](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L53-L68) `hlHrConsumeRefusesAStaleGrant`<br>An authorization cannot be consumed after its expiry window closes. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4908) |
| [HR-HL-05](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L70-L84) `hlHrCanceledAuthorizationNeverFires`<br>A canceled authorization cannot then be consumed. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4906) [🎯](./mutations/HubRegistry/README.md#hubregistry-8622) |
| [HR-HL-06](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L86-L102) `hlHrScheduledCallMaturesAfterExactlyThePolicyDelay`<br>An authorization matures after exactly the delay the pool's policy sets. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9037) |
| [HR-HL-07](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L104-L113) `hlHrPoolCreationSeatsTheAccountItNames`<br>Creating a pool makes the named account its manager. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4912) |
| [HR-HL-08](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L115-L126) `hlHrTokenPrecisionMatchesTheAssetRow`<br>The registry reports an asset's decimals as recorded at registration. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4913) |
| [HR-HL-09](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L128-L139) `hlHrSetPolicyInstallsExactlyTheAddressItNames`<br>Setting a policy installs it and voids pending authorizations, even if unchanged. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6423) [🎯](./mutations/HubRegistry/README.md#hubregistry-8620) |
| [HR-HL-10](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L141-L150) `hlHrAssetRegistrationRecordsThePrecisionItNames`<br>Registering an asset records it with exactly the precision the call names. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9020) |
| [HR-HL-11](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L152-L163) `hlHrPoolCreationRecordsTheCurrencyAndSoleManagerItNames`<br>Creating a pool records the currency it names and makes the named manager its only manager. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9021) |
| [HR-HL-12](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L165-L176) `hlHrRevokingTheOnlySeatLeavesALivePoolWithNoManager`<br>Removing the only manager of a newly created pool leaves the pool live with no manager. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9026) |
| [HR-HL-13](./specs/core/hub/HubRegistry/HubRegistry_metadata_properties.spec#L22-L36) `hlHrSetMetadataStoresTheBytesVerbatim`<br>Metadata written as one or more whole 32-byte words reads back exactly as written. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9540) [🎯](./mutations/HubRegistry/README.md#hubregistry-9541) [🎯](./mutations/HubRegistry/README.md#hubregistry-9544) |
| [HR-HL-14](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L178-L190) `hlHrPoolIdIsTheChainOverTheLocalId`<br>A pool id is the network id above the local id, distinct per pair and nonzero on a nonzero network. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9940) |
| [HR-HL-15](./specs/core/hub/HubRegistry/HubRegistry_metadata_properties.spec#L38-L66) `hlHrSetMetadataWritesOnlyItsPoolsMetadata`<br>SetMetadata writes only the named pool's metadata; every other field reads the same under any key. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9541) [🎯](./mutations/HubRegistry/README.md#hubregistry-9542) |
| [HR-HL-16](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_high_level.spec#L192-L204) `hlHrPoolCreationLeavesThePoolWithoutAPolicy`<br>Creating a pool installs no policy and leaves its policy nonce at zero. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-8807) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [HR-RV-01](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L6-L22) `rvHrRegisterPoolRefusesAnUnusableOrRepeatedOpening`<br>Pool registration fails for a non-admin, no manager, an unregistered currency, or a repeat. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4915) |
| [HR-RV-02](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L24-L38) `rvHrUpdateManagerRefusesANonWardAnAbsentPoolOrANullSeat`<br>A manager update fails for a non-admin, a nonexistent pool, or the zero address. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4916) |
| [HR-RV-03](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L40-L59) `rvHrUpdateCurrencyRefusesANonWardAnAbsentPoolOrARescaling`<br>A currency change fails for a non-admin, missing pool, unregistered asset or different decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4917) |
| [HR-RV-04](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L61-L74) `rvHrSetPolicyRefusesANonWardOrAnAbsentPool`<br>Installing a policy fails for a non-admin or on a nonexistent pool. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-8950) |
| [HR-RV-05](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L76-L90) `rvHrSetHubRequestManagerRefusesANonWardOrAnAbsentPool`<br>Setting a hub request manager fails for a non-admin or on a nonexistent pool. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4919) |
| [HR-RV-06](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L92-L106) `rvHrSetBridgingHookIsNeverRefusedToAWardOnALivePool`<br>An admin can always set or clear an existing pool's bridging hook. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4920) |
| [HR-RV-07](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L108-L126) `rvHrInitiateAuthorizationRefusesANonWardAnInPolicyCallOrATakenSlot`<br>Scheduling a call fails for a non-admin, no policy, a call with no delay, or one already pending. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4921) |
| [HR-RV-08](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L128-L144) `rvHrCancelAuthorizationRefusesANonWardOrAnEmptySlot`<br>Cancelling an authorization fails for a non-admin or when none is pending. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4922) |
| [HR-RV-09](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L146-L167) `rvHrConsumeAuthorizationIsNeverRefusedToThePolicyInsideItsWindow`<br>The pool's policy can always consume a matured authorization until it expires. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4923) [🎯](./mutations/HubRegistry/README.md#hubregistry-6416) |
| [HR-RV-10](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L169-L180) `rvHrTokenDecimalsRefusesAnAssetTheRegistryDoesNotCarry`<br>Reading an unregistered asset's decimals fails rather than returning zero. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4924) |
| [HR-RV-11](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L182-L192) `rvHrAuthIdAlwaysAnswers`<br>Computing an authorization id never fails, even for a pool without a policy. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4925) |
| [HR-RV-12](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L194-L208) `rvHrRetiredGrantIsNeverConsumed`<br>A policy install voids earlier authorizations, even when it reinstalls the same policy. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4918) [🎯](./mutations/HubRegistry/README.md#hubregistry-8625) |
| [HR-RV-13](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L210-L221) `rvHrRelyRefusesANonWard`<br>Granting admin rights fails for a non-admin caller. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4928) |
| [HR-RV-14](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L223-L234) `rvHrDenyRefusesANonWard`<br>Revoking admin rights fails for a non-admin caller. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4928) |
| [HR-RV-15](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L236-L254) `rvHrRegisterAssetIsNeverRefusedToAWardOnAFreshAssetWithinTheCap`<br>An admin can always register a new, non-empty asset id with up to 18 decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4914) |
| [HR-RV-16](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L256-L268) `rvHrConsumeAuthorizationRefusesEveryCallerButTheInstalledPolicy`<br>Consuming an authorization fails for every caller but the pool's policy, admins included. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6416) [🎯](./mutations/HubRegistry/README.md#hubregistry-9022) |
| [HR-RV-17](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L270-L289) `rvHrRegisterPoolAdmitsAUsableOpeningOfAChainedPool`<br>An admin never fails to register a new pool of a nonzero network with a manager and known currency. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9941) |
| [HR-RV-18](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L291-L304) `rvHrAssetIdPrecisionReadAnswersTheRowOrRefuses`<br>Reading an asset's decimals fails exactly if it is unregistered or ETH is sent, else gives them. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9023) [🎯](./mutations/HubRegistry/README.md#hubregistry-9027) |
| [HR-RV-19](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L306-L319) `rvHrPoolPrecisionReadAnswersItsCurrencyRowOrRefuses`<br>Reading a pool's decimals fails exactly for a missing pool or sent ETH, else gives its currency's. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9024) [🎯](./mutations/HubRegistry/README.md#hubregistry-9028) |
| [HR-RV-20](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L321-L338) `rvHrRegisterAssetRefusesANonWardAnOversizedPrecisionOrARescale`<br>Asset registration fails for a non-admin, over 18 decimals, or changing nonzero decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4928) [🎯](./mutations/HubRegistry/README.md#hubregistry-8411) |
| [HR-RV-21](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L340-L358) `rvHrRegisterAssetIsNeverRefusedToAWardFillingOrKeepingARegisteredPrecision`<br>An admin can always re-register an asset with unchanged decimals, or up to 18 if they were zero. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4914) [🎯](./mutations/HubRegistry/README.md#hubregistry-8413) |
| [HR-RV-22](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L360-L375) `rvHrCancelAuthorizationIsNeverRefusedToAWardWithAPendingGrant`<br>An admin can always cancel an authorization pending under the current policy. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-8805) |
| [HR-RV-23](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reverts.spec#L377-L390) `rvHrSetBridgingHookRefusesANonWardOrAnAbsentPool`<br>Setting a bridging hook fails for a non-admin, a pool never created, or sent ETH. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4928) [🎯](./mutations/HubRegistry/README.md#hubregistry-8806) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [HR-RC-01](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L3-L16) `rcHrRegisterAssetIsReachable`<br>A new asset can be registered with its own decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6400) [🎯](./mutations/HubRegistry/README.md#hubregistry-6401) |
| [HR-RC-02](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L18-L31) `rcHrRegisterAssetAtThePrecisionCapIsReachable`<br>An asset can be registered with the maximum of 18 decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6400) |
| [HR-RC-03](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L33-L45) `rcHrRegisterAssetWithZeroPrecisionIsReachable`<br>An asset can be registered with zero decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6400) |
| [HR-RC-04](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L47-L64) `rcHrRegisterPoolIsReachable`<br>A pool can be created with its currency and first manager. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6402) [🎯](./mutations/HubRegistry/README.md#hubregistry-6403) |
| [HR-RC-05](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L66-L83) `rcHrRegisterAssetThenRegisterPoolIsReachable`<br>An asset and a pool in that currency can be set up from an empty registry. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6402) |
| [HR-RC-06](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L85-L96) `rcHrUpdateManagerSeatsAManagerIsReachable`<br>A new manager can be added to an existing pool. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6404) |
| [HR-RC-07](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L98-L109) `rcHrUpdateManagerRevokesASeatIsReachable`<br>A pool manager can have its rights revoked. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6404) |
| [HR-RC-08](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L111-L125) `rcHrUpdateCurrencyRedenominatesThePoolIsReachable`<br>An existing pool can switch to another currency with the same decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6405) |
| [HR-RC-09](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L127-L138) `rcHrSetBridgingHookAttachesIsReachable`<br>A bridging hook can be set on a pool that had none. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9038) |
| [HR-RC-10](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L140-L152) `rcHrSetBridgingHookDetachesIsReachable`<br>A pool's bridging hook can be removed. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6406) [🎯](./mutations/HubRegistry/README.md#hubregistry-9038) |
| [HR-RC-11](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L154-L168) `rcHrSetHubRequestManagerWiresANetworkIsReachable`<br>An admin can set a pool's request manager for a network that had none. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6408) |
| [HR-RC-12](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L170-L184) `rcHrSetPolicyInstallsTheFirstPolicyIsReachable`<br>A pool with no policy yet can have its first one installed. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6409) |
| [HR-RC-13](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L186-L201) `rcHrSetPolicyUninstallsWithTheZeroAddressIsReachable`<br>An installed policy can be removed, which also voids pending authorizations. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6410) |
| [HR-RC-14](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L203-L215) `rcHrSetPolicyAtTheLastNamespaceIsReachable`<br>A pool can still make its last allowed policy change. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6411) |
| [HR-RC-15](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L217-L238) `rcHrOrphanedScheduleCanBeRemadeAfterAPolicySwapIsReachable`<br>A policy change lets the same call be scheduled again under the new policy. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6412) |
| [HR-RC-16](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L240-L257) `rcHrInitiateAuthorizationAtTheMaturityCeilingIsReachable`<br>An authorization can mature at the latest time the registry can record. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6413) |
| [HR-RC-17](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L259-L274) `rcHrCancelAuthorizationIsReachable`<br>A pending authorization can be canceled before it matures. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6415) |
| [HR-RC-18](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L276-L293) `rcHrConsumeAuthorizationAtTheMaturityInstantIsReachable`<br>An authorization can be consumed at the exact moment it matures. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6418) |
| [HR-RC-19](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L295-L313) `rcHrConsumeAuthorizationAtTheExpiryEdgeIsReachable`<br>An authorization can be consumed at the last second of its expiry window. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6419) |
| [HR-RC-20](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L315-L331) `rcHrAuthorityRoundTripIsReachable`<br>Admin rights can be granted and revoked again in one transaction. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6420) [🎯](./mutations/HubRegistry/README.md#hubregistry-6421) |
| [HR-RC-21](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L333-L351) `rcHrPolicyViewsReportTheInstallIsReachable`<br>A new policy can be installed on a pool, advancing its policy nonce by one. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6411) [🎯](./mutations/HubRegistry/README.md#hubregistry-6422) |
| [HR-RC-22](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L353-L366) `rcHrAuthIdChangesAcrossAPolicySwapIsReachable`<br>A policy change can give an unchanged call a new authorization id. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6412) |
| [HR-RC-23](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L368-L382) `rcHrLivePoolLosesItsOnlyManagerIsReachable`<br>An admin can create a pool and then remove its only manager. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9025) |
| [HR-RC-24](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L384-L396) `rcHrRegisterAssetFillsAnUnsetPrecisionIsReachable`<br>An asset registered with zero decimals can later get nonzero decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6400) [🎯](./mutations/HubRegistry/README.md#hubregistry-8412) |
| [HR-RC-25](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_reachability.spec#L398-L410) `rcHrRegisterAssetRepeatsAtItsOwnPrecisionIsReachable`<br>An asset can be registered again with the same nonzero decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-6400) [🎯](./mutations/HubRegistry/README.md#hubregistry-8414) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [HR-AC-01](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L3-L17) `acHrWardRightsMoveOnlyForAWard`<br>Only an admin can grant or revoke admin rights on the hub registry. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4901) |
| [HR-AC-02](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L19-L37) `acHrAssetRowMovesOnlyForAWard`<br>Only an admin can register an asset or change its decimals. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4900) |
| [HR-AC-03](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L39-L54) `acHrPoolCurrencyMovesOnlyForAWard`<br>Only an admin can create a pool or change its currency. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4900) |
| [HR-AC-04](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L56-L70) `acHrManagerRightsMoveOnlyForAWard`<br>Only an admin can grant or revoke pool manager rights. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4900) |
| [HR-AC-05](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L72-L90) `acHrPolicyRowMovesOnlyForAWard`<br>Only an admin can set a pool's policy. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4900) |
| [HR-AC-06](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L92-L106) `acHrBridgingHookMovesOnlyForAWard`<br>Only an admin can set a pool's bridging hook. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4900) |
| [HR-AC-07](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L108-L123) `acHrRequestManagerMovesOnlyForAWard`<br>Only an admin can set a pool's request manager for a network. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4900) |
| [HR-AC-08](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L125-L140) `acHrAuthorizationIsScheduledOnlyByAWard`<br>Only an admin can schedule a time-locked authorization. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4900) |
| [HR-AC-09](./specs/core/hub/HubRegistry/single/HubRegistry_single_pool_access_control.spec#L142-L159) `acHrPendingAuthorizationMovesOnlyForAWardOrThePolicy`<br>Only an admin or the pool's policy can change a pending authorization. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-4900) |

### Hub_LinkedHubCore

- The chart of accounts bounded to 3 symbolic accounts
- One symbolic pool
- One symbolic share class
- Holding rows bounded to 2 symbolic rows
- Both deficit counters pinned to one symbolic network
- Loops running up to 4 iterations
- Metadata
  - The three metadata writers kept on the surface, their free form bytes and strings read under the 512 byte hashing bound
  - The registry, ledger and class metadata writers they call answered without effect

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [HU-VS-01](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_valid_state.spec#L16-L22) `journalDebitsEqualCredits`<br>A pool's total debits always equal its total credits. | ✅ | [🎯](./mutations/Accounting/README.md#accounting-621) [🎯](./mutations/Hub/README.md#hub-3046) |
| [HU-VS-02](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_valid_state.spec#L24-L28) `holdingAccountIdsExist`<br>Every account a holding posts to exists in the pool's books. | ✅ | [🎯](./mutations/Hub/README.md#hub-3) [🎯](./mutations/Hub/README.md#hub-4) [🎯](./mutations/Hub/README.md#hub-5) [🎯](./mutations/Hub/README.md#hub-3006) |
| [HU-VS-03](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_valid_state.spec#L30-L38) `initializedHoldingHasAccounts`<br>An opened holding always has all four of its accounts set. | ✅ | [🎯](./mutations/Hub/README.md#hub-3) [🎯](./mutations/Hub/README.md#hub-4) [🎯](./mutations/Hub/README.md#hub-5) [🎯](./mutations/Hub/README.md#hub-3006) |
| [HU-VS-04](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_valid_state.spec#L40-L43) `initializedHoldingAssetRegistered`<br>A holding exists only for a registered asset. | ✅ | [🎯](./mutations/Hub/README.md#hub-6) [🎯](./mutations/Hub/README.md#hub-7) |
| [HU-VS-05](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_valid_state.spec#L45-L48) `registeredPoolIsLocal`<br>An existing pool always belongs to this hub's own network. | ✅ | [🎯](./mutations/Hub/README.md#hub-1) [🎯](./mutations/Hub/README.md#hub-2) |
| [HU-VS-06](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_valid_state.spec#L50-L53) `initializedHoldingImpliesShareClassExists`<br>A holding exists only on an existing share class. | ✅ | [🎯](./mutations/Hub/README.md#hub-4825) |
| [HU-VS-07](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_valid_state.spec#L55-L61) `ledgerAccountImpliesRegisteredPool`<br>An accounting account exists only in a created pool. | ✅ | [🎯](./mutations/Hub/README.md#hub-4826) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [HU-ST-01](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L120-L133) `stHuPolicySwapAdvancesTheAuthorizationNamespace`<br>Changing a pool's policy voids the authorizations scheduled under the old one. | ✅ | [🎯](./mutations/Hub/README.md#hub-3013) |
| [HU-ST-02](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L135-L151) `stHuLedgerMoveIsFiledUnderAFreshJournal`<br>Any change to a pool's books is filed under one new journal id. | ✅ | [🎯](./mutations/Hub/README.md#hub-3020) |
| [HU-ST-03](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L153-L171) `stHuValueMoveIsBookedExactly`<br>A change in a holding's value is booked at its exact size on both sides. | ✅ | [🎯](./mutations/Hub/README.md#hub-9033) |
| [HU-ST-04](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L173-L187) `stHuPriceSourceSwapNeverRewiresTheAccounts`<br>No call changes both an opened holding's price source and one of its accounts. | ✅ | [🎯](./mutations/Hub/README.md#hub-3012) |
| [HU-ST-05](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L189-L198) `stHuEveryHubCallLeavesTheLedgerLocked`<br>No hub call leaves accounting unlocked. | ✅ | [🎯](./mutations/Hub/README.md#hub-9142) |
| [HU-ST-06](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L200-L221) `stHuProtocolValueMovesLandOnTheHoldingsOwnRow`<br>A holding's value change moves its own account's net equally when no other slot shares it. | ✅ | [🎯](./mutations/Hub/README.md#hub-9148) |
| [HU-ST-07](./specs/core/hub/Hub_LinkedHubCore/Hub_LinkedHubCore_metadata_properties.spec#L5-L14) `stHuEveryHubCallLeavesTheLedgerLockedWithTheWritersLive`<br>No hub call leaves accounting unlocked, the metadata writers and the rescue included. | ✅ | [🎯](./mutations/Hub/README.md#hub-9481) [🎯](./mutations/Hub/README.md#hub-9486) [🎯](./mutations/Hub/README.md#hub-9487) |
| [HU-ST-08](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L223-L239) `stHuLedgerTotalsMoveOnlyThroughJournalling`<br>A pool's total debits and credits change only through journal postings. | ✅ | [🎯](./mutations/Hub/README.md#hub-9035) |
| [HU-ST-09](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L241-L259) `stHuShareLedgerIsNeverMovedByTheHub`<br>No hub call changes a share class's issued or revoked shares on any network. | ✅ | [🎯](./mutations/Hub/README.md#hub-8060) |
| [HU-ST-10](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L261-L274) `stHuDeficitGateIsNeverMovedByTheHub`<br>No hub call moves a share class or a network into or out of deficit. | ✅ | [🎯](./mutations/Hub/README.md#hub-8061) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [HU-VT-01](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L5-L19) `vtHuAuthorizationTenureStepsByAtMostOne`<br>A pool's policy install counter grows by at most one per call, never going back. | ✅ | [🎯](./mutations/Hub/README.md#hub-4821) |
| [HU-VT-02](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L21-L35) `vtHuJournalSequenceStepsByAtMostOne`<br>A pool's journal number advances by at most one per call and never goes back. | ✅ | [🎯](./mutations/Hub/README.md#hub-4822) |
| [HU-VT-03](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L37-L51) `vtHuShareClassSequenceStepsByAtMostOne`<br>A pool's share class count grows by at most one per call and never shrinks. | ✅ | [🎯](./mutations/Hub/README.md#hub-4823) |
| [HU-VT-04](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L53-L67) `vtHuAccountWiringMovesOnlyThroughWiringEntries`<br>A holding's ledger accounts change only when it is opened or an account is reassigned. | ✅ | [🎯](./mutations/Hub/README.md#hub-4809) |
| [HU-VT-05](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L69-L83) `vtHuValuationMovesOnlyThroughItsInstallers`<br>A holding's valuation changes only when it is opened or its valuation is updated. | ✅ | [🎯](./mutations/Hub/README.md#hub-4810) |
| [HU-VT-06](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L85-L100) `vtHuSnapshotHookMovesOnlyThroughItsSetter`<br>A pool's snapshot hook changes only through its dedicated setter. | ✅ | [🎯](./mutations/Hub/README.md#hub-4811) |
| [HU-VT-07](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L102-L116) `vtHuSnapshotHookHearsOnlyARevaluation`<br>Only a hub revaluation notifies the snapshot hook, at most once per call. | ✅ | [🎯](./mutations/Hub/README.md#hub-8304) |
| [HU-VT-08](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_transitions.spec#L276-L289) `vtHuEveryOutboundSendCarriesTheAttachedValue`<br>A hub call that sends a message passes on the attached ETH, or none inside a batch. | ✅ | [🎯](./mutations/Hub/README.md#hub-8816) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [HU-HL-01](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L3-L17) `hlHuAmountRoundTripKeepsEveryAccountNet`<br>A holding deposit and an equal withdrawal cancel out on every ledger account. | ✅ | [🎯](./mutations/Hub/README.md#hub-4808) |
| [HU-HL-02](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L19-L38) `hlHuValueRoundTripKeepsTheHoldingAccountNetWhenGainAndLossAreOtherAccounts`<br>Undoing a value increase restores a holding's account if gain and loss post elsewhere. | ✅ | [🎯](./mutations/Hub/README.md#hub-4808) |
| [HU-HL-03](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L40-L54) `hlHuShareClassAnnouncementCarriesTheClassSalt`<br>A NotifyShareClass message carries the salt the share class was created with. | ✅ | [🎯](./mutations/Hub/README.md#hub-8062) |
| [HU-HL-04](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L56-L73) `hlHuFileWiresOnlyTheNamedSlot`<br>A hub configuration update sets only the named dependency, to the given address. | ✅ | [🎯](./mutations/Hub/README.md#hub-8301) |
| [HU-HL-05](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L75-L89) `hlHuShareClassAnnouncementCarriesTheCallersRegistrar`<br>A NotifyShareClass message carries the share token registrar the caller chose. | ✅ | [🎯](./mutations/Hub/README.md#hub-8302) |
| [HU-HL-06](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L91-L110) `hlHuRevaluationNotifiesTheHookExactlyWhenTheAssetsNetworkIsInSync`<br>A revaluation notifies a set snapshot hook exactly when its network is in sync. | ✅ | [🎯](./mutations/Hub/README.md#hub-8303) |
| [HU-HL-07](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L112-L137) `hlHuJournalPrimitivePostsOneBalancedPair`<br>A holding posting books the amount on its designated debit and credit accounts only, zero on none. | ✅ | [🎯](./mutations/Hub/README.md#hub-9145) |
| [HU-HL-08](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L139-L153) `hlHuSetAdaptersAnnouncesTheSessionItInstalls`<br>Changing a pool's adapters sends that network one message naming the session installed locally. | ✅ | [🎯](./mutations/Hub/README.md#hub-9146) |
| [HU-HL-09](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L155-L170) `hlHuSetAdaptersSendsBeforeItInstalls`<br>Reconfiguring a network's adapters sends the remote update before it installs the new local set. | ✅ | [🎯](./mutations/Hub/README.md#hub-9710) |
| [HU-HL-10](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L172-L198) `hlHuValuationDropNeverRaisesTheHoldingRowsValue`<br>A value drop credits the holding's account in full and never raises its balance if debit-normal. | ✅ | [🎯](./mutations/Hub/README.md#hub-9147) |
| [HU-HL-11](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L200-L226) `hlHuInitializationBooksTheTrackedAmountAsPrincipal`<br>Opening a holding books its tracked amount, priced by the given valuation, on its amount accounts. | ✅ | [🎯](./mutations/Hub/README.md#hub-9477) |
| [HU-HL-12](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L228-L244) `hlHuSharePriceUpdateAccruesExactlyWhenAFeeHookIsInstalled`<br>A share price update accrues fees once via any installed fee hook, for that pool and class. | ✅ | [🎯](./mutations/Hub/README.md#hub-9478) |
| [HU-HL-13](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L246-L262) `hlHuSharePriceUpdateStampedNowAccruesExactlyWhenAFeeHookIsInstalled`<br>A current-time share price update accrues fees once via any installed hook, for its pool and class. | ✅ | [🎯](./mutations/Hub/README.md#hub-9478) |
| [HU-HL-14](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L264-L280) `hlHuAssetPriceNotificationAccruesExactlyWhenAFeeHookIsInstalled`<br>An asset price notice accrues fees once via any installed fee hook, for that pool and class. | ✅ | [🎯](./mutations/Hub/README.md#hub-9483) |
| [HU-HL-15](./specs/core/hub/Hub_LinkedHubCore/Hub_LinkedHubCore_metadata_properties.spec#L18-L37) `hlHuMetadataWritersAndRescueWriteNoHubField`<br>The metadata writers and the token rescue move no hub field and leave no batch transient set. | ✅ | [🎯](./mutations/Hub/README.md#hub-9007) |
| [HU-HL-16](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L282-L293) `hlHuRuleBookInstallAlwaysOpensANewTenure`<br>Installing a policy, even the current one, voids earlier scheduled authorizations. | ✅ | [🎯](./mutations/Hub/README.md#hub-8300) |
| [HU-HL-17](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L295-L312) `hlHuSharePriceUpdateStampedNowAsksThePolicyAboutACallFreeOfTheClock`<br>Retrying a current-time share price update later gives the policy the same calldata. | ✅ | [🎯](./mutations/Hub/README.md#hub-8108) |
| [HU-HL-18](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_high_level.spec#L314-L324) `hlHuSharePriceUpdateStampedNowRecordsTheExecutionTime`<br>A current-time share price update stores the price with the block time. | ✅ | [🎯](./mutations/Hub/README.md#hub-8815) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [HU-RV-01](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L3-L17) `rvHuFileRefusesANonWardOrAnUnknownName`<br>Rewiring the hub fails for a non-admin or an unknown dependency name. | ✅ | [🎯](./mutations/Hub/README.md#hub-4814) |
| [HU-RV-02](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L19-L36) `rvHuCreatePoolRefusesARepeatOrAnUnusableOpening`<br>A pool opens only once, by an admin, on this network, with a manager and a registered currency. | ✅ | [🎯](./mutations/Hub/README.md#hub-4815) |
| [HU-RV-03](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L38-L52) `rvHuSetPolicyRefusesAnUnauthorisedInstall`<br>Setting a policy fails on a missing pool or for a caller neither admin nor pool manager. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) |
| [HU-RV-04](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L54-L70) `rvHuInitiateAuthorizationRefusesAnUnauthorisedOrDuplicateSchedule`<br>A delayed call cannot be scheduled by a non-manager, without a policy or delay, or twice. | ✅ | [🎯](./mutations/Hub/README.md#hub-4816) |
| [HU-RV-05](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L72-L87) `rvHuCancelAuthorizationRefusesANonManagerOrAnEmptySlot`<br>Cancelling an authorization fails for a non-manager or when none is pending. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) |
| [HU-RV-06](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L89-L102) `rvHuUpdateHubManagerRefusesANonManagerOrANullSeat`<br>Changing a pool's managers fails for a non-manager, a missing pool, or the zero address. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) |
| [HU-RV-07](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L104-L121) `rvHuUpdateCurrencyRefusesARescalingRedenomination`<br>A currency change fails for a non-manager, missing pool, unknown currency, or different decimals. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) |
| [HU-RV-08](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L123-L137) `rvHuCreateAccountRefusesARepeatOrAnUnusableOpening`<br>Creating an account fails for a non-manager, an empty id, or an existing account. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) |
| [HU-RV-09](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L139-L154) `rvHuSetHoldingAccountIdRefusesAnAbsentRowOrHolding`<br>A holding account change fails for a non-manager, a missing account or an unopened holding. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) |
| [HU-RV-10](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L156-L170) `rvHuUpdateHoldingValuationRefusesANullSourceOrClosedHolding`<br>A price source change fails for a non-manager, the zero address, or an unopened holding. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) |
| [HU-RV-11](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L172-L186) `rvHuManagerCallRefusesAMisdeclaredValue`<br>A manager call fails unless it declares the attached ETH (none in a batch), or zero if remote. | ✅ | [🎯](./mutations/Hub/README.md#hub-4817) [🎯](./mutations/Hub/README.md#hub-8828) |
| [HU-RV-12](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L188-L201) `rvHuRequestCallbackRefusesAnyCallerButTheRegisteredManager`<br>A request callback fails for any caller but the pool's request manager. | ✅ | [🎯](./mutations/Hub/README.md#hub-4818) |
| [HU-RV-13](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L203-L217) `rvHuPricePoolPerAssetAnswersForAnUnopenedHolding`<br>An asset price read on a holding not yet opened never fails. | ✅ | [🎯](./mutations/Hub/README.md#hub-4820) |
| [HU-RV-14](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L219-L240) `rvHuInitializeHoldingRefusesARepeatOrAnUnusableOpening`<br>A holding opens only once, by a manager, with a share class, known asset, price source and accounts. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) |
| [HU-RV-15](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L242-L257) `rvHuManagerFacingEntryRefusesANonManager`<br>Every manager-facing pool entry fails for a non-manager, except an admin setting the pool's policy. | ✅ | [🎯](./mutations/Hub/README.md#hub-9140) |
| [HU-RV-16](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L259-L275) `rvHuJournalPrimitivesRefuseANonWard`<br>Holding amount and value postings fail for a non-admin, even for a zero amount. | ✅ | [🎯](./mutations/Hub/README.md#hub-9141) |
| [HU-RV-17](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L277-L302) `rvHuJournalPrimitiveIsNeverRefusedUnbalanced`<br>Postings fail exactly on non-admin, excess ether, or if nonzero missing account, overflow, spent id. | ✅ | [🎯](./mutations/Hub/README.md#hub-9036) |
| [HU-RV-18](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L304-L316) `rvHuSetAdaptersRefusesWhileBatching`<br>Changing a pool's adapters fails while the gateway is batching. | ✅ | [🎯](./mutations/Hub/README.md#hub-9144) |
| [HU-RV-19](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L318-L341) `rvHuEnforcedEntryPassesTheInstalledPolicy`<br>Policed pool entries fail on a refusing policy and on success ask it once, bar an admin setting one. | ✅ | [🎯](./mutations/Hub/README.md#hub-8431) [🎯](./mutations/Hub/README.md#hub-9479) |
| [HU-RV-20](./specs/core/hub/Hub_LinkedHubCore/Hub_LinkedHubCore_metadata_properties.spec#L41-L62) `rvHuPoolMetadataWriterPassesBothGates`<br>Pool metadata writes fail for a non-manager or refusing policy, and success asks the policy once. | ✅ | [🎯](./mutations/Hub/README.md#hub-9480) |
| [HU-RV-21](./specs/core/hub/Hub_LinkedHubCore/Hub_LinkedHubCore_metadata_properties.spec#L64-L86) `rvHuAccountMetadataWriterPassesBothGates`<br>Account metadata writes fail for a non-manager or refusing policy, and success asks the policy once. | ✅ | [🎯](./mutations/Hub/README.md#hub-9484) |
| [HU-RV-22](./specs/core/hub/Hub_LinkedHubCore/Hub_LinkedHubCore_metadata_properties.spec#L88-L110) `rvHuShareClassMetadataWriterPassesBothGates`<br>Share class renames fail for a non-manager or refusing policy, and success asks the policy once. | ✅ | [🎯](./mutations/Hub/README.md#hub-9485) |
| [HU-RV-23](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L343-L367) `rvHuSpokeCallRevokePassesThePolicy`<br>Revoking a spoke call fails for a non-manager or a refusing policy; success sends one revocation. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) [🎯](./mutations/Hub/README.md#hub-8431) [🎯](./mutations/Hub/README.md#hub-8812) [🎯](./mutations/Hub/README.md#hub-9479) |
| [HU-RV-24](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L369-L383) `rvHuFileSucceedsForAWardNamingADependency`<br>Setting one of the hub's four dependencies never fails for an admin. | ✅ | [🎯](./mutations/Hub/README.md#hub-8818) |
| [HU-RV-25](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L385-L404) `rvHuShareClassMessagesRefuseAnUncreatedClass`<br>A restriction update, a vault update or a share class notice fails for a class never created. | ✅ | [🎯](./mutations/Hub/README.md#hub-8822) |
| [HU-RV-26](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L406-L423) `rvHuWardPolicyInstallOutlivesARefusingRuleBook`<br>An admin's policy change on an existing pool never fails and needs no policy approval. | ✅ | [🎯](./mutations/Hub/README.md#hub-8826) |
| [HU-RV-27](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reverts.spec#L425-L441) `rvHuCancelAuthorizationIsNeverRefusedToAnAdmittedManager`<br>Cancelling an authorization fails exactly for a non-manager, a refusing policy or none pending. | ✅ | [🎯](./mutations/Hub/README.md#hub-4813) [🎯](./mutations/Hub/README.md#hub-8827) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [HU-RC-01](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L3-L20) `rcHuCreatePoolSeatsADelegateIsReachable`<br>A hub admin can create a pool and make another address its manager. | ✅ | [🎯](./mutations/Hub/README.md#hub-6200) |
| [HU-RC-02](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L22-L38) `rcHuCreatePoolSeatsTheOpeningWardIsReachable`<br>A hub admin can create a pool and make itself its manager. | ✅ | [🎯](./mutations/Hub/README.md#hub-6200) |
| [HU-RC-03](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L40-L54) `rcHuCreatePoolAtTheSmallestCurrencyIsReachable`<br>A pool can be created with the lowest valid currency id. | ✅ | [🎯](./mutations/Hub/README.md#hub-6201) |
| [HU-RC-04](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L56-L76) `rcHuPoolBirthThenDelegationIsReachable`<br>A pool can be created and given a second manager in one transaction. | ✅ | [🎯](./mutations/Hub/README.md#hub-6202) |
| [HU-RC-05](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L78-L90) `rcHuSecondOperatorAppointmentIsReachable`<br>A pool manager can appoint another manager alongside itself. | ✅ | [🎯](./mutations/Hub/README.md#hub-6202) |
| [HU-RC-06](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L92-L109) `rcHuOperatorSeatRoundTripIsReachable`<br>A pool manager can be appointed and removed again in one transaction. | ✅ | [🎯](./mutations/Hub/README.md#hub-6202) |
| [HU-RC-07](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L111-L123) `rcHuOperatorCanStandItselfDownIsReachable`<br>A pool manager can remove itself as manager. | ✅ | [🎯](./mutations/Hub/README.md#hub-6203) |
| [HU-RC-08](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L125-L143) `rcHuGovernanceInstallsTheFirstRuleBookIsReachable`<br>A hub admin who is not a pool manager can install a pool's first policy. | ✅ | [🎯](./mutations/Hub/README.md#hub-6204) |
| [HU-RC-09](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L145-L163) `rcHuOperatorSwapsTheRuleBookIsReachable`<br>A pool manager can replace a pool's policy with a different one. | ✅ | [🎯](./mutations/Hub/README.md#hub-6205) |
| [HU-RC-10](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L165-L180) `rcHuRuleBookCanBeRemovedIsReachable`<br>A pool's policy can be removed, leaving the pool unrestricted. | ✅ | [🎯](./mutations/Hub/README.md#hub-6206) |
| [HU-RC-11](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L182-L200) `rcHuPrivilegedCallSchedulingIsReachable`<br>A pool manager can schedule a timelocked call under the pool's policy. | ✅ | [🎯](./mutations/Hub/README.md#hub-6207) [🎯](./mutations/HubRegistry/README.md#hubregistry-6414) |
| [HU-RC-12](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L202-L223) `rcHuScheduledCallWithdrawalIsReachable`<br>A privileged call can be scheduled and cancelled in one transaction. | ✅ | [🎯](./mutations/Hub/README.md#hub-6208) [🎯](./mutations/HubRegistry/README.md#hubregistry-6414) |
| [HU-RC-13](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L225-L245) `rcHuFirstShareClassIsReachable`<br>A pool manager can add the pool's first share class, using up its salt. | ✅ | [🎯](./mutations/Hub/README.md#hub-6209) |
| [HU-RC-14](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L247-L262) `rcHuDebitNormalRowCanBeOpenedIsReachable`<br>An asset or expense account can be opened as debit-normal. | ✅ | [🎯](./mutations/Hub/README.md#hub-6210) |
| [HU-RC-15](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L264-L279) `rcHuCreditNormalRowCanBeOpenedIsReachable`<br>An equity or liability account can be opened as credit-normal. | ✅ | [🎯](./mutations/Hub/README.md#hub-6211) |
| [HU-RC-16](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L281-L307) `rcHuAccountsThenHoldingIsReachable`<br>Two ledger accounts and a holding can be opened in one transaction. | ✅ | [🎯](./mutations/Hub/README.md#hub-6210) |
| [HU-RC-17](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L309-L336) `rcHuHoldingOpeningValueBookedAsPrincipalIsReachable`<br>Opening a holding can book its whole initial value as principal in one journal. | ✅ | [🎯](./mutations/Hub/README.md#hub-6212) |
| [HU-RC-18](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L338-L363) `rcHuHoldingOpeningWithNothingToBookIsReachable`<br>A holding with no value can be opened without posting any journal. | ✅ | [🎯](./mutations/Hub/README.md#hub-6213) |
| [HU-RC-19](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L365-L383) `rcHuHoldingWithARepeatedSlotIsReachable`<br>A holding can be opened with the same account on its debit and credit sides. | ✅ | [🎯](./mutations/Hub/README.md#hub-6214) [🎯](./mutations/Hub/README.md#hub-8820) |
| [HU-RC-20](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L385-L409) `rcHuHoldingMarkUpIsBookedIsReachable`<br>A holding's first mark-up can be booked in full as a balanced journal entry. | ✅ | [🎯](./mutations/Hub/README.md#hub-6215) |
| [HU-RC-21](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L411-L437) `rcHuHoldingMarkDownToZeroIsReachable`<br>A holding that still has assets can be written down to zero value. | ✅ | [🎯](./mutations/Hub/README.md#hub-6215) |
| [HU-RC-22](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L439-L462) `rcHuRevaluationLeavingTheBooksUntouchedIsReachable`<br>A revaluation finding a holding's value unchanged can leave the books untouched. | ✅ | [🎯](./mutations/Hub/README.md#hub-6216) |
| [HU-RC-23](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L464-L488) `rcHuDepositPostingIsReachable`<br>An admin can book a deposit as a debit to the holding and a matching credit. | ✅ | [🎯](./mutations/Hub/README.md#hub-6217) |
| [HU-RC-24](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L490-L514) `rcHuWithdrawalPostingIsReachable`<br>An admin can book a withdrawal as the mirror image of a deposit. | ✅ | [🎯](./mutations/Hub/README.md#hub-6217) |
| [HU-RC-25](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L516-L539) `rcHuAmountRoundTripIsReachable`<br>An equal deposit and withdrawal can share one journal and cancel out. | ✅ | [🎯](./mutations/Hub/README.md#hub-6218) |
| [HU-RC-26](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L541-L562) `rcHuZeroAmountPostingIsAcceptedIsReachable`<br>A zero-amount booking can succeed and leave the books untouched. | ✅ | [🎯](./mutations/Hub/README.md#hub-6213) |
| [HU-RC-27](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L564-L583) `rcHuLargestAmountPostingIsReachable`<br>The largest possible amount can be booked in one call onto empty accounts. | ✅ | [🎯](./mutations/Hub/README.md#hub-6219) |
| [HU-RC-28](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L585-L609) `rcHuValueGainPostingIsReachable`<br>An admin can book a value gain as a debit to the holding and a gain credit. | ✅ | [🎯](./mutations/Hub/README.md#hub-6220) |
| [HU-RC-29](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L611-L635) `rcHuValueLossPostingIsReachable`<br>An admin can book a value loss as a loss debit and a credit to the holding. | ✅ | [🎯](./mutations/Hub/README.md#hub-6220) |
| [HU-RC-30](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L637-L658) `rcHuZeroValuePostingIsAcceptedIsReachable`<br>A zero revaluation can succeed and leave the books untouched. | ✅ | [🎯](./mutations/Hub/README.md#hub-6216) |
| [HU-RC-31](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L660-L682) `rcHuMinimalBalancedJournalIsReachable`<br>A pool manager can post a balanced journal of one debit and one credit. | ✅ | [🎯](./mutations/Hub/README.md#hub-6221) |
| [HU-RC-32](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L684-L706) `rcHuWiderBalancedJournalIsReachable`<br>A pool manager can post a balanced journal of two debits and two credits. | ✅ | [🎯](./mutations/Hub/README.md#hub-6221) |
| [HU-RC-33](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L708-L729) `rcHuEmptyJournalIsReachable`<br>An empty journal can be posted, using a journal number while booking nothing. | ✅ | [🎯](./mutations/Hub/README.md#hub-6222) |
| [HU-RC-34](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L731-L754) `rcHuZeroValuedJournalPairIsReachable`<br>A zero-value journal can mark an account updated without changing any balance. | ✅ | [🎯](./mutations/Hub/README.md#hub-6223) |
| [HU-RC-35](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L756-L774) `rcHuSnapshotHookRoundTripIsReachable`<br>A pool's NAV snapshot hook can be set and then removed again. | ✅ | [🎯](./mutations/Hub/README.md#hub-6224) |
| [HU-RC-36](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L776-L795) `rcHuShareClassAnnouncementIsReachable`<br>A share class can be announced to a spoke in a single message with its payload. | ✅ | [🎯](./mutations/Hub/README.md#hub-6225) |
| [HU-RC-37](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L797-L814) `rcHuPaidRequestCallbackIsReachable`<br>The pool's request manager can send a paid request callback to its spoke. | ✅ | [🎯](./mutations/Hub/README.md#hub-6226) |
| [HU-RC-38](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L816-L834) `rcHuUnpaidRequestCallbackIsReachable`<br>The pool's request manager can send an unpaid request callback to its spoke. | ✅ | [🎯](./mutations/Hub/README.md#hub-6226) |
| [HU-RC-39](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L836-L853) `rcHuWardAuthorityRoundTripIsReachable`<br>An admin can grant hub admin rights to another account and revoke them again. | ✅ | [🎯](./mutations/Hub/README.md#hub-6227) |
| [HU-RC-40](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L855-L867) `rcHuWardCanStandItselfDownIsReachable`<br>An admin can revoke its own hub admin rights. | ✅ | [🎯](./mutations/Hub/README.md#hub-6227) |
| [HU-RC-41](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L869-L879) `rcHuUnopenedHoldingPricesAtParIsReachable`<br>A pool can price an asset at par when its holding was never opened. | ✅ | [🎯](./mutations/Hub/README.md#hub-6228) |
| [HU-RC-42](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L881-L892) `rcHuPassingPolicyReadIsReachable`<br>A pool's newly installed policy can be read back right away. | ✅ | [🎯](./mutations/Hub/README.md#hub-6229) |
| [HU-RC-43](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L894-L911) `rcHuGovernanceReplacesAStandingRuleBookIsReachable`<br>An admin can replace a pool's policy without the old policy's consent. | ✅ | [🎯](./mutations/Hub/README.md#hub-9149) |
| [HU-RC-44](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L913-L933) `rcHuValuationDropRaisesACreditNormalHoldingRowIsReachable`<br>A manager's downward revaluation can raise the balance of a holding's credit-normal account. | ✅ | [🎯](./mutations/Hub/README.md#hub-9150) |
| [HU-RC-45](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L935-L956) `rcHuManualJournalMovesAHoldingRowButNotItsValueIsReachable`<br>A manager can move a holding's own account by journal while the holding's value stays zero. | ✅ | [🎯](./mutations/Hub/README.md#hub-9151) |
| [HU-RC-46](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L958-L979) `rcHuRewiredHoldingRowLeavesItsValueUnmatchedIsReachable`<br>A manager can repoint a holding to an empty account, leaving its value matched by none. | ✅ | [🎯](./mutations/Hub/README.md#hub-9152) |
| [HU-RC-47](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L981-L997) `rcHuRegistryDirectRequestManagerRewireSendsNoSyncIsReachable`<br>A registry admin can change a network's request manager without telling the spoke. | ✅ | [🎯](./mutations/HubRegistry/README.md#hubregistry-9153) |
| [HU-RC-48](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L999-L1013) `rcHuHubPathRequestManagerRewireDispatchesTheSyncIsReachable`<br>A manager can change a network's request manager through the hub and tell the spoke. | ✅ | [🎯](./mutations/Hub/README.md#hub-9154) |
| [HU-RC-49](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1015-L1028) `rcHuWardSwapsARefusingRuleBookWithoutItsConsentIsReachable`<br>An admin can replace a pool's policy that refuses every call without asking it. | ✅ | [🎯](./mutations/Hub/README.md#hub-9482) |
| [HU-RC-50](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1030-L1046) `rcHuRevocationPassesAConsentingPolicyIsReachable`<br>A pool manager can revoke a spoke call authorization with the policy's approval. | ✅ | [🎯](./mutations/Hub/README.md#hub-8430) |
| [HU-RC-51](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1048-L1059) `rcHuFileRepointsTheMultiAdapterIsReachable`<br>A hub admin can repoint the hub's multi-adapter. | ✅ | [🎯](./mutations/Hub/README.md#hub-8817) |
| [HU-RC-52](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1061-L1080) `rcHuFreshPoolManagerInstallsTheFirstRuleBookUnpolicedIsReachable`<br>A new pool's manager, not an admin, can install its first policy unchecked. | ✅ | [🎯](./mutations/Hub/README.md#hub-8819) |
| [HU-RC-53](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1082-L1102) `rcHuSharedAmountSlotsKeepTheHoldingAccountFlatIsReachable`<br>A new holding can book its value as debit and credit on one shared account. | ✅ | [🎯](./mutations/Hub/README.md#hub-6214) [🎯](./mutations/Hub/README.md#hub-8820) |
| [HU-RC-54](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1104-L1121) `rcHuHoldingAccountRepointedOntoItsCounterpartIsReachable`<br>A pool manager can make a valued holding's debit and credit accounts the same. | ✅ | [🎯](./mutations/Hub/README.md#hub-8821) |
| [HU-RC-55](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1123-L1139) `rcHuAssetPriceNoticeForAnUnknownAssetAndClassIsReachable`<br>A par asset price can be sent for an unknown asset and an uncreated share class. | ✅ | [🎯](./mutations/Hub/README.md#hub-6228) [🎯](./mutations/Hub/README.md#hub-8823) |
| [HU-RC-56](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1141-L1156) `rcHuSharePriceNoticeForAnUncreatedClassIsReachable`<br>A zero, unstamped share price can be sent for an uncreated share class. | ✅ | [🎯](./mutations/Hub/README.md#hub-8824) |
| [HU-RC-57](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_reachability.spec#L1158-L1173) `rcHuShareClassNoticeConsultsTheInstalledPolicyIsReachable`<br>The pool's policy can approve a share class notice. | ✅ | [🎯](./mutations/Hub/README.md#hub-8825) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [HU-AC-01](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L3-L18) `acHuWardSetMovesOnlyForAWard`<br>Only a hub admin can grant or revoke hub admin rights. | ✅ | [🎯](./mutations/Hub/README.md#hub-4800) |
| [HU-AC-02](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L20-L43) `acHuWiringMovesOnlyForAWard`<br>Only a hub admin can change the hub's messaging contracts or its fee accrual hook. | ✅ | [🎯](./mutations/Hub/README.md#hub-4801) |
| [HU-AC-03](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L45-L62) `acHuPoolCurrencyMovesOnlyForAWardOrManager`<br>Only a hub admin or a pool manager can change a pool's accounting currency. | ✅ | [🎯](./mutations/Hub/README.md#hub-4802) |
| [HU-AC-04](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L64-L82) `acHuManagerSeatMovesOnlyForAWardOrManager`<br>Only a hub admin or a pool manager can add or remove a pool manager. | ✅ | [🎯](./mutations/Hub/README.md#hub-4802) |
| [HU-AC-05](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L84-L104) `acHuPoolPolicyMovesOnlyForAWardOrManager`<br>Only a hub admin or a pool manager can set a pool's policy. | ✅ | [🎯](./mutations/Hub/README.md#hub-4803) |
| [HU-AC-06](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L106-L123) `acHuAuthorizationLedgerMovesThroughTheHubOnlyForAPoolManager`<br>Only a pool manager can change a pool's pending timelocked calls via the hub. | ✅ | [🎯](./mutations/Hub/README.md#hub-4804) |
| [HU-AC-07](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L125-L140) `acHuBridgingHookMovesOnlyForAPoolManager`<br>Only a pool manager can change a pool's bridging hook. | ✅ | [🎯](./mutations/Hub/README.md#hub-4804) |
| [HU-AC-08](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L142-L159) `acHuRequestManagerSeatMovesOnlyForAPoolManager`<br>Only a pool manager can set a pool's request manager for a network. | ✅ | [🎯](./mutations/Hub/README.md#hub-4804) |
| [HU-AC-09](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L161-L177) `acHuSnapshotHookMovesOnlyForAPoolManager`<br>Only a pool manager can change a pool's snapshot hook. | ✅ | [🎯](./mutations/Hub/README.md#hub-4804) |
| [HU-AC-10](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L179-L196) `acHuHoldingValuationMovesOnlyForAPoolManager`<br>Only a pool manager can install or change a holding's valuation. | ✅ | [🎯](./mutations/Hub/README.md#hub-4805) |
| [HU-AC-11](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L198-L215) `acHuHoldingAccountWiringMovesOnlyForAPoolManager`<br>Only a pool manager can set which ledger accounts a holding posts to. | ✅ | [🎯](./mutations/Hub/README.md#hub-4805) |
| [HU-AC-12](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L217-L234) `acHuHoldingValueMovesThroughTheHubOnlyForAPoolManager`<br>Only a pool manager can change a holding's booked value through the hub. | ✅ | [🎯](./mutations/Hub/README.md#hub-4805) |
| [HU-AC-13](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L236-L261) `acHuShareClassRegisterMovesOnlyForAPoolManager`<br>Only a pool manager can add a share class or use up a share class salt. | ✅ | [🎯](./mutations/Hub/README.md#hub-4804) |
| [HU-AC-14](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L263-L281) `acHuSharePriceMovesOnlyForAPoolManager`<br>Only a pool manager can change a share class price or its timestamp. | ✅ | [🎯](./mutations/Hub/README.md#hub-4804) |
| [HU-AC-15](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L283-L299) `acHuAccountCreationNeedsAPoolManager`<br>Only a pool manager can open a new ledger account. | ✅ | [🎯](./mutations/Hub/README.md#hub-4804) |
| [HU-AC-16](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L301-L319) `acHuAccountDebitMovesThroughTheHubOnlyForAWardOrManager`<br>Only a hub admin or a pool manager can post debits to a pool's ledger account. | ✅ | [🎯](./mutations/Hub/README.md#hub-4806) |
| [HU-AC-17](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L321-L339) `acHuAccountCreditMovesThroughTheHubOnlyForAWardOrManager`<br>Only a hub admin or a pool manager can post credits to a pool's ledger account. | ✅ | [🎯](./mutations/Hub/README.md#hub-4806) |
| [HU-AC-18](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L341-L358) `acHuJournalCounterMovesThroughTheHubOnlyForAWardOrManager`<br>Only a hub admin or a pool manager can record a new journal in a pool's ledger. | ✅ | [🎯](./mutations/Hub/README.md#hub-4806) |
| [HU-AC-19](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L360-L379) `acHuRequestCallbackNeedsTheRegisteredRequestManager`<br>Only a pool's request manager for a network can send request callbacks there. | ✅ | [🎯](./mutations/Hub/README.md#hub-4807) |
| [HU-AC-20](./specs/core/hub/Hub_LinkedHubCore/single/Hub_LinkedHubCore_access_control.spec#L381-L399) `acHuOutboundInstructionNeedsAPoolManagerOrARequestFulfilment`<br>The hub sends a cross-chain message only for a pool manager or a request callback. | ✅ | [🎯](./mutations/Hub/README.md#hub-4804) |

### HubHandler_LinkedHubCore

- The chart of accounts bounded to 3 symbolic accounts
- One symbolic pool
- One symbolic share class
- Holding rows bounded to 2 symbolic rows
- Both deficit counters pinned to one symbolic network
- Loops running up to 4 iterations

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [HH-VS-01](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_valid_state.spec#L5-L9) `postedDebitsCoverCarriedValue`<br>Any two holdings are together never worth more than the pool's posted debits. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4448) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [HH-ST-01](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L127-L146) `stHhReportedAssetMoveIsBookedExactly`<br>An initialized holding's value change is booked in full as a debit and a credit. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-9062) |
| [HH-ST-02](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L148-L161) `stHhShareLedgerMovesWithItsNetworkLegs`<br>A share class's total supply changes by exactly the net change across networks. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-3109) |
| [HH-ST-03](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L163-L176) `stHhSnapshotFlagCostsANonce`<br>A network's snapshot mark changes only when its report nonce advances by one. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-3104) |
| [HH-ST-04](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L178-L191) `stHhNetworkDeficitCountFollowsTheShortRows`<br>The network's shortfall count follows the number of holdings in shortfall. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-3110) |
| [HH-ST-05](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L193-L206) `stHhClassDeficitCountFollowsTheShortRows`<br>A share class's shortfall count follows the number of its holdings in shortfall. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8052) [🎯](./mutations/HubHandler/README.md#hubhandler-3110) |
| [HH-ST-06](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L208-L221) `stHhNegativeNetworkCountFollowsTheShortNetworks`<br>The count of networks with a negative share balance follows their actual number. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8054) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [HH-VT-01](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L5-L19) `vtHhSnapshotSequenceStepsByAtMostOne`<br>A network's report nonce advances by at most one per call. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4440) |
| [HH-VT-02](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L21-L35) `vtHhJournalSequenceStepsByAtMostOne`<br>A pool's journal entry number advances by at most one per call. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4441) |
| [HH-VT-03](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L37-L52) `vtHhAccountStampRecordsTheCurrentBlock`<br>An account's last-updated time, when it changes, is the current block time. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-9063) |
| [HH-VT-04](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L54-L70) `vtHhHoldingFlowHistoryNeverShrinks`<br>A holding's cumulative inflow and outflow totals never decrease. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4443) |
| [HH-VT-05](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L72-L88) `vtHhNetworkShareHistoryNeverShrinks`<br>A network's cumulative issued and revoked share totals never decrease. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4444) |
| [HH-VT-06](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L90-L106) `vtHhLedgerRowTotalsNeverShrink`<br>An account's cumulative debit and credit totals never decrease. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4445) |
| [HH-VT-07](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_transitions.spec#L108-L123) `vtHhSnapshotHookHearsOnlyAReport`<br>Only share and asset reports send the snapshot hook a sync notice, at most once. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8311) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [HH-HL-01](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L5-L20) `hlHhTransferConservesTheClassTotal`<br>Moving shares between networks leaves the share class's total supply unchanged. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8433) |
| [HH-HL-02](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L22-L38) `hlHhTransferDebitsOriginByTheSentAmount`<br>A transfer revokes on the origin network exactly the amount its message sends. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4412) [🎯](./mutations/HubHandler/README.md#hubhandler-6520) [🎯](./mutations/HubHandler/README.md#hubhandler-6523) |
| [HH-HL-03](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L40-L56) `hlHhTransferSendsTheAmountItMints`<br>A transfer issues on the target network exactly the amount its message sends. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4411) [🎯](./mutations/HubHandler/README.md#hubhandler-6520) |
| [HH-HL-04](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L58-L71) `hlHhTransferKeepsTheOriginNetwork`<br>A transfer's message states the network the shares left. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4413) |
| [HH-HL-05](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L73-L86) `hlHhTransferKeepsTheTargetNetwork`<br>A transfer's message is addressed to the requested target network. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4413) |
| [HH-HL-06](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L88-L102) `hlHhHooklessTransferKeepsTheReceiver`<br>With no bridging hook, a transfer keeps the receiver the caller requested. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4414) |
| [HH-HL-07](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L104-L118) `hlHhHooklessTransferKeepsTheAmount`<br>With no bridging hook, a transfer sends the amount the caller requested. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4412) [🎯](./mutations/HubHandler/README.md#hubhandler-6523) |
| [HH-HL-08](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L120-L134) `hlHhHooklessTransferKeepsTheRefund`<br>With no bridging hook, the transfer keeps the caller's refund address. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4415) |
| [HH-HL-09](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L136-L151) `hlHhTransferSendsExactlyOneMessage`<br>One transfer emits exactly one outbound message. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4416) [🎯](./mutations/HubHandler/README.md#hubhandler-6523) |
| [HH-HL-10](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L153-L166) `hlHhTransferSendsAnExecuteTransfer`<br>The message a share transfer sends is an ExecuteTransferShares message. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4416) [🎯](./mutations/HubHandler/README.md#hubhandler-6523) |
| [HH-HL-11](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L168-L185) `hlHhAssetReportLandsGrossOnItsSide`<br>A spoke's asset report adds its full amount to its incoming or outgoing total. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4417) [🎯](./mutations/HubHandler/README.md#hubhandler-6511) |
| [HH-HL-12](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L187-L204) `hlHhAssetReportLeavesTheOppositeSideAlone`<br>An asset inflow report never changes the outflow total, and vice versa. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4419) |
| [HH-HL-13](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L206-L221) `hlHhAssetReportLeavesOtherIncreasesAlone`<br>One asset's report cannot change another asset's incoming total. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4417) |
| [HH-HL-14](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L223-L238) `hlHhAssetReportLeavesOtherDecreasesAlone`<br>One asset's report cannot change another asset's outgoing total. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4418) |
| [HH-HL-15](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L240-L255) `hlHhAssetReportLeavesOtherValuesAlone`<br>One asset's report cannot revalue another asset's holding. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4417) |
| [HH-HL-16](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L257-L273) `hlHhShareReportLandsOnItsCounter`<br>A network's share report adds its full amount to its issued or revoked total. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4411) [🎯](./mutations/HubHandler/README.md#hubhandler-6515) [🎯](./mutations/HubHandler/README.md#hubhandler-6516) |
| [HH-HL-17](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L275-L289) `hlHhShareReportMovesTheClassTotalInStep`<br>A share report changes the class supply by the reported amount and direction. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4410) [🎯](./mutations/HubHandler/README.md#hubhandler-6517) |
| [HH-HL-18](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L291-L304) `hlHhRequestForwardsExactlyOnce`<br>An investor request is forwarded once and only once. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4420) |
| [HH-HL-19](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L306-L319) `hlHhRequestReachesTheRegisteredManager`<br>An investor request goes to the pool's request manager for its asset's network. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4420) |
| [HH-HL-20](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L321-L332) `hlHhRequestForwardsThePool`<br>An investor request arrives at the manager under the pool it was sent for. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4420) |
| [HH-HL-21](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L334-L345) `hlHhRequestForwardsTheShareClass`<br>An investor request arrives at the manager under the share class it was sent for. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4420) |
| [HH-HL-22](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L347-L358) `hlHhRequestForwardsTheAsset`<br>An investor request arrives at the manager under the asset it was sent for. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4420) |
| [HH-HL-23](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L360-L373) `hlHhRequestForwardsThePayload`<br>An investor request reaches its manager with the payload unaltered. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4420) |
| [HH-HL-24](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L375-L392) `hlHhSyncNotificationOnlyOnAClosedBatch`<br>A share report notifies the pool's snapshot hook exactly when it closes a batch. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8053) [🎯](./mutations/HubHandler/README.md#hubhandler-8313) |
| [HH-HL-25](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L394-L411) `hlHhFileWiresOnlyTheNamedSlot`<br>An admin updating one handler dependency leaves the other three unchanged. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8310) |
| [HH-HL-26](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L413-L431) `hlHhAssetReportNotifiesTheHookExactlyOnAClosedBatch`<br>An asset report notifies the snapshot hook exactly when it closes a batch. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8312) |
| [HH-HL-27](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L433-L462) `hlHhAssetReportJournalsTheRealizedValueOnlyOnAnInitializedHolding`<br>An asset report books exactly the holding's value change to its own accounts, none if uninitialized. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-9085) |
| [HH-HL-28](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L464-L481) `hlHhAcceptedAssetReportSpendsItsNetworksNonceOnce`<br>An accepted asset report spends exactly its own network's next snapshot nonce, once. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9086) |
| [HH-HL-29](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L483-L499) `hlHhAcceptedShareReportSpendsItsNetworksNonceOnce`<br>An accepted share report spends exactly its own network's next snapshot nonce, once. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-9086) |
| [HH-HL-30](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L501-L516) `hlHhReportOfAnotherNetworkLeavesTheTrackedCountersAlone`<br>An asset report from one network never changes another network's shortfall counts. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8055) |
| [HH-HL-31](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L518-L533) `hlHhTransferLeavesTheClassCountersAlone`<br>A share transfer leaves a share class's issued and revoked totals unchanged. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8432) [🎯](./mutations/HubHandler/README.md#hubhandler-8433) |
| [HH-HL-32](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L535-L547) `hlHhHooklessTransferKeepsTheExtraGasLimit`<br>With no bridging hook, a transfer keeps the caller's extra gas limit. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8832) |
| [HH-HL-33](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_high_level.spec#L549-L561) `hlHhTransferKeepsThePoolAndClass`<br>A bridging hook cannot redirect a share transfer to another pool or share class. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8833) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [HH-RV-01](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L3-L17) `rvHhFileRefusesANonWardOrAnUnknownName`<br>Rewiring the hub handler fails for a non-admin caller or an unknown parameter. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4430) |
| [HH-RV-02](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L19-L34) `rvHhRequestRefusesANonWardOrANetworkWithNoManager`<br>A request fails for a non-admin or when the asset's network has no request manager. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4437) |
| [HH-RV-03](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L36-L52) `rvHhUpdateSharesRefusesANonWardAnUnknownClassOrAMissedNonce`<br>A share report fails for a non-admin, an unknown share class, or a wrong nonce. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4433) [🎯](./mutations/HubHandler/README.md#hubhandler-4434) |
| [HH-RV-04](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L54-L70) `rvHhUpdateAssetsRefusesANonWardOrAMissedNonce`<br>An asset report fails for a non-admin or on a bad nonce. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4434) |
| [HH-RV-05](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L72-L110) `rvHhUninitializedAssetReportIsNeverStrandedByTheShortfallTally`<br>The hub never blocks an admin's in-order report on an uninitialized holding, barring overflow. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8314) [🎯](./mutations/HubHandler/README.md#hubhandler-8438) |
| [HH-RV-06](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L112-L127) `rvHhTransferRefusesANonWardOrAnUnknownShareClass`<br>A share transfer fails for a non-admin or a share class the hub does not know. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8437) |
| [HH-RV-07](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L129-L156) `rvHhHooklessTransferIsNeverStrandedWhileTheCountersHaveRoom`<br>Barring overflow, the hub never blocks an admin's known-class transfer with no bridging hook or ETH. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8315) [🎯](./mutations/HubHandler/README.md#hubhandler-8439) |
| [HH-RV-08](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L158-L169) `rvHhRelyRefusesANonWard`<br>Granting admin rights on the hub handler fails for a non-admin caller. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4435) |
| [HH-RV-09](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L171-L182) `rvHhDenyRefusesANonWard`<br>Revoking admin rights on the hub handler fails for a non-admin caller. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4436) |
| [HH-RV-10](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L184-L199) `rvHhRequestIsNeverRefusedOnANetworkThatHasAManager`<br>The hub handler never blocks an admin's request when the asset's network has a request manager. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6504) [🎯](./mutations/HubHandler/README.md#hubhandler-6505) |
| [HH-RV-11](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L201-L232) `rvHhShareReportIsNeverStrandedWhileTheCountersHaveRoom`<br>The hub never blocks an admin's in-order share report on a known class, barring overflow. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8439) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8050) |
| [HH-RV-12](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L234-L289) `rvHhOpenedAssetReportFailsOnlyOnAnOverflowOrAMissingAccount`<br>An admin's in-order report on an opened holding fails only on overflow or a missing account. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8314) [🎯](./mutations/HubHandler/README.md#hubhandler-8438) [🎯](./mutations/HubHandler/README.md#hubhandler-8829) |
| [HH-RV-13](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reverts.spec#L291-L308) `rvHhRegisterAssetRefusesExactlyANonWardAnOversizedPrecisionOrARescaling`<br>Asset registration fails exactly on ether, a non-admin, a zero id, over 18 or changed set decimals. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4431) [🎯](./mutations/HubHandler/README.md#hubhandler-4437) [🎯](./mutations/HubHandler/README.md#hubhandler-8434) [🎯](./mutations/HubHandler/README.md#hubhandler-8435) [🎯](./mutations/HubHandler/README.md#hubhandler-8830) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [HH-RC-01](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L3-L17) `rcHhRegisterAssetIsReachable`<br>A spoke can register a new asset on the hub with its declared decimals. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6500) |
| [HH-RC-02](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L19-L33) `rcHhRegisterAssetAtTheDecimalsCeilingIsReachable`<br>An asset with 18 decimals, the maximum accepted, can be registered. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6501) |
| [HH-RC-03](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L35-L49) `rcHhRegisterAssetAtZeroDecimalsIsReachable`<br>An asset with zero decimals can be registered. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6502) |
| [HH-RC-04](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L51-L76) `rcHhAssetDepositOnALiveHoldingIsReachable`<br>A deposit report can raise a holding's value with a balanced journal entry. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6506) |
| [HH-RC-05](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L78-L103) `rcHhAssetWithdrawalOnALiveHoldingIsReachable`<br>A withdrawal report can lower a holding's value with a balanced journal entry. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6507) |
| [HH-RC-06](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L105-L126) `rcHhAssetReportOnAnUnopenedHoldingIsReachable`<br>A report on a holding never set up can be counted with no journal entry. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6508) |
| [HH-RC-07](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L128-L147) `rcHhAssetReportClosingTheBatchIsReachable`<br>An asset report can close its network's batch, marking the network in sync. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6509) |
| [HH-RC-08](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L149-L168) `rcHhEmptyAssetBatchClosingIsReachable`<br>An asset report moving nothing can still close its network's batch. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6510) |
| [HH-RC-09](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L170-L185) `rcHhFirstAssetReportOfANetworkIsReachable`<br>A network's first report, at nonce zero, can close its first batch. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6512) |
| [HH-RC-10](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L187-L208) `rcHhAssetWithdrawalEnteringDeficitIsReachable`<br>A withdrawal report can put a holding in deficit, raising its deficit counts. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8058) [🎯](./mutations/HubHandler/README.md#hubhandler-6513) |
| [HH-RC-11](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L210-L231) `rcHhAssetDepositLeavingDeficitIsReachable`<br>A deposit report can clear a holding's deficit, lowering the class and network deficit counts. | ✅ | [🎯](./mutations/Holdings/README.md#holdings-8058) [🎯](./mutations/HubHandler/README.md#hubhandler-6514) |
| [HH-RC-12](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L233-L252) `rcHhShareReportClosingTheBatchIsReachable`<br>A share report can close its network's batch, marking the network in sync. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6518) |
| [HH-RC-13](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L254-L275) `rcHhMarkerOnlyShareReportIsReachable`<br>A share report with no movement can still close its network's batch. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6519) |
| [HH-RC-14](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L277-L299) `rcHhBridgeReachableWithAHookRewritingTheAmount`<br>A bridging hook can resize a share transfer, the burn matching the new amount. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6521) |
| [HH-RC-15](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L301-L328) `rcHhBridgeLeavesAThirdNetworkAloneIsReachable`<br>A transfer between two networks can leave a third network's shares unchanged. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6522) |
| [HH-RC-16](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L330-L350) `rcHhBridgeEmptyingTheOriginNetworkIsReachable`<br>A network's whole share position can leave in one transfer. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6524) |
| [HH-RC-17](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L352-L377) `rcHhBridgeDrivesTheOriginNetworkNegativeIsReachable`<br>A transfer can move more shares off a network than it has, marking it negative. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6525) |
| [HH-RC-18](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L379-L392) `rcHhFileRepointsTheHubIsReachable`<br>An admin can switch the handler to a different hub. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6526) |
| [HH-RC-19](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L394-L407) `rcHhFileRepointsTheHoldingsLedgerIsReachable`<br>An admin can switch the handler to a different holdings contract. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6527) |
| [HH-RC-20](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L409-L422) `rcHhFileRepointsTheSenderIsReachable`<br>An admin can switch the handler to a different outbound message sender. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6528) |
| [HH-RC-21](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L424-L437) `rcHhFileRepointsTheShareClassBookIsReachable`<br>An admin can point the hub handler at a different share class manager. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6529) |
| [HH-RC-22](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L439-L456) `rcHhRelyThenDenyRoundTripIsReachable`<br>Admin rights on the hub handler can be granted and then revoked again. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6530) [🎯](./mutations/HubHandler/README.md#hubhandler-6531) |
| [HH-RC-23](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L458-L477) `rcHhRegisterThenReportOnTheAssetIsReachable`<br>An asset can be registered and receive a reported inflow in one transaction. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6503) |
| [HH-RC-24](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L479-L500) `rcHhIssueThenRevokeRestoresTheClassTotalIsReachable`<br>Shares a network issues can be revoked again, leaving total supply unchanged. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6532) |
| [HH-RC-25](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L502-L523) `rcHhSnapshotSetThenClearedIsReachable`<br>A report can mark a network's state as a snapshot and the next can clear it. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6533) |
| [HH-RC-26](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L525-L554) `rcHhBridgeAndReturnLegIsReachable`<br>Shares can be bridged to another network and back with no net change on either. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6534) |
| [HH-RC-27](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L556-L574) `rcHhShareReportPastTheClassSupplyIsReachable`<br>A reported revocation can exceed all issued shares, making total supply negative. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-6525) [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8051) |
| [HH-RC-28](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L576-L597) `rcHhShortfallLandsOnTheReportingNetworkIsReachable`<br>A shortfall can count against the reporting network, not the asset's home network. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8056) |
| [HH-RC-29](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L599-L621) `rcHhBridgeClearsAShortTargetNetworkIsReachable`<br>A bridge can clear a network's negative share balance, total supply unchanged. | ✅ | [🎯](./mutations/ShareClassManager/README.md#shareclassmanager-8057) |
| [HH-RC-30](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L623-L636) `rcHhZeroDecimalAssetTakesItsDecimalsLaterIsReachable`<br>An asset recorded with zero decimals can later take its real decimals. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8436) [🎯](./mutations/HubHandler/README.md#hubhandler-8831) |
| [HH-RC-31](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_reachability.spec#L638-L655) `rcHhZeroPrecisionAssetCanBeCorrectedByARepeatRegistration`<br>A new asset registered with zero decimals can be corrected by registering it again. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-8436) [🎯](./mutations/HubHandler/README.md#hubhandler-8831) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [HH-AC-01](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L3-L18) `acHhOnlyAWardMovesTheWardSet`<br>Only an admin can grant or revoke admin rights on the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4406) |
| [HH-AC-02](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L20-L34) `acHhOnlyAWardMovesTheHubPointer`<br>Only an admin can change which hub inbound messages are applied to. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4400) |
| [HH-AC-03](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L36-L50) `acHhOnlyAWardMovesTheHoldingsPointer`<br>Only an admin can change which holdings contract inbound asset reports update. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4400) |
| [HH-AC-04](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L52-L67) `acHhOnlyAWardMovesTheSenderPointer`<br>Only an admin can change the contract the hub handler sends messages through. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4400) |
| [HH-AC-05](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L69-L84) `acHhOnlyAWardMovesTheShareClassManagerPointer`<br>Only an admin can change which share class manager inbound share reports update. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4400) |
| [HH-AC-06](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L86-L101) `acHhOnlyAWardRegistersAnAsset`<br>Only an admin can register an asset on the hub via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4403) |
| [HH-AC-07](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L103-L118) `acHhOnlyAWardMovesAnAssetDecimals`<br>Only an admin can set an asset's decimals via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4403) |
| [HH-AC-08](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L120-L135) `acHhOnlyAWardMovesADepositHistory`<br>Only an admin can record incoming assets on a holding via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-09](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L137-L152) `acHhOnlyAWardMovesAWithdrawalHistory`<br>Only an admin can record outgoing assets on a holding via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-10](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L154-L169) `acHhOnlyAWardMovesACarriedValue`<br>Only an admin can change a holding's value via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-11](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L171-L189) `acHhOnlyAWardMovesTheDeficitCount`<br>Only an admin can change the counts of holdings in deficit via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-12](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L191-L205) `acHhOnlyAWardMovesASyncMarker`<br>Only an admin can change whether a network is marked in sync, via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-13](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L207-L222) `acHhOnlyAWardMovesASnapshotNonce`<br>Only an admin can change a network's snapshot sequence number via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-14](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L224-L239) `acHhOnlyAWardMovesANetworkIssuedTotal`<br>Only an admin can change a network's issued share total via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4402) |
| [HH-AC-15](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L241-L256) `acHhOnlyAWardMovesANetworkRevokedTotal`<br>Only an admin can change a network's revoked share total via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4402) |
| [HH-AC-16](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L258-L275) `acHhOnlyAWardMovesTheClassSupply`<br>Only an admin can change a share class's total supply via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4402) |
| [HH-AC-17](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L277-L292) `acHhOnlyAWardMovesALedgerDebit`<br>Only an admin can debit a pool's accounting account via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-18](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L294-L309) `acHhOnlyAWardMovesALedgerCredit`<br>Only an admin can credit a pool's accounting account via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-19](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L311-L326) `acHhOnlyAWardMovesTheJournalCounter`<br>Only an admin can book accounting journal entries via the hub handler. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4401) |
| [HH-AC-20](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L328-L343) `acHhOnlyAWardForwardsAnInvestorRequest`<br>Only an admin can make the hub handler forward an investor request. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4404) |
| [HH-AC-21](./specs/core/hub/HubHandler_LinkedHubCore/HubHandler_LinkedHubCore_access_control.spec#L345-L359) `acHhOnlyAWardSendsAnOutboundMessage`<br>Only an admin can make the hub handler send a message to another network. | ✅ | [🎯](./mutations/HubHandler/README.md#hubhandler-4405) |

### Spoke

- One symbolic pool
- One symbolic share class
- One ERC20 asset row in the escrow
- ERC20 and ERC6909 tokens modeled in CVL (standard, up to 3 holders, uint128 balances, 6 to 18 decimals, no share token hooks, a holder pulling its own tokens only against an allowance to itself)

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SP-ST-01](./specs/core/spoke/Spoke/Spoke_transitions.spec#L113-L127) `stSpQueuedMoveIsBackedByTheEscrow`<br>Each queued asset change is matched by the same change in the escrow's spendable balance. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-211) [🎯](./mutations/Spoke/README.md#spoke-216) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SP-VT-01](./specs/core/spoke/Spoke/Spoke_transitions.spec#L5-L20) `vtSpBridgeNeverAnnouncesAnEmptyTransfer`<br>The spoke never announces a cross-chain share transfer of nothing. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4600) |
| [SP-VT-02](./specs/core/spoke/Spoke/Spoke_transitions.spec#L22-L40) `vtSpBridgeNeverAnnouncesBackToThisNetwork`<br>A cross-chain share transfer is never announced back to this same network. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4601) |
| [SP-VT-03](./specs/core/spoke/Spoke/Spoke_transitions.spec#L42-L59) `vtSpAssetRegistrationNeverAnnouncesTheNullId`<br>An asset registration message never has an empty asset id. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4602) |
| [SP-VT-04](./specs/core/spoke/Spoke/Spoke_transitions.spec#L61-L76) `vtSpAssetRegistrationRespectsTheDecimalsCeiling`<br>An asset registration message never states more than the maximum decimals. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4603) [🎯](./mutations/Spoke/README.md#spoke-9944) |
| [SP-VT-05](./specs/core/spoke/Spoke/Spoke_transitions.spec#L78-L91) `vtSpOneCallHandsOverAtMostOneMessage`<br>One call on the spoke sends at most one outbound message. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4604) |
| [SP-VT-06](./specs/core/spoke/Spoke/Spoke_transitions.spec#L93-L109) `vtSpEveryOutboundSendCarriesTheAttachedValue`<br>Every outbound message forwards the ETH its caller attached, and none from inside a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8514) [🎯](./mutations/Spoke/README.md#spoke-9704) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [SP-HL-01](./specs/core/spoke/Spoke/Spoke_high_level.spec#L3-L19) `hlSpBridgeDebitsTheNamedOwner`<br>A bridge transfer takes the shares from the owner the caller named. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4605) |
| [SP-HL-02](./specs/core/spoke/Spoke/Spoke_high_level.spec#L21-L37) `hlSpBridgeBurnsTheBridgedShares`<br>The shares a bridge transfer takes are destroyed on this network. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4606) |
| [SP-HL-03](./specs/core/spoke/Spoke/Spoke_high_level.spec#L39-L56) `hlSpBridgeParksNoSharesOnTheSpoke`<br>A bridge transfer leaves no shares behind on the spoke. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4606) |
| [SP-HL-04](./specs/core/spoke/Spoke/Spoke_high_level.spec#L58-L73) `hlSpBridgeAnnouncesExactlyOneTransfer`<br>A bridge transfer sends exactly one outgoing transfer message. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4607) |
| [SP-HL-05](./specs/core/spoke/Spoke/Spoke_high_level.spec#L75-L88) `hlSpBridgeAnnouncesTheBurnedAmount`<br>A bridge transfer message carries the amount of shares burned. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4608) |
| [SP-HL-06](./specs/core/spoke/Spoke/Spoke_high_level.spec#L90-L103) `hlSpBridgeAnnouncesTheNamedReceiver`<br>A bridge transfer message carries the receiver the caller named. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4609) |
| [SP-HL-07](./specs/core/spoke/Spoke/Spoke_high_level.spec#L105-L123) `hlSpBridgeSpendsTheOwnersApproval`<br>A bridge transfer deducts the amount bridged from the owner's finite approval to the spoke. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4605) |
| [SP-HL-08](./specs/core/spoke/Spoke/Spoke_high_level.spec#L125-L140) `hlSpBridgeReportsNoShareMovementToTheHub`<br>A bridge transfer queues no share change for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4610) |
| [SP-HL-09](./specs/core/spoke/Spoke/Spoke_high_level.spec#L142-L158) `hlSpNoteDepositMovesNoTokens`<br>Recording a deposit that already reached the escrow transfers no tokens. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4611) |
| [SP-HL-10](./specs/core/spoke/Spoke/Spoke_high_level.spec#L160-L175) `hlSpRevokeSpendsTheCallersApproval`<br>A revocation spends the caller's approval to the spoke. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8500) |
| [SP-HL-11](./specs/core/spoke/Spoke/Spoke_high_level.spec#L177-L192) `hlSpTransferSharesFromSpendsTheNamedSendersApproval`<br>A forced share transfer spends the holder's approval to the named sender. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8501) |
| [SP-HL-12](./specs/core/spoke/Spoke/Spoke_high_level.spec#L194-L210) `hlSpDepositDebitsTheCaller`<br>A deposit is paid from the caller's own balance. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8502) |
| [SP-HL-13](./specs/core/spoke/Spoke/Spoke_high_level.spec#L212-L223) `hlSpRegisterAssetAnnouncesToTheNamedNetwork`<br>Registering an asset announces it to the network the caller named. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8503) |
| [SP-HL-14](./specs/core/spoke/Spoke/Spoke_high_level.spec#L225-L238) `hlSpFileRepointsOnlyTheNamedRoute`<br>Changing the spoke's gateway or message sender leaves the other unchanged. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8504) |
| [SP-HL-15](./specs/core/spoke/Spoke/Spoke_high_level.spec#L240-L254) `hlSpIssueMintsToTheRecipientNamed`<br>Issued shares are minted to the recipient the caller named. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8505) |
| [SP-HL-16](./specs/core/spoke/Spoke/Spoke_high_level.spec#L256-L272) `hlSpWithdrawalPaysTheReceiverNamed`<br>A withdrawal pays the receiver the caller named. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8506) |
| [SP-HL-17](./specs/core/spoke/Spoke/Spoke_high_level.spec#L274-L291) `hlSpReservedWithdrawalPaysTheReceiverNamed`<br>A withdrawal of reserved funds pays the receiver the caller named. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8507) |
| [SP-HL-18](./specs/core/spoke/Spoke/Spoke_high_level.spec#L293-L308) `hlSpSharePayoutReachesTheReceiverNamed`<br>A share payout from the escrow reaches the receiver the caller named. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8508) |
| [SP-HL-19](./specs/core/spoke/Spoke/Spoke_high_level.spec#L310-L326) `hlSpManagerCallNamesItsCallerOnTheWire`<br>A manager call tells the hub who made it and forwards the caller's ETH, none inside a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8510) [🎯](./mutations/Spoke/README.md#spoke-9702) [🎯](./mutations/Spoke/README.md#spoke-9703) |
| [SP-HL-20](./specs/core/spoke/Spoke/Spoke_high_level.spec#L328-L341) `hlSpBridgeAnnouncesTheNamedSender`<br>A bridge transfer message carries the sender the caller named. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8511) |
| [SP-HL-21](./specs/core/spoke/Spoke/Spoke_high_level.spec#L343-L354) `hlSpShortBridgeAnnouncesTheCallerAsSender`<br>The short-form bridge transfer reports its caller as the sender. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8512) |
| [SP-HL-22](./specs/core/spoke/Spoke/Spoke_high_level.spec#L356-L368) `hlSpRequestRelaysThePaymentModeItWasHanded`<br>A request reaches the hub paid or unpaid, as the request manager asked. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8513) |
| [SP-HL-23](./specs/core/spoke/Spoke/Spoke_high_level.spec#L370-L382) `hlSpRevokeLeavesNoApprovalToTheRegistrar`<br>A revocation leaves the spoke with no open approval to the share registrar. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8515) |
| [SP-HL-24](./specs/core/spoke/Spoke/Spoke_high_level.spec#L384-L398) `hlSpBridgeLeavesNoApprovalToTheRegistrar`<br>A bridge transfer leaves the spoke with no open approval to the share registrar. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8516) |
| [SP-HL-25](./specs/core/spoke/Spoke/Spoke_high_level.spec#L400-L413) `hlSpFileWritesTheValueIntoTheNamedRoute`<br>Setting the spoke's gateway or message sender stores exactly the address given. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9270) |
| [SP-HL-26](./specs/core/spoke/Spoke/Spoke_high_level.spec#L415-L446) `hlSpEscrowResolvedEntryNeverMovesAnotherPoolsEscrowTokens`<br>A deposit or payout for one pool leaves another pool's escrow untouched unless paying out to it. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9273) |
| [SP-HL-27](./specs/core/spoke/Spoke/Spoke_high_level.spec#L448-L465) `hlSpBridgeBurnsBeforeItAnnounces`<br>A bridge transfer burns the shares before it announces the transfer to the destination. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9707) |
| [SP-HL-28](./specs/core/spoke/Spoke/Spoke_high_level.spec#L467-L502) `hlSpRecoverTokensMovesOnlyTheRescuedBalances`<br>A token rescue moves only the spoke's and the receiver's balances, no spoke field under any key. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9008) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [SP-RV-01](./specs/core/spoke/Spoke/Spoke_reverts.spec#L3-L15) `rvSpFileRefusesANonWardOrAnUnknownName`<br>Configuring the spoke fails for a non-admin caller or an unknown setting. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4617) |
| [SP-RV-02](./specs/core/spoke/Spoke/Spoke_reverts.spec#L17-L29) `rvSpDepositRefusesANonManager`<br>A deposit into the pool escrow fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-03](./specs/core/spoke/Spoke/Spoke_reverts.spec#L31-L43) `rvSpNoteDepositRefusesANonManager`<br>Counting assets already in the escrow fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-04](./specs/core/spoke/Spoke/Spoke_reverts.spec#L45-L57) `rvSpWithdrawRefusesANonManager`<br>A withdrawal from the pool escrow fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-05](./specs/core/spoke/Spoke/Spoke_reverts.spec#L59-L72) `rvSpWithdrawReservedRefusesANonManager`<br>Paying out reserved assets fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-06](./specs/core/spoke/Spoke/Spoke_reverts.spec#L74-L86) `rvSpReserveRefusesANonManager`<br>Reserving pool assets fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-07](./specs/core/spoke/Spoke/Spoke_reverts.spec#L88-L101) `rvSpUnreserveRefusesANonManager`<br>Releasing reserved assets fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-08](./specs/core/spoke/Spoke/Spoke_reverts.spec#L103-L115) `rvSpSubmitQueuedAssetsRefusesANonManager`<br>Reporting asset changes to the hub fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-09](./specs/core/spoke/Spoke/Spoke_reverts.spec#L117-L129) `rvSpIssueRefusesANonManager`<br>Issuing share tokens fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-10](./specs/core/spoke/Spoke/Spoke_reverts.spec#L131-L143) `rvSpRevokeRefusesANonManager`<br>Revoking share tokens fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-11](./specs/core/spoke/Spoke/Spoke_reverts.spec#L145-L157) `rvSpWithdrawSharesRefusesANonManager`<br>Moving shares out of the pool escrow fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-12](./specs/core/spoke/Spoke/Spoke_reverts.spec#L159-L171) `rvSpSubmitQueuedSharesRefusesANonManager`<br>Reporting share changes to the hub fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-13](./specs/core/spoke/Spoke/Spoke_reverts.spec#L173-L185) `rvSpTransferSharesFromRefusesANonManager`<br>A forced share transfer fails unless the caller is a pool manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4616) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-14](./specs/core/spoke/Spoke/Spoke_reverts.spec#L187-L202) `rvSpBridgeRefusesAStrangerToTheOwner`<br>A cross-chain share transfer fails unless the caller is the owner or an admin, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4620) [🎯](./mutations/Spoke/README.md#spoke-9700) |
| [SP-RV-15](./specs/core/spoke/Spoke/Spoke_reverts.spec#L204-L218) `rvSpBridgeRefusesANonBridgerOwner`<br>A cross-chain share transfer fails unless the pool lets the owner bridge shares. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4621) |
| [SP-RV-16](./specs/core/spoke/Spoke/Spoke_reverts.spec#L220-L234) `rvSpBridgeRefusesAZeroAmount`<br>A cross-chain share transfer of zero shares fails. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4622) |
| [SP-RV-17](./specs/core/spoke/Spoke/Spoke_reverts.spec#L236-L250) `rvSpBridgeRefusesALocalDestination`<br>A cross-chain share transfer to this same network fails. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4623) |
| [SP-RV-18](./specs/core/spoke/Spoke/Spoke_reverts.spec#L252-L266) `rvSpRequestRefusesAnyCallerButTheRegisteredManager`<br>A request to the hub fails unless the direct caller is the pool's request manager, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4624) [🎯](./mutations/Spoke/README.md#spoke-9701) |
| [SP-RV-19](./specs/core/spoke/Spoke/Spoke_reverts.spec#L268-L279) `rvSpRelyRefusesANonWard`<br>Granting spoke admin rights fails unless the caller is an admin. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4617) |
| [SP-RV-20](./specs/core/spoke/Spoke/Spoke_reverts.spec#L281-L292) `rvSpDenyRefusesANonWard`<br>Revoking spoke admin rights fails unless the caller is an admin. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4617) |
| [SP-RV-21](./specs/core/spoke/Spoke/Spoke_reverts.spec#L294-L314) `rvSpSubmitQueuedAssetsIsNeverRefusedToASeatedManager`<br>With no policy and no ETH sent, a non-admin pool manager can always report asset changes. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4618) |
| [SP-RV-22](./specs/core/spoke/Spoke/Spoke_reverts.spec#L316-L336) `rvSpSubmitQueuedSharesIsNeverRefusedToASeatedManager`<br>With no policy and no ETH sent, a non-admin pool manager can always report share changes. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4618) |
| [SP-RV-23](./specs/core/spoke/Spoke/Spoke_reverts.spec#L338-L351) `rvSpManagerCallIsNeverRefusedToAnyCaller`<br>A manager call with no ETH attached never fails, whoever the caller is. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4619) |
| [SP-RV-24](./specs/core/spoke/Spoke/Spoke_reverts.spec#L354-L368) `rvSpBridgeRefusesAnOwnerTheRegistrarVetoes`<br>A cross-chain share transfer fails when the share token's restriction rejects that owner and amount. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8517) [🎯](./mutations/Spoke/README.md#spoke-9708) |
| [SP-RV-25](./specs/core/spoke/Spoke/Spoke_reverts.spec#L370-L389) `rvSpProtectedEntriesRefuseAForeignReentrantCaller`<br>While a call is in progress, registering an asset or bridging shares fails for any other caller. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9706) |
| [SP-RV-26](./specs/core/spoke/Spoke/Spoke_reverts.spec#L391-L405) `rvSpBridgeRefusesAnUnregisteredShareClass`<br>A cross-chain share transfer fails when the share class has no share token. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8929) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [SP-RC-01](./specs/core/spoke/Spoke/Spoke_reachability.spec#L3-L22) `rcSpRegisterAssetMintsAFreshIdForAnUnknownErc20`<br>An unprivileged caller can register a new ERC20 and create its asset id. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6000) |
| [SP-RC-02](./specs/core/spoke/Spoke/Spoke_reachability.spec#L24-L42) `rcSpRegisterAssetReannouncesTheStoredIdIsReachable`<br>Re-registering an asset can announce to the hub the id it was first given. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6001) |
| [SP-RC-03](./specs/core/spoke/Spoke/Spoke_reachability.spec#L44-L62) `rcSpRegisterAssetMintsAFreshIdForAnErc6909Tranche`<br>An ERC6909 token can be registered as an asset and get a fresh id. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6002) |
| [SP-RC-04](./specs/core/spoke/Spoke/Spoke_reachability.spec#L64-L79) `rcSpRegisterAssetAtTheDecimalsCeilingIsReachable`<br>An asset with the maximum allowed decimals can still be registered with the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6003) |
| [SP-RC-05](./specs/core/spoke/Spoke/Spoke_reachability.spec#L81-L101) `rcSpFirstDepositIntoAnEmptyRowIsReachable`<br>A first deposit into empty escrow can be fully spendable and queued for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6004) |
| [SP-RC-06](./specs/core/spoke/Spoke/Spoke_reachability.spec#L103-L125) `rcSpDepositIntoAHoldingRowIsReachable`<br>A deposit can top up an escrow balance and queue its amount for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6004) |
| [SP-RC-07](./specs/core/spoke/Spoke/Spoke_reachability.spec#L127-L145) `rcSpDepositOfThePayersWholeBalanceIsReachable`<br>A payer can deposit their entire balance of an asset in one call. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6005) |
| [SP-RC-08](./specs/core/spoke/Spoke/Spoke_reachability.spec#L147-L169) `rcSpDepositOfAnErc6909TrancheIsReachable`<br>An ERC6909 token can be deposited into the pool escrow. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6006) |
| [SP-RC-09](./specs/core/spoke/Spoke/Spoke_reachability.spec#L171-L193) `rcSpNoteDepositCreditsCustodyWithoutMovingTokens`<br>A deposit can be recorded in the escrow and queued without any token transfer. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6007) |
| [SP-RC-10](./specs/core/spoke/Spoke/Spoke_reachability.spec#L195-L214) `rcSpZeroAmountDepositIsReachable`<br>A zero-amount deposit can succeed and still queue an update for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6008) |
| [SP-RC-11](./specs/core/spoke/Spoke/Spoke_reachability.spec#L216-L239) `rcSpWithdrawPaysTheReceiverIsReachable`<br>A withdrawal can pay the receiver from escrow and queue a decrease for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6009) |
| [SP-RC-12](./specs/core/spoke/Spoke/Spoke_reachability.spec#L241-L257) `rcSpWithdrawEmptiesTheFreeBalanceExactlyIsReachable`<br>A withdrawal can empty an escrow balance that has nothing reserved. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6010) |
| [SP-RC-13](./specs/core/spoke/Spoke/Spoke_reachability.spec#L259-L278) `rcSpWithdrawLeavesTheHeldPartBehindIsReachable`<br>A withdrawal can pay out all unreserved funds while reservations stay intact. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6010) |
| [SP-RC-14](./specs/core/spoke/Spoke/Spoke_reachability.spec#L280-L300) `rcSpWithdrawToTheCallerThemselvesIsReachable`<br>A manager can withdraw assets to their own account. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6009) |
| [SP-RC-15](./specs/core/spoke/Spoke/Spoke_reachability.spec#L302-L323) `rcSpFirstHoldOnARowIsReachable`<br>A first reservation can hold funds in escrow and queue a decrease for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6011) |
| [SP-RC-16](./specs/core/spoke/Spoke/Spoke_reachability.spec#L325-L347) `rcSpHoldOnTheWholeFreeBalanceIsReachable`<br>A reservation can cover the whole escrow balance, leaving nothing spendable. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6012) |
| [SP-RC-17](./specs/core/spoke/Spoke/Spoke_reachability.spec#L349-L372) `rcSpReleaseOfTheWholeHoldIsReachable`<br>A reservation can be released in full, queuing an increase for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6013) |
| [SP-RC-18](./specs/core/spoke/Spoke/Spoke_reachability.spec#L374-L398) `rcSpWithdrawReservedPaysHeldFundsOutIsReachable`<br>Reserved funds can be paid out in full with no new update queued for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6014) |
| [SP-RC-19](./specs/core/spoke/Spoke/Spoke_reachability.spec#L400-L421) `rcSpIssueMintsSharesIsReachable`<br>Shares can be issued to an investor, with the issuance queued for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6015) |
| [SP-RC-20](./specs/core/spoke/Spoke/Spoke_reachability.spec#L423-L443) `rcSpZeroShareIssuanceIsReachable`<br>A zero-share issuance can succeed and still queue an update for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6015) |
| [SP-RC-21](./specs/core/spoke/Spoke/Spoke_reachability.spec#L445-L469) `rcSpRevokeBurnsTheCallersSharesIsReachable`<br>A caller's shares can be revoked and burned, with the decrease queued for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6016) |
| [SP-RC-22](./specs/core/spoke/Spoke/Spoke_reachability.spec#L471-L489) `rcSpRevokeDownToAnEmptyClassIsReachable`<br>A pool manager holding every share can revoke the share class's entire supply. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6016) |
| [SP-RC-23](./specs/core/spoke/Spoke/Spoke_reachability.spec#L491-L515) `rcSpWithdrawSharesHandsCustodyOutIsReachable`<br>Escrowed shares can go to an investor without minting or reporting to the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6017) |
| [SP-RC-24](./specs/core/spoke/Spoke/Spoke_reachability.spec#L517-L537) `rcSpTransferSharesFromMovesAHoldingIsReachable`<br>A manager can move shares between two holders without changing supply. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6018) |
| [SP-RC-25](./specs/core/spoke/Spoke/Spoke_reachability.spec#L539-L556) `rcSpSubmitQueuedAssetsReportsToTheHubIsReachable`<br>Queued asset updates can be sent to the hub for the asset the caller chose. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6019) |
| [SP-RC-26](./specs/core/spoke/Spoke/Spoke_reachability.spec#L558-L575) `rcSpSubmitQueuedSharesWithNoExtraGasIsReachable`<br>Queued share updates can reach the hub without asking for extra execution gas. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6020) |
| [SP-RC-27](./specs/core/spoke/Spoke/Spoke_reachability.spec#L577-L602) `rcSpBridgeDrivenByTheOwnerIsReachable`<br>A share owner can bridge their own shares to another network, burning them here. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6021) |
| [SP-RC-28](./specs/core/spoke/Spoke/Spoke_reachability.spec#L604-L627) `rcSpBridgeDrivenByAWardForTheOwnerIsReachable`<br>An admin can bridge shares on an owner's behalf, paid from the owner's balance. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6022) |
| [SP-RC-29](./specs/core/spoke/Spoke/Spoke_reachability.spec#L629-L651) `rcSpBridgeOfTheWholeHoldingIsReachable`<br>An owner's entire share balance can be bridged to another network at once. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6023) |
| [SP-RC-30](./specs/core/spoke/Spoke/Spoke_reachability.spec#L653-L672) `rcSpBridgeThroughTheShortEntryPointIsReachable`<br>A holder can bridge shares with the simplified call and no extra destination gas. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6021) |
| [SP-RC-31](./specs/core/spoke/Spoke/Spoke_reachability.spec#L674-L692) `rcSpPaidRequestIsRelayedIsReachable`<br>The request manager can relay a paid investor request to the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6024) |
| [SP-RC-32](./specs/core/spoke/Spoke/Spoke_reachability.spec#L694-L712) `rcSpUnpaidRequestIsRelayedIsReachable`<br>The request manager can relay an unpaid investor request to the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6024) |
| [SP-RC-33](./specs/core/spoke/Spoke/Spoke_reachability.spec#L714-L733) `rcSpManagerCallFromAnUnseatedCallerIsReachable`<br>A caller with no manager or admin rights can send a manager call to the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6025) |
| [SP-RC-34](./specs/core/spoke/Spoke/Spoke_reachability.spec#L735-L752) `rcSpRelyThenDenyRoundTripIsReachable`<br>Spoke admin rights can be granted and revoked again in the same transaction. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6026) [🎯](./mutations/Spoke/README.md#spoke-6027) |
| [SP-RC-35](./specs/core/spoke/Spoke/Spoke_reachability.spec#L754-L773) `rcSpDepositThenWithdrawRoundTripIsReachable`<br>A deposit then an equal withdrawal can restore the escrow balance, queuing both. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6029) |
| [SP-RC-36](./specs/core/spoke/Spoke/Spoke_reachability.spec#L775-L798) `rcSpHoldThenPayOutIsReachable`<br>Funds can be reserved then paid out, with a single update queued for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6014) |
| [SP-RC-37](./specs/core/spoke/Spoke/Spoke_reachability.spec#L800-L822) `rcSpIssueThenWithdrawSharesIsReachable`<br>Shares can be issued into escrow then sent to an investor, queuing one update. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6017) |
| [SP-RC-38](./specs/core/spoke/Spoke/Spoke_reachability.spec#L824-L847) `rcSpDepositThenSubmitQueuedAssetsIsReachable`<br>A deposit can be made and reported to the hub in one transaction. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6019) |
| [SP-RC-39](./specs/core/spoke/Spoke/Spoke_reachability.spec#L849-L872) `rcSpHoldThenReleaseRoundTripIsReachable`<br>Assets can be reserved and released again, with both changes queued for the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6012) |
| [SP-RC-40](./specs/core/spoke/Spoke/Spoke_reachability.spec#L874-L900) `rcSpWithdrawOfAnErc6909TrancheIsReachable`<br>An ERC6909 asset can be paid out of the pool escrow to a receiver. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6028) |
| [SP-RC-41](./specs/core/spoke/Spoke/Spoke_reachability.spec#L902-L915) `rcSpFileRepointsTheGatewayIsReachable`<br>An admin can point the spoke at a different gateway. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6030) |
| [SP-RC-42](./specs/core/spoke/Spoke/Spoke_reachability.spec#L917-L930) `rcSpFileRepointsTheMessageSenderIsReachable`<br>An admin can replace the spoke's outbound message sender. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6030) |
| [SP-RC-43](./specs/core/spoke/Spoke/Spoke_reachability.spec#L932-L956) `rcSpHoldBeyondTheRowsCustodyIsReachable`<br>A pool can reserve more assets than its escrow currently holds. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6011) |
| [SP-RC-44](./specs/core/spoke/Spoke/Spoke_reachability.spec#L958-L976) `rcSpAWardCanSeatTheGatewayOnTheWardRoll`<br>An admin can make the gateway an admin, which can then reconfigure the spoke. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9272) |
| [SP-RC-45](./specs/core/spoke/Spoke/Spoke_reachability.spec#L978-L990) `rcSpFileAcceptsTheNullAddress`<br>An admin can set the spoke's gateway to the zero address. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9271) |
| [SP-RC-46](./specs/core/spoke/Spoke/Spoke_reachability.spec#L992-L1015) `rcSpBridgeToAnUndeliverableReceiverBurnsAndAnnounces`<br>An owner can bridge shares to a receiver that is not a valid address, burning them here. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9275) |
| [SP-RC-47](./specs/core/spoke/Spoke/Spoke_reachability.spec#L1017-L1037) `rcSpBridgeAnnouncesASenderNeitherCallerNorOwner`<br>A non-admin share owner can bridge naming a sender who is neither the caller nor the owner. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9276) |
| [SP-RC-48](./specs/core/spoke/Spoke/Spoke_reachability.spec#L1039-L1054) `rcSpRegisterAssetAtZeroDecimalsIsReachable`<br>An asset with zero decimals can be registered and announced to the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8642) |
| [SP-RC-49](./specs/core/spoke/Spoke/Spoke_reachability.spec#L1056-L1075) `rcSpBatchedRequestPassesTheGateWhenTheGatewayIsTheRequestManagerIsReachable`<br>A batching account can relay a request when the gateway is the request manager. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-8930) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [SP-AC-01](./specs/core/spoke/Spoke/Spoke_access_control.spec#L3-L18) `acSpWardListMovesOnlyForAWard`<br>Only an admin can grant or revoke admin rights on the spoke. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-5301) [🎯](./mutations/Spoke/README.md#spoke-5302) |
| [SP-AC-02](./specs/core/spoke/Spoke/Spoke_access_control.spec#L20-L34) `acSpGatewayIsRepointedOnlyByAWard`<br>Only an admin can change the gateway the spoke sends messages through. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-5300) |
| [SP-AC-03](./specs/core/spoke/Spoke/Spoke_access_control.spec#L36-L51) `acSpMessageSenderIsRepointedOnlyByAWard`<br>Only an admin can change the contract that sends the spoke's messages. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-5300) |
| [SP-AC-04](./specs/core/spoke/Spoke/Spoke_access_control.spec#L53-L67) `acSpQueuedAssetDeltasComeOnlyFromAManager`<br>Only a pool manager can queue asset changes for the hub through the spoke, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4612) [🎯](./mutations/Spoke/README.md#spoke-9705) |
| [SP-AC-05](./specs/core/spoke/Spoke/Spoke_access_control.spec#L69-L83) `acSpQueuedShareDeltasComeOnlyFromAManager`<br>Only a pool manager can queue share changes for the hub through the spoke, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4612) [🎯](./mutations/Spoke/README.md#spoke-9705) |
| [SP-AC-06](./specs/core/spoke/Spoke/Spoke_access_control.spec#L85-L100) `acSpEscrowCustodyMovesOnlyForAManager`<br>Through the spoke, only a pool manager can change escrow balances or reservations, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4612) [🎯](./mutations/Spoke/README.md#spoke-9705) |
| [SP-AC-07](./specs/core/spoke/Spoke/Spoke_access_control.spec#L102-L120) `acSpQueueSubmissionsLeaveOnlyForAManager`<br>Only a pool manager can submit queued asset or share updates to the hub, even in a batch. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4612) [🎯](./mutations/Spoke/README.md#spoke-9705) |
| [SP-AC-08](./specs/core/spoke/Spoke/Spoke_access_control.spec#L122-L136) `acSpShareSupplyGrowsThroughTheSpokeOnlyForAManager`<br>Only a pool manager can issue new shares through the spoke. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4612) |
| [SP-AC-09](./specs/core/spoke/Spoke/Spoke_access_control.spec#L138-L157) `acSpUnauthorizedCannotReduceAHoldersBalance`<br>Only the holder or a pool manager can reduce a balance not approved to the spoke. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4612) |
| [SP-AC-10](./specs/core/spoke/Spoke/Spoke_access_control.spec#L159-L175) `acSpBridgeStartsOnlyForABridgerOrAWard`<br>Only a listed bridger or an admin can start a cross-chain share transfer. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4614) |
| [SP-AC-11](./specs/core/spoke/Spoke/Spoke_access_control.spec#L177-L194) `acSpInvestorRequestsLeaveOnlyForTheRequestManager`<br>Only the pool's request manager can send an investor request to the hub. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4615) |
| [SP-AC-12](./specs/core/spoke/Spoke/Spoke_access_control.spec#L196-L216) `acSpApprovalToTheSpokeIsSpentOnlyByItsOwnerAWardOrAManagersForcedTransfer`<br>An approval to the spoke is spent only by its holder, an admin, or a manager's forced transfer. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-4613) [🎯](./mutations/Spoke/README.md#spoke-8509) |
| [SP-AC-13](./specs/core/spoke/Spoke/Spoke_access_control.spec#L218-L242) `acSpRegisterAssetSeatsNobodyInAnyPool`<br>Registering an asset changes no one's admin, manager or bridger rights and no pool role. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9274) |
| [SP-AC-14](./specs/core/spoke/Spoke/Spoke_access_control.spec#L244-L263) `acSpShareSupplyMovesOnlyThroughTheClassRegistrar`<br>The spoke changes a token's supply only through the share class's registrar, never directly. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9709) |

### SpokeRegistry

- Outside the configurations named below
  - One symbolic pool
  - One symbolic share class
- Two keys
  - Two symbolic pools and two symbolic share classes, four class keys of which at least two are distinct, the null class admitted
  - Where a rule compares two pools, two distinct pools and one symbolic share class

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [SR-VS-01](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L46-L49) `poolCreatedAtNotFuture`<br>A pool's creation time is never in the future. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1202) |
| [SR-VS-02](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L51-L54) `poolCreatedAboveEnvFloor`<br>A created pool's creation time is never before the earliest assumed block time. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1203) |
| [SR-VS-03](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L56-L59) `shareTokenIffRegistrar`<br>A share class has a share token exactly when it has a token registrar. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1204) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1210) |
| [SR-VS-04](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L61-L64) `shareClassImpliesActivePool`<br>A share class exists only on a created pool. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1205) |
| [SR-VS-05](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L66-L69) `tokenRowScIdImpliesPoolId`<br>A share token never resolves to a share class without also resolving to a pool. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1207) |
| [SR-VS-06](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L71-L77) `shareTokenHasCanonicalReverseRow`<br>The registered share token resolves back to its own pool and share class. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1208) |
| [SR-VS-07](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L79-L86) `onlyCurrentShareTokenHoldsALookupRow`<br>Only a share class's current token resolves to it, so a replaced token does not. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1209) |
| [SR-VS-08](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L88-L91) `nullTokenRowPoolIdZero`<br>The zero address never resolves to a pool as a share token. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1210) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1230) |
| [SR-VS-09](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L93-L96) `requestManagerImpliesActivePool`<br>A request manager can only be installed on a created pool. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1211) |
| [SR-VS-10](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L98-L101) `managerImpliesActivePool`<br>Manager rights exist only on a created pool. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1212) |
| [SR-VS-11](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L103-L106) `bridgerImpliesActivePool`<br>Bridger rights exist only on a created pool. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1213) |
| [SR-VS-12](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L108-L111) `policyImpliesNonzeroNonce`<br>A pool with an installed policy always has a non-zero policy nonce. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1214) |
| [SR-VS-13](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L113-L116) `policyNonceImpliesActivePool`<br>A pool's policy nonce is non-zero only once the pool is created. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1215) |
| [SR-VS-14](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L118-L121) `nullAuthIdRowEmpty`<br>No authorization is ever recorded under the zero authorization id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1216) |
| [SR-VS-15](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L123-L130) `noPolicyEverMeansNoAuthorizations`<br>A pool that never had a policy set holds no authorizations. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1217) |
| [SR-VS-16](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L132-L139) `assetForwardResolvesBack`<br>A registered asset id's token and token id map back to that same asset id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1218) |
| [SR-VS-17](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L141-L151) `assetReverseResolvesBack`<br>An asset id assigned to a token resolves back to that token and token id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1219) |
| [SR-VS-18](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L153-L160) `registeredAssetIdWellFormed`<br>A registered asset id has a non-zero network id and an already issued number. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1220) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1237) |
| [SR-VS-19](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L162-L165) `unregisteredAssetRowEmpty`<br>An unregistered asset id holds no leftover token id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1221) |
| [SR-VS-20](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L167-L170) `nullAssetNeverRegistered`<br>The zero asset address is never registered under any token id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1221) |
| [SR-VS-21](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L172-L175) `sharePriceStampImpliesShareClass`<br>A share price timestamp can only exist once the share class does. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1222) |
| [SR-VS-22](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L177-L180) `sharePriceValueImpliesShareClass`<br>A share price value can only exist once the share class does. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1222) |
| [SR-VS-23](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L182-L185) `assetPriceStampImpliesShareClass`<br>An asset price timestamp can only exist once the share class does. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1223) |
| [SR-VS-24](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L187-L190) `assetPriceValueImpliesShareClass`<br>An asset price value can only exist once the share class does. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1223) |
| [SR-VS-25](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L192-L195) `assetPriceStampImpliesRegisteredAsset`<br>An asset price timestamp can only exist for a registered asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1224) |
| [SR-VS-26](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L197-L200) `assetPriceValueImpliesRegisteredAsset`<br>An asset price value can only exist for a registered asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1224) |
| [SR-VS-27](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L202-L205) `linkedVaultIsRegistered`<br>A vault can only be linked after it was registered. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1225) |
| [SR-VS-28](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L207-L210) `unregisteredVaultPoolIdZero`<br>An unregistered vault has no pool recorded. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1226) |
| [SR-VS-29](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L212-L215) `unregisteredVaultScIdZero`<br>An unregistered vault has no share class recorded. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1226) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1231) |
| [SR-VS-30](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L217-L220) `unregisteredVaultAssetIdZero`<br>An unregistered vault has no asset id recorded. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1226) |
| [SR-VS-31](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L222-L225) `unregisteredVaultTokenIdZero`<br>An unregistered vault has no token id recorded. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1226) |
| [SR-VS-32](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L227-L236) `registeredVaultMatchesAssetRegistry`<br>A registered vault's asset id always resolves to the vault's asset and token id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1227) |
| [SR-VS-33](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L238-L246) `registeredVaultSitsOnALiveShareClass`<br>Every registered vault belongs to a share class that has a share token. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1228) |
| [SR-VS-34](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L248-L251) `nullVaultNeverRegistered`<br>The zero address is never a registered vault. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1229) |
| [SR-VS-35](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L253-L260) `noAuthorizationsUnderZeroNonce`<br>No authorization is ever recorded under a zero policy nonce. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1260) |
| [SR-VS-36](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L262-L266) `noAuthorizationsAheadOfTenure`<br>No authorization is recorded for a policy nonce the pool has not reached yet. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1258) |
| [SR-VS-37](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L268-L275) `currentTenureAuthorizationsCarryTheInstalledPolicy`<br>Under the current policy nonce, only the installed policy has authorizations. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1259) |
| [SR-VS-38](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L277-L280) `onlyADeployedContractOwnsATokenRow`<br>Only an address with deployed code can resolve to a pool as a share token. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1210) |
| [SR-VS-39](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L38-L47) `classTokenResolvesBack`<br>A share class's live token looks back up to exactly that pool and share class. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9561) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9564) |
| [SR-VS-40](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L49-L57) `lookupRowPointsBack`<br>A token that looks up to a pool and share class is the token that share class holds. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9562) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9564) |
| [SR-VS-41](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L59-L69) `twoClassKeysNeverShareAToken`<br>No two share classes, in one pool or two, ever hold the same share token. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9560) |
| [SR-VS-42](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L71-L74) `nullAddressOwnsNoLookupRowOnAnyKey`<br>The zero address never looks up to a pool as a share token. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9563) |
| [SR-VS-43](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L282-L285) `sharePriceStampNotFuture`<br>A share price timestamp never runs ahead of the chain clock. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8630) |
| [SR-VS-44](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L287-L290) `assetPriceStampNotFuture`<br>An asset price timestamp never runs ahead of the chain clock. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8631) |
| [SR-VS-45](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_valid_state.spec#L292-L295) `onlyADeployedContractHoldsAVaultRow`<br>Only an address with deployed code is ever a registered vault. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1229) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8923) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SR-ST-01](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L149-L161) `stSrRegisteredVaultStartsUnlinked`<br>A newly registered vault starts unlinked, so it does not serve investors yet. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1232) |
| [SR-ST-02](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L163-L176) `stSrPolicySwapAdvancesTheInstallCounter`<br>A policy change bumps the pool's version, retiring outstanding authorizations. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1235) |
| [SR-ST-03](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L178-L195) `stSrGrantsLandOnlyOnTheInstalledPolicy`<br>Authorizations are granted only under the pool's current policy and version. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1236) |
| [SR-ST-04](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L197-L208) `stSrPriceStampsNeverRewind`<br>Price timestamps never decrease, so an older price cannot replace a newer one. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1238) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1240) |
| [SR-ST-05](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L210-L225) `stSrAuthorizationLedgerStepsByOneInTheStandingNamespace`<br>Authorization counts change by one, only under the current policy and version. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1247) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1248) |
| [SR-ST-06](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L227-L241) `stSrRegisteredVaultKeepsItsIdentity`<br>A registered vault's pool, share class and asset never change. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1251) |
| [SR-ST-07](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L243-L252) `stSrPoolEntryStampIsWrittenOnce`<br>A pool's creation time, once set, never changes. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-1252) |
| [SR-ST-08](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L78-L95) `stSrRoleSeatsArePerPool`<br>A call changing a manager or bridger right on one pool changes no such right on another. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9566) |
| [SR-ST-09](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L254-L268) `stSrAtMostOneVaultLinkMovesPerCall`<br>A call changes the linked status of at most one vault. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9031) |
| [SR-ST-10](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L270-L286) `stSrRoleSeatsMoveOnlyOnTheirOwnWriter`<br>Manager rights change only through updateManager and bridger rights only through updateBridger. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9203) |
| [SR-ST-11](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L97-L114) `stSrRegisteredVaultIsNeverTakenByAnotherPool`<br>No call, even one for another pool, reassigns a registered vault. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8924) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SR-VT-01](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L5-L18) `vtSrShareTokenNeverClears`<br>A share class never loses its share token once it has one. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4533) |
| [SR-VT-02](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L20-L34) `vtSrPolicyTenureCounterStepsForwardByOne`<br>A call leaves the pool's policy nonce unchanged or raises it by one. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4534) |
| [SR-VT-03](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L36-L50) `vtSrAssetCounterStepsForwardByOne`<br>A call leaves the asset id counter unchanged or raises it by one. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4535) |
| [SR-VT-04](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L52-L69) `vtSrIssuedAssetNumberKeepsItsBinding`<br>An issued asset id always refers to the same token and token id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4535) |
| [SR-VT-05](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L71-L84) `vtSrRegisteredAssetKeepsItsNumber`<br>A token's asset id never changes once assigned. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4536) |
| [SR-VT-06](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L86-L108) `vtSrPoolRolesMoveOneAccountAtATime`<br>One call changes the manager or bridger rights of at most one account. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4537) |
| [SR-VT-07](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L110-L127) `vtSrAuthorizationLedgerMovesOnlyThroughItsThreeWriters`<br>Authorizations change only through the grant, revoke and consume calls. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4531) |
| [SR-VT-08](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_transitions.spec#L129-L145) `vtSrVaultLinkMovesOnlyOnLinkOrUnlink`<br>A vault's linked status changes only through linkVault or unlinkVault. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9070) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [SR-HL-01](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L3-L18) `hlSrConsumptionSpendsExactlyOneGrant`<br>An authorization covers exactly one use by the pool's policy. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4527) |
| [SR-HL-02](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L20-L39) `hlSrPolicySwapOrphansOutstandingGrants`<br>After a policy change, earlier authorizations can no longer be spent. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4528) |
| [SR-HL-03](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L41-L56) `hlSrAuthorizationLedgerCountsGrants`<br>Repeated authorizations add up, and a revocation removes only one of them. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4527) |
| [SR-HL-04](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L58-L71) `hlSrCreatedAssetPairResolvesToIssuedId`<br>A newly registered asset maps to the id its registration returned. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4529) |
| [SR-HL-05](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L73-L84) `hlSrCreatedAssetIdResolvesToItsAsset`<br>A new asset id resolves back to the registered asset address. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4530) |
| [SR-HL-06](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L86-L99) `hlSrCreatedAssetIdResolvesToItsTokenId`<br>A new asset id resolves back to the registered token id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4530) |
| [SR-HL-07](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L101-L115) `hlSrAssetQuoteIsStoredExactlyAsSent`<br>An asset price update stores exactly the price and timestamp the hub sent. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4532) |
| [SR-HL-08](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L117-L129) `hlSrShareQuoteIsStoredExactlyAsSent`<br>A share price update stores exactly the price and timestamp the hub sent. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9067) |
| [SR-HL-09](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L131-L151) `hlSrShareClassBindsExactlyTheTokenAndRegistrarNamed`<br>A share class gets the given token and registrar, and the token maps back to it. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9068) |
| [SR-HL-10](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L153-L165) `hlSrAddedPoolIsStampedWithTheBlockTime`<br>Adding a pool records the current block time, never zero, and the pool then reads as active. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9069) |
| [SR-HL-11](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L167-L183) `hlSrTokenSwapDeletesTheOutgoingTokensRow`<br>Moving a share class to a new token leaves the old token resolving to no pool or class. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9202) |
| [SR-HL-12](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L118-L142) `hlSrRoleGrantLandsOnTheNamedPoolOnly`<br>A manager or bridger update sets that right as asked and leaves the other right and pool alone. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9565) |
| [SR-HL-13](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L185-L197) `hlSrIssuedAssetIdIsTheChainOverTheCounter`<br>A new asset id joins its nonzero network id with the next count, and the count advances by one. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9943) |
| [SR-HL-14](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_high_level.spec#L199-L215) `hlSrLinkCallMovesOnlyItsOwnVaultsFlag`<br>Linking or unlinking one vault never changes another vault's linked status. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9210) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [SR-RV-01](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L3-L16) `rvSrAddPoolRefusesANonWardOrAPoolAlreadyOpen`<br>Adding a pool fails for a non-admin or a pool already added. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4502) |
| [SR-RV-02](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L18-L35) `rvSrAddShareClassRefusesARepeatOrAnUnusableRegistration`<br>Adding a share class fails for a non-admin, inactive pool, repeat, or missing token or registrar. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4500) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6103) |
| [SR-RV-03](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L37-L53) `rvSrLinkTokenRefusesANonWardADeadPoolOrAnUnusableToken`<br>Relinking a share token fails for a non-admin, an inactive pool, or a missing token or registrar. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4500) |
| [SR-RV-04](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L55-L69) `rvSrSetRequestManagerRefusesANonWardOrAPoolNeverOpened`<br>Setting a request manager fails for a non-admin or a pool never added. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4500) |
| [SR-RV-05](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L71-L85) `rvSrUpdateManagerRefusesANonWardOrAPoolNeverOpened`<br>Changing a pool manager fails for a non-admin or a pool never added. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4500) |
| [SR-RV-06](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L87-L101) `rvSrUpdateBridgerRefusesANonWardOrAPoolNeverOpened`<br>Changing a bridger fails for a non-admin or a pool never added. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4500) |
| [SR-RV-07](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L103-L115) `rvSrSetPolicyRefusesANonWardOrAPoolNeverOpened`<br>Setting a policy fails for a non-admin or a pool never added. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4500) |
| [SR-RV-08](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L117-L129) `rvSrAuthorizeRefusesANonWardOrAPoolWithNoPolicy`<br>Recording an authorization fails for a non-admin or a pool without a policy. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4503) |
| [SR-RV-09](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L131-L146) `rvSrUnauthorizeRefusesANonWardOrAGrantThatDoesNotStand`<br>Revoking an authorization fails for a non-admin or with none outstanding under the current policy. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4504) |
| [SR-RV-10](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L148-L164) `rvSrConsumeAuthorizationRefusesAnyCallerButThePolicyOrAnEmptySlot`<br>Consuming an authorization fails for any caller but the pool's policy, or with none outstanding. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4505) |
| [SR-RV-11](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L166-L187) `rvSrRegisterVaultRefusesARepeatOrAMismatchedRegistration`<br>Vault registration fails for a non-admin, unknown class, repeat, wrong asset, or undeployed vault. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4501) |
| [SR-RV-12](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L189-L210) `rvSrLinkVaultRefusesAnUnknownOrAlreadyLinkedVault`<br>Vault linking fails for a non-admin, unknown asset or vault, linked vault, or registration mismatch. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4506) |
| [SR-RV-13](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L212-L232) `rvSrUnlinkVaultRefusesAnUnroutedOrMismatchedVault`<br>Vault unlinking fails for a non-admin, unknown asset, unlinked vault, or registration mismatch. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4507) |
| [SR-RV-14](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L234-L248) `rvSrCreateAssetIdRefusesANonWardANullAssetOrAPairAlreadyNumbered`<br>Creating an asset id fails for a non-admin, the zero address, or an asset already registered. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4508) |
| [SR-RV-15](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L250-L267) `rvSrUpdatePricePoolPerShareRefusesANonWardADeadClassOrAnOlderStamp`<br>A share price update fails for a non-admin, an unknown class, or an older price. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4501) |
| [SR-RV-16](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L269-L287) `rvSrUpdatePricePoolPerAssetRefusesANonWardAnUnknownAssetOrAnOlderStamp`<br>An asset price update fails for a non-admin, an unknown class or asset, or an older price. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4501) |
| [SR-RV-17](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L289-L302) `rvSrIdToAssetFailClosedRefusesAnUnregisteredId`<br>A fail-closed asset lookup reverts on an unregistered id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4509) |
| [SR-RV-18](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L304-L315) `rvSrIdToAssetLookupAlwaysAnswers`<br>The plain asset lookup never fails, even for an unregistered id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4509) |
| [SR-RV-19](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L317-L329) `rvSrAssetToIdFailClosedRefusesAnUnregisteredPair`<br>A fail-closed asset id lookup reverts on an unregistered asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4510) |
| [SR-RV-20](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L331-L342) `rvSrAssetToIdLookupAlwaysAnswers`<br>The plain asset id lookup never fails, even for an unregistered asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4510) |
| [SR-RV-21](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L344-L356) `rvSrPricePoolPerShareCheckedRefusesANeverComputedPrice`<br>A validity-checked share price read fails when no price was ever set. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4511) |
| [SR-RV-22](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L358-L370) `rvSrPricePoolPerAssetCheckedRefusesANeverComputedPrice`<br>A validity-checked asset price read fails when no price was ever set. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4512) |
| [SR-RV-23](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L372-L383) `rvSrRelyRefusesANonWard`<br>Granting registry admin rights fails for a non-admin. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4538) |
| [SR-RV-24](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L385-L396) `rvSrDenyRefusesANonWard`<br>Revoking registry admin rights fails for a non-admin. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4538) |
| [SR-RV-25](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L398-L407) `rvSrAddPoolRefusesTheNullPool`<br>Adding pool zero always fails, whoever calls and whatever the registry holds. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9200) |
| [SR-RV-26](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L409-L424) `rvSrStandingGrantIsSpendableUnderAnyNamedCaller`<br>An unpaid spend of an outstanding grant by the pool's policy never fails, whatever caller it names. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9209) |
| [SR-RV-27](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L426-L442) `rvSrHonestSharePriceUpdateIsAccepted`<br>An admin's share price update for a known class never fails unless older or future-dated. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8632) |
| [SR-RV-28](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L444-L461) `rvSrHonestAssetPriceUpdateIsAccepted`<br>An admin's asset price update for a known class and asset never fails unless older or future-dated. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8633) |
| [SR-RV-29](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L463-L473) `rvSrSharePriceStampedAheadOfTheClockIsRefused`<br>A future-dated share price update always fails, whoever calls. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8634) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8921) |
| [SR-RV-30](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L475-L485) `rvSrAssetPriceStampedAheadOfTheClockIsRefused`<br>A future-dated asset price update always fails, whoever calls. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8635) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8922) |
| [SR-RV-31](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L487-L506) `rvSrUnlinkVaultIsNeverRefusedForALinkedVaultFiledUnderItsIds`<br>Unlinking a linked vault never fails for an admin naming its pool, share class and asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8925) |
| [SR-RV-32](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reverts.spec#L508-L522) `rvSrSetRequestManagerIsNeverRefusedForAWardOnAnOpenedPool`<br>Setting a request manager never fails for an admin on an added pool. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8926) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [SR-RC-01](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L3-L16) `rcSrAddPoolIsReachable`<br>A pool can be added, recording the current block time as its creation time. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6100) |
| [SR-RC-02](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L18-L30) `rcSrPoolReadsActiveRightAfterAddIsReachable`<br>A newly added pool can read as active in the same transaction. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6102) |
| [SR-RC-03](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L32-L51) `rcSrTokenSwapRetiringTheOutgoingTokenIsReachable`<br>A share class can move to a new token, unmapping the old one. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6104) |
| [SR-RC-04](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L53-L71) `rcSrRegistrarSwapKeepingTheIncumbentTokenIsReachable`<br>A share class can change its registrar while keeping its token. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6104) |
| [SR-RC-05](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L73-L86) `rcSrSetRequestManagerIsReachable`<br>A request manager can be installed on a pool that had none. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6105) |
| [SR-RC-06](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L88-L100) `rcSrClearingTheRequestManagerIsReachable`<br>A pool's request manager can be cleared, switching requests off. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6105) |
| [SR-RC-07](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L102-L116) `rcSrGrantingOneManagerSeatIsReachable`<br>Manager rights can be granted to one account while another stays without them. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6106) |
| [SR-RC-08](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L118-L133) `rcSrRevokingOneOfTwoManagerSeatsIsReachable`<br>One of two pool managers can lose its rights while the other keeps them. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6106) |
| [SR-RC-09](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L135-L149) `rcSrGrantingTheBridgerSeatAloneIsReachable`<br>An account can get bridger rights without holding manager rights. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6107) |
| [SR-RC-10](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L151-L167) `rcSrFirstPolicyInstallIsReachable`<br>A pool's first policy can be installed, starting its install count at one. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6108) |
| [SR-RC-11](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L169-L185) `rcSrPolicyReplacementIsReachable`<br>A pool's policy can be replaced by another, advancing its install count. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6109) |
| [SR-RC-12](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L187-L202) `rcSrClearingThePolicyIsReachable`<br>A pool's policy can be removed, which still advances its install count. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6109) |
| [SR-RC-13](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L204-L220) `rcSrAuthorizeIsReachable`<br>An admin can grant an authorization under the pool's installed policy. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6110) |
| [SR-RC-14](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L222-L237) `rcSrTwoOutstandingGrantsAreReachable`<br>The same call can be authorized twice, with both grants counted. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6124) |
| [SR-RC-15](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L239-L258) `rcSrUnauthorizeEmptyingTheRowIsReachable`<br>An unspent authorization can be revoked, leaving none outstanding. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6111) |
| [SR-RC-16](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L260-L279) `rcSrPolicyConsumingAGrantIsReachable`<br>The pool's policy can spend an authorization granted for it. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6112) |
| [SR-RC-17](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L281-L299) `rcSrConsumingOneOfTwoGrantsIsReachable`<br>The policy can spend one of two authorizations, leaving the other for later. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6112) |
| [SR-RC-18](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L301-L322) `rcSrPolicySwapPuttingGrantsBeyondRevocationIsReachable`<br>After a policy change, a revocation can leave the old policy's grant in place. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6111) |
| [SR-RC-19](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L324-L342) `rcSrFirstAssetNumberIsReachable`<br>The first asset can be registered and resolves both ways between id and asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6113) |
| [SR-RC-20](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L344-L364) `rcSrSecondAssetPairTakingItsOwnNumberIsReachable`<br>A second asset can be registered under its own distinct id. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6113) |
| [SR-RC-21](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L366-L381) `rcSrNumberingAMultiTokenAssetPairIsReachable`<br>An asset id can be issued for one token of a multi-token (ERC6909) contract. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6114) |
| [SR-RC-22](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L383-L398) `rcSrAssetNumberOnTheSmallestOriginChainIsReachable`<br>An asset id can be issued under network id 1, the lowest valid one. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6115) |
| [SR-RC-23](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L400-L417) `rcSrCheckedAssetLookupsAfterNumberingAreReachable`<br>A new asset id can be looked up both ways at once, without reverting. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6116) |
| [SR-RC-24](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L419-L439) `rcSrRegisterVaultIsReachable`<br>A vault can be registered for a share class and an ERC20 asset, still unlinked. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6117) |
| [SR-RC-25](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L441-L460) `rcSrRegisterMultiTokenVaultIsReachable`<br>A vault can be registered for one token of a multi-token asset contract. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6117) |
| [SR-RC-26](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L462-L476) `rcSrLinkVaultIsReachable`<br>A registered vault can be linked so that it starts serving investors. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6118) |
| [SR-RC-27](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L478-L492) `rcSrUnlinkVaultIsReachable`<br>A linked vault can be unlinked while staying registered. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6119) |
| [SR-RC-28](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L494-L517) `rcSrWithdrawingAndRestoringVaultServiceIsReachable`<br>A vault can be linked, unlinked and linked again in one transaction. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6119) |
| [SR-RC-29](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L519-L535) `rcSrLinkedVaultReadingBackThroughItsViewsIsReachable`<br>A newly linked vault can read back as linked and registered. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6118) |
| [SR-RC-30](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L537-L553) `rcSrFirstSharePriceIsReachable`<br>A share class can receive its first price together with its computation time. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6121) |
| [SR-RC-31](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L555-L570) `rcSrSharePriceMovingToALaterStampIsReachable`<br>A share price can be replaced by one computed at a later time. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6121) |
| [SR-RC-32](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L572-L589) `rcSrSharePriceCorrectionAtTheSameStampIsReachable`<br>A share price can be corrected while keeping the same computation time. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6120) |
| [SR-RC-33](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L591-L608) `rcSrFirstAssetPriceIsReachable`<br>An asset of a share class can receive its first price with its computation time. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6122) |
| [SR-RC-34](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L610-L624) `rcSrCheckedSharePriceReadAfterAWriteIsReachable`<br>A share price just written can be read back through the validity-checked read. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6121) |
| [SR-RC-35](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L626-L641) `rcSrCheckedAssetPriceReadAfterAWriteIsReachable`<br>An asset price just written can be read back through the validity-checked read. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6122) |
| [SR-RC-36](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L643-L662) `rcSrFreshWardOpeningAPoolIsReachable`<br>A newly added admin can open a pool in the same transaction. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6101) |
| [SR-RC-37](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L664-L678) `rcSrDenyLeavingTheCallersOwnSeatIsReachable`<br>An admin can revoke another admin while keeping its own rights. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6123) |
| [SR-RC-38](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L680-L702) `rcSrWholePoolBringUpIsReachable`<br>A pool can go from unregistered to a linked vault in one transaction. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6101) |
| [SR-RC-39](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L704-L719) `rcSrRevokingOneOfTwoBridgerSeatsIsReachable`<br>One of two bridgers can lose bridger rights while the other keeps them. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6107) |
| [SR-RC-40](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L721-L735) `rcSrYearOldSharePriceIsAcceptedAndServedIsReachable`<br>An admin can store a share price a year old, and a later checked read returns it. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9204) |
| [SR-RC-41](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L737-L753) `rcSrTokenSwapKeepsTheStandingSharePriceIsReachable`<br>An admin can move a share class to a new token while its share price and timestamp stay. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9206) |
| [SR-RC-42](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L755-L776) `rcSrSharePriceLandsWithoutItsAssetPriceAndAnOlderAssetQuoteFollowsIsReachable`<br>An admin can post a share price with no asset price, then an asset price older than it. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9207) |
| [SR-RC-43](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L778-L801) `rcSrOneGrantKeyIsSpentUnderTwoDistinctCallersIsReachable`<br>A pool's policy can spend two grants of the same call on behalf of two different callers. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9208) |
| [SR-RC-44](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L146-L165) `rcSrSecondClassGoesLiveBesideALiveFirstClassIsReachable`<br>An admin can add a second share class to a pool beside a live first, each with its own token. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9567) |
| [SR-RC-45](./specs/core/spoke/SpokeRegistry/SpokeRegistry_two_keys_properties.spec#L167-L183) `rcSrTokenSwapBesideALiveSecondClassIsReachable`<br>An admin can swap a share class to a new token, retiring the old, while another keeps its own. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9567) |
| [SR-RC-46](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_reachability.spec#L803-L818) `rcSrTwoVaultsLinkedForOneAssetIsReachable`<br>An admin can link two vaults for the same pool, share class and asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-6118) [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-8927) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [SR-AC-01](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L3-L18) `acSrWardSetMovesOnlyForAWard`<br>Only an admin can grant or revoke admin rights on the spoke registry. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4526) |
| [SR-AC-02](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L20-L35) `acSrPoolEntryStampMovesOnlyForAWard`<br>Only an admin can add a pool to the spoke. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4513) |
| [SR-AC-03](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L37-L56) `acSrShareClassBindingMovesOnlyForAWard`<br>Only an admin can set a share class's token or registrar. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4514) |
| [SR-AC-04](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L58-L77) `acSrTokenLookupRowMovesOnlyForAWard`<br>Only an admin can change which pool and share class a token maps to. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4514) |
| [SR-AC-05](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L79-L95) `acSrRequestManagerMovesOnlyForAWard`<br>Only an admin can change a pool's request manager. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4515) |
| [SR-AC-06](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L97-L112) `acSrManagerRightsMoveOnlyForAWard`<br>Only an admin can grant or revoke pool manager rights. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4516) |
| [SR-AC-07](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L114-L129) `acSrBridgerRightsMoveOnlyForAWard`<br>Only an admin can grant or revoke bridger rights (cross-chain share transfers). | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4517) |
| [SR-AC-08](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L131-L149) `acSrPolicyInstallMovesOnlyForAWard`<br>Only an admin can install, replace or remove a pool's policy. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4518) |
| [SR-AC-09](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L151-L166) `acSrAuthorizationGrantedOnlyByAWard`<br>Only an admin can grant an authorization. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4519) |
| [SR-AC-10](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L168-L185) `acSrAuthorizationSpentOrRevokedOnlyByAWardOrTheInstalledPolicy`<br>Only an admin or the pool's installed policy can spend or revoke an authorization. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4520) |
| [SR-AC-11](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L187-L212) `acSrAssetRegistryMovesOnlyForAWard`<br>Only an admin can register an asset or change its id mapping. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4521) |
| [SR-AC-12](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L214-L241) `acSrVaultRegistrationMovesOnlyForAWard`<br>Only an admin can register a vault or change what it is registered for. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4522) |
| [SR-AC-13](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L243-L258) `acSrVaultLinkFlagMovesOnlyForAWard`<br>Only an admin can link or unlink a vault. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4523) |
| [SR-AC-14](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L260-L278) `acSrSharePriceMovesOnlyForAWard`<br>Only an admin can update a share class price. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4524) |
| [SR-AC-15](./specs/core/spoke/SpokeRegistry/single/SpokeRegistry_access_control.spec#L280-L298) `acSrAssetPriceMovesOnlyForAWard`<br>Only an admin can update an asset price. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-4525) |

### SnapshotQueue

- Single pool
  - One symbolic pool
  - One symbolic share class
  - Queue rows bounded to 3 symbolic rows on 3 distinct assets
- Multi pool
  - Two symbolic pools
  - One share class per pool
  - Queue rows bounded to 3 symbolic rows on 3 distinct assets
- All assets
  - One symbolic pool
  - One symbolic share class
  - Every asset row
- Unpinned
  - No pool, share class or asset pinned

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [SQ-VS-01](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_valid_state.spec#L9-L12) `zeroDeltaIsNonPositive`<br>A fully netted share queue is never flagged as an issuance. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1402) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1403) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1404) |
| [SQ-VS-02](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_valid_state.spec#L14-L17) `queuedAssetCounterMatchesOutstandingRows`<br>The pending-asset count always equals the number of assets with queued flows. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1405) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1406) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1407) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1408) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1409) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1410) |
| [SQ-VS-03](./specs/core/spoke/SnapshotQueue/SnapshotQueue_all_assets_properties.spec#L10-L13) `queuedAssetCounterCountsEveryNonEmptyRow`<br>A share class's pending-asset count is its number of assets with queued flows, however many exist. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-9360) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SQ-ST-01](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_transitions.spec#L46-L66) `stSqAssetFlushEmptiesTheRowAndSpendsOneOrdinal`<br>A flushed asset queue is emptied, no longer counted as pending, and uses one nonce. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1411) |
| [SQ-ST-02](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_transitions.spec#L68-L84) `stSqShareFlushHandsOverTheWholeNet`<br>A share flush empties the share queue, never leaving part of it behind. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1412) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6320) |
| [SQ-ST-03](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_transitions.spec#L86-L101) `stSqOpeningAnAssetRowSpendsNoOrdinalAndMovesNoShareNet`<br>Opening an asset queue uses no nonce and keeps the queued share amount. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1413) |
| [SQ-ST-04](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_transitions.spec#L103-L118) `stSqAssetFlowTakesOneSideOnly`<br>Queueing an asset flow updates its deposit or withdrawal side, never both. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1414) |
| [SQ-ST-05](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_transitions.spec#L120-L132) `stSqNonceAdvancesByAtMostOne`<br>The update nonce advances by at most one per call, so no number is skipped. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1417) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1419) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5117) |
| [SQ-ST-06](./specs/core/spoke/SnapshotQueue/SnapshotQueue_multi_pool_properties.spec#L5-L29) `stSqShareRowIsPerPool`<br>Changing one pool's share queue never touches another pool's share queue. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1415) |
| [SQ-ST-07](./specs/core/spoke/SnapshotQueue/SnapshotQueue_multi_pool_properties.spec#L31-L50) `stSqAssetRowIsPerPool`<br>Changing one pool's asset queue never touches another pool's asset queues. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1416) |
| [SQ-ST-08](./specs/core/spoke/SnapshotQueue/SnapshotQueue_multi_pool_properties.spec#L52-L73) `stSqQueueSidesDoNotCrossPools`<br>Changing one pool's share queue never touches another pool's asset queues. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-1416) |
| [SQ-ST-09](./specs/core/spoke/SnapshotQueue/SnapshotQueue_unpinned_properties.spec#L5-L24) `stSqTheShareNetMovesOnlyUnderShareEntries`<br>Only queueing or flushing shares moves a class's queued net share amount or its direction. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-9029) |
| [SQ-ST-10](./specs/core/spoke/SnapshotQueue/SnapshotQueue_unpinned_properties.spec#L26-L50) `stSqAssetQueuesMoveOnlyUnderAssetEntries`<br>Only queueing or flushing assets moves an asset queue or a class's pending-asset count. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-9030) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SQ-VT-01](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_transitions.spec#L5-L27) `vtSqAssetRowsMoveOneAtATime`<br>A call changes the queued amounts of at most one asset. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5122) |
| [SQ-VT-02](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_transitions.spec#L29-L42) `vtSqOnlyAFlushSpendsAnOrdinal`<br>Only flushing shares or assets advances the update nonce. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5113) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [SQ-HL-01](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L5-L17) `hlSqShareFlushReportsATruthfulSnapshot`<br>A share update claims a snapshot exactly when no asset flows are still queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5103) |
| [SQ-HL-02](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L19-L34) `hlSqAssetFlushReportsATruthfulSnapshot`<br>An asset update claims a snapshot exactly when nothing else is still queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5104) |
| [SQ-HL-03](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L36-L55) `hlSqAssetPayloadCarriesTheNetMagnitude`<br>An asset update reports the net of queued deposits and withdrawals. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5105) |
| [SQ-HL-04](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L57-L75) `hlSqAssetPayloadPointsTowardTheLargerSide`<br>An asset update is an increase exactly when queued deposits exceed withdrawals. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5106) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5114) |
| [SQ-HL-05](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L77-L94) `hlSqSharePayloadCarriesTheNetMagnitude`<br>A share update reports the net of queued issuances and revocations. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5107) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5115) |
| [SQ-HL-06](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L96-L113) `hlSqSharePayloadCarriesTheNetSign`<br>A share update is an issuance exactly when queued issuances exceed revocations. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5108) |
| [SQ-HL-07](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L115-L126) `hlSqSecondAssetFlushReportsNothing`<br>Flushing clears an asset's queue, so an immediate second update reports zero. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5109) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5121) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6321) |
| [SQ-HL-08](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L128-L138) `hlSqSecondShareFlushReportsNothing`<br>Flushing clears the share queue, so an immediate second update reports zero. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5110) |
| [SQ-HL-09](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L140-L152) `hlSqConsecutiveAssetFlushesSpendConsecutiveOrdinals`<br>Back-to-back asset updates carry consecutive nonces. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5111) |
| [SQ-HL-10](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L154-L166) `hlSqConsecutiveShareFlushesSpendConsecutiveOrdinals`<br>Back-to-back share updates carry consecutive nonces. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5112) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6309) |
| [SQ-HL-11](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_high_level.spec#L168-L182) `hlSqZeroAmountQueueingCannotStallTheSnapshot`<br>Queueing a zero asset amount leaves the snapshot status unchanged. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5103) |
| [SQ-HL-12](./specs/core/spoke/SnapshotQueue/SnapshotQueue_all_assets_properties.spec#L17-L31) `hlSqShareFlushSnapshotMeansTheClassIsDrained`<br>A share flush claims a snapshot exactly when it leaves the share class with nothing queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-9363) |
| [SQ-HL-13](./specs/core/spoke/SnapshotQueue/SnapshotQueue_all_assets_properties.spec#L33-L47) `hlSqAssetFlushSnapshotMeansTheClassIsDrained`<br>An asset flush claims a snapshot exactly when it leaves the share class with nothing queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-9364) |
| [SQ-HL-14](./specs/core/spoke/SnapshotQueue/SnapshotQueue_unpinned_properties.spec#L54-L81) `hlSqAnAssetFlushEmptiesItsOwnRowOnly`<br>An asset flush empties that asset's queue, changing no other asset queue and not the queued shares. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-9361) [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-9365) |
| [SQ-HL-15](./specs/core/spoke/SnapshotQueue/SnapshotQueue_unpinned_properties.spec#L83-L105) `hlSqAShareFlushClearsOnlyItsNet`<br>A share flush zeroes the queued shares and moves neither an asset queue nor the pending-asset count. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-9362) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [SQ-RV-01](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reverts.spec#L3-L16) `rvSqQueueAssetsRefusesANonWard`<br>Queueing an asset flow fails for a non-admin caller. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6323) |
| [SQ-RV-02](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reverts.spec#L18-L31) `rvSqQueueSharesRefusesANonWard`<br>Queueing a share change fails for a non-admin caller. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6317) |
| [SQ-RV-03](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reverts.spec#L33-L46) `rvSqFlushAssetsRefusesANonWard`<br>Flushing an asset queue to the hub fails for a non-admin caller. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6318) |
| [SQ-RV-04](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reverts.spec#L48-L60) `rvSqFlushSharesRefusesANonWard`<br>Flushing the share queue to the hub fails for a non-admin caller. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6323) |
| [SQ-RV-05](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reverts.spec#L62-L74) `rvSqRelyRefusesANonWard`<br>Granting queue admin rights fails for a non-admin caller. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6323) |
| [SQ-RV-06](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reverts.spec#L76-L88) `rvSqDenyRefusesANonWard`<br>Revoking queue admin rights fails for a non-admin caller. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6319) |
| [SQ-RV-07](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reverts.spec#L90-L106) `rvSqFlushAssetsAlwaysAdmitsAWardWhileOrdinalsRemain`<br>An asset flush by an admin never fails while update nonces remain. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5116) |
| [SQ-RV-08](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reverts.spec#L108-L124) `rvSqFlushSharesAlwaysAdmitsAWardWhileOrdinalsRemain`<br>A share flush by an admin never fails while update nonces remain. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6322) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [SQ-RC-01](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L3-L16) `rcSqQueueAssetsCanOpenADepositRow`<br>A deposit can open a new asset queue for the share class. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6300) |
| [SQ-RC-02](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L18-L31) `rcSqQueueAssetsCanOpenAWithdrawalRow`<br>A withdrawal can open a new asset queue for the share class. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6300) |
| [SQ-RC-03](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L33-L50) `rcSqQueueAssetsCanAccumulateASecondDeposit`<br>A second deposit can add to an asset queue without counting the asset twice. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6302) |
| [SQ-RC-04](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L52-L66) `rcSqQueueAssetsCanHoldBothSidesOfOneRow`<br>One asset queue can hold a deposit and a withdrawal at the same time. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6303) |
| [SQ-RC-05](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L68-L83) `rcSqQueueAssetsCanLeaveThreeAssetsOutstandingAtOnce`<br>Three assets of one share class can have queued flows at the same time. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6302) |
| [SQ-RC-06](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L85-L99) `rcSqQueueAssetsCanAbsorbTheFullAccumulatorWidth`<br>A single deposit of the maximum amount can be queued for an asset. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6301) |
| [SQ-RC-07](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L101-L114) `rcSqQueueAssetsCanMakeAClassOutstandingWithOneUnit`<br>A single unit of an asset can open a queue for the share class. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6301) |
| [SQ-RC-08](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L116-L131) `rcSqZeroAmountQueueLeavesTheSnapshotAvailable`<br>After a zero-amount asset call, the share class can still report a snapshot. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6300) |
| [SQ-RC-09](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L133-L146) `rcSqQueueSharesCanOpenAnIssuanceNet`<br>An issuance can open a net issuance on an empty share queue. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6304) |
| [SQ-RC-10](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L148-L161) `rcSqQueueSharesCanOpenARevocationNet`<br>A revocation can open a net revocation on an empty share queue. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6304) |
| [SQ-RC-11](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L163-L178) `rcSqQueueSharesCanGrowAnIssuanceNet`<br>A further issuance can grow a queued net issuance. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6305) |
| [SQ-RC-12](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L180-L195) `rcSqQueueSharesCanShrinkAnIssuanceNet`<br>A smaller revocation can reduce a queued net issuance without reversing it. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6306) |
| [SQ-RC-13](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L197-L210) `rcSqQueueSharesCanCancelAnIssuanceNetExactly`<br>An equal revocation can cancel a queued net issuance back to zero. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6306) |
| [SQ-RC-14](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L212-L225) `rcSqQueueSharesCanAbsorbTheFullNetWidth`<br>A single share issuance of the maximum amount can be queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6304) |
| [SQ-RC-15](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L227-L244) `rcSqZeroShareQueueLeavesTheNetPayloadIntact`<br>After a zero-share call, a queued issuance can still be reported unchanged. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6307) |
| [SQ-RC-16](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L246-L265) `rcSqALargerRevocationCanFlipTheQueuedNet`<br>A larger revocation can turn a queued net issuance into a net revocation. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6305) |
| [SQ-RC-17](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L267-L287) `rcSqAssetFlushCanHandOverAndReportASnapshot`<br>Flushing the last queued asset can report its amount as a snapshot. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6310) |
| [SQ-RC-18](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L289-L303) `rcSqAssetFlushWithASiblingRowOutstandingCanReportNoSnapshot`<br>An asset update can report an amount but no snapshot if another asset is queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6313) |
| [SQ-RC-19](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L305-L319) `rcSqAssetFlushWithSharesOutstandingCanReportNoSnapshot`<br>An asset update can report no snapshot while a share change is still queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6314) |
| [SQ-RC-20](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L321-L337) `rcSqAssetFlushCanNetTowardTheDepositSide`<br>An asset queued both ways can be reported to the hub as a net deposit. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6311) |
| [SQ-RC-21](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L339-L355) `rcSqAssetFlushCanNetTowardTheWithdrawalSide`<br>An asset queued both ways can be reported to the hub as a net withdrawal. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6312) |
| [SQ-RC-22](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L357-L377) `rcSqEmptyAssetFlushStillSpendsAnOrdinal`<br>A flush of an empty asset queue can still use up an update nonce. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6310) |
| [SQ-RC-23](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L379-L397) `rcSqOffsettingRowCanFlushZeroYetClearTheCount`<br>An asset whose flows cancel out can be flushed as a zero snapshot update. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6312) |
| [SQ-RC-24](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L399-L420) `rcSqShareFlushCanHandOverAnIssuanceAndReportASnapshot`<br>With no asset change queued, a share issuance can reach the hub as a snapshot. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6307) |
| [SQ-RC-25](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L422-L438) `rcSqShareFlushCanHandOverARevocationAndReportASnapshot`<br>With no asset change queued, a share revocation can reach the hub as a snapshot. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6308) |
| [SQ-RC-26](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L440-L455) `rcSqShareFlushWithAnAssetRowOutstandingCanReportNoSnapshot`<br>A share update can report no snapshot while an asset change is still queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6308) |
| [SQ-RC-27](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L457-L475) `rcSqEmptyShareFlushStillSpendsAnOrdinal`<br>With nothing queued, a share flush can still use a nonce and report a snapshot. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6308) |
| [SQ-RC-28](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L477-L502) `rcSqAnOutOfSyncClassCanBeBroughtBackIntoSync`<br>A share class with queued shares and assets can be flushed back into sync. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6313) |
| [SQ-RC-29](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L504-L528) `rcSqAnAssetRowCanBeReusedAfterAFlush`<br>An asset queue can be flushed and reused with nothing left over. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6314) |
| [SQ-RC-30](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L530-L543) `rcSqDenyCanRemoveAnotherAccountsQueueSeat`<br>An admin can revoke another account's admin rights on the queue. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6316) |
| [SQ-RC-31](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L545-L564) `rcSqAFreshlyReliedAccountCanMoveTheQueue`<br>A newly added admin can queue share changes right away. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6315) |
| [SQ-RC-32](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L566-L579) `rcSqAWardCanDenyItself`<br>An admin can revoke its own admin rights on the queue. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6316) |
| [SQ-RC-33](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L581-L599) `rcSqQueuedSharesViewCanReadBackAFreshNet`<br>The share queue view can show an issuance just queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6305) |
| [SQ-RC-34](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_reachability.spec#L601-L618) `rcSqQueuedAssetsViewCanReadBackAFreshFlow`<br>The asset queue view can show a deposit just queued. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-6301) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [SQ-AC-01](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_access_control.spec#L3-L20) `acSqOnlyAWardMovesTheShareNet`<br>Only an admin can change the net share amount queued for the hub. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5101) |
| [SQ-AC-02](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_access_control.spec#L22-L40) `acSqOnlyAWardMovesAnAssetRow`<br>Only an admin can change an asset's queued deposits and withdrawals. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5100) |
| [SQ-AC-03](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_access_control.spec#L42-L57) `acSqOnlyAWardMovesTheOutstandingAssetCount`<br>Only an admin can change the count of assets with queued flows. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5100) |
| [SQ-AC-04](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_access_control.spec#L59-L73) `acSqOnlyAWardSpendsAFlushOrdinal`<br>Only an admin can advance the nonce that orders queued updates for the hub. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5101) |
| [SQ-AC-05](./specs/core/spoke/SnapshotQueue/single/SnapshotQueue_single_pool_access_control.spec#L75-L90) `acSqOnlyAWardMovesTheWardBit`<br>Only an admin can grant or revoke admin rights on the queue. | ✅ | [🎯](./mutations/SnapshotQueue/README.md#snapshotqueue-5102) |

### Escrow

- Outside the configurations named below
  - One symbolic share class
  - One asset row
  - Two symbolic reservers
  - Two symbolic reasons
- Multi row
  - One symbolic share class
  - Two asset rows
  - Two symbolic reservers
  - Two symbolic reasons
- Multi share class
  - Two symbolic share classes
  - One asset row
- Unpinned
  - No share class, asset row, reserver or reason pinned
  - The ward-gated token movers (authTransferTo, both recoverTokens) on the surface, their token and ETH calls left unresolved and havocking only foreign storage
- Custody
  - No share class, asset row, reserver or reason pinned
  - The token movers authTransferTo and recoverTokens against a scene-local permissive token: balances as the model holds them, a transfer beyond the sender's balance succeeds moving nothing, every transfer answers true
  - SafeTransferLib.safeTransfer replaced by the same token model, its call encoding, code check and return decoding trusted
  - A ward caller attaching no value

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [PE-VS-01](./specs/core/spoke/Escrow/single/Escrow_valid_state.spec#L12-L15) `reservedMatchesBucketSum`<br>A holding's reserved amount always equals the sum of its reservations. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-2402) [🎯](./mutations/PoolEscrow/README.md#poolescrow-2403) [🎯](./mutations/PoolEscrow/README.md#poolescrow-2404) [🎯](./mutations/PoolEscrow/README.md#poolescrow-5209) |
| [PE-VS-02](./specs/core/spoke/Escrow/Escrow_multi_row_properties.spec#L5-L9) `everyAssetRowMatchesItsBucketSum`<br>For every asset, the reserved amount equals the sum of that asset's reservations. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-2413) |
| [PE-VS-03](./specs/core/spoke/Escrow/Escrow_unpinned_properties.spec#L5-L8) `reservedIsTheSumOfEveryBucket`<br>Every holding's reserved amount equals the sum of all its reservations. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9395) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [PE-ST-01](./specs/core/spoke/Escrow/single/Escrow_transitions.spec#L5-L18) `stPePayoutBoundedByFreeBalance`<br>No call can reduce a holding by more than its prior available balance. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-2407) |
| [PE-ST-02](./specs/core/spoke/Escrow/single/Escrow_transitions.spec#L20-L35) `stPeOneCallTouchesOneBucket`<br>A single call changes at most one reservation. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-2411) |
| [PE-ST-03](./specs/core/spoke/Escrow/single/Escrow_transitions.spec#L37-L51) `stPeOnlyAWithdrawalLowersTheCustodyTotal`<br>Only a withdrawal lowers the holding total. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5207) |
| [PE-ST-04](./specs/core/spoke/Escrow/Escrow_multi_share_class_properties.spec#L5-L23) `stPeEscrowRowIsPerShareClass`<br>A call never changes the escrow holdings of two share classes at once. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-8802) |
| [PE-ST-05](./specs/core/spoke/Escrow/Escrow_unpinned_properties.spec#L42-L65) `stPeACallMovesAtMostOneHoldingRow`<br>A call moves the total or reserved amount of at most one holding. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9018) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [PE-VT-01](./specs/core/spoke/Escrow/Escrow_unpinned_properties.spec#L12-L38) `vtPeTotalMovesOnlyOnFlowsAndReservedOnlyOnReservations`<br>Only deposit or withdraw moves a holding's total, only reserve or unreserve its reserved amount. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9019) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [PE-HL-01](./specs/core/spoke/Escrow/single/Escrow_high_level.spec#L3-L16) `hlPeAvailableBalanceIsTheUnreservedCustody`<br>The available balance is the unreserved part of the holding, never below zero. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5203) |
| [PE-HL-02](./specs/core/spoke/Escrow/single/Escrow_high_level.spec#L18-L32) `hlPeSweepingTheFreeBalanceLeavesTheEarmarks`<br>Withdrawing all the available balance leaves only the reserved amount held. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5204) |
| [PE-HL-03](./specs/core/spoke/Escrow/single/Escrow_high_level.spec#L34-L48) `hlPeReserveThenReleaseRestoresTheBucket`<br>Reserving then releasing the same amount restores the reservation. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5205) |
| [PE-HL-04](./specs/core/spoke/Escrow/single/Escrow_high_level.spec#L50-L64) `hlPeTakeInThenPayOutRestoresTheCustody`<br>A deposit and withdrawal of the same amount leave the holding total unchanged. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5204) |
| [PE-HL-05](./specs/core/spoke/Escrow/single/Escrow_high_level.spec#L66-L80) `hlPeCustodyRoundTripTouchesNoHold`<br>A deposit and withdrawal of the same amount leave the reserved amount unchanged. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5204) |
| [PE-HL-06](./specs/core/spoke/Escrow/single/Escrow_high_level.spec#L82-L97) `hlPeAnyHoldCapsEveryPayout`<br>A new reservation by any reserver immediately caps what can be withdrawn. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5206) |
| [PE-HL-07](./specs/core/spoke/Escrow/Escrow_custody_properties.spec#L5-L28) `hlPeAuthTransferToMovesExactlyTheAmount`<br>A forced transfer sends exactly the named amount of the token to the named receiver, once. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9581) |
| [PE-HL-08](./specs/core/spoke/Escrow/Escrow_custody_properties.spec#L30-L59) `hlPeCustodyMoversWriteNoBook`<br>A forced transfer or token recovery writes no holding, reservation or ward, under any key. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9009) |
| [PE-HL-09](./specs/core/spoke/Escrow/Escrow_unpinned_properties.spec#L69-L92) `hlPeDepositMovesExactlyItsOwnRow`<br>A deposit changes only its holding's total, adding exactly the amount. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9391) |
| [PE-HL-10](./specs/core/spoke/Escrow/Escrow_unpinned_properties.spec#L94-L117) `hlPeWithdrawMovesExactlyItsOwnRow`<br>A withdrawal changes only its holding's total, taking off exactly the amount. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9392) [🎯](./mutations/PoolEscrow/README.md#poolescrow-9397) |
| [PE-HL-11](./specs/core/spoke/Escrow/Escrow_unpinned_properties.spec#L119-L145) `hlPeReserveMirrorsBucketAndAggregateExactly`<br>Reserving adds the amount to the reservation and to its holding's reserved amount alone. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9393) |
| [PE-HL-12](./specs/core/spoke/Escrow/Escrow_unpinned_properties.spec#L147-L173) `hlPeUnreserveMirrorsBucketAndAggregateExactly`<br>Releasing takes the amount off the reservation and off its holding's reserved amount alone. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9047) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [PE-RV-01](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L3-L16) `rvPeDepositRefusesANonWard`<br>A deposit fails for a caller without admin rights. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6822) |
| [PE-RV-02](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L18-L36) `rvPeWithdrawRefusesANonWardOrAPayoutBeyondTheFreeBalance`<br>A withdrawal fails for a non-admin, an over-reserved holding, or more than is available. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-8953) |
| [PE-RV-03](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L38-L51) `rvPeReserveRefusesANonWard`<br>A reservation fails for a caller without admin rights. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6822) |
| [PE-RV-04](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L53-L69) `rvPeUnreserveRefusesANonWardOrAShortBucket`<br>Releasing a reservation fails for a non-admin or for more than it holds. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6820) |
| [PE-RV-05](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L71-L83) `rvPeRelyRefusesANonWard`<br>Granting escrow admin rights fails for a non-admin caller. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6822) |
| [PE-RV-06](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L85-L97) `rvPeDenyRefusesANonWard`<br>Revoking escrow admin rights fails for a non-admin caller. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6821) |
| [PE-RV-07](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L99-L110) `rvPeAvailableBalanceOfAlwaysAnswers`<br>The available balance query never fails, even when a holding is over-reserved. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5213) |
| [PE-RV-08](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L112-L125) `rvPeAuthTransferToRefusesANonWard`<br>Transferring assets out of the escrow fails for a non-admin caller. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5214) |
| [PE-RV-09](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L127-L139) `rvPeRecoverTokensRefusesANonWard`<br>Recovering ETH or an ERC20 token fails for a non-admin caller. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5215) |
| [PE-RV-10](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L141-L154) `rvPeRecoverTokensByTokenIdRefusesANonWard`<br>Recovering a token by token id fails for a non-admin caller. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5216) |
| [PE-RV-11](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L156-L171) `rvPeUnreserveAlwaysAdmitsAKeyReleasingItsWholeBucket`<br>An admin can always release a reservation, up to its full amount. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5210) |
| [PE-RV-12](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L173-L185) `rvPeDenyAlwaysAdmitsAWardStandingItselfDown`<br>An admin can always revoke its own admin rights. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5212) |
| [PE-RV-13](./specs/core/spoke/Escrow/Escrow_custody_properties.spec#L63-L76) `rvPeAuthTransferToRefusesAShortCustody`<br>A forced transfer of more than the escrow holds fails, even on a token that would allow it. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9580) |
| [PE-RV-14](./specs/core/spoke/Escrow/single/Escrow_reverts.spec#L187-L203) `rvPeWithdrawAlwaysAdmitsAWardWithinTheFreeBalance`<br>An admin's withdrawal of at most the free balance never fails. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-8920) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [PE-RC-01](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L3-L18) `rcPeDepositBooksAnErc6909Row`<br>A deposit can increase the holding of an ERC6909 token. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6802) |
| [PE-RC-02](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L20-L36) `rcPeDepositReopensAFullyEncumberedRow`<br>A deposit into a fully reserved holding can make funds available again. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6803) |
| [PE-RC-03](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L38-L52) `rcPeWithdrawEmptiesAnUnencumberedRow`<br>A withdrawal can empty a holding that has no reservations. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6804) |
| [PE-RC-04](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L54-L70) `rcPeWithdrawSweepsTheAdvertisedFreeBalance`<br>The whole available balance can be withdrawn at once while reservations remain. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5208) [🎯](./mutations/PoolEscrow/README.md#poolescrow-6805) |
| [PE-RC-05](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L72-L88) `rcPeWithdrawLeavesTheRowStillPaying`<br>A partial withdrawal from a reserved holding can leave funds still available. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6806) |
| [PE-RC-06](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L90-L104) `rcPeWithdrawBooksAPayoutToANullReceiver`<br>A withdrawal can be recorded with the zero address as receiver. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6807) |
| [PE-RC-07](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L106-L123) `rcPeReserveOpensABucket`<br>A new reservation can be opened, raising the holding's reserved amount. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6808) |
| [PE-RC-08](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L125-L141) `rcPeReserveBooksABucketForAnUnauthorizedKey`<br>An admin can reserve funds for another account that has no admin rights. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6809) |
| [PE-RC-09](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L143-L162) `rcPeUnreserveEmptiesAForeignBucket`<br>An admin can fully release another account's reservation. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6810) |
| [PE-RC-10](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L164-L185) `rcPeTwoReserversHoldOneRowAtOnce`<br>Two reservers can hold reservations on the same holding at once. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6811) |
| [PE-RC-11](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L187-L209) `rcPeOneReserverHoldsTwoReasons`<br>One reserver can hold two reservations on a holding under different reasons. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6812) |
| [PE-RC-12](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L211-L228) `rcPeReserveBeyondCustodyFreezesTheRow`<br>One reservation can over-reserve a holding, leaving nothing available. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6813) |
| [PE-RC-13](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L230-L248) `rcPeReserveEncumbersTheWholeFreeBalance`<br>One reservation can take the whole available balance, leaving nothing free. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6808) |
| [PE-RC-14](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L250-L266) `rcPeUnreserveClearsTheRowsEarmark`<br>Releasing one full reservation can leave the holding with nothing reserved. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6814) |
| [PE-RC-15](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L268-L285) `rcPeUnreserveLeavesTheBucketStillHolding`<br>A reservation can be partly released, keeping the rest reserved. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6815) |
| [PE-RC-16](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L287-L312) `rcPeReleasingOneBucketLeavesTheOtherHeld`<br>One reserver's reservation can be released in full while another's stays intact. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6816) |
| [PE-RC-17](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L314-L335) `rcPeReleaseLetsAnOverEncumberedRowPayOutAgain`<br>After a release, an over-reserved holding can allow withdrawals again. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6813) |
| [PE-RC-18](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L337-L361) `rcPeHoldPayoutReleaseWidensTheFreeBalance`<br>A reservation can hold funds back from a withdrawal, then free them on release. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6805) |
| [PE-RC-19](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L363-L377) `rcPeDenyRevokesAnotherWard`<br>An admin can revoke another admin while keeping its own rights. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6818) |
| [PE-RC-20](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L379-L399) `rcPeFreshlyReliedAccountBooksADeposit`<br>A newly added admin can record a deposit in the same block. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-6817) |
| [PE-RC-21](./specs/core/spoke/Escrow/single/Escrow_reachability.spec#L401-L417) `rcPeBucketReadsBackTheHold`<br>The reservation lookup can report exactly the amount just reserved. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9046) |
| [PE-RC-22](./specs/core/spoke/Escrow/Escrow_custody_properties.spec#L80-L105) `rcPeAuthTransferToMovesCustodyWithoutTheLedger`<br>An admin can force tokens out of a fully backed holding, its recorded total left as it was. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9582) |
| [PE-RC-23](./specs/core/spoke/Escrow/Escrow_custody_properties.spec#L107-L133) `rcPeRecoverTokensMovesCustodyWithoutTheLedger`<br>An admin can recover tokens from a fully backed holding, its recorded total left as it was. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-8106) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [PE-AC-01](./specs/core/spoke/Escrow/single/Escrow_access_control.spec#L3-L17) `acPeOnlyWardMovesTheCustodyTotal`<br>Only an admin can change an escrow holding's total. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5200) |
| [PE-AC-02](./specs/core/spoke/Escrow/single/Escrow_access_control.spec#L19-L33) `acPeOnlyWardMovesTheAggregateHold`<br>Only an admin can change an escrow holding's reserved amount. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5201) |
| [PE-AC-03](./specs/core/spoke/Escrow/single/Escrow_access_control.spec#L35-L50) `acPeOnlyWardMovesAReservationBucket`<br>Only an admin can change a reserver's reservation. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5201) |
| [PE-AC-04](./specs/core/spoke/Escrow/single/Escrow_access_control.spec#L52-L66) `acPeOnlyWardMovesAWardSeat`<br>Only an admin can grant or revoke admin rights on the escrow. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-5202) |
| [PE-AC-05](./specs/core/spoke/Escrow/Escrow_unpinned_properties.spec#L177-L193) `acPeWardSeatMovesOnlyThroughRelyOrDeny`<br>Admin rights change only through rely, which grants them, or deny, which revokes them. | ✅ | [🎯](./mutations/PoolEscrow/README.md#poolescrow-9390) [🎯](./mutations/PoolEscrow/README.md#poolescrow-9396) |

### Spoke_LinkedSpokeCore

- Every entry on one symbolic pool
- One symbolic share class
- Asset ids drawn from the queue's bounded set
- ERC20 and ERC6909 tokens modeled in CVL (standard, up to 3 holders, uint128 balances, 6 to 18 decimals, no share token hooks, a holder pulling its own tokens only against an allowance to itself)
- Rows
  - Escrow rows and reservation buckets read through the escrow's getters, not mirrored
  - The queueing legs pinned to the one share class, their asset ids drawn from the queue's bounded set
- Multi pool
  - A second symbolic pool, read only through its escrow, a second linked Escrow instance the factory answers for it; a third pool is not analysed
  - Escrow rows and reservation buckets read through the escrows' getters, not mirrored

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [SP-VS-01](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_valid_state.spec#L22-L26) `queuedAssetRowImpliesRegisteredAsset`<br>Every asset with a queued flow is a registered asset. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6033) |
| [SP-VS-02](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_valid_state.spec#L28-L32) `escrowedRowImpliesRegisteredAsset`<br>The escrow holds or reserves only registered assets. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6033) |
| [SP-VS-03](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_valid_state.spec#L34-L37) `escrowedCustodyImpliesCreatedPool`<br>The escrow holds or reserves funds only for a pool the spoke has created. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6034) |
| [SP-VS-04](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_valid_state.spec#L39-L43) `queueActivityImpliesCreatedPool`<br>Updates are queued for the hub only for a pool the spoke has created. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6034) |
| [SP-VS-05](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_valid_state.spec#L45-L49) `registeredAssetIsLocal`<br>Every asset id registered on a spoke encodes that spoke's own network. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6036) |
| [SP-VS-06](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_valid_state.spec#L51-L60) `escrowCustodyIsBackedOrNoted`<br>The escrow's tokens plus noted deposits always cover the assets on its books. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6037) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SP-ST-02](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_transitions.spec#L30-L51) `stSpSpendableBalanceMoveIsQueuedForTheHub`<br>Each change in unreserved escrow assets is queued for the hub as an equal change. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-205) |
| [SP-ST-03](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_transitions.spec#L53-L70) `stSpDrainingAQueueRowSpendsANonceAndItsSlot`<br>Emptying an asset's queued flow uses one sequence number and dequeues the asset. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-206) [🎯](./mutations/Spoke/README.md#spoke-210) |
| [SP-ST-04](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_transitions.spec#L72-L92) `stSpShareSupplyMovesOnlyAsReported`<br>Outside hub submissions and share bridging, every share supply change is queued. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-207) |
| [SP-ST-05](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_transitions.spec#L94-L116) `stSpEscrowCustodyMovesWithItsTokens`<br>Except withdrawals and noted deposits, escrow tokens and booked assets change equally. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6031) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SP-VT-07](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_transitions.spec#L5-L26) `vtSpNoSpokeEntryMovesAPrice`<br>No spoke call moves the share price, an asset price or the time either was last set. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9329) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [SP-HL-29](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L3-L18) `hlSpDepositThenWithdrawRestoresCustody`<br>A deposit then an equal withdrawal leaves the escrowed amount unchanged. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5500) |
| [SP-HL-30](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L20-L36) `hlSpDepositThenWithdrawRestoresTheQueuedNet`<br>A deposit then an equal withdrawal nets to zero in the hub's pending report. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5501) |
| [SP-HL-31](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L38-L54) `hlSpReserveThenUnreserveLeavesCustodyUntouched`<br>Reserving then unreserving an amount leaves the escrowed amount unchanged. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5502) |
| [SP-HL-32](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L56-L72) `hlSpReserveThenUnreserveRestoresTheAggregateHold`<br>Reserving then unreserving an amount restores the total reserved in escrow. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5503) |
| [SP-HL-33](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L74-L90) `hlSpReserveThenUnreserveRestoresTheReserversBucket`<br>Reserving then unreserving restores the reserver's reservation under that reason. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5504) |
| [SP-HL-34](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L92-L109) `hlSpReserveThenUnreserveRestoresTheQueuedNet`<br>Reserving then unreserving an amount nets to zero in the hub's pending report. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5505) |
| [SP-HL-35](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L111-L126) `hlSpWithdrawReservedPaysOutTheRequestedAmount`<br>A reserved withdrawal takes exactly the requested amount out of escrow. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5500) |
| [SP-HL-36](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L128-L144) `hlSpWithdrawReservedConsumesTheHold`<br>A reserved withdrawal uses up an equal amount of the escrow's reservations. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5503) |
| [SP-HL-37](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L146-L172) `hlSpWithdrawReservedLeavesTheQueueUntouched`<br>A reserved withdrawal queues nothing for the hub. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5506) |
| [SP-HL-38](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L174-L188) `hlSpRevokePullsSharesFromTheCaller`<br>A revoke debits the revoked shares from the caller's balance. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5507) |
| [SP-HL-39](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L190-L204) `hlSpRevokeBurnsThePulledShares`<br>A revoke lowers the share token supply by the revoked amount. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5508) |
| [SP-HL-40](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L206-L220) `hlSpRevokeParksNoSharesOnTheSpoke`<br>A revoke leaves no shares behind on the spoke. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5508) |
| [SP-HL-41](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L222-L235) `hlSpRevokeQueuesTheBurnAsADecrease`<br>A revoke queues a share decrease of exactly the revoked amount for the hub. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5509) |
| [SP-HL-42](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L237-L252) `hlSpRegisterAssetSendsOneRegistrationAnnouncement`<br>Each asset registration sends exactly one RegisterAsset message. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5510) |
| [SP-HL-43](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L254-L266) `hlSpRegisterAssetAnnouncesTheStoredId`<br>An asset registration announces the id the registry stores for that asset. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5511) |
| [SP-HL-44](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L268-L282) `hlSpAssetFlushAnnouncesAnUpdateForTheFlushedRow`<br>Submitting queued assets sends an asset update for the requested asset. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5512) |
| [SP-HL-45](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L284-L301) `hlSpAssetFlushReportsTheRowsNetSize`<br>An asset update reports the size of the net flow queued for that asset. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5513) |
| [SP-HL-46](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L303-L319) `hlSpAssetFlushReportsTheRowsNetDirection`<br>An asset update reports the direction of the net flow queued for that asset. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5514) |
| [SP-HL-47](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L321-L334) `hlSpAssetFlushReportsTheStoredSequenceNumber`<br>An asset update is sent with the queue's current sequence number. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5515) |
| [SP-HL-48](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L336-L349) `hlSpAssetFlushEmptiesTheRow`<br>Submitting an asset's queued flow clears it, so it is not reported twice. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5516) |
| [SP-HL-49](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L351-L362) `hlSpShareFlushAnnouncesAShareUpdate`<br>Submitting queued shares always sends a share update to the hub. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5517) |
| [SP-HL-50](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L364-L377) `hlSpShareFlushReportsTheQueuedNetSize`<br>Submitting queued share changes reports their net size to the hub. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5518) |
| [SP-HL-51](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L379-L392) `hlSpShareFlushReportsTheQueuedNetDirection`<br>A share update reports the direction of the queued net share change. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5519) |
| [SP-HL-52](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L394-L407) `hlSpShareFlushReportsTheStoredSequenceNumber`<br>A share update is sent with the queue's current sequence number. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5520) |
| [SP-HL-53](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L409-L420) `hlSpShareFlushClearsTheQueuedNet`<br>Submitting queued shares clears them, so no share change is reported twice. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5521) |
| [SP-HL-54](./specs/core/spoke/Spoke_LinkedSpokeCore/Spoke_LinkedSpokeCore_rows_properties.spec#L5-L40) `hlSpEscrowEntryMovesItsOwnRowByExactlyTheAmount`<br>A deposit adds the amount to its holding, a withdrawal takes it, a reserved one from both holds too. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9071) |
| [SP-HL-55](./specs/core/spoke/Spoke_LinkedSpokeCore/Spoke_LinkedSpokeCore_rows_properties.spec#L42-L76) `hlSpEscrowEntryTouchesOnlyItsOwnRow`<br>A deposit or withdrawal leaves every other holding and reservation in the escrow unchanged. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9071) [🎯](./mutations/SpokeLinked/README.md#spokelinked-9321) |
| [SP-HL-56](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L422-L443) `hlSpReserveRaisesBucketAndHoldByExactlyTheAmount`<br>A hold raises the reserver's and the total reservation by the amount, leaving custody alone. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9072) |
| [SP-HL-57](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L445-L466) `hlSpUnreserveLowersBucketAndHoldByExactlyTheAmount`<br>A release lowers the reserver's and the total reservation by the amount, leaving custody alone. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9073) |
| [SP-HL-58](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L468-L511) `hlSpZeroAmountEntryLeavesTheQueueUntouched`<br>A zero deposit, withdrawal, hold, release, issue or revoke leaves the hub's pending report alone. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9324) |
| [SP-HL-59](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L513-L530) `hlSpFlushSpendsExactlyOneNonce`<br>Every update sent to the hub, an empty one included, uses up exactly one sequence number. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9325) |
| [SP-HL-60](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L532-L563) `hlSpRegisterAssetGrantsNoSeat`<br>Registering an asset changes no admin right, manager, bridger, request manager or policy. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9326) |
| [SP-HL-61](./specs/core/spoke/Spoke_LinkedSpokeCore/Spoke_LinkedSpokeCore_multi_pool_properties.spec#L5-L46) `hlSpEscrowResolvedEntryLeavesAnotherPoolsEscrowAlone`<br>An escrow call on one pool never changes another pool's escrow books or lowers its balances. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9610) |
| [SP-HL-62](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L565-L586) `hlSpNoteDepositBooksExactlyItsAmount`<br>A noted deposit, counted or not, books the given amount in escrow without moving any tokens. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9066) |
| [SP-HL-63](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L588-L608) `hlSpWithdrawMovesTokensAndBookTogether`<br>A withdrawal to another address changes escrow tokens and books equally. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6038) |
| [SP-HL-64](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L610-L631) `hlSpWithdrawReservedMovesTokensAndBookTogether`<br>A reserved withdrawal to another address changes escrow tokens and books equally. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-6039) |
| [SP-HL-65](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_high_level.spec#L633-L655) `hlSpNoteThenReserveLeavesTheFreeBalanceAndQueuedNet`<br>A deposit noted then reserved leaves free funds and the hub's pending report unchanged. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9066) [🎯](./mutations/SpokeLinked/README.md#spokelinked-5502) [🎯](./mutations/SpokeLinked/README.md#spokelinked-8931) [🎯](./mutations/SpokeLinked/README.md#spokelinked-9072) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [SP-RC-50](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_reachability.spec#L3-L22) `rcSpWithdrawToAnotherAddressIsReachable`<br>A withdrawal can pay another address and lower the escrowed amount by as much. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-5522) |
| [SP-RC-51](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_reachability.spec#L24-L39) `rcSpReRegistrationCanAnnounceNewDecimalsUnderTheSameId`<br>A known asset can be registered again with new decimals under the same asset id. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9327) |
| [SP-RC-52](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_reachability.spec#L41-L61) `rcSpSharePayoutDrainsABookedShareTokenRow`<br>A share payout can move share tokens out of escrow while the escrow's books still count them. | ✅ | [🎯](./mutations/SpokeLinked/README.md#spokelinked-9328) |
| [SP-RC-53](./specs/core/spoke/Spoke_LinkedSpokeCore/single/Spoke_LinkedSpokeCore_reachability.spec#L63-L77) `rcSpARescueToTheEscrowLeavesUnbookedTokens`<br>A ward's rescue to the escrow leaves it holding tokens no holding books. | ✅ | [🎯](./mutations/Spoke/README.md#spoke-9011) |

### SpokeHandler_LinkedSpokeRegistry

- One symbolic pool
- One symbolic share class
- ERC20 tokens modeled in CVL (standard, up to 3 holders, uint128 balances, 6 to 18 decimals, no share token hooks, a holder pulling its own tokens only against an allowance to itself)
- A share token deployment through the registrar never refused, and a restriction update never refused for its hook
- An escrow the factory deploys for a pool taken to land at the address the factory answers for that pool
- Entry gate
  - The share metadata writer kept on the surface
- Vault pointer
  - The real share token compiled for its vault pointer, a pointer write dispatched to it

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SH-ST-01](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L94-L114) `stShVaultLinkNeverMovesItsIdentity`<br>Only a registered vault is linked or unlinked, keeping its pool, class and asset. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-3208) |
| [SH-ST-02](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L116-L133) `stShShareClassBindsTheRegistrarsTokenAndItsLookupBack`<br>A new share token is the one the registrar deployed, and maps back to its pool and class. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-3210) [🎯](./mutations/SpokeHandler/README.md#spokehandler-3219) |
| [SH-ST-03](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L135-L148) `stShPolicySwapAdvancesTheAuthorizationNamespace`<br>Changing a pool's policy invalidates its outstanding authorizations. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-3205) |
| [SH-ST-04](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L150-L165) `stShAuthorizationsMoveOnlyWithinAStandingNamespace`<br>Authorizations can be granted or revoked only while the pool's policy stays unchanged. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-3209) |
| [SH-ST-05](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L167-L190) `stShPricesNeedTheRowsTheyPriceTo`<br>Prices are recorded only for existing share classes and registered assets. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-3206) |
| [SH-ST-06](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L192-L209) `stShPriceStampsNeverRunAheadOfTheClock`<br>A price update never records a timestamp later than the spoke's current time. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8636) [🎯](./mutations/SpokeHandler/README.md#spokehandler-8637) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SH-VT-01](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L6-L22) `vtShLiveShareClassKeepsItsTokenAndRegistrar`<br>A share class with a token never changes its token or registrar. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4739) |
| [SH-VT-02](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L24-L38) `vtShShareTokenKeepsThePoolItBacks`<br>A share token assigned to a pool stays assigned to that pool. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4739) |
| [SH-VT-03](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L40-L56) `vtShShareSupplyMovesOnlyOnTheTransferExecutor`<br>Only an incoming share transfer can change token supply through the handler. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4729) |
| [SH-VT-04](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L58-L73) `vtShManagerSeatMovesOnlyOnItsRoleMessage`<br>Through the handler, a pool manager role changes only on a manager update message. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4733) |
| [SH-VT-05](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_transitions.spec#L75-L90) `vtShBridgerSeatMovesOnlyOnItsRoleMessage`<br>Through the handler, a share bridging role changes only on a bridger update message. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4734) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [SH-HL-01](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L4-L18) `hlShExecuteTransferSharesMintsExactlyTheBridgedAmount`<br>An incoming cross-chain share transfer mints exactly the transferred amount. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4727) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6708) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6709) |
| [SH-HL-02](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L20-L36) `hlShExecuteTransferSharesPaysTheReceiverInFull`<br>The receiver of an incoming share transfer gets the full transferred amount. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4728) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6707) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6709) |
| [SH-HL-03](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L38-L54) `hlShExecuteTransferSharesAddressedPastTheHandlerLeavesNoResidue`<br>The handler keeps none of the shares it mints for a transfer to another receiver. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4728) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6707) |
| [SH-HL-04](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L56-L69) `hlShDeployAndLinkLeavesTheNewVaultLinked`<br>A deploy-and-link vault update leaves the new vault linked. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4730) |
| [SH-HL-05](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L71-L92) `hlShDeployAndLinkRegistersTheVaultRowTheMessageNames`<br>A deploy-and-link update registers the vault for the named pool, class and asset. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-9074) |
| [SH-HL-06](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L94-L108) `hlShLinkThenUnlinkRestoresTheRoutingFlag`<br>Linking then unlinking a vault restores its original linked status. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4732) |
| [SH-HL-07](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L110-L134) `hlShLinkThenUnlinkLeavesTheVaultIdentityUntouched`<br>Linking then unlinking a vault leaves its pool, share class and asset unchanged. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4732) |
| [SH-HL-08](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L136-L148) `hlShAddPoolAsksTheEscrowFactoryExactlyOnce`<br>Registering a pool requests exactly one escrow from the escrow factory. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4735) |
| [SH-HL-09](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L150-L160) `hlShAddPoolAsksTheEscrowFactoryForThatPool`<br>Registering a pool creates the escrow for that same pool. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4735) |
| [SH-HL-10](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L162-L175) `hlShRequestCallbackIsDeliveredExactlyOnce`<br>A request callback is delivered exactly once. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4736) |
| [SH-HL-11](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L177-L190) `hlShRequestCallbackReachesTheRegisteredManager`<br>A request callback goes to the pool's registered request manager. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4737) |
| [SH-HL-12](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L192-L205) `hlShGrantThenRevokeLeavesTheLedgerUntouched`<br>An authorization granted then revoked leaves all authorizations as they were. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4738) |
| [SH-HL-13](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L207-L219) `hlShAddShareClassDeploysUnderTheMessagesPoolBrandedSalt`<br>A new share token uses the message's salt, which must encode the pool's id. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8022) |
| [SH-HL-14](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L221-L232) `hlShRequestCallbackDeliversThePool`<br>A request callback reaches the manager with the pool the message names. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8023) |
| [SH-HL-15](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L234-L245) `hlShRequestCallbackDeliversTheShareClass`<br>A request callback reaches the manager with the share class the message names. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8024) |
| [SH-HL-16](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L247-L258) `hlShRequestCallbackDeliversTheAsset`<br>A request callback reaches the manager with the asset the message names. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8025) |
| [SH-HL-17](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L260-L273) `hlShRequestCallbackDeliversThePayload`<br>A request callback reaches the manager with its payload unaltered. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8026) |
| [SH-HL-18](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L275-L286) `hlShAddShareClassBindsTheRegistrarTheMessageNames`<br>A new share class is bound to the named registrar and the token that registrar deployed. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8401) |
| [SH-HL-19](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L288-L304) `hlShFileMovesOnlyThePointerItNames`<br>Updating the registry or escrow factory address changes only that address. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8402) |
| [SH-HL-20](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L306-L319) `hlShUpdateRestrictionReachesTheClassTokenWithItsPayload`<br>A restriction update is passed unaltered to the registrar of the class's token. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8407) |
| [SH-HL-21](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L321-L337) `hlShLinkAndUnlinkNeverTouchATokenVaultPointer`<br>A link or unlink message never writes a share token's vault pointer. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9243) |
| [SH-HL-22](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/SpokeHandler_LinkedSpokeRegistry_entry_gate_properties.spec#L5-L56) `hlShUpdateShareMetadataWritesNoHandlerOrRegistryField`<br>A metadata update writes no field of the handler or the registry, under any key. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-9006) |
| [SH-HL-23](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_high_level.spec#L339-L350) `hlShSetPolicyAlwaysAdvancesTheNamespace`<br>Setting a policy, even the current one, installs it and voids outstanding authorizations. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6714) [🎯](./mutations/SpokeHandler/README.md#spokehandler-8400) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [SH-RV-01](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L3-L16) `rvShFileRefusesANonWardOrAnUnknownName`<br>Changing the handler's configuration fails for a non-admin or an unknown key. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4714) |
| [SH-RV-02](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L18-L32) `rvShAddPoolRefusesAnUntrustedCallerOrAPoolAlreadyOpen`<br>Adding a pool fails for an unauthorized caller or a pool already added. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4715) |
| [SH-RV-03](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L34-L54) `rvShAddShareClassRefusesAnUntrustedCallerANullRegistrarOrATakenSlot`<br>Adding a share class fails if unauthorized, with no registrar or pool, or a taken class or token. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4716) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6701) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6702) |
| [SH-RV-04](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L56-L71) `rvShUpdateManagerRefusesAnUntrustedCallerOrAPoolNeverOpened`<br>Changing pool manager rights fails for an unauthorized caller or a pool not yet added. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6729) |
| [SH-RV-05](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L73-L88) `rvShUpdateBridgerRefusesAnUntrustedCallerOrAPoolNeverOpened`<br>Changing share bridging rights fails for an unauthorized caller or a pool not yet added. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6729) |
| [SH-RV-06](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L90-L105) `rvShSetPolicyRefusesAnUntrustedCallerOrAPoolNeverOpened`<br>Setting a policy fails for an unauthorized caller or a pool not yet added. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6729) |
| [SH-RV-07](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L107-L122) `rvShAuthorizeRefusesAnUntrustedCallerOrAPoolWithNoRuleBook`<br>Authorizing a call fails for an unauthorized caller or a pool with no policy. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4719) |
| [SH-RV-08](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L124-L140) `rvShUnauthorizeRefusesAnUntrustedCallerOrAGrantThatDoesNotStand`<br>Revoking an authorization fails for an unauthorized caller or when none is outstanding. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6718) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6729) |
| [SH-RV-09](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L142-L159) `rvShUpdatePricePoolPerShareRefusesAnUntrustedCallerAnAbsentClassOrAnOlderStamp`<br>A share price update fails for an unauthorized caller, an unknown class or an older timestamp. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4718) |
| [SH-RV-10](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L161-L180) `rvShUpdatePricePoolPerAssetRefusesAnUntrustedCallerAnUnknownRowOrAnOlderStamp`<br>An asset price update fails if unauthorized, for an unknown class or asset, or an older timestamp. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4718) |
| [SH-RV-11](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L182-L197) `rvShSetRequestManagerRefusesAnUntrustedCallerOrAPoolNeverOpened`<br>Assigning a request manager fails for an unauthorized caller or a pool not yet added. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6729) |
| [SH-RV-12](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L199-L215) `rvShRequestCallbackRefusesAnUntrustedCallerOrAPoolWithNoRequestManager`<br>A request callback fails for an unauthorized caller or a pool without a deployed request manager. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6729) |
| [SH-RV-13](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L217-L233) `rvShUpdateRestrictionRefusesAnUntrustedCallerOrAClassTheSpokeDoesNotCarry`<br>A transfer restriction update fails for an unauthorized caller or a class not on this network. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4721) |
| [SH-RV-14](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L235-L254) `rvShExecuteTransferSharesRefusesAnAbsentClassOrAMalformedReceiver`<br>Receiving bridged shares fails for an unauthorized caller, unknown class or malformed receiver. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4722) |
| [SH-RV-15](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L256-L279) `rvShUpdateVaultLinkRefusesAnUnknownOrMismatchedVault`<br>Linking a vault fails for an unauthorized caller or an unregistered, linked or mismatched vault. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4723) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6704) |
| [SH-RV-16](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L281-L303) `rvShUpdateVaultUnlinkRefusesAnUnroutedOrMismatchedVault`<br>Unlinking a vault fails for an unauthorized caller or an unlinked or mismatched vault. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4740) |
| [SH-RV-17](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L305-L326) `rvShUpdateVaultDeployAndLinkRefusesAnUnknownAssetADeadClassOrATakenAddress`<br>Deploying a vault fails for an unauthorized caller, an unknown asset or class, or a taken address. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8954) |
| [SH-RV-18](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L328-L351) `rvShAddShareClassAcceptsACleanRegistration`<br>Adding a share class that passes every check always goes through. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4716) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6701) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6702) [🎯](./mutations/SpokeHandler/README.md#spokehandler-8021) |
| [SH-RV-19](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L353-L364) `rvShRelyRefusesANonWard`<br>Granting admin rights over the handler fails for a non-admin. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4725) |
| [SH-RV-20](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L366-L377) `rvShDenyRefusesANonWard`<br>Revoking admin rights over the handler fails for a non-admin. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4726) |
| [SH-RV-21](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L379-L395) `rvShUpdateManagerIsNeverRefusedOnAPoolTheHubOpened`<br>Changing pool manager rights never fails for an authorized caller on an added pool. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4717) |
| [SH-RV-22](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L397-L414) `rvShUnauthorizeIsNeverRefusedOnAGrantThatStands`<br>Revoking an outstanding authorization always goes through for an authorized caller. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4720) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6718) |
| [SH-RV-23](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L416-L432) `rvShRequestCallbackIsNeverRefusedOnAPoolThatHasARequestManager`<br>The handler never rejects an admin's callback to a pool's deployed request manager. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4724) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6725) |
| [SH-RV-24](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L434-L446) `rvShAddShareClassRefusesASaltOutsideItsPool`<br>Adding a share class fails unless its salt begins with the pool's id. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8020) |
| [SH-RV-25](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L448-L458) `rvShSharePriceStampedAheadOfTheClockIsRefused`<br>A share price update stamped later than the spoke's current time fails. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8638) |
| [SH-RV-26](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L460-L470) `rvShAssetPriceStampedAheadOfTheClockIsRefused`<br>An asset price update stamped later than the spoke's current time fails. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8639) |
| [SH-RV-27](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reverts.spec#L472-L494) `rvShUpdateVaultUnlinkIsNeverRefusedForALinkedVault`<br>Unlinking a vault from the pool, class and asset it is linked to never fails for an admin. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8928) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [SH-RC-01](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L3-L19) `rcShAddPoolOpensThePoolIsReachable`<br>A new pool can be registered on the spoke together with its escrow. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-9075) |
| [SH-RC-02](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L21-L38) `rcShUnlinkSwitchesOffALinkedVaultIsReachable`<br>A linked vault can be unlinked while keeping its registered asset. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6705) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6706) |
| [SH-RC-03](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L40-L62) `rcShTransferSharesForAnEmptyLegIsReachable`<br>A zero-amount incoming share transfer can succeed, changing no supply or balance. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6710) |
| [SH-RC-04](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L64-L84) `rcShTransferSharesAddressedToTheHandlerIsReachable`<br>Shares bridged in can be addressed to the handler itself and stay on it. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6711) |
| [SH-RC-05](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L86-L102) `rcShManagerSeatRoundTripIsReachable`<br>An account can be granted and then stripped of pool manager rights. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6712) |
| [SH-RC-06](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L104-L120) `rcShBridgerSeatRoundTripIsReachable`<br>An account can be granted and then stripped of share bridging rights. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6713) |
| [SH-RC-07](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L122-L138) `rcShPolicyRemovalIsReachable`<br>A pool's policy can be removed, which also invalidates its outstanding authorizations. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6715) |
| [SH-RC-08](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L140-L160) `rcShGrantsStackOnOneRowIsReachable`<br>The same call can hold two outstanding authorizations at once. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6716) [🎯](./mutations/SpokeHandler/README.md#spokehandler-6717) |
| [SH-RC-09](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L162-L178) `rcShSharePriceMovingToALaterStampIsReachable`<br>A share price can be replaced by a more recent one. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6719) |
| [SH-RC-10](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L180-L197) `rcShSharePriceCanBeMarkedToZeroIsReachable`<br>A non-zero share price can be replaced by a zero price. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6720) |
| [SH-RC-11](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L199-L216) `rcShAssetPriceMovingToALaterStampIsReachable`<br>A registered asset's price can be replaced by a more recent one. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6722) |
| [SH-RC-12](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L218-L231) `rcShRequestManagerAppointmentIsReachable`<br>A pool without a request manager can be assigned one. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6723) |
| [SH-RC-13](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L233-L249) `rcShRequestManagerWithdrawalIsReachable`<br>A pool's request manager can be removed by setting the zero address. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6724) |
| [SH-RC-14](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L251-L264) `rcShFileRepointsTheEscrowFactoryIsReachable`<br>An admin can repoint the handler to a different escrow factory. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6726) |
| [SH-RC-15](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L266-L279) `rcShFileRepointsTheSpokeRegistryIsReachable`<br>An admin can repoint the handler to a different spoke registry. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6727) |
| [SH-RC-16](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L281-L298) `rcShWardRoundTripIsReachable`<br>Admin rights over the handler can be granted and then revoked. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6728) |
| [SH-RC-17](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/SpokeHandler_LinkedSpokeRegistry_vault_pointer_properties.spec#L5-L27) `rcShLinkLeavesTheTokenVaultPointerUnset`<br>The hub can link an ERC-20 vault while the share token names no vault for that asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9241) |
| [SH-RC-18](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/SpokeHandler_LinkedSpokeRegistry_vault_pointer_properties.spec#L29-L51) `rcShUnlinkLeavesTheTokenPointingAtTheUnlinkedVault`<br>The hub can unlink an ERC-20 vault while the share token keeps naming it for that asset. | ✅ | [🎯](./mutations/SpokeRegistry/README.md#spokeregistry-9242) |
| [SH-RC-19](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L300-L314) `rcShSharePriceStampedAtTheLocalClockIsReachable`<br>A share price stamped at the spoke's current time can be recorded. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-6719) [🎯](./mutations/SpokeHandler/README.md#spokehandler-8640) |
| [SH-RC-20](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_reachability.spec#L316-L330) `rcShAssetPriceStampedAtTheLocalClockIsReachable`<br>A registered asset's price stamped at the spoke's current time can be recorded. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8641) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [SH-AC-01](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L3-L17) `acShWardRosterMovesOnlyForAWard`<br>Only a handler admin can grant or revoke admin rights on the spoke handler. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4713) |
| [SH-AC-02](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L19-L33) `acShRegistryPointerMovesOnlyForAWard`<br>Only a handler admin can point the handler at a different spoke registry. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4700) |
| [SH-AC-03](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L35-L50) `acShEscrowFactoryPointerMovesOnlyForAWard`<br>Only a handler admin can point the handler at a different escrow factory. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4700) |
| [SH-AC-04](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L52-L66) `acShPoolRowMovesOnlyForAWard`<br>Only a handler admin can register a pool on the spoke. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4701) |
| [SH-AC-05](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L68-L85) `acShShareClassBindingMovesOnlyForAWard`<br>Through the handler, only its admin can set a share class's token and registrar. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4702) |
| [SH-AC-06](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L87-L104) `acShTokenLookupMovesOnlyForAWard`<br>Through the handler, only its admin can change which class a share token belongs to. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4702) |
| [SH-AC-07](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L106-L120) `acShManagerSeatMovesOnlyForAWard`<br>Through the handler, only its admin can grant or revoke a pool manager role. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4703) |
| [SH-AC-08](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L122-L136) `acShBridgerSeatMovesOnlyForAWard`<br>Through the handler, only its admin can grant or revoke a pool's share bridging role. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4704) |
| [SH-AC-09](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L138-L155) `acShPolicyRowMovesOnlyForAWard`<br>Through the handler, only its admin can change a pool's policy. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4705) |
| [SH-AC-10](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L157-L171) `acShAuthorizationLedgerMovesOnlyForAWard`<br>Through the handler, only its admin can grant or revoke a pool authorization. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4706) |
| [SH-AC-11](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L173-L190) `acShSharePriceMovesOnlyForAWard`<br>Through the handler, only its admin can update a share class price or its timestamp. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4707) |
| [SH-AC-12](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L192-L209) `acShAssetPriceMovesOnlyForAWard`<br>Through the handler, only its admin can update an asset price or its timestamp. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4708) |
| [SH-AC-13](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L211-L238) `acShVaultRowMovesOnlyForAWard`<br>Only a handler admin can change a vault's registration or its linked status. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4709) |
| [SH-AC-14](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L240-L254) `acShRequestManagerPointerMovesOnlyForAWard`<br>Through the handler, only its admin can change a pool's request manager. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4710) |
| [SH-AC-15](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L256-L270) `acShShareSupplyMovesOnlyForAWard`<br>Through the handler, only its admin can change a share token's total supply. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4711) |
| [SH-AC-16](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L272-L286) `acShHolderBalanceMovesOnlyForAWard`<br>Through the handler, only its admin can change a holder's share balance. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4711) |
| [SH-AC-17](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/single/SpokeHandler_LinkedSpokeRegistry_access_control.spec#L288-L302) `acShRequestCallbackGoesOutOnlyForAWard`<br>Through the handler, only its admin can send the request manager a request callback. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-4712) |
| [SH-AC-18](./specs/core/spoke/SpokeHandler_LinkedSpokeRegistry/SpokeHandler_LinkedSpokeRegistry_entry_gate_properties.spec#L60-L72) `acShUpdateShareMetadataRefusesANonWard`<br>Changing a share token's name or symbol through the handler fails for a non-admin. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-9240) |

### SpokeHandler_LinkedRegistryFactory

- One symbolic pool
- One symbolic share class
- ERC20 tokens modeled in CVL (standard, up to 3 holders, uint128 balances, 6 to 18 decimals, no share token hooks, a holder pulling its own tokens only against an allowance to itself)
- A share token deployment through the registrar never refused, and a restriction update never refused for its hook

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [SH-ST-07](./specs/core/spoke/SpokeHandler_LinkedRegistryFactory/SpokeHandler_LinkedRegistryFactory_properties.spec#L5-L18) `stShPoolRowNeedsTheEscrowDeployBehindIt`<br>A pool is registered on the spoke only if its escrow can also be deployed. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-3207) |
| [SH-ST-08](./specs/core/spoke/SpokeHandler_LinkedRegistryFactory/SpokeHandler_LinkedRegistryFactory_properties.spec#L20-L35) `stShEscrowBrandComesWithThePoolRow`<br>An escrow is linked to a pool only in the call that registers that same pool. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8027) |
| [SH-ST-09](./specs/core/spoke/SpokeHandler_LinkedRegistryFactory/SpokeHandler_LinkedRegistryFactory_properties.spec#L37-L50) `stShPoolRowOpensOnlyAfterItsEscrowIsBranded`<br>A pool is registered on the spoke only after its escrow was linked to it in the same call. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-9260) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [SH-RC-21](./specs/core/spoke/SpokeHandler_LinkedRegistryFactory/SpokeHandler_LinkedRegistryFactory_properties.spec#L54-L67) `rcShAddPoolBrandsItsEscrowAndOpensTheRowIsReachable`<br>Adding a pool can create its escrow and register the pool in one message. | ✅ | [🎯](./mutations/SpokeHandler/README.md#spokehandler-8403) |

### EscrowFactory

- Wiring
  - The deployed escrow is not compiled: the ward calls the factory makes to it are recorded, its own response to them is not modelled

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [PEF-VT-01](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L7-L17) `vtPefEscrowAddressNeverMoves`<br>A pool's escrow address never changes, whatever call the factory receives. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-2704) |
| [PEF-VT-02](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L19-L33) `vtPefEscrowPoolRecordMovesOnlyOnADeployment`<br>An address's pool registration changes only when an escrow is deployed. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-8405) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [PEF-HL-01](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L37-L44) `hlPefEscrowAddressIsCallerIndependent`<br>A pool's escrow address does not depend on who asks. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-2703) |
| [PEF-HL-02](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L46-L53) `hlPefDeployedEscrowIsRecordedUnderItsPool`<br>A newly deployed escrow is registered to the pool it was deployed for. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-8030) |
| [PEF-HL-03](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L55-L65) `hlPefDeploymentBrandsOnlyTheEscrowItProduced`<br>Deploying an escrow changes the pool registration of no other address. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-8031) |
| [PEF-HL-04](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L67-L74) `hlPefFileInstallsTheSpokeItNames`<br>Setting the spoke stores the address that later escrows get as admin. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-8404) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [PEF-RV-01](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L78-L90) `rvPefFileRefusesANonWardOrAnUnknownName`<br>Updating the factory's configuration fails for a non-admin or an unknown parameter. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-8033) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [PEF-RC-01](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_wiring_properties.spec#L25-L34) `rcPefNewEscrowReachesTheHandOver`<br>An admin can deploy a pool escrow in three admin changes, the first and last made on it. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-9601) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [PEF-AC-01](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L94-L109) `acPefFactoryStateMovesOnlyForAWard`<br>Only an admin can change the factory's spoke or its admin rights. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-2705) |
| [PEF-AC-02](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L111-L121) `acPefMintingAnEscrowRefusesANonWard`<br>Only an admin can deploy a pool's escrow. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-2706) |
| [PEF-AC-03](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L123-L135) `acPefDeployedEscrowIsHandedToRootAndSpoke`<br>A new escrow has Root and the spoke as admins, but not the factory. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-8032) |
| [PEF-AC-04](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_properties.spec#L137-L149) `acPefWiringGrantsNobodyElse`<br>Deploying an escrow grants no admin rights beyond Root and the spoke. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-8406) |
| [PEF-AC-05](./specs/core/spoke/factories/EscrowFactory/EscrowFactory_wiring_properties.spec#L5-L21) `acPefNewEscrowIssuesExactlyTheHandOver`<br>A deployment adds Root, then the spoke, as escrow admins, then drops the factory, nothing more. | ✅ | [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-9600) [🎯](./mutations/PoolEscrowFactory/README.md#poolescrowfactory-9602) |

### MultiAdapter

- Loops running up to 8 iterations
- Outside the configurations named below
  - One symbolic pool, never the global pool 0
  - One symbolic share class
  - Adapters bounded to 8 symbolic adapters
  - One symbolic remote chain, the network every roster and vote row is read and written under
- Payload
  - One symbolic pool, never the global pool 0
  - One symbolic share class
  - Adapters bounded to 8 symbolic adapters
  - One symbolic remote chain, the network every roster and vote row is read and written under
- Refund
  - One symbolic pool, never the global pool 0
  - One symbolic share class
  - Adapters bounded to 8 symbolic adapters
  - One symbolic remote chain, the network every roster and vote row is read and written under
  - The refund address one compiled sink, accepting or refusing
  - No live adapter is the adapter contract or the refund address
- Unpinned
  - No pool, network or session pinned

#### Valid State

| Properties | Status | Mutations |
|---|---|---|
| [MA-VS-01](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L22-L25) `activeSetSessionCurrentOrCleared`<br>The active outbound adapter set is always the current session's, or none. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1802) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1814) |
| [MA-VS-02](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L27-L33) `clearedActiveSetEmpty`<br>An outbound adapter set without a session id is always empty. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1806) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1811) |
| [MA-VS-03](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L35-L47) `activeSetMirrorsSessionList`<br>The active outbound adapters always match the active session's adapter list. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1809) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1810) |
| [MA-VS-04](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L49-L55) `sessionsAboveCounterEmpty`<br>No adapters are ever configured for a future session id. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1802) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1803) |
| [MA-VS-05](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L57-L63) `blockedAboveCounterEmpty`<br>A future session id is never blocked. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1803) |
| [MA-VS-06](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L65-L68) `sessionZeroNeverConfigured`<br>Session zero never has adapters and is never blocked. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1806) |
| [MA-VS-07](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L70-L81) `registeredAdapterWellFormed`<br>An adapter's threshold is between one and its quorum, which is within the adapter cap. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1804) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1805) |
| [MA-VS-08](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L83-L92) `adapterQuorumEqualsListLength`<br>Every registered adapter's quorum equals its session's adapter count. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1805) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1817) |
| [MA-VS-09](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L94-L104) `sessionThresholdUniform`<br>All adapters of a session share one threshold. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1806) |
| [MA-VS-10](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L106-L116) `listedAdapterHoldsItsIndex`<br>Each adapter in a session votes in the slot matching its list position. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1807) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1808) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1812) |
| [MA-VS-11](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L118-L128) `adapterIdPointsAtItsOwnSlot`<br>A registered adapter's id always leads back to that adapter in the session list. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1806) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1807) |
| [MA-VS-12](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L130-L137) `blockedExcludesLive`<br>A blocked session never also has live adapters. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1812) [🎯](./mutations/MultiAdapter/README.md#multiadapter-8952) |
| [MA-VS-13](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L139-L150) `blockedThresholdWithinList`<br>A blocked session keeps a threshold between one and its adapter count. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1813) |
| [MA-VS-14](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L152-L159) `blockedAtCounterWasActive`<br>A blocked current session is always recorded as having been active. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1809) |
| [MA-VS-15](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L161-L171) `blockedCounterClearsActiveSet`<br>While the current session is blocked, the pool has no outbound adapters. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1809) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1810) [🎯](./mutations/MultiAdapter/README.md#multiadapter-1811) |
| [MA-VS-16](./specs/core/messaging/MultiAdapter/single/MultiAdapter_valid_state.spec#L173-L181) `blockedListDistinct`<br>A blocked session never lists the same adapter twice. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1816) |
| [MA-VS-17](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L21-L55) `futureSessionTallyEmpty`<br>No message stamped with a session id not yet issued has any votes recorded. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9620) |

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [MA-ST-01](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L74-L87) `stMaSessionAdvanceRestampsTheRoute`<br>A session id change immediately makes outbound messages use that session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1820) |
| [MA-ST-02](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L89-L116) `stMaBlockStashesTheLiveRosterWhole`<br>Blocking a session saves its adapters, threshold and whether it was active. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1822) |
| [MA-ST-03](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L118-L135) `stMaBlockingASupersededSessionKeepsTheRoute`<br>Blocking an old session leaves the active outbound adapters unchanged. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1826) |
| [MA-ST-04](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L137-L163) `stMaUnblockRestoresExactlyWhatWasStashed`<br>Unblocking a session restores exactly the adapters and threshold it had. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1818) |
| [MA-ST-05](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L165-L196) `stMaUnblockReopensTheRouteOnlyForTheSessionInService`<br>An unblocked session resumes sending with its stashed adapters only if still active. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-1819) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9426) |
| [MA-ST-06](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L198-L217) `stMaFileWritesExactlyTheDependencyItsKeyNames`<br>A configuration update sets only the named dependency, to the given address. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8524) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [MA-VT-01](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L5-L18) `vtMaSessionIdNeverRegresses`<br>A pool's session counter never decreases. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5418) |
| [MA-VT-02](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L20-L33) `vtMaSessionIdAdvancesByOneAtMost`<br>A single call advances a pool's session counter by at most one. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5418) |
| [MA-VT-03](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L35-L55) `vtMaRosterMoveSparesOtherSessions`<br>Changing one session's adapter list leaves other sessions' lists unchanged. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5419) |
| [MA-VT-04](./specs/core/messaging/MultiAdapter/single/MultiAdapter_transitions.spec#L57-L70) `vtMaSessionIdMovesOnlyByRotation`<br>A pool's session id changes only when a new adapter set is installed. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5423) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [MA-HL-01](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L3-L31) `hlMaExecuteSpendsATallyOnlyAtTheThreshold`<br>Executing a message consumes its votes only if enough adapters confirmed it. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-4901) |
| [MA-HL-02](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L33-L51) `hlMaExecuteSparesTheSecondLaneOutsideTheQuorum`<br>With a single adapter, execution leaves the second adapter's votes untouched. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8110) |
| [MA-HL-03](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L53-L71) `hlMaExecuteSparesTheThirdLaneOutsideTheQuorum`<br>With at most two adapters, execution leaves the third adapter's votes untouched. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-4900) |
| [MA-HL-04](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L73-L85) `hlMaExecuteForwardsExactlyOneMessage`<br>A successful execution passes exactly one message to the gateway. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5420) |
| [MA-HL-05](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L87-L97) `hlMaExecuteForwardsOnTheArrivalChain`<br>An executed message reaches the gateway tagged with the network it came from. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5421) |
| [MA-HL-06](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L99-L111) `hlMaHandleForwardsAtMostOnce`<br>One adapter delivery passes at most one message to the gateway. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5420) |
| [MA-HL-07](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L113-L125) `hlMaVoteNeverReachesTheGateway`<br>A vote alone never passes a message to the gateway. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5422) |
| [MA-HL-08](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L127-L141) `hlMaRotationKeepsEarlierSessionAdapterIds`<br>A new adapter set leaves existing sessions' adapter ids unchanged. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5424) |
| [MA-HL-09](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L143-L158) `hlMaRotationKeepsEarlierSessionQuorums`<br>A new adapter set leaves existing sessions' quorums unchanged. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5424) |
| [MA-HL-10](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L160-L175) `hlMaRotationKeepsEarlierSessionThresholds`<br>A new adapter set leaves existing sessions' thresholds unchanged. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5424) |
| [MA-HL-11](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L177-L193) `hlMaBlockThenUnblockKeepsTheThreshold`<br>Blocking then unblocking a session keeps its confirmation threshold unchanged. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5425) |
| [MA-HL-12](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L195-L209) `hlMaSendFansOutOncePerRosterMember`<br>One outbound send makes as many dispatches as the active set has adapters. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5426) [🎯](./mutations/MultiAdapter/README.md#multiadapter-7007) |
| [MA-HL-13](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L211-L221) `hlMaSendCarriesTheRequestedChain`<br>An outbound message is sent to the destination network the caller named. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5427) [🎯](./mutations/MultiAdapter/README.md#multiadapter-7007) |
| [MA-HL-14](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L223-L233) `hlMaSendCarriesTheRefundAddressItWasGiven`<br>An outbound message passes on the refund address the caller gave. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5428) [🎯](./mutations/MultiAdapter/README.md#multiadapter-7007) |
| [MA-HL-15](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L235-L251) `hlMaSendWrapsThePayloadWithTheLiveSessionId`<br>An outbound message is stamped with the active session id. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5429) |
| [MA-HL-16](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L253-L272) `hlMaSendLeavesTheLiveRouteWhereItStands`<br>Sending a message leaves the active adapter set unchanged. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5430) |
| [MA-HL-17](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L274-L286) `hlMaSendCarriesTheGasLimitItWasGiven`<br>Every adapter dispatch uses the gas limit the caller gave. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8700) |
| [MA-HL-18](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L288-L304) `hlMaSendReachesEachRosterMemberOnce`<br>One outbound send reaches each adapter of the active set exactly once. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7007) [🎯](./mutations/MultiAdapter/README.md#multiadapter-8701) |
| [MA-HL-19](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L306-L318) `hlMaSendFundsEachDispatchWithItsOwnQuote`<br>Each adapter dispatch is paid exactly the fee that adapter quoted. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8702) |
| [MA-HL-20](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L320-L332) `hlMaSendDispatchesTheStampedPayloadAtFullLength`<br>An outbound message is two bytes longer than its payload, so nothing is cut. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9077) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9078) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9079) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9087) |
| [MA-HL-21](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L59-L82) `hlMaExecuteForwardsOnlyOverAThresholdOfPositiveLanes`<br>An execution succeeds only if the message already had enough confirmations for the threshold. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9621) |
| [MA-HL-22](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L84-L96) `hlMaExecuteSpendsTheTallyBeforeForwarding`<br>An execution updates the message's votes before the gateway call, never after it. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9061) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9622) |
| [MA-HL-23](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L98-L110) `hlMaExecuteLowersTheTallyItSpends`<br>An execution lowers the first adapter's vote count on the message it releases. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9061) |
| [MA-HL-24](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L112-L124) `hlMaExecuteHandsTheGatewayTheUnstampedMessage`<br>An execution passes the gateway one message, the original without its session stamp. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9623) |
| [MA-HL-25](./specs/core/messaging/MultiAdapter/MultiAdapter_refund_properties.spec#L6-L28) `hlMaSendRefundsTheSurplusAndKeepsNoEth`<br>A send refunds any ETH its adapters were not paid, and pays any shortfall from its own balance. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9626) |
| [MA-HL-26](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L334-L345) `hlMaSendPaysEachAdapterTheQuoteItGaveItself`<br>Each adapter dispatch is paid the quote that same adapter gave just before it. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9627) |
| [MA-HL-27](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L126-L138) `hlMaHandleHandsTheGatewayTheUnstampedMessage`<br>A delivery that releases a message passes the gateway the original without its session stamp. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9623) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9625) |
| [MA-HL-28](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L5-L33) `hlMaVoteMovesOnlyTheTallyOfItsOwnPoolSessionAndMessage`<br>A vote moves no tally but the one keyed by its routed pool and whole stamped message. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9630) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9631) |
| [MA-HL-29](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L35-L63) `hlMaHandleMovesOnlyTheTallyOfItsOwnPoolSessionAndMessage`<br>A delivery moves no tally but the one keyed by its routed pool and whole stamped message. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9630) |
| [MA-HL-30](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L65-L93) `hlMaExecuteMovesOnlyTheTallyOfItsOwnPoolSessionAndMessage`<br>An execution spends no tally but the one keyed by its routed pool and whole stamped message. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9630) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9631) |
| [MA-HL-31](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L95-L105) `hlMaHandleIsJudgedByTheRoutedPoolsRosterAndManagers`<br>A delivery succeeds only for an adapter of the routed pool, sent by it or that pool's manager. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9632) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9633) |
| [MA-HL-32](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L107-L117) `hlMaVoteIsJudgedByTheRoutedPoolsRosterAndManagers`<br>A vote succeeds only for an adapter of the routed pool, cast by it or that pool's manager. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9632) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9633) |
| [MA-HL-33](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L119-L129) `hlMaExecuteIsJudgedByTheRoutedPoolsRosterAndManagers`<br>An execution succeeds only for an adapter of the routed pool, made by it or that pool's manager. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9632) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9633) |
| [MA-HL-34](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L131-L143) `hlMaVoteAsksTheRouterAboutTheMessagesOwnPoolsConfiguration`<br>On a vote, the router is told whether the message's pool is configured on the arrival network. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9634) |
| [MA-HL-35](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L145-L155) `hlMaVoteHandsTheRouterTheUnstampedMessage`<br>On a vote, both router questions are asked about the message without its session stamp. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9635) |
| [MA-HL-36](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L140-L155) `hlMaHandleSpendsTheTallyBeforeForwarding`<br>A delivery that releases a message updates its votes before the gateway call, never after. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9628) |
| [MA-HL-37](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L347-L361) `hlMaRotationBumpsTheSessionToTheNamedTarget`<br>A new adapter set moves the session id up by one, to the named id, which the route then uses. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9060) |
| [MA-HL-38](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L363-L382) `hlMaUnblockEmptiesTheStashIntoItsOwnSession`<br>A successful unblock empties the session's stash and puts the stashed roster back in that session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9428) |
| [MA-HL-39](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L157-L166) `hlMaVoteAsksTheFiledParserNotTheGasModel`<br>A vote takes the message's pool and routing from the configured parser. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8527) |
| [MA-HL-40](./specs/core/messaging/MultiAdapter/MultiAdapter_unpinned_properties.spec#L168-L191) `hlMaExecuteForwardsOnlyOverTheRoutedPoolsOwnTally`<br>An execution succeeds only if the routed pool's positive votes met its threshold. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8876) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9630) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9631) |
| [MA-HL-41](./specs/core/messaging/MultiAdapter/single/MultiAdapter_high_level.spec#L384-L399) `hlMaRotationCallsNoAdapter`<br>Setting, blocking or unblocking adapters asks no adapter for a quote or a send. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8877) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [MA-RV-01](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L5-L18) `rvMaFileRefusesANonWardOrAnUnknownName`<br>A configuration update fails for a non-admin caller or an unknown setting. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5403) |
| [MA-RV-02](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L20-L31) `rvMaUpdateManagerRefusesANonWard`<br>Only an admin can grant or revoke adapter manager rights for a pool. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5403) |
| [MA-RV-03](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L33-L47) `rvMaSetAdaptersRefusesWithoutTransportAuthority`<br>Only an admin or the pool's adapter manager can replace the pool's adapter set. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5404) |
| [MA-RV-04](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L49-L62) `rvMaSetAdaptersRefusesAnUnreachableThreshold`<br>An adapter set whose threshold exceeds its adapter count is rejected. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5407) |
| [MA-RV-05](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L64-L78) `rvMaSetAdaptersRefusesAZeroThresholdOverALiveRoster`<br>A non-empty adapter set with a zero threshold is rejected. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5408) |
| [MA-RV-06](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L80-L94) `rvMaSetAdaptersRefusesARepeatedAdapter`<br>An adapter set listing the same adapter twice is rejected. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5409) |
| [MA-RV-07](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L96-L109) `rvMaSetAdaptersRefusesAnUnexpectedSessionId`<br>Installing an adapter set fails unless it names the next session id. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5410) |
| [MA-RV-08](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L111-L129) `rvMaSetAdaptersOnAFreshRosterAlwaysGoesThroughForAWard`<br>An admin can always move a pool onto one new adapter at the next session id. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5411) |
| [MA-RV-09](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L131-L146) `rvMaBlockSessionRefusesAnOutsiderOrASessionWithNoRoster`<br>Blocking a session fails for an unauthorized caller or a session with no adapters. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5404) |
| [MA-RV-10](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L148-L162) `rvMaUnblockSessionRefusesWithoutTransportAuthority`<br>Only an admin or a pool's adapter manager can unblock the pool's sessions. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5404) |
| [MA-RV-11](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L164-L176) `rvMaUnblockSessionRefusesWithoutAStashedSession`<br>Unblocking fails for a session that is not blocked. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5412) |
| [MA-RV-12](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L178-L191) `rvMaUnblockSessionOfASingleStashAlwaysGoesThroughForAWard`<br>An admin can always unblock a blocked single-adapter session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5413) |
| [MA-RV-13](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L193-L206) `rvMaHandleRefusesAStrangerSubmission`<br>A delivery fails unless sent by the adapter or the adapter manager of the message's pool. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5405) |
| [MA-RV-14](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L208-L221) `rvMaVoteRefusesAStrangerSubmission`<br>A vote fails unless cast by the adapter or the adapter manager of the message's pool. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5405) |
| [MA-RV-15](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L223-L236) `rvMaExecuteRefusesAStrangerSubmission`<br>An execution fails unless made by the adapter or the adapter manager of the message's pool. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5405) |
| [MA-RV-16](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L238-L249) `rvMaExecuteRefusesAnUnregisteredAdapter`<br>Executing a message fails for an adapter not registered in any session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5406) |
| [MA-RV-17](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L251-L262) `rvMaHandleRefusesAnUnregisteredSender`<br>An address registered in no session cannot deliver a message in its own name. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5406) |
| [MA-RV-18](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L264-L275) `rvMaVoteRefusesAnUnregisteredSender`<br>An address registered in no session cannot cast a vote in its own name. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5406) |
| [MA-RV-19](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L277-L288) `rvMaExecuteRefusesAnUnregisteredSender`<br>An address registered in no session cannot execute a message in its own name. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5406) |
| [MA-RV-20](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L290-L301) `rvMaSendRefusesANonWard`<br>Only an admin can send an outbound message. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5403) |
| [MA-RV-21](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L303-L314) `rvMaSendRefusesWithoutALiveRoster`<br>A send with no active adapters fails instead of dropping the message. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5414) |
| [MA-RV-22](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L316-L327) `rvMaEstimateRefusesWithoutALiveRoster`<br>A cost estimate with no active adapters fails instead of quoting zero. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5415) |
| [MA-RV-23](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L329-L339) `rvMaQuorumAlwaysAnswers`<br>Reading a pool's adapter quorum never fails, even with no active adapters. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5416) |
| [MA-RV-24](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L341-L351) `rvMaThresholdAlwaysAnswers`<br>Reading a pool's adapter threshold never fails, even with no active adapters. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5416) |
| [MA-RV-25](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L353-L365) `rvMaNextActiveSessionIdRefusesARouteWhoseCounterIsSpent`<br>Looking up the next session id fails once the session counter is exhausted. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5417) |
| [MA-RV-26](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L367-L380) `rvMaHandleRefusesASessionTheAdapterIsNotIn`<br>A delivery fails if the adapter is not part of the message's session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8705) |
| [MA-RV-27](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L382-L394) `rvMaVoteRefusesASessionTheAdapterIsNotIn`<br>A vote fails if the adapter it names is not part of the message's session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9421) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9427) |
| [MA-RV-28](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L396-L409) `rvMaExecuteRefusesASessionTheAdapterIsNotIn`<br>An execution fails if the adapter it names is not part of the message's session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9422) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9427) |
| [MA-RV-29](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L411-L423) `rvMaSelfDeliveryHandleRefusesASessionTheSenderIsNotIn`<br>An adapter's own delivery fails if it is not part of the message's session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9423) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9427) |
| [MA-RV-30](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L425-L437) `rvMaSelfDeliveryVoteRefusesASessionTheSenderIsNotIn`<br>An adapter's own vote fails if it is not part of the message's session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9424) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9427) |
| [MA-RV-31](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L439-L451) `rvMaSelfDeliveryExecuteRefusesASessionTheSenderIsNotIn`<br>An adapter's own execution fails if it is not part of the message's session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9425) [🎯](./mutations/MultiAdapter/README.md#multiadapter-9427) |
| [MA-RV-32](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L453-L466) `rvMaFileAcceptsAWardUnderAKnownName`<br>Setting the gateway, parser or gas model never fails for an admin. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8526) |
| [MA-RV-33](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reverts.spec#L468-L486) `rvMaSendFailsWhenARosterAdapterRefuses`<br>A send and its quote both fail when one active adapter rejects them. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8878) [🎯](./mutations/MultiAdapter/README.md#multiadapter-8880) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [MA-RC-01](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L3-L21) `rcMaFirstRosterInstallIsReachable`<br>A pool's first adapter set can be a single adapter at threshold one. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7003) |
| [MA-RC-02](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L23-L39) `rcMaRosterCapInstallIsReachable`<br>An eight-adapter set, the maximum, can be installed requiring unanimity. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7000) |
| [MA-RC-03](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L41-L58) `rcMaIntermediateThresholdInstallIsReachable`<br>An adapter set can require two of its three adapters to agree. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7001) |
| [MA-RC-04](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L60-L79) `rcMaManagerHaltsARouteWithTheEmptySetIsReachable`<br>A pool's adapter manager can halt its outbound route with an empty adapter set. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7002) |
| [MA-RC-05](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L81-L97) `rcMaBlockingTheSessionInServiceIsReachable`<br>The active session can be blocked, stopping the pool's outbound messages. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7005) |
| [MA-RC-06](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L99-L122) `rcMaBlockThenUnblockRestoresTheRouteIsReachable`<br>A session can be blocked and unblocked, restoring its adapters and threshold. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7003) |
| [MA-RC-07](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L124-L140) `rcMaBlockingASupersededSessionKeepsTheRouteIsReachable`<br>An old session can be blocked while the active one keeps carrying messages. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7004) |
| [MA-RC-08](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L142-L159) `rcMaUnblockingASupersededSessionKeepsTheRouteIsReachable`<br>An old session can be unblocked without replacing the active adapter set. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7006) |
| [MA-RC-09](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L161-L192) `rcMaSendResumesAfterABlockAndUnblockIsReachable`<br>A blocked and unblocked session can send outbound messages again. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7014) |
| [MA-RC-10](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L194-L225) `rcMaSecondDeliveryCompletesTheQuorumIsReachable`<br>With two required adapters, the second delivery can release the message. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7015) |
| [MA-RC-11](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L227-L257) `rcMaExecuteReleasesThePayloadAfterTwoProofsIsReachable`<br>A message can be released by an execution after two adapters voted. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8706) |
| [MA-RC-12](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L259-L277) `rcMaThresholdOneLetsOneAdapterForwardAloneIsReachable`<br>With a threshold of one, a single adapter can release a message alone. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7008) |
| [MA-RC-13](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L279-L301) `rcMaExecuteDrivesASilentLaneNegativeIsReachable`<br>An execution can leave a vote count negative for an adapter that never voted. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7009) |
| [MA-RC-14](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L303-L327) `rcMaSupersededSessionStillDeliversIsReachable`<br>An in-flight message of a replaced adapter set can still be delivered. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7010) |
| [MA-RC-15](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L329-L344) `rcMaManagerSeatGrantAndRevokeIsReachable`<br>A pool's adapter manager can be appointed and removed again. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7011) |
| [MA-RC-16](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L346-L364) `rcMaFileRepointsBothWiringTargetsIsReachable`<br>Both the gateway and the message reader can be replaced. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7012) |
| [MA-RC-17](./specs/core/messaging/MultiAdapter/single/MultiAdapter_reachability.spec#L366-L381) `rcMaWardSeatGrantAndRevokeIsReachable`<br>Admin rights can be granted to a new account and revoked again. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-7013) |
| [MA-RC-18](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L159-L182) `rcMaSamePayloadIsReleasedTwiceIsReachable`<br>A message can be released twice when its only adapter delivers it again. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9636) |
| [MA-RC-19](./specs/core/messaging/MultiAdapter/MultiAdapter_refund_properties.spec#L32-L53) `rcMaRejectingRefundAddressUndoesTheWholeFanOutIsReachable`<br>A refund address that refuses the surplus can undo a send after its adapter was paid. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9637) |
| [MA-RC-20](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L184-L203) `rcMaAManagerReleasesAPayloadNoAdapterDeliveredIsReachable`<br>A pool manager can release a message in an adapter's name without any adapter delivering it. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-9638) |
| [MA-RC-21](./specs/core/messaging/MultiAdapter/MultiAdapter_payload_properties.spec#L205-L236) `rcMaManagerAloneReleasesATwoOfTwoMessageIsReachable`<br>The pool's adapter manager can release a two-of-two message no adapter confirmed. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8879) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [MA-AC-01](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L3-L17) `acMaOnlyWardMovesTheWardSet`<br>Only an admin can grant or revoke admin rights. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5400) |
| [MA-AC-02](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L19-L34) `acMaOnlyWardModifiesManagerRights`<br>Only an admin can appoint or remove a pool's adapter manager. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5400) |
| [MA-AC-03](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L36-L50) `acMaOnlyWardRepointsTheGateway`<br>Only an admin can change the gateway that receives confirmed messages. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5400) [🎯](./mutations/MultiAdapter/README.md#multiadapter-8525) |
| [MA-AC-04](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L52-L66) `acMaOnlyWardRepointsTheParser`<br>Only an admin can replace the reader that determines a message's pool. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5400) [🎯](./mutations/MultiAdapter/README.md#multiadapter-8525) |
| [MA-AC-05](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L68-L83) `acMaOnlyWardFansOutASend`<br>Only an admin can send outbound messages through the adapters. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5400) |
| [MA-AC-06](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L85-L101) `acMaOnlyWardOrManagerAdvancesTheSession`<br>Only an admin or the pool's adapter manager can start a new adapter session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5401) |
| [MA-AC-07](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L103-L126) `acMaOnlyWardOrManagerMovesTheLiveRoute`<br>Only an admin or the pool's adapter manager can change the outbound adapter set. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5401) |
| [MA-AC-08](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L128-L149) `acMaOnlyWardOrManagerMovesASessionRoster`<br>Only an admin or the pool's adapter manager can change a session's adapter list. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5401) |
| [MA-AC-09](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L151-L176) `acMaOnlyWardOrManagerMovesAnAdapterRegistration`<br>Only an admin or the pool's adapter manager can change an adapter's registration. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5401) |
| [MA-AC-10](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L178-L202) `acMaOnlyWardOrManagerMovesTheBlockedStash`<br>Only an admin or the pool's adapter manager can block or unblock a session. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5401) |
| [MA-AC-11](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L204-L238) `acMaOnlyAdapterOrManagerMovesAVoteTally`<br>Only a registered adapter or the pool's adapter manager can change the votes on a message. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-8109) |
| [MA-AC-12](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L240-L257) `acMaOnlyAdapterOrManagerReachesTheGateway`<br>Only a registered adapter or the pool's adapter manager can pass a message to the gateway. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5402) |
| [MA-AC-13](./specs/core/messaging/MultiAdapter/single/MultiAdapter_access_control.spec#L259-L273) `acMaOnlyWardRepointsTheMessageGas`<br>Only an admin can replace the gas model the adapters read. | ✅ | [🎯](./mutations/MultiAdapter/README.md#multiadapter-5400) [🎯](./mutations/MultiAdapter/README.md#multiadapter-8525) |

### Gateway

- Every configuration
  - Loops running up to 3 iterations
  - The pause, the message model and the adapter modeled in CVL: a free pause flag, per message tables keyed by the message hash, an adapter recording its quote and its send
  - The processor call answering a free success or failure, its own effects and gas metering dropped
  - A recovered token never refusing its transfer
  - The batch envelope withBatch off the parametric surface outside the withBatch and reentrant lanes
  - In the pool isolation witness, the message model answering a pool other than the one the message encodes at byte 1, for a pool bearing kind at least 9 bytes long
  - Outside the access control rules, a batch region carried as its length and a content token, its running gas and the locator list as plain values; the bytes a flush reads back are free beyond their length
  - A rule about an open batch starts its send inside one, from arbitrary batch state and before any flush, and holds two regions it compares at distinct storage keys
- Withbatch
  - The batcher callback one of four compiled shapes (lock, fail to lock, queue two messages, queue one then pause the protocol); any other callback runs nothing
  - The pause thrown by the callback standing for a guardian acting in the same transaction
- Reentrant
  - The adapter a compiled stand in that quotes a stored fee and, once armed, repays one underpaid batch from inside its own send
  - One re entry of the adapter send explored, deeper re entry pruned
  - The callback shapes of the withBatch lane

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [GW-ST-01](./specs/core/messaging/Gateway/single/Gateway_transitions.spec#L40-L60) `stGwTheProcessorAndAdapterAreReachedOnlyThroughTheirEntries`<br>Only handle and retry reach the processor, and only send and repay an adapter. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9832) |
| [GW-ST-02](./specs/core/messaging/Gateway/single/Gateway_transitions.spec#L62-L77) `stGwAnOpenBatchReachesTheAdapterOnlyThroughRepay`<br>Inside an open batch no call but a repay quotes or dispatches, leaving delivery to the flush. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9858) |
| [GW-ST-03](./specs/core/messaging/Gateway/Gateway_withbatch_properties.spec#L7-L23) `stGwFailureLedgerMovesOnlyThroughItsOwnEntriesAcrossBatches`<br>WithBatch included, a failure count rises only by handle, falls only by retry or clear. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9807) [🎯](./mutations/Gateway/README.md#gateway-9861) |
| [GW-ST-04](./specs/core/messaging/Gateway/Gateway_withbatch_properties.spec#L25-L39) `stGwNoCallLeavesABatchOpen`<br>WithBatch included, no call made with no batch open leaves a batch flag set. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9862) |
| [GW-ST-05](./specs/core/messaging/Gateway/single/Gateway_transitions.spec#L79-L98) `stGwBatchRegionsMoveOnlyUnderSend`<br>In an open batch only a send moves a batch, its gas total or the locator list. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9014) |
| [GW-ST-06](./specs/core/messaging/Gateway/single/Gateway_transitions.spec#L100-L112) `stGwAnOpenBatchBooksNoDebt`<br>Inside an open batch no call books a debt. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9015) |
| [GW-ST-07](./specs/core/messaging/Gateway/single/Gateway_transitions.spec#L114-L135) `stGwUnderpaidMovesOnlyThroughSendAndRepay`<br>A debt copy is booked only by a send and retired only by a repay, and only they move its gas. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9017) |

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [GW-VT-01](./specs/core/messaging/Gateway/single/Gateway_transitions.spec#L6-L22) `vtGwDependenciesMoveOnlyUnderFile`<br>The adapter, processor and gas model change only through a file call. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9016) |
| [GW-VT-02](./specs/core/messaging/Gateway/single/Gateway_transitions.spec#L24-L36) `vtGwOnlyTheTrafficEntriesConsultThePause`<br>Only handle, retry, send and repay check the pause, so it never blocks any other entry. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9806) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [GW-HL-01](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L6-L28) `hlGwUnbatchedSendSplitsTheValueExactly`<br>An unbatched send pays exactly the quote and refunds the rest, or books a debt and refunds all. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9808) |
| [GW-HL-02](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L30-L43) `hlGwBatchedSendTakesNoValue`<br>A send inside an open batch takes no value, refunds nothing and dispatches nothing. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9809) |
| [GW-HL-03](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L45-L62) `hlGwRepaySplitsTheValueExactly`<br>A repay forwards exactly the adapter's quote and refunds the rest. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9810) |
| [GW-HL-04](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L64-L77) `hlGwProcessorCallIsCappedBelowTheReserve`<br>Each inbound processor call gets its budget less the reserve, and the last carries no ether. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9811) |
| [GW-HL-05](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L79-L88) `hlGwEachProcessedSubMessageWasSourceCheckedFirst`<br>Up to three deep, each inbound message is source checked before it is priced and processed. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9064) |
| [GW-HL-06](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L90-L108) `hlGwPaidUnbatchedSendReachesTheAdapterInTheSameCall`<br>A paid unbatched send goes out at once on its own network, payload and gas, booking no debt. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9854) |
| [GW-HL-07](./specs/core/messaging/Gateway/Gateway_withbatch_properties.spec#L43-L70) `hlGwWithBatchWritesNoPersistentField`<br>A batch envelope writes no persistent gateway field, under any key. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9010) |
| [GW-HL-08](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L110-L145) `hlGwBatchedSendAppendsToItsOwnRegionOnly`<br>A batched send appends only to its own network and pool batch, registering it once when empty. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9859) |
| [GW-HL-09](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L147-L203) `hlGwTheUnpaidFlagIsInertInsideABatch`<br>Inside an open batch the unpaid flag changes nothing, and neither setting books a debt. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9860) |
| [GW-HL-10](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L205-L226) `hlGwFileWritesExactlyTheNamedDependency`<br>Repointing a dependency changes exactly the one it names and leaves the other two alone. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9830) |
| [GW-HL-11](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L228-L248) `hlGwRetryAndClearRetireExactlyOneOwnCopy`<br>A retry or a clear lowers its own failure count by one and touches no other. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9831) |
| [GW-HL-12](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L250-L262) `hlGwRetryForwardsItsOwnMessageOnce`<br>A retry passes the processor exactly the message it names, once, under the network it names. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9833) |
| [GW-HL-13](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L264-L279) `hlGwAFailedOneMessageBatchBooksOneCopyUnderItsKey`<br>A one-message batch books one failure under its own key exactly when its processing failed. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9834) |
| [GW-HL-14](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L281-L298) `hlGwRepayOfTheLastCopyDeletesItsRow`<br>Repaying the last unpaid copy of a batch deletes its debt record, gas included, and no other. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9835) |
| [GW-HL-15](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L300-L322) `hlGwAnUnpaidShortfallBooksOneCopyAndRefundsAll`<br>An unpaid send outside a batch books a short payment as one debt copy and refunds every wei. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9836) |
| [GW-HL-16](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L324-L338) `hlGwHandleBooksItsFailuresUnderItsSourceNetwork`<br>An inbound batch of up to three messages books exactly its failed calls, under its own network. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9850) |
| [GW-HL-17](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L340-L353) `hlGwFramingIsReadFromTheProcessorAndGasFromMessageGas`<br>Message details come only from the processor, gas limits only from the gas model. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-8528) [🎯](./mutations/Gateway/README.md#gateway-8529) [🎯](./mutations/Gateway/README.md#gateway-8530) |
| [GW-HL-18](./specs/core/messaging/Gateway/single/Gateway_high_level.spec#L355-L365) `hlGwHandlePricesEachMessageForItsOwnNetwork`<br>The gateway prices every inbound message for its own network. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-8883) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [GW-RV-01](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L6-L20) `rvGwFileRevertsIffGateFails`<br>Repointing a dependency fails exactly for a non-admin caller or an unknown name, paused or not. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9812) |
| [GW-RV-02](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L22-L34) `rvGwUpdateManagerRevertsIffCallerNotWard`<br>Setting a pool manager fails exactly when the caller is not an admin, paused or not. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9813) |
| [GW-RV-03](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L36-L47) `rvGwSendRefusedToANonWard`<br>An outbound send fails for a caller who is not an admin, whatever the batching state. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9814) |
| [GW-RV-04](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L49-L60) `rvGwHandleRequiresAWardOrThePoolManager`<br>An inbound batch fails unless the caller is an admin or a manager of the batch's pool. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9815) |
| [GW-RV-05](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L62-L73) `rvGwClearRequiresAWardOrThePoolManager`<br>Clearing a failed message fails unless the caller is an admin or a manager of its pool. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9816) |
| [GW-RV-06](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L75-L85) `rvGwPauseHaltsTheFourTrafficEntries`<br>While the protocol is paused, no message is handled, retried, sent or repaid. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9817) |
| [GW-RV-07](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L87-L122) `rvGwAdminEntriesStayLiveUnderPause`<br>A pause never stops an admin from changing admins, clearing a failure or recovering funds. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-8881) [🎯](./mutations/Gateway/README.md#gateway-9818) |
| [GW-RV-08](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L124-L135) `rvGwHandleRejectsLocallySourcedBatch`<br>A batch claiming to come from this very network is refused. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9819) |
| [GW-RV-09](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L137-L146) `rvGwRetryPropagatesAProcessorRevert`<br>A retry reverts when the processor refuses the message, keeping the failure on record. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9820) |
| [GW-RV-10](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L148-L159) `rvGwRetryRefusedWithoutRecordedFailure`<br>A message with no failure on record cannot be retried. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9821) |
| [GW-RV-11](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L161-L172) `rvGwClearRefusedWithoutRecordedFailure`<br>A message with no failure on record cannot be cleared, not even by an admin. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9822) |
| [GW-RV-12](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L174-L185) `rvGwSendRejectsEmptyOrOversizedMessage`<br>An empty message or one past the maximum message size cannot be sent. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9823) |
| [GW-RV-13](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L187-L199) `rvGwSendRefusesUnderpaymentOutsideUnpaidMode`<br>Outside a batch and outside unpaid mode, a send never succeeds on less than the adapter's quote. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9824) |
| [GW-RV-14](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L201-L211) `rvGwBatchedSendRefusesValue`<br>A send inside an open batch fails if it carries ether, since the flush pays for the batch. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9825) |
| [GW-RV-15](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L213-L231) `rvGwNothingJoinsABatchMidFlush`<br>While a batch is being sent out, no message joins it and no new batch opens. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9826) |
| [GW-RV-16](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L233-L244) `rvGwHandleRejectsAMessageFromTheWrongSource`<br>A batch carrying a message bound to another source network is refused whole. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9827) |
| [GW-RV-17](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L246-L256) `rvGwHandleRevertsOnAnOverrunningTail`<br>A batch whose framed message overruns the remaining bytes is refused whole. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9828) |
| [GW-RV-18](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L258-L268) `rvGwHandleRevertsWhenTheBudgetIsBelowTheReserve`<br>A message whose processing budget is below the failure reserve makes its batch fail. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9829) |
| [GW-RV-19](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L270-L282) `rvGwSendRejectsGasAboveTheDestinationCap`<br>A send outside a batch whose gas exceeds the destination's batch cap is refused. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9855) |
| [GW-RV-20](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L284-L299) `rvGwBatchCapIsCheckedOnTheRunningSum`<br>A batched send fails when its gas would push the batch's running total past the destination cap. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9856) |
| [GW-RV-21](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L301-L314) `rvGwClearRevertsIffOutsiderOrNoRecordedFailure`<br>A clear fails exactly with no failure on record or a caller neither admin nor manager of its pool. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-8881) [🎯](./mutations/Gateway/README.md#gateway-9816) [🎯](./mutations/Gateway/README.md#gateway-9818) [🎯](./mutations/Gateway/README.md#gateway-9822) [🎯](./mutations/Gateway/README.md#gateway-9895) |
| [GW-RV-22](./specs/core/messaging/Gateway/single/Gateway_reverts.spec#L316-L330) `rvGwRetryRevertsIffPausedValuedUnrecordedOrRefused`<br>A retry fails exactly when paused, sent ether, not recorded as failed, or the processor refuses it. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-8882) [🎯](./mutations/Gateway/README.md#gateway-9817) [🎯](./mutations/Gateway/README.md#gateway-9820) [🎯](./mutations/Gateway/README.md#gateway-9821) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [GW-RC-01](./specs/core/messaging/Gateway/single/Gateway_reachability.spec#L6-L21) `rcGwAPoolManagerInjectsAMessageNoQuorumApproved`<br>A pool manager who is not an admin can execute an inbound message with no adapter quorum. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9837) |
| [GW-RC-02](./specs/core/messaging/Gateway/single/Gateway_reachability.spec#L23-L51) `rcGwPermissionlessRetryPreemptsAClear`<br>An outsider can retry a freshly failed message before an admin clears it, so the clear fails. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9838) |
| [GW-RC-03](./specs/core/messaging/Gateway/single/Gateway_reachability.spec#L53-L68) `rcGwRepayIsUnguardedMidDispatch`<br>A repay can succeed and dispatch while a batch is being sent out. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9839) |
| [GW-RC-04](./specs/core/messaging/Gateway/single/Gateway_reachability.spec#L70-L89) `rcGwUnpaidShortfallIsBookedAndRefunded`<br>An unpaid send outside a batch can book its shortfall and refund every wei it carried. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9840) |
| [GW-RC-05](./specs/core/messaging/Gateway/single/Gateway_reachability.spec#L91-L103) `rcGwMultiMessageBatchIsDeliveredWhole`<br>An inbound batch of two different messages can be processed whole, each with its own call. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9852) |
| [GW-RC-06](./specs/core/messaging/Gateway/single/Gateway_reachability.spec#L105-L126) `rcGwUnderpaidGasLimitIsOverwrittenByTheLatestSend`<br>A second unpaid copy of a batch can replace the gas terms the first copy booked. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9853) |
| [GW-RC-07](./specs/core/messaging/Gateway/single/Gateway_reachability.spec#L128-L142) `rcGwBatchedSendDefersPaymentIsReachable`<br>A send inside an open batch can be queued with no payment and no adapter call. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9857) |
| [GW-RC-08](./specs/core/messaging/Gateway/Gateway_withbatch_properties.spec#L74-L92) `rcGwFlushDispatchesAfterAMidBatchPause`<br>A batch can still be sent out after the protocol is paused mid batch, in one transaction. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9863) |
| [GW-RC-09](./specs/core/messaging/Gateway/Gateway_reentrant_properties.spec#L5-L26) `rcGwRepayRunsInsideTheFlush`<br>A repay can retire a debt from inside a batch send, re-entered by the adapter being paid. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9864) |
| [GW-RC-10](./specs/core/messaging/Gateway/single/Gateway_reachability.spec#L144-L157) `rcGwHandlePricesAndForwardsAMessage`<br>An inbound batch can reach the processor, priced for this network. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-8884) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [GW-AC-01](./specs/core/messaging/Gateway/single/Gateway_access_control.spec#L3-L19) `acGwOnlyAWardOrThePoolsManagerFeedsTheProcessor`<br>Only an admin, a manager of the message's pool, or a retry can pass a message to the processor. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9800) |
| [GW-AC-02](./specs/core/messaging/Gateway/single/Gateway_access_control.spec#L21-L33) `acGwOnlyAWardMovesTheWardBit`<br>Only an admin can grant or revoke admin rights on the gateway. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9801) |
| [GW-AC-03](./specs/core/messaging/Gateway/single/Gateway_access_control.spec#L35-L47) `acGwOnlyAWardMovesAManagerSeat`<br>Only an admin can appoint or remove a pool manager. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9802) |
| [GW-AC-04](./specs/core/messaging/Gateway/single/Gateway_access_control.spec#L49-L66) `acGwOnlyAWardRepointsADependency`<br>Only an admin can repoint the adapter, the processor or the gas model. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9803) |
| [GW-AC-05](./specs/core/messaging/Gateway/single/Gateway_access_control.spec#L68-L83) `acGwOnlyAWardSends`<br>Outside a batch envelope, only an admin books a debt, and only an admin or a repay dispatches. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9804) |
| [GW-AC-06](./specs/core/messaging/Gateway/single/Gateway_access_control.spec#L85-L100) `acGwOnlyAWardOrThePoolsManagerClearsAFailedMessage`<br>A failed message is cleared only by an admin, a manager of its pool, or a retry. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9805) |

### MessageDispatcher

- Loops running up to 8 iterations
- The local hub and spoke handlers taken as recording what they are handed, their own effects dropped; the hub handler may refuse a request at any time
- The share class symbol cast and the 128 byte name padding replaced by hand checked transcriptions
- Native
  - A refund callee only accepts the ETH it is sent (optimistic fallback): it never calls back into the dispatcher nor forces ETH on it
  - No neighbour the dispatcher routes to, and no refund address, is the dispatcher itself
  - Only messages of at most 91 bytes are hashed, so no send is cut by the hashing bound
- Wire
  - A message longer than the Gateway's 1000 byte limit left out of the length claim
  - MessageLib's byte readers replaced by hand checked transcriptions

#### Variable Transitions

| Properties | Status | Mutations |
|---|---|---|
| [MD-VT-01](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_transitions.spec#L5-L16) `vtMdEachEntryReachesAtMostOneNeighbour`<br>Each dispatcher call reaches at most one destination, local or cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8707) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [MD-HL-01](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L5-L23) `hlMdNotifyPoolRoutesByDestination`<br>A pool notification for this network is handled locally, otherwise sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8708) |
| [MD-HL-02](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L25-L45) `hlMdUpdateAssetsRoutesByThePoolsHubChain`<br>The local hub gets asset updates for its own pools; others are sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8709) |
| [MD-HL-03](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L47-L68) `hlMdExecuteTransferSharesRoutesByTheTargetChain`<br>A share transfer to this network is executed locally, otherwise sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8710) |
| [MD-HL-04](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L70-L91) `hlMdLocalManagerUpdateReachesTheContractOfItsKind`<br>A local manager update reaches the spoke, adapter or gateway contract it is for. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8711) |
| [MD-HL-05](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L93-L112) `hlMdFileRepointsOnlyTheNamedSlot`<br>A configuration update sets only the routing address it names. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8712) |
| [MD-HL-06](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L114-L125) `hlMdEachSendReachesExactlyOneNeighbour`<br>Every completed send reaches exactly one destination, a local handler or the gateway. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9764) |
| [MD-HL-07](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L127-L150) `hlMdNotifyShareClassRoutesByDestination`<br>A share class announcement for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9741) |
| [MD-HL-08](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L152-L173) `hlMdNotifyShareMetadataRoutesByDestination`<br>A share metadata update for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9742) |
| [MD-HL-09](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L175-L198) `hlMdNotifyPricePoolPerShareRoutesByDestination`<br>A share price update for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9743) |
| [MD-HL-10](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L200-L222) `hlMdNotifyPricePoolPerAssetRoutesByTheAssetsChain`<br>An asset price update for a local asset is handled here, otherwise sent once to the asset's network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9744) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9771) |
| [MD-HL-11](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L224-L245) `hlMdUpdateRestrictionRoutesByDestination`<br>A transfer restriction update for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9745) |
| [MD-HL-12](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L247-L269) `hlMdManagerCallFromHubRoutesByDestination`<br>A hub manager call for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9746) |
| [MD-HL-13](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L271-L294) `hlMdUpdateVaultRoutesByTheAssetsChain`<br>A vault update for a local asset is handled here, otherwise sent once to the asset's network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9747) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9772) |
| [MD-HL-14](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L296-L317) `hlMdSetRequestManagerRoutesByDestination`<br>A request manager change for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9748) |
| [MD-HL-15](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L319-L340) `hlMdAuthorizeSpokeCallRoutesByDestination`<br>A spoke call authorization for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9749) |
| [MD-HL-16](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L342-L363) `hlMdUnauthorizeSpokeCallRoutesByDestination`<br>A spoke call revocation for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9750) |
| [MD-HL-17](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L365-L386) `hlMdSetPolicyRoutesByDestination`<br>A policy change for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9751) |
| [MD-HL-18](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L388-L409) `hlMdUpdateManagerRoutesByDestination`<br>A manager update for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9752) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9764) |
| [MD-HL-19](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L411-L431) `hlMdScheduleUpgradeRoutesByDestination`<br>An upgrade schedule for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9753) |
| [MD-HL-20](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L433-L453) `hlMdCancelUpgradeRoutesByDestination`<br>An upgrade cancellation for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9754) |
| [MD-HL-21](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L455-L478) `hlMdInitiateTransferSharesRoutesByThePoolsHubChain`<br>A share transfer start for a locally hubbed pool is handled here, else sent once to the pool's hub. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9755) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9773) |
| [MD-HL-22](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L480-L502) `hlMdUpdateHoldingAmountRoutesByThePoolsHubChain`<br>A holding amount report for a locally hubbed pool is handled here, else sent once to the pool's hub. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9757) |
| [MD-HL-23](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L504-L526) `hlMdUpdateSharesRoutesByThePoolsHubChain`<br>A share supply report for a locally hubbed pool is handled here, else sent once to the pool's hub. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9758) |
| [MD-HL-24](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L528-L549) `hlMdRegisterAssetRoutesByDestination`<br>An asset registration for this network is handled here, otherwise sent once to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9759) |
| [MD-HL-25](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L551-L573) `hlMdRequestRoutesByThePoolsHubChain`<br>A request for a locally hubbed pool is handled here, else sent once to the pool's hub. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9760) |
| [MD-HL-26](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L575-L596) `hlMdManagerCallFromSpokeRoutesByThePoolsHubChain`<br>A spoke manager call for a locally hubbed pool is handled here, else sent once to the pool's hub. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9761) |
| [MD-HL-27](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L598-L620) `hlMdRequestCallbackRoutesByTheAssetsChain`<br>A request callback for a local asset is handled here, otherwise sent once to the asset's network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9762) |
| [MD-HL-28](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L622-L638) `hlMdSetPoolAdaptersShipsPaidOverTheGatewayOnce`<br>An adapter set announcement is sent once, paid, via the gateway to its network, never handled here. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9763) |
| [MD-HL-29](./specs/core/messaging/MessageDispatcher/MessageDispatcher_native_properties.spec#L5-L25) `hlMdKeepsNoneOfTheValueItIsHanded`<br>A send keeps none of the ETH it is paid, so the dispatcher's balance never grows. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9765) |
| [MD-HL-30](./specs/core/messaging/MessageDispatcher/MessageDispatcher_native_properties.spec#L27-L82) `hlMdHoldingAmountShimActsAsUpdateAssetsForAnyPrice`<br>A holding amount report behaves exactly like an asset report, whatever price it carries. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9766) |
| [MD-HL-31](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L640-L657) `hlMdOnlyRequestsCallbacksAndForwardedTransfersTravelUnpaid`<br>Every message other than a request, a request callback or a share transfer travels paid. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9767) |
| [MD-HL-32](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L659-L678) `hlMdRequestAndCallbackCarryTheCallersPaymentMode`<br>A request or request callback sent cross-chain travels paid or unpaid as its caller chose. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9768) |
| [MD-HL-33](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L680-L696) `hlMdForwardedTransferRunsUnpaidExactlyWhenItArrivedFromElsewhere`<br>A share transfer to another network travels unpaid exactly when it started elsewhere. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9769) |
| [MD-HL-34](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L698-L713) `hlMdNotifyPoolSendsToTheNamedChain`<br>A pool announcement for another network is sent through the gateway to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9740) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9776) |
| [MD-HL-35](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L715-L731) `hlMdUpdateAssetsSendsToThePoolsHubChain`<br>An asset report for a pool hubbed elsewhere is sent through the gateway to that hub. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9757) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9774) |
| [MD-HL-36](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L733-L750) `hlMdExecuteTransferSharesSendsToTheTargetChain`<br>A share transfer to another network is sent through the gateway to that network. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9756) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9775) |
| [MD-HL-37](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_high_level.spec#L752-L771) `hlMdLocalAssetReportCarriesTheFlushedNet`<br>A local asset report reaches the hub handler with every field of the report intact. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9757) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9770) |
| [MD-HL-38](./specs/core/messaging/MessageDispatcher/MessageDispatcher_wire_properties.spec#L5-L18) `hlMdEveryWireMessageMeasuresItsOwnLength`<br>Every message within the gateway's size limit decodes to exactly its own length. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9777) |
| [MD-HL-39](./specs/core/messaging/MessageDispatcher/MessageDispatcher_wire_properties.spec#L20-L40) `hlMdEveryPoolMessageCarriesItsPoolAtOffsetOne`<br>Every message about a pool carries that pool's id right after its type byte. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9778) |
| [MD-HL-40](./specs/core/messaging/MessageDispatcher/MessageDispatcher_wire_properties.spec#L42-L63) `hlMdEveryWireMessageYieldsItsExtraGasLimit`<br>Every message yields the extra gas limit its send was given, or zero for a type without one. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9779) |
| [MD-HL-41](./specs/core/messaging/MessageDispatcher/MessageDispatcher_wire_properties.spec#L65-L78) `hlMdSetPoolAdaptersWiresItsTargetSession`<br>Setting a pool's adapters sends one message with its threshold and target session. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8886) |
| [MD-HL-42](./specs/core/messaging/MessageDispatcher/MessageDispatcher_wire_properties.spec#L80-L92) `hlMdEverySendCarriesItsFrozenTypeCode`<br>Every sent message starts with the type code of the function that sent it. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8887) |
| [MD-HL-43](./specs/core/messaging/MessageDispatcher/MessageDispatcher_wire_properties.spec#L94-L112) `hlMdTransferCarriesTheNamedReceiver`<br>A share transfer passes its receiver to the spoke handler if local, else in one message. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8888) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8889) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [MD-RV-01](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reverts.spec#L5-L19) `rvMdFileRefusesANonWardOrAnUnknownName`<br>Reconfiguring the dispatcher fails for a non-admin caller or an unknown setting name. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8713) |
| [MD-RV-02](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reverts.spec#L21-L33) `rvMdSetPoolAdaptersRefusesTheLocalNetwork`<br>A pool adapter update addressed to this network fails instead of being dropped. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-8714) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [MD-RC-01](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L5-L17) `rcMdNotifyPoolLocalLegIsReachable`<br>A pool announcement for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-02](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L19-L31) `rcMdNotifyPoolRemoteLegIsReachable`<br>A pool announcement for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-03](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L33-L47) `rcMdNotifyShareClassLocalLegIsReachable`<br>A share class announcement for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-04](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L49-L63) `rcMdNotifyShareClassRemoteLegIsReachable`<br>A share class announcement for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-05](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L65-L77) `rcMdNotifyShareMetadataLocalLegIsReachable`<br>A share metadata update for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-06](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L79-L91) `rcMdNotifyShareMetadataRemoteLegIsReachable`<br>A share metadata update for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-07](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L93-L107) `rcMdNotifyPricePoolPerShareLocalLegIsReachable`<br>A share price update for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-08](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L109-L123) `rcMdNotifyPricePoolPerShareRemoteLegIsReachable`<br>A share price update for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-09](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L125-L138) `rcMdNotifyPricePoolPerAssetLocalLegIsReachable`<br>An asset price update for a local asset can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-10](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L140-L153) `rcMdNotifyPricePoolPerAssetRemoteLegIsReachable`<br>An asset price update for an asset of another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-11](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L155-L168) `rcMdUpdateRestrictionLocalLegIsReachable`<br>A transfer restriction update for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-12](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L170-L182) `rcMdUpdateRestrictionRemoteLegIsReachable`<br>A transfer restriction update for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-13](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L184-L197) `rcMdManagerCallFromHubLocalLegIsReachable`<br>A hub manager call for this network can be handled by the local envoy. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9040) |
| [MD-RC-14](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L199-L212) `rcMdManagerCallFromHubRemoteLegIsReachable`<br>A hub manager call for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9040) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-15](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L214-L228) `rcMdUpdateVaultLocalLegIsReachable`<br>A vault update for a local asset can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-16](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L230-L244) `rcMdUpdateVaultRemoteLegIsReachable`<br>A vault update for an asset of another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-17](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L246-L258) `rcMdSetRequestManagerLocalLegIsReachable`<br>A request manager change for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-18](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L260-L272) `rcMdSetRequestManagerRemoteLegIsReachable`<br>A request manager change for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-19](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L274-L286) `rcMdAuthorizeSpokeCallLocalLegIsReachable`<br>A spoke call authorization for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-20](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L288-L300) `rcMdAuthorizeSpokeCallRemoteLegIsReachable`<br>A spoke call authorization for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-21](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L302-L314) `rcMdUnauthorizeSpokeCallLocalLegIsReachable`<br>A spoke call revocation for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-22](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L316-L328) `rcMdUnauthorizeSpokeCallRemoteLegIsReachable`<br>A spoke call revocation for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-23](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L330-L342) `rcMdSetPolicyLocalLegIsReachable`<br>A policy change for this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-24](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L344-L356) `rcMdSetPolicyRemoteLegIsReachable`<br>A policy change for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-25](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L358-L370) `rcMdUpdateManagerLocalLegIsReachable`<br>A manager update for this network can be handled by the local manager registry. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-26](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L372-L384) `rcMdUpdateManagerRemoteLegIsReachable`<br>A manager update for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-27](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L386-L397) `rcMdScheduleUpgradeLocalLegIsReachable`<br>An upgrade schedule for this network can be handled by the local root. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-28](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L399-L410) `rcMdScheduleUpgradeRemoteLegIsReachable`<br>An upgrade schedule for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-29](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L412-L423) `rcMdCancelUpgradeLocalLegIsReachable`<br>An upgrade cancellation for this network can be handled by the local root. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-30](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L425-L436) `rcMdCancelUpgradeRemoteLegIsReachable`<br>An upgrade cancellation for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-31](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L438-L452) `rcMdInitiateTransferSharesLocalLegIsReachable`<br>A share transfer start for a locally hubbed pool can be handled by the local hub handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9039) |
| [MD-RC-32](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L454-L468) `rcMdInitiateTransferSharesRemoteLegIsReachable`<br>A share transfer start for a pool hubbed elsewhere can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9039) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-33](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L470-L484) `rcMdExecuteTransferSharesLocalLegIsReachable`<br>A share transfer to this network can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-34](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L486-L500) `rcMdExecuteTransferSharesRemoteLegIsReachable`<br>A share transfer to another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-35](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L502-L515) `rcMdUpdateAssetsLocalLegIsReachable`<br>An asset report for a locally hubbed pool can be handled by the local hub handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-36](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L517-L530) `rcMdUpdateAssetsRemoteLegIsReachable`<br>An asset report for a pool hubbed elsewhere can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-37](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L532-L545) `rcMdUpdateHoldingAmountLocalLegIsReachable`<br>A holding amount report for a locally hubbed pool can be handled by the local hub handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-38](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L547-L560) `rcMdUpdateHoldingAmountRemoteLegIsReachable`<br>A holding amount report for a pool hubbed elsewhere can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-39](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L562-L575) `rcMdUpdateSharesLocalLegIsReachable`<br>A share supply report for a locally hubbed pool can be handled by the local hub handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-40](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L577-L590) `rcMdUpdateSharesRemoteLegIsReachable`<br>A share supply report for a pool hubbed elsewhere can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-41](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L592-L604) `rcMdRegisterAssetLocalLegIsReachable`<br>An asset registration for this network can be handled by the local hub handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-42](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L606-L618) `rcMdRegisterAssetRemoteLegIsReachable`<br>An asset registration for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-43](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L620-L632) `rcMdRequestLocalLegIsReachable`<br>A request for a locally hubbed pool can be handled by the local hub handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-44](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L634-L646) `rcMdRequestRemoteLegIsReachable`<br>A request for a pool hubbed elsewhere can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-45](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L648-L660) `rcMdManagerCallFromSpokeLocalLegIsReachable`<br>A spoke manager call for a locally hubbed pool can be handled by the local envoy. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-46](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L662-L674) `rcMdManagerCallFromSpokeRemoteLegIsReachable`<br>A spoke manager call for a pool hubbed elsewhere can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-47](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L676-L689) `rcMdRequestCallbackLocalLegIsReachable`<br>A request callback for a local asset can be handled by the local spoke handler. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9781) |
| [MD-RC-48](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L691-L704) `rcMdRequestCallbackRemoteLegIsReachable`<br>A request callback for an asset of another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |
| [MD-RC-49](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_reachability.spec#L706-L718) `rcMdSetPoolAdaptersRemoteLegIsReachable`<br>An adapter set announcement for another network can be sent cross-chain. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9780) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [MD-AC-01](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_access_control.spec#L3-L16) `acMdWardSetMovesOnlyForAWard`<br>Only an admin can grant or revoke admin rights on the message dispatcher. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-1904) |
| [MD-AC-02](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_access_control.spec#L18-L39) `acMdWiringMovesOnlyForAWard`<br>Only an admin can change the contracts the dispatcher routes messages through. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-1903) |
| [MD-AC-03](./specs/core/messaging/MessageDispatcher/single/MessageDispatcher_access_control.spec#L41-L54) `acMdNoUnwardedCallerReachesANeighbour`<br>Only an admin can make the dispatcher deliver or send a message. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-1901) [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-1902) |

### MessageProcessor

- Loops running up to 8 iterations
- The hub handler, spoke handler, envoy and upgrade schedule authority taken as recording what they are handed, their own effects dropped; the hub handler may refuse a request at any time
- The adapter router taken as recording the adapter set it is handed, its own checks and effects dropped
- The share class name and symbol conversions replaced by an arbitrary string no longer than the field, which no property reads
- MessageLib's byte readers replaced by hand checked transcriptions
- The wired hub handler, spoke handler, envoy, gateway, adapter router and schedule authority taken as deployed contracts, in the rules that a well formed message is never refused
- Joint
  - The real MessageDispatcher compiled beside the processor, a message carried between them by MessageLib's own encoder
- Refusing handler
  - The hub handler a compiled stub that refuses every request

#### State Transitions

| Properties | Status | Mutations |
|---|---|---|
| [MP-ST-01](./specs/core/messaging/MessageProcessor/single/MessageProcessor_transitions.spec#L5-L29) `stMpOwnStateMovesOnlyUnderItsAdminEntries`<br>The processor's admins move only under rely or deny, its routing only under file. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9911) |

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [MP-HL-01](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L5-L24) `hlMpEachHubKindReachesExactlyItsHandler`<br>A hub message makes exactly one call: to the envoy for a spoke manager call, else the hub handler. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9912) |
| [MP-HL-02](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L26-L43) `hlMpSourceNetworkReachesTheSourceKeyedHandlersUnchanged`<br>Transfer requests, holding and share reports and spoke manager calls pass on their source network. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9913) |
| [MP-HL-03](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L45-L70) `hlMpManagerUpdateReachesOnlyTheContractOfItsKind`<br>A manager update reaches exactly the one contract its manager kind selects. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9914) |
| [MP-HL-04](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L72-L85) `hlMpInboundTransferRunsWithNoRefundAndNoValue`<br>An inbound share transfer reaches the hub handler unpaid, refund-free, with its source network. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9915) |
| [MP-HL-05](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L87-L106) `hlMpEachSpokeKindReachesExactlyItsHandler`<br>A spoke message other than an adapter set makes one call, to the envoy only for a hub manager call. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9916) |
| [MP-HL-06](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L108-L122) `hlMpEachRootKindReachesExactlyTheScheduleAuthority`<br>An upgrade schedule or cancellation makes exactly one call, to the upgrade authority. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9917) |
| [MP-HL-07](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L124-L139) `hlMpPoolAdaptersUpdateReachesExactlyTheAdapterRouter`<br>An adapter set update makes exactly one call, to the adapter router, with its source network. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9918) |
| [MP-HL-08](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L141-L150) `hlMpFramedLengthIsTheDecodedLength`<br>A well-formed message's length is exactly the bytes its decoder reads. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8520) |
| [MP-HL-09](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L152-L176) `hlMpSourceNetworkOfEachKindIsItsHome`<br>Each message type is accepted only from its home network, three spoke reports from any. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8521) |
| [MP-HL-10](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L178-L198) `hlMpRouteFallsBackToTheGlobalSetOnlyForAFirstAdapterSet`<br>Messages route by their pool, a pool's first adapter set over the global adapters. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8522) |
| [MP-HL-11](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L200-L216) `hlMpPoolAdaptersUpdateForwardsTheAnnouncedSession`<br>An adapter set update passes its pool and target session to the adapter router. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8870) |
| [MP-HL-12](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L218-L224) `hlMpEveryFramedLengthIsPositive`<br>The processor never reports a zero message length. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8520) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8872) |
| [MP-HL-13](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L226-L246) `hlMpEachWireCodeReachesItsNamedEntry`<br>Each message type but a manager update calls only the handler function it names. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8873) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9912) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9916) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9917) |
| [MP-HL-14](./specs/core/messaging/MessageProcessor/single/MessageProcessor_high_level.spec#L248-L261) `hlMpInboundTransferExecutionHandsOnTheWireReceiver`<br>A share transfer execution passes the spoke handler the receiver it carries. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8874) |
| [MP-HL-15](./specs/core/messaging/MessageProcessor/MessageProcessor_joint_properties.spec#L28-L43) `hlMpRoundTripAdapterUpdateInstallsTheSenderSession`<br>The target session an adapter set update is encoded with reaches the adapter router. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8871) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [MP-RV-01](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L3-L15) `rvMpUnknownKindIsRefused`<br>A message whose type code names no message type is refused. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9900) |
| [MP-RV-02](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L17-L29) `rvMpOutOfRangeManagerKindIsRefused`<br>A manager update naming no manager kind is refused. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9901) |
| [MP-RV-03](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L31-L43) `rvMpOutOfRangeVaultKindIsRefused`<br>A vault update naming no vault update kind is refused. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9902) |
| [MP-RV-04](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L45-L56) `rvMpHandleRefusesANonWard`<br>A message from a non-admin is refused. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9903) |
| [MP-RV-05](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L58-L69) `rvMpEmptyMessageIsRefused`<br>An empty message is refused. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9904) |
| [MP-RV-06](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L71-L86) `rvMpWellFormedHubKindIsNeverRefused`<br>A well formed hub message from an admin goes through whenever the hub handler accepts it. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9905) |
| [MP-RV-07](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L88-L102) `rvMpWellFormedSpokeKindIsNeverRefused`<br>A well formed spoke message from an admin always goes through. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9906) |
| [MP-RV-08](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L104-L118) `rvMpWellFormedRootKindIsNeverRefused`<br>A well formed upgrade schedule or cancellation from an admin always goes through. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9907) |
| [MP-RV-09](./specs/core/messaging/MessageProcessor/MessageProcessor_refusing_handler_properties.spec#L5-L17) `rvMpRefusedRequestPropagatesToTheGateway`<br>A request the hub handler refuses makes the whole message fail. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9908) |
| [MP-RV-10](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reverts.spec#L120-L131) `rvMpMessageLengthRefusesAnUnknownKind`<br>Computing the length of a message fails when it is empty or its type is unknown. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8523) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8875) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [MP-RC-01](./specs/core/messaging/MessageProcessor/single/MessageProcessor_reachability.spec#L3-L16) `rcMpPoolAdaptersUpdateReachesTheAdapterRouter`<br>An adapter set update can complete on the adapter router. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9919) |
| [MP-RC-02](./specs/core/messaging/MessageProcessor/MessageProcessor_joint_properties.spec#L5-L24) `rcMpLocalAndRoundTripTransferInitiationHandDifferentRefunds`<br>A share transfer can reach the hub handler with a different refund address locally than by message. | ✅ | [🎯](./mutations/MessageDispatcher/README.md#messagedispatcher-9920) |

#### Access Control

| Properties | Status | Mutations |
|---|---|---|
| [MP-AC-01](./specs/core/messaging/MessageProcessor/single/MessageProcessor_access_control.spec#L3-L16) `acMpWardSetMovesOnlyForAWard`<br>Only an admin can grant or revoke admin rights over the processor. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9909) |
| [MP-AC-02](./specs/core/messaging/MessageProcessor/single/MessageProcessor_access_control.spec#L18-L39) `acMpWiringMovesOnlyForAWard`<br>Only an admin can rewire the contracts the processor routes messages to. | ✅ | [🎯](./mutations/MessageProcessor/README.md#messageprocessor-9910) |

### Gateway_LinkedMessageProcessor

- Loops running up to 3 iterations: a batch of more than 3 messages is not analysed
- MessageLib's byte readers replaced by hand checked transcriptions
- The protocol pause answered by a free flag
- Processing cut at the per message step: the capped processor call, the failure ledger and both events dropped, the budget underflow kept
- Only messages of at most 512 bytes are hashed
- Outside the configurations named below
  - The real MessageProcessor compiled as the message classifier (framing, pool, required source), the real GasService as the gas model with its figures read as free values; the processor's message handling is never reached
- Framing
  - The real MessageProcessor read for the framing alone, its pool and source readers and the gas limit answering free values
- Source
  - The real MessageProcessor read for the required source alone, its message length a free positive value, its pool reader and the gas limit answering free values
- Pool
  - The real MessageProcessor read for the framing and the pool, its source reader and the gas limit answering free values

#### High Level

| Properties | Status | Mutations |
|---|---|---|
| [GW-HL-19](./specs/core/messaging/Gateway_LinkedMessageProcessor/Gateway_LinkedMessageProcessor_pool_properties.spec#L6-L17) `hlGwHandleForwardsOnlyTheBatchPool`<br>Every message an accepted batch forwards belongs to the pool the batch was authorized for. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9886) |

#### Reverts

| Properties | Status | Mutations |
|---|---|---|
| [GW-RV-23](./specs/core/messaging/Gateway_LinkedMessageProcessor/Gateway_LinkedMessageProcessor_framing_properties.spec#L14-L39) `rvGwHandleRevertsOnAnUnknownTypeByte`<br>A batch fails when one of its first three messages has an unknown message type. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-8107) |
| [GW-RV-24](./specs/core/messaging/Gateway_LinkedMessageProcessor/Gateway_LinkedMessageProcessor_source_properties.spec#L6-L19) `rvGwHandleRefusesAnUpgradeFromANetworkOtherThanMainnet`<br>An upgrade message opening a batch fails unless sent from mainnet. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9888) |
| [GW-RV-25](./specs/core/messaging/Gateway_LinkedMessageProcessor/Gateway_LinkedMessageProcessor_source_properties.spec#L21-L34) `rvGwHandleRefusesAHubMessageOffItsPoolHome`<br>A hub-to-spoke message opening a batch fails unless sent from its pool's hub. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9888) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8531) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8532) |
| [GW-RV-26](./specs/core/messaging/Gateway_LinkedMessageProcessor/Gateway_LinkedMessageProcessor_source_properties.spec#L36-L50) `rvGwHandleRefusesAnAssetMessageOffItsHomeOrForAnIsoAsset`<br>An asset message opening a batch fails for an ISO-4217 asset or if sent off its home network. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9888) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8531) [🎯](./mutations/MessageProcessor/README.md#messageprocessor-8885) |

#### Reachability

| Properties | Status | Mutations |
|---|---|---|
| [GW-RC-11](./specs/core/messaging/Gateway_LinkedMessageProcessor/Gateway_LinkedMessageProcessor_properties.spec#L6-L19) `rcGwPoolManagerInjectsAHubOriginatedMessage`<br>A pool manager that is not an admin can deliver a hub message for its pool to processing. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9887) |
| [GW-RC-12](./specs/core/messaging/Gateway_LinkedMessageProcessor/Gateway_LinkedMessageProcessor_properties.spec#L21-L39) `rcGwUnknownTailRevertsTheBatchAfterItsFirstMessageWasForwarded`<br>A batch can fail on an unknown second message after its first went to processing. | ✅ | [🎯](./mutations/Gateway/README.md#gateway-9889) |

---

## Mutation Testing

A mutant is a deliberate fault planted in the source under [`mutations/`](./mutations); a rule catches it by flipping to Violated on the mutated source, and one mutant may be caught by several rules. `./certora/mutations/run_mutation.sh <band> <id>` (for example `Accounting 602`) plants it, runs the rule and restores the source. The script prints the verdict when the prover runs locally; on the cloud, `certoraRun` returns once the job is submitted, and the verdict is in the job's web report. Each entry names its fault and shows the diff; the Mutations column of the property tables links to it.

| Band | Mutants | Rules that catch them | Entries |
|---|---|---|---|
| Accounting | 66 | 78 | [`mutations/Accounting/README.md`](./mutations/Accounting/README.md) |
| Gateway | 74 | 72 | [`mutations/Gateway/README.md`](./mutations/Gateway/README.md) |
| Holdings | 125 | 120 | [`mutations/Holdings/README.md`](./mutations/Holdings/README.md) |
| Hub | 122 | 147 | [`mutations/Hub/README.md`](./mutations/Hub/README.md) |
| HubHandler | 93 | 108 | [`mutations/HubHandler/README.md`](./mutations/HubHandler/README.md) |
| HubRegistry | 112 | 112 | [`mutations/HubRegistry/README.md`](./mutations/HubRegistry/README.md) |
| MessageDispatcher | 61 | 99 | [`mutations/MessageDispatcher/README.md`](./mutations/MessageDispatcher/README.md) |
| MessageProcessor | 33 | 31 | [`mutations/MessageProcessor/README.md`](./mutations/MessageProcessor/README.md) |
| MultiAdapter | 117 | 135 | [`mutations/MultiAdapter/README.md`](./mutations/MultiAdapter/README.md) |
| PoolEscrow | 61 | 63 | [`mutations/PoolEscrow/README.md`](./mutations/PoolEscrow/README.md) |
| PoolEscrowFactory | 14 | 13 | [`mutations/PoolEscrowFactory/README.md`](./mutations/PoolEscrowFactory/README.md) |
| ShareClassManager | 104 | 95 | [`mutations/ShareClassManager/README.md`](./mutations/ShareClassManager/README.md) |
| SnapshotQueue | 69 | 77 | [`mutations/SnapshotQueue/README.md`](./mutations/SnapshotQueue/README.md) |
| Spoke | 114 | 139 | [`mutations/Spoke/README.md`](./mutations/Spoke/README.md) |
| SpokeHandler | 100 | 100 | [`mutations/SpokeHandler/README.md`](./mutations/SpokeHandler/README.md) |
| SpokeLinked | 35 | 38 | [`mutations/SpokeLinked/README.md`](./mutations/SpokeLinked/README.md) |
| SpokeRegistry | 145 | 174 | [`mutations/SpokeRegistry/README.md`](./mutations/SpokeRegistry/README.md) |

---

## Reproducing the Results

The Certora Prover runs either remotely on Certora's cloud or locally from a build. Both modes share
the setup below.

### Prerequisites

For Ubuntu 24.04; a step-by-step walkthrough is in this setup
[tutorial](https://alexzoid.com/first-steps-with-certora-fv-catching-a-real-bug#heading-setup).

1. Install Java (this report's runs used Java 19.0.2)

```bash
sudo apt update
sudo apt install default-jre
java -version
```

2. Install [pipx](https://pipx.pypa.io/), which keeps Python CLI tools in isolated environments

```bash
sudo apt install pipx
pipx ensurepath
```

3. Install the Certora CLI, pinned to the prover version of this report

```bash
pipx install certora-cli==8.19.2
```

4. Install solc-select and the compiler this project builds with

```bash
pipx install solc-select
solc-select install 0.8.28
solc-select use 0.8.28
```

5. Create the versioned symlink the confs name. They reference the compiler as `solc0.8.28` without
the dash, while solc-select installs a generic `solc`:

```bash
mkdir -p ~/.local/bin
ln -sf ~/.solc-select/artifacts/solc-0.8.28/solc-0.8.28 ~/.local/bin/solc0.8.28
```

Check that `~/.local/bin` is on your `PATH`, and add it if not:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

6. From the repository root, fetch the library sources the contracts import. They are git submodules, and `--recursive` reaches the nested ones the confs name by path:

```bash
git submodule update --init --recursive
```

Each conf asks the machine that runs it for a 40 GB Java heap (`-Xmx40g` in `java_args`) and up to 8 rules at a time (`max_concurrent_rules`), both set in `certora/confs/_base.conf`. On a smaller machine, lower them there, or pass `--max_concurrent_rules <n>` to `certoraRun`.

### Remote Execution

Set up a Certora key, available free through Certora's [Discord](https://discord.gg/certora) or their
website:

```bash
echo "export CERTORAKEY=<your_certora_api_key>" >> ~/.bashrc
```

> **Note:** a local prover, if installed, takes priority. To force the cloud, add `--server production`:
> ```bash
> certoraRun certora/confs/core/hub/Accounting/single_pool_valid_state.conf --server production
> ```

### Local Execution

Follow the build instructions in the
[CertoraProver repository (v8.19.2)](https://github.com/Certora/CertoraProver/tree/8.19.2). Once built,
the local prover takes priority over the cloud by default.

### Running the Suites

The flags below narrow a run to part of a conf, and each answers one question:

| Flag | What it does | When to reach for it |
|---|---|---|
| `--rule <name>` | runs one property instead of the conf | a property that did not decide, so it gets the budget rather than sharing it |
| `--exclude_rule <names>` | runs the conf without those properties | the rest of a conf whose one heavy property is being run separately |

#### [Accounting](./confs/core/hub/Accounting)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/hub/Accounting/metadata_properties.conf
certoraRun certora/confs/core/hub/Accounting/multi_pool_properties.conf
certoraRun certora/confs/core/hub/Accounting/single_pool_access_control.conf
certoraRun certora/confs/core/hub/Accounting/single_pool_high_level.conf
certoraRun certora/confs/core/hub/Accounting/single_pool_reachability.conf
certoraRun certora/confs/core/hub/Accounting/single_pool_reverts.conf
certoraRun certora/confs/core/hub/Accounting/single_pool_transitions.conf
certoraRun certora/confs/core/hub/Accounting/single_pool_valid_state.conf
```

#### [Holdings](./confs/core/hub/Holdings)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/hub/Holdings/census_properties.conf
certoraRun certora/confs/core/hub/Holdings/multi_network_properties.conf
certoraRun certora/confs/core/hub/Holdings/multi_pool_properties.conf
certoraRun certora/confs/core/hub/Holdings/multi_share_class_properties.conf
certoraRun certora/confs/core/hub/Holdings/single_pool_access_control.conf
certoraRun certora/confs/core/hub/Holdings/single_pool_high_level.conf
certoraRun certora/confs/core/hub/Holdings/single_pool_reachability.conf
certoraRun certora/confs/core/hub/Holdings/single_pool_reverts.conf
certoraRun certora/confs/core/hub/Holdings/single_pool_transitions.conf
certoraRun certora/confs/core/hub/Holdings/single_pool_valid_state.conf
```

#### [ShareClassManager](./confs/core/hub/ShareClassManager)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/hub/ShareClassManager/multi_pool_properties.conf
certoraRun certora/confs/core/hub/ShareClassManager/multi_share_class_properties.conf
certoraRun certora/confs/core/hub/ShareClassManager/single_pool_access_control.conf
certoraRun certora/confs/core/hub/ShareClassManager/single_pool_high_level.conf
certoraRun certora/confs/core/hub/ShareClassManager/single_pool_reachability.conf
certoraRun certora/confs/core/hub/ShareClassManager/single_pool_reverts.conf
certoraRun certora/confs/core/hub/ShareClassManager/single_pool_transitions.conf
certoraRun certora/confs/core/hub/ShareClassManager/single_pool_valid_state.conf
certoraRun certora/confs/core/hub/ShareClassManager/unpinned_properties.conf
```

#### [HubRegistry](./confs/core/hub/HubRegistry)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/hub/HubRegistry/metadata_properties.conf
certoraRun certora/confs/core/hub/HubRegistry/multi_pool_properties.conf
certoraRun certora/confs/core/hub/HubRegistry/single_pool_access_control.conf
certoraRun certora/confs/core/hub/HubRegistry/single_pool_high_level.conf
certoraRun certora/confs/core/hub/HubRegistry/single_pool_reachability.conf
certoraRun certora/confs/core/hub/HubRegistry/single_pool_reverts.conf
certoraRun certora/confs/core/hub/HubRegistry/single_pool_transitions.conf
certoraRun certora/confs/core/hub/HubRegistry/single_pool_valid_state.conf
```

#### [Hub_LinkedHubCore](./confs/core/hub/Hub_LinkedHubCore)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/hub/Hub_LinkedHubCore/access_control.conf
certoraRun certora/confs/core/hub/Hub_LinkedHubCore/high_level.conf
certoraRun certora/confs/core/hub/Hub_LinkedHubCore/metadata_properties.conf
certoraRun certora/confs/core/hub/Hub_LinkedHubCore/reachability.conf
certoraRun certora/confs/core/hub/Hub_LinkedHubCore/reverts.conf
certoraRun certora/confs/core/hub/Hub_LinkedHubCore/transitions.conf
certoraRun certora/confs/core/hub/Hub_LinkedHubCore/valid_state.conf
```

#### [HubHandler_LinkedHubCore](./confs/core/hub/HubHandler_LinkedHubCore)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/hub/HubHandler_LinkedHubCore/access_control.conf
certoraRun certora/confs/core/hub/HubHandler_LinkedHubCore/high_level.conf
certoraRun certora/confs/core/hub/HubHandler_LinkedHubCore/reachability.conf
certoraRun certora/confs/core/hub/HubHandler_LinkedHubCore/reverts.conf
certoraRun certora/confs/core/hub/HubHandler_LinkedHubCore/transitions.conf
certoraRun certora/confs/core/hub/HubHandler_LinkedHubCore/valid_state.conf
```

#### [Spoke](./confs/core/spoke/Spoke)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/spoke/Spoke/access_control.conf
certoraRun certora/confs/core/spoke/Spoke/high_level.conf
certoraRun certora/confs/core/spoke/Spoke/reachability.conf
certoraRun certora/confs/core/spoke/Spoke/reverts.conf
certoraRun certora/confs/core/spoke/Spoke/transitions.conf
```

#### [SpokeRegistry](./confs/core/spoke/SpokeRegistry)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/spoke/SpokeRegistry/access_control.conf
certoraRun certora/confs/core/spoke/SpokeRegistry/high_level.conf
certoraRun certora/confs/core/spoke/SpokeRegistry/reachability.conf
certoraRun certora/confs/core/spoke/SpokeRegistry/reverts.conf
certoraRun certora/confs/core/spoke/SpokeRegistry/transitions.conf
certoraRun certora/confs/core/spoke/SpokeRegistry/two_keys_properties.conf
certoraRun certora/confs/core/spoke/SpokeRegistry/valid_state.conf
```

#### [SnapshotQueue](./confs/core/spoke/SnapshotQueue)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/spoke/SnapshotQueue/all_assets_properties.conf
certoraRun certora/confs/core/spoke/SnapshotQueue/multi_pool_properties.conf
certoraRun certora/confs/core/spoke/SnapshotQueue/single_pool_access_control.conf
certoraRun certora/confs/core/spoke/SnapshotQueue/single_pool_high_level.conf
certoraRun certora/confs/core/spoke/SnapshotQueue/single_pool_reachability.conf
certoraRun certora/confs/core/spoke/SnapshotQueue/single_pool_reverts.conf
certoraRun certora/confs/core/spoke/SnapshotQueue/single_pool_transitions.conf
certoraRun certora/confs/core/spoke/SnapshotQueue/single_pool_valid_state.conf
certoraRun certora/confs/core/spoke/SnapshotQueue/unpinned_properties.conf
```

#### [Escrow](./confs/core/spoke/Escrow)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/spoke/Escrow/access_control.conf
certoraRun certora/confs/core/spoke/Escrow/custody_properties.conf
certoraRun certora/confs/core/spoke/Escrow/high_level.conf
certoraRun certora/confs/core/spoke/Escrow/multi_row_properties.conf
certoraRun certora/confs/core/spoke/Escrow/multi_share_class_properties.conf
certoraRun certora/confs/core/spoke/Escrow/reachability.conf
certoraRun certora/confs/core/spoke/Escrow/reverts.conf
certoraRun certora/confs/core/spoke/Escrow/transitions.conf
certoraRun certora/confs/core/spoke/Escrow/unpinned_properties.conf
certoraRun certora/confs/core/spoke/Escrow/valid_state.conf
```

#### [Spoke_LinkedSpokeCore](./confs/core/spoke/Spoke_LinkedSpokeCore)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/spoke/Spoke_LinkedSpokeCore/high_level.conf
certoraRun certora/confs/core/spoke/Spoke_LinkedSpokeCore/multi_pool_properties.conf
certoraRun certora/confs/core/spoke/Spoke_LinkedSpokeCore/reachability.conf
certoraRun certora/confs/core/spoke/Spoke_LinkedSpokeCore/rows_properties.conf
certoraRun certora/confs/core/spoke/Spoke_LinkedSpokeCore/transitions.conf
certoraRun certora/confs/core/spoke/Spoke_LinkedSpokeCore/valid_state.conf
```

#### [SpokeHandler_LinkedSpokeRegistry](./confs/core/spoke/SpokeHandler_LinkedSpokeRegistry)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/spoke/SpokeHandler_LinkedSpokeRegistry/access_control.conf
certoraRun certora/confs/core/spoke/SpokeHandler_LinkedSpokeRegistry/entry_gate_properties.conf
certoraRun certora/confs/core/spoke/SpokeHandler_LinkedSpokeRegistry/high_level.conf
certoraRun certora/confs/core/spoke/SpokeHandler_LinkedSpokeRegistry/reachability.conf
certoraRun certora/confs/core/spoke/SpokeHandler_LinkedSpokeRegistry/reverts.conf
certoraRun certora/confs/core/spoke/SpokeHandler_LinkedSpokeRegistry/transitions.conf
certoraRun certora/confs/core/spoke/SpokeHandler_LinkedSpokeRegistry/vault_pointer_properties.conf
```

#### [SpokeHandler_LinkedRegistryFactory](./confs/core/spoke/SpokeHandler_LinkedRegistryFactory)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/spoke/SpokeHandler_LinkedRegistryFactory/properties.conf
```

#### [EscrowFactory](./confs/core/spoke/factories/EscrowFactory)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/spoke/factories/EscrowFactory/properties.conf
certoraRun certora/confs/core/spoke/factories/EscrowFactory/wiring_properties.conf
```

#### [MultiAdapter](./confs/core/messaging/MultiAdapter)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/messaging/MultiAdapter/access_control.conf
certoraRun certora/confs/core/messaging/MultiAdapter/high_level.conf
certoraRun certora/confs/core/messaging/MultiAdapter/payload_properties.conf
certoraRun certora/confs/core/messaging/MultiAdapter/reachability.conf
certoraRun certora/confs/core/messaging/MultiAdapter/refund_properties.conf
certoraRun certora/confs/core/messaging/MultiAdapter/reverts.conf
certoraRun certora/confs/core/messaging/MultiAdapter/transitions.conf
certoraRun certora/confs/core/messaging/MultiAdapter/unpinned_properties.conf
certoraRun certora/confs/core/messaging/MultiAdapter/valid_state.conf
```

#### [Gateway](./confs/core/messaging/Gateway)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/messaging/Gateway/access_control.conf
certoraRun certora/confs/core/messaging/Gateway/high_level.conf
certoraRun certora/confs/core/messaging/Gateway/reachability.conf
certoraRun certora/confs/core/messaging/Gateway/reentrant_properties.conf
certoraRun certora/confs/core/messaging/Gateway/reverts.conf
certoraRun certora/confs/core/messaging/Gateway/transitions.conf
certoraRun certora/confs/core/messaging/Gateway/withbatch_properties.conf
```

#### [MessageDispatcher](./confs/core/messaging/MessageDispatcher)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/messaging/MessageDispatcher/access_control.conf
certoraRun certora/confs/core/messaging/MessageDispatcher/high_level.conf
certoraRun certora/confs/core/messaging/MessageDispatcher/native_properties.conf
certoraRun certora/confs/core/messaging/MessageDispatcher/reachability.conf
certoraRun certora/confs/core/messaging/MessageDispatcher/reverts.conf
certoraRun certora/confs/core/messaging/MessageDispatcher/transitions.conf
certoraRun certora/confs/core/messaging/MessageDispatcher/wire_properties.conf
```

#### [MessageProcessor](./confs/core/messaging/MessageProcessor)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/messaging/MessageProcessor/access_control.conf
certoraRun certora/confs/core/messaging/MessageProcessor/high_level.conf
certoraRun certora/confs/core/messaging/MessageProcessor/joint_properties.conf
certoraRun certora/confs/core/messaging/MessageProcessor/reachability.conf
certoraRun certora/confs/core/messaging/MessageProcessor/refusing_handler_properties.conf
certoraRun certora/confs/core/messaging/MessageProcessor/reverts.conf
certoraRun certora/confs/core/messaging/MessageProcessor/transitions.conf
```

#### [Gateway_LinkedMessageProcessor](./confs/core/messaging/Gateway_LinkedMessageProcessor)

Every rule in these decided, so each runs whole:

```bash
certoraRun certora/confs/core/messaging/Gateway_LinkedMessageProcessor/framing_properties.conf
certoraRun certora/confs/core/messaging/Gateway_LinkedMessageProcessor/pool_properties.conf
certoraRun certora/confs/core/messaging/Gateway_LinkedMessageProcessor/properties.conf
certoraRun certora/confs/core/messaging/Gateway_LinkedMessageProcessor/source_properties.conf
```

---

## Resources

- [Certora Tutorials](https://docs.certora.com/en/latest/docs/user-guide/tutorials.html): official Certora documentation and guided tutorials
- AlexZoid [personal site](https://alexzoid.com) and [FV resources](https://github.com/alexzoid-eth/fv-resources): write-ups on verifying real protocols, alongside a curated collection of specs, examples and references
- [Updraft Assembly & Formal Verification Course](https://updraft.cyfrin.io/courses/formal-verification): comprehensive video course covering assembly and formal verification from the ground up
- [Find Highs Using Certora Formal Verification](https://dacian.me/find-highs-before-external-auditors-using-certora-formal-verification): practical guide with a companion [repo](https://github.com/devdacian/solidity-fuzzing-comparison) of simplified examples based on real code and bugs from private audits
- [RareSkills Certora Book](https://rareskills.io/tutorials/certora-book): structured tutorial covering CVL syntax, patterns, and common pitfalls
