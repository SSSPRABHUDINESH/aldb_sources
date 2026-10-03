**ETCD** parameters current behaviour

| etcd.setup | etcd.config\_forcewrite | etcd setup pre-exists | Expected Behaviour | Current Behaviour |
| :---- | :---- | :---- | :---- | :---- |
| TRUE | TRUE | TRUE | Setup and Reset ETCD | Setup ETCD and overwrite the existing configuration file. |
| TRUE | TRUE | FALSE | Setup ETCD | Setup ETCD and overwrite the [default configuration file](https://paste.googleplex.com/5522779635056640#l=10). |
| TRUE | FALSE | TRUE | Display ERROR | If `etcd.config_forcewrite` is `false`, Bootstrap will fail because the [default etcd config](https://paste.googleplex.com/5522779635056640#l=10) file is not overwritten and still points to localhost not the [etcd nodes](https://paste.googleplex.com/5993420003868672#l=10). |
| TRUE | FALSE | FALSE | Setup ETCD | If `etcd.config_forcewrite` is `false`, Bootstrap will fail because the [default etcd config](https://paste.googleplex.com/5522779635056640#l=10) file is not overwritten and still points to localhost not the [etcd nodes](https://paste.googleplex.com/5993420003868672#l=10). |
| FALSE | TRUE | TRUE | No action. Bootstrap should work | If `etcd.setup` is `false`, then [bootstrap\_etcd.yml](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/main.yml;l=110-111) won’t be invoked. Then Install role will throw an [error](https://paste.googleplex.com/5850622457937920#l=56) if user didn’t provide ssl certificates, if he provides bootstrap will work. |
| FALSE | TRUE | FALSE | No action. Bootstrap should throw ERROR | If `etcd.setup` is `false`, then [bootstrap\_etcd.yml](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/main.yml;l=110-111) won’t be invoked. Then Install role will throw [error](https://paste.googleplex.com/5850622457937920#l=56) if user didn’t provide ssl certificates, if he provides bootstrap will throw an error. |
| FALSE | FALSE | TRUE | No action. Bootstrap should work | No action. Bootstrap should work |
| FALSE | FALSE | FALSE | No action. Bootstrap should throw ERROR | Install role will throw [error](https://paste.googleplex.com/5850622457937920#l=56) if user didn’t provide ssl certificates, if he provides bootstrap will throw an error. |

### Install Role

### 

| etcd.setup | etcd.config\_forcewrite | etcd setup pre-exists | Expected Behaviour | Actual Behaviour | Actual behaviour matches Expected behaviour | Reason |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| TRUE | TRUE | TRUE | Setup and Reset ETCD | Setup and Reset ETCD | Yes |  |
| TRUE | TRUE | FALSE | Setup ETCD | Setup and Reset ETCD | Yes | Setup ETCD and overwrite the [default configuration file](https://paste.googleplex.com/5522779635056640#l=10). |
| TRUE | FALSE | TRUE | Display ERROR | No Error | No |  |
| TRUE | FALSE | FALSE | Setup ETCD | No Error | No |  |
| FALSE | TRUE | TRUE | No action.  Bootstrap should work | No action. Bootstrap should work | Yes (If user passes ssl certs via deployment\_spec) | If `etcd.setup` is `false`, then [bootstrap\_etcd.yml](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/main.yml;l=110-111) won’t be invoked. Then Install role will throw an [error](https://paste.googleplex.com/5850622457937920#l=56) if user didn’t provide ssl certificates, if he provides bootstrap will work. |
| FALSE | TRUE | FALSE | No action. Bootstrap should throw ERROR | No action. Bootstrap should throw ERROR | Yes | If `etcd.setup` is `false`, then [bootstrap\_etcd.yml](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/main.yml;l=110-111) won’t be invoked. Then Install role will throw [error](https://paste.googleplex.com/5850622457937920#l=56) if user didn’t provide ssl certificates, if he provides bootstrap will throw an error. |
| FALSE | FALSE | TRUE | No action. Bootstrap should work | No action. Bootstrap should work | Yes |  |
| FALSE | FALSE | FALSE | No action. Bootstrap should throw ERROR | Install role will throw and ERROR | Partially Yes (error from install role) | Install role will throw [error](https://paste.googleplex.com/5850622457937920#l=56) if user didn’t provide ssl certificates, if he provides bootstrap will throw an error. |

### 

### Bootstrap Role

| etcd.setup | etcd.config\_forcewrite | etcd setup pre-exists | Expected Behaviour | Actual Behaviour | Result (Passed/Failed) | Reason |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| TRUE | TRUE | TRUE | Setup and Reset ETCD | Setup and Reset ETCD | Yes |  |
| TRUE | TRUE | FALSE | Setup ETCD | Setup and Reset ETCD | Yes |  |
| TRUE | FALSE | TRUE | Display ERROR | Fails during bootstrap | Yes | If `etcd.config_forcewrite` is `false`, Bootstrap will fail because the [default etcd config](https://paste.googleplex.com/5522779635056640#l=10) file is not overwritten and still points to localhost not the [etcd nodes](https://paste.googleplex.com/5993420003868672#l=10). |
| TRUE | FALSE | FALSE | Setup ETCD | Fails during bootstrap | No | If `etcd.config_forcewrite` is `false`, Bootstrap will fail because the [default etcd config](https://paste.googleplex.com/5522779635056640#l=10) file is not overwritten and still points to localhost not the [etcd nodes](https://paste.googleplex.com/5993420003868672#l=10). |
| FALSE | TRUE | TRUE | No action.  Bootstrap should work | No action.  Bootstrap should work | Yes |  |
| FALSE | TRUE | FALSE | No action. Bootstrap should throw ERROR | No action. Bootstrap should throw ERROR | Yes | If `etcd.setup` is `false`, then [bootstrap\_etcd.yml](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/main.yml;l=110-111) won’t be invoked. Then Install role will throw [error](https://paste.googleplex.com/5850622457937920#l=56) if user didn’t provide ssl certificates, if he provides bootstrap will throw an error. |
| FALSE | FALSE | TRUE | No action. Bootstrap should work | No action. Bootstrap should work | Yes |  |
| FALSE | FALSE | FALSE | No action. Bootstrap should throw ERROR | Bootstrap should throw ERROR | Yes |  |

### 

Necessity of **etcd.config\_forcewrite**: 

Given the observed behavior in all test cases, which fall into two scenarios, we can remove the `etcd.config_forcewrite` variable and instead set `force: true` in the configuration.

1\. If `etcd.setup` is `false`, then [bootstrap\_etcd.yml](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/main.yml;l=110-111) won’t be invoked. Then there is no value addition for giving a variable “etcd.config\_forcewrite”.  
2\. If `etcd.setup` is `true`, then [bootstrap\_etcd.yml](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/main.yml;l=110-111) will be invoked. If `etcd.config_forcewrite` is `false`, Bootstrap will fail because the [default etcd config](https://paste.googleplex.com/5522779635056640#l=10) file is not overwritten and still points to localhost not the [etcd nodes](https://paste.googleplex.com/5993420003868672#l=10). Hence `etcd.config_forcewrite` should be `true` to bootstrap etcd successfully.

If a user is providing an existing etcd cluster and does not wish to set up a new one or override the current configuration file, they must set `etcd.setup` to `false`.

### 

| etcd.setup | etcd.config\_forcewrite | etcd setup pre-exists | Expected Behavior |
| ----- | ----- | ----- | ----- |
| **TRUE** | **TRUE** | **TRUE** | Install, overwrite config, and Bootstrap etcd |
| **TRUE** | **TRUE** | **FALSE** | Install and Bootstrap etcd |
| **TRUE** | **FALSE** | **TRUE** | **No action** (Skip etcd install/bootstrap) |
| **TRUE** | **FALSE** | **FALSE** | Install and Bootstrap etcd |
| **FALSE** | **X** | **TRUE** | **No action** (Skip etcd install/bootstrap, setup is handled externally) |
| **FALSE** | **X** | **FALSE** | **Throw early error** in precheck indicating etcd setup does not exist |

### How the Simplified Design Overwrites Config (Without the Intermediate Variable)

### The new configuration uses **force: true** inside the bootstrap template task, and controls the execution of all etcd setup tasks using this direct gate:

### 

  not etcd\_setup\_preexists | bool or (etcd.config\_forcewrite | default(false) | bool)  
    
  

### 

### Case 1: TRUE | TRUE | TRUE (Setup=True, Force=True, Pre-exists=True)

* **Context:** Customer wishes to install etcd, but an etcd cluster is already running. They set config\_forcewrite: true because they explicitly want the installer to rewrite the configuration and reset/re-bootstrap the consensus store.

#### 1\. Expression Evaluation

* etcd.setup | bool \$\\rightarrow\$ **true**  
* not etcd\_setup\_preexists | bool \$\\rightarrow\$ not true \$\\rightarrow\$ **false**  
* etcd.config\_forcewrite | bool \$\\rightarrow\$ **true**  
* **Result:** true and (false or true) \$\\rightarrow\$ **true** (Execute etcd tasks)

#### 2\. Playbook Execution Flow

1. **Prechecks:** Registers etcd\_setup\_preexists: true. No errors are raised because etcd.setup is true.  
2. **install\_etcd.yml**: Executes and validates package presence.  
3. **generate\_certs\_etcd.yml**: Self-signed certificates are generated (if cert paths are empty).  
4. **bootstrap\_etcd.yml**: Runs the templating task. Since it uses force: true, it overwrites the existing configuration /etc/etcd/etcd.conf with new cluster nodes. The etcd service is restarted to load the new config.  
* **Outcome:** **Install, overwrite config, and Bootstrap etcd**.  
  ---

### Case 2: TRUE | TRUE | FALSE (Setup=True, Force=True, Pre-exists=False)

* **Context:** Fresh deployment with no existing etcd service. The customer sets both options to true (standard fresh installation).

#### 1\. Expression Evaluation

* etcd.setup | bool \$\\rightarrow\$ **true**  
* not etcd\_setup\_preexists | bool \$\\rightarrow\$ not false \$\\rightarrow\$ **true**  
* etcd.config\_forcewrite | bool \$\\rightarrow\$ **true**  
* **Result:** true and (true or true) \$\\rightarrow\$ **true** (Execute etcd tasks)

#### 2\. Playbook Execution Flow

1. **Prechecks:** Registers etcd\_setup\_preexists: false.  
2. **install\_etcd.yml**: Runs and installs the etcd RPM package (which creates the RHEL default config file).  
3. **generate\_certs\_etcd.yml**: Generates self-signed certificates.  
4. **bootstrap\_etcd.yml**: Runs template configuration. Because of force: true, it overwrites the RPM package's default config file. The systemd service is then enabled and started.  
* **Outcome:** **Install and Bootstrap etcd**.  
  ---

#### Case 3: TRUE | FALSE | TRUE (Setup=True, Force=False, Pre-exists=True)

* ### **Logical Check:** not etcd\_setup\_preexists | bool or (etcd.config\_forcewrite | bool)

  * ### not true \$\\rightarrow\$ false

  * ### etcd.config\_forcewrite \$\\rightarrow\$ false

  * ### Result: false or false \$\\rightarrow\$ **false**

* ### **Execution Behavior:** Because it evaluates to false, the template configuration task is **skipped entirely**. The template force: true is never reached.

* ### **Result:** **No Action**. The customer's pre-existing, custom configuration file is completely untouched.

#### Case 4: TRUE | FALSE | FALSE (Setup=True, Force=False, Pre-exists=False)

* ### **Logical Check:** not etcd\_setup\_preexists | bool or (etcd.config\_forcewrite | bool)

  1. ### not false \$\\rightarrow\$ true

  2. ### etcd.config\_forcewrite \$\\rightarrow\$ false

  3. ### Result: true or false \$\\rightarrow\$ **true**

* ### **Execution Behavior:**

  1. ### The install role prepares and downloads the etcd RPM package onto the system.

  2. ### The RPM installer creates a default /etc/etcd/etcd.conf file on the node.

  3. ### Ansible proceeds to execute the templating task since the gate condition evaluated to **true**.

  4. ### Since the template task hardcodes **force: true**, it unconditionally overwrites the default /etc/etcd/etcd.conf created by the RPM package.

* ### **Result:** **Install and Bootstrap etcd**. The cluster configuration is correctly written, and the service is bootstrapped.

### 

### Case 5: FALSE | X | TRUE (Setup=False, Force=X, Pre-exists=True)

* **Context:** Customer has set up their own etcd cluster independently (or provided custom etcd\_nodes). They tell our orchestrator **not** to manage/install/bootstrap it (etcd.setup \= false).

#### 1\. Expression Evaluation

* etcd.setup | bool \$\\rightarrow\$ **false**  
* **Result:** false and (...) \$\\rightarrow\$ **false** (Skip all etcd tasks)

#### 2\. Playbook Execution Flow

1. **Prechecks:**  
   * Probing detects the existing etcd (or etcd\_nodes group is defined), so: etcd\_setup\_preexists: true.  
   * The early failure check is skipped because not etcd\_setup\_preexists evaluates to false:  
   * yaml  
   *  when:  
   *    \- not etcd.setup | default(false) | bool  \# Evaluates to true  
   *    \- not etcd\_setup\_preexists | bool         \# Evaluates to false (skips task)  
2. **install\_etcd.yml**, **generate\_certs\_etcd.yml**, and **bootstrap\_etcd.yml**: Skipped completely because the gate condition is false.  
* **Outcome:** **No action**. The existing etcd cluster is untouched, and the orchestrator moves to install the remaining database components.  
  ---

### Case 6: FALSE | X | FALSE (Setup=False, Force=X, Pre-exists=False)

* **Context:** Customer tells the orchestrator **not** to install etcd (etcd.setup \= false), but they did **not** provide any pre-existing etcd cluster in the inventory, nor is etcd active on target nodes. This is an invalid setup.

#### 1\. Expression Evaluation

* etcd.setup | bool \$\\rightarrow\$ **false**  
* **Result:** false and (...) \$\\rightarrow\$ **false** (Skip all etcd tasks)

#### 2\. Playbook Execution Flow

1. **Prechecks:**  
   * Probing does not find any running etcd cluster, and no active etcd\_nodes are in the inventory, so etcd\_setup\_preexists: false.  
   * The failure task triggers because both conditions are met:  
   * yaml  
   *  when:  
   *    \- not etcd.setup | default(false) | bool  \# Evaluates to true (setup is false)  
   *    \- not etcd\_setup\_preexists | bool         \# Evaluates to true (setup does not pre-exist)  
   * Execution halts immediately with the diagnostic message.  
* **Outcome:** **Throws an early error**, preventing broken/incomplete downstream deployments.  
* 

# **Design Proposal: Aligning etcd Installation and Bootstrapping Behavior**

## Background & Objectives

During past deployments and testing, the behavior of the install role with respect to etcd configuration and deployment was identified as a key friction point. Specifically:

1. install\_etcd.yml is run unconditionally even when the user has set etcd.setup to false.  
2. There is no automated detection of whether an etcd setup already exists (if customers passed an existing etcd cluster).  
3. Setting etcd.setup to false without a pre-existing etcd environment succeeds silently during preparation but fails key database cluster initialization tasks downstream (e.g. cluster manager bootstrap).

This proposal outlines a simplified implementation to achieve optimal override behavior across the target matrix. To clean up the parameter surface and meet these objectives, we will repurpose how the existing configuration variable etcd.config\_forcewrite is utilized.

## The Target Configuration Matrix

We must ensure that the orchestrator aligns with the following logical matrix:

| etcd.setup | etcd.config\_forcewrite | etcd setup pre-exists | Expected Behavior |
| :---: | :---: | :---: | ----- |
| TRUE | TRUE | TRUE | Install, overwrite config, and Bootstrap etcd |
| TRUE | TRUE | FALSE | Install and Bootstrap etcd |
| TRUE | FALSE | TRUE | **No action** (Skip etcd install/bootstrap) |
| TRUE | FALSE | FALSE | Install and Bootstrap etcd |
| FALSE | X | TRUE | **No action** (Skip etcd install/bootstrap) |
| FALSE | X | FALSE | **Throw early error** in precheck indicating etcd setup does not exist |

## **How we will check if etcd setup already exists ("etcd setup pre-exists")**

To determine if etcd setup pre-exists is **TRUE** or **FALSE**, We determine setup pre-existence purely dynamically. During the **precheck** phase within tasks/precheck.yml, we execute two checks on the target dcs\_nodes (which resolves dynamically to etcd\_nodes if provided, and falls back to cluster manager or database nodes):

1. Check if the systemd etcd service is currently active (active).  
2. Check if the standard config file /etc/etcd/etcd.conf is present.

If both the service is active and the config exists on all target etcd nodes, we conclude that etcd setup pre-exists \= TRUE. If either is missing on any target host, it evaluates to FALSE.

## **Meeting the right exit criteria (target matrix)**

### Refactoring config\_forcewrite usage

Existing location: [CS Link](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/bootstrap_etcd.yml;l=158)

Currently, config\_forcewrite is passed directly in the template task as force: "{{ etcd.config\_forcewrite }}". After installation, if a default configuration file exists (placed by the RHEL etcd package manager), this parameter prevents overwriting, leading to a stalled or failed bootstrap during clean installs.

**The Solution:**

1. Hardcode **force: true** inside the templating task. By design, if execution reaches the configuration step, we *always* intend to write/overwrite configuration.  
2. Relocate block execution checks to task when conditions. Tasks across install\_etcd.yml and bootstrap\_etcd.yml will only run if: etcd.setup | bool and (not etcd\_setup\_preexists or etcd.config\_forcewrite | bool)

### Case-by-Case Execution Traces

* **Case 1 (TRUE | TRUE | TRUE):** Because etcd.config\_forcewrite is true, the condition evaluates to true. The installer runs, and overwrites the config file with force: true.  
* **Case 2 (TRUE | TRUE | FALSE):** Because not etcd\_setup\_preexists is true, the condition evaluates to true. The installer runs, templates config, and bootstraps.  
* **Case 3 (TRUE | FALSE | TRUE):** The condition evaluates to false. Installation and configuration tasks are **skipped**. Pre-existing state is preserved.  
* **Case 4 (TRUE | FALSE | FALSE):** Because not etcd\_setup\_preexists is true, the condition evaluates to true. Installs etcd and templates with force: true (overriding default package files).  
* **Case 5 (FALSE | X | TRUE):** The condition evaluates to false. All etcd setup tasks are skipped. The installer assumes external management and proceeds.  
* **Case 6 (FALSE | X | FALSE):** Evaluates to false. Early check inside precheck triggers failure.

### Do we need to include condition inside \`generate\_certs\_etcd.yml\`:  Yes, we do need to run generate\_certs\_etcd.yml in all cases, but we **only run the file-definition tasks, while selective shielding prevents fresh cert generation for Cases 3 and 5\.**

Here is the exact breakdown of how and why we handle these concerns:

1\. Should we place the condition on generate\_certs\_etcd.yml?

Yes, but only on the generation block (tasks 8–84), not the entire file.

* generate\_certs\_etcd.yml has two jobs:  
  1. Generate new self-signed certificates on the local controller (if key\_file, cert\_file etc., are empty).  
  2. Define the path variables etcd\_tls\_key, etcd\_tls\_cert, and etcd\_tls\_ca so that other roles (like the Cluster Manager) know where to find the certificates.  
* We cannot skip the entire file, because if we do, the path variables won't be defined, and the subsequent Cluster Manager (CM) configuration tasks will fail indicating undefined variables.  
* Therefore, we place the gate condition not etcd\_setup\_preexists | bool or etcd.config\_forcewrite | bool only on the cert-generation block:

yaml  
 \- name: Generate self-signed certificates for ETCD  
   block:  
     \# ... cert generation shell tasks ...  
   when:  
     \- etcd.ssl.enable | bool  
     \- etcd.ssl.key\_file is none or etcd.ssl.key\_file \== ""  
     \# ...  
     \- etcd.setup | default(false) | bool  
     \- not etcd\_setup\_preexists | bool or (etcd.config\_forcewrite | default(false) | bool) \# \<-- GATING ADDED HERE  
---

2\. For Case 3, and 5: Are customer certs sufficient? Will we generate them?

Customer certs are sufficient, and we will NOT generate new certs in these cases.  
If we generate new certs, it would trigger a certificate mismatch, and the Cluster Manager would fail to establish a trusted TLS connection to the existing etcd store.

* For Case 5 (**FALSE | X | TRUE**): The user runs with etcd.setup \= false. The generation block is skipped by the check \- etcd.setup | default(false) | bool. The role validates that the customer supplied their own paths using Validate ETCD SSL paths and defines those paths to be used.  
* For Case 3 (**TRUE | FALSE | TRUE**): The user runs with etcd.setup \= true but the setup pre-exists and config\_forcewrite is false. Because not etcd\_setup\_preexists evaluates to false, the generation block is skipped. The role assumes the certificates from the initial deployment run already exist in the final output directory and maps those path variables.

Summary  
The proposal is already aligned with this requirement: generate\_certs\_etcd.yml remains included in the component execution list to define connection paths, but the action of creating new self-signed keys is shielded for Cases 3 and 5\.

**Below is the precise technical breakdown of our test validation workflow:**

1. **prepare.yml**: Executes initial installation and cluster bootstrapping on new virtual machines. This process generates the /etc/etcd/etcd.conf asset and initializes the etcd daemon. Upon completion, a healthy cluster is established, setting **etcd\_setup\_preexists: true**.  
2. **test.yml**: Iterates through the installation role across varied logic states (toggling etcd.setup and etcd.config\_forcewrite). The orchestrator dynamically detects the existing environment on the target infrastructure.  
3. **Validation Logic**: We assert that the role correctly omits or applies changes to the service state and configuration templates as defined by the target matrix.

**Why gating is not required in last phase:**

Implementing gates on these concluding operations within generate\_certs\_etcd.yml would introduce a regression. For external etcd deployments (etcd.setup: false) with SSL active, the orchestrator would fail to map provided asset locations to the etcd\_tls\_ca and etcd\_tls\_key variables, resulting in an environment failure.

Maintaining this ungated assertion ensures that the operator must fulfill one of the following requirements:

1. Explicitly set etcd.ssl.enable: false if encrypted transport is unnecessary.  
2. Supply valid certificate paths when SSL remains active.

**Why gating is needed in generation phase:**

Absent these gates, an update to an existing managed cluster (where etcd.setup: true) initiated from a new control plane lacking local temporary assets would trigger several critical failures:

1. The creates parameter in Ansible would fail to detect the necessary files on the local controller.  
2. The orchestrator would misidentify the environment as new, leading to the **unintended generation of fresh self-signed credentials**.  
3. These new assets would overwrite existing certificates on target hosts; consequently, since the active etcd processes rely on *legacy* keys, secure communication would break and dependent services would terminate.

**What is heartbeat\_interval:**

1. **Definition of heartbeat\_interval:** This parameter specifies the frequency (measured in milliseconds) at which the etcd leader issues heartbeats to followers to preserve the Raft consensus state. The default value is 100 ms.  
2. **Configuration Persistence:** The value is persisted within /etc/etcd/etcd.conf on the database nodes. The orchestrator manages this via the   
3. etcd.conf.j2 template:  
4. Shell implementation:

```sh
ETCD_HEARTBEAT_INTERVAL="{{ etcd.heartbeat_interval }}"
```

5. **Validating forcewrite and restart behavior:**  
   * **Initial Bootstrap (prepare.yml):** Since the interval is initially unset, it defaults to 100. Consequently, the configuration file reflects ETCD\_HEARTBEAT\_INTERVAL="100" across the cluster.  
   * **Target State Execution (test.yml):** By updating to heartbeat\_interval: 200 while config\_forcewrite is true, the orchestrator executes the templating task. Ansible identifies the delta between the existing "100" and the new "200" value, overwrites the target file, and reports a status of changed: true.  
   * **Operational Outcome:** This change notification invokes the Restart etcd service handler, ensuring the daemon is cycled to apply the updated interval.

This lifecycle confirms that both the file metadata and the systemd ActiveEnterTimestamp are successfully updated, fulfilling the required validation criteria.

## Rationale for meta: flush\_handlers Integration

**Directive Definition:** The ansible.builtin.meta: flush\_handlers directive is an internal control mechanism that enforces the immediate execution of queued service handlers, specifically those notified during prior configuration tasks.

---

**Technical Challenge (Test Failure Analysis):** By default, the orchestrator employs a lazy execution model for notifications, deferring restarts **until the conclusion of the entire playbook**. This behavior resulted in the following sequence during automated testing:

Operational friction points identified:

1. Configuration tasks completed and successfully notified the restart handler.  
2. Downstream validation tasks executed against the legacy daemon state.  
3. The etcd service cycled only *after* the play finished.

Consequently, the verification phase recorded the **pre-restart** timestamps, triggering a false-negative result.

---

**Proposed Solution:** Inserting the flush\_handlers directive following the installation role forces the orchestrator to **suspend task execution and immediately cycle the etcd daemon** before proceeding.

This ensures the target exit criteria are met:

1. Service restarts are prioritized and completed.  
2. Verification tasks successfully capture the **post-restart** state.  
3. Integration tests validate the operational lifecycle accurately.

# **Design Proposal: Aligning etcd Installation and Bootstrapping Behavior**

# **Background & Objectives**

During past deployments and testing, the behavior of the install role with respect to etcd configuration and deployment was identified as a key friction point. Specifically:

* install\_etcd.yml is run unconditionally even when the user has set etcd.setup to false.  
* There is no automated detection of whether an etcd setup already exists (either dedicated external hosts or previously deployed service instances).  
* Setting etcd.setup to false without a pre-existing etcd environment succeeds silently during preparation but fails key database cluster initialization tasks downstream (e.g. cluster manager bootstrap).

This proposal outlines a simplified, production-grade implementation that avoids adding temporary intermediate facts (minimizing registry variables) and achieves optimal override behavior across the target matrix. To clean up the parameter surface and meet these objectives, we will repurpose how the existing configuration variable etcd.config\_forcewrite is utilized.

# **The Target Configuration Matrix**

We must ensure that the orchestrator aligns with the following logical matrix:

| etcd.setup | etcd.config\_forcewrite | etcd setup pre-exists | Expected Behavior |
| :---- | :---- | :---- | :---- |
| TRUE | TRUE | TRUE | Install, overwrite config, and Bootstrap etcd |
| TRUE | TRUE | FALSE | Install and Bootstrap etcd |
| TRUE | FALSE | TRUE | No action (Skip etcd install/bootstrap) |
| TRUE | FALSE | FALSE | Install and Bootstrap etcd |
| FALSE | X | TRUE | No action (Skip etcd install/bootstrap, setup is handled externally) |
| FALSE | X | FALSE | Throw early error in precheck indicating etcd setup does not exist |

# **Question 1: How we will check if etcd setup already exists ("etcd setup pre-exists")**

To determine if etcd setup pre-exists is TRUE or FALSE, we will perform a dual check covering both inventory configuration (static) and remote system state (dynamic runtime probing):

## **1\. Static Pre-existence Check**

If the user passes external/custom etcd hostgroups in their inventory (i.e., groups\['etcd\_nodes'\] is defined and contains active hosts), they are supplying their own etcd setup externally.

## **2\. Dynamic Runtime Check**

If the user does not pass dedicated etcd\_nodes, the orchestrator falls back to running etcd on primary\_instance\_nodes or cluster\_manager\_nodes. To determine if etcd has already been set up on these hosts, we will run two checks during the precheck phase within tasks/precheck.yml:

1. Check if the systemd etcd service is currently active (active).  
2. Check if the standard config file /etc/etcd/etcd.conf is present.

If both the service is active and the config exists on all target etcd nodes, we conclude that etcd setup pre-exists \= TRUE.

```
- name: Determine if etcd setup pre-exists on each node
  block:
    - name: Check if etcd service is active
      ansible.builtin.command: systemctl is-active etcd
      register: etcd_service_status
      failed_when: false
      changed_when: false
      check_mode: no
    - name: Check if etcd configuration exists
      ansible.builtin.stat:
        path: /etc/etcd/etcd.conf
      register: etcd_conf_stat
    - name: Set host fact for etcd pre-existence
      ansible.builtin.set_fact:
        etcd_setup_preexists_on_host: >-
          {{ etcd_service_status.stdout == 'active' or etcd_conf_stat.stat.exists }}
      when: inventory_hostname in dcs_nodes
    - name: Set global etcd setup pre-existence state
      ansible.builtin.set_fact:
        etcd_setup_preexists: >-
          {{ (groups['etcd_nodes'] is defined and groups['etcd_nodes'] | length > 0) or (dcs_nodes | map('extract', hostvars, 'etcd_setup_preexists_on_host') | select('defined') | list | length == dcs_nodes | length and dcs_nodes | map('extract', hostvars, 'etcd_setup_preexists_on_host') | select('defined') | all) }}
      delegate_to: localhost
      run_once: true
```

