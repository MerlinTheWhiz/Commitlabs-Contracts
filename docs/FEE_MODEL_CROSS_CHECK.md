# Fee Model Cross-Check: Documentation vs Implementation

## Executive Summary

This document provides a comprehensive cross-check between the fee model documented in `docs/FEES.md` and the actual implementation in the contracts, specifically focusing on `commitment_core`.

## Current State Analysis

### Documentation Requirements (docs/FEES.md)

#### commitment_core Fee Types

1. **Commitment Creation Fee**
   - Charged when a user creates a new commitment
   - Rate: Basis points (0-10000) of commitment amount
   - Calculation: `fee_amount = (amount * fee_bps) / 10000`
   - Storage: `CreationFeeBps`, `CollectedFees(Address)`
   - User transfers full amount; fee is credited to `CollectedFees(asset)`
   - Commitment created with `amount_locked = amount - creation_fee`

2. **Early Exit Fee**
   - Penalty on early exit; goes to protocol
   - Percentage from commitment rules (stored as protocol revenue)
   - Added to `CollectedFees(asset)`
   - Rest returned to owner

#### Required Functions (per docs/FEES.md)

**Admin Functions:**

- `set_creation_fee_bps(bps)` - Set creation fee rate
- `set_fee_recipient(recipient)` - Set treasury address
- `withdraw_fees(asset_address, amount)` - Withdraw collected fees

**Getter Functions:**

- `get_creation_fee_bps()` - Get current creation fee rate
- `get_fee_recipient()` - Get treasury address
- `get_collected_fees(asset)` - Get collected fees for an asset

#### Required Storage Keys

- `FeeRecipient` - Treasury address for fee withdrawals
- `CreationFeeBps` - Creation fee rate in basis points
- `CollectedFees(Address)` - Per-asset collected fee tracking

### Current Implementation Status

#### ✅ Implemented in shared_utils/fees.rs

- `BPS_SCALE` constant (10000)
- `BPS_MAX` constant (10000)
- `fee_from_bps(amount, bps)` - Fee calculation function
- `net_after_fee_bps(amount, bps)` - Net amount after fee

#### ❌ Missing in commitment_core

1. **Storage Keys:**
   - `FeeRecipient` - NOT DEFINED
   - `CreationFeeBps` - NOT DEFINED
   - `CollectedFees(Address)` - NOT DEFINED

2. **Fee Collection Logic:**
   - `create_commitment` - Does NOT collect creation fee
   - `early_exit` - Calculates penalty but does NOT store it as protocol revenue

3. **Admin Functions:**
   - `set_creation_fee_bps` - NOT IMPLEMENTED
   - `set_fee_recipient` - NOT IMPLEMENTED
   - `withdraw_fees` - NOT IMPLEMENTED

4. **Getter Functions:**
   - `get_creation_fee_bps` - NOT IMPLEMENTED
   - `get_fee_recipient` - NOT IMPLEMENTED
   - `get_collected_fees` - NOT IMPLEMENTED

## Implementation Gap Analysis

### Critical Gaps

1. **No Fee Collection Infrastructure**
   - Storage keys for fee tracking are missing
   - No mechanism to collect creation fees
   - Early exit penalties are not retained as protocol revenue

2. **No Fee Administration**
   - Cannot configure fee rates
   - Cannot set fee recipient
   - Cannot withdraw collected fees

3. **No Fee Transparency**
   - No way to query current fee rates
   - No way to check collected fees
   - No way to verify fee recipient

### Security Considerations

1. **Trust Boundaries**
   - Fee configuration must be admin-only
   - Fee withdrawal must be admin-only
   - Fee recipient must be set before withdrawals
   - Withdrawal amount must not exceed collected fees

2. **Arithmetic Safety**
   - Use `shared_utils::fees::fee_from_bps` for safe calculation
   - Check for overflow when adding to collected fees
   - Check for underflow when subtracting fees from amounts

3. **Reentrancy Protection**
   - Fee collection in `create_commitment` is protected by existing guard
   - Fee withdrawal needs reentrancy protection
   - Early exit fee retention is protected by existing guard

4. **Authorization**
   - All fee admin functions require `require_admin`
   - Fee withdrawal requires `require_auth` on caller

## Implementation Plan

### Phase 1: Storage Infrastructure

Add to `DataKey` enum:

```rust
FeeRecipient,
CreationFeeBps,
CollectedFees(Address),
```

### Phase 2: Fee Collection

1. **Modify `create_commitment`:**
   - Read `CreationFeeBps` from storage
   - Calculate fee using `shared_utils::fees::fee_from_bps`
   - Transfer full amount from user
   - Add fee to `CollectedFees(asset)`
   - Create commitment with `amount - fee`
   - Update TVL with net amount (not including fee)

2. **Modify `early_exit`:**
   - Calculate penalty (already done)
   - Add penalty to `CollectedFees(asset)` instead of just keeping it
   - Transfer only `returned` amount to owner (already done)

### Phase 3: Admin Functions

Implement:

- `set_creation_fee_bps(caller, bps)` - Validate bps <= 10000
- `set_fee_recipient(caller, recipient)` - Validate not zero address
- `withdraw_fees(caller, asset, amount)` - Check recipient set, sufficient fees

### Phase 4: Getter Functions

Implement:

- `get_creation_fee_bps()` - Return 0 if not set
- `get_fee_recipient()` - Return Option<Address>
- `get_collected_fees(asset)` - Return 0 if not set

### Phase 5: Testing

Add tests for:

- Fee calculation accuracy
- Fee collection on creation
- Fee retention on early exit
- Fee withdrawal with various scenarios
- Admin-only access control
- Edge cases (zero fees, max fees, insufficient collected fees)

## Alignment with attestation_engine

The attestation_engine contract already implements a similar fee model:

- `set_attestation_fee` - Sets fee amount and asset
- `set_fee_recipient` - Sets treasury
- `withdraw_fees` - Withdraws collected fees
- Fee collection in `attest` function
- Proper storage keys and getters

The commitment_core implementation should follow the same patterns for consistency.

## Success Criteria

1. ✅ All storage keys defined
2. ✅ Fee collection on `create_commitment`
3. ✅ Fee retention on `early_exit`
4. ✅ All admin functions implemented with proper auth
5. ✅ All getter functions implemented
6. ✅ Comprehensive test coverage
7. ✅ Rustdoc comments on all public functions
8. ✅ Security assumptions documented
9. ✅ Arithmetic safety verified
10. ✅ Reentrancy protection verified
