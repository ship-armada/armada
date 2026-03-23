# Transaction Signing Architecture

## Overview

Armada uses signature-derived keys to enable shielded transactions without requiring users to manage separate seed phrases or install new wallet extensions. Users sign a structured message once with their existing wallet (MetaMask, etc.), and this signature becomes the entropy source for deriving all privacy-related keys.

This approach prioritizes integration ease and minimal UX friction while accepting phishing as the primary residual risk.

## Design Principles

### Axiom 1: Canonical Secret

The `root_secret` is the authority. Wallet signatures are entropy sources, not identities.

- `signature_bytes â†’ HKDF â†’ root_secret`
- Users persist/export `root_secret`, not "re-sign later"
- Re-signing is unreliable convenience, not a recovery path (see Session Management)

This sidesteps wallet determinism debates entirely. Different wallets, different versions, hardware vs software â€” none of it matters once the root secret is derived and backed up.

### Axiom 2: One Shielded Identity, Many Rails

Users have one shielded identity across all integrators. Fragmenting keys per integrator would kill the anonymity set and undermine the privacy flywheel.

- Shared anonymity set is sacred
- Integrators are access paths, not identities
- Domain-separated sub-derivation allows integrator-specific scoping without fragmenting identity

### Axiom 3: Spending Keys Never Leave User Context

This is a hard invariant, not a best practice:

- Not passed to integrators
- Not cached beyond immediate use
- Not logged
- Not abstracted behind "helpful" SDK helpers

Integrators only ever receive viewing keys. If an integrator requests a spending key, that's a red flag.

## Enrollment Flow

### User Experience

Externally, enrollment feels like: "Connect wallet â†’ Generate private balance."

One wallet signature, one forced backup, done. The backup is the recovery path — not re-signing. The enrollment signature includes a timestamp (`issuedAt`) that makes it inherently non-reproducible: a second signature produces different bytes, a different root secret, and a different shielded identity. This is by design. Users must understand during enrollment that losing their recovery secret means losing access.

### Technical Flow

```
1. User connects wallet (MetaMask, WalletConnect, etc.)

2. SDK presents EIP-712 typed data for signing:
   - domain.name: "Armada Protocol"
   - domain.chainId: <current chain> (informational only)
   - domain.verifyingContract: <Armada registry address>
   - message.purpose: "Generate privacy keys (NOT a transaction)"
   - message.issuedAt: <timestamp>
   - message.version: "1"
   
   Note: The enrollment signature does not bind the canonical identity to a 
   single chain. chainId is included for wallet UX only. Chain specificity 
   is introduced in subkey derivation, not at the root level.

   Note: issuedAt MUST use millisecond precision (Unix epoch, milliseconds)
   and the SDK MUST enforce monotonicity (never reuse a previous timestamp).
   This ensures payload uniqueness even if a future wallet implements 
   deterministic signing (RFC 6979), where identical payloads produce 
   identical signatures.

3. User signs â†’ signature_bytes (65 bytes)

4. SDK derives root secret:
   root_secret = HKDF(
     salt: "armada-v1",
     IKM: signature_bytes,
     info: "root",
     length: 32
   )

5. SDK displays anti-phish checksum (see below)

6. SDK forces export of root_secret as recovery backup
   - Download as file, OR
   - Copy to clipboard with confirmation, OR
   - Display as QR code
   
7. User confirms backup â†’ enrollment complete
```

### Why EIP-712?

Using EIP-712 typed data instead of `personal_sign`:

- Wallets display structured fields, not opaque hex
- Domain separation prevents cross-site signature reuse
- `verifyingContract` binds to a specific deployment
- Clearer to users that this is not a transaction

### Why Signature-Derived, Not Mnemonic-Based?

Railgun wallets use BIP-39 mnemonics with BIP-32 hierarchical derivation — the standard crypto wallet pattern. Armada deliberately departs from this for integrator UX reasons:

- **No second seed phrase.** Users already have a wallet. Asking them to generate, back up, and manage a separate 12/24-word mnemonic for their shielded identity adds friction that kills integrator adoption. "Connect wallet → sign → done" is the product requirement.
- **Existing wallet as entropy source.** An ECDSA signature over structured data provides ≥128 bits of entropy (from the signing nonce `k`), which is sufficient for HKDF input. The entropy quality is different from BIP-39 (structured vs uniform), but HKDF is specifically designed to extract uniform key material from non-uniform sources.
- **Tradeoff acknowledged.** The mnemonic approach gives higher raw entropy (128–256 bits, uniform) and well-studied BIP-32 derivation paths. The signature approach gives lower UX friction at the cost of phishing as the primary threat vector and `issuedAt`-dependent non-reproducibility. This is an explicit product decision, not a cryptographic preference.

