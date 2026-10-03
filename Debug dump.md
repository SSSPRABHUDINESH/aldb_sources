Code for complete tar creation:

Command to execute: 

```sh
./alloydomni_invd_debug.sh collect --output-dir /tmp --timestamp 20260115_154400
sudo ./alloydomni_invd_debug.sh collect --output-dir /tmp --timestamp $(date +%Y%m%d_%H%M%S)
```

Command to unzip: 

```sh
tar -xzf /tmp/debug_satyasais-failover-db1_20260115_154400.tar.gz && for f in *.tar.gz; do tar -xzf "$f" && rm "$f"; done
```

Alternative: Unzip to a specific directory

```sh
mkdir -p extracted_debug && tar -xzf /tmp/debug_satyasais-failover-db1_20260115_154400.tar.gz -C extracted_debug && cd extracted_debug && for f in *.tar.gz; do tar -xzf "$f" && rm "$f"; done
```

Code inside \``` alloydomni_invd_debug.sh` ``:

```sh
#!/bin/bash

set -euo pipefail

cleanup() {
    if [[ -d "${BASE_DIR:-}/alloydbomni" ]]; then
        echo "[INFO] Cleaning up temporary directory..."
        rm -rf "${BASE_DIR}/alloydbomni"
    fi
}
trap cleanup EXIT

BASE_DIR=""
TIMESTAMP=""
METRICS_URL="http://localhost:9187/metrics"
DATA_ROOT_DIR="/data"
LOG_LINES="1000"
LOG_SINCE=""
LOG_UNTIL=""
LOG_PRIORITY=""
CAPTURE_DB_LOGS=true
SERVICE_MAPS=()
PROCESSED_SERVICES=()

PGBACKREST_BIN=$(command -v pgbackrest || echo "/usr/bin/pgbackrest")
HAPROXY_SOCKET="/var/lib/haproxy/stats"

usage() {
    echo "Usage: $0 collect --output-dir <base_path> --timestamp <YYYYMMDD_HHMMSS> [OPTIONS]"
    echo ""
    echo "Mapping Options:"
    echo "  --map \"svc:conf:log\"       Override paths. Example: --map \"haproxy:/etc/h.cfg:/var/log/h.log\""
    echo ""
    echo "Log Selection Options:"
    echo "  --capture-db-logs <true|false>  Capture internal DB logs for alloydbomni (default: true)"
    echo ""
    echo "Log Filter Options (applies to systemd services):"
    echo "  -n, --lines <N>           Last N lines to capture (default: 1000)"
    echo "  --since <T>             Logs since 'YYYY-MM-DD HH:MM:SS' or relative ('1h', '30m')"
    echo "  --until <T>             Logs until a specific time"
    echo "  -p, --priority <LVL>      emerg, crit, err, warning, info, debug"
    exit 1
}

if [[ "${1:-}" != "collect" ]]; then usage; fi
shift

while [[ $# -gt 0 ]]; do
    case $1 in
        -o|--output-dir)      BASE_DIR="$2"; shift 2 ;;
        -t|--timestamp)       TIMESTAMP="$2"; shift 2 ;;
        --map)                SERVICE_MAPS+=("$2"); shift 2 ;;
        --metrics-url)        METRICS_URL="$2"; shift 2 ;;
        --capture-db-logs)    CAPTURE_DB_LOGS="$2"; shift 2 ;;
        -n|--lines)           LOG_LINES="$2"; shift 2 ;;
        --since)              LOG_SINCE="$2"; shift 2 ;;
        --until)              LOG_UNTIL="$2"; shift 2 ;;
        -p|--priority)        LOG_PRIORITY="$2"; shift 2 ;;
        *) echo "Unknown option: $1" >&2; usage ;;
    esac
done

if [[ -z "$BASE_DIR" || -z "$TIMESTAMP" ]]; then usage; fi

HOSTNAME=$(hostname)
OUTPUT_DIR="${BASE_DIR}/alloydbomni/${HOSTNAME}"
mkdir -p "$OUTPUT_DIR"

get_journal_args() {
    local args="-n ${LOG_LINES} --no-pager"
    [[ -n "$LOG_SINCE" ]]    && args="${args} --since '${LOG_SINCE}'"
    [[ -n "$LOG_UNTIL" ]]    && args="${args} --until '${LOG_UNTIL}'"
    [[ -n "$LOG_PRIORITY" ]] && args="${args} -p ${LOG_PRIORITY}"
    echo "$args"
}

smart_copy() {
    local src=$1
    local dest_dir=$2
    [[ -z "$src" ]] && return
    if [[ -e "$src" ]]; then
        mkdir -p "$dest_dir"
        if [[ -d "$src" ]]; then
            cp -rf "$src/." "$dest_dir/" 2>/dev/null || true
        else
            cp -f "$src" "$dest_dir/" 2>/dev/null || true
        fi
    fi
}

collect_system_info() {
    local sys_dir="$OUTPUT_DIR/system_info"
    mkdir -p "$sys_dir"

    echo "[INFO] Collecting System Info..."
    {
        set +e
        echo "=== OS INFORMATION (uname -a) ==="
        uname -a
        echo -e "\n=== KERNEL LOGS (journalctl -k) ==="
        journalctl -k --no-pager
        echo -e "\n=== PROCESS INFO (top) ==="
        top -b -n 1 | head -n 50
        echo -e "\n=== PROCESS INFO (ps) ==="
        ps -eo pid,ppid,user,%cpu,%mem,rsz,comm --sort=-%cpu
        echo -e "\n=== DISK INFO (df -lh) ==="
        df -lh
        echo -e "\n=== VMSTAT ==="
        vmstat 1 5
        echo -e "\n=== MEMORY (free -m) ==="
        free -m
        echo -e "\n=== NETWORK (ip addr) ==="
        ip addr
        echo -e "\n=== SS (ss -tunp) ==="
        ss -tunp
        echo -e "\n=== INSTALLED RPMS ==="
        rpm -qa | grep -E 'alloydb|etcd|haproxy|keepalived|pgbouncer|pgbackrest' | sort
        set -e
    } > "$sys_dir/diagnostics.out" 2>&1

    if command -v dnf >/dev/null; then
        dnf history > "$sys_dir/dnf_history.out" 2>&1 || true
    fi
}

process_service() {
    local name=$1
    local custom_conf=${2:-}
    local custom_log=${3:-}
    local comp_dir="$OUTPUT_DIR/$name"

    mkdir -p "$comp_dir/configs" "$comp_dir/logs"
    echo "[INFO] Processing $name..."
    PROCESSED_SERVICES+=("$name")

    systemctl status "$name" --no-pager > "$comp_dir/systemd-status.out" 2>&1 || true

    if [[ "$name" == "pgbackrest" ]]; then
        echo "[INFO] Collecting pgBackRest info..."
        "$PGBACKREST_BIN" info > "$comp_dir/pgbackrest_info.out" 2>&1 || echo "[WARN] pgBackRest info failed"
    fi

    if [[ "$name" == "haproxy" ]]; then
        echo "[INFO] Collecting HAProxy stats via Socket..."
        if [[ -S "$HAPROXY_SOCKET" ]]; then
            local raw_csv="$comp_dir/backend_stats.csv"
            python3 -c "import socket; s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM); s.connect('$HAPROXY_SOCKET'); s.send(b'show stat\n'); print(s.makefile().read()); s.close()" > "$raw_csv" 2>/dev/null || echo "[WARN] HAProxy socket read failed"
            if [[ -f "$raw_csv" ]]; then
                {
                    echo "PROXY_NAME,SERVICE_NAME,CUR_CONN,TOT_CONN,STATUS,LAST_CHECK"
                    grep -v '^#' "$raw_csv" | cut -d',' -f1,2,5,8,18,58 | column -s, -t
                } > "$comp_dir/backend_stats_summary.txt" 2>/dev/null || true
            fi
        else
            echo "[WARN] HAProxy socket not found at $HAPROXY_SOCKET"
        fi
    fi

    if [[ -n "$custom_conf" ]]; then
        smart_copy "$custom_conf" "$comp_dir/configs"
    else
        case "$name" in
            "haproxy") cp -f /etc/haproxy/conf.d/*.cfg "$comp_dir/configs/" 2>/dev/null || true ;;
            "pgbouncer") smart_copy "/etc/pgbouncer/pgbouncer.ini" "$comp_dir/configs" ;;
            "keepalived") smart_copy "/etc/keepalived/keepalived.conf" "$comp_dir/configs" ;;
            "alloydbomni_node_manager") smart_copy "/etc/alloydbomni/node_manager.toml" "$comp_dir/configs" ;;
            "alloydbomni_cluster_manager") smart_copy "/etc/alloydbomni/cluster_manager.yaml" "$comp_dir/configs" ;;
            "etcd") smart_copy "/etc/etcd/etcd.conf" "$comp_dir/configs" ;;
            "pgbackrest") smart_copy "/etc/pgbackrest/pgbackrest.conf" "$comp_dir/configs" ;;
            alloydbomni[0-9]*)
                local v
                v=$(echo "$name" | grep -o '[0-9]\+')
                smart_copy "${DATA_ROOT_DIR}/${v}/postgresql.conf" "$comp_dir/configs"
                ;;
        esac
    fi

    local j_args
    j_args=$(get_journal_args)
    eval "journalctl -u ${name} ${j_args}" > "$comp_dir/logs/journal.log" 2>&1 || true

    if [[ -n "$custom_log" ]]; then
        smart_copy "$custom_log" "$comp_dir/logs"
    else
        case "$name" in
            "haproxy") smart_copy "/var/log/haproxy.log" "$comp_dir/logs" ;;
            "keepalived") smart_copy "/var/log/keepalived.log" "$comp_dir/logs" ;;
            "pgbouncer") smart_copy "/var/log/pgbouncer/pgbouncer.log" "$comp_dir/logs" ;;
            "pgbackrest")
                smart_copy "/var/log/pgbackrest" "$comp_dir/logs/pgbackrest_dir"
                smart_copy "/var/log/pgbackrest/archive-push.log" "$comp_dir/logs"
                ;;
            alloydbomni[0-9]*)
                smart_copy "/var/log/alloydbomni" "$comp_dir/logs/alloydbomni_system"

                if [[ "$CAPTURE_DB_LOGS" = true ]]; then
                    local v
                    v=$(echo "$name" | grep -o '[0-9]\+')
                    local db_log_dir="${DATA_ROOT_DIR}/${v}/log"
                    if [[ -d "$db_log_dir" ]]; then
                        mkdir -p "$comp_dir/logs/db_internal"
                        find "$db_log_dir" -maxdepth 1 -type f \( -name "*.log" -o -name "*.internal" \) ! -name "*audit*" -exec cp -f {} "$comp_dir/logs/db_internal/" \; 2>/dev/null || true
                    fi
                fi
                ;;
        esac
    fi

    [[ "$name" == "alloydbomni_monitor" ]] && curl -s --connect-timeout 5 "${METRICS_URL}" > "$comp_dir/metrics.out" 2>&1 || true
}

collect_services() {
    if [[ ${#SERVICE_MAPS[@]} -gt 0 ]]; then
        for entry in "${SERVICE_MAPS[@]}"; do
            if [[ "$entry" != *:* ]]; then
                echo "[ERROR] Invalid map format: $entry. Use svc:conf:log" >&2
                continue
            fi
            IFS=':' read -r s_name s_conf s_log <<< "$entry"
            if rpm -q "$s_name" >/dev/null 2>&1; then
                process_service "$s_name" "$s_conf" "$s_log"
            else
                echo "[WARN] Service $s_name specified in map is not installed. Skipping..."
            fi
        done
    else
        local SERVICES=("haproxy" "pgbouncer" "keepalived" "pgbackrest" "alloydbomni_monitor" "alloydbomni_cluster_manager" "alloydbomni_node_manager" "etcd")
        mapfile -t DB_PKGS < <(rpm -qa --queryformat '%{NAME}\n' | grep '^alloydbomni[0-9]\+$' || true)
        for pkg in "${DB_PKGS[@]}"; do process_service "$pkg"; done
        for svc in "${SERVICES[@]}"; do
            if rpm -q "$svc" >/dev/null 2>&1; then
                process_service "$svc"
            fi
        done
        true
    fi
}

create_tarball() {
    echo -e "\n--- COLLECTION SUMMARY ---"
    printf "%-30s | %-10s\n" "Service Name" "Status"
    echo "---------------------------------------------"
    for s in "${PROCESSED_SERVICES[@]}"; do
        printf "%-30s | %-10s\n" "$s" "COLLECTED"
    done
    echo "---------------------------------------------"

    local node_root="${BASE_DIR}/alloydbomni/${HOSTNAME}"
    local staging_dir="${BASE_DIR}/tar_staging_${TIMESTAMP}"
    mkdir -p "$staging_dir"

    echo "[INFO] Creating component-level tarballs (3rd Level)..."

    # 1. Tar the system_info directory
    if [[ -d "$node_root/system_info" ]]; then
        tar -czf "$staging_dir/system_info.tar.gz" -C "$node_root" "system_info"
    fi

    # 2. Tar each service directory individually
    for service in "${PROCESSED_SERVICES[@]}"; do
        if [[ -d "$node_root/$service" ]]; then
            echo "  -> Compressing $service..."
            tar -czf "$staging_dir/${service}.tar.gz" -C "$node_root" "$service"
        fi
    done

    # 3. Create the Node-level tarball containing the component tarballs (2nd Level)
    local tar_name="debug_${HOSTNAME}_${TIMESTAMP}.tar.gz"
    echo "[INFO] Creating node-level tarball: $tar_name"
   
    # We tar the contents of the staging directory
    tar -czf "${BASE_DIR}/${tar_name}" -C "$staging_dir" .

    echo "[SUCCESS] Tarball location: ${BASE_DIR}/${tar_name}"

    # Cleanup staging
    rm -rf "$staging_dir"
}

collect_system_info
collect_services
create_tarball

```

