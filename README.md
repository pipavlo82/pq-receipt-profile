# PQ Receipt Profile v0

Post-quantum provenance: what survives Q-day in an execution-receipt system, what breaks, and what it costs to fix.

Status: draft v0 · Author: Pavlo Shtomko (@pipavlo82) · Scope: ReceiptOS / Chronicle / Crystal Artifact, with direct mappings to ATP Proof-of-Cognition receipts and ERC-8313 signature profiles.

---

## 0. Why this document exists

A receipt is a long-lived object. Its entire value is deferred: it is verified *later* — in a dispute next year, an audit in five, a reputation history in ten. Any receipt system therefore inherits a threat most execution systems can ignore: record now, forge later.

When a cryptographically relevant quantum computer (CRQC) arrives, ECDSA falls to Shor's algorithm. At that point anyone can derive private keys from public keys and forge historical attestations — sign anything, from any agent's identity, backdated in appearance. For a system whose one promise is "a historical record you can trust," this is not a peripheral risk. It attacks the promise itself.

This profile maps the receipt stack into three layers — integrity, attribution, randomness — and states plainly: layer 1 is already post-quantum safe and needs zero work; layer 2 breaks and has a concrete, costed fix; layer 3 is the forward-looking completion that makes the chain of trust whole.

The principle throughout is the same one that governs the rest of the stack: *the receipt gates nothing and scores nothing; it proves. This profile only extends how long the proof stays trustworthy.*

---

## 1. Integrity — the trace (already post-quantum safe)

What it is: every recomputable commitment in the stack — receipt_root, Chronicle's artifact_root / collection_root / portfolio_root, Merkle event roots, crystal_hash, the golden-vector fixtures that lock canonicalization across codebases.

Quantum impact: Grover's algorithm gives at most a quadratic speedup against hash preimages. SHA-256 retains ~128-bit post-quantum security; keccak256 likewise. No known quantum attack breaks recomputability.

Consequence: the heart of the system — *recompute it yourself, don't trust the issuer* — survives the quantum transition unchanged. Same evidence + same rules = same root, before and after Q-day.

A note on anchored receipts: a hash committed on-chain *before* Q-day proves that exactly this content existed at that time. Content-and-time binding survives even if every signature in the world becomes forgeable, because forging the content would require a hash preimage, not a key. Anchoring is therefore not just a timestamping convenience — it is the quantum-transition bridge.

A note on the Crystal Artifact: the Tier-1 QR envelope carries only {crystal_version, receipt_root, mutation_hashes, crystal_hash} — no signatures at all. A printed crystal is a pure hash object and is therefore fully post-quantum durable as-is. The most physically portable artifact in the stack happens to be the most quantum-durable one.

Action required: none. This layer needs articulation, not engineering.

---

## 2. Attribution — the signature (breaks; fix is known and costed)

What it is: the signatures field of a receipt — who attested. In ATP terms, the signatures array of ρ; in ReceiptOS terms, any attestation binding an identity to a receipt; in ERC-8313 terms, the signatures section of a PIM.

Quantum impact: ECDSA (and every other discrete-log scheme) falls to Shor. Post-Q-day, an attacker derives the private key from any public key and signs arbitrary statements as any historical identity. Integrity of *what happened* survives (layer 1); who vouched for it becomes forgeable.

The fix — dual-signature attestation profile:

Each receipt attestation carries two signatures over the same canonical receipt hash:

attestation = {
 receipt_hash: H(Canon(receipt_body)), // layer-1 object, already PQ-safe
 sig_ecdsa: ECDSA(receipt_hash), // today's verifiability, cheap
 sig_mldsa: ML-DSA-65(receipt_hash), // FIPS 204, quantum-resistant
 signer: <address / DID>,
 signer_pq_key: keccak256(mldsa_public_key) // hash of PQ pubkey, not the key itself
}
- Verifiers today check ECDSA (cheap, universal).
- Verifiers after Q-day check ML-DSA-65 (the ECDSA signature becomes decorative).
- During the transition, both must verify — a forged attestation would need to break both schemes simultaneously.
- Publishing keccak256(pubkey) instead of the raw ML-DSA key keeps payloads small (ML-DSA-65 public keys are 1,952 bytes; the raw key is revealed only when verification is exercised). This is the same pattern ERC-8313 already sketches with its signerHashed field.

What it costs — real numbers, not hand-waving:

On-chain ML-DSA-65 verification is expensive but has been driven down materially: an 82% gas reduction from naive baseline is achievable via NTT inner loops in inline assembly and Barrett-reduction optimization (prior work: pipavlo82, Solidity ML-DSA-65 verification research; methodology: *gas per secure bit*, enabling apples-to-apples comparison of PQ schemes on-chain). Off-chain verification is trivially cheap (milliseconds).

The practical profile:
- Off-chain receipts (the default in ReceiptOS/ATP): dual-sign everything; verification cost is negligible. There is no economic reason not to.
- On-chain attestations (anchors, escrow acceptances): anchor the *hash* (layer 1, free of PQ concerns) and keep PQ signatures off-chain and exercisable on demand — verify on-chain only when disputed. This keeps gas costs out of the happy path entirely.

Migration story for existing receipts: receipts signed ECDSA-only before adoption of this profile are not lost. Re-attest them (dual-sign the same receipt_hash) any time *before* Q-day, and anchor the re-attestation. The layer-1 anchor proves the content predates the re-attestation; the new dual signature carries attribution forward. Nothing needs to be re-executed — only re-signed.

Standards mapping:
- ERC-8313 (PIM): the spec's signatures.type field already reserves "fips-204" and "fips-205" as future values and defines signerHashed for large PQ keys. This profile is a concrete instantiation of that placeholder, with gas data.
- ATP Proof-of-Cognition: ρ's signatures array accepts multiple proofs per receipt; a dual-sig attestation drops in without schema change. The envelope's proofs[] (JWS today) extends the same way.
- Composition note (Step 5): eligibility conformance is unaffected — this profile changes nothing about *what* the receipt proves, only how long its attribution stays unforgeable. A PQ appendix is a natural future addition, not a change to §5.

---

## 3. Randomness — the generator (forward-looking completion)

What it is: every place in a receipt lifecycle that consumes randomness whose fairness someone may later dispute: signature nonces, session_id generation, validator/audit sampling (e.g. which work-units get audited in a campaign), challenge generation for dispute protocols.

Why it belongs in this profile: a receipt proves what happened — but if the *selection* of what got audited, or the nonce inside a signature, came from manipulable randomness, a sophisticated attacker attacks the process around the receipt rather than the receipt itself. "Verifiable after the fact" — the system's core property — should extend to its randomness.

The fix — PQ-VRF: a verifiable random function built on post-quantum primitives (prior work: Re4ctoR — PQC CSPRNG/VRF with ML-DSA-65 dual signatures, FIPS-compliance work, sealed entropy core). A VRF output is randomness *with a proof*: anyone can verify post-hoc that the value was generated correctly from the committed seed, and PQ construction means that proof survives Q-day.

Concretely:
- audit sampling in coordination networks (ATP campaigns / repository-audit work-units) draws from VRF output → the *selection itself* becomes a receipt-grade, recomputably-fair fact;
- session identifiers and protocol nonces derive from the sealed entropy core → no "chosen nonce" attacks, and the derivation is attestable;
- dispute challenges are VRF-generated → neither party can grind them.

Status: this layer is optional for a v0 profile and intentionally scoped as *completion, not requirement*. Layers 1–2 stand alone. Layer 3 is what turns "quantum-safe receipts" into a complete post-quantum chain of trust: where the randomness came from → who signed → what happened — every link verifiable, every link surviving Q-day.

---

## 4. Summary table

| Layer | Object | Q-day impact | Action | Cost |
|---|---|---|---|---|
| 1. Integrity | roots, Merkle, crystal_hash, anchors | survives (Grover only halves hash security) | none — articulate | zero |
| 2. Attribution | receipt signatures | breaks (Shor vs ECDSA) | dual-sig ECDSA + ML-DSA-65; signerHashed pattern; re-attest legacy receipts pre-Q-day | off-chain ~free; on-chain only on dispute, with known 82%-reduced gas path |
| 3. Randomness | nonces, session_ids, audit sampling | degraded trust in process fairness | PQ-VRF (Re4ctoR pattern) | optional in v0 |

The one-sentence version: the proof already survives; the vouching needs a second signature; the dice can be made provably fair — and all three components exist today.

---

## 5. Non-goals

In keeping with the boundary held everywhere else in this stack:
- This profile does not add scoring, gating, settlement, ownership, or reputation semantics to receipts.
- It does not propose changing Ethereum's own ECDSA (anchor transactions inherit whatever the L1 migrates to; that is EF-level work outside this scope).
- It does not require any receipt consumer to verify PQ signatures today — ECDSA remains the cheap default until it isn't.

The receipt still gates nothing and scores nothing. It just keeps proving — longer.
