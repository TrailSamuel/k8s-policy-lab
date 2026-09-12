# k8s-policy-lab

![Kyverno Policy Tests](https://github.com/TrailSamuel/k8s-policy-lab/actions/workflows/kyverno-test.yaml/badge.svg)

Learning Kubernetes admission control by building it up from nothing.

The idea: a default cluster will happily run a privileged container that mounts the host
filesystem. Nothing stops it. I want to understand the mechanism that does stop it, so I'm
starting with a throwaway local cluster and adding controls one at a time.

Built in the open, this repo is a work log, not a finished tool.

## Phase 1

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

## Phase 3 - admission control with Kyverno

Phases 1 and 2 both ended with nothing stopping it. This phase adds the thing that does.

[Kyverno](https://kyverno.io/) is an admission controller. It runs as pods in the cluster and
intercepts every manifest on its way to the API server, checking it against policies before the
object is created. RBAC decides whether an identity may perform an action. Admission control
decides whether the specific object is acceptable. A request has to pass both, so a ServiceAccount
fully authorised to create pods can still have a privileged pod rejected.

The first policy `policies/disallow-privileged.yaml`, blocks the exact manifest from Phase 1. It
rejects any pod with `privileged: true` at either the pod or container level. Applying the Phase 1
privileged pod now returns:

    admission webhook "validate.kyverno.svc-fail" denied the request
    disallow-privileged: Privileged containers are not allowed.

The same manifest that gave a root shell on the node two phases ago is now refused at admission,
before it schedules.

`validationFailureAction: Enforce` rejects violations outright. Set to `Audit`, the same policy
logs violations without blocking. Audit first is the production default, since an enforce policy
dropped onto a cluster with running workloads can block legitimate deployments before you know what
breaks. This lab uses Enforce because it is disposable and the rejection is the point.

A policy is only useful if it blocks the bad pod without blocking good ones. A plain
non privileged pod still creates normally which is where Phase 4 will assert automatically.

Files: `policies/disallow-privileged.yaml`.

## Phase 4 - automated policy tests in CI

Phase 3 verified the policy by hand: apply a bad pod, watch it get rejected. This phase makes that
check automatic and repeatable so a broken policy can't pass unnoticed.

The Kyverno CLI runs policy tests without a cluster. A test definition
(`tests/kyverno-test.yaml`) pairs each policy with sample resources and the result each should
produce: the compliant pod passes the privileged pod fails admission. Running `kyverno test tests/`
asserts both.

    Test Summary: 2 tests passed and 0 tests failed

The same test runs in CI on every push. `.github/workflows/kyverno-test.yaml` spins up a clean
Linux runner, installs the pinned CLI version and runs the suite. If a policy ever stops blocking
what it should the workflow fails before the change merges. CI first caught a stale action version
this wa ywhich is the point, a clean machine surfaces problems a local setup hides.

Files: `tests/`, `.github/workflows/kyverno-test.yaml`.

## Phase 5 - mapping to the CIS Benchmark

The CIS Kubernetes Benchmark is the industry checklist for locking down a cluster. This phase adds
three more policies so the lab covers a real slice of it then maps each policy to the exact control
it satisfies.

Three policies join the privileged-pod one from Phase 3:

- `disallow-host-namespaces` blocks pods that share the node's network, process, or IPC space.
- `disallow-host-path` blocks pods that mount the node's filesystem.
- `require-run-as-nonroot` blocks pods that run as root.

All four run in the same test suite and the same CI check. Writing the non root policy turned up a
useful bug: the first version only caught pods that explicitly asked to run as root, and let through
pods that simply said nothing which is the more common case. The test caught it before it ever
looked like it was working.

The mapping itself lives in `docs/cis-mapping.md`. Each policy is tied to its CIS control, with the
benchmark version stated up front, because the control numbers shift between versions and a mapping
that does not say which version it targets is not worth much.

Files: `policies/`, `tests/`, `docs/cis-mapping.md`.

## Where it's going

Rough order, subject to change as I learn what's actually interesting:

5. Mapping the policies back to CIS Kubernetes Benchmark controls

## Note

Anything under `workloads/` is unsafe on purpose. Disposable local clusters only.