# Practice Tasks

Seventeen original drills, written from scratch to exercise the CKA and CKS curricula. These
are not exam questions and bear no relation to any real exam content — they are scenarios
invented for this repository.

**How to use:** run these on a throwaway cluster (see the README). Start a timer, keep the
solutions collapsed, and write down what you got wrong. A task is only "done" when the
verification command passes.

Difficulty and time targets assume you already know Kubernetes and are drilling for speed.

| # | Domain | Topic | Target |
| --- | --- | --- | --- |
| 1 | CKA / Workloads | Deployment, expose, scale, rollout, rollback | 8 min |
| 2 | CKA / Scheduling | Node labels, nodeAffinity, taints and tolerations | 7 min |
| 3 | CKA / Cluster Architecture | Static pod on a worker node | 5 min |
| 4 | CKA / Workloads | Multi-container pod with a shared emptyDir | 5 min |
| 5 | CKA / Storage | StorageClass with WaitForFirstConsumer, PV, PVC | 8 min |
| 6 | CKA / Services & Networking | Service types and DNS resolution | 6 min |
| 7 | CKA / Services & Networking | NetworkPolicy default-deny plus one allowed namespace | 8 min |
| 8 | CKA / Cluster Architecture | RBAC for a ServiceAccount, verified with auth can-i | 6 min |
| 9 | CKA / Cluster Architecture | etcd snapshot save and status | 5 min |
| 10 | CKA / Cluster Architecture | Drain, maintain and uncordon a node | 5 min |
| 11 | CKA / Troubleshooting | Broken kubelet configuration | 8 min |
| 12 | CKA / Troubleshooting | Broken kube-apiserver static pod | 10 min |
| 13 | CKS / Minimize Microservice Vulnerabilities | Enforce PSS restricted and fix a rejected pod | 8 min |
| 14 | CKS / Monitoring & Runtime Security | seccomp RuntimeDefault and a Localhost profile | 7 min |
| 15 | CKS / Cluster Setup | Audit policy fragment | 8 min |
| 16 | CKS / Supply Chain Security | Trivy scan gate and cosign verification | 7 min |
| 17 | CKS / Cluster Hardening | Encrypt Secrets at rest and re-encrypt existing ones | 10 min |

---

## Task 1 — Deployment lifecycle

**Domain:** CKA / Workloads & Scheduling · **Target:** 8 min

In namespace `catalog` (create it), deploy `nginx:1.25` as a Deployment named `catalog-web`
with 3 replicas listening on container port 80. Expose it as a ClusterIP Service named
`catalog-web` on port 80. Configure the rolling update strategy so that no replica is ever
unavailable during an update and at most one extra pod may exist. Scale to 5 replicas, then
update the image to `nginx:1.27`, recording the change cause as `upgrade to 1.27`. Finally,
roll back to the previous revision.

<details>
<summary>Solution</summary>

```bash
k create ns catalog
k -n catalog create deploy catalog-web --image=nginx:1.25 --replicas=3 --port=80 \
  --dry-run=client -o yaml > catalog-web.yaml
```

Edit `catalog-web.yaml` to add the strategy under `spec`:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

```bash
k apply -f catalog-web.yaml
k -n catalog expose deploy catalog-web --name=catalog-web --port=80 --target-port=80
k -n catalog scale deploy catalog-web --replicas=5
k -n catalog set image deploy/catalog-web nginx=nginx:1.27
k -n catalog annotate deploy catalog-web kubernetes.io/change-cause='upgrade to 1.27' --overwrite
k -n catalog rollout status deploy/catalog-web
k -n catalog rollout history deploy/catalog-web
k -n catalog rollout undo deploy/catalog-web
```

</details>

**Verification**

```bash
k -n catalog get deploy catalog-web -o jsonpath='{.spec.replicas}{" "}{.spec.template.spec.containers[0].image}{" "}{.spec.strategy.rollingUpdate}{"\n"}' && k -n catalog get ep catalog-web
```

