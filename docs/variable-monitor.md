# Use status variables to monitor

Using standard queries, you can check the status of replication across the cluster from a database client. The status variables associated with replication begin with the prefix `wsrep_`. You can display all of the variables with their current values with one query. The results depend on the Galera version running on your server.

```{.bash data-prompt="mysql>"}
mysql> SHOW GLOBAL STATUS LIKE 'wsrep_%';
```
??? example "Expected output"

    ```{.text .no-copy}

    +-------------------------+-------+
    | Variable_name          | Value |
    +------------------------+-------+
    | wsrep_protocol_version | 11    |
    | ...                    | ...   |
    | wsrep_ready            | ON    |
    +------------------------+-------+
    ```
    
## Track cluster membership and health

wsrep_cluster_size

wsrep_cluster_status

## Check node readiness and connectivity

wsrep_ready

wsrep_connected

wsrep_local_state_comment

## Analyze replication lag and flow control impact

wsrep_local_recv_queue_avg

wsrep_flow_control_paused

## Check outbound replication performance

### Check outbound replication performance

Use the `wsrep_local_recv_queue_avg` variable to evaluate how well a node processes incoming replication events. This metric shows the average length of the receive queue, which indicates whether the node applies transactions as fast as it receives them. A consistently elevated value suggests potential I/O bottlenecks, slow applier threads, or other factors that impair replication performance.

The `wsrep_local_send_queue_avg` variable provides the average number of transactions queued for replication since the last `FLUSH STATUS` command. Elevated values—especially those well above 0.0—signal replication delays. These delays may stem from constrained network links, high-latency connections, or insufficient capacity on remote nodes to receive transactions.

```{.bash data-prompt="mysql>"}
mysql> SHOW GLOBAL STATUS LIKE 'wsrep_local_send_queue_avg';
```
??? example "Expected output"

    ```{.text .no-copy}

    +----------------------------+------------+
    | Variable_name              | Value      |
    +----------------------------+------------+
    | wsrep_local_send_queue_avg | .20414418  |
    +----------------------------+------------+
    ```
### Troubleshoot actions

If the `wsrep_local_send_queue_avg` value remains high, it indicates that the node is
unable to replicate transactions quickly enough to keep up with the workload. This
typically results in replication delays and cluster performance issues. To reduce the
backlog, consider the following actions:

* Check the network between nodes. Look for latency, dropped packets, or limited
  bandwidth that might slow replication. Tools like `iperf`, `ping`, and `traceroute`
  can help you identify weak network links or routing problems.

* Monitor the load on each node. High CPU usage or disk I/O, either locally or on
  connected peers, can delay replication. Use system monitoring tools to review
  performance and reduce competing workloads when necessary.

* Adjust replication thread settings. If the application workload allows it,
  increasing the `wsrep_slave_threads` value can improve parallel replication and help
  reduce queue size.

* Analyze flow control behavior. Review `wsrep_flow_control_sent` and
  `wsrep_flow_control_paused` metrics to determine whether this node is slowing down
  replication due to pressure from slower peers.

* Review cluster node balance. Try to avoid running nodes with significantly different
  hardware or network performance in the same cluster. Imbalanced environments often
  result in persistent replication lag.

* Enable compression if available. Compressing replication traffic can reduce network
  usage and improve performance, particularly in environments with slower or
  congested network links.
  
## Monitor node status

<table>
  <thead>
    <tr>
      <th style="width: 260px;">Variable</th>
      <th>What it indicates</th>
      <th>Example values</th>
      <th>Typical cause when not OK</th>
      <th>Troubleshooting tips</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>wsrep_ready</code></td>
      <td>Node readiness to handle queries</td>
      <td><code>ON</code>, <code>OFF</code></td>
      <td>SST in progress, Donor state, Desynced, local issues (disk, memory)</td>
      <td>
        * Check <code>wsrep_local_state_comment</code> (should be <code>Synced</code>)<br>
        * Wait for SST/IST to finish<br>
        * Review logs and system resource usage
      </td>
    </tr>
    <tr>
      <td><code>wsrep_local_state_comment</code></td>
      <td>Node’s current role/state</td>
      <td><code>Joining</code>, <code>Donor</code>, <code>Synced</code>, etc.</td>
      <td>Node is syncing or recovering</td>
      <td>
        * Monitor SST/IST progress<br>
        * Wait until state changes to <code>Synced</code><br>
        * Avoid restarting other nodes during SST
      </td>
    </tr>
    <tr>
      <td><code>wsrep_local_cert_failures</code></td>
      <td>Number of failed certifications</td>
      <td>0, 1, 5, ...</td>
      <td>Conflicting transactions or replication failures</td>
      <td>
        * Investigate frequent conflicts<br>
        * Review queries or retry failed operations
      </td>
    </tr>
    <tr>
      <td><code>wsrep_local_bf_aborts</code></td>
      <td>Transactions aborted due to brute-force conflict resolution</td>
      <td>0, 10, 100, ...</td>
      <td>High contention on hot rows or large transactions</td>
      <td>
        * Identify conflicting queries<br>
        * Break up large writes<br>
        * Reduce contention through schema or logic changes
      </td>
    </tr>
  </tbody>
