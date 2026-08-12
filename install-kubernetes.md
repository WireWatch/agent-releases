# Install the Wirewatch agent on Kubernetes

This is the route for probing targets that only exist inside your cluster or on its network. If you
just need an agent on a single host, the [Docker route](./install-docker.md) is one command, and the
[binary route](./install-binary.md) covers hosts that cannot run containers at all.

The agent connects **outbound only**. No inbound port has to be opened, no Service is created, and
one agent serves one organisation.

## Before you start

You need a cluster, `kubectl` pointed at it, and a **default StorageClass** — the manifest requests a
1 GiB `ReadWriteOnce` volume without naming a class, so your cluster's default is used. Check with
`kubectl get storageclass`; exactly one should be marked `(default)`.

You will also need an **enrolment token**, which the app issues on its Agents screen. The token is
**single-use** and expires in an hour, so issue it when you are ready to apply, not before.

## The command

The app renders this with your values already filled in. It applies to whichever namespace your
current context selects — the manifest names no namespace of its own, so `kubectl create namespace
wirewatch && kubectl config set-context --current --namespace wirewatch` first if you want it
separate.

```
kubectl create secret generic wirewatch-agent \
  --from-literal=token=<your-token> \
  --from-literal=server=<your-server-url>

kubectl apply -f https://github.com/WireWatch/agent-releases/releases/download/agent-v0.1.1/agent-k8s.yaml
```

## Confirm it worked

```
kubectl get pods -l app.kubernetes.io/name=wirewatch-agent
kubectl logs -l app.kubernetes.io/name=wirewatch-agent -c agent
```

You are looking for one line:

```
level=INFO msg="agent enrolled" agentId=… server=… configDir=/var/lib/wirewatch
```

Your agent appears on the app's Agents screen within a minute, and monitors you assign to it start
reporting.

**Delete the pod once before you rely on it.**

```
kubectl delete pod -l app.kubernetes.io/name=wirewatch-agent
```

The replacement must log `agent already enrolled; ignoring --token`, and **must not** log a second
`agent enrolled`. That is the proof it resumed from the credential on the volume rather than trying
to re-enrol with a token that is by then spent. It is also the exact failure that `agent-v0.1.0` had,
so it is worth confirming rather than assuming.

## Verify what you are applying

The manifest is signed. To check it before it touches your cluster:

```
curl -fsSL -O https://github.com/WireWatch/agent-releases/releases/download/agent-v0.1.1/agent-k8s.yaml \
            -O https://github.com/WireWatch/agent-releases/releases/download/agent-v0.1.1/agent-k8s.yaml.cosign.bundle

cosign verify-blob --bundle agent-k8s.yaml.cosign.bundle \
  --certificate-identity https://github.com/WireWatch/wirewatch/.github/workflows/agent-release.yml@refs/tags/agent-v0.1.1 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  agent-k8s.yaml
```

`Verified OK` means the file came from the release workflow. Then `kubectl apply -f agent-k8s.yaml`
from the local copy rather than from the URL, so that what you apply is what you verified.

**A signature that does not verify means the manifest is not ours — do not apply it.**

## What the manifest does, and what not to change

**Your token goes into a Secret, not into the pod spec.** The container reads `AGENT_TOKEN` and
`AGENT_SERVER` from `secretKeyRef`. This is the one meaningful advantage of this route over Docker,
where the token lands in the process arguments and stays readable in `docker inspect`.

**`replicas: 1`, and never more.** One enrolled agent is one credential. Two pods sharing it are
duplicate execution for the same monitor and location, which the control plane's fencing-token check
rejects — the second pod's results are discarded and its probes are wasted. Scale by enrolling **more
agents**, never by adding replicas of one.

**`strategy: Recreate`, because the volume is `ReadWriteOnce`.** A rolling update would need two pods
holding the same volume at once, and on most storage backends the new pod simply never schedules.

**`fsGroupChangePolicy: OnRootMismatch` is load-bearing — do not delete it.** Without it the kubelet
re-applies group write to the volume on *every* mount. The agent's `0600` key becomes `0660`, the
agent refuses to load it, and every pod after the first dies on a token that is already spent.

**The `repair-key-permissions` init container is recovery, not ceremony.** It exists for volumes
enrolled under `agent-v0.1.0`, whose key is already `0660` and whose single-use token is already
gone. Delete it and those agents never come back. It runs as the pod's own non-root uid, which owns
the key — there is no root container anywhere in this manifest.

**Both images are pinned by digest**, not by tag, and both run `runAsNonRoot` with
`allowPrivilegeEscalation: false` and all capabilities dropped.

## If something goes wrong

**The pod is `Pending` and the PVC is `Pending`.** No default StorageClass, or none that can satisfy
`ReadWriteOnce` on the node the pod landed on. `kubectl describe pvc wirewatch-agent` names the
reason.

**`kubectl exec … -- sh` fails with `"sh": executable file not found in $PATH`.** The image is
distroless: it contains the agent and nothing else, deliberately. To inspect the credential, mount
the same PVC in a short-lived pod pinned to the same node, rather than trying to get a shell in this
one.

**`agent already enrolled; ignoring --token`, and you wanted a fresh enrolment.** The PVC outlives
the Deployment. Deleting and re-applying reuses the old identity — usually what you want. To
genuinely start over, delete the PVC and the Secret too, then issue a new token. Deleting only the
Deployment is the common mistake, and it looks like the new token was rejected when in fact it was
never read.

**`assignment fetch failed; keeping last-known set`.** The agent cannot reach the control plane. It
keeps probing whatever it was last assigned rather than going dark, and recovers on its own. If it
persists, check that the cluster can reach your server URL outbound on 443.

## Removing it

```
kubectl delete deployment wirewatch-agent
kubectl delete pvc wirewatch-agent
kubectl delete secret wirewatch-agent
```

**Revoke the agent in the app as well.** Deleting the workload stops it probing; it does not tell the
control plane the agent is gone, and an agent left enrolled but absent is one your account still
counts and your monitors may still be assigned to.
