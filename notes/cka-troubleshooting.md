# CKA Troubleshooting

Troubleshooting is the single heaviest CKA domain (30%). Most of it is a small number of
repeatable procedures. Learn the procedures, not the anecdotes.

The universal first three commands, in this order:

```bash
k get po -A -o wide | grep -Ev '\s+Running\s|\s+Completed\s'
k describe po <pod> -n <ns> | sed -n '/Events/,$p'
k get events -n <ns> --sort-by=.lastTimestamp | tail -20
```

---

## 1. Pod state decision tree

```
Pod not healthy
├── STATUS = Pending
│   ├── Events: "0/N nodes are available"
│   │   ├── "Insufficient cpu/memory"        -> requests too big / cluster full
│   │   ├── "node(s) had untolerated taint"  -> add toleration or untaint
│   │   ├── "didn't match node selector"     -> nodeSelector/nodeAffinity vs node labels
│   │   ├── "didn't match pod anti-affinity" -> topologyKey / replica count
│   │   └── "node(s) were unschedulable"     -> cordoned node -> kubectl uncordon
│   ├── Events mention volume / "waiting for first consumer"
│   │   -> PVC unbound: no matching PV, wrong storageClassName, or WaitForFirstConsumer
│   └── No events at all -> scheduler is down (check static pod kube-scheduler)
├── STATUS = ImagePullBackOff / ErrImagePull
│   -> wrong image name/tag, private registry without imagePullSecrets, no egress to registry
├── STATUS = CrashLoopBackOff
│   -> the container starts and exits; read the PREVIOUS logs, not the current ones
├── STATUS = CreateContainerConfigError
│   -> referenced ConfigMap/Secret or key does not exist
├── STATUS = CreateContainerError / RunContainerError
│   -> bad command/entrypoint, missing binary, bad mount path, permission denied
├── STATUS = Init:0/1, Init:CrashLoopBackOff
│   -> an initContainer is failing; logs need -c <initContainerName>
├── STATUS = OOMKilled (in container status, pod may show CrashLoopBackOff)
│   -> memory limit too low or a leak
├── STATUS = Evicted
│   -> node pressure (disk/inode/memory); pod is BestEffort/Burstable and got picked
├── STATUS = Terminating (stuck)
│   -> finalizers, unreachable node, or a process ignoring SIGTERM
└── STATUS = Running but not Ready (0/1)
    -> readiness probe failing; describe shows "Readiness probe failed: ..."
```

### Confirming each state

| Symptom | Command that confirms it | What to look for |
| --- | --- | --- |
| Pending – scheduling | `k describe po <p> -n <ns> \| grep -A20 Events` | `FailedScheduling` message names the exact predicate |
| Pending – volume | `k get pvc -n <ns>; k describe pvc <c> -n <ns>` | `Pending`, `no persistent volumes available`, `waiting for first consumer` |
| ImagePullBackOff | `k describe po <p> -n <ns> \| grep -i -A3 'failed to pull'` | 401/403 = auth, `not found` = tag typo, timeout = egress |
| CrashLoopBackOff | `k logs <p> -n <ns> --previous` | The actual application error and its exit code |
| CreateContainerConfigError | `k describe po <p> -n <ns> \| grep -i 'configmap\|secret'` | `couldn't find key X in ConfigMap/Secret Y` |
| OOMKilled | `k get po <p> -n <ns> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'` | Literally `OOMKilled`, exit code 137 |
| Evicted | `k get po -A --field-selector=status.phase=Failed`; `k describe node <n>` | Node conditions `DiskPressure`/`MemoryPressure` True |
| Terminating stuck | `k get po <p> -n <ns> -o jsonpath='{.metadata.finalizers}'` | Non-empty finalizers, or node `NotReady` |
| Not Ready | `k describe po <p> -n <ns> \| grep -i 'probe'` | `Readiness probe failed: HTTP probe failed with statuscode: 500` |

Useful follow-ups:

```bash
k get po <p> -n <ns> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" restarts="}{.restartCount}{" reason="}{.state.waiting.reason}{"\n"}{end}'
k delete po <p> -n <ns> --force --grace-period=0                     # last resort
k patch po <p> -n <ns> -p '{"metadata":{"finalizers":null}}'          # unstick Terminating
k get po -A --field-selector=status.phase=Failed -o name | xargs -r k delete   # clean evicted
```

---

## 2. Node NotReady procedure

