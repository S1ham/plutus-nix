# 🧾 Comprehensive Tutorial: Public Funds Escrow Smart Contract

This comprehensive tutorial covers both the on-chain validator (`PublicFunds.hs`) and off-chain code (`Main.hs`) for a multi-signature escrow smart contract. This system enables secure fund management requiring multiple approvals before release.


---

## 📚 Table of Contents

1. [📦 Imports Overview](#1-imports-overview)
2. [🗃️ Data Structures](#2-data-structures)
3. [🔧 On-Chain Helper Functions](#3-on-chain-helper-functions)
4. [🧠 Core Validator Logic](#4-core-validator-logic)
5. [⚙️ Validator Script Compilation](#5-validator-script-compilation)
6. [🔌 Off-Chain Components](#6-off-chain-components)
7. [🔄 Contract Endpoints](#7-contract-endpoints)
8. [🎮 Emulator Trace](#8-emulator-trace)
9. [🚀 Deployment Workflow](#9-deployment-workflow)
10. [🧪 Testing Strategy](#10-testing-strategy)
11. [✅ Best Practices](#11-best-practices)
12. [📘 Glossary of Terms](#12-glossary-of-terms)

---

## 🏗️ System Architecture

### Complete System Architecture Diagram

<img width="3485" height="3496" alt="deepseek_mermaid_20251222_6c6e6d" src="https://github.com/user-attachments/assets/2fbb2c21-bf55-4e11-9f03-e618b52c6b24" />


### Data Flow Architecture
<img width="9464" height="3428" alt="deepseek_mermaid_20251222_38a973 (1)" src="https://github.com/user-attachments/assets/42cb441b-20d1-4d3b-a8ba-8abb551345b3" />



## 1. 📦 Imports Overview

### On-Chain Imports (`PublicFunds.hs`)

| Module | Purpose |
|--------|---------|
| `Plutus.V2.Ledger.Api` | Core Plutus types: `Validator`, `ScriptContext`, `PubKeyHash` |
| `Plutus.V2.Ledger.Contexts` | Transaction context utilities: `txSignedBy`, `scriptContextTxInfo` |
| `Plutus.V1.Ledger.Interval` | Time interval operations: `contains`, `to`, `from` |
| `PlutusTx` | Template Haskell compilation: `compile`, `unstableMakeIsData` |
| `PlutusTx.Prelude` | Inlinable Plutus functions |
| `Codec.Serialise` | Script serialization to file |
| `Data.ByteString` | Byte string manipulation |
| `Cardano.Api` | Cardano node integration |

### Off-Chain Imports (`Main.hs`)

| Module | Purpose |
|--------|---------|
| `Control.Monad` | Monadic operations |
| `Ledger` | Wallet and address utilities |
| `Ledger.Ada` | ADA handling and value construction |
| `Plutus.Contract` | Contract monad and endpoints |
| `Plutus.Trace.Emulator` | Emulator trace execution |
| `Wallet.Emulator.Wallet` | Mock wallet management |
| `OnChain.PublicFunds` | Import of the on-chain validator |

---

## 2. 🗃️ Data Structures

### On-Chain Data Types

#### `EscrowDatum`
The contract state stored on-chain:

<img width="1295" height="1082" alt="deepseek_mermaid_20251222_95eec5" src="https://github.com/user-attachments/assets/51189c88-4b7a-4f57-808a-39f1645e6d1b" />


| Field | Type | Description |
|-------|------|-------------|
| `edDepositor` | `PubKeyHash` | Fund depositor's public key hash |
| `edBeneficiary` | `PubKeyHash` | Intended recipient's public key hash |
| `edOfficials` | `[PubKeyHash]` | List of m authorized officials |
| `edApprovals` | `[PubKeyHash]` | Accumulated approval signatures |
| `edRequired` | `Integer` | Minimum approvals needed (n ≤ m) |
| `edDeadline` | `POSIXTime` | Deadline for action execution |

#### `EscrowAction`
The redeemer type for state transitions:

| Action | Purpose | Required Signer |
|--------|---------|-----------------|
| `Approve` | Official adds approval | Approving official |
| `Release` | Beneficiary claims funds | Beneficiary |
| `Refund` | Depositor reclaims funds | Depositor |

---

## 3. 🔧 On-Chain Helper Functions

### Signature and Time Validation

```haskell
{-# INLINABLE signedBy #-}
signedBy :: PubKeyHash -> ScriptContext -> Bool
```
Checks if a specific public key signed the transaction.

```haskell
{-# INLINABLE beforeDeadline #-}
beforeDeadline :: POSIXTime -> ScriptContext -> Bool

{-# INLINABLE afterDeadline #-}
afterDeadline :: POSIXTime -> ScriptContext -> Bool
```
Validates transaction timing relative to deadline.

### Approval Logic

```haskell
{-# INLINABLE uniqueApproval #-}
uniqueApproval :: PubKeyHash -> EscrowDatum -> Bool
```
Ensures an official approves only once and is authorized.

---

## 4. 🧠 Core Validator Logic

### `mkValidator` Function
The main validation logic with three execution paths:

#### **Approve Path**
```haskell
Approve ->
    traceIfFalse "deadline passed" (beforeDeadline (edDeadline d) ctx) &&
    traceIfFalse "invalid approver" (uniqueApproval signer d)
```
**Conditions:**
- Must execute before deadline
- Exactly one signer required
- Signer must be an unauthorized official

#### **Release Path**
```haskell
Release ->
    traceIfFalse "deadline passed" (beforeDeadline (edDeadline d) ctx) &&
    traceIfFalse "not enough approvals" (length (edApprovals d) >= edRequired d) &&
    traceIfFalse "beneficiary signature missing" (signedBy (edBeneficiary d) ctx)
```
**Conditions:**
- Must execute before deadline
- Minimum approvals met (n)
- Beneficiary signature required

#### **Refund Path**
```haskell
Refund ->
    traceIfFalse "deadline not reached" (afterDeadline (edDeadline d) ctx) &&
    traceIfFalse "approvals already sufficient" (length (edApprovals d) < edRequired d) &&
    traceIfFalse "depositor signature missing" (signedBy (edDepositor d) ctx)
```
**Conditions:**
- Must execute after deadline
- Insufficient approvals
- Depositor signature required

---

## 5. ⚙️ Validator Script Compilation

### Type Conversion Wrapper

```haskell
{-# INLINABLE mkValidatorUntyped #-}
mkValidatorUntyped :: BuiltinData -> BuiltinData -> BuiltinData -> ()
```
Wraps the typed validator for Plutus on-chain compatibility.

### Script Compilation

```haskell
validator :: Validator
validator =
    mkValidatorScript
        $$(PlutusTx.compile [|| mkValidatorUntyped ||])
```
Compiles the validator to Plutus Core for blockchain deployment.

---

## 6. 🔌 Off-Chain Components

### Contract Schema

```haskell
type EscrowSchema =
        Endpoint "lock" ()
    .\/ Endpoint "approve" ()
    .\/ Endpoint "release" ()
    .\/ Endpoint "refund" ()
```
Defines the available contract endpoints.

### Script Address

```haskell
escrowAddress :: Address
escrowAddress = scriptAddress validator
```
Derives the script address from the compiled validator.

### Datum Creation Helper

```haskell
mkDatum :: Wallet -> Wallet -> [Wallet] -> POSIXTime -> EscrowDatum
mkDatum depositor beneficiary officials deadline =
    EscrowDatum
        { edDepositor   = mockWalletPaymentPubKeyHash depositor
        , edBeneficiary = mockWalletPaymentPubKeyHash beneficiary
        , edOfficials   = fmap mockWalletPaymentPubKeyHash officials
        , edApprovals   = []
        , edRequired    = 2
        , edDeadline    = deadline
        }
```
Creates initial datum with empty approvals list.

---

## 7. 🔄 Contract Endpoints

### Workflow Sequence Diagram

<img width="5349" height="5346" alt="deepseek_mermaid_20251222_ea20a5" src="https://github.com/user-attachments/assets/572c8c60-bc01-42dd-bc18-c1b16bdd7f05" />


### `lock` Endpoint

```haskell
lock :: Contract () EscrowSchema Text ()
lock = do
    let datum = mkDatum (knownWallet 1) (knownWallet 2)
                    [knownWallet 3, knownWallet 4] 20_000
        tx = mustPayToTheScript datum (Ada.lovelaceValueOf 10_000_000)
    void $ submitTxConstraints validator tx
```
**Purpose:** Locks funds in the escrow contract with initial datum.

### `approve` Endpoint

```haskell
approve :: Contract () EscrowSchema Text ()
approve = do
    utxos <- utxosAt escrowAddress
    case utxos of
        [(oref, _)] ->
            void $ submitTxConstraintsSpending validator utxos
                (mustSpendScriptOutput oref (Redeemer $ toBuiltinData Approve))
        _ -> logError @String "No script UTxO"
```
**Purpose:** Official adds their approval to the escrow.

### `release` Endpoint

```haskell
release :: Contract () EscrowSchema Text ()
release = do
    utxos <- utxosAt escrowAddress
    case utxos of
        [(oref, _)] ->
            void $ submitTxConstraintsSpending validator utxos
                (mustSpendScriptOutput oref (Redeemer $ toBuiltinData Release))
        _ -> logError @String "No script UTxO"
```
**Purpose:** Beneficiary claims funds after sufficient approvals.

---

## 8. 🎮 Emulator Trace

### Complete Workflow Simulation

```haskell
trace :: EmulatorTrace ()
trace = do
    -- Wallet 1 locks funds
    h1 <- activateContractWallet (knownWallet 1) lock
    void $ Emulator.waitNSlots 1

    -- Official 3 approves
    h3 <- activateContractWallet (knownWallet 3) approve
    void $ Emulator.waitNSlots 1

    -- Official 4 approves (reaching required 2 approvals)
    h4 <- activateContractWallet (knownWallet 4) approve
    void $ Emulator.waitNSlots 1

    -- Beneficiary releases funds
    h2 <- activateContractWallet (knownWallet 2) release
    void $ Emulator.waitNSlots 1
```

### State Transition Diagram

<img width="1777" height="2969" alt="deepseek_mermaid_20251222_eb2c0d" src="https://github.com/user-attachments/assets/7f6bd011-a0ec-4455-8488-38fe04ac7ac6" />


---

## 9. 🚀 Deployment Workflow

### Deployment Architecture


<img width="10077" height="541" alt="deepseek_mermaid_20251222_fb42ff" src="https://github.com/user-attachments/assets/db7fd65e-b8af-4b83-90b1-772376cb15b3" />


### Step 1: Compile Validator
```bash
cabal build
```

### Step 2: Run Emulator Test
```bash
cabal run public-funds-emulator
```

### Step 3: Serialize Script (Example)
```haskell
-- Save validator to file
saveVal :: IO ()
saveVal = writeValidatorToFile "escrow.plutus" validator
```

### Step 4: Deploy to Testnet
1. Load validator in Plutus Application Backend (PAB)
2. Initialize contract with parameters
3. Fund the contract address
4. Distribute approval credentials to officials

---

## 10. 🧪 Testing Strategy

### Test Architecture

<img width="5395" height="1491" alt="deepseek_mermaid_20251222_56c7dd" src="https://github.com/user-attachments/assets/35891619-7947-4d9d-b1d6-a59a07934fa0" />



### Unit Tests
- **Individual Actions:** Test each redeemer in isolation
- **Time Boundaries:** Transactions at deadline ± 1 slot
- **Signature Validation:** Correct/incorrect signers for each action

### Integration Tests
- **Complete Workflow:** Lock → Approve × n → Release
- **Failed Release:** Attempt release with insufficient approvals
- **Timeout Refund:** Allow deadline to pass, then refund
- **Duplicate Approval:** Attempt same official approving twice

### Edge Cases
- Empty officials list
- Zero required approvals (immediate release)
- All officials required (n = m)
- Single official (n = m = 1)
- Deadline in past at contract creation

---

## 11. ✅ Best Practices

### Security Considerations
1. **Parameter Validation:** Ensure n ≤ m during datum creation
2. **Time Handling:** Use strict inequalities for deadline checks
3. **Signature Enforcement:** Require exact signers for each action
4. **Approval Tracking:** Prevent duplicate approvals through list membership checks

### Gas Optimization
1. **List Operations:** Consider gas costs for large m values
2. **Datum Size:** Keep approval list reasonable (off-chain tracking alternative)
3. **Script Complexity:** Minimize trace operations in production

### User Experience
1. **Clear Error Messages:** Use descriptive `traceIfFalse` messages
2. **Off-Chain Tracking:** Maintain approval state off-chain for UI
3. **Deadline Notifications:** Alert stakeholders approaching deadlines
4. **Multi-Signature Tools:** Provide tools for official coordination

---

## 12. 📘 Glossary of Terms

| Term | Definition |
|------|------------|
| **Escrow** | Funds held by a third party until conditions are met |
| **Multi-Signature** | Authorization requiring multiple signatures |
| **m-of-n** | Governance requiring m approvals from n officials |
| **Datum** | On-chain state data determining contract behavior |
| **Redeemer** | Action specification triggering state transitions |
| **PubKeyHash** | Cryptographic hash of a public key |
| **POSIXTime** | Unix timestamp for deadline specification |
| **Validator** | Smart contract logic validating transactions |
| **ScriptContext** | Transaction context (signatures, time, inputs/outputs) |
| **Emulator Trace** | Simulation environment for testing smart contracts |
| **Contract Endpoint** | Off-chain interface for user interaction |
| **UTxO** | Unspent Transaction Output (blockchain state element) |
| **Bech32** | Human-readable address encoding used in Cardano |

---

## 🎯 Project Requirements Alignment

### Anti-Corruption Features Implemented


<img width="2061" height="2455" alt="deepseek_mermaid_20251222_98a198" src="https://github.com/user-attachments/assets/ac483a64-3ebc-4a81-b45e-3efd291c3143" />


This escrow system provides robust multi-signature governance suitable for:
- **Government Fund Management:** Preventing unauthorized access to public funds
- **Grant Disbursements:** Ensuring proper oversight before release
- **Corporate Approvals:** Requiring multiple signatures for large transactions
- **DAO Fund Allocation:** Transparent, community-governed fund distribution
- **Anti-Corruption Systems:** Creating accountable, auditable fund release mechanisms

The system successfully implements all required features for the CP108 Final Project, providing a complete anti-corruption solution for public fund management through blockchain automation.
