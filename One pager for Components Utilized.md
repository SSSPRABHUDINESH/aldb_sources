# **Packages Utilized in various Architectures**

## 

| Architecture | Host Group | Required Packages | Optional Packages |
| ----- | ----- | ----- | ----- |
| **Standalone** | primary\_instance\_nodes | alloydbomni, alloydbomni\_cluster\_manager, alloydbomni\_node\_manager, pgbouncer | Pgbackrest, alloydbomni\_monitor |
|  | backup\_repo\_nodes | alloydbomni\_node\_manager | Pgbackrest |
|  | readpool\_instance\_nodes | alloydbomni, alloydbomni\_node\_manager | alloydbomni\_monitor |
| **Resilient HA** | primary\_instance\_nodes | alloydbomni, etcd, alloydbomni\_cluster\_manager, alloydbomni\_node\_manager, pgbouncer, keepalived | Pgbackrest, alloydbomni\_monitor |
|  | backup\_repo\_nodes | alloydbomni\_node\_manager | Pgbackrest |
|  | readpool\_instance\_nodes | alloydbomni, alloydbomni\_node\_manager | alloydbomni\_monitor |
| **Scalable HA** | primary\_instance\_nodes | alloydbomni, etcd, alloydbomni\_cluster\_manager, alloydbomni\_node\_manager | Pgbackrest, alloydbomni\_monitor |
|  | backup\_repo\_nodes | alloydbomni\_node\_manager | Pgbackrest |
|  | load\_balancer\_nodes | alloydbomni\_node\_manager, haproxy, pgbouncer, keepalived | N/A |
|  | readpool\_instance\_nodes | alloydbomni, alloydbomni\_node\_manager | alloydbomni\_monitor |

## 