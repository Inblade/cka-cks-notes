# CKA / CKS Study Notes

Working notes and self-made practice drills for the Certified Kubernetes Administrator
(CKA) and Certified Kubernetes Security Specialist (CKS) curricula. Everything here was
written from hands-on production operations experience and the public upstream
documentation, then compressed into the shortest form that still runs. The goal is not to
re-explain Kubernetes from zero — it is to give an operator who already runs clusters a
dense, copy-pasteable reference for the specific mechanics both exams expect you to
perform under time pressure: imperative `kubectl`, control-plane recovery, etcd
snapshots, RBAC review, Pod Security Standards, seccomp/AppArmor, and supply-chain
controls. Commands are written against Kubernetes 1.30-1.33 and `kubeadm`-built clusters.

## Repository structure

```
cka-cks-notes/
├── README.md
├── LICENSE
├── .gitignore
├── notes/
│   ├── cka-core.md               # workloads, scheduling, services, storage, kubectl drills
│   ├── cka-troubleshooting.md    # pod/node triage trees, static pods, etcd backup & restore
│   ├── cks-cluster-hardening.md  # apiserver flags, audit, encryption at rest, RBAC, kubelet
│   └── cks-runtime-security.md   # PSS, seccomp, AppArmor, Falco, image scanning, sandboxing
└── docs/
    ├── exam-strategy.md          # time budgeting, aliases, vim, context discipline, verification
    └── practice-tasks.md         # 16 original drills with solutions and verification steps
```

## How to use this

1. Build a throwaway cluster you are allowed to break. Two options that mirror the exam
   environment closely enough:

   ```bash
   # Option A - kind, fastest loop, good for workloads/RBAC/PSS/NetworkPolicy
   kind create cluster --name study --image kindest/node:v1.33.0 --config - <<'EOF'
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
     - role: control-plane
     - role: worker
     - role: worker
   EOF

   # Option B - kubeadm on 3 VMs, required for etcd restore, static pods,
   # kubelet config, certificate and upgrade practice
   kubeadm init --pod-network-cidr=10.244.0.0/16
   ```

   Use Option B at least once. Several CKA/CKS domains (etcd restore, broken static pod
   recovery, kubelet hardening, `kubeadm upgrade`) cannot be practised properly on a
   managed or containerised control plane.

2. Read `notes/cka-core.md` and drill the "kubectl commands worth memorizing" table until
   you can produce a Deployment manifest imperatively without thinking.
3. Read `notes/cka-troubleshooting.md` with a terminal open, and deliberately break things:
   corrupt a static pod manifest, stop `kubelet`, fill a disk, delete a CNI config.
4. Move to the CKS notes. Apply each control to the cluster from step 1 and verify it
   actually blocks what it claims to block.
5. Work `docs/practice-tasks.md` end to end with a timer, solutions covered. Then re-read
   `docs/exam-strategy.md` and repeat the tasks you missed.

## What this is NOT

- **Not an exam dump.** It contains no real exam questions, no recalled or paraphrased
  task text, no leaked scenarios, and no answer keys to anything but the drills written
  here from scratch. Sharing or seeking actual exam content violates the certification
  program's NDA and is a fast route to a revoked certificate.
- **Not a guarantee.** Reading these notes will not pass the exam for you. Both exams are
  hands-on and time-boxed; only repeated practice on a real cluster moves the needle.
- **Not a replacement for the official documentation.** During the exam you are allowed a
  specific documentation domain. Learn to navigate it — these notes assume you will.
- **Not affiliated with, sponsored by, or endorsed by** the Linux Foundation or the Cloud
  Native Computing Foundation. CKA, CKS, and Kubernetes are trademarks of their respective
  owners.
- **Not version-pinned forever.** Curriculum weights and Kubernetes APIs change. Always
  confirm the current published curriculum before you sit the exam.

## Curriculum domains

### CKA — Certified Kubernetes Administrator

| Domain | Weight | Covered in |
| --- | --- | --- |
| Cluster Architecture, Installation & Configuration | 25% | `notes/cka-core.md`, `notes/cka-troubleshooting.md` |
| Workloads & Scheduling | 15% | `notes/cka-core.md` |
| Services & Networking | 20% | `notes/cka-core.md` |
| Storage | 10% | `notes/cka-core.md` |
| Troubleshooting | 30% | `notes/cka-troubleshooting.md` |

### CKS — Certified Kubernetes Security Specialist

| Domain | Weight | Covered in |
| --- | --- | --- |
| Cluster Setup | 15% | `notes/cks-cluster-hardening.md` |
| Cluster Hardening | 15% | `notes/cks-cluster-hardening.md` |
| System Hardening | 10% | `notes/cks-cluster-hardening.md`, `notes/cks-runtime-security.md` |
| Minimize Microservice Vulnerabilities | 20% | `notes/cks-runtime-security.md` |
| Supply Chain Security | 20% | `notes/cks-runtime-security.md` |
| Monitoring, Logging and Runtime Security | 20% | `notes/cks-runtime-security.md` |

CKS requires an active CKA certification before you can sit it. Plan the two together:
CKA first, then CKS while the muscle memory is still fresh.

## Conventions used in these notes

- `k` is assumed to be aliased to `kubectl`.
- `$do` expands to `--dry-run=client -o yaml`, `$now` to `--force --grace-period=0`.
- Node-level commands assume `kubeadm` defaults: `/etc/kubernetes/manifests`,
  `/etc/kubernetes/pki`, `/var/lib/kubelet/config.yaml`, containerd as the CRI.
- Anything destructive is marked. Do not paste it into a cluster you care about.

## License

MIT — see [LICENSE](LICENSE). Copyright (c) 2026 Danylo Kochetov.
