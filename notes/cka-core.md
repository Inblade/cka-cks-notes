# CKA Core: Workloads, Scheduling, Networking, Storage

Targets Kubernetes 1.30-1.33. Everything below assumes `alias k=kubectl` and
`export do='--dry-run=client -o yaml'`.

---

## 1. kubectl commands worth memorizing

This is the highest-leverage section in the whole repo. On a timed exam you never hand-write
a Deployment — you generate it and edit three fields.

### Generators

```bash
# Single pod, generated manifest
k run nginx --image=nginx:1.27 $do > pod.yaml
k run tmp --image=busybox:1.36 --restart=Never --rm -it -- sh      # throwaway debug pod
k run web --image=nginx --port=80 --labels=app=web,tier=front
k run probe --image=nginx --dry-run=client -o yaml \
  --command -- sleep 3600

# Deployment
k create deploy web --image=nginx:1.27 --replicas=3 $do > deploy.yaml
k create deploy web --image=nginx:1.27 --replicas=3 --port=80 $do > deploy.yaml

# Service (expose an existing object)
k expose deploy web --port=80 --target-port=8080 --name=web-svc          # ClusterIP
k expose deploy web --port=80 --type=NodePort --name=web-np
k create svc clusterip web-svc --tcp=80:8080 $do                        # no selector work
k create svc externalname ext --external-name=api.example.internal

# Jobs, CronJobs, ConfigMaps, Secrets, ServiceAccounts, RBAC
k create job pi --image=perl:5.34 -- perl -Mbignum=bpi -wle 'print bpi(200)'
k create cronjob report --image=busybox:1.36 --schedule='*/5 * * * *' -- /bin/sh -c 'date'
k create job manual-run --from=cronjob/report                # fire a CronJob immediately
k create cm app-cfg --from-literal=LOG_LEVEL=debug --from-file=./app.properties
k create secret generic db --from-literal=password='s3cr3t'
k create secret docker-registry regcred --docker-server=registry.example.com \
  --docker-username=ci --docker-password="$PASS"
k create sa deploy-bot
k create role reader --verb=get,list,watch --resource=pods,pods/log -n app
k create rolebinding reader-bind --role=reader --serviceaccount=app:deploy-bot -n app
k create clusterrole node-viewer --verb=get,list --resource=nodes
k create clusterrolebinding node-viewer-bind --clusterrole=node-viewer \
  --serviceaccount=app:deploy-bot
k create ingress web --rule='shop.example.com/*=web-svc:80,tls=web-tls'
k create quota tight --hard=cpu=2,memory=4Gi,pods=10 -n app
k create pdb web-pdb --selector=app=web --min-available=2
```

### Mutation and rollout

```bash
k set image deploy/web nginx=nginx:1.27.3                 # container name = "nginx"
k set image deploy/web '*=nginx:1.27.3'                   # every container
k set resources deploy/web -c=nginx --requests=cpu=100m,memory=128Mi \
                                    --limits=cpu=500m,memory=256Mi
k set serviceaccount deploy/web deploy-bot
k set env deploy/web LOG_LEVEL=debug
k set env deploy/web --from=configmap/app-cfg

k rollout status deploy/web
k rollout status deploy/web --timeout=60s
k rollout history deploy/web
k rollout history deploy/web --revision=3
k rollout undo deploy/web
k rollout undo deploy/web --to-revision=2
k rollout restart deploy/web                              # rolls pods without changing spec
k rollout pause deploy/web && k rollout resume deploy/web

k scale deploy/web --replicas=5
k scale deploy/web --replicas=5 --current-replicas=3      # optimistic guard
k scale sts/db --replicas=3
k autoscale deploy/web --min=2 --max=10 --cpu-percent=70  # creates an HPA
```

### Labels, annotations, taints, node lifecycle

```bash
k label node worker-1 disktype=ssd
k label pod web-0 tier=front --overwrite
k label pods --all env=lab
k label node worker-1 disktype-                           # trailing dash = remove
k annotate deploy web kubernetes.io/change-cause='bump to 1.27.3' --overwrite

k taint node worker-1 workload=batch:NoSchedule
k taint node worker-1 workload=batch:NoSchedule-          # remove
k taint node worker-1 dedicated=gpu:NoExecute

k cordon worker-1                                         # mark unschedulable only
k drain worker-1 --ignore-daemonsets --delete-emptydir-data
k drain worker-1 --ignore-daemonsets --delete-emptydir-data --force --timeout=120s
k uncordon worker-1
```

