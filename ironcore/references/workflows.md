# IronCore Workflows & YAML Skeletons

Apply order matters: dependencies first, wait for each ready state.
Proven local examples live in `manifests/` and `manifests/variants/root-volume/`;
official scenarios in the repo at `ironcore-api/config/samples/e2e/` (kustomize-based).

## Workflow 1: VM with persistent root volume (recommended)

Blueprints: `manifests/variants/root-volume/00-volume.yaml` + `03-machine.yaml`.

```yaml
# 00-volume.yaml – boot disk, populated from OS image
apiVersion: storage.ironcore.dev/v1alpha1
kind: Volume
metadata: {name: <vm>-root-volume}
spec:
  volumeClassRef: {name: <volumeclass>}      # from: kubectl get volumeclasses
  resources: {storage: 20Gi}
  dataSource:
    osImage: {image: <registry-ref>, architecture: amd64}
---
# 03-machine.yaml
apiVersion: compute.ironcore.dev/v1alpha1
kind: Machine
metadata: {name: <vm>}
spec:
  machineClassRef: {name: <machineclass>}    # from: kubectl get machineclasses
  ignitionRef: {name: <ignition-secret>, key: ignition.yaml}
  networkInterfaces:
  - name: primary
    networkInterfaceRef: {name: <nic>}       # or ephemeral (workflow 3)
  volumes:
  - name: root-disk                          # first volume = root disk
    device: vda
    volumeRef: {name: <vm>-root-volume}
```

## Workflow 2: VM with ephemeral root disk (localDisk)

Simplest path, no separate Volume object. Blueprint: `manifests/03-machine.yaml`.

```yaml
spec:
  volumes:
  - name: root-disk
    localDisk:
      image: <registry-ref>                  # pulled directly by the pool provider
```

## Workflow 3: Fully ephemeral (VM + NIC + VIP in one manifest)

Blueprint: `ironcore-api/config/samples/e2e/bases/machine-with-ephemeral-resources/machine.yaml`.

```yaml
spec:
  networkInterfaces:
  - name: primary
    ephemeral:
      networkInterfaceTemplate:
        spec:
          networkRef: {name: <network>}
          ipFamilies: [IPv4]
          ips: [{value: 10.0.0.12}]          # static; OR ephemeral:
          # ips: [{ephemeral: {prefixTemplate: {spec: {ipFamily: IPv4, prefixLength: 32, parentRef: {name: <prefix>}}}}}]
          virtualIP:
            ephemeral: {virtualIPTemplate: {spec: {type: Public, ipFamily: IPv4}}}
  volumes:
  - name: root-disk
    ephemeral:
      volumeTemplate:
        spec:
          volumeClassRef: {name: <class>}
          resources: {storage: 10Gi}
          dataSource: {osImage: {image: <registry-ref>}}
```

Ephemeral objects live and die with the Machine (owner coupling, annotation
`common.ironcore.dev/ephemeral-managed-by`).

## Workflow 4: Network foundation (before the VMs)

```yaml
# Network
apiVersion: networking.ironcore.dev/v1alpha1
kind: Network
metadata: {name: <network>}
---
# NetworkInterface (static IP)
apiVersion: networking.ironcore.dev/v1alpha1
kind: NetworkInterface
metadata:
  name: <vm>-nic
  labels: {app: web}                          # labels = LB target groups!
spec:
  networkRef: {name: <network>}
  ipFamilies: [IPv4]
  ips: [{value: 10.0.0.12}]
  virtualIP:                                  # optional: public IP
    virtualIPRef: {name: <vip>}               # or an ephemeral template
```

Wait: `kubectl wait --for=jsonpath='{.status.state}'=Available network/<network> --timeout=60s`

## Workflow 5: IPAM (prefix hierarchy for dynamic IPs)

```yaml
apiVersion: ipam.ironcore.dev/v1alpha1
kind: Prefix
metadata:
  name: root
  labels: {root-prefix: customer-1}
spec: {prefix: 10.0.0.0/24}
---
apiVersion: ipam.ironcore.dev/v1alpha1
kind: Prefix
metadata: {name: child-01}
spec:
  ipFamily: IPv4
  prefixLength: 28
  parentSelector: {matchLabels: {root-prefix: customer-1}}   # or parentRef: {name: root}
```
Await `status.phase: Allocated`; the parent's `status.used` shows allocations (incl. /32 NIC IPs).

## Workflow 6: External connectivity

```yaml
# Public IP 1:1 onto one interface
apiVersion: networking.ironcore.dev/v1alpha1
kind: VirtualIP
spec: {type: Public, ipFamily: IPv4, targetRef: {name: <nic>}}   # targetRef may be set by the controller
---
# Outbound internet for all NICs of a network
apiVersion: networking.ironcore.dev/v1alpha1
kind: NATGateway
spec: {type: Public, ipFamily: IPv4, networkRef: {name: <network>}}
---
# LoadBalancer (fans ports out to NICs labeled app=web)
apiVersion: networking.ironcore.dev/v1alpha1
kind: LoadBalancer
spec:
  type: Public                                # Internal allows spec.ips
  ipFamilies: [IPv4]
  networkRef: {name: <network>}
  networkInterfaceSelector: {matchLabels: {app: web}}
  ports: [{port: 80}]
```

## Workflow 7: Firewall (NetworkPolicy)

```yaml
apiVersion: networking.ironcore.dev/v1alpha1
kind: NetworkPolicy
spec:
  networkRef: {name: <network>}
  networkInterfaceSelector: {matchLabels: {app: web}}
  policyTypes: [Ingress]
  ingress:
  - from: [{ipBlock: {cidr: 0.0.0.0/0}}]      # or objectSelector (kind: NetworkInterface/LoadBalancer)
    ports: [{protocol: TCP, port: 22}]
```
Note: network-scoped – the selector only matches NICs in the same network.

## Workflow 8: Network peering (bidirectional!)

Both networks must reference each other (cross-namespace via networkRef.namespace is possible),
otherwise `status.peerings[].state` stays Pending:

```yaml
# network1: spec.peerings: [{name: to-net2, networkRef: {name: network2}}]
# network2: spec.peerings: [{name: to-net1, networkRef: {name: network1}}]
```

## Operations

```bash
kubectl patch machine <vm> -p '{"spec":{"power":"Off"}}' --type=merge   # power off
kubectl get machine <vm> -w                                              # watch state
kubectl describe machine <vm>                                            # conditions/events
kubectl delete machine <vm>                                              # ephemerals are cleaned up
```
