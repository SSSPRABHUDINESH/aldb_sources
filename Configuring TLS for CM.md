**Cluster Manager TLS Configuration**

**Summary:** This document details the automation of TLS configuration for AlloyDB Omni components (etcd, Cluster Manager, and Node Manager) using Ansible. Shared certificates are used to secure communications, ensuring end-to-end encryption between the server and client component.

**Requirements:**

* **TLS should be disabled by default**. When enabled, all three services (etcd, CM, NM) must update **simultaneously** to avoid connection errors.  
* Users can enable TLS and provide their own certificates.  
* The system will handle clients providing certificates. 

**Interface & Implementation:**

### Ansible Role Specifications

* **Role Name: install**  
* **Input Parameters:**

```
ssl:
  enabled: false
  key_file: null
  cert_file: null
  ca_file: null
```

* **ETCD TLS Configuration:**  
  * Ansible ensures an SSL directory (`/var/lib/etcd/ssl`) exists.  
  * If the user provides certificates, the `key_file`, `cert_file`, and `ca_file` are copied to `/var/lib/etcd/ssl/` on the dcs nodes.  
  * Update `/etc/etcd/etcd.conf` to use TLS  certificates for Internal and Client Communication.  
  * Restarts the `etcd` service.  
  * **Key Configuration Snippet (inside** `/etc/etcd/etcd.conf`**):**

```
ETCD_TRUSTED_CA_FILE="/var/lib/etcd/ssl/rootca.crt"
ETCD_CERT_FILE="/var/lib/etcd/ssl/etcd.crt"
ETCD_KEY_FILE="/var/lib/etcd/ssl/etcd.key"

ETCD_PEER_TRUSTED_CA_FILE="/var/lib/etcd/ssl/rootca.crt"
ETCD_PEER_CERT_FILE="/var/lib/etcd/ssl/etcd.crt"
ETCD_PEER_KEY_FILE="/var/lib/etcd/ssl/etcd.key"
```

  *   
* **Cluster Manager (CM) TLS Configuration:**  
  * **Objective:** Enable CM to communicate securely with the TLS-secured etcd cluster and NM.  
  * **Method:** Inside `/install_alloydbomni_cluster_manager.yml`  
    * Ansible ensures an SSL directory (`/etc/alloydbomni/cm/ssl/`) exists.  
    * User provides certificates the `key_file`, `cert_file`, and `ca_file` are copied to `/etc/alloydbomni/cm/ssl/` on the cm nodes.  
    * Update `/etc/alloydbomni/cluster_manager.yaml` to use TLS certificate paths for CM Internal Communication , CM to ETCD , CM to NM.  
    * Restarts the `cluster_manager` service.  
  * **Key Configuration will be provided by Tarun(**`/etc/alloydbomni/cluster_manager.yaml`**).**  
* **Node manager TLS Configuration:**  
  * **Objective:** Secure internal management traffic between Node Manager Nodes.  
  * **Method:** Inside `/install_alloydbomni_node_manager.yml`  
    * Ansible ensures an SSL directory (`/etc/alloydbomni/nm/ssl/`) exists.  
    * User provides certificates the `key_file`, `cert_file`, and `ca_file` are copied to `/etc/alloydbomni/nm/ssl/` on the db nodes.  
    * Update `/etc/alloydbomni/node_manager.toml` to use TLS certificate paths.  
    * Restarts the `node_manager` service.  
  * **Key Configuration Snippet (inside** `/etc/alloydbomni/node_manager.toml`**):**

```
no_tls = false
tls_cert = "/etc/alloydbomni/cm/ssl/nm.crt"
tls_key = "/etc/alloydbomni/cm/ssl/nm.key"
client_ca = ["/etc/alloydbomni/nm/ssl/rootca.crt"]
```

