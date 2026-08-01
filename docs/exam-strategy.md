# Exam Strategy

Operational notes on how to work through a hands-on Kubernetes exam efficiently. Nothing here
describes actual exam content — this is about process, tooling and time management, all of
which you can practise on your own cluster.

---

## 1. Time management

Both CKA and CKS are 2 hours (120 minutes) of hands-on work in a live terminal, with roughly
15-20 tasks, each carrying a published percentage weight. The arithmetic that follows from
that:

| Quantity | Value |
| --- | --- |
| Total time | 120 min |
| Reserve for setup (aliases, vim, first read-through) | 4 min |
| Reserve for final verification pass | 10 min |
| Working time | ~106 min |
| Tasks | ~17 |
| Average per task | ~6 min |
| Per weight point | ~1.06 min per 1% of the exam |

Budget per task = its weight percentage, in minutes, roughly 1:1. A 4% task deserves about
4 minutes; a 9% task deserves about 9. Weights are shown next to each task — read them.

### The flag-and-skip rule

If a task has consumed **1.5x its budget** and you are not clearly minutes from done, flag it
and move on. A 4% task you have spent 6 minutes on is now competing with two untouched 5%
tasks. This single rule is worth more than any amount of extra Kubernetes knowledge.

Practical version:

1. First pass: do every task you can finish inside its budget. Flag anything expensive.
2. Second pass: return to flagged tasks, largest weight first.
3. Third pass: verification. Re-read the task text, confirm namespace and cluster.

Do not start a task you cannot finish with under 4 minutes left; spend that time verifying
work you already did instead.

### Score model

- Scoring is **partial credit** and **per task independent**. A task with four requirements
  where you satisfied three usually scores three quarters, not zero.
- There are **no marks for elegance**. An imperative one-liner scores identically to a
  hand-crafted, commented manifest. Nobody reads your YAML.
- Failing task 3 does not affect task 4. Never let a bad task poison your pacing.
- Because credit is partial, always do the easy part of a hard task. If you cannot make the
  webhook work, still create the Namespace, ServiceAccount and RoleBinding it asked for.

---

## 2. Context discipline

Each task specifies the cluster to work on and gives you the command. **Run it first, every
single time, before reading the rest of the task.** Work done on the wrong cluster scores zero
and you will not notice until the end.

```bash
kubectl config use-context <given-context-name>
kubectl config get-contexts
kubectl config current-context
```

Make it a physical habit: copy the context line, paste, enter, then read the task. Also pin the
namespace when a task lives in one:

```bash
kubectl config set-context --current --namespace=app
kubectl config view --minify | grep namespace
```

Then reset it, or remember it, before the next task. A namespace left set from a previous task
is the second most common way to lose points. If in doubt, pass `-n` explicitly on every
command and never rely on the default.

---

## 3. First four minutes: environment setup

```bash
# Aliases and shell completion
alias k=kubectl
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
source <(kubectl completion bash)
complete -F __start_kubectl k

# Persist them so a new shell or an ssh round trip does not lose them
cat >> ~/.bashrc <<'EOF'
alias k=kubectl
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
source <(kubectl completion bash)
complete -F __start_kubectl k
EOF
```

Usage:

```bash
k create deploy web --image=nginx $do > web.yaml
k delete po web-0 $now
```

Vim, which matters more than it sounds:

```bash
cat > ~/.vimrc <<'EOF'
set expandtab
set tabstop=2
set shiftwidth=2
set number
set ai
EOF
```

Inside vim, before pasting YAML from the documentation:

```
:set paste
```

Without `:set paste`, autoindent compounds every line and the YAML you paste is silently
broken. Turn it off with `:set nopaste` when you go back to typing. Other survival commands:

| Keystroke | Effect |
| --- | --- |
| `:set paste` / `:set nopaste` | Paste mode on/off |
| `:%s/old/new/g` | Replace everywhere |
| `gg=G` | Re-indent whole file (do not use on YAML) |
| `V` then `>` / `<` | Visual line select, indent/outdent |
| `.` | Repeat last change |
| `u` / `Ctrl-r` | Undo / redo |
| `:wq` / `:q!` | Save and quit / discard |