**Why this matters:** `maxUnavailable: 0` is the setting that makes a rollout genuinely
zero-downtime, and it is also the setting that hangs forever when the cluster has no spare
capacity. Knowing both halves is the difference between a safe deploy and an outage.

---

## Task 2 — Affinity, taints and tolerations

**Domain:** CKA / Workloads & Scheduling · **Target:** 7 min

Label one worker node `tier=analytics`. Taint that same node
`workload=analytics:NoSchedule`. In namespace `analytics`, create a Deployment `cruncher`
with 2 replicas of `busybox:1.36` running `sleep 3600`, which must schedule **only** on nodes
labelled `tier=analytics` and must tolerate the taint. Additionally ensure the two replicas
never share a node if more than one analytics node exists.

<details>
<summary>Solution</summary>

```bash
k label node worker-1 tier=analytics
k taint node worker-1 workload=analytics:NoSchedule
k create ns analytics
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cruncher
  namespace: analytics
spec:
  replicas: 2
  selector:
    matchLabels: {app: cruncher}
  template:
    metadata:
      labels: {app: cruncher}
    spec:
      tolerations:
        - key: workload
          operator: Equal
          value: analytics
          effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - {key: tier, operator: In, values: ["analytics"]}
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels: {app: cruncher}
                topologyKey: kubernetes.io/hostname
      containers:
        - name: cruncher
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
```

```bash
k apply -f cruncher.yaml
```

</details>

**Verification**

```bash
k -n analytics get po -o wide && k describe node worker-1 | grep -i taints
```

**Why this matters:** a toleration only grants permission to land on a tainted node; it does
not attract anything there. Dedicated node pools always need the pair — toleration plus
affinity — and forgetting one half is the classic cause of "why is my GPU job on a general
worker".

---

## Task 3 — Static pod

**Domain:** CKA / Cluster Architecture · **Target:** 5 min

On node `worker-1`, create a static pod named `heartbeat` running `busybox:1.36` with the
command `sh -c 'while true; do date; sleep 30; done'`. Confirm the control plane sees the
mirror pod. Then remove it cleanly.

<details>
<summary>Solution</summary>

```bash
ssh worker-1
sudo -i

grep staticPodPath /var/lib/kubelet/config.yaml       # confirm /etc/kubernetes/manifests

cat > /etc/kubernetes/manifests/heartbeat.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: heartbeat
  namespace: default
spec:
  containers:
    - name: heartbeat
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date; sleep 30; done"]
EOF

crictl ps | grep heartbeat
exit
exit

k get po -A | grep heartbeat        # appears as heartbeat-worker-1

# Removal: kubectl delete does not work, the kubelet recreates it
ssh worker-1 'sudo rm /etc/kubernetes/manifests/heartbeat.yaml'
k get po -A | grep heartbeat        # gone
```

</details>

**Verification**

```bash
k get po heartbeat-worker-1 -o jsonpath='{.metadata.ownerReferences[0].kind}{"\n"}'   # Node
```

**Why this matters:** static pods are how the control plane itself runs on a kubeadm cluster.
Understanding that the kubelet reads a directory — with no API server in the loop — is the
prerequisite for every control-plane recovery procedure.

---

## Task 4 — Multi-container pod with a shared volume

**Domain:** CKA / Workloads & Scheduling · **Target:** 5 min

In namespace `logging`, create a pod `tailer` with two containers sharing an `emptyDir` mounted
at `/data`. Container `writer` (`busybox:1.36`) appends a timestamped line to `/data/app.log`
every 5 seconds. Container `reader` (`busybox:1.36`) tails that file. Neither container may run
as root.

<details>
<summary>Solution</summary>

```bash
k create ns logging
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tailer
  namespace: logging
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
  volumes:
    - name: data
      emptyDir: {}
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"$(date -Iseconds) tick\" >> /data/app.log; sleep 5; done"]
      volumeMounts:
        - {name: data, mountPath: /data}
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "until [ -f /data/app.log ]; do sleep 1; done; tail -f /data/app.log"]
      volumeMounts:
        - {name: data, mountPath: /data, readOnly: true}
```

