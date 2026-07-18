# driven-infra — Phased IaC Implementation Plan

**Goal:** a single Git repository that can build two Kubernetes environments from scratch — a local 3-node devstack on your Ubuntu desktop and a production AKS cluster in Azure — with every prerequisite tool, VM, cluster, and add-on defined as code.

**Stack:** Azure · Kubernetes · Terraform · Ansible · Vagrant/libvirt · Argo CD

---

## 1. How your requirements map to the design

| Requirement | How the plan satisfies it |
|---|---|
| Local devstack, as close to prod as possible, 1 control plane + 2 workers | Three real Ubuntu VMs (Vagrant + libvirt/KVM) bootstrapped into a cluster with **kubeadm** — the same components (containerd, kubelet, CNI) a production cluster runs. Phases 1–2. |
| Azure infrastructure with AKS via Terraform | Terraform with a remote **azurerm** state backend builds the VNet + AKS cluster (2 worker nodes, managed control plane). Phase 3. |
| Use Ansible where it makes sense | Ansible owns everything that is *machine configuration*: your workstation's tooling, and turning bare VMs into Kubernetes nodes. Terraform owns *cloud resources*. Phase 0 and 2. |
| All prerequisite tools in the IaC | A single tiny `bootstrap.sh` installs Ansible; from then on an Ansible playbook installs and pins every other tool (Terraform, Vagrant, libvirt, kubectl, Helm, az, …). Phase 0. |
| Phased, instructive | Six phases, each with a goal, the files it adds, a "definition of done" you can verify, and the concepts it teaches. |

### The mental model: who owns what

The clean division of labor — worth internalizing because it answers "which tool do I reach for?" for the rest of the project:

- **Terraform** declares *resources that a cloud API can create*: resource groups, networks, the AKS cluster itself. It excels at "make reality match this description" for things with APIs.
- **Ansible** configures *machines you can SSH into* (your desktop, the devstack VMs): install packages, write config files, run `kubeadm`. It excels at idempotent, ordered configuration of an OS.
- **Vagrant** declares *local VMs* the way Terraform declares cloud resources. In the devstack it plays the role Azure plays in production: it supplies the machines.
- **Argo CD (GitOps)** owns *everything inside Kubernetes* once a cluster exists: cert-manager, the Gateway, monitoring, your apps. Both clusters sync from the same `gitops/` directory, which is what keeps devstack and prod in parity — and fits the repo's name: the infra is *driven* from Git.

```
                YOUR DESKTOP (devstack)                    AZURE (prod)
        ┌────────────────────────────────┐      ┌────────────────────────────────┐
        │ Vagrant + libvirt (Phase 1)    │      │ Terraform (Phase 3)            │
        │  ├─ cp-1      (control plane)  │      │  ├─ VNet + subnet              │
        │  ├─ worker-1                   │      │  └─ AKS                        │
        │  └─ worker-2                   │      │      ├─ managed control plane  │
        │ Ansible + kubeadm (Phase 2)    │      │      └─ 2 worker nodes         │
        └───────────────┬────────────────┘      └───────────────┬────────────────┘
                        │                                       │
                        └───────────── Argo CD syncs ───────────┘
                                          from
                              gitops/  (Phases 4–5: cert-manager,
                              Gateway API, Prometheus/Grafana)
```

Note the one honest asymmetry: AKS's control plane is managed by Azure, so you never see a control-plane *node* there. The devstack's `cp-1` is your window into what Azure hides — that's a feature for learning, and it's why kubeadm-on-VMs is more instructive than kind or k3s.

---

## 2. Target repository layout

Built up phase by phase; shown complete here so you always know where a file belongs.

