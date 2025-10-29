# Hot Warm Cold

## Firstly configure minio

```bash
docker compose up minio -d
```

### Access Minio

http://localhost:9001/minio/login

Access Key:
```
AKIAIOSFODNN7EXAMPLE
```
Secret Key:
```
wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

#### Setup repository


Open Minio web UI at http://localhost:9001/minio/login and add a bucket "test", and set the permissions to read and write.

## Secondly start the Elasticsearch cluster

```
docker compose up -d
```

###

Check cluster is up and in the green:

```bash
$ watch curl -s http://localhost:9200/_cluster/health
```

Wait until you see:

```json
{"cluster_name":"docker","status":"green","timed_out":false,"number_of_nodes":3,"number_of_data_nodes":3,"active_primary_shards":27,"active_shards":52,"relocating_shards":0,"initializing_shards":0,"unassigned_shards":0,"unassigned_primary_shards":0,"delayed_unassigned_shards":0,"number_of_pending_tasks":0,"number_of_in_flight_fetch":0,"task_max_waiting_in_queue_millis":0,"active_shards_percent_as_number":100.0}
```

### Setup Elasticsearch cluster

#### Setup script


```bash
$ ./01_setup_cluster                                                                                   ✔  2m 39s  16:30:24
++ docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' minio
+ docker exec -it hot curl -H 'Content-Type: application/json' --data '{"type":"s3","settings":{"bucket":"test","endpoint":"172.26.0.2:9000","protocol":"http"}}' -XPUT http://hot:9200/_snapshot/my_minio_repository
{"acknowledged":true}+ docker exec -it hot curl -H 'Content-Type: application/json' --data-binary @/usr/share/elasticsearch/config/docker_host/cluster.settings -XPUT http://hot:9200/_cluster/settings
{"acknowledged":true,"persistent":{"indices":{"lifecycle":{"poll_interval":"1s"}}},"transient":{}}+ docker exec -it hot curl -H 'Content-Type: application/json' --data-binary @/usr/share/elasticsearch/config/docker_host/ilm.policy -XPUT http://hot:9200/_ilm/policy/hot-warm-cold
{"acknowledged":true}+ docker exec -it hot curl -H 'Content-Type: application/json' --data-binary @/usr/share/elasticsearch/config/docker_host/ilm_hot-delete.policy -XPUT http://hot:9200/_ilm/policy/hot-delete
{"acknowledged":true}+ docker exec -it hot curl -H 'Content-Type: application/json' --data-binary @/usr/share/elasticsearch/config/docker_host/index.template -XPUT http://hot:9200/_index_template/test_indexes
{"acknowledged":true}
```

#### Validate cluster

From Kibana dev tools : http://localhost:5601/app/dev_tools#/console

```
GET /_cat/nodes?v
GET _nodes?filter_path=nodes.*.name,nodes.*.roles
GET _cat/plugins
```

### Generate documents

_From the same directory and docker compose up_

ILM is configured to use 1000 per generation, i.e. roll over every 1000 documents.

Change THREADS and DOCS_PER_THREAD to change how many docs to generate.

Run:
```
docker run  -e DOCS_PER_THREAD=6000 -e THREADS=1 --network=$(docker network ls | grep docker-hot-warm-cold_esnet | cut -d ' ' -f 1) -v $PWD:/usr/share/logstash/config/docker_host docker.elastic.co/logstash/logstash:7.10.0 bin/logstash -f config/docker_host/ls.config --pipeline.workers=1
```

> [!TIP]
> Run this script to generate documents `./02_generate_docs`

### Watch the generations - command line

Requires curl and jq installed.

```bash
watch -n .5 ' echo "* all docs in standard index* "; curl -s localhost:9200/.ds-test-*/_count | jq; echo "* all docs including those in searchable snapshot *"; curl -s localhost:9200/test/_count | jq; curl -s localhost:9200/_cat/shards?v'
```


### Watch the generations - Kibana line

```
GET /_cat/shards/.ds-test-*?v
GET .ds-test-*/_ilm/explain

GET .ds-test-*/_count
GET _data_stream/test
GET /_cat/indices?v

```

Repeat generation documents to create new generations.

### See the snapshot in Minio

http://localhost:9001/minio/test/

Command line:

Requires curl and jq installed.

```bash
watch -n .5  ' echo "* all docs in standard index* "; curl -s localhost:9200/.ds-test-*/_count | jq; echo "* all docs including those in searchable snapshot *"; curl -s localhost:9200/test/_count | jq; curl -s localhost:9200/_cat/shards?v'
```

Kibana:
```
GET restored-.ds-test-000001
GET test/_count
GET /_cat/shards/restored*?v
```

What kind of magic is this ?
