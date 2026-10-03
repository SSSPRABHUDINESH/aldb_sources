## **Ansible Version Compatibility Issue**

### **Scenario:**

A templating error was identified during testing of the Ansible install role. This error is due to an incompatibility between the orchestrator's required Ansible version and the maximum supported version on RHEL 9 (which is Ansible 2.15).

Specifically, the issue occurs when the control node is running RHEL 9\. However, Earlier testing confirmed that this error does not occur when the control node is a Debian machine running Ansible version \>= 2.16.3.

### **Sample Failing Snippet from [main.yml](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/defaults/main.yml):**

```
pgbackrest:
  location: "https://apt.postgresql.org/pub/repos/yum/common/redhat/{{ rhel_version }}"
  version: "2.55.1-1PGDG"
  packagename: "pgbackrest-{{ pgbackrest['version'] }}.rhel{{ ansible_facts['distribution_major_version'] }}.{{ ansible_facts['architecture'] }}.rpm"`
```

### **Error Message:**

The error is a `templating` failure, specifically an `AnsibleError` occurring during a conditional check:

```
TASK [google.alloydbomni_orchestrator.install : Create Artifact Registry repository configuration file] ***
fatal: [10.128.0.16]: FAILED! => {"msg": "The conditional check ''pkg.dev' in package_location' failed. The error was: An unhandled exception occurred while templating '{{ pgbouncer.location }}'. Error was a <class 'ansible.errors.AnsibleError'>, original message: An unhandled exception occurred while templating '{'location': 'https://apt.postgresql.org/pub/repos/yum/common/redhat/{{ rhel_version }}', 'version': '1.24.1-42PGDG', 'packagename': \"pgbouncer-{{ pgbouncer['version'] }}.rhel{{ ansible_facts['distribution_major_version'] }}.{{ ansible_facts['architecture'] }}.rpm\"}'. Error was a <class 'ansible.errors.AnsibleError'>, original message: An unhandled except..."}
```

### **Root Cause Analysis (Ansible & RHEL Version Mismatch)**

* **RHEL 9 Limitation:** The officially supported Ansible version for **RHEL 9** is **2.14**. Manual installation of higher versions is complicated because RHEL 9 only supports Python 3.9, whereas Ansible 2.16 requires Python 3.10.  
* **Orchestrator Requirement:** The Ansible orchestrator collection requires a minimum Ansible version of **`>=2.16.14`**.  
* **Maximum RHEL 9 Supported Version:** The highest stable version RHEL 9 currently supports is **2.15.13**.  
* **RHEL 10 Comparison:** RHEL 10 (recently launched in May 2025\) successfully runs Ansible **2.16.14**.  
* **Impact:** If the orchestrator is launched requiring Ansible 2.16, it will fail for RHEL 9 customers, which is a significant user base. 

*Note: Jinja templating is known to be fully functional from Ansible version 2.16.4.*

### **Expected Solution:**

The required orchestrator version (`>=2.16.14`) exceeds the highest supported version on RHEL 9 (**2.15.13**).

1. **Modify Orchestrator Code:** Adapt the Ansible orchestrator collection code to be compatible with the RHEL 9 supported version, **2.15.13**.

### **Solution:**

A temporary workaround is to hardcode the version variable in the configuration:

```
pgbackrest:
  location: "https://apt.postgresql.org/pub/repos/yum/common/redhat/{{ rhel_version }}"
  version: "2.55.1-1PGDG"
  packagename: "pgbackrest-2.55.1-1PGDG.rhel{{ ansible_facts['distribution_major_version'] }}.{{ ansible_facts['architecture'] }}.rpm"`
```

