# Cluster scaling, node failure, and quorum

Documented here are scenarios in which the size of a cluster may change, how the cluster behaves, and how to restore service function when impacted.

### Graceful removal of a node
  - Shutting down a node with monit (or decreasing cluster size by one) will cause the node to gracefully leave the cluster.
  - Cluster size is reduced by one and maintains healthy state. Cluster will continue to operate, even with a single node, as long as other nodes left gracefully.

### Adding and rejoining nodes
- A node started with monit (or added by increasing cluster size) will automatically join the cluster.

### Unexpected shutdown of a node in three node cluster
  - Use `kill -9` to simulate an unexpected shutdown of a node in a three node cluster.
  - Initially, the cluster understands that the node did not exit gracefully and that the intended cluster size is 3.
  - After a timeout (6 seconds by default), the two remaining nodes "clean" any memory of the missing node and *form a 2-node cluster* with the same cluster id.
  - Because the cluster id and gcomm address have not changed, the node will join the cluster when restarted with monit.

### Unexpected shutdown of a node in a two node cluster
  - Use `kill -9` to simulate an unexpected shutdown of a node when only two of three nodes are healthy.
  - The remaining node does not have quorum and will not accept connections.
  - To fix this cluster you must bring down the last node, bootstrap the node with the latest data, then have the other nodes join the bootstrapped node.

### Re-bootstrapping the cluster after quorum is lost
  - The start script will currently bootstrap node 0 only on initial deploy. Re-bootstrapping requires a manual bootstrap. For more information on manually bootstrapping a cluster, see [Bootstrapping Galera](bootstrapping.md).
  - The node with the most up-to-date information should be bootstrapped.

### One node partitioned from the other two
  - This can be simulated by adding iptables rules (see below) to the VM of node 0 preventing it from communicating with the other two VMs.
  - The two-node side of the partition constitutes a healthy cluster because the two nodes have quorum (greater than half). The single node is part of a "non-primary component" (meaning an unhealthy subset of the cluster) until it can rejoin the other two nodes.
  - All nodes suspend writes once they notice something is wrong with the cluster (write requests hang). After the six second grace period nearly all requests to nodes in a non-primary component will fail with the error `WSREP has not yet prepared this node for application use`. In some clients this may manifest itself as `unknown error`. Nodes in the primary component will resume fulfilling write requests.
  - If the partition is dropped, the single node will rejoin the healthy cluster as long as no nodes were bootstrapped while the partition was up.
  - If the single node is bootstrapped, it will create a new one-node cluster. The result:
    - There are now two clusters, one cluster with a single node and another cluster with two nodes.
    - This split-brain scenario will not be healed even if the network partition is removed.
    - Both clusters will consider themselves healthy, and the single-node cluster will accept new data even though it cannot perform any kind of replication.
    - The monit control scripts will never attempt to bootstrap after the first bosh deploy, so the danger here is limited to operator actions.

#### iptables rules for simulating partition
On the node you wish to partition, execute the following:
```
iptables -F && \ # optional - flush existing rules
iptables -A INPUT -p tcp --destination-port 4567 -j DROP && \
iptables -A INPUT -p tcp --destination-port 4568 -j DROP && \
iptables -A INPUT -p tcp --destination-port 4444 -j DROP && \
iptables -A INPUT -p tcp --destination-port 3306 && \
iptables -A OUTPUT -p tcp --destination-port 4567 -j DROP && \
iptables -A OUTPUT -p tcp --destination-port 4568 -j DROP && \
iptables -A OUTPUT -p tcp --destination-port 4444 -j DROP && \
iptables -A OUTPUT -p tcp --destination-port 3306
```
