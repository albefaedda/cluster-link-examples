# Cluster Link Hybrid Cloud CP to CC and viceversa

In this module I'm going to setup Cluster Link from Confluent Platform to Confluent Cloud and viceversa.
For the Confluent Platform to Confluent Cloud setup of Cluster Linking we are going to use source initiaded configuration which is recommended for Hybrid Cloud.

## Download Confluent Platform

Here I'm going to download the Confluent Platform and manually configure Kafka and ZooKeeper.

```sh 
cd /usr/local/
curl -O https://packages.confluent.io/archive/7.3/confluent-7.3.3.tar.gz

tar xzf confluent-7.3.3.tar.gz
```

Setup environment variables to simplify commands usage: 

```sh
export CONFLUENT_HOME=/usr/local/confluent-7.3.3
export CONFLUENT_CONFIG=/usr/local/configs
export PATH=$PATH:$CONFLUENT_HOME/bin
```

## Configure Kafka and ZooKeeper

Make a copy of `server.properties` and `zookeeper.properties` to use in this installation. 

```sh
cp server.properties $CONFLUENT_CONFIG/server-cluster-linking.properties
cp zookeeper.properties $CONFLUENT_CONFIG/zookeeper-cluster-linking.properties
```

Now let's update the configuration: 

In our zookeeper configuration file we want to modify the `data.dir`, we set this to `/tmp/zookeeper-clusterlinking`.

In our kafka configuration file, there are a few properties we need to modify for our installation: 

These must be added to the existing file:

```properties
inter.broker.listener.name=SASL_PLAINTEXT

sasl.enabled.mechanisms=SCRAM-SHA-512

sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

listener.name.sasl_plaintext.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="kafka" password="kafka-secret";

confluent.reporters.telemetry.auto.enable=false

confluent.cluster.link.enable=true

password.encoder.secret=encoder-secret
```

These are modifications or uses of existing configs:

```properties
listeners=SASL_PLAINTEXT://:9092

advertised.listeners=SASL_PLAINTEXT://:9092

log.dirs=/tmp/kafka-logs-1

zookeeper.connect=localhost:2181 (should already be set this way)

offsets.topic.replication.factor=1 (should already be set this way)

confluent.license.topic.replication.factor=1 (should already be set this way)
```

## Launch ZooKeeper and Kafka

After updating these parameters, we can launch ZooKeeper

```sh
zookeeper-server-start $CONFLUENT_CONFIG/zookeeper-cluster-linking.properties
```

Now that zookeeper is running, we can run the commands to create SASL SCRAM credentials on the cluster for two users: one to be used by the Kafka cluster, and the other for running commands against the cluster.

- Run this command to create credentials on the cluster for a user called “kafka” that will be used by the Kafka cluster itself.

```sh
kafka-configs --zookeeper localhost:2181 --alter --add-config \
  'SCRAM-SHA-512=[iterations=8192,password=kafka-secret]' \
  --entity-type users --entity-name kafka
```

- Run this command to create credentials on the cluster for a user called “admin” that you will use to run commands against this cluster.

```sh
kafka-configs --zookeeper localhost:2181 --alter --add-config \
  'SCRAM-SHA-512=[iterations=8192,password=admin-secret]' \
  --entity-type users --entity-name admin
```

Create a file with the admin credentials to authenticate when you run commands against the Confluent Platform cluster.

```properties
sasl.mechanism=SCRAM-SHA-512
security.protocol=SASL_PLAINTEXT
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin-secret";
```

I'll call this file `CP-command.config` and save it in the config folder $CONFLUENT_CONFIG.

Now we can launch Kafka passing the configuration we updated as a parameter to the script.

```sh
kafka-server-start $CONFLUENT_CONFIG/server-cluster-linking.properties
```

Get the cluster-id by running the following command: 

```sh
kafka-cluster cluster-id --bootstrap-server localhost:9092 --config CP-command.config
Cluster ID: 2ZGqhrlOTWeWbATfneDG_Q
```

---

## Confluent Cloud

For Cluster Linking to work, we need to create a Dedicated cluster with public endpoint in Confluent Cloud. 

Install the Confluent Cloud CLI using this command: 

```sh
curl -sL --http1.1 https://cnfl.io/cli | sh -s -- latest
```

Login with your email and password: 

```sh
confluent login
```

Select an environment, or create a new one, and create a dedicated cluster: 

```sh
confluent environment list
confluent environment use <my-env>

confluent kafka cluster create cl-destination-cluster --type dedicated --cloud aws --region eu-west-1 --cku 1 --availability single-zone
```

Now list and select your cluster for use: 

```sh
confluent kafka cluster list

 <your-kafka-cluster-id> | cl-destination-cluster | DEDICATED | aws      | eu-west-1 | single-zone  | UP  

confluent kafka cluster use <your-kafka-cluster-id>
```