`--force` on `drain` is required only when unmanaged (bare) pods exist. `--delete-emptydir-data`
is required whenever any pod on the node uses an `emptyDir`, which in practice is almost always.

### Inspection and output shaping

```bash
k get po -o wide
k get po -A --sort-by=.metadata.creationTimestamp
k get po --sort-by=.status.containerStatuses[0].restartCount
k get no --sort-by=.metadata.name
k get events -A --sort-by=.lastTimestamp | tail -30
k get po -l 'app in (web,api),tier!=batch'
k get po --field-selector=status.phase=Running,spec.nodeName=worker-1
k get deploy web -o yaml > current.yaml
k get po -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP'

# jsonpath
k get no -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}{"\n"}'
k get po -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
k get po web-0 -o jsonpath='{.spec.containers[*].image}{"\n"}'
k get no -o jsonpath='{.items[*].spec.taints}' | tr ' ' '\n'
k get pv -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.capacity.storage}{"\n"}{end}'
k get secret db -o jsonpath='{.data.password}' | base64 -d

# schema discovery, no browser needed
k explain pod.spec.containers.securityContext
k explain deploy.spec.strategy --recursive
k explain networkpolicy.spec.ingress --recursive | head -40
k api-resources
k api-resources --namespaced=false
k api-resources --api-group=networking.k8s.io
k api-versions

# authorization checks
k auth can-i create pods -n app
k auth can-i --list -n app
k auth can-i get secrets -n prod --as=system:serviceaccount:app:deploy-bot
k auth can-i --list --as=system:serviceaccount:app:deploy-bot -n app
k auth whoami
```

### Live debugging

```bash
k describe po web-0
k logs web-0 -c sidecar --since=10m --tail=100
k logs web-0 --previous                      # logs of the last crashed container
k logs -l app=web --max-log-requests=10 -f
k exec -it web-0 -c app -- sh
k debug -it web-0 --image=busybox:1.36 --target=app     # ephemeral container, no restart
k debug node/worker-1 -it --image=busybox:1.36          # host-namespace debug pod
k cp web-0:/etc/nginx/nginx.conf ./nginx.conf -c app
k port-forward svc/web-svc 8080:80
k top no
k top po -A --sort-by=memory
k delete po web-0 --force --grace-period=0              # $now
```

---

## 2. Workload controllers

| Controller | Identity | Ordering | Storage | Use it when |
| --- | --- | --- | --- | --- |
| Deployment | Interchangeable, random suffix | None | Shared or none | Stateless HTTP/gRPC services, anything horizontally identical |
| StatefulSet | Stable ordinal name `x-0..x-N`, stable DNS | Ordered create/delete/update by default | Per-replica PVC via `volumeClaimTemplates` | Databases, quorum systems, brokers, anything needing stable identity |
| DaemonSet | One pod per matching node | N/A | Usually hostPath | Node agents: CNI, log shippers, node exporters, CSI node plugins |
| Job | Run to completion | `completions`/`parallelism` | Any | Batch work, migrations, one-shot tasks |
| CronJob | Creates Jobs on a schedule | N/A | Any | Recurring batch: backups, reports, cleanup |
| ReplicaSet | Owned by a Deployment | None | Any | Rarely created directly; know it exists for rollback |

### Deployment rolling update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 4
  revisionHistoryLimit: 5
  minReadySeconds: 10
  progressDeadlineSeconds: 300
  strategy:
    type: RollingUpdate        # or Recreate
    rollingUpdate:
      maxSurge: 1              # extra pods above replicas; int or "25%"
      maxUnavailable: 0        # pods allowed below replicas; int or "25%"
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: web}
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{containerPort: 80}]
```

| Setting | Effect |
| --- | --- |
| `maxSurge=1, maxUnavailable=0` | Zero-downtime, needs spare capacity, slowest |
| `maxSurge=0, maxUnavailable=1` | No extra capacity needed, briefly under-provisioned |
| `maxSurge=25%, maxUnavailable=25%` | Default; fast, tolerates a capacity dip |
| `type: Recreate` | Kills all old pods first. Required for RWO volumes and singleton apps |

Both cannot be `0` — the rollout would never progress. Percentages round `maxSurge` up and
`maxUnavailable` down.

### StatefulSet specifics

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless          # must exist, must be a headless Service
  replicas: 3
  podManagementPolicy: OrderedReady # or Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0                  # only ordinals >= partition are updated (canary)
  selector:
    matchLabels: {app: db}
  template:
    metadata:
      labels: {app: db}
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: db
          image: postgres:16
          volumeMounts:
            - {name: data, mountPath: /var/lib/postgresql/data}
  volumeClaimTemplates:
    - metadata: {name: data}
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast
        resources:
          requests: {storage: 10Gi}
```

