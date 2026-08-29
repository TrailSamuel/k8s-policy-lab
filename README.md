# k8s-policy-lab

Learning Kubernetes admission control by building it up from nothing.

The idea: a default cluster will happily run a privileged container that mounts the host
filesystem. Nothing stops it. I want to understand the mechanism that does stop it, so I'm
starting with a throwaway local cluster and adding controls one at a time.

Built in the open, this repo is a work log, not a finished tool.

## Right now

A single-node [kind](https://kind.sigs.k8s.io/) cluster and one deliberately privileged pod.

```bash
kind create cluster --config cluster/kind-config.yaml --name policy-lab
kubectl apply -f workloads/privileged-pod.yaml
kubectl get pod privileged-pod          # Running. Nothing objected.
```

The pod requests `privileged: true` and mounts the node's root filesystem at `/host`. Both are
supported, documented Kubernetes features and a default cluster has no opinion about either, the
API server validated the manifest and scheduled it.

What that buys an attacker:

```bash
kubectl exec -it privileged-pod -- sh
ls /host                  # the node's root filesystem
cat /host/etc/shadow      # readable
chroot /host sh
whoami                    # root
```

No CVE, no exploit, no escape technique. A manifest anyone with `create pod` permission could
submit results in root on the node.

**Scope:** kind runs each node as a Docker container, so `/host` is that container's filesystem,
not the laptop's. The compromise is real within the cluster's trust boundary and stops there. On a
real cluster the same manifest reaches the actual node, kubelet credentials, other pods' secrets,
the container runtime socket. Same mechanism, larger attack surface.

Tear down with `kind delete cluster --name policy-lab`.
Requires Docker running, plus kind and kubectl.

## Where it's going

Rough order, subject to change as I learn what's actually interesting:

1. More misconfigured workloads, host networking, root containers, missing resource limits, mutable image tags
2. RBAC and namespaces, including an over-permissioned ServiceAccount to show what escalation looks like
3. Kyverno policies that block each of the above at admission
4. Tests asserting every policy rejects the bad manifest and accepts the fixed one
5. Mapping the policies back to CIS Kubernetes Benchmark controls

## Note

Anything under `workloads/` is unsafe on purpose. Disposable local clusters only.