Any future SDK that needs to interop with mnemonic-derived Railgun wallets must support both derivation paths. This spec covers signature-derived keys only.

## Key Derivation Hierarchy

### HKDF Semantics

We use HKDF-SHA-256 per RFC 5869 (i.e., HKDF instantiated with HMAC-SHA-256). Not HKDF-SHA-512 or other variants — the hash function is pinned, not implementation-choice:

```
PRK = HKDF-Extract(salt, IKM)
OKM = HKDF-Expand(PRK, info, L)
```

For root derivation:
- `IKM = signature_bytes`
- `salt = "armada-v1"` (protocol-wide constant, versioned)
- `info = "root"`
- `L = 32` (output length in bytes)

Note on salt: the static salt means HKDF-Extract is identical across all users — per-user entropy comes entirely from the IKM. This is acceptable per RFC 5869 ("if not available, [salt] is set to a string of HashLen zeros"), but means the salt provides no additional protection if the entropy source is ever weakened. A future version could include the wallet address in the salt (`"armada-v1:" + wallet_address`) for defense in depth without changing the UX.

For subkey derivation, `root_secret` becomes the IKM and `info` varies per key type.

### Derivation Tree

All keys derive from `root_secret` using HKDF with explicit info strings:

```
root_secret
â”‚
â”œâ”€â”€ spending_key = HKDF(root_secret, info="spend:v1")
â”‚
â”œâ”€â”€ viewing_key = HKDF(root_secret, info="view:v1")
â”‚
â””â”€â”€ integrator-scoped keys (optional)
    â”œâ”€â”€ HKDF(root_secret, info="integrator:borderless:v1")
    â”œâ”€â”€ HKDF(root_secret, info="integrator:foo:v1")
    â””â”€â”€ ...
```

Integrator-scoped derivation provides isolation and revocation without fragmenting the user's anonymity set. It must never be used to create separate shielded identities.

### Bytes to Field Element Mapping

This section is normative for Armada v1 implementations, subject to confirmation that inherited Railgun witness handling matches these assumptions. Every implementation must match this specification exactly once the upstream assumptions are confirmed.

This is the boundary where cross-implementation drift and silent truncation bugs actually occur — the Privacy Pools SDK bug was a bytes-to-scalar conversion failure, not a KDF failure. But there is a subtlety the earlier iterations of this spec missed: Railgun's key architecture involves multiple algebraic structures, and spending keys and viewing keys may require different modular reductions.

**The three relevant algebraic orders:**

Railgun's circuits use Groth16 over BN254 (alt_bn128). All circuit signals are elements of BN254's scalar field F_r. However, inside those circuits, elliptic curve operations happen on Baby Jubjub — a twisted Edwards curve defined over F_r with its own, smaller, prime-order subgroup. Railgun's documentation states that viewing keys use Ed25519 (EdDSA), an entirely separate curve — though the exact implementation conventions (key clamping, reduction modulus) have not been independently verified from this spec's perspective.

```
BN254 scalar field order (all circuit signals live here):
  r = 21888242871839275222246405745257275088548364400416034343698204186575808495617
    = 0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000001
    (254 bits)

Baby Jubjub prime-order subgroup (spending key scalars are meaningful here):
  l = 2736030358979909402780800718157159386076813972158567259200215660948447373041
    = 0x060c89ce5c263405370a08b6d0302b0bab3eedb83920ee0a677297dc392126f1
    (251 bits)
  Curve group order = 8 * l (cofactor 8)
  Relationship: l divides into r approximately 8 times (floor(r/l) = 7)

Ed25519 group order (viewing key scalars, if Railgun's Ed25519 scheme is inherited):
  l_ed = 7237005577332262213973186563042994240857116359379907606001950938285454250989
    (253 bits, entirely unrelated to r)
```

These are three different moduli. A single "reduce mod r" is not sufficient to specify the correct conversion for all key types.

**Why this matters:**

