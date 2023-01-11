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
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput -1 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091'
```

You should see something like this in your output:

```bash
62673 records sent, 12534.6 records/sec (11.95 MB/sec), 1648.0 ms avg latency, 2346.0 ms max latency.
87184 records sent, 17436.8 records/sec (16.63 MB/sec), 1973.0 ms avg latency, 2391.0 ms max latency.
106752 records sent, 21350.4 records/sec (20.36 MB/sec), 1552.4 ms avg latency, 1609.0 ms max latency.
99488 records sent, 19889.6 records/sec (18.97 MB/sec), 1592.8 ms avg latency, 1806.0 ms max latency.
91568 records sent, 18313.6 records/sec (17.47 MB/sec), 1799.5 ms avg latency, 1868.0 ms max latency.
92512 records sent, 18502.4 records/sec (17.65 MB/sec), 1774.6 ms avg latency, 1804.0 ms max latency.
91152 records sent, 18230.4 records/sec (17.39 MB/sec), 1779.7 ms avg latency, 1889.0 ms max latency.
73584 records sent, 14690.4 records/sec (14.01 MB/sec), 2022.3 ms avg latency, 2736.0 ms max latency.
36032 records sent, 7206.4 records/sec (6.87 MB/sec), 4647.9 ms avg latency, 5171.0 ms max latency.
73488 records sent, 14697.6 records/sec (14.02 MB/sec), 2343.4 ms avg latency, 2698.0 ms max latency.
79216 records sent, 15843.2 records/sec (15.11 MB/sec), 2099.9 ms avg latency, 2339.0 ms max latency.
84000 records sent, 16800.0 records/sec (16.02 MB/sec), 1953.8 ms avg latency, 2035.0 ms max latency.
1000000 records sent, 16315.342949 records/sec (15.56 MB/sec), 1951.54 ms avg latency, 5171.00 ms max latency, 1802 ms 50th, 2572 ms 95th, 4898 ms 99th, 5167 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 1:03.22 total
```

Note: if we run that test again with the `--print-metrics` flag being passed in, we will get a significant amount of usable metrics to describe the test as it ran against the cluster:

```bash
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput -1 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 --print-metrics'
```

You'll see a large number of metrics being returned at the end of the run:

```
Metric Name                                                                                                            Value
app-info:commit-id:{client-id=perf-producer-client}                                                                  : 9abb940238fc825f9fe29bfde7270c1709e084ba
app-info:start-time-ms:{client-id=perf-producer-client}                                                              : 1673452900214
app-info:version:{client-id=perf-producer-client}                                                                    : 7.3.1-ce
kafka-metrics-count:count:{client-id=perf-producer-client}                                                           : 178.000
```

### Start by looking at the following metrics

| Metric Name | Description |
|---|---|
| record-size-avg | Important when testing with your own application data |
| batch-size-avg | The average number of bytes sent per partition per-request.  Is the producer filling their batches?  If they're not, you may want to look at further optimisation to improve this. |
| bufferpool-wait-ratio | The percentage of time spent waiting for the network.  Everything is in the record accumulator.  Is the test flawed?  Is the producer just hanging?  If you see a high value here, you may be able to increase the batch size to improve performance. |
| request-latency-avg | Sender thread latency - the time spent by the sender thread to get a response back from the broker. |
| record-queue-time-avg | This metric covers the average time (in ms) your record batches spend in the record accumulator (in the send buffer). |
| produce-throttle-time-avg | This metric only applies if you have quotas installed |
| record-retry-rate | Network issues would cause this to increase.  Some is unlikely to be a problem - lots is cause for concern. |
| compression-rate-avg | How much are you compressing your data in batches on average? |
| records-per-request | Particularly in the event where messages may be larger in size, this will give you an idea as to how much data is going out per each request that is being made. |
| select-rate | The rate at which the sender thread sends from the producer queue (outgoing) |

**request-latency-avg** + **record-queue-time-avg** = the average of the total amount of time fulfilling a request

### What have we learned from the first test?

Highlights:
- `--throughput -1` means the Producer sends data as much as it can; messages are produced as quickly as possible, with no throttling limit.
- Average latency ranges from 1552.4 ms to 4647.9 ms - this seems to be something that we should be able to improve.

## Second performance test

The second test involves setting `acks=all` - however in our case, as we're running Kafka 3.2.1, this will make no difference to the test (as this is now the default for a producer)[https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html#acks].  Prior to Kafka 3.0 (the default setting was 1)[https://docs.confluent.io/cloud/current/client-apps/optimizing/durability.html#producer].





---
```bash
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all linger.ms=100 batch.size=300000 --print-metrics'
```

spurious stuff below - to delete

docker exec -it kafka-1 /bin/bash -c 'kafka-topics --bootstrap-server kafka-1:9092 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'


docker exec -it broker1 /bin/bash -c 'kafka-topics --bootstrap-server broker1:9092 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'

```bash
docker exec -it kafka-1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput -1 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=kafka-1:9092 acks=all'

time docker exec -it kafka-1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=kafka-1:9092 acks=all linger.ms=100 batch.size=300000 --print-metrics'
```


broker1 kafka-cluster cluster-id --bootstrap-server broker1:9091