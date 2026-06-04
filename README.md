
# 🛡️ Anticimex Ansible Automation – Operations Documentation

This repository contains the Ansible Automation Platform used to operate and maintain RHEL and Ubuntu servers across the following environments:

* PROD

* CFG

* EDU

* UAT


The automation enforces OS baselines, security hardening, kernel lifecycle managejment, agent installation, and logging hygiene in a repeatable and auditable way.

Auditable & Repeatable meaningEvery role prints a role‑banner and structured task output describing exactly what configuration it applies. An operator must be able to determine what policies were enforced on a server only by reading the execution output, without inspecting the code.



## 1. Access to the Ansible Server

### 1.1 Login

The Ansible control node is:

```
axabtansibel
IP: 10.140.22.8
```


Verify access:

```
ssh root@10.140.22.8 -p 22
```

🔐 The root password for the Ansible server is stored in KeePass in ITS.kdbx under **axabtansibel** (used only for first-time key install).

---

### 1.2 SSH Trust Between Ansible → Target Hosts (Mandatory)
Ansible connects from the Ansible server to the managed servers using SSH keys.

This is NOT your workstation key.
This is NOT for logging into Ansible.
This is a machine‑to‑machine trust requirement.

From inside the Ansible server:

```
ssh-keygen -t rsa -b 4096
```

Install the key on the target host:
```
ssh-copy-id root@<target-host>
```
Verification:
```
ssh root@<target-host>
```

If password is requested → Ansible will fail.


### 1.3 Target Host Requirement

The target host must allow public key authentication:
```
/etc/ssh/sshd_config
PubkeyAuthentication yes
```

Restart sshd if changed:
```
systemctl restart sshd
```




##  2. Entering the Ansible Automation Repository

After login to the Ansible server:

```
cd ~/ansible-automation
```

If the directory does not exist or may be outdated, re‑clone:

```
cd ~
rm -rf ansible-automation
git clone https://github.com/axab-iac-org/ansible-automation.git
cd ansible-automation
```


### Repository layout
```
.
├── hosts.ini                     # Inventory (all environments and hosts)
├── setup.yml                     # Main orchestration playbook
└── roles/                        # All reusable roles
    ├── os_settings/
    ├── kernel_cleanup_and_upgrade/
    ├── selinux/
    ├── s1_r7_install/
    └── ...
```

---

## 3. Inventory: Hosts and Environments

### 3.1 List Hosts
```
ansible-inventory -i hosts.ini --list
```


---


###  5. Running Playbooks and Roles
5.1 General Command Pattern
```
ansible-playbook -i hosts.ini setup.yml \
  -e target=<ENV> \
  --tags <ROLE_TAG> \
  --limit <HOSTNAME>
```


### Parameters:
Option	Description
```
-i hosts.ini	Specify the inventory file
-e target=<ENV>	Select environment group (e.g., UAT, PROD, CFG)
--tags <TAG>	Run specific role only
--limit <HOST>	Limit execution to one or more specific hosts
```


Note: The command must be executed from the repository root directory:
Running the playbook from another directory will cause Ansible to not find hosts.ini, roles, or relative file paths.

```
cd ~/ansible-automation
```

### 6. Test a role
Run SELinux role:
```
ansible-playbook -i hosts.ini setup.yml -e target=UAT --tags selinux --limit testvm-rhel
```

Run OS settings role:
```
ansible-playbook -i hosts.ini setup.yml -e target=UAT --tags os_settings --limit testvm-rhel
```

Run SentinelOne & Rapid7 install role:
```
ansible-playbook -i hosts.ini setup.yml -e target=UAT --tags s1_r7_install --limit testvm-ubuntu
```

Kernel cleanup and upgrade
```
ansible-playbook -i hosts.ini setup.yml -e target=UAT --tags kernel_cleanup_and_upgrade --limit testvm-rhel
```



---

# os_settings

**Scope:** Operating system baseline, hardening and operational compliance


This role establishes the Anticimex operating baseline for a server.
After execution the host is considered compliant and eligible for production onboarding.




## Important Variables

| Variable | Purpose | Typical Value |
|----|----|----|
| `enable_firewall` | Enables OS firewall management | true/false |
| `is_k8s_node` | Skip host-level networking changes | true on Kubernetes nodes |
| `hosts_dns_domain` | FQDN suffix | anticimex.local |
| `timesync_ntp_servers` | Time sources | NTP pool |
| `selinux_state` | SELinux mode | permissive/enforcing |
| `fail2ban` | Enable intrusion blocking | true |
| `security_update_calendar` | Monthly patch schedule | `*-*-15 18:00:00` |



