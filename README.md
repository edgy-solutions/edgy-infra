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

## Verification status

**Verified 2026-08-08 against a real, mixed-version Proxmox cluster.**
Five LXC nodes spanning PVE **8.4.5** and **9.2.2** in a single cluster;
k3s v1.31.4, 5/5 Ready; traefik, local-path and metrics-server healthy.
Both playbooks re-run at `changed=0, failed=0` — idempotence demonstrated,
not asserted.

The scaffold was authored from module and API contracts rather than from a
successful run, and essentially every assumption in it was wrong in some
specific way. The findings below are recorded because each is cheap to hit
again and expensive to diagnose from its error message: in most cases the
message names something other than the actual cause.

### The nine first-run findings

| # | Symptom | Actual cause |
|---|---------|--------------|
| 1 | `couldn't resolve module/action community.general.proxmox` | The Proxmox modules **moved to `community.proxmox`**; `community.general` 13.x ships none of them. Installing `community.general` succeeds and the module is still gone. |
| 2 | `Failed to import the required Python library (proxmoxer)` | Ansible's `auto` interpreter discovery prefers the newest python on `PATH`, so modules ran under a uv-managed 3.12 while proxmoxer sat in the system 3.10. **The library was already installed.** Pin `ansible_python_interpreter`. |
| 3 | `CERTIFICATE_VERIFY_FAILED` | Proxmox ships a self-signed certificate. Now the explicit `proxmox_validate_certs` variable. |
| 4 | `500: Only root can pass arbitrary filesystem paths` | Not a privilege problem at all. Modern PVE **enforces `<STORAGE>:<SIZE>`**, so a bare size is parsed as a filesystem path. Use structured `disk_volume`. |
| 5 | `403: Permission check failed (/, Sys.Modify)` | Setting `nameserver` at create time demands `Sys.Modify` on `/`. Omit it — Proxmox then inherits the host's resolver, which is what you wanted. |
| 6 | `403: changing feature flags (except nesting) is only allowed for root@pam` | An API token may set **`nesting` and nothing else**, at any ACL level. See *the permission ceiling* below — this one has consequences. |
| 7 | `ENOENT` on an absolute path that demonstrably exists | `delegate_to` **inherits the play's `connection: local`**, so the task ran on the control node. Override `ansible_connection` on each delegated task. |
| 8 | `Path /etc/pve/lxc/<vmid>.conf does not exist` — for containers that do exist | `/etc/pve/lxc` is a **symlink to the local node's** directory. `/etc/pve` is replicated; that path is not. Use `/etc/pve/nodes/<node>/lxc`. |
| 9 | Raw `lxc.*` options duplicating on every run | `blockinfile` is wrong for `/etc/pve`. Proxmox re-emits the file and hoists `#` lines into the container *description*, detaching markers from the lines they bracket. Replaced with a deterministic rewrite that converges from an already-duplicated file. |

And one that appears only on the **second** run: `community.proxmox`'s
`update` default flipped to `true` in 1.0.0, and the update path is broken in
2.0.0 (`cmode` defaults to the sentinel `default`, which is not in the API
enum). Run 1 passes, run 2 fails — the worst shape for a bug, because it
looks like the first run broke something. `update: false` is what makes
re-runs safe, at the documented cost that inventory size changes are not
applied by re-running.

### The permission ceiling, and why it improved the result

Finding #6 is load-bearing. An API token cannot create a privileged
container and cannot set any feature flag except `nesting`; no ACL grant
lifts either, because it is a rule about *who you are*, not what you hold.
That forced, in order:

- **`keyctl` applied over root SSH** rather than through the token, then
- **unprivileged containers**, which in turn forced
- **`KubeletInUserNamespace`** — kubelet writes global kernel tunables at
  startup (`vm/overcommit_memory`, `kernel/panic`, `kernel/panic_on_oops`)
  which a user namespace cannot write, and without the gate it crash-loops
  in `activating` rather than failing outright.

This is a **better** outcome than the plan. Most k3s-in-LXC recipes reach for
privileged containers to dodge a long tail of cgroup issues; the permission
model refused, and the cluster runs at a stricter isolation level with the
same functionality. The root SSH channel ended up used for exactly two things
— `keyctl` and the raw `lxc.*` options — rather than becoming the general
path of least resistance.

**Unprivileged + nesting + keyctl + `KubeletInUserNamespace` is the proven
configuration.** Prefer it for an ARM edge rebuild as well: it is the one
with evidence behind it, not the privileged fallback everyone expects to
need. `node_type: vm` remains the escape hatch — but note that flipping
`lxc_unprivileged` back to `false` is *not* an escape, because that path is
closed to a token entirely.

### Mixed-version clusters

This cluster runs PVE 8.4.5 and 9.2.2 side by side, which Proxmox tolerates
during upgrades. Be aware that **the same restriction reports different
errors per version** — finding #6 surfaced as the explicit feature-flag
message on 8.4.5 and as a generic `Sys.Modify` denial on 9.2.2, which made
one problem look like two unrelated ones and cost a diagnostic cycle.

If a failure splits cleanly along node lines and makes no sense, check
`pvesh get /nodes/<node>/version` before anything else. Long-lived version
splits accumulate this confusion; resolve them when convenient.

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

## Prerequisites

**0. Point `KUBECONFIG` at this cluster, and verify it, before any `kubectl`
command.** This one is not about provisioning — it is about every session
afterwards, and it is listed first because getting it wrong is silent and
immediate.