We can also create an environment variable with the cluster-id to be used in the next commands: 

```sh
export CC_CLUSTER_ID=<your-kafka-cluster-id>
```

--- 

## Populate Confluent Platform cluster

Create a topic in the Confluent Platform with a single partition so that order is easier to see: 

```sh
kafka-topics --create --topic topic-on-cp --partitions 1 --replication-factor 1 --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config
```

You can get a list of topic in the Confluent Platform with this command: 

```sh
kafka-topics --list --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config
```

You can get your topic's configuration details with this command: 

```sh
kafka-topics --describe --topic topic-on-cp --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config

Topic: topic-on-cp	TopicId: YZSx1u6zRemCeRz8PCUIFA	PartitionCount: 1	ReplicationFactor: 1	Configs: 
	Topic: topic-on-cp	Partition: 0	Leader: 0	Replicas: 0	Isr: 0	Offline: 
```

With the following command you can send a sequence of numbers as messages to the topic you just created: 

```sh
seq 1 500000 | kafka-console-producer --topic topic-on-cp --bootstrap-server localhost:9092 --producer.config $CONFLUENT_CONFIG/CP-command.config
```

With the following command you can consume the messages you just produced to the topic: 

```sh
kafka-console-consumer --topic topic-on-cp --from-beginning --bootstrap-server localhost:9092 --consumer.config $CONFLUENT_CONFIG/CP-command.config
```

---

## Set up the privileges in Confluent Cloud

Create an API Key in Confluent Cloud. In this module I'll be creating a user API Key but it should be a Service Account API Key in a PROD environment. 
This is the API Key that the Cluster Linking will be using to migrate the data to Confluent Cloud

```sh
confluent api-key create --resource $CC_CLUSTER_ID

+------------+------------------------------------------------------------------+
| API Key    | KEY1KEY1KEY1KEY1                                                 |
| API Secret | SECRET1SECRET1SECRET1SECRET1SECRET1SECRET1SECRET1SECRET1SECRET1  |
+------------+------------------------------------------------------------------+
```

--- 

## Mirror data from Confluent Platform to Confluent Cloud

We are going to create a Cluster Link source initiated, from Confluent Platform to Confluent Cloud. 

To create this source initiated link, you must create both halves of the Cluster Link: the first half on Confluent Cloud, the second half on Confluent Platform.

Create a link configuration file for Confluent Cloud called `clusterlink-hybrid-dst.config` and containing: 

```properties
link.mode=DESTINATION
connection.mode=INBOUND
```

The combination of the configurations link.mode=DESTINATION and connection.mode=INBOUND tell the cluster link that it is the Destination half of a source initiated Cluster Link. These two configurations must be used together.

If you want to add any configurations to your Cluster Link (such as consumer offset sync or auto-create mirror topics) `clusterlink-hybrid-dst.config` is the file where you would add them. Cluster Link configurations are always set on the Destination Cluster Link (not the Source Cluster Link).

Create the Destination Cluster Link on Confluent Cloud.

```sh
confluent kafka link create from-on-prem-link --cluster $CC_CLUSTER_ID \
  --source-cluster $CP_CLUSTER_ID \
  --config-file $CONFLUENT_CONFIG/clusterlink-hybrid-dst.config

Created cluster link "from-on-prem-link".
```

You can list and describe the Cluster Links on Confluent Cloud with the following commands:

```sh
confluent kafka link list --cluster $CC_CLUSTER_ID

confluent kafka link configuration list 
```

Create security credentials for the Cluster Link on Confluent Platform. This security credential will be used to read topic data and metadata from the source cluster.

```sh
kafka-configs --bootstrap-server localhost:9092 --alter --add-config \
  'SCRAM-SHA-512=[iterations=8192,password=1LINK2RUL3TH3MALL]' \
  --entity-type users --entity-name cp-to-cloud-link \
  --command-config $CONFLUENT_CONFIG/CP-command.config

Completed updating config for user cp-to-cloud-link.
```

Create a link configuration file `$CONFLUENT_CONFIG/clusterlink-CP-src.config` for the source cluster link on Confluent Platform with the following entries:

```properties
link.mode=SOURCE
connection.mode=OUTBOUND

bootstrap.servers=<CC-BOOTSTRAP-SERVER>
ssl.endpoint.identification.algorithm=https
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<CC-link-api-key>' password='<CC-link-api-secret>';

local.listener.name=SASL_PLAINTEXT
local.security.protocol=SASL_PLAINTEXT
local.sasl.mechanism=SCRAM-SHA-512
local.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="cp-to-cloud-link" password="1LINK2RUL3TH3MALL";
```

