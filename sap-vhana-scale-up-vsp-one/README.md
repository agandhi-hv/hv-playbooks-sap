# Virtual SAP HANA (vHANA) Scale-Up: VSP One Block Storage Provisioning

Ansible pipeline that provisions VSP One Block storage for a vHANA scale-up host (ESXi, SAN boot).

Tested on: hitachivantara.vspone_block 4.4.2, ansible-core 2.16.14, Python 3.9.18, VSP One Block 24.

## Pick the main playbook
| Connection | Playbook | Status |
|---|---|---|
| Direct FC (no switch) | `vhana_main_direct.yml` | Validated |
| FC switch | `vhana_main_switch.yml` | Not validated on the reference testbed |

## Direct FC workflow
1. `0_storagesystem_validate.yml`: validates the storage model and serial.
2. `1_disk_drives_fact.yml`: finds free drives, checks the RAID6 minimum.
3. `2_ddp_pool_create.yml`: creates the DDP pool.
4. `3_load_nodes.yml`: reads `nodes_info.csv`.
5. `4_storage_port_create.yml`: configures the storage ports (port security off for direct FC).
6. `5_hostgroup_create_direct.yml`: adds one WWN to each default host group (for example 1A-G00 and 2A-G00).
7. `6_vsp1_volume_creation_mapping_direct.yml`: creates the volumes and maps them to both ports.

## Files to edit
- `nodes_info.csv`: `hostname,port1,wwn1,port2,wwn2`. The hostname is used to name the volumes.
- `vsp_direct_global_variables.yml`: pool name, volume sizes, host mode and options.
- Vault file `~/playbooks/ansible_vault_vars/ansible_vault_storage_var.yml` (not included): storage_address, storage_serial, vault_storage_username, vault_storage_secret.

## Default LUN layout
| LUN | Volume | Size |
|---|---|---|
| 0 | ESXi OS (boot) | 256 GB |
| 1 | OS | 500 GB |
| 2 | Shared | 2048 GB |
| 3-6 | Log x4 | 128 GB each |
| 7-10 | Data x4 | 1024 GB each |

## Run

    ansible-playbook -J vhana_main_direct.yml

## Notes for direct FC
- Direct FC must use the default host groups (1A-G00, 2A-G00). Custom host groups cause fabric login failures.
- Deleting volumes: unmap first, and power off the host. Volumes with active I/O cannot be unmapped.
- The host group playbook (5_hostgroup_create_direct.yml) ignores errors, so a failed task will not stop the run. Verify host groups and LUN paths after the run.
- Volumes are created in a fixed order and mapped to LUNs 0-10 in that order: ESXi OS (LUN 0, boot), OS, shared, 4 log, 4 data. Do not reorder. LDEV IDs may not be contiguous.
- Volume names: `<hostname>esxi_os`, `<hostname>_os`, `<hostname>_shared`, `<hostname>_log_01-04`, `<hostname>_data_01-04`.
- All volumes are created with compression and data reduction share enabled (hardcoded, no dedup). Compression-enabled volumes must be blocked before a format, see below.
- The volume playbook stops on the first error. Do not rerun it against existing volumes without checking first.
- To re-format the boot LUN, power off the host, then run on the storage CLI:

```
raidcom modify ldev -ldev_id 0 -status blk
raidcom initialize ldev -ldev_id 0 -operation fmt
raidcom add lun -port CL1-A -lun_id 0 -ldev_id 0
raidcom add lun -port CL2-A -lun_id 0 -ldev_id 0
```

  Wait until FMT_STS shows - before re-adding the LUNs.
