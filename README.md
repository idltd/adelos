# Adelos

> **I'm back. It's me. You don't know me — but you know it is me.**

Pseudonymous hardware proof of continuity. A FOSS counter to centralised digital ID.

---

## What this is

Adelos is a small, cheap, open-source hardware device that lets you prove you are the same person who was here before — **without revealing who you are.**

There is no central authority. No issuer. No registry. No name. The device recognises you because you can produce the correct cryptographic response, not because anyone looked you up.

You build it yourself from open hardware and open firmware. Nobody can revoke it. Nobody is notified when you use it. Nobody can correlate your activity across services unless they physically hold both your device and your data.

## Why it exists

Governments and corporations are pushing centralised digital identity: a single issuer that vouches for you to everyone else. That architecture lets the issuer revoke you, correlate everything you do, and serve as a permanent point of surveillance and failure.

Adelos inverts it. The "issuer" is an £8 device you made. The proof is yours alone. Recognition without identification — you know it is the same one, you do not know who.

This is **recognition without identification.** The system can verify continuity (it is the same one) while disclosing no identity (not who). That contradiction is only apparent — the device is exactly what makes "you don't know me but you know it is me" simultaneously true.

## How it works (in one line)

The device computes `HMAC(K, salt)` with a key `K` burned irreversibly into the chip, gated by your fingerprint, delivered over Bluetooth to a web app. The output unwraps your encrypted data. The key never leaves the chip; the output is used and immediately discarded.

## The build

A four-part device:

- **ESP32-C3** — Bluetooth radio, on-die HMAC engine, and an eFuse-burned key that no software can read.
- **Fingerprint sensor** — biometric presence; it is your body, not just your device.
- **Battery** — standalone, no tether.
- **Case** — protection and tamper-evidence.

Approx. £4–12 per unit. No separate secure-element chip required.

Full parts list, session protocol, pairing channel, and the salt-ratchet design are in the spec.

## Documentation

- ➡️ **[Hardware spec — parts & reasoning](dongle-parts-and-reasoning.md)** — the device, BOM, session protocol, salt ratchet.
- ➡️ **[What it does (for everyone)](claims-overview.md)** — proving things without revealing who you are, in plain language.
- ➡️ **[Verified claims — technical architecture](claims-technical.md)** — issuer/holder/verifier, delegated self-signing, anonymous credentials, revocation.
- ➡️ **[Futures](futures.md)** — replacing government digital ID scheme by scheme, and the open problem of hosts using the dongle wisely.

## Revocation, in one line

A revoke is a doc signed by the key itself, saying "this key is dead." Once seen, the key is cancelled and **nothing it ever signed can be trusted** — compromise may have happened at any time. Self-revocation uses the same primitive as proof: the key that says "it is me" can sign "it is no longer me." The fingerprint gate is what stops a thief signing it. Users can renounce anything they said; other users choose what to believe. No central arbiter — just provable statements and individual judgement.

## What it does and does not do

**It does:**
- Prove continuity — you were here before and you are back.
- Bind that proof to your body via biometrics.
- Keep you pseudonymous — no identity is ever disclosed.
- Work with no third party, ever — no issuer, no revocation authority.
- Run on Windows 10 and other systems where modern platform key-derivation (FIDO PRF) is unavailable.

**It does not:**
- Prove *who* you are to a stranger — only that you control this device.
- Anonymise your network traffic or IP.
- Prevent a government from demanding identity by legal means at a higher layer.

## Status

Early. Design and rationale are published; firmware, web library, and hardware files are in progress.

## License

FOSS. License to be finalised — intended to be permissive and reproducible by anyone.
