<!-- BEGIN Title -->
# sap_patching Ansible Role
<!-- END Title -->

## Description
<!-- BEGIN Description -->
The Ansible Role `sap_patching` automates the patching of different components of SAP Systems.

It provides tasks for preparing the patching source files, applying the patches and stopping/starting the system as and when required. The role provides advanced restart strategies such as ABAP kernel bootstrap or Rolling Kernel Switch (RKS) for kernel updates.

### Compatible Components
The following components can be patched with this role:

- ABAP Kernel
- HANA Database client (both local and central installation)
- HANA Database platform edition (with or without HSR and cluster)
- SAP Host Agent (both automated and manual update)
- SAP Web Dispatcher

### Patching Steps
The following table summarises the steps which can/must be performed when patching each of the compatible components.

| Component \ Step | Prepare | Application stop | Apply | Application start | Restart | Post |
| --- | --- | --- | --- | --- | --- | --- |
| ABAP kernel | ✓ | Optional | ✓ | No | Full restart, bootstrap or RKS | Manual `saproot.sh`, only with RKS |
| HANA DB client | ✓ | Required | ✓ | Required | N/A | N/A |
| HANA DB platform | ✓ | Optional | ✓ | If stopped | N/A | N/A |
| Host Agent | ✓ | No | ✓ | No | No | N/A |
| Web Dispatcher | ✓ | No | ✓ | No | Full restart | No |

