# edgy-infra

Shared cluster provisioning and deploy harness. Stands up Kubernetes on
Proxmox and deploys workloads to it.

**Convergence intent, stated so the scope is citable:** this exists
because "provision a k3s cluster on Proxmox and deploy to it" serves
**more than one project** — OpenDDIL and Invincible Agent today, others
later. Homing it inside either project's repository would misdeclare its
scope. The repo name is a claim about what this is for, and it is
deliberately project-neutral.

---

## The layer boundary — read before adding anything

This repo owns the **substrate**. It stops where a cluster begins.

**In scope:**
- Proxmox provisioning (LXC or VM)
- k3s cluster formation
- Rancher downstream import
- The helmfile **harness** that invokes deploys

**Out of scope — permanently:**
- Any project's Helm charts
- Any project's values or topology
- Anything tied to a project's release cycle

**The test:** *if changing it requires knowing which project you are
deploying, it does not belong here.*

Per-project release definitions and values live **in their own
repositories** and are consumed here **by reference** (path or git
source), never by copy. Without that line this repo becomes the place
where each project's deployment truth quietly forks from the project
itself — a two-truths incubator with an innocent name.

---

## ⚠ Verification status

**Not yet run against a real Proxmox host.** Authored from module and API
contracts, not from a successful run. Treat the first execution as a
debugging session.

Most likely to need adjustment:

1. **LXC + k3s tuning** (`roles/k3s_node_prep`) — nesting / keyctl /
   AppArmor / `/dev/kmsg` / snapshotter. Well-documented but
   Proxmox-version sensitive. This is exactly why `node_type: vm` exists
   as a one-variable escape.
2. **Storage and bridge names** — `local-lvm` / `vmbr0` are defaults.
3. **Template presence** — the playbook checks and prints the download
   command rather than guessing.

All playbooks are idempotent and safe to re-run.

---

## Secrets — the rule, from the first commit

**Nothing sensitive enters git. Ever.**

Proxmox API tokens, SSH keys, node addresses, Rancher registration
tokens: all externalized. Every secret-bearing file has a committed
`.example` twin; the real one is gitignored.

```
inventory.example.yml       committed
inventory.yml               GITIGNORED — your node addresses
group_vars/all.example.yml  committed
group_vars/all.yml          GITIGNORED — your tokens
kubeconfig                  GITIGNORED
```

Use `ansible-vault` or environment injection for tokens. This repo has
the highest secret-adjacency of anything in the portfolio — it touches
hypervisor credentials — so the sanitization sweep applies here as a
standing rule even while the repo is private.

---

## Architecture is a first-class variable

`arch: amd64` today. `arch: arm64` is a supported value, not a future
port — image sources, the k3s binary, and per-arch sysctls all key off
it. The lab cluster is amd64; an ARM edge-cluster rebuild uses the same
playbooks with one variable changed. Retrofitting arch-awareness later
is the rework this avoids.

---

## Cluster shape

Mirrors the existing edge cluster's pattern: **1 k3s server + 4 agents**
across 4 live Proxmox nodes, with the server **colocated alongside one
agent** on the same physical node.

```
pve-1  →  k3s-server-1  +  k3s-agent-1     (colocated)
pve-2  →  k3s-agent-2
pve-3  →  k3s-agent-3
pve-4  →  k3s-agent-4
```

Single server, not HA embedded etcd — deliberately. With one server, the
fifth Proxmox node returning from repair is simply `+1 agent`; with a
3-node etcd quorum its return changes the quorum math. This is a test
cluster with a playbook that makes rebuild cheap, so an HA control plane
buys overhead and nothing else.

---

## Rancher: import, never join

There is an existing Rancher management ("local") cluster on this
Proxmox. **These playbooks never touch it.**

They stand up a *fresh* k3s cluster and then **import it into Rancher as
a downstream managed cluster**. Running workloads on Rancher's local
cluster is a named anti-pattern — it is the management plane, and this
stack (multiple brokers, multiple Restate instances, multiple Postgres)
is exactly the tenant that degrades it.

Two bonuses: blast radius on existing infrastructure is zero, and it
mirrors how Rancher is typically run in production (managing downstream
clusters), so the lab rehearses the real management topology too.

---

## Usage

```bash
pip install ansible proxmoxer requests
ansible-galaxy collection install community.general kubernetes.core

cd ansible
cp inventory.example.yml inventory.yml
cp group_vars/all.example.yml group_vars/all.yml
# edit both

ansible-playbook -i inventory.yml provision.yml   # containers/VMs
ansible-playbook -i inventory.yml k3s.yml         # k3s + kubeconfig
ansible-playbook -i inventory.yml rancher-import.yml   # optional

export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes        # expect 5 Ready
```

Deploy a project:

```bash
cd ../helmfile
helmfile -e lab apply
```

---

## Switching to VMs

If LXC fights:

```yaml
# group_vars/all.yml
node_type: vm
```

Re-run `provision.yml`. The k3s role skips all container-specific tuning
automatically. Taking this escape is not a defeat — on a cluster whose
purpose is validating *someone else's chart*, runtime debugging is
contamination. Two hours into cgroup behaviour, switch.

---

## Discipline

The verification rules from the OpenDDIL contracts' `PRINCIPLES.md` apply
here with extra force, because Ansible's characteristic failure is
**exit-zero-but-wrong**:

- **Believed vs. confirmed** — a playbook that ran is not a cluster that
  works. Check the end state, not the return code.
- **Verify the rendered artifact** — `helmfile template` before
  `helmfile apply`; diff rendered object names against what you expect.
- **Verify before multiplying** — one node before four.
