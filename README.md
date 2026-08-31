# k8s-policy-lab

Learning Kubernetes admission control by building it up from nothing.

The idea: a default cluster will happily run a privileged container that mounts the host
filesystem. Nothing stops it. I want to understand the mechanism that does stop it, so I'm
starting with a throwaway local cluster and adding controls one at a time.

Built in the open, this repo is a work log, not a finished tool.

## Right now

A single node [kind](https://kind.sigs.k8s.io/) cluster and one deliberately privileged pod.

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

## Phase 2 - RBAC and token escalation

Split the cluster into `dev` and `prod` namespaces, each with a secret and gave `dev`'s
ServiceAccount a Role with one line too many: `create` on pods, alongside `get`/`list` on secrets.

Every pod gets its ServiceAccount token mounted at
`/var/run/secrets/kubernetes.io/serviceaccount/token` by default. So a compromised pod
inherits its SA's API permissions. From inside the pod using only that token:

- **Read every secret in the namespace.** The `data` values are base64, not encrypted so
  `base64 -d` reverses them in one command.
- **Hit the namespace boundary.** The same token requesting `prod` secrets gets a clean 403: a
  `Role` is namespaced so it can't cross into `prod`.
- **Create a privileged pod.** `create` on pods includes creating a privileged one, which
  chains straight into the Phase 1 node compromise.

The full chain: compromised pod → mounted token → read secrets → create privileged pod → root on
the node. A single over broad verb in a Role (`create` on pods) is the whole distance from read
some secrets to own the host.

The namespace boundary held for secrets but did nothing about pod creation, because that's a
different control. RBAC scopes what an identity can do, it doesn't stop a permitted action from
being dangerous. That's the gap admission control fills next.

Files: `rbac/`, `workloads/app-pod.yaml`.

## Where it's going

Rough order, subject to change as I learn what's actually interesting:

3. Kyverno policies that block each of the above at admission
4. Tests asserting every policy rejects the bad manifest and accepts the fixed one
5. Mapping the policies back to CIS Kubernetes Benchmark controls

## Note

Anything under `workloads/` is unsafe on purpose. Disposable local clusters only.