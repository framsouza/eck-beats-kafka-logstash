# Complex architecture running ECK, Kafka, logstash & beats

On this page, you are going to learn how to configure beats on Kubernetes and use kafka + logstash to insert events on elasticsearch running on top of ECK.
Let assume you have a Kubernetes environment with `n` namespaces and there are 2 namespaces which you have to send the output to two different topics on Kafka. You also need separate pipelines to process these data and insert them into two differents indices on Elasticsearch.

### Scenario
- Kubernetes version 1.21 / ECK 1.9.1 / Elastic Stack 7.16.2 / Kafka 3.0.0
- Beats running as deployment to collect logs from 2 namespaces 
- Beats sending output logs to two different kafka topics depending on the namespace
- Kafka deployed by Strimzi (Kafka operator)
- Kafka running with 3 brokers
- 2 Kafka topics / app1 & app2 (both has 10 partitions)
- Logstash running multiples pipelines where which one connects to a different topic and send the events to elasticsearch
- Logstash proper filter configured to be able to have kafka metadata fields added to the events

### Deploying

- Follow [stack helm deployment](https://github.com/framsouza/eck-resources-with-helm-charts) instructions to deploy elasticsearch & kibana
- helm add [strimzi](https://strimzi.io/blog/2018/11/01/using-helm/) repo and install kafka operator
- Once kafka operator was installed, apply [kafka.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/kafka.yaml) & [app1-topic.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/app1-topic.yaml), [app2-topic.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/app2-topic.yaml). It will create a Kafka cluster with 3 brokers and 2 topics (10 partitions each)
- The [banana-app.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/banana-app.yaml) will run on ns `app1`, and [apple-app.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/apple-app.yaml) will run on ns `app2`, apply both manifests.
- Use a port-forward to be able to access theses applications via a browser, once you access it at this point you should have received messages in the kafka topics.
- Deploy beats, it will run a deployment because in this scenario there's no need to run it as daemonset. It's using an autodiscovery feature and also collects events from only 2 namespaces. 
- Take a look in the [configmap-logstash.yaml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/configmap-logstash.yml) this is how our multiple pipelines on logstash is configured. 
- Apply the configmap-logstash.yaml and right after that apply the [logstash.yml](https://github.com/framsouza/eck-beats-kafka-logstash/blob/main/logstash.yml) deployment


### Flow
Filebeat (with autodiscovery and filtering by namespace) -> Kafka <- Logstash -> Elasticsearch.

High level architecture

![Image from iOS](https://user-images.githubusercontent.com/16880741/147740761-1df66d41-6c84-49ea-ae96-3d6eb2d9894d.jpg)


## Settings to be considered
### Partition Strategy

Partition assignment defines how a client uses to distribute partition ownership amongst consumer instances. The default strategy is `range` which is ideal when you are consuming from *ONLY* one topic as per Kafka documentation: `The range assignor works on a per-topic basis. For each topic,….`.
In some cases when you are consuming from more than one topic, for instance `topic_pattern => app.*`, you may want to change this strategy to have a better distribution between partitions and consumer.

To check how the consumer clients are consuming the partitions, you can use the following command: `kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group logstash`

Whenever a consumer enters or leaves a consumer group, the brokers rebalance the partitions across consumers, meaning Kafka handles load balancing with respect to the number of partitions per application instance for you. By default, when a rebalance happens, all consumers drop their partitions and are reassigned new ones (which is called the “eager” protocol). To reduce this partition shuffling on stateful services, you can use the `sticky` strategy. This assignor makes some attempt to keep partition numbers assigned to the same instance, as long as they remain in the group, while still evenly distributing the partitions across members

It depends on your current configuration, but keep in mind the strategy it's best for your use case.

### Rebalance

There are cases where you may see a lot of rebalancing in your consumer. There are some scenarios where the rebalance is triggered:
1. When you change the number of partitions of a topic (add or remove)
2. When your kafka cluster is overloaded and can handle the current load
3. When you add (or remove) consumers into the consumer group
4. When a consumer is considered dead

To handle #4 you should investigate the heartbeats in your logstash kafka input, also `max_poll_interval` which is the amount of time the consumer can be idle before fetching more records, also `session_timeout_ms` is a good one to adjust in case you have to wait more before tagging the consumer as unavailable. 

### Consumer groups

The default is `consumer_thread = 1`. It means only one process will connect in every single partition, you might want to adjust it in order to improve the consumer performance. It's highly recommended to use the same number of partitions, which means each consumer group will connect in one partition to collect the data. Using the default configuration means that only 1 consumer will connect in ALL the partitions to collect the events.

### Events consuming
`max_poll_record` is responsible to collect the events from Kafka, by default it will collect 500 events by time. How much kafka fetches on each round trip will depend on `fetch_max_bytes`. If you adjust `max_poll_record` to collect more data (e.g 1000) means that logstash input waits longer for data before pushing it further down the pipeline. These adjustments will depend on the producer rate, but it's something you should evaluate and keep in mind.

### Reingest events already processed by consumer/backfill missing data
If for some reason you want to reinsert data into your cluster and notice missing data, first you need to make sure you have a good retention period in your topic, by default kafka use 7 days of the retention period. Once your data is in the kafka cluster, you can use logstash to collect these events.
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
Checking the data into elasticsearch, you will see 4 new fields were added: `topic, consumer_group, partition, offset`.
With that you have kafka input properly configured to be able to re-insert data. Next steps:

1. Create a new consumer_group listening to the same topic
2. Setting the `auto_offset_reset` to `earliest` to retrieve old data from topics
3. Configure the following filter to drop any duplicated data you might have
```
  if [@metadata][kafka][offset] < MIN_OFFSET or [@metadata][kafka][offset] > MAX_OFFSET {
    drop {}
  }
```

With that, you are checking the offset to check which one was the last one committed and reingest it from there.

### Useful commands for troubleshooting

- list kafka topics: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-topics.sh --list --bootstrap-server localhost:9092`
- check events inside a topic: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic app1  --from-beginning`
- check topics settings (retention period): `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic app`
- check consumer group list: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list`
- check consumer id by one consumer group: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group logstash`

Happy Kafka! 