```bash
export KUBECONFIG="$(git rev-parse --show-toplevel)/ansible/kubeconfig"
kubectl config current-context      # MUST print: edgy-lab
```

**Why this is a required action and not a note.** This cluster's kubeconfig
is **not** in `~/.kube/config`. It is written to `ansible/kubeconfig` and
lives only there. A shell that has not exported the variable does not fail —
it silently resolves to whatever `~/.kube/config` has current, and on the
machine this was built from, **that default is a long-lived production
cluster**, not the lab.

The failure mode is the dangerous shape: **a command aimed at the lab lands
on another cluster, succeeds, and returns plausible output.** Nothing
errors. The namespaces even have familiar names. The only tell is age — this
lab is days old, the default context's namespaces are years old — and nobody
checks age before running a read.

Same class as the ACL blind spots below: **a boundary that exists but does
not announce itself.** *What the token deliberately cannot see* records what
this tooling is prevented from reaching; this records what your shell will
reach for when you have not told it otherwise. Treat `current-context`
returning anything but `edgy-lab` as a stop.

*Corollary for anyone operating under a "this cluster only" instruction:*
verify the context **before** the first command, not after the first
surprising result. A read that hit the wrong cluster has already happened by
the time its output looks odd — and reads are not automatically harmless:
they appear in audit logs and are indistinguishable from a probe.

---

Five further things must exist before `provision.yml` will get anywhere.
Each was a real blocker on the first run; none is discoverable from the
error you get without it.

**1. A Proxmox API token, and the ACL grants it needs.** The token's rights
are the intersection of the *token's* ACL and the *user's* — granting only
the token changes nothing if the user lacks the role:

```bash
# what the playbooks actually need
pveum acl modify /pool/<pool>            --tokens 'user@pve!id' --roles PVEVMAdmin
pveum acl modify /storage/local          --tokens 'user@pve!id' --roles PVEDatastoreUser
pveum acl modify /storage/local-lvm      --tokens 'user@pve!id' --roles PVEDatastoreUser
pveum acl modify /sdn/zones/localnetwork/<bridge> --tokens 'user@pve!id' --roles PVESDNUser

# read-only visibility — grant to BOTH the token and the user, or it has no effect
pveum acl modify / --tokens 'user@pve!id' --roles PVEAuditor
pveum acl modify / --users  'user@pve'    --roles PVEAuditor
```

**2. The pool must already exist.** An ACL may name a pool that was never
created; `pool:` then fails at create time.

```bash
pvesh create /pools --poolid <pool>
```

**3. The LXC template, on *every* node that will host a container.** `local`
is node-local storage — a template on the node you run commands from proves
nothing about the others. `provision.yml` checks per node and prints the
exact fix, but seeding up front is faster:

```bash
for n in <node1> <node2> <node3>; do
  pvesh create /nodes/$n/aplinfo --storage local --template <template>.tar.zst
done
```

**4. Root SSH from the control node to one Proxmox node.** Two tasks cannot
go through the API at all — `keyctl` (root@pam only) and the raw `lxc.*`
options (not exposed by the API). Both use `pvesh` or `/etc/pve/nodes/...`,
which reach the whole cluster from a single node, so one host is enough.

```bash
ssh-keygen -t ed25519 -C edgy-infra-lab      # on the control node
# then, on the Proxmox host:
echo '<pubkey>' >> /root/.ssh/authorized_keys
```

**5. Static IPs from OUTSIDE the DHCP pool.** These nodes take static
addresses. *"It did not answer a ping"* is **not** evidence an address is
free — it only means nothing holds a lease right now, and the server can
hand out that same address tomorrow. Read the pool bounds off the DHCP
server (OPNsense/pfSense: *Services → DHCPv4 → \[interface\] → Range*) and
choose from outside it, or reserve static mappings. Cross-check `arp -a`.

---

## What the token deliberately cannot see

The least-privilege token has blind spots. They are the design working, but
they will mislead you if you use the API to *verify* rather than to act:

- **`GET /pools` returns `{"data":[]}`** without `Pool.Audit` — an empty
  list, not a 403. A pool that exists reads as absent. `GET /pools/<id>`
  *does* say `Permission check failed (/pool/<id>, Pool.Audit)`, so query
  the specific pool when you need a real answer.
- **A child ACL replaces the inherited one; it does not union with it.**
  `PVEAuditor` on `/` never reaches `/pool/<pool>` once that path has its own
  ACL — which is exactly why `Sys.Audit` worked everywhere while `Pool.Audit`
  was denied on the pool alone.
- **Node capacity reads as `0`** without `Sys.Audit`: `maxcpu` and `maxmem`
  come back zero rather than erroring, so a 32-core node looks empty.
- **`GET /cluster/resources?type=vm` returns an empty list** without
  `VM.Audit`, so an occupied cluster looks idle.

The pattern is worth internalising: **Proxmox's list endpoints filter to what
you may audit and return success.** Absence in a list is not evidence of
absence in the cluster. When a pre-flight check reports "nothing there",
confirm it against a single-object endpoint that can actually deny you.

---

## Usage

```bash
pip install --user ansible-core proxmoxer requests
# NOTE: community.proxmox, not community.general — the Proxmox modules moved,
# and community.general 13.x ships none of them.
cd ansible
ansible-galaxy collection install -r requirements.yml

cp inventory.example.yml inventory.yml
cp group_vars/all.example.yml group_vars/all.yml
# edit both

export PROXMOX_TOKEN='<token secret>'    # never written to a file

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
