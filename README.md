# `kafka-producer-perf-test` Walkthrough

Walkthrough project covering a tuning scenario using the `kafka-producer-perf-test` tool.  

We're going to set up a 3-node cluster (with a single Zookeeper instance) and walk through a tuning exercise using `kafka-producer-perf-test`.

## Starting the cluster

Start the 3-node cluster by running:

```bash
docker-compose up
```

## Initial sanity check to ensure the cluster is up

We can quickly check the status of the cluster using `zookeeper-shell`:

```bash
docker-compose exec zookeeper zookeeper-shell localhost:2181
```

```
ls /
[admin, brokers, cluster, config, consumers, controller, controller_epoch, feature, isr_change_notification, latest_producer_id_block, leadership_priority, log_dir_event_notification, zookeeper]
```

```
get /controller
{"version":1,"brokerid":3,"timestamp":"1673441159606"}
```

```
get /cluster/id
{"version":"1","id":"C9xzVquNTHajDcfvy-7LKA"}
```

```
ls /brokers/ids
[1, 2, 3]
```

```
get /brokers/ids/1
{"features":{},"listener_security_protocol_map":{"BROKER":"PLAINTEXT","PLAINTEXT_HOST":"PLAINTEXT"},"endpoints":["BROKER://broker1:9091","PLAINTEXT_HOST://localhost:29091"],"jmx_port":-1,"port":9091,"host":"broker1","version":5,"tags":{},"timestamp":"1673382717473"}
get /brokers/ids/2
{"features":{},"listener_security_protocol_map":{"BROKER":"PLAINTEXT","PLAINTEXT_HOST":"PLAINTEXT"},"endpoints":["BROKER://broker2:9092","PLAINTEXT_HOST://localhost:29092"],"jmx_port":-1,"port":9092,"host":"broker2","version":5,"tags":{},"timestamp":"1673441159443"}
get /brokers/ids/3
{"features":{},"listener_security_protocol_map":{"BROKER":"PLAINTEXT","PLAINTEXT_HOST":"PLAINTEXT"},"endpoints":["BROKER://broker3:9093","PLAINTEXT_HOST://localhost:29093"],"jmx_port":-1,"port":9093,"host":"broker3","version":5,"tags":{},"timestamp":"1673441159415"}
```

### List all available topics 

```bash
docker-compose exec broker1 kafka-topics --list --bootstrap-server broker1:9091
```

Using `--describe` instead of `--list` will list partition information:

```bash
docker-compose exec broker1 kafka-topics --describe --bootstrap-server broker1:9091
```

And finally, for some optional tests:

```bash
docker-compose exec broker1 kafka-topics --describe --bootstrap-server broker1:9091 --at-min-isr-partitions
	Topic: output	Partition: 0	Leader: 1	Replicas: 1	Isr: 1	Offline:

docker-compose exec broker1 kafka-topics --describe --bootstrap-server broker1:9091 --unavailable-partitions

docker-compose exec broker1 kafka-topics --describe --bootstrap-server broker1:9091 --under-min-isr-partitions

docker-compose exec broker1 kafka-topics --describe --bootstrap-server broker1:9091 --under-replicated-partitions
```

## Create the initial `demo-perf-topic` for performance testing

```bash
docker-compose exec broker1 kafka-topics --bootstrap-server broker1:9091 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2
```

If the operation has been successful, you should see:

```bash
Created topic demo-perf-topic.
```

Let's now describe the newly-created `demo-perf-topic`:

```bash
docker-compose exec broker1 kafka-topics --bootstrap-server broker1:9091 --describe --topic demo-perf-topic
```

You should see something like this:

```
Topic: demo-perf-topic	TopicId: tDa-tXO4Sr-f5gkavptSXw	PartitionCount: 1	ReplicationFactor: 3	Configs: min.insync.replicas=2
	Topic: demo-perf-topic	Partition: 0	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3	Offline:
```

## Start our performance testing

```bash
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all linger.ms=100 batch.size=300000 --print-metrics'
```

You should see something like this in your output:

```bash
249481 records sent, 49896.2 records/sec (47.58 MB/sec), 142.5 ms avg latency, 546.0 ms max latency.
250371 records sent, 50034.2 records/sec (47.72 MB/sec), 6.0 ms avg latency, 17.0 ms max latency.
237600 records sent, 47012.3 records/sec (44.83 MB/sec), 9.2 ms avg latency, 310.0 ms max latency.
1000000 records sent, 49982.506123 records/sec (47.67 MB/sec), 49.50 ms avg latency, 546.00 ms max latency, 7 ms 50th, 347 ms 95th, 418 ms 99th, 431 ms 99.9th.
```

### Start by looking at the following metrics

| Metric Name | Description |
|---|---|
| record-size-avg | important when testing with your OWN application |
| batch-size-avg | am i filling the batches or not?  If you’re not filling the batch size you may want to look there. |
| bufferpool-wait-ratio | percentage of time spent waiting for the network.  Everything is in the record accumulator.  Is the test flawed?  Is the producer just hanging?  If you see a high value here, you may be able to increase the batch size to improve performance. |
| request-latency-avg | sender thread latency - the time spent by the sender thread (that’s item 5 on the previous slide) to get a response back from the broker. |
| record-queue-time-avg | how many seconds you spend in the record accumulator (in the send buffer) |
| produce-throttle-time-avg | applies if you have quotas installed |
| record-retry-rate | hopefully low!  Network issues would increase this.  Some is unlikely to be a problem - lots is cause for concern. |
| compression-rate-avg | how much are you compressing? |
| records-per-request | particularly in the event where messages may be larger in size, this will give you an idea as to how much data is going out per each request that is being made. |
| select-rate | rate at which the sender thread sends from the producer queue (outgoing) |

**request-latency-avg** + **record-queue-time-avg** = the average of the total amount of time fulfilling a request










spurious stuff below - to delete

docker exec -it kafka-1 /bin/bash -c 'kafka-topics --bootstrap-server kafka-1:9092 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'


docker exec -it broker1 /bin/bash -c 'kafka-topics --bootstrap-server broker1:9092 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'

```bash
docker exec -it kafka-1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput -1 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=kafka-1:9092 acks=all'

time docker exec -it kafka-1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=kafka-1:9092 acks=all linger.ms=100 batch.size=300000 --print-metrics'
```


broker1 kafka-cluster cluster-id --bootstrap-server broker1:9091