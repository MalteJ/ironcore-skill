# IronCore Troubleshooting

Things get stuck in Pending all the time. This is the diagnosis map –
work symptom → cause → check. Primary tools: `-o wide`, `kubectl describe` (conditions + events),
and the status fields listed in api-overview.md.

## Machine stuck in Pending

Check in this order:

```bash
kubectl describe machine <name>     # conditions + events are the fastest signal
kubectl get machineclass            # does spec.machineClassRef.name exist?
kubectl get machinepools -o wide    # is any pool Ready and matches selector/taints?
```

| Symptom (event/condition) | Cause | Fix |
|---|---|---|
| no `machinePoolRef` set | scheduler found no pool | check `machinePoolSelector` labels vs pool labels; check pool `status.state: Ready`; do the Machine's `tolerations` cover pool `taints`? |
| `IgnitionNotReady` | secret missing/wrong ns/wrong key/not rendered | secret must be in the Machine's namespace, key `ignition.yaml`, populated with rendered Ignition JSON (see images-ignition.md) |
| `networkInterfaces[].state: Pending` | referenced NIC missing or not Available | check that NIC's status (below) |
| `volumes[].state: Pending` | referenced Volume missing/not Available, or image pull failing | check Volume status; for `localDisk.image`/`osImage`: registry reachable **from the pool provider**? image actually Ignition-capable? |
| class exists but machine never Running | provider error | pool's poollet/provider logs; `kubectl describe machinepool <pool>` |

## Machine Running but nothing works

- No SSH: the image doesn't evaluate Ignition, or keys landed under the wrong user/path.
  Ignition is first-boot only – you must recreate the Machine after fixing the secret.
- Attachment state shows Pending but object looks fine: the referenced object's UID may have
  changed after recreation (NICs/Volumes use UID-based references via `machineRef`/`claimRef`).

## Volume stuck in Pending

- `volumeClassRef` missing/typo → without a class, no in-tree provisioner acts (it expects an external controller).
- No `volumePool` advertises the class → compare `kubectl get volumepools -o yaml` `status.availableVolumeClasses`.
- Pool taints not tolerated by `spec.tolerations`.
- `dataSource`: snapshot not Ready (`state: Ready` required) or osImage unreachable from the pool.
- Encryption: Secret must exist with key `encryptionKey`, base64-encoded 256-bit key,
  type `storage.ironcore.dev/volume-encryption`.
- Resize rejected: only growth and only when the class has `resizePolicy: ExpandOnly`.

## NetworkInterface stuck in Pending

- `networkRef` Network not `Available` yet – wait/recreate.
- Ephemeral IP never allocated: the referenced `Prefix` (via `parentRef`/`parentSelector` in the
  prefixTemplate) doesn't exist, isn't Allocated, or is exhausted → check parent Prefix `status.phase` and `status.used`.
- `ips[].value` static IP already in use inside the network.

## VirtualIP / LoadBalancer / NATGateway with empty status

- VirtualIP without `status.ip`: target NIC not Available; `targetRef` is UID-based – recreating the NIC orphans the VIP (recreate the VIP).
- LoadBalancer without `status.ips`: Network not Available; Internal type requires `spec.ips`;
  **targets missing → check labels**: `networkInterfaceSelector` must match NIC labels in the same network.
- NATGateway `portsPerNetworkInterface` must be a power of 2.

## NetworkPolicy matches nothing

- Selector only matches NICs **in the same network** (network-scoped, not namespace/global).
- Peers via `objectSelector` need `kind: NetworkInterface|LoadBalancer` + correct labels.
- Ports: without explicit ports the rule does not meaningfully restrict/match – always specify ports.

## Network peering stays Pending

Peering must be defined mutually on **both** networks (matching `peerings[]` entries,
optionally cross-namespace via `networkRef.namespace`). Watch `status.peerings[].state` → Ready.

## Bucket stuck in Pending

- `bucketClassRef` is required and must exist and be advertised by a BucketPool.
- `status.access.endpoint` + credentials secret appear only at `Available`.

## Deletion hangs

- Finalizers are held while the poollet/provider tears down the backend resource.
  If a provider/poollet is down, objects stay in deletion – check pool state first.
- MachineClass/VolumeClass deletion is blocked while in use (finalizers on classes).
- Referenced (non-ephemeral) NICs/Volumes **survive** `kubectl delete machine` – clean up separately.

## Getting onto the machine

```bash
# remote exec via the apiserver subresource (apiserver → machinepoollet → IRI Exec):
kubectl get --raw /apis/compute.ironcore.dev/v1alpha1/namespaces/<ns>/machines/<name>/exec
# returns a tokenized URL for an interactive session (provider-dependent)
# usual path: SSH with the keys injected via Ignition onto the VirtualIP/public address
```
