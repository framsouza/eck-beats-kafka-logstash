# Running Kafka with ECK, filebeat & logstash

_(WIP)_

- Follow stack helm deployment instructions to deploy elasticsearch & kibana
- helm add strimzi repo and install kafka operator
- add kafka cluster and create topics via manifest
- list kafka topics: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-topics.sh --list --bootstrap-server localhost:9092`
- check events inside a topic: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic app1  --from-beginning`
- check topics settings: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic app`
- check consumer group list: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list`
- check consumer id by one consumer group: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group logstash`

### Flow
Filebeat (with autodiscovery and filtering by namespace) -> Kafka -> Logstash -> Elasticsearch

## Partition Strategy

Partition assingment define how client uses to distribute partition ownership amongst consumer instances. The default strategy is `range` which is ideally when you are consuming from *ONLY* one topic. 
In some case when you are consuming from more than one topic, for instance `topic_pattern => app.*`, you may want to change this strategy to have a better distribution between partitions and consumer. Check the following scenario:

2 Topics:
- app1 has 8 partitions
- app2 has 2 partitions

1 Consumer group running with 3 consumer_thread and 1 logstash intance. Using the default strategy, the distribution will be the following:

```
GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
logstash        app1            0          39              39              0               logstash-0-43df8a06-b36d-42da-976b-f0493571e46b /10.0.1.71      logstash-0
logstash        app2            0          10              10              0               logstash-0-43df8a06-b36d-42da-976b-f0493571e46b /10.0.1.71      logstash-0
logstash        app1            1          13              13              0               logstash-0-43df8a06-b36d-42da-976b-f0493571e46b /10.0.1.71      logstash-0
logstash        app1            2          8               8               0               logstash-0-43df8a06-b36d-42da-976b-f0493571e46b /10.0.1.71      logstash-0
logstash        app2            1          7               7               0               logstash-1-32199444-c97d-4b49-9c65-d7400e0c628a /10.0.1.71      logstash-1
logstash        app1            3          3               3               0               logstash-1-32199444-c97d-4b49-9c65-d7400e0c628a /10.0.1.71      logstash-1
logstash        app1            4          8               8               0               logstash-1-32199444-c97d-4b49-9c65-d7400e0c628a /10.0.1.71      logstash-1
logstash        app1            5          9               9               0               logstash-1-32199444-c97d-4b49-9c65-d7400e0c628a /10.0.1.71      logstash-1
logstash        app1            6          5               5               0               logstash-2-776a1d2f-5e39-4e86-a0d1-6a4c04e9509c /10.0.1.71      logstash-2
```
- logstash 0 has 4 partitions assigned
- logstash 1 has 4 partitions assigned
- logstash 2 has 1 partitions assigned

Changing partition strategy to use `round_robin`:
```
GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
logstash        app3            0          0               0               0               logstash-1-89c80e91-2382-4675-8129-147cc5f5e3fb /10.0.1.72      logstash-1
logstash        app1            1          13              13              0               logstash-1-89c80e91-2382-4675-8129-147cc5f5e3fb /10.0.1.72      logstash-1
logstash        app1            4          8               8               0               logstash-1-89c80e91-2382-4675-8129-147cc5f5e3fb /10.0.1.72      logstash-1
logstash        app1            7          12              12              0               logstash-1-89c80e91-2382-4675-8129-147cc5f5e3fb /10.0.1.72      logstash-1
logstash        app1            0          39              39              0               logstash-0-42091b96-43a7-41e0-ac7f-709067b560ab /10.0.1.72      logstash-0
logstash        app2            1          7               7               0               logstash-0-42091b96-43a7-41e0-ac7f-709067b560ab /10.0.1.72      logstash-0
logstash        app1            3          3               3               0               logstash-0-42091b96-43a7-41e0-ac7f-709067b560ab /10.0.1.72      logstash-0
logstash        app1            6          5               5               0               logstash-0-42091b96-43a7-41e0-ac7f-709067b560ab /10.0.1.72      logstash-0
logstash        app2            0          10              10              0               logstash-2-96de7e2f-1827-4676-9b93-e0a3a291d2fe /10.0.1.72      logstash-2
logstash        app1            2          8               8               0               logstash-2-96de7e2f-1827-4676-9b93-e0a3a291d2fe /10.0.1.72      logstash-2
logstash        app1            5          9               9               0               logstash-2-96de7e2f-1827-4676-9b93-e0a3a291d2fe /10.0.1.72      logstash-2
```
- logstash 0 has 4 partitions assigned
- logstash 1 has 4 partitions assigned
- logstash 2 has 3 partitions assigned

Keep in mind on the examples above we are running 3 consumer_thread (logstash-0, logstash-1, logstash-2). The default is consumer_thread = 1. It means only one process will connect in every single partition, you might want to adjust it in order to improve the consumer performance. 
