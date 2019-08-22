# Apache Sling Journal based Content Distribution - Kafka Demo

This modules demonstrates [Apache Sling Journal Based Distribution](https://github.com/apache/sling-org-apache-sling-distribution-journal) on [Apache Kafka](https://kafka.apache.org).
By going through the instructions, you will setup a basic Kafka deployment as well as Sling instances configured with Journal based distribution.

# Setup

## Apache Kafka

### Download

The following commands will download Kafka and extract it locally in the `kafka_2.12-2.1.0` folder.

```
curl  http://archive.apache.org/dist/kafka/2.1.0/kafka_2.12-2.1.0.tgz | tar xz
```

### Start

From the `kafka_2.12-2.1.0` folder run the commands

```
bin/zookeeper-server-start.sh config/zookeeper.properties &
bin/kafka-server-start.sh config/server.properties &
```

### Create topics

From the `kafka_2.12-2.1.0` folder, once your Kafka setup is started, create the topics with the following commands

```
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic aemdistribution_package
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic aemdistribution_discovery
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic aemdistribution_command
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic aemdistribution_status
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic aemdistribution_event
```


## Run author and publish

The project is configured to start an `author` instance on port `9090` and a `publish` publish instance on port `9091`. Make sure that Apache Maven is available on your setup and if needed [install Apache Maven](https://maven.apache.org/install.html). Clone this project on your local file system, then run the following command

```
mvn clean install
```

The [slingstart-maven-plugin](https://sling.apache.org/components/slingstart-maven-plugin/repository-mojo.html) will take care of starting the Sling instances. Have a look at the [provisioning model files](src/main/provisioning) to see what OSGi bundles and configurations are used.

# Play

Once you have done the setup, you should be able to play with it. The setup has created a `forwardPublisher` Pub agent on `author` and a `forwardSubscriber` Sub agent on `publish`.



## List the `forwardPublisher` agent queues

You can get the list of queues managed by the Pub agents by running the command

```
curl -u admin:admin http://localhost:9090/libs/sling/distribution/services/agents/forwardPublisher/queues.json
```

The `forwardSubscriber` should automatically subscribe to the `forwardPublisher` and appear as a queue in the response as shown below

```json
{"sling:resourceType":"sling/distribution/service/agent/queue/list","items":["bd1f9e8c-b5c8-4f68-90d4-5eb9e4911ada-forwardSubscriber"]}
```

## Get an `forwardPublisher` agent queue status

You can get the status of a queue managed by the Pub agent by running the command similar to the following (for the queue `bd1f9e8c-b5c8-4f68-90d4-5eb9e4911ada-forwardSubscriber`)

```
curl -u admin:admin http://localhost:9090/libs/sling/distribution/services/agents/forwardPublisher/queues/bd1f9e8c-b5c8-4f68-90d4-5eb9e4911ada-forwardSubscriber.json

```

When the queue contains items, you'll get a response similar to the one below

```json
{"capabilities":["removable","clearable"],"sling:resourceType":"sling/distribution/service/agent/queue","state":"RUNNING","items":["aemdistribution_package-0@67"],"itemsCount":"1","empty":false}
``` 

## Distribute content

First, create some test content on `author` using the [Apache Sling POST servlet](https://sling.apache.org/documentation/bundles/manipulating-content-the-slingpostservlet-servlets-post.html). The command below creates a resource `test` under `/content/test`.

```
curl -u admin:admin -F"sling:resourceType=some/resource" -F"title=Test" http://localhost:9090/content/test
```

Then distribute this test content using the `forwardPublisher` agent

```
curl -v -u admin:admin http://localhost:9090/libs/sling/distribution/services/agents/forwardPublisher -d 'action=ADD' -d 'path=/content/test'
```

Finally check on the `publish` instance that the test content has been distributed.

```
curl -u admin:admin http://localhost:9091/content/test.json
```

which should repond with a `200` HTTP status and the content

```json
{"jcr:primaryType":"sling:OrderedFolder","jcr:createdBy":"","jcr:created":"Thu Aug 22 2019 16:45:12 GMT+0200","title":"Test","sling:resourceType":"some/resource"}
```