## Operational Impact

| Area | Change Type | Service Risk | Reboot Required |
|----|----|----|----|
| SSH configuration | Immediate | Possible disconnect if misconfigured | No |
| Firewall rules | Immediate | Can block application ports | No |
| sysctl kernel parameters | Runtime kernel behavior | Rare but possible networking effects | No |
| Package installation | Adds tools only | Safe | No |
| Time synchronization | Adjusts clock | Sensitive for DB clusters | No |
| Security updates policy | Future behavior only | Safe | No |
| Oracle pre-shutdown unit | Shutdown behavior | DB only | No |
| Fail2ban | Blocks IP after failures | Possible lockout | No |


## Idempotency

This role is fully idempotent.

Running it multiple times will:
- Not reinstall packages unnecessarily
- Not duplicate configuration
- Does not remove manually added rules outside the managed rule set
- Not restart services unless configuration changed

Safe to run repeatedly for drift correction.

## Non-Goals (Safety Guarantees)

This role will never:

- Reboot the server automatically (unless explicitly enabled)
- Upgrade the OS distribution version
- Remove application packages
- Modify database configuration
- Change network interfaces or IP addresses



## Functional Areas

### 1) Identity & Access
Ensures the system can always be managed remotely.

- Adds hostname → IP mapping in `/etc/hosts`
- Configures SSH daemon and ssh_setttings
- Prepares controlled shutdown for Oracle db when user types systemctl reboot on a rhel server.



### 2) Base Operating Environment
Installs the operational tooling required for support and automation.

Includes:
- troubleshooting utilities (network, filesystem, monitoring)
- Python runtime environment for automation
- consistent shell tooling across distributions



### 3) Security Hardening
Applies baseline protections required for production operation.

- SELinux (RHEL) and AppArmor (Ubuntu) are placed in monitoring mode (permissive/complain). The role does not enforce mandatory access control policies.
- Fail2Ban intrusion protection
- Kernel security parameters (`sysctl`)
- Optional firewall configuration

Result:
The server resists common attack patterns and logs suspicious activity.


### 4) Time Synchronization
Enables and configures NTP.

Why this matters:
- certificate validation
- clustered workloads
- Kerberos authentication
- log correlation
- database replication integrity

Result:
No authentication or monitoring failures caused by clock drift.


### 5) Update Strategy (Controlled Patching)
Prevents unexpected outages caused by automatic updates.

- Disables unattended full upgrades
- Enables security updates only
- Prevents automatic reboots

Result:
All updates occur during planned maintenance windows only.


### 6) Service Behavior Controls
Configures operational runtime behavior.

- RDP service schedule (if installed)
- system reboot behavior reporting
- predictable package state


## Special Modes

### Kubernetes Nodes
If `is_k8s_node=true`, the role avoids modifying:
- firewall rules
- network sysctl
- AppArmor

These are controlled by the cluster instead of the OS.


## Reboots
The role  **does not reboot automatically**.

Instead it reports when a reboot is required so operators can schedule it safely.


**Risk level:** Medium
(Security policies and networking behavior may change)


## Operator verification (banner output)

After execution verify:

- SSH still accessible
- Time synchronization active
- Security services running
- No blocked application ports
- No critical warnings reported


## When to run

- Immediately after provisioning a server
- After manual configuration drift
- After OS reinstall
- Before installing monitoring/security agents
- Before handing a host to application teams



---

# selinux

**Scope:** SELinux state management + denial visibility + targeted relabeling (RHEL-family only)

This role is designed to **surface and triage SELinux issues** in a controlled way.
It can set SELinux mode, collect AVC denials, apply *explicit* fcontext fixes for application paths,
and (optionally) generate a **local policy module** from observed denials (Rapid7-focused).

> **OS Scope:** RHEL-family only. The role is skipped on non-RedHat systems.


## Intended Use / Safety Position

**Primary environments:**
- TEST
- UAT
- POC

**Not enabled in PROD by default.**

Reason: SELinux enforcement changes and relabel operations can break workloads if applied without review.


## What the role does

### 1) Guardrails / Scope
- Skips execution unless `ansible_facts.os_family == 'RedHat'`
- Ensures SELinux tooling is present:
  - `policycoreutils`
  - `policycoreutils-python-utils`
  - `libselinux-utils`
  - `setools-console`

### 2) SELinux mode management
- Normalizes:
  - `selinux_state` (default: `enforcing`)
  - `selinux_policy` (default: `targeted`)
