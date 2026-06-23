# Adelos — Verified Claims: Technical Architecture

The plain-language version is in the [overview](claims-overview.md). This document assumes familiarity with public-key crypto, HMAC, signatures, and the verifiable-credentials model.

---

## 1. What the dongle provides, and what it doesn't

The Adelos device is a hardware-bound, biometric-gated key holder. Its primitive is:

```
HMAC(K, salt) -> 32 bytes
```

where `K` is burned into the ESP32-C3 eFuse, never extractable, and each invocation is gated by an on-device fingerprint match. From `K` the device can also derive stable keypairs (e.g. `HMAC(K, context)` seeded), giving it a controllable set of cryptographic identities.

What the bare primitive gives you:

- **Possession proof** — the holder controls `K`.
- **Continuity** — the same `K` produces the same outputs over time, so a verifier can confirm "same controller as before."
- **Biometric binding** — each use requires the enrolled finger.
- **Hardware binding** — `K` cannot be copied off the device by software.

What it does **not** give you:

- **Attribute attestation.** The device cannot originate a trustworthy claim like "over 18." Verified attributes require an external issuer whose signature a verifier trusts.
- **Unlinkability, for free.** A naively reused key or a directly-presented issuer signature becomes a stable correlator.

The rest of this document is about building a verified-claims system *on top of* the dongle, and the two non-obvious problems that have to be solved.

## 2. The three roles

Standard verifiable-credentials shape:

- **Issuer** — attests a claim about the holder's pseudonymous key. E.g. a social platform analyses account history and concludes the controller is an adult. The issuer may also know the real-world identity; that knowledge stays issuer-side.
- **Holder** — the user plus the Adelos device. Stores credentials encrypted, unwrapped only by dongle + fingerprint. Presents proofs.
- **Verifier** — checks a presented proof: that a trusted issuer signed the claim, that it's bound to a key the holder controls, that control is demonstrated live, and that it's unrevoked.

The issuer is unavoidable. "Over 18" has to be vouched for by someone; the dongle binds and releases that vouch, it does not manufacture it.

## 3. Worked example: attested adulthood

1. **Enrolment (once).** Controller of key `X` proves control of `X` to the issuer (social platform). The platform, from behavioural evidence, decides the controller is an adult. It may know `X` ↔ Fred Smith; that linkage stays in its records.

2. **Issuance.** The platform signs an attestation. **Critically, the signature is over the pseudonymous key `X`, not over "Fred Smith".** The portable artifact is identity-free: *"controller of `X` is an adult."*

3. **Presentation.** The holder presents, to a verifier, the claim plus a live proof of control of `X` (fingerprint-gated dongle response). Verifier checks the issuer signature and the live control proof.

4. **Result.** Verifier concludes: "a source I trust says this adult-controller exists, and they are present now." No name, no birthdate, no issuer contact.

This is correct as far as it goes — but steps 3–4 as written have two flaws.

## 4. Problem 1 — the issuer must not be in the per-use loop

If the holder presents the issuer's signature directly on every use, that's fine for the issuer's online presence (it's an offline signature) — but if the scheme were instead "ask the issuer to sign each presentation," the issuer becomes a per-transaction age-verification signing service. That is both a surveillance vector (issuer sees every use) and a bad institutional look (selling/again-and-again signing age checks).

**Fix: delegated self-signing.**

- The issuer signs **once**, and the attestation includes a delegation: *"controller of `X` is an adult, and `X` is authorised to self-assert this claim until [expiry]."*
- The holder then **self-signs** per-verifier assertions: *"I, controller of `X`, assert adulthood,"* presented together with the issuer's delegating certificate as the root of trust.
- The verifier checks: (a) the holder's self-signature verifies under `X`; (b) the issuer's cert authorises `X` to self-assert this claim; (c) it's unexpired and unrevoked.

The issuer is now never contacted at use time. It signed a **capability**, not a transaction. Per-use issuer signing never happens.

