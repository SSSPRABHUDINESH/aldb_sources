# **Design Proposal: Enhancing Ansible remove Role to Support Removing Nodes from DBCluster**

* **Author:** [Satya Sai Sontenam (xWF)](mailto:satyasais@google.com)  
* **Target Collection:** google.alloydbomni\_orchestrator  
* **Target Role Path:** roles/remove  
* **Status:** Proposal / Review

---

## 1\. Overview & Background

The google.alloydbomni\_orchestrator.remove Ansible role automates the scale-in and decommissioning of AlloyDB Omni nodes. Currently (as implemented in [CL 940460252](https://cl.corp.google.com/940460252)), the remove role supports scaling in **ReadPool DBInstance** resources.

### The Gap

The remove role does not currently support scaling in **DBCluster nodes** (such as HA Standby nodes). Specifically:

1. It lacks logic to decommission DBCluster standby nodes while ensuring surviving primary and standby database nodes remain online and operational.  
2. It does not automate the lifecycle transition from HA to single-node Standalone (e.g., stopping Keepalived and removing VIP configurations across surviving nodes).  
3. It needs safeguards to prevent the active Primary node from being decommissioned accidentally.

### Objective

This design extends the google.alloydbomni\_orchestrator.remove Ansible role to support removing **DBCluster DB nodes** (High Availability standbys) cleanly, safely, and with zero downtime for write traffic.

---

## 2\. Key Requirements & Guiding Principles

1. **Two-Phase Teardown Architecture:**  
   * **Phase 1: Logical De-registration:** Cluster Manager (CM) updates cluster state in etcd, drains traffic (reloads Keepalived/PgBouncer), drops replication slots on the Primary, and marks instances for deletion.  
   * **Phase 2: Physical Teardown:** Ansible executes host-level cleanup on decommissioned hosts by stopping runtime services (alloydbomni, keepalived, pgbouncer, alloydbomni\_monitor), deleting the PostgreSQL data directory (/data), and uninstalling RPM packages.  
2. **Primary Node Protection Guardrail:**  
   * Attempting to remove the active Primary host is **strictly forbidden**. The playbook must validate node roles via Cluster Manager and fail immediately if a primary node removal is attempted before a switchover.  
3. **Control Plane Protection:**  
   * Only data plane components (alloydbomni, keepalived, pgbouncer, alloydbomni\_monitor) are removed from target nodes. Control plane components (alloydbomni\_cluster\_manager, etcd) hosted on surviving DB nodes must be left untouched.  
4. **Dynamic VIP & Load Balancer Disbandment:**  
   * **Scaling HA (e.g., 3-Node HA → 2-Node HA):** Exclude removed host IPs from KeepalivedConfig CRs; surviving nodes reload Keepalived seamlessly.  
   * **Transitioning to Standalone (2-Node HA → 1-Node Standalone):** Delete KeepalivedConfig CRs, stop and disable Keepalived on surviving primary nodes as Virtual IP failover is no longer required.  
5. **Stateful Persistence & Idempotency:**  
   * Maintain a pending cleanup cache (remove\_pending.json) on localhost so mid-flight failures during physical teardown can be retried safely without losing context.

---

## 3\. Component Teardown Matrix by Node Type

When a host is removed from a DBCluster, the remove role classifies services on the host and performs targeted cleanup:

| Component Package | DB Primary Node | DB Standby Node (DBCluster) | ReadPool Node (DBInstance) | Action on Target Decommissioned Node |
| :---- | :---: | :---: | :---: | :---- |
| **alloydbomni** (PostgreSQL Engine) | Protect (Blocked) | Teardown | Teardown | Stop alloydbomni18 service & purge /data |
| **alloydbomni\_monitor** | Protect (Blocked) | Teardown | Teardown | Stop & disable systemd service |
| **alloydbomni\_node\_manager** | Protect (Blocked) | Teardown | Teardown | Stop & disable systemd service |
| **pgbouncer** (Connection Pooler) | Protect (Blocked) | Teardown | Teardown | Stop & disable systemd service |
| **keepalived** (Virtual IP Manager) | Protect (Blocked) | Teardown | N/A | Stop & disable systemd service |
| **alloydbomni\_cluster\_manager** | Protect (Blocked) | Keep (If shared CM host) | N/A | **Do Not Touch** if shared control node |
| **etcd** (Consensus Store) | Protect (Blocked) | Keep (If shared DCS host) | N/A | **Do Not Touch** if shared control node |

---

## 4\. Topology Scaling Scenarios

### Case 1: 3-Node HA to 2-Node HA (Scale-In)

* **User Action:** Decreases numberOfStandbys: 1 and removes Db3 IP/hostname from DeploymentSpec.PrimaryInstanceNodes.Hosts.  
* **Execution Flow:**  
  1. CM updates KeepalivedConfig CRs to exclude Db3. Db1 and Db2 reload Keepalived.  
  2. CM issues DeleteDatabase gRPC request to Db3 Node Manager.  
  3. Db1 (Primary) drops a replication slot for Db3.  
  4. Ansible runs teardown\_host on Db3 (stops database/proxy services, purges data directory, uninstalls data plane RPMs).

### Case 2: 2-Node HA to 1-Node Standalone (VIP Disbandment)

* **User Action:** Sets numberOfStandbys: 0 and removes Db2 from DeploymentSpec.PrimaryInstanceNodes.Hosts.  
* **Execution Flow:**  
  1. CM deletes all KeepalivedConfig CRs from etcd.  
  2. Node Manager on surviving Db1 (Primary) stops and disables Keepalived service.  
  3. CM issues DeleteDatabase to Db2 Node Manager.  
  4. Ansible runs teardown\_host on Db2. Db1 continues serving direct read/write traffic as a standalone cluster.

---

## 5\. Detailed Ansible Architecture & Workflow

```
[Start remove Role] ──► [Prechecks & Inventory Validation]
                              │
                              ▼
                [Primary Node Safety Check] ──► (Fail if active primary requested for removal)
                              │
                              ▼
                [Resolve Target DBCluster Hosts]
                              │
                              ▼
           [Update Pending Cleanup Cache on Localhost]
                              │
                              ▼
       [Apply Updated DBCluster Spec to Cluster Manager]
                              │
                              ▼
         [Poll CM & Verify Logical De-registration / HA State]
                              │
                              ▼
       [Execute Physical Teardown on Target VMs (teardown_host)]
                              │
                              ▼
         [Verify Health of Surviving DBCluster (DBClusterReady)]
                              │
                              ▼
          [Update Pending Cache & Output Execution Summary]
```

### Modular Task Breakdown

* tasks/main.yml: Orchestrates prechecks, spec re-application, logical verification, physical teardown, and surviving cluster verification.  
* tasks/prechecks.yml:  
  * Verifies Cluster Manager gRPC accessibility.  
  * Queries current cluster topology via status role.  
  * Asserts that resource\_type (either DBInstance or DBCluster) is specified.  
* **tasks/resolve\_cluster\_hosts.yml (New):**  
  * Compares current inventory hosts in groups\['primary\_instance\_nodes'\] against active DBCluster st7atus returned by CM.  
  * Calculates host differences and verifies numberOfStandbys reduction.  
  * Identifies active primary host IP from CM status and asserts target host is **not** primary.  
* **tasks/reapply\_dbcluster.yml (New):**  
  * Invokes google.alloydbomni\_orchestrator.bootstrap to apply the reduced DBCluster spec (numberOfStandbys, host list, VIP options) to Cluster Manager.  
* **tasks/verify\_cluster\_scale\_in.yml (New):**  
  * Polls CM status until DBCluster condition reflects the updated standby count and target Instance CRs are marked as removed.  
* tasks/teardown\_host.yml:  
  * Executes physical teardown on removed host:  
    1. Stops systemd units: alloydbomni18, keepalived, pgbouncer, alloydbomni\_monitor, alloydbomni\_node\_manager.  
    2. Purges data directories (/data, /var/log/alloydbomni).  
    3. Invokes google.alloydbomni\_orchestrator.uninstall role to remove data plane RPM packages.

---

## 6\. Playbook Interface & Usage Example

### Ansible Variables

| Variable | Type | Default | Description |
| :---- | :---- | :---- | :---- |
| resource\_type | string | DBCluster | Target resource type to scale in (DBCluster or DBInstance). |
| deployment\_spec | string | Mandatory | Path to updated deployment specification file or YAML string. |
| deletion\_poll\_timeout | int | 300 | Max seconds to wait for CM logical scale-in verification. |
| deletion\_poll\_interval | int | 10 | Seconds between status polling attempts. |

### Sample Playbook (samples/playbooks/remove\_cluster.yml)

```
---
- name: Scale-In AlloyDB Omni DBCluster Node
  hosts: all
  vars:
    ansible_become: true
    ansible_user: <ansible_user>
    ansible_ssh_private_key_file: <path_to_private_key>
    resource_type: DBCluster
  roles:
    - role: google.alloydbomni_orchestrator.remove
```

### Execution Command

```
ansible-playbook -i inventory.yml samples/playbooks/remove_cluster.yml \
  -e resource_type=DBCluster \
  -e deployment_spec=deployment_spec_updated.yaml
```

---

## 7\. Failure Handling & Rollback Procedures

| Failure Point | Impact | Recovery / Rollback Strategy |
| :---- | :---- | :---- |
| **Primary Removal Attempted** | Blocked immediately during prechecks. | Playbook terminates before making any changes. User must perform switchover first. |
| **CM Connection Failure during Apply** | CM state unchanged; no VM modified. | Playbook fails gracefully in logical phase. Re-run playbook once CM is reachable. |
| **Physical Teardown Failure on VM** | Host removed logically in CM, but packages/files remain on VM. | Host address is recorded in remove\_pending.json. Re-running the remove role reads the cache and retries physical teardown on failed hosts. |

---

## 8\. Verification & Testing Plan

1. **Basic Scale-In Test (3-Node HA → 2-Node HA):**  
   * Reduce numberOfStandbys from 2 to 1 and remove Db3.  
   * Verify Keepalived on Db1 and Db2 reloads without dropping client connections.  
   * Verify Db3 data directory /data is deleted and RPM packages are uninstalled.  
   * Verify surviving cluster status is DBClusterReady with HAReady=True.  
2. **Standalone Transition Test (2-Node HA → 1-Node Standalone):**  
   * Reduce numberOfStandbys from 1 to 0 and remove Db2.  
   * Verify Keepalived service is stopped and disabled on surviving Db1.  
   * Verify read/write traffic to Db1 continues uninterrupted.  
3. **Negative Safety Test (Primary Node Protection):**  
   * Submit removal spec targeting active primary host Db1.  
   * Verify playbook aborts during prechecks with explicit error message: "Primary host Db1 cannot be removed. Perform switchover first."  
4. **Idempotency & Cache Test:**  
   * Simulate SSH network failure during physical teardown of Db3.  
   * Re-run remove role. Verify cache resolution loads Db3 from remove\_pending.json and completes physical teardown.