```
driven-infra/
├── README.md                     # quickstart + pointers into docs/
├── Makefile                      # single entrypoint: make bootstrap / devstack / azure / ...
├── docs/                         # one instructive guide per phase
│   ├── 00-workstation.md
│   ├── 01-devstack-vms.md
│   ├── 02-kubernetes-kubeadm.md
│   ├── 03-azure-aks.md
│   ├── 04-gitops-addons.md
│   └── 05-ci-and-operations.md
├── bootstrap/
│   ├── bootstrap.sh              # ONLY hand-run script: installs Python + Ansible
│   └── versions.yml              # pinned versions for every tool (single source of truth)
├── ansible/
│   ├── ansible.cfg
│   ├── inventories/
│   │   ├── local/hosts.yml       # localhost (workstation setup)
│   │   └── devstack/hosts.yml    # cp-1, worker-1, worker-2 (static IPs)
│   ├── playbooks/
│   │   ├── workstation.yml       # Phase 0: install all tools on your desktop
│   │   ├── cluster.yml           # Phase 2: bare VMs -> Kubernetes cluster
│   │   └── argocd-bootstrap.yml  # Phase 4: install Argo CD, register the repo
│   └── roles/
│       ├── workstation_tools/    # terraform, vagrant+libvirt, kubectl, helm, az, k9s
│       ├── k8s_common/           # swap, kernel modules, sysctl, /etc/hosts
│       ├── containerd/           # runtime install + SystemdCgroup config
│       ├── kubeadm_install/      # kubelet/kubeadm/kubectl, apt pin + hold
│       ├── control_plane/        # kubeadm init, admin.conf, join token
│       ├── worker/               # kubeadm join
│       └── cilium/               # CNI install via Helm
├── devstack/
│   └── Vagrantfile               # 3 VMs, static IPs, libvirt provider
├── terraform/
│   ├── bootstrap-state/          # run-once: RG + storage account for remote state
│   ├── modules/
│   │   ├── network/              # VNet, subnets, NSG
│   │   └── aks/                  # cluster, node pool, identity, outputs
│   └── envs/
│       └── prod/                 # backend "azurerm", composes the modules
├── gitops/
│   ├── bootstrap/                # Argo CD app-of-apps ("root" Application)
│   ├── addons/                   # base manifests/Helm values per add-on
│   │   ├── cert-manager/
│   │   ├── gateway/              # Gateway API CRDs + Envoy Gateway
│   │   └── monitoring/           # kube-prometheus-stack
│   └── clusters/
│       ├── devstack/             # overlays: what's different locally
│       └── aks/                  # overlays: what's different in Azure
└── .github/workflows/
    └── ci.yml                    # fmt/validate/lint for tf, ansible, yaml, manifests
```

---

## 3. Key decisions and why

**kubeadm on libvirt VMs for the devstack.** kubeadm is the upstream, vendor-neutral cluster bootstrapper — it's what "real" clusters (and your CKA-style learning) are built on. Real VMs mean real kernels, real networking, and real systemd, so Ansible has genuine work to do and the devstack behaves like production machines. libvirt/KVM is native to Ubuntu, faster than VirtualBox, and fully scriptable.

**Cilium as the CNI in both environments.** AKS offers "Azure CNI powered by Cilium" as a first-class option, so running open-source Cilium in the devstack gives you the same dataplane family in both places — the strongest parity available. Bonus: Cilium's LB-IPAM + L2 announcements can hand out LoadBalancer IPs on your desktop network, which solves the classic "LoadBalancer services hang `<pending>` on bare metal" problem without adding MetalLB.

**Gateway API with Envoy Gateway (instead of ingress-nginx).** You asked for a Kubernetes Gateway — Gateway API is the modern successor to Ingress. Envoy Gateway is a reference-quality implementation that runs identically on the devstack and AKS, so your `HTTPRoute`s are portable. (Azure's own implementation, Application Gateway for Containers, is Azure-only; we note it as a future option in Phase 3's docs.)

**Argo CD for GitOps.** Argo CD's UI makes the sync/drift/health model *visible*, which is worth a lot while learning; Flux is the leaner alternative and the docs will note the trade-off. The app-of-apps pattern means one root Application per cluster pulls in everything else.

**Terraform remote state in Azure Storage.** Production-grade from day one: state is shared, durable, and locked via blob leases. The chicken-and-egg (state storage must exist before the backend can use it) is solved by a tiny run-once `bootstrap-state` stack that itself uses local state — a pattern you'll meet in every real Terraform shop.

**One `versions.yml` to pin everything.** Kubernetes, Terraform, Cilium, Helm chart versions all live in one file that Ansible, the Vagrantfile, and docs reference. Reproducibility is the whole point of IaC; unpinned versions quietly break it.

