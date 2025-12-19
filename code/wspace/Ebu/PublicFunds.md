# 🧾 Detailed Tutorial: Understanding and Using `PublicFunds.hs`

This tutorial covers the `PublicFunds.hs` module, a sophisticated escrow smart contract for multi-signature governance. This contract enables secure fund management with approval-based release mechanisms and timeout-based refunds.

---

## 📚 Table of Contents

1. [📦 Imports Overview](#1-imports-overview)
2. [🗃️ Data Structures](#2-data-structures)
3. [🔧 Helper Functions](#3-helper-functions)
4. [🧠 Core Validator Logic](#4-core-validator-logic)
5. [⚙️ Validator Script Compilation](#5-validator-script-compilation)
6. [🔧 Deployment Utilities](#6-deployment-utilities)
7. [🧪 Practical Usage Example](#7-practical-usage-example)
8. [🧷 Testing Strategy](#8-testing-strategy)
9. [✅ Best Practices](#9-best-practices)
10. [📘 Glossary of Terms](#10-glossary-of-terms)

---

## 1. 📦 Imports Overview

### Plutus API Modules

* **Plutus.V2.Ledger.Api:**
  Provides fundamental types such as `POSIXTime`, `PubKeyHash`, `ScriptContext`, and validator construction functions.

* **Plutus.V2.Ledger.Contexts:**
  Contains utility functions for transaction context validation (`txSignedBy`, `scriptContextTxInfo`).

* **Plutus.V1.Ledger.Interval:**
  Supplies interval functions (`contains`, `to`, `from`) for time-based validations.

### Compilation & Serialization

* **PlutusTx:**
  Enables script compilation (`compile`, `unstableMakeIsData`) and data conversion (`unsafeFromBuiltinData`).

* **PlutusTx.Prelude:**
  Basic Plutus scripting functions including `traceError` and `traceIfFalse`.

### Cardano API & Utilities

* **Cardano.Api / Cardano.Api.Shelley:**
  Provides serialization utilities for converting Plutus scripts to Cardano's format.

* **Codec.Serialise & Data.ByteString:**
  Used for serializing validator scripts to file.

* **Data.Text & Prelude:**
  Standard Haskell libraries for text manipulation and basic operations.

---

## 2. 🗃️ Data Structures

### `EscrowDatum`

Defines the contract's state with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `edDepositor` | `PubKeyHash` | The public key hash of the fund depositor |
| `edBeneficiary` | `PubKeyHash` | The public key hash of the intended recipient |
| `edOfficials` | `[PubKeyHash]` | List of m officials authorized to approve |
| `edApprovals` | `[PubKeyHash]` | Collection of approvals received so far |
| `edRequired` | `Integer` | Number of approvals (n) required for release |
| `edDeadline` | `POSIXTime` | Deadline for approvals and release |

### `EscrowAction`

Defines the redeemer actions:

| Action | Purpose |
|--------|---------|
| `Approve` | Official adds their approval to the datum |
| `Release` | Beneficiary claims funds after sufficient approvals |
| `Refund` | Depositor reclaims funds if deadline passes with insufficient approvals |

---

## 3. 🔧 Helper Functions

### `signedBy`

```haskell
{-# INLINABLE signedBy #-}
signedBy :: PubKeyHash -> ScriptContext -> Bool
```
Checks if a specific public key hash signed the transaction.

### `beforeDeadline` & `afterDeadline`

```haskell
{-# INLINABLE beforeDeadline #-}
beforeDeadline :: POSIXTime -> ScriptContext -> Bool

{-# INLINABLE afterDeadline #-}
afterDeadline :: POSIXTime -> ScriptContext -> Bool
```
Validate whether the transaction occurs before or after the specified deadline.

### `uniqueApproval`

```haskell
{-# INLINABLE uniqueApproval #-}
uniqueApproval :: PubKeyHash -> EscrowDatum -> Bool
```
Ensures an official hasn't already approved and is in the officials list.

---

## 4. 🧠 Core Validator Logic

### `mkValidator`

Main validation logic implementing three distinct workflows:

#### **Approve Action**
- Must be executed before deadline
- Requires exactly one signer
- Signer must be an official who hasn't approved yet

#### **Release Action**
- Must be executed before deadline
- Requires at least `n` approvals (where `n = edRequired`)
- Requires beneficiary's signature

#### **Refund Action**
- Must be executed after deadline
- Requires fewer than `n` approvals
- Requires depositor's signature

---

## 5. ⚙️ Validator Script Compilation

### `mkValidatorUntyped`

Wraps the typed validator for compatibility with Plutus on-chain scripts using `BuiltinData`.

### `validator`

Compiles the validator into a Plutus Core script ready for blockchain deployment:

```haskell
validator :: Validator
validator =
    mkValidatorScript
        $$(PlutusTx.compile [|| mkValidatorUntyped ||])
```

---

## 6. 🔧 Deployment Utilities

*(Note: The provided code snippet doesn't include deployment utilities, but typical deployment would involve:)*

- **Script Serialization:** Convert validator to CBOR format
- **Address Generation:** Create Bech32 address for the script
- **Datum Construction:** Helper functions to create `EscrowDatum`
- **Redeemer Construction:** Helper functions for `EscrowAction`

---

## 7. 🧪 Practical Usage Example

```haskell
-- Example datum creation
exampleDatum :: EscrowDatum
exampleDatum = EscrowDatum
    { edDepositor = "pubkey1..."
    , edBeneficiary = "pubkey2..."
    , edOfficials = ["official1...", "official2...", "official3..."]
    , edApprovals = []
    , edRequired = 2
    , edDeadline = 1700000000000
    }

-- Typical workflow:
-- 1. Deploy contract with initial datum (empty approvals)
-- 2. Officials submit "Approve" transactions
-- 3. After reaching required approvals, beneficiary can "Release"
-- 4. If deadline passes with insufficient approvals, depositor can "Refund"
```

---

## 8. 🧷 Testing Strategy

### Critical Test Scenarios
- **Approval Flow:** Test individual official approvals and duplicate approval prevention
- **Threshold Validation:** Test release with exactly `n-1`, `n`, and `n+1` approvals
- **Time Boundaries:** Test transactions immediately before and after deadline
- **Signature Validation:** Test all actions with correct/incorrect signatures
- **Edge Cases:** Empty officials list, zero required approvals, etc.

---

## 9. ✅ Best Practices

### Security Considerations
- **Signature Validation:** Always verify signers match expected parties
- **Time Validation:** Use strict before/after checks without tolerance
- **Approval Tracking:** Prevent duplicate approvals from same official
- **Threshold Logic:** Ensure `n ≤ m` (required ≤ total officials)

### Development Practices
- Use `traceIfFalse` with descriptive messages for debugging
- Validate all preconditions before processing
- Consider gas costs for on-chain list operations
- Test with various (m, n) combinations

---

## 10. 📘 Glossary of Terms

| Term | Definition |
|------|------------|
| **Escrow** | A financial arrangement where funds are held by a third party until conditions are met |
| **Multi-signature** | Requiring multiple parties to authorize a transaction |
| **Datum** | On-chain state data that determines contract behavior |
| **Redeemer** | Action specification that triggers state transitions |
| **PubKeyHash** | Cryptographic hash of a public key used for authorization |
| **POSIXTime** | Unix timestamp representation for deadlines |
| **m-of-n** | Governance model requiring m approvals out of n officials |
| **Validator** | Smart contract logic that validates transaction legitimacy |
| **ScriptContext** | Transaction context including signatures, time range, etc. |
| **BuiltinData** | Raw Plutus data type for on-chain serialization |

---

## 🔄 State Transition Diagram

```
Initial State
    ├── [Approve] → (Add approval if before deadline and official)
    ├── [Release] → (Transfer to beneficiary if approvals ≥ n and before deadline)
    └── [Refund] → (Return to depositor if after deadline and approvals < n)
```

This contract implements a robust escrow system suitable for treasury management, grant distributions, or any scenario requiring multi-party governance over fund release.
