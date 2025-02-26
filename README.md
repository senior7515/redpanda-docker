# Redpanda in Docker

Try [Redpanda](https://github.com/vectorizedio/redpanda) in Docker.

## Build

```sh
make build
```

Beware: the build process is **very** resource intensive !

I tried very hard to enforce some reasonable limits, where "reasonable" is tailored for an 8 cores / 16GB laptop.

The main problem here is the inability to limit the first `cmake` workers, which are forced to the available number of cores by upstream, including in a runtime-fetched dependency ([Seastar](https://github.com/scylladb/seastar)) that can not be patched beforehand.

As these workers are in fact `cc1plus` instances, a program known for its frugality, spawning 8 of them is almost guaranteed to end with an OOM, unless you are doing nothing else with your computer.

`docker build` seems to happily ignore any cpu/memory restriction, leaving no choice but a two step build:
  - build a base image with all required dependencies
  - spawn a builder container (with constrained resources thanks to [LXCFS](https://github.com/lxc/lxcfs), if available) to do the heavy work itself, and commit the resulting container as our Redpanda image

Of course none of this should be necessary if upstream did not enforce `-j$(nproc)` without any override possibility (see `redpanda/cmake/oss.cmake.in` and `seastar/cooking_recipe.cmake` for details, search for `build_concurrency_factor`). Docker could also include LXCFS stuff to have reliable resource limits, it would be nice.

## Run

```sh
make run
```

Redpanda should be accessible from ports `19092` (kafka) and `19644` (admin).

This command will show Redpanda logs, use `^C` to quit.

Beware: data will **not** be persisted, but you may use a volume if needed.

## Usage

This assume standard Kafka tools (tested with `net-misc/kafka-bin-2.6.0`).

### Tools

```sh
$ ./kafka-acls.sh --bootstrap-server localhost:19092 --list
Error while executing ACL command: org.apache.kafka.common.errors.UnsupportedVersionException: The broker does not support DESCRIBE_ACLS

$ ./kafka-topics.sh --bootstrap-server localhost:19092 --topic test --create --partitions 1 --replication-factor 1
Created topic test.

$ ./kafka-topics.sh --bootstrap-server localhost:19092 --list
test

$ ./kafka-topics.sh --bootstrap-server localhost:19092 --topic test --describe
Topic: test    PartitionCount: 1    ReplicationFactor: 1    Configs: partition_count=1,replication_factor=1
    Topic: test    Partition: 0    Leader: 1    Replicas: 1    Isr: 1
```

### Consumer

```sh
$ ./kafka-console-consumer.sh --bootstrap-server localhost:19092 --topic test
# no output yet, but should show producer messages
```

### Producer

```sh
$ ./kafka-console-producer.sh --bootstrap-server localhost:19092 --topic test --compression-codec snappy
>hello there !
>^D
```

## Bugs

I lost messages when the topic has more than one partition, e.g:

```sh
$ ./kafka-topics.sh --bootstrap-server localhost:19092 --topic test2 --create --partitions 4 --replication-factor 1
Created topic test2.

$ ./kafka-topics.sh --bootstrap-server localhost:19092 --topic test2 --describe
Topic: test2    PartitionCount: 4    ReplicationFactor: 1    Configs: partition_count=4,replication_factor=1
    Topic: test2    Partition: 0    Leader: 1    Replicas: 1    Isr: 1
    Topic: test2    Partition: 1    Leader: 1    Replicas: 1    Isr: 1
    Topic: test2    Partition: 2    Leader: 1    Replicas: 1    Isr: 1
    Topic: test2    Partition: 3    Leader: 1    Replicas: 1    Isr: 1

$ ./kafka-console-consumer.sh --bootstrap-server localhost:19092 --topic test2

$ dmesg | ./kafka-console-producer.sh --bootstrap-server localhost:19092 --topic test2 --compression-codec snappy; dmesg | wc -l
1482

# then ^C the consumer, its processed messages should match dmesg lines
^CProcessed a total of 1191 messages
# oops !
```

This happens only with more than one partition, but never with a real Kafka.

It's worth noting that when I start `kafka-console-consumer.sh` with `--from-beginning`, I get all the messages:
  - 1st run: processed 1570 messages, OK
  - 2nd run: processed 1570 messages, OK
  - 3rd run: processed 1346 messages, oops !
  - restart with `--from-beginning`: processed 4710 messages, OK (= 1570 * 3)

May be worth trying to reproduce with something more advanced, like [aiokafka](https://github.com/aio-libs/aiokafka).