---

## 4. The phases

Each phase ends with a working, verifiable state — you never have two broken layers at once.

---

### Phase 0 — Repo scaffold & workstation bootstrap (Ansible)

**Goal:** your Ubuntu desktop gets every prerequisite tool from code, and the repo skeleton exists.

The one unavoidable manual step in any IaC project is installing the first tool. We shrink it to a ~15-line `bootstrap.sh` that installs Python and Ansible (via `pipx`), then *everything else* is a real Ansible playbook run against `localhost`:

```bash
git clone git@github.com:<you>/driven-infra && cd driven-infra
./bootstrap/bootstrap.sh          # installs Ansible only
make workstation                  # ansible-playbook playbooks/workstation.yml -K
```

The `workstation_tools` role installs and pins: Terraform (HashiCorp apt repo), Vagrant + the vagrant-libvirt plugin, the libvirt/KVM stack (`qemu-kvm`, `libvirt-daemon-system`, your user added to the `libvirt` group), kubectl (pkgs.k8s.io repo), Helm, Azure CLI, and quality-of-life tools (k9s, jq, kustomize). Versions come from `bootstrap/versions.yml`.

**What you'll learn:** Ansible fundamentals on the safest possible target (your own machine) — inventories, `ansible.cfg`, roles, handlers, apt-repo management, idempotency (`changed=0` on the second run is the point), and why `become` + `-K` exist.

**Definition of done:** `make workstation` twice in a row → second run reports zero changes; `terraform version`, `vagrant version`, `virsh list`, `kubectl version --client`, `az version` all succeed; you're in the `libvirt` group.

---

### Phase 1 — Devstack VMs (Vagrant + libvirt)

**Goal:** three Ubuntu VMs with fixed IPs, reachable by SSH — machines, no Kubernetes yet.

A single `Vagrantfile` defines the cluster from a loop over a node table:

| VM | Role (later) | IP | vCPU | RAM |
|---|---|---|---|---|
| cp-1 | control plane | 10.10.0.10 | 2 | 4 GB |
| worker-1 | worker | 10.10.0.11 | 2 | 4 GB |
| worker-2 | worker | 10.10.0.12 | 2 | 4 GB |

