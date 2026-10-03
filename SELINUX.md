In general, **SELinux needs a specific directory path because it cannot protect files and directories without labeling them.**  
Here is the general explanation of why data\_dir is required when SELinux is active:

1. **Process vs. File Isolation (Mandatory Access Control)**: Under SELinux, a process (like the database engine) runs in a restricted context or domain (e.g., postgresql\_t). The process is only allowed to access files that are tagged with a specific file context type (specifically postgresql\_db\_t).  
2. **Explicit Path Mapping**: To tell the Linux kernel, *"This directory belongs to the database and should be allowed,"* you must register the specific path in the SELinux policy database using semanage fcontext:  
3. bash  
4.  semanage fcontext \-a \-t postgresql\_db\_t "/my/custom/data/path(/.\*)?"  
5.    
6. If you do not specify a target directory path (i.e., data\_dir is empty), SELinux has no pattern to register.  
7. **Label Application (Relabeling)**: After the path is registered, the system must run restorecon recursively on that exact path to apply the postgresql\_db\_t label to the physical files and directories. You cannot run restorecon without a target location.

**Why setupplaybook have selinux policies:**

What are these tasks doing?  
**1\. Labeling the Mount Point (mnt\_t)**  
yaml  
         `- name: Add SELinux fcontext for {{ data_dir... }}`  
           `setype: mnt_t`  
   
This registers mapping rules and runs restorecon to tag the parent disk mount directory (like /mnt/disks/) with the type mnt\_t (mount type).  
**2\. Compiling Manual Custom Policies (semodule \-i)**  
yaml  
         `- name: Create SELinux CIL file for mnt_t write access`  
           `content: "(allow postgresql_t mnt_t (dir (write)))..."`  
   
Because standard default Linux policies don't allow a database to write files directly under disk mount points, this block writes raw SELinux rule files (.cil files) and compiles them into the kernel via semodule \-i.

* alloydb\_mnt\_t\_policy.cil: Grants the database permission to write to /mnt.  
* kdump\_alloydb.cil: Grants the database permission to write to crash dumps.

**3\. Database Directory Override (postgresql\_db\_t)**  
yaml  
         `- name: Add SELinux fcontext for {{ data_dir }} directory`  
           `setype: postgresql_db_t`  
   
This is a manual fallback to force postgresql\_db\_t on the main data directory. As the comment states, this was only added as a temporary redundancy because the RPM installer wasn't relabeling the folders properly at the time.

**Problems faced for running a single test:**

Here is a one-line summary of each problem we faced and resolved during this effort:

1. **SELinux Forced Permissive**: Common setup execution in   
2. prerequisites.yml explicitly called setenforce 0 on startup, forcing SELinux into permissive mode on all VMs.  
3. **Installer Safety Check Fail**: The Ansible installer role defaulted to disable\_selinux: true which triggered a critical pre-check failure on the enforcing controller host.  
4. **Missing Dictionary Variable Scope**: The standalone   
5. prepare.yml playbook defined data\_dir at the top level but lacked it inside the alloydbomni dict, crashing the newly active SELinux pre-checks.

**Failed tests:**

1. Scalable\_ha\_ansible\_connection\_pooler\_test:  
   1.   
2. scalable \_ha\_ansible\_bootstrap\_test:  
   1.   
3. scalable\_ha\_ctl\_apply\_test:  
   

# **Conceptual Study Guide: Understanding SELinux (Security-Enhanced Linux)**

If you need to discuss or present SELinux tomorrow to your team or manager, this guide explains all the core concepts, terminologies, and real-world mechanics using simple, everyday analogies and basic language.

---

## **1\. What is SELinux? (The Security Guard Analogy)**

In standard Linux, security is Discretionary (DAC)—meaning if you own a file, you decide who gets permission to read or write to it (using chmod or chown). If a hacker gains access to the root user or runs a process as root, they can access *any* file on the system.

SELinux (Security-Enhanced Linux) changes this by adding Mandatory Access Control (MAC).

### The Analogy:

Think of a secure building:

* Standard Linux (DAC): If you have the master key (run as root), you can unlock every door, go into the server room, read private vaults, or change the security codes.  
* SELinux (MAC): Every person (Process) and every room (File/Port/Directory) has a specific Security Badge (Label).  
  * Even if the janitor has a master key (root privileges), their badge only lists access to "Hallways" and "Utility Closets". If they try to open the "Classified Server Room", the security guard (SELinux in the Kernel) blocks them immediately because their badge label does not allow entry.

---

## **2\. Core Concepts & Terminology**

SELinux checks three things before allowing any action: Subject, Object, and Action.

* Subject (Process): A running program on the OS (e.g., postgresql server process).  
* Object: The resource the program wants to access (e.g., a file, directory, or network port).  
* Action: What the program is trying to do (e.g., read, write, execute, or bind to a port).

### SELinux Labels (Contexts)

To make decisions, SELinux tags every Subject and Object with a label. You can view these labels by running standard commands with the \-Z flag (e.g., ls \-Z or ps \-eZ).

A label looks like this: system\_u:object\_r:postgresql\_db\_t:s0

Only the third part—the Type (ending in \_t)—matters for most configurations (this is called Type Enforcement):

* Process Domain Typings: The database process runs in the postgresql\_t domain.  
* File Context Typings: Database data folders are tagged with the postgresql\_db\_t file type.  
* The kernel reads the policy database, which says: *"Only processes running in the postgresql\_t domain are allowed to write to directories labeled postgresql\_db\_t."*

---

## **3\. SELinux Modes**

SELinux operates in one of three modes:

| Mode | What it does | Real-World equivalent |
| :---- | :---- | :---- |
| Enforcing | Blocks unauthorized actions and logs the details. | Security guard blocks the intruder and writes a report. |
| Permissive | Allows unauthorized actions to happen, but still logs them. | Security guard lets the intruder pass but writes down a warning report (used for testing/debugging). |
| Disabled | SELinux is completely off; no policies are checked or logged. | Security guard is gone. |

---

## **4\. Key Everyday Commands to Know**

Here are the basic commands you run on a Linux server to manage SELinux:

* Check Status: sestatus reports if SELinux is active and whether it is in Enforcing or Permissive mode.  
* Check/Change Mode at Runtime:  
  * getenforce: prints current mode (Enforcing or Permissive).  
  * setenforce 1: switches mode to Enforcing.  
  * setenforce 0: switches mode to Permissive.  
* Verify Labels:  
  * Files: ls \-Z /var/lib/alloydb  
  * Processes: ps \-eZ | grep postgres  
* Define Target Labels (Persistent Rules): When we want to mk  
  * semanage fcontext \-a \-t postgresql\_db\_t "/data(/.\*)?" tells the kernel: *"From now on, any file created under /data should default to the postgresql\_db\_t label."*  
* Apply Labels (Apply persistent rules to disk):  
  * restorecon \-R \-F /data recursively triggers the file system to read the rules and apply the labels onto the files.

---

## **5\. What is an AVC Denial? (Debugging SELinux)**

When a process is blocked by SELinux, the check failure is called an AVC (Access Vector Cache) Denial.

* Where is it logged? Inside /var/log/audit/audit.log.  
* Example log entry:

```
type=AVC msg=audit(1625828456.123:456): avc: denied { write } for pid=789 comm="postgres" name="data" dev="sda1" ino=9876 scontext=system_u:system_r:postgresql_t:s0 tcontext=unlabeled_t tclass=dir
```

* This log tells you:  
  * Action: denied { write }  
  * Who: comm="postgres" running in domain type postgresql\_t  
  * Target: Directory name="data" labeled as unlabeled\_t  
  * The Problem: The database could not write data because the directory was not labeled correctly\! Relabeling it to postgresql\_db\_t fixes the issue.

More details:

### **1\. The "People" are OS Processes, not Database Users**

When you install and start AlloyDB Omni, the RHEL 9 operating system starts a background process (a running program).