tmux, only if the environment permits it and you already use it daily. A second pane for
`watch -n2 'k get po -n app'` while you edit in the first is genuinely useful; fumbling with
prefix keys under time pressure is not. The exam runs inside a browser-based remote desktop or
terminal, so terminal keybindings can behave unexpectedly and mouse selection may be awkward.
If you have not rehearsed tmux in that environment, do not introduce it on exam day.

---

## 4. Imperative-first workflow

The generator is almost always faster than the editor. Generate, then patch the two or three
fields the task actually asks about.

```bash
# 1. Generate
k create deploy web -n app --image=nginx:1.27 --replicas=3 $do > web.yaml

# 2. Edit only what the task requires
vi web.yaml

# 3. Apply
k apply -f web.yaml

# 4. Verify
k -n app rollout status deploy/web
```

When the object already exists, prefer in-place mutation over rewriting a manifest:

```bash
k -n app set image deploy/web nginx=nginx:1.27.3
k -n app scale deploy/web --replicas=5
k -n app edit deploy/web
k -n app patch deploy/web --type=merge -p '{"spec":{"template":{"spec":{"nodeSelector":{"disktype":"ssd"}}}}}'
k -n app annotate deploy/web kubernetes.io/change-cause='image bump' --overwrite
```

For anything the generators do not cover, a heredoc beats fighting the editor:

```bash
cat <<'EOF' | k apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: app
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
EOF
```

Quote the heredoc delimiter (`<<'EOF'`) so the shell does not expand `$` inside your YAML.

Some objects cannot be generated at all — NetworkPolicy, PV, PVC, StorageClass, RuntimeClass,
PriorityClass, most `securityContext` work. For those, either use a heredoc from memory or
copy the closest example out of the documentation and edit it. Know which is which before the
exam so you do not waste 30 seconds discovering that `kubectl create networkpolicy` does not
exist.

### Do not fight the editor

Ranked by speed for a small change to a live object:

1. `k set image` / `k scale` / `k label` / `k patch` — no editor at all.
2. `k edit` — for a couple of fields on an existing object.
3. `k get <obj> -o yaml > f.yaml`, edit, `k replace -f f.yaml --force` — when `edit` rejects
   an immutable field.
4. Heredoc — for a brand new object with no generator.

If `k edit` throws an immutable-field error, it saves your attempt to a temp file and prints
the path. Use `k replace --force -f <that file>` rather than retyping.

---

## 5. Using the allowed documentation

You get one browser tab with a specific documentation domain (kubernetes.io and its
sub-sites; the exact allowed list is published in the exam handbook — read it beforehand).
Efficiency here is a skill, so practise it.

- **Search inside the site**, not with a general search engine. Type a distinctive phrase, not
  a question: `networkpolicy default deny`, `seccomp localhost profile`, `runtimeclass`.
- **Go straight to the YAML block**, use `Ctrl-F` for the field you need, copy just that
  fragment. Do not read the prose.
- **Know your five landing pages** cold, so you can navigate without searching:
  Network Policies, Configure a Pod to Use a PersistentVolume, Pod Security Standards,
  Encrypting Confidential Data at Rest, Audit Logging (for CKS).
- **Bookmark before the exam** if bookmarks are permitted in your environment; otherwise
  memorise the shortest search phrase that lands each page first.
- **`kubectl explain` is faster than the browser** for field names, and it never lies about
  the version you are actually running:

  ```bash
  k explain networkpolicy.spec.ingress --recursive
  k explain pod.spec.securityContext.seccompProfile
  k explain sts.spec.volumeClaimTemplates --recursive | head -30
  ```

- **`k api-resources`** resolves the "what is the short name / is it namespaced / which API
  group" question instantly.
- Never paste from the browser into vim without `:set paste`.

---

## 6. Verifying your own work

Verification is where partial credit turns into full credit, and it costs seconds.

For every task, before you leave it:

1. **Re-read the task statement.** Count the requirements. Tasks routinely bundle four things
   into two sentences: create the Deployment, in namespace X, with N replicas, exposed on
   port P. Missing the fourth clause is the most common failure mode.
2. **Confirm the cluster and namespace.** `k config current-context`, then
   `k get <thing> -n <ns>`.
3. **Confirm the object exists and is healthy.**

   ```bash
   k -n app get deploy,po,svc
   k -n app rollout status deploy/web --timeout=30s
   k -n app describe deploy web | head -30
   ```