- Applies state with `ansible.posix.selinux`
- If a state change requires reboot:
  - sets `os_settings_reboot_required=true`
  - appends a reason to `os_settings_reboot_reasons`
  - **reboots only** if `reboot_when_required=true`

### 3) dbora service handling (Oracle)
- Detects whether legacy SysV exists:
  - `/etc/init.d/dbora`
- Detects whether a native systemd unit exists:
  - `/etc/systemd/system/dbora.service`
- Runs migration tasks **only** if legacy SysV script exists. This fixes the issues we had with /usr/bin/su and xauth denials


## 4) CIFS backup mount SELinux context enforcement (Known limitation + manual workaround)

Targets mounts like:
- `/opt/<hostname>-backup` (pattern: `^/opt/.+-backup$`)
- Only if the mount `fstype == cifs` (from `ansible_mounts`)

Behavior:
- Reads `/etc/fstab` entry for the mount if present
- Ensures mount options include:
  - `context=<SELINUX_CONTEXT>` (default `system_u:object_r:var_t:s0`)
  - `_netdev`
  - `nofail`
- **Does not remount at runtime by default**
  - runtime remount is opt-in via `cifs_backup_apply_runtime=true`
  - reason: CIFS remount with `context=` can be disruptive (EINVAL / credential prompts)

### Current Status (important)
In some environments, the backup mount SELinux context is **not applied automatically yet** (or cannot be safely applied online).

If the mount still shows `cifs_t` after running the role, you must follow the manual guide below.

Quick indicator on the server that manual action is needed:
```
ls -ldZ /opt/<hostname>-backup
```

if you see: system_u:object_r:cifs_t:s0

then a manual fix required
Manual guide (in repo): selinux_context_on_backup.md
https://github.com/axab-iac-org/ansible-automation/blob/main/selinux_context_on_backup.md


### 5) Custom fcontexts (explicit relabel)
- Applies `selinux_fcontexts` via `community.general.sefcontext` (best-effort)
- Runs `restorecon` for provided paths (best-effort)
- Reports:
  - any sefcontext failures
  - which paths actually got relabeled

### 6) Rapid7 handling (SELinux hygiene)
- Shows post-restorecon SELinux labels for key Rapid7 files
  - `restorecon -Rv /opt/rapid7/ir_agent`
  - restart known Rapid7 service names (if present)

### 7) AVC visibility + reporting
Collects AVC denials (DENIED only), preference order:
1. `ausearch` (preferred)
2. `/var/log/audit/audit.log*`
3. journald
4. additional configured log globs

Then:
- builds a unique summary including full paths when possible
- optional inode-based inference for missing paths (capped/timeboxed)
- prints a high-level breakdown of most actionable denied file targets

### 8) Optional local policy module generation (Rapid7-focused)
If enabled, the role can generate and install a local allow module:
- `selinux_build_local_module=true`
- Default module name: `rapid7_local` (overrideable)

Behavior:
- Filters AVCs to Rapid7 path denials (`/opt/rapid7/ir_agent/`)
- Explicitly excludes `init_t` denials (by design)
- Builds with `audit2allow -M`
- Installs with `semodule -i`
- Skips if module is already installed (idempotent)


## Important Variables

| Variable | Purpose | Typical Value |
|---|---|---|
| `selinux_state` | SELinux mode | `permissive` / `enforcing` |
| `selinux_policy` | SELinux policy | `targeted` |
| `reboot_when_required` | Allow role to reboot if required by SELinux state changes | `false` |
| `selinux_avc_lookback` | AVC lookback window | `15m`, `1h`, `boot` |
| `selinux_avc_max_lines` | Cap on collected AVC lines | `5000` |
| `selinux_inode_lookup_max` | Max inode lookups for missing paths | `50` |
| `selinux_find_timeout_sec` | Timeout per inode `find` | `10` |
| `selinux_fcontexts` | Custom fcontext rules (app paths only) | list of `{path,setype}` |
| `selinux_cifs_backup_context` | SELinux label for CIFS backup mount | `system_u:object_r:var_t:s0` |
| `cifs_backup_apply_runtime` | Force disruptive unmount/mount to apply context immediately | `false` |
| `selinux_build_local_module` | Build+install Rapid7 local module from AVCs | `false` |
| `selinux_local_module_name` | Module name to install | `rapid7_local` |


## Operational Impact

| Area | Change Type | Service Risk | Reboot Required |
|---|---|---|---|
| SELinux state/policy | Policy enforcement behavior | Medium–High | Sometimes |
| CIFS fstab mount options | Persistent mount behavior | Medium | No (but remount/reboot needed for effect) |
| restorecon / relabel | Filesystem label changes | Medium | No |
| dbora migration | Service management | Low (Oracle) | No |
| Local policy module install | Kernel policy change | Medium | No |

