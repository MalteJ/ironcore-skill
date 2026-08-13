# Worked Example: "Create me a Debian VM"

The complete interaction the skill expects, end to end.

## 1. User prompt

> "Erstell mir eine Debian VM"

## 2. Agent explores (read-only, no questions yet)

```bash
export KUBECONFIG=<project>/kubeconfig.yaml
kubectl api-resources | grep ironcore.dev      # confirms IronCore API is reachable
kubectl get machineclasses                     # e.g. memory-intel1-2cpu (2 CPU / 1 GiB picos)
kubectl get machinepools -o wide               # e.g. "shared" pool, state Ready
kubectl get volumeclasses volumepools
kubectl get networks                           # none yet
kubectl get machines -A                        # existing machines, naming patterns
```

## 3. Agent presents the proposal (STOPS here)

```markdown
**Proposal: Debian VM "debian-01"**

| Decision | My default | Open? |
|---|---|---|
| Namespace | <current> | |
| MachineClass | memory-intel1-2cpu (2 vCPU, 1 GiB) | only class available – OK for a test VM? |
| OS image | <registry>/debian:13 (Ignition-capable) | confirm image ref |
| Boot disk | persistent Volume 10 Gi, class <default-class> | alternative: localDisk (simpler, data lost with VM) |
| Network | new Network "debian-01-net" | or attach to an existing one? |
| IP | static 10.0.0.12/32 | or IPAM via Prefix? |
| Public IP | VirtualIP (public) | needed for SSH from outside, yes/no? |
| Outbound | NATGateway | needed for apt updates |
| Firewall | NetworkPolicy: TCP/22 + TCP/80 from 0.0.0.0/0 | restrict SSH to a CIDR? |
| Ignition | user "core" + your SSH key(s) from image/authorized_keys | additional users/config? |
| Power | On | |

**Objects to create (in order):**
Secret (ignition) → Network → NetworkInterface → VirtualIP → NATGateway →
NetworkPolicy → Volume (root) → Machine

OK to proceed, or what should I adjust?
```

## 4. User confirms (possibly with adjustments), agent executes

```bash
mkdir -p manifests/debian-01
# write files: 00-ignition-secret.yaml, 01-network.yaml, 02-network-interface.yaml,
#              03-virtualip.yaml, 04-natgateway.yaml, 05-network-policy.yaml,
#              06-volume.yaml, 07-machine.yaml
kubectl apply -f manifests/debian-01/00-ignition-secret.yaml
kubectl apply -f manifests/debian-01/01-network.yaml
kubectl wait --for=jsonpath='{.status.state}'=Available network/debian-01-net --timeout=60s
kubectl apply -f manifests/debian-01/02-network-interface.yaml
kubectl wait --for=jsonpath='{.status.state}'=Available networkinterface/debian-01-nic --timeout=120s
kubectl get networkinterface debian-01-nic -o jsonpath='{.status.ips}'   # verify effective IP
kubectl apply -f manifests/debian-01/03-virtualip.yaml manifests/debian-01/04-natgateway.yaml manifests/debian-01/05-network-policy.yaml
kubectl apply -f manifests/debian-01/06-volume.yaml
kubectl wait --for=jsonpath='{.status.state}'=Available volume/debian-01-root --timeout=180s
kubectl apply -f manifests/debian-01/07-machine.yaml
kubectl get machine debian-01 -w     # until state: Running, attachments Attached
```

## 5. Agent reports back

- `status.machineID`, effective IPs (`networkinterface.status.ips`, `virtualip.status.ip`)
- how to connect (SSH user + which IP)
- where the manifests live for future edits
- reminder: referenced NIC/Volume survive `kubectl delete machine`

## Anti-patterns (never do this)

- ❌ `kubectl apply` anything before the user approves the proposal table
- ❌ asking questions one by one instead of one structured proposal
- ❌ inventing a MachineClass/image instead of listing what exists
- ❌ skipping the readiness waits between apply steps
