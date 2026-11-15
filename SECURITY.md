# Security Policy

## Reporting a vulnerability

**Do not open a public GitHub issue or pull request for security vulnerabilities.**

Instead, use GitHub's private vulnerability reporting:

1. Go to the [Security tab](https://github.com/zendyofficial/ajo/security) of this repository.
2. Click **Report a vulnerability** to open a private security advisory.

### What to include in your report

- **Affected component(s):** Specify whether the issue affects `contracts/ajo-circle`, `sdk` (`@ajo/sdk`), `frontend`, `cli` (`@ajo/cli`), or a combination of components.
- **Technical description & impact:** Explain the root cause, the preconditions required, and the worst-case impact on circle funds, authorization, or user keys.
- **Reproduction steps or Proof of Concept (PoC):** Provide a minimal Rust unit test (`contracts/ajo-circle/src/test.rs`), SDK/CLI script, or step-by-step reproduction against a local or testnet environment.
- **Suggested mitigation:** If applicable, describe how the issue can be remediated or mitigated during circle migration.

---

## Scope

### In scope

| Component | Path | Security-critical areas in scope |
|---|---|---|
| **Soroban Smart Contract** | `contracts/ajo-circle` | Custody of every circle's pot, `require_auth()` checks across `create_circle`, `join_circle`, `leave_circle`, `cancel_circle`, and `contribute`, payout index and rotation invariants in `disburse`, cycle deadline calculations, and state transition integrity (`Forming`, `Active`, `Completed`, `Cancelled`). |
| **TypeScript SDK** | `sdk/` (`@ajo/sdk`) | Non-custodial transaction construction, Soroban XDR assembly, simulation footprint handling, argument encoding for `contribute`/`disburse`, and contract event parsing (`discoverCircleIds`). |
| **Web Application** | `frontend/` | Client-side transaction building (`frontend/src/lib/contract.ts`), Freighter wallet signing handoff, recipient/amount display integrity, and protection against transaction-parameter tampering prior to user signature. |
| **Command-Line Client** | `cli/` (`@ajo/cli`) | Local secret key / keypair handling, environment and CLI flag parsing, non-custodial signing flows, and prevention of accidental secret leakage in logs or error output. |

### Out of scope

The following are explicitly **out of scope**:

- **Documented protocol trade-offs and social defection:** Uncollateralized member defection (e.g., a member skipping contributions, accumulating `missed_count` strikes, or still receiving the pot when `current_cycle` reaches their index), partial payouts after a missed deadline, or delayed `disburse` calls when no participant triggers disbursement at the deadline (see [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md)).
- **Arbitrary / malicious token contracts chosen by a circle creator:** `create_circle` accepts a caller-specified `token: Address`. Vulnerabilities, fee-on-transfer mechanics, or admin freezes inside third-party token contracts are outside Ajo's scope unless they expose a bug in `ajo-circle`'s handling of standard SEP-41 / Stellar Asset Contract tokens.
- **Third-party infrastructure and wallets:** Vulnerabilities or downtime in public Stellar RPC / Horizon nodes, the Freighter wallet extension itself, or user endpoint compromise.
- **Denial of Service (DoS) & automated noise:** Volumetric DoS against public testnet RPC endpoints, rate-limiting issues, UI/cosmetic bugs with no financial or cryptographic impact, and automated scanner output without a working proof of concept.

### Testnet vs. mainnet eligibility

Ajo currently runs on **Stellar public testnet** with test funds only. However, because `contracts/ajo-circle`, `@ajo/sdk`, `frontend`, and `@ajo/cli` are designed to run on mainnet without changes to their authorization or custody model:

- **Qualifying testnet findings:** Any reproducible vulnerability in contract logic, authorization, transaction building, or key handling that would put real funds or signatures at risk on mainnet **does qualify** as an in-scope security finding.
- **Non-qualifying testnet-only issues:** Operational issues limited strictly to testnet infrastructure (such as Friendbot faucet availability, testnet reset state, or public testnet RPC rate limits) do **not** qualify.

---

## Severity rubric

| Severity | Criteria | Examples in Ajo |
|---|---|---|
| **Critical** | Direct, unauthorized loss or permanent freezing of funds in an `Active` circle, or compromise of user private keys. | Bypassing `require_auth()` to drain or redirect a circle's pot; withdrawing funds outside `disburse`'s rotation; permanently locking collected contributions inside `contracts/ajo-circle` so even post-deadline `disburse` cannot release them; exfiltrating secret keys in `@ajo/cli` or `frontend`. |
| **High** | Manipulation of transaction construction or circle state that causes users to sign unintended transfers or corrupts payout accounting. | `@ajo/sdk` or `frontend` constructing a transaction with an altered `contribution_amount`, wrong contract address, or spoofed parameters; allowing a non-member to alter `Circle.members` or reset `current_cycle`; bypassing `AlreadyContributed` or `CircleStatus` checks. |
| **Medium** | Griefing, state corruption, or authorization bypass with bounded non-custodial impact outside documented design assumptions. | Preventing legitimate members from calling `leave_circle` or `cancel_circle` while `Forming`; integer overflow/underflow or panic in edge-case contract inputs that bricks a specific circle before funds are deposited. |
| **Low** | Defense-in-depth improvements, minor information disclosure, or edge cases with no direct impact on funds or signatures. | Non-exploitable simulation discrepancies, minor CLI validation omissions, or hardened error handling recommendations. |

---

## Response timeline

We aim to meet the following response targets for all valid private security reports:

- **Initial acknowledgment:** Within **48 hours** of report submission.
- **Triage & severity assessment:** Within **5 business days**, confirming reproduction, scope, and severity classification.
- **Fix & migration preparation:**
  - **Critical / High:** Target remediation and redeploy within **7–14 days**.
  - **Medium / Low:** Target remediation within **30 days** or the next scheduled release.
- **Coordinated disclosure:** Public advisory publication is coordinated with the reporter once a patched contract/client is deployed and active circles have had a migration window.

---

## Disclosure & incident response for an immutable contract

As documented in [docs/DEPLOYMENT.md](./docs/DEPLOYMENT.md), `contracts/ajo-circle` has **no on-chain upgrade mechanism** (`update_current_contract_wasm` is intentionally absent so no admin key can swap code beneath an active circle). Consequently, fixing a vulnerability in the smart contract **always requires deploying a new contract instance with a new contract ID and migrating users**, rather than patching the existing address in place.

When a vulnerability affecting `contracts/ajo-circle` is confirmed, incident response follows a five-step **redeploy and migration procedure**:

1. **Private triage & embargo:**
   - The vulnerability remains under strict embargo in the GitHub Security Advisory while a patch and regression test (`contracts/ajo-circle/src/test.rs`) are developed and verified in isolation.
   - Public disclosure is held until the patched contract is deployed and active circles on the vulnerable contract ID have been safely wound down or migrated.

2. **Halting new deposits (`Forming` and `Active` circles):**
   - `frontend`, `@ajo/sdk`, and `@ajo/cli` are updated immediately to stop creating new circles or submitting new `contribute` transactions against the deprecated contract ID.
   - For circles still in `CircleStatus::Forming` (where `contribute` is rejected and zero funds are held by the contract), creators and members are instructed to call `cancel_circle` or `leave_circle` on the old contract ID.

3. **Draining / settling in-flight `Active` circles:**
   - Because `contracts/ajo-circle` holds funds only for the **current cycle** and releases the entire collected pot on every `disburse` call, active circles do not need to complete all remaining cycles on the vulnerable contract to recover funds currently in custody.
   - Once either all members have contributed for the active cycle or the current cycle deadline (`started_at + (current_cycle + 1) * cycle_length_secs`) passes, any participant or maintainer can invoke the permissionless `disburse` entrypoint on the old contract ID to flush the current pot to that cycle's scheduled recipient, leaving the contract with a **zero token balance** for that circle before further contributions occur.

4. **Redeployment & client cutover:**
   - A patched `ajo_circle.wasm` is built and deployed to the network, producing a new `CONTRACT_ID` (see [docs/DEPLOYMENT.md](./docs/DEPLOYMENT.md)).
   - `NEXT_PUBLIC_AJO_CONTRACT_ID` in `frontend` and default contract references in `sdk`/`cli` are pointed to the new contract ID.
   - Circles that halted mid-rotation on the old contract ID can re-create a circle on the new contract ID with the remaining rotation order.

5. **Coordinated advisory publication:**
   - Once client defaults point to the new contract ID and in-flight cycle pots on the old contract ID have been disbursed, the GitHub Security Advisory is published with full technical details, affected commit/WASM hashes, the new contract ID, and credit to the reporter.
