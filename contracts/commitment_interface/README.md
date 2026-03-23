# Integration Guide: Commitment Interface

This guide documents the interface-only ABI exported by `commitment_interface`.
As of interface version `2`, it mirrors the live `commitment_core` commitment
schema so downstream bindings can detect drift before deployment.

---

## 1. Interface Overview

The `CommitmentInterface` provides a standardized ABI surface for the live
commitment contracts on the Soroban network.

### Metadata & Constants

* **Interface Version:** `2`
* **Event Symbols:** `created`, `settled`, `exited`

### Function Signatures

| Function | Arguments | Return Type | Description |
|:---------|:----------|:------------|:------------|
| `initialize` | `env: Env, admin: Address, nft_contract: Address` | `Result<(), Error>` | Initializes admin and linked NFT contract. |
| `create_commitment` | `env: Env, owner: Address, amount: i128, asset_address: Address, rules: CommitmentRules` | `Result<String, Error>` | Creates a commitment and returns its string id. |
| `get_commitment` | `env: Env, commitment_id: String` | `Result<Commitment, Error>` | Fetches the full live commitment record. |
| `get_owner_commitments` | `env: Env, owner: Address` | `Result<Vec<String>, Error>` | Lists commitment ids owned by an address. |
| `get_total_commitments` | `env: Env` | `Result<u64, Error>` | Reads the global commitment counter. |
| `settle` | `env: Env, commitment_id: String` | `Result<(), Error>` | Settles an expired commitment. |
| `early_exit` | `env: Env, commitment_id: String, caller: Address` | `Result<(), Error>` | Exits an active commitment early. |

### Data Structures (Rust)

```rust
pub struct CommitmentRules {
    pub duration_days: u32,
    pub max_loss_percent: u32,
    pub commitment_type: String,
    pub early_exit_penalty: u32,
    pub min_fee_threshold: i128,
    pub grace_period_days: u32,
}

pub struct Commitment {
    pub commitment_id: String,
    pub owner: Address,
    pub nft_token_id: u32,
    pub rules: CommitmentRules,
    pub amount: i128,
    pub asset_address: Address,
    pub created_at: u64,
    pub expires_at: u64,
    pub current_value: i128,
    pub status: String,
}
```

---

## 2. Frontend Integration (TypeScript)

The TypeScript bindings are located in the root `/bindings` directory.

### Build Workflow

Before use, the definitions must be compiled into JavaScript:

```bash
cd bindings
npm install
npm run build
```

### Usage Example

```typescript
import { Contract, Networks } from '../bindings'; 

const contract = new Contract({
  networkPassphrase: Networks.Testnet, 
  rpcUrl: 'https://soroban-testnet.stellar.org',
});

// Example: Calling get_commitment
async function checkCommitment(commitment_id: string) {
  try {
    const commitment = await contract.get_commitment({ commitment_id });
    console.log('Commitment Details:', commitment);
  } catch (err) {
    console.error("Error fetching commitment:", err);
  }
}
```

---

## 3. Error Reference

Integration errors return a `u32` code mapped to the following definitions:

| Code | Name | Meaning | Recommended Action |
|:-----|:-----|:--------|:-------------------|
| 1 | `NotFound` | Requested resource does not exist. | Verify the commitment id exists via `get_commitment`. |
| 2 | `Unauthorized` | Caller failed an authorization check. | Ensure the transaction is signed by the correct address. |
| 3 | `AlreadyInitialized` | `initialize` called more than once. | Check contract state before initialization. |
| 4+ | `Validation / state errors` | Live contracts may reject invalid amounts, durations, or states. | Follow the core contract error surface for runtime handling. |

---

## 4. Maintenance & Synchronization

To keep the interface aligned with live contracts:

1. Update the interface crate types and signatures.
2. Run the drift checks:

   ```bash
   cargo test -p commitment_interface
   ```

3. Build WASM if bindings need regeneration:
   ```bash
   stellar contract build
   ```

4. Sync bindings if you publish them from this repo:
   ```bash
   stellar contract bindings typescript \
     --wasm target/wasm32v1-none/release/commitment_interface.wasm \
     --output-dir bindings \
     --overwrite
   ```

5. Rebuild Types:
   ```bash
   cd bindings && npm run build
   ```

---

## Additional Resources

- [Soroban Documentation](https://soroban.stellar.org/docs)
- [Stellar CLI Reference](https://developers.stellar.org/docs/tools/developer-tools)

---