```bash
k apply -f tailer.yaml
```

</details>

**Verification**

```bash
k -n logging logs tailer -c reader --tail=3
```

**Why this matters:** the sidecar pattern with a shared `emptyDir` is the foundation of log
shipping, cache warming and config templating. The `readOnly` mount on the consumer side and
a non-root `fsGroup` are what make it safe.

---

## Task 5 — Storage with WaitForFirstConsumer

**Domain:** CKA / Storage · **Target:** 8 min

Create a StorageClass `local-delayed` with no dynamic provisioner and
`volumeBindingMode: WaitForFirstConsumer`. Create a 2Gi hostPath PersistentVolume `pv-reports`
at `/mnt/reports` on `worker-1` using that class, with reclaim policy `Retain` and access mode
`ReadWriteOnce`. In namespace `reports`, create a PVC `reports-data` requesting 1Gi from that
class, and a pod `report-writer` (`busybox:1.36`, `sleep 3600`) that mounts it at `/reports`.
Confirm the PVC only binds once the pod is scheduled.

<details>
<summary>Solution</summary>

```bash
ssh worker-1 'sudo mkdir -p /mnt/reports'
k create ns reports
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-delayed
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-reports
spec:
  capacity: {storage: 2Gi}
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-delayed
  hostPath:
    path: /mnt/reports
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - {key: kubernetes.io/hostname, operator: In, values: ["worker-1"]}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: reports-data
  namespace: reports
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-delayed
  resources:
    requests: {storage: 1Gi}
---
apiVersion: v1
kind: Pod
metadata:
  name: report-writer
  namespace: reports
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - {name: data, mountPath: /reports}
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: reports-data
```

```bash
k apply -f storage.yaml
# Before the pod exists the PVC stays Pending with "waiting for first consumer" - that is correct.
k -n reports get pvc reports-data
```

</details>

**Verification**

```bash
k -n reports get pvc reports-data -o jsonpath='{.status.phase}{" -> "}{.spec.volumeName}{"\n"}' && k -n reports exec report-writer -- touch /reports/ok
```

**Why this matters:** `WaitForFirstConsumer` exists because binding a volume before scheduling
the pod pins the pod to wherever the volume happened to land — usually the wrong zone. A PVC
sitting `Pending` with no pod is expected behaviour, not a bug, and misreading that wastes a
lot of debugging time.

---

## Task 6 — Services and cluster DNS

**Domain:** CKA / Services & Networking · **Target:** 6 min

In namespace `shop`, run a Deployment `api` (2 replicas, `nginx:1.27`, port 80). Expose it
three ways: a ClusterIP Service `api` on port 8080 targeting 80; a NodePort Service `api-np`
on port 8080 with node port `31080`; and a headless Service `api-headless`. Then resolve all
three from a temporary pod.

<details>
<summary>Solution</summary>

```bash
k create ns shop
k -n shop create deploy api --image=nginx:1.27 --replicas=2 --port=80
k -n shop expose deploy api --name=api --port=8080 --target-port=80
k -n shop expose deploy api --name=api-np --port=8080 --target-port=80 --type=NodePort
k -n shop patch svc api-np --type=merge -p '{"spec":{"ports":[{"port":8080,"targetPort":80,"nodePort":31080}]}}'
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-headless
  namespace: shop
spec:
  clusterIP: None
  selector: {app: api}
  ports:
    - {port: 8080, targetPort: 80}
```

```bash
k apply -f api-headless.yaml

k run probe -n shop --image=busybox:1.36 --rm -it --restart=Never -- \
  sh -c 'nslookup api.shop.svc.cluster.local; nslookup api-headless.shop.svc.cluster.local; wget -qO- --timeout=3 http://api.shop:8080 | head -3'
```

</details>

**Verification**