- The combination of configurations `link.mode=SOURCE` and `connection.mode=OUTBOUND` tell the Cluster Link that it is the source-half of a source initiated Cluster Link. These configurations must be used together.
- The middle section tells the Cluster Cink the `bootstrap.servers` of the Confluent Cloud destination cluster for it to reach out to, and the authentication credentials to use. Cluster Linking to Confluent Cloud uses TLS and SASL_PLAIN. This is needed so that the Confluent Cloud cluster knows to accept the incoming request.
- The last section, where lines are prefixed with `local`, contains the security credentials to use with the source cluster (Confluent Platform) to read data.

We can now create the source Cluster Link on Confluent Platform, using the following command, specifying the configuration file from the previous step.

```sh
kafka-cluster-links --bootstrap-server localhost:9092 \
     --create --link from-on-prem-link \
     --config-file $CONFLUENT_CONFIG/clusterlink-CP-src.config \
     --cluster-id $CC_CLUSTER_ID --command-config $CONFLUENT_CONFIG/CP-command.config

Cluster link 'from-on-prem-link' creation successfully completed.
```

You can list Cluster Links on Confluent Platform with this command:

```sh
kafka-cluster-links --list --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config
```

---

## Create Mirror Topic and copy data to Confluent Cloud

The following command establishes a mirror of the original `topic-on-cp` topic, using the cluster link `from-on-prem-link`.

```sh
confluent kafka mirror create topic-on-cp --link from-on-prem-link

Created mirror topic "topic-on-cp".
```

- The mirror topic name must match the original topic name. 
- A mirror topic must specify the link to its source topic at creation time. This ensures that the mirror topic is a clean slate, with no conflicting data or metadata.

List the mirror topics on the link.

```sh
confluent kafka mirror list --cluster lkc-w5qdyg 

      Link Name     | Mirror Topic Name | Source Topic Name | Mirror Status | Status Time (ms) | Num Partition | Max Per Partition Mirror Lag  
--------------------+-------------------+-------------------+---------------+------------------+---------------+-------------------------------
  from-on-prem-link | topic-on-cp       | topic-on-cp       | ACTIVE        |                0 |             1 |    0
```

Consume from the mirror topic on the destination cluster to verify it.

On Confluent Cloud, run a consumer to consume messages from the mirror topic to consume the messages you originally produced to the Confluent Platform topic in previous steps.

```sh
confluent kafka topic consume topic-on-cp --from-beginning
```

---

## Mirror Data from Confluent Cloud to Confluent Platform

Let's create a Confluent Cloud to Confluent Platform Link. 

First of all, we need to create another API Key for this Link:
In this module I'll be creating a user API Key but it should be a Service Account API Key in a PROD environment. 
This is the API Key that the Cluster Linking will be using to migrate the data to Confluent Platform

```sh
confluent api-key create --resource $CC_CLUSTER_ID
```

+------------+------------------------------------------------------------------+
| API Key    | KEY2KEY2KEY2KEY2                                                 |
| API Secret | SECRET2SECRET2SECRET2SECRET2SECRET2SECRET2SECRET2SECRET2SECRET2  |
+------------+------------------------------------------------------------------+

Let's now create a configuration file called `clusterlink-cloud-to-CP.config` for the Cluster Link:

```properties
bootstrap.servers=<CC-BOOTSTRAP-SERVER>
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='KEY2KEY2KEY2KEY2' password='SECRET2SECRET2SECRET2SECRET2SECRET2SECRET2SECRET2SECRET2SECRET2';
```

Create the Cluster Link to Confluent Platform.
The following command creates the cluster link on an unsecured Confluent Platform cluster. If you have security set up on your Confluent Platform cluster, you must pass security credentials to this command with --command-config.

```sh
kafka-cluster-links --bootstrap-server localhost:9092 \
      --create --link from-cloud-link \
      --config-file $CONFLUENT_CONFIG/clusterlink-cloud-to-CP.config \
      --cluster-id $CC_CLUSTER_ID --command-config $CONFLUENT_CONFIG/CP-command.config

Cluster link 'from-cloud-link' creation successfully completed.
```

Check that the link exists with the `kafka-cluster-links --list` command, as follows.

```sh
kafka-cluster-links --list --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config

Link name: 'from-on-prem-link', link ID: 'wyCiaGEETWGdjTFwoJ_0CQ', remote cluster ID: 'lkc-w5qdyg', local cluster ID: '2ZGqhrlOTWeWbATfneDG_Q', remote cluster available: 'true'
Link name: 'from-cloud-link', link ID: 'AW5yrFN8RjyaMyRGV3i_lQ', remote cluster ID: 'lkc-w5qdyg', local cluster ID: '2ZGqhrlOTWeWbATfneDG_Q', remote cluster available: 'true'
```

Your output should show the previous `from-on-prem-link` you created along with the new `from-cloud-link`

Create topic in Confluent Cloud and Mirror Data to Confluent Platform:

