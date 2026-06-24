# Hardware-Bound Auth Dongle
## Parts List and Design Reasoning

---

## 1. Parts List

| Component | Approx. Cost | Role |
|---|---|---|
| ESP32-C3 Mini module | £2–4 | Microcontroller, BLE radio, eFuse key storage, HMAC derivation engine, onboard LED |
| Fingerprint sensor | £1–3 | Biometric presence authorisation — user must match to permit a derivation |
| Battery + holder | £0.50–2 | Standalone power — CR2032 for slim build, small LiPo for rechargeable |
| Case | £0.50–3 | Physical protection and tamper-evidence |
| **TOTAL** | **~£4–12** | Per unit at hobbyist quantities |

No separate secure element IC. The ESP32-C3's on-die eFuse and HMAC peripheral replace the ATECC608A.

---

## 2. Build Status

| Component / layer | Status |
|---|---|
| Commissioning software | **Working** — connects to ESP32, selects architecture across variants |
| eFuse key burning | **Designed; not yet applied** — deliberately held until firmware is proven |
| Secure Boot v2 | **Designed; not yet applied** — held for the same reason (irreversible) |
| Fingerprint sensor | **Specified; not yet sourced or integrated** |
| BLE session protocol | **Designed; next build step** |
| Web library | **Designed; follows BLE layer** |
| Physical prototype | **Not yet assembled end-to-end** |

---

## 3. Why These Parts

### 3.1 ESP32-C3 Mini

The C3 is the entire device. It provides:

- **BLE radio** — connects to the host browser via Web Bluetooth (Chromium-based browsers only).
- **eFuse key blocks** — 256-bit HMAC key burned at provisioning, write- and read-protected in silicon. The key cannot be extracted by software and is permanent.
- **On-die HMAC peripheral** — computes `HMAC(K, salt)` in hardware without K ever appearing on a bus or in RAM accessible to application code.
- **Flash encryption and Secure Boot v2** — eFuse bits lock the firmware at commissioning. Post-lock, only OTA-signed updates are accepted; JTAG and UART download mode are disabled.
- **Onboard LED** — anti-MITM pairing channel: blinks the device public key fingerprint on first connection so the host camera can read and pin it, defeating BLE man-in-the-middle substitution.

### 3.2 Fingerprint Sensor

Biometric presence gate. The dongle will not respond to a derivation request until the enrolled fingerprint matches. This is stronger than a button press:

- A button proves physical proximity. A fingerprint proves physical proximity *and* identity.
- A malicious page that connects to the dongle silently cannot extract key material — the user must biometrically authorise each unlock.
- The fingerprint template is stored on the sensor module itself (most off-the-shelf modules have on-board flash and match-on-chip) — it does not pass through the C3 and is never transmitted over BLE.

Suitable modules: R307, AS608, or equivalent UART fingerprint sensors with match-on-chip. Wired directly to the C3's UART.

### 3.3 Battery

Makes the device standalone — no USB tether required during use.

- **CR2032** — simplest, no charging circuit, ultra-slim, ~6–12 months typical BLE use.
- **Small LiPo + charging IC** — rechargeable, higher capacity, adds a USB-C port and charge controller to the BOM.

CR2032 is the simpler v1 path.

### 3.4 Case

Physical protection and tamper-evidence. A well-fitted case makes opportunistic hardware tampering visible. Not a security boundary on its own, but raises the bar for casual physical attack.

### 3.5 What Is Not In The BOM

- **ATECC608A** — removed. C3 eFuse HMAC covers its function.
- **USB connector (for use)** — not required. BLE is the session transport. USB is disabled post-commissioning by Secure Boot.
- **Display** — not required. The LED covers the one visual output needed (pairing channel).
- **Passives** (resistor for LED, decoupling caps) — pennies, omitted from cost table.

---

## 4. How It Connects and What It Returns

*The session protocol below is the designed specification. The BLE layer is not yet implemented — it is the next build step.*

### 4.1 Connection

Web Bluetooth (BLE GATT) from a Chromium-based browser. No driver, no native app, no USB. The user is assumed to be physically present — they provide the fingerprint.

First connection requires pairing: the LED blinks the device static public key fingerprint; the host camera reads it; the browser pins that key. Subsequent ECDH key exchanges are authenticated against the pinned key, defeating BLE MITM substitution.

### 4.2 Session Protocol

