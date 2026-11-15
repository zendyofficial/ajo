# Architecture

## No backend, on purpose

Most dApps quietly reintroduce a trusted third party through the back door:
a backend that indexes contract state into a database, builds transactions
on the user's behalf, or gatekeeps which reads/writes are allowed. That
backend's uptime, honesty, and security become part of the trust model —
even though the contract itself is trustless.

Ajo doesn't have one. There is no server between a user's wallet and the
Soroban contract. This has three concrete consequences:

1. **Every write is a direct wallet-signed call.** `frontend/src/lib/contract.ts`
   builds an unsigned transaction, simulates it against Soroban RPC to
   compute fees and footprint, and hands the assembled transaction to the
   connected wallet (Freighter) to sign. The signed transaction is submitted
   straight to RPC — Ajo's own infrastructure never touches it.
2. **Every read is a simulated call, not a database query.** `getCircle`,
   `hasContributed`, `missedCount`, and `totalCircles` all `simulateTransaction`
   against a throwaway, zero-balance account. No funds or prior state are
   required to read — only to write.
3. **Discovery reads the contract's own event log.** "What circles exist?"
   is answered by `getEvents`, filtered to this contract's `(circle,
   created)` topic, not by a database Ajo maintains and could misrepresent.
   See `discoverCircleIds` in `frontend/src/lib/contract.ts`.

## Why join order is payout order

A circle's payout order is fixed the instant it fills, and it's simply
join order: whoever joined first is paid first. This was a deliberate
choice over two alternatives:

- **Creator-assigned order** would require trusting the creator not to pay
  themselves first, or pay a favored member early.
- **On-chain randomization** (e.g. hashing the ledger sequence at
  activation) is well-understood on Stellar but adds a source of
  complexity and a plausible "was this manipulated?" question for a
  reviewer to have to independently verify.

Join order is simpler to reason about, trivially verifiable by reading
`Circle.members` directly, and removes an entire category of "trust the
implementation" concern.

## Why a missed deadline still triggers a partial payout

Requiring every member to contribute before `disburse` can succeed sounds
safer, but it creates a real failure mode: one member who never contributes
freezes everyone else's money in the pot indefinitely, since Soroban
contracts have no way to unilaterally seize a missing member's would-be
contribution.

Instead, `disburse` succeeds once *either* every member has paid, *or* the
cycle deadline has passed — paying out whatever was actually collected.
A non-payer is recorded with a strike (`missed_count`) rather than being
able to hold the group's money hostage. This trades a small amount of
payout-amount unpredictability for the much stronger guarantee that funds
can never get permanently stuck.

## Native XLM by default

Circles default to Stellar's native asset. A Stellar Asset Contract (SAC)
token — including a stablecoin — works identically, since `contribute` and
`disburse` both go through the generic `token::Client` interface. Native
XLM is the default specifically so that joining a circle doesn't require a
new trustline first: any wallet already funded with XLM to pay network fees
can join without an extra setup step.

## Trust and failure assumptions

The design choices above remove a custodial middleman, but they rest on
explicit operational and economic assumptions. Anyone evaluating or joining a
circle should understand what the contract guarantees, what it assumes, and
what happens when each assumption breaks.

### 1. Liveness assumption: permissionless `disburse` with no built-in keeper

- **What the contract assumes:** `disburse` is permissionless — anyone can
  call it once all members have contributed for `current_cycle` or the cycle
  deadline has elapsed — but the protocol runs **no automated keeper** and
  pays **no on-chain caller incentive**. At least one participant (typically
  the current cycle's recipient) is assumed to submit `disburse` promptly
  when a cycle finishes.
- **What happens when this breaks:** Cycle deadlines are calculated from the
  circle's activation timestamp rather than the previous disbursement time:
  `deadline = started_at + (current_cycle + 1) * cycle_length_secs`
  (`contracts/ajo-circle/src/lib.rs`). Because `current_cycle` only advances
  when `disburse` is actually invoked, failing to call `disburse` on time
  delays the transition into the next cycle while the wall-clock schedule for
  subsequent cycles continues ticking. If `disburse` for cycle `N` is called
  after cycle `N + 1`'s deadline has already passed, cycle `N + 1` opens with
  its deadline already expired ("burned cycles"), meaning anyone could
  immediately trigger `disburse` for cycle `N + 1` before members have a
  normal window to contribute.

### 2. Token trust boundary: `circle.token` is arbitrary and unvalidated

- **What the contract assumes:** `create_circle` accepts an arbitrary
  `token: Address` chosen by the circle creator and interacts with it via
  `soroban_sdk::token::Client` (`transfer` into the contract during
  `contribute` and `transfer` out to the recipient during `disburse`). The
  contract does not maintain a token allowlist or verify that `token` is a
  canonical Stellar Asset Contract (SAC).
- **What happens when this breaks:** A circle's custody guarantees are
  bounded by the honesty and spec-compliance of `circle.token`:
  - **Malicious or upgradable custom token:** A token contract controlled by
    an external admin can freeze transfers, blacklist a recipient address so
    `disburse` reverts permanently, or mint/burn balances arbitrarily.
  - **Fee-on-transfer or rebasing tokens:** `disburse` computes the payout as
    `paid_count * circle.contribution_amount` rather than querying the
    contract's token balance delta. If `token.transfer` deducts a transfer
    fee on deposit, the contract holds less than `pot`, causing `disburse`'s
    outbound `transfer` to fail and locking the circle.
  - **Verification requirement:** Members must independently verify that
    `circle.token` matches the expected SAC address (e.g., native XLM or a
    trusted stablecoin SAC) before joining or contributing.

### 3. Counterparty assumption: strikes (`missed_count`) are informational only

- **What the contract assumes:** The contract enforces **non-custodial pot
  custody**, not collateralized credit enforcement. When `disburse` executes
  after a deadline, any member who did not pay in `current_cycle` has their
  `DataKey::MissedCount(circle_id, member)` counter incremented, and the
  recipient receives the partial pot collected so far.
- **What happens when this breaks:** Strikes carry **no on-chain penalty,
  slashing, or payout disqualification**:
  - **Free-riding by later recipients:** Because `disburse` selects the
    recipient purely by join index (`circle.members.get(circle.current_cycle)`),
    a member who skips every prior cycle's contribution and accumulates
    strikes will still receive the full pot collected during their own turn
    when `current_cycle` reaches their position (unless other members inspect
    `missed_count` and stop contributing in response).
  - **Early-recipient defection:** A member who joins early (e.g., index `0`)
    receives the first cycle's pot and can stop contributing in all remaining
    cycles (`1..max_members`) with no on-chain financial recourse, reducing
    subsequent payouts to `(max_members - 1) * contribution_amount`.
  - **Implication:** Ajo eliminates the risk of an *organizer* running away
    with the pot, but circles formed with untrusted strangers remain exposed
    to member defection unless combined with off-chain social trust or future
    collateral/slashing rules.

### 4. Immutability assumption: no upgrade path

- **What the contract assumes:** `contracts/ajo-circle` intentionally omits
  any admin role or `update_current_contract_wasm` entrypoint so that no
  deployer or third party can alter the rules of an active circle after
  funds are deposited.
- **What happens when this breaks:** Any bug in the contract logic or change
  in protocol behavior requires deploying a **new contract instance** at a
  new contract ID (see [docs/DEPLOYMENT.md](./DEPLOYMENT.md)). Existing
  circles on the old contract ID cannot be patched in place; they must
  either be cancelled/left while `Forming`, or have their current cycle pot
  flushed via `disburse` before participants recreate the circle on the new
  deployment.
