# Elasticsearch cheat sheet

Hey! I do tons of Elastic administration and this is a list of the most useful/used commands.

## Restore from snapshot

The restoration process tends to be relatively easy if the cluster is freshly launched or the system indices names do not conflict with the ones in the restoration name. With the commands below, you can fully restore a snapshot.

- First, we need to close all indices, disabling any write/read operation. Note: this stop all current operations, including your Kibana access, so you should get all relevant data before executing it.
  ```
  curl -X POST -u "elastic:ELASTIC-PASSWORD" "http://localhost:9200/_all/_close"
  ```
- Then, start the restoration process with the following command.

  ```
  curl -X POST -u "elastic:ELASTIC-PASSWORD" -k "http://localhost:9200/_snapshot/repository-name/snapshot-name/_restore" -H 'Content-Type: application/json' -d'
  {
      "ignore_unavailable": true,
      "include_aliases": false
  }'
  ```

- When an index is restored, it gets open. Normally, you'll be able to access Kibana rather soon, as from my experience, system indices are the first ones to be restored. In Kibana you can follow the restoration process in Stack Management -> Snapshot and Restore -> Restore status. If you are not able to access Kibana yet, you can check the restoration process with the following commands.
  ```
  curl -u "elastic:ELASTIC-PASSWORD" -k "http://localhost:9200/_cat/indices"
  curl -u "elastic:ELASTIC-PASSWORD" -k "http://localhost:9200/_recovery"
  curl -u "elastic:ELASTIC-PASSWORD" -k "http://localhost:9200/_cat/recovery"
  ```
- Sometimes, after finishing the restoration, some indices remain closed as they were not affected by the restoration. To open all of them again, execute the following command.
  ```
  curl -X POST -u "elastic:ELASTIC-PASSWORD" "http://localhost:9200/_all/_close"
  ```

## Manual storage balancing

Elastic balances the shards allocated to each node trying to achieve a more or less same number of shards per node. This generates problems as shards could occupy vastly different ammounts of space, creating disbalance of disk usage between different nodes. The points below explain how to manually and automatically prevent and solve this.  

Link to [elastic documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#modules-cluster) regarding this topic.


- Get storage usage per elastic node. 
  ```
  GET _cat/allocation?v
  ```
- List all indices, their node allocation and size. Ordered by descending order by size.
  ```
  GET _cat/shards?v&s=store:desc
  ```
- Command to change the disk watermark low and high. Low: no additional shards will be allocated if the storage is above that value. High: shards will be moved to another node if storages surpasses this value. (previously 90%)

  ```
  PUT _cluster/settings
  {
    "transient": {
      "cluster.routing.allocation.disk.watermark.low": "65%",
      "cluster.routing.allocation.disk.watermark.high": "85%"
    }
  }
  ```


- Move one index from one node to another. This is useful because Elasticsearch does not balance the indices based in its size, as it simply tries to have a similar number of shards in each node.
  ```
  POST /_cluster/reroute
  {
      "commands": [
          {
              "move": {
                  "index": "index-name",
                  "shard": 0,
                  "from_node": "node-origin",
                  "to_node": "node-final"
              }
          }
      ]
  }
  ```