1. NOVA Orchestrator spec doc: [NOVA Orchestrator Spec](https://docs.google.com/document/d/1zLnq1wbvWognZkJPREpV4rLmb74FAttXvpwBnoKllvs/edit?tab=t.iv5pu4wkveic)  
2. alloydbomni\_cm\_dump bash scrip with all sourcest: [https\://paste.googleplex.com/4878109314777088](https://paste.googleplex.com/4878109314777088)   
3. alloydbomni\_nm\_dump bash script: [https\://paste.googleplex.com/4667401138470912](https://paste.googleplex.com/4667401138470912)  
4. Output for nm\_dump code: [https\://paste.googleplex.com/6594275972349952](https://paste.googleplex.com/6594275972349952)  
5. Debug dump role: [https\://critique.corp.google.com/cl/853087881](https://critique.corp.google.com/cl/853087881)  
6. Output for debug dump role: [https\://paste.googleplex.com/4803358974148608\#l=80](https://paste.googleplex.com/4803358974148608#l=80)  
7. CL with all collection sources for cm and nm scripts: [https\://critique.corp.google.com/cl/858077015/analysis](https://critique.corp.google.com/cl/858077015/analysis) in first snapshot  
8. Snapshot 3: CL: same 7th. 

\$ g4d debug-dump

\$ `PROJECT=alloydb-nova-tvc-sandbox`  
`CLUSTER=satyasais-debug-dump`  
`NODES="db:3,haproxy:2,control"`

`$ cd storage/alloydb/nova/test`

`$ ./gce/cluster-setup.sh create cluster=$CLUSTER nodes="$NODES" project="$PROJECT"`

`$ scp -i ~/.ssh/google_compute_engine  -r ./orchestrator/ ${USER}_google_com@nic0.${CLUSTER}-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com:/tmp/`

control\_zone \= "us-central1-c"  
environment\_name \= "satyasais-debug-dump"  
project\_id \= "alloydb-nova-tvc-sandbox"

\$ popd  
/google/src/cloud/satyasais/debug-dump/google3/storage/alloydb/nova/test

\$ gcloud compute instances list \--project=alloydb-nova-tvc-sandbox \--filter="name\~satyasais-debug-dump-"  
NAME                           ZONE           MACHINE\_TYPE   PREEMPTIBLE  INTERNAL\_IP  EXTERNAL\_IP  STATUS  
satyasais-debug-dump-db3       us-central1-a  n2-highmem-16               10.1.0.4                  RUNNING  
satyasais-debug-dump-db2       us-central1-b  n2-highmem-16               10.1.0.6                  RUNNING  
satyasais-debug-dump-haproxy2  us-central1-b  n2-standard-2               10.1.0.2                  RUNNING  
satyasais-debug-dump-control   us-central1-c  n2-standard-2               10.1.0.7                  RUNNING  
satyasais-debug-dump-db1       us-central1-c  n2-highmem-16               10.1.0.3                  RUNNING  
satyasais-debug-dump-haproxy1  us-central1-c  n2-standard-2               10.1.0.5                  RUNNING

Run the following command to log into the control VM  
  \$ ssh \-o StrictHostKeyChecking=no   satyasais\_google\_com@nic0.satyasais-debug-dump-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

Note: Find cluster setup scripts in /tmp/ directory in your control VM

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.`satyasais-debug-dump`-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.`satyasais-debug-dump[`-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-1-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.`satyasais-debug-dump[`-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-1-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)

**Installing and bootstrapping the cluster:**

`$ /tmp/setup-ssh-for-cluster.sh`

`$ /tmp/orchestrator/alloydb-setup.sh prepare -c /tmp/cluster.conf`

`$ /tmp/orchestrator/alloydb-setup.sh create -c /tmp/cluster.conf`

**`On db1:`** 

`$ vi alloydbomni_cm_dump.sh`

`Copy contents and paste: https://paste.googleplex.com/4878109314777088`  
   
`$ vi alloydbomni_nm_dump.sh`

`Copy contents and paste: https://paste.googleplex.com/4667401138470912`

\$ sudo sh alloydbomni\_nm\_dump.sh \-out\=/tmp/debug\_dump \-tag\=20250114 \-c\=all

\$ sudo sh alloydbomni\_cm\_dump.sh \-out\=/tmp/debug\_dump \-tag\=20250114 \-c\=all

\$ tar \-tvf /tmp/debug\_dump/alloydbomni\_20250114.tgz

Results: [https\://paste.googleplex.com/6240704399540224](https://paste.googleplex.com/6240704399540224)

Creating a custom collection with debug\_dump role:

\$ g4d testing\_debug\_dump\_role

\$ mkdir /tmp/orchestrator/ansible/output

\$ storage/alloydb/nova/automation/ansible/build\_ansible\_collection.sh \-v 0.0.1

\$ ls /tmp/orchestrator/ansible/output  
google-alloydbomni\_orchestrator-0.0.1.tar.gz

\$ PROJECT=alloydb-nova-tvc-sandbox  
CLUSTER=satyasais-debug-role  
NODES="db:3,haproxy:2,control"

\$ cd storage/alloydb/nova/test/

\$ ./gce/cluster-setup.sh create cluster=\$CLUSTER nodes="\$NODES" project="\$PROJECT"

```
control_zone = "us-central1-c"
environment_name = "satyasais-debug-role"
project_id = "alloydb-nova-tvc-sandbox"


$ popd
/google/src/cloud/satyasais/testing_debug_dump_role/google3/storage/alloydb/nova/test


$ gcloud compute instances list --project=alloydb-nova-tvc-sandbox --filter="name~satyasais-debug-role-"
NAME                           ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
satyasais-debug-role-db3       us-central1-a  n2-highmem-16               10.1.0.2                  RUNNING
satyasais-debug-role-db2       us-central1-b  n2-highmem-16               10.1.0.4                  RUNNING
satyasais-debug-role-haproxy2  us-central1-b  n2-standard-2               10.1.0.3                  RUNNING
satyasais-debug-role-control   us-central1-c  n2-standard-2               10.1.0.7                  RUNNING
satyasais-debug-role-db1       us-central1-c  n2-highmem-16               10.1.0.5                  RUNNING
satyasais-debug-role-haproxy1  us-central1-c  n2-standard-2               10.1.0.6                  RUNNING

Run the following command to log into the control VM
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-role-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com
```

\$ scp \-i \~/.ssh/google\_compute\_engine  \-r ./orchestrator/ \${USER}\_google\_com@nic0.\${USER}-debug-role[\-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](http://failover-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

```
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-role-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-role-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-role-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-role-haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com
```

\$ scp \-i \~/.ssh/google\_compute\_engine  \-r /tmp/orchestrator/ansible/output/google-alloydbomni\_orchestrator-0.0.1.tar.gz \${USER}\_google\_com@nic0.\${CLUSTER}-[control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](http://control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com/):/tmp/

Inside Control node:  
\$ /tmp/setup-ssh-for-cluster.sh

\$ /tmp/orchestrator/[alloydb-setup.sh](http://alloydb-setup.sh/) prepare \-c /tmp/cluster.conf ssl="enabled" ansible\_collection\_path=/tmp/google-alloydbomni\_orchestrator-0.0.1.tar.gz

\$ /tmp/setup-ssh-for-cluster.sh

/tmp/orchestrator/alloydb-setup.sh prepare \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-setup.sh create \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-setup.sh status \-c /tmp/cluster.conf

Copying bash scripts on all nodes from workspace::

\$ scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_cm\_dump.sh [`satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

\$ scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_nm\_dump.sh [`satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

On db2:  
scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_cm\_dump.sh [`satyasais_google_com@nic0.`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-role-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`:/tmp/  
scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_nm\_dump.sh [`satyasais_google_com@nic0.satyasais-debug-role-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-role-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

On db3:  
scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_cm\_dump.sh [`satyasais_google_com@nic0.`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-role-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`:/tmp/  
scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_nm\_dump.sh [`satyasais_google_com@nic0.satyasais-debug-role-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-role-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

On haproxy node:

scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_nm\_dump.sh [`satyasais_google_com@nic0.satyasais-debug-role-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-role-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

On haproxy node 2:

scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_nm\_dump.sh [`satyasais_google_com@nic0.satyasais-debug-role-`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`:/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-role-`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com` "chmod \+x /tmp/alloydbomni\_\*.sh"

Inside control node:  
Creating an dump\_debug.yaml:

```
---
- name: Collect Debug Logs and System Status from AlloyDB Omni Cluster
  hosts: all
  become: true
  gather_facts: true

  vars:
    # You can override defaults here for specific test runs
    dump_debug_collection_type: "status"
    dump_debug_tag: "test_run_{{ ansible_date_time.date }}"

  roles:
    - role: google.alloydbomni_orchestrator.dump_debug

  post_tasks:
    - name: Final Confirmation
      ansible.builtin.debug:
        msg: "Collection workflow complete. Please check {{ dump_debug_local_dest }} for the final bundle."
      run_once: true
```

Inside control node:  
Creating hosts file:

\# control  
ansible-galaxy collection install community.postgresql  
sudo dnf install \-y python3-psycopg2

\#create host file  
cp deployment\_spec.yaml hosts.yaml  
vi hosts.yaml

\#Replace SA and add in hosts.yaml  
ansible\_user: sa\_106801092749283222743  
ansible\_ssh\_private\_key\_file: /home/satyasais\_google\_com/ssh-key-cluster-sa

\#Running dump\_debug.yaml:  
   
ansible-playbook \-i hosts.yaml dump\_debug.yaml

tar \-tvf alloydbomni\_cluster\_20260120\_142333.tar.gz

For destroying:

`$ /tmp/orchestrator/alloydb-setup.sh destroy -c /tmp/cluster.conf`  
`$ rm -rf /tmp/orchestrator/ansible/output`

Would you like me to update the Ansible `tasks/main.yml` to reflect these single-letter flag changes as well?

## **TRAIL 2:**

Creating a custom collection with debug\_dump role:

\$ g4d \-f dump\_debug\_2

\$ mkdir /tmp/orchestrator/ansible/output

\$ storage/alloydb/nova/automation/ansible/build\_ansible\_collection.sh \-v 0.0.1

\$ ls /tmp/orchestrator/ansible/output  
google-alloydbomni\_orchestrator[\-0.0.1.tar.gz](http://-0.0.1.tar.gz)

\$ rm /tmp/orchestrator/ansible/output  
google-alloydbomni\_orchestrator[\-0.0.1.tar.gz](http://-0.0.1.tar.gz)

\$ PROJECT=alloydb-nova-tvc-sandbox  
CLUSTER=satyasais-dbug2-role  
NODES="db:3,haproxy:2,control"

\$ cd storage/alloydb/nova/test/

\$ ./gce/cluster-setup.sh create cluster=\$CLUSTER nodes="\$NODES" project="\$PROJECT"

```
control_zone = "us-central1-c"
environment_name = "satyasais-dbug2-role"
project_id = "alloydb-nova-tvc-sandbox"


$ popd
/google/src/cloud/satyasais/dump_debug_2/google3/storage/alloydb/nova/test


$ gcloud compute instances list --project=alloydb-nova-tvc-sandbox --filter="name~satyasais-dbug2-role-"
NAME                           ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
satyasais-dbug2-role-db3       us-central1-a  n2-highmem-16               10.1.0.4                  RUNNING
satyasais-dbug2-role-db2       us-central1-b  n2-highmem-16               10.1.0.2                  RUNNING
satyasais-dbug2-role-haproxy2  us-central1-b  n2-standard-2               10.1.0.6                  RUNNING
satyasais-dbug2-role-control   us-central1-c  n2-standard-2               10.1.0.7                  RUNNING
satyasais-dbug2-role-db1       us-central1-c  n2-highmem-16               10.1.0.5                  RUNNING
satyasais-dbug2-role-haproxy1  us-central1-c  n2-standard-2               10.1.0.3                  RUNNING

Run the following command to log into the control VM
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug2-role-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com
```

\$ scp \-i \~/.ssh/google\_compute\_engine  \-r ./orchestrator/ \${USER}\_google\_com@nic0.\${CLUSTER}[\-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](http://failover-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

```
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug2-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug2-role-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug2-role-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug2-role-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug2-role-haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com
```

\$ scp \-i \~/.ssh/google\_compute\_engine  \-r /tmp/orchestrator/ansible/output/google-alloydbomni\_orchestrator-0.0.1.tar.gz \${USER}\_google\_com@nic0.\${CLUSTER}-[control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](http://control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com/):/tmp/

Inside Control node:  
\$ /tmp/setup-ssh-for-cluster.sh

\$ /tmp/orchestrator/[alloydb-ansible.sh](http://alloydb-setup.sh/) prepare \-c /tmp/cluster.conf ansible\_collection\_path=/tmp/google-alloydbomni\_orchestrator-0.0.1.tar.gz

\$ /tmp/setup-ssh-for-cluster.sh

/tmp/orchestrator/alloydb-ansible.sh create \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-ansible.sh status \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-ansible.sh destroy \-c /tmp/cluster.conf

Copying bash scripts on all nodes from workspace::

sudo vi /usr/local/bin/alloydbomni\_cm\_dump  
sudo vi /usr/local/bin/alloydbomni\_nm\_dump

sudo chmod \+x /usr/local/bin/alloydbomni\_cm\_dump   
sudo chmod \+x /usr/local/bin/alloydbomni\_nm\_dump

sudo rm /usr/local/bin/alloydbomni\_cm\_dump.sh  
sudo rm /usr/local/bin/alloydbomni\_nm\_dump.sh

\# if new role is not found do as below

```

Running:[satyasais_google_com@satyasais-dbug1-role-control ~]$ ansible-galaxy collection install /tmp/google-alloydbomni_orchestrator-0.0.1.tar.gz --force
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Installing 'google.alloydbomni_orchestrator:0.0.1' to '/home/satyasais_google_com/.ansible/collections/ansible_collections/google/alloydbomni_orchestrator'
google.alloydbomni_orchestrator:0.0.1 was installed successfully
```

Inside control node:  
Creating an dump\_debug.yaml:  
Inside control node:

```sh
---
- name: AlloydbOmni Debug Dump Collection
  hosts: all
  become: true
  gather_facts: true
  vars:
    ansible_user: sa_106769072263793341629
    ansible_ssh_private_key_file: ~/ssh-key-cluster-sa
    dump_debug_collection_type: status
    dump_debug_local_dest: "/home/satyasais_google_com/debug_dump_test"
    dump_debug_tag: "20260227"
  roles:
    - role: google.alloydbomni_orchestrator.dump_debug

```

```
---
- name: Deploy AlloyDB Omni
  hosts: localhost
  vars:
   ansible_user: sa_106665853547205570503
   ansible_ssh_private_key_file: ~/ssh-key-cluster-sa
  roles:
   - role: google.alloydbomni_orchestrator.dump_debug
```

\#Replace SA and add in hosts.yaml  
ansible\_user: sa\_106769072263793341629  
ansible\_ssh\_private\_key\_file: /home/satyasais\_google\_com/ssh-key-cluster-sa

\#Running dump\_debug.yaml:  
   
ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml

tar \-tvf alloydbomni\_cluster\_20260120\_142333.tar.gz

For destroying:

`$ /tmp/orchestrator/alloydb-setup.sh destroy -c /tmp/cluster.conf`  
`$ rm -rf /tmp/orchestrator/ansible/output`

Would you like me to update the Ansible `tasks/main.yml` to reflect these single-letter flag changes as well?

**On control:**

/tmp/orchestrator/alloydb-ansible.sh destroy \-c /tmp/cluster.conf

**On workspace:**  
rm /tmp/orchestrator/ansible/output/google-alloydbomni\_orchestrator[\-0.0.1.tar.gz](http://-0.0.1.tar.gz)

storage/alloydb/nova/automation/ansible/build\_ansible\_collection.sh \-v 0.0.1

**On control:**

rm /tmp/google-alloydbomni\_orchestrator[\-0.0.1.tar.gz](http://-0.0.1.tar.gz)

On workspace:  
scp \-i \~/.ssh/google\_compute\_engine  \-r /tmp/orchestrator/ansible/output/google-alloydbomni\_orchestrator-0.0.1.tar.gz \${USER}\_google\_com@nic0.\${CLUSTER}-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com:/tmp/

\$ /tmp/orchestrator/[alloydb-ansible.sh](http://alloydb-setup.sh/) prepare \-c /tmp/cluster.conf ansible\_collection\_path=/tmp/google-alloydbomni\_orchestrator-0.0.1.tar.gz

/tmp/orchestrator/alloydb-ansible.sh create \-c /tmp/cluster.conf

/tmp/orchestrator/alloydb-ansible.sh status \-c /tmp/cluster.conf

/tmp/orchestrator/alloydb-ansible.sh destroy \-c /tmp/cluster.conf

`ansible-galaxy collection install /tmp/google-alloydbomni_orchestrator-0.0.1.tar.gz --force`

ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml

tar \-tvf /home/satyasais\_google\_com/debug\_dump\_test/alloydbomni\_cluster\_20260227.tar.gz

tar \-xOzf /home/satyasais\_google\_com/debug\_dump\_test/alloydbomni\_cluster\_20260227[.tar.gz](http://.tar.gz) alloydbomni/satyasais-dbug2-role-db1/alloydbomni18/systemd-status.out

Sample command: tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_20260227[.tar.gz](http://.tar.gz)  
tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_20260227.tar.gz alloydbomni/satyasais-debug-sh-db1/alloydbomni18/systemd-status.out

## **TRAIL 3: Without tag**

Creating a custom collection with debug\_dump role:

\$ g4d \-f dump\_debug\_4

\$ mkdir /tmp/orchestrator/ansible/output

\$ storage/alloydb/nova/automation/ansible/build\_ansible\_collection.sh \-v 0.0.1

\$ ls /tmp/orchestrator/ansible/output  
google-alloydbomni\_orchestrator[\-0.0.1.tar.gz](http://-0.0.1.tar.gz)

\$ rm /tmp/orchestrator/ansible/output  
google-alloydbomni\_orchestrator[\-0.0.1.tar.gz](http://-0.0.1.tar.gz)

\$ PROJECT=alloydb-nova-tvc-sandbox  
CLUSTER=satyasais-dbug4-role  
NODES="db:3,haproxy:2,control"

\$ cd storage/alloydb/nova/test/

\$ ./gce/cluster-setup.sh create cluster=\$CLUSTER nodes="\$NODES" project="\$PROJECT"

```
control_zone = "us-central1-c"
environment_name = "satyasais-dbug4-role"
project_id = "alloydb-nova-tvc-sandbox"


$ popd
/google/src/cloud/satyasais/dump_debug_4/google3/storage/alloydb/nova/test


$ gcloud compute instances list --project=alloydb-nova-tvc-sandbox --filter="name~satyasais-dbug4-role-"
NAME                           ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
satyasais-dbug4-role-db3       us-central1-a  n2-highmem-16               10.1.0.5                  RUNNING
satyasais-dbug4-role-db2       us-central1-b  n2-highmem-16               10.1.0.3                  RUNNING
satyasais-dbug4-role-haproxy2  us-central1-b  n2-standard-2               10.1.0.2                  RUNNING
satyasais-dbug4-role-control   us-central1-c  n2-standard-2               10.1.0.7                  RUNNING
satyasais-dbug4-role-db1       us-central1-c  n2-highmem-16               10.1.0.4                  RUNNING
satyasais-dbug4-role-haproxy1  us-central1-c  n2-standard-2               10.1.0.6                  RUNNING

Run the following command to log into the control VM
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug4-role-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com
```

\$ scp \-i \~/.ssh/google\_compute\_engine  \-r ./orchestrator/ \${USER}\_google\_com@nic0.\${CLUSTER}[\-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](http://failover-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

```
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug4-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug4-role-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug4-role-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug4-role-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dbug4-role-haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com
```

\$ scp \-i \~/.ssh/google\_compute\_engine  \-r /tmp/orchestrator/ansible/output/google-alloydbomni\_orchestrator-0.0.1.tar.gz \${USER}\_google\_com@nic0.\${CLUSTER}-[control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](http://control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com/):/tmp/

Inside Control node:  
\$ /tmp/setup-ssh-for-cluster.sh

\$ /tmp/orchestrator/[alloydb-ansible.sh](http://alloydb-setup.sh/) prepare \-c /tmp/cluster.conf ansible\_collection\_path=/tmp/google-alloydbomni\_orchestrator-0.0.1.tar.gz

\$ /tmp/setup-ssh-for-cluster.sh

/tmp/orchestrator/alloydb-ansible.sh create \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-ansible.sh status \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-ansible.sh destroy \-c /tmp/cluster.conf

Copying bash scripts on all nodes from workspace::

sudo vi /usr/local/bin/alloydbomni\_cm\_dump  
sudo vi /usr/local/bin/alloydbomni\_nm\_dump

sudo chmod \+x /usr/local/bin/alloydbomni\_cm\_dump   
sudo chmod \+x /usr/local/bin/alloydbomni\_nm\_dump

sudo rm /usr/local/bin/alloydbomni\_cm\_dump  
sudo rm /usr/local/bin/alloydbomni\_nm\_dump

\# if new role is not found do as below

```

Running:[satyasais_google_com@satyasais-dbug1-role-control ~]$ ansible-galaxy collection install /tmp/google-alloydbomni_orchestrator-0.0.1.tar.gz --force
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Installing 'google.alloydbomni_orchestrator:0.0.1' to '/home/satyasais_google_com/.ansible/collections/ansible_collections/google/alloydbomni_orchestrator'
google.alloydbomni_orchestrator:0.0.1 was installed successfully
```

Inside control node:  
Creating an dump\_debug.yaml:  
Inside control node:

```sh
---
- name: AlloydbOmni Debug Dump Collection
  hosts: all
  become: true
  gather_facts: true
  vars:
    ansible_user: sa_100799427004031040855
    ansible_ssh_private_key_file: ~/ssh-key-cluster-sa
    dump_debug_collection_type: status
    dump_debug_local_dest: "/home/satyasais_google_com/debug_dump_test"
    dump_debug_tag: ""
  roles:
    - role: google.alloydbomni_orchestrator.dump_debug

```

```
---
- name: Deploy AlloyDB Omni
  hosts: localhost
  vars:
   ansible_user: sa_106665853547205570503
   ansible_ssh_private_key_file: ~/ssh-key-cluster-sa
  roles:
   - role: google.alloydbomni_orchestrator.dump_debug
```

\#Replace SA and add in hosts.yaml  
ansible\_user: sa\_106769072263793341629  
ansible\_ssh\_private\_key\_file: /home/satyasais\_google\_com/ssh-key-cluster-sa

\#Running dump\_debug.yaml:  
   
ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml

tar \-tvf alloydbomni\_cluster\_20260120\_142333.tar.gz

For destroying:

`$ /tmp/orchestrator/alloydb-setup.sh destroy -c /tmp/cluster.conf`  
`$ rm -rf /tmp/orchestrator/ansible/output`

Would you like me to update the Ansible `tasks/main.yml` to reflect these single-letter flag changes as well?

**On control:**

/tmp/orchestrator/alloydb-ansible.sh destroy \-c /tmp/cluster.conf

**On workspace:**  
rm /tmp/orchestrator/ansible/output/google-alloydbomni\_orchestrator[\-0.0.1.tar.gz](http://-0.0.1.tar.gz)

storage/alloydb/nova/automation/ansible/build\_ansible\_collection.sh \-v 0.0.1

**On control:**

rm /tmp/google-alloydbomni\_orchestrator[\-0.0.1.tar.gz](http://-0.0.1.tar.gz)

On workspace:  
scp \-i \~/.ssh/google\_compute\_engine  \-r /tmp/orchestrator/ansible/output/google-alloydbomni\_orchestrator-0.0.1.tar.gz \${USER}\_google\_com@nic0.\${CLUSTER}-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com:/tmp/

\$ /tmp/orchestrator/[alloydb-ansible.sh](http://alloydb-setup.sh/) prepare \-c /tmp/cluster.conf ansible\_collection\_path=/tmp/google-alloydbomni\_orchestrator-0.0.1.tar.gz

/tmp/orchestrator/alloydb-ansible.sh create \-c /tmp/cluster.conf

/tmp/orchestrator/alloydb-ansible.sh status \-c /tmp/cluster.conf

/tmp/orchestrator/alloydb-ansible.sh destroy \-c /tmp/cluster.conf

`ansible-galaxy collection install /tmp/google-alloydbomni_orchestrator-0.0.1.tar.gz --force`

ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml

ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml \\  
  \-e 'dump\_debug\_journal\_spec="--since \\"2 hours ago\\""'

tar \-tvf /home/satyasais\_google\_com/debug\_dump\_test/alloydbomni\_cluster\_02032608\_1.tar.gz

tar \-xOzf /home/satyasais\_google\_com/debug\_dump\_test/alloydbomni\_cluster\_02032608\_1[.tar.gz](http://.tar.gz) alloydbomni/satyasais-dbug3-role-db1/alloydbomni18/systemd-status.out

Sample command: tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_20260227[.tar.gz](http://.tar.gz)  
tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_20260227.tar.gz alloydbomni/satyasais-debug-sh-db1/alloydbomni18/systemd-status.out

cd \~/.ansible/collections/ansible\_collections/google/alloydbomni\_orchestrator/roles/dump\_debug/

vi \~/.ansible/collections/ansible\_collections/google/alloydbomni\_orchestrator/roles/dump\_debug/tasks/main.yml

**Commands for journal ctl logs:**

Inside control node:  
Creating an dump\_debug.yaml:  
Inside control node:

```sh
---
- name: AlloydbOmni Debug Dump Collection
  hosts: all
  become: true
  gather_facts: true
  vars:
    ansible_user: sa_113795771384390393994
    ansible_ssh_private_key_file: ~/ssh-key-cluster-sa
    dump_debug_collection_type: all
    dump_debug_local_dest: "/tmp/debug_dump_test"
    dump_debug_tag: "pgbackrest_bucket_issue"
  roles:
    - role: google.alloydbomni_orchestrator.dump_debug

```

`dump_debug_collection_type`

ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml \-e `dump_debug_collection_type=journal`

tar \-tvf /tmp/debug\_dump\_test/alloydbomni\_cluster\_pgbackrest\_bucket\_issue.tar.gz

tar \-xOzf /tmp/debug\_dump\_test/alloydbomni\_cluster\_pgbackrest\_bucket\_issue.tar.gz alloydbomni/satyasais-bk-ts3-db1/alloydbomni\_node\_manager/journalctl.out

Sample command: tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_20260227[.tar.gz](http://.tar.gz)

tar \-xOzf /home/satyasais\_google\_com/debug\_dump\_test/alloydbomni\_cluster\_03032606.tar.gz alloydbomni/satyasais-debug-sh-db1/alloydbomni18/systemd-status.out

**Journals for config:**

**Commands for journal ctl logs:**

Inside control node:  
Creating an /tmp/dump\_debug.yaml:  
Inside control node:

```sh
---
- name: AlloydbOmni Debug Dump Collection
  hosts: all
  become: true
  gather_facts: true
  vars:
    ansible_user: sa_102844249206914475928
    ansible_ssh_private_key_file: ~/ssh-key-cluster-sa
    dump_debug_collection_type: all
    dump_debug_local_dest: "/tmp/debug_dump_test"
    dump_debug_tag: "manuel_restore_issue"
  roles:
    - role: google.alloydbomni_orchestrator.dump_debug

```

`dump_debug_collection_type`

ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml \-e `dump_debug_tag=monitor_issue`

`gsutil -m cp *.tar.gz gs://nova_debug_dumps/backup_issue/`  
ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml \-e `dump_debug_collection_type=config`

ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml

Via [alloydb-ansible.sh](http://alloydb-ansible.sh) file:

/orchestrator/alloydb-ansible.sh debug\_dump \-c /cluster.conf

tar \-tvf /tmp/debug\_dump\_test/alloydbomni\_cluster\_pgbackrest\_bucket\_issue\_1.tar.gz

tar \-tvf /tmp/debug\_dump\_test/alloydbomni\_cluster\_monitor\_issue.tar.gz

tar \-xOzf /tmp/debug\_dump\_test/alloydbomni\_cluster\_monitor\_issue.tar.gz alloydbomni/satyasais-bk-sl10-db2/alloydbomni\_monitor/systemd-status.out

tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_after\_delete\_backup.tar.gz alloydbomni/satyasais-bk-sl6-db1/alloydbomni\_cluster\_manager/journalctl.out

/tmp/debug\_dump/alloydbomni\_cluster\_after\_delete\_backup\_2.tar.gz

Sample command: tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_20260227[.tar.gz](http://.tar.gz)

tar \-xOzf /home/satyasais\_google\_com/debug\_dump\_test/alloydbomni\_cluster\_03032606.tar.gz alloydbomni/satyasais-debug-sh-db1/alloydbomni18/systemd-status.out  
To gemini:

```
The above parts are working fine, Now new comments on next file
HE asked me to modify few, I'll give you total contents of the file and comment, then give me the complete contents by adding the requested feature, without omitting any of the existed code.
Would you like me to now apply these same "passive parameter" logic changes to the Ansible tasks/main.yml to ensure it also avoids overriding worker script defaults?
yes, if it is required as per the below mentioned comments are those.
Contents inside tasks/main.yml:
---
- name: Determine Node Groups and Calculate Dynamic Tag
block:
- name: Set initial timestamp tag if not provided
ansible.builtin.set_fact:
base_tag: "{{ dump_debug_tag if dump_debug_tag | length > 0 else now(fmt='%d%m%y%H') }}"
- name: Check for existing final tarballs to determine suffix
delegate_to: localhost
become: false
run_once: true
block:
- name: Find existing cluster tarballs
ansible.builtin.find:
paths: "{{ dump_debug_local_dest }}"
patterns: "alloydbomni_cluster_{{ base_tag }}*.tar.gz"
register: existing_tars
- name: Calculate final tag with suffix
ansible.builtin.set_fact:
final_tag: >-
{% if existing_tars.matched == 0 %}{{ base_tag }}{% else %}{{ base_tag }}_{{ existing_tars.matched }}{% endif %}
- name: Set facts for node groups
ansible.builtin.set_fact:
primary_instance_nodes: "{{ groups['primary_instance_nodes'] | default([]) }}"
readpool_instance_nodes: "{{ groups['readpool_instance_nodes'] | default([]) }}"
load_balancer_nodes: "{{ groups['load_balancer_nodes'] | default([]) }}"
all_nodes: "{{ (groups['primary_instance_nodes'] | default([])) + (groups['readpool_instance_nodes'] | default([])) + (groups['load_balancer_nodes'] | default([])) | unique }}"
cluster_manager_nodes: >-
{{ groups['cluster_manager_nodes'] if 'cluster_manager_nodes' in groups and (groups['cluster_manager_nodes'] | length > 0)
else groups['primary_instance_nodes'] if 'primary_instance_nodes' in groups else [] }}
run_once: true
- name: Ensure local output directory exists
ansible.builtin.file:
path: "{{ dump_debug_local_dest }}"
state: directory
mode: '0755'
become: false
delegate_to: localhost
run_once: true
- name: Execute Node and Cluster Collection Binaries
block:
# --- CM DUMP SECTION ---
- name: Check for CM dump binary
ansible.builtin.stat:
path: "{{ dump_debug_binary_path }}/alloydbomni_cm_dump"
register: cm_bin_stat
when: inventory_hostname in cluster_manager_nodes
- name: Warn if CM binary is missing
ansible.builtin.debug:
msg: "warning: alloydbomni_cm_dump not found on {{ inventory_hostname }}. Skipping..."
when:
- inventory_hostname in cluster_manager_nodes
- not cm_bin_stat.stat.exists
- name: Run CM dump binary
ansible.builtin.command:
cmd: "{{ dump_debug_binary_path }}/alloydbomni_cm_dump -c {{ dump_debug_collection_type }} -o {{ dump_debug_remote_staging }} -t {{ final_tag }} -s {{ dump_debug_journal_spec }}"
become: true
when:
- inventory_hostname in cluster_manager_nodes
- cm_bin_stat.stat.exists
register: cm_result
failed_when: false
# --- NM DUMP SECTION ---
- name: Check for NM dump binary
ansible.builtin.stat:
path: "{{ dump_debug_binary_path }}/alloydbomni_nm_dump"
register: nm_bin_stat
when: inventory_hostname in all_nodes
- name: Warn if NM binary is missing
ansible.builtin.debug:
msg: "warning: alloydbomni_nm_dump not found on {{ inventory_hostname }}. Skipping..."
when:
- inventory_hostname in all_nodes
- not nm_bin_stat.stat.exists
- name: Run NM dump binary
ansible.builtin.command:
cmd: "{{ dump_debug_binary_path }}/alloydbomni_nm_dump -c {{ dump_debug_collection_type }} -o {{ dump_debug_remote_staging }} -t {{ final_tag }} -s {{ dump_debug_journal_spec }} -p {{ dump_debug_pg_data_dir }}"
become: true
when:
- inventory_hostname in all_nodes
- nm_bin_stat.stat.exists
register: nm_result
failed_when: false
- name: Transfer and Consolidate Dumps
block:
- name: Fetch CM and NM tarballs to controller
ansible.builtin.fetch:
src: "{{ dump_debug_remote_staging }}/alloydbomni_{{ item }}_{{ final_tag }}.tgz"
dest: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/"
flat: no
loop: ["cm", "nm"]
ignore_errors: true
- name: Finalize Consolidated Debug Dump Archive
delegate_to: localhost
run_once: true
become: false
block:
- name: Create staging directory for extraction
ansible.builtin.file:
path: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/extracted"
state: directory
mode: '0755'
- name: Find all fetched node tarballs recursively
ansible.builtin.find:
paths: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/"
recurse: yes
patterns: "*.tgz"
register: fetched_tars
- name: Extract node tarballs into shared directory
ansible.builtin.unarchive:
src: "{{ item.path }}"
dest: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/extracted"
remote_src: no
loop: "{{ fetched_tars.files }}"
when: fetched_tars.matched > 0
- name: Create final cluster-level tarball
ansible.builtin.shell:
cmd: "tar -czf {{ dump_debug_local_dest }}/alloydbomni_cluster_{{ final_tag }}.tar.gz alloydbomni"
chdir: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/extracted"
when: fetched_tars.matched > 0
- name: Final Success Output
ansible.builtin.debug:
msg:
- "Debug dump is stored at location: {{ dump_debug_local_dest }}/alloydbomni_cluster_{{ final_tag }}.tar.gz"
when: fetched_tars.matched > 0
- name: Clean up local staging
ansible.builtin.file:
path: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}"
state: absent
- name: Cleanup Remote Artifacts
ansible.builtin.file:
path: "{{ dump_debug_remote_staging }}/alloydbomni_{{ item }}_{{ final_tag }}.tgz"
state: absent
loop: ["cm", "nm"]
become: true
ignore_errors: true

```

Would you like me to update the Ansible role's `defaults/main.yml` and `tasks/main.yml` to also remove the default path and support this dynamic detection?

**Verify.yml**

```
- name: AlloydbOmni Debug Dump Collection
  hosts: all
  become: true
  gather_facts: true
  vars:
    dump_debug_local_dest: "/tmp/debug_dump"
    dump_debug_tag: "automation_test"
    dump_debug_journal_spec: ""

  roles:
    - role: google.alloydbomni_orchestrator.dump_debug

  tasks:
    - name: Verify and List Generated Tarball
      delegate_to: localhost
      run_once: true
      become: true
      block:
        - name: Ensure tarball exists
          ansible.builtin.stat:
            path: "{{ dump_debug_local_dest }}/alloydbomni_cluster_{{ dump_debug_tag }}.tar.gz"
          register: tarball_check

        - name: Assert Tarball exists
          ansible.builtin.assert:
            that: tarball_check.stat.exists
            fail_msg: "FAILED: Debug dump tarball not found at {{ dump_debug_local_dest }}"

        - name: List Tarball contents for logs
          ansible.builtin.command:
            cmd: "tar -tvf {{ dump_debug_local_dest }}/alloydbomni_cluster_{{ dump_debug_tag }}.tar.gz"
          register: tar_list
          changed_when: false

        - name: Display Tarball Structure
          ansible.builtin.debug:
            var: tar_list.stdout_lines

# Play 2: Extraction and Deep Content Verification
- name: Deep Content Verification
  hosts: localhost
  become: true
  vars:
    dump_output_dir: "/tmp/debug_dump"
    dump_tag: "automation_test"
    extraction_path: "/tmp/verify_extraction"
    tarball_path: "{{ dump_output_dir }}/alloydbomni_cluster_{{ dump_tag }}.tar.gz"

  tasks:
    - name: Create extraction directory
      ansible.builtin.file:
        path: "{{ extraction_path }}"
        state: directory
        mode: '0755'

    - name: Extract master tarball
      ansible.builtin.unarchive:
        src: "{{ tarball_path }}"
        dest: "{{ extraction_path }}"
        remote_src: yes

    - name: Find collected host directories
      ansible.builtin.find:
        paths: "{{ extraction_path }}/alloydbomni"
        file_type: directory
      register: host_dirs

    - name: Validate Internal File Integrity
      vars:
        sample_host: "{{ host_dirs.files[0].path }}"
      block:
        - name: Check for Postgres Config (Dynamic Path Detection Test)
          ansible.builtin.stat:
            path: "{{ sample_host }}/alloydbomni18/postgresql.conf"
          register: pg_stat

        - name: Assert Postgres config is not empty
          ansible.builtin.assert:
            that: 
              - pg_stat.stat.exists
              - pg_stat.stat.size > 0
            fail_msg: "FAILED: postgresql.conf is missing or empty."

    - name: Success Cleanup
      ansible.builtin.file:
        path: "{{ item }}"
        state: absent
      loop:
        - "{{ extraction_path }}"
        - "{{ dump_output_dir }}"
      when: success_cleanup | default(true)
```

**Output:** [https\://fusion2.corp.google.com/ci/guitar/projects/nova-orchestrator-integrat...](https://fusion2.corp.google.com/ci/guitar/projects/nova-orchestrator-integration-tests/workflows/%2F%2Fstorage%2Falloydb%2Fadmin%2Ftesting%2Fguitar%2Fnova%2Fintegration_tests:nova_orchestrator_integration_tests/activity/65418114-8108-38c5-9252-a596a3632078:0/invocations/393d75dc-eac7-4196-9dd4-55476f02dc32/targets/%2F%2Fstorage%2Ftesting%2Flusti%2Ftests%2Fnova%2Fintegration_tests%2Forchestrator:nova_scalable_ha_ansible_debug_dump_test/log)

Via [alloydb-ansible.sh](http://alloydb-ansible.sh) file:

/orchestrator/alloydb-ansible.sh debug\_dump \-c /cluster.conf

   97  rm alloydbomni\_dump.sh   
   98  vi alloydbomni\_dump.sh  
   99  chmod \+x alloydbomni\_dump.sh   
  100  ./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-out=/tmp/debug\_dump \-tag=20250114

\$ g4d testing\_alloydbomni\_dump

\$ PROJECT=alloydb-nova-tvc-sandbox  
CLUSTER=satyasais-debug-cmcli  
NODES="db:3,haproxy:2,control"

\$ cd storage/alloydb/nova/test/

\$ ./gce/cluster-setup.sh create cluster=\$CLUSTER nodes="\$NODES" project="\$PROJECT"

```
control_zone = "us-central1-c"
environment_name = "satyasais-debug-cmcli"
project_id = "alloydb-nova-tvc-sandbox"


$ popd
/google/src/cloud/satyasais/testing_alloydbomni_dump/google3/storage/alloydb/nova/test


$ gcloud compute instances list --project=alloydb-nova-tvc-sandbox --filter="name~satyasais-debug-cmcli-"
NAME                            ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
satyasais-debug-cmcli-db3       us-central1-a  n2-highmem-16               10.1.0.2                  RUNNING
satyasais-debug-cmcli-db2       us-central1-b  n2-highmem-16               10.1.0.3                  RUNNING
satyasais-debug-cmcli-haproxy2  us-central1-b  n2-standard-2               10.1.0.5                  RUNNING
satyasais-debug-cmcli-control   us-central1-c  n2-standard-2               10.1.0.7                  RUNNING
satyasais-debug-cmcli-db1       us-central1-c  n2-highmem-16               10.1.0.4                  RUNNING
satyasais-debug-cmcli-haproxy1  us-central1-c  n2-standard-2               10.1.0.6                  RUNNING

Run the following command to log into the control VM
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-cmcli-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

```

\$ scp \-i \~/.ssh/google\_compute\_engine  \-r ./orchestrator/ \${USER}\_google\_com@nic0.\${CLUSTER}[\-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](http://failover-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

```
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-cmctl-1-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-cmctl-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-cmctl-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-cmctl-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-cmctl-haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com
```

Inside Control node:  
\$ /tmp/setup-ssh-for-cluster.sh

/tmp/orchestrator/alloydb-setup.sh prepare \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-setup.sh create \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-setup.sh status \-c /tmp/cluster.conf

Copying bash scripts on all nodes from workspace::

On db1:

scp /google/src/cloud/satyasais/testing\_alloydbomni\_dump/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_{cm,nm}\_dump.sh \\  
[satyasais\_google\_com@nic0.satyasais-debug-cmcli-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.`satyasais-debug-cmcli-db1`.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

On db2:

scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_{cm,nm}\_dump.sh \\  
[satyasais\_google\_com@nic0.satyasais-debug-cmcli-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-cmcli-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

On db3:

scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_{cm,nm}\_dump.sh \\  
[satyasais\_google\_com@nic0.satyasais-debug-cmcli-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-cmcli-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

On haproxy1:

scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_{cm,nm}\_dump.sh \\  
[satyasais\_google\_com@nic0.satyasais-debug-cmcli-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-cmcli-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

On haproxy2:

scp /google/src/cloud/satyasais/testing\_debug\_dump\_role/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_{cm,nm}\_dump.sh \\  
[satyasais\_google\_com@nic0.satyasais-debug-cmcli-haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.satyasais-debug-cmcli-haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com) "chmod \+x /tmp/alloydbomni\_\*.sh"

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-out=/tmp/debug\_dump \-tag=20250114

tar \-tvf /tmp/debug\_dump/alloydbomni\_20250114.tgz

**Trail 2:**

   97  rm alloydbomni\_dump.sh   
   98  vi alloydbomni\_dump.sh  
   99  chmod \+x alloydbomni\_dump.sh   
  100  ./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-out=/tmp/debug\_dump \-tag=20250114

vi /tmp/alloydbomni\_nm\_dump.sh  
vi /tmp/alloydbomni\_cm\_dump.sh  
chmod \+x /tmp/alloydbomni\_nm\_dump.sh  
chmod \+x /tmp/alloydbomni\_cm\_dump.sh  
ls \-ltr /tmp/

sudo mv /tmp/alloydbomni\_cm\_dump.sh /usr/local/bin/   
sudo mv /tmp/alloydbomni\_nm\_dump.sh /usr/local/bin/   
sudo chmod \+x /usr/local/bin/alloydbomni\_\*\_dump.sh  
ls \-ltr /usr/local/bin/

sudo vi /usr/local/bin/alloydbomni\_cm\_dump  
sudo vi /usr/local/bin/alloydbomni\_nm\_dump

sudo chmod \+x /usr/local/bin/alloydbomni\_cm\_dump   
sudo chmod \+x /usr/local/bin/alloydbomni\_nm\_dump

sudo rm /usr/local/bin/alloydbomni\_cm\_dump  
sudo rm /usr/local/bin/alloydbomni\_nm\_dump

\$ g4d \-f testing\_alloydbomni\_dump\_2

\$ PROJECT=alloydb-nova-tvc-sandbox  
CLUSTER=satyasais-debug-sh  
NODES="db:3,haproxy:2,control"

\$ cd storage/alloydb/nova/test/

\$ ./gce/cluster-setup.sh create cluster=\$CLUSTER nodes="\$NODES" project="\$PROJECT"

```
control_zone = "us-central1-c"
environment_name = "satyasais-debug-sh"
project_id = "alloydb-nova-tvc-sandbox"


$ popd
/google/src/cloud/satyasais/testing_alloydbomni_dump_2/google3/storage/alloydb/nova/test


$ gcloud compute instances list --project=alloydb-nova-tvc-sandbox --filter="name~satyasais-debug-sh-"
NAME                         ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
satyasais-debug-sh-db3       us-central1-a  n2-highmem-16               10.1.0.5                  RUNNING
satyasais-debug-sh-db2       us-central1-b  n2-highmem-16               10.1.0.3                  RUNNING
satyasais-debug-sh-haproxy2  us-central1-b  n2-standard-2               10.1.0.2                  RUNNING
satyasais-debug-sh-control   us-central1-c  n2-standard-2               10.1.0.7                  RUNNING
satyasais-debug-sh-db1       us-central1-c  n2-highmem-16               10.1.0.4                  RUNNING
satyasais-debug-sh-haproxy1  us-central1-c  n2-standard-2               10.1.0.6                  RUNNING

Run the following command to log into the control VM
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-sh-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

```

\$ scp \-i \~/.ssh/google\_compute\_engine  \-r ./orchestrator/ \${USER}\_google\_com@nic0.\${CLUSTER}[\-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](http://failover-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

```
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-sh-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-sh-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-sh-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-sh-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-debug-sh-haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com
```

Inside Control node:  
\$ /tmp/setup-ssh-for-cluster.sh

/tmp/orchestrator/alloydb-ansible.sh prepare \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-ansible.sh create \-c /tmp/cluster.conf  
/tmp/orchestrator/alloydb-ansible.sh status \-c /tmp/cluster.conf

Copying bash scripts on all nodes from workspace::

On db1:

scp /google/src/cloud/satyasais/testing\_alloydbomni\_dump\_2/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_{cm,nm}\_dump.sh \\  
[satyasais\_google\_com@nic0.](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-db1`[.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com` "chmod \+x /tmp/alloydbomni\_\*.sh"

On db2:

scp /google/src/cloud/satyasais/testing\_alloydbomni\_dump\_2/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_{cm,nm}\_dump.sh \\  
[satyasais\_google\_com@nic0.](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-db2.`[us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com` "chmod \+x /tmp/alloydbomni\_\*.sh"

On db3:

scp /google/src/cloud/satyasais/testing\_alloydbomni\_dump\_2/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_{cm,nm}\_dump.sh \\  
[satyasais\_google\_com@nic0.](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-db3`[.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com` "chmod \+x /tmp/alloydbomni\_\*.sh"

On haproxy1:

scp /google/src/cloud/satyasais/testing\_alloydbomni\_dump\_2/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_nm\_dump.sh \\  
[satyasais\_google\_com@nic0.](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-`[haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com` "chmod \+x /tmp/alloydbomni\_\*.sh"

On haproxy2:

scp /google/src/cloud/satyasais/testing\_alloydbomni\_dump\_2/google3/experimental/users/satyasais/cm\_nm\_dump/alloydbomni\_nm\_dump.sh \\  
[satyasais\_google\_com@nic0.](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-`[haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com):/tmp/

ssh [`satyasais_google_com@nic0.`](mailto:satyasais_google_com@nic0.satyasais-debug-role-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com)`satyasais-debug-sh-haproxy2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com` "chmod \+x /tmp/alloydbomni\_\*.sh"

./alloydbomni\_dump.sh \\  
  \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \\  
  \-u sa\_114827027510688913411 \\  
  \-spec /tmp/deployment\_spec.yaml \\  
  \-out=/tmp/debug\_dump \\  
  \-tag=20260227 \\  
  \-c=status

```sh
./alloydbomni_dump.sh -k /home/satyasais_google_com/ssh-key-cluster-sa -u sa_114827027510688913411 -spec /tmp/deployment_spec.yaml -out=/tmp/debug_dump -tag=20260227 -c=status
```

New command: 

```sh
./alloydbomni_dump.sh -k /home/satyasais_google_com/ssh-key-cluster-sa -u sa_114827027510688913411 -d /tmp/deployment_spec.yaml -o=/tmp/debug_dump -t 20260227 -c status
```

NEW now:

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_114827027510688913411 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump \-t 20260227 \-c status

New now:

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_114827027510688913411 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump \-t 20260227 \-c status

Command for no tag:

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_114827027510688913411 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump

tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_02032606[.tar.gz](http://.tar.gz)

tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_02032606\_1.tar.gz

tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_02032606\_1.tar.gz alloydbomni/satyasais-debug-sh-db1/alloydbomni18/systemd-status.out  
tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_02032606\_1.tar.gz alloydbomni/satyasais-debug-sh-db3/alloydbomni\_cluster\_manager/systemd-status.out

Command for journal

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_114827027510688913411 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump \-c journal

/tmp/debug\_dump/alloydbomni\_cluster\_03032606[.tar.gz](http://.tar.gz)

tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_03032606[.tar.gz](http://.tar.gz)

tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_03032606[.tar.gz](http://.tar.gz) alloydbomni/satyasais-debug-sh-db1/alloydbomni18/journalctl.out

tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_03032606[.tar.gz](http://.tar.gz) alloydbomni/satyasais-debug-sh-haproxy2/keepalived/journalctl.out

All:

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_114827027510688913411 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump \-c all

tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_03032606\_1[.tar.gz](http://.tar.gz)

tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_03032606\_1.tar.gz alloydbomni/satyasais-debug-sh-db3/alloydbomni\_cluster\_manager/journalctl.out

**Command for config files:**

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_114827027510688913411 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_114827027510688913411 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump \-s "--since 2026-03-05"

./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_111363304028171725403 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump \-s "--since \\"2 hours ago\\""

/tmp/debug\_dump/alloydbomni\_cluster\_06032604.tar.gz

tar \-tvf /tmp/debug\_dump/alloydbomni\_cluster\_06032611.tar.gz

tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_05032613.tar.gz alloydbomni/satyasais-debug-sh-db1/alloydbomni\_cluster\_manager/cluster\_manager.toml

tar \-xOzf /tmp/debug\_dump/alloydbomni\_cluster\_05032609.tar.gz alloydbomni/satyasais-debug-sh-haproxy2/keepalived/journalctl.out

alloydbomni\_dump \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-u sa\_111363304028171725403 \-d /tmp/deployment\_spec.yaml \-o /tmp/debug\_dump

tar \-tvf /tmp/debug\_dump\_logs/alloydbomni\_cluster\_scalable\_ha\_ansible[.tar.gz](http://.tar.gz) 

gsutil \-m cp gs://nova\_debug\_dumps/nova\_issue/alloydbomni\_cluster\_scalable\_ha\_ansible[.tar.gz](http://.tar.gz) /tmp/debug\_dump\_logs/  
We have 2 architectures now currently I was working for a non Ansible environment.  
Four artifacts:  
1\. google.alloydbomni\_orchestrator.dump\_debug \- Ansible role \- Should be packed inside Ansible collection tarball  
2\. alloydbomni\_dump \- Executable \- Should be packed inside Orchestrator RPM (CMCLI)  
3\. alloydbomni\_nm\_dump \- Executable \- Should be packed inside Node manager RPM  
4\. alloydbomni\_cm\_dump \- Executable \- Should be packed inside Cluster manager RPM  
CLI  
\[sa\_user\]\$ alloydbomni\_nm\_dump \[-c \<collection-type\>,..\] –out=\<dir\> –tag=\<tag\>  
\[sa\_user\]\$ alloydbomni\_cm\_dump \[-c \<collection-type\>,..\] –out=\<dir\> –tag=\<tag\>  
\[sa\_user\]\$ alloydbomni\_dump \[-k \<ssh key\>\] \[-c \<collection-type\>,...\] –out=\<dir\> –tag=\<tag\>  
\<collection-types\>: If no values given then use status  
status : collect systemd status (DEFAULT)  
journal: collect journalctl logs  
logs   : copy the log files  
config : collect the configuration files  
all    : collect all the supported collection types  
Examples  
\# Create /tmp/debug\_dump/alloydbomni\_20250114.tgz with only status  
\$ sudo alloydbomni\_nm\_dump \-out=/tmp/debug\_dump \-tag=20250114   
\# Collect only logs  
\$ sudo alloydbomni\_nm\_dump \-out=/tmp/debug\_dump \-tag=20250114 \-c logs  
\# Collect logs and configuration files  
\$ sudo alloydbomni\_nm\_dump \-out=/tmp/debug\_dump \-tag=20250114 \-c logs,config  
\# Full  
\$ sudo alloydbomni\_nm\_dump \-out=/tmp/debug\_dump \-tag=20250114 \-c all  
Now I prepared alloydbomni\_nm\_dump.sh and alloydbomni\_cm\_dump.sh file and I given to people who created cm and NM RPMs, to package inside them.  
Content inside alloydbomni\_nm\_dump.sh:  
\#\!/bin/bash  
set \-euo pipefail  
SERVICES=(  
"alloydbomni\_node\_manager"  
"alloydbomni\_monitor"  
"etcd"  
"haproxy"  
"keepalived"  
"pgbouncer"  
"pgbackrest"  
)  
HOSTNAME=\$(hostname)  
TYPES="status"  
OUT\_DIR=""  
TAG=""  
usage() {  
echo "Usage: \$0 \[-c \<collection-type\>,..\] \-out=\<dir\> \-tag=\<tag\>"  
exit 1  
}  
for arg in "\$@"; do  
case \$arg in  
\-c=\*) TYPES="\${arg\#\*=}" ;;  
\-c) shift; TYPES="\$1" ;;  
\-out=\*) OUT\_DIR="\${arg\#\*=}" ;;  
\-tag=\*) TAG="\${arg\#\*=}" ;;  
\*) echo "Unknown option: \$arg"; usage ;;  
esac  
done  
if \[\[ \-z "\$OUT\_DIR" || \-z "\$TAG" \]\]; then usage; fi  
TMP\_DIR=\$(mktemp \-d)  
trap 'rm \-rf "\$TMP\_DIR"' EXIT  
mapfile \-t DB\_SERVICES \< \<(systemctl list-unit-files \--type=service \--all | grep \-oE '^alloydbomni\[0-9\]+' || true)  
ALL\_COMPONENTS=(\$(printf "%s\\n" "\${SERVICES\[@\]}" "\${DB\_SERVICES\[@\]}" | sort \-u))  
for svc in "\${ALL\_COMPONENTS\[@\]}"; do  
COLLECT\_ROOT="\${TMP\_DIR}/alloydbomni/\${HOSTNAME}/\${svc}"  
\# Check if the service unit file actually exists on the system  
if systemctl list-unit-files "\$svc.service" \>/dev/null 2\>&1; then  
mkdir \-p "\$COLLECT\_ROOT"  
if \[\[ "\$TYPES" \== \*"status"\* || "\$TYPES" \== \*"all"\* || \-n "\$TYPES" \]\]; then  
echo "\[INFO\] Collecting systemd status for \$svc..."  
systemctl status "\$svc" \-n 0 \--no-pager \> "\$COLLECT\_ROOT/systemd-status.out" 2\>&1 || true  
fi  
else  
echo "\[DEBUG\] Service \$svc not found on this system. Skipping..."  
fi  
done  
mkdir \-p "\$OUT\_DIR"  
TARBALL="\${OUT\_DIR}/alloydbomni\_\${TAG}.tgz"  
if \[\[ \-f "\$TARBALL" \]\]; then  
echo "\[INFO\] Appending to existing tarball: \$TARBALL"  
tar \-xzf "\$TARBALL" \-C "\$TMP\_DIR"  
fi  
tar \-czf "\$TARBALL" \-C "\$TMP\_DIR" alloydbomni  
echo "\[SUCCESS\] Final Tarball location: \$TARBALL"  
content inside alloydbomni\_cm\_dump.sh:  
\#\!/bin/bash  
set \-euo pipefail  
SERVICE\_NAME="alloydbomni\_cluster\_manager"  
HOSTNAME=\$(hostname)  
TYPES="status"  
OUT\_DIR=""  
TAG=""  
usage() {  
echo "Usage: \$0 \[-c \<collection-type\>,..\] \-out=\<dir\> \-tag=\<tag\>"  
exit 1  
}  
for arg in "\$@"; do  
case \$arg in  
\-c=\*) TYPES="\${arg\#\*=}" ;;  
\-c) shift; TYPES="\$1" ;;  
\-out=\*) OUT\_DIR="\${arg\#\*=}" ;;  
\-tag=\*) TAG="\${arg\#\*=}" ;;  
\*) echo "Unknown option: \$arg"; usage ;;  
esac  
done  
if \[\[ \-z "\$OUT\_DIR" || \-z "\$TAG" \]\]; then usage; fi  
TMP\_DIR=\$(mktemp \-d)  
COLLECT\_ROOT="\${TMP\_DIR}/alloydbomni/\${HOSTNAME}/\${SERVICE\_NAME}"  
mkdir \-p "\$COLLECT\_ROOT"  
trap 'rm \-rf "\$TMP\_DIR"' EXIT  
if \[\[ "\$TYPES" \== \*"status"\* || "\$TYPES" \== \*"all"\* || \-n "\$TYPES" \]\]; then  
echo "\[INFO\] Collecting systemd status for \$SERVICE\_NAME..."  
systemctl status "\$SERVICE\_NAME" \-n 0 \--no-pager \> "\$COLLECT\_ROOT/systemd-status.out" 2\>&1 || true  
fi  
mkdir \-p "\$OUT\_DIR"  
TARBALL="\${OUT\_DIR}/alloydbomni\_\${TAG}.tgz"  
if \[\[ \-f "\$TARBALL" \]\]; then  
echo "\[INFO\] Appending to existing tarball: \$TARBALL"  
tar \-xzf "\$TARBALL" \-C "\$TMP\_DIR"  
fi  
tar \-czf "\$TARBALL" \-C "\$TMP\_DIR" alloydbomni  
echo "\[SUCCESS\] Final Tarball location: \$TARBALL"  
Output after exeucting those 2 scripts in my local machine  
\[satyasais\_google\_com@satyasais-debug-dump-db1 \~\]\$ sudo sh alloydbomni\_cm\_dump.sh \-out=/tmp/debug\_dump \-tag=20250114  
\[INFO\] Collecting systemd status for alloydbomni\_cluster\_manager...  
\[SUCCESS\] Final Tarball location: /tmp/debug\_dump/alloydbomni\_20250114.tgz  
\[satyasais\_google\_com@satyasais-debug-dump-db1 \~\]\$ sudo sh alloydbomni\_nm\_dump.sh \-out=/tmp/debug\_dump \-tag=20250114  
\[INFO\] Collecting systemd status for alloydbomni17...  
\[INFO\] Collecting systemd status for alloydbomni\_monitor...  
\[INFO\] Collecting systemd status for alloydbomni\_node\_manager...  
\[INFO\] Collecting systemd status for etcd...  
\[DEBUG\] Service haproxy not found on this system. Skipping...  
\[DEBUG\] Service keepalived not found on this system. Skipping...  
\[INFO\] Collecting systemd status for pgbackrest...  
\[DEBUG\] Service pgbouncer not found on this system. Skipping...  
\[INFO\] Appending to existing tarball: /tmp/debug\_dump/alloydbomni\_20250114.tgz  
\[SUCCESS\] Final Tarball location: /tmp/debug\_dump/alloydbomni\_20250114.tgz  
\[satyasais\_google\_com@satyasais-debug-dump-db1 \~\]\$ tar \-tvf /tmp/debug\_dump/alloydbomni\_20250114.tgz  
drwxr-xr-x root/root         0 2026-01-19 15:54 alloydbomni/  
drwxr-xr-x root/root         0 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/  
drwxr-xr-x root/root         0 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/alloydbomni17/  
\-rw-r--r-- root/root       272 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/alloydbomni17/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/alloydbomni\_monitor/  
\-rw-r--r-- root/root       195 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/alloydbomni\_monitor/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       541 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/etcd/  
\-rw-r--r-- root/root       385 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/etcd/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/pgbackrest/  
\-rw-r--r-- root/root       220 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/pgbackrest/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/alloydbomni\_cluster\_manager/  
\-rw-r--r-- root/root       560 2026-01-19 15:54 alloydbomni/satyasais-debug-dump-db1/alloydbomni\_cluster\_manager/systemd-status.out  
Now I need to develop the alloydbomni\_dump.sh script. Now in the same stream I was asked to write alloydbomni\_dump.sh, As you can see the architectural diagram, we will be having three nodes  
every node will be having 1 tar each and those tars will be again we will do a single tar, here alloydbomni\_dump script calls alloydbomni\_cm\_dump and alloydbomni\_nm\_dump scripts. then it will execute the scripts then it will fetch the generated tarballs to the control node and then tar them again to get the below output. This final script alloydbomni\_dump.sh will be packed inside orchestrator RPM (cmctl). Hence please give the alloydbomni\_dump.sh script  
the final out should be look like below  
alloydbomni/  
├── xx.xx.xx.xx  
│   ├── alloydbomni17  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_cluster\_manager  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_monitor  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_node\_manager  
│   │   └── systemd-status.out  
│   └── etcd  
│       └── systemd-status.out  
│   └── pgbackrest  
│       └── systemd-status.out  
│   └── haproxy  
│       └── systemd-status.out  
├── yy.yy.yy.yy  
│   ├── alloydbomni17  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_cluster\_manager  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_monitor  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_node\_manager  
│   │   └── systemd-status.out  
│   └── etcd  
│       └── systemd-status.out  
│   └── pgbackrest  
│       └── systemd-status.out  
│   └── haproxy  
│       └── systemd-status.out  
└── zz.zz.zz.zz  
│   ├── alloydbomni17  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_cluster\_manager  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_monitor  
│   │   └── systemd-status.out  
│   ├── alloydbomni\_node\_manager  
│   │   └── systemd-status.out  
│   └── etcd  
│       └── systemd-status.out  
│   └── pgbackrest  
│       └── systemd-status.out  
│   └── haproxy  
│       └── systemd-status.out  
For my Ansible environment, I written the role and it is working fine, since the alloydbomni\_nm\_dump.sh and alloydbomni\_cm\_dump.sh are not packaged inside cm and NM rpm's respectively hence, we have copied those scripts into all db1, db2, db3, haproxy1, haproxy2 vm's manuelly and given chmod \+x permissions. and it is working fine you can see code and results below  
This is the code:  
\---  
\- name: Initialize Debug Dump Environment  
  block:  
    \- name: Generate timestamp fact  
      ansible.builtin.set\_fact:  
        dump\_time: "{{ now(utc=False, fmt='%Y%m%d\_%H%M%S') }}"  
      run\_once: true  
      delegate\_to: localhost  
    \- name: Ensure remote staging directory exists  
      ansible.builtin.file:  
        path: "{{ dump\_debug\_remote\_staging }}"  
        state: directory  
        mode: '0755'  
        owner: root  
        group: root  
      become: true  
\- name: Execute Node and Cluster Collection Scripts  
  block:  
    \- name: Run alloydbomni\_cm\_dump script  
      ansible.builtin.command:  
        \# For testing: "{{ dump\_debug\_script\_path }}/alloydbomni\_cm\_dump.sh"  
        \# For production: "/usr/bin/alloydbomni\_cm\_dump"  
        \# TODO(satyasais): Change script path to /usr/bin/alloydbomni\_cm\_dump once the RPM is ready  
        cmd: "{{ dump\_debug\_script\_path }}/alloydbomni\_cm\_dump.sh \-c={{ dump\_debug\_collection\_type }} \-out={{ dump\_debug\_remote\_staging }} \-tag={{ dump\_time }}"  
      become: true  
      register: cm\_result  
      changed\_when: "cm\_result.rc \== 0"  
      failed\_when:  
        \- cm\_result.rc \!= 0  
        \- "'No such file' not in cm\_result.msg"  
    \- name: Run alloydbomni\_nm\_dump script  
      ansible.builtin.command:  
        \# For testing: "{{ dump\_debug\_script\_path }}/alloydbomni\_nm\_dump.sh"  
        \# For production: "/usr/bin/alloydbomni\_nm\_dump"  
        \# TODO(satyasais): Change script path to /usr/bin/alloydbomni\_nm\_dump once the RPM is ready  
        cmd: "{{ dump\_debug\_script\_path }}/alloydbomni\_nm\_dump.sh \-c={{ dump\_debug\_collection\_type }} \-out={{ dump\_debug\_remote\_staging }} \-tag={{ dump\_time }}"  
      become: true  
      register: nm\_result  
      changed\_when: "nm\_result.rc \== 0"  
      failed\_when:  
        \- nm\_result.rc \!= 0  
        \- "'No such file' not in nm\_result.msg"  
\- name: Transfer and Consolidate Dumps  
  block:  
    \- name: Define remote tarball path  
      ansible.builtin.set\_fact:  
        remote\_tarball: "{{ dump\_debug\_remote\_staging }}/alloydbomni\_{{ dump\_time }}.tgz"  
    \- name: Fetch node tarballs to controller staging area  
      ansible.builtin.fetch:  
        src: "{{ remote\_tarball }}"  
        dest: "{{ dump\_debug\_local\_dest }}/staging\_{{ dump\_time }}/"  
        flat: no  
\- name: Finalize Master Archive  
  delegate\_to: localhost  
  run\_once: true  
  block:  
    \- name: Find all fetched node tarballs  
      ansible.builtin.find:  
        paths: "{{ dump\_debug\_local\_dest }}/staging\_{{ dump\_time }}"  
        recurse: yes  
        patterns: "\*.tgz"  
      register: fetched\_tars  
    \- name: Extract node tarballs to normalize structure  
      ansible.builtin.unarchive:  
        src: "{{ item.path }}"  
        dest: "{{ dump\_debug\_local\_dest }}/staging\_{{ dump\_time }}"  
        remote\_src: no  
      loop: "{{ fetched\_tars.files }}"  
    \- name: Create cluster-level consolidated tarball  
      ansible.builtin.shell:  
        cmd: "tar \-czf alloydbomni\_cluster\_{{ dump\_time }}.tar.gz alloydbomni"  
        chdir: "{{ dump\_debug\_local\_dest }}/staging\_{{ dump\_time }}"  
      changed\_when: true  
    \- name: Move final tarball to main output directory  
      ansible.builtin.command:  
        cmd: "mv {{ dump\_debug\_local\_dest }}/staging\_{{ dump\_time }}/alloydbomni\_cluster\_{{ dump\_time }}.tar.gz {{ dump\_debug\_local\_dest }}/"  
      changed\_when: true  
    \- name: Print successful bundle location  
      ansible.builtin.debug:  
        msg: "Full cluster debug bundle created: {{ dump\_debug\_local\_dest }}/alloydbomni\_cluster\_{{ dump\_time }}.tar.gz"  
    \- name: Clean up local staging directory  
      ansible.builtin.file:  
        path: "{{ dump\_debug\_local\_dest }}/staging\_{{ dump\_time }}"  
        state: absent  
\- name: Cleanup Remote Artifacts  
  ansible.builtin.file:  
    path: "{{ dump\_debug\_remote\_staging }}"  
    state: absent  
  become: true  
result:  
\# Contents inside hosts.yaml  
\[satyasais\_google\_com@satyasais-debug-role-control tmp\]\$ cat hosts.yaml  
alloydbomni:  
  vars:  
    cluster\_name: "satyasais-debug-role"  
    ansible\_user: sa\_106801092749283222743  
    ansible\_ssh\_private\_key\_file: /home/satyasais\_google\_com/ssh-key-cluster-sa  
  children:  
    primary\_instance\_nodes:  
      hosts:  
        satyasais-debug-role-db1:  
        satyasais-debug-role-db2:  
        satyasais-debug-role-db3:  
    cluster\_manager\_nodes:  
      hosts:  
        satyasais-debug-role-db1:  
        satyasais-debug-role-db2:  
        satyasais-debug-role-db3:  
    load\_balancer\_nodes:  
      hosts:  
        satyasais-debug-role-haproxy1:  
        satyasais-debug-role-haproxy2:  
\# Contents inside dump\_debug.yaml  
\[satyasais\_google\_com@satyasais-debug-role-control tmp\]\$ cat dump\_debug.yaml  
\---  
\- name: Collect Debug Logs and System Status from AlloyDB Omni Cluster  
  hosts: all  
  become: true  
  gather\_facts: true  
  vars:  
    \# You can override defaults here for specific test runs  
    dump\_debug\_collection\_type: "status"  
    dump\_debug\_tag: "test\_run\_{{ ansible\_date\_time.date }}"  
  roles:  
    \- role: google.alloydbomni\_orchestrator.dump\_debug  
  post\_tasks:  
    \- name: Final Confirmation  
      ansible.builtin.debug:  
        msg: "Collection workflow complete. Please check {{ dump\_debug\_local\_dest }} for the final bundle."  
      run\_once: true  
\# Execution of dump\_debug.yaml  
\[satyasais\_google\_com@satyasais-debug-role-control tmp\]\$ ansible-playbook \-i hosts.yaml dump\_debug.yaml  
PLAY \[Collect Debug Logs and System Status from AlloyDB Omni Cluster\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
TASK \[Gathering Facts\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
ok: \[satyasais-debug-role-db1\]  
ok: \[satyasais-debug-role-db2\]  
ok: \[satyasais-debug-role-db3\]  
ok: \[satyasais-debug-role-haproxy1\]  
ok: \[satyasais-debug-role-haproxy2\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Generate timestamp fact\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
ok: \[satyasais-debug-role-db1 \-\> localhost\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Ensure remote staging directory exists\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1\]  
changed: \[satyasais-debug-role-haproxy1\]  
changed: \[satyasais-debug-role-db3\]  
changed: \[satyasais-debug-role-haproxy2\]  
changed: \[satyasais-debug-role-db2\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Run alloydbomni\_cm\_dump script\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1\]  
ok: \[satyasais-debug-role-haproxy1\]  
ok: \[satyasais-debug-role-haproxy2\]  
changed: \[satyasais-debug-role-db3\]  
changed: \[satyasais-debug-role-db2\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Run alloydbomni\_nm\_dump script\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1\]  
changed: \[satyasais-debug-role-haproxy1\]  
changed: \[satyasais-debug-role-db3\]  
changed: \[satyasais-debug-role-haproxy2\]  
changed: \[satyasais-debug-role-db2\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Define remote tarball path\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
ok: \[satyasais-debug-role-db1\]  
ok: \[satyasais-debug-role-db2\]  
ok: \[satyasais-debug-role-db3\]  
ok: \[satyasais-debug-role-haproxy1\]  
ok: \[satyasais-debug-role-haproxy2\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Fetch node tarballs to controller staging area\] \*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1\]  
changed: \[satyasais-debug-role-haproxy1\]  
changed: \[satyasais-debug-role-db3\]  
changed: \[satyasais-debug-role-db2\]  
changed: \[satyasais-debug-role-haproxy2\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Find all fetched node tarballs\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
ok: \[satyasais-debug-role-db1 \-\> localhost\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Extract node tarballs to normalize structure\] \*\*\*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1 \-\> localhost\] \=\> (item={'path': '/tmp/debug\_outputs/staging\_20260120\_142333/satyasais-debug-role-db1/tmp/alloydbomni\_debug\_staging/alloydbomni\_20260120\_142333.tgz', 'mode': '0644', 'isdir': False, 'ischr': False, 'isblk': False, 'isreg': True, 'isfifo': False, 'islnk': False, 'issock': False, 'uid': 343751141, 'gid': 343751141, 'size': 1023, 'inode': 84135758, 'dev': 2050, 'nlink': 1, 'atime': 1768919017.0637188, 'mtime': 1768919017.0637188, 'ctime': 1768919017.0637188, 'gr\_name': 'satyasais\_google\_com', 'pw\_name': 'satyasais\_google\_com', 'wusr': True, 'rusr': True, 'xusr': False, 'wgrp': False, 'rgrp': True, 'xgrp': False, 'woth': False, 'roth': True, 'xoth': False, 'isuid': False, 'isgid': False})  
changed: \[satyasais-debug-role-db1 \-\> localhost\] \=\> (item={'path': '/tmp/debug\_outputs/staging\_20260120\_142333/satyasais-debug-role-haproxy1/tmp/alloydbomni\_debug\_staging/alloydbomni\_20260120\_142333.tgz', 'mode': '0644', 'isdir': False, 'ischr': False, 'isblk': False, 'isreg': True, 'isfifo': False, 'islnk': False, 'issock': False, 'uid': 343751141, 'gid': 343751141, 'size': 761, 'inode': 134492501, 'dev': 2050, 'nlink': 1, 'atime': 1768919017.105719, 'mtime': 1768919017.105719, 'ctime': 1768919017.105719, 'gr\_name': 'satyasais\_google\_com', 'pw\_name': 'satyasais\_google\_com', 'wusr': True, 'rusr': True, 'xusr': False, 'wgrp': False, 'rgrp': True, 'xgrp': False, 'woth': False, 'roth': True, 'xoth': False, 'isuid': False, 'isgid': False})  
changed: \[satyasais-debug-role-db1 \-\> localhost\] \=\> (item={'path': '/tmp/debug\_outputs/staging\_20260120\_142333/satyasais-debug-role-db2/tmp/alloydbomni\_debug\_staging/alloydbomni\_20260120\_142333.tgz', 'mode': '0644', 'isdir': False, 'ischr': False, 'isblk': False, 'isreg': True, 'isfifo': False, 'islnk': False, 'issock': False, 'uid': 343751141, 'gid': 343751141, 'size': 1016, 'inode': 742506, 'dev': 2050, 'nlink': 1, 'atime': 1768919017.1117194, 'mtime': 1768919017.1117194, 'ctime': 1768919017.1117194, 'gr\_name': 'satyasais\_google\_com', 'pw\_name': 'satyasais\_google\_com', 'wusr': True, 'rusr': True, 'xusr': False, 'wgrp': False, 'rgrp': True, 'xgrp': False, 'woth': False, 'roth': True, 'xoth': False, 'isuid': False, 'isgid': False})  
changed: \[satyasais-debug-role-db1 \-\> localhost\] \=\> (item={'path': '/tmp/debug\_outputs/staging\_20260120\_142333/satyasais-debug-role-db3/tmp/alloydbomni\_debug\_staging/alloydbomni\_20260120\_142333.tgz', 'mode': '0644', 'isdir': False, 'ischr': False, 'isblk': False, 'isreg': True, 'isfifo': False, 'islnk': False, 'issock': False, 'uid': 343751141, 'gid': 343751141, 'size': 1021, 'inode': 52246085, 'dev': 2050, 'nlink': 1, 'atime': 1768919017.1157193, 'mtime': 1768919017.1157193, 'ctime': 1768919017.1157193, 'gr\_name': 'satyasais\_google\_com', 'pw\_name': 'satyasais\_google\_com', 'wusr': True, 'rusr': True, 'xusr': False, 'wgrp': False, 'rgrp': True, 'xgrp': False, 'woth': False, 'roth': True, 'xoth': False, 'isuid': False, 'isgid': False})  
changed: \[satyasais-debug-role-db1 \-\> localhost\] \=\> (item={'path': '/tmp/debug\_outputs/staging\_20260120\_142333/satyasais-debug-role-haproxy2/tmp/alloydbomni\_debug\_staging/alloydbomni\_20260120\_142333.tgz', 'mode': '0644', 'isdir': False, 'ischr': False, 'isblk': False, 'isreg': True, 'isfifo': False, 'islnk': False, 'issock': False, 'uid': 343751141, 'gid': 343751141, 'size': 759, 'inode': 100823395, 'dev': 2050, 'nlink': 1, 'atime': 1768919017.1207194, 'mtime': 1768919017.1207194, 'ctime': 1768919017.1207194, 'gr\_name': 'satyasais\_google\_com', 'pw\_name': 'satyasais\_google\_com', 'wusr': True, 'rusr': True, 'xusr': False, 'wgrp': False, 'rgrp': True, 'xgrp': False, 'woth': False, 'roth': True, 'xoth': False, 'isuid': False, 'isgid': False})  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Create cluster-level consolidated tarball\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1 \-\> localhost\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Move final tarball to main output directory\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1 \-\> localhost\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Print successful bundle location\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
ok: \[satyasais-debug-role-db1 \-\> localhost\] \=\> {  
    "msg": "Full cluster debug bundle created: /tmp/debug\_outputs/alloydbomni\_cluster\_20260120\_142333.tar.gz"  
}  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Clean up local staging directory\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1 \-\> localhost\]  
TASK \[google.alloydbomni\_orchestrator.dump\_debug : Cleanup Remote Artifacts\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
changed: \[satyasais-debug-role-db1\]  
changed: \[satyasais-debug-role-db2\]  
changed: \[satyasais-debug-role-haproxy1\]  
changed: \[satyasais-debug-role-haproxy2\]  
changed: \[satyasais-debug-role-db3\]  
TASK \[Final Confirmation\] \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
ok: \[satyasais-debug-role-db1\] \=\> {  
    "msg": "Collection workflow complete. Please check /tmp/debug\_outputs for the final bundle."  
}  
PLAY RECAP \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*  
satyasais-debug-role-db1   : ok=15   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0    
satyasais-debug-role-db2   : ok=7    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0    
satyasais-debug-role-db3   : ok=7    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0    
satyasais-debug-role-haproxy1 : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0    
satyasais-debug-role-haproxy2 : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0    
\# contents inside debug\_outputs:  
\[satyasais\_google\_com@satyasais-debug-role-control tmp\]\$ cd debug\_outputs/  
\[satyasais\_google\_com@satyasais-debug-role-control debug\_outputs\]\$ ls  
alloydbomni\_cluster\_20260120\_135729.tar.gz  alloydbomni\_cluster\_20260120\_142333.tar.gz  
\# untar of latest tar  
\[satyasais\_google\_com@satyasais-debug-role-control debug\_outputs\]\$ tar \-tvf alloydbomni\_cluster\_20260120\_142333.tar.gz  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni17/  
\-rw-r--r-- root/root       272 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni17/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_monitor/  
\-rw-r--r-- root/root       195 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_monitor/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       542 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/etcd/  
\-rw-r--r-- root/root       381 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/etcd/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/pgbackrest/  
\-rw-r--r-- root/root       220 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/pgbackrest/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_cluster\_manager/  
\-rw-r--r-- root/root       561 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_cluster\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       545 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/haproxy/  
\-rw-r--r-- root/root       163 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/haproxy/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/keepalived/  
\-rw-r--r-- root/root       186 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/keepalived/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/pgbouncer/  
\-rw-r--r-- root/root       192 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/pgbouncer/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni17/  
\-rw-r--r-- root/root       272 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni17/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_monitor/  
\-rw-r--r-- root/root       195 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_monitor/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       542 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/etcd/  
\-rw-r--r-- root/root       385 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/etcd/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/pgbackrest/  
\-rw-r--r-- root/root       220 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/pgbackrest/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_cluster\_manager/  
\-rw-r--r-- root/root       561 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_cluster\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni17/  
\-rw-r--r-- root/root       272 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni17/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_monitor/  
\-rw-r--r-- root/root       195 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_monitor/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       542 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/etcd/  
\-rw-r--r-- root/root       385 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/etcd/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/pgbackrest/  
\-rw-r--r-- root/root       220 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/pgbackrest/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_cluster\_manager/  
\-rw-r--r-- root/root       561 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_cluster\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       545 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/haproxy/  
\-rw-r--r-- root/root       163 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/haproxy/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/keepalived/  
\-rw-r--r-- root/root       186 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/keepalived/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/pgbouncer/  
\-rw-r--r-- root/root       192 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/pgbouncer/systemd-status.out  
In the same way i'm expecting an bash script for my non Ansible environment, later my python team will be packing this bash script inside orchestrator cmctl RPM. Hence please alloydbomni\_dump.sh script, with all best practices and command to run it follows all the practices mentioned earlier.  
For your reference, I'll give some details how other bash scripts are being exeucted from my control node to other nodes, these are the references below  
contents inside alloydb-setup.sh  
\#\!/bin/bash  
set \-euo pipefail  
script\_dir=\$(dirname "\$0")  
script\_name=\$(basename "\$0")  
source "\$script\_dir/defaults.conf"  
source "\$script\_dir"/utils.sh  
export etcd\_repo\_url="gs://alloydb-omni-custom-releases/alloydbomni/bundle/0.0.2/rpms"  
export pgdg\_reporpms\_url="https\://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86\_64/pgdg-redhat-repo-latest.noarch.rpm"  
export alloydb\_rpm\_repo\_url="https\://us-central1-yum.pkg.dev/projects/alloydb-omni-sandbox/nova"  
export omni\_svc\_rpm\_repo\_url="https\://us-central1-yum.pkg.dev/projects/alloydb-omni-sandbox/nova"  
export pg\_version=17  
export pg\_data="/data"  
export pg\_user="postgres"  
export pg\_passwd="postgres"  
export pg\_port="5432"  
export pg\_group="postgres"  
export pg\_password="postgres"  
export cleanup\_packages="true"  
export cm\_port="6703"  
export etcd\_peer\_port="2380"  
export etcd\_client\_port="2379"  
export ansible\_collection\_path="gs://alloydb-nova-sandbox/orchestrator/ansible/latest"  
export orchestrator\_rpm\_path="https\://us-central1-yum.pkg.dev/projects/alloydb-omni-sandbox/nova"  
export cmctl\_version="0.0.1-\*\_RC00.rhel9"  
export client="ansible"  
export num\_db\_hosts=1  
export omni\_svc\_version="" \# e.g. "0.0.1-1.rhel9"  
export ma\_svc\_version="" \# e.g. "0.4.0-1.rhel9"  
export haproxy\_version=""  
export haproxy\_hosts=""  
export alloydb\_version=""  
export debug="false"  \# Set to "true" to enable debug|verbose output  
export install\_ansible\_collection="true"  
export protobuf\_version="3.14.0-16.el9.noarch"  
export grpcio\_version="1.46.7-10.el9.x86\_64"  
export googleapis\_common\_protos\_version="1.56.0-4.el9.noarch"  
export cpu="2"  
export memory="64Gi"  
export virtual\_ip="10.1.0.100"  
export autoFailoverTriggerThreshold=2  
export healthcheckPeriodSeconds=10  
\# TODO(b/474520936): Make this "enabled" once SSL is supported for CM for etcd communication.  
export ssl="disabled"  
function alloydb\_cleanup\_packages() {  
  deployment\_spec\_file="/tmp/deployment\_spec.yaml"  
  deployment\_spec\_create "\$deployment\_spec\_file"  
  cat \> /tmp/uninstall.yaml \<\< EOF  
\---  
\- name: Uninstall AlloyDB Omni  
  hosts: all  
  vars:  
    ansible\_become: true  
    ansible\_user: \${service\_account}  
    ansible\_ssh\_private\_key\_file: \~/ssh-key-cluster-sa  
    cleanup\_data: true  
    cleanup\_config: true  
    force: true  
  roles:  
   \- role: google.alloydbomni\_orchestrator.uninstall  
EOF  
  run cat /tmp/uninstall.yaml  
  debug\_spec= && \[\[ "\$debug" \= "true" \]\] && debug\_spec="-e log\_level=DEBUG \-vvv"  
  run ansible-playbook /tmp/uninstall.yaml \-i "\$deployment\_spec\_file" "\$debug\_spec"  
  return 0  
}  
function alloydb\_destroy() {  
  alloydb\_delete || true  
  \# TODO(rkhosla): Remove this once node manager includes all these cleanup steps.  
  run\_on\_all\_hosts "  
    sudo rm \-f /etc/alloydbomni/node\_manager\_roles.toml;  
    sudo rm \-rf /var/lib/postgresql/\$pg\_version/data;  
    sudo rm \-rf \$pg\_data/\$pg\_version;  
    sudo rm \-rf \$pg\_data/kvstore;  
    sudo rm \-rf \$pg\_data/.pgpass;  
    sudo rm \-rf /var/lib/nm;  
    sudo rm \-rf /etc/haproxy || true;  
    sudo rm \-rf /etc/keepalived || true;  
    sudo rm \-rf /etc/systemd/system/haproxy.service.d || true;  
    sudo rm \-rf /etc/systemd/system/keepalived.service.d || true;  
    sudo rm \-rf /etc/systemd/system/alloydbomni\$pg\_version.service.d/override.conf;"  
  \[\[ "\$cleanup\_packages" \= "true" \]\] && alloydb\_cleanup\_packages  
  rm \-f /tmp/.alloydb\_prepare.\$pg\_version;  
}  
function deployment\_spec\_create {  
  deployment\_spec\_file=\$1  
  deployment\_spec\_tmpl="\${deployment\_spec\_file}.tmpl"  
  LOAD\_BALANCER\_SECTION=""  
  if \[\[ \${\#haproxy\_hosts\[@\]} \-gt 0 \]\]; then  
    LOAD\_BALANCER\_SECTION=\$(cat \<\<EOS  
    load\_balancer\_nodes:  
      hosts:  
VAR\_HAPROXY\_HOSTS  
EOS  
)  
  fi  
    \# Install RPMs on all the db hosts  
  cat \> "\${deployment\_spec\_tmpl}" \<\< EOF  
alloydbomni:  
  vars:  
    cluster\_name: "\$cluster"  
  children:  
    primary\_instance\_nodes:  
      hosts:  
VAR\_DB\_HOSTS  
    cluster\_manager\_nodes:  
      hosts:  
VAR\_DB\_HOSTS  
\${LOAD\_BALANCER\_SECTION}  
EOF  
  var\_db\_hosts=\$(printf "        %s:\#EOL\#" "\${db\_hosts\[@\]}")  
  sed \-i "s|VAR\_DB\_HOSTS|\${var\_db\_hosts}|g" "\${deployment\_spec\_tmpl}"  
  var\_haproxy\_hosts=\$(printf "        %s:\#EOL\#" "\${haproxy\_hosts\[@\]}")  
  sed \-i "s|VAR\_HAPROXY\_HOSTS|\${var\_haproxy\_hosts}|g" "\${deployment\_spec\_tmpl}"  
  \# Prepare the deployment spec file from the template.  
  sed "s|\#EOL\#|\\n|g" "\${deployment\_spec\_tmpl}" \> "\$deployment\_spec\_file"  
  run cat "\${deployment\_spec\_file}"  
}  
function resource\_spec\_create {  
  resource\_spec\_file=\$1  
  resource\_spec\_tmpl="\${resource\_spec\_file}.tmpl"  
  cat \> "\${resource\_spec\_file}" \<\< EOF  
\---  
Secret:  
  metadata:  
    name: db-pw-\$cluster  
  spec:  
    type: Opaque  
    data:  
      \$cluster: \$(echo \-n "\$pg\_password" | base64)  
\---  
DBCluster:  
  metadata:  
    name: \$cluster  
  spec:  
    databaseVersion: 17.5.0  
    mode: ""  
    primarySpec:  
      adminUser:  
        passwordRef:  
          name: db-pw-\$cluster  
      resources:  
        cpu: \$cpu  
        memory: \$memory  
      dbLoadBalancerOptions:  
        gcp:  
          loadBalancerIP: "\$virtual\_ip"  
          loadBalancerType: "internal"  
          loadBalancerInterface: "eth0"  
EOF  
  if \[\[ \$num\_db\_hosts \-gt 1 \]\]; then  
    let num\_standbys=num\_db\_hosts-1  
    export num\_standbys  
    yq 'select(has("DBCluster")) |= (  
    .DBCluster.spec.availability.numberOfStandbys \= (env(num\_standbys) | tonumber) |  
    .DBCluster.spec.availability.enableAutoFailover \= true |  
    .DBCluster.spec.availability.autoFailoverTriggerThreshold \= (env(autoFailoverTriggerThreshold) | tonumber) |  
    .DBCluster.spec.availability.healthcheckPeriodSeconds \= (env(healthcheckPeriodSeconds) | tonumber)  
    )' "\${resource\_spec\_file}" \> "\${resource\_spec\_tmpl}"  
    unset num\_standbys  
    mv "\${resource\_spec\_tmpl}" "\${resource\_spec\_file}"  
  fi  
  run cat "\${resource\_spec\_file}"  
}  
function ansible\_collection\_install {  
  if \[\[ "\$ansible\_collection\_path" \== "gs:/"\*  \]\]; then  
    run "gsutil \-v" || error "gsutil is not installed. Please install gsutil before running this script."  
    ansible\_collection\_name=\$(basename "\$ansible\_collection\_path")  
    ansible\_collection\_path=\$(gsutil ls "\$ansible\_collection\_path" | grep google-alloydbomni\_orchestrator | tail \-n 1\)  
    run gsutil cp "\$ansible\_collection\_path" "/tmp/\$ansible\_collection\_name"  
    ansible\_collection\_path="/tmp/\$ansible\_collection\_name"  
  fi  
  ansible-galaxy \--version ||  
    error "ansible-galaxy is not installed. Please install ansible-galaxy before running this script."  
  run "tar \-xOf \$ansible\_collection\_path MANIFEST.json  | jq .file\_manifest\_file"  
  run ansible-galaxy collection install \--force "\$ansible\_collection\_path"  
  run "ansible-galaxy collection list | grep \-q google.alloydbomni\_orchestrator" ||  
    error "Failed to install alloydb omni orchestrator ansible collection."  
}  
function install\_google\_ar\_plugin {  
  cat \<\<EOF | sudo tee /etc/yum.repos.d/google-ar-plugin.repo \> /dev/null  
\[ar-plugin\]  
name=Artifact Registry Plugin  
baseurl=https\://packages.cloud.google.com/yum/repos/dnf-plugin-artifact-registry-stable  
enabled=1  
gpgcheck=1  
repo\_gpgcheck=1  
gpgkey=https\://packages.cloud.google.com/yum/doc/yum-key.gpg https\://packages.cloud.google.com/yum/doc/rpm-package-key.gpg  
EOF  
  run sudo dnf install \-y dnf-plugin-artifact-registry || error "install\_google\_ar\_plugin: Failed to install dnf-plugin-artifact-registry"  
}  
function install\_orchestrator\_package {  
  if \[\[ "\$orchestrator\_rpm\_path" \== "https\://"\* \]\]; then  
      \[\[ \-z "\$cmctl\_version" \]\] && error "install\_orchestrator\_package: cmctl\_version is not specified"  
      if \[\[ "\$orchestrator\_rpm\_path" \== \*"-yum.pkg.dev"\* \]\]; then  
          install\_google\_ar\_plugin  
      fi  
      cat \<\<EOF | sudo tee "/etc/yum.repos.d/google-ar-alloydbomni\_orchestrator.repo" \> /dev/null  
\[alloydbomni\_orchestrator\]  
name=Google Artifact Registry Packages  
baseurl=\${orchestrator\_rpm\_path}  
enabled=1  
repo\_gpgcheck=1  
gpgcheck=0  
gpgkey=https\://us-central1-yum.pkg.dev/doc/repo-signing-key.gpg  
EOF  
      run sudo dnf makecache \-y  
      run sudo dnf install \-y "alloydbomni\_orchestrator-\${cmctl\_version}" || error "install\_orchestrator\_package: Failed to install alloydbomni\_orchestrator-\${cmctl\_version}"  
  elif \[\[ "\$orchestrator\_rpm\_path" \== "gs://"\* \]\]; then  
      gsutil \-v || error "install\_orchestrator\_package: gsutil is not installed. Please install gsutil."  
      local orchestrator\_gcs\_path  
      orchestrator\_gcs\_path=\$(gsutil ls "\$orchestrator\_rpm\_path" | grep alloydbomni\_orchestrator | tail \-n 1\)  
      \[\[ \-z "\$orchestrator\_gcs\_path" \]\] && error "install\_orchestrator\_package: No matching package found in GCS path: \$orchestrator\_rpm\_path"  
      local orchestrator\_file\_name  
      orchestrator\_file\_name=\$(basename "\$orchestrator\_gcs\_path")  
      run gsutil cp "\$orchestrator\_gcs\_path" "/tmp/\$orchestrator\_file\_name"  
      run sudo dnf install \-y "/tmp/\$orchestrator\_file\_name" || error "install\_orchestrator\_package: Failed to install \$orchestrator\_file\_name"  
  else  
      if \[\[ "\$orchestrator\_rpm\_path" \!= \*".rpm" \]\]; then  
          error "install\_orchestrator\_package: The provided path does not end with .rpm extension: \$orchestrator\_rpm\_path"  
      fi  
      \[\[ \! \-f "\$orchestrator\_rpm\_path" \]\] && error "install\_orchestrator\_package: Local file not found at \$orchestrator\_rpm\_path"  
      run sudo dnf install \-y "\$orchestrator\_rpm\_path" || error "install\_orchestrator\_package: Failed to install local package"  
  fi  
}  
function alloydb\_prepare {  
  \[\[ \-z "\$service\_account" \]\] && error "service\_account is not specified"  
  ansible\_install\_path="\~/.ansible/collections/ansible\_collections/google/alloydbomni\_orchestrator"  
  \[\[ "\$install\_ansible\_collection" \= "true" \]\] && ansible\_collection\_install  
  run\_on\_all\_hosts "sudo setenforce 0;"  
  \[\[ \$num\_haproxy\_hosts \-gt 0 \]\] && run\_on\_haproxy\_hosts "sudo setenforce 0;"  
  if \[\[ "\$omni\_svc\_rpm\_repo\_url" \== "file://"\* \]\]; then  
    local\_path="\${omni\_svc\_rpm\_repo\_url\#\#file://}";  
    run\_on\_all\_hosts "  
      ls \$local\_path;  
      \[\[ \! \-d \$local\_path \]\] && echo "error: \$local\_path does not exist" && exit 1;  
      sudo dnf install \-y createrepo;  
      sudo createrepo \--update \$local\_path;"  
  fi  
  if \[\[ "\$omni\_svc\_rpm\_repo\_url" \== "gs://"\* \]\]; then  
    local\_path="/home/\${service\_account}/omnisvc"  
    run\_on\_all\_hosts "  
      mkdir \-p \$local\_path;  
      gsutil cp \-r \$omni\_svc\_rpm\_repo\_url \$local\_path;  
      ls \-l \$local\_path;  
      sudo dnf install \-y createrepo;  
      sudo createrepo \--update \$local\_path;"  
    omni\_svc\_rpm\_repo\_url="file://\$local\_path"  
  fi  
  if \[\[ "\$etcd\_repo\_url" \== "gs://"\* \]\]; then  
    local\_path="/home/\${service\_account}/etcd"  
    etcd\_rpm\_path=\$(gsutil ls "\$etcd\_repo\_url" | grep "/etcd-" | tail \-n 1\)  
    run\_on\_db\_hosts "  
      mkdir \-p \$local\_path;  
      gsutil cp \-r \$etcd\_rpm\_path \$local\_path;  
      ls \-l \$local\_path;  
      sudo dnf install \-y createrepo;  
      sudo createrepo \--update \$local\_path;"  
    etcd\_repo\_url="file://\$local\_path"  
  fi  
  deployment\_spec\_file="/tmp/deployment\_spec.yaml"  
  deployment\_spec\_create "\$deployment\_spec\_file"  
  alloydb\_version\_spec= && \[\[ \-n "\$alloydb\_version" \]\] && alloydb\_version\_spec="version: \\"\$alloydb\_version\\""  
  if \[\[ "\${ssl:-}" \= "enabled" \]\]; then  
    ssl\_config="      ssl:  
        enabled: true"  
  else  
    ssl\_config="      ssl:  
        \# TODO(b/474520936): Remove it once SSL for CM for etcd communication is working.  
        enabled: false"  
  fi  
  cat \> /tmp/install.yaml \<\< EOF  
\---  
\- name: Deploy AlloyDB Omni  
  hosts: all  
  vars:  
    disable\_gpgcheck: true  
    ansible\_become: true  
    ansible\_user: \${service\_account}  
    ansible\_ssh\_private\_key\_file: \~/ssh-key-cluster-sa  
    setup\_etcd: true  
    etcd:  
      config\_forcewrite: true  
      repo\_url: "\$etcd\_repo\_url"  
\$ssl\_config  
    alloydbomni:  
      major\_version: "\$pg\_version"  
      repo\_url: "\$alloydb\_rpm\_repo\_url"  
      \$alloydb\_version\_spec  
    alloydbomni\_monitor:  
      repo\_url: "\$alloydb\_rpm\_repo\_url"  
      version: "\$ma\_svc\_version"  
    alloydbomni\_cluster\_manager:  
      repo\_url: "\$omni\_svc\_rpm\_repo\_url"  
      VAR\_OMNI\_SVC\_VERSION  
    alloydbomni\_node\_manager:  
      repo\_url: "\$omni\_svc\_rpm\_repo\_url"  
      VAR\_OMNI\_SVC\_VERSION  
    haproxy:  
      VAR\_HAPROXY\_SVC\_VERSION  
  roles:  
   \- role: google.alloydbomni\_orchestrator.install  
EOF  
  omni\_svc\_version\_spec=""  
  \[\[ \-n "\$omni\_svc\_version" \]\] && omni\_svc\_version\_spec="version: \$omni\_svc\_version"  
  sed \-i "s|VAR\_OMNI\_SVC\_VERSION|\$omni\_svc\_version\_spec|g" /tmp/install.yaml  
  haproxy\_version\_spec=""  
  \[\[ \-n "\$haproxy\_version" \]\] && haproxy\_version\_spec="version: \$haproxy\_version"  
  sed \-i "s|VAR\_HAPROXY\_SVC\_VERSION|\$haproxy\_version\_spec|g" /tmp/install.yaml  
  run cat /tmp/install.yaml  
  debug\_spec= && \[\[ "\$debug" \= "true" \]\] && debug\_spec="-e log\_level=DEBUG \-vvv"  
  run ansible-playbook /tmp/install.yaml \-i "\$deployment\_spec\_file" "\$debug\_spec"  
  if \[\[ "\${ssl:-}" \= "enabled" \]\]; then  
      etcd\_endpoints\_list=(\$(echo "\${db\_hosts\_ips\[@\]}" | xargs \-n 1 | sed "s/\$/:\$etcd\_client\_port/g" | sed 's/^/https:\\/\\//'))  
      curl\_flags="--cacert /var/lib/etcd/ssl/rootca.crt \--cert /var/lib/etcd/ssl/etcd.crt \--key /var/lib/etcd/ssl/etcd.key"  
  else  
      etcd\_endpoints\_list=(\$(echo "\${db\_hosts\_ips\[@\]}" | xargs \-n 1 | sed "s/\$/:\$etcd\_client\_port/g" | sed 's/^/http:\\/\\//'))  
      curl\_flags=""  
  fi  
  \# Wait for etcd to be ready  
  for ((i=0; i \< 20; i++)); do  
    run\_on\_db\_hosts "  
      sudo systemctl status etcd \--lines=0 && \\  
      sudo curl \-sL \$curl\_flags \\"\${etcd\_endpoints\_list\[0\]}/health\\" | grep \-q '\\"health\\":\\"true\\"'" && break  
    sleep 10  
  done  
  \# Install client i.e. psql on the current node for testing  
  rpm \-qa | grep \-q pgdg-redhat-repo-latest || run sudo dnf install \-y "\$pgdg\_reporpms\_url";  
  rpm \-qa | grep \-q postgresql\$pg\_version || run sudo dnf install \-y "postgresql\$pg\_version";  
  rpm \-qa | grep \-q nmap-ncat || run sudo dnf install \-y nmap-ncat;  
  rpm \-qa | grep \-q epel-release || run sudo dnf install \-y epel-release;  
  rpm \-qa | grep \-q yq || run sudo dnf install \-y yq;  
  rpm \-qa | grep \-q golang || run sudo dnf install \-y golang;  
  run go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest;  
  run\_on\_db\_hosts "  
    sudo journalctl \--rotate;  
    sudo journalctl \--vacuum-time=1s;"  
  install\_orchestrator\_package  
  cm\_setup  
  run\_on\_db\_hosts "rpm \-qa | egrep \\"(alloydbomni|etcd)\\";"  
  touch /tmp/.alloydb\_prepare.\$pg\_version  
}  
function cm\_init {  
  etcd\_endpoints=\$(printf "\\"%s:\$etcd\_client\_port\\"\\n" "\${db\_hosts\_ips\[@\]}" | jq \-sc .)  
  cat \> /tmp/init.json \<\<EOF  
{  
  "config": {  
    "cluster": {  
      "cluster\_name": "\$cluster"  
    },  
    "dcs": {  
      "endpoints": \$etcd\_endpoints  
    }  
  }  
}  
EOF  
  run cat /tmp/init.json  
  for host in "\${db\_hosts\[@\]}"; do  
    run "\$(go env GOPATH)/bin/grpcurl \-plaintext \-d @ \$host:\$cm\_port cmproto.ClusterManager.Initialize \< /tmp/init.json"  
    sleep 2  
  done  
}  
function cm\_setup() {  
  \# TODO(lujjwal): Remove the override once node manager config is in place.  
  run\_on\_db\_hosts "  
    sudo mkdir \-p /etc/systemd/system/alloydbomni\$pg\_version.service.d;  
    echo "\[Service\]" | sudo tee /etc/systemd/system/alloydbomni\$pg\_version.service.d/override.conf;  
    echo "LimitNICE=-16" | sudo tee \-a /etc/systemd/system/alloydbomni\$pg\_version.service.d/override.conf;  
    sudo systemctl daemon-reload;  
    sudo systemctl show \-p Environment alloydbomni\$pg\_version.service;  
    sudo grep \-q enable\_reflection /etc/alloydbomni/cluster\_manager.yaml ||  
      sudo sed \-i '/listen\_addr: \\":\$cm\_port\\"/a \\    \# Whether to enable reflection for the API server.\\n    enable\_reflection: true' /etc/alloydbomni/cluster\_manager.yaml;"  
  run\_on\_db\_hosts "  
    sudo systemctl restart alloydbomni\_node\_manager;  
    sudo systemctl status alloydbomni\_node\_manager;"  
  \# Start cluster manager  
  run\_on\_db\_hosts "  
    sudo systemctl restart alloydbomni\_cluster\_manager;  
    sudo systemctl status alloydbomni\_cluster\_manager; exit 0;"  
}  
function ansible\_alloydb\_bootstrap() {  
  \[\[ \-z \$pg\_version \]\] && error "ansible\_alloydb\_bootstrap: pg\_version is not specified"  
  run sudo dnf install \-y python3-protobuf"\${protobuf\_version:+-}\${protobuf\_version}"  
  run sudo dnf install \-y python3-googleapis-common-protos"\${googleapis\_common\_protos\_version:+-}\${googleapis\_common\_protos\_version}"  
  run sudo dnf install \-y python3-grpcio"\${grpcio\_version:+-}\${grpcio\_version}"  
  deployment\_spec\_file="/tmp/deployment\_spec.yaml"  
  deployment\_spec\_create "\$deployment\_spec\_file"  
  resource\_spec\_file="/tmp/resource\_spec.yaml"  
  resource\_spec\_create "\$resource\_spec\_file"  
  cat \> /tmp/bootstrap.yaml \<\< EOF  
\---  
\- name: Deploy AlloyDB Omni  
  hosts: all  
  vars:  
   ansible\_user: \${service\_account}  
   ansible\_ssh\_private\_key\_file: \~/ssh-key-cluster-sa  
  roles:  
   \- role: google.alloydbomni\_orchestrator.bootstrap  
EOF  
  debug\_spec= && \[\[ "\$debug" \= "true" \]\] && debug\_spec="-e log\_level=DEBUG \-vvv"  
  run ansible-playbook /tmp/bootstrap.yaml \-i "\$deployment\_spec\_file" \-e resource\_spec="\$resource\_spec\_file" "\$debug\_spec"  
}  
function get\_cm\_leader\_host() {  
  export leader\_host  
  leader\_host=\$(get\_cm\_leader)  
  if \[\[ \-z "\$leader\_host" \]\]; then  
    echo "error: Could not get leader host" && return 1  
  fi  
}  
function grpcurl\_alloydb\_bootstrap {  
  cm\_init  
  \# TODO(ammaarahmad): Check if we need to remove dependency on cluster name.  
  generate\_secret\_file "/tmp/secret.json" "db-pw-\$cluster" "\$cluster" "\$pg\_password"  
  cat \> /tmp/dbcluster.json \<\<EOF  
{  
  "type": {  
    "kind": "DBCluster"  
  },  
  "metadata": {  
    "name": "\$cluster"  
  },  
  "resource\_spec": {  
    "primarySpec": {  
      "adminUser": {  
        "passwordRef": {  
          "name": "db-pw-\$cluster"  
        }  
      },  
      "resources": {  
        "cpu": "\$cpu",  
        "memory": "\$memory"  
      },  
      "dbLoadBalancerOptions": {  
        "gcp": {  
          "loadBalancerIP": "\$virtual\_ip",  
          "loadBalancerType": "internal",  
          "loadBalancerInterface": "eth0"  
        }  
      }  
    }  
  }  
}  
EOF  
  if \[\[ \$num\_db\_hosts \-gt 1 \]\]; then  
    let num\_standbys=num\_db\_hosts-1  
    jq ".resource\_spec.availability.numberOfStandbys \= \$num\_standbys | .resource\_spec.availability.enableAutoFailover \= true | .resource\_spec.availability.autoFailoverTriggerThreshold \= \$autoFailoverTriggerThreshold | .resource\_spec.availability.healthcheckPeriodSeconds \= \$healthcheckPeriodSeconds" /tmp/dbcluster.json \> /tmp/\$\$.json &&  
    mv /tmp/\$\$.json /tmp/dbcluster.json  
  fi  
  haproxy\_hosts\_list=\$(printf '"%s"\\n' "\${haproxy\_hosts\[@\]}" | jq \-sc .)  
  db\_hosts\_list=\$(printf '"%s"\\n' "\${db\_hosts\[@\]}" | jq \-sc .)  
  jq \--argjson db\_hosts "\$db\_hosts\_list" '.deployment\_spec.primary\_instance\_nodes.hosts \+= \$db\_hosts' /tmp/dbcluster.json \> /tmp/\$\$.json  
  mv /tmp/\$\$.json /tmp/dbcluster.json  
  if \[\[ \$num\_haproxy\_hosts \-gt 0 \]\]; then  
    haproxy\_hosts\_list=\$(printf '"%s"\\n' "\${haproxy\_hosts\[@\]}" | jq \-sc .)  
    jq ".deployment\_spec.load\_balancer\_nodes.hosts \+= \$haproxy\_hosts\_list" /tmp/dbcluster.json \> /tmp/\$\$.json  
    mv /tmp/\$\$.json /tmp/dbcluster.json  
  fi  
  get\_cm\_leader\_host  
  echo "Applying secret"  
  run cat /tmp/secret.json  
  run "\$(go env GOPATH)/bin/grpcurl \-plaintext \-d @ \$leader\_host:\$cm\_port cmproto.ClusterManager.Apply \< /tmp/secret.json"  
  echo "Applying dbcluster"  
  run cat /tmp/dbcluster.json  
  run "\$(go env GOPATH)/bin/grpcurl \-plaintext \-d @ \$leader\_host:\$cm\_port cmproto.ClusterManager.Apply \< /tmp/dbcluster.json"  
  \# Wait for cluster to be ready  
  echo "AlloyDB Omni cluster created. Allow some time for it to be ready."  
}  
\#TODO(rkhosla): Add code for secret deletion once Delete API support this feature.  
function grpcurl\_alloydb\_delete {  
  get\_cm\_leader\_host  
  cat \> /tmp/delete\_dbcluster.json \<\<EOF  
{  
    "type": {  
      "kind": "DBCluster"  
    },  
    "name": "\$cluster"  
}  
EOF  
  run cat /tmp/delete\_dbcluster.json  
  run "\$(go env GOPATH)/bin/grpcurl \-plaintext \-d @ \$leader\_host:\$cm\_port cmproto.ClusterManager.Delete \< /tmp/delete\_dbcluster.json"  
}  
function cmctl\_alloydb\_bootstrap {  
  \[\[ \-z \$pg\_version \]\] && error "cmctl\_alloydb\_bootstrap: pg\_version is not specified"  
  rpm \-qa | grep alloydbomni\_orchestrator ||  
    error "error: alloydbomni\_orchestrator is not installed."  
  resource\_spec\_file="/tmp/resource\_spec.yaml"  
  resource\_spec\_create "\$resource\_spec\_file"  
  deployment\_spec\_file="/tmp/deployment\_spec.yaml"  
  deployment\_spec\_create "\$deployment\_spec\_file"  
  run /usr/local/bin/cmctl apply \-d "\$deployment\_spec\_file" \-r "\$resource\_spec\_file"  
}  
function ansible\_alloydb\_delete {  
  deployment\_spec\_file="/tmp/deployment\_spec.yaml"  
  deployment\_spec\_create "\$deployment\_spec\_file"  
  debug\_spec=&& \[\[ "\$debug" \= "true" \]\] && debug\_spec="-e log\_level=DEBUG \-vvv"  
  cat \> /tmp/delete.yaml \<\< EOF  
\---  
\- name: Delete AlloyDB Omni Resource  
  hosts: localhost  
  vars:  
   cluster\_name: \${cluster}  
   ansible\_user: \${service\_account}  
   ansible\_ssh\_private\_key\_file: \~/ssh-key-cluster-sa  
   force: true  
  roles:  
   \- role: google.alloydbomni\_orchestrator.delete  
EOF  
  run cat /tmp/delete.yaml  
  run ansible-playbook /tmp/delete.yaml \-i "\$deployment\_spec\_file" "\$debug\_spec" \-e "resource\_name=\${cluster}" \-e "resource\_type=DBCluster"  
}  
function alloydb\_delete() {  
  case \$client in  
    ansible) ansible\_alloydb\_delete;;  
    grpcurl) grpcurl\_alloydb\_delete;;  
    \*) error "invalid delete method \$client"; usage; exit 1;;  
  esac  
}  
function alloydb\_create() {  
  \[\[ \-z \$pg\_version \]\] && error "alloydb\_config: pg\_version is not specified"  
  \[\[ \! \-f /tmp/.alloydb\_prepare.\$pg\_version \]\] && alloydb\_prepare  
  case \$client in  
    ansible) ansible\_alloydb\_bootstrap;;  
    grpcurl) grpcurl\_alloydb\_bootstrap;;  
    cmctl) cmctl\_alloydb\_bootstrap;;  
    \*) error "invalid bootstrap method \$client"; usage; exit 1;;  
  esac  
}  
function grpcurl\_alloydb\_status {  
  cat \> /tmp/get-secret.json \<\<EOF  
{  
    "type": {  
      "kind": "Secret"  
    },  
    "name": "db-pw-\$cluster"  
}  
EOF  
  run cat /tmp/get-secret.json  
  run "\$(go env GOPATH)/bin/grpcurl \-plaintext \-d @ \$db\_host:\$cm\_port cmproto.ClusterManager.Get \< /tmp/get-secret.json"  
  cat \> /tmp/get-dbcluster.json \<\<EOF  
{  
    "type": {  
      "kind": "DBCluster"  
    },  
    "name": "\$cluster"  
}  
EOF  
  run cat /tmp/get-dbcluster.json  
  run "\$(go env GOPATH)/bin/grpcurl \-plaintext \-d @ \$db\_host:\$cm\_port cmproto.ClusterManager.Get \< /tmp/get-dbcluster.json"  
}  
function cmctl\_alloydb\_status {  
  deployment\_spec\_file="/tmp/deployment\_spec.yaml"  
  deployment\_spec\_create "\$deployment\_spec\_file"  
  run /usr/local/bin/cmctl  get \-d "\$deployment\_spec\_file" \-t Secret \-n "db-pw-\$cluster" \-o yaml  
  run /usr/local/bin/cmctl  get \-d "\$deployment\_spec\_file" \-t DBCluster \-n "\$cluster" \-o yaml  
}  
function ansible\_alloydb\_status {  
  cat \> /tmp/status.yaml \<\< EOF  
\---  
\- name: Get AlloyDB Omni status  
  hosts: localhost  
  vars:  
   ansible\_user: \${service\_account}  
   ansible\_ssh\_private\_key\_file: \~/ssh-key-cluster-sa  
  roles:  
   \- role: google.alloydbomni\_orchestrator.status  
EOF  
  deployment\_spec\_file="/tmp/deployment\_spec.yaml"  
  deployment\_spec\_create "\$deployment\_spec\_file"  
  run ansible-playbook /tmp/status.yaml \-i "\$deployment\_spec\_file" \-e resource\_type="Secret" \-e name="db-pw-\$cluster"  
  run ansible-playbook /tmp/status.yaml \-i "\$deployment\_spec\_file" \-e resource\_type="DBCluster" \-e name="\$cluster"  
}  
function alloydb\_status() {  
  case \$client in  
    ansible) ansible\_alloydb\_status;;  
    grpcurl) grpcurl\_alloydb\_status;;  
    cmctl) cmctl\_alloydb\_status;;  
    \*) error "invalid bootstrap method \$client"; usage; exit 1;;  
  esac  
  run\_on\_db\_hosts "sudo systemctl status alloydbomni\$pg\_version;"  
  run\_on\_db\_hosts "sudo systemctl status alloydbomni\_monitor;"  
}  
function alloydb\_test {  
  \[\[ \-z \$pg\_port \]\] && error "alloydb\_test: pg\_port is not specified"  
  \[\[ \-z \$pg\_version \]\] && error "alloydb\_test: pg\_version is not specified"  
  \[\[ \-z \$pg\_passwd \]\] && error "alloydb\_test: pg\_passwd is not specified"  
  export log\_file=\${log\_file:="/tmp/alloydb\_test.log"}  
  \[\[ \-n "\${test\_list:-}" \]\] && test\_list=\$(echo "\$test\_list" | tr ',' ' ')  
  \[\[ \-n "\${skip\_list:-}" \]\] && skip\_list=\$(echo "\$skip\_list" | tr ',' ' ')  
  source "\${script\_dir}"/alloydb-test.sh  
  run\_test || echo \-e "\\n\#\#\# test failed \#\#\#"  
}  
function usage {  
  echo \-e "\\n  
usage: alloydb-setup.sh \[-h\] {prepare|create|destroy} \-c \<cluster conf\> \<var\>=\<val\>  
where var:  
  service\_account       \- Required. Service Account Name  
  cluster               \- Required. Cluster Name.  
  pg\_version            \- Optional. PG version. Default is 17  
  db\_hosts              \- Optional. Comma separated no space e.g. node1,node2,node3  
                          Else generated with cluster name as prefix  
  db\_hosts\_ips          \- Optional. Space separated e.g. node1\_ip node2\_ip node3\_ip  
                          Else generated from the names of db\_hosts  
  pg\_data               \- Optional. Default: /data  
  pg\_user               \- Optional. Default postgres  
  cleanup\_packages      \- Optional. Default true. Set to 'false' to disable removal of rpms  
  etcd\_repo\_url         \- etcd repo url. Default:  
                          https\://ftp.postgresql.org/pub/repos/yum/common/pgdg-rhel9-extras/redhat/rhel-9-x86\_64  
                          can also be like: gs://alloydb-omni-custom-releases/alloydbomni/bundle/0.0.2/rpms/  
  pgdg\_reporpms\_url     \- Optional. For Postgres Client. default:  
                          https\://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86\_64/pgdg-redhat-repo-latest.noarch.rpm  
  alloydb\_rpm\_repo\_url  \- Optional. Repo URL for AllloyDB Omni RPMs. Default:  
                          https\://us-central1-yum.pkg.dev/projects/alloydb-omni-sandbox/nova  
  omni\_svc\_rpm\_repo\_url \- Optional. Repo URL for Omni Service RPMs. Default:  
                          https\://us-central1-yum.pkg.dev/projects/alloydb-omni-sandbox/nova  
  ansible\_collection\_path-location of AlloyDB Omni Ansible Collection. Default:  
                          gs://alloydb-nova-sandbox/orchestrator/ansible/latest  
  orchestrator\_rpm\_path      \- AlloyDB Omni Orchestrator CLI RPM. Default:  
                          Artifact Registry  
  client                \- Optional. Client to control the cluster.  
                          Default: ansible. Supported values: ansible, grpcurl, cmctl  
  omni\_svc\_version      \- Optional. Version of Omni Service. e.g. 0.0.1-1.rhel9  
  haproxy\_version       \- Optional. Version of haproxy. e.g. 2.8.14  
  debug                 \- Optional. Set to 'true' to enable debug|verbose output  
  test\_list            \-  Optional. Comma separated list of tests to run.  
                          Default: All tests in alloydb-test.sh  
                          applicable only for test command  
  skip\_list            \-  Optional. Comma separated list of tests to skip.  
                          Default: None  
                          applicable only for test command  
  ssl                   \- Optional. Default disabled. Set ssl=enabled to enable SSL.  
Example:  
\\\$ alloydb-setup.sh destroy \-c /tmp/cluster.conf cleanup\_packages=false  
\\\$ alloydb-setup.sh prepare \-c /tmp/cluster.conf \\  
  ansible\_collection\_path=gs://alloydb-nova-sandbox/orchestrator/ansible/latest \\  
  ssl=disabled  
\\\$ alloydb-setup.sh create \-c /tmp/cluster.conf  
"  
}  
function main {  
  cmd=\${1:-}; shift  
  case \$cmd in  
    prepare | create | status | cleanup | destroy | test ) ;;  
    \-h | \--help ) usage;;  
    \* ) error "invalid command \$cmd"; usage;;  
  esac  
  parse\_args "\$@" || usage  
  \[\[ \-z "\$prompt" \]\] && echo \-e "Run with 'prompt=1' to run with prompt\\n"  
  \[\[ \-z \$cluster \]\] && error "cluster name is not specified" && usage  
  \[\[ \-z "\$service\_account" \]\] && error "service\_account is not specified" && usage  
  export db\_hosts=\${db\_hosts:-"\$cluster-db1"}  
  IFS=',' read \-ra db\_hosts \<\<\< "\$db\_hosts"  
  export db\_host="\${db\_host:=\${db\_hosts\[0\]}}"  
  local haproxy\_hosts\_str="\${haproxy\_hosts:-}"  
  if \[\[ \-z "\$haproxy\_hosts\_str" \]\]; then  
    haproxy\_hosts=()  
  else  
    IFS=',' read \-ra haproxy\_hosts \<\<\< "\$haproxy\_hosts\_str"  
  fi  
  export num\_haproxy\_hosts=\${\#haproxy\_hosts\[@\]}  
  export num\_db\_hosts=\${\#db\_hosts\[@\]}  
  \[\[ \-n "\${db\_hosts\_ips:-}" \]\] && IFS=' ' read \-ra db\_hosts\_ips \<\<\< "\$db\_hosts\_ips"  
  \[\[ \-z "\${db\_hosts\_ips:-}" \]\] &&  
    export db\_hosts\_ips=(\$(echo "\${db\_hosts\[@\]}" | xargs \-n 1 getent hosts | awk '{print \$1}'))  
  eval alloydb\_"\$cmd"  
}  
main "\$@"  
contents inside cluster.conf:  
\[satyasais\_google\_com@satyasais-debug-cmcli-control \~\]\$ cat /tmp/cluster.conf   
export project=alloydb-nova-tvc-sandbox  
export cluster=satyasais-debug-cmcli  
export db\_hosts=satyasais-debug-cmcli-db1,satyasais-debug-cmcli-db2,satyasais-debug-cmcli-db3  
export haproxy\_hosts=satyasais-debug-cmcli-haproxy1,satyasais-debug-cmcli-haproxy2  
export external\_host=  
export db\_hosts\_ips="10.1.0.4 10.1.0.3 10.1.0.2"  
export haproxy\_hosts\_ips="10.1.0.6 10.1.0.5"  
export service\_account=sa\_105211340377577433636  
I'm running this command: /tmp/orchestrator/alloydb-setup.sh prepare \-c /tmp/cluster.conf  
I have private key in this location: /home/satyasais\_google\_com/ssh-key-cluster-sa  
My command should look like this:  
./alloydbomni\_dump.sh \-k /home/satyasais\_google\_com/ssh-key-cluster-sa \-out=/tmp/debug\_dump \-tag=20250114  
and the output should be as same like structure i got for Ansible role execution:  
\[satyasais\_google\_com@satyasais-debug-role-control debug\_outputs\]\$ tar \-tvf alloydbomni\_cluster\_20260120\_142333.tar.gz  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni17/  
\-rw-r--r-- root/root       272 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni17/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_monitor/  
\-rw-r--r-- root/root       195 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_monitor/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       542 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/etcd/  
\-rw-r--r-- root/root       381 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/etcd/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/pgbackrest/  
\-rw-r--r-- root/root       220 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/pgbackrest/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_cluster\_manager/  
\-rw-r--r-- root/root       561 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db1/alloydbomni\_cluster\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       545 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/haproxy/  
\-rw-r--r-- root/root       163 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/haproxy/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/keepalived/  
\-rw-r--r-- root/root       186 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/keepalived/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/pgbouncer/  
\-rw-r--r-- root/root       192 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy1/pgbouncer/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni17/  
\-rw-r--r-- root/root       272 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni17/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_monitor/  
\-rw-r--r-- root/root       195 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_monitor/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       542 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/etcd/  
\-rw-r--r-- root/root       385 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/etcd/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/pgbackrest/  
\-rw-r--r-- root/root       220 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/pgbackrest/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_cluster\_manager/  
\-rw-r--r-- root/root       561 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db2/alloydbomni\_cluster\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni17/  
\-rw-r--r-- root/root       272 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni17/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_monitor/  
\-rw-r--r-- root/root       195 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_monitor/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       542 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/etcd/  
\-rw-r--r-- root/root       385 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/etcd/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/pgbackrest/  
\-rw-r--r-- root/root       220 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/pgbackrest/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_cluster\_manager/  
\-rw-r--r-- root/root       561 2026-01-20 14:23 alloydbomni/satyasais-debug-role-db3/alloydbomni\_cluster\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/alloydbomni\_node\_manager/  
\-rw-r--r-- root/root       545 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/alloydbomni\_node\_manager/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/haproxy/  
\-rw-r--r-- root/root       163 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/haproxy/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/keepalived/  
\-rw-r--r-- root/root       186 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/keepalived/systemd-status.out  
drwxr-xr-x root/root         0 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/pgbouncer/  
\-rw-r--r-- root/root       192 2026-01-20 14:23 alloydbomni/satyasais-debug-role-haproxy2/pgbouncer/systemd-status.out

```sh
#!/bin/bash
# Copyright 2026 Google LLC
# This product is licensed under the terms of service found at https://cloud.google.com/terms/alloydb-omni-preview-tos

set -euo pipefail

# Default Values
SSH_KEY=""
OUT_DIR=""
TAG=""
SPEC_FILE=""
COLLECTION_TYPE="status"
REMOTE_TMP="/tmp"
SERVICE_ACCOUNT=""
BINARY_PATH="/usr/local/bin"

function usage {
    echo "Usage: $0 -k <ssh_key> -u <user> -d <deployment_spec.yaml> -o <dir> [-t <tag>] [-c <collection-type>]"
    exit 1
}

while [[ $# -gt 0 ]]; do
    case $1 in
        -k) SSH_KEY="$2"; shift 2 ;;
        -u) SERVICE_ACCOUNT="$2"; shift 2 ;;
        -d) SPEC_FILE="$2"; shift 2 ;;
        -o) OUT_DIR="$2"; shift 2 ;;
        -t) TAG="$2"; shift 2 ;;
        -c) COLLECTION_TYPE="$2"; shift 2 ;;
        *) echo "error: Unknown option: $1"; usage ;;
    esac
done

if [[ -z "$SSH_KEY" || -z "$SERVICE_ACCOUNT" || -z "$SPEC_FILE" || -z "$OUT_DIR" ]]; then
    usage
fi

# New: Generate default tag if not provided (DDMMYYHH)
if [[ -z "$TAG" ]]; then
    TAG=$(date +"%d%m%y%H")
fi

# New: Suffixing logic for DDMMYYHH_<n>
BASE_TAG=$TAG
n=1
while [[ -d "${OUT_DIR}/staging_${TAG}" ]]; do
    TAG="${BASE_TAG}_${n}"
    ((n++))
done

if ! command -v yq &> /dev/null; then
    echo "error: 'yq' is not installed."
    exit 1
fi

function get_hosts {
    local path=$1
    yq eval "$path | keys | .[]" "$SPEC_FILE" 2>/dev/null || echo ""
}

PRIMARY_DB_NODES=($(get_hosts ".alloydbomni.children.primary_instance_nodes.hosts"))
LB_NODES=($(get_hosts ".alloydbomni.children.load_balancer_nodes.hosts"))
READPOOL_NODES=($(get_hosts ".alloydbomni.children.readpool_instance_nodes.hosts"))
CM_SPEC_NODES=($(get_hosts ".alloydbomni.children.cluster_manager_nodes.hosts"))

if [[ ${#CM_SPEC_NODES[@]} -gt 0 ]]; then
    CM_NODES=("${CM_SPEC_NODES[@]}")
else
    CM_NODES=("${PRIMARY_DB_NODES[@]}")
fi

NM_NODES=($(printf "%s\n" "${PRIMARY_DB_NODES[@]}" "${READPOOL_NODES[@]}" "${LB_NODES[@]}" | sort -u))
ALL_HOSTS=($(printf "%s\n" "${CM_NODES[@]}" "${NM_NODES[@]}" | sort -u))

LOCAL_STAGING="${OUT_DIR}/staging_${TAG}"
mkdir -p "$LOCAL_STAGING"
# Note: We remove staging but keep the final tarball
trap 'rm -rf "$LOCAL_STAGING"' EXIT

echo "Starting collection with tag: $TAG"

# --- CM & NM Execution ---
for host in "${ALL_HOSTS[@]}"; do
    ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no "$SERVICE_ACCOUNT@$host" \
        "[[ -f $BINARY_PATH/alloydbomni_cm_dump ]] && sudo $BINARY_PATH/alloydbomni_cm_dump -c $COLLECTION_TYPE -o $REMOTE_TMP -t $TAG || true;
         [[ -f $BINARY_PATH/alloydbomni_nm_dump ]] && sudo $BINARY_PATH/alloydbomni_nm_dump -c $COLLECTION_TYPE -o $REMOTE_TMP -t $TAG || true"
done

# --- Fetch and Consolidate ---
for host in "${ALL_HOSTS[@]}"; do
    mkdir -p "$LOCAL_STAGING/$host"
    for suffix in "cm" "nm"; do
        REMOTE_FILE="$REMOTE_TMP/alloydbomni_${suffix}_${TAG}.tgz"
        if scp -i "$SSH_KEY" -o StrictHostKeyChecking=no "$SERVICE_ACCOUNT@$host:$REMOTE_FILE" "$LOCAL_STAGING/$host/" 2>/dev/null; then
            tar -xzf "$LOCAL_STAGING/$host/alloydbomni_${suffix}_${TAG}.tgz" -C "$LOCAL_STAGING"
            ssh -i "$SSH_KEY" "$SERVICE_ACCOUNT@$host" "sudo rm -f $REMOTE_FILE"
        fi
    done
done

FINAL_TARBALL="${OUT_DIR}/alloydbomni_cluster_${TAG}.tar.gz"
if [[ -d "$LOCAL_STAGING/alloydbomni" ]]; then
    tar -czf "$FINAL_TARBALL" -C "$LOCAL_STAGING" alloydbomni
    echo "Debug dump is stored at location: $FINAL_TARBALL"
else
    echo "error: No data collected."
    exit 1
fi

```

Command:   
\# Running with default timestamp (auto-suffixing)  
./alloydbomni\_dump.sh \-k \~/ssh-key-cluster-sa \-u sa\_user \-d /tmp/spec.yaml \-o /tmp/debug\_dump

\# Running with specific tag  
./alloydbomni\_dump.sh \-k \~/ssh-key-cluster-sa \-u sa\_user \-d /tmp/spec.yaml \-o /tmp/debug\_dump \-t 20260227

Ansible role:

```
---
- name: Determine Unique Tag and Initialize Environment
  delegate_to: localhost
  run_once: true
  become: false
  block:
    - name: Generate default tag (DDMMYYHH)
      ansible.builtin.set_fact:
        final_tag: "{{ dump_debug_tag if dump_debug_tag | length > 0 else now(fmt='%d%m%y%H') }}"

    - name: Check for existing staging directory
      ansible.builtin.stat:
        path: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}"
      register: staging_check

    - name: Resolve Tag Collision (Suffixing logic)
      block:
        - name: Search for existing suffixed directories
          ansible.builtin.find:
            paths: "{{ dump_debug_local_dest }}"
            patterns: "staging_{{ final_tag }}_*"
            file_type: directory
          register: suffixed_dirs

        - name: Calculate next suffix
          ansible.builtin.set_fact:
            final_tag: "{{ final_tag }}_{{ (suffixed_dirs.matched + 1) }}"
      when: staging_check.stat.exists

- name: Set facts for node groups
  ansible.builtin.set_fact:
    primary_instance_nodes: "{{ groups['primary_instance_nodes'] | default([]) }}"
    readpool_instance_nodes: "{{ groups['readpool_instance_nodes'] | default([]) }}"
    load_balancer_nodes: "{{ groups['load_balancer_nodes'] | default([]) }}"
    all_nodes: "{{ (groups['primary_instance_nodes'] | default([])) + (groups['readpool_instance_nodes'] | default([])) + (groups['load_balancer_nodes'] | default([])) | unique }}"
    cluster_manager_nodes: >-
      {{ groups['cluster_manager_nodes'] if 'cluster_manager_nodes' in groups and (groups['cluster_manager_nodes'] | length > 0)
         else groups['primary_instance_nodes'] if 'primary_instance_nodes' in groups else [] }}
  run_once: true

- name: Initialize Local Directories
  ansible.builtin.file:
    path: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/extracted"
    state: directory
    mode: '0755'
  become: false
  delegate_to: localhost
  run_once: true

- name: Execute Remote Binaries
  block:
    - name: Run CM dump
      ansible.builtin.command:
        cmd: "{{ dump_debug_binary_path }}/alloydbomni_cm_dump -c {{ dump_debug_collection_type }} -o {{ dump_debug_remote_staging }} -t {{ final_tag }}"
      become: true
      when: inventory_hostname in cluster_manager_nodes
      failed_when: false

    - name: Run NM dump
      ansible.builtin.command:
        cmd: "{{ dump_debug_binary_path }}/alloydbomni_nm_dump -c {{ dump_debug_collection_type }} -o {{ dump_debug_remote_staging }} -t {{ final_tag }}"
      become: true
      when: inventory_hostname in all_nodes
      failed_when: false

- name: Transfer and Consolidate
  block:
    - name: Fetch tarballs
      ansible.builtin.fetch:
        src: "{{ dump_debug_remote_staging }}/alloydbomni_{{ item }}_{{ final_tag }}.tgz"
        dest: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/"
        flat: no
      loop: ["cm", "nm"]
      ignore_errors: true

    - name: Extract and Archive
      delegate_to: localhost
      run_once: true
      become: false
      block:
        - name: Find fetched files
          ansible.builtin.find:
            paths: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/"
            recurse: yes
            patterns: "*.tgz"
          register: fetched_tars

        - name: Extract
          ansible.builtin.unarchive:
            src: "{{ item.path }}"
            dest: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/extracted"
            remote_src: no
          loop: "{{ fetched_tars.files }}"
          when: fetched_tars.matched > 0

        - name: Create final master tarball
          ansible.builtin.shell:
            cmd: "tar -czf {{ dump_debug_local_dest }}/alloydbomni_cluster_{{ final_tag }}.tar.gz alloydbomni"
            chdir: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}/extracted"
          when: fetched_tars.matched > 0

    - name: Clean up Staging
      ansible.builtin.file:
        path: "{{ dump_debug_local_dest }}/staging_{{ final_tag }}"
        state: absent
      delegate_to: localhost
      run_once: true
      become: false

- name: Cleanup Remote Artifacts
  ansible.builtin.file:
    path: "{{ dump_debug_remote_staging }}/alloydbomni_{{ item }}_{{ final_tag }}.tgz"
    state: absent
  loop: ["cm", "nm"]
  become: true
  ignore_errors: true

```

```
---
dump_debug_binary_path: "/usr/local/bin"
dump_debug_remote_staging: "/tmp"
dump_debug_local_dest: "/tmp/debug_dump"
# If empty, role will generate DDMMYYHH with suffix handling
dump_debug_tag: ""
dump_debug_collection_type: "status"
```

```
# Running with default timestamp (auto-suffixing)
ansible-playbook -i /tmp/deployment_spec.yaml /tmp/dump_debug.yaml

# Running with manual tag
ansible-playbook -i /tmp/deployment_spec.yaml /tmp/dump_debug.yaml -e "dump_debug_tag=20260227"

```

## **TRAIL 3: Without tag**

Creating a custom collection with debug\_dump role:

\$ g4d \-f dump\_config\_test\_1

cd storage/alloydb/nova/test

\# While creating cluster make the following lifecycle  change in service\_account in this file [google3/storage/alloydb/nova/test/gce/terraform/db.tf](http://google3/storage/alloydb/nova/test/gce/terraform/db.tf) `add the lifecycle part below service one`

```
service_account {
email = google_service_account.admin.email
scopes = ["cloud-platform"]
}

lifecycle {
ignore_changes = [attached_disk]
}
```

`orchestrator/nova.sh create cluster="satyasais-dc1" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="ansible"`

`orchestrator/nova.sh init cluster="satyasais-dc1" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="ansible"`

`orchestrator/nova.sh test cluster="satyasais-dc1" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="ansible"`

`orchestrator/nova.sh sync cluster="satyasais-dc1" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="ansible"`

`orchestrator/nova.sh destroy cluster="satyasais-dc1" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="ansible"`

```
control_zone = "us-central1-c"
environment_name = "satyasais-dc1"
project_id = "alloydb-nova-tvc-sandbox"


$ popd
/google/src/cloud/satyasais/dump_config_test_1/google3/storage/alloydb/nova/test


$ gcloud compute instances list --project=alloydb-nova-tvc-sandbox --filter="name~satyasais-dc1-"
NAME                    ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
satyasais-dc1-db3       us-central1-a  n2-highmem-16               10.1.0.3                  RUNNING
satyasais-dc1-db2       us-central1-b  n2-highmem-16               10.1.0.2                  RUNNING
satyasais-dc1-control   us-central1-c  n2-standard-2               10.1.0.7                  RUNNING
satyasais-dc1-db1       us-central1-c  n2-highmem-16               10.1.0.5                  RUNNING
satyasais-dc1-haproxy1  us-central1-c  n2-standard-2               10.1.0.4                  RUNNING
satyasais-dc1-haproxy2  us-central1-c  n2-standard-2               10.1.0.6                  RUNNING


$ gcloud storage buckets describe gs://satyasais-dc1-gcs-backups --project=alloydb-nova-tvc-sandbox
creation_time: 2026-04-13T05:17:26+0000
default_storage_class: REGIONAL
generation: 1776057446368767720
labels:
  goog-terraform-provisioned: 'true'
location: US-CENTRAL1
location_type: region
metageneration: 1
name: satyasais-dc1-gcs-backups
public_access_prevention: inherited
soft_delete_policy:
  effectiveTime: '2026-04-13T05:17:26.501000+00:00'
  retentionDurationSeconds: '604800'
storage_url: gs://satyasais-dc1-gcs-backups/
uniform_bucket_level_access: true
update_time: 2026-04-13T05:17:26+0000

Run the following command to log into the control VM
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dc1-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

Note: Find cluster setup scripts in /tmp/ directory in your control VM
```

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dc1-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dc1-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dc1-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dc1-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dc1-haproxy2.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

Inside Control node:  
\$ /tmp/[setup-ssh-for-cluster.sh](http://setup-ssh-for-cluster.sh)

**On all nodes**

`sudo setenforce 0`  
`sudo getenforce             ## Validate` 

`/tmp/run-all.sh "sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config"`  
`/tmp/run-all.sh "sudo setenforce 0"`  
`/tmp/run-all.sh "sudo getenforce"`

**`From nova script: from control node`**  
`orchestrator/alloydb-ansible.sh debug_dump -c ./cluster.conf`

`gsutil -m cp /tmp/debug_dump_logs/alloydbomni_cluster_satyasais-bkda2.tar.gz gs://nova_debug_dumps/backup_issue/`

ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml \-e `dump_debug_collection_type=config`

`sudo vi /usr/local/bin/alloydbomni_nm_dump`

tar \-tvf /tmp/debug\_dump\_logs/alloydbomni\_cluster\_satyasais-dc1.tar.gz

tar \-xOzf /tmp/debug\_dump\_test/alloydbomni\_cluster\_etcd\_config\_forcewrite\_issue\_2.tar.gz alloydbomni/satyasais-def4-db1/alloydbomni\_cluster\_manager/journalctl.out

tar \-xOzf /tmp/debug\_dump\_test/alloydbomni\_cluster\_etcd\_config\_forcewrite\_issue\_2.tar.gz alloydbomni/satyasais-def4-db1/etcd/journalctl.out

tar \-xOzf /tmp/debug\_dump\_test/alloydbomni\_cluster\_etcd\_config\_forcewrite\_issue\_2.tar.gz alloydbomni/satyasais-def4-db1/alloydbomni\_cluster\_manager/systemd-status.out

```
sudo tee /usr/local/bin/alloydbomni_nm_dump > /dev/null << 'EOF'
#!/bin/bash

# Copyright 2026 Google LLC
# This product is licensed under the terms of service found at https://cloud.google.com/terms/alloydb-omni-preview-tos

set -euo pipefail

SERVICES=("alloydbomni_node_manager" "alloydbomni_monitor" "haproxy" "keepalived" "pgbouncer" "pgbackrest")
HOSTNAME=$(hostname)
TYPE="all"
OUT_DIR=""
TAG=""
JOURNAL_SPEC="-n 1000"
PG_DATA_DIR=""
NM_ROLES_FILE="/etc/alloydbomni/node_manager/node_manager_roles.toml"
PROG=$(basename "$0")

declare -A CONFIG_MAP
CONFIG_MAP["alloydbomni_node_manager"]="/etc/alloydbomni/node_manager/node_manager.toml"
CONFIG_MAP["haproxy"]="/etc/haproxy/haproxy.cfg"
CONFIG_MAP["keepalived"]="/etc/keepalived/keepalived.conf"
CONFIG_MAP["pgbouncer"]="/etc/pgbouncer/pgbouncer.ini"
CONFIG_MAP["pgbackrest"]="/etc/pgbackrest/conf.d"

function usage {
cat <<EOF
usage: $PROG [-c <collection>] [-s <size spec>] -o <output dir> -t <dump tag>
where:
  c - Optional. collection type i.e. 'status','config','logs','all'. Default 'all'
  o - Required. output directory for the debug contents.
  t - Required. tag to be used for the debug contents.
  s - Optional. journalctl size spec. Default "-n 1000".
                other options: "--since 2026-03-01", "--since \"2 hours ago\""
EOF
    exit 1
}

function cleanup {
    rm -rf "$OUT_DIR/alloydbomni/$HOSTNAME"
}

## main
ID=$(id -u)
if [[ "$ID" -ne 0 ]]; then
    echo "alloydbomni_nm_dump: error: please run this script with root privileges."
    exit 1
fi

while [[ $# -gt 0 ]]; do
    case $1 in
        -c) TYPE="$2"; shift 2 ;;
        -s) JOURNAL_SPEC="$2"; shift 2 ;;
        -o) OUT_DIR="$2"; shift 2 ;;
        -t) TAG="$2"; shift 2 ;;
        *) echo "error: Unknown option: $1"; usage ;;
    esac
done

if [[ -z "$OUT_DIR" || -z "$TAG" ]]; then
    echo "error: insufficient parameters."
    usage
fi

if [[ -f "$NM_ROLES_FILE" ]]; then
    DETECTED_PATH=$(grep "data_mount_path" "$NM_ROLES_FILE" | sed -E "s/.*data_mount_path *= *['\"]([^'\"]+)['\"].*/\1/" || true)
    if [[ -n "$DETECTED_PATH" ]]; then
        PG_DATA_DIR="$DETECTED_PATH"
    fi
fi

mkdir -p "$OUT_DIR"
trap "cleanup" EXIT

mapfile -t DB_SERVICES < <(systemctl list-unit-files --type=service --all | grep -oE '^alloydbomni[0-9]+' || true)
ALL_COMPONENTS=($(printf "%s\n" "${SERVICES[@]}" "${DB_SERVICES[@]}" | sort -u))

for svc in "${ALL_COMPONENTS[@]}"; do
    COLLECT_ROOT="$OUT_DIR/alloydbomni/$HOSTNAME/$svc"
    mkdir -p "$COLLECT_ROOT"
    if systemctl list-unit-files "$svc.service" >/dev/null 2>&1; then

        # Collect systemd status
        if [[ "$TYPE" == "status" || "$TYPE" == "all" ]]; then
            echo "Collecting systemd status for $svc..."
            systemctl status "$svc" -n 0 --no-pager > "$COLLECT_ROOT/systemd-status.out" 2>&1 || true
        fi

        # Collect journalctl logs
        if [[ "$TYPE" == "logs" || "$TYPE" == "all" ]]; then
            echo "Collecting journalctl logs for $svc..."
            journal_args=("-u" "$svc")
            eval "journal_args+=($JOURNAL_SPEC)"
            journal_args+=("--no-pager")
            journalctl "${journal_args[@]}" > "$COLLECT_ROOT/journalctl.out" 2>&1 || true
        fi

        # Collect Configuration Files
        if [[ "$TYPE" == "config" || "$TYPE" == "all" ]]; then
            CONF="${CONFIG_MAP[$svc]:-}"

            if [[ -z "$CONF" && "$svc" =~ ^alloydbomni[0-9]+$ ]]; then
                if [[ -n "$PG_DATA_DIR" ]]; then
                    V_NUM=$(echo "$svc" | grep -oE '[0-9]+')
                    CONF="${PG_DATA_DIR}/${V_NUM}/db/postgresql.conf"
                fi
            fi

            if [[ -n "$CONF" ]]; then
                if sudo test -e "$CONF"; then
                    echo "Collecting config for $svc from $CONF..."
                    sudo cp -rp "$CONF" "$COLLECT_ROOT/"
                    sudo chown -R "$(id -u):$(id -g)" "$COLLECT_ROOT"
                else
                    echo "warning: Config file $CONF not found for $svc"
                fi
            fi
        fi
    else
        echo "warning: Service $svc not found on this system. Skipping..."
    fi
done

TARBALL="${OUT_DIR}/alloydbomni_nm_${TAG}.tgz"

tar -czf "$TARBALL" -C "$OUT_DIR" "alloydbomni/$HOSTNAME"
echo "alloydbomni_nm_dump: $(date '+%Y-%m-%d %H:%M:%S'): Dump created at location $TARBALL"
EOF
```

cm\_dump

```
sudo tee /usr/local/bin/alloydbomni_cm_dump > /dev/null << 'EOF'
#!/bin/bash

# Copyright 2026 Google LLC
# This product is licensed under the terms of service found at https://cloud.google.com/terms/alloydb-omni-preview-tos

set -euo pipefail

SERVICES=("alloydbomni_cluster_manager" "etcd")
HOSTNAME=$(hostname)
TYPE="all"
OUT_DIR=""
TAG=""
PROG=$(basename "$0")
JOURNAL_SPEC="-n 1000"

declare -A CONFIG_MAP
CONFIG_MAP["alloydbomni_cluster_manager"]="/etc/alloydbomni/cluster_manager/cluster_manager.toml"
CONFIG_MAP["etcd"]="/etc/etcd/etcd.conf"

function usage {
cat <<EOF
usage: $PROG [-c <collection>] [-s <size spec>] -o <output dir> -t <dump tag>
where:
  c - optional. collection type i.e. 'status','config','journal','all'. Default 'all'
  o - required. output directory for the debug contents.
  t - required. tag to be used for the debug contents.
  s - Optional. journalctl size spec. Default "-n 1000", i.e. 1000 lines.
                other options: "--since 2026-03-01", "--since \"2 hours ago\""
EOF
    exit 1
}

function cleanup {
    rm -rf "$OUT_DIR/alloydbomni/$HOSTNAME"
}

## main
ID=$(id -u)
if [[ "$ID" -ne 0 ]]; then
    echo "alloydbomni_cm_dump: error: please run this script with root privileges."
    exit 1
fi

while [[ $# -gt 0 ]]; do
    case $1 in
        -c) TYPE="$2"; shift 2 ;;
        -s) JOURNAL_SPEC="$2"; shift 2 ;;
        -o) OUT_DIR="$2"; shift 2 ;;
        -t) TAG="$2"; shift 2 ;;
        *) echo "error: Unknown option: $1"; usage ;;
    esac
done

if [[ -z "$OUT_DIR" || -z "$TAG" ]]; then
    echo "error: insufficient parameters."
    usage
fi

mkdir -p "$OUT_DIR"
trap "cleanup" EXIT
for svc in "${SERVICES[@]}"; do
    COLLECT_ROOT="$OUT_DIR/alloydbomni/$HOSTNAME/$svc"
    mkdir -p "$COLLECT_ROOT"

    # Collect systemd status
    if [[ "$TYPE" == "status" || "$TYPE" == "all" ]]; then
        echo "Collecting systemd status for $svc..."
        systemctl status "$svc" -n 0 --no-pager > "$COLLECT_ROOT/systemd-status.out" 2>&1 || true
    fi

    # Collect journalctl logs
    if [[ "$TYPE" == "logs" || "$TYPE" == "all" ]]; then
        echo "Collecting journalctl logs for $svc..."
        journal_args=("-u" "$svc")
        eval "journal_args+=($JOURNAL_SPEC)"
        journal_args+=("--no-pager")
        journalctl "${journal_args[@]}" > "$COLLECT_ROOT/journalctl.out" 2>&1 || true
    fi

    # Collect config files
    if [[ "$TYPE" == "config" || "$TYPE" == "all" ]]; then
        CONF="${CONFIG_MAP[$svc]:-}"
        if [[ -n "$CONF" && -f "$CONF" ]]; then
            echo "Collecting config for $svc from $CONF..."
            cp "$CONF" "$COLLECT_ROOT/"
        fi
    fi
done

TARBALL="${OUT_DIR}/alloydbomni_cm_${TAG}.tgz"

tar -czf "$TARBALL" -C "$OUT_DIR" "alloydbomni/$HOSTNAME"
echo "alloydbomni_cm_dump: $(date '+%Y-%m-%d %H:%M:%S'): Dump created at location $TARBALL"
EOF

```

## **TRAIL 3: Without tag**

Creating a custom collection with debug\_dump role:

\$ g4d \-f dump\_config\_cmctl\_1

cd storage/alloydb/nova/test

\# While creating cluster make the following lifecycle  change in service\_account in this file [google3/storage/alloydb/nova/test/gce/terraform/db.tf](http://google3/storage/alloydb/nova/test/gce/terraform/db.tf) `add the lifecycle part below service one`

```
service_account {
email = google_service_account.admin.email
scopes = ["cloud-platform"]
}

lifecycle {
ignore_changes = [attached_disk]
}
```

`orchestrator/nova.sh create cluster="satyasais-dcc2" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="alloydbctl"`

`orchestrator/nova.sh init cluster="satyasais-dcc2" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="alloydbctl"`

`orchestrator/nova.sh test cluster="satyasais-dcc2" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="alloydbctl"`

`orchestrator/nova.sh sync cluster="satyasais-dcc2" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="alloydbctl"`

`orchestrator/nova.sh destroy cluster="satyasais-dcc2" nodes="db:3,haproxy:2,control" project="alloydb-nova-tvc-sandbox" client="alloydbctl"`

```
control_zone = "us-central1-c"
environment_name = "satyasais-dcc2"
project_id = "alloydb-nova-tvc-sandbox"


$ popd
/google/src/cloud/satyasais/dump_config_cmctl_1/google3/storage/alloydb/nova/test


$ gcloud compute instances list --project=alloydb-nova-tvc-sandbox --filter="name~satyasais-dcc2-"
NAME                     ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
satyasais-dcc2-db3       us-central1-a  n2-highmem-16               10.1.0.2                  RUNNING
satyasais-dcc2-db2       us-central1-b  n2-highmem-16               10.1.0.5                  RUNNING
satyasais-dcc2-control   us-central1-c  n2-standard-2               10.1.0.7                  RUNNING
satyasais-dcc2-db1       us-central1-c  n2-highmem-16               10.1.0.4                  RUNNING
satyasais-dcc2-haproxy1  us-central1-c  n2-standard-2               10.1.0.6                  RUNNING
satyasais-dcc2-haproxy2  us-central1-c  n2-standard-2               10.1.0.3                  RUNNING


$ gcloud storage buckets describe gs://satyasais-dcc2-gcs-backups --project=alloydb-nova-tvc-sandbox
creation_time: 2026-04-13T09:03:01+0000
default_storage_class: REGIONAL
generation: 1776070981772765119
labels:
  goog-terraform-provisioned: 'true'
location: US-CENTRAL1
location_type: region
metageneration: 1
name: satyasais-dcc2-gcs-backups
public_access_prevention: inherited
soft_delete_policy:
  effectiveTime: '2026-04-13T09:03:01.895000+00:00'
  retentionDurationSeconds: '604800'
storage_url: gs://satyasais-dcc2-gcs-backups/
uniform_bucket_level_access: true
update_time: 2026-04-13T09:03:01+0000

Run the following command to log into the control VM
  $ ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dcc2-control.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com

Note: Find cluster setup scripts in /tmp/ directory in your control VM
```

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dcc2-db1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dcc2-db2.us-central1-b.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dcc2-db3.us-central1-a.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dcc2-haproxy1.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

`ssh -o StrictHostKeyChecking=no   satyasais_google_com@nic0.satyasais-dcc2-haproxy2.us-central1-c.c.alloydb-nova-tvc-sandbox.internal.gcpnode.com`

Inside Control node:  
\$ /tmp/[setup-ssh-for-cluster.sh](http://setup-ssh-for-cluster.sh)

**On all nodes**

`sudo setenforce 0`  
`sudo getenforce             ## Validate` 

`/tmp/run-all.sh "sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config"`  
`/tmp/run-all.sh "sudo setenforce 0"`  
`/tmp/run-all.sh "sudo getenforce"`

**`From nova script: from control node`**  
`orchestrator/alloydb-ansible.sh debug_dump -c ./cluster.conf`

`gsutil -m cp /tmp/debug_dump_logs/alloydbomni_cluster_satyasais-bkda2.tar.gz gs://nova_debug_dumps/backup_issue/`

ansible-playbook \-i /tmp/deployment\_spec.yaml /tmp/dump\_debug.yaml \-e `dump_debug_collection_type=config`

`sudo vi /usr/local/bin/alloydbomni_nm_dump`

`sudo vi /usr/local/bin/alloydbomni_cm_dump`

tar \-tvf /tmp/debug\_dump\_logs/alloydbomni\_cluster\_satyasais-dc1.tar.gz

tar \-xOzf /tmp/debug\_dump\_test/alloydbomni\_cluster\_etcd\_config\_forcewrite\_issue\_2.tar.gz alloydbomni/satyasais-def4-db1/alloydbomni\_cluster\_manager/journalctl.out

tar \-xOzf /tmp/debug\_dump\_test/alloydbomni\_cluster\_etcd\_config\_forcewrite\_issue\_2.tar.gz alloydbomni/satyasais-def4-db1/etcd/journalctl.out

tar \-xOzf /tmp/debug\_dump\_test/alloydbomni\_cluster\_etcd\_config\_forcewrite\_issue\_2.tar.gz alloydbomni/satyasais-def4-db1/alloydbomni\_cluster\_manager/systemd-status.out

```
sudo tee /usr/local/bin/alloydbomni_nm_dump > /dev/null << 'EOF'
#!/bin/bash

# Copyright 2026 Google LLC
# This product is licensed under the terms of service found at https://cloud.google.com/terms/alloydb-omni-preview-tos

set -euo pipefail

SERVICES=("alloydbomni_node_manager" "alloydbomni_monitor" "haproxy" "keepalived" "pgbouncer" "pgbackrest")
HOSTNAME=$(hostname)
TYPE="all"
OUT_DIR=""
TAG=""
JOURNAL_SPEC="-n 1000"
PG_DATA_DIR=""
NM_ROLES_FILE="/etc/alloydbomni/node_manager/node_manager_roles.toml"
PROG=$(basename "$0")

declare -A CONFIG_MAP
CONFIG_MAP["alloydbomni_node_manager"]="/etc/alloydbomni/node_manager/node_manager.toml"
CONFIG_MAP["haproxy"]="/etc/haproxy/haproxy.cfg"
CONFIG_MAP["keepalived"]="/etc/keepalived/keepalived.conf"
CONFIG_MAP["pgbouncer"]="/etc/pgbouncer/pgbouncer.ini"
CONFIG_MAP["pgbackrest"]="/etc/pgbackrest/conf.d"

function usage {
cat <<EOF
usage: $PROG [-c <collection>] [-s <size spec>] -o <output dir> -t <dump tag>
where:
  c - Optional. collection type i.e. 'status','config','logs','all'. Default 'all'
  o - Required. output directory for the debug contents.
  t - Required. tag to be used for the debug contents.
  s - Optional. journalctl size spec. Default "-n 1000".
                other options: "--since 2026-03-01", "--since \"2 hours ago\""
EOF
    exit 1
}

function cleanup {
    rm -rf "$OUT_DIR/alloydbomni/$HOSTNAME"
}

## main
ID=$(id -u)
if [[ "$ID" -ne 0 ]]; then
    echo "alloydbomni_nm_dump: error: please run this script with root privileges."
    exit 1
fi

while [[ $# -gt 0 ]]; do
    case $1 in
        -c) TYPE="$2"; shift 2 ;;
        -s) JOURNAL_SPEC="$2"; shift 2 ;;
        -o) OUT_DIR="$2"; shift 2 ;;
        -t) TAG="$2"; shift 2 ;;
        *) echo "error: Unknown option: $1"; usage ;;
    esac
done

if [[ -z "$OUT_DIR" || -z "$TAG" ]]; then
    echo "error: insufficient parameters."
    usage
fi

if [[ -f "$NM_ROLES_FILE" ]]; then
    DETECTED_PATH=$(grep "data_mount_path" "$NM_ROLES_FILE" | sed -E "s/.*data_mount_path *= *['\"]([^'\"]+)['\"].*/\1/" || true)
    if [[ -n "$DETECTED_PATH" ]]; then
        PG_DATA_DIR="$DETECTED_PATH"
    fi
fi

mkdir -p "$OUT_DIR"
trap "cleanup" EXIT

mapfile -t DB_SERVICES < <(systemctl list-unit-files --type=service --all | grep -oE '^alloydbomni[0-9]+' || true)
ALL_COMPONENTS=($(printf "%s\n" "${SERVICES[@]}" "${DB_SERVICES[@]}" | sort -u))

for svc in "${ALL_COMPONENTS[@]}"; do
    COLLECT_ROOT="$OUT_DIR/alloydbomni/$HOSTNAME/$svc"
    mkdir -p "$COLLECT_ROOT"
    if systemctl list-unit-files "$svc.service" >/dev/null 2>&1; then

        # Collect systemd status
        if [[ "$TYPE" == "status" || "$TYPE" == "all" ]]; then
            echo "Collecting systemd status for $svc..."
            systemctl status "$svc" -n 0 --no-pager > "$COLLECT_ROOT/systemd-status.out" 2>&1 || true
        fi

        # Collect journalctl logs
        if [[ "$TYPE" == "logs" || "$TYPE" == "all" ]]; then
            echo "Collecting journalctl logs for $svc..."
            journal_args=("-u" "$svc")
            eval "journal_args+=($JOURNAL_SPEC)"
            journal_args+=("--no-pager")
            journalctl "${journal_args[@]}" > "$COLLECT_ROOT/journalctl.out" 2>&1 || true
        fi

        # Collect Configuration Files
        if [[ "$TYPE" == "config" || "$TYPE" == "all" ]]; then
            CONF="${CONFIG_MAP[$svc]:-}"

            if [[ -z "$CONF" && "$svc" =~ ^alloydbomni[0-9]+$ ]]; then
                if [[ -n "$PG_DATA_DIR" ]]; then
                    V_NUM=$(echo "$svc" | grep -oE '[0-9]+')
                    CONF="${PG_DATA_DIR}/${V_NUM}/db/postgresql.conf"
                fi
            fi

            if [[ -n "$CONF" ]]; then
                if sudo test -e "$CONF"; then
                    echo "Collecting config for $svc from $CONF..."
                    sudo cp -rp "$CONF" "$COLLECT_ROOT/"
                    sudo chown -R "$(id -u):$(id -g)" "$COLLECT_ROOT"
                else
                    echo "warning: Config file $CONF not found for $svc"
                fi
            fi
        fi
    else
        echo "warning: Service $svc not found on this system. Skipping..."
    fi
done

TARBALL="${OUT_DIR}/alloydbomni_nm_${TAG}.tgz"

tar -czf "$TARBALL" -C "$OUT_DIR" "alloydbomni/$HOSTNAME"
echo "alloydbomni_nm_dump: $(date '+%Y-%m-%d %H:%M:%S'): Dump created at location $TARBALL"
EOF
```

cm\_dump

```
sudo tee /usr/local/bin/alloydbomni_cm_dump > /dev/null << 'EOF'
#!/bin/bash

# Copyright 2026 Google LLC
# This product is licensed under the terms of service found at https://cloud.google.com/terms/alloydb-omni-preview-tos

set -euo pipefail

SERVICES=("alloydbomni_cluster_manager" "etcd")
HOSTNAME=$(hostname)
TYPE="all"
OUT_DIR=""
TAG=""
PROG=$(basename "$0")
JOURNAL_SPEC="-n 1000"

declare -A CONFIG_MAP
CONFIG_MAP["alloydbomni_cluster_manager"]="/etc/alloydbomni/cluster_manager/cluster_manager.toml"
CONFIG_MAP["etcd"]="/etc/etcd/etcd.conf"

function usage {
cat <<EOF
usage: $PROG [-c <collection>] [-s <size spec>] -o <output dir> -t <dump tag>
where:
  c - optional. collection type i.e. 'status','config','journal','all'. Default 'all'
  o - required. output directory for the debug contents.
  t - required. tag to be used for the debug contents.
  s - Optional. journalctl size spec. Default "-n 1000", i.e. 1000 lines.
                other options: "--since 2026-03-01", "--since \"2 hours ago\""
EOF
    exit 1
}

function cleanup {
    rm -rf "$OUT_DIR/alloydbomni/$HOSTNAME"
}

## main
ID=$(id -u)
if [[ "$ID" -ne 0 ]]; then
    echo "alloydbomni_cm_dump: error: please run this script with root privileges."
    exit 1
fi

while [[ $# -gt 0 ]]; do
    case $1 in
        -c) TYPE="$2"; shift 2 ;;
        -s) JOURNAL_SPEC="$2"; shift 2 ;;
        -o) OUT_DIR="$2"; shift 2 ;;
        -t) TAG="$2"; shift 2 ;;
        *) echo "error: Unknown option: $1"; usage ;;
    esac
done

if [[ -z "$OUT_DIR" || -z "$TAG" ]]; then
    echo "error: insufficient parameters."
    usage
fi

mkdir -p "$OUT_DIR"
trap "cleanup" EXIT
for svc in "${SERVICES[@]}"; do
    COLLECT_ROOT="$OUT_DIR/alloydbomni/$HOSTNAME/$svc"
    mkdir -p "$COLLECT_ROOT"

    # Collect systemd status
    if [[ "$TYPE" == "status" || "$TYPE" == "all" ]]; then
        echo "Collecting systemd status for $svc..."
        systemctl status "$svc" -n 0 --no-pager > "$COLLECT_ROOT/systemd-status.out" 2>&1 || true
    fi

    # Collect journalctl logs
    if [[ "$TYPE" == "logs" || "$TYPE" == "all" ]]; then
        echo "Collecting journalctl logs for $svc..."
        journal_args=("-u" "$svc")
        eval "journal_args+=($JOURNAL_SPEC)"
        journal_args+=("--no-pager")
        journalctl "${journal_args[@]}" > "$COLLECT_ROOT/journalctl.out" 2>&1 || true
    fi

    # Collect config files
    if [[ "$TYPE" == "config" || "$TYPE" == "all" ]]; then
        CONF="${CONFIG_MAP[$svc]:-}"
        if [[ -n "$CONF" && -f "$CONF" ]]; then
            echo "Collecting config for $svc from $CONF..."
            cp "$CONF" "$COLLECT_ROOT/"
        fi
    fi
done

TARBALL="${OUT_DIR}/alloydbomni_cm_${TAG}.tgz"

tar -czf "$TARBALL" -C "$OUT_DIR" "alloydbomni/$HOSTNAME"
echo "alloydbomni_cm_dump: $(date '+%Y-%m-%d %H:%M:%S'): Dump created at location $TARBALL"
EOF

```