PVCs created from `volumeClaimTemplates` are **not** deleted when the StatefulSet is deleted
(unless `persistentVolumeClaimRetentionPolicy` is set). Pod DNS is
`db-0.db-headless.<ns>.svc.cluster.local`.

### Job and CronJob

```yaml
apiVersion: batch/v1
kind: Job
metadata: {name: migrate}
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 4
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 300
  completionMode: Indexed        # pods get JOB_COMPLETION_INDEX
  template:
    spec:
      restartPolicy: OnFailure   # Never or OnFailure only
      containers:
        - {name: migrate, image: busybox:1.36, command: ["sh","-c","echo done"]}
---
apiVersion: batch/v1
kind: CronJob
metadata: {name: nightly}
spec:
  schedule: "0 2 * * *"
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid      # Allow | Forbid | Replace
  startingDeadlineSeconds: 120
  suspend: false
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - {name: job, image: busybox:1.36, command: ["sh","-c","date"]}
```

---

## 3. Scheduling

### nodeSelector — simplest, exact-match only

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### Affinity and anti-affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - {key: topology.kubernetes.io/zone, operator: In, values: ["eu-central-1a","eu-central-1b"]}
              - {key: node-role.kubernetes.io/control-plane, operator: DoesNotExist}
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - {key: disktype, operator: In, values: ["ssd"]}
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: {app: web}
          topologyKey: kubernetes.io/hostname      # one web pod per node
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 50
          podAffinityTerm:
            labelSelector:
              matchLabels: {app: cache}
            topologyKey: topology.kubernetes.io/zone
```

`topologyKey` is the node label that defines "same place". `kubernetes.io/hostname` = per node,
`topology.kubernetes.io/zone` = per zone. Node affinity operators: `In`, `NotIn`, `Exists`,
`DoesNotExist`, `Gt`, `Lt`. Pod (anti)affinity supports only `In`, `NotIn`, `Exists`, `DoesNotExist`.

### Taints and tolerations

| Effect | New pods | Running pods |
| --- | --- | --- |
| `NoSchedule` | Rejected unless tolerated | Untouched |
| `PreferNoSchedule` | Scheduler avoids the node, soft | Untouched |
| `NoExecute` | Rejected unless tolerated | Evicted unless tolerated |

```bash
k taint node worker-1 dedicated=gpu:NoSchedule
k taint node worker-2 maintenance=true:NoExecute
k describe node worker-1 | grep -i taints
```

```yaml
spec:
  tolerations:
    - {key: dedicated, operator: Equal, value: gpu, effect: NoSchedule}
    - {key: maintenance, operator: Exists, effect: NoExecute, tolerationSeconds: 300}
    - {operator: Exists}                     # tolerate everything (DaemonSet pattern)
```

Tolerations grant permission; they do not attract. To both tolerate and target, pair a
toleration with `nodeSelector`/`nodeAffinity`. Control-plane nodes carry
`node-role.kubernetes.io/control-plane:NoSchedule`.

### topologySpreadConstraints

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule       # or ScheduleAnyway
      labelSelector:
        matchLabels: {app: web}
      minDomains: 3
      nodeAffinityPolicy: Honor
      matchLabelKeys: ["pod-template-hash"]  # spread per revision, not across revisions
```

`maxSkew` = maximum allowed difference in matching-pod count between the most and least
populated topology domain.

### PriorityClass and preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: {name: high-priority}
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority       # or Never
description: "Latency-critical user-facing services"
```

```yaml
spec:
  priorityClassName: high-priority