Refer to the [Execution](#execution) section for details on each component and its patching workflow, and to the [Testing](#testing) section for the versions which were tested.
<!-- END Description -->

<!-- BEGIN Dependencies -->
<!-- END Dependencies -->

## Prerequisites
<!-- BEGIN Prerequisites -->
**Control Node:**

- Patching files must be downloaded before this role is executed. See [Source Patching Files](#source-patching-files).
- The SAPCAR executable must be present in `sap_patching_sapcar_path_local`, when `sap_patching_sapcar_on_target` is set to `false`.

**Managed Nodes:**

- The SAPCAR executable must be present in `sap_patching_sapcar_path` and must be executable by the user owning the patching files.<br>
Set `sap_patching_sapcar_on_target` to `false` to copy it from the control node instead.
- The staging directories of the patched components must be accessible and writable by the user owning the patching files.
- The user owning the patching files must exist on every host in the play. See [File ownership](#file-ownership).

### Source Patching Files
Patching files must be downloaded and placed in the appropriate source directory or directories before running this role. You can obtain patching files manually using the SAP Download Manager or the [community.sap_launchpad Ansible collection](https://github.com/sap-linuxlab/community.sap_launchpad).

> **NOTE:** The kernel source directory must be separate from all other source directories and can contain ONLY kernel related SAR and info files, because kernel SAR files cannot be safely identified by their name alone.

### Staging directories
Both the source files and the extracted patches must be stored in directories accessible from the system which is being patched. Two independent variables per component control how the files get there:

- `sap_patching_<component>_files_on_target` - where the source SAR files come from.
  - `true` (default) - The SAR files are already present on the managed nodes in `sap_patching_<component>_source` and are extracted in place.
  - `false` - The SAR files are on the control node in `sap_patching_<component>_source_local` and are copied to `sap_patching_<component>_source` on the managed nodes during the prepare step.
- `sap_patching_<component>_files_shared` - whether the staging area `sap_patching_<component>_base` is one shared filesystem, e.g. NFS.
  - `true` (default) - _This is the simplest and most efficient option._ The copy and the SAR extraction are executed only once, because all hosts see the same directory. The shared directory has to be accessible by all hosts that belong to one SAP system. It can differ between SAP systems, i.e. Dev, QA and Prod can have different NFS shares.
  - `false` - The copy and the SAR extraction are executed on every host separately.

Directory and permission checks are always executed on every host in the play, regardless of these two variables.

> **NOTE:** When `files_on_target` is `false` and `files_shared` is `false`, the same files are copied from the control node to every host. This copy is throttled to one host at a time, unless `sap_patching_disable_throttle` is set to `true`.

### File ownership
All files created in the staging area are owned by a single operating system user and group for the whole run:

- User - `sap_patching_file_owner`, or `<sap_patching_sap_system_sid>adm` when it is not set.
- Group - `sap_patching_file_group`, which defaults to `sapsys`.

> **LIMITATION:** The owner is uniform for the whole run, so a system where the SAP HANA database and the ABAP system use different SIDs is not compatible with the derived owner. Only one of `hdbadm` and `<sid>adm` can own the staging files. Set `sap_patching_file_owner` to a user which exists on every host in the play, for example the user owning the shared NFS filesystem.
<!-- END Prerequisites -->

## Execution
<!-- BEGIN Execution -->
### Execution plan
The patching process is controlled using the list `sap_patching_execution_plan`. The steps in the execution plan and their order determine what happens and when. For example, to patch the ABAP kernel and restart the system you can use the following execution plan:

```yaml
sap_patching_execution_plan:
  - abap_kernel_prepare
  - abap_kernel_apply
  - abap_kernel_restart
```

The behaviour of each step can be controlled using the step specific variables described in [Role Variables](#role-variables).

### Available steps
Every item of `sap_patching_execution_plan` must be one of the following steps. Any other value fails the validation before the first step is executed.

| Step | Component | Description |
| --- | --- | --- |
| `abap_kernel_prepare` | ABAP kernel | Validate the kernel SAR files and extract them into the staging area. |
| `abap_kernel_apply` | ABAP kernel | Copy the prepared kernel to the global SAP executable directory. |
| `abap_kernel_restart` | ABAP kernel | Restart the system using `sap_patching_kernel_restart_strategy`. |
| `hdb_client_prepare` | HANA DB client | Validate the HANA DB client SAR files and extract them into the staging area. |
| `hdb_client_apply` | HANA DB client | Update the HANA DB client with `hdbinst`. Requires a stopped ABAP system. |
| `hdb_server_prepare` | HANA DB platform | Validate the HANA DB server SAR files and extract them into the staging area. |
| `hdb_server_apply` | HANA DB platform | Update the HANA DB server with `hdblcm`. |
| `hdb_server_update_prepare` | HANA DB platform | Execute the update preparation phase of `hdblcm` only. |
| `hdb_server_update_resume` | HANA DB platform | Reserved, not implemented. See [Limitations](#limitations). |
| `host_agent_prepare` | SAP Host Agent | Validate the SAP Host Agent SAR files and extract them into the staging area. |
| `host_agent_apply_auto` | SAP Host Agent | Update the SAP Host Agent using autoupdate. |
| `host_agent_apply_manual` | SAP Host Agent | Update the SAP Host Agent on each host with `saphostexec`. |
| `webdisp_prepare` | SAP Web Dispatcher | Validate the Web Dispatcher SAR files and extract them into the staging area. |
| `webdisp_apply` | SAP Web Dispatcher | Copy the prepared Web Dispatcher to its global executable directory. |
| `webdisp_restart` | SAP Web Dispatcher | Restart all Web Dispatcher instances of the host. |
| `abap_stop_system` | ABAP | Stop the whole ABAP system. |
| `abap_start_system` | ABAP | Start the whole ABAP system. |

> **NOTE:** There is no step for `saproot.sh`. It is executed as part of `abap_kernel_restart` and `webdisp_restart`, except with the RKS restart strategy. See [Kernel Patching](#kernel-patching).

The patching process of each component consists of several steps (typicaly at least two):

- **Prepare:**
  This validates and prepares patch files. SAR files are read from a source directory and then extracted and archived in an appropriate directory in preparation for application. For kernel this includes extraction of kernel patches in the correct order (see below).
- **Apply:**
  This executes the patching process. In some cases this step may require that the system is down (see the table above). Generally this step copies files prepared in the prepare step to the correct location, sets the permissions (if required) and executes other required task as per the component type.
- **Stop/Start/Restart:**
  This allows to stop/start the whole system. Kernel/Webdisp restart is a special step which deals with the intricacies of ABAP kernel patching (e.g. bootstrap, saproot.sh, etc).
- **Post:**
  Performs any post-patching actions. Currently only ABAP kernel Rolling Kernel Switch needs a specific post action, which has to be executed manually.

### Kernel Patching

The following steps are available:

- **Kernel Preparation:**
  - Step name `abap_kernel_prepare`.
  - Validates and locates kernel SAR files in the source directory.
  - NOTE: Due to the difficulties with _safely_ identifying kernel SAR files the source directory can contain ONLY the kernel SAR files (or nothing).
  - Handles duplicate detection and patch number extraction.
  - Handles both autodetect and manual kernel version selection.
- **Kernel Application:**
  - Step name `abap_kernel_apply`.
  - Validates kernel directories and files.
  - Copies kernel files to the global SAP executable directory.
- **ABAP Kernel Restart:**
  - Step name `abap_kernel_restart`.
  - Available for RKS, full system restart and bootstrap strategies.
  - Ensures a proper update of all special files, bootstrap, system restart and execution of saproot.sh.
- **Post-Patching Permission Adjustment:**
  - `saproot.sh` adjusts permissions (icmbnd* and sapuxuserchk) after patching.
  - It is executed automatically as part of `abap_kernel_restart` with the `all` and `bootstrap` restart strategies.
  - With the `rks` restart strategy it is NOT executed, because the role does not wait for the Rolling Kernel Switch to complete. It has to be executed manually after all instances have been restarted.

> **NOTE:** There is no execution plan step for `saproot.sh`. Adding one to `sap_patching_execution_plan` fails the validation of the plan.

See [sample-sap_patching_kernel.yml](../../playbooks/sample-sap_patching_kernel.yml) and [sample-sap_patching_full.yml](../../playbooks/sample-sap_patching_full.yml).

#### Preparation Modes

The default (and recommended) mode is `autodetect`. This is set using `sap_patching_kernel_prepare_version: autodetect`. This will take all SAR files from the source directory, perform checks and prepare a new kernel to be applied.

Two additional modes are available when the source directory does not contain a full set of kernel SAR files:

- **Delta Mode:**
  - Used when the source directory contains only `dw*.sar` files and no `SAPEXE`/`SAPEXEDB` files, which is not a full kernel.
  - The prepared kernel gets the suffix **-delta**, for example `k793p330-delta`, to mark it as a delta over the latest full kernel rather than a complete kernel.
  - The full kernel, for example `k793p300`, must be prepared beforehand from a separate source directory.
  - To reach patch level 330, apply the full kernel `k793p300` first and then run the patching process again to apply `k793p330-delta`.
  - This is prone to user error, so the recommendation is to prepare a full kernel every time patching is executed.
- **Append-to Mode:**
  - Used to patch files other than `disp+work`, for example `R3trans` or `tp`, on top of an already prepared kernel.
  - Enabled by setting `sap_patching_kernel_prepare_append_to_version` to the prepared kernel which is topped up.
  - The kernel patch number does not change, because `disp+work` is not replaced, but the remaining kernel files are updated.
  - The source directory must contain kernel patches and must not contain `dw*.sar` or `SAPEXE*.sar` files, because those would change the kernel patch number.

#### Kernel FAQ

- **My system is running on multiple platforms and I need to patch them both**
  - Separate all folders and create a structure which takes both platforms into the account.
  - Run `abap_kernel_prepare` twice each time with different settings.
  - Run `abap_kernel_apply` twice as well and make sure that it is executed once on a server for each platform (e.g. once on linuxx86_64 and once on linuxs390x).

### HANA DB Client Patching

The following steps are available:

- **Client Preparation:**
  - Step name `hdb_client_prepare`.
  - Validates and prepares the HANA DB client installation files.
  - Ensures the correct version and patch level are selected for deployment.
- **Client Application:**
  - Step name `hdb_client_apply`.
  - Installs or updates the HANA DB client on target systems.
  - Both individual and shared client installations can be updated, controlled by `sap_patching_hdb_client_install_path` and `sap_patching_hdb_client_shared_path`.
  - With a shared installation, `hdbinst` is executed only once, on a single host.
  - The ABAP system must be stopped, because `hdbinst` cannot replace the client libraries which are locked by the running work processes.

The step `hdb_client_apply` has to be surrounded by the stop and the start of the ABAP system. The role does not add them, they are steps of the execution plan:

```yaml
sap_patching_execution_plan:
  - abap_stop_system
  - hdb_client_apply
  - abap_start_system
```

See [sample-sap_patching_others.yml](../../playbooks/sample-sap_patching_others.yml) and [sample-sap_patching_full.yml](../../playbooks/sample-sap_patching_full.yml).

### HANA DB Server Patching

The following steps are available:

- **Server Preparation:**
  - Step name `hdb_server_prepare`.
  - Validates and prepares the HANA DB server installation files.
  - Ensures the correct version and patch level are selected for deployment.
- **Server Application Preparation:**
  - Step name `hdb_server_update_prepare`.
  - Performs all update preparation steps upto the downtime phase, by passing `--prepare_update` to `hdblcm`.
  - Optional step which reduces the downtime duration.
- **Server Application:**
  - Step name `hdb_server_apply`.
  - Updates the HANA DB server on target systems.
  - If update preparation was executed beforehand, it will automatically continue with the downtime phase of the update.
  - If update preparation was NOT executed, it will automatically perform the full update/upgrade process.
- **Server Application Resume:**
  - Step name `hdb_server_update_resume`.
  - Reserved for resuming an interrupted update, but NOT implemented. See [Limitations](#limitations).

When `sap_patching_hdb_server_is_clustered` is `true`, the role puts the SAP HANA cluster resource into maintenance mode for the duration of the update and relocates it when required. The relocation does not wait for the takeover to complete, so `sap_patching_hdb_server_cluster_relocate_timeout` limits how long the role waits for it.

See [sample-sap_patching_hdb_server.yml](../../playbooks/sample-sap_patching_hdb_server.yml) and [sample-sap_patching_full.yml](../../playbooks/sample-sap_patching_full.yml).

#### HDBLCM Configuration file

Dynamically generated configuration file is used to control how HDBLCM is behaving. This is similar to the behaviour of ```community.sap_install.sap_hana_install role```. The configuration file is created in folder `ansible_templates/<github_hash>/<hostname>_hdblcm_configfile.*` inside the directory of the applied HANA DB server. Where:

- `<github_hash>` is a unique version of the HDBLCM. This changes with each HANA DB Server patch and it will force generation of a new config file even if one existed for a previous version of HDBLCM.
- `<hostname>_hdblcm_configfile.*` represent several files which start with the HANA DB server hostname. These are used to generate the config file (suffix .cfg) which is then passed to HDBLCM during execution.

The configuration file is generated only when it doesn't exist. This is inline with ```community.sap_install.sap_hana_install role``` behaviour. This also allows users to define their own additional variables in format `sap_patching_hdb_server_cf_<configfile_variable>` which will replace the default values in HBBLCM configuration file. This can be used to alter the behaviour of HDBLCM during the update/upgrade. For more details see the comments in [tasks/hana/common/hdblcm_configfile.yml](tasks/hana/common/hdblcm_configfile.yml).

### Host Agent Patching

The following steps are available:

- **Host Agent Preparation:**
  - Step name `host_agent_prepare`.
  - Validates and prepares the installation files.
  - Ensures the correct version and patch level are selected for deployment.
- **Host Agent Application:**
  - Step name `host_agent_apply_auto` for automatic agent update (See N1473974) or `host_agent_apply_manual` for manual update on each server.
  - Installs or updates the host agent on target systems.
  - The host agent is always installed locally on every host in the play. Only the staging area can be shared, using `sap_patching_host_agent_files_shared`.

### SAP Web Dispatcher Patching

The following steps are available:

- **WebDisp Preparation:**
  - Step name `webdisp_prepare`.
  - Validates and prepares the installation files.
  - Ensures the correct version and patch level are selected for deployment.
- **WebDisp Application:**
  - Step name `webdisp_apply`.
  - Installs or updates the Web Dispatcher on target systems.
  - Every host has its own global executable directory, so the patch is applied on all hosts in the play.
- **WebDisp Restart:**
  - Step name `webdisp_restart`.
  - Ensures proper update of all special files, bootstrap, system restart and execution of saproot.sh.

See [sample-sap_patching_others.yml](../../playbooks/sample-sap_patching_others.yml) and [sample-sap_patching_full.yml](../../playbooks/sample-sap_patching_full.yml).

### Limitations

- **One file owner per run.** All files created in the staging area are owned by a single user, so a system where the SAP HANA database and the ABAP system use different SIDs cannot have both `<sid>adm` users own their own files. See [File ownership](#file-ownership).
- **There are no steps to stop or start the HANA database on its own.** Only `abap_stop_system` and `abap_start_system` are available. The HANA database is stopped and started by `hdblcm` as part of the update.
- **The kernel source directory cannot be shared with other components.** It must contain only kernel SAR and info files. See [Source Patching Files](#source-patching-files).
- **Not every step is idempotent.** If the role fails, the state of the system has to be checked before it is executed again.
<!-- END Execution -->

### Execution Flow
<!-- BEGIN Execution Flow -->
1. Validate the global variables, the file ownership and the SAPCAR executable.
2. Validate every step of the execution plan defined in `sap_patching_execution_plan` and execute the validators of its component. Each validator is executed only once per run.
3. Execute the steps in the order defined in `sap_patching_execution_plan`. Each step detects on which hosts in the play the component is present, and elects a primary host for the actions which must be executed only once.
4. Show the summary of the executed steps.

If any task fails on any host, the role stops on all hosts in the play and reports which steps were not executed.

> **NOTE:** When `sap_patching_hdb_server_is_clustered` is `true` and the role fails, the cluster can be left in a state which the role does not clean up. The maintenance mode can still be enabled and the relocation of the SAP HANA clone resource can still be enforced by a location constraint.
<!-- END Execution Flow -->

### Example
<!-- BEGIN Execution Example -->
Complete sample playbooks are available in the [playbooks](../../playbooks/) directory.

Example of patching the SAP Host Agent and the HANA database on the `hana` hosts, followed by the SAP Host Agent and the ABAP kernel on the `app` hosts. Each host group is patched by its own play, because the staging files of one run are owned by a single user. See [File ownership](#file-ownership).

```yaml
---
- name: Ansible Play for SAP patching - HANA database hosts
  hosts: hana
  become: true
  any_errors_fatal: true
  tasks:
    - name: Execute Ansible Role sap_patching
      ansible.builtin.include_role:
        name: sap_patching
      vars:
        sap_patching_sap_system_sid: "H01"
        sap_patching_sapcar_path: "/install"
        sap_patching_sapcar_file_name: "SAPCAR.EXE"
        sap_patching_host_agent_base: "/install/host_agent"
        sap_patching_host_agent_source: "/install/host_agent/source"
        sap_patching_hdb_server_base: "/install/hana/server"
        sap_patching_hdb_server_source: "/install/hana/server/source"
        sap_patching_hdb_server_activate_admin_userstore_key: "HDB_SYSTEMDB"
        sap_patching_execution_plan:
          - host_agent_prepare
          - host_agent_apply_manual
          - hdb_server_prepare
          - hdb_server_apply

- name: Ansible Play for SAP patching - ABAP application hosts
  hosts: app
  become: true
  any_errors_fatal: true
  tasks:
    - name: Execute Ansible Role sap_patching
      ansible.builtin.include_role:
        name: sap_patching
      vars:
        sap_patching_sap_system_sid: "S4H"
        sap_patching_sapcar_path: "/install"
        sap_patching_sapcar_file_name: "SAPCAR.EXE"
        sap_patching_host_agent_base: "/install/host_agent"
        sap_patching_host_agent_source: "/install/host_agent/source"
        sap_patching_kernel_base: "/install/kernel"
        sap_patching_kernel_source: "/install/kernel/source"
        sap_patching_kernel_restart_strategy: "all"
        sap_patching_execution_plan:
          - host_agent_prepare
          - host_agent_apply_manual
          - abap_kernel_prepare
          - abap_kernel_apply
          - abap_kernel_restart
```
<!-- END Execution Example -->

## Testing
<!-- BEGIN Testing -->
The following versions were used to test this role. Older versions are expected to work, but they were not verified.

| Component | Tested | Expected to work |
| --- | --- | --- |
| ABAP Kernel | S/4 HANA 2023+ with kernel 7.93 | NW 7.50+ |
| HANA Database client | Not recorded | - |
| HANA Database platform edition | HANA 2.07-09 | HANA 2.04+ |
| SAP Host Agent | Not recorded | 7.22+ |
| SAP Web Dispatcher | 7.93+ | 7.7+ |

The following SAP HANA scenarios were tested:

- Full upgrade in "one go" and upgrade split into prepare and upgrade steps.
- HANA System Replication enabled systems, with upto four systems in multi target configuration.
- Two-node HA cluster with Pacemaker and HANA System Replication.

> **NOTE:** The cluster commands are available for Red Hat (`pcs`) and SUSE (`crmsh`).
<!-- END Testing -->

## Maintainers
<!-- BEGIN Maintainers -->
- Rob Dobozy at SAP
- [Rob Dobozy](https://github.com/rob0d) (Author)
- [Marcel Mamula](https://github.com/marcelmamula)
<!-- END Maintainers -->

## Role Variables
<!-- BEGIN Role Variables -->
### sap_patching_execution_plan
- _Type:_ `list` of type `str`
- _Default:_ `[]`

The list of steps executed by this role, in the given order.<br>
Must be a non-empty list and every item must be one of the steps listed in [Available steps](#available-steps), otherwise the role fails before the first step is executed.

### sap_patching_sap_system_sid
- _Type:_ `string`

The SAP System ID (SID) of the patched system, for example `S4H`.<br>
Mandatory for every run, because it is used by `hdbinst` during the HANA client update and it derives the owner of the staging files as `<sap_patching_sap_system_sid>adm`.<br>
Use `sap_patching_hdb_system_sid` or `sap_patching_wdp_system_sid` if the SAP HANA database or the SAP Web Dispatcher use a different SID.

### sap_patching_sapcar_path
- _Type:_ `string`
- _Default:_ `/install`

The path to the directory on managed nodes where the SAPCAR executable is located.<br>
The executable is copied to this directory when `sap_patching_sapcar_on_target` is `false`.

### sap_patching_sapcar_file_name
- _Type:_ `string`
- _Default:_ `SAPCAR.EXE`

The name of the SAPCAR executable file.<br>
The file must be executable by the user owning the patching files.

### sap_patching_sapcar_on_target
- _Type:_ `boolean`
- _Default:_ `true`

Whether the SAPCAR executable is already present on the managed nodes.<br>
Set it to `false` to copy the executable from `sap_patching_sapcar_path_local` on the control node to `sap_patching_sapcar_path` on every host in the play, before the first step of the execution plan is executed.<br>
The copy is executed only when the execution plan contains a step which extracts SAR files, which are `abap_kernel_prepare`, `hdb_client_prepare`, `hdb_server_prepare`, `host_agent_prepare` and `webdisp_prepare`.<br>
The copied file is owned by the user and the group of the staging files and it is throttled to one host at a time, unless `sap_patching_disable_throttle` is set to `true`.

### sap_patching_sapcar_path_local
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on the control node where the SAPCAR executable is located.<br>
Mandatory when `sap_patching_sapcar_on_target` is `false`, otherwise it is ignored.

### sap_patching_file_owner
- _Type:_ `string`
- _Default:_ `''`

The name of the operating system user which owns all extracted files.<br>
An empty value means that the user is derived as `<sap_patching_sap_system_sid>adm`.<br>
Set this variable to override it, for example to a shared user owning an NFS share.

> **NOTE:** The user must exist on every host in the play. See [File ownership](#file-ownership) for the limitation of a single owner per run.

### sap_patching_file_group
- _Type:_ `string`
- _Default:_ `sapsys`

The name of the operating system group which owns all extracted files.

### sap_patching_kernel_base
- _Type:_ `string`
- _Default:_ `''`

The path to the base kernel directory where version specific directories are created during the prepare step, for example `/install/kernel`.<br>
If `sap_patching_kernel_files_shared` is `true`, this path should point to a shared filesystem accessible by all hosts.

### sap_patching_kernel_source
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on managed nodes where the kernel SAR files are located, for example `/install/kernel/source`.<br>
The files are moved to `sap_patching_kernel_base/<kernel_version>` during the prepare step, which leaves this directory empty.

> **NOTE:** This directory must contain **only** kernel SAR and info files. See [Source Patching Files](#source-patching-files).

### sap_patching_kernel_source_local
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on the control node (localhost) where the kernel SAR files are located.<br>
Used only when `sap_patching_kernel_files_on_target` is `false`, when the files are copied from this path to `sap_patching_kernel_source` on the managed nodes during the prepare step.

### sap_patching_kernel_prepare_version
- _Type:_ `string`
- _Default:_ `autodetect`

The kernel release version to use during the prepare step.<br>
Version options:

- `autodetect` - Detect the version from the extracted `disp+work` file.
- `<NNN>` - An exact kernel version, for example `793`. The kernel patch is determined from the name of the SAR files.

### sap_patching_kernel_to_apply
- _Type:_ `string`
- _Default:_ `''`

The name of the directory relative to `sap_patching_kernel_base`, or an absolute path to the kernel applied by the step `abap_kernel_apply`.<br>
Applied only when the step `abap_kernel_prepare` was not executed in the same run, because a freshly prepared kernel always takes precedence over this variable.<br>
If it is not set and prepare was not executed, it falls back to `sap_patching_kernel_to_apply_symlink`.

### sap_patching_kernel_files_on_target
- _Type:_ `bool`
- _Default:_ `true`

Select whether the kernel SAR files are already present on the managed nodes.<br>
Options:

- `true` - The files are already on the managed nodes in `sap_patching_kernel_source` and the prepare step verifies and extracts them in place.
- `false` - The files are on the control node in `sap_patching_kernel_source_local` and the prepare step copies them to the managed nodes.

### sap_patching_kernel_files_shared
- _Type:_ `bool`
- _Default:_ `true`

Select whether the kernel staging area `sap_patching_kernel_base` is shared among multiple hosts.<br>
Options:

- `true` - The files are copied and extracted only once, because the directory is shared.
- `false` - The files are copied and extracted on all target hosts separately.

> **NOTE:** An existing shared filesystem accessible by all hosts is expected if `true`.

### sap_patching_kernel_restart_strategy
- _Type:_ `string`
- _Default:_ `rks`

The restart strategy used by the step `abap_kernel_restart`.<br>
Strategy options:

- `all` - Restart the whole system, including SapStartService.
- `rks` - Restart using Rolling Kernel Switch (RKS).
- `bootstrap` - Restart only `sapstartsrv` and run `sapcpe`. Use this when a kernel update is combined with a HANA client update and the system should be restarted only once.

> **NOTE:** Ensure that your SAP System is compatible with RKS and adjust the SAP profile parameters accordingly. If it is not, use the `all` strategy.

### sap_patching_kernel_prepare_append_to_version
- _Type:_ `string`
- _Default:_ `''`

The kernel release and patch to append to, in the format `<kernel_version>p<patch_number>`, for example `793p320`.<br>
Used when files other than `dw` or `SAPEXE` are patched, for example when `R3trans` is added to an existing kernel patch. See [Append-to mode](#preparation-modes).

### sap_patching_hdb_client_base
- _Type:_ `string`
- _Default:_ `''`

The path to the base HANA DB client directory where version specific directories are created during the prepare step, for example `/install/hana/client`.<br>
If `sap_patching_hdb_client_files_shared` is `true`, this path should point to a shared filesystem accessible by all hosts.

### sap_patching_hdb_client_source
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on managed nodes where the HANA DB client SAR files are located, for example `/install/hana/client/source`.

### sap_patching_hdb_client_source_local
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on the control node (localhost) where the HANA DB client SAR files are located.<br>
Used only when `sap_patching_hdb_client_files_on_target` is `false`, when the files are copied from this path to `sap_patching_hdb_client_source` on the managed nodes during the prepare step.

### sap_patching_hdb_client_to_apply
- _Type:_ `string`
- _Default:_ `''`

The name of the directory relative to `sap_patching_hdb_client_base`, or an absolute path to the HANA DB client applied by the step `hdb_client_apply`.<br>
Applied only when the step `hdb_client_prepare` was not executed in the same run, because a freshly prepared client always takes precedence over this variable.<br>
If it is not set and prepare was not executed, it falls back to `sap_patching_hdb_client_to_apply_symlink`.

### sap_patching_hdb_client_install_path
- _Type:_ `string`
- _Default:_ `/usr/sap/<sap_patching_sap_system_sid>/hdbclient`

The path to the HANA DB client directory updated for an individual installation.<br>
Sets the `--path` parameter of the `hdbinst` command.

> **NOTE:** Mutually exclusive with `sap_patching_hdb_client_shared_path`. Because this variable is set by default, it has to be set to an empty string when a shared installation is updated, otherwise the role fails.

### sap_patching_hdb_client_shared_path
- _Type:_ `string`

The path to the HANA DB client directory updated for a shared installation.<br>
Sets the `--sapmnt` parameter of the `hdbinst` command, for example `/usr/sap/hdbclient`.<br>
When it is set, `hdbinst` is executed only once, on a single host.

> **NOTE:** Mutually exclusive with `sap_patching_hdb_client_install_path`. Exactly one of the two must be set to a non-empty value.

### sap_patching_hdb_client_files_on_target
- _Type:_ `bool`
- _Default:_ `true`

Select whether the HANA DB client SAR files are already present on the managed nodes.<br>
Options:

- `true` - The files are already on the managed nodes in `sap_patching_hdb_client_source` and the prepare step verifies and extracts them in place.
- `false` - The files are on the control node in `sap_patching_hdb_client_source_local` and the prepare step copies them to the managed nodes.

### sap_patching_hdb_client_files_shared
- _Type:_ `bool`
- _Default:_ `true`

Select whether the HANA DB client staging area `sap_patching_hdb_client_base` is shared among multiple hosts.<br>
It describes only the staging area, not the installation itself, which is controlled by `sap_patching_hdb_client_install_path` and `sap_patching_hdb_client_shared_path`.<br>
Options:

- `true` - The files are copied and extracted only once, because the base directory is shared.
- `false` - The files are copied and extracted on all target hosts separately.

Directory and permission checks are always executed on every host, regardless of this variable.

### sap_patching_hdb_system_sid
- _Type:_ `string`

The SAP HANA Database System ID (SID), for example `HDB`.<br>
Set it only if the database uses a different SID than `sap_patching_sap_system_sid`.<br>
Not used for HANA DB client patching.

### sap_patching_hdb_server_base
- _Type:_ `string`
- _Default:_ `''`

The path to the base HANA DB server directory where version specific directories are created during the prepare step, for example `/install/hana/server`.<br>
If `sap_patching_hdb_server_files_shared` is `true`, this path should point to a shared filesystem accessible by all hosts.

### sap_patching_hdb_server_source
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on managed nodes where the HANA DB server SAR files are located, for example `/install/hana/server/source`.

### sap_patching_hdb_server_source_local
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on the control node (localhost) where the HANA DB server SAR files are located.<br>
Used only when `sap_patching_hdb_server_files_on_target` is `false`, when the files are copied from this path to `sap_patching_hdb_server_source` on the managed nodes during the prepare step.

### sap_patching_hdb_server_to_apply
- _Type:_ `string`
- _Default:_ `''`

The name of the directory relative to `sap_patching_hdb_server_base`, or an absolute path to the HANA DB server applied by the step `hdb_server_apply`.<br>
If it is not set, the step `hdb_server_prepare` populates the correct directory, or it falls back to `sap_patching_hdb_server_to_apply_symlink`.

### sap_patching_hdb_server_install_path
- _Type:_ `string`
- _Default:_ `/usr/sap/<sap_patching_hdb_system_sid>/SYS/exe/hdb`

The path to the existing HANA DB server installation, used to check that installation.<br>
It falls back to `sap_patching_sap_system_sid` when `sap_patching_hdb_system_sid` is not set.<br>
There is no need to set this variable unless the installation is non-standard.

### sap_patching_hdb_server_activate_system_user
- _Type:_ `bool`
- _Default:_ `true`

Select whether the system user (SYSTEM) is activated prior to patching.

### sap_patching_hdb_server_activate_admin_userstore_key
- _Type:_ `string`

The name of the SAP HANA user store key (`hdbuserstore`) used when the SYSTEM user is activated.<br>
It must be a key with permissions to activate the SYSTEM user, but not the SYSTEM user itself.<br>
If it is not set, `sap_patching_hdb_server_activate_admin_user` and `sap_patching_hdb_server_activate_admin_password` are used instead.

### sap_patching_hdb_server_activate_admin_user
- _Type:_ `string`

The name of the SAP HANA admin user used when the SYSTEM user is activated.<br>
Required when `sap_patching_hdb_server_activate_system_user` is `true` and `sap_patching_hdb_server_activate_admin_userstore_key` is not set.

### sap_patching_hdb_server_activate_admin_password
- _Type:_ `string`

The password of `sap_patching_hdb_server_activate_admin_user`.<br>
Required when `sap_patching_hdb_server_activate_system_user` is `true` and `sap_patching_hdb_server_activate_admin_userstore_key` is not set.

### sap_patching_hdb_server_is_clustered
- _Type:_ `bool`
- _Default:_ `false`

Select whether the database is clustered, so that this role manages the cluster maintenance during patching.<br>
Cluster steps are available for Red Hat (`pcs`) and SUSE (`crmsh`).

### sap_patching_hdb_server_cluster_saphana_clone_name
- _Type:_ `string`
- _Default:_ `''`

The name of the cluster resource for the HANA DB server managed by this role, used to relocate the resource and to put it in maintenance mode during patching.<br>
Required when `sap_patching_hdb_server_is_clustered` is `true`.<br>
Examples for SID `H01` and instance number `90`:

- Clone for the `SAPHanaController` resource agent in a SAPHanaSR-angi cluster.
  - SUSE: `mst_SAPHanaCon_H01_HDB90`
  - Red Hat: `cln_SAPHanaCon_H01_HDB90`
- Clone for the `SAPHana` resource agent in a SAPHanaSR classic cluster.
  - SUSE: `mls_SAPHana_H01_HDB90`
  - Red Hat: `cln_SAPHana_H01_HDB90`

> **NOTE:** No default value is provided, because this role does not have an instance number variable from which the name could be composed.

### sap_patching_hdb_server_cluster_relocate_timeout
- _Type:_ `int`
- _Default:_ `1800`

The time in seconds to wait for the takeover after the cluster resource was relocated.<br>
The cluster commands for the relocation do not wait for its completion, and the takeover stops and starts the database, so the required time depends on the size of the database.<br>
The state of the replication is unchanged until the takeover happens.

### sap_patching_hdb_server_files_on_target
- _Type:_ `bool`
- _Default:_ `true`

Select whether the HANA DB server SAR files are already present on the managed nodes.<br>
Options:

- `true` - The files are already on the managed nodes in `sap_patching_hdb_server_source` and the prepare step verifies and extracts them in place.
- `false` - The files are on the control node in `sap_patching_hdb_server_source_local` and the prepare step copies them to the managed nodes.

### sap_patching_hdb_server_files_shared
- _Type:_ `bool`
- _Default:_ `true`

Select whether the HANA DB server staging area `sap_patching_hdb_server_base` is shared among multiple hosts.<br>
Options:

- `true` - The files are copied and extracted only once, because the directory is shared.
- `false` - The files are copied and extracted on all target hosts separately.

> **NOTE:** An existing shared filesystem accessible by all hosts is expected if `true`.

### sap_patching_hdb_server_cf_system_user
- _Type:_ `string`

The name of the database user used for patching.<br>
This is a parameter of the `hdblcm` configuration file template. See [HDBLCM configuration file](#hdblcm-configuration-file).<br>
Read the [SAP documentation](https://help.sap.com/docs/SAP_HANA_PLATFORM/2c1988d620e04368aa4103bf26f17727/df3de8c31cef45c0847d2804b97604ea.html) to understand the implications, for example that the XSA update does not work with this option.

### sap_patching_hdb_server_cf_system_user_password
- _Type:_ `string`

The password of the database user defined in `sap_patching_hdb_server_cf_system_user`.<br>
This is a parameter of the `hdblcm` configuration file template.

### sap_patching_hdb_server_cf_&lt;parameter&gt;
- _Type:_ `string`

Any variable with the prefix `sap_patching_hdb_server_cf_` replaces the value of the matching parameter in the generated `hdblcm` configuration file.<br>
See [HDBLCM configuration file](#hdblcm-configuration-file).

### sap_patching_host_agent_base
- _Type:_ `string`
- _Default:_ `''`

The path to the base SAP Host Agent directory where version specific directories are created during the prepare step, for example `/install/host_agent`.<br>
If `sap_patching_host_agent_files_shared` is `true`, this path should point to a shared filesystem accessible by all hosts.

### sap_patching_host_agent_source
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on managed nodes where the SAP Host Agent SAR files are located, for example `/install/host_agent/source`.

### sap_patching_host_agent_source_local
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on the control node (localhost) where the SAP Host Agent SAR files are located.<br>
Used only when `sap_patching_host_agent_files_on_target` is `false`, when the files are copied from this path to `sap_patching_host_agent_source` on the managed nodes during the prepare step.

### sap_patching_host_agent_to_apply
- _Type:_ `string`
- _Default:_ `''`

The name of the directory relative to `sap_patching_host_agent_base`, or an absolute path to the SAP Host Agent applied by the steps `host_agent_apply_auto` and `host_agent_apply_manual`.<br>
Applied only when the step `host_agent_prepare` was not executed in the same run, because a freshly prepared host agent always takes precedence over this variable.<br>
If it is not set and prepare was not executed, it falls back to `sap_patching_host_agent_to_apply_symlink`.

### sap_patching_host_agent_autoupdate_path
- _Type:_ `string`
- _Default:_ `''`

The path to the directory where the SAP Host Agent looks for updates when autoupdate is configured, for example `/install/HostAgentUpdateDir`.<br>
This is the `DIR_NEW` parameter of the host profile, see [SAP Note 1473974](https://me.sap.com/notes/1473974).<br>
Used by the step `host_agent_apply_auto`.

### sap_patching_host_agent_files_on_target
- _Type:_ `bool`
- _Default:_ `true`

Select whether the SAP Host Agent SAR files are already present on the managed nodes.<br>
Options:

- `true` - The files are already on the managed nodes in `sap_patching_host_agent_source` and the prepare step verifies and extracts them in place.
- `false` - The files are on the control node in `sap_patching_host_agent_source_local` and the prepare step copies them to the managed nodes.

### sap_patching_host_agent_files_shared
- _Type:_ `bool`
- _Default:_ `true`

Select whether the SAP Host Agent staging area `sap_patching_host_agent_base` is shared among multiple hosts.<br>
It describes only the staging area, because the host agent itself is always installed locally on every host in the play.<br>
Options:

- `true` - The files are copied and extracted only once, because the base directory is shared.
- `false` - The files are copied and extracted on all target hosts separately.

Directory and permission checks are always executed on every host, regardless of this variable.

### sap_patching_wdp_system_sid
- _Type:_ `string`

The SAP Web Dispatcher System ID (SID), for example `WDP`.<br>
Set it only if the Web Dispatcher uses a different SID than `sap_patching_sap_system_sid`.

### sap_patching_webdisp_base
- _Type:_ `string`
- _Default:_ `''`

The path to the base SAP Web Dispatcher directory where version specific directories are created during the prepare step, for example `/install/webdisp`.<br>
If `sap_patching_webdisp_files_shared` is `true`, this path should point to a shared filesystem accessible by all hosts.

### sap_patching_webdisp_source
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on managed nodes where the SAP Web Dispatcher SAR files are located, for example `/install/webdisp/source`.

### sap_patching_webdisp_source_local
- _Type:_ `string`
- _Default:_ `''`

The path to the directory on the control node (localhost) where the SAP Web Dispatcher SAR files are located.<br>
Used only when `sap_patching_webdisp_files_on_target` is `false`, when the files are copied from this path to `sap_patching_webdisp_source` on the managed nodes during the prepare step.

### sap_patching_webdisp_to_apply
- _Type:_ `string`
- _Default:_ `''`

The name of the directory relative to `sap_patching_webdisp_base`, or an absolute path to the SAP Web Dispatcher applied by the step `webdisp_apply`.<br>
Applied only when the step `webdisp_prepare` was not executed in the same run, because a freshly prepared Web Dispatcher always takes precedence over this variable.<br>
If it is not set and prepare was not executed, it falls back to `sap_patching_webdisp_to_apply_symlink`.

### sap_patching_webdisp_files_on_target
- _Type:_ `bool`
- _Default:_ `true`

Select whether the SAP Web Dispatcher SAR files are already present on the managed nodes.<br>
Options:

- `true` - The files are already on the managed nodes in `sap_patching_webdisp_source` and the prepare step verifies and extracts them in place.
- `false` - The files are on the control node in `sap_patching_webdisp_source_local` and the prepare step copies them to the managed nodes.

### sap_patching_webdisp_files_shared
- _Type:_ `bool`
- _Default:_ `true`

Select whether the SAP Web Dispatcher staging area `sap_patching_webdisp_base` is shared among multiple hosts.<br>
Options:

- `true` - The files are copied and extracted only once, because the directory is shared.
- `false` - The files are copied and extracted on all target hosts separately.

> **NOTE:** An existing shared filesystem accessible by all hosts is expected if `true`.

### Optional fine tuning variables
Changing the following variables is generally not recommended, as they are set to optimal values for most use cases.

#### sap_patching_kernel_rks_wait_timeout
- _Type:_ `int`
- _Default:_ `900`

The Rolling Kernel Switch start wait timeout in seconds.<br>
It must be higher than the SAP profile parameter `rdisp/server_startup/max_time`, whose default is 600 seconds.<br>
See the [SAP documentation for RKS](https://help.sap.com/docs/ABAP_PLATFORM_NEW/1ba3197c1aa7489882770103e3a610dc/78cf5bf734c9419e82e7bac87daa7e60.html?locale=en-US).

#### sap_patching_kernel_rks_soft_timeout
- _Type:_ `int`
- _Default:_ `3600`

The Rolling Kernel Switch soft shutdown timeout in seconds.<br>
It must be higher than the SAP profile parameter `rdisp/shutdown/max_time`, whose default is 3600 seconds.

#### sap_patching_kernel_to_apply_symlink
- _Type:_ `string`
- _Default:_ `ansible_to_apply`

The name of the symbolic link in `sap_patching_kernel_base` pointing to the kernel directory used by the patching process.<br>
Used when `sap_patching_kernel_to_apply` is not set and the step `abap_kernel_prepare` was not executed in the same run.

#### sap_patching_hdb_client_to_apply_symlink
- _Type:_ `string`
- _Default:_ `ansible_to_apply`

The name of the symbolic link in `sap_patching_hdb_client_base` pointing to the HANA DB client directory used by the patching process.

#### sap_patching_hdb_server_to_apply_symlink
- _Type:_ `string`
- _Default:_ `ansible_to_apply`

The name of the symbolic link in `sap_patching_hdb_server_base` pointing to the HANA DB server directory used by the patching process.

#### sap_patching_host_agent_to_apply_symlink
- _Type:_ `string`
- _Default:_ `ansible_to_apply`

The name of the symbolic link in `sap_patching_host_agent_base` pointing to the SAP Host Agent directory used by the patching process.

#### sap_patching_webdisp_to_apply_symlink
- _Type:_ `string`
- _Default:_ `ansible_to_apply`

The name of the symbolic link in `sap_patching_webdisp_base` pointing to the SAP Web Dispatcher directory used by the patching process.

#### sap_patching_disable_throttle
- _Type:_ `bool`
- _Default:_ `false`

Select whether the copies from the control node run on all hosts at once.<br>
Relevant for the patching files when the `files_on_target` variables are `false` and the `files_shared` variables are `false`, because that is the only case when the same files are copied from the control node to every host separately.<br>
Relevant for the SAPCAR executable whenever `sap_patching_sapcar_on_target` is `false`.<br>
Options:

- `false` - The copy is throttled to one host at a time, which is slower, but it prevents the control node and the network from being saturated.
- `true` - All hosts are copied to in parallel, which is faster on a fast network with few hosts.
<!-- END Role Variables -->