* In standard Linux terms, this process might run under a system user account named `postgres` or `alloydbomni`.  
* In **SELinux terms**, the moment that program starts executing, the kernel forces it to wear a specific security badge (a **Domain**).

For AlloyDB Omni, that process badge will usually look something like `postgresql_t`. SELinux doesn't care if you log into the database as `john_doe` or `bobby_tables`; it only sees one massive background program running on the processor under the `postgresql_t` badge.

### **2\. The "Rooms" are the Physical Files on RHEL 9**

When the AlloyDB Omni RPM is installed, it creates physical directories on your RHEL 9 hard drive to store configuration, logs, and actual database blocks (for example, under `/var/lib/alloydb` or `/mnt/disks/pgsql`).

SELinux goes to those folders and stamps a physical badge (a **File Context**) onto them—usually `postgresql_db_t`.

### **How the Guard Approves it:**

The security guard (the Linux Kernel) looks at its master rulebook and checks the match:

* **The Person (Process):** A program running with the `postgresql_t` badge.  
* **The Room (Object):** A folder on disk labeled `postgresql_db_t`.  
* **The Rule:** *"Programs wearing `postgresql_t` are allowed to read/write to folders labeled `postgresql_db_t`."*  
* **Result:** Access Granted.

### **What happens if you try to change the folder?**

Let's say you decide to change your AlloyDB Omni data storage directory to a brand new folder you just made called `/my_custom_storage`.

By default, RHEL 9 will give that new folder a generic badge (like `default_t` or `unlabeled_t`). The next time AlloyDB Omni starts up and tries to write its database files there, the security guard stops it:

> *"Hold on. Your process badge is `postgresql_t`, but this room's badge is `default_t`. I don't have a rule that allows those two colors to mix. Blocked\!"*

This is exactly why an administrator has to step in with those `semanage` and `restorecon` commands—to explicitly paint that new folder with the correct `postgresql_db_t` badge so the security guard allows the program inside.

# **Implementation Proposal: Enable & Clean Up SELinux in VM Integration Tests**

This document presents the detailed execution plan for implementing the decisions from the meeting regarding SELinux enablement in CTL (Control Language) tests and cleaning up legacy policy tasks from the Ansible playbooks.

---

## 1\. Goal Overview

1. **Enable SELinux Enforcement on VMs**: Let VMs keep SELinux in enforcing mode by default during integration tests.  
2. **Enable SELinux in Deployment Specs**: Configure disable\_selinux: false across all orchestrator/CTL test deployment spec configurations.  
3. **Ansible Clean-up**: Remove redundant/legacy SELinux policy tasks (e.g., CIL file creation, system label updates) from all integration test setup and native playbooks, delegating policy application to the node/cluster manager.  
4. **Prioritize CTL Tests**: Verify the standalone CTL test first (standalone\_ctl\_apply\_test.go) before enabling it for other suites.

---

## 2\. Detailed Technical Changes

### A. Turn on SELinux by Default in Tests

We will edit prerequisites.yml to remove the task that disables SELinux:

```
-# TODO(mehtaaman): Remove this once New RPM rule supports SELinux.
-- name: Disable SELinux enforcement
-  ansible.builtin.command: setenforce 0
-  changed_when: false
```

---

### B. Configure disable\_selinux: false in Deployment Specs

We will configure disable\_selinux: false in the alloydbomni.vars section of all deployment templates and generated specs so the installer executes SELinux policy tasks.

Affected files:

1. **deployment\_spec\_standalone.yml**:

```
 alloydbomni:
   vars:
     disable_gpgcheck: "{{ gpg_check_enabled | bool }}"
+    disable_selinux: false
```

2. **deployment\_spec\_scalable\_ha.yml**:

```
 alloydbomni:
   vars:
     disable_gpgcheck: "{{ gpg_check_enabled | bool }}"
+    disable_selinux: false
```

3. **deployment\_spec\_resilient\_ha.yml**:

