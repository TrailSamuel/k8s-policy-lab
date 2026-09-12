# CIS Kubernetes Benchmark mapping

Each enforcing policy in this repo maps to one or more controls in section 5.2 (Pod Security
Standards) of the CIS Kubernetes Benchmark. Control numbers follow CIS Kubernetes Benchmark
v1.12.0, which uses Pod Security Admission numbering: 5.2.1 is "ensure that the cluster has at
least one active policy control mechanism" and the specific controls follow from 5.2.2.

This lab's cluster runs Kubernetes v1.37, which is newer than current benchmark coverage (the
latest release, v2.0.1 from June 2026, targets Kubernetes v1.34 to v1.35). The v1.12.0 numbering
is stable across the v1.7.x to v1.12.x line and is used here as the most recent confirmed 5.2
numbering. Re-verify against the benchmark version matching your target cluster before citing for
audit.

| Policy | CIS control | What it enforces |
|---|---|---|
| `disallow-privileged` | 5.2.2 Minimize the admission of privileged containers | Rejects any pod with `privileged: true` at pod or container level. |
| `disallow-host-namespaces` | 5.2.3 host PID, 5.2.4 host IPC, 5.2.5 host network | Rejects pods sharing the host's PID, IPC, or network namespace. |
| `disallow-host-path` | 5.2.12 Minimize the admission of HostPath volumes | Rejects any pod mounting a `hostPath` volume. |
| `require-run-as-nonroot` | 5.2.7 Minimize the admission of root containers | Requires `runAsNonRoot: true`. Rejects pods that set it false or omit it. |

## Notes

These policies enforce a subset of the CIS 5.2 Pod Security controls, chosen because each maps
directly to a misconfiguration demonstrated in Phases 1 and 2 of this lab. They are not a complete
CIS implementation. Natural next additions within 5.2 are 5.2.6 allowPrivilegeEscalation, 5.2.9 and
5.2.10 capabilities, and 5.2.13 host ports.

The 5.2 numbering shifts between benchmark versions. PSP era benchmarks (v1.6.x, Kubernetes v1.20
and earlier) start privileged containers at 5.2.1 and have no hostPath control. PSA era benchmarks
(v1.7.x onward) insert a policy mechanism control at 5.2.1, pushing everything down one which is
the numbering used here.

Each mapping is verified by the test suite in `tests/`, so a policy that stops enforcing its
control fails CI rather than silently drifting from the benchmark it claims to satisfy.
