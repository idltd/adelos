# Adelos — Futures

Forward-looking design notes. These are directions, not finished specifications.

---

## 1. Replacing government digital ID, scheme by scheme

The centralised digital-ID proposals circulating in the UK, EU, and elsewhere share a common architecture: a central issuer, a credential bound to your real identity, and verifiers who trust (or query) that issuer. Adelos can deliver the *legitimate functions* of each without the central issuer, the identity binding, or the surveillance surface.

The pattern in every case below is the same: an issuer attests a single predicate against your pseudonymous key; you present it minimally and unlinkably; no one logs the use.

| Government digital-ID use | What it actually needs | Adelos equivalent |
|---|---|---|
| **Age verification** (online goods, venues, content) | Proof of "over N", nothing else | Issuer attests `age ≥ N` against key; holder presents predicate proof. No name, no DOB, no log. |
| **Right to work / right to rent** | Proof of an entitlement status | Issuer (employer-facing authority) attests "entitled"; holder proves it per check, unlinkably. |
| **Proof of residency / jurisdiction** | Proof you belong to a region for a service | Issuer attests region; holder discloses only the region, not an address. |
| **Benefit / service eligibility** | Proof you qualify, and that you're one person | Issuer attests eligibility; one-per-human property prevents double-claiming without identifying. |
| **Professional licensing** | Proof you hold a current licence | Licensing body attests; holder proves licence status to clients without revealing identity. |
| **Voting / democratic participation** (where pseudonymous is acceptable) | Proof of one eligible person, one vote | Issuer attests eligibility; one-per-human prevents duplication; ballot stays unlinked to identity. |
| **Account recovery / "prove it's still you"** | Proof of continuity of control | Native to the device — continuity is the base primitive. |
| **Anti-bot / anti-sybil** ("prove you're human") | Proof of unique personhood | Biometric-gated unique key; one real person, one credential, no identity disclosed. |

The argument to make publicly: **every defensible function of digital ID is a predicate attestation, and predicate attestations do not require a central identity register.** The register is a policy choice, not a technical necessity — and it is the part that creates the surveillance. Adelos demonstrates the functions are separable from the register.

What Adelos deliberately does **not** replicate:

- A universal, government-readable identity lookup.
- Cross-service correlation by design.
- Central revocation-as-control (the ability to switch a person off everywhere at once).

Those are not missing features. They are the things the project exists to refuse.

## 2. Host software — using the dongle wisely (open area)

Everything described so far assumes the **host software** (the SSD app, the browser, the wallet) uses the dongle correctly. This is the project's least-developed area and its most important residual risk.

The dongle is a sound primitive. A primitive used carelessly is still unsafe. Concretely, the host is responsible for:

- **Disposing of derived secrets.** The 32-byte HMAC output must be zeroed immediately after use. A host that caches it, logs it, or writes it to disk silently destroys the whole guarantee. (Covered in the parts/reasoning spec, but it is a *host* obligation, not something the dongle can enforce.)
- **Correct salt construction and scoping.** Per-verifier pseudonyms, claim contexts, and unlinkability all depend on the host deriving salts correctly. A host that reuses a salt across verifiers reintroduces the correlation the design removes.
- **Honest presentation.** The host must present only the predicate the user authorised, not over-disclose. The dongle can't see what the host sends onward.
- **Revocation handling.** The host must check for and honour key-signed revokes. A host that skips the check trusts a dead key.
- **Pairing integrity.** The LED/camera pairing channel only works if the host implements it faithfully. A lazy host that skips out-of-band verification is open to BLE MITM.
- **Not exfiltrating the fingerprint path.** If a host design ever routes biometric data through itself, that's a new leak. Match-on-sensor must be preserved.

The honest framing: **Adelos moves the trust boundary from a remote central authority to the local host plus the dongle.** That is a large improvement — local and auditable beats remote and opaque — but it is not zero trust. A compromised host at the moment of use can still observe plaintext after decryption. The dongle cannot fix a hostile host; it can only ensure the *key* never leaks and *presence* is always required.

Future work here:

- A reference host library that implements all of the above correctly, so application developers don't each re-derive the footguns.
- A formal-ish checklist / conformance test for "is this host using the dongle safely."
- Delivery-integrity measures for the host itself (reproducible builds, pinned service workers, SRI) so the host the user runs is the host that was audited.

This section is explicitly an **open invitation**: the hardware and primitive are the easy part. Making hosts that use them wisely, by default, is where the project most needs design attention.

## 3. Other directions

- **Multi-dongle sealed backup.** Provisioning that seals a recoverable secret to dongles 2 and 3 for safe-deposit/under-the-bed backup, so loss of the primary isn't catastrophic.
- **Anonymous-credential integration.** A concrete BBS+ (or successor) binding so unlinkable predicate proofs are real, not just described.
- **Standards interop.** Presenting as a W3C Verifiable Credentials holder/wallet so existing issuers and verifiers work without bespoke wire formats.
- **Multi-transport.** USB/NFC alongside BLE for hosts and browsers without Web Bluetooth (noting the Chromium-only limitation of the web transports).