| Step | Actor | Action |
|---|---|---|
| 1 | Client | Connects to dongle over BLE |
| 2 | Both | ECDH key exchange — shared session key derived; all subsequent communication AEAD-encrypted |
| 3 | Client | Requests a derivation nonce |
| 4 | User | Provides fingerprint — biometric presence confirmed on-device; gates nonce release |
| 5 | Dongle | Generates a fresh random nonce; records it as pending (single-use, short-lived); returns it over encrypted channel |
| 6 | Client | Sends `{ salt, nonce }` over the encrypted channel |
| 7 | Dongle | Verifies nonce is pending and unexpired; removes it (consumed); computes `HMAC(K, salt ‖ nonce)` in hardware — K never leaves the chip |
| 8 | Dongle | Returns 32-byte result — the device-originated nonce guarantees the same output is never returned twice |
| 9 | Client | Uses the 32 bytes as required; zeroes them immediately after use |

The nonce originates from the dongle, not the client. This is what enforces uniqueness: the client cannot force a repeat output by replaying a previous request, and a hostile client probing the device over BLE cannot obtain the same answer twice even for an identical salt.

### 4.3 What the Client Does With the 32 Bytes

The 32-byte output is a key-sized secret, freshly derived and never repeated. What the client does with it is application policy — for example:

- Use it directly as a key-wrapping key to unwrap an encrypted data store
- Feed it into a KDF to derive multiple sub-keys
- Use it as a signing input bound to a specific operation

Whatever the use, the 32 bytes must be zeroed from memory immediately after — never written to disk, logged, cached, or stored. If they are, the hardware binding is broken.

The dongle computes `HMAC(K, salt ‖ nonce)` for whatever salt it receives — granularity, key hierarchy, and application logic are the caller's concern.

---

## 5. Salt Ratchet — Rolling Double Copy

### 5.1 The Problem It Solves

A captured salt is a static credential: present it to the dongle and get the correct 32 bytes indefinitely. The salt ratchet limits the useful window of a captured salt to at most two genuine unlocks.

### 5.2 Mechanism

| State | Detail |
|---|---|
| Stored | Salt N and Salt N-1 (rolling double copy) |
| On unlock | Decrypt with Salt N, re-encrypt with new Salt N+1, discard Salt N-1 |
| New state | Salt N+1 (current) and Salt N (previous) |
| Stolen salt window | At most two genuine unlocks before the captured salt rotates out |
| Rollback risk | Salt store must be integrity-protected; rewinding to a captured salt defeats the ratchet |
| Backup recovery | Separate long-lived recovery salt; does not rotate; used only to re-establish a fresh chain |

Two copies are kept so a crash mid-rotation does not produce a state where neither salt is valid. On recovery, the system falls back to Salt N and re-attempts rotation.

### 5.3 Threat Interactions

- **Stolen salt only** — useless after two genuine unlocks. Attacker must race before it rotates out.
- **Stolen device only** — useless without the salt store. The device needs a valid salt as input.
- **Both stolen** — full compromise. Physical separation of device and salt store is the mitigation. Backup dongles address recovery.
- **Salt store rolled back** — defeats the ratchet. Integrity-protect the salt record (e.g. HMAC of the salt file) to make rewinding detectable.

---

## 6. Security Boundary Summary

*Items marked (designed) are part of the specified security model but not yet enforced — they will be once the corresponding layer is built or applied.*

- K never leaves the ESP32-C3 — eFuse read-protected, HMAC computed on-die. *(designed; active once eFuse is burned)*
- Derived key material (32-byte output) exists in client memory for milliseconds only — zeroed immediately after use. *(designed; active once client library is built)*
- Device-originated nonce guarantees the dongle never returns the same output twice — a hostile client cannot obtain a repeat response even for an identical salt. *(designed; active once BLE layer is implemented)*
- Salt ratchet limits the value of a captured salt to a two-unlock window at the application layer.
- Biometric presence (fingerprint) required per derivation, gating nonce release — silent remote extraction is blocked. *(designed; active once fingerprint sensor is integrated)*
- Pairing channel (LED + camera) authenticates first contact — BLE MITM is blocked. *(designed; active once BLE layer is implemented)*
- Flash encryption + Secure Boot lock the firmware — a stolen device cannot be reflashed into a compliant oracle. *(designed; active once Secure Boot is applied)*
- **Residual ceiling:** a compromised host OS at the moment of decryption can read plaintext from memory after the DEK is applied. This is not addressable by the dongle.