For spending keys: Baby Jubjub scalars `k` and `k + l` produce the same public key and the same nullifiers. Since `r / l ≈ 8`, there are ~8 distinct F_r values that map to each Baby Jubjub point. Reducing mod `r` produces a valid circuit input, but the algebraically meaningful canonical range for spending key uniqueness is `[0, l)`, not `[0, r)`. Whether Railgun's existing key derivation reduces to `[0, l)` or `[0, r)` determines whether Armada should match. If Railgun reduces to `[0, r)` and relies on the circuit to handle equivalence, mod-r is correct. If Railgun reduces to `[0, l)`, mod-r would produce keys that work but aren't in canonical form.

For viewing keys: if Armada inherits Railgun's Ed25519 viewing key infrastructure for scanning and decryption, then viewing key scalars must be valid Ed25519 private keys — meaning reduction mod `l_ed`, not mod `r`. Reducing a 32-byte HKDF output mod `r` and handing it to Ed25519 code would produce a key that is not in the correct range. Alternatively, if Armada defines its own viewing key format that operates purely within the BN254 circuit (e.g., for in-circuit decryption using Poseidon or similar), then mod-r may be correct. The spec must state which path Armada takes.

**Spending key conversion (pending Andrew confirmation):**

The target algorithm for spending key derivation:

```
1. Start with raw HKDF output: 32 bytes (Uint8Array)
2. Interpret as unsigned integer, BIG-ENDIAN (most significant byte first)
3. Reduce to the correct modulus:
   
   Option A (if Railgun reduces mod r):
     spending_scalar = bigint_from_bytes_be(hkdf_output) % r
     Range: [0, r). Valid circuit input. ~8 F_r values per Baby Jubjub point.
   
   Option B (if Railgun reduces mod l for canonical form):
     spending_scalar = bigint_from_bytes_be(hkdf_output) % l
     Range: [0, l). Canonical Baby Jubjub scalar. One value per point.
```

Andrew must determine which option matches Railgun's existing key derivation. The choice affects whether keys derived by Armada are interoperable with keys derived by other Railgun wallets (which use mnemonic-based BIP-32 derivation, not signature-based HKDF).

For Option A, the modular reduction bias is negligible: 2^256 / r ≈ 5.3, bias on the order of 2^{-254}. Rejection sampling is not required.

For Option B, the bias is larger but still negligible: 2^256 / l ≈ 4.2 × 10^25, but since l is 251 bits and the input is 256 bits, the bias is on the order of 2^{-246}. Still cryptographically irrelevant.

**Viewing key conversion (architecture decision required):**

Armada must choose one of two paths:

- **Path 1: Inherit Railgun's Ed25519 viewing keys.** Viewing key derivation must follow Ed25519 conventions: reduce mod `l_ed`, apply Ed25519 key clamping if required. The HKDF output for viewing keys is converted differently from spending keys. This enables interoperability with Railgun scanning infrastructure.

- **Path 2: Define Armada-native viewing keys.** Viewing keys are BN254 F_r elements (reduced mod `r`) used for in-circuit operations only. This is simpler but breaks compatibility with Railgun's existing viewing key sharing and scanning.

The derivation tree currently uses a single HKDF-from-root pattern for both key types. This is fine — the divergence happens at the bytes-to-scalar step, not at the HKDF step. But the conversion function must be key-type-aware.

This decision is not yet made. The spec will be updated once the architecture is settled.

**What is pinned down regardless of the above choices:**

- **Byte order**: Big-endian. Not little-endian, not platform-native. The first byte of HKDF output is the most significant byte of the integer.
- **Integer type**: Arbitrary-precision (`BigInt` in JS, `uint256` in Solidity). See IC-1.
- **Reduction**: Modular reduction (not truncation, not masking, not "take the low N bits") against the appropriate modulus for each key type.
- **Output range**: `[0, modulus)` where modulus is `r`, `l`, or `l_ed` depending on key type and architecture decisions above.

**Boundary test vectors:**

These vectors test the bytes-to-integer-to-reduced-scalar pipeline. They are specified for BN254 mod-r reduction. If the final spending key modulus is `l` instead of `r`, equivalent vectors must be computed for mod-l and published before the spec is complete.

These are concrete and executable. Any implementation that produces different outputs for mod-r reduction is broken.