```

Built-ins: `system-cluster-critical` (2000000000) and `system-node-critical` (2000001000).

### Resources, QoS, LimitRange

```yaml
resources:
  requests: {cpu: "250m", memory: "256Mi"}
  limits:   {cpu: "500m", memory: "512Mi"}
```

| QoS class | Condition | Eviction order |
| --- | --- | --- |
| `Guaranteed` | Every container has requests == limits for **both** cpu and memory | Evicted last |
| `Burstable` | At least one request set, but not matching limits everywhere | Evicted second |
| `BestEffort` | No requests and no limits anywhere | Evicted first |

```bash
k get po web-0 -o jsonpath='{.status.qosClass}{"\n"}'
```

Requests drive scheduling; limits drive enforcement. Exceeding a memory limit gets the
container OOMKilled; exceeding a CPU limit only throttles it. Namespace defaults come from a
`LimitRange`; namespace ceilings from a `ResourceQuota`.

### Manual and static scheduling

```yaml
spec:
  nodeName: worker-2      # bypasses the scheduler entirely
  schedulerName: my-scheduler
```

---

## 4. Services and networking

| Type | Reachable from | Notes |
| --- | --- | --- |
| `ClusterIP` | Inside the cluster | Default. Virtual IP, load-balanced by kube-proxy |
| `NodePort` | `<anyNodeIP>:30000-32767` | Superset of ClusterIP; port assigned or set via `nodePort` |
| `LoadBalancer` | External LB address | Superset of NodePort; needs a cloud/MetalLB provider |
| `ExternalName` | Inside the cluster | CNAME only, no proxying, no selector |
| Headless (`clusterIP: None`) | Inside the cluster | No VIP; DNS returns pod IPs directly |

```yaml
apiVersion: v1
kind: Service
metadata: {name: web-svc}
spec:
  type: ClusterIP
  selector: {app: web}
  ports:
    - name: http
      port: 80              # service port
      targetPort: 8080      # container port or named port
      protocol: TCP
  sessionAffinity: ClientIP
---
apiVersion: v1
kind: Service
metadata: {name: db-headless}
spec:
  clusterIP: None
  selector: {app: db}
  ports: [{port: 5432, targetPort: 5432}]
```

`internalTrafficPolicy: Local` and `externalTrafficPolicy: Local` keep traffic on the
receiving node — the latter preserves the client source IP for NodePort/LoadBalancer.

### Endpoints and EndpointSlice

```bash
k get endpoints web-svc
k get endpointslices -l kubernetes.io/service-name=web-svc
k describe endpointslice web-svc-abc12
```

Empty endpoints almost always means a selector/label mismatch or failing readiness probes.
`EndpointSlice` (`discovery.k8s.io/v1`) is the scalable replacement for `Endpoints`; both are
populated for compatibility. A Service with no selector lets you point at external IPs by
creating the `EndpointSlice` yourself.

### DNS

| Object | Record |
| --- | --- |
| Service | `web-svc.app.svc.cluster.local` |
| Named port | `_http._tcp.web-svc.app.svc.cluster.local` (SRV) |
| Headless member | `db-0.db-headless.app.svc.cluster.local` |
| Pod (rare) | `10-244-1-7.app.pod.cluster.local` |

### kube-proxy modes

| Mode | Data path | Complexity | Notes |
| --- | --- | --- | --- |
| `iptables` | Sequential chain match | O(n) rules | Default for years, fine to a few thousand services |
| `ipvs` | Kernel hash table | O(1) lookup | Better at scale, real LB algorithms (`rr`, `lc`, `sh`), needs ipvs modules |
| `nftables` | nft rulesets | O(1)-ish | Newer alternative, GA in recent releases |

```bash
k -n kube-system get cm kube-proxy -o yaml | grep -A3 'mode:'
k -n kube-system logs ds/kube-proxy | head
iptables-save | grep KUBE-SVC | head
ipvsadm -Ln
```

### Ingress and IngressClass

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["shop.example.com"]
      secretName: shop-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix          # Prefix | Exact | ImplementationSpecific
            backend:
              service:
                name: api-svc
                port: {number: 8080}
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port: {name: http}
```

An Ingress without a running controller does nothing. Check `k get ingressclass` first.

### NetworkPolicy basics

