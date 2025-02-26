---
title: Distributed deployment
weight: 50
---

Since version v0.8.0 Qdrant supports a distributed deployment mode.
In this mode, multiple Qdrant services communicate with each other to distribute the data across the peers to extend the storage capabilities and increase stability.

To enable distributed deployment - enable the cluster mode in the [configuration](../configuration) or using the ENV variable: `QDRANT__CLUSTER__ENABLED=true`.

```yaml
cluster:
  # Use `enabled: true` to run Qdrant in distributed deployment mode
  enabled: true
  # Configuration of the inter-cluster communication
  p2p:
    # Port for internal communication between peers
    port: 6335

  # Configuration related to distributed consensus algorithm
  consensus:
    # How frequently peers should ping each other.
    # Setting this parameter to lower value will allow consensus
    # to detect disconnected node earlier, but too frequent
    # tick period may create significant network and CPU overhead.
    # We encourage you NOT to change this parameter unless you know what you are doing.
    tick_period_ms: 100
```

With default configuration, Qdrant will use port `6335` for its internal communication.
All peers should be accessible on this port from within the cluster, but make sure to isolate this port from outside access, as it might be used to perform write operations.

Additionally, the first peer of the cluster should be provided with its URL, so it could tell other nodes how it should be reached.
Use the `uri` CLI argument to provide the URL to the peer:

```bash
./qdrant --uri 'http://qdrant_node_1:6335'
```

Subsequent peers in a cluster must know at least one node of the existing cluster to synchronize through it with the rest of the cluster.

To do this, they need to be provided with a bootstrap URL:

```bash
./qdrant --bootstrap 'http://qdrant_node_1:6335'
```

The URL of the new peers themselves will be calculated automatically from the IP address of their request.
But it is also possible to provide them individually using `--uri` argument.

```text
USAGE:
    qdrant [OPTIONS]

OPTIONS:
        --bootstrap <URI>
            Uri of the peer to bootstrap from in case of multi-peer deployment. If not specified -
            this peer will be considered as a first in a new deployment

        --uri <URI>
            Uri of this peer. Other peers should be able to reach it by this uri.
            
            This value has to be supplied if this is the first peer in a new deployment.
            
            In case this is not the first peer and it bootstraps the value is optional. If not
            supplied then qdrant will take internal grpc port from config and derive the IP address
            of this peer on bootstrap peer (receiving side)

```

After a successful synchronization you can observe the state of the cluster through the [REST API](https://qdrant.github.io/qdrant/redoc/index.html?v=master#tag/cluster):

```http
GET /cluster
```

Example result:

```json
{
  "result": {
    "status": "enabled",
    "peer_id": 11532566549086892000,
    "peers": {
      "9834046559507417430": {
        "uri": "http://172.18.0.3:6335/"
      },
      "11532566549086892528": {
        "uri": "http://qdrant_node_1:6335/"
      }
    },
    "raft_info": {
      "term": 1,
      "commit": 4,
      "pending_operations": 1,
      "leader": 11532566549086892000,
      "role": "Leader"
    }
  },
  "status": "ok",
  "time": 5.731e-06
}
```

## Raft

Qdrant is using the [Raft](https://raft.github.io/) consensus protocol to maintain consistency regarding the cluster topology and the collections structure.

Operation with points, on the other hand, are not going through the consensus infrastructure.
Qdrant is not intended to have strong transaction guarantees, which allows it to perform point operations with low overhead.
In practice, it means that Qdrant does not guarantee atomic distributed updates but allows you to wait until the [operation is complete](../points/#awaiting-result) to see the results of your writes.

Collection operations, on the contrary, are part of the consensus which guarantees that all operations are durable and eventually executed by all nodes.
In practice it means that a majority of node agree on what operations should be applied before the service will perform them.

Practically, it means that if the cluster is in a transition state - either electing a new leader after a failure or starting up, the collection update operations will be denied.

You may use the cluster [REST API](https://qdrant.github.io/qdrant/redoc/index.html?v=master#tag/cluster) to check the state of the consensus.

## Sharding

A Collection in Qdrant is made of one or several shards.
Each shard is an independent storage of points which is able to perform all operations provided by collections.
Points are distributed among shards according to the [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing) algorithm, so that shards are managing non-intersecting subsets of points.

During the creation of the collection, shards are evenly distributed across all existing nodes.
Each node knows where all parts of the collection are stored through the [consensus protocol](./#raft), so if it is time to search - each node could query all other nodes to obtain the full search result.

You can define number of shards in your create-collection request:

```http
PUT /collections/{collection_name}

{
    "name": "example_collection",
    "vectors": {
      "size": 300,
      "distance": "Cosine"
    },
    "shard_number": 6
}
```

```python
from qdrant_client import QdrantClient
from qdrant_client.http import models

client = QdrantClient(host="localhost", port=6333)

client.recreate_collection(
    name="{collection_name}",
    vectors_config=models.VectorParams(size=300, distance=models.Distance.COSINE),
    shard_number=6
)
```

We recommend selecting the number of shards as a factor of the number of nodes you are currently running in your cluster.
For example, if you have 3 nodes, 6 shards could be a good option.

### Cluster scaling

If you want to extend your cluster with new nodes or some nodes become slower than the others, it might be helpful to re-balance the shard distribution in the cluster.

*Since version v0.9.0*, Qdrant allows moving shards between nodes in the cluster and removing nodes from the cluster.

This functionality unlocks the ability to dynamically scale the cluster size without downtime.

Qdrant provides the information regarding the current shard distribution in the cluster with the [Collection Cluster info API](https://qdrant.github.io/qdrant/redoc/index.html?v=v0.9.0#tag/cluster/operation/collection_cluster_info).

Use the [Update collection cluster setup API](https://qdrant.github.io/qdrant/redoc/index.html?v=v0.9.0#tag/cluster/operation/update_collection_cluster) to initiate the shard transfer:

```http
POST /collections/{collection_name}/cluster

{
  "move_shard": {
    "shard_id": 0,
    "from_peer_id": 381894127,
    "to_peer_id": 467122995
  }
}
```

After the transfer is initiated, the service will keep both copies of the shard updated until the transfer is complete.
It will also make sure the transferred shard indexing process is keeping up before performing a final switch. This way, Qdrant ensures that there will be no degradation in performance at the end of the transfer.

In case you want to downscale the cluster, you can move all shards away from a peer and then remove the peer using [Remove peer from the cluster API](https://qdrant.github.io/qdrant/redoc/index.html?v=v0.9.0#tag/cluster/operation/remove_peer).

```http
DELETE /cluster/peer/{peer_id}
```

After that, Qdrant will exclude the node from the consensus, and the instance will be ready for the shutdown.

## Replication

Currently work-in-progress, see [Roadmap](https://qdrant.to/roadmap)
