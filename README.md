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

For our initial testing, we're going to create a topic (`demo-perf-topic`) with a single partition.  Note that we are creating replica partitions (`--replication-factor 3`) and we're ensuring that the producer will be forced to wait if less than 2 partitions are up-to-date (`--config min.insync.replicas=2`):

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

Let's run `kafka-producer-perf-test` just to see how long it takes to send 1,000,000 messages (`--num-records`) over to the broker:

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

The last line tells you the total time taken to run the test (1m 03s).

Note: if we run that test again with the `--print-metrics` flag being passed in to `kafka-producer-perf-test`, we will get a significant amount of usable metrics to describe the test as it ran against the cluster:

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
- `compression-rate` and `compression-rate-avg` are 1.000 (no compression).
- `bufferpool-wait-ratio` is `0.717` (71% of the time the producer is blocked from running `send()`)
- `request-latency-avg` is `4.331`
- `record-queue-time-avg` is `1751.218` (clearly there's some Producer latency...)

## Second performance test

The second test involves setting `acks=all` - however in our case, [as we're running Kafka 3.3.0](https://www.confluent.io/blog/introducing-confluent-platform-7-3/), this will make no difference to our test [as this is now the default for a producer](https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html#acks).  

Prior to Kafka 3.0 [the default setting was 1](https://docs.confluent.io/cloud/current/client-apps/optimizing/durability.html#producer).

If you are running with an earlier version of Confluent Platform, you'll probably notice `acks=all` will have an impact on both latency and throughput; you'll see worse numbers after adding that.

## Third performance test

Looking at the metrics output from the first test, one noteworthy metric is `record-queue-time-avg`:

```
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                              : 2330.826
```

This suggests that we're seeing some latency from the producer.

- Producer is queueing more than it’s dequeuing:
  - The Produce Request size is too low: too many small network requests (all of which are waiting for acks)
  - We need to produce with bigger batches

What happens if we set `linger.ms` to 100?  Adding this configuration should give the Producer more time to create larger batches.

- Note: setting `linger.ms` between 10 and 100 is generally considered to be good practice. Increasing this value beyond 100 is unlikely to yield positive results and setting `linger.ms` to a value that is higher than 1000 will have detrimental effects on overall performance.

```bash
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput -1 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all linger.ms=100 --print-metrics'
```

Here's some of the output:

```
41601 records sent, 6623.3 records/sec (6.32 MB/sec), 1158.0 ms avg latency, 4956.0 ms max latency.
74992 records sent, 14998.4 records/sec (14.30 MB/sec), 3298.5 ms avg latency, 4987.0 ms max latency.
77488 records sent, 15359.4 records/sec (14.65 MB/sec), 2078.3 ms avg latency, 2680.0 ms max latency.
89504 records sent, 17900.8 records/sec (17.07 MB/sec), 2076.9 ms avg latency, 2818.0 ms max latency.
93744 records sent, 18748.8 records/sec (17.88 MB/sec), 1699.6 ms avg latency, 1876.0 ms max latency.
78240 records sent, 15648.0 records/sec (14.92 MB/sec), 2139.6 ms avg latency, 2589.0 ms max latency.
86960 records sent, 17322.7 records/sec (16.52 MB/sec), 1840.2 ms avg latency, 2319.0 ms max latency.
68368 records sent, 13673.6 records/sec (13.04 MB/sec), 2082.0 ms avg latency, 2746.0 ms max latency.
96704 records sent, 19340.8 records/sec (18.44 MB/sec), 1981.4 ms avg latency, 2808.0 ms max latency.
99088 records sent, 19813.6 records/sec (18.90 MB/sec), 1660.1 ms avg latency, 1723.0 ms max latency.
100112 records sent, 20022.4 records/sec (19.09 MB/sec), 1616.4 ms avg latency, 1695.0 ms max latency.
78208 records sent, 14800.9 records/sec (14.12 MB/sec), 1687.8 ms avg latency, 2985.0 ms max latency.
368 records sent, 36.7 records/sec (0.03 MB/sec), 3629.0 ms avg latency, 13164.0 ms max latency.
1000000 records sent, 13324.627910 records/sec (12.71 MB/sec), 2143.04 ms avg latency, 15624.00 ms max latency, 1767 ms 50th, 2804 ms 95th, 14939 ms 99th, 15587 ms 99.9th.
[...]
docker exec -it broker1 /bin/bash -c   0.06s user 0.06s system 0% cpu 1:17.20 total
```

### Conclusion

```
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                              : 0.692
```

- Same results as before (bufferpool-wait-ratio=69%) because `batch.size` is already reached
- `linger.ms` only triggers when `batch.size` is not reached
- We need to increase the `batch.size`

## Fourth performance test

We're going to remove `linger.ms` and increase the batch.size `batch.size=300000` instead.

```bash
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput -1 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 --print-metrics'
```

Output:

```
114936 records sent, 22968.8 records/sec (21.90 MB/sec), 703.8 ms avg latency, 2301.0 ms max latency.
174329 records sent, 34851.9 records/sec (33.24 MB/sec), 1030.3 ms avg latency, 2184.0 ms max latency.
176714 records sent, 35167.0 records/sec (33.54 MB/sec), 691.9 ms avg latency, 1098.0 ms max latency.
154135 records sent, 30820.8 records/sec (29.39 MB/sec), 1024.6 ms avg latency, 1516.0 ms max latency.
127401 records sent, 24839.3 records/sec (23.69 MB/sec), 1010.4 ms avg latency, 1431.0 ms max latency.
108986 records sent, 21792.8 records/sec (20.78 MB/sec), 1404.0 ms avg latency, 2333.0 ms max latency.
113735 records sent, 22584.4 records/sec (21.54 MB/sec), 1255.0 ms avg latency, 2880.0 ms max latency.
1000000 records sent, 27467.245310 records/sec (26.19 MB/sec), 1029.19 ms avg latency, 2880.00 ms max latency, 925 ms 50th, 2126 ms 95th, 2401 ms 99th, 2800 ms 99.9th.
[...]
docker exec -it broker1 /bin/bash -c   0.06s user 0.05s system 0% cpu 38.812 total
```

### Conclusion

```
producer-metrics:batch-size-avg:{client-id=perf-producer-client}                                                     : 299883.033
producer-metrics:bufferpool-wait-ratio:{client-id=perf-producer-client}                                              : 0.235
producer-metrics:record-queue-time-avg:{client-id=perf-producer-client}                                              : 1008.798
producer-metrics:request-size-avg:{client-id=perf-producer-client}                                                   : 299597.291
```

Much better throughput and latency in general (38 seconds for this run; 1 minute 17 seconds previously).

- batch-size-avg=191010 now batch.size is not reached -> need to use linger.ms
- bufferpool-wait-ratio=12% -> improvement (from 66%)
- record-queue-time-avg=397 -> improvement (from 2391)
- request-size-avg=253660 -> almost identical to batch.size -> Sender thread is sending ASAP because linger.ms=0
  - We need to increase linger.ms

### Note - had to restart due to disk space issues with Docker

```
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 100000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 --print-metrics'
375684 records sent, 75136.8 records/sec (71.66 MB/sec), 2.3 ms avg latency, 482.0 ms max latency.
443330 records sent, 88666.0 records/sec (84.56 MB/sec), 2.3 ms avg latency, 19.0 ms max latency.
1000000 records sent, 83647.009619 records/sec (79.77 MB/sec), 2.40 ms avg latency, 482.00 ms max latency, 2 ms 50th, 4 ms 95th, 12 ms 99th, 28 ms 99.9th.
```

## Testing larger workloads

So far, we've been looking at a fairly constrained test with a workload that is fixed in size.  What happens if we start to increase the size of the workload?  Does the profile of the test change in a meaningful way?  Is further optimisation necessary and if so, what choices do we have?


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 100000 --num-records 2000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 --print-metrics'
375684 records sent, 75136.8 records/sec (71.66 MB/sec), 2.3 ms avg latency, 482.0 ms max latency.
443330 records sent, 88666.0 records/sec (84.56 MB/sec), 2.3 ms avg latency, 19.0 ms max latency.
1000000 records sent, 83647.009619 records/sec (79.77 MB/sec), 2.40 ms avg latency, 482.00 ms max latency, 2 ms 50th, 4 ms 95th, 12 ms 99th, 28 ms 99.9th.

## 2m records

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 100000 --num-records 2000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 --print-metrics'
381387 records sent, 76277.4 records/sec (72.74 MB/sec), 3.9 ms avg latency, 483.0 ms max latency.
445434 records sent, 89086.8 records/sec (84.96 MB/sec), 2.7 ms avg latency, 51.0 ms max latency.
390284 records sent, 77947.7 records/sec (74.34 MB/sec), 18.0 ms avg latency, 498.0 ms max latency.
400501 records sent, 80100.2 records/sec (76.39 MB/sec), 18.7 ms avg latency, 535.0 ms max latency.
2000000 records sent, 81709.359807 records/sec (77.92 MB/sec), 9.30 ms avg latency, 535.00 ms max latency, 2 ms 50

## adding linger.ms
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 100000 --num-records 2000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100 --print-metrics'

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 100000 --num-records 2000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100 --print-metrics'
368368 records sent, 73658.9 records/sec (70.25 MB/sec), 3.3 ms avg latency, 504.0 ms max latency.
431121 records sent, 86224.2 records/sec (82.23 MB/sec), 3.0 ms avg latency, 16.0 ms max latency.
437248 records sent, 87449.6 records/sec (83.40 MB/sec), 2.8 ms avg latency, 29.0 ms max latency.
397388 records sent, 78395.7 records/sec (74.76 MB/sec), 3.5 ms avg latency, 506.0 ms max latency.
2000000 records sent, 80243.941582 records/sec (76.53 MB/sec), 10.40 ms avg latency, 806.00 ms max latency, 3 ms 50th, 5 ms 95th, 426 ms 99th, 789 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 26.630 total



time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 2000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100 --print-metrics'

---

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 --print-metrics'

docker-compose exec broker1 kafka-topics --bootstrap-server broker1:9091 --topic demo-perf-topic3 --replication-factor 3 --partitions 12 --create --config min.insync.replicas=2

```
Created topic demo-perf-topic3.
```

## Final settings

```bash
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 100000 --num-records 3000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100 --print-metrics'
367625 records sent, 73525.0 records/sec (70.12 MB/sec), 3.4 ms avg latency, 533.0 ms max latency.
402345 records sent, 78079.8 records/sec (74.46 MB/sec), 3.1 ms avg latency, 279.0 ms max latency.
428138 records sent, 85627.6 records/sec (81.66 MB/sec), 6.5 ms avg latency, 409.0 ms max latency.
427052 records sent, 85393.3 records/sec (81.44 MB/sec), 2.9 ms avg latency, 52.0 ms max latency.
332446 records sent, 66475.9 records/sec (63.40 MB/sec), 92.2 ms avg latency, 951.0 ms max latency.
2000000 records sent, 77802.847584 records/sec (74.20 MB/sec), 18.65 ms avg latency, 951.00 ms max latency, 3 ms 50th, 6 ms 95th, 670 ms 99th, 934 ms 99.9th.
```

```
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 27.775 total
```


docker exec -it broker1 /bin/bash -c   0.07s user 0.06s system 0% cpu 43.961 total


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 100000 --num-records 3000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=snappy linger.ms=100 --print-metrics'
---
```bash
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all linger.ms=100 batch.size=300000 --print-metrics'
```

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 60000 --num-records 3000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=snappy linger.ms=100 --print-metrics'

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 3000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=snappy linger.ms=100 --print-metrics'


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 60000 --num-records 3000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=snappy linger.ms=100 --print-metrics'


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 70000 --num-records 3000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=snappy linger.ms=100 --print-metrics'


## Final optimisation and settings - 4m records in ~1m

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 70000 --num-records 4000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=snappy linger.ms=100 --print-metrics'
docker exec -it broker1 /bin/bash -c   0.08s user 0.19s system 0% cpu 1:00.34 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 70000 --num-records 4000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'
docker exec -it broker1 /bin/bash -c   0.07s user 0.10s system 0% cpu 58.877 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 80000 --num-records 4000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'

```
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 80000 --num-records 4000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'
357369 records sent, 71473.8 records/sec (68.16 MB/sec), 3.3 ms avg latency, 556.0 ms max latency.
404891 records sent, 80978.2 records/sec (77.23 MB/sec), 3.0 ms avg latency, 12.0 ms max latency.
414713 records sent, 82909.4 records/sec (79.07 MB/sec), 2.9 ms avg latency, 15.0 ms max latency.
271723 records sent, 54344.6 records/sec (51.83 MB/sec), 33.4 ms avg latency, 905.0 ms max latency.
409211 records sent, 81842.2 records/sec (78.05 MB/sec), 2.9 ms avg latency, 12.0 ms max latency.
412470 records sent, 82477.5 records/sec (78.66 MB/sec), 2.9 ms avg latency, 14.0 ms max latency.
118816 records sent, 23701.6 records/sec (22.60 MB/sec), 165.2 ms avg latency, 2571.0 ms max latency.
374218 records sent, 74843.6 records/sec (71.38 MB/sec), 76.1 ms avg latency, 2926.0 ms max latency.
412335 records sent, 82351.7 records/sec (78.54 MB/sec), 2.9 ms avg latency, 13.0 ms max latency.
281140 records sent, 51434.3 records/sec (49.05 MB/sec), 43.5 ms avg latency, 1422.0 ms max latency.
427468 records sent, 85425.3 records/sec (81.47 MB/sec), 34.2 ms avg latency, 1423.0 ms max latency.
4000000 records sent, 69812.901424 records/sec (66.58 MB/sec), 22.88 ms avg latency, 2926.00 ms max latency, 3 ms 50th, 6 ms 95th, 794 ms 99th, 2733 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.07s user 0.09s system 0% cpu 59.007 total
```

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 90000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'


## With 12 partitions

```bash
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 90000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'
273188 records sent, 51907.3 records/sec (49.50 MB/sec), 15.9 ms avg latency, 610.0 ms max latency.
270259 records sent, 53179.7 records/sec (50.72 MB/sec), 83.6 ms avg latency, 1556.0 ms max latency.
306818 records sent, 61326.8 records/sec (58.49 MB/sec), 28.1 ms avg latency, 864.0 ms max latency.
395667 records sent, 79117.6 records/sec (75.45 MB/sec), 5.1 ms avg latency, 24.0 ms max latency.
212679 records sent, 42434.0 records/sec (40.47 MB/sec), 130.7 ms avg latency, 1000.0 ms max latency.
17878 records sent, 3169.9 records/sec (3.02 MB/sec), 1284.2 ms avg latency, 5645.0 ms max latency.
125499 records sent, 25094.8 records/sec (23.93 MB/sec), 2095.8 ms avg latency, 8746.0 ms max latency.
381952 records sent, 76390.4 records/sec (72.85 MB/sec), 5.4 ms avg latency, 81.0 ms max latency.
313874 records sent, 62774.8 records/sec (59.87 MB/sec), 39.7 ms avg latency, 510.0 ms max latency.
282636 records sent, 56527.2 records/sec (53.91 MB/sec), 55.3 ms avg latency, 1496.0 ms max latency.
180381 records sent, 36047.4 records/sec (34.38 MB/sec), 85.6 ms avg latency, 2491.0 ms max latency.
264188 records sent, 52837.6 records/sec (50.39 MB/sec), 109.3 ms avg latency, 1705.0 ms max latency.
382009 records sent, 73975.4 records/sec (70.55 MB/sec), 4.9 ms avg latency, 231.0 ms max latency.
250895 records sent, 50179.0 records/sec (47.85 MB/sec), 100.9 ms avg latency, 1897.0 ms max latency.
212349 records sent, 42469.8 records/sec (40.50 MB/sec), 128.6 ms avg latency, 1740.0 ms max latency.
4000000 records sent, 51295.861706 records/sec (48.92 MB/sec), 120.27 ms avg latency, 8746.00 ms max latency, 5 ms 50th, 470 ms 95th, 2497 ms 99th, 6900 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.07s user 0.10s system 0% cpu 1:19.73 total
```

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 120000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 120000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'
250101 records sent, 49039.4 records/sec (46.77 MB/sec), 27.9 ms avg latency, 663.0 ms max latency.
351576 records sent, 70287.1 records/sec (67.03 MB/sec), 31.0 ms avg latency, 785.0 ms max latency.
386734 records sent, 77331.3 records/sec (73.75 MB/sec), 4.7 ms avg latency, 78.0 ms max latency.
208240 records sent, 41623.0 records/sec (39.69 MB/sec), 179.0 ms avg latency, 2009.0 ms max latency.
380438 records sent, 76057.2 records/sec (72.53 MB/sec), 5.1 ms avg latency, 56.0 ms max latency.
359603 records sent, 71877.5 records/sec (68.55 MB/sec), 16.3 ms avg latency, 627.0 ms max latency.
341823 records sent, 65208.5 records/sec (62.19 MB/sec), 9.3 ms avg latency, 471.0 ms max latency.
216996 records sent, 43390.5 records/sec (41.38 MB/sec), 184.5 ms avg latency, 2183.0 ms max latency.
400109 records sent, 80005.8 records/sec (76.30 MB/sec), 4.2 ms avg latency, 16.0 ms max latency.
277942 records sent, 55300.8 records/sec (52.74 MB/sec), 43.3 ms avg latency, 1178.0 ms max latency.
263804 records sent, 52760.8 records/sec (50.32 MB/sec), 110.0 ms avg latency, 1471.0 ms max latency.
358811 records sent, 71762.2 records/sec (68.44 MB/sec), 38.2 ms avg latency, 1059.0 ms max latency.
4000000 records sent, 63553.599517 records/sec (60.61 MB/sec), 41.31 ms avg latency, 2183.00 ms max latency, 5 ms 50th, 113 ms 95th, 1206 ms 99th, 1972 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.07s user 0.12s system 0% cpu 1:04.70 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 130000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
302800 records sent, 60378.9 records/sec (57.58 MB/sec), 7.1 ms avg latency, 560.0 ms max latency.
351925 records sent, 70370.9 records/sec (67.11 MB/sec), 36.0 ms avg latency, 1050.0 ms max latency.
309345 records sent, 46260.7 records/sec (44.12 MB/sec), 5.8 ms avg latency, 2879.0 ms max latency.
347844 records sent, 69513.2 records/sec (66.29 MB/sec), 149.9 ms avg latency, 3547.0 ms max latency.
205918 records sent, 41150.7 records/sec (39.24 MB/sec), 103.9 ms avg latency, 1901.0 ms max latency.
312982 records sent, 62583.9 records/sec (59.68 MB/sec), 32.0 ms avg latency, 479.0 ms max latency.
382830 records sent, 76550.7 records/sec (73.00 MB/sec), 9.5 ms avg latency, 251.0 ms max latency.
373135 records sent, 74582.3 records/sec (71.13 MB/sec), 9.3 ms avg latency, 203.0 ms max latency.
293299 records sent, 57952.8 records/sec (55.27 MB/sec), 29.2 ms avg latency, 787.0 ms max latency.
428423 records sent, 85667.5 records/sec (81.70 MB/sec), 55.0 ms avg latency, 985.0 ms max latency.
392651 records sent, 78498.8 records/sec (74.86 MB/sec), 5.5 ms avg latency, 26.0 ms max latency.
4000000 records sent, 64896.085144 records/sec (61.89 MB/sec), 36.50 ms avg latency, 3547.00 ms max latency, 6 ms 50th, 81 ms 95th, 798 ms 99th, 3297 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 1:03.18 total


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 140000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 130000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'

almost down to a minute

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 140000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
320240 records sent, 63818.3 records/sec (60.86 MB/sec), 7.7 ms avg latency, 572.0 ms max latency.
236762 records sent, 47314.5 records/sec (45.12 MB/sec), 131.3 ms avg latency, 1906.0 ms max latency.
340630 records sent, 67665.9 records/sec (64.53 MB/sec), 18.4 ms avg latency, 347.0 ms max latency.
363482 records sent, 72696.4 records/sec (69.33 MB/sec), 20.2 ms avg latency, 544.0 ms max latency.
338042 records sent, 67581.4 records/sec (64.45 MB/sec), 12.8 ms avg latency, 388.0 ms max latency.
372560 records sent, 74482.2 records/sec (71.03 MB/sec), 16.2 ms avg latency, 334.0 ms max latency.
395334 records sent, 79066.8 records/sec (75.40 MB/sec), 5.8 ms avg latency, 54.0 ms max latency.
247840 records sent, 49538.3 records/sec (47.24 MB/sec), 133.3 ms avg latency, 1919.0 ms max latency.
405022 records sent, 80988.2 records/sec (77.24 MB/sec), 5.2 ms avg latency, 26.0 ms max latency.
298946 records sent, 59789.2 records/sec (57.02 MB/sec), 110.6 ms avg latency, 1429.0 ms max latency.
399201 records sent, 79760.4 records/sec (76.07 MB/sec), 5.3 ms avg latency, 24.0 ms max latency.
4000000 records sent, 67410.428393 records/sec (64.29 MB/sec), 34.82 ms avg latency, 1919.00 ms max latency, 6 ms 50th, 121 ms 95th, 928 ms 99th, 1699 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 1:00.96 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 160000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
4000000 records sent, 66210.914869 records/sec (63.14 MB/sec), 42.46 ms avg latency, 2936.00 ms max latency, 6 ms 50th, 184 ms 95th, 1050 ms 99th, 2572 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.07s user 0.05s system 0% cpu 1:02.02 total


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 160000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 150000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'
docker exec -it broker1 /bin/bash -c   0.07s user 0.12s system 0% cpu 1:06.83 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 160000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
docker exec -it broker1 /bin/bash -c   0.07s user 0.04s system 0% cpu 1:00.32 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 160000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=0 batch.size=400000 compression.type=lz4 linger.ms=100'
4000000 records sent, 60201.977635 records/sec (57.41 MB/sec), 21.59 ms avg latency, 1736.00 ms max latency, 4 ms 50th, 103 ms 95th, 440 ms 99th, 970 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.08s system 0% cpu 1:08.91 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 160000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=50'


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 180000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=70'


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 160000 --num-records 4000000 --topic demo-perf-topic3 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
[2023-01-11 21:16:04,959] WARN [Producer clientId=perf-producer-client] Error while fetching metadata with correlation id 1 : {demo-perf-topic3=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient)
330057 records sent, 65985.0 records/sec (62.93 MB/sec), 8.3 ms avg latency, 820.0 ms max latency.
389992 records sent, 77967.2 records/sec (74.36 MB/sec), 4.7 ms avg latency, 200.0 ms max latency.
398046 records sent, 79577.4 records/sec (75.89 MB/sec), 12.7 ms avg latency, 234.0 ms max latency.
293663 records sent, 18954.6 records/sec (18.08 MB/sec), 4.1 ms avg latency, 11985.0 ms max latency.
396520 records sent, 79256.4 records/sec (75.58 MB/sec), 919.6 ms avg latency, 11988.0 ms max latency.
371810 records sent, 74347.1 records/sec (70.90 MB/sec), 5.8 ms avg latency, 79.0 ms max latency.
403869 records sent, 80725.4 records/sec (76.99 MB/sec), 4.0 ms avg latency, 63.0 ms max latency.
407350 records sent, 81470.0 records/sec (77.70 MB/sec), 3.9 ms avg latency, 17.0 ms max latency.
390072 records sent, 77983.2 records/sec (74.37 MB/sec), 4.1 ms avg latency, 31.0 ms max latency.
417212 records sent, 83375.7 records/sec (79.51 MB/sec), 3.7 ms avg latency, 15.0 ms max latency.
4000000 records sent, 63521.303457 records/sec (60.58 MB/sec), 96.17 ms avg latency, 11988.00 ms max latency, 4 ms 50th, 8 ms 95th, 163 ms 99th, 11979 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.07s user 0.05s system 0% cpu 1:04.57 total




time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 160000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'

docker-compose exec broker1 kafka-topics --bootstrap-server broker1:9091 --topic demo-perf-topic2 --replication-factor 3 --partitions 12 --create --config min.insync.replicas=2


## 12 partitions winner
time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 180000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
353450 records sent, 70647.6 records/sec (67.37 MB/sec), 6.5 ms avg latency, 582.0 ms max latency.
390766 records sent, 78153.2 records/sec (74.53 MB/sec), 14.2 ms avg latency, 336.0 ms max latency.
305988 records sent, 61148.7 records/sec (58.32 MB/sec), 43.4 ms avg latency, 878.0 ms max latency.
387568 records sent, 77498.1 records/sec (73.91 MB/sec), 7.3 ms avg latency, 85.0 ms max latency.
361439 records sent, 72258.9 records/sec (68.91 MB/sec), 11.7 ms avg latency, 195.0 ms max latency.
406714 records sent, 81294.0 records/sec (77.53 MB/sec), 6.3 ms avg latency, 52.0 ms max latency.
359079 records sent, 71815.8 records/sec (68.49 MB/sec), 20.9 ms avg latency, 352.0 ms max latency.
409429 records sent, 81853.1 records/sec (78.06 MB/sec), 6.9 ms avg latency, 119.0 ms max latency.
360616 records sent, 71878.8 records/sec (68.55 MB/sec), 9.2 ms avg latency, 142.0 ms max latency.
402802 records sent, 80496.0 records/sec (76.77 MB/sec), 8.6 ms avg latency, 151.0 ms max latency.
4000000 records sent, 75018.754689 records/sec (71.54 MB/sec), 12.81 ms avg latency, 878.00 ms max latency, 6 ms 50th, 29 ms 95th, 214 ms 99th, 564 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.07s user 0.06s system 0% cpu 54.873 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 200000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'

## and

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 180000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
[2023-01-11 21:36:42,947] WARN [Producer clientId=perf-producer-client] Error while fetching metadata with correlation id 1 : {demo-perf-topic2=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient)
346407 records sent, 69253.7 records/sec (66.05 MB/sec), 6.2 ms avg latency, 672.0 ms max latency.
444707 records sent, 88888.1 records/sec (84.77 MB/sec), 3.8 ms avg latency, 15.0 ms max latency.
413208 records sent, 82592.0 records/sec (78.77 MB/sec), 4.7 ms avg latency, 63.0 ms max latency.
453512 records sent, 90702.4 records/sec (86.50 MB/sec), 3.7 ms avg latency, 17.0 ms max latency.
438778 records sent, 87738.1 records/sec (83.67 MB/sec), 3.7 ms avg latency, 16.0 ms max latency.
420592 records sent, 83252.6 records/sec (79.40 MB/sec), 3.7 ms avg latency, 214.0 ms max latency.
448940 records sent, 89734.2 records/sec (85.58 MB/sec), 8.5 ms avg latency, 215.0 ms max latency.
380592 records sent, 76072.8 records/sec (72.55 MB/sec), 5.0 ms avg latency, 100.0 ms max latency.
450831 records sent, 90148.2 records/sec (85.97 MB/sec), 3.4 ms avg latency, 14.0 ms max latency.
4000000 records sent, 84525.495002 records/sec (80.61 MB/sec), 4.64 ms avg latency, 672.00 ms max latency, 4 ms 50th, 7 ms 95th, 26 ms 99th, 175 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 48.709 total


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 200000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
4000000 records sent, 84327.697432 records/sec (80.42 MB/sec), 7.07 ms avg latency, 638.00 ms max latency, 3 ms 50th, 6 ms 95th, 65 ms 99th, 578 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.07s user 0.04s system 0% cpu 48.813 total


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 220000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 220000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
385539 records sent, 77107.8 records/sec (73.54 MB/sec), 3.9 ms avg latency, 519.0 ms max latency.
383897 records sent, 76748.7 records/sec (73.19 MB/sec), 5.2 ms avg latency, 234.0 ms max latency.
446230 records sent, 89228.2 records/sec (85.09 MB/sec), 3.5 ms avg latency, 11.0 ms max latency.
446093 records sent, 89182.9 records/sec (85.05 MB/sec), 3.5 ms avg latency, 11.0 ms max latency.
442350 records sent, 88452.3 records/sec (84.35 MB/sec), 3.5 ms avg latency, 11.0 ms max latency.
450087 records sent, 90017.4 records/sec (85.85 MB/sec), 3.4 ms avg latency, 11.0 ms max latency.
429478 records sent, 85895.6 records/sec (81.92 MB/sec), 3.6 ms avg latency, 12.0 ms max latency.
445733 records sent, 89111.0 records/sec (84.98 MB/sec), 3.4 ms avg latency, 12.0 ms max latency.
424336 records sent, 84867.2 records/sec (80.94 MB/sec), 3.6 ms avg latency, 18.0 ms max latency.
4000000 records sent, 85711.836805 records/sec (81.74 MB/sec), 3.70 ms avg latency, 519.00 ms max latency, 3 ms 50th, 6 ms 95th, 7 ms 99th, 30 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.07s user 0.04s system 0% cpu 48.079 total


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 240000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 260000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 260000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
377610 records sent, 75491.8 records/sec (71.99 MB/sec), 3.9 ms avg latency, 516.0 ms max latency.
430121 records sent, 86007.0 records/sec (82.02 MB/sec), 3.6 ms avg latency, 14.0 ms max latency.
432881 records sent, 86576.2 records/sec (82.57 MB/sec), 3.6 ms avg latency, 12.0 ms max latency.
455299 records sent, 91041.6 records/sec (86.82 MB/sec), 3.4 ms avg latency, 13.0 ms max latency.
430588 records sent, 86048.8 records/sec (82.06 MB/sec), 3.6 ms avg latency, 13.0 ms max latency.
437456 records sent, 87473.7 records/sec (83.42 MB/sec), 3.5 ms avg latency, 29.0 ms max latency.
453879 records sent, 90775.8 records/sec (86.57 MB/sec), 3.4 ms avg latency, 13.0 ms max latency.
432859 records sent, 86571.8 records/sec (82.56 MB/sec), 3.5 ms avg latency, 13.0 ms max latency.
448233 records sent, 89646.6 records/sec (85.49 MB/sec), 3.4 ms avg latency, 12.0 ms max latency.
4000000 records sent, 86668.255585 records/sec (82.65 MB/sec), 3.55 ms avg latency, 516.00 ms max latency, 3 ms 50th, 6 ms 95th, 7 ms 99th, 11 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 47.533 total

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 280000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
376801 records sent, 75360.2 records/sec (71.87 MB/sec), 3.9 ms avg latency, 528.0 ms max latency.
432367 records sent, 86456.1 records/sec (82.45 MB/sec), 3.6 ms avg latency, 15.0 ms max latency.
448301 records sent, 89624.4 records/sec (85.47 MB/sec), 3.5 ms avg latency, 14.0 ms max latency.
404658 records sent, 80915.4 records/sec (77.17 MB/sec), 3.8 ms avg latency, 25.0 ms max latency.
440208 records sent, 88006.4 records/sec (83.93 MB/sec), 3.5 ms avg latency, 14.0 ms max latency.
439012 records sent, 87749.8 records/sec (83.68 MB/sec), 3.5 ms avg latency, 14.0 ms max latency.
451263 records sent, 90198.5 records/sec (86.02 MB/sec), 3.4 ms avg latency, 15.0 ms max latency.
443775 records sent, 88684.1 records/sec (84.58 MB/sec), 3.4 ms avg latency, 14.0 ms max latency.
432138 records sent, 86358.5 records/sec (82.36 MB/sec), 3.5 ms avg latency, 15.0 ms max latency.
4000000 records sent, 86001.161016 records/sec (82.02 MB/sec), 3.55 ms avg latency, 528.00 ms max latency, 3 ms 50th, 6 ms 95th, 7 ms 99th, 12 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 47.906 total


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 300000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'



time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 300000 --num-records 4000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'
375426 records sent, 75085.2 records/sec (71.61 MB/sec), 3.9 ms avg latency, 511.0 ms max latency.
430106 records sent, 86021.2 records/sec (82.04 MB/sec), 3.6 ms avg latency, 18.0 ms max latency.
448163 records sent, 89596.8 records/sec (85.45 MB/sec), 3.4 ms avg latency, 13.0 ms max latency.
407336 records sent, 81467.2 records/sec (77.69 MB/sec), 3.9 ms avg latency, 80.0 ms max latency.
419486 records sent, 83897.2 records/sec (80.01 MB/sec), 3.6 ms avg latency, 16.0 ms max latency.
447539 records sent, 89472.0 records/sec (85.33 MB/sec), 3.4 ms avg latency, 14.0 ms max latency.
435787 records sent, 87140.0 records/sec (83.10 MB/sec), 3.5 ms avg latency, 20.0 ms max latency.
450821 records sent, 90110.1 records/sec (85.94 MB/sec), 3.4 ms avg latency, 14.0 ms max latency.
417415 records sent, 83466.3 records/sec (79.60 MB/sec), 3.7 ms avg latency, 28.0 ms max latency.
4000000 records sent, 85333.333333 records/sec (81.38 MB/sec), 3.60 ms avg latency, 511.00 ms max latency, 3 ms 50th, 6 ms 95th, 7 ms 99th, 15 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 48.255 total


time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 300000 --num-records 6000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=400000 compression.type=lz4 linger.ms=100'




time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 300000 --num-records 6000000 --topic demo-perf-topic2 --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=500000 compression.type=lz4 linger.ms=100'
390325 records sent, 78002.6 records/sec (74.39 MB/sec), 4.8 ms avg latency, 523.0 ms max latency.
451594 records sent, 90264.6 records/sec (86.08 MB/sec), 4.5 ms avg latency, 13.0 ms max latency.
455881 records sent, 91176.2 records/sec (86.95 MB/sec), 4.4 ms avg latency, 12.0 ms max latency.
172200 records sent, 34385.0 records/sec (32.79 MB/sec), 101.8 ms avg latency, 3073.0 ms max latency.
444202 records sent, 88840.4 records/sec (84.72 MB/sec), 154.1 ms avg latency, 3080.0 ms max latency.
454439 records sent, 90833.3 records/sec (86.63 MB/sec), 4.5 ms avg latency, 12.0 ms max latency.
348127 records sent, 69583.6 records/sec (66.36 MB/sec), 46.0 ms avg latency, 943.0 ms max latency.
435535 records sent, 87020.0 records/sec (82.99 MB/sec), 5.2 ms avg latency, 28.0 ms max latency.
443798 records sent, 86679.3 records/sec (82.66 MB/sec), 4.6 ms avg latency, 152.0 ms max latency.
424544 records sent, 84857.9 records/sec (80.93 MB/sec), 10.7 ms avg latency, 172.0 ms max latency.
456771 records sent, 91299.4 records/sec (87.07 MB/sec), 4.4 ms avg latency, 19.0 ms max latency.
362918 records sent, 72511.1 records/sec (69.15 MB/sec), 66.6 ms avg latency, 969.0 ms max latency.
452149 records sent, 90393.6 records/sec (86.21 MB/sec), 4.5 ms avg latency, 14.0 ms max latency.
439998 records sent, 83841.1 records/sec (79.96 MB/sec), 4.5 ms avg latency, 329.0 ms max latency.
6000000 records sent, 80183.888384 records/sec (76.47 MB/sec), 29.31 ms avg latency, 3080.00 ms max latency, 5 ms 50th, 12 ms 95th, 841 ms 99th, 2876 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 1:16.33 total

## 6m records in ~1m

ocker-compose exec broker1 kafka-topics --bootstrap-server broker1:9091 --topic demo-perf-topic --replication-factor 3 --partitions 24 --create --config min.insync.replicas=2
Created topic demo-perf-topic.
Can it be done?

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 300000 --num-records 6000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=500000 compression.type=lz4 linger.ms=100'

time docker exec -it broker1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 300000 --num-records 6000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=broker1:9091 acks=all batch.size=300000 compression.type=lz4 linger.ms=100'
345789 records sent, 69130.1 records/sec (65.93 MB/sec), 4.7 ms avg latency, 585.0 ms max latency.
350139 records sent, 70027.8 records/sec (66.78 MB/sec), 18.1 ms avg latency, 441.0 ms max latency.
261030 records sent, 52206.0 records/sec (49.79 MB/sec), 96.5 ms avg latency, 1519.0 ms max latency.
301978 records sent, 60383.5 records/sec (57.59 MB/sec), 27.7 ms avg latency, 609.0 ms max latency.
284211 records sent, 56090.6 records/sec (53.49 MB/sec), 52.7 ms avg latency, 835.0 ms max latency.
163491 records sent, 32574.4 records/sec (31.07 MB/sec), 732.1 ms avg latency, 3070.0 ms max latency.
325196 records sent, 62465.6 records/sec (59.57 MB/sec), 30.3 ms avg latency, 539.0 ms max latency.
233126 records sent, 45532.4 records/sec (43.42 MB/sec), 85.3 ms avg latency, 1201.0 ms max latency.
378480 records sent, 75680.9 records/sec (72.17 MB/sec), 8.2 ms avg latency, 241.0 ms max latency.
372723 records sent, 74470.1 records/sec (71.02 MB/sec), 8.5 ms avg latency, 190.0 ms max latency.
383272 records sent, 76654.4 records/sec (73.10 MB/sec), 8.2 ms avg latency, 234.0 ms max latency.
395884 records sent, 79161.0 records/sec (75.49 MB/sec), 4.4 ms avg latency, 44.0 ms max latency.
389196 records sent, 77823.6 records/sec (74.22 MB/sec), 5.0 ms avg latency, 73.0 ms max latency.
400524 records sent, 80072.8 records/sec (76.36 MB/sec), 4.2 ms avg latency, 12.0 ms max latency.
339804 records sent, 67920.0 records/sec (64.77 MB/sec), 88.8 ms avg latency, 961.0 ms max latency.
290588 records sent, 57966.9 records/sec (55.28 MB/sec), 57.4 ms avg latency, 543.0 ms max latency.
298009 records sent, 59578.0 records/sec (56.82 MB/sec), 48.1 ms avg latency, 1044.0 ms max latency.
348977 records sent, 69795.4 records/sec (66.56 MB/sec), 5.4 ms avg latency, 211.0 ms max latency.
6000000 records sent, 63826.392213 records/sec (60.87 MB/sec), 50.13 ms avg latency, 3070.00 ms max latency, 5 ms 50th, 268 ms 95th, 1029 ms 99th, 1938 ms 99.9th.
docker exec -it broker1 /bin/bash -c   0.06s user 0.04s system 0% cpu 1:36.34 total

----

spurious stuff below - to delete

docker exec -it kafka-1 /bin/bash -c 'kafka-topics --bootstrap-server kafka-1:9092 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'


docker exec -it broker1 /bin/bash -c 'kafka-topics --bootstrap-server broker1:9092 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'

```bash
docker exec -it kafka-1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput -1 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=kafka-1:9092 acks=all'

time docker exec -it kafka-1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=kafka-1:9092 acks=all linger.ms=100 batch.size=300000 --print-metrics'
```


broker1 kafka-cluster cluster-id --bootstrap-server broker1:9091