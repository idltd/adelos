# Contributing to Adelos

Adelos is early. Design and rationale are published; firmware, web library, and hardware files are in progress. Contributions are welcome at the right layer for the current stage.

## Right now — design discussion

The most valuable thing you can do is engage with the design. Open areas are described in [futures.md](futures.md), particularly:

- **Host software correctness** — what does a reference host library look like, and what's the conformance checklist for hosts using the dongle safely?
- **Anonymous credential integration** — a concrete BBS+ (or successor) binding for unlinkable predicate proofs.
- **Standards interop** — presenting as a W3C Verifiable Credentials holder so existing issuers and verifiers work without bespoke wire formats.

Use [GitHub Discussions](../../discussions) for design questions, proposals, and critique. Public discussion is how the design gets better before code exists.

## Coming soon — code

Separate repositories for firmware, the web library, and the reference host library will appear here as they reach a state worth publishing. Code contribution guidance will live in those repos.

Do not send code PRs to this repo — it is documentation only.

## Security issues

See [SECURITY.md](SECURITY.md). Report privately, not via public issues.

## License

By contributing you agree your contribution is licensed under GPL-3.0, consistent with the rest of the project.
