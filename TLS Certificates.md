TLS (Transport Layer Security) certificates, it helps to think of them as **digital passports** for your servers.

Just as a passport proves to a border agent that you are who you say you are (because a trusted government issued it), a TLS certificate proves to a client (like a browser or another server) that your server is legitimate and not an impostor.

### The Three Pillars of TLS

TLS certificates provide three essential security functions:

* **Authentication:** Confirms that the server you are talking to is the real "owner" of the domain or service.  
* **Encryption:** "Scrambles" the data being sent so that if a hacker intercepts it, they only see gibberish.  
* **Integrity:** Ensures that the data hasn't been tampered with or altered while traveling across the network.

### How the "Handshake" Works

When your Ansible scripts configure TLS, they are setting up a process called a **TLS Handshake**. Here is the simplified flow:

1. **The Greeting:** The client (e.g., your Data Path Service) says "Hello" to the server (e.g., ETCD).5  
2. **The ID Check:** The server sends its **TLS Certificate** (which contains its **Public Key**).6  
3. **The Verification:** The client checks if a "Trusted Authority" (like your internal company CA) signed that certificate.7  
4. **The Secret Key:** Once trusted, they both agree on a "session key" to encrypt all further conversation.8

---

### Key Terms You’ll See in Your Tasks

Since you are working with Ansible and ETCD, you will run into these specific components:

| Component | What it is | Why it matters |
| :---- | :---- | :---- |
| **Private Key** | A secret file on the server. | It must **never** be shared. It is used to "sign" and decrypt data. |
| **Public Key** | A file shared with everyone. | It allows others to encrypt data that only your Private Key can open. |
| **CA (Certificate Authority)** | The "Boss" who signs certificates. | In your case, this is likely an internal tool (like HashiCorp Vault or a custom script). |
| **CSR (Signing Request)** | An unsigned application. | You generate this first, then send it to the CA to get your actual Certificate. |
| **mTLS (Mutual TLS)** | Two-way authentication. | In ETCD clusters, both the client AND the server must show certificates to each other. |

### **3\. How the Ansible Template should look**

To get the Python code to generate the file above, your `deployment_spec.yaml` (or `resource_spec.yaml`) needs to pass these values as variables.

**Your proposed modification to the Ansible task:**

YAML

```
    - name: Copy deployment_spec.yaml
      ansible.builtin.copy:
        dest: "/tmp/deployment_spec.yaml"
        content: |
          alloydbomni:
            vars:
              cluster_name: {{ cluster_name }}
              # Add these lines for the Python Orchestrator to read:
              etcd_protocol: "https"
              etcd_tls_enabled: true
              etcd_root_ca: "/var/lib/etcd/ssl/rootca.crt"
              etcd_cert: "/var/lib/etcd/ssl/etcd.crt"
              etcd_key: "/var/lib/etcd/ssl/etcd.key"
```

Questions:

