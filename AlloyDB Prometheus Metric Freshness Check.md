# **AlloyDB Prometheus Metric Freshness Check**

## **1\. Objective**

The goal of this task is to manually verify that AlloyDB metrics generated at the database are being scraped by Prometheus in a timely manner over a continuous period of time. This check allows us to evaluate time contributed by AlloyDB Omni monitor to overall metric freshness

## **2\. Metric Freshness Verification**

### **A. Goal of the Verification**

The goal is to manually measure and report the duration it takes for Prometheus to observe a database event. The database event would be triggered manually e.g. a SQL command, AlloyDB Omni monitor observes it through a metric and Prometheus ingests it by scraping the monitor.

```
metric_delay = T_event - T_observation
T_event = timestamp when an event happened on the database
T_observation = timestamp when Prometheus received the event
```

### **B. Experimental Plan & Step-by-Step Instructions**

This experiment involves two main parts: performing an action on the database server and running the script on your machine to observe the outcome.

**Step 1: On the Database Server Host**

You will need SSH access to the server where the AlloyDB Omni monitor is running.

1. **Check Service Status:** First, ensure the service is running.

```
systemctl status alloydbomni-16
blaze test -c opt //storage/testing/lusti/tests/list_rpms:lusti_list_standalone_rpms_test_integration_e2e_env_in_a_cluster_no_env_deps \
  --notest_loasd \
  --test_strategy=local

```

2. **Prepare the Command:** You will stop the service and immediately get the timestamp. The best way to do this is with a combined command. Have this command ready to run:

```
sudo systemctl stop alloydbomni-16 && date +%s
```

This command will stop the service and, upon successful completion, print the current Unix timestamp. This timestamp will be your .

**Step 2: On Your Machine (e.g., standalone-external1)**

1. **Run the Script:** Start the new measurement script provided below. It will first check that the metric `alloydb_omni_database_postgresql_up` is currently `1` (meaning "up").  
2. **Trigger the Event:** When the script prompts you, execute the prepared command from Step 1 on the database server.  
3. **Enter the Timestamp:** Copy the timestamp printed by the command in the previous step and paste it into the script when it asks for the `$T_{\text{event}}$`.  
4. **Observe:** The script will now poll Prometheus until it sees the `alloydb_omni_database_postgresql_up` metric change to `0`. It will then calculate and display the total delay.

**Step 3: Clean Up (On the Database Server Host)**

After the experiment is complete, remember to restart the AlloyDB Omni monitor service.

```
sudo systemctl start alloydbomni-16
```

### **C. Automated Freshness Script:** [https\://paste.googleplex.com/5950632770666496](https://paste.googleplex.com/5950632770666496)

### **D. Trails Output:**  

* ### [https\://screenshot.googleplex.com/BkzD5dLkgNYNxVT](https://screenshot.googleplex.com/BkzD5dLkgNYNxVT)

* ### [https\://screenshot.googleplex.com/G4Ze2kjpxwTeZNL](https://screenshot.googleplex.com/G4Ze2kjpxwTeZNL)

* ### [https\://screenshot.googleplex.com/8sbnYSKi7jprJsn](https://screenshot.googleplex.com/8sbnYSKi7jprJsn)

### **E. Recording and Analyzing Results**

After running the experiment, use this section to document your findings. It is recommended to run the test 3-5 times to get an average delay and account for any variations.

**Results Table:**

| Test Run | Tevent (Unix Timestamp) | Tobservation​ (Unix Timestamp) | Metric Delay (seconds) |
| :---: | :---: | :---: | :---: |
| 1 | 1754037607 | 1754037627 | 20 |
| 2 | 1754038872 | 1754038889 | 17 |
| 3 | 1754038751 | 1754038770 | 19 |
| **Average** |  |  | **18.7** |

**Analysis:**

* **Measured Delay:** The `Metric Delay` measured represents the total time from when the `alloydbomni` service was stopped until Prometheus's next scrape successfully recorded the `alloydb_omni_database_postgresql_up=0` metric.

### **F. Conclusion and Summary**

Based on the experiment, the average measured delay for a state change on the database host to be reflected as a metric in Prometheus is **18.7 seconds**.

This measured delay is well within our acceptable limits and aligns with the theoretical maximum delay calculated from the AlloyDB monitor's refresh cycle and our Prometheus scrape interval. The test confirms that the monitoring pipeline is reporting metric state changes in a timely fashion.