```sh
confluent kafka topic create my-cloud-topic --partitions 1
Created topic "my-cloud-topic".
```

Start a producer to send some data into `my-cloud-topic`.

```sh
confluent kafka topic produce my-cloud-topic --cluster $CC_CLUSTER_ID
```

Write some messages, press ENTER between each message and CTRL-C when you're done.


Mirror the `my-cloud-topic` on Confluent Platform, using the command: 

```sh
kafka-mirrors --create --mirror-topic my-cloud-topic --link from-cloud-link --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config

Created topic my-cloud-topic.
```

On Confluent Platform, check the mirror topic status by running `kafka-mirrors --describe` on the `from-cloud-link`.

```sh
kafka-mirrors --describe --link from-cloud-link --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config

Topic: my-cloud-topic	LinkName: from-cloud-link	LinkId: AW5yrFN8RjyaMyRGV3i_lQ	SourceTopic: my-cloud-topic	State: ACTIVE	SourceTopicId: AAAAAAAAAAAAAAAAAAAAAA	StateTime: 2023-04-20 16:03:05
	Partition: 0	State: ACTIVE	DestLogEndOffset: 4	LastFetchSourceHighWatermark: 4	Lag: 0	TimeSinceLastFetchMs: 56848
```


Consume the data from the on-prem mirror topic.

```sh
kafka-console-consumer --topic my-cloud-topic --from-beginning --bootstrap-server localhost:9092 --consumer.config $CONFLUENT_CONFIG/CP-command.config
```

View the configuration of your cluster link:

```sh
kafka-configs --describe --cluster-link from-cloud-link --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config
```

Promote the mirror topics to normal topics.

On Confluent Cloud promote the mirror topic called `topic-on-cp`:

```sh
confluent kafka mirror promote topic-on-cp --link from-on-prem-link --cluster $CC_CLUSTER_ID

  Mirror Topic Name | Partition | Partition Mirror Lag | Last Source Fetch Offset | Error Message | Error Code  
--------------------+-----------+----------------------+--------------------------+---------------+-------------
  topic-on-cp       |         0 |                    0 |                   500000 |               |     
```

Rerunning the previous command, you get the following message:
`Topic 'topic-on-cp' has already stopped its mirror from 'from-on-prem-link'`

On Confluent Platform, promote the mirror topic called `my-cloud-topic`:

```sh
kafka-mirrors --promote --topics my-cloud-topic --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config

Calculating max offset and ms lag for mirror topics: [my-cloud-topic]
Finished calculating max offset lag and max lag ms for mirror topics: [my-cloud-topic]
Request for stopping topic my-cloud-topic's mirror was successfully scheduled. Please use the describe command with the --pending-stopped-only option to monitor progress.
```

Rerunning the previous command, you get the following message:
`Calculating max offset and ms lag for mirror topics: [my-cloud-topic]
Finished calculating max offset lag and max lag ms for mirror topics: [my-cloud-topic]. Error encountered while stopping topic my-cloud-topic's mirror: java.util.concurrent.ExecutionException: org.apache.kafka.common.errors.InvalidRequestException: Topic 'my-cloud-topic' has already stopped its mirror from 'from-cloud-link'`

## Delete the Cluster Links

### List Cluster Links on Confluent Platform

```sh
kafka-cluster-links --list --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config

Link name: 'from-on-prem-link', link ID: 'wyCiaGEETWGdjTFwoJ_0CQ', remote cluster ID: 'lkc-w5qdyg', local cluster ID: '2ZGqhrlOTWeWbATfneDG_Q', remote cluster available: 'true'

Link name: 'from-cloud-link', link ID: 'AW5yrFN8RjyaMyRGV3i_lQ', remote cluster ID: 'lkc-w5qdyg', local cluster ID: '2ZGqhrlOTWeWbATfneDG_Q', remote cluster available: 'true'
```

### Delete the Cluster Links in Confluent Platform

```sh
kafka-cluster-links --delete --link from-on-prem-link --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config
kafka-cluster-links --delete --link from-cloud-link --bootstrap-server localhost:9092 --command-config $CONFLUENT_CONFIG/CP-command.config
```

You will get confirmation that the links were deleted as output for each command.

### List Cluster Links in Confluent Cloud

```sh
confluent kafka link list

        Name        |     Source Cluster     | Destination Cluster | State  | Error | Error Message  
--------------------+------------------------+---------------------+--------+-------+----------------
  from-on-prem-link | 2ZGqhrlOTWeWbATfneDG_Q |                     | ACTIVE |       |               
```

### Delete the cluster links on Confluent Cloud

```sh
confluent kafka link delete from-on-prem-link

Are you sure you want to delete cluster link "from-on-prem-link"?
To confirm, type "from-on-prem-link". To cancel, press Ctrl-C: from-on-prem-link
Deleted cluster link "from-on-prem-link".
```

