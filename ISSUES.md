# Issues & Optimizations Checklist

## Bugs / Functional Issues

- [x] **1. `ansible.cfg` invisible from repo root** — Document that all playbooks must be run from the `playbooks/` directory so `ansible.cfg` (and its `roles_path`) is picked up correctly.
- [x] **2. New inventory vars not consumed** — `ocp_cluster.managed_clusterset_name` / `managed_clusterset_namespace` exist in inventory but are hardcoded in `configure_regional_hub_addons` and `configure_managed_cluster_addons` templates.
- [ ] **3. `full-global-hub-setup.yaml` Tier 2 missing `global_hub_custom_dashboards`** — The ACM/Hive variant doesn't deploy custom Grafana dashboards to Regional Hubs; the IPI variant does.
- [ ] **4. `aws_provision_ocp_ipi` overwrites `~/.aws/credentials [default]`** — Should use a named profile instead of clobbering the default.
- [ ] **5. Standalone playbooks under `playbooks/acm/` and `playbooks/argo/` can't resolve `ocp_cluster.name`** — These rely on an inventory host var not present in `group_vars/all.yaml`. Also, `acm-global-hub-policy-install.yaml` has a stale name/description.

## Credentials

- [ ] **6. Plaintext credentials in inventory** — AWS keys and SMTP app password should be `ansible-vault encrypt_string` encrypted.

## Platform Bugs (External / Upstream)

- [ ] **10. `hive-controllers` CrashLoopBackOff on MCE 2.11.x — broken OpenAPI schema for `UserPermission`**

  **Affected versions:** MCE 2.11.0–2.11.2 / ACM 2.16.x / OCP 4.19–4.20 (any combination that ships MCE 2.11)

  **Symptom:** `hive-controllers` enters CrashLoopBackOff immediately after ACM/MCE installation. The `ClusterDeployment` object is created by Ansible but never gets a `status` block — Hive's reconciler never runs. The Ansible play fails at "Wait for ClusterDeployment to start provisioning" with "Unknown error" after all retries.

  **Root cause:** The `UserPermission` API type was added to `stolostron/cluster-lifecycle-api` on 2025-11-20 ([commit 5d78c3f](https://github.com/stolostron/cluster-lifecycle-api/commit/5d78c3f)). The generated OpenAPI swagger doc (`zz_generated.swagger_doc_generated.go`) emits a `$ref` to `UserPermissionStatus` as a named definition but never includes that type in the `definitions` section. When `hive-controllers` starts, Hive's `resource.Helper` (via `k8s.io/kubectl`) fetches the full OpenAPI v2 schema from the API server (which aggregates the broken `clusterview/v1alpha1` schema from `ocm-proxyserver`), the schema parser hits the dangling `$ref`, and the controller fatals:

  ```
  level=fatal msg="unable to create resource helper" controller=remoteingress
    error="error getting OpenAPISchema: SchemaError(...UserPermission.status):
    unknown model in reference: \"...UserPermissionStatus\""
  ```

  The `hive-operator` itself is also broken by the same schema error when reconciling `HiveConfig`, so it cannot push `disabledControllers` changes to the `hive-controllers` deployment. The `kube-apiserver` caches the broken schema even when `ocm-proxyserver` is scaled to 0, so scaling down the aggregated API server does not help.

  **Affected Hive controllers** (those that initialise a `resource.Helper`): `remoteingress`, `controlplanecerts`. All other controllers checked (`syncidentityprovider`, `clusterrelocate`, `velerobackup`, `clusterversion`, `dnsendpoint`, `unreachable`, `hibernation`) do not use `resource.Helper` and are unaffected. Core provisioning controllers (`clusterdeployment`, `clusterprovision`, `clusterdeprovision`, `dnszone`) are also unaffected.

  **Upstream fix exists** (not yet shipped): `stolostron/cluster-lifecycle-api` commit [`4c06e6b`](https://github.com/stolostron/cluster-lifecycle-api/commit/4c06e6b) — "Fix Hive OpenAPI schema error by adding OpenAPIModelName() for all API types" (2026-03-24). Needs to be pulled into a MCE 2.11.x z-stream release.

  **Last known good version combination:**
  | Component | Last Good | Notes |
  |---|---|---|
  | MCE | 2.10.x | Predates `UserPermission` type entirely |
  | ACM | 2.15.x | Ships with MCE 2.10 |
  | OCP | 4.18 (EUS), 4.19, 4.20 (EUS) | Supported by MCE 2.10 |

  **Workaround for existing broken clusters** (bypass `hive-operator` by patching the deployment directly):
  ```bash
  # Disable only the two affected controllers; all provisioning controllers remain active
  oc patch deployment hive-controllers -n hive \
    --type=json \
    -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args/1",
          "value": "clustersync,machinepool,remoteingress,controlPlaneCerts"}]'
  ```
  The `hive-operator` will attempt to reconcile this back but will itself fail on the same schema error, so the patch persists until MCE is updated.

  **For new cluster builds:** Use ACM 2.15.x / MCE 2.10.x until a MCE 2.11.x z-stream ships the upstream fix. The IPI provisioning path (`full-global-hub-ipi-setup.yaml`) is unaffected since it does not use Hive at all.

  **Support ticket data:**
  - Upstream regression commit: `stolostron/cluster-lifecycle-api@5d78c3f` (2025-11-20)
  - Upstream fix commit: `stolostron/cluster-lifecycle-api@4c06e6b` (2026-03-24)
  - Component: `ocm-proxyserver` image (`multicloud-manager-rhel9`)
  - Hive version on affected cluster: `v1.2.5242-188a312`
  - controller-runtime version: `v0.22.4`

## Optimizations

- [ ] **7. `waitfor_argo_apps` times out serially** — Apps are waited on one-by-one; worst-case wait is `n_apps × 20 min` even though ArgoCD syncs them in parallel.
- [ ] **8. `configure_search_addon` CRD wait always runs** — 30-minute retry loop fires on every run, including idempotent re-runs where the CRD already exists.
- [x] **9. `ansible.cfg` missing readable output format** — Adding `stdout_callback = yaml` would improve readability of long provisioning runs.