```
Vector 1: All zeros (identity element)
  Input:    0x0000000000000000000000000000000000000000000000000000000000000000
  Scalar:   0

Vector 2: r exactly (reduces to zero)
  Input:    0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000001
  Scalar:   0

Vector 3: r - 1 (maximum valid field element, passes through unchanged)
  Input:    0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000000
  Scalar:   0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000000

Vector 4: r + 1 (just above field order, reduces to 1)
  Input:    0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000002
  Scalar:   1

Vector 5: All 0xFF (maximum 256-bit value, exercises full reduction)
  Input:    0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
  Scalar:   0x0e0a77c19a07df2f666ea36f7879462e36fc76959f60cd29ac96341c4ffffffa

Vector 6: 2r + 42 (non-trivial reduction to small value)
  Input:    0x60c89ce5c263405370a08b6d0302b0ba5067d090f372e12287c3eb27e000002c
  Scalar:   42
```

These boundary vectors specifically test the bytes→scalar conversion. They are separate from and complementary to the IC-4 end-to-end derivation vectors. An implementation can pass IC-4's happy path while silently mishandling edge-case byte values — these vectors close that gap.

**Zero-scalar safety:**

Zero is a valid member of F_r (and of any subgroup), and the conversion algorithm can produce it (e.g., if HKDF output happens to equal the modulus exactly). However, zero is degenerate as a secret key — a zero spending key maps to the Baby Jubjub identity point, producing predictable nullifiers, and a zero viewing key provides no confidentiality. Whether Railgun's circuits reject zero witness inputs or treat them as valid is an implementation detail that must be checked. If any derived secret is required to be nonzero, the SDK should re-derive with an incremented info string (e.g., `"spend:v1:1"`) rather than silently accepting zero. This is astronomically unlikely in practice (probability on the order of 2^{-254}) but the spec should not leave it ambiguous. Note: the re-derive-with-incremented-info pattern introduces a second derivation branch that must be mirrored everywhere the same secret is reconstructed; Andrew should confirm this is compatible with Railgun wallet expectations before committing to a specific nonzero-enforcement mechanism.

**Scope of application:**

This conversion applies at the point where HKDF byte output becomes a key used in cryptographic operations (circuit witness inputs, Ed25519 signing, etc.). `root_secret` itself is stored and exported as raw 32 bytes — it does not need field reduction because it never enters a circuit or signing operation directly.

### Versioning

The `:v1` suffix in info strings provides upgrade paths. If we need to change derivation logic:

- New users get `v2` keys
- Existing users can migrate (re-derive from same root)
- Old keys remain valid during transition

This covers intentional spec changes. For implementation bugs where the spec was correct but the SDK diverged, see IC-5 (Bug-Triggered Migration) in the Implementation Constraints section.

### Per-Chain / Per-Asset Subkeys (Future)

If needed, the hierarchy can extend:

```
spending_key
â”œâ”€â”€ HKDF(spending_key, info="chain:1:asset:USDC")
â”œâ”€â”€ HKDF(spending_key, info="chain:8453:asset:USDC")
â””â”€â”€ ...
```

This is not required for v1 but the structure supports it.

## Implementation Constraints

These constraints are non-negotiable. They exist because the gap between "spec says 256-bit key" and "implementation silently produces a 53-bit key" is where real systems break. The Privacy Pools SDK disclosed a bug in this class in February 2025: `bytesToNumber` (JavaScript IEEE 754 float, ~53-bit precision) was used instead of `bytesToBigInt` in master secret derivation. Per the team's disclosure, this reduced effective key entropy from 256-bit to ~53-bit, making targeted brute-force computationally feasible. See References for the primary source.

Armada's derivation path has the same class of exposure anywhere bytes become scalars. These constraints close that gap.

### IC-1: No JavaScript `Number` in Derivation Paths

All byte-to-scalar conversions MUST use arbitrary-precision integers (`BigInt` in JavaScript, `uint256` in Solidity). JavaScript `Number` MUST NOT appear anywhere between `signature_bytes` input and final key output. This includes intermediate steps, logging, assertions, and test helpers.

Lint rule: any file touching key derivation should fail CI if it imports or calls `bytesToNumber`, `parseInt` on hex key material, or `Number()` on byte buffers.

### IC-2: Entropy Floor Canary

After derivation, the SDK SHOULD assert:

```javascript
if (rootSecret < (1n << 64n)) {
  throw new Error("root_secret entropy below safety floor — possible truncation bug");
}
```