```bash
# 1. What does the control plane think?
k get no -o wide
k describe node worker-1 | sed -n '/Conditions/,/Addresses/p'
k describe node worker-1 | grep -iE 'pressure|kubelet|taints'
```

Conditions to read: `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`. `Ready=Unknown`
with `NodeStatusUnknown` means the kubelet stopped reporting — the node is not necessarily down.

```bash
# 2. Get on the node
ssh worker-1
sudo -i

# 3. Kubelet first, it is the answer ~70% of the time
systemctl status kubelet
systemctl is-enabled kubelet
journalctl -u kubelet -f
journalctl -u kubelet --since '10 min ago' --no-pager | tail -50
systemctl restart kubelet

# 4. Kubelet configuration and flags
cat /var/lib/kubelet/config.yaml            # staticPodPath, clusterDNS, cgroupDriver, auth
cat /var/lib/kubelet/kubeadm-flags.env      # --container-runtime-endpoint etc.
cat /etc/kubernetes/kubelet.conf            # kubeconfig the kubelet uses, server: address

# 5. Container runtime
systemctl status containerd
journalctl -u containerd --since '10 min ago' --no-pager | tail -30
crictl info | head -40
crictl ps -a
crictl pods
crictl logs <container-id>
crictl images

# 6. Certificates
kubeadm certs check-expiration
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates
kubeadm certs renew all && systemctl restart kubelet    # control plane; restarts needed

# 7. Resource exhaustion
df -h /var /var/lib/containerd /var/lib/kubelet
df -i /var                                   # inodes: a full inode table looks like free disk
free -m
top -b -n1 | head -15

# 8. Connectivity to the API server
curl -sk https://<control-plane-ip>:6443/healthz
ss -lntp | grep -E '10250|6443|2379'
```

Common root causes, ranked by how often they actually happen:

| Cause | Signature |
| --- | --- |
| kubelet stopped / masked | `systemctl status kubelet` inactive or `Unit is masked` |
| Bad kubelet config YAML | kubelet restarts in a loop, journal shows a parse error and the line |
| Container runtime down | kubelet logs `connection refused` on the CRI socket |
| Disk or inode pressure | Node taints itself `node.kubernetes.io/disk-pressure:NoSchedule`, pods evicted |
| Expired certificates | `x509: certificate has expired or is not yet valid` in kubelet/apiserver logs |
| swap enabled unexpectedly | kubelet refuses to start unless `failSwapOn: false` |
| CNI plugin missing | Node `Ready=False`, reason `NetworkPluginNotReady`, `cni plugin not initialized` |

---

## 3. Static pods and control-plane recovery

The kubelet watches a directory on disk and runs whatever manifests it finds there, with no
API server involved. That is why it is the recovery path when the API server itself is down.

```bash
grep staticPodPath /var/lib/kubelet/config.yaml     # normally /etc/kubernetes/manifests
ls -l /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

Rules that matter:

- The kubelet reacts to file changes within seconds. There is no `kubectl apply`; you edit
  the file, and the pod is recreated. There is no rollout, no revision history.
- Static pods appear in the API as mirror pods named `<pod>-<nodeName>`. You cannot delete
  them with `kubectl` — the kubelet recreates them. Move the file to remove the pod.
- A syntactically invalid manifest means the pod simply never appears. No event, no error in
  `kubectl`, because the API server may be the thing that is broken.

### Recovering a broken kube-apiserver manifest

```bash
ssh control-plane
sudo -i

# kubectl is dead, so work through the runtime
crictl ps -a | grep -E 'kube-apiserver|etcd'
crictl logs --tail=100 $(crictl ps -a --name kube-apiserver -q | head -1)
ls /var/log/pods/                                     # kube-system_kube-apiserver-<node>_<uid>/
tail -100 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log

# Take the manifest out of the watched directory so the kubelet stops thrashing
mkdir -p /root/manifest-backup
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/manifest-backup/
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Fix it in /tmp (bad flag, bad indentation, wrong hostPath, wrong --etcd-servers, ...)
vi /tmp/kube-apiserver.yaml

# Put it back and watch the kubelet pick it up
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
watch -n2 'crictl ps | grep apiserver'
journalctl -u kubelet -f | grep -i apiserver

