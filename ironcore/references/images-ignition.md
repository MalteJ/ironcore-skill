# IronCore OS Images & Ignition

The two most common failure modes when creating Machines are a non-Ignition-capable image
and a malformed ignition secret. Read this before creating any VM.

## Images: OCI references, no CRD

- There is **no Image CRD**. An image is just a string: `name:tag` or `name@sha256:<digest>`,
  used in `Machine.spec.volumes[].localDisk.image` and `Volume.spec.dataSource.osImage.image`.
- Images follow the **IronCore OCI Image spec** (github.com/ironcore-dev/ironcore-image, OCI-SPEC.md):
  an index manifest, per-architecture artifact manifests with layers
  `kernel`, `initramfs`, `rootfs` (+ optional `squashfs`, `uki`, `iso`, `disk`),
  and a config containing the kernel `commandLine`.
- `dataSource.osImage.architecture` selects the variant from the multi-arch index (e.g. `amd64`).
- **The pool provider pulls the image**, not the API server → the registry must be reachable
  from the pool. Registry auth is provider-specific.
- Images are evaluated **at creation/bootstrap time only**. Changing a tag later does not affect
  existing disks/VMs.

Building custom images (tooling from the ironcore-image repo):

```bash
ironcore-image build --tag my-image:latest \
  --config arch=amd64,rootfs=./rootfs.ext4,initramfs=./initramfs.img,kernel=./vmlinuz,cmdline="console=ttyS0"
ironcore-image tag my-image:latest ghcr.io/<org>/my-image:latest
ironcore-image push ghcr.io/<org>/my-image:latest
```

## Ignition instead of cloud-init

- **IronCore does not support cloud-init.** Machines accept Ignition data only, and the OS image
  itself must evaluate it (e.g. from the initramfs). Plain cloud images will boot without SSH keys
  or any configuration.
- Flow: user provides a Secret (type `compute.ironcore.dev/ignition`, default key `ignition.yaml`)
  in the Machine's namespace → `Machine.spec.ignitionRef` (SecretKeySelector) → the poollet reads it
  and passes it via gRPC (IRI) to the provider (libvirt: firmware channel `opt/com.coreos/config`).
- **The key contains rendered Ignition JSON, despite the `.yaml` name.** Never hand-write the JSON.
  Author in **Butane** and transpile.
- Ignition is applied on first boot only; changing the Secret later has no effect on running/old instances.

### Canonical pipeline (from the official e2e samples)

```yaml
# ignition.yaml – Butane source (this YAML is NOT what goes into the Secret)
variant: fcos
version: 1.4.0
passwd:
  users:
    - name: core
      groups: [wheel]
      shell: /bin/bash
      ssh_authorized_keys:
        - ssh-ed25519 AAAA... your-key
storage:
  files:
    - path: /etc/hostname
      mode: 0644
      contents:
        inline: my-vm
systemd:
  units:
    - name: ssh.service
      enabled: true
```

```bash
# Render + create secret in one step (from the e2e regenerate-ignition.sh):
butane -d . ignition.yaml | \
  kubectl create secret generic ignition \
    --from-file=ignition.yaml=/dev/stdin --dry-run=client -o yaml > ignition-secret.yaml
kubectl apply -f ignition-secret.yaml
```

Requires `butane` (https://coreos.github.io/butane/). Local files can be embedded from disk via
`contents.local:` and the `-d .` flag.

### Troubleshooting

- Machine stuck in Pending, event `IgnitionNotReady` → Secret missing, wrong namespace,
  wrong key, or malformed content.
- VM boots but no SSH access → image does not evaluate Ignition, or authorized_keys ended up
  under the wrong user/path (check with the guest console / provider exec).