</table>


#### wsrep_ready

The `wsrep_ready` variable indicates whether the node is ready to accept write operations. If this value is not "ON," the node is not prepared to handle write requests, which may affect the overall functionality of the cluster.

```{.bash data-prompt="mysql>"}
mysql> SHOW GLOBAL STATUS LIKE 'wsrep_ready';
```
??? example "Expected output"

    ```{.text .no-copy}
    
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | wsrep_ready   | ON    |
    +---------------+-------+
    ```
    
#### wsrep_local_state_comment

Check the node with a `SHOW STATUS` with `wsrep_local_state_comment`.

```{.bash data-prompt="mysql>"}
mysql> SHOW GLOBAL STATUS LIKE 'wsrep_local_state_comment';
```

    ```{.text .no-copy}
    
    +-----------------------------+-----------+
    | Variable_name               | Value     |
    +-----------------------------+-----------+
    | wsrep_local_state_comment   | Synced    |
    +-----------------------------+-----------+
    ```

Check for these results:

* `Joining` or `Donor`: The node is syncing with either SST or IST. This operation is expected during startup.

* `Disconnected` or `Initialized`: The node has not yet joined the cluster or has lost connection.

If the node is syncing, wait until the operation finishes and changes to `Synced`.

If the node is stuck, do the following:

* Check the database error log (/var/log/mysql/error.log)

* Make sure there is enough disk space

* Confirm that the done node is reachable and healthy
  
Check the network connectivity:

* Test ping and port access, especially TCP 4567, 4568, 4444.

* Check firewall and hostnames

Restart the node if needed, and then monitor the log for progress.


    
To ensure effective monitoring of your Percona XtraDB Cluster, set up specific triggers in addition to the usual MySQL alerting:

### Check the cluster state of each node

<table>
  <thead>
    <tr>
      <th style="width: 240px;">Variable</th>
      <th>What it indicates</th>
      <th>Example values</th>
      <th>Typical cause when not OK</th>
      <th>Troubleshooting tips</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>wsrep_cluster_status</code></td>
      <td>Cluster-wide health</td>
      <td><code>Primary</code>, <code>Non-Primary</code></td>
      <td>Quorum loss, network partition, misconfiguration</td>
      <td>
        * Ensure a majority of nodes are running and reachable<br>
        * Check <code>wsrep_cluster_address</code> and firewall rules<br>
        * Bootstrap the cluster if needed
      </td>
    </tr>
    <tr>
      <td><code>wsrep_cluster_size</code></td>
      <td>Number of nodes currently in the cluster</td>
      <td>1, 2, 3, ...</td>
      <td>Other nodes are offline or unreachable</td>
      <td>
        * Check node status across the cluster<br>
        * Restart missing nodes carefully
      </td>
    </tr>
    <tr>
      <td><code>wsrep_connected</code></td>
      <td>Whether the node is connected to the cluster</td>
      <td><code>ON</code>, <code>OFF</code></td>
      <td>Node cannot communicate with cluster peers</td>
      <td>
        * Confirm IP and port access (4567, 4568, 4444)<br>
        * Verify network interfaces and cluster address
      </td>
    </tr>
    <tr>
      <td><code>wsrep_flow_control_sent</code></td>
      <td>Times this node paused replication to slow the cluster</td>
      <td>0, 50, 1000, ...</td>
      <td>The node’s receive queue is full, replication is lagging</td>
      <td>
        * Check disk and network speed<br>
        * Reduce long-running writes<br>
        * Avoid underpowered nodes in the cluster
      </td>
    </tr>
    <tr>
      <td><code>wsrep_flow_control_recv</code></td>
      <td>Times this node was paused by others via flow control</td>
      <td>0, 20, 300, ...</td>
      <td>Other nodes are struggling to keep up, often due to resource issues</td>
      <td>
        * Look at cluster-wide disk/network performance<br>
        * Monitor if a specific node is consistently causing throttling
      </td>
    </tr>
  </tbody>
