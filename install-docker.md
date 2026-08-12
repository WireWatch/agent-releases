# Install the Wirewatch agent with Docker

This is the shortest route: one command, no unit file, no build. If your host cannot run containers,
use the [binary route](./install-binary.md) instead. If you are deploying onto Kubernetes, use the
[Kubernetes route](./install-kubernetes.md), which keeps your token out of the process arguments.

The agent connects **outbound only**. No inbound port has to be opened, and one agent serves one
organisation.

## Before you start

You need Docker, and a user that can talk to it — either a member of the `docker` group or `sudo` in
front of the command. Check with `docker ps`; if that works without `sudo`, the command below works
as written.

You will also need an **enrolment token**, which the app issues on its Agents screen. The token is
**single-use** and expires in an hour, so issue it when you are ready to run the command, not before.

## The command

Replace `<your-token>` and `<your-server-url>` with the values the app shows you — the app renders
this block with your values already filled in.

```
mkdir -p "$HOME/.wirewatch-agent"
docker run -d --name wirewatch-agent --restart unless-stopped \
  --user "$(id -u):$(id -g)" \
  -v "$HOME/.wirewatch-agent:/var/lib/wirewatch" \
  ghcr.io/wirewatch/agent:0.1.1 \
  --token=<your-token> --server=<your-server-url> --config-dir=/var/lib/wirewatch
```

## Confirm it worked

```
docker logs wirewatch-agent
```

You are looking for one line:

```
level=INFO msg="agent enrolled" agentId=… server=… configDir=/var/lib/wirewatch
```

Your agent appears on the app's Agents screen within a minute, and monitors you assign to it start
reporting. `ls -l "$HOME/.wirewatch-agent"` should show four files, with `agent.key` at `0600`.

**Restart it once before you rely on it.**

```
docker restart wirewatch-agent && sleep 5 && docker logs --tail 2 wirewatch-agent
```

The second start must say `agent already enrolled; ignoring --token`. That is the proof that the
credential on the volume is what the agent resumes from — not the token, which by then is spent. An
agent that re-enrolled instead would be an agent that cannot survive a reboot.

## Know where your token ends up

**The token is passed as a process argument, and Docker keeps process arguments forever.** After
enrolment it is spent and useless, but until you remove the container it stays readable:

```
docker inspect wirewatch-agent --format '{{.Config.Cmd}}'
```

This is worth knowing rather than discovering. On a host where other people can run `docker inspect`
and your token has not yet been used — the window between issuing it and the container starting —
that is a live credential in a readable place. The window is small and the token is single-use, but
if it matters to you, the [Kubernetes route](./install-kubernetes.md) puts the token in a Secret
instead, and the [binary route](./install-binary.md) keeps it out of the persistent unit file.

## Verify what you are running

The image is signed. To check it before you run it:

```
cosign verify ghcr.io/wirewatch/agent:0.1.1 \
  --certificate-identity https://github.com/WireWatch/wirewatch/.github/workflows/agent-release.yml@refs/tags/agent-v0.1.1 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

Note this is `cosign verify`, not the `cosign verify-blob` the binary route uses — one verifies an
image in a registry, the other a file you downloaded. Output names the workflow, the tag and the
source commit that built it, and reports the image digest. That digest is the same one the
Kubernetes manifest pins, so all three routes run identical bytes.

`--certificate-identity` names **this** release's tag and no other. Each release's notes print the
identity for that version.

**A signature that does not verify means the image is not ours — do not run it.**

## What the flags are for

**`--restart unless-stopped`** is what brings the agent back after a reboot or a crash. Without it
the first power cut ends your monitoring quietly. It stays stopped only if *you* stopped it.

**`--user "$(id -u):$(id -g)"`** runs the agent as you rather than as root, and it must match the
ownership of the volume directory. That is why `mkdir -p` comes first: created by Docker instead, the
directory would be owned by root and the agent could not write its credential.

**The volume is the whole point.** `$HOME/.wirewatch-agent` holds the private key, the leaf
certificate, the CA chain and the agent's identity. Remove it and the agent has nothing to resume
from; the next start needs a fresh token because the old one is spent. Back it up or do not — but
know that deleting it is not a reset, it is a re-enrolment.

**`--config-dir` must match the volume's mount point.** They both say `/var/lib/wirewatch` above.
Point them at different directories and the agent enrols into one and looks in the other, spending
the single-use token for nothing.

## If something goes wrong

**`agent already enrolled; ignoring --token`, and you wanted a fresh enrolment.** The volume outlives
the container. Removing and re-running the container reuses the old identity — which is usually what
you want. To genuinely start over, remove both:

```
docker rm -f wirewatch-agent
rm -rf "$HOME/.wirewatch-agent"
```

then issue a new token. Doing only the first half is the common mistake, and it looks like the new
token was rejected when in fact it was never read.

**The agent starts and then logs `assignment fetch failed; keeping last-known set`.** It cannot
reach the control plane. The agent keeps probing whatever it was last assigned rather than going
dark, so a brief outage costs you nothing, and it recovers on its own. If it persists, check that the
host can reach your server URL outbound on 443.

**Permission errors on the key.** The volume directory must be owned by the same uid the container
runs as. If you created it with `sudo` or moved it between hosts, `chown -R "$(id -u):$(id -g)"
"$HOME/.wirewatch-agent"` puts it back.
