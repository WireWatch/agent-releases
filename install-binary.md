# Install the Wirewatch agent from the release binary

This is the route for hosts that cannot run containers — corporate policy, an older distribution, an
appliance, a single-board machine on a factory floor. If your host *can* run containers, the Docker
route in the [Wirewatch app](https://app.wirewatch.io) is shorter: one line and no unit file.

The agent connects **outbound only**. No inbound port has to be opened, and one agent serves one
organisation.

## Before you start

**Install [`cosign`](https://docs.sigstore.dev/cosign/system_config/installation/) first.** It is a
prerequisite of this route, and the command below enforces it rather than asking: on a host without
cosign the block stops at the verification step and installs nothing.

You will also need an **enrolment token**, which the app issues on its Agents screen. The token is
**single-use**.

## The command

Replace `<your-token>` and `<your-server-url>` with the values the app shows you, then paste the
whole block. It is one `&&`-chain on purpose — see [what it guarantees](#what-the-chain-guarantees).

```
ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/') &&
curl -fsSL -O https://github.com/WireWatch/agent-releases/releases/download/agent-v0.1.1/agent_0.1.1_linux_$ARCH.tar.gz -O https://github.com/WireWatch/agent-releases/releases/download/agent-v0.1.1/SHA256SUMS -O https://github.com/WireWatch/agent-releases/releases/download/agent-v0.1.1/SHA256SUMS.cosign.bundle &&
cosign verify-blob --bundle SHA256SUMS.cosign.bundle --certificate-identity https://github.com/WireWatch/wirewatch/.github/workflows/agent-release.yml@refs/tags/agent-v0.1.1 --certificate-oidc-issuer https://token.actions.githubusercontent.com SHA256SUMS &&
sha256sum --ignore-missing -c SHA256SUMS &&
tar -xzf agent_0.1.1_linux_$ARCH.tar.gz &&
sudo install -m 0755 agent /usr/local/bin/wirewatch-agent &&
sudo timeout 90 sh -c 'f="/var/lib/wirewatch/agent.key /var/lib/wirewatch/agent.crt /var/lib/wirewatch/agent.json"; /usr/local/bin/wirewatch-agent --token=<your-token> --server=<your-server-url> --config-dir=/var/lib/wirewatch & p=$!; while ! ls $f >/dev/null 2>&1 && kill -0 $p 2>/dev/null; do sleep 1; done; kill $p 2>/dev/null; wait $p 2>/dev/null; ls $f >/dev/null 2>&1' &&
sudo tee /etc/systemd/system/wirewatch-agent.service >/dev/null <<'EOF' &&
[Unit]
Description=Wirewatch agent
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/wirewatch-agent --config-dir=/var/lib/wirewatch
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now wirewatch-agent
```

The archive holds a single bare `agent` — no nested directory. `./agent --version` prints `0.1.1`.

The `--certificate-identity` above names **this** release's tag and no other. Each release's notes
print the identity for that version; the app's enrolment screen renders it for whichever version it
is offering you.

## Confirm it worked

```
systemctl is-enabled wirewatch-agent    # enabled
systemctl is-active wirewatch-agent     # active
```

Your agent appears on the app's Agents screen within a minute, and monitors you assign to it start
reporting.

**Reboot the host once before you rely on it.** Two lines of that unit are the whole point of this
route: `WantedBy=multi-user.target` is what `systemctl enable` acts on, so the agent starts again
after a reboot with nobody logged in, and `Restart=always` brings it back from a crash, an OOM kill,
or a control plane that was briefly unreachable at boot. Without them the agent runs until the
machine does not, and the first power cut ends your monitoring quietly.

## What the chain guarantees

**The signature runs before the checksum, and that ordering matters.** A checksum fetched from the
same host as the artifact proves the download was not truncated. It proves nothing about
provenance — whoever can serve you a modified tarball can serve a matching `SHA256SUMS` beside it.
The signature is the control that answers that question, so it runs first and the checksum then runs
against a file the signature has already vouched for.

**Every step is `&&`-chained, and that is load-bearing.** If any step fails, nothing after it runs:
no binary is installed, no unit is written, and — the one that cannot be undone — **your single-use
enrolment token is not spent.** The same token still works once you have fixed the cause. That holds
for a missing cosign, a signature that does not verify, a checksum that does not match, and an
enrolment the control plane refuses.

**Your token never reaches the unit file.** Enrolment and execution are deliberately separate: the
`timeout 90 sh -c …` line runs the agent once in the foreground *with* the token, and ends as soon
as the credential is on disk. The unit installed on the next line carries **no `--token` at all** —
it starts from that credential. So the token is never written to `/etc/systemd/system`, and a
restart never needs it again.

That line ends when the **whole** credential is present — the private key, the leaf and the
metadata — because those are every file the next start needs in order to resume rather than
re-enrol. Its exit status is "the credential is complete", so a refused token stops the chain and no
unit is installed for a service that could never start.

## If something goes wrong

**`no file was verified`** from the checksum step means the archive never downloaded — usually a
wrong architecture or a network failure mid-transfer. Nothing was installed and your token is
unspent; fix the cause and paste the block again.

This is worth knowing because `curl` will not tell you: with several URLs it reports only the *last*
transfer's status, so a 404 on the archive can still exit `0`. The checksum step is what catches it.

**`--config-dir` must match in both places.** It appears in the enrolment line and in the unit's
`ExecStart`, and they must name the same directory. Enrolling into one and starting from another
spends the single-use token and leaves the service with nothing to resume from. The directory is
created `0700` with the key at `0600`, owned by root, since both steps run under `sudo`.

**A signature that does not verify means the file is not ours — do not run it.** See the
[verification instructions](./README.md#verify-what-you-downloaded) for how to check any release
asset on its own.