(~12 GB RAM total while running; we'll trim if your desktop is tight.) Static IPs matter because the Ansible inventory in `inventories/devstack/hosts.yml` refers to them, and because kubeadm certificates embed the control-plane endpoint. The Ubuntu box version is pinned in `versions.yml` like everything else.

Deliberate choice: **Vagrant only creates machines; it does not provision them.** We skip Vagrant's Ansible provisioner and run Ansible separately in Phase 2. That keeps the "supply machines" and "configure machines" layers cleanly separated — exactly the split you'll have in Azure, where Terraform supplies and (managed) tooling configures.

**What you'll learn:** libvirt networking, Vagrant boxes and the provider model, why static IPs + inventory files beat dynamic discovery for a fixed-size lab.

**Definition of done:** `make devstack-up` → `vagrant status` shows 3 running VMs; `ansible -i inventories/devstack all -m ping` succeeds; VMs can ping each other; `make devstack-destroy && make devstack-up` rebuilds cleanly.

---

### Phase 2 — Bare VMs → Kubernetes cluster (Ansible + kubeadm)

**Goal:** `kubectl get nodes` on your desktop shows `cp-1`, `worker-1`, `worker-2` — all `Ready`.

This is the most instructive phase in the repo: `cluster.yml` performs, in code, exactly what the official "creating a cluster with kubeadm" docs describe by hand:

1. **`k8s_common`** — disable swap, load `overlay` + `br_netfilter`, set sysctls (`ip_forward`, bridge-nf-call-iptables), populate `/etc/hosts`. *Why:* kubelet refuses swap; pod networking needs bridged traffic to traverse iptables.
2. **`containerd`** — install the runtime, generate config with `SystemdCgroup = true`. *Why:* kubelet and the runtime must agree on the cgroup driver, and this mismatch is the #1 classic kubeadm failure.
3. **`kubeadm_install`** — kubelet/kubeadm/kubectl from pkgs.k8s.io at the pinned minor version, then `apt-mark hold`. *Why:* an unattended upgrade that bumps kubelet out of skew with the control plane breaks nodes.
4. **`control_plane`** (cp-1 only) — `kubeadm init` driven by a `ClusterConfiguration` file (not CLI flags — config files are reviewable IaC), pod CIDR `10.244.0.0/16`, control-plane endpoint `10.10.0.10`. Fetches `admin.conf` to your desktop, merged into `~/.kube/config` as context `devstack`. Generates a fresh join token.
5. **`worker`** (worker-1/2) — `kubeadm join` with the token, guarded so re-runs are no-ops.
6. **`cilium`** — install Cilium via Helm at a pinned chart version. Until the CNI lands, nodes sit `NotReady` — you'll see that happen live, which teaches what a CNI actually does.

Idempotency is enforced with guards like "skip init if `/etc/kubernetes/admin.conf` exists" — you can re-run `make devstack-cluster` any time.

**What you'll learn:** what kubeadm actually does (certs, static pods, kubeconfig), control plane vs worker anatomy, the runtime/kubelet/CNI relationship, token-based joins, multi-host Ansible orchestration (facts from one host driving tasks on another).

**Definition of done:** 3 nodes `Ready`; `kubectl get pods -A` healthy; a test nginx Deployment with 2 replicas lands across both workers and responds via a NodePort; full teardown/rebuild (`make devstack-destroy devstack-up devstack-cluster`) works end-to-end.

---

### Phase 3 — Azure foundation: AKS via Terraform

**Goal:** a production AKS cluster (managed control plane + 2 workers) built entirely by Terraform, with remote state.

**Step 3a — run-once state bootstrap** (`terraform/bootstrap-state/`): a minimal local-state stack creating a resource group, storage account, and `tfstate` blob container. You run it once, note the outputs, and its local state file stays gitignored. The docs explain the chicken-and-egg honestly rather than hiding it.

**Step 3b — the real stack** (`terraform/envs/prod/`) with `backend "azurerm"`, composed from two modules you write (not registry modules — writing them is the instruction):

- `modules/network` — VNet `10.20.0.0/16`, an AKS subnet, NSG. *Why bring your own VNet:* AKS will happily create one, but owning the network is what real environments need (peering, private endpoints, firewalls later).
- `modules/aks` — `azurerm_kubernetes_cluster` with: system-assigned managed identity (no service-principal secrets), a default node pool of **2 nodes** (mirroring the devstack's two workers; Azure runs the control plane), **Azure CNI powered by Cilium** (network_plugin `azure`, network_dataplane `cilium` — the parity decision from §3), a pinned `kubernetes_version` matching the devstack's minor version, and outputs for the kubeconfig.

Authentication for local runs is simply `az login` (your identity); CI auth via OIDC is deferred to Phase 5 so you learn it separately. Cost honesty: this cluster costs real money (~$150–200/month at 2×D2s-class nodes); the docs include `terraform destroy` and a "stop the node pool" runbook, and nothing else in the repo assumes AKS is always up.

**What you'll learn:** Terraform module design, variables/outputs/composition, remote state and locking, plan-vs-apply discipline, managed identities, and precisely what AKS manages for you (compare `kubectl get pods -n kube-system` here vs the devstack — same shapes, different owners).

**Definition of done:** `terraform plan` clean after apply (no perma-diff); `az aks get-credentials` merges context `aks`; `kubectl --context aks get nodes` shows 2 `Ready` nodes; state visible in the storage container; destroy + re-apply works.

---

### Phase 4 — Add-ons on both clusters via GitOps (Argo CD)

**Goal:** both clusters run cert-manager, Gateway API (Envoy Gateway), and Prometheus/Grafana — and the *only* way anything gets installed from now on is a Git commit.

One small imperative step per cluster (`ansible/playbooks/argocd-bootstrap.yml`): install Argo CD and apply the cluster's **root Application** pointing at `gitops/clusters/<name>/`. From there, app-of-apps takes over.

Structure: `gitops/addons/*` holds shared base config per add-on; `gitops/clusters/{devstack,aks}/` holds thin overlays for what legitimately differs:

- **cert-manager** — same everywhere; devstack overlay adds a self-signed ClusterIssuer, the AKS overlay a Let's Encrypt issuer (when you attach DNS).
- **gateway** — Gateway API CRDs + Envoy Gateway, one shared `Gateway`, portable `HTTPRoute`s. On AKS the Gateway's LoadBalancer Service gets an Azure LB IP automatically; on the devstack the overlay enables **Cilium LB-IPAM + L2 announcements** with a small IP pool so the same Service gets a LAN-reachable IP. This asymmetry-and-its-fix is one of the best bare-metal-vs-cloud lessons in the project.
- **monitoring** — kube-prometheus-stack; overlays differ only in resource sizing and storage class (`local-path`/host storage on devstack, Azure Disk on AKS).

**What you'll learn:** GitOps sync/health/drift (change a value in Git, watch Argo CD converge; `kubectl edit` something, watch it revert), base/overlay design, Gateway API resource model (GatewayClass → Gateway → HTTPRoute), and cert-manager's Issuer/Certificate flow.

**Definition of done:** Argo CD UI shows all apps `Synced/Healthy` on both clusters; a demo app is reachable through the Gateway on a devstack LAN IP and an Azure public IP with TLS; Grafana shows node metrics for all nodes in both clusters; a Git revert rolls back a change with no kubectl involved.

---

### Phase 5 — CI, operations & docs polish

**Goal:** the repo defends its own correctness and a stranger (or future-you) can operate it from the docs.

GitHub Actions on every PR: `terraform fmt -check` + `validate` + `tflint`, `ansible-lint`, `yamllint`, and `kubeconform` over rendered GitOps manifests — every layer gets a linter. A `terraform plan` job with Azure OIDC federated credentials (no stored secrets — the modern pattern, and the Phase-3 deferral pays off here as its own lesson) is the stretch goal. Runbooks in `docs/05`: rebuild devstack from zero, AKS cost stop/start, upgrading the pinned Kubernetes version across *both* environments (one `versions.yml` edit → devstack first, then AKS — a real upgrade workflow in miniature), and disaster-recovery notes on Terraform state.

**Definition of done:** CI green on main; an intentionally misformatted PR fails; `README.md` walks zero → both clusters in order; every phase doc reviewed against what was actually built.

---

## 5. Sequencing, effort & session plan

Phases are strictly ordered by dependency: 0 → 1 → 2 → 4(devstack) can proceed with no Azure account touched; 3 → 4(aks) can start any time after 0. A sensible rhythm is one phase per working session. Working mode (per `CLAUDE.md`): Joel implements everything; Claude breaks each phase into small tasks via the `breakdown` skill, guides, and reviews — every task ends with a verification Joel runs himself:

| Phase | Rough effort | Azure cost incurred |
|---|---|---|
| 0 Workstation bootstrap | 1 session | none |
| 1 Devstack VMs | 1 session | none |
| 2 kubeadm cluster | 1–2 sessions (the meaty one) | none |
| 3 Terraform + AKS | 1–2 sessions | starts here (destroyable) |
| 4 GitOps add-ons | 1–2 sessions | LB + disks marginal |
| 5 CI & polish | 1 session | none |

## 6. Known risks & gotchas (so they don't surprise us)

**Desktop resources:** ~12 GB RAM for the devstack while running; we can drop workers to 2–3 GB each if needed. **libvirt group membership** requires re-login after Phase 0. **Version skew:** kubelet must stay within one minor of the control plane — the `apt-mark hold` + `versions.yml` discipline exists for this. **Vagrant box choice** for libvirt (a pinned Ubuntu 24.04 box) is validated in Phase 1 before anything depends on it. **AKS cost** is real; nothing in the design requires AKS to stay up between sessions. **Secrets:** nothing sensitive is ever committed — kubeconfigs, join tokens, and Terraform state are all gitignored or remote; the CI phase adds a secret-scanning check.

---

## 7. Immediate next step

Phase 0: ask Claude to break the phase down (`breakdown` skill). You'll build the directory tree, `bootstrap.sh`, `versions.yml`, the `workstation_tools` role, `Makefile`, and `docs/00-workstation.md` yourself, task by task, verifying each one as you go — with Claude guiding and reviewing, never implementing (see `CLAUDE.md`).
