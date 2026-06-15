# How to Use These Playbooks

## Deploy the entire environment (Tiers 1, 2, and 3):

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml
```

## Deploy ONLY a new batch of Managed Clusters (Tier 3):
Let's say a month from now you add `managed-cluster-3` and `managed-cluster-4` to your inventory file. You don't want to touch the Hubs. Just run:

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags "tier3"
```

## Deploy a specific tier, but limit it to a single new cluster:

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags "tier3" --limit "managed-cluster-3"
```

## Example Inventory
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