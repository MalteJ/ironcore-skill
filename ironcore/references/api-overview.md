# IronCore API – Object Overview

All groups live under `*.ironcore.dev/v1alpha1` (aggregated API server, no CRDs).
Verified against the sources in github.com/ironcore-dev/ironcore (`api/*`).

## Objects by group

### compute.ironcore.dev

| Kind | Scope | Core spec fields | Status |
|---|---|---|---|
| Machine | ns | `machineClassRef` (required), `machinePoolSelector`/`machinePoolRef`, `power` (On/Off), `networkInterfaces[]`, `volumes[]`, `ignitionRef` (SecretKeySelector), `guestConfig.hostname`, `tolerations[]` | `machineID`, `state` (Pending/Running/Shutdown/Terminating/Terminated), `networkInterfaces[]`/`volumes[]` (each Pending→Attached), `conditions` (Ready, VolumesReady, NetworkInterfacesReady) |
| MachineClass | cluster | `capabilities` (cpu, memory) | – |
| MachinePool | cluster | `providerID`, `taints[]` | `state` (Pending/Ready/Error/Offline), `availableMachineClasses[]`, `capacity`/`allocatable` |

Machine source unions:
- NIC entry: `networkInterfaceRef` (existing NIC) OR `ephemeral.networkInterfaceTemplate`
- Volume entry: `volumeRef` OR `localDisk{image,sizeLimit}` OR `ephemeral.volumeTemplate` (`emptyDisk` is deprecated)

### storage.ironcore.dev

| Kind | Scope | Core fields | Status |
|---|---|---|---|
| Volume | ns | `volumeClassRef` (effectively required!), `volumePoolSelector`, `resources.storage`, `encryption.secretRef`, `dataSource` (`volumeSnapshotRef` \| `osImage{image,architecture}`), `unclaimable`, `claimRef` (set by the attach controller) | `state` (Pending/Available/Error), `access{secretRef,driver,handle,attributes}` (only once Available) |
| VolumeSnapshot | ns | `volumeRef` | `state` (Pending/Ready/Failed), `snapshotID` |
| VolumePool / BucketPool | cluster | `providerID`, `taints[]` | state, `available*Classes[]`, capacity/allocatable |
| VolumeClass / BucketClass | cluster | `capabilities` (iops/tps), `resizePolicy` (Static/ExpandOnly) | – |
| Bucket | ns | `bucketClassRef` (required), `bucketPoolSelector` | `state`, `access{secretRef,endpoint}` (S3) |

Storage secret types: `storage.ironcore.dev/volume-encryption` (user-provided: key `encryptionKey`,
base64 256-bit key), `storage.ironcore.dev/volume-auth` (provider-generated, referenced in status).

### networking.ironcore.dev

| Kind | Core fields | Status |
|---|---|---|
| Network | `peerings[]` (name, networkRef{name,namespace}, prefixes); `incomingPeerings` managed by controller | `state` (Pending/Available/Error), `peerings[].state` (Pending/Ready/Error) |
| NetworkInterface | `networkRef` (required), `machineRef` (UID), `ipFamilies[]`, `ips[]` (`value` \| `ephemeral.prefixTemplate`), `prefixes[]`, `virtualIP` (ref \| ephemeral) | `state`, effective `ips[]`, `prefixes[]`, `virtualIP` |
| VirtualIP | `type` (Public only), `ipFamily`, `targetRef` (UID → NIC) | `ip` |
| LoadBalancer | `type` (Public/Internal), `networkRef`, `networkInterfaceSelector` (labels!), `ports[]`, `ips[]` only for Internal | `ips[]` |
| NATGateway | `type` (Public only), `networkRef`, `portsPerNetworkInterface` (power of 2, default 2048) | `ips[]` |
| NetworkPolicy | `networkRef`, `networkInterfaceSelector`, `ingress[]`/`egress[]` (peers: objectSelector \| ipBlock, ports+endPort) | `conditions` |
| LoadBalancerRouting | **controller-generated, never create manually** | – |

NICs need labels if they should be LoadBalancer targets (e.g. `app: web`).
The implementation lives in **ironcore-net** (apinetlet/metalnetlet/dpservice) – transparent to users.

### ipam.ironcore.dev

| Kind | Core fields | Status |
|---|---|---|
| Prefix | `ipFamily`, `prefix` (CIDR) \| `prefixLength`, `parentRef` \| `parentSelector` (empty = root prefix) | `phase` (Pending/Allocated), `used[]` |
| PrefixAllocation | internal / created by the PrefixController – users create Prefixes, not Allocations | `phase` (Pending/Allocated/Failed, terminal), `prefix` |

An IP is a prefix of length /32 (v4) or /128 (v6). NICs/LoadBalancers obtain IPs statically
(`value`) or ephemerally via `prefixTemplate` (parent chosen via `parentRef`/`parentSelector`).

### core.ironcore.dev

- **ResourceQuota** (ns): `hard` ResourceList (`cpu`, `memory`, `storage`, `tps`, `iops`, `requests.*`,
  `count/<res>.<group>`, `class/machine.<name>`), optional `scopeSelector` (MachineClass/VolumeClass/BucketClass).
  Enforced admission-side (HTTP 403 on exceed); observe `status.used`.

## Object graph (compact)

```
Machine → MachineClass, MachinePool, Secret(ignition), NetworkInterface[], Volume[]
NetworkInterface → Network, Machine, VirtualIP, Prefix(ephemeral)
VirtualIP → NIC (targetRef)     LoadBalancer/NATGateway/NetworkPolicy → Network (+ label-select NICs)
Volume → VolumeClass, VolumePool, VolumeSnapshot(dataSource), Machine(claimRef), Secret(encryption)
Prefix → Prefix (parent)        Bucket → BucketClass, BucketPool
Network ↔ Network (peering, bidirectional, the only cross-namespace reference)
```

## Lifecycle conventions

Workload: `Pending → Running` (Machine) or `Pending → Available` (Volume/Bucket/Network/NIC).
Pools: `Pending → Ready`. Prefix: `Pending → Allocated`. Snapshot: `Pending → Ready`.
Attachments inside a Machine: `Pending → Attached`. Transition times in `lastStateTransitionTime`.
