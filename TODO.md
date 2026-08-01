# TODO — driven-infra

Progress tracker for the phased IaC build. The **plan and the "why"** live in
[docs/PLAN.md](docs/PLAN.md); this file is only the checklist. When reality drifts
from the plan, fix `docs/PLAN.md` first, then reconcile the boxes here.

**Working mode (see [CLAUDE.md](CLAUDE.md)):** Joel implements every item; Claude
breaks each phase into small tasks, guides, and reviews. Each task ends with a
verification Joel runs himself — a box is only checked once that check passes on
Joel's machine.

**Legend:** `[ ]` not started · `[~]` in progress · `[x]` done & verified

Phases are strictly ordered by dependency. `0 → 1 → 2 → 4(devstack)` needs no
Azure account; `3 → 4(aks)` can start any time after `0`.

---

## Phase 0 — Repo scaffold & workstation bootstrap (Ansible)

**Goal:** the Ubuntu desktop gets every prerequisite tool from code.

- [ ] Repo skeleton: create the directory tree from [docs/PLAN.md §2](docs/PLAN.md)
- [ ] `bootstrap/versions.yml` — pinned versions for every tool (single source of truth)
- [ ] `bootstrap/bootstrap.sh` — ~15 lines: install Python + Ansible via `pipx`
- [ ] `ansible/ansible.cfg` + `inventories/local/hosts.yml` (localhost)
- [ ] `roles/workstation_tools` — Terraform, Vagrant + vagrant-libvirt, qemu/libvirt stack, kubectl, Helm, az, k9s, jq, kustomize (versions from `versions.yml`)
- [ ] `playbooks/workstation.yml` + `Makefile` target `make workstation`
- [ ] `docs/00-workstation.md`
- **DoD:** `make workstation` twice → second run reports **0 changes**; `terraform version`, `vagrant version`, `virsh list`, `kubectl version --client`, `az version` all succeed; user is in the `libvirt` group.

## Phase 1 — Devstack VMs (Vagrant + libvirt)

**Goal:** three Ubuntu VMs with fixed IPs, reachable by SSH — no Kubernetes yet.

- [ ] `devstack/Vagrantfile` — 3 VMs from a node table, libvirt provider, static IPs (cp-1 `10.10.0.10`, worker-1 `.11`, worker-2 `.12`), box pinned in `versions.yml`
- [ ] `inventories/devstack/hosts.yml` — the three static IPs
- [ ] `Makefile` targets: `devstack-up`, `devstack-destroy`, `devstack-status`
- [ ] `docs/01-devstack-vms.md`
- **DoD:** `make devstack-up` → `vagrant status` shows 3 running; `ansible -i inventories/devstack all -m ping` succeeds; VMs ping each other; `destroy && up` rebuilds cleanly.

## Phase 2 — Bare VMs → Kubernetes cluster (Ansible + kubeadm)

**Goal:** `kubectl get nodes` shows cp-1, worker-1, worker-2 all `Ready`.

- [ ] `roles/k8s_common` — swap off, `overlay`+`br_netfilter`, sysctls, `/etc/hosts`
- [ ] `roles/containerd` — runtime install, `SystemdCgroup = true`
- [ ] `roles/kubeadm_install` — kubelet/kubeadm/kubectl at pinned minor, then `apt-mark hold`
- [ ] `roles/control_plane` — `kubeadm init` via `ClusterConfiguration` file, fetch admin.conf → merge as context `devstack`, generate join token
- [ ] `roles/worker` — `kubeadm join`, guarded so re-runs are no-ops
- [ ] `roles/cilium` — install CNI via Helm at pinned chart version
- [ ] `playbooks/cluster.yml` + `Makefile` target `devstack-cluster`
- [ ] `docs/02-kubernetes-kubeadm.md`
- **DoD:** 3 nodes `Ready`; `kubectl get pods -A` healthy; test nginx (2 replicas) lands across both workers and answers via NodePort; full `destroy → up → cluster` works end-to-end.

## Phase 3 — Azure foundation: AKS via Terraform

**Goal:** production AKS (managed control plane + 2 workers) built by Terraform, remote state.

- [ ] `terraform/bootstrap-state/` — run-once local-state stack: RG + storage account + `tfstate` container
- [ ] `terraform/modules/network` — VNet `10.20.0.0/16`, AKS subnet, NSG
- [ ] `terraform/modules/aks` — `azurerm_kubernetes_cluster`, system-assigned identity, 2-node default pool, Azure CNI powered by Cilium, pinned `kubernetes_version`, kubeconfig output
- [ ] `terraform/envs/prod/` — `backend "azurerm"`, composes both modules
- [ ] `Makefile` targets: `azure-plan`, `azure-apply`, `azure-destroy`
- [ ] `docs/03-azure-aks.md` — incl. cost stop/start runbook
- **DoD:** `terraform plan` clean after apply (no perma-diff); `az aks get-credentials` merges context `aks`; `kubectl --context aks get nodes` shows 2 `Ready`; state visible in the container; destroy + re-apply works.

## Phase 4 — Add-ons on both clusters via GitOps (Argo CD)

**Goal:** both clusters run cert-manager, Gateway API (Envoy Gateway), Prometheus/Grafana — installs happen only via Git commit.

- [ ] `playbooks/argocd-bootstrap.yml` — install Argo CD + apply each cluster's root Application (app-of-apps)
- [ ] `gitops/addons/cert-manager` (+ devstack self-signed / aks Let's Encrypt overlays)
- [ ] `gitops/addons/gateway` — Gateway API CRDs + Envoy Gateway; devstack overlay enables Cilium LB-IPAM + L2 announcements
- [ ] `gitops/addons/monitoring` — kube-prometheus-stack; overlays for sizing + storage class
- [ ] `gitops/bootstrap/` root Application + `gitops/clusters/{devstack,aks}/` overlays
- [ ] `docs/04-gitops-addons.md`
- **DoD:** Argo CD UI all apps `Synced/Healthy` on both clusters; demo app reachable through the Gateway on a devstack LAN IP and an Azure public IP with TLS; Grafana shows node metrics on both; a Git revert rolls a change back with no `kubectl`.

## Phase 5 — CI, operations & docs polish

**Goal:** the repo defends its own correctness; docs let a stranger operate it.

- [ ] `.github/workflows/ci.yml` — `terraform fmt -check` + `validate` + `tflint`, `ansible-lint`, `yamllint`, `kubeconform`
- [ ] Stretch: `terraform plan` job with Azure OIDC federated creds (no stored secrets)
- [ ] `docs/05-ci-and-operations.md` — runbooks: rebuild devstack from zero, AKS cost stop/start, version bump across both envs, TF state disaster recovery
- [ ] `README.md` — quickstart: zero → both clusters, in order
- **DoD:** CI green on main; an intentionally misformatted PR fails; every phase doc reviewed against what was actually built.

---

## Backlog / future (not yet scheduled)

- [ ] Logging: Prometheus + Loki + Grafana (moved here from `README.md`)
- [ ] Azure Application Gateway for Containers as an alternative Gateway impl (noted in Phase 3 docs)
- [ ] Flux as an alternative to Argo CD (trade-off noted in PLAN §3)
