# Security Policy

## Reporting a Vulnerability

If you find a security issue — in the design, the protocol, the hardware spec, or any future code — please report it privately rather than opening a public issue.

Email: **adelos-security@idltd.com**

Please include enough detail to reproduce or understand the issue. We will respond as soon as we are able.

## Scope

At this stage Adelos is a published design. The attack surface includes:

- The session protocol (ECDH, AEAD, device-originated nonce, salt ratchet)
- The pairing channel (LED/camera fingerprint verification)
- The verified-claims architecture (anonymous credentials, unlinkability)
- The hardware boundary assumptions (eFuse, HMAC-on-die, Secure Boot)

Findings against the design are as welcome as findings against code.

## Out of Scope

- Attacks that require full physical control of both the device and the salt store simultaneously — this is the acknowledged residual ceiling, documented in the spec.
- Network-layer anonymity — explicitly out of scope for this project.
