# Running Kafka with ECK, filebeat & logstash

_(WIP)_

- Follow stack helm deployment instructions to deploy elasticsearch & kibana
- helm add strimzi repo and install kafka operator
- add kafka cluster and create topics via manifest
- list kafka topics: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-topics.sh --list --bootstrap-server localhost:9092`
- check events inside a topic: `kubectl exec kafka-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic app1  --from-beginning`

### Flow
Filebeat (with autodiscovery and filtering by namespace) -> Kafka -> Logstash -> Elasticsearch
