# Install the Wirewatch agent on Kubernetes

**The install steps for this route now live at
[docs.wirewatch.io/agents/kubernetes/](https://docs.wirewatch.io/agents/kubernetes/).** That page is
built from the same repository as the app's enrolment screen and moves with each release; this file
used to carry a second copy of it, which is a copy that drifts.

Read [the route overview](https://docs.wirewatch.io/agents/) first if you do not have a token yet —
it covers getting one, picking a route, and confirming the agent came up.

The notes below are kept here because they are not on the docs site.

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
