### 1. Check cluster nodes via
```
GET localhost:9200/_cat/nodes?v
```

```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.17.0.3           38          96  65    7.08    5.75     3.25 dil       -      es01
172.17.0.2           25          96  65    7.08    5.75     3.25 dil       -      es03
172.17.0.7           18          96  65    7.08    5.75     3.25 ilm       -      esmaster01
172.17.0.6           25          96  65    7.08    5.75     3.25 ilm       -      esmaster02
172.17.0.5           30          96  65    7.08    5.75     3.25 ilm       *      esmaster03
172.17.0.4           39          96  65    7.08    5.75     3.25 dil       -      es02
```

All nodes joined the cluster, master is always esmaster03.

### 2. Shut down the current master. For your convenience, the node names are the same as docker container names. To stop a node, simply stop it via docker, e.g.

```
docker stop esmaster03
```

```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.17.0.2           43          90  53    5.07    4.03     1.88 dil       -      es03
172.17.0.3           45          90  53    5.07    4.03     1.88 dil       -      es01
172.17.0.7           38          90  53    5.07    4.03     1.88 ilm       -      esmaster01
172.17.0.6           20          90  53    5.07    4.03     1.88 ilm       *      esmaster02
172.17.0.4           29          90  53    5.07    4.03     1.88 dil       -      es02
```

Master is now esmaster02.

What about health?
```
GET localhost:9200/_cluster/health?pretty
```
```
{
  "cluster_name" : "es-docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 1,
  "active_shards" : 2,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

### 3. While you are at this stage, with 2 masters-eligible nodes, create an index with 2 shards and 1 replica and try indexing some data. It doesn't really matter what you index, just create some index with correct settings and put in a few documents.

```
PUT localhost:9200/docs
{
    "settings": {
        "index.number_of_shards": 2,
        "index.number_of_replicas": 1
    }
}
```
Putting documents like:
```
POST localhost:9200/docs/_doc
{
    "doc": 1,
    "field1" : "data1",
    "field2" : "data2"
}
```
Writes were accepted without any problems.

### 4. Stop another master-eligible node. Just to get a point across - try stopping the master-eligible node that is not currently elected master. Your cluster should now be in a bad state. Try listing all nodes via `_cat/nodes`. Did it work?

```
docker stop esmaster01
```

Trying to get nodes now:
```
GET localhost:9200/_cat/nodes?v
```
```
{
  "error" : {
    "root_cause" : [
      {
        "type" : "master_not_discovered_exception",
        "reason" : null
      }
    ],
    "type" : "master_not_discovered_exception",
    "reason" : null
  },
  "status" : 503
}
```

Indexing data resulted with:

```
{
    "error": {
        "root_cause": [
            {
                "type": "cluster_block_exception",
                "reason": "blocked by: [SERVICE_UNAVAILABLE/2/no master];"
            }
        ],
        "type": "cluster_block_exception",
        "reason": "blocked by: [SERVICE_UNAVAILABLE/2/no master];"
    },
    "status": 503
}
```

### 5. In this degraded state, try searching for data. A simple search at the `_search` enpoint is enough, do not bother writing a query. Does searching still work?

```
GET localhost:9200/docs/_search
```

```
{
    "took": 557,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "docs",
                "_type": "_doc",
                "_id": "TUMYCW8B_4gDtwfQOqRp",
                "_score": 1.0,
                "_source": {
                    "doc": 1,
                    "field1": "data1",
                    "field2": "data2"
                }
            },
            {
                "_index": "docs",
                "_type": "_doc",
                "_id": "TkMYCW8B_4gDtwfQUaTL",
                "_score": 1.0,
                "_source": {
                    "doc": 2,
                    "field1": "data1",
                    "field2": "data2"
                }
            },
            {
                "_index": "docs",
                "_type": "_doc",
                "_id": "T0MYCW8B_4gDtwfQXaQx",
                "_score": 1.0,
                "_source": {
                    "doc": 3,
                    "field1": "data1",
                    "field2": "data2"
                }
            }
        ]
    }
}
```

### 6. Bring back one of the dead master-eligible nodes.

```
docker start esmaster01 
```
No problem starting node, then bringing back remaining.

```
docker start esmaster03
```

```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.17.0.4           25          95  64    7.23    5.89     3.50 dil       -      es02
172.17.0.2           38          95  64    7.23    5.89     3.50 dil       -      es03
172.17.0.3           30          95  64    7.23    5.89     3.50 dil       -      es01
172.17.0.6           25          95  64    7.23    5.89     3.50 ilm       -      esmaster02
172.17.0.7           39          95  64    7.23    5.89     3.50 ilm       -      esmaster03
172.17.0.5           18          95  64    7.23    5.89     3.50 ilm       *      esmaster01
```

### 7. Inspect your shard layout via
```
GET localhost:9200/_cat/shards
```

```
docs  1     p      STARTED 3      4.4kb    172.17.0.4  es02
docs  1     r      STARTED 3      4.4kb    172.17.0.3  es01
docs  0     r      STARTED 1      1.7kb    172.17.0.4  es02
docs  0     p      STARTED 1      1.7kb    172.17.0.2  es03
```

Stopping another node.
```
docker stop es02
```
Shards after:
```
docs  1     p      STARTED 3      4.4kb    172.17.0.3  es01
docs  1     r      UNASSIGNED                    
docs  0     p      STARTED 1      1.7kb    172.17.0.2  es03
docs  0     r      UNASSIGNED
```

Cluster health is yellow, querying index-level health:
```
GET localhost:9200/_cat/indices
```

```
green open docs UptLo421gThzgExRnkGO0U 3 1 1 0 13.3kb 5.1kb
```

After a while:
```
docs  1     p      STARTED 3      4.4kb    172.17.0.2  es03
docs  1     r      STARTED 3      4.4kb    172.17.0.3  es01
docs  0     r      STARTED 1      1.7kb    172.17.0.2  es03
docs  0     p      STARTED 1      1.7kb    172.17.0.3  es01
```

Health is again green.

### 8. Take down another node and repeat tasks from 7. Again, keep in mind that by stopping a host, you may have cut off your access via the port you were using. In that case, try using a different port (9200, 9201, 9202). With only one data node left, the cluster cannot assign replica indices, you should be in yellow state.

```
docker stop es01
```
```
docs  1     p      STARTED 3      4.4kb    172.17.0.2  es03
docs  1     r      UNASSIGNED
docs  0     r      STARTED 1      1.7kb    172.17.0.2  es03
docs  0     p      UNASSIGNED
```

Health is yellow now.
```
GET localhost:9200/_cluster/health?pretty
```

```
{
  "cluster_name" : "es-docker-cluster",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 4,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 3,
  "active_shards" : 3,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 3,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 50.0
}
```

### 9. Try searching and indexing in this state. Are both operations working?

Indexing and searching is working.

### 10. Take down the last data node. Access the cluster via master node, via port 9210, 9211 or 9212. What is cluster health now? Index-level health? Shard allocation?

```
docker stop es03
```
Cluster health is red.

```
docs    1 p UNASSIGNED    
docs    1 r UNASSIGNED    
docs    0 p UNASSIGNED    
docs    0 r UNASSIGNED
```

### 11. Bring back the data nodes one by one and observe what happens. Did you lose any data when all 3 data nodes went down?

Restarted all nodes, health goes to green and search is working again. Data were restored.
