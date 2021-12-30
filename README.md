# Complex architecture running ECK, Kafka, logstash & beats
_(WIP)_

On this page, you are going to learn how to configure beats on Kubernetes and use kafka + logstash to insert events on elasticsearch running on top of ECK.
Let assume you have a Kubernetes environment with `n` namespaces and there are 2 namespaces which you have to send the output to two differents topic on Kafka. You also need separates pipelines to process these data and insert into two differents indices on Elasticsearch.

### Scenario
- Kubernetes version 1.21 / ECK 1.9.1 / Elastic Stack 7.16.2 / Kafka 3.0.0
- Beats running as a deployment to collect logs from 2 namespaces 
- Beats sending output logs to two different kafka topics depending on the namespace
- Kafka deployed by Strimzi (Kafka operator)
- Kafka running with 3 brokers
- 2 Kafka topics / app1 & app2 (both has 10 partitions)
- Logstash running multiples pipelines where which one connects to different topic and send the events to elasticsearch
- Logstash proper filter configured to be able to have kafka metadata fields added to the events

### Deploying

- Follow [stack helm deployment](https://github.com/framsouza/eck-resources-with-helm-charts) instructions to deploy elasticsearch & kibana
- helm add [strimzi](https://strimzi.io/blog/2018/11/01/using-helm/) repo and install kafka operator
- Once kafka operator was installed, apply [kafka.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/kafka.yaml) & [app1-topic.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/app1-topic.yaml), [app2-topic.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/app2-topic.yaml). It will create a Kafka cluster with 3 brokers and 2 topics (10 partitions each)
- The [banana-app.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/banana-app.yaml) will run on ns `app1`, and [apple-app.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/apple-app.yaml) will run on ns `app2`, apply both manifests.
- Use a port-forward to be able to access theses application via browser, once you access it at this point you should have received messages in the kafka topics.
- Deploy beats, it will run a deployment because on this scenario there's no needed to run it as daemonset. It's using a autodiscovery feature and also collecting events from only 2 namespaces. 
- Take a look in the [configmap-logstash.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/configmap-logstash.yml) this is how our multitiple pipeline on logstash is configured. 
- Apply the configmap-logstash.yaml and right after that apply the [logstash.yml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/logstash.yml) deployment


### Flow
Filebeat (with autodiscovery and filtering by namespace) -> Kafka <- Logstash -> Elasticsearch

## Settings to be considered
### Partition Strategy

Partition assingment define how client uses to distribute partition ownership amongst consumer instances. The default strategy is `range` which is ideally when you are consuming from *ONLY* one topic as per Kafka documentation: `The range assignor works on a per-topic basis.For each topic,….`.
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

### Consumer groups

Keep in mind on the examples above we are running 3 consumer_thread (logstash-0, logstash-1, logstash-2). The default is `consumer_thread = 1`. It means only one process will connect in every single partition, you might want to adjust it in order to improve the consumer performance. 

### Events consuming
`max_poll_record` is responsible to collect the events from Kafka, by default it will collect 500 events by time. You might want to increase it to get better collection, it will also depending on the producer rate, but it's something you should evaluate and keep in mind.

### Reingest events already processed by consumer / backfill missing data
If for some reason you want to reinsert data into your cluster and noticed missing data, first you need to make sure you have a good retention period in your topic, by default kafka use 7 days of retention period. Once your data in the kafka cluster, you can use logstash to collect these events.
To do so, before anything you need to have the following configuration in your kafka input `decorate_events => basic` it will add the metadata into the events. Then, you need to add the mutate filter into the game to be able to add these metadata into the output event:

```

    filter {
      mutate{
        add_field => { "[topic]" => "%{[@metadata][kafka][topic]}"}
        add_field => { "[consumer_group]" => "%{[@metadata][kafka][consumer_group]}"}
        add_field => { "[partition]" => "%{[@metadata][kafka][partition]"}
        add_field => { "[offset]" => "%{[@metadata][kafka][offset]}"}
      }
    }
```
Checking the data into elasticsearch, you will see 4 new fields was added: `topic, consumer_group, partition, offset`.
With that you have kafka input properly configured to be able to re-insert data. Next steps:

1. Create a new consumer_group listening to the same topic
2. Setting the `auto_offset_reset` to `earliest` to retrieve old data from topics
3. Configure the following filter to drop any duplicated data you might have
```
  if [@metadata][kafka][offset] < MIN_OFFSET or [@metadata][kafka][offset] > MAX_OFFSET {
    drop {}
  }
```

With that, you are checking the offset to check which one was the last one commited and reingest it from there.


### Useful commands to troubleshooting

- list kafka topics: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-topics.sh --list --bootstrap-server localhost:9092`
- check events inside a topic: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic app1  --from-beginning`
- check topics settings (retention period): `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic app`
- check consumer group list: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list`
- check consumer id by one consumer group: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group logstash`
