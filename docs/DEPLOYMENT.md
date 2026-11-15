# Deploying the contract

There's no upgrade mechanism on `ajo-circle` — redeploying always produces a
**new contract id**, and every circle that existed under the old id is
orphaned (still queryable at the old id if you keep the RPC/network the
same, but the frontend needs to point at the new one to create or interact
with anything new). Budget for that before redeploying against real usage.

## Prerequisites

- [`stellar` CLI](https://developers.stellar.org/docs/tools/developer-tools) installed
- A funded Stellar account to deploy from (testnet: fund via
  [the Laboratory](https://laboratory.stellar.org/#account-creator?network=test)
  or `stellar keys generate --fund`)

## Build and test first

```bash
cd contracts
cargo test --workspace
cd ajo-circle
stellar contract build
```

Produces `contracts/target/wasm32v1-none/release/ajo_circle.wasm`.

## Deploy

```bash
stellar keys generate deployer --network testnet --fund

stellar contract deploy \
  --wasm target/wasm32v1-none/release/ajo_circle.wasm \
  --source deployer \
  --network testnet
```

This prints the new contract id.

## Point the frontend at it

Set `NEXT_PUBLIC_AJO_CONTRACT_ID` in `frontend/.env.local` to the new id,
then restart the dev server (`rm -rf frontend/.next` first if it was
already running — Next.js can serve a stale build with the old env value
baked in otherwise).

For mainnet, also update `NEXT_PUBLIC_STELLAR_RPC_URL` and
`NEXT_PUBLIC_STELLAR_NETWORK_PASSPHRASE`, and `NEXT_PUBLIC_NATIVE_TOKEN_ID`
to mainnet's native XLM SAC address — it differs from testnet's.

## If `stellar keys ... --fund` or `stellar contract deploy` fail to reach the network

Some sandboxed/containerized environments have a broken TLS trust store for
compiled Rust binaries specifically, while `fetch`/`curl` from other
runtimes on the same machine work fine against the same hosts. If
`stellar`'s own network calls fail with a generic connection error but
`curl https://soroban-testnet.stellar.org` (or the equivalent in Node)
succeeds, that's almost certainly what's happening — it's an environment
quirk, not a problem with the contract or the deploy target.

Workaround: do the upload + create-contract calls directly against RPC with
`@stellar/stellar-sdk` instead of going through the `stellar` binary:

```js
import fs from "node:fs";
import { Keypair, TransactionBuilder, Networks, BASE_FEE, Operation, rpc, Address } from "@stellar/stellar-sdk";

const server = new rpc.Server("https://soroban-testnet.stellar.org");
const kp = Keypair.fromSecret(DEPLOYER_SECRET);

async function submit(tx) {
  const sim = await server.simulateTransaction(tx);
  const prepared = rpc.assembleTransaction(tx, sim).build();
  prepared.sign(kp);
  const sent = await server.sendTransaction(prepared);
  let result = await server.getTransaction(sent.hash);
  while (result.status === "NOT_FOUND") {
    await new Promise((r) => setTimeout(r, 1500));
    result = await server.getTransaction(sent.hash);
  }
  return result;
}

const account = await server.getAccount(kp.publicKey());
const wasm = fs.readFileSync("target/wasm32v1-none/release/ajo_circle.wasm");

const uploadTx = new TransactionBuilder(account, { fee: BASE_FEE, networkPassphrase: Networks.TESTNET })
  .addOperation(Operation.uploadContractWasm({ wasm }))
  .setTimeout(60)
  .build();
const { returnValue: wasmHashScVal } = await submit(uploadTx);
const wasmHash = wasmHashScVal.bytes();

const account2 = await server.getAccount(kp.publicKey());
const createTx = new TransactionBuilder(account2, { fee: BASE_FEE, networkPassphrase: Networks.TESTNET })
  .addOperation(Operation.createCustomContract({ address: new Address(kp.publicKey()), wasmHash }))
  .setTimeout(60)
  .build();
const { returnValue: addressScVal } = await submit(createTx);
console.log("CONTRACT_ID:", Address.fromScAddress(addressScVal.address()).toString());
```

This is exactly what `stellar contract deploy` does under the hood — same
two operations, same network calls — just issued from a runtime whose TLS
stack happens to work in that environment. It's not a different or less
trustworthy deployment path, only a different HTTP client making the same
requests.

## Soroban storage TTL, state archival, and long-running circles

Soroban enforces state archival on all contract code, contract instances, and
persistent ledger entries. Every ledger entry has a **Time-To-Live (TTL)**
measured in ledgers (~5 seconds per ledger). If an entry's TTL reaches zero
without being extended, the entry is moved to the network's **archived ledger
state**, and any transaction touching that entry fails until it is explicitly
restored.

### How `ajo-circle` uses storage today

| Storage tier | Keys stored (`contracts/ajo-circle/src/lib.rs`) | Current `extend_ttl` behavior |
|---|---|---|
| **Instance storage** (`env.storage().instance()`) | `DataKey::NextCircleId` (plus the contract instance and uploaded WASM bytecode) | **None.** The contract does not call `env.storage().instance().extend_ttl(...)`. |
| **Persistent storage** (`env.storage().persistent()`) | `DataKey::Circle(id)`, `DataKey::Contribution(id, cycle, member)`, `DataKey::MissedCount(id, member)` | **None.** The contract does not call `env.storage().persistent().extend_ttl(...)`. |

### Lifetime bounds under the current implementation

Because `contracts/ajo-circle` does not yet invoke `extend_ttl` on-chain:

1. **Contract instance & WASM TTL (~30 days default minimum):**
   New contract instances receive the network's minimum instance/temporary-style
   TTL floor unless explicitly extended. While writes to `NextCircleId` (during
   `create_circle`) bump the instance entry up to `min_persistent_entry_ttl`
   when modified, a deployment that goes weeks without a new `create_circle`
   call — or whose underlying WASM code entry is never extended — will see its
   instance/WASM expire even while existing circles are still mid-rotation.
2. **Persistent circle entries (~120 days default minimum on creation/write):**
   On Stellar Mainnet and Testnet, `min_persistent_entry_ttl` is approximately
   **2,073,600 ledgers (~120 days)** and `max_entry_ttl` is **6,312,000 ledgers
   (~365 days)** (assuming ~5s ledgers). Writing to `DataKey::Circle(id)`
   (on `create_circle`, `join_circle`, or `disburse`) only guarantees that the
   entry's TTL is at least `min_persistent_entry_ttl` from that write — it does
   **not** automatically extend the TTL to a full year, nor does `contribute`
   rewrite `DataKey::Circle(id)` (it only writes `DataKey::Contribution(...)`).
3. **Maximum safe circle duration without manual TTL extension:**
   - A **12-member monthly circle runs for ~365 days**, which exceeds the
     default persistent entry TTL and the contract instance/WASM TTL unless
     rent is extended along the way.
   - Under the current contract implementation (prior to adding on-chain
     `extend_ttl` calls), a circle is only guaranteed to remain live without
     manual rent intervention if **both** the contract instance/WASM TTL covers
     the entire rotation **and** the interval between `DataKey::Circle(id)`
     writes (`join_circle` and each cycle's `disburse`) stays safely under
     **~90 days** (with total duration under **~90–100 days** unless extended).

### What happens if a circle is archived mid-rotation

If `DataKey::Circle(id)`, a member's `DataKey::Contribution` / `MissedCount`
entry, the contract instance, or the contract's token balance entry in the
Stellar Asset Contract (SAC) expires while a circle is `Active`:

- **Funds are not destroyed, but the pot becomes unreachable until restored:**
  Simulating or submitting `contribute` or `disburse` will fail because Soroban
  cannot read or modify archived persistent entries without a restore
  operation first.
- **Cycle deadlines continue to advance in wall-clock time:** Because
  `deadline = started_at + (current_cycle + 1) * cycle_length_secs`, time spent
  while the circle is archived still counts against the active cycle's
  deadline.

### Who is responsible for paying rent?

Soroban storage rent is **permissionless**. Any funded Stellar account — the
contract deployer, the circle creator, or any circle member — can extend the
TTL of the contract instance, WASM code, or persistent entries before they
expire, or restore them after archival.

For production or long-running circles (such as multi-month rotations), the
**deployer or circle creator** should proactively extend the contract instance
and WASM TTL close to `max_entry_ttl` (~535,000+ ledgers for multi-week
 buffers, up to ~6,312,000 ledgers / 1 year) right after deployment:

```bash
# Extend the contract instance and uploaded WASM bytecode TTL (e.g. by 535,680 ledgers / ~31 days, or up to max_entry_ttl)
stellar contract extend \
  --id <CONTRACT_ID> \
  --source deployer \
  --network testnet \
  --ledgers-to-extend 535680
```

### Restoring an archived contract or circle entry

If the contract instance, WASM bytecode, or a circle's persistent storage entry
has been archived due to TTL expiration, restore it using `stellar contract restore`
before resuming `contribute` or `disburse` calls:

```bash
# 1. Restore the contract instance and its backing WASM code
stellar contract restore \
  --id <CONTRACT_ID> \
  --source deployer \
  --network testnet

# 2. If a transaction simulation via @stellar/stellar-sdk or the CLI reports an
#    expired persistent entry (e.g. DataKey::Circle(id)), assembleTransaction()
#    automatically attaches a RestoreFootprintOp preamble when simulating the
#    read/write, or you can restore the footprint directly via the CLI.
```

> **Note for future contract versions:** Once on-chain `extend_ttl` calls are
> added to `create_circle`, `join_circle`, `contribute`, and `disburse`, active
> circles will automatically bump the TTL of both `env.storage().instance()`
> and `DataKey::Circle(id)` as part of each member interaction, shifting incremental
> rent costs into normal transaction fees.
