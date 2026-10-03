## **Components for various architectures**

## 

## **Standalone Architecture:**

No of VM’s: 1 \- DB, 1- External VM  
Reference Documentation: [End to End Standalone DB Setup Guide](https://docs.google.com/document/d/1jCh9Dg7sWa9LSl6k1J_muCyg4ZjZWXdrxe2vZWnUSVE/edit?resourcekey=0-gbAxu_NWw7F9wZxlEiNlnQ&tab=t.0)

### Inside DB:

1. AlloyDB RPM  
2. AlloyDB monitor  
3. Pg backrest  
4. Pgbouncer

### Inside External VM:

1. Prometheus (version 2.53.0)  
2. Logstash (version 7.17.25)  
3. Elastic search (version 7.17.25)  
4. Pgbench  
5. Pgbackrest

## **Resilient Architecture:**

No of VM’s: 3-DB's, 1- External VM  
Reference Documentation: [End to End Resilient DB Setup Guide](https://docs.google.com/document/d/1EuAAXQXzXF7Avs7re-bSKipNHRAZoHgp7elDClweBE8/edit?tab=t.0)

### Inside DB’s:

1. AlloyDB RPM  
2. AlloyDB monitor  
3. Patroni  
4. ETCD  
5. Keepalived  
6. Pg backrest  
7. Pgbouncer

### Inside External VM:

1. Prometheus (version 2.53.0)  
2. Logstash (version 7.17.25)  
3. Elastic search (version 7.17.25)  
4. Pgbackrest  
5. pgbench

## **Scalable Architecture:**

No of VM’s:  3-DB's, 1-External VM, 2- HAproxy VM’s(HAproxy 1, HAproxy 2\)  
Reference Documentation: [End to End Scalable DB Setup Guide](https://docs.google.com/document/d/1JUOGlltnLdvlQK2A28pTwgN5wvCpbPu8VjEZ-K3HLrQ/edit?tab=t.0)

### Inside DB’s:

1. AlloyDB RPM  
2. AlloyDB monitor  
3. Patroni  
4. ETCD  
5. Pgbouncer  
6. Pgbackrest

### Inside HAproxy node’s:

1. Keepalived  
2. HAproxy

### Inside External VM:

1. Prometheus (version 2.53.0)  
2. Logstash (version 7.17.25)  
3. Elastic search (version 7.17.25)  
4. Pgbackrest  
5. pgbench