```bash
k -n shop get svc,ep && curl -s --max-time 3 http://$(k get no -o jsonpath='{.items[1].status.addresses[0].address}'):31080 | head -3
```

**Why this matters:** a headless Service returns pod IPs, a ClusterIP returns one virtual IP.
That difference determines whether a client load-balances through kube-proxy or does its own
discovery — which is exactly why StatefulSets require a headless Service.

---

## Task 7 — NetworkPolicy default-deny plus a namespace allow

**Domain:** CKA / Services & Networking · **Target:** 8 min

Namespace `payments` runs a pod labelled `app=ledger` on port 8080. Apply a policy so that:
all ingress and egress in `payments` is denied by default; pods in `payments` may still resolve
DNS; and `app=ledger` accepts TCP 8080 only from pods in a namespace labelled
`network=trusted`. Create namespace `frontend` labelled `network=trusted` and namespace
`external` without that label, and prove the difference.

<details>
<summary>Solution</summary>

```bash
k create ns payments
k create ns frontend && k label ns frontend network=trusted
k create ns external
k -n payments run ledger --image=nginxinc/nginx-unprivileged:1.27 --labels=app=ledger --port=8080
k -n payments expose po ledger --name=ledger --port=8080 --target-port=8080
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: payments
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
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-trusted-to-ledger
  namespace: payments
spec:
  podSelector:
    matchLabels: {app: ledger}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: {network: trusted}
      ports:
        - {protocol: TCP, port: 8080}
```

```bash
k apply -f netpol.yaml
```

</details>

**Verification**

```bash
k run p1 -n frontend --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://ledger.payments:8080 >/dev/null && echo ALLOWED; k run p2 -n external --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://ledger.payments:8080 || echo DENIED
```

**Why this matters:** the default-deny plus explicit-allow pattern is the only NetworkPolicy
design that scales. The DNS egress rule is the part everyone forgets, and its absence breaks
every name lookup in the namespace in a way that looks like an application bug.

---

## Task 8 — RBAC for a ServiceAccount

**Domain:** CKA / Cluster Architecture · **Target:** 6 min

In namespace `ci`, create a ServiceAccount `pipeline`. Grant it exactly: read pods and pod
logs in `ci`, and update Deployments in `ci` — nothing else, and nothing in any other
namespace. Prove the grants and the absence of extra grants.

<details>
<summary>Solution</summary>

```bash
k create ns ci
k -n ci create sa pipeline
k -n ci create role pipeline-role \
  --verb=get,list,watch --resource=pods,pods/log
k -n ci create role deploy-updater \
  --verb=get,list,update,patch --resource=deployments.apps
k -n ci create rolebinding pipeline-read \
  --role=pipeline-role --serviceaccount=ci:pipeline
k -n ci create rolebinding pipeline-deploy \
  --role=deploy-updater --serviceaccount=ci:pipeline
```

Equivalent single-Role form:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: pipeline-role, namespace: ci}
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: pipeline-read, namespace: ci}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pipeline-role
subjects:
  - kind: ServiceAccount
    name: pipeline
    namespace: ci
```

</details>

**Verification**

```bash
k auth can-i --list -n ci --as=system:serviceaccount:ci:pipeline; k auth can-i delete pods -n ci --as=system:serviceaccount:ci:pipeline; k auth can-i list pods -n default --as=system:serviceaccount:ci:pipeline
```

Expect `yes` for `get/list pods` and `update deployments` in `ci`, and `no` for the other two.

**Why this matters:** `kubectl auth can-i --as=` is how you prove a permission model instead of
hoping. Testing the negatives — what the identity must *not* be able to do — is the half of
RBAC review that catches real over-permissioning.

---

## Task 9 — etcd snapshot

**Domain:** CKA / Cluster Architecture · **Target:** 5 min

Take a snapshot of etcd on the control-plane node, save it to `/opt/etcd-snapshot.db`, and
print its status as a table. Use the certificates from the running etcd static pod.

<details>
<summary>Solution</summary>

```bash
ssh control-plane
sudo -i