This is a diagnostic canary, not a proof of correctness. It catches the specific class of float-truncation bug that hit Privacy Pools (where ~53-bit values were produced instead of ~256-bit). But a broken implementation can produce garbage above 2^64 and pass this check. The actual defenses against derivation bugs are IC-4 (test vectors) and the normative Bytes to Field Element Mapping section — those do the real work. This assertion is a cheap early-warning tripwire, not a substitute.

The same assertion applies to `spending_key` and `viewing_key` after their respective HKDF derivations.

### IC-3: Opaque Byte Passthrough to HKDF

Raw `signature_bytes` (all 65 bytes: r ‖ s ‖ v) are passed to HKDF as an opaque `Uint8Array`. The SDK MUST NOT parse the signature into (r, s, v) components and reconstruct them through numeric types before HKDF input. Any intermediate representation that touches `Number` is a truncation vector.

```
// CORRECT: pass raw bytes
const rootSecret = hkdf(signatureBytes, salt, info, length);

// WRONG: parse components through Number
const r = Number("0x" + sig.slice(0, 64));  // truncated
const s = Number("0x" + sig.slice(64, 128)); // truncated
```

### IC-4: Published End-to-End Test Vectors

**Status: Provisional.** The boundary test vectors in the Bytes to Field Element Mapping section above are concrete and executable now. The end-to-end vectors below cannot be finalized until the SDK implementation exists, because they require a specific `signature_bytes` input to be run through the actual HKDF + field mapping pipeline. This section defines the requirement and format; the spec is not considered complete until concrete values replace the placeholders.

At least one full-chain test vector MUST be published alongside this spec and validated in CI:

```
Input:
  signature_bytes: 0x[known 65-byte hex value]
  
Expected outputs:
  root_secret:  0x[32-byte hex]
  spending_key: 0x[32-byte hex]  (after field mapping)
  viewing_key:  0x[32-byte hex]  (after field mapping)
  checksum:     [4-byte hex]
```

Any implementation that produces different outputs for the canonical test input is broken. This is the single cheapest defense against silent derivation bugs — if Privacy Pools had shipped test vectors, `bytesToNumber` would have failed them on the first run.

Concrete vectors will be published in `TX_SIGNING_VECTORS.json` once the SDK implementation is finalized. Until then, the boundary vectors above provide partial coverage of the most dangerous conversion boundary.

### IC-5: Bug-Triggered Migration Capability

The versioning scheme (`:v1` → `:v2` info strings) covers intentional spec upgrades. It does not cover the case where the spec was correct but an implementation diverged — which is what happened with Privacy Pools' SDK.

The system MUST support derivation integrity migration as a capability:

1. **Detection**: Identify whether a user's existing keys were derived with a known-buggy implementation. Approaches include comparing stored keys against correct test vector outputs, checking the IC-2 entropy floor, or flagging all keys derived before a known patch date.
2. **Re-derivation**: Produce correct keys from the same `root_secret`. This is straightforward because `root_secret` is stored/exported as raw bytes (Axiom 1) and never re-derived through the buggy path.
3. **On-chain re-keying**: Migrate shielded note ownership from old (weak) keys to corrected keys without moving funds out of the pool.

The specific mechanism for step 3 depends on Railgun's circuit constraints and Armada's transaction model. Privacy Pools demonstrated one pattern — zero-value withdrawal proofs to nullify old notes, batched via multicall, with replacement notes keyed to the corrected derivation. Whether Railgun's inherited circuits support zero-value withdrawals, or whether re-keying requires a different approach (e.g., self-transfers within the shielded pool), is an implementation question that must be validated against the actual circuit spec before committing to a mechanism.

The requirement is the capability. The mechanism is TBD pending circuit validation.

## Anti-Phish Checksum

A short, human-recognizable fingerprint derived from the root secret:

```
checksum = first_4_bytes(SHA256(root_secret || "armada-check"))
```

Display format options:
- 8 hex characters: `a3f2 91c8`
- Emoji pair: ðŸŒŠðŸ”
- Short wordlist: `ocean-lock`

### When Displayed

- Immediately after enrollment (prominent)
- In "Security / Privacy Settings" (always accessible)
- Optionally on transaction confirmation screens

### User Guidance

"This is your privacy fingerprint. If it ever changes unexpectedly, assume your keys are compromised and contact support."

The checksum doesn't prevent phishing â€” it makes successful phishing *detectable*.

## Session Management

### Daily Use

After enrollment, users can restore their session by:

1. **Pasting recovery secret** — the canonical recovery path. Always works regardless of wallet state, device, or browser.
2. **Encrypted local storage** (optional, device-specific) — "remember me" convenience, see below.
3. **Re-signing** — unreliable. Because enrollment includes `issuedAt`, re-signing produces a different signature and therefore a different identity. Re-signing can only match the original if the wallet produces deterministic signatures for identical payloads AND the SDK replays the exact original `issuedAt` value, which requires the SDK to have stored it. UX copy must never frame re-signing as a dependable recovery path. If the SDK offers re-signing at all, it must include mismatch detection (compare derived checksum against stored checksum) and a clear failure message directing users to paste their recovery secret.

### Encrypted Local Storage (Optional)

For "remember me on this device" functionality:

```
1. User opts in to local storage
2. SDK prompts for passcode (6+ chars)
3. root_secret encrypted with:
   encryption_key = PBKDF2(passcode, salt=random, iterations=â‰¥100000)
   (or argon2id where WebCrypto support is available)
4. Encrypted blob stored in localStorage/IndexedDB
5. To unlock: user enters passcode â†’ decrypt â†’ root_secret in memory
```

This is opt-in. Default behavior is stateless (paste secret or re-sign).

## Integrator Access Model

### What Integrators Receive

Integrators receive **viewing keys only**, scoped and time-limited:

```javascript
// SDK returns scoped viewing key
const viewKey = await armada.getViewingKey({
  scope: {
    chains: [1, 8453],
    assets: ["USDC"],
  },
  ttl: 24 * 60 * 60, // 24 hours default
});
```

**TTL enforcement model:** TTL is enforced client-side — the SDK refuses to use an expired viewing key. This is not a cryptographic guarantee. The viewing key itself is a derived secret; anyone who obtains it can use it indefinitely by ignoring the SDK's TTL check. TTL is a policy boundary, not a cryptographic one. On-chain viewing key revocation is out of scope for v1 but should be considered if viewing key leakage becomes a practical concern.

### What Integrators Can Do

With a viewing key, integrators can:
- Display shielded balances
- Construct transaction intents
- Estimate fees
- Show locally reconstructed transaction history within the scope and TTL of the viewing key

### What Integrators Cannot Do

Without a spending key, integrators cannot:
- Sign transactions
- Move funds
- Authorize any on-chain action

Spending keys are derived just-in-time in the user's browser context, used to sign, then cleared from memory.

## Security Model

### What This Approach Buys

- Works with existing wallets (MetaMask, Ledger, WalletConnect)
- No new browser extension required
- Familiar UX for crypto users
- Single enrollment, persistent identity

### What This Approach Costs

- Phishing is the root compromise mode
- One successful phishing signature = full key compromise
- Users must actually back up their recovery secret

### Mitigations

| Threat | Mitigation |
|--------|------------|
| Phishing | EIP-712 domain separation, anti-phish checksum, activity shaping makes large thefts noisy |
| XSS | Web worker isolation for key operations, keys cleared after use |
| Malicious extensions | No full mitigation; users must trust their browser environment |
| Wallet signing changes | Recovery secret is canonical; re-signing is convenience only |
| Derivation truncation | Normative bytes-to-field mapping, IC-1 (BigInt-only), IC-3 (opaque passthrough), IC-4 (test vectors incl. boundary vectors), IC-2 (entropy canary), IC-5 (migration capability) |

### Honest Assessment

If a user signs a malicious EIP-712 message on a phishing site, they're compromised. This is the cost of wallet-native UX.

What makes Armada defensible despite this:
- Typed data + domain separation reduces silent phishing
- Anti-phish checksum provides detectability
- Activity shaping + liquidity constraints make large thefts operationally harder and statistically visible

This is not "drain instantly and vanish" like mixing protocols. That matters for regulatory positioning and actual user protection.

## Explicitly Out of Scope

The following are not addressed in this design and should not be conflated with the signing architecture:

- Social recovery
- Guardian / multisig recovery
- On-chain key rotation
- Hardware wallet-specific flows (beyond standard EIP-712 signing)

These may be explored in future iterations but are not part of the v1 signing model.

## Future: MFKDF Integration

The key hierarchy supports adding multi-factor key derivation later:

```
wallet_signature (possession factor)
       +
    TOTP code (knowledge factor)
       â†“
   root_secret (same derivation output)
```

