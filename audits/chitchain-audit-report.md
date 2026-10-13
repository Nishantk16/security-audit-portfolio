================================================================
PROJECT: ChitChain (Soroban ROSCA/Chit Fund Protocol)

REPO: github.com/Nishantk16/chitchain
CONTRACTS: registry, chit_chain
REVIEWER: Nishant (Nishantk16)
DATE: September 2026
================================================================

## FINDING #1 [MEDIUM] — Unbounded Registry Storage Growth Enables Resource-Exhaustion DoS

### Title
`register_circle()` has no cap on the number of circles that can be registered, allowing unbounded growth of on-chain storage structures.

### Submitted By
Nishantk16

### Impact
`RegistryContract::register_circle()` accepts new circle registrations with no upper bound on total count. Each successful call:
1. Writes a new `CircleEntry` to persistent storage under `DataKey::CircleEntry(circle)`.
2. Appends the circle's address to a single `Vec<Address>` stored under `DataKey::CircleList` in **instance storage**.
3. Increments a `CircleCount` counter.

Because `CircleList` is one growing `Vec` stored as a single instance-storage entry, every `register_circle()` call must **read, deserialize, mutate, re-serialize, and rewrite the entire list** — not just append an isolated record. As the list grows:
- The cost (CPU + storage read/write fees) of every future registration increases monotonically.
- Instance storage entries carry Soroban's stricter footprint/size constraints than persistent storage; a sufficiently large `CircleList` risks exceeding the ledger's per-entry size limits, at which point **`register_circle()` becomes permanently uncallable for anyone** — a hard denial of service on new circle creation for the entire protocol, not just the attacker.
- Any downstream function that reads the full `CircleList` (e.g., an enumeration/listing view) degrades in cost alongside it.

Note that `register_circle()` requires `admin.require_auth()` — the caller supplies an `admin: Address` parameter and that address must sign. This limits *casual* spam somewhat (an attacker needs a signing key for the `admin` on each call), but does **not** prevent a single actor from generating many keypairs (which is free) and driving `CircleCount` up arbitrarily, since there is no relationship enforced between the `admin` address and any real-world identity or stake. The auth check bounds *who* can register, not *how many* registrations occur.

### Proof of Concept
```rust
// Before the fix, register_circle() had no ceiling:
pub fn register_circle(
    env: Env,
    circle: Address,
    name: Symbol,
    admin: Address,
) -> Result<(), RegistryError> {
    admin.require_auth();
    if !env.storage().instance().has(&DataKey::Admin) {
        return Err(RegistryError::NotInitialized);
    }
    let key = DataKey::CircleEntry(circle.clone());
    if env.storage().persistent().has(&key) {
        return Err(RegistryError::AlreadyRegistered);
    }
    // ... entry written, CircleList.push_back(circle), CircleCount += 1 ...
    // NO CHECK ANYWHERE ON HOW MANY CIRCLES ALREADY EXIST.
}
```
An attacker script can:
1. Generate N distinct Stellar keypairs locally (free, off-chain, no funding required to *generate* — only to submit transactions).
2. For each, call `register_circle(new_circle_address, name, new_admin_address)` with the freshly generated `admin` signing its own call.
3. Repeat until `CircleList` approaches the instance-storage size ceiling, at which point every subsequent call — from any user — reverts or becomes prohibitively expensive due to the read-modify-write cost on the bloated `Vec`.

The only real-world cost to the attacker is Soroban transaction/resource fees per call — no protocol-level rate limit or cap existed to stop this before the fix.

### Mitigation
Fixed in commit `12a37e4` by introducing a hard ceiling:
```rust
pub const MAX_CIRCLES: u32 = 10_000;
...
let count: u32 = env.storage().instance().get(&DataKey::CircleCount).unwrap_or(0);
if count >= MAX_CIRCLES {
    return Err(RegistryError::RegistryFull);
}
```
This check runs before any storage mutation in `register_circle()`, so once the cap is hit, further registrations cleanly revert with a new `RegistryError::RegistryFull` variant instead of degrading storage cost unboundedly.

**Residual consideration (not yet addressed):** the fix caps total *count* but `CircleList` remains a single growing `Vec` in instance storage. At 10,000 entries this Vec could still be large depending on `Address` encoding size; a follow-up improvement would be to move enumeration to an indexed/paginated persistent-storage pattern instead of one monolithic `Vec`, so read costs for any single operation stay flat regardless of total registered circles.

### Assessed Type
`DoS` / Resource Exhaustion — Unbounded on-chain data structure growth.

### Sponsor Discussion
Self-identified during Day87 review of the codebase's own commit history while researching prior fixes for this audit-report exercise. Fixed proactively (commit `12a37e4`, 11 Sep 2026) before any external report — no live exploitation observed since ChitChain is a solo hackathon/portfolio project on testnet, not a funded live protocol.