# Read the real values rather than assuming them
grep -E 'listen-client-urls|cert-file|key-file|trusted-ca-file|data-dir' \
  /etc/kubernetes/manifests/etcd.yaml

ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-snapshot.db --write-out=table
```

If `etcdctl` is not installed on the host, run it inside the etcd container:

```bash
crictl exec -it $(crictl ps --name etcd -q | head -1) sh -c \
  'ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key endpoint health'
```

</details>

**Verification**

```bash
ssh control-plane 'sudo ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-snapshot.db --write-out=table'
```

**Why this matters:** the snapshot is the entire cluster state in one file. The flags never
change, so this should be pure recall — and the snapshot file is itself a Secret, since it
contains every Secret in the cluster.

---

## Task 10 — Node maintenance

**Domain:** CKA / Cluster Architecture · **Target:** 5 min

Take `worker-2` out of service for maintenance without disrupting DaemonSet pods, evicting
workloads that use `emptyDir`. Confirm nothing is scheduled there, then return it to service.

<details>
<summary>Solution</summary>

```bash
k get po -A -o wide --field-selector spec.nodeName=worker-2
k drain worker-2 --ignore-daemonsets --delete-emptydir-data --timeout=120s
k get no worker-2                       # Ready,SchedulingDisabled
k get po -A -o wide --field-selector spec.nodeName=worker-2   # only DaemonSet pods remain

# ... maintenance ...

k uncordon worker-2
k get no
```

If the drain refuses because of bare (uncontrolled) pods:

```bash
k drain worker-2 --ignore-daemonsets --delete-emptydir-data --force
```

</details>

**Verification**

```bash
k get no worker-2 -o jsonpath='{.spec.unschedulable}{"\n"}'   # empty after uncordon
```

**Why this matters:** `drain` without `--ignore-daemonsets` and `--delete-emptydir-data` fails
on essentially every real cluster, and `--force` silently destroys unmanaged pods. Knowing
which flag does what before you need it is the point.

---

## Task 11 — Broken kubelet configuration

**Domain:** CKA / Troubleshooting · **Target:** 8 min

`worker-1` shows `NotReady`. Diagnose and fix it, then confirm workloads schedule there again.
To create the fault first (on the node): change `staticPodPath` in
`/var/lib/kubelet/config.yaml` to `/etc/kubernetes/manifest` (note the missing `s`), or set
`authentication.anonymous.enabled` to an invalid value such as `yes`, then
`systemctl restart kubelet`.

<details>
<summary>Solution</summary>

```bash
k get no -o wide
k describe node worker-1 | sed -n '/Conditions/,/Addresses/p'

ssh worker-1
sudo -i

systemctl status kubelet                       # activating (auto-restart) or failed
journalctl -u kubelet --since '5 min ago' --no-pager | tail -40
# Look for: "failed to parse kubelet flag" / "unmarshal errors" / the exact line number

cp /var/lib/kubelet/config.yaml /root/kubelet-config.yaml.bak
vi /var/lib/kubelet/config.yaml
#   staticPodPath: /etc/kubernetes/manifests      <- restore the correct path
#   authentication.anonymous.enabled: false       <- must be a real boolean

systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
journalctl -u kubelet -f | head -20

crictl ps                                       # containers coming back
exit
exit

k get no worker-1
k get po -A -o wide --field-selector spec.nodeName=worker-1
```

</details>

**Verification**

```bash
k get no worker-1 -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}{"\n"}'   # True
```

**Why this matters:** the kubelet is the answer to most `NotReady` nodes, and its journal
names the exact broken line. The reflex to run `journalctl -u kubelet` before anything else
saves minutes every time.

---

## Task 12 — Broken kube-apiserver static pod

**Domain:** CKA / Troubleshooting · **Target:** 10 min

`kubectl` returns `The connection to the server ... was refused`. Restore the control plane.
To create the fault first, on the control-plane node add an invalid flag such as
`- --this-flag-does-not-exist=true` to `/etc/kubernetes/manifests/kube-apiserver.yaml`, or
point `--etcd-servers` at the wrong port.

<details>
<summary>Solution</summary>

```bash
ssh control-plane
sudo -i

