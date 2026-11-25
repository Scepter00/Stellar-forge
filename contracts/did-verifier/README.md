# DID Verifier

The `did-verifier` Soroban contract verifies W3C-style verifiable credentials on Stellar. It parses the credential JSON, checks the proof signature against the issuer's public key using Ed25519 or Secp256k1, and optionally enforces expiration, revocation status, and required claims in the credential subject. Each failed check returns a specific error, and verification succeeds only when every requested check passes.

## Public functions

- `verify_credential(env, credential_json, issuer_public_key, key_type, revocation_registry, revocation_proof, schema)` — Parses the credential, verifies its signature, and then applies the expiration, revocation, and schema checks when those arguments are supplied.
- `verify_ed25519(env, public_key, message, signature)` — Verifies an Ed25519 signature over the credential message with the Soroban host verifier (`crypto.rs`).
- `verify_secp256k1(env, public_key, message, signature)` — Recovers the ECDSA Secp256k1 signer for recovery IDs 0 and 1 and compares it with the issuer's public key (`crypto.rs`).
- `is_revoked(env, registry, credential_id, proof)` — Checks whether a credential ID is revoked in a bitmap registry or by verifying a Merkle proof against the registry root (`revocation.rs`).
- `validate_schema(credential_subject, schema)` — Checks that every required claim exists in the credential subject and matches its expected boolean, string, or numeric value (`schema.rs`).

## Build and test

Run the following commands from the repository root:

```bash
# Build the contract
cargo build --manifest-path contracts/did-verifier/Cargo.toml --target wasm32-unknown-unknown --release

# Run the contract tests
cargo test --manifest-path contracts/did-verifier/Cargo.toml
```
