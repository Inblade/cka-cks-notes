# CKS: Runtime Security, Workload Hardening and Supply Chain

Workload-level controls: Pod Security Standards, seccomp, AppArmor, runtime detection with
Falco, image scanning and signing, and sandboxed runtimes. Kubernetes 1.30-1.33.

---

## 1. Pod Security Standards

Three profiles, enforced by the built-in Pod Security admission controller via **namespace
labels**. There is no PodSecurityPolicy any more.

| Profile | Intent | Blocks |
| --- | --- | --- |
| `privileged` | Unrestricted | Nothing. Only for infrastructure namespaces |
| `baseline` | Prevent known privilege escalations | `privileged`, hostNetwork/PID/IPC, hostPath, host ports, most capabilities, unsafe sysctls, `hostProcess` |
| `restricted` | Hardened, current best practice | Everything baseline blocks, plus enforces non-root, no privilege escalation, dropped capabilities, and a seccomp profile |

Three modes, each independent and each with its own version pin:

| Mode | Effect |
| --- | --- |
| `enforce` | Rejects violating pods at admission |
| `audit` | Allows the pod, records a violation annotation in the audit log |
| `warn` | Allows the pod, returns a warning to the user's client |

```bash
k label ns app \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.33 \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=v1.33 \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=v1.33 \
  --overwrite

k get ns app --show-labels
k label ns app pod-security.kubernetes.io/enforce-              # remove enforcement
```

A common rollout pattern is `enforce=baseline` with `audit=restricted,warn=restricted`: existing
workloads keep running while you collect the list of what would break.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.33
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.33
```

Enforcement applies to pods, not to controllers — a Deployment is admitted and its pods are
rejected. Check `k get rs -n app -o yaml | grep -i -A3 'FailedCreate'` or the Deployment events
when replicas never appear.

### What `restricted` requires

| Requirement | Field |
| --- | --- |
| Run as non-root | `securityContext.runAsNonRoot: true` |
| No privilege escalation | `securityContext.allowPrivilegeEscalation: false` |
| Drop all capabilities | `securityContext.capabilities.drop: ["ALL"]` (only `NET_BIND_SERVICE` may be added back) |
| Seccomp enabled | `securityContext.seccompProfile.type: RuntimeDefault` (or `Localhost`) |
| Not privileged | `securityContext.privileged: false` |
| No host namespaces | `hostNetwork`, `hostPID`, `hostIPC` all unset/false |
| Restricted volume types | No `hostPath`; `configMap`, `secret`, `emptyDir`, `pvc`, `projected`, `downwardAPI`, `ephemeral` are fine |
| No host ports | `ports[].hostPort` unset |
| Default or safe sysctls only | No unsafe `securityContext.sysctls` |

A pod that passes `restricted`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
  namespace: app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginxinc/nginx-unprivileged:1.27
      securityContext:
        allowPrivilegeEscalation: false
        privileged: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - {name: tmp, mountPath: /tmp}
        - {name: cache, mountPath: /var/cache/nginx}
        - {name: run, mountPath: /var/run}
  volumes:
    - {name: tmp, emptyDir: {}}
    - {name: cache, emptyDir: {}}
    - {name: run, emptyDir: {}}
```

`readOnlyRootFilesystem` is not required by `restricted`, but it is nearly free once you have
mapped the writable paths and it removes most in-container persistence.

```bash
# See exactly why a pod was rejected
k apply -f pod.yaml            # the admission error lists every violated field
k -n app get events --field-selector reason=FailedCreate
k label ns app pod-security.kubernetes.io/warn=restricted --overwrite   # dry-run everything
k -n app run test --image=nginx --dry-run=server -o yaml                # server-side check
```

---

## 2. seccomp

seccomp filters the syscalls a container may make. The default for a plain pod is
`Unconfined` — no filter at all.

| Type | Meaning |
| --- | --- |
| `RuntimeDefault` | The container runtime's built-in profile; blocks ~50 dangerous syscalls. The right default |
| `Localhost` | A JSON profile file on the node, referenced by relative path |
| `Unconfined` | No filtering |

```yaml
spec:
  securityContext:                 # pod-level, inherited by all containers
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: app:1.0
      securityContext:             # container-level overrides pod-level
        seccompProfile:
          type: Localhost
          localhostProfile: profiles/audit.json
```

Profiles live under the kubelet's seccomp root, `/var/lib/kubelet/seccomp/`, so
`localhostProfile: profiles/audit.json` resolves to
`/var/lib/kubelet/seccomp/profiles/audit.json` on the node running the pod. The file must
exist on **every** node the pod may land on, or the pod fails with
`cannot load seccomp profile`.

```bash
mkdir -p /var/lib/kubelet/seccomp/profiles
```

An original deny-by-default profile that allows only what a trivial static server needs:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    {
      "names": [
        "accept4", "bind", "brk", "close", "epoll_create1", "epoll_ctl",
        "epoll_pwait", "exit", "exit_group", "fcntl", "fstat", "futex",
        "getdents64", "getpid", "getsockname", "listen", "lseek", "mmap",
        "mprotect", "munmap", "nanosleep", "newfstatat", "openat", "read",
        "recvfrom", "rt_sigaction", "rt_sigprocmask", "sendto", "set_robust_list",
        "setsockopt", "socket", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

An audit-only profile that logs everything instead of blocking — the practical way to build an
allow list without breaking the workload:

```json
{
  "defaultAction": "SCMP_ACT_LOG"
}
```

Actions: `SCMP_ACT_ERRNO` (fail the call), `SCMP_ACT_LOG` (allow and log to the kernel audit
log), `SCMP_ACT_ALLOW`, `SCMP_ACT_KILL` (kill the process), `SCMP_ACT_TRACE`.

### Testing a profile

```bash
# 1. Deploy with the audit profile, exercise the app, then read what it actually used
grep -h 'SECCOMP' /var/log/syslog | tail -50
ausearch -m SECCOMP -ts recent 2>/dev/null | tail

# 2. Confirm the profile is attached inside the container
k exec -it hardened -n app -- grep Seccomp /proc/1/status
#   Seccomp: 2       -> filter mode active (0 = disabled, 1 = strict, 2 = filter)
#   Seccomp_filters: 1

# 3. Confirm a blocked syscall really fails
k exec -it hardened -n app -- chmod 777 /tmp      # expect "Operation not permitted"
```

`RuntimeDefault` can also be made the node-wide default with the kubelet flag
`--seccomp-default=true` (or `seccompDefault: true` in the kubelet config), which applies it
to every container that does not set its own profile.

---

## 3. AppArmor

AppArmor confines a process by path-based file, capability, and network rules. It is a Linux
LSM, so profiles are loaded on the node, not in Kubernetes.

```bash
# Node side
aa-status
apparmor_status | head -20
cat /sys/module/apparmor/parameters/enabled          # expect "Y"

apparmor_parser -q /etc/apparmor.d/k8s-deny-write    # load
apparmor_parser -R /etc/apparmor.d/k8s-deny-write    # unload
apparmor_parser -r /etc/apparmor.d/k8s-deny-write    # replace/reload
aa-status | grep k8s-
```

An original profile that blocks writes anywhere under `/etc` inside the container:

```
#include <tunables/global>

profile k8s-deny-etc-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,
  network,
  capability,

  deny /etc/** w,
  deny /etc/** l,
  deny mount,
  deny /proc/sys/** w,
}
```

Referencing it from a pod — the annotation form is deprecated, the field is GA since 1.30:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: confined
  namespace: app
spec:
  securityContext:
    appArmorProfile:
      type: Localhost                 # RuntimeDefault | Localhost | Unconfined
      localhostProfile: k8s-deny-etc-write
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        appArmorProfile:
          type: Localhost
          localhostProfile: k8s-deny-etc-write
```

Legacy annotation form, still accepted on older clusters, useful to recognise:

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-etc-write
```

Verify:

```bash
k exec -it confined -n app -- cat /proc/1/attr/current     # k8s-deny-etc-write (enforce)
k exec -it confined -n app -- touch /etc/probe             # Permission denied
k describe po confined -n app | grep -i apparmor
```

If the profile is not loaded on the node the pod schedules to, the pod fails to start with a
`Blocked` / `cannot enforce AppArmor profile` message. Profiles must be distributed to every
node — in production via a DaemonSet or the node image, not by hand.

| Comparison | seccomp | AppArmor |
| --- | --- | --- |
| Filters | Syscalls | File paths, capabilities, network, mounts |
| Scope | Per container | Per container |
| Distribution | `/var/lib/kubelet/seccomp/` | Kernel-loaded via `apparmor_parser` |
| Kubernetes field | `securityContext.seccompProfile` | `securityContext.appArmorProfile` |
| Availability | Everywhere | Debian/Ubuntu/SUSE; SELinux is the RHEL equivalent |

---

## 4. Falco: runtime threat detection

Falco consumes kernel-level syscall events and evaluates them against rules in userspace.

```
        containers / host processes
                  |
     syscalls -> driver (modern eBPF probe | kmod | eBPF)
                  |
             Falco userspace
        rules + macros + lists  ->  outputs (stdout, file, gRPC, HTTP, Falcosidekick)
```

Driver choice: `modern_ebpf` (CO-RE, no kernel headers, preferred), `ebpf` (legacy probe),
`kmod` (kernel module). Falco can also read Kubernetes audit events as a second source.

```bash
systemctl status falco falco-modern-bpf
journalctl -u falco -f
falco --version
falco -L                                  # list all loaded rules
falco -l "Terminal shell in container"    # show one rule
falco --validate /etc/falco/falco_rules.local.yaml
falco -r /etc/falco/falco_rules.local.yaml -o json_output=true
systemctl restart falco
```

| Path | Contents |
| --- | --- |
| `/etc/falco/falco.yaml` | Daemon config: outputs, driver, buffered_outputs, `rules_files` list |
| `/etc/falco/falco_rules.yaml` | Upstream rules. Do not edit; it is replaced on upgrade |
| `/etc/falco/falco_rules.local.yaml` | Your rules and overrides. Loaded last, so it wins |
| `/etc/falco/rules.d/` | Drop-in directory for additional rule files |
| `/var/log/falco.log` / journal | Alert output, depending on config |

### Rule anatomy

```yaml
- rule: <unique name>
  desc: <what and why>
  condition: <sysdig-style filter expression>
  output: <alert text with %fields>
  priority: <EMERGENCY|ALERT|CRITICAL|ERROR|WARNING|NOTICE|INFORMATIONAL|DEBUG>
  tags: [list]
  enabled: true
```

Supporting constructs: `- macro:` (a named reusable condition) and `- list:` (a named set of
values). Append to an existing rule or list with `append: true`.

### Original custom rules

```yaml
# /etc/falco/falco_rules.local.yaml
- list: allowed_shell_images
  items: ["docker.io/library/busybox", "registry.example.internal/ops/debug-toolbox"]

- macro: interactive_shell
  condition: >
    proc.name in (bash, sh, dash, ash, zsh, ksh)
    and evt.type in (execve, execveat)
    and evt.dir = <

- rule: Interactive shell spawned in application container
  desc: >
    A shell process started inside a running container that is not an approved
    debug image. Application containers should never spawn interactive shells at
    runtime; this is a strong indicator of an intrusion or of manual access that
    bypasses the deployment pipeline.
  condition: >
    interactive_shell
    and container.id != host
    and not container.image.repository in (allowed_shell_images)
  output: >
    Shell spawned in container
    (user=%user.name uid=%user.uid shell=%proc.name parent=%proc.pname
     cmdline=%proc.cmdline container=%container.name image=%container.image.repository:%container.image.tag
     k8s_ns=%k8s.ns.name k8s_pod=%k8s.pod.name)
  priority: WARNING
  tags: [container, shell, mitre_execution]

- rule: Write below etc in container
  desc: >
    A process inside a container opened a file under /etc for writing. Container
    filesystems should be immutable at runtime; configuration arrives via mounted
    ConfigMaps and Secrets, never by editing /etc in place.
  condition: >
    open_write
    and container.id != host
    and fd.name startswith /etc
    and not proc.name in (dpkg, apt, apt-get, rpm, yum, microdnf)
  output: >
    Write under /etc in container
    (file=%fd.name proc=%proc.name cmdline=%proc.cmdline user=%user.name
     container=%container.name image=%container.image.repository
     k8s_ns=%k8s.ns.name k8s_pod=%k8s.pod.name)
  priority: ERROR
  tags: [filesystem, container, mitre_persistence]

- rule: Package manager executed in running container
  desc: >
    Installing packages at runtime defeats image scanning and immutability. It
    usually means an attacker is staging tooling inside a compromised pod.
  condition: >
    spawned_process
    and container.id != host
    and proc.name in (apt, apt-get, apk, yum, dnf, microdnf, pip, pip3, npm, gem)
  output: >
    Package manager run in container
    (proc=%proc.cmdline user=%user.name container=%container.name
     image=%container.image.repository k8s_ns=%k8s.ns.name k8s_pod=%k8s.pod.name)
  priority: NOTICE
  tags: [container, software_mgmt]
```

Test by triggering the behaviour and watching the output:

```bash
falco --validate /etc/falco/falco_rules.local.yaml
systemctl restart falco
k exec -it web-0 -n app -- sh -c 'echo x'          # should fire the shell rule
journalctl -u falco -f | grep -i 'Shell spawned'
```

### Falco versus audit logs

| | Kubernetes audit log | Falco |
| --- | --- | --- |
| Source | API server requests | Kernel syscalls on the node |
| Sees | Who asked the API for what | What processes actually did inside containers |
| Misses | Anything not going through the API | Nothing syscall-visible, but no API context by itself |
| Example it catches | `kubectl exec` was invoked by user X | The shell that `exec` started, and every command it ran |

They are complements, not alternatives. Audit answers "who requested it"; Falco answers "what
then happened on the node".

---

## 5. Image scanning and supply chain security

### Trivy

```bash
trivy image nginx:1.27
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed myapp:1.4.2
trivy image --scanners vuln,secret,misconfig myapp:1.4.2
trivy image --format json --output report.json myapp:1.4.2
trivy image --input myapp.tar                       # offline, from docker save
trivy fs --severity HIGH,CRITICAL /src              # source tree: deps, secrets, IaC
trivy repo https://github.com/example/app
trivy config ./deploy                               # Kubernetes/Terraform/Dockerfile misconfig
trivy k8s --report summary cluster                  # scan running workloads
trivy image --download-db-only                      # pre-warm the DB before an air-gapped run
```

`--exit-code 1` is what turns a scan into a gate — the CI step fails when findings at the
requested severity exist. `--ignore-unfixed` keeps the gate actionable.

### SBOM

```bash
trivy image --format cyclonedx --output sbom.cdx.json myapp:1.4.2
trivy image --format spdx-json --output sbom.spdx.json myapp:1.4.2
trivy sbom sbom.cdx.json --severity HIGH,CRITICAL   # rescan an SBOM as new CVEs land
syft myapp:1.4.2 -o cyclonedx-json > sbom.cdx.json
grype sbom:sbom.cdx.json --fail-on high
```

An SBOM's value is that you can re-scan yesterday's build against today's vulnerability data
without rebuilding it.

### Signing with cosign

```bash
cosign generate-key-pair                                    # cosign.key / cosign.pub
cosign sign --key cosign.key registry.example.com/app:1.4.2
cosign verify --key cosign.pub registry.example.com/app:1.4.2

# Keyless (OIDC identity + transparency log), the modern default
cosign sign registry.example.com/app:1.4.2
cosign verify \
  --certificate-identity-regexp='^https://github\.com/example/app/' \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  registry.example.com/app:1.4.2

# Attach and verify an SBOM or provenance attestation
cosign attest --key cosign.key --type cyclonedx --predicate sbom.cdx.json \
  registry.example.com/app:1.4.2
cosign verify-attestation --key cosign.pub --type cyclonedx registry.example.com/app:1.4.2

cosign triangulate registry.example.com/app:1.4.2            # where the signature lives
cosign copy registry.example.com/app:1.4.2 mirror.internal/app:1.4.2
```

Always sign and verify by **digest**. A tag is mutable, so a tag-based signature check can be
defeated by repointing the tag.

### Enforcing at admission

Three approaches, in increasing order of practicality:

```yaml
# 1. ImagePolicyWebhook - built into the API server. Enable the plugin and give it a config.
#    /etc/kubernetes/admission/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /etc/kubernetes/admission/imagepolicy-kubeconfig.yaml
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false        # fail closed; true means a webhook outage allows everything
```

```yaml
    # kube-apiserver flags
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
    - --admission-control-config-file=/etc/kubernetes/admission/admission-config.yaml
```

```yaml
# 2. Kyverno - verify signatures at admission and mutate on the way in
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: verify-app-images
      match:
        any:
          - resources:
              kinds: ["Pod"]
              namespaces: ["app", "prod"]
      verifyImages:
        - imageReferences: ["registry.example.com/app/*"]
          mutateDigest: true          # rewrite the tag to the resolved digest
          verifyDigest: true
          required: true
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      <cosign public key>
                      -----END PUBLIC KEY-----
```

```yaml
# 3. OPA Gatekeeper - constraint templates in Rego
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: only-internal-registry
spec:
  match:
    kinds:
      - {apiGroups: [""], kinds: ["Pod"]}
    excludedNamespaces: ["kube-system"]
  parameters:
    repos: ["registry.example.com/"]
```

### Image hygiene

| Practice | Why |
| --- | --- |
| Distroless or `scratch` base images | No shell, no package manager, no `curl` — most post-exploitation tooling is simply absent |
| Alpine / `-slim` where distroless is impractical | Smaller attack surface than a full distro base |
| Multi-stage builds | Compilers, build secrets and dev dependencies never reach the runtime layer |
| Pin by digest, not tag | `image: app@sha256:abcd...` is immutable; `app:latest` is whatever someone pushed last |
| `imagePullPolicy: Always` for mutable tags | Otherwise a node keeps serving a stale, possibly vulnerable cached layer |
| `imagePullPolicy: IfNotPresent` for digests | Digests are immutable, so re-pulling buys nothing |
| Non-root `USER` in the Dockerfile | Required for `runAsNonRoot: true` to work without a numeric override |
| No secrets in layers | `trivy image --scanners secret` catches the obvious ones; layers are forever |

```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: registry.example.com/app@sha256:5f8c9a2b1e4d7c3a9f0b6e2d8c4a1f7b3e9d0c6a2f8b4e1d7c3a9f0b6e2d8c4a
      imagePullPolicy: IfNotPresent
```

```bash
k create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=ci --docker-password="$TOKEN" -n app
k patch sa default -n app -p '{"imagePullSecrets":[{"name":"regcred"}]}'

# Resolve a tag to its digest before pinning
crane digest registry.example.com/app:1.4.2
docker buildx imagetools inspect registry.example.com/app:1.4.2
```

---

## 6. Runtime sandboxing with RuntimeClass

For untrusted or multi-tenant workloads, replace the shared-kernel container boundary with a
stronger one. gVisor intercepts syscalls in a userspace kernel; Kata runs each pod in a
lightweight VM.

Node side, containerd `/etc/containerd/config.toml`:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
```

```bash
systemctl restart containerd
crictl info | jq '.config.containerd.runtimes | keys'
```

Kubernetes side:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc                     # must match the containerd runtime name
scheduling:
  nodeSelector:
    sandbox: "gvisor"              # only nodes that actually have runsc installed
  tolerations:
    - {key: sandbox, operator: Equal, value: gvisor, effect: NoSchedule}
overhead:
  podFixed:
    cpu: "50m"
    memory: "64Mi"
---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata: {name: kata}
handler: kata
---
apiVersion: v1
kind: Pod
metadata: {name: untrusted, namespace: tenant-a}
spec:
  runtimeClassName: gvisor
  containers:
    - name: app
      image: untrusted-app:1.0
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: {drop: ["ALL"]}
```

```bash
k get runtimeclass
k exec -it untrusted -n tenant-a -- dmesg | head    # gVisor reports its own kernel identity
k exec -it untrusted -n tenant-a -- uname -r
```

`scheduling.nodeSelector` matters: without it, a sandboxed pod can land on a node with no
`runsc` handler and fail with `RunContainerError: failed to get sandbox runtime`.
`overhead` makes the scheduler account for the sandbox's own resource cost.

---

## 7. Runtime hardening checklist

| Control | Verify with |
| --- | --- |
| Namespace enforces PSS `restricted` | `k get ns <ns> --show-labels` |
| No privileged pods anywhere | `k get po -A -o json \| jq -r '.items[] \| select(.spec.containers[].securityContext.privileged==true) \| .metadata.name'` |
| No host namespaces | `k get po -A -o json \| jq -r '.items[] \| select(.spec.hostNetwork or .spec.hostPID or .spec.hostIPC) \| "\(.metadata.namespace)/\(.metadata.name)"'` |
| No hostPath mounts | `k get po -A -o json \| jq -r '.items[] \| select(.spec.volumes[]?.hostPath) \| "\(.metadata.namespace)/\(.metadata.name)"'` |
| seccomp active in the container | `k exec <pod> -- grep Seccomp /proc/1/status` |
| AppArmor profile applied | `k exec <pod> -- cat /proc/1/attr/current` |
| Images pinned by digest | `k get po -A -o jsonpath='{range .items[*].spec.containers[*]}{.image}{"\n"}{end}' \| grep -v '@sha256:'` |
| Service account token not mounted | `k get po <pod> -o jsonpath='{.spec.automountServiceAccountToken}'` |
| Falco running on every node | `k -n falco get ds` / `systemctl status falco` |
| Signatures verified at admission | Apply an unsigned image and confirm it is rejected |