## 5. Problem 2 — delegation alone does not give unlinkability

Self-signing under `X` solves per-use issuer involvement but **not** correlation. If the holder keeps presenting the same `X` and the same issuer cert, then:

- Every verifier sees the same `X` → cross-verifier correlation; `X` is a stable tracking handle.
- The issuer cert points back to the issuer → uses are traceable to the voucher.

Two approaches, in increasing order of strength and effort:

### 5a. Per-verifier pseudonyms (cheap, partial)

Derive a fresh key per verifier from the dongle: `X_v = f(HMAC(K, verifier_id))`. Each verifier sees a distinct key. This breaks cross-verifier correlation on the *key*, but:

- The issuer cert is bound to one root key. Naively, presenting under `X_v` breaks the chain. You need the issuer to delegate to a **root** from which per-verifier children are *provably derived* (e.g. a verifiable derivation, or the issuer signs the root and the holder proves `X_v` descends from it without revealing the root). This is a real sub-protocol.
- It does not by itself hide the issuer's identity at presentation, nor prevent a colluding issuer+verifier from re-linking.

### 5b. Anonymous credentials (proper, harder)

Use a credential scheme designed for this: BBS+, CL signatures, or a zk-credential construction.

- The issuer issues a credential on the claim, bound to a holder secret (which the dongle holds/gates).
- At presentation, the holder produces a **zero-knowledge proof**: "an issuer (optionally: from this trusted set) signed an adulthood claim bound to a key I control," with the proof **freshly randomised each time** so two presentations are cryptographically unlinkable.
- Selective disclosure is native: prove the predicate (`age ≥ 18`) without revealing the datum (DOB).
- Issuer is offline at presentation; verifier learns only the predicate; presentations don't correlate.

Adelos does not invent these schemes. It integrates one, and contributes the properties the scheme lacks on its own: a **hardware-bound, biometric-gated, non-exportable holder secret** with **physical-presence enforcement** per proof. The dongle is the holder's secure element; the credential math is layered on top.

## 6. Honest limits

- **Trust is not correctness.** The crypto proves the issuer *said* it and that the holder controls the key. It does not prove the issuer was *right*. Garbage attestation in, garbage trust out.
- **Issuer-side linkage persists.** The issuer knew `X` ↔ Fred at issuance. Unlinkability is achieved toward verifiers and across uses (and, in 5b, can hide the issuer's identity at presentation). It is **not** achieved toward the issuer. A compelled or breached issuer can leak the original linkage.
- **Collusion re-identifies.** If issuer and verifier pool their data, the holder can be unmasked. The model assumes non-collusion or holder-chosen counterparties.
- **Revocation is an open problem.** If an issuer withdraws a claim, the verifier must learn this *without* a phone-home that reintroduces surveillance. Candidate mechanisms: short-lived attestations with reissuance, cryptographic accumulators, privacy-preserving revocation lists. Each has tradeoffs; none is free.
- **No network-layer anonymity.** Adelos protects identity within the proof. IP/network anonymity is orthogonal (Tor/VPN territory).

## 7. Standards alignment

The issuer/holder/verifier split, selective disclosure, and predicate proofs map onto the W3C Verifiable Credentials data model and the anonymous-credential literature (BBS+ in particular). Where possible, Adelos should present as a standard holder/wallet with a hardware-bound, biometric-gated holder secret, rather than inventing wire formats — so existing issuers and verifiers can interoperate.

## 8. Summary

- The dongle gives possession, continuity, hardware binding, biometric gating.
- An external issuer supplies verified attributes — unavoidably.
- **Delegated self-signing** keeps the issuer out of the per-use loop.
- **Anonymous credentials** keep uses unlinkable and untraceable to the issuer.
- Adelos's distinct contribution is the hardware-bound, biometric-gated, non-exportable, presence-enforced **holder** — the secure element for an anonymous-credential wallet, built FOSS for a few pounds.
