# Formal Verification Report: Accountable Credit Vaults

- Repository: https://github.com/Cyfrin/audit-2025-09-accountable
- Audit Commit Hash: [c163cec](https://github.com/Cyfrin/audit-2025-09-accountable/commit/c163cec2b80ef3071bc117b4216cd2e0adae8e13)
- Fixes Commit Hash: [979c0eb](https://github.com/Cyfrin/audit-2025-09-accountable/commit/979c0ebe4bd5860fe9b2e446f9fac2ae3919b39c)
- Date: October 2025
- Author: [@alexzoid](https://x.com/alexzoid) ([@cyfrin](https://x.com/cyfrin) private formal verification engagement) 
- Certora Prover version: 8.1.1

---

## Table of Contents

1. [About Accountable Credit Vaults](#about-accountable-credit-vaults)
2. [Formal Verification Approach](#formal-verification-approach)
   - [Assumptions](#assumptions)
3. [Formal Verification Methodology](#formal-verification-methodology)
   - [Types of Properties](#types-of-properties)
   - [Verification Process](#verification-process)
4. [Verification Properties](#verification-properties)
   - [Valid State](#valid-state)
   - [State Transitions](#state-transitions)
   - [High Level](#high-level)
   - [EIP20 Compliance](#eip20-compliance)
   - [EIP4626 Compliance](#eip4626-compliance)
   - [EIP7540 Compliance](#eip7540-compliance)
5. [Real Issues Properties](#real-issues-properties)
   - [[CRITICAL] Cancelling blocks withdrawal queue (Issue 9)](#issue-9)
   - [[MEDIUM] Bypass of transfer restrictions (Issue 4)](#issue-4)
   - [[MEDIUM] Instant fulfillRedeemRequest doesn't reserve liquidity (Issue 13)](#issue-13)
   - [[MEDIUM] Invalid maxWithdraw() check (Issue 33)](#issue-33)
   - [[LOW] Reserved assets extraction (Issue 27)](#issue-27)
   - [[LOW] Missing controller validation (Issue 24)](#issue-24)
   - [[INFO] Zero amount transfer rejection (Issue 25)](#issue-25)
6. [Setup and Execution Instructions](#setup-and-execution-instructions)
   - [Required source code modifications](#required-source-code-modifications)
   - [Verification Execution](#verification-execution)
   - [Running Verifications](#running-verifications)
   - [Advanced Options](#advanced-options)

## About Accountable Credit Vaults

Accountable Credit Vaults establishes a permissioned credit infrastructure facilitating direct capital allocation between liquidity providers and verified borrowers. Built on asynchronous vault architecture conforming to ERC7540 specifications, the protocol enables withdrawal queue management and flexible credit strategies through modular loan administration components.

## Formal Verification Approach

The verification environment tests both `FixedTerm` and `OpenTerm` strategy implementations. Global properties are validated across both configurations, while simpler properties are verified with `FixedTerm` only. Token interactions are simplified through CVL modeling. The verification suite systematically checks compliance with EIPs.

### Assumptions

Assumptions are constraints applied during verification to make the problem tractable for the prover. They are classified as **Safe** (no impact on security guarantees) or **Unsafe** (may limit coverage).

#### Safe Assumptions

These assumptions reflect real-world constraints or simplify non-critical aspects without compromising verification validity:

- ERC20 tokens implemented in CVL, limited to 5 users per contract for tractability
- Block timestamp bounded to realistic values, block number non-zero
- Message sender assumed distinct from contracts under verification

#### Unsafe Assumptions

These assumptions reduce verification scope to avoid prover timeouts but potentially may miss edge cases:

- Loop unrolling capped at 3 iterations, bitwise operations overapproximated 
- `reservedLiquidityBacked` proved only with `OpenTerm` due to [discussion](https://github.com/Cyfrin/audit-2025-09-accountable/issues/27#issuecomment-3381510866)
- `securityAdmin()`, `operationsAdmin()` simplified into CVL ghosts
- `isVerified()` and `areVerified()` simplified into CVL ghosts
- `OpenTerm._processAvailableWithdrawals()` removed from verification due to prover issues

## Formal Verification Methodology

Certora Formal Verification (FV) provides mathematical proofs of smart contract correctness by verifying code against a formal specification. It complements techniques like testing and fuzzing, which can only sometimes detect bugs based on predefined properties. In contrast, Certora FV examines all possible states and execution paths in a contract.

Simply put, the formal verification process involves crafting properties (similar to writing tests) in CVL language and submitting them alongside compiled Solidity smart contracts to a remote prover. This prover essentially transforms the contract bytecode and rules into a mathematical model and determines the validity of rules.

### Types of Properties

When constructing properties in formal verification, we mainly deal with two types: **Invariants** and **Rules**.

#### Invariants
- Conditions that MUST **always remain true** throughout the contract's lifecycle.
- Process:
  1. Define an initial condition for the contract's state.
  2. Execute an external function.
  3. Confirm the invariant still holds after execution.
- Example: "Requests outside queue bounds must be empty"
- Use Case: Ensures **Valid State** properties - critical state constraints that MUST never be violated.
- Feature: Proven invariants can be reused in other properties with the `requireInvariant` keyword.

#### Rules
- Flexible checks for specific behaviors or conditions.
- Structure:
  1. Setup: Set assumptions (e.g., "Requests outside queue bounds must be empty").
  2. Execution: Simulate contract behavior by calling external functions.
  3. Verification:
     - Use `assert` to check if a condition is **always true** (e.g., "Queue should decrease by exactly the processed amount").
     - Use `satisfy` to verify a condition is **reachable** (e.g., "Queue pointer can advance past the empty head (prevents deadlock)").
- Example: "Any call to processUpToShares(amount) should decrease the total shares in the queue by that amount"
- Use Case: Verifies a broad range of properties, from simple state changes to complex business logic.

### Verification Process
The process is divided into two stages: **Setup** and **Crafting Properties**.

#### Setup
This stage prepares the contract and prover for verification:
- Resolve external contract calls and dependencies.
- Simplify complex operations (e.g., math or bitwise calculations) for prover compatibility.
- Install storage hooks to monitor state changes.
- Address prover limitations (e.g., timeouts or incompatibilities).

#### Crafting Properties
This stage defines and implements the properties:
- Write properties in plain English for clarity.
- Categorize properties by purpose (e.g., Valid State, State Transition, EIP Compliance).
- Prove valid state invariants as a foundation for further rules

---

## Verification Properties

The verification properties are categorized into two distinct types:

1. **Valid State (VS)**: System-wide invariants that MUST always hold true. These properties define the fundamental constraints of the protocol, such as accounting consistency and structural integrity. Once proven, these invariants serve as trusted assumptions in other properties via `requireInvariant`, reducing verification complexity.

2. **State Transition (ST)**: Properties that verify the correctness of transitions between valid states. Building upon the valid state invariants, these properties ensure the protocol's state machine operates correctly and that state changes are both authorized and sequentially valid.

3. **High Level (HL)**: Business logic and economic properties that cover the whole system from the users' point of view. Main focus on behavior of specific functions.

4. **EIP Compliance (EIP)**: Properties that verify the protocol's adherence to Ethereum Improvement Proposal standards. 

Links to specific Certora Prover runs are provided for each property, with status indicators:
- ✅ Verified successfully
- ❌ Violated (indicates a potential issue)

### Valid State

Valid State properties define the fundamental invariants that must always hold true throughout the protocol's lifecycle. These properties are organized by contract and proven as invariants, meaning they are checked to hold after every possible function execution from any valid initial state.

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [VS-01](./specs/valid_state.spec#L54-L58) | `vaultCannotBeOwnerInOperator` | Vault cannot be an owner in operator relationships | ✅ |  |
| [VS-02](./specs/valid_state.spec#L60-L64) | `operatorOwnerNonZero` | Zero address cannot be an owner in operator relationships | ✅ |  |
| [VS-03](./specs/valid_state.spec#L66-L69) | `zeroAddressNoBalance` | Zero address cannot hold vault shares | ✅ |  |
| [VS-04](./specs/valid_state.spec#L71-L75) | `vaultNoAllowances` | Vault never approves anyone to spend any tokens | ✅ |  |
| [VS-05](./specs/valid_state.spec#L77-L85) | `vaultHoldsQueuedShares` | Vault's balance of its own shares must cover queued and redeemable shares | ✅ |  |
| [VS-06](./specs/valid_state.spec#L87-L90) | `totalAssetsBackedByBalance` | Total assets tracked must be backed by vault's actual balance | ✅ |  |
| [VS-07](./specs/valid_state.spec#L92-L95) | `reservedLiquidityBacked` | Reserved liquidity must not exceed total assets | [❌](https://prover.certora.com/output/52567/4fbec9433ca24d3999cbb10f3a16d213/?anonymousKey=cb5dca2ddcd5ad781c719f6aae4051f9846085ad) | Issue: [Reserved assets could be extracted from the Vault](#issue-27) |
| [VS-08](./specs/valid_state.spec#L99-L111) | `zeroControllerEmptyState` | Zero address must have empty state for all vault fields | [❌](https://prover.certora.com/output/52567/acc42433123e4b289c0f84e69fa52a44/?anonymousKey=e60b3d66b5574868073bfde4218b385aa2fe5f2a) | Issue: [Missing controller validation](#issue-24) |
| [VS-09](./specs/valid_state.spec#L113-L119) | `depositStateConsistency` | If maxMint is zero, depositAssets must also be zero | ✅ |  |
| [VS-10](./specs/valid_state.spec#L121-L126) | `mintPriceSetIfMintable` | If there are mintable shares, mint price must be set | ✅ |  |
| [VS-11](./specs/valid_state.spec#L128-L133) | `pendingRedeemImpliesQueueRequest` | Pending redeem shares must have valid request in queue | ✅ |  |
| [VS-12](./specs/valid_state.spec#L175-L178) | `pendingRedeemBacked` | Total pending redeem requests must equal totalQueuedShares | ✅ |  |
| [VS-13](./specs/valid_state.spec#L180-L183) | `queueOrdering` | nextRequestId must be within queue bounds or one position after | ✅ |  |
| [VS-14](./specs/valid_state.spec#L185-L188) | `queuePointersConsistency` | Queue pointers must be consistent (both zero or both non-zero) | ✅ |  |
| [VS-15](./specs/valid_state.spec#L190-L194) | `validRequestIds` | All non-zero request IDs must be within valid queue bounds | ✅ |  |
| [VS-16](./specs/valid_state.spec#L196-L202) | `requestIdControllerConsistency` | Controller's requestId must match request's controller field | ✅ |  |
| [VS-17](./specs/valid_state.spec#L204-L209) | `withdrawalRequestControllerConsistency` | Request's controller must map back to same requestId | ✅ |  |
| [VS-18](./specs/valid_state.spec#L211-L218) | `activeRequestsHaveShares` | Non-empty requests in queue must have non-zero shares | ✅ |  |
| [VS-19](./specs/valid_state.spec#L220-L225) | `uniqueControllerRequestId` | A controller can have at most one active request ID | ✅ |  |
| [VS-20](./specs/valid_state.spec#L227-L230) | `totalQueuedSharesMatchesRequests` | Total queued shares equals sum of all withdrawal requests | ✅ |  |
| [VS-21](./specs/valid_state.spec#L232-L262) | `controllerRequestIdConsistency` | Queue boundary requests must maintain bidirectional mapping | ✅ |  |
| [VS-22](./specs/valid_state.spec#L264-L274) | `sequentialRequestCounting` | Queue requests must be counted sequentially | ✅ |  |
| [VS-23](./specs/valid_state.spec#L276-L279) | `instantRequestEmpty` | Instant request ID must have empty state | ✅ |  |
| [VS-24](./specs/valid_state.spec#L281-L286) | `processedRequestsEmpty` | All processed requests must be completely empty | ✅ |  |
| [VS-25](./specs/valid_state.spec#L288-L292) | `futureRequestsEmpty` | Requests outside queue bounds must be empty | ✅ |  |
| [VS-26](./specs/valid_state.spec#L294-L299) | `pendingRedeemMatchesQueueShares` | Vault's pendingRedeemRequest must match queue shares | ✅ |  |
| [VS-27](./specs/valid_state.spec#L301-L304) | `totalMaxWithdrawNotExceedReserved` | Total maxWithdraw cannot exceed reserved liquidity | [❌](https://prover.certora.com/output/52567/af600209eef7404080e25e0ecc70589f/?anonymousKey=7b51592395a9f6ec802c89f89c0d789bb76540c9) | Issue: [Instant fulfillRedeemRequest doesn’t reserve liquidity](#issue-13) |

✅ All passed after fixes (local runs for `FixedTerm`: [1](reports/fixed_valid_state_48.html), [2](reports/fixed_valid_state_50.html), [3](reports/fixed_valid_state_51.html), [4](reports/fixed_valid_state_52.html), [5](reports/fixed_valid_state_53.html), [6](reports/fixed_valid_state_54.html); `OpenTerm`: [1](reports/open_valid_state_55.html), [2](reports/open_valid_state_56.html), [3](reports/open_valid_state_57.html), [4](reports/open_valid_state_58.html)).

### State Transitions

State Transition properties verify the correctness of transitions between valid states. These properties ensure that state changes occur only under the right conditions, such as calls to specific functions or time elapsing.

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [ST-01](./specs/state_transition.spec#L4-L30) | `shareTransferMustBeToVerifiedAddress` | Share transfers must only go to verified addresses or the vault itself | [❌](https://prover.certora.com/output/52567/b953632aecec42f8bddaf4ebb2a74471/?anonymousKey=37843116e90548773ab763348c7c7f059d4760fa) | Issue: [Bypass of transfer restrictions](#issue-4) |

✅ All [passed](https://prover.certora.com/output/52567/68959381f803443db5ccbd1fad0d1d56/?anonymousKey=1ea64a34ed56bea4c0eb5f7bc175de9a73537277) after fixes.

### High Level

These properties verify business logic and economic properties that cover the whole system from the users' point of view.

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [HL-01](./specs/high_level.spec#L4-L26) | `processUpToSharesDecreasesQueueShares` | Processing shares decreases queue by exact amount processed | ✅ |  |
| [HL-02](./specs/high_level.spec#L28-L54) | `processUpToRequestIdIncreasesNextId` | Processing requests advances nextRequestId appropriately | ✅ |  |
| [HL-03](./specs/high_level.spec#L56-L75) | `queueNotDeadlockOnEmptyEntry` | Queue can advance past empty entries to prevent deadlock | [❌](https://prover.certora.com/output/52567/57215c2865ee4ea99cef925f2426da37/?anonymousKey=abf3eabb5d6b0dbcf85266f08069455487c7653a) | Issue: [Cancelling blocks withdrawal queue](#issue-9) |

✅ All passed after fixes ([HL-01/02](https://prover.certora.com/output/52567/b7e478b4e6244861b798d727aba835a8/?anonymousKey=049f20fefa52220117d584107797b1ddca267dfb) and [HL-03](https://prover.certora.com/output/52567/c4a4056756af43a794d7c87bf36297a0/?anonymousKey=2c1a4745a7e5d42b8b84394d01ac7b78e8e43b49)).

### EIP20 Compliance

Prove contract is compatible with EIP20 (https://eips.ethereum.org/EIPS/eip-20). 

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [EIP20-01](./specs/eip20_compliance.spec#L5-L14) | `eip20_totalSupplyIntegrity` | totalSupply() returns correct total token supply | ✅ |  |
| [EIP20-02](./specs/eip20_compliance.spec#L16-L25) | `eip20_balanceOfIntegrity` | balanceOf() returns correct balance for any account | ✅ |  |
| [EIP20-03](./specs/eip20_compliance.spec#L27-L36) | `eip20_allowanceIntegrity` | allowance() returns correct spending allowance | ✅ |  |
| [EIP20-04](./specs/eip20_compliance.spec#L38-L75) | `eip20_transferIntegrity` | transfer() correctly updates balances and maintains invariants | ✅ |  |
| [EIP20-05](./specs/eip20_compliance.spec#L77-L98) | `eip20_transferMustRevert` | transfer() reverts on insufficient balance or invalid addresses | ✅ |  |
| [EIP20-06](./specs/eip20_compliance.spec#L100-L111) | `eip20_transferSupportZeroAmount` | Zero amount transfers must be treated as normal transfers | [❌](https://prover.certora.com/output/52567/9c9c3c73f4d64f9baf1284ced4f4a8f5/?anonymousKey=160f0b0d10e3f688f1981708e4aa3819e7023a80) | Issue: [ERC20 zero amount transfer rejection](#issue-25) |
| [EIP20-07](./specs/eip20_compliance.spec#L113-L160) | `eip20_transferFromIntegrity` | transferFrom() correctly updates balances and allowances | ✅ |  |
| [EIP20-08](./specs/eip20_compliance.spec#L162-L188) | `eip20_transferFromMustRevert` | transferFrom() reverts on insufficient balance/allowance | ✅ |  |
| [EIP20-09](./specs/eip20_compliance.spec#L190-L201) | `eip20_transferFromSupportZeroAmount` | Zero amount transferFrom must be treated as normal | [❌](https://prover.certora.com/output/52567/9c9c3c73f4d64f9baf1284ced4f4a8f5/?anonymousKey=160f0b0d10e3f688f1981708e4aa3819e7023a80) | Issue: [ERC20 zero amount transfer rejection](#issue-25) |
| [EIP20-10](./specs/eip20_compliance.spec#L203-L241) | `eip20_approveIntegrity` | approve() correctly sets allowances without affecting balances | ✅ |  |
| [EIP20-11](./specs/eip20_compliance.spec#L243-L260) | `eip20_approveMustRevert` | approve() reverts on zero address operations | ✅ |  |

✅ All [passed](https://prover.certora.com/output/52567/e0230dd1a8b44d4e8df7111b5e66741f/?anonymousKey=f87c2890f3a5cf744c08c20e7b05461c3eba735d) after fixes.

### EIP4626 Compliance

Prove contract is compatible with EIP4626 (https://eips.ethereum.org/EIPS/eip-4626).

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [EIP4626-01](./specs/eip4626_compliance.spec#L9-L13) | `eip4626_assetIntegrity` | Asset MUST be an EIP-20 token contract | ✅ |  |
| [EIP4626-02](./specs/eip4626_compliance.spec#L15-L20) | `eip4626_assetMustNotRevert` | asset() MUST NOT revert | ✅ |  |
| [EIP4626-03](./specs/eip4626_compliance.spec#L26-L29) | `eip4626_totalAssetsIntegrity` | Total assets MUST match tracked amount | ✅ |  |
| [EIP4626-04](./specs/eip4626_compliance.spec#L31-L36) | `eip4626_totalAssetsMustNotRevert` | totalAssets() MUST NOT revert | ✅ |  |
| [EIP4626-05](./specs/eip4626_compliance.spec#L42-L55) | `eip4626_convertToSharesMustNotDependOnCaller` | convertToShares MUST be caller-agnostic | ✅ |  |
| [EIP4626-06](./specs/eip4626_compliance.spec#L57-L66) | `eip4626_convertToSharesMustNotRevert` | convertToShares MUST NOT revert on reasonable input | ✅ |  |
| [EIP4626-07](./specs/eip4626_compliance.spec#L68-L78) | `eip4626_convertToSharesRoundDown` | convertToShares MUST round down | ✅ |  |
| [EIP4626-08](./specs/eip4626_compliance.spec#L84-L97) | `eip4626_convertToAssetsMustNotDependOnCaller` | convertToAssets MUST be caller-agnostic | ✅ |  |
| [EIP4626-09](./specs/eip4626_compliance.spec#L99-L108) | `eip4626_convertToAssetsMustNotRevert` | convertToAssets MUST NOT revert on reasonable input | ✅ |  |
| [EIP4626-10](./specs/eip4626_compliance.spec#L110-L122) | `eip4626_convertToAssetsRoundDown` | convertToAssets MUST round down | ✅ |  |
| [EIP4626-11](./specs/eip4626_compliance.spec#L128-L140) | `eip4626_maxDepositNoHigherThanActual` | maxDeposit MUST NOT exceed actual maximum | ✅ |  |
| [EIP4626-12](./specs/eip4626_compliance.spec#L142-L157) | `eip4626_maxDepositDoesNotDependOnUserBalance` | maxDeposit MUST NOT rely on user balance | ✅ |  |
| [EIP4626-13](./specs/eip4626_compliance.spec#L169-L180) | `eip4626_previewDepositNoMoreThanActualShares` | previewDeposit MUST NOT exceed actual shares | ✅ |  |
| [EIP4626-14](./specs/eip4626_compliance.spec#L182-L197) | `eip4626_previewDepositMustIgnoreLimits` | previewDeposit MUST ignore deposit limits | ✅ |  |
| [EIP4626-15](./specs/eip4626_compliance.spec#L199-L210) | `eip4626_previewDepositMustIncludeFees` | previewDeposit MUST include fees | ✅ |  |
| [EIP4626-16](./specs/eip4626_compliance.spec#L212-L225) | `eip4626_previewDepositMustNotDependOnCaller` | previewDeposit MUST be caller-agnostic | ✅ |  |
| [EIP4626-17](./specs/eip4626_compliance.spec#L227-L242) | `eip4626_previewDepositMayRevertOnlyWithDepositRevert` | previewDeposit MAY revert only if deposit would | ✅ |  |
| [EIP4626-18](./specs/eip4626_compliance.spec#L248-L269) | `eip4626_depositIntegrity` | deposit() MUST mint exact shares for assets | ✅ |  |
| [EIP4626-19](./specs/eip4626_compliance.spec#L271-L284) | `eip4626_depositRespectsApproveTransfer` | deposit() MUST respect ERC20 allowances | ✅ |  |
| [EIP4626-20](./specs/eip4626_compliance.spec#L286-L301) | `eip4626_depositMustRevertIfCannotDeposit` | deposit() MUST revert if cannot transfer all assets | ✅ |  |
| [EIP4626-21](./specs/eip4626_compliance.spec#L307-L319) | `eip4626_maxMintNoHigherThanActual` | maxMint MUST NOT exceed actual maximum | ✅ |  |
| [EIP4626-22](./specs/eip4626_compliance.spec#L321-L338) | `eip4626_maxMintDoesNotDependOnUserBalance` | maxMint MUST NOT rely on user balance | ✅ |  |
| [EIP4626-23](./specs/eip4626_compliance.spec#L340-L352) | `eip4626_maxMintZeroIfDisabled` | maxMint MUST return 0 if mints disabled | ✅ |  |
| [EIP4626-24](./specs/eip4626_compliance.spec#L364-L375) | `eip4626_previewMintNoFewerThanActualAssets` | previewMint MUST NOT underestimate assets needed | ✅ |  |
| [EIP4626-25](./specs/eip4626_compliance.spec#L377-L392) | `eip4626_previewMintMustIgnoreLimits` | previewMint MUST ignore mint limits | ✅ |  |
| [EIP4626-26](./specs/eip4626_compliance.spec#L394-L405) | `eip4626_previewMintMustIncludeFees` | previewMint MUST include fees | ✅ |  |
| [EIP4626-27](./specs/eip4626_compliance.spec#L407-L420) | `eip4626_previewMintMustNotDependOnCaller` | previewMint MUST be caller-agnostic | ✅ |  |
| [EIP4626-28](./specs/eip4626_compliance.spec#L422-L437) | `eip4626_previewMintMayRevertOnlyWithMintRevert` | previewMint MAY revert only if mint would | ✅ |  |
| [EIP4626-29](./specs/eip4626_compliance.spec#L443-L464) | `eip4626_mintIntegrity` | mint() MUST mint exact shares for assets | ✅ |  |
| [EIP4626-30](./specs/eip4626_compliance.spec#L466-L480) | `eip4626_mintRespectsApproveTransfer` | mint() MUST respect ERC20 allowances | ✅ |  |
| [EIP4626-31](./specs/eip4626_compliance.spec#L482-L497) | `eip4626_mintMustRevertIfCannotMint` | mint() MUST revert if cannot mint exact shares | ✅ |  |
| [EIP4626-32](./specs/eip4626_compliance.spec#L503-L517) | `eip4626_maxWithdrawNoHigherThanActual` | maxWithdraw MUST NOT exceed actual maximum | [❌](https://prover.certora.com/output/52567/ef88bd2d76b74cafb175f8d026e484b3/?anonymousKey=599db11fbc5df1632ff4006c69a03f836b23fa6c) | Issue: [Invalid maxWithdraw() check in withdraw()](#issue-33) |
| [EIP4626-33](./specs/eip4626_compliance.spec#L519-L533) | `eip4626_maxWithdrawZeroIfDisabled` | maxWithdraw MUST return 0 if withdrawals disabled | ✅ |  |
| [EIP4626-34](./specs/eip4626_compliance.spec#L535-L543) | `eip4626_maxWithdrawMustNotRevert` | maxWithdraw MUST NOT revert | ✅ |  |
| [EIP4626-35](./specs/eip4626_compliance.spec#L555-L588) | `eip4626_withdrawIntegrity` | withdraw() burns shares and sends exact assets | ✅ |  |
| [EIP4626-36](./specs/eip4626_compliance.spec#L590-L610) | `eip4626_withdrawMustRevertIfCannotWithdraw` | withdraw() MUST revert if cannot transfer assets | ✅ |  |
| [EIP4626-37](./specs/eip4626_compliance.spec#L616-L631) | `eip4626_maxRedeemNoHigherThanActual` | maxRedeem MUST NOT exceed actual maximum | ✅ |  |
| [EIP4626-38](./specs/eip4626_compliance.spec#L633-L647) | `eip4626_maxRedeemZeroIfDisabled` | maxRedeem MUST return 0 if redemption disabled | ✅ |  |
| [EIP4626-39](./specs/eip4626_compliance.spec#L649-L657) | `eip4626_maxRedeemMustNotRevert` | maxRedeem MUST NOT revert | ✅ |  |
| [EIP4626-40](./specs/eip4626_compliance.spec#L669-L702) | `eip4626_redeemIntegrity` | redeem() burns exact shares and sends assets | ✅ |  |

✅ All [passed](https://prover.certora.com/output/52567/b68458889b344943b862d02a710e6c75/?anonymousKey=a081c292dfaacbf72a613e80cdbb1d1507dd8203) after fixes.

### EIP7540 Compliance

Prove contract is compatible with EIP7540 (https://eips.ethereum.org/EIPS/eip-7540) - Asynchronous ERC-4626 Tokenized Vaults.

| Property | Name | Description | Status | Notes |
|----------|------|-------------|--------|-------|
| [EIP7540-01](./specs/eip7540_compliance.spec#L9-L13) | `eip7540_previewWithdrawMustRevert` | previewWithdraw MUST revert for all callers and inputs | ✅ |  |
| [EIP7540-02](./specs/eip7540_compliance.spec#L19-L23) | `eip7540_previewRedeemMustRevert` | previewRedeem MUST revert for all callers and inputs | ✅ |  |
| [EIP7540-03](./specs/eip7540_compliance.spec#L29-L43) | `eip7540_requestRedeemMustRemoveSharesFromOwner` | Shares MUST be removed from owner custody on requestRedeem | ✅ |  |
| [EIP7540-04](./specs/eip7540_compliance.spec#L45-L64) | `eip7540_requestRedeemMustRevertIfCannotRequest` | requestRedeem MUST revert if shares cannot be requested | ✅ |  |
| [EIP7540-05](./specs/eip7540_compliance.spec#L66-L76) | `eip7540_requestRedeemMustRespectOwnerOrOperator` | Owner MUST be msg.sender or have approved operator | ✅ |  |
| [EIP7540-06](./specs/eip7540_compliance.spec#L82-L92) | `eip7540_redeemControllerMustBeCallerOrOperator` | Controller MUST be msg.sender or have approved operator | ✅ |  |
| [EIP7540-07](./specs/eip7540_compliance.spec#L98-L107) | `eip7540_pendingRedeemRequestMustNotRevert` | pendingRedeemRequest MUST NOT revert on valid input | ✅ |  |
| [EIP7540-08](./specs/eip7540_compliance.spec#L111-L125) | `eip7540_pendingRedeemRequestIndependentOfCaller` | pendingRedeemRequest MUST NOT vary by caller | ✅ |  |
| [EIP7540-09](./specs/eip7540_compliance.spec#L134-L148) | `eip7540_claimableRedeemRequestIndependentOfCaller` | claimableRedeemRequest MUST NOT vary by caller | ✅ |  |
| [EIP7540-10](./specs/eip7540_compliance.spec#L150-L159) | `eip7540_claimableRedeemRequestMustNotRevert` | claimableRedeemRequest MUST NOT revert on valid input | ✅ |  |
| [EIP7540-11](./specs/eip7540_compliance.spec#L165-L174) | `eip7540_setOperatorMustReturnTrue` | setOperator MUST return True | ✅ |  |
| [EIP7540-12](./specs/eip7540_compliance.spec#L176-L187) | `eip7540_setOperatorMustSetStatus` | setOperator MUST set the operator status to approved value | ✅ |  |
| [EIP7540-13](./specs/eip7540_compliance.spec#L193-L203) | `eip7540_withdrawControllerMustBeCallerOrOperator` | Withdraw controller MUST be msg.sender or have approved operator | ✅ |  |
| [EIP7540-14](./specs/eip7540_compliance.spec#L211-L229) | `eip7540_requestIdZeroUsesControllerOnly` | When requestId==0, MUST use controller to discriminate state | ✅ |  |
| [EIP7540-15](./specs/eip7540_compliance.spec#L231-L246) | `eip7540_requestIdConsistentlyZero` | If any requestId is 0, all MUST be 0 | ✅ |  |
| [EIP7540-16](./specs/eip7540_compliance.spec#L248-L270) | `eip7540_requestMustNotSkipClaimableState` | Request MUST NOT skip the Claimable state | ✅ |  |

✅ All [passed](https://prover.certora.com/output/52567/1db575f26c5946a0b764616703ccd439/?anonymousKey=a43406f2678ff4927542044a19ecef1ef61980bf) before and after fixes.

## Real Issues Properties

This section documents vulnerabilities discovered during the manual audit (including issues by all participants) and formal verification process. Each issue demonstrates how formal properties detected the vulnerability and confirmed its resolution after applying the fix. 

<a id="issue-9"></a>
### [CRITICAL] Cancelling redeem requests permanently blocks the withdrawal queue ([#9](https://github.com/Cyfrin/audit-2025-09-accountable/issues/9))

`AccountableWithdrawalQueue` can deadlock at the head if the current head entry (`_queue.nextRequestId`) is fully removed (e.g., by a cancel that zeroes `shares` and clears `controller`) without advancing `nextRequestId`.

❌ Violated: https://prover.certora.com/output/52567/57215c2865ee4ea99cef925f2426da37/?anonymousKey=abf3eabb5d6b0dbcf85266f08069455487c7653a

```solidity
// processUpToShares can advance past an empty head entry, preventing queue deadlock
rule queueNotDeadlockOnEmptyEntry(env e) {

    setupValidState(e);

    mathint nextIdBefore = ghostQueueNextRequestId128;
    mathint lastIdBefore = ghostQueueLastRequestId128;

    require(nextIdBefore > 0 && nextIdBefore < lastIdBefore, 
       "Require at least 2 requests in queue (head + one more)");

    // Require that head request is empty (controller == address(0)), as happens after a cancel
    require(ghostQueueRequestsController[nextIdBefore] == 0, 
       "Require that head request is empty");

    processUpToShares(e, max_uint256);

    // Verify that the queue pointer can advance past the empty head (prevents deadlock)
    satisfy(ghostQueueNextRequestId128 > nextIdBefore);
}
```

✅ Passed with final fixes: https://prover.certora.com/output/52567/c4a4056756af43a794d7c87bf36297a0/?anonymousKey=2c1a4745a7e5d42b8b84394d01ac7b78e8e43b49

<a id="issue-4"></a>
### [MEDIUM] Complete bypass of transfer restrictions on vault share token is possible ([#4](https://github.com/Cyfrin/audit-2025-09-accountable/issues/4))

In `AccountableVault.sol` (which is inherited by the `AccountableAsyncRedeemVault`, we have certain transfer restrictions (KYC, if from address is subject to a throttle timestamp), applied in `_checkTransfer()` function.

❌ Violated (in `claimCancelRedeemRequest`): https://prover.certora.com/output/52567/b953632aecec42f8bddaf4ebb2a74471/?anonymousKey=37843116e90548773ab763348c7c7f059d4760fa

```solidity
// Any share transfer must go to a verified address or the vault itself
// This catches cases where internal _transfer bypasses the _checkTransfer validation
rule shareTransferMustBeToVerifiedAddress(env e, method f, address to)
    filtered { f -> !EXCLUDED_FUNCTION(f) }
{
    setupValidState(e);

    mathint balanceBefore = ghostERC20Balances128[_Vault][to];
    mathint totalSupplyBefore = ghostERC20TotalSupply256[_Vault];

    calldataarg args;
    f(e, args);

    mathint balanceAfter = ghostERC20Balances128[_Vault][to];
    mathint totalSupplyAfter = ghostERC20TotalSupply256[_Vault];

    // Check if address is verified after the transaction
    bool isVerified = ghostAllowed[to];

    // If this is a mint operation, total supply would increase
    bool isMintOperation = totalSupplyAfter > totalSupplyBefore;

    // If balance increased via transfer (not minting), recipient must be verified or the vault
    assert(balanceAfter > balanceBefore && !isMintOperation =>
           (isVerified || to == _Vault),
           "Share transfers must only go to verified addresses or the vault");
}
```

✅ Passed with final fixes: https://prover.certora.com/output/52567/68959381f803443db5ccbd1fad0d1d56/?anonymousKey=1ea64a34ed56bea4c0eb5f7bc175de9a73537277

<a id="issue-13"></a>
### [MEDIUM] Manual/Instant `fulfillRedeemRequest` doesn’t reserve liquidity ([#13](https://github.com/Cyfrin/audit-2025-09-accountable/issues/13))

Manual fulfillment paths (`fulfillRedeemRequest`) and the instant branch of `requestRedeem` mark shares as claimable without increasing `reservedLiquidity`.

❌ Violated: https://prover.certora.com/output/52567/af600209eef7404080e25e0ecc70589f/?anonymousKey=7b51592395a9f6ec802c89f89c0d789bb76540c9

```solidity
// Total maxWithdraw across all users cannot exceed reserved liquidity
invariant totalMaxWithdrawNotExceedReserved(env e)
    TOTAL_MAX_WITHDRAW_ASSETS() <= ghostReservedLiquidity256
```

✅ Verified after fixes: https://prover.certora.com/output/52567/00ba79bba8064665a1a2df62bfc3e74e/?anonymousKey=50770252acecca136ae5ace305573e483bde2eb1

<a id="issue-33"></a>
### [MEDIUM] Invalid `maxWithdraw()` check in `withdraw()` ([#33](https://github.com/Cyfrin/audit-2025-09-accountable/issues/33))

Vault incorrectly checks `maxWithdraw(receiver)` instead of `maxWithdraw(controller/owner)`.

❌ A property is violated: https://prover.certora.com/output/52567/ef88bd2d76b74cafb175f8d026e484b3/?anonymousKey=599db11fbc5df1632ff4006c69a03f836b23fa6c

```
// MUST NOT be higher than the actual maximum that would be accepted
rule eip4626_maxWithdrawNoHigherThanActual(env e, uint256 assets, address receiver, address owner) {

    setup(e);

    storage init = lastStorage;

    mathint limit = maxWithdraw(e, owner) at init;

    withdraw@withrevert(e, assets, receiver, owner) at init;
    bool reverted = lastReverted;

    // Withdrawals above the limit must revert
    assert(assets > limit => reverted, "Withdraw above limit MUST revert");
}
```

✅ Passed after fixes: https://prover.certora.com/output/52567/8e7cfdf612d64a4cb7e5d9d9d939968e/?anonymousKey=a961467ded443bd1cab3718ca882be71f38887e9

<a id="issue-27"></a>
### [LOW] Reserved assets could be extracted from the Vault ([#27](https://github.com/Cyfrin/audit-2025-09-accountable/issues/27))

Some strategy functions can release assets without checking if those assets are part of `reservedLiquidity`.

❌ Violated (in `OpenTerm`): https://prover.certora.com/output/52567/4fbec9433ca24d3999cbb10f3a16d213/?anonymousKey=cb5dca2ddcd5ad781c719f6aae4051f9846085ad

```solidity
// Reserved liquidity must not exceed total assets
invariant reservedLiquidityBacked(env e)
    ghostReservedLiquidity256 <= ghostTotalAssets256
```

✅ Verified after fixes (in `OpenTerm`): https://prover.certora.com/output/52567/6c23fc5e692e4f6b81a27bb662599293/?anonymousKey=154591958effceb15daabd64f146f81fcb361bd6

Excluded from the `FixedTerm` configuration due to [discussion](https://github.com/Cyfrin/audit-2025-09-accountable/issues/27#issuecomment-3381510866).

<a id="issue-24"></a>
### [LOW] Missing controller validation in `AccountableAsyncRedeemVault::requestRedeem` allows zero address state ([#24](https://github.com/Cyfrin/audit-2025-09-accountable/issues/24))

The `requestRedeem()` function fails to call `_checkController(controller)` validation, allowing the zero address to accumulate vault state. 

❌ A property is violated: https://prover.certora.com/output/52567/acc42433123e4b289c0f84e69fa52a44/?anonymousKey=e60b3d66b5574868073bfde4218b385aa2fe5f2a

```
// VS-08: Zero address must have empty state for all vault fields
invariant zeroControllerEmptyState(env e)
    ghostVaultStatesMaxMint256[0] == 0 &&
    ghostVaultStatesMaxWithdraw256[0] == 0 &&
    ghostVaultStatesDepositAssets256[0] == 0 &&
    ghostVaultStatesRedeemShares256[0] == 0 &&
    ghostVaultStatesDepositPrice256[0] == 0 &&
    ghostVaultStatesMintPrice256[0] == 0 &&
    ghostVaultStatesRedeemPrice256[0] == 0 &&
    ghostVaultStatesWithdrawPrice256[0] == 0 &&
    ghostVaultStatesPendingRedeemRequest256[0] == 0 &&
    ghostRequestIds128[0] == 0
filtered { f -> !EXCLUDED_FUNCTION(f) } { preserved with (env eFunc) { SETUP(e, eFunc); } }
```

✅ Passed after fixes: [reports/fixed_valid_state_48.html](reports/fixed_valid_state_48.html)

<a id="issue-25"></a>
### [INFO] ERC20 zero amount transfer rejection ([#25](https://github.com/Cyfrin/audit-2025-09-accountable/issues/25))

The `_checkTransfer` function reverts on zero-amount transfers, violating ERC-20 standard which mandates that transfers of 0 values [MUST be treated](https://eips.ethereum.org/EIPS/eip-20#transfer) as normal transfers.

❌ A property is violated: https://prover.certora.com/output/52567/9c9c3c73f4d64f9baf1284ced4f4a8f5/?anonymousKey=160f0b0d10e3f688f1981708e4aa3819e7023a80

```
// EIP20-06: Verify transfer() handles zero amount transfers correctly
// EIP-20: "Transfers of 0 values MUST be treated as normal transfers and fire the Transfer event."
rule eip20_transferSupportZeroAmount(env e, address to, uint256 amount) {

    setup(e);

    // Perform transfer
    transfer(e, to, amount);

    // Zero amount transfers must succeed
    satisfy(amount == 0);
}

// EIP20-09: Verify transferFrom() handles zero amount transfers correctly
// EIP-20: "Transfers of 0 values MUST be treated as normal transfers and fire the Transfer event."
rule eip20_transferFromSupportZeroAmount(env e, address from, address to, uint256 amount) {

    setup(e);

    // Perform the transferFrom
    transferFrom(e, from, to, amount);

    // Zero amount transferFrom must succeed
    satisfy(amount == 0);
}
```

✅ Passed after fixes: https://prover.certora.com/output/52567/e0230dd1a8b44d4e8df7111b5e66741f/?anonymousKey=f87c2890f3a5cf744c08c20e7b05461c3eba735d

## Setup and Execution Instructions

For step-by-step installation steps refer to this setup [tutorial](https://alexzoid.com/first-steps-with-certora-fv-catching-a-real-bug#heading-setup).  

### Required source code modifications

No modifications are required to the source code for Certora verification. The verification setup uses harness contracts that wrap the original contracts to enable formal verification without modifying production code.

### Verification Execution

#### Running Verifications

1. **Valid State Properties (Fixed-Term Strategy)**:

   ```bash
   # Run all valid state invariants for fixed-term strategy
   certoraRun certora/confs/fixed/fixed_valid_state.conf
   ```

2. **Valid State Properties (Open-Term Strategy)**:

   ```bash
   # Run all valid state invariants for open-term strategy
   certoraRun certora/confs/open/open_valid_state.conf
   ```

3. **State Transition Properties**:

   ```bash
   # Run state transition verification
   certoraRun certora/confs/state_transition.conf
   ```

4. **High-Level Business Logic**:

   ```bash
   # Run high-level properties verification
   certoraRun certora/confs/high_level.conf
   ```

5. **EIP Compliance Verification**:

   ```bash
   # Run EIP-20 compliance checks
   certoraRun certora/confs/eip20_compliance.conf

   # Run EIP-4626 tokenized vault compliance
   certoraRun certora/confs/eip4626_compliance.conf

   # Run EIP-7540 async vault compliance
   certoraRun certora/confs/eip7540_compliance.conf
   ```

#### Advanced Options

To optimize verification time or debug issues, you can run specific rules:

1. **Run with specific rule**:
   ```bash
   certoraRun certora/confs/fixed/fixed_valid_state.conf --rule vaultHoldsQueuedShares
   certoraRun certora/confs/high_level.conf --rule queueNotDeadlockOnEmptyEntry
   ```

2. **Run with specific external function**:
   ```bash
   certoraRun certora/confs/state_transition.conf --method "requestRedeem(uint256,address,address)"
   certoraRun certora/confs/eip4626_compliance.conf --method "deposit(uint256,address)"
   ```
