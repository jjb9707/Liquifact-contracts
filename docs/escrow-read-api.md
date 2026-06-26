# Escrow Read API

Complete catalog of all public read-only views on `LiquifactEscrow`. All functions are pure reads:
no state mutation, no authorization required unless specified otherwise.

**Integrator note:** Return types, defaults, and absent-key behavior documented for each view match
the on-chain implementation exactly. Off-chain tooling should use these views rather than
re-implementing storage reads to guarantee identical semantics.

---

## Index

**Core Escrow State:**
- [get_escrow](#get_escrow--invoiceescrow)
- [get_version](#get_version--u32)
- [get_escrow_summary](#get_escrow_summary--escrowsummary)

**Immutable Bindings:**
- [get_funding_token](#get_funding_token--address)
- [get_treasury](#get_treasury--address)
- [get_registry_ref](#get_registry_ref--optionaddress)

**Admin & Governance:**
- [get_pending_admin](#get_pending_admin--optionaddress)
- [get_legal_hold](#get_legal_hold--bool)
- [get_legal_hold_clear_delay](#get_legal_hold_clear_delay--u64)
- [get_legal_hold_clearable_at](#get_legal_hold_clearable_at--optionu64)

**Funding Constraints:**
- [get_funding_deadline](#get_funding_deadline--optionu64)
- [is_funding_expired](#is_funding_expired--bool)
- [get_min_contribution_floor](#get_min_contribution_floor--i128)
- [get_max_unique_investors_cap](#get_max_unique_investors_cap--optionu32)
- [get_max_per_investor_cap](#get_max_per_investor_cap--optioni128)

**Maturity & Settlement:**
- [has_maturity_lock](#has_maturity_lock--bool)
- [get_funding_close_snapshot](#get_funding_close_snapshot--optionfundingclosesnapshot)

**Per-Investor State:**
- [get_contribution](#get_contributioninvestor-address--i128)
- [get_unique_funder_count](#get_unique_funder_count--u32)
- [get_investor_yield_bps](#get_investor_yield_bpsinvestor-address--i64)
- [get_investor_claim_not_before](#get_investor_claim_not_beforeinvestor-address--u64)
- [is_investor_claimed](#is_investor_claimedinvestor-address--bool)
- [is_investor_refunded](#is_investor_refundedinvestor-address--bool)
- [compute_investor_payout](#compute_investor_payoutinvestor-address--i128)

**Attestations:**
- [get_primary_attestation_hash](#get_primary_attestation_hash--optionbytesn32)
- [get_attestation_append_log](#get_attestation_append_log--vecbytesn32)
- [is_attestation_revoked](#is_attestation_revokedindex-u32--bool)

**Collateral Metadata:**
- [get_sme_collateral_commitment](#get_sme_collateral_commitment--optionsmecollateralcommitment)

**Allowlist:**
- [is_allowlist_active](#is_allowlist_active--bool)
- [is_investor_allowlisted](#is_investor_allowlistedinvestor-address--bool)

**Distributed Principal:**
- [get_distributed_principal](#get_distributed_principal--i128)

---

## Core Escrow State

### `get_escrow() → InvoiceEscrow`

**Storage key:** `DataKey::Escrow`  
**Signature:** `pub fn get_escrow(env: Env) -> InvoiceEscrow`

Returns the full escrow snapshot containing all core state fields.

**Requires initialization:** Yes — emits [`EscrowError::EscrowNotInitialized`] (code 20) if called before `init`.

**Return value:**
- `InvoiceEscrow` struct with fields: `invoice_id`, `admin`, `sme_address`, `amount`, `funding_target`, `funded_amount`, `yield_bps`, `maturity`, `status`.

---

### `get_version() → u32`

**Storage key:** `DataKey::Version`  
**Signature:** `pub fn get_version(env: Env) -> u32`

Returns the stored schema version written by `init` (see `SCHEMA_VERSION`).

**Requires initialization:** No  
**Default when absent:** `0`

**Return value:**
- `u32` schema version (current production: `6`).
- Returns `0` if called before `init`.

---

### `get_escrow_summary() → EscrowSummary`

**Signature:** `pub fn get_escrow_summary(env: Env) -> EscrowSummary`

Bundles multiple read-only values in a single host invocation, optimizing read latency and gas efficiency for off-chain indexers and frontend rendering.

**Requires initialization:** Yes — panics via `get_escrow` if escrow is not initialized.

**Return value:** `EscrowSummary` struct containing:
- `escrow: InvoiceEscrow` — Full escrow snapshot.
- `has_maturity_lock: bool` — True when `escrow.maturity > 0`.
- `legal_hold: bool` — True if compliance hold is active.
- `funding_close_snapshot: EscrowCloseSnapshot` — Custom option-like enum (`None` or `Some(FundingCloseSnapshot)`).
- `unique_funder_count: u32` — Distinct address count.
- `is_allowlist_active: bool` — Allowlist gate status.
- `schema_version: u32` — Contract schema version.
- `sme_collateral_commitment: CollateralCommitmentSnapshot` — Custom option-like enum (`None` or `Some(SmeCollateralCommitment)`).
- `has_primary_attestation: bool` — Primary attestation binding status.
- `attestation_log_length: u32` — Number of append-log entries.

---

## Immutable Bindings

### `get_funding_token() → Address`

**Storage key:** `DataKey::FundingToken`  
**Signature:** `pub fn get_funding_token(env: Env) -> Address`

Returns the SEP-41 token contract address bound to this escrow instance at `init`.

**Immutable:** Set once at `init`; cannot change after deploy.  
**Requires initialization:** Yes — emits [`EscrowError::FundingTokenNotSet`] (code 21) if called before `init`.

**Return value:**
- `Address` of the funding token contract.
- This is the only token that `sweep_terminal_dust` may transfer to the treasury.

---

### `get_treasury() → Address`

**Storage key:** `DataKey::Treasury`  
**Signature:** `pub fn get_treasury(env: Env) -> Address`

Returns the protocol treasury address that receives terminal dust sweeps.

**Immutable:** Set once at `init`; cannot change after deploy.  
**Requires initialization:** Yes — emits [`EscrowError::TreasuryNotSet`] (code 22) if called before `init`.

**Return value:**
- `Address` of the treasury.
- The treasury must authorize `sweep_terminal_dust`; the admin cannot sweep unless it is also the treasury.

---

### `get_registry_ref() → Option<Address>`

**Storage key:** `DataKey::RegistryRef`  
**Signature:** `pub fn get_registry_ref(env: Env) -> Option<Address>`

Returns the optional registry contract address supplied at `init`, or `None` when absent.

**Immutable:** Set once at `init`; cannot change after deploy.  
**Requires initialization:** No  
**Default when absent:** `None`

**Non-authority model:**
- `RegistryRef` is a **read-only discoverability hint** for off-chain indexers only.
- No on-chain logic in this contract reads or calls this address.
- Its presence **does not** prove registry membership; call the registry contract directly to verify.
- The key is omitted from instance storage entirely when `registry = None` at `init`.

**Return value:**
- `Some(Address)` when a registry was configured.
- `None` otherwise.

---

## Admin & Governance

### `get_pending_admin() → Option<Address>`

**Storage key:** `DataKey::PendingAdmin`  
**Signature:** `pub fn get_pending_admin(env: Env) -> Option<Address>`

Returns the proposed successor admin waiting for `accept_admin`, or `None` when no handover is in progress.

**Requires initialization:** No  
**Default when absent:** `None`

**Return value:**
- `Some(Address)` when a handover is pending.
- `None` when no `propose_admin` has been issued, or after a successful `accept_admin`.

---

### `get_legal_hold() → bool`

**Storage key:** `DataKey::LegalHold`  
**Signature:** `pub fn get_legal_hold(env: Env) -> bool`

Returns `true` when a compliance hold is active; blocks `settle`, `withdraw`, `claim_investor_payout`, `fund`, and `sweep_terminal_dust`.

**Requires initialization:** No  
**Default when absent:** `false`

---

### `get_legal_hold_clear_delay() → u64`

**Storage key:** `DataKey::LegalHoldClearDelay`  
**Signature:** `pub fn get_legal_hold_clear_delay(env: Env) -> u64`

Returns the configured minimum delay (in seconds) between `request_clear_legal_hold` and `set_legal_hold(false)`.

**Requires initialization:** No  
**Default when absent:** `0` (no delay enforced; hold can be cleared immediately)

---

### `get_legal_hold_clearable_at() → Option<u64>`

**Storage key:** `DataKey::LegalHoldClearableAt`  
**Signature:** `pub fn get_legal_hold_clearable_at(env: Env) -> Option<u64>`

Returns the earliest ledger timestamp at which a pending legal-hold clear may be applied, or `None` when no clear request has been recorded.

**Requires initialization:** No  
**Default when absent:** `None`

**Return value:**
- `Some(timestamp)` after `request_clear_legal_hold` is called.
- `None` when no request is pending (or after a successful clear removes the key).

---

## Funding Constraints

### `get_funding_deadline() → Option<u64>`

**Storage key:** `DataKey::FundingDeadline`  
**Signature:** `pub fn get_funding_deadline(env: Env) -> Option<u64>`

Returns the optional funding deadline (ledger timestamp). After this timestamp passes, `fund` calls are rejected.

**Requires initialization:** No  
**Default when absent:** `None` (no deadline — funding is open indefinitely)

**Return value:**
- `Some(timestamp)` when configured at `init`.
- `None` when no deadline was set.

---

### `is_funding_expired() → bool`

**Signature:** `pub fn is_funding_expired(env: Env) -> bool`

Returns `true` when a funding deadline is set **and** `Env::ledger().timestamp() > deadline`.

**Requires initialization:** No  
**Default when absent:** `false` (no deadline set → never expired)

**Logic:**
```
if FundingDeadline exists:
    return ledger.timestamp() > deadline
else:
    return false
```

---

### `get_min_contribution_floor() → i128`

**Storage key:** `DataKey::MinContributionFloor`  
**Signature:** `pub fn get_min_contribution_floor(env: Env) -> i128`

Returns the minimum per-call funding amount in token base units. Applies to every `fund` / `fund_with_commitment` call.

**Requires initialization:** No (but written as `0` at `init`)  
**Default when absent:** `0` (no extra floor beyond "amount must be positive")

**Notes:**
- The floor applies to **each individual deposit**, not to cumulative principal.
- Written as `0` even when unconfigured at `init`, so reads always succeed post-init.

---

### `get_max_unique_investors_cap() → Option<u32>`

**Storage key:** `DataKey::MaxUniqueInvestorsCap`  
**Signature:** `pub fn get_max_unique_investors_cap(env: Env) -> Option<u32>`

Returns the optional cap on distinct investor addresses. Reflects the current stored cap, including any reduction via `lower_max_unique_investors`.

**Requires initialization:** No  
**Default when absent:** `None` (unlimited investors)

**Return value:**
- `Some(u32)` when configured.
- `None` when no cap was set at `init`.

---

### `get_max_per_investor_cap() → Option<i128>`

**Storage key:** `DataKey::MaxPerInvestorCap`  
**Signature:** `pub fn get_max_per_investor_cap(env: Env) -> Option<i128>`

Returns the optional immutable cap on cumulative principal for a single investor address.

**Requires initialization:** No  
**Default when absent:** `None` (unlimited per-investor)

**Return value:**
- `Some(i128)` when configured at `init`.
- `None` when unconfigured.

---

## Maturity & Settlement

### `has_maturity_lock() → bool`

**Derived from:** `DataKey::Escrow.maturity`  
**Signature:** `pub fn has_maturity_lock(env: Env) -> bool`

Returns `true` when `InvoiceEscrow::maturity > 0` and `settle()` is gated by ledger time.

**Requires initialization:** Yes — calls `get_escrow` internally.

**Logic:**
```
return get_escrow().maturity > 0
```

**Return value:**
- `true` — settlement requires `Env::ledger().timestamp() >= maturity`.
- `false` — `maturity == 0`; no time lock, funded escrow can settle immediately.

---

### `get_funding_close_snapshot() → Option<FundingCloseSnapshot>`

**Storage key:** `DataKey::FundingCloseSnapshot`  
**Signature:** `pub fn get_funding_close_snapshot(env: Env) -> Option<FundingCloseSnapshot>`

Returns the pro-rata denominator snapshot captured exactly once when the escrow first transitioned from open (0) to funded (1).

**Requires initialization:** No  
**Default when absent:** `None` (escrow has not yet reached funded status)

**Immutable once written:** the snapshot is never updated after the status-0-to-1 transition.

**Return value:**
- `None` until the escrow reaches `status == 1`.
- `Some(FundingCloseSnapshot)` with fields:
  - `total_principal: i128` — `funded_amount` at close (includes over-funding past target).
  - `funding_target: i128` — Snapshot of target at close time.
  - `closed_at_ledger_timestamp: u64` — Ledger timestamp of the funding transition.
  - `closed_at_ledger_sequence: u32` — Ledger sequence at transition.

---

## Per-Investor State

### `get_contribution(investor: Address) → i128`

**Storage key:** `DataKey::InvestorContribution(investor)` (persistent)  
**Signature:** `pub fn get_contribution(env: Env, investor: Address) -> i128`

Returns the cumulative principal contributed by `investor` in token base units.

**Requires initialization:** No  
**Default when absent:** `0` (never contributed)  
**Storage type:** Persistent (independent TTL per address; see ADR-007)

---

### `get_unique_funder_count() → u32`

**Storage key:** `DataKey::UniqueFunderCount`  
**Signature:** `pub fn get_unique_funder_count(env: Env) -> u32`

Returns the count of distinct investor addresses with non-zero contributions. Initialized to `0` at `init`.

**Requires initialization:** No (but written as `0` at `init`)  
**Default when absent:** `0`

**Notes:** counts distinct chain accounts, not real-world persons (Sybil resistance is not a goal of this counter).

---

### `get_investor_yield_bps(investor: Address) → i64`

**Storage key:** `DataKey::InvestorEffectiveYield(investor)` (persistent)  
**Signature:** `pub fn get_investor_yield_bps(env: Env, investor: Address) -> i64`

Returns the effective annualized yield in basis points locked in at the investor's first deposit.

**Requires initialization:** Yes — reads `get_escrow()` for the base yield fallback.  
**Default when absent:** falls back to `InvoiceEscrow::yield_bps` (base yield for legacy / simple `fund` positions)  
**Storage type:** Persistent

**Return value:**
- Investor's tier-selected `yield_bps` when set via `fund_with_commitment`.
- Base `InvoiceEscrow::yield_bps` for simple `fund` deposits or pre-v2 positions.

---

### `get_investor_claim_not_before(investor: Address) → u64`

**Storage key:** `DataKey::InvestorClaimNotBefore(investor)` (persistent)  
**Signature:** `pub fn get_investor_claim_not_before(env: Env, investor: Address) -> u64`

Returns the earliest ledger timestamp at which `claim_investor_payout` may succeed for this investor.

**Requires initialization:** No  
**Default when absent:** `0` (no extra gate beyond settled status)  
**Storage type:** Persistent

**Return value:**
- `0` for simple `fund` deposits or when `committed_lock_secs == 0`.
- `now + committed_lock_secs` at deposit time for tiered commitments.

---

### `is_investor_claimed(investor: Address) → bool`

**Storage key:** `DataKey::InvestorClaimed(investor)` (persistent)  
**Signature:** `pub fn is_investor_claimed(env: Env, investor: Address) -> bool`

Returns `true` when the investor has exercised `claim_investor_payout` after settlement.

**Requires initialization:** No  
**Default when absent:** `false`  
**Storage type:** Persistent

**Notes:** written once and never unset. A second `claim_investor_payout` call is a no-op (idempotent) rather than an error.

---

### `is_investor_refunded(investor: Address) → bool`

**Storage key:** `DataKey::InvestorRefunded(investor)`  
**Signature:** `pub fn is_investor_refunded(env: Env, investor: Address) -> bool`

Returns `true` when an investor's principal has been returned via `refund` in a cancelled (status 4) escrow.

**Requires initialization:** No  
**Default when absent:** `false`

**Notes:** written once; prevents double-refund. After `refund` succeeds, `get_contribution` for the same address returns `0`.

---

### `compute_investor_payout(investor: Address) → i128`

**Signature:** `pub fn compute_investor_payout(env: Env, investor: Address) -> i128`

Computes the gross payout (principal share + yield coupon) for `investor` using the funding-close snapshot as the pro-rata denominator. **Authorization:** none — pure read.

**Requires initialization:** No (returns `0` for all pre-funded states)  
**Default when absent:** `0` when snapshot is missing or investor has no contribution

**Formula (truncating integer division):**
```
coupon       = total_principal × effective_yield_bps / 10_000  (floor)
settle_pool  = total_principal + coupon
gross_payout = contribution × settle_pool / total_principal     (floor)
```

**Returns `0` when:**
- `FundingCloseSnapshot` does not exist (escrow not yet funded).
- Investor has no contribution (`get_contribution(investor) == 0`).

**Error:** emits [`EscrowError::ComputePayoutArithmeticOverflow`] (code 129) if intermediate multiplication overflows `i128`.

**Invariant:** sum of payouts across all investors ≤ `total_principal + coupon`; rounding residual is swept by `sweep_terminal_dust`.

---

## Attestations

### `get_primary_attestation_hash() → Option<BytesN<32>>`

**Storage key:** `DataKey::PrimaryAttestationHash`  
**Signature:** `pub fn get_primary_attestation_hash(env: Env) -> Option<BytesN<32>>`

Returns the single-set 32-byte attestation digest (e.g. SHA-256 of an IPFS CID or document bundle), or `None` when not yet bound.

**Requires initialization:** No  
**Default when absent:** `None`

**Notes:** single-write; once set via `bind_primary_attestation_hash`, the key cannot be overwritten.

---

### `get_attestation_append_log() → Vec<BytesN<32>>`

**Storage key:** `DataKey::AttestationAppendLog`  
**Signature:** `pub fn get_attestation_append_log(env: Env) -> Vec<BytesN<32>>`

Returns the append-only audit chain of 32-byte digests. Empty `Vec` when no entries have been appended.

**Requires initialization:** No  
**Default when absent:** empty `Vec`

**Bounded:** at most `MAX_ATTESTATION_APPEND_ENTRIES` (32) entries. Revocation via `revoke_attestation_digest` does not remove entries; use `is_attestation_revoked(index)` to check revocation status.

---

### `is_attestation_revoked(index: u32) → bool`

**Storage key:** `DataKey::AttestationRevoked(index)`  
**Signature:** `pub fn is_attestation_revoked(env: Env, index: u32) -> bool`

Returns `true` when the append-log entry at `index` has been revoked via `revoke_attestation_digest`.

**Requires initialization:** No  
**Default when absent:** `false` (not revoked)

**Notes:** revocation marks an entry as superseded without removing the original digest (preserves auditability).

---

## Collateral Metadata

### `get_sme_collateral_commitment() → Option<SmeCollateralCommitment>`

**Storage key:** `DataKey::SmeCollateralPledge`  
**Signature:** `pub fn get_sme_collateral_commitment(env: Env) -> Option<SmeCollateralCommitment>`

Returns the SME-reported collateral pledge metadata, or `None` when never recorded.

**Requires initialization:** No  
**Default when absent:** `None`

**Return value:**
- `Some(SmeCollateralCommitment)` with fields:
  - `asset: Symbol` — Off-chain asset symbol.
  - `amount: i128` — Reported collateral amount (always positive).
  - `recorded_at: u64` — Ledger timestamp at time of recording.

**⚠ Record-only:** this is **not** an enforced on-chain asset lock, custody proof, or encumbrance. Risk teams must verify supporting evidence outside this contract.

---

## Allowlist

### `is_allowlist_active() → bool`

**Storage key:** `DataKey::AllowlistActive`  
**Signature:** `pub fn is_allowlist_active(env: Env) -> bool`

Returns `true` when the investor allowlist gate is enabled. When active, only addresses with `is_investor_allowlisted == true` may call `fund` or `fund_with_commitment`.

**Requires initialization:** No  
**Default when absent:** `false`

---

### `is_investor_allowlisted(investor: Address) → bool`

**Storage key:** `DataKey::InvestorAllowlisted(investor)` (persistent)  
**Signature:** `pub fn is_investor_allowlisted(env: Env, investor: Address) -> bool`

Returns `true` when `investor` is permitted to fund when the allowlist gate is active.

**Requires initialization:** No  
**Default when absent:** `false`  
**Storage type:** Persistent (independent TTL per address)

**Notes:** only consulted during funding when `is_allowlist_active() == true`. When the allowlist is inactive, all investors may fund regardless of this flag.

---

## Distributed Principal

### `get_distributed_principal() → i128`

**Storage key:** `DataKey::DistributedPrincipal`  
**Signature:** `pub fn get_distributed_principal(env: Env) -> i128`

Returns the running total of principal already returned to investors via `refund` (cancelled escrow) or to the SME via `withdraw`.

**Requires initialization:** No  
**Default when absent:** `0`

**Usage:** `sweep_terminal_dust` uses this to compute outstanding investor liabilities in cancelled escrows:
```
outstanding = funded_amount - distributed_principal
assert balance - sweep_amt >= outstanding
```

---

## Default/Absent Semantics Summary

| View | Default when key absent | Option vs Default | Requires init |
|------|------------------------|-------------------|---------------|
| `get_escrow` | error (code 20) | N/A — error | Yes |
| `get_version` | `0` | default | No |
| `get_funding_token` | error (code 21) | N/A — error | Yes |
| `get_treasury` | error (code 22) | N/A — error | Yes |
| `get_registry_ref` | `None` | Option | No |
| `get_pending_admin` | `None` | Option | No |
| `get_legal_hold` | `false` | default | No |
| `get_legal_hold_clear_delay` | `0` | default | No |
| `get_legal_hold_clearable_at` | `None` | Option | No |
| `get_funding_deadline` | `None` | Option | No |
| `is_funding_expired` | `false` | default | No |
| `get_min_contribution_floor` | `0` | default | No |
| `get_max_unique_investors_cap` | `None` | Option | No |
| `get_max_per_investor_cap` | `None` | Option | No |
| `has_maturity_lock` | error (via get_escrow) | N/A | Yes |
| `get_funding_close_snapshot` | `None` | Option | No |
| `get_contribution` | `0` | default | No |
| `get_unique_funder_count` | `0` | default | No |
| `get_investor_yield_bps` | base `yield_bps` | default (fallback) | Yes |
| `get_investor_claim_not_before` | `0` | default | No |
| `is_investor_claimed` | `false` | default | No |
| `is_investor_refunded` | `false` | default | No |
| `compute_investor_payout` | `0` | default | No |
| `get_primary_attestation_hash` | `None` | Option | No |
| `get_attestation_append_log` | `[]` | default | No |
| `is_attestation_revoked` | `false` | default | No |
| `get_sme_collateral_commitment` | `None` | Option | No |
| `is_allowlist_active` | `false` | default | No |
| `is_investor_allowlisted` | `false` | default | No |
| `get_distributed_principal` | `0` | default | No |
| `get_escrow_summary` | error (via get_escrow) | N/A | Yes |

**Option vs Default rationale:**

- **Option-returning views** model keys that are semantically absent vs. present: `None` means "not configured" or "not yet reached." Callers must handle both states distinctly (e.g. `None` for `get_registry_ref` means no registry integration, not a zero address).
- **Default-returning views** model keys with a natural zero/false value when absent. The absent state is operationally equivalent to the default; old escrow deployments predating an additive key behave identically to new deployments that wrote the default (see ADR-007).
- **Error-on-absent views** (`get_escrow`, `get_funding_token`, `get_treasury`) model required preconditions: the contract is not usable without these keys and any caller that reaches an absent state has a bug upstream.