**Risk level:** Medium–High
(SELinux changes can cause application access issues.)


## Idempotency

- Safe to rerun.
- fcontext rules are applied declaratively (present).
- `restorecon` is best-effort; repeated runs are safe.
- Local policy module install is skipped if already installed.


## Non-Goals (Safety Guarantees)

This role will never:
- Relabel core system binaries as a “fix” (e.g. `/usr/bin/su`, `/usr/bin/xauth`) dbora migrations takes care of this
- Automatically enforce broad policy changes in PROD without explicit operator intent
- Attempt automatic SSH key bootstrapping
- Assume CIFS mounts can be fixed with `restorecon`/`semanage fcontext` alone


## Operator Verification (banner output)

After execution, check the output for:
- SELinux state/policy applied (and reboot requirement)
- CIFS backup mount detection + desired context + fstab update status
- Which custom paths were relabeled
- AVC summary counts + breakdown
- Rapid7 label output and optional module generation

---


# kernel_cleanup_and_upgrade

**Scope:** Kernel lifecycle maintenance, `/boot` hygiene, and controlled upgrade execution (RHEL-family only)

This role is intended for RHEL-family systems (incl. Oracle Linux) and performs:

- Safe kernel retention and cleanup (never removes the running kernel)
- `/boot` free-space gating and cleanup
- Disables unattended DNF updates that can conflict with controlled patching
- Optionally kicks off `dnf upgrade --refresh` inside a persistent `tmux` session


## Prerequisites

The host must have:

- RHEL-family OS with DNF available
- Working package repositories
- Sufficient disk space in `/var` and `/boot` for kernel packages
- SSH connectivity stable (upgrade runs in tmux if SSH drops)


## Important Variables

| Variable | Purpose | Typical Value |
|---|---|---|
| `installonly_limit_value` | Kernel retention cap via `installonly_limit` in `/etc/dnf/dnf.conf` | `2` |
| `boot_min_free_mb` | Minimum `/boot` free space required to *run upgrade phase* | `360` |
| `cleanup_only` | If `true`, perform cleanup only and never start upgrade | `false` |
| `kernel_stream` | Optional preference (`uek` to standardize on UEK where applicable) | unset / `uek` |
| `dnf_automatic_remove_pkg` | If `true`, remove the dnf-automatic package (otherwise just disable) | `false` |


## Operational Impact

| Area | Change Type | Service Risk | Reboot Required |
|---|---|---|---|
| Kernel package removal | Immediate | Low (guarded; running kernel preserved) | No |
| `/boot` file cleanup | Immediate | Low | No |
| DNF unattended updates | Immediate | Low | No |
| DNF upgrade execution | Controlled, long-running | Medium (patch-level changes) | Usually after upgrade |
| tmux session creation | Immediate | None | No |

**Operational Risk:** Medium
(System packages are modified; upgrades can affect services and require reboot planning.)


## Idempotency

This role is safe to rerun.

- Cleanup operations will not repeat unnecessarily
- Kernel removal logic is guarded by safety assertions
- If kernels are already within retention limits, no changes occur
- `dnf-automatic` disable logic remains consistent across reruns


## Non-Goals (Safety Guarantees)

This role will never:

- Run on Debian/Ubuntu (explicit fail-fast)
- Remove the running kernel
- Remove all kernels of a given stream (RHCK/UEK)
- Perform reboot automatically (unless you add reboot logic externally)
- Modify application configuration


## Role Behavior

### 1) Guardrails

The role aborts when:

- OS is Debian/Ubuntu (`ID` or `ID_LIKE=debian`)
- `VERSION_ID` matches `9.6` and `cleanup_only=false` (prevents repeated upgrade attempts)

### 2) Unattended update control

- Ensures `apply_updates=no` in `/etc/dnf/automatic.conf`
- Stops/disables/masks dnf-automatic timers/services
- Removes any related systemd drop-ins
- Reloads systemd daemon

### 3) `/boot` gating + cleanup

- Measures `/boot` free space
- If `/boot` free space is below `boot_min_free_mb`:
  - Performs kernel pruning/cleanup
  - Ends the host run after cleanup (upgrade does not start in the same run)

### 4) Kernel retention policy

- Uses DNF oldinstallonly pruning and/or explicit per-stream cleanup
- Keeps:
  - newest kernel per stream (RHCK/UEK)
  - running kernel