---

## FINDING #2 [MEDIUM] — Incorrect Cross-Contract Authorization Caused Permanent State-Transition Failure

### Title
`update_circle_status()` required the wrong party's signature, causing a real, unrecoverable transaction failure whenever a circle attempted to transition to `Active`.

### Submitted By
Nishantk16

### Impact
`RegistryContract::update_circle_status()` is called **cross-contract** by the `chit_chain` contract itself, from inside `join_circle()` (when a circle reaches its member quota and should flip to `Active`) and `execute_payout()`. The transaction triggering this call is signed by a regular **member** or an **operator** — never by the circle's stored `admin`.

The original implementation required:
```rust
entry.admin.require_auth();
```
— i.e., it demanded a signature from the circle's `admin` address specifically. Since the actual caller (member/operator) never holds that key, every real invocation reverted with `Auth(InvalidAction)`.

**Effect: the circle's status could never transition to `Active` on-chain**, even though all functional preconditions (member count, round setup) were correctly met. This is a full **availability failure of a core protocol invariant** — a chit circle that can never leave its initial state is a chit circle that can never actually run.

Critically, this bug was **invisible in the test suite**: `registry_integration_test.rs` used `env.mock_all_auths()`, which bypasses all `require_auth()` checks unconditionally. The tests validated business logic assuming authorization succeeded, but never validated *which party* the authorization was actually being requested from — so a call requiring the wrong signer still passed every test. It was only caught on a real multi-signer **testnet** transaction, where mocking isn't available and the actual signer mismatch surfaced.

### Proof of Concept
Sequence that reproduces the failure (pre-fix):
1. `admin` registers a circle via `register_circle()`.
2. Circle members join via `chit_chain::join_circle()`, signing with their own member keys.
3. When the Nth member joins and the circle should activate, `join_circle()` internally calls:
```rust
   registry_client.update_circle_status(&circle_address, &ACTIVE_STATUS);
```
   This cross-contract call executes in the context of the transaction submitted by the **joining member**, not the admin.
4. Inside `update_circle_status()`, `entry.admin.require_auth()` demands the admin's signature — which is absent from this transaction.
5. Transaction fails: `HostError: Error(Auth, InvalidAction)`.
6. The circle is permanently stuck pre-Active; no valid caller combination can ever satisfy this check through the normal flow, since the admin never initiates `join_circle()` or `execute_payout()` in the intended design.

### Mitigation
Fixed in commit `115bb24` by requiring authorization from the **calling circle contract itself**, not the stored admin:
```rust
pub fn update_circle_status(
    env: Env,
    circle: Address,
    status_code: u32,
) -> Result<(), RegistryError> {
    circle.require_auth();   // <-- was: entry.admin.require_auth()
    let key = DataKey::CircleEntry(circle.clone());
    let mut entry: CircleEntry = env.storage().persistent().get(&key).ok_or(RegistryError::NotFound)?;
    entry.status_code = status_code;
    env.storage().persistent().set(&key, &entry);
    ...
}
```
A contract's own `require_auth()` inside a cross-contract call **it itself initiated** is satisfied automatically by the Soroban host — no external signature is needed, and the check still correctly restricts the update to the registered circle contract specifically (an unrelated contract cannot forge this, since it cannot make `circle`'s own address satisfy `circle.require_auth()`).

The fix also added `upgrade()` / `get_version()` to the registry contract (mirroring `chit_chain`'s existing pattern) and a `set_registry_contract()` admin-only escape hatch on `chit_chain`, so that any already-deployed circle could be repointed at a corrected registry instance without redeploying or losing member/round history — since the registry itself had no prior upgrade path.

### Assessed Type
Access Control — Incorrect authorization target causing broken core functionality (Availability).

### Sponsor Discussion
Self-identified via a real testnet transaction failure (not caught by the mocked test suite) during Level 2/3 development. Fixed same-day (commit `115bb24`, 11 Jul 2026), with an explicit note left in the commit message flagging the test-suite gap (`mock_all_auths()` masking signer-identity bugs) as a follow-up item — not yet remediated with targeted auth assertions as of this report.

================================================================
SUMMARY
2 findings — both Medium severity, both self-identified and 
fixed prior to this report. Finding #2 (auth) is the more 
architecturally significant: it demonstrates how mock-based 
auth testing can hide a class of bug that only appears in real 
multi-signer conditions — a durable lesson for future contract 
test design. Finding #1 (storage cap) is a textbook Soroban 
resource-exhaustion pattern, now bounded but with a noted 
residual scaling concern in the CircleList data structure.
================================================================
