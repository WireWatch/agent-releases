# Wirewatch agent

The Wirewatch agent probes services inside your own network — private APIs, internal
hosts, anything the public internet cannot reach — and reports the results to Wirewatch.
One agent serves one organisation: it enrols with a single-use token, authenticates with
mTLS, and connects outbound only, so no inbound port has to be opened.

This repository holds released agent builds and nothing else.

**Your enrolment token comes from the Wirewatch app: https://app.wirewatch.io** — the
enrolment screen issues it and shows the exact command for your platform. Container
image: `ghcr.io/wirewatch/agent`.

The three install routes are documented at
**[docs.wirewatch.io/agents](https://docs.wirewatch.io/agents/)**, which is public — read them
before you have an account open and decide which one you want:

**[Docker](https://docs.wirewatch.io/agents/docker/)** — one command, no unit file. The shortest
route, and the right one for most hosts.

**[Kubernetes](https://docs.wirewatch.io/agents/kubernetes/)** — for probing targets that only exist
inside your cluster or on its network. Your token goes into a Secret rather than into process
arguments.

**[Release binary](https://docs.wirewatch.io/agents/binary/)** — the route for hosts that cannot run
containers, with a systemd unit that survives a reboot.

Those pages are built from the same repository as the app's enrolment screen, so they move with each
release. The files in this repository ([Docker](./install-docker.md),
[Kubernetes](./install-kubernetes.md), [binary](./install-binary.md)) now hold only the
per-route troubleshooting notes that are not on the docs site.

All three run identical bytes: the container image the first two use is the same digest,
and it is built from the same release as the binary.

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