```
 alloydbomni:
   vars:
     disable_gpgcheck: "{{ gpg_check_enabled | bool }}"
+    disable_selinux: false
```

4. **deployment\_spec\_sep\_cm\_db\_multi\_cluster.yml**:

```
 alloydbomni:
   vars:
     disable_gpgcheck: "{{ gpg_check_enabled | bool }}"
+    disable_selinux: false
```

5. **deployment\_spec\_scalable\_ha\_disable\_pgbackrest.yml**:

```
 alloydbomni:
   vars:
     disable_gpgcheck: "{{ gpg_check_enabled | bool }}"
+    disable_selinux: false
```

6. **deployment\_spec\_scalable\_ha\_readpool.yml**:

```
 alloydbomni:
   vars:
     disable_gpgcheck: "{{ gpg_check_enabled | bool }}"
+    disable_selinux: false
```

7. **deployment\_spec\_sep\_etcd\_cm\_multi\_cluster-1.yml** and **\-2.yml**:

```
 alloydbomni:
   vars:
     disable_gpgcheck: "{{ gpg_check_enabled | bool }}"
+    disable_selinux: false
```

8. **prepare.yml (Standalone Ctl)**:

```
     content: |
       alloydbomni:
         vars:
+          disable_selinux: false
           cluster_manager:
             name: "{{ cluster_name }}"
```

---

### C. Remove Redundant SELinux Playbook Tasks

We will clean up tasks that manually install policy utils, write CIL policy description files, compile policy modules, or force restorecon commands because management is now handled natively by the RPM/node manager.

#### 1\. Clean setup\_database\_node.yaml (Lines 56-110)

Remove the manual dependency checks, CIL creations, and folder relabel commands:

```
-        - name: Install SELinux dependencies
-          ansible.builtin.yum:
-            name: policycoreutils-python-utils
-            state: present
-
-        # SELinux allows search access to mnt_t labelled directories by default.
-        - name: Add SELinux fcontext for {{ data_dir.split('/')[0:2]|join('/') }} directory
-          community.general.sefcontext:
-            target: "{{ data_dir.split('/')[0:2]|join('/') }}(/.*)?"
-            setype: mnt_t
-            state: present
-
-        - name: Apply SELinux context to {{ data_dir.split('/')[0:2]|join('/') }} directory
-          ansible.builtin.command: restorecon -Rv {{ data_dir.split('/')[0:2]|join('/') }}
-          register: restorecon_result
-          changed_when: restorecon_result.rc == 0
-
-        - name: Create SELinux CIL file for mnt_t write access
-          ansible.builtin.copy:
-            dest: "/tmp/alloydb_mnt_t_policy.cil"
-            content: |
-              (allow postgresql_t mnt_t (dir (write)))
-              (allow postgresql_t mnt_t (file (open read write getattr)))
-            owner: root
-            group: root
-            mode: '0644'
-
-        - name: Create SELinux policy to allow kdump to write to the crash directory
-          ansible.builtin.copy:
-            dest: "/tmp/kdump_alloydb.cil"
-            content: |
-              (allow postgresql_t kdump_crash_t (dir (search write add_name)))
-              (allow postgresql_t kdump_crash_t (file (create open read write)))
-            mode: '0644'
-
-        - name: Install SELinux CIL policy for mnt_t write access
-          ansible.builtin.command: semodule -i /tmp/alloydb_mnt_t_policy.cil
-
-        - name: Install kdump_alloydb SELinux module
-          ansible.builtin.shell: semodule -i /tmp/kdump_alloydb.cil
-
-        # This rule is applied during RPM installation but for reason it is not reflecting
-        # at runtime. Hence this redundancy.
-        # TODO(siddarts): figure out the reason for the above issue.
-        - name: Add SELinux fcontext for {{ data_dir }} directory
-          community.general.sefcontext:
-            target: "{{ data_dir }}(/.*)?"
-            setype: postgresql_db_t
-            state: present
-
-        - name: Apply SELinux context to {{ data_dir }} directory
-          ansible.builtin.command: restorecon -Rv {{ data_dir }}
-          register: restorecon_result_data_dir
-          changed_when: restorecon_result_data_dir.rc == 0
```