Benefits:
- Phishing the wallet signature alone is insufficient
- "Recover without seed phrase" becomes a product feature
- Positions Armada as "account-like" rather than "wallet-backed"

This changes the mental model significantly and is a v2 consideration. The current hierarchy is designed to support it without migration pain.

## Implementation Checklist

### v1 Requirements

**Spec blockers** (must be resolved before spec is considered complete):

- [ ] **Andrew confirms spending key modulus**: Does Railgun reduce spending key scalars mod `r` (BN254 scalar field, Option A) or mod `l` (Baby Jubjub subgroup order, Option B)? Both produce valid Baby Jubjub points, but the canonical range determines interoperability with existing Railgun wallets and whether in-circuit range checks pass. If mod `l`, boundary test vectors must be recomputed.
- [ ] **Andrew confirms viewing key architecture**: Does Armada inherit Railgun's Ed25519 viewing keys (requiring mod `l_ed` reduction and Ed25519 key conventions), or define Armada-native viewing keys operating in F_r? This determines whether the derivation tree can use a single conversion function or needs key-type-aware reduction.
- [ ] **Andrew confirms byte endianness**: Does Railgun's existing tooling interpret spending key bytes as big-endian?
- [ ] **Andrew confirms zero-scalar acceptance**: Do the circuits accept zero as a valid spending key / nullifier input? If nonzero is required, confirm the re-derive-with-incremented-info pattern is compatible with existing Railgun wallet expectations.
- [ ] **HKDF hash function**: Confirm HKDF-SHA-256 (not SHA-512 or other variant).
- [ ] **IC-4 concrete vectors**: end-to-end test vectors with real hex values (blocked on SDK implementation and above modulus decisions)

**Implementation requirements:**

- [ ] EIP-712 message structure defined and documented
- [ ] HKDF derivation implemented with versioned info strings
- [ ] Anti-phish checksum displayed post-enrollment
- [ ] Forced recovery secret export before first use
- [ ] Viewing key scoping and TTL defaults
- [ ] Web worker isolation for spending key operations
- [ ] Clear-after-use for all key material in memory
- [ ] Bytes-to-field-element mapping implemented per normative spec (big-endian, correct modulus per key type, BigInt-only)
- [ ] Boundary test vectors for bytes-to-scalar conversion validated in CI (recomputed if modulus changes from r to l)
- [ ] IC-1: No `Number` in derivation path; lint rule enforced in CI
- [ ] IC-2: Entropy floor canary on root_secret, spending_key, viewing_key
- [ ] IC-3: Opaque byte passthrough verified (no intermediate numeric parsing)
- [ ] IC-4: End-to-end test vectors published and validated in CI
- [ ] IC-5: Bug-triggered migration capability validated against Railgun circuit constraints

### v1.x Considerations

- [ ] Encrypted local storage option
- [ ] Re-signing convenience flow with mismatch detection
- [ ] Integrator telemetry (aggregate only)
- [ ] MFKDF prototype

## References

- [EIP-712: Typed structured data hashing and signing](https://eips.ethereum.org/EIPS/eip-712)
- [ERC-2494: Baby Jubjub Elliptic Curve](https://eips.ethereum.org/EIPS/eip-2494) — defines Baby Jubjub over BN254's F_r; subgroup order `l`, cofactor 8
- [BN254 For The Rest Of Us](https://hackmd.io/@jpw/bn254) — field parameters, scalar field order `r`, relationship to Baby Jubjub
- [HKDF: RFC 5869](https://tools.ietf.org/html/rfc5869)
- [Railgun: Wallets and Keys](https://docs.railgun.org/wiki/learn/wallets-and-keys) — spending keys on Baby Jubjub (BIP-32), viewing keys described as Ed25519
- [Railgun: Using Private Tokens](https://docs.railgun.org/wiki/learn/using-private-tokens) — nullifier derivation from spending key + Merkle path
- [Privacy Pools](https://github.com/ameensol/privacy-pools)
- [Privacy Pools SDK: key entropy truncation bug (disclosed Feb 2025)](https://x.com/0xprivacypools/status/2036128249525272887) — `bytesToNumber` (JS float precision) used instead of `bytesToBigInt` in master secret derivation, reducing effective entropy from 256-bit to ~53-bit
- [Privacy Pools Core SDK repository](https://github.com/0xbow-io/privacy-pools-core)
- [MFKDF](https://mfkdf.com)