4. **Diff intent against reality.** Read back what the API actually stored:

   ```bash
   k -n app get deploy web -o yaml | grep -A6 'strategy:'
   k -n app get po -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
   k -n app get netpol default-deny -o yaml
   diff <(k -n app get deploy web -o yaml | yq 'del(.metadata.managedFields)') expected.yaml
   ```

5. **Test the behaviour, not just the object**, when the task is about behaviour:

   ```bash
   k auth can-i list pods -n app --as=system:serviceaccount:app:deploy-bot
   k run probe -n other --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- --timeout=3 http://api.app:8080
   k exec -it hardened -n app -- grep Seccomp /proc/1/status
   ```

A creation task that produced an object stuck in `Pending` or `CrashLoopBackOff` may still
score, but check whether the failure is caused by your own mistake — a typo'd image, a
missing ConfigMap — before you move on.

---

## 7. Tasks that send you to a node

Some tasks require work on a node over ssh: etcd snapshots, static pods, kubelet config,
upgrades, loading AppArmor or seccomp profiles.

```bash
ssh node01                # the task tells you the hostname; do not guess
sudo -i                   # most of it needs root
# ... do the work ...
exit                      # leave the root shell
exit                      # leave the node -- CRITICAL
```

Rules:

- **Always `exit` back to the base terminal** when the task is done. Doing the next task on the
  wrong host wastes minutes and can damage work you already completed.
- Your aliases do not exist on the node unless you set them there. On a node, type `kubectl`,
  or re-source your `.bashrc` additions.
- `kubectl` on a worker node may have no kubeconfig at all. Node-side work is `systemctl`,
  `journalctl`, `crictl`, `etcdctl` and file editing, not `kubectl`.
- Back up before you edit anything in `/etc/kubernetes/manifests` or
  `/var/lib/kubelet/config.yaml`:

  ```bash
  cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
  ```

- After changing a static pod manifest or kubelet config, wait and verify. The kubelet takes a
  few seconds, and a broken manifest gives no `kubectl` error because the API server is the
  thing that broke.

  ```bash
  systemctl restart kubelet
  watch -n2 'crictl ps | grep -E "apiserver|etcd"'
  ```

- Prefer `sudo -i` once over prefixing every command with `sudo`; forgetting `sudo` on one
  command in a sequence produces confusing partial failures.

---

## 8. Environment notes

- **Copy-paste**: the browser terminal often uses `Ctrl-Shift-C` / `Ctrl-Shift-V`, and
  right-click may be bound to paste. Confirm this in the first minute, not mid-task. Some
  environments mangle long pastes — for critical YAML, paste into a file and read it back
  before applying.
- **Multi-line pastes into vim** need `:set paste`. Every time.
- **The terminal is remote**, so there is input latency. Type deliberately; a mistyped
  `--force --grace-period=0` on the wrong object is unrecoverable.
- **Scrollback may be limited.** Pipe long output through `head`, `tail`, or into a file rather
  than scrolling: `k get po -A > /tmp/pods.txt`.
- **A notepad is provided** in the exam interface. Use it to track flagged task numbers and
  their weights, so your second pass is ordered by value rather than by memory.
- **Clock**: the interface shows remaining time. Glance at it at the end of every task, not
  mid-task — checking constantly costs focus.
- **Breaks**: you may step away, but the clock keeps running. Plan not to.

---

## 9. Pre-exam checklist

- [ ] Completed the full task set in `docs/practice-tasks.md` under time, twice.
- [ ] Can generate a Deployment, Service, Job, CronJob, Role and RoleBinding imperatively
      without looking anything up.
- [ ] Can write a NetworkPolicy from memory, including the DNS egress rule.
- [ ] Can perform an etcd snapshot and a full restore on a kubeadm cluster, unaided.
- [ ] Can recover a deliberately broken `kube-apiserver` static pod with `kubectl` unavailable.
- [ ] Can label a namespace for Pod Security Standards and fix a pod that the policy rejects.
- [ ] Know the exact `.vimrc` and alias block, and can type it in under 60 seconds.
- [ ] Have read the current exam handbook: allowed documentation, ID requirements, workspace
      rules, retake policy, and the certification's period of validity.
- [ ] Know the current curriculum version and its domain weights.