#### 2\. Clean Native Playbooks

Clean the manual semodule installation blocks from the native test playbooks in native/playbooks/:

* **standalone\_backup\_restore\_test.yaml**: Remove task "Install SELinux policy for pgbackrest".  
* **standalone\_alloydbomni\_monitor\_test.yaml**: Remove task "Install SELinux policy for monitor test".  
* **standalone\_central\_logging\_test.yaml**: Remove task "Add SELinux rules for syslog".  
* **standalone\_connection\_pooler\_test.yaml**: Remove task "Create SELinux policy to allow init to connect to the pg port" and "Install pgport\_alloydb SELinux module".

# **Proposal: Global SELinux VM Enablement for Integration Tests**

This proposal outlines the implementation plan for **Part 1 of the SELinux Enforcing Mode integration**: ensuring that SELinux Targeted Enforcing mode is enabled globally and automatically on all virtual machines (DB nodes, Load Balancer nodes, controllers, etcd) upon creation.

---

## 1\. Context & Present State

Currently, RHEL 9 virtual machines spin up with SELinux in **Permissive mode** because the default GCE startup script rhel9.sh (embedded in the GCE platform library) runs:

```
# Disable SELinux entirely. We might relax this in the future, but see
# b/360393236 for issues with rootless runs.
setenforce 0
```

To work around this, we currently have manual individual tasks in playbooks like setup\_database\_node.yaml and setup\_load\_balancer\_node.yaml that explicitly switch SELinux back to enforcing mode:

```
- name: Ensure SELinux is in enforcing mode
  ansible.posix.selinux:
    policy: targeted
    state: enforcing
```

This is fragile, redundant, and does not apply to other nodes (such as the controller or etcd nodes).

---

## 2\. Proposed Options

We propose three approaches to solve this centrally.

### Approach A: Go-Side Central Enablement in nova\_test\_runner.go (Recommended)

After GCE VMs are created and SSH access is configured, we can run a central SSH shell command to turn SELinux Enforcing mode back on before any setup playbooks are started.

#### Implementation in nova\_test\_runner.go:

We can execute the command inside the createInstance task queue:

```
// In storage/testing/lusti/tests/nova/integration_tests/nova_test_runner.go

// Inside func createInstance() around Line 626:
	// Configure SSH access for the created instance.
	if err = configureSSHAccess(clusterInstance, lustiTest); err != nil {
		errs <- fmt.Errorf("Failed to configure SSH access. Error: %v", err)
		return
	}

	// Enable SELinux targeted enforcing mode.
	if err = enableSELinuxTargetedEnforcing(clusterInstance, lustiTest); err != nil {
		errs <- fmt.Errorf("Failed to enable SELinux enforcing. Error: %v", err)
		return
	}
```

And implement the helper:

```
func enableSELinuxTargetedEnforcing(instance *Instance, lustiTest *lusti.Test) error {
	lustiTest.Infof("Enabling SELinux Targeted Enforcing mode on host [%s]...", instance.Params.InstanceName)
	
	// We execute setenforce 1 to transition the kernel mode dynamically
	cmd := "sudo setenforce 1"
	_, _, err := instance.VMClient.Run(lustiTest.Ctx(), cmd)
	if err != nil {
		return fmt.Errorf("failed to execute '%s': %v", cmd, err)
	}
	
	lustiTest.Infof("Successfully enabled SELinux Targeted Enforcing mode on host [%s]", instance.Params.InstanceName)
	return nil
}
```

* **Pros**: 100% centralized. Runs on *every* single VM. Guarantees Enforcing mode before any package installation or service runs list.  
* **Cons**: None.

---

### Approach B: Leverage a Shared Initialization Playbook

Ensure all VMs run a central setup\_selinux.yml play during the cluster bootstrap process.

Add a call to a central playbook:

```
# setup_selinux.yml
- hosts: all
  become: true
  tasks:
    - name: Enable SELinux on all nodes
      ansible.posix.selinux:
        policy: targeted
        state: enforcing
```

And loop this playbook first inside nova\_test\_runner.go before component-specific plays.

* **Pros**: Pure Ansible configuration.  
* **Cons**: Requires maintaining another YAML file and adding code inside Go to execute it across all nodes before main configurations.

---

### Approach C: Parameterize the GCE VM Startup Script (rhel9.sh)

Modify /storage/speckle/vm/tests/perfgate/lib/platforms/startup\_scripts/rhel9.sh to read a GCE metadata flag (e.g. enable-selinux=true) and skip setenforce 0.

* **Pros**: Most "native" boot state.  
* **Cons**: Modifies shared libraries outside our testing suite (storage/speckle), which runs the risk of breaking other product/performance pipelines.

---

## 3\. Redundancy Cleanup Scope

By implementing **Approach A** (Go-side SSH activation), several manual configuration steps and legacy policy scripts in the setup playbooks become redundant and should be cleaned up.

*(Note: In accordance with project instructions, files inside the deprecated /native/ directory are excluded from this cleanup)*:

### A. Remove Redundant 'SELinux Enforcing' Switch Tasks:

These tasks manually switch the OS state on a node-by-node basis, which is no longer needed:

1. **setup\_database\_node.yaml**: Delete lines 40–43.  
2. **setup\_load\_balancer\_node.yaml**: Delete lines 12–15.  
3. **setup\_controller\_node.yaml**: Delete lines 22–25.  
4. **setup\_external\_node.yaml**: Delete lines 26–29.

### B. Remove Redundant Custom Policy Compilation & Directory Mappings:

Because official SELinux policies are metadata-managed and delivered directly inside the product RPMs, these custom test workarounds are no longer required:

1. **setup\_database\_node.yaml**: Delete lines 61–109.  
   * This eliminates manual CIL policy files generation/injection (alloydb\_mnt\_t\_policy.cil and kdump\_alloydb.cil).  
   * This eliminates manual file labelling (community.general.sefcontext and restorecon) which are now cleanly handled by the central installer role automatically.

---

## 4\. Recommended Path

We recommend **Approach A** (Go-side SSH execution in nova\_test\_runner.go immediately after boot and SSH verification) coupled with the **Redundancy Cleanup** in Section 3\. It keeps changes isolated to the nova\_test\_runner Go package, ensures robust error diagnostics if SELinux fails to engage, and results in a cleaner, less redundant Ansible setup.

### C. Remove Redundant Custom Policy Tasks in Ansible Collection Roles:

Because directories and processes for etcd, keepalived, and HAProxy are now fully configured by latest RPM/Node Manager policies, manual CIL generation and policy injection blocks inside compile roles are no longer required:

1. **install/tasks/install\_haproxy.yml**: Remove Configure HAProxy SELinux Policies block (lines 2–31).  
2. **install/tasks/install\_keepalived.yml**: Remove Configure Keepalived SELinux Policies block (lines 2–37).  
3. **install/tasks/bootstrap\_etcd.yml**: Remove Configure etcd SELinux Policies and Contexts block (lines 2–56).  
4. **install/tasks/prepare\_install.yml**: Remove Install SELinux python utilities task (lines 55–59). With manual semanage/semodule runs deleted from Ansible, VM hosts do not require python-utils libraries to process native playbook commands.  
5. **install/tasks/precheck.yml**: Remove semanage utility verification checks (lines 205–219). Prechecking for semanage is obsolete since the playbook will not run manual semanage calls.

  \#\#\# Execution Proposal: \[RPO\]\[RTO\] SELinux \\u2013 Enable SELinux Post-Bootstrap (with Service Restart Workaround)  
  \\u2500\\u2500\\u2500\\u2500\\u2500\\u2500  
  \#\#\# 1\. Scenario Parameters & Architectural Context  
  \\u2022 Scenario: \[RPO\]\[RTO\] SELinux \- Enable SELinux post bootstrap  
  \\u2022 Cluster: satyasais-rto-test-2 (GCP Project: alloydb-nova-tvc-sandbox)  
  \\u2022 Nodes Involved: All 5 cluster nodes (db1, db2, db3, haproxy1, haproxy2) \+ Control VM (control).  
  \\u2022 Root Cause & Workaround Context (Issue b/539767645\#comment5 https\://b.corp.google.com/issues/539767645\#comment5):  
      \\u2022 When transitioning from Permissive to Enforcing on an already bootstrapped cluster, the Linux kernel enforces security labels  
      immediately.  
      \\u2022 Because PostgreSQL (alloydbomni18) had open file descriptors initialized under the old permissive security labels, file  
      relabeling (restorecon / nm\_config) only takes effect once PostgreSQL is restarted.  
      \\u2022 Per confirmation from Ujjwal, Ammaar, and your manager, we will execute the SELinux transition \+ Ansible update\_selinux \+  
      service restart to benchmark the true operational RTO/RPO for this transition.

  \\u2500\\u2500\\u2500\\u2500\\u2500\\u2500  
  \#\#\# 2\. Standardized 3-Iteration Execution Protocol (Runs 1, 2, and 3\)

  For each iteration:

  1\. Step 0: Baseline Verification & Permissive Mode Assertion:  
      \\u2022 Confirm all 5 nodes start in Permissive mode (setenforce 0, SELINUX=permissive).  
      \\u2022 Confirm all services are active and streaming replication lag is 0 bytes.  
      \\u2022 Confirm active VIP 10.1.0.50 is bound on haproxy1.  
  2\. Step 1: Start Workload Baseline (start\_load.sh):  
      \\u2022 Start rtoprober and rpoprober against 10.1.0.50:5432.  
      \\u2022 Start HammerDB transactional workload against 10.1.0.50:6432.  
      \\u2022 Allow 15 seconds to establish steady-state traffic baseline.  
  3\. Step 2: Disruption & SELinux Transition Sequence:  
      \\u2022 (A) Set SELinux to Enforcing: Execute sudo setenforce 1 across all 5 nodes (db1, db2, db3, haproxy1, haproxy2) and capture  
      proof (getenforce).  
      \\u2022 (B) Trigger Ansible SELinux Update: Execute ./orchestrator/alloydb-ansible.sh update \-c ./cluster.conf from the Control VM  
      (runs update\_selinux.yml to apply security policies via nm\_config and cm\_config).  
      \\u2022 (C) Service Restart: Restart alloydbomni18 (and alloydbomni\_cluster\_manager / alloydbomni\_node\_manager) on the DB nodes to  
      apply the newly reconciled SELinux contexts.  
      \\u2022 Pause to allow services to reconcile, promote/reconnect, and resume traffic.  
  4\. Step 3: Metric Calculation & Workload Termination:  
      \\u2022 Run parse\_results.sh to measure exact recovery time (RTO) and data loss (RPO).  
      \\u2022 Cleanly terminate probers and HammerDB.  
  5\. Step 4: Diagnostic Dump Capture (debug\_dump):  
      \\u2022 Execute ./orchestrator/alloydb-ansible.sh debug\_dump \-c ./cluster.conf on the Control VM with all 5 nodes online.  
      \\u2022 Harvest the single diagnostic .tar.gz archive for that run.  
  6\. Step 5: Post-Run Baseline Restoration:  
      \\u2022 Reset all 5 nodes back to Permissive mode (setenforce 0).  
      \\u2022 Verify all services return to active baseline before starting the next run.  
  7\. Step 6: Deliverables Archiving:  
      \\u2022 Archive logs, SELinux proof outputs, and diagnostic tarball to:  
	  \\u2022 CitC Workspace: storage/alloydb/nova/test/orchestrator/rto\_rpo\_testing/logs/selinux\_post\_bootstrap\_runX/  
	  \\u2022 Workstation Backup: \~/rto\_rpo\_test\_logs/selinux\_post\_bootstrap\_runX/

