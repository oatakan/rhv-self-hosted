# Copilot instructions (rhv-self-hosted)

## What this repo is
- Ansible playbooks + local roles for deploying oVirt/RHV (including self-hosted engine) and for provisioning/deprovisioning nested nodes on providers (oVirt, VMware, KubeVirt).
- Heavy lifting comes from Ansible Galaxy collections/roles (see `collections/requirements.yml` and `roles/requirements.yml`).

## Key entry playbooks (run targets)
- `provision_self_hosted_engine_ovirt.yml`: installs/configures an oVirt Engine on the target host. Loads `vars/rhel_{{ ansible_distribution_major_version }}.yml` + `vars/engine.yml`, registers to RHN on RHEL, then runs roles like `common`, `ovirt.ovirt.engine_setup`, `ovirt-engine-config`, `ovirt-host-vdsm-hooks`, `edk2`.
- `provision_setup_host_ovirt.yml`: 2-phase workflow: (1) configure packages/storage on the host, then (2) from localhost configure datacenter/cluster/hosts/templates using the oVirt API (vars in `vars/host.yml`).
- `provision_infra_{ovirt,vmware,kubevirt,multi}.yml`: runs on localhost (`connection: local`) and uses `node-config/node-*.yml` to create nested nodes via external roles (`ansible-role-ovirt`, `ansible-role-vmware`, `ansible-role-kubevirt`).
- `remove_infra_{ovirt,vmware,multi}.yml`: deprovisions the above via `role_action: deprovision`.
- Smaller entrypoints: `add-host.yml` (role `ovirt-engine-add-host`), `add-storage-to-engine.yml` (roles `ovirt-engine-export-local-nfs` + `ovirt-engine-add-storage`), `add-gpu-passthrough.yml` (roles `oatakan.rhel_gpu_passthrough` + `edk2`).

## Setup + common commands
- Install dependencies:
  - `ansible-galaxy collection install -r collections/requirements.yml`
  - `ansible-galaxy role install -r roles/requirements.yml -p roles`
- Typical runs (examples):
  - `ansible-playbook -i hosts-21 provision_self_hosted_engine_ovirt.yml`
  - `ansible-playbook -i hosts-21 provision_setup_host_ovirt.yml`
  - `ansible-playbook provision_infra_vmware.yml` (runs on localhost)

## Repo-specific conventions to follow when editing
- Variable layering:
  - OS/version-specific repos: `vars/rhel_*.yml` (`repos.rhv_engine`, `repos.rhv_host`). Note: `vars/rhel_10.yml` currently defines `rhv_host` twice—be careful when modifying.
  - Engine setup defaults: `vars/engine.yml` (`ovirt_engine_setup_version`, `ovirt_repositories_ovirt_release_rpm`, `ovirt_engine_config`).
  - oVirt API/infra objects (datacenter/clusters/hosts/storages/templates): `vars/host.yml`.
- Local-vs-remote split is intentional:
  - Host prep runs on the remote host (`become: true`).
  - oVirt API operations run locally and use `delegate_to: localhost` + `ovirt.ovirt.ovirt_auth` (see `provision_setup_host_ovirt.yml`).
  - Many API roles are wrapped in `block/rescue` to retry once; preserve this pattern for flakey API timing.
- Engine tuning is done via the CLI `engine-config` in role `ovirt-engine-config` using the list `ovirt_engine_config` (version-specific via `--cver` and also `general`).
- Some roles execute scripts from `roles/*/files/` (e.g. `roles/ovirt-engine-add-host/files/add_host`, `roles/ovirt-engine-add-storage/files/storage_setup`) and run them with `command:`; keep quoting/argument formatting stable.
- RHN registration/unregistration is guarded and always cleaned up in `always:` blocks unless `rhn_offline: true`.

## Security/ops notes
- Inventory `hosts-21` contains `ansible_password` inline; avoid logging secrets (use `no_log: true` like existing playbooks).
