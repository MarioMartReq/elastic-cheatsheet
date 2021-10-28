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

- List all indices, their node allocation and size. Ordered by descending order by size.
  ```
  GET _cat/shards?v&s=store:desc
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
