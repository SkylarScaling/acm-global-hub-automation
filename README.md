# acm-global-hub-automation

Reusable Ansible automation for deploying a three-tier ACM Global Hub architecture on AWS-hosted OpenShift clusters, with multi-cluster observability via MCO and custom Grafana dashboards.

## Architecture

```
┌─────────────────────────────────────────────┐
│  Tier 1 — Global Hub                        │
│  ACM Global Hub · MCO · ODF · ArgoCD        │
│  Manages: spokes_clusterset (regional hubs) │
└────────────────────┬────────────────────────┘
                     │ ACM
        ┌────────────┴────────────┐
        ▼                         ▼
┌───────────────┐         ┌───────────────┐
│ Tier 2        │         │ Tier 2        │
│ Regional Hub  │   ...   │ Regional Hub  │
│ ACM · MCO     │         │ ACM · MCO     │
│ ODF · ArgoCD  │         │ ODF · ArgoCD  │
└──────┬────────┘         └──────┬────────┘
       │ ACM                     │ ACM
   ┌───┴───┐                 ┌───┴───┐
   ▼       ▼                 ▼       ▼
 Tier 3  Tier 3            Tier 3  Tier 3
Managed Managed           Managed Managed
Cluster Cluster           Cluster Cluster
```

Each tier is an OpenShift cluster on AWS. The Global Hub is the central ACM instance; Regional Hubs are themselves ACM hubs that manage leaf clusters in their region. Observability metrics flow upward to the Global Hub's central Grafana.

## Prerequisites

### Local tools
Install these automatically after cloning:
```bash
cd playbooks
ansible-playbook ocp/install-ocp-prerequisites.yaml --ask-become-pass
```

This installs: `oc`, `openshift-install`, `aws` CLI, and required Python libraries (`kubernetes`, `openshift`, `boto3`). Force-update existing binaries with `-e force_update=true`.

Manual prerequisites (not installed by the playbook):
- Python 3 + pip
- Ansible (`pip3 install ansible`)
- `ansible-galaxy collection install kubernetes.core ansible.posix`

### Local files required
These must exist on the control node before running any playbook:

| File | Purpose |
|---|---|
| `~/pull-secret.json` | Red Hat pull secret (from console.redhat.com) |
| `~/.ssh/id_ed25519.pub` | SSH public key injected into cluster nodes |

### AWS
The AWS account needs permissions for EC2, Route53, S3, IAM, and ELB. The automation will request an EIP quota increase automatically if the current quota is below the threshold.

## Provisioning Strategies

Two top-level playbooks exist depending on how Regional Hubs and Managed Clusters are provisioned:

| Playbook | Tier 1 | Tier 2 & 3 |
|---|---|---|
| `full-global-hub-ipi-setup.yaml` | IPI | IPI (kubeconfig generated locally) |
| `full-global-hub-setup.yaml` | IPI | ACM/Hive (kubeconfig extracted from Hive) |

Both are structured as sequential plays tagged `tier1`, `tier2`, `tier3`, and `summary`, so individual tiers can be targeted independently.

## Running Playbooks

> **All commands must be run from the `playbooks/` directory** so that `ansible.cfg` (which sets the `roles_path`) is picked up correctly.

### Full environment
```bash
cd playbooks
ansible-playbook full-global-hub-ipi-setup.yaml -i inventory.yaml
```

### Single tier
```bash
cd playbooks
# Provision/configure Regional Hubs only
ansible-playbook full-global-hub-ipi-setup.yaml -i inventory.yaml --tags tier2

# Add a single new managed cluster
ansible-playbook full-global-hub-ipi-setup.yaml -i inventory.yaml --tags tier3 --limit managed-cluster-3
```

## Inventory

All cluster configuration is defined in a single inventory YAML file. Create one based on the example in `playbooks/README.md`.

### Inventory groups

| Group | Role | Required `ocp_cluster` vars |
|---|---|---|
| `hub_cluster` | Global Hub (exactly one) | `name`, `spokes_clusterset` |
| `regional_hubs` | Regional Hubs | `name`, `parent_clusterset`, `spokes_clusterset` |
| `managed_clusters` | Leaf clusters | `name`, `parent_clusterset` |

### ClusterSet variables

- **`ocp_cluster.parent_clusterset`** — the ACM ClusterSet this cluster joins on its parent hub. Must match the parent hub's `spokes_clusterset`.
- **`ocp_cluster.spokes_clusterset`** — the ACM ClusterSet that this hub's managed clusters will be placed into.

These are separate from the Ansible inventory group names, which only control which tier of automation runs.

### Per-cluster variables

| Variable | Where | Purpose |
|---|---|---|
| `regional_hub` | `managed_clusters` hosts | Name of the parent Regional Hub host |
| `gitops_version` | `managed_clusters` hosts | GitOps operator version to enforce via ACM policy (`1.18` or `1.19`) |
| `install_config` | all hosts | Node instance types and replica counts |

### Shared `all.vars`

```yaml
all:
  vars:
    home_dir: < /home/youruser >
    tmp_dir: "{{ home_dir }}/tmp"
    openshift_version: "4.20"
    ocp_patch_version: "18"
    base_domain: <base_domain>
    ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    pull_secret: "{{ lookup('file', '~/pull-secret.json') | from_json }}"
    aws:
      account_id: <account_id>
      aws_access_key_id: <key_id>
      aws_secret_access_key: <secret>
      aws_region: <aws_region>
    smtp:
      host: "smtp.gmail.com"
      port: "587"
      from_address: "<email>"
      require_tls: "true"
      username: "<email>"
      app_password: "<app_password>"
      default_to: "<default_recipient>"
      esp_to: "<esp_recipient>"
```

Kubeconfigs are written to `{{ tmp_dir }}/acm-global-hub/install/{{ ocp_cluster.name }}/auth/kubeconfig`.

## Day 2 Configuration

After provisioning, Day 2 config is applied via ArgoCD Applications pointing to external GitOps repos. The default repo URLs in role defaults point to `SkylarScaling/gitops-install-*`. Override these in your inventory or role vars for your own environment:

| Component | Role | Default repo |
|---|---|---|
| ACM | `argo/apps_install_acm` | `gitops-install-acm` |
| ODF | `argo/apps_install_odf` | `gitops-install-odf` |
| MCO | `argo/apps_install_mco` | `gitops-install-mco` |
| Global Hub | `argo/apps_install_global_hub` | `gitops-install-global-hub` |
| COO | `argo/apps_install_coo` | `gitops-install-coo` |

## Optional Features

Set these in your inventory `all.vars` or per-host to enable:

| Variable | Default | Description |
|---|---|---|
| `enable_uwm` | `false` | Deploy a User Workload Monitoring ACM policy to managed clusters. Required if monitoring workloads running in user namespaces (e.g. ArgoCD metrics). |
| `force_update` | `false` | Force reinstall of OCP binaries even if already present. |
