# CKS: Cluster Setup and Cluster Hardening

Control-plane and identity-layer controls: API server configuration, auditing, encryption at
rest, RBAC review, service account hygiene, kubelet hardening, and network default-deny.
All paths assume a `kubeadm` cluster on Kubernetes 1.30-1.33.

---

## 1. kube-apiserver flags that matter

Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`. It is a static pod: save the file and
the kubelet restarts the API server. A typo here takes the cluster down, so keep a copy.

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
vi /etc/kubernetes/manifests/kube-apiserver.yaml
watch -n2 'crictl ps | grep apiserver'
k get --raw='/readyz?verbose'
```

| Flag | Recommended value | Why |
| --- | --- | --- |
| `--anonymous-auth` | `false` | Removes the `system:anonymous` / `system:unauthenticated` identity |
| `--authorization-mode` | `Node,RBAC` | Never `AlwaysAllow`. `Node` restricts kubelets to their own objects |
| `--enable-admission-plugins` | `NodeRestriction,...` | `NodeRestriction` stops a kubelet editing other nodes/pods and self-labelling into privileged label prefixes |
| `--disable-admission-plugins` | remove nothing critical | Verify `ServiceAccount` and `ValidatingAdmissionPolicy` remain enabled |
| `--audit-policy-file` | `/etc/kubernetes/audit/policy.yaml` | Without a policy, no auditing happens |
| `--audit-log-path` | `/var/log/kubernetes/audit/audit.log` | `-` means stdout; a file is preferred |
| `--audit-log-maxage` | `30` | Days of retention |
| `--audit-log-maxbackup` | `10` | Number of rotated files |
| `--audit-log-maxsize` | `100` | MB per file before rotation |
| `--encryption-provider-config` | `/etc/kubernetes/enc/enc.yaml` | Encrypts Secrets at rest in etcd |
| `--encryption-provider-config-automatic-reload` | `true` | Picks up key rotation without a restart |
| `--tls-cipher-suites` | explicit modern list | Drops CBC/3DES/RC4 suites |
| `--tls-min-version` | `VersionTLS12` (or 13) | Blocks TLS 1.0/1.1 |
| `--profiling` | `false` | Removes `/debug/pprof` information disclosure |
| `--service-account-lookup` | `true` | Tokens are validated against the API, so deleting a SA revokes them |
| `--service-account-key-file` / `--service-account-signing-key-file` | kubeadm defaults | Required for bound token projection |
| `--request-timeout` | `60s` | Caps long-running request abuse |
| `--kubelet-certificate-authority` | `/etc/kubernetes/pki/ca.crt` | Makes the API server verify kubelet serving certs |
| `--secure-port` / `--insecure-port` | `6443` / removed | The insecure port no longer exists in modern releases; make sure nothing re-adds it |

```yaml
    - --anonymous-auth=false
    - --authorization-mode=Node,RBAC
    - --enable-admission-plugins=NodeRestriction,ServiceAccount,PodSecurity
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
    - --encryption-provider-config-automatic-reload=true
    - --profiling=false
    - --service-account-lookup=true
    - --request-timeout=60s
    - --tls-min-version=VersionTLS12
    - --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
```

Any file path referenced by a flag must also be mounted into the static pod:

```yaml
    volumeMounts:
      - {name: audit-policy, mountPath: /etc/kubernetes/audit, readOnly: true}
      - {name: audit-log, mountPath: /var/log/kubernetes/audit}
      - {name: enc, mountPath: /etc/kubernetes/enc, readOnly: true}
  volumes:
    - name: audit-policy
      hostPath: {path: /etc/kubernetes/audit, type: DirectoryOrCreate}
    - name: audit-log
      hostPath: {path: /var/log/kubernetes/audit, type: DirectoryOrCreate}
    - name: enc
      hostPath: {path: /etc/kubernetes/enc, type: DirectoryOrCreate}
```

Forgetting the volume mount is the number one reason a correct-looking audit or encryption
change silently fails and the API server crashloops.

---

## 2. Audit policy

Levels, from cheapest to most expensive:

| Level | Records |
| --- | --- |
| `None` | Nothing. Use it to drop noise |
| `Metadata` | Who, what, when, verb, resource — no bodies |
| `Request` | Metadata plus the request body |
| `RequestResponse` | Metadata plus request and response bodies |

Rules are evaluated top-down; the first match wins. `omitStages` suppresses whole event stages
(`RequestReceived`, `ResponseStarted`, `ResponseComplete`, `Panic`) — dropping
`RequestReceived` roughly halves log volume with no loss of useful signal.

