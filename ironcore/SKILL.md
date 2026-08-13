---
name: ironcore
description: Operate the IronCore IaaS platform – create and manage VMs (Machines), Volumes, Networks, IPs, LoadBalancers, NATGateways and Buckets via kubectl. Use for requests like "create a VM", "set up a Debian machine", "create a volume", "configure network/firewall/IP", OS images or Ignition. IMPORTANT: NEVER apply infrastructure changes directly – always present a detailed proposal with all open decisions first and wait for user approval.
---

# IronCore IaaS

IronCore is an IaaS API (aggregated Kubernetes API server, groups `*.ironcore.dev/v1alpha1`).
Objects are declared via kubectl; schedulers, poollets and providers materialize them.

## Golden rule: Proposal first, execution second

Users often send simple requests ("Create me a Debian VM", "I need 500 GiB of storage").
These requests are **always incomplete**. NEVER run `kubectl apply` directly. Instead:

### Step 1: Explore the environment (read-only, no confirmation needed)

ALWAYS make sure kubectl talks to the IronCore API server before anything else.
Use the kubeconfig the project provides (usually a kubeconfig.yaml in the project root):

```bash
export KUBECONFIG=<project>/kubeconfig.yaml
# sanity check – the IronCore API groups must be visible:
kubectl api-resources | grep ironcore.dev

# Discover what's available – this grounds the proposal:
kubectl get machineclasses                    # available flavors (CPU/RAM)
kubectl get machinepools -o wide              # pools, state, taints
kubectl get volumeclasses volumepools -o wide # storage classes/pools
kubectl get networks                          # existing networks
kubectl get machines volumes networkinterfaces # current namespace inventory
```

### Step 2: Present a proposal

Show the user a concrete configuration proposal. List every decision with a sensible
default and alternatives. Decision catalog, depending on the request:

| Decision | Options / questions for the user |
|---|---|
| Name + namespace | Default: the project's configured/working namespace |
| MachineClass (instance type) | pick from `kubectl get machineclasses`, state CPU+RAM explicitly |
| OS image | which image (e.g. Debian 13)? source/registry ref? is it Ignition-capable? |
| Boot disk | `localDisk.image` (simple, pool-local) **or** persistent `Volume` with `dataSource.osImage` (survives the VM; choose class+size) |
| Network | existing `Network` or create a new one? |
| IP addresses | static (`ips[].value`) or IPAM (`ephemeral.prefixTemplate` + Prefix)? IPv4/IPv6? |
| Public IP? | `VirtualIP` (1:1 host IP) yes/no |
| Outbound internet? | `NATGateway` needed? |
| LoadBalancer? | port(s), Public/Internal, target NICs via label (e.g. `app: web`) |
| Firewall | `NetworkPolicy`: which ports, from where (e.g. SSH/22 restricted to specific CIDRs)? |
| Ignition/SSH | SSH keys (from `image/authorized_keys`), user, hostname, extra config? |
| Additional storage | data volumes (size, VolumeClass), encryption, snapshots? |
| Power | start `On` immediately? |

Proposal format: short table of assumptions + explicitly flag open questions + list the
resulting objects (e.g. "Secret ignition, Network, NetworkInterface, VirtualIP, NetworkPolicy, Machine").
Then: **wait for approval or adjustments.** Only then step 3.

### Step 3: Execution after approval

Write manifests into the project's manifest directory (if it has one, e.g. `manifests/<name>/`,
keep any existing convention like numbered files in apply order), then apply in this order,
waiting for each ready state:

1. Ignition Secret
2. Network → wait for `status.state: Available`
3. (optional) Prefix (IPAM) → `phase: Allocated`
4. NetworkInterface → `state: Available` (verify effective IPs in status)
5. (optional) VirtualIP / NATGateway / LoadBalancer / NetworkPolicy
6. (optional persistent) Volume → `state: Available`
7. Machine → `status.state: Running`, watch attachments go `Pending → Attached`

```bash
kubectl get machine <name> -w
kubectl describe machine <name>   # conditions + events (e.g. IgnitionNotReady)
```

YAML skeletons and details: [references/workflows.md](references/workflows.md)
Object model, fields, status phases: [references/api-overview.md](references/api-overview.md)
Images & Ignition (pitfalls!): [references/images-ignition.md](references/images-ignition.md)

## Hard-learned rules (always apply)

- **No cloud-init** – Ignition only. The Secret key `ignition.yaml` contains **rendered Ignition JSON**
  (transpiled via Butane; never hand-write it). The OS image itself must evaluate Ignition.
- Ignition Secret lives in the same namespace as the Machine; default key `ignition.yaml`.
- NEVER set `*PoolRef` yourself – always use `*PoolSelector` (the scheduler sets the ref).
- First volume in `spec.volumes` = root disk (`device: vda`). Use `localDisk`, not the deprecated `emptyDisk`.
- Always set `volumeClassRef` on Volumes (otherwise they stay Pending without an external provisioner).
- NEVER create `LoadBalancerRouting` – it is generated by the load balancer controller.
- Network peering must be defined **mutually** on both Networks.
- Deletion order: referenced (non-ephemeral) NICs/Volumes survive the Machine and must be deleted separately.