# kubectl is unavailable, so work through the container runtime
crictl ps -a | grep -E 'kube-apiserver|etcd'
crictl logs --tail=50 $(crictl ps -a --name kube-apiserver -q | head -1)
# or straight off disk
tail -60 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log

# Stop the restart loop while you edit
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

vi /tmp/kube-apiserver.yaml
#   remove the bogus flag / fix --etcd-servers=https://127.0.0.1:2379
#   check indentation: every flag is a list item under command:

mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
watch -n2 'crictl ps | grep apiserver'

# Once the container is Running:
k get --raw='/readyz?verbose'
k get no
k -n kube-system get po
```

</details>

**Verification**

```bash
k get --raw='/readyz?verbose' | tail -3 && k get po -n kube-system | grep apiserver
```

**Why this matters:** when the API server is down, `kubectl` is useless and every diagnostic
has to come from `crictl`, `/var/log/pods` and the kubelet journal. Moving the manifest out of
the watched directory before editing stops the kubelet from thrashing while you work.

---

## Task 13 — Enforce Pod Security Standards restricted

**Domain:** CKS / Minimize Microservice Vulnerabilities · **Target:** 8 min

Create namespace `tenant-a` and enforce the `restricted` Pod Security Standard on it, pinned to
version `v1.33`, with `audit` and `warn` also set to `restricted`. Then deploy a pod
`web` from `nginxinc/nginx-unprivileged:1.27` that is accepted by the policy. First confirm
that a naive `nginx:1.27` pod is rejected, and read the rejection message.

<details>
<summary>Solution</summary>

```bash
k create ns tenant-a
k label ns tenant-a \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.33 \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=v1.33 \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=v1.33 \
  --overwrite

# This is rejected; read the message, it lists every violated field
k -n tenant-a run bad --image=nginx:1.27
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: tenant-a
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: web
      image: nginxinc/nginx-unprivileged:1.27
      ports:
        - {containerPort: 8080}
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

```bash
k apply -f web.yaml
```

</details>

**Verification**

```bash
k get ns tenant-a --show-labels && k -n tenant-a get po web -o jsonpath='{.status.phase}{" "}{.spec.securityContext.seccompProfile.type}{"\n"}'
```

**Why this matters:** `restricted` rejects the default image of almost every popular workload,
because they run as root and never drop capabilities. Rolling out PSS is mostly the work of
finding non-root variants and adding four fields — do it in `warn` mode first and read the
warnings.

---

## Task 14 — seccomp

**Domain:** CKS / Monitoring, Logging and Runtime Security · **Target:** 7 min