- Never removes running kernel
- Asserts at least one kernel per present stream remains

### 5) Controlled upgrade execution (when allowed)

If `/boot` gate is satisfied and `cleanup_only=false`:

- Ensures `tmux` is installed
- Creates a `tmux` session named `anticimex`
- Executes: `dnf upgrade -y --refresh` inside the session
- Waits for DNF/RPM locks to clear
- Leaves execution durable even if SSH disconnects

To attach a running kernel upgrade:

```
tmux attach -t anticimex
```



---


# s1_r7_install

**Scope:** Endpoint security monitoring onboarding (EDR + telemetry)

This role installs and enrolls the host into the organization security platforms:

- **SentinelOne** (EDR protection)
- **Rapid7 Insight Agent** (visibility & vulnerability telemetry)

The role stages the correct package for the OS, installs it, registers the host using provided tokens, and verifies service status.


## Prerequisites

The host must have:

- Working DNS resolution
- Correct system time (NTP/chrony recommended)
- Outbound connectivity to Rapid7 and SentinelOne endpoints (per network/security policy)
- `ca-certificates` installed (TLS trust)
- systemd available and running
- Sufficient disk space in `/var/tmp` and `/opt`
- A functional package manager (`dnf`/`yum` or `apt`/`dpkg`)


## Important Variables

| Variable | Purpose | Typical Value |
|---|---|---|
| `rapid7_token` | Registers host in Rapid7 tenant | from `R7_TOKEN` |
| `sentinelone_site_token` | Registers host in SentinelOne console | from `S1_TOKEN` |
| `r7_pkg_src` | Path to Rapid7 installer used by the playbook | `./rapid7-insight-agent-*.rpm/.deb` |
| `s1_pkg_src` | Path to SentinelOne installer used by the playbook | `./SentinelAgent_*.rpm/.deb` |


## Operational Impact

| Area | Change Type | Service Risk | Reboot Required |
|---|---|---|---|
| Agent packages installed | Immediate | Low | No |
| New services enabled/started | Immediate | Low–Medium | No |
| Outbound network traffic | Continuous | Medium (proxy/firewall/DNS can block check-in) | No |
| Files under `/opt` (vendor-managed) | Persistent | Low | No |
| Security posture | Increases | Low | No |

**Operational Risk:** Medium
(Enrollment depends on valid tokens and outbound network reachability.)


## Idempotency

This role is safe to rerun.

- If an agent is already installed and enrolled, the role should not re-enroll unnecessarily.
- If an agent is installed but not enrolled, the role attempts enrollment again.
- Services are ensured running after execution.


## Non-Goals (Safety Guarantees)

This role will never:

- Change firewall rules or open ports
- Modify SSH configuration
- Modify SELinux/AppArmor policies
- Upgrade the OS
- Reboot the server automatically
- Print or log enrollment tokens


## What the role does

### 1) Platform detection
- Automatically selects RPM or DEB installer based on OS
- Skips Rapid7 on Oracle Linux with UEK kernel (unsupported combination)

### 2) Rapid7 agent
- Stages installer to `/var/tmp`
- Installs package
- Registers agent using provided token (only if not already enrolled)
- Enables and starts `ir_agent.service`
- Removes installer after completion

### 3) SentinelOne agent
- Stages installer to `/var/tmp`
- Installs package
- Applies site token using `sentinelctl`
- Starts agent via CLI and systemd service
- Removes installer after completion

### 4) Observability
- Shows registration return codes (without leaking tokens)
- Displays service status for both agents
- Confirms the role is safe to rerun when already enrolled


## Required Inputs

### 1) Export tokens before running the playbook

```
export R7_TOKEN="..."
export S1_TOKEN="..."
```

```
Variable	             Purpose
rapid7_token	         Registers host in Rapid7 tenant
sentinelone_site_token	 Registers host in SentinelOne console
```

*** Installer files must exist in the repository ***

```
roles/s1_r7_install/files/
├── rapid7-insight-agent_4.0.13.32-1_amd64.deb
├── rapid7-insight-agent-4.0.13.32-1.x86_64.rpm
├── SentinelAgent_linux_x86_64_v24_3_1_29.deb
└── SentinelAgent_linux_x86_64_v24_3_1_29.rpm
```

If these files are missing, the playbook fails during the staging step.



Operator should verify in banner
   * Agents installed
   * Services running
   * No enrollment errors
   * Host visible in security consoles


When to run
   * New server onboarding
   * Reinstall missing agents
   * Security visibility gap
   * After rebuilding a system
