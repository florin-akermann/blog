### TLDR

- Kafka-connect is a low-code solution to move data in and out kafka.
- New data sources and sinks can be defined and spawned in minutes.
- Constraining your system to only use kafka-connect and kafka topics for data in- and outbound means you only integrate
  your business logic code with one API/Protocol: kafka and only one paradigm: event-driven.

### Kafka-connect to the Rescue

Kafka-connect is a great tool for moving data in and out of kafka without writing and maintaining a lot of code. The
kafka-connect ecosystem contains
over [120 different connectors](https://www.confluent.io/product/confluent-connectors/?utm_medium=sem&utm_source=google&utm_campaign=ch.sem_br.nonbrand_tp.prs_tgt.kafka-connectors_mt.xct_rgn.emea_lng.eng_dv.all_con.kafka-connect&utm_term=kafka%20connect&creative=&device=c&placement=&gclid=EAIaIQobChMI8si6yfm-8gIV0ACiAx0zwQsFEAAYASAAEgJeLvD_BwE) (JDBC, MQ, S3, File, etc...)
. So the chances are high that your datasource and datasink is among them. Our team recently experienced the value
of a low code solution at first hand.

Two weeks before 'go-live' we learned that upstream system X stopped publishing its trade relevant data in production.
They will publish again at some point as they are the golden source for this data (To be fair, they stopped a while ago:
We just did not notice...). Anyhow, we needed this data 'NOW' as our latest delivery depends on it. After some back and
forth we agreed on with upstream system X that we get to tap their applications datastore directly as a tactical
solution.

So far so good. It became quickly clear that we needed to regularly poll the upstream system's datastore. However, how
are we supposed to pipe this data into our system? We did not have any component readily available for this kind of
datastore. We were prepared to receive this data via public kafka topics. Will we be able to test and write a bug free
datastore poller in less than two weeks?
(Of course! ... and then production feeds us humble pie ...)

Luckily we were using kafka-connect sink-connectors already for a while to put data at the end of our kafka-streams into
oracle db. Therefore, it seemed like an obvious idea to use a source connector in addition to all the sink connectors we
had running.

Our standard data flow topology for all the public fpml topics is as follows:
Our xml-to-avro-kafka-streams application reads from the public-fpml-topic. It publishes a transformed version of the
message in avro to the internal-avro-topic. This internal-topic is used to enforce our data model by leveraging the
confluent schema registry. The kafka-connect-jdbc-sink-connector reads the shiny, validated messages on this topic and
persists them on our oracle db. Note that the only code we write is the kafka-streams app which is stateless aside from
the public topic consumer offset. Luckily the kafka-streams library takes care of that. Moreover, through the offset
handling kafka-streams apps provide at-least-once-delivery guarantee as a default.

```
 +--------------------+   +-----------------------------+   +----------v----------+  +---------------+   +--------------+
 |                    |   |                             |   |                     |  |               |   |              |
 | public-fpml-topic  +-->+-fpml-to-avro-transformation-+-->| internal-avro-topic-+->| kafka-connect +-->| oracle db    |
 |                    |   |   [kafka-streams-app]       |   |                     |  | [jdbc-sink]   |   |              |
 +--------------------+   +-----------------------------+   +---------------------+  +---------------+   +--------------+
```

Given the above topology, 'all' we had to do was piping this tactical source into our internal topic and align the data
with our data model. The final result looks like this.

```
+---------------------+   +---------------+  +------------------+  +---------------------+
|                     |   | kafka-connect |  | upstream-system  |  | transformer         |
| upstream datastore  +---> [jdbc-source] +--> data topic       +--> [kafka-streams-app  |
|                     |   |               |  |                  |  |                     |
+---------------------+   +---------------+  +------------------+  +---+-----------------+
                                                                       |
                                                                       |
 +--------------------+   +-----------------------------+   +----------v----------+  +---------------+   +--------------+
 |                    |   |                             |   |                     |  |               |   |              |
 | public-fpml-topic  +-->+-fpml-to-avro-transformation-+-->| internal-avro-topic-+->| kafka-connect +-->| oracle db    |
 |                    |   |   [kafka-streams-app]       |   |                     |  | [jdbc-sink]   |   |              |
 +--------------------+   +-----------------------------+   +---------------------+  +---------------+   +--------------+
```


We use the jdbc-source-connector to publish the upstreams systems data on a private kafka-avro-topic. A stateless
kafka-stream application is transforming and enriching the data from this topic and publishes it to our
internal-avro-topic.

Structuring the data flow topology has the following benefits:

- No code required for the polling of the 'upstream datastore'.
- This source connector is battle tested by kafka users all over the globe.
- The jdbc source-connector contains several out-of-the-box features regarding stateful poll logic (
  at-least-once-delivery-guarantee, etc.) and is highly configurable.
- We limit the coding needed to the kafka-streams applications, of which we have written already ~14 applications
  before. In other words, we code against familiar apis and stay within the event-driven paradigm for our applications.
- 'Fast' setup as the infrastructure was already in place. In theory, our operations team could have spawned this source
  connector directly in production with a single rest call.
- We can stop the tactical solution by a single rest call. We just delete the connector as
  described [in the rest api reference documentation](https://docs.confluent.io/platform/current/connect/references/restapi.html)
- Scale: Kafka-connect is designed to run on multiple instances and share the workload across these instances. The
  orchestration happens via kafka-connect internal topics.
- The chances of a new developer knowing kafka-connect from a previous job are considerably higher than a new developer
  knowing a homegrown solution.

All in all, kafka-connect seems to make our lives easier. We get fast, stable and scalable software out of the box. By
constraining ourselves to this tool we limit the number of APIs and protocols we need to master. A new datastore type,
source or sink, can be integrated easily without having deep knowledge about the data store in question. Albeit, good
understanding of the datastore evidently leads to a better connector configuration.

#### A shortened Example

This is a brief illustration of the recently add jdbc-source-connector and by no means intended to be a run-it-yourself
example. Assuming you have kafka-connect listening on port 8083 on your local machine you can do the following.

    curl localhost:8083/connectors/system-X-source/config -H "Content-Type: application/json" -d @dataBelow.json

With the following content in ``dataBelow.json``

```
{
  "connection.password": "${file:/path/to/props/on/host/secret.properties:connection.password}",
  "connection.url": "${file:/path/to/props/on/host/secret.properties:connection.url}",
  "connection.user": "${file:/path/to/props/on/host/secret.properties:connection.user}",
  "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
  "poll.interval.ms": 3600000,
  "topic.prefix": "sometopic",
  "mode": "timestamp",
  "timestamp.column.name": "SOME_TIME_STAMP_FIELD",
  "query": "select * from ...",
  "transforms": "createKey",
  "transforms.createKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
  "transforms.createKey.fields": "LEGACYID"
  ...
}
```

This json configuration is close to what we have running in production. The jdbc-source-connector polls every hour (
3600000 ms). It only loads updated rows based on the ``SOME_TIME_STAMP_FIELD`` of the table in the select and maintains
the state of this 'last poll time' itself. Moreover, as an example of all the configuration possibilities: we extract
the ``LEGACYID`` as the key for the kafka records. More options and features can be
found [here](https://docs.confluent.io/kafka-connect-jdbc/current/source-connector/index.html)

That's it. Nothing more is needed to poll this database and publish the updates to the target
topic.

#### PS:

- An even more powerful alternative than the jdbc-source-connector would be Debezium. Debezium is a change data capture
  tool made available as a kafka-connect plugin. It taps directly into the databases change log and can therefore react
  upon data changes in the source data store, rather than polling the datastore every x milliseconds. However, this
  requires certain database privileges.
- Performance: on our busiest topics we have seen the jdbc-sink instances to persist ~175k records per minute (upserts
  on a composite key).
- Docs: [Kafka-connect rest api reference](https://docs.confluent.io/platform/current/connect/references/restapi.html)
- [Here](https://florin-akermann.github.io/etl-kafka) is a bit dated and convoluted example on how to spin up things
  locally.
- Kafka-connect keeps its state in kafka topics. Its default operation-mode is 'distributed'. There are no peculiar requirements for a fancy file system. This makes it relatively easy to run it on kubernetes and friends.