In namespace `tenant-a`, create pod `sec-default` running `nginxinc/nginx-unprivileged:1.27`
with the `RuntimeDefault` seccomp profile, and prove a filter is active inside the container.
Then place an audit-only profile at `/var/lib/kubelet/seccomp/profiles/audit.json` on
`worker-1` and create pod `sec-local` that uses it.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata: {name: sec-default, namespace: tenant-a}
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: web
      image: nginxinc/nginx-unprivileged:1.27
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: {drop: ["ALL"]}
```

```bash
k apply -f sec-default.yaml
k -n tenant-a exec sec-default -- grep -E 'Seccomp|Seccomp_filters' /proc/1/status
# Seccomp: 2  and Seccomp_filters: 1  -> a filter is loaded
```

```bash
ssh worker-1
sudo -i
mkdir -p /var/lib/kubelet/seccomp/profiles
cat > /var/lib/kubelet/seccomp/profiles/audit.json <<'EOF'
{
  "defaultAction": "SCMP_ACT_LOG"
}
EOF
exit
exit
```

```yaml
apiVersion: v1
kind: Pod
metadata: {name: sec-local, namespace: tenant-a}
spec:
  nodeName: worker-1
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
    - name: web
      image: nginxinc/nginx-unprivileged:1.27
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: {drop: ["ALL"]}
```

```bash
k apply -f sec-local.yaml
```

</details>

**Verification**

```bash
k -n tenant-a exec sec-default -- grep Seccomp: /proc/1/status && k -n tenant-a get po sec-local -o jsonpath='{.spec.securityContext.seccompProfile}{"\n"}'
```

**Why this matters:** `localhostProfile` is a path relative to `/var/lib/kubelet/seccomp/`, and
the file must exist on every node the pod might land on. A missing file produces a pod that
fails to start with a message most people misread as an image problem.

---

## Task 15 — Audit policy

**Domain:** CKS / Cluster Setup · **Target:** 8 min

Configure API server auditing: write a policy at `/etc/kubernetes/audit/policy.yaml` that logs
Secret access at `Metadata` level only, logs RBAC object mutations at `RequestResponse`, drops
health-endpoint noise entirely, and records everything else at `Metadata`. Log to
`/var/log/kubernetes/audit/audit.log` with 7 days of retention and 5 rotated files.

<details>
<summary>Solution</summary>

```bash
ssh control-plane
sudo -i
mkdir -p /etc/kubernetes/audit /var/log/kubernetes/audit

cat > /etc/kubernetes/audit/policy.yaml <<'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  - level: None
    nonResourceURLs: ["/healthz*", "/readyz*", "/livez*", "/version", "/metrics"]
  - level: Metadata
EOF

cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add the flags:

```yaml
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=7
    - --audit-log-maxbackup=5
    - --audit-log-maxsize=100
```

And the mounts — this is the step that is easy to miss:

```yaml
    volumeMounts:
      - {name: audit-policy, mountPath: /etc/kubernetes/audit, readOnly: true}
      - {name: audit-log, mountPath: /var/log/kubernetes/audit}
  volumes:
    - name: audit-policy
      hostPath: {path: /etc/kubernetes/audit, type: DirectoryOrCreate}
    - name: audit-log
      hostPath: {path: /var/log/kubernetes/audit, type: DirectoryOrCreate}
```

```bash
watch -n2 'crictl ps | grep apiserver'
k get --raw='/readyz?verbose' | tail -3
k -n default get secrets
tail -5 /var/log/kubernetes/audit/audit.log
```

</details>

**Verification**

```bash
ssh control-plane "sudo grep -c '\"resource\":\"secrets\"' /var/log/kubernetes/audit/audit.log"
```

**Why this matters:** a policy file with no corresponding `hostPath` volume means the API
server cannot read it and crashloops, or starts with no auditing at all. Flags and mounts are
one change, never two.

---

## Task 16 — Supply chain gate

**Domain:** CKS / Supply Chain Security · **Target:** 7 min

Scan the image `nginx:1.25` and fail the command if any fixable HIGH or CRITICAL vulnerability
is present. Produce a CycloneDX SBOM for it. Then sign a locally tagged image with cosign and
verify the signature.

<details>
<summary>Solution</summary>

```bash
# Gate: non-zero exit when fixable HIGH/CRITICAL findings exist
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 nginx:1.25
echo "exit code: $?"

# Human-readable detail
trivy image --severity HIGH,CRITICAL nginx:1.25 | head -40

# SBOM
trivy image --format cyclonedx --output nginx-sbom.cdx.json nginx:1.25
trivy sbom nginx-sbom.cdx.json --severity CRITICAL

# Configuration scanning of your own manifests
trivy config ./manifests
trivy fs --scanners vuln,secret --severity HIGH,CRITICAL ./src

# Signing (key-based, works offline)
cosign generate-key-pair
docker tag nginx:1.25 localhost:5000/nginx:1.25
docker push localhost:5000/nginx:1.25
COSIGN_PASSWORD='' cosign sign --key cosign.key --yes localhost:5000/nginx:1.25
cosign verify --key cosign.pub localhost:5000/nginx:1.25

# Pin by digest for deployment
crane digest localhost:5000/nginx:1.25
```