Policies are additive and namespaced. A pod selected by **any** policy for a direction becomes
default-deny for that direction; a pod selected by none is unrestricted.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-allow
  namespace: app
spec:
  podSelector:
    matchLabels: {app: web}
  policyTypes: ["Ingress","Egress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: {kubernetes.io/metadata.name: ingress-nginx}
        - podSelector:
            matchLabels: {app: api}
      ports:
        - {protocol: TCP, port: 8080}
  egress:
    - to:
        - podSelector:
            matchLabels: {app: db}
      ports: [{protocol: TCP, port: 5432}]
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels: {k8s-app: kube-dns}
      ports: [{protocol: UDP, port: 53}, {protocol: TCP, port: 53}]
```

Two entries in one `from` list are OR'ed. A `namespaceSelector` and `podSelector` inside the
**same** list item are AND'ed — the indentation is the whole meaning. Always remember to allow
DNS egress when you add an egress policy.

---

## 5. Storage

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: {name: pv-data}
spec:
  capacity: {storage: 10Gi}
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain      # Retain | Delete | (Recycle, deprecated)
  storageClassName: manual
  volumeMode: Filesystem                     # or Block
  hostPath: {path: /mnt/data}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data, namespace: app}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: manual
  resources:
    requests: {storage: 5Gi}
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: fast}
provisioner: kubernetes.io/no-provisioner    # or a CSI driver name
volumeBindingMode: WaitForFirstConsumer      # or Immediate
reclaimPolicy: Delete
allowVolumeExpansion: true
```

| Access mode | Short | Meaning |
| --- | --- | --- |
| `ReadWriteOnce` | RWO | Mounted read-write by a single **node** |
| `ReadOnlyMany` | ROX | Mounted read-only by many nodes |
| `ReadWriteMany` | RWX | Mounted read-write by many nodes (NFS, CephFS) |
| `ReadWriteOncePod` | RWOP | Mounted read-write by exactly one **pod** |

| Reclaim policy | On PVC delete |
| --- | --- |
| `Retain` | PV goes to `Released`, data kept, needs manual cleanup before reuse |
| `Delete` | PV and backing volume are deleted |

`volumeBindingMode: WaitForFirstConsumer` delays binding until a pod is scheduled, so the
volume lands in the same zone/node as the pod. With `Immediate` on a multi-zone cluster you
routinely get a pod stuck `Pending` because its volume is in the wrong zone. A PVC in
`Pending` with `WaitForFirstConsumer` and no pod is normal, not an error.

```bash
k get pv,pvc -A
k describe pvc data -n app
k get sc
k patch pvc data -n app -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'  # expansion
```

### ConfigMap and Secret consumption

```yaml
spec:
  serviceAccountName: deploy-bot
  automountServiceAccountToken: false
  containers:
    - name: app
      image: app:1.0
      envFrom:
        - configMapRef: {name: app-cfg}
        - secretRef: {name: db}
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: {name: db, key: password}
        - name: NODE_NAME
          valueFrom:
            fieldRef: {fieldPath: spec.nodeName}
      volumeMounts:
        - {name: cfg, mountPath: /etc/app, readOnly: true}
        - {name: tls, mountPath: /etc/tls, readOnly: true}
  volumes:
    - name: cfg
      configMap:
        name: app-cfg
        defaultMode: 0444
        items:
          - {key: app.properties, path: app.properties}
    - name: tls
      secret:
        secretName: shop-tls
        defaultMode: 0400
```

Volume-mounted ConfigMaps/Secrets update in place (with kubelet sync delay, ~60s); values
injected via `env` do **not** — the pod must restart. `subPath` mounts also do not update.
Force a restart with `k rollout restart deploy/web`.

---

## 6. Quick reference: things that cost people marks

- Forgetting `-n <namespace>`. Set it once: `k config set-context --current --namespace=app`.
- `k expose` needs the object to already have the right labels; verify with `k get ep`.
- `maxUnavailable: 0` with no spare node capacity = rollout stuck forever, not an error.
- `k drain` fails on bare pods without `--force` and on emptyDir without `--delete-emptydir-data`.
- A NetworkPolicy egress rule that forgets DNS breaks every name lookup in the pod.
- `k create job --from=cronjob/x` is the only sane way to test a CronJob without waiting.
- `k explain --recursive` replaces most documentation lookups and is much faster.