</table>

Connect to the MySQL server using a MySQL client or command-line tool to access one of the nodes in the cluster. 

Next, verify that the `wsrep_cluster_status` does not equal "Primary." This status indicates whether the cluster is stable. If the status is not "Primary," the cluster may be experiencing issues, such as a split-brain scenario or insufficient nodes to form a quorum.

```{.bash data-prompt="mysql>"}
mysql> SHOW GLOBAL STATUS LIKE 'wsrep_cluster_status';
```

??? example "Expected output"

    ```{.text .no-copy}
    
    +-----------------------------+-------------+
    | Variable_name               | Value       |
    +-----------------------------+-------------+
    | wsrep_cluster_status        | Primary     |
    +-----------------------------+-------------+
    ```
    
### Verify node state

Check the [`wsrep_connected`](wsrep-status-index.md#`wsrep_connected`) and [`wsrep_ready`](wsrep-status-index.md#wsrep_ready) variables to ensure both equal "ON."

The `wsrep_connected` variable indicates whether the node is connected to the cluster. If this value is not "ON," the node is disconnected and cannot participate in cluster operations.


```{.bash data-prompt="mysql>"}
mysql> SHOW GLOBAL STATUS LIKE 'wsrep_connected';
```



### Monitor replication conflicts  

Identify excessive replication conflicts by monitoring the [`wsrep_local_cert_failures`](wsrep-status-index.md#wsrep_local_cert_failures)  and the [`wsrep_local_bf_aborts`](wsrep-status-index.md#wsrep_local_bf_aborts) variables.

The `wsrep_local_cert_failures` variable tracks the number of certification failures during the replication process. Certification failures occur when a node attempts to apply a write operation that conflicts with another operation already applied to the cluster. A high number of certification failures can indicate frequent write conflicts, leading to performance issues and increased latency.

```{.bash data-prompt="mysql>"}
mysql> SHOW GLOBAL STATUS LIKE 'wsrep_local_cert_failures';
```

??? example "Expected output"

    ```{.text .no-copy}

    +-----------------------------+-------+
    | Variable_name               | Value |
    +-----------------------------+-------+
    | wsrep_local_cert_failures   | 0     |
    +-----------------------------+-------+
    ```

The `wsrep_local_bf_aborts` variable tracks the number of aborts due to conflicts with write sets being processed. These conflicts typically happen when multiple nodes attempt to write to the same data simultaneously, resulting in conflicts that require one operation to be aborted.



* You can identify excessive flow control messages by monitoring the 
  [`wsrep_flow_control_sent`](wsrep-status-index.md#wsrep_flow_control_sent) 
  and [`wsrep_flow_control_recv`](wsrep-status-index.md#wsrep_flow_control_recv) 
  variables.

  Flow control messages are signals used in the cluster to manage the 
  flow of replication traffic between nodes. They help prevent a 
  situation where a node becomes overwhelmed with incoming data, 
  especially if it is lagging behind in processing transactions. 
  By regulating the flow of data, these messages ensure that all 
  nodes can keep up with the replication process without losing data 
  integrity.

  The `wsrep_flow_control_sent` variable counts the number of flow 
  control messages sent by the node to manage replication traffic. 
  Conversely, the `wsrep_flow_control_recv` variable tracks the 
  number of flow control messages received by the node, indicating 
  how often the node has to pause or slow down its processing to 
  accommodate the flow control mechanism.

  Monitoring these variables allows you to assess the frequency of 
  flow control messages in the cluster. A high number of these 
  messages may indicate performance issues, such as nodes struggling 
  to keep up with replication, prompting you to investigate and 
  optimize the cluster's performance.


  * Large replication queues indicate a backlog of transactions waiting to be processed in a database cluster. You can identify these queues by monitoring the [`wsrep_local_recv_queue`](wsrep-status-index.md#wsrep_local_recv_queue) variable. When the replication queue grows significantly, it suggests that the system struggles to keep up with incoming changes, which can lead to delays in data synchronization across nodes. This situation may result in increased latency for read and write operations, potential data inconsistencies, and a negative impact on overall system performance. Addressing large replication queues is crucial for maintaining efficient database operations and ensuring timely data availability.

## Gather cluster metrics

Gathering cluster metrics for long-term analysis and visualization plays a crucial role in maintaining the health and performance of a database cluster. Consistent collection of specific performance data over time allows for the creation of graphs that facilitate monitoring and evaluation. Tracking these metrics enables the identification of trends, early detection of issues, and informed decision-making to optimize overall cluster performance. The following list outlines essential metrics that should be collected to ensure effective monitoring and analysis.

* Queue Sizes:

  The `wsrep_local_recv_queue` and `wsrep_local_send_queue` variables provide insights into the sizes of the local receive and send queues in a database cluster. 
  
  * The `wsrep_local_recv_queue` tracks the number of transactions waiting to be processed by the local node. A large receive queue may indicate that the node struggles to keep up with incoming replication traffic, potentially leading to delays in data synchronization and increased latency for read and write operations.
  
  * The `wsrep_local_send_queue` monitors the number of transactions that the local node has sent to other nodes but have not yet been acknowledged. A large send queue can suggest that other nodes are unable to process incoming changes quickly enough, which may also contribute to replication delays and affect overall system performance.
  
  Understanding these queue sizes is essential for diagnosing performance issues and ensuring efficient data replication across the cluster.

* Flow control metrics:

  The `wsrep_flow_control_sent` and `wsrep_flow_control_recv` variables provide important information about the flow control mechanism in a database cluster.
  
  * The `wsrep_flow_control_sent` variable indicates the number of flow control messages sent by the local node to other nodes. Flow control messages help manage the rate of data replication, ensuring that nodes do not become overwhelmed with incoming transactions. A high number of sent flow control messages may suggest that the local node frequently needs to slow down the replication process to maintain stability.
  
  * The `wsrep_flow_control_recv` variable tracks the number of flow control messages received by the local node from other nodes. This metric reflects how often the local node must pause or slow down its operations due to requests from other nodes. A high count of received flow control messages can indicate that the local node is experiencing pressure from its peers, which may lead to delays in processing transactions.
  
  Monitoring these flow control metrics is essential for understanding the dynamics of data replication within the cluster and for identifying potential performance bottlenecks.

* Replication metrics:

  The `wsrep_replicated` and `wsrep_received` variables provide critical insights into the replication process within a database cluster.
  
  * The `wsrep_replicated` variable indicates the total number of transactions that the local node has successfully replicated to other nodes in the cluster. This metric reflects the effectiveness of the replication process and helps assess how much data has been shared across the cluster. A high value for `wsrep_replicated` suggests that the node actively participates in the replication process and contributes to data consistency across all nodes.
  
  * The `wsrep_received` variable tracks the total number of transactions that the local node has received from other nodes. This metric shows how many transactions have been sent to the local node for processing. A high value for `wsrep_received` indicates that the node is receiving a significant amount of data from its peers, which can impact its performance if the incoming transaction rate exceeds its processing capacity.
  
  Understanding these replication metrics is essential for evaluating the health and efficiency of the replication process in the cluster. Monitoring both `wsrep_replicated` and `wsrep_received` helps identify potential issues related to data synchronization and performance bottlenecks.

  The `wsrep_replicated_bytes` and `wsrep_received_bytes` variables provide important information about the volume of data involved in the replication process within a database cluster.
  
  * The `wsrep_replicated_bytes` variable indicates the total number of bytes that the local node has successfully replicated to other nodes in the cluster. This metric reflects the amount of data shared across the cluster and helps assess the efficiency of the replication process. A high value for `wsrep_replicated_bytes` suggests that the node actively participates in data replication, contributing to overall data consistency.
  
  * The `wsrep_received_bytes` variable tracks the total number of bytes that the local node has received from other nodes. This metric shows the volume of data sent to the local node for processing. A high value for `wsrep_received_bytes` indicates that the node is receiving a significant amount of data from its peers, which can affect performance if the incoming data rate exceeds the node's processing capacity.
  
  Understanding these replication metrics is essential for evaluating the health and efficiency of the replication process in the cluster. Monitoring both `wsrep_replicated_bytes` and `wsrep_received_bytes` helps identify potential issues related to data synchronization and performance bottlenecks.

## Use Percona Monitoring and Management

[Percona Monitoring and Management](https://www.percona.com/doc/percona-monitoring-and-management/index.html) includes two dashboards to monitor PXC:

1. PXC/Galera Cluster Overview:

    ![image](_static/pmm.pxc-galera-cluster-overview.png)

2. PXC/Galera Graphs:

    ![image](_static/pmm.pxc-galera-graphs.png)

    These dashboards are available from the menu:

    ![image](_static/pmm.menu.ha.png)

Please refer to the [official documentation](https://www.percona.com/doc/percona-monitoring-and-management/index.html) for details on Percona Monitoring and Management installation and setup.


