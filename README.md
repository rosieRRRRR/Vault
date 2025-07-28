## **L1 Ordinal Vault: 1-Year Lock**

## Specification

This guide provides a proposed developer-focused walkthrough for constructing a trustless Bitcoin Layer 1 time-lock vault for any Ordinal inscription.

### Build Flow Summary

1. **Send the inscription to a clean Taproot key**
2. **Wait for confirmation** and extract the block timestamp
3. **Parse duration** from the Epoch Clock (e.g. 31536000 seconds (1 year))
4. **Compute expiry = block\_timestamp + duration**
5. **Build a Taproot script** committing to expiry
6. **Wrap the script in Taproot**, derive the address
7. **Send the inscription to the vault** (Taproot address)

This guide uses real commands and test vector data to support direct replication. Script and witness validation is included.

- **Type**: Trustless inscription time-lock vault
- **Chain**: Bitcoin mainnet (Taproot required)
- **Enforcement**: Bitcoin consensus via `OP_CHECKLOCKTIMEVERIFY`
- **Model**: Two-transaction construction
- **Reference**: Epoch Clock [Inscription 96733659](https://ordinals.com/inscription/96733659)





```text
// FUNDING TX (Step 1)
// Send inscription to clean Taproot internal key. Confirm block.
// This separates inscription funding from vault creation to ensure accurate timestamp anchoring.

bitcoin-cli gettransaction <funding_txid> true | jq '.blockhash'
bitcoin-cli getblockheader <blockhash> | jq '.time'  → start_timestamp
```

// EPOCH CLOCK JSON: Immutable on-chain reference for a fixed-duration timer (31536000 = 1 year)
```json
{
  "duration_seconds": 31536000
}
```

```text
// EXPIRY
// Calculate absolute expiry using consensus-verified block timestamp.
expiry = start_timestamp + 31536000
```

```text
// SCRIPT
// OP_CLTV-based lock using explicit expiry and pubkey.
// For expiry = 1789536000, pubkey = 02c1e9ce2b263ccff7598ae2485e2a53e91ad5d9ce55dfb1fa4c7068c39e0fdd0c
// Resulting script_hex below is compiled from:
//   <1789536000> OP_CLTV OP_DROP <pubkey> OP_CHECKSIG
script_hex = 040027aa6ab1752102c1e9ce2b263ccff7598ae2485e2a53e91ad5d9ce55dfb1fa4c7068c39e0fdd0cac
```

```text
// TAPTREE ROOT
// For single-leaf script, taproot_root = tapleaf_hash
// taproot_root = SHA256(script || 0xc0) = f13ebe6d5bfbe3ae5a604758b3592e8346ab2008197f1eca7c41b8740da34d23
```

```bash
// TAPROOT ADDRESS
// Wrap the script into a Taproot leaf and derive address using internal pubkey.
createtaprootaddress '{
  "internal_pubkey": "001122...",
  "script_tree": [
    { "script": "<expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP <pubkey> OP_CHECKSIG" }
  ]
}'
```

```text
// VAULT TX (Step 2)
// Send the inscription into the Taproot vault.
// Use `ord` or manual PSBT construction.
// Example:
ord wallet send --destination bc1p... --fee-rate 5.0
```

---

```text
// SPENDING RULE
// Transaction spending this UTXO must have nLockTime >= expiry.
nLockTime >= expiry_timestamp
```

```text
// WITNESS STACK
// Spend path stack from top to bottom: signature, script, control block.
<signature> <leaf_script> <control_block>
```

```text
// CONTROL BLOCK
// Single-leaf script control block (no Merkle path).
// Required structure:
// control_block = <leaf_version (0xc0)> || <internal_pubkey>
```

---

```text
// OPTIONAL: HASH COMMITMENT TO EPOCH CLOCK (AUDIT-ONLY BRANCH)
// Do not place in execution path. Include as separate Taproot leaf if auditing is required.
<raw_json> OP_SHA256 <3f06bba9c8d1aaeb414ea3df77161c0b781c1d3e4a5d46fa925ab617fd0a7da4> OP_EQUALVERIFY
```

---

```text
// PUBLIC KEY GENERATION
// Use Bitcoin Taproot-compatible tools to generate a compressed pubkey.
// Example (using Bitcoin Core descriptor):
// getnewaddress "" "bech32m"
// getaddressinfo <address> | jq -r .pubkey

// AFTER VAULT TX
// Verify inscription landed in correct vault address:
// ord wallet inscriptions | grep bc1p...

// TEST VECTOR (Grouped by phase)

// [inputs]
funding_txid:     e2f6...abcd
block_timestamp: 1748000000
pubkey:           02c1e9ce2b263ccff7598ae2485e2a53e91ad5d9ce55dfb1fa4c7068c39e0fdd0c
expiry:           1789536000
script_hex:       040027aa6ab1752102c1e9ce2b263ccff7598ae2485e2a53e91ad5d9ce55dfb1fa4c7068c39e0fdd0cac
// decoded: <1789536000> CLTV DROP <pubkey> CHECKSIG
leaf_version:     c0
tapleaf_hash:     f13ebe6d5bfbe3ae5a604758b3592e8346ab2008197f1eca7c41b8740da34d23
scriptPubKey:     5120f13ebe6d5bfbe3ae5a604758b3592e8346ab2008197f1eca7c41b8740da34d23
taproot_output:   OP_1 f13ebe6d5bfbe3ae5a604758b3592e8346ab2008197f1eca7c41b8740da34d23 // scriptPubKey = P2TR
address:          tb1p4nmsd8eyv9p7vmhdjjgrksdcy0a8eyqspml9c4p5pq0l4d7eqjzqdqjdy7
```

```text
// CONTROL BLOCK (SINGLE LEAF)
// Minimal control block for single Taproot leaf vault.
// control_block = 0xc0 || <internal_pubkey>
```

```text
// WITNESS SIG TYPE
// Use SIGHASH_DEFAULT (BIP341) unless your use case requires a different hash type.
```



## Trustless Time Model

| Component   | Source                                  |
| ----------- | --------------------------------------- |
| Start time  | Block timestamp of vault-funding tx     |
| Duration    | Parsed off-chain from Epoch Clock JSON  |
| Value       | e.g. 31,536,000 seconds (365 days)      |
| Expiry time | start\_timestamp + duration (hardcoded) |
| Enforcement | `OP_CHECKLOCKTIMEVERIFY` via Taproot    |

---

```text
// FAILURE CONDITIONS
// Funds cannot be spent if:
// - nLockTime < expiry_timestamp
// - Signature is invalid
// - Script does not match output
```

```text
// OUTPUT VALIDATION
// Confirm the Taproot root matches the output scriptPubKey:
// SHA256(taptree_root) == scriptPubKey suffix (32 bytes)
// You can use tools like:
// bitcoin-cli decodescript <script_hex>
// tapleaf --script "<script>" --leaf-version=0xc0
// Or compute explicitly:
// script_root = SHA256(script + leaf_version)
// taproot_output = OP_1 <SHA256(script_root)> => matches scriptPubKey
```

```text
// CONTROL BLOCK STRUCTURE
// For single-leaf script:
control_block = <leaf_version (0xc0)> || <internal_pubkey> || <merkle_path (empty)>
```



### **Further Epoch Clock related reading**

**Epoch Clock**

A deterministic, decentralized time standard for on-chain contract enforcement  

https://github.com/rosieRRRRR/The-Epoch-Clock 




**Epoch Clock BitVM Integration**

This is a proposed integration of the Epoch Clock and BitVM. Trustless, time-aware covenant coordination on Bitcoin Layer 1. Combines a deterministic on-chain Epoch Clock with BitVM’s logic proving framework for enforceable spending conditions using Taproot scripts and PSBTs — all without oracles, soft forks, or new opcodes.

https://github.com/rosieRRRRR/CovenantClock 




**Epoch Clock Rust Iplementation**

This is a proposed integration of the Epoch Clock and BitVM using Rust. Trustless Rust implementation of Bitcoin epoch logic for time-based contract enforcement and BitVM coordination.

https://github.com/rosieRRRRR/Epoch-Clock-Rust-Implementation-




## License
MIT License. No warranty. Use at your own risk.

**Author**
Designed by Rosie, 2025

https://github.com/rosieRRRRR
