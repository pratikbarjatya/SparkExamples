# Spark - Kafka Example
Spark Practice and Execise

Words count through Kafka
# 1. Set up Kafka:
- For info on how to download & install Kafka please read here: https://kafka.apache.org/quickstart.
- Copy the default `config/server.properties` and `config/zookeeper.properties` configuration files from your downloaded kafka folder to a safe place.

In order to set up your kafka streams in your local machine make sure that your configuration files contain the following:
- **Broker config (server.properties)**
  - The id of the broker. This must be set to a unique integer for each broker.
    `broker.id=0`
  - Port that the socket server listens to
    `port=9092`
- **Zookeeper connection string (see zookeeper docs for details).**
  `zookeeper.connect=localhost:2181`
  `Zookeeper config (zookeeper.properties)`
- **SNAPSHOT directory**
  `dataDir=/tmp/zookeeper`
- **Port at which the clients will connect**
  `clientPort=2181`
- **disable the per-ip limit on the number of connections since this is a non-production config**
  `maxClientCnxns=0`

# 2. Run Kafka & send messages:
Having your configuration all set up you can go ahead and start the zookeeper server:
`$zookeeper-server-start PATH/TO/zookeeper.properties`
```
[2020–03–06 12:44:04,061] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
[2020–03–06 12:44:04,063] INFO autopurge.snapRetainCount set to 3 (org.apache.zookeeper.server.DatadirCleanupManager)
[2020–03–06 12:44:04,063] INFO autopurge.purgeInterval set to 0 (org.apache.zookeeper.server.DatadirCleanupManager)
```
Then start the kafka server:
```
$kafka-server-start PATH/TO/server.properties
[2020–03–06 12:44:56,353] INFO KafkaConfig values:
…
[2020–03–06 12:44:56,394] INFO starting (kafka.server.KafkaServer)
```
Now that the server are up and running let’s choose a topic and send a couple of message:
```
$echo “this is just a test” | kafka-console-producer — broker-list localhost:9092 — topic new_topic
```
Let’s check if everything went through by creating a simple consumer:
```
$kafka-console-consumer — zookeeper localhost:2181 — topic new_topic — from-beginning
```
this is just a test
Awesome, the messages are sent and received!

# 3. Spark Streaming
There are two approaches for integrating Spark with Kafka: Reciever-based and Direct (No Recievers). Please read more details on the architecture and pros/cons of using each one of them here.

Lets try both approaches with the common word-count problem on the kafka stream we just created.
## A - Reciever-based approach:
In the streaming application code, import KafkaUtils and create an input DStream calling the createStream function. Handle the returned stream as a normal RDD:
```
import sys
from pyspark import SparkContext, SparkConf
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils
from uuid import uuid1
if __name__ == “__main__”:
    sc = SparkContext(appName=”PythonStreamingRecieverKafkaWordCount”)
    ssc = StreamingContext(sc, 2) # 2 second window
    broker, topic = sys.argv[1:]
    kvs = KafkaUtils.createStream(ssc, \
                                  broker, \
                                  “raw-event-streaming-consumer”,\{topic:1}) 
    lines = kvs.map(lambda x: x[1])
    counts = lines.flatMap(lambda line: line.split(“ “)) 
                  .map(lambda word: (word, 1)) \
                  .reduceByKey(lambda a, b: a+b)
    counts.pprint()
    ssc.start()
    ssc.awaitTermination()
```

# B - Direct approach:
In the streaming application code, import KafkaUtils and create an input DStream calling the createDirectStream function. Handle the returned stream as a normal RDD:
```
import sys
from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils
if __name__ == “__main__”:
    sc = SparkContext(appName=”PythonStreamingDirectKafkaWordCount”)
    ssc = StreamingContext(sc, 2)
    brokers, topic = sys.argv[1:]
    kvs = KafkaUtils.createDirectStream(ssc, [topic],{“metadata.broker.list”: brokers})
    lines = kvs.map(lambda x: x[1])
    counts = lines.flatMap(lambda line: line.split(“ “)) \
                  .map(lambda word: (word, 1)) \
                  .reduceByKey(lambda a, b: a+b)
    counts.pprint()
    ssc.start()
    ssc.awaitTermination()
```
Run the above Spark applications as following:
```
$ bin/spark-submit — packages org.apache.spark:spark-streaming-kafka-0–8_2.11:2.0.0 spark-direct-kafka.py localhost:9092 new_topic
```
After sending one more message the output is:
```
— — — — — — — — — — — — — — — — — — — — — -
Time: 2020–03–06 13:04:10
 — — — — — — — — — — — — — — — — — — — — — -
(u’this’, 1)
(u’a’, 1)
(u’test’, 1)
(u’is’, 1)
(u’just’, 1)
```