```yaml
# /etc/kubernetes/audit/policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived
rules:
  # Never log Secret/ConfigMap bodies
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]

  # Full bodies for anything that can change security posture
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
      - group: "policy"
        resources: ["poddisruptionbudgets"]
      - group: "admissionregistration.k8s.io"
        resources: ["validatingwebhookconfigurations", "mutatingwebhookconfigurations"]

  # Pod exec/attach/portforward are the classic lateral-movement primitives
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  # Request bodies for pod and workload mutations
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["pods", "serviceaccounts"]
      - group: "apps"
        resources: ["deployments", "daemonsets", "statefulsets"]

  # Drop high-volume, low-value noise
  - level: None
    users: ["system:kube-scheduler", "system:kube-controller-manager"]
    verbs: ["get", "list", "watch"]
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get", "list", "watch"]
  - level: None
    nonResourceURLs: ["/healthz*", "/readyz*", "/livez*", "/version", "/metrics"]

  # Catch-all
  - level: Metadata
```

```bash
tail -f /var/log/kubernetes/audit/audit.log | jq 'select(.verb=="create")'
jq -r 'select(.objectRef.resource=="secrets") | [.user.username,.verb,.objectRef.namespace,.objectRef.name] | @tsv' \
  /var/log/kubernetes/audit/audit.log | sort | uniq -c | sort -rn | head
```

---

## 3. Encryption of Secrets at rest

By default, Secrets are stored base64-encoded — not encrypted — in etcd.

```yaml
# /etc/kubernetes/enc/enc.yaml   (chmod 600, root-owned)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      # The FIRST provider is used for writes; all are tried for reads.
      - aescbc:
          keys:
            - name: key1
              secret: <base64 of 32 random bytes>
      - identity: {}
```

```bash
head -c 32 /dev/urandom | base64        # generate a key
```

| Provider | Notes |
| --- | --- |
| `identity` | No encryption. Placing it first disables encryption for that resource |
| `secretbox` | XSalsa20 + Poly1305, fast, 32-byte key |
| `aescbc` | AES-CBC with PKCS#7, 32-byte key, widely used in exercises |
| `aesgcm` | Fast, but requires strict key rotation discipline; avoid unless automated |
| `kms` (v2) | External KMS provider, envelope encryption, the production answer |

KMS v2 shape:

```yaml
    providers:
      - kms:
          apiVersion: v2
          name: vault-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          timeout: 3s
      - identity: {}
```

### Applying and verifying

```bash
# 1. Write the config, mount it, add --encryption-provider-config, wait for the API server
k get --raw='/readyz?verbose'

# 2. Existing Secrets are still plaintext until rewritten. Re-encrypt everything:
k get secrets -A -o json | k replace -f -
# Same for configmaps if they are in the resource list:
k get configmaps -A -o json | k replace -f -

# 3. Prove it, by reading the raw value straight out of etcd
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | hexdump -C | head
# Expect a "k8s:enc:aescbc:v1:key1:" prefix, not readable plaintext.
```

To disable encryption again, put `identity: {}` first and re-run the `replace` pass.

---

## 4. RBAC audit and least privilege

### Reading the current state

```bash
k get clusterroles
k get clusterrolebindings -o wide
k get rolebindings -A -o wide
k describe clusterrole cluster-admin
k get clusterrolebinding -o json | jq -r '
  .items[] | select(.roleRef.name=="cluster-admin")
  | [.metadata.name, (.subjects // [] | map(.kind+"/"+(.namespace//"-")+"/"+.name) | join(","))]
  | @tsv'
```

### Finding dangerous permissions

```bash
# Wildcard verbs or wildcard resources anywhere
k get clusterroles -o json | jq -r '
  .items[] | select(.rules[]? | (.verbs[]? == "*") or (.resources[]? == "*"))
  | .metadata.name'

# Roles that can read secrets
k get clusterroles,roles -A -o json | jq -r '
  .items[] | select(.rules[]? | (.resources[]? == "secrets") and ((.verbs[]? == "get") or (.verbs[]? == "list")))
  | "\(.kind) \(.metadata.namespace // "-")/\(.metadata.name)"'

# What a specific identity can do
k auth can-i --list -n app --as=system:serviceaccount:app:deploy-bot
k auth can-i create pods -n kube-system --as=system:serviceaccount:app:deploy-bot
k auth can-i '*' '*' --as=system:serviceaccount:app:deploy-bot
k auth can-i create pods --as=jane --as-group=developers -n app
```

### Verbs and resources that are effectively cluster-admin

