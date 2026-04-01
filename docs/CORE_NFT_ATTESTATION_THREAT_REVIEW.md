# Core NFT Attestation Threat Review

## Core NFT Attestation call graph

This document reviews the end-to-end trust boundaries between:
- `commitment_core`
- `commitment_nft`
- `attestation_engine`

The goal is to make the operational and security assumptions around their call graph explicit for reviewers and integrators.

## Call graph summary

### Write path: `commitment_core -> commitment_nft`

`commitment_core` makes the following outbound calls into `commitment_nft`:

- `create_commitment -> mint`
- `settle -> settle`
- `early_exit -> mark_inactive`

These are state-coupling calls. Core holds the canonical commitment and asset-custody state, while NFT mirrors lifecycle state for ownership and metadata workflows.

### Read path: `attestation_engine -> commitment_core`

`attestation_engine` makes read-oriented calls into `commitment_core`:

- `attest -> commitment_exists -> get_commitment`
- `get_health_metrics -> get_commitment`
- `verify_compliance -> get_commitment`
- `record_drawdown -> get_commitment`
- analytics helpers that derive protocol state from core reads

These calls treat `commitment_core` as the source of truth for commitment existence, status, value, and rules.

## Trust boundaries

### `commitment_core`

- Trusted to custody assets and store the canonical commitment record.
- Trusted to keep its lifecycle state consistent with outbound NFT updates.
- Exposes `get_commitment` without auth so downstream protocol contracts can read state.

### `commitment_nft`

- Trusted to represent the same lifecycle state as core for the linked token.
- Should only accept privileged lifecycle writes from authorized protocol contracts.
- Is downstream of core for create/settle/early-exit state mirroring.

### `attestation_engine`

- Trusted to read, not mutate, core commitment state through the call graph covered here.
- Uses core reads to validate existence and derive compliance outcomes.
- Maintains its own verifier auth boundary; it should not become a write proxy into core.

## Threat review

### 1. Cross-contract auth drift

Threat:
- If `commitment_core` and `commitment_nft` disagree on who may invoke lifecycle functions, the mirrored NFT state can drift or be manipulated independently.

Current observations:
- `commitment_core` identifies itself as the downstream caller on outbound NFT calls.
- `commitment_nft::mint` contains caller-based authorization checks.
- `commitment_nft::settle` is not currently restricted to `commitment_core` according to the live code and repo limitations list.

Reviewer focus:
- Confirm that every privileged NFT lifecycle mutation is bound to the intended protocol caller.
- Treat ABI drift between core and NFT lifecycle calls as a high-priority review item.

### 2. Partial-state risk on outbound NFT calls

Threat:
- `commitment_core` writes local state before invoking `commitment_nft`. If the downstream call fails and rollback is incomplete, users could end up with locked assets or orphaned commitments.

Current observations:
- `create_commitment`, `settle`, and `early_exit` all update core state before the outbound NFT call.
- The design relies on Soroban transaction rollback semantics to keep the whole operation atomic when the downstream contract errors.

Reviewer focus:
- Verify rollback behavior on NFT-call failure for each lifecycle path.
- Prefer deterministic tests for this behavior because it is central to asset safety.

### 3. Reentrancy across contract boundaries

Threat:
- A downstream contract or token call re-enters core before the first invocation completes, causing duplicate state transitions or balance corruption.

Current observations:
- `commitment_core` uses a storage-backed reentrancy guard around `create_commitment`, `settle`, `early_exit`, and `allocate`.
- `commitment_nft` and `attestation_engine` also use their own guards.

Reviewer focus:
- Confirm the guard is set before external calls and cleared on every path.
- Confirm no externally callable function bypasses the guard for asset-moving flows.

### 4. Read-path trust from `attestation_engine`

Threat:
- If `attestation_engine` reads malformed, stale, or unexpected core state, compliance outputs can become misleading.

Current observations:
- `attestation_engine` treats `commitment_core::get_commitment` as canonical.
- The read path is intentionally unauthenticated so protocol contracts can compose.
- `verify_compliance` and related functions derive compliance directly from core status and rules.

Reviewer focus:
- Confirm `get_commitment` remains a pure read interface from the perspective of downstream consumers.
- Confirm downstream readers fail closed when core is uninitialized, missing, or returns malformed data.

### 5. ABI coupling and upgrade drift

Threat:
- Even when access control is correct, argument-shape drift between contracts can break lifecycle transitions at runtime.

Current observations:
- The repository already tracks known limitations around core/NFT interface mismatch risk.
- This call graph is tightly coupled by string-based contract invocation.

Reviewer focus:
- Treat interface drift between core and NFT as a deployment-blocking risk.
- Keep tests around the live call shape, not only around local business logic.

## Recommended review checklist

- Verify `commitment_core` outbound NFT calls use the intended caller identity.
- Verify downstream NFT failures roll back the parent core transaction.
- Verify `attestation_engine` only depends on read-only core interfaces in this call graph.
- Verify lifecycle status transitions remain consistent across core and NFT contracts.
- Verify known ABI and auth gaps are tracked before audit sign-off.

## Current residual risks

- The repo’s known limitations still include auth and ABI concerns in the core/NFT boundary.
- `attestation_engine` correctness is downstream of core state correctness because it reads core as the source of truth.
- Any upgrade or refactor touching `get_commitment`, NFT lifecycle entrypoints, or caller conventions should trigger a fresh cross-contract review.
