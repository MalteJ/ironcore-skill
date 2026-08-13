# ironcore-skill

An [Agent Skills](https://agentskills.io)-format skill for operating **IronCore**,
the open-source IaaS platform: create and manage Machines (VMs), Volumes, Networks,
IPs, LoadBalancers, NATGateways and Buckets via `kubectl` against the IronCore API.

Works with any agent harness implementing the Agent Skills standard, e.g.
[pi](https://github.com/earendil-works/pi-coding-agent), [Claude Code](https://claude.com/claude-code)
and [OpenAI Codex](https://github.com/openai/codex).

## What the skill does

- **Proposal-first workflow**: users often ask for infrastructure in one sentence
  ("create me a Debian VM"). The skill turns the agent into a consultant first –
  it explores the environment (available MachineClasses, pools, networks), presents a
  complete configuration proposal (instance type, image, networking, IPs, firewall,
  Ignition/SSH, storage) with open questions, and **only applies after user approval**.
- **Correct apply order**: ignition secret → network → (IPAM) → network interface →
  (VirtualIP/NATGateway/LoadBalancer/NetworkPolicy) → volume → machine, waiting per step
  for the right status phases.
- **Hard-learned rules** baked in: Ignition instead of cloud-init, Butane pipeline,
  OCI image semantics, scheduling via selectors only, ephemeral vs. persistent resources,
  bidirectional network peering, and more.

## Contents

```
ironcore/
├── SKILL.md                        # agent instructions (frontmatter + workflow)
└── references/
    ├── api-overview.md             # IronCore object model, fields, status phases
    ├── workflows.md                # YAML skeletons & apply order per use case
    └── images-ignition.md          # OCI images + Ignition pipeline, pitfalls
```

## Installation

Clone this repo, then make the `ironcore/` directory visible to your agent harness:

### pi

Copy or link into your global or project skill directory:

```bash
mkdir -p ~/.pi/agent/skills
ln -s "$(pwd)/ironcore" ~/.pi/agent/skills/ironcore
# or project-local:  ln -s "$(pwd)/ironcore" .pi/skills/ironcore
```

### Claude Code

```bash
mkdir -p ~/.claude/skills
ln -s "$(pwd)/ironcore" ~/.claude/skills/ironcore
# or project-local: .claude/skills/ironcore
```

### OpenAI Codex

```bash
mkdir -p ~/.agents/skills
ln -s "$(pwd)/ironcore" ~/.agents/skills/ironcore
```

Once installed, the skill activates automatically when the task matches its description,
or invoke it explicitly (pi: `/skill:ironcore`, Claude Code: `/ironcore`, Codex: `$ironcore`).

## Prerequisites on the target environment

- `kubectl` with a kubeconfig pointing at an IronCore API server
- optional: `butane` (to author Ignition configuration)

## Background

Built from a structured analysis of the upstream repositories
[ironcore-dev/ironcore](https://github.com/ironcore-dev/ironcore) and
[ironcore-dev/ironcore-image](https://github.com/ironcore-dev/ironcore-image),
plus the official end-to-end samples and usage guides at
[ironcore.dev](https://ironcore.dev/iaas/usage-guides).

## License

[Apache License 2.0](LICENSE)
