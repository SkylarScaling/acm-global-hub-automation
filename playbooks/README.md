# How to Use These Playbooks

## Playbook Selection

| Playbook | Architecture | Spoke Provisioning |
|---|---|---|
| `full-global-hub-setup.yaml` | Three-tier Global Hub | Tier 2+3 via ACM/Hive |
| `full-global-hub-ipi-setup.yaml` | Three-tier Global Hub | All tiers via IPI |
| `hub-spoke-setup.yaml` | Two-tier Hub+Spoke | Spokes via ACM/Hive |
| `hub-spoke-ipi-setup.yaml` | Two-tier Hub+Spoke | All clusters via IPI |

---

## Three-Tier Global Hub

### Deploy the entire environment (Tiers 1, 2, and 3):

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml
```

### Deploy ONLY a new batch of Managed Clusters (Tier 3):
Let's say a month from now you add `managed-cluster-3` and `managed-cluster-4` to your inventory file. You don't want to touch the Hubs. Just run:

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags "tier3"
```

### Deploy a specific tier, but limit it to a single new cluster:

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags "tier3" --limit "managed-cluster-3"
```

---

## Two-Tier Hub+Spoke (no Global Hub)

The hub+spoke playbooks deploy a single ACM hub with direct managed cluster attachment — no Regional Hubs, no ACM Global Hub. The hub gets ACM, ODF, MCO, and GitOps enforcement policies.

### Deploy hub and all spokes (all IPI):

```bash
ansible-playbook playbooks/hub-spoke-ipi-setup.yaml -i inventory.yaml
```

### Deploy hub and all spokes (spokes via Hive):

```bash
ansible-playbook playbooks/hub-spoke-setup.yaml -i inventory.yaml
```

### Deploy hub only, add spokes later:

```bash
ansible-playbook playbooks/hub-spoke-ipi-setup.yaml -i inventory.yaml --tags "hub"
# ... later, add spokes to inventory ...
ansible-playbook playbooks/hub-spoke-ipi-setup.yaml -i inventory.yaml --tags "spokes"
```

### Add a single new spoke:

```bash
ansible-playbook playbooks/hub-spoke-ipi-setup.yaml -i inventory.yaml --tags "spokes" --limit "managed-cluster-3"
```

---

## Example Inventories

### Global Hub Inventory
```yaml
all:
  vars:
    home_dir: /home/sscaling
    tmp_dir: "{{ home_dir }}/tmp"
    openshift_version: "4.20"
    ocp_patch_version: "13"
    odf_git_repo_revision: "main"
    force_update: true
    base_domain: "sandbox123.opentlc.com"
    ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    pull_secret: "{{ lookup('file', '~/pull-secret.json') | from_json }}"
    aws:
      account_id: <aws_account_id>
      aws_access_key_id: <aws_access_key_id>
      aws_secret_access_key: <aws_secret_access_key>
      aws_region: <aws_region>
    smtp:
      host: "smtp.gmail.com"
      port: "587"
      from_address: "<your_email>@gmail.com"
      require_tls: "true"
      username: "<your_username>@gmail.com"
      app_password: "<16 char app password>"
      default_to: "<default_email>@gmail.com"
      esp_to: "<esp_rule_email>@gmail.com"
  children:
    hub_cluster:
      hosts:
        global-hub:
          ansible_connection: local
          ocp_cluster:
            name: "global-hub"
            spokes_clusterset: regional-hubs
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.2xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
            infra:
              instance_type: "m6a.4xlarge"
              replicas: "3"
            storage:
              instance_type: "m6a.4xlarge"
              replicas: "3"
    regional_hubs:
      hosts:
        regional-hub-1:
          ansible_connection: local
          ocp_cluster:
            name: "regional-hub-1"
            parent_clusterset: regional-hubs
            spokes_clusterset: managed-clusters
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.2xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
            infra:
              instance_type: "m6a.4xlarge"
              replicas: "3"
            storage:
              instance_type: "m6a.4xlarge"
              replicas: "3"
        regional-hub-2:
          ansible_connection: local
          ocp_cluster:
            name: "regional-hub-2"
            parent_clusterset: regional-hubs
            spokes_clusterset: managed-clusters
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.2xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
            infra:
              instance_type: "m6a.4xlarge"
              replicas: "3"
            storage:
              instance_type: "m6a.4xlarge"
              replicas: "3"
    managed_clusters:
      hosts:
        managed-cluster-1:
          ansible_connection: local
          regional_hub: regional-hub-1
          gitops_version: "1.18"
          ocp_cluster:
            name: "managed-cluster-1"
            parent_clusterset: managed-clusters
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
        managed-cluster-2:
          ansible_connection: local
          regional_hub: regional-hub-2
          gitops_version: "1.19"
          ocp_cluster:
            name: "managed-cluster-2"
            parent_clusterset: managed-clusters
          install_config:    
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
```

### Hub+Spoke Inventory

Simpler than the global hub inventory: no `regional_hubs` group, no `regional_hub` per-spoke variable. Managed clusters reference their clusterset via `parent_clusterset`; the hub targets them via `spokes_clusterset`.

```yaml
all:
  vars:
    home_dir: /home/sscaling
    tmp_dir: "{{ home_dir }}/tmp"
    openshift_version: "4.20"
    ocp_patch_version: "13"
    force_update: true
    base_domain: "sandbox123.opentlc.com"
    ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    pull_secret: "{{ lookup('file', '~/pull-secret.json') | from_json }}"
    aws:
      account_id: <aws_account_id>
      aws_access_key_id: <aws_access_key_id>
      aws_secret_access_key: <aws_secret_access_key>
      aws_region: <aws_region>
    smtp:
      host: "smtp.gmail.com"
      port: "587"
      from_address: "<your_email>@gmail.com"
      require_tls: "true"
      username: "<your_username>@gmail.com"
      app_password: "<16 char app password>"
      default_to: "<default_email>@gmail.com"
      esp_to: "<esp_rule_email>@gmail.com"
  children:
    hub_cluster:
      hosts:
        acm-hub:
          ansible_connection: local
          ocp_cluster:
            name: "acm-hub"
            spokes_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.2xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
            infra:
              instance_type: "m6a.4xlarge"
              replicas: "3"
            storage:
              instance_type: "m6a.4xlarge"
              replicas: "3"
    managed_clusters:
      hosts:
        managed-cluster-1:
          ansible_connection: local
          gitops_version: "1.18"
          ocp_cluster:
            name: "managed-cluster-1"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
        managed-cluster-2:
          ansible_connection: local
          gitops_version: "1.19"
          ocp_cluster:
            name: "managed-cluster-2"
            parent_clusterset: managed-clusters
          install_config:
            cluster_name: "{{ ocp_cluster.name }}"
            control_plane:
              instance_type: "m6a.xlarge"
            workers:
              instance_type: "m6a.xlarge"
              replicas: "3"
```