# Once the container is Running, kubectl comes back
k get --raw='/readyz?verbose'
k get po -n kube-system
```

Typical breakages to rehearse: an invalid `--flag`, a typo in `--etcd-servers`, a wrong
`hostPath` for `/etc/kubernetes/pki`, a missing `- ` in the command list, a wrong
`containerPort`/`--secure-port` mismatch, a bad `--service-cluster-ip-range`.

### Creating a static pod on purpose

```bash
ssh worker-1
sudo -i
k run node-agent --image=busybox:1.36 --restart=Never \
  --dry-run=client -o yaml -- sleep 3600 > /etc/kubernetes/manifests/node-agent.yaml
crictl ps | grep node-agent
exit
k get po -A | grep node-agent          # shows as node-agent-worker-1
```

To delete: remove the file. Nothing else works.

### Where logs live

| Source | Location |
| --- | --- |
| Any container on the node | `/var/log/pods/<ns>_<pod>_<uid>/<container>/0.log` |
| Same, symlinked | `/var/log/containers/*.log` |
| Via runtime | `crictl logs [--tail=N] [-f] <container-id>` |
| kubelet | `journalctl -u kubelet` |
| containerd | `journalctl -u containerd` |
| kubeadm-created control plane | Container logs above, plus `k -n kube-system logs <pod>` when the API is up |

---

## 4. etcd backup and restore

Backups are pure muscle memory. The flags never change.

### Save a snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Verify it

```bash
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table
```

If `etcdctl` is not on the host, read the real values off the running manifest and use the
etcd image:

```bash
grep -E 'cert-file|key-file|trusted-ca-file|listen-client-urls|data-dir' \
  /etc/kubernetes/manifests/etcd.yaml
```

### Full restore procedure

> Destructive. Do this only on a cluster you can lose.

```bash
ssh control-plane
sudo -i

# 1. Restore the snapshot into a NEW data directory. Never restore over /var/lib/etcd.
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restore

# 2. Stop the control plane by moving the static pod manifests away
mkdir -p /root/manifests-off
mv /etc/kubernetes/manifests/*.yaml /root/manifests-off/
# wait until nothing is left
watch -n2 'crictl ps | grep -E "etcd|kube-apiserver"'

# 3. Point the etcd static pod at the restored data directory
vi /root/manifests-off/etcd.yaml
#   volumes:
#     - name: etcd-data
#       hostPath:
#         path: /var/lib/etcd-restore     <-- change this hostPath, NOT the mountPath
#         type: DirectoryOrCreate
# The container mountPath stays /var/lib/etcd, and --data-dir stays /var/lib/etcd.

chown -R etcd:etcd /var/lib/etcd-restore 2>/dev/null || true

# 4. Put the manifests back
mv /root/manifests-off/*.yaml /etc/kubernetes/manifests/

# 5. Verify
watch -n2 'crictl ps | grep -E "etcd|apiserver"'
k get --raw='/readyz?verbose'
k get no
k get po -A
```

Notes that save you:

- `snapshot restore` writes a new data dir; it does not talk to a running etcd, so no
  `--endpoints`/certs are needed for the restore itself.
- Restarting the kubelet (`systemctl restart kubelet`) after step 4 speeds up pickup.
- For a stacked multi-node control plane, restore on every control-plane node with matching
  `--initial-cluster` values, or rebuild members after restoring one.
- If the task gives a different snapshot path or data dir, use theirs, not the ones above.

---

## 5. Control-plane and cluster health checks

```bash
k get --raw='/readyz?verbose'
k get --raw='/livez?verbose'
k get --raw='/healthz'
k get --raw='/metrics' | head
k cluster-info
k cluster-info dump --output-directory=/tmp/dump --all-namespaces
k get componentstatuses            # deprecated but still present on many clusters
k -n kube-system get po -o wide
k get apiservices | grep -v True   # broken aggregated APIs, e.g. metrics-server
```

Certificates:

```bash
kubeadm certs check-expiration
kubeadm certs renew apiserver
kubeadm certs renew all
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A2 Validity
k get csr
k certificate approve <csr-name>
```

---

## 6. kubeadm upgrade flow

Upgrade order: control plane first, one node at a time, then workers. Never skip a minor
version.

```bash
### On the first control-plane node
apt-get update
apt-cache madison kubeadm | head                       # find the exact patch version
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.33.1-1.1
apt-mark hold kubeadm
kubeadm version

kubeadm upgrade plan
kubeadm upgrade apply v1.33.1

# kubelet + kubectl are NOT upgraded by kubeadm
k drain control-plane --ignore-daemonsets --delete-emptydir-data
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.33.1-1.1 kubectl=1.33.1-1.1
apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
k uncordon control-plane

### On every other control-plane node
kubeadm upgrade node        # instead of "upgrade apply"

### On each worker
k drain worker-1 --ignore-daemonsets --delete-emptydir-data
ssh worker-1 'sudo -i'
apt-mark unhold kubeadm && apt-get install -y kubeadm=1.33.1-1.1 && apt-mark hold kubeadm
kubeadm upgrade node
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.33.1-1.1 kubectl=1.33.1-1.1
apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
exit
k uncordon worker-1
k get no                    # confirm the new version before moving to the next node
```

Kubelets may be up to three minor versions older than the API server, never newer.

---

## 7. Networking and DNS debugging

### CNI

```bash
ls /etc/cni/net.d/                     # empty or malformed = nodes stay NotReady
ls /opt/cni/bin/
k -n kube-system get po -l k8s-app=... -o wide     # the CNI DaemonSet, name varies
k -n kube-system logs ds/<cni-daemonset>
k describe node worker-1 | grep -i podcidr
ip a; ip route; ip link show type bridge
```

Symptoms: `NetworkPluginNotReady`, pods stuck in `ContainerCreating` with
`failed to set up sandbox ... plugin type ... not found`, or pod IPs never assigned.

### DNS

```bash
k -n kube-system get po -l k8s-app=kube-dns -o wide
k -n kube-system logs -l k8s-app=kube-dns --tail=50
k -n kube-system get cm coredns -o yaml
k -n kube-system get svc kube-dns                        # usually 10.96.0.10

k run tmp --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default
k run tmp --image=busybox:1.36 --rm -it --restart=Never -- nslookup web-svc.app.svc.cluster.local
k run tmp --image=busybox:1.36 --rm -it --restart=Never -- cat /etc/resolv.conf
k run tmp --image=nicolaka/netshoot --rm -it --restart=Never -- dig +short web-svc.app.svc.cluster.local

k exec -it web-0 -- cat /etc/resolv.conf
```

Checklist when name resolution fails:

1. Are the CoreDNS pods `Running` and `Ready`?
2. Does `kube-dns` Service have endpoints? `k -n kube-system get ep kube-dns`
3. Does the pod's `/etc/resolv.conf` nameserver match the `kube-dns` ClusterIP?
4. Does `clusterDNS` in `/var/lib/kubelet/config.yaml` match it too?
5. Is a NetworkPolicy blocking UDP/TCP 53 egress?
6. Is the CoreDNS `Corefile` valid? A bad edit crashes the loop, check the logs.

### Service reachability

```bash
k get svc,ep -n app
k get endpointslices -n app -l kubernetes.io/service-name=web-svc
k describe svc web-svc -n app | grep -i endpoints
k run tmp --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://web-svc.app:80
k port-forward -n app svc/web-svc 8080:80
```

Empty endpoints -> selector/label mismatch or the pods are not Ready.
Endpoints present but no traffic -> NetworkPolicy, kube-proxy, or `targetPort` is wrong.

---

## 8. Events and audit trail

```bash
k get events -A --sort-by=.lastTimestamp | tail -40
k get events -n app --field-selector type=Warning
k get events -n app --field-selector involvedObject.name=web-0
k describe po web-0 -n app | sed -n '/Events/,$p'
k -n kube-system logs kube-scheduler-control-plane --tail=50
k -n kube-system logs kube-controller-manager-control-plane --tail=50
```

Events are kept for one hour by default. If the incident is older, go to the component logs.

---

## 9. Triage cheat sheet

| Question | Command |
| --- | --- |
| What is broken right now? | `k get po -A \| grep -Ev 'Running\|Completed'` |
| Why is this pod not scheduled? | `k describe po <p> -n <ns>` -> Events |
| What did the app say before it died? | `k logs <p> -n <ns> --previous` |
| Is the node itself sick? | `k describe node <n>` -> Conditions |
| Is the control plane sick? | `k get --raw='/readyz?verbose'` |
| Is the API server even up? | `crictl ps \| grep apiserver` on the control-plane node |
| Are certs expired? | `kubeadm certs check-expiration` |
| Is DNS working? | `k run tmp --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default` |
| Does the Service have backends? | `k get ep <svc> -n <ns>` |
| Can this identity do that? | `k auth can-i <verb> <res> --as=system:serviceaccount:<ns>:<sa>` |
| Where did all the disk go? | `df -h; df -i; crictl images; journalctl --disk-usage` |