Deployment then references the digest, never the tag:

```yaml
    containers:
      - name: web
        image: localhost:5000/nginx@sha256:<digest>
        imagePullPolicy: IfNotPresent
```

</details>

**Verification**

```bash
trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 nginx:1.25; echo "gate exit=$?"; cosign verify --key cosign.pub localhost:5000/nginx:1.25 >/dev/null && echo SIGNATURE_OK
```

**Why this matters:** a scan without `--exit-code 1` is a report nobody reads; with it, it is a
gate. And signature verification is only meaningful against a digest — a tag can be repointed
at an unsigned image after the check passes.

---

## Task 17 — Encrypt Secrets at rest

**Domain:** CKS / Cluster Hardening · **Target:** 10 min

Enable `aescbc` encryption for Secrets in etcd. Create a Secret **before** enabling it, and
confirm afterwards that the pre-existing Secret is still stored in plaintext until you
re-encrypt it. Then re-encrypt all Secrets and confirm again.

<details>
<summary>Solution</summary>

```bash
k create ns vault-demo
k -n vault-demo create secret generic pre-existing --from-literal=token=plaintext-value

ssh control-plane
sudo -i
mkdir -p /etc/kubernetes/enc
head -c 32 /dev/urandom | base64            # copy this value

cat > /etc/kubernetes/enc/enc.yaml <<'EOF'
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: REPLACE_WITH_BASE64_32_BYTES
      - identity: {}
EOF
chmod 600 /etc/kubernetes/enc/enc.yaml

cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add the flag and the mount:

```yaml
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
    - --encryption-provider-config-automatic-reload=true
...
    volumeMounts:
      - {name: enc, mountPath: /etc/kubernetes/enc, readOnly: true}
  volumes:
    - name: enc
      hostPath: {path: /etc/kubernetes/enc, type: DirectoryOrCreate}
```

```bash
watch -n2 'crictl ps | grep apiserver'
k get --raw='/readyz?verbose' | tail -3

# New Secrets are encrypted; the old one is not yet
k -n vault-demo create secret generic after-enc --from-literal=token=new-value

ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/vault-demo/pre-existing | hexdump -C | head -5   # plaintext visible

ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/vault-demo/after-enc | hexdump -C | head -5      # k8s:enc:aescbc:v1:key1

# Rewrite every existing Secret through the API so it is stored encrypted
kubectl get secrets -A -o json | kubectl replace -f -

# Confirm the old one is now encrypted too
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/vault-demo/pre-existing | hexdump -C | head -5
```

</details>

**Verification**

```bash
ssh control-plane "sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key get /registry/secrets/vault-demo/pre-existing | head -c 200 | strings | grep -q 'k8s:enc:aescbc' && echo ENCRYPTED || echo PLAINTEXT"
```

**Why this matters:** enabling encryption only affects future writes. Every Secret created
before the change stays readable in an etcd snapshot until you rewrite it — which is why
`kubectl get secrets -A -o json | kubectl replace -f -` is the second half of the task, not an
optional extra.

---

## Suggested drill schedule

| Session | Tasks | Focus |
| --- | --- | --- |
| 1 | 1, 2, 4, 6 | Imperative speed, generators, no editor |
| 2 | 5, 7, 8 | Objects with no generator: write them from memory |
| 3 | 3, 9, 10, 12 | Node access, static pods, etcd, `crictl` |
| 4 | 11, 12 | Break it yourself first, then fix under a 10-minute timer |
| 5 | 13, 14, 17 | Workload and control-plane hardening |
| 6 | 15, 16 | Audit and supply chain |
| 7 | All, timed | Full run at 120 minutes total, flag-and-skip discipline |

Track two numbers per session: how many tasks you finished inside the target, and how many
times you had to open the documentation. Both should trend down.
