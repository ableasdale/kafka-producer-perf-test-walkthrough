# `kafka-producer-perf-test` Walkthrough

Walkthrough project covering a tuning scenario using the `kafka-producer-perf-test` tool

Start the 3-node cluster by running:

```bash
docker-compose up
```

Sanity check the connection (this will fail right now):

```bash
docker exec -it broker1 /bin/bash -c 'kafka-topics --bootstrap-server broker1:9091 --describe --topic demo-perf-topic'
```


### Create the initial test topic for performance testing

```bash
docker exec -it broker1 /bin/bash -c 'kafka-topics --bootstrap-server broker1:9091 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'
```

If the operation has been successful, you should see:

```bash
Created topic demo-perf-topic.
```

To confirm, you can use `--describe`

```bash
docker exec -it broker1 /bin/bash -c 'kafka-topics --bootstrap-server broker1:9091 --describe --topic demo-perf-topic'
```

### Run a basic performance test

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

spurious stuff below - to delete

docker exec -it kafka-1 /bin/bash -c 'kafka-topics --bootstrap-server kafka-1:9092 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'


docker exec -it broker1 /bin/bash -c 'kafka-topics --bootstrap-server broker1:9092 --topic demo-perf-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2'

```bash
docker exec -it kafka-1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput -1 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=kafka-1:9092 acks=all'

time docker exec -it kafka-1 /bin/bash -c 'KAFKA_OPTS="" kafka-producer-perf-test --throughput 50000 --num-records 1000000 --topic demo-perf-topic --record-size 1000 --producer-props bootstrap.servers=kafka-1:9092 acks=all linger.ms=100 batch.size=300000 --print-metrics'
```