1. If I pass the local path from resource spec, how should it copy and do change in config and restart?  
2. How can I change the `dcs.endpoints` protocol from `http` to `https` in the `cluster_manager.yaml`?"  
3. "Does the `DBCluster` resource kind support a `tls` or `ssl` block in the `resource_spec`?"  
4. "What variable in the `resource_spec` controls the `no_tls` setting in `node_manager.toml`?"  
5. If cluster manager \-\> cluster.yaml can be modified from vars ( [https\://critique.corp.google.com/cl/830912344/depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni\_orchestrator/roles/install/tasks/install\_alloydbomni\_cluster\_manager.yml](https://critique.corp.google.com/cl/830912344/depot/google3/storage/alloydb/nova/automation/ansible/collections/google/alloydbomni_orchestrator/roles/install/tasks/install_alloydbomni_cluster_manager.yml))

6. But it should be able to pass from cluster manager and node manager from resource spec, as python is handling from local server-\> implementing ssh is the only way, but it is difficult.  
7. This cl will shows you flexibility for etcd [https\://critique.corp.google.com/cl/814093438](https://critique.corp.google.com/cl/814093438) 

You should generate certificates and pass them through it .

Things it will help:

1. Do prepare and check the below files, it will show empty  
2. Do bootstrap and again check below files. It show

From db node: 

```
[satyasais_google_com@satyasais-failover-db1 ~]$ cd /etc/alloydbomni/
[satyasais_google_com@satyasais-failover-db1 alloydbomni]$ ls
cluster_manager.yaml  node_manager_roles.toml  node_manager.toml
[satyasais_google_com@satyasais-failover-db1 alloydbomni]$ cat cluster_manager.yaml 
cat: cluster_manager.yaml: Permission denied
[satyasais_google_com@satyasais-failover-db1 alloydbomni]$ sudo cat cluster_manager.yaml 
api_server:
    enable_reflection: true
    listen_addr: :6703
cluster:
    cluster_name: satyasais-failover
    instance_name: satyasais-failover-db1
controller_runtime_manager:
    cluster_addr: :8088
    cluster_health_probe_addr: :8089
    node_addr: :8090
    node_health_probe_addr: :8091
dcs:
    dial_timeout: 10
    endpoints:
        - 10.1.0.3:2379
        - 10.1.0.5:2379
        - 10.1.0.6:2379
    ttl: 30
    type: etcd
log:
    severity: INFO
[satyasais_google_com@satyasais-failover-db1 alloydbomni]$ ls
cluster_manager.yaml  node_manager_roles.toml  node_manager.toml
[satyasais_google_com@satyasais-failover-db1 alloydbomni]$ sudo cat node_manager_roles.toml 
roles = ['database']

[config]
pg_version = '17.5.0'
[satyasais_google_com@satyasais-failover-db1 alloydbomni]$ sudo cat node_manager.toml 
[logging]
verbosity = 0

[grpcsrv]
listen_addr = ":6700"
no_tls = true
tls_cert = ""
tls_key = ""
client_ca = []
enable_reflection = true
exit_on_stop = true
[satyasais_google_com@satyasais-failover-db1 alloydbomni]$ sudo cat cluster_manager.yaml 
api_server:
    enable_reflection: true
    listen_addr: :6703
cluster:
    cluster_name: satyasais-failover
    instance_name: satyasais-failover-db1
controller_runtime_manager:
    cluster_addr: :8088
    cluster_health_probe_addr: :8089
    node_addr: :8090
    node_health_probe_addr: :8091
dcs:
    dial_timeout: 10
    endpoints:
        - 10.1.0.3:2379
        - 10.1.0.5:2379
        - 10.1.0.6:2379
    ttl: 30
    type: etcd
log:
    severity: INFO
[satyasais_google_

```

```
[satyasais_google_com@satyasais-failover-db1 etc]$ sudo cat etcd/etcd.conf
ETCD_NAME="satyasais-failover-db1"
ETCD_DATA_DIR="/var/lib/etcd/data"
ETCD_LISTEN_PEER_URLS="http://10.1.0.3:2380,http://127.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.1.0.3:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.1.0.3:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.1.0.3:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER="satyasais-failover-db1=http://10.1.0.3:2380,satyasais-failover-db2=http://10.1.0.5:2380,satyasais-failover-db3=http://10.1.0.6:2380"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_AUTO_COMPACTION_RETENTION="1"
ETCD_AUTO_COMPACTION_MODE="revision"


```

1. Create infra for resilient setup

\$ g4d etcd-ssl

\$ `PROJECT=alloydb-nova-tvc-sandbox`  
`CLUSTER=satyasais-etcd-tls`  
`NODES="db:3,control"`

`$ cd storage/alloydb/nova/test`

`$ ./gce/cluster-setup.sh create cluster=$CLUSTER nodes="$NODES" project="$PROJECT"`

`$ scp -i ~/.ssh/google_compute_engine  -r ./orchestrator/ ${USER}_google_com@nic0.${CLUSTER}-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com:/tmp/`

control\_zone \= "us-central1-c"  
environment\_name \= "satyasais-etcd-tls"  
project\_id \= "alloydb-nova-tvc-sandbox"

\$ gcloud compute instances list \--project=alloydb-nova-tvc-sandbox \--filter="name\~satyasais-etcd-tls-"  
NAME                        ZONE           MACHINE\_TYPE   PREEMPTIBLE  INTERNAL\_IP  EXTERNAL\_IP  STATUS  
satyasais-etcd-tls-db3      us-central1-a  n2-highmem-16               10.1.0.2                  RUNNING  
satyasais-etcd-tls-db2      us-central1-b  n2-highmem-16               10.1.0.3                  RUNNING  
satyasais-etcd-tls-control  us-central1-c  n2-standard-2               10.1.0.5                  RUNNING  
satyasais-etcd-tls-db1      us-central1-c  n2-highmem-16               10.1.0.4                  RUNNING

Run the following command to log into the control VM  
  \$ ssh \-o StrictHostKeyChecking=no   satyasais\_google\_com@nic0.satyasais-etcd-tls-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

Note: Find cluster setup scripts in /tmp/ directory in your control VM

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.`satyasais-etcd-tls`-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.`satyasais-etcd-tls[`-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-1-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.`satyasais-etcd-tls[`-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-1-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)

## **1\. \[Security\]\[TLS\] Generate the Shared Certificates**

Perform these steps once on your local machine. These certs will serve both etcd and CM.

### **Create the CA and Node Certificate**

Bash

```
# 1. Create CA
openssl genrsa -out rootca.key 2048
openssl req -x509 -new -nodes -key rootca.key -sha256 -days 3650 -out rootca.crt -subj "/CN=AlloyDB-CA"

# 2. Create SAN Config (Include all node IPs)
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
subjectAltName = @alt_names
[alt_names]
IP.1 = 127.0.0.1
IP.2 = 10.1.0.3
IP.3 = 10.1.0.4
IP.4 = 10.1.0.6
EOF

# 3. Create Signed Node Certificate
openssl genrsa -out etcd.key 2048
openssl req -new -key etcd.key -out etcd.csr -subj "/CN=alloydb-node" -config openssl.cnf
openssl x509 -req -in etcd.csr -CA rootca.crt -CAkey rootca.key -CAcreateserial \
  -out etcd.crt -days 365 -sha256 -extensions v3_req -extfile openssl.cnf
```

ETCD cl:  
[https\://critique.corp.google.com/cl/854483444](https://critique.corp.google.com/cl/854483444) 

\-\> Copy certs to db nodes  
\-\> Modify etcd.config with cert locations.  
\-\> restart etcd service

To allow the **AlloyDB Omni Cluster Manager** to write operations to your now-secured `etcd` cluster, you must update its configuration file to use `https` endpoints and point to your SSL certificates.

### **1\. Update the `cluster_manager.yaml` file**

Modify your configuration file (usually located at `/etc/alloydb/cluster_manager/cluster_manager.yaml`) to include the following lines under the `dcs` block.

YAML

```
# Distributed Consensus System (DCS) configuration.
dcs:
    # The type of DCS to use (e.g., etcd).
    type: etcd
    # Timeout for dialing the DCS.
    dial_timeout: 10
    # List of DCS endpoint addresses (USE HTTPS)
    endpoints:
        - "https://10.1.0.3:2379"
        - "https://10.1.0.4:2379"
        - "https://10.1.0.6:2379"
    # SSL/TLS Configuration for Cluster Manager to talk to ETCD
    tls_ca_file: "/var/lib/etcd/ssl/rootca.crt"
    tls_cert_file: "/var/lib/etcd/ssl/etcd.crt"
    tls_key_file: "/var/lib/etcd/ssl/etcd.key"
    # Time-to-live before DCS leader lock expires
    ttl: 30
```

### **2\. Apply the Changes**

After saving the file, you must restart the Cluster Manager service for it to pick up the new secure connection settings:

Bash

```
sudo systemctl restart alloydbomni-cluster-manager
```

### **3\. How to confirm it's working**

Check the logs of the Cluster Manager. If it cannot connect to `etcd`, you will see "context deadline exceeded" or "certificate signed by unknown authority" errors.

Bash

```
sudo journalctl -u alloydbomni-cluster-manager -f
```

If it is working correctly, you should see logs indicating it has successfully connected to the DCS and is participating in leadership election (e.g., `"successfully acquired leader lock"`).

**4\. Cluster manager will throw errors if we don’t add etcd certs:**

Gpaste: [https\://paste.googleplex.com/5499126093578240](https://paste.googleplex.com/5499126093578240) 

**Configure TLS Certificates for CM Server**

1. ## **Overview**

This task secures the communication channel between the **AlloyDB Omni Cluster Manager (CM)** and the **DCS**, which is powered by `etcd`. By implementing Mutual TLS (mTLS), we ensure that only authorized Cluster Manager instances can modify the cluster state, participate in leader elections, or trigger failovers.

2. ## **Technical Context**

In our current deployment, the Cluster Manager and `etcd` reside on the same set of nodes. While they share the same physical host, the communication occurs over the network stack via IP addresses.

* **Role of Ansible:** Ansible acts as the **Certificate Orchestrator**. It generates the certificates and injects the security configuration into the `cluster_manager.yaml` file.  
* **Role of Python Orchestrator:** The Python team's logic manages the operational lifecycle of the CM. Our Ansible tasks are designed to be **non-destructive**, using `blockinfile` to ensure TLS settings persist even after Python-driven updates.

3. ## **Key Configuration Components**

To achieve a secure handshake, the following parameters are injected into the dcs: section of the CM configuration:

| Parameter | Purpose |
| :---- | :---- |
| *endpoints* | Switched from http to https to trigger the TLS handshake. |
| *tls\_ca\_file* | The Root CA certificate used to verify the identity of the etcd server. |
| *tls\_cert\_file* | The Client Certificate that proves the CM's identity to etcd. |
| *tls\_key\_file* | The Private Key used to sign the CM's authentication requests. |

4. ## **Implementation Strategy**

We have adopted a **shared-certificate model**. Due to the co-location of the Cluster Manager (CM) and `etcd` on the same nodes, we leverage the pre-existing node-specific certificates generated for `etcd`. This approach streamlines key management while preserving comprehensive encryption.

```
---
- name: Prepare for installing {{ alloydbomni_cluster_manager.package_name }} from {{ alloydbomni_cluster_manager.repo_url }}
  ansible.builtin.include_tasks: "{{ role_path }}/tasks/install_package.yml"
  vars:
    repo_url: "{{ alloydbomni_cluster_manager.repo_url }}"
    package_name: "{{ alloydbomni_cluster_manager.package_name }}"
    package_version: "{{ alloydbomni_cluster_manager.version | default('') }}"
    repo_gpg_key_url: "{{ alloydbomni_cluster_manager.repo_gpg_key_url | default('') }}"
  when: inventory_hostname in cluster_manager_nodes

- name: Inject real ETCD endpoints and TLS config
  ansible.builtin.blockinfile:
    path: /etc/alloydbomni/cluster_manager.yaml
    insertafter: '^\s*type:\s*etcd'
    marker: "# {mark} ANSIBLE MANAGED ENDPOINTS & TLS #"
    block: |
        {% if etcd.ssl.enabled | default(false) %}
            # Enforce HTTPS Endpoints
            endpoints:
        {% for host in groups['cluster_manager_nodes'] %}
              - https://{{ hostvars[host]['ansible_default_ipv4']['address'] }}:2379
        {% endfor %}
            tls_ca_file: "/var/lib/etcd/ssl/rootca.crt"
            tls_cert_file: "/var/lib/etcd/ssl/etcd.crt"
            tls_key_file: "/var/lib/etcd/ssl/etcd.key"
        {% endif %}
  when: 
    - inventory_hostname in cluster_manager_nodes
    - etcd.ssl.enabled | default(false)
  notify: Restart cluster manager service

- name: Enable and start cluster manager service
  ansible.builtin.systemd:
    name: alloydbomni_cluster_manager
    state: started
    enabled: true
  when: inventory_hostname in cluster_manager_nodes
 
```

Even with the certs in place, the Cluster Manager will fail if it still tries to use `http` instead of `https`. Since the Python team might be generating the `endpoints:` list, you should add one more task to ensure the protocol is updated to `https`:

YAML

```
- name: Add https prefix to DCS endpoints
  ansible.builtin.replace:
    path: /etc/alloydbomni/cluster_manager.yaml
    # This regex looks for: 
    # 1. Indentation (\s+)
    # 2. A hyphen and a space (-\s)
    # 3. An IP address pattern (\d+\.\d+\.\d+\.\d+)
    regexp: '^(\s+-\s+)(\d+\.\d+\.\d+\.\d+:\d+)'
    replace: '\1https://\2'
  when: 
    - inventory_hostname in cluster_manager_nodes
    - etcd.ssl.enabled | default(false)
  notify: Restart cluster manager service

```

5. **Verification:**

Check the logs of the Cluster Manager. If it cannot connect to `etcd`, you will see "context deadline exceeded" or "certificate signed by unknown authority" errors.

Bash

```
sudo journalctl -u alloydbomni-cluster-manager -f
```

If it is working correctly, you should see logs indicating it has successfully connected to the DCS and is participating in leadership election (e.g., `"successfully acquired leader lock"`).

Questions:

1. If we need to configure Cluster manager with a separate TLS connection, then we need to create a different keys, then that will be a different scenario.

**Configure TLS Certificates for Data Path Services**

The **Data Path Services** encompass the **Node Manager** and the **AlloyDB Omni Database engine**. This task ensures that the internal management traffic (Node Manager commands) and the database engine connections are encrypted and authenticated.

Data Path Services refers to the connection from the application to the database and the internal replication traffic.

1. ## **Implementation Logic**

* **Cert Reuse:** We utilize the existing etcd/cluster-wide certificates located at `/var/lib/etcd/ssl/`.  
* **Ansible's Role:** 1\. Ensure certificates are present on all DB nodes (already done in etcd task). 2\. Update the `DBCluster` resource spec to include `certPath`, `keyPath`, and `caPath`.  
* **Python's Role:** The `alloydb_cluster` module reads the spec and restarts the Node Manager with TLS enabled.

**Update your `resource_spec.yaml` task like this:**

```
- name: Create resource_spec.yaml with Data Path TLS
  ansible.builtin.copy:
    dest: "/tmp/resource_spec.yaml"
    content: |
      ---
      DBCluster:
        metadata:
          name: "{{ cluster_name }}"
        spec:
          primarySpec:
            # THIS SECURES THE DATA PATH (Node Manager & DB)
            tls:
              certPath: "/var/lib/etcd/ssl/etcd.crt"
              keyPath: "/var/lib/etcd/ssl/etcd.key"
              caPath: "/var/lib/etcd/ssl/rootca.crt"
            adminUser:
              passwordRef:
                name: db-pw-{{ cluster_name }}
```

2. ## **Verification**

A successful Data Path TLS configuration is confirmed when the Node Manager logs show: `"gRPC server listening with TLS enabled on :6700"`

**The resulting `node_manager.toml` will look like this:**

```
[grpcsrv]
listen_addr = ":6700"
no_tls = false                # Changed by Python
tls_cert = "/var/lib/etcd/ssl/etcd.crt"
tls_key = "/var/lib/etcd/ssl/etcd.key"
client_ca = ["/var/lib/etcd/ssl/rootca.crt"]
```

TLS KT

[Standup- AlloyDB - 2026/03/11 10:00 GMT+05:30 - Recording](https://drive.google.com/file/d/1z6WS74RMnjwV8oq63qfp2mwdiw9lgsO-/view?usp=sharing&resourcekey=0-Dbj11PASErbGKE76I3hrZw) Recording by aman

