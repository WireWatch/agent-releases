# Wirewatch agent

The Wirewatch agent probes services inside your own network — private APIs, internal
hosts, anything the public internet cannot reach — and reports the results to Wirewatch.
One agent serves one organisation: it enrols with a single-use token, authenticates with
mTLS, and connects outbound only, so no inbound port has to be opened.

This repository holds released agent builds and nothing else.

**Installation and enrolment instructions live in the Wirewatch app:
https://app.wirewatch.io** — the enrolment screen issues your token and shows the exact
command for your platform. Container image: `ghcr.io/wirewatch/agent`.

## Verify what you downloaded

Each release carries a `SHA256SUMS` file, and each asset carries its own
`.cosign.bundle` signature.

Checksums:

```
sha256sum -c SHA256SUMS
```

Signatures are keyless — there is no public key to trust, and no key of ours to leak.
Every release's notes print the command in full, including the
`--certificate-identity` for that version:

```
cosign verify-blob \
  --bundle SHA256SUMS.cosign.bundle \
  --certificate-identity <the identity printed in this release's notes> \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  SHA256SUMS
```

Each archive verifies the same way against its own bundle. A signature that does not
verify means the file is not ours — do not run it.
