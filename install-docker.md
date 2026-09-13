# Install the Wirewatch agent with Docker

**The install steps for this route now live at
[docs.wirewatch.io/agents/docker/](https://docs.wirewatch.io/agents/docker/).** That page is built
from the same repository as the app's enrolment screen and moves with each release; this file used
to carry a second copy of it, which is a copy that drifts.

Read [the route overview](https://docs.wirewatch.io/agents/) first if you do not have a token yet —
it covers getting one, picking a route, and confirming the agent came up.

The notes below are kept here because they are not on the docs site.

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