| Permission | Escape path |
| --- | --- |
| `create pods` in any namespace | Mount `hostPath: /`, or set `hostPID` + `privileged`, and own the node |
| `create pods` + a privileged ServiceAccount in that namespace | Run as that SA and inherit its rights |
| `escalate` on roles | Grant yourself permissions you do not have |
| `bind` on roles/clusterroles | Bind an existing powerful ClusterRole to yourself |
| `impersonate` users/groups/serviceaccounts | Become `system:masters` or any admin |
| `get`/`list` secrets cluster-wide | Read every token, TLS key, and credential in the cluster |
| `create serviceaccounts/token` (TokenRequest) | Mint a token for any SA, including privileged ones |
| `create pods/exec`, `pods/attach` | Execute inside an existing privileged pod |
| `get nodes/proxy` | Talk to the kubelet API directly, run commands on the node |
| `update`/`patch` on daemonsets/deployments in kube-system | Ship a privileged workload cluster-wide |
| `patch` on nodes | Remove taints, change labels, redirect scheduling |
| `create` on `certificatesigningrequests/approval` | Issue yourself a client cert for any identity |

Least-privilege construction:

```bash
k create sa deploy-bot -n app
k create role pod-reader -n app --verb=get,list,watch --resource=pods,pods/log
k create rolebinding pod-reader-bind -n app --role=pod-reader \
  --serviceaccount=app:deploy-bot
k auth can-i list pods -n app --as=system:serviceaccount:app:deploy-bot   # yes
k auth can-i list pods -n prod --as=system:serviceaccount:app:deploy-bot  # no
k auth can-i delete pods -n app --as=system:serviceaccount:app:deploy-bot # no
```

Prefer `Role`/`RoleBinding` over cluster scope; name resources explicitly with
`resourceNames` where the workload only needs one object:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: cfg-reader, namespace: app}
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-cfg"]
    verbs: ["get"]
```

`resourceNames` cannot restrict `list`, `watch`, `create`, or `deletecollection` — those are
collection-level verbs.

---

## 5. Service account hardening

Every pod gets a token mounted unless you say otherwise. Most workloads never call the API.

```yaml
# Disable at the ServiceAccount level (applies to all pods using it)
apiVersion: v1
kind: ServiceAccount
metadata: {name: app-sa, namespace: app}
automountServiceAccountToken: false
---
# Or per pod, which overrides the ServiceAccount setting
apiVersion: v1
kind: Pod
metadata: {name: web, namespace: app}
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false
  containers:
    - {name: web, image: nginx:1.27}
```

```bash
k get sa default -n app -o yaml
k patch sa default -n app -p '{"automountServiceAccountToken":false}'
k get po -A -o json | jq -r '
  .items[] | select(.spec.automountServiceAccountToken != false)
  | "\(.metadata.namespace)/\(.metadata.name)"' | head -20
```

Modern tokens are **bound service account tokens**: audience-scoped, time-limited, and tied to
the pod's lifetime. Deleting the pod or the ServiceAccount invalidates them (with
`--service-account-lookup=true`). Legacy non-expiring Secret-based tokens are no longer created
automatically — do not create them by hand.

Explicit projection when a workload needs a scoped token:

```yaml
spec:
  containers:
    - name: app
      image: app:1.0
      volumeMounts:
        - {name: api-token, mountPath: /var/run/secrets/tokens, readOnly: true}
  volumes:
    - name: api-token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: internal-api
```

```bash
# TokenRequest from the CLI
k create token app-sa -n app --duration=10m --audience=internal-api
k create token app-sa -n app --bound-object-kind=Pod --bound-object-name=web
```

---

## 6. kubelet hardening

The kubelet is a root-equivalent API on every node. `/var/lib/kubelet/config.yaml`:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false                 # no unauthenticated access to :10250
  webhook:
    enabled: true                  # delegate authn to the API server
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook                    # never AlwaysAllow; delegate authz to the API server
readOnlyPort: 0                    # disables the unauthenticated :10255 port
protectKernelDefaults: true        # refuse to start if sysctls are not as expected
makeIPTablesUtilChains: true
rotateCertificates: true           # rotate the client cert automatically
serverTLSBootstrap: true           # request a serving cert instead of self-signing
streamingConnectionIdleTimeout: 5m
eventRecordQPS: 5
tlsCipherSuites:
  - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
```

```bash
systemctl restart kubelet
systemctl status kubelet
journalctl -u kubelet --since '2 min ago' --no-pager | tail -30

# Prove the anonymous hole is closed (run from another node)
curl -sk https://worker-1:10250/pods | head          # expect 401 Unauthorized
curl -sk http://worker-1:10255/pods | head           # expect connection refused
```

The equivalent CLI flags (`--anonymous-auth=false`, `--authorization-mode=Webhook`,
`--read-only-port=0`, `--client-ca-file=...`) live in `/var/lib/kubelet/kubeadm-flags.env` on
kubeadm clusters. Config file values win over most defaults; flags win over the config file.
Fix whichever the cluster actually uses, then confirm with the curl checks above.

Also check node file permissions:

```bash
ls -l /etc/kubernetes/manifests/                  # want 600 root:root
ls -l /etc/kubernetes/pki/                        # keys 600, certs 644
ls -l /var/lib/kubelet/config.yaml                # 600 root:root
chmod 600 /etc/kubernetes/manifests/*.yaml
chown -R root:root /etc/kubernetes/pki
```

---

## 7. CIS benchmark and kube-bench

`kube-bench` implements the CIS Kubernetes Benchmark checks and prints the exact remediation
text for every failure. It is the fastest way to turn "harden the cluster" into a task list.

```bash
# As a one-shot job on the control-plane node
k apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-master.yaml
k logs -f job/kube-bench

# Or directly on the node
kube-bench run --targets=master
kube-bench run --targets=node,etcd,policies
kube-bench run --targets=master --check=1.2.1,1.2.6
kube-bench run --targets=master | grep -A3 '^\[FAIL\]'
kube-bench run --json | jq -r '.Controls[].tests[].results[] | select(.status=="FAIL") | "\(.test_number) \(.test_desc)"'
```

Sections map to: 1.x control plane components, 2.x etcd, 3.x control plane configuration,
4.x worker nodes, 5.x policies (RBAC, Pod Security, network). Treat `[WARN]` items as
decisions to document, not as noise.

---

## 8. NetworkPolicy default-deny

```yaml
# Deny all ingress AND egress in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: app
spec:
  podSelector: {}          # every pod in the namespace
  policyTypes: ["Ingress", "Egress"]
---
# Then re-open exactly what is needed. DNS first, or nothing will resolve.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: app
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels: {kubernetes.io/metadata.name: kube-system}
          podSelector:
            matchLabels: {k8s-app: kube-dns}
      ports:
        - {protocol: UDP, port: 53}
        - {protocol: TCP, port: 53}
---
# Allow ingress only from a specific namespace, on one port
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend-ns
  namespace: app
spec:
  podSelector:
    matchLabels: {app: api}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: {purpose: frontend}
      ports:
        - {protocol: TCP, port: 8080}
```

```bash
k label ns frontend purpose=frontend
k get netpol -A
k describe netpol default-deny-all -n app
# verify from inside a pod that should be blocked
k run probe -n other --image=busybox:1.36 --rm -it --restart=Never -- \
  wget -qO- --timeout=3 http://api.app:8080
```

Every namespace should have a default-deny baseline. Policies are enforced by the CNI —
confirm the installed plugin actually implements NetworkPolicy before trusting them.
Block metadata-service egress (`169.254.169.254`) explicitly on cloud nodes.

---

## 9. Restricting etcd access

etcd holds every Secret in the cluster in whatever form the encryption config leaves them.
Anyone who can reach port 2379 with a valid client cert owns the cluster.

```bash
grep -E 'client-cert-auth|peer-client-cert-auth|listen-client-urls|advertise-client-urls|cert-file|key-file|trusted-ca-file' \
  /etc/kubernetes/manifests/etcd.yaml
```

Requirements:

- `--client-cert-auth=true` and `--peer-client-cert-auth=true`.
- `--listen-client-urls` bound to `127.0.0.1` and the node's private IP only — never `0.0.0.0`.
- `--trusted-ca-file` and `--peer-trusted-ca-file` set to the dedicated etcd CA
  (`/etc/kubernetes/pki/etcd/ca.crt`), which must be separate from the cluster CA.
- `--auto-tls=false` and `--peer-auto-tls=false`.
- Data directory `/var/lib/etcd` owned by the etcd user, mode `700`.
- Host firewall: 2379/2380 reachable only from control-plane nodes.

```bash
ls -ld /var/lib/etcd                    # want drwx------
chmod 700 /var/lib/etcd
ss -lntp | grep -E '2379|2380'
```

Encrypted snapshots must be treated as Secrets too: an `etcdctl snapshot save` file is a full
copy of the cluster state. Store it with restricted permissions and delete it when done.

---

## 10. Upgrades and patching as a security control

Unpatched control planes are the most common real-world Kubernetes compromise path. The
security-relevant parts of the upgrade discipline:

```bash
kubeadm upgrade plan                                  # shows available patch/minor versions
kubeadm certs check-expiration                        # certs silently expire after 1 year
kubeadm certs renew all && systemctl restart kubelet
k get no -o wide                                      # kubelet + container runtime versions
crictl version
apt list --upgradable 2>/dev/null | grep -E 'containerd|runc|kube'
```

- Track upstream CVE announcements for the API server, kubelet, `runc`, and containerd.
- Upgrade one minor version at a time, control plane before workers.
- Patch node OS packages on the same cadence; a `runc` container-escape CVE is a cluster
  compromise regardless of how well the API server is configured.
- Rotate certificates before expiry, not after; an expired kubelet client cert makes the node
  `NotReady` and the fix requires node access.
- Re-run `kube-bench` after every upgrade — new versions add and change defaults.
