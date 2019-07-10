# yggdrasil
A service to helps us connect all of our services together. At least, locally.

---

## To Run

Make sure you have https://docs.docker.com/compose/ set up on your machine.

Run `docker-compose up`.

### Viewing Redis Status
- If you do not have redis-cli, run `brew install redis`. Alternatively, install redis-cli via your self-investigated method of choice.
- After Redis has launched, run `redis-cli ping`. You should get a response of PONG.

### Viewing Kafka Status
After logs are finished and services are up, navigate to http://localhost:8000 for a UI view of current Kafka brokers and topics.

## Constituent Services
### Redis
https://hub.docker.com/_/redis

https://redis.io/topics/rediscli

### Apache Kafka

#### General Kafka + Docker reading
https://jaceklaskowski.gitbooks.io/apache-kafka/kafka-docker.html

#### confluent/kafka-rest
https://docs.confluent.io/current/kafka-rest/quickstart.html#produce-and-consume-json-messages

#### landoop/kafka-topics-ui
https://github.com/Landoop/kafka-topics-ui

#### wurstmeister/kafka-docker

Dockerfile for [Apache Kafka](http://kafka.apache.org/)

The image is available directly from [Docker Hub](https://hub.docker.com/r/wurstmeister/kafka/)


##### Notes

- modify the ```KAFKA_ADVERTISED_HOST_NAME``` in [docker-compose.yml](https://raw.githubusercontent.com/wurstmeister/kafka-docker/master/docker-compose.yml) to match your docker host IP (Note: Do not use localhost or 127.0.0.1 as the host ip if you want to run multiple brokers.)
- if you want to customize any Kafka parameters, simply add them as environment variables in ```docker-compose.yml```, e.g. in order to increase the ```message.max.bytes``` parameter set the environment to ```KAFKA_MESSAGE_MAX_BYTES: 2000000```. To turn off automatic topic creation set ```KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'```
- Kafka's log4j usage can be customized by adding environment variables prefixed with ```LOG4J_```. These will be mapped to ```log4j.properties```. For example: ```LOG4J_LOGGER_KAFKA_AUTHORIZER_LOGGER=DEBUG, authorizerAppender```

**NOTE:** There are several 'gotchas' with configuring networking. If you are not sure about what the requirements are, please check out the [Connectivity Guide](https://github.com/wurstmeister/kafka-docker/wiki/Connectivity) in the [Wiki](https://github.com/wurstmeister/kafka-docker/wiki)

##### Usage

Add more brokers:

- ```docker-compose scale kafka=3```

**Note**

The default ```docker-compose.yml``` should be seen as a starting point. By default each broker will get a new port number and broker id on restart. Depending on your use case this might not be desirable. If you need to use specific ports and broker ids, modify the docker-compose configuration accordingly, e.g. [docker-compose-single-broker.yml](https://github.com/wurstmeister/kafka-docker/blob/master/docker-compose-single-broker.yml):

- ```docker-compose -f docker-compose-single-broker.yml up```

##### Broker IDs

You can configure the broker id in different ways

1. explicitly, using ```KAFKA_BROKER_ID```
2. via a command, using ```BROKER_ID_COMMAND```, e.g. ```BROKER_ID_COMMAND: "hostname | awk -F'-' '{print $$2}'"```

If you don't specify a broker id in your docker-compose file, it will automatically be generated (see [https://issues.apache.org/jira/browse/KAFKA-1070](https://issues.apache.org/jira/browse/KAFKA-1070). This allows scaling up and down. In this case it is recommended to use the ```--no-recreate``` option of docker-compose to ensure that containers are not re-created and thus keep their names and ids.


##### Automatically create topics

If you want to have kafka-docker automatically create topics in Kafka during
creation, a ```KAFKA_CREATE_TOPICS``` environment variable can be
added in ```docker-compose.yml```.

Here is an example snippet from ```docker-compose.yml```:

        environment:
          KAFKA_CREATE_TOPICS: "Topic1:1:3,Topic2:1:1:compact"

```Topic 1``` will have 1 partition and 3 replicas, ```Topic 2``` will have 1 partition, 1 replica and a `cleanup.policy` set to `compact`. Also, see FAQ: [Topic compaction does not work](https://github.com/wurstmeister/kafka-docker/wiki#topic-compaction-does-not-work)

If you wish to use multi-line YAML or some other delimiter between your topic definitions, override the default `,` separator by specifying the `KAFKA_CREATE_TOPICS_SEPARATOR` environment variable.

For example, `KAFKA_CREATE_TOPICS_SEPARATOR: "$$'\n'"` would use a newline to split the topic definitions. Syntax has to follow docker-compose escaping rules, and [ANSI-C](https://www.gnu.org/software/bash/manual/html_node/ANSI_002dC-Quoting.html) quoting.

##### Advertised hostname

You can configure the advertised hostname in different ways

1. explicitly, using ```KAFKA_ADVERTISED_HOST_NAME```
2. via a command, using ```HOSTNAME_COMMAND```, e.g. ```HOSTNAME_COMMAND: "route -n | awk '/UG[ \t]/{print $$2}'"```

When using commands, make sure you review the "Variable Substitution" section in [https://docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/)

If ```KAFKA_ADVERTISED_HOST_NAME``` is specified, it takes precedence over ```HOSTNAME_COMMAND```


##### Advertised port

If the required advertised port is not static, it may be necessary to determine this programatically. This can be done with the `PORT_COMMAND` environment variable.

```
PORT_COMMAND: "docker port $$(hostname) 9092/tcp | cut -d: -f2"
```

This can be then interpolated in any other `KAFKA_XXX` config using the `_{PORT_COMMAND}` string, i.e.

```
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://1.2.3.4:_{PORT_COMMAND}
```

##### Listener Configuration

It may be useful to have the [Kafka Documentation](https://kafka.apache.org/documentation/) open, to understand the various broker listener configuration options.

Since 0.9.0, Kafka has supported [multiple listener configurations](https://issues.apache.org/jira/browse/KAFKA-1809) for brokers to help support different protocols and discriminate between internal and external traffic. Later versions of Kafka have deprecated ```advertised.host.name``` and ```advertised.port```.

**NOTE:** ```advertised.host.name``` and ```advertised.port``` still work as expected, but should not be used if configuring the listeners.

##### Example

The example environment below:

```
HOSTNAME_COMMAND: curl http://169.254.169.254/latest/meta-data/public-hostname
KAFKA_ADVERTISED_LISTENERS: INSIDE://:9092,OUTSIDE://_{HOSTNAME_COMMAND}:9094
KAFKA_LISTENERS: INSIDE://:9092,OUTSIDE://:9094
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
KAFKA_INTER_BROKER_LISTENER_NAME: INSIDE
```

Will result in the following broker config:

```
advertised.listeners = OUTSIDE://ec2-xx-xx-xxx-xx.us-west-2.compute.amazonaws.com:9094,INSIDE://:9092
listeners = OUTSIDE://:9094,INSIDE://:9092
inter.broker.listener.name = INSIDE
```

##### Rules

* No listeners may share a port number.
* An advertised.listener must be present by protocol name and port number in the list of listeners.

##### Tutorial

[http://wurstmeister.github.io/kafka-docker/](http://wurstmeister.github.io/kafka-docker/)
