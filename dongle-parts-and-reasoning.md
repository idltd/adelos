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

## 2. Why These Parts

### 2.1 ESP32-C3 Mini

The C3 is the entire device. It provides:

- **BLE radio** — connects to the host browser via Web Bluetooth (Chromium-based browsers only).
- **eFuse key blocks** — 256-bit HMAC key burned at provisioning, write- and read-protected in silicon. The key cannot be extracted by software and is permanent.
- **On-die HMAC peripheral** — computes `HMAC(K, salt)` in hardware without K ever appearing on a bus or in RAM accessible to application code.
- **Flash encryption and Secure Boot v2** — eFuse bits lock the firmware at commissioning. Post-lock, only OTA-signed updates are accepted; JTAG and UART download mode are disabled.
- **Onboard LED** — anti-MITM pairing channel: blinks the device public key fingerprint on first connection so the host camera can read and pin it, defeating BLE man-in-the-middle substitution.

### 2.2 Fingerprint Sensor

Biometric presence gate. The dongle will not respond to a derivation request until the enrolled fingerprint matches. This is stronger than a button press:

- A button proves physical proximity. A fingerprint proves physical proximity *and* identity.
- A malicious page that connects to the dongle silently cannot extract key material — the user must biometrically authorise each unlock.
- The fingerprint template is stored on the sensor module itself (most off-the-shelf modules have on-board flash and match-on-chip) — it does not pass through the C3 and is never transmitted over BLE.

Suitable modules: R307, AS608, or equivalent UART fingerprint sensors with match-on-chip. Wired directly to the C3's UART.

### 2.3 Battery

Makes the device standalone — no USB tether required during use.

- **CR2032** — simplest, no charging circuit, ultra-slim, ~6–12 months typical BLE use.
- **Small LiPo + charging IC** — rechargeable, higher capacity, adds a USB-C port and charge controller to the BOM.

CR2032 is the simpler v1 path.

### 2.4 Case

Physical protection and tamper-evidence. A well-fitted case makes opportunistic hardware tampering visible. Not a security boundary on its own, but raises the bar for casual physical attack.

### 2.5 What Is Not In The BOM

- **ATECC608A** — removed. C3 eFuse HMAC covers its function.
- **USB connector (for use)** — not required. BLE is the session transport. USB is disabled post-commissioning by Secure Boot.
- **Display** — not required. The LED covers the one visual output needed (pairing channel).
- **Passives** (resistor for LED, decoupling caps) — pennies, omitted from cost table.

---

## 3. How It Connects and What It Returns

### 3.1 Connection

Web Bluetooth (BLE GATT) from a Chromium-based browser. No driver, no native app, no USB. The user is assumed to be physically present — they provide the fingerprint.

First connection requires pairing: the LED blinks the device static public key fingerprint; the host camera reads it; the browser pins that key. Subsequent ECDH key exchanges are authenticated against the pinned key, defeating BLE MITM substitution.

### 3.2 Session Protocol

| Step | Actor | Action |
|---|---|---|
| 1 | Browser | Connects to dongle over BLE (Web Bluetooth) |
| 2 | Both | ECDH key exchange — shared session key derived, channel encrypted with AEAD |
| 3 | User | Provides fingerprint — biometric presence confirmed on-device |
| 4 | Browser | Sends a salt over the encrypted channel |
| 5 | Dongle | Computes `HMAC(K, salt)` in hardware; K is in eFuse, never leaves the chip |
| 6 | Dongle | Returns 32-byte result over encrypted channel |
| 7 | Browser | Uses 32 bytes to unwrap key(s), performs operation, zeroes the 32 bytes immediately |

### 3.3 What the Browser Does With the 32 Bytes

The 32-byte output is used as a key-wrapping key (or KDF input) to unwrap the Data Encryption Key(s) for the encrypted store. Immediately after that operation the 32 bytes are zeroed from memory.

The 32-byte output is **never** written to disk, logged, cached in a cookie, or stored anywhere. If it is, the device binding is broken and the store is compromised by whoever has that copy.

The consuming application decides granularity: one salt per session for a single master unwrap key, or one salt per secret for per-item key isolation. The dongle computes `HMAC(K, salt)` for whatever salt it receives — the policy is the application's concern.

---

## 4. Salt Ratchet — Rolling Double Copy

### 4.1 The Problem It Solves

A captured salt is a static credential: present it to the dongle and get the correct 32 bytes indefinitely. The salt ratchet limits the useful window of a captured salt to at most two genuine unlocks.

### 4.2 Mechanism

| State | Detail |
|---|---|
| Stored | Salt N and Salt N-1 (rolling double copy) |
| On unlock | Decrypt with Salt N, re-encrypt with new Salt N+1, discard Salt N-1 |
| New state | Salt N+1 (current) and Salt N (previous) |
| Stolen salt window | At most two genuine unlocks before the captured salt rotates out |
| Rollback risk | Salt store must be integrity-protected; rewinding to a captured salt defeats the ratchet |
| Backup recovery | Separate long-lived recovery salt; does not rotate; used only to re-establish a fresh chain |

Two copies are kept so a crash mid-rotation does not produce a state where neither salt is valid. On recovery, the system falls back to Salt N and re-attempts rotation.

### 4.3 Threat Interactions

- **Stolen salt only** — useless after two genuine unlocks. Attacker must race before it rotates out.
- **Stolen device only** — useless without the salt store. The device needs a valid salt as input.
- **Both stolen** — full compromise. Physical separation of device and salt store is the mitigation. Backup dongles address recovery.
- **Salt store rolled back** — defeats the ratchet. Integrity-protect the salt record (e.g. HMAC of the salt file) to make rewinding detectable.

---

## 5. Security Boundary Summary

- K never leaves the ESP32-C3 — eFuse read-protected, HMAC computed on-die.
- Derived key material (32-byte output) exists in browser memory for milliseconds only — zeroed immediately after use.
- Salt ratchet limits the value of a captured salt to a two-unlock window.
- Biometric presence (fingerprint) required per derivation — silent remote extraction is blocked.
- Pairing channel (LED + camera) authenticates first contact — BLE MITM is blocked.
- Flash encryption + Secure Boot lock the firmware — a stolen device cannot be reflashed into a compliant oracle.
- **Residual ceiling:** a compromised host OS at the moment of decryption can read plaintext from memory after the DEK is applied. This is not addressable by the dongle.
