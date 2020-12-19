<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Spark Architecture](#spark-architecture)
  - [Basic Property](#basic-property)
- [Spark Performance](#spark-performance)
  - [Memory and GC](#memory-and-gc)
    - [Memory Management](#memory-management)
    - [GC Tuning](#gc-tuning)
    - [Other Tuning](#other-tuning)
- [Spark Component](#spark-component)
  - [Data Input](#data-input)
    - [Manage Input Schema](#manage-input-schema)
    - [Kafka Dstream](#kafka-dstream)
      - [Manage Offset](#manage-offset)
      - [Manage Offset in Data Store](#manage-offset-in-data-store)
  - [Data output](#data-output)
    - [Client Factory](#client-factory)
    - [Data Sender](#data-sender)
  - [Utils](#utils)
    - [Log](#log)
    - [Reflect](#reflect)
  - [Cache](#cache)
    - [Guava Cache](#guava-cache)
  - [Data Store](#data-store)
    - [Hbase](#hbase)
  - [Config](#config)
    - [Type Safe Config](#type-safe-config)
      - [Example](#example)
      - [Loading from ConfigFactory](#loading-from-configfactory)
      - [Loading with FallBack](#loading-with-fallback)
      - [Loading String](#loading-string)
      - [Config Value Resolution](#config-value-resolution)
      - [Duration/Memory/List/Boolean Helper](#durationmemorylistboolean-helper)
    - [Full TypeSafe Config Example](#full-typesafe-config-example)
    - [DB Connection Pool via TypeSafe Config](#db-connection-pool-via-typesafe-config)
- [Spark Jobs](#spark-jobs)
  - [Spark Stream vs Structure Stream](#spark-stream-vs-structure-stream)
    - [Spark RDD(DStream) Streaming](#spark-rdddstream-streaming)
    - [Structured Streaming](#structured-streaming)
    - [Distinctions](#distinctions)
      - [Real Streaming](#real-streaming)
      - [RDD vs. DataFrames/DataSet](#rdd-vs-dataframesdataset)
      - [Processing With the Vent Time, Handling Late Data](#processing-with-the-vent-time-handling-late-data)
      - [End-to-End Guarantees](#end-to-end-guarantees)
      - [Restricted or Flexible](#restricted-or-flexible)
      - [Throughput/Concurrency](#throughputconcurrency)
      - [Conclusion](#conclusion)
  - [Dstream(RDD) Job](#dstreamrdd-job)
    - [Dstream Generator](#dstream-generator)
    - [Dstream Job](#dstream-job)
  - [Structure Stream Job](#structure-stream-job)
    - [StreamingQueryListener](#streamingquerylistener)
      - [onQueryStarted](#onquerystarted)
      - [onQueryProgress](#onqueryprogress)
      - [onQueryTerminated](#onqueryterminated)
    - [Base Job](#base-job)
    - [Structure Stream Job](#structure-stream-job-1)
      - [Convert from Dataset<Row> to Dataset<T>](#convert-from-datasetrow-to-datasett)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# Spark Architecture

## Basic Property

https://spark.apache.org/docs/latest/configuration.html#spark-properties

# Spark Performance

## Memory and GC

During Spark run time,  Yarn will monitor Spark memory usage and kill the task if its memory footprint crosses a threshold. This will cause temporary executor lost failure. With existing executor gone, micro batches have to be re-processed in reduced executor resources.   Also all in memory cache needs to re-built which takes lot of time.  These added up failures cause significant delay.

So it is very important to manage memory correctly

https://www.tutorialdocs.com/article/spark-memory-management.html

### Memory Management

Executor acts as a JVM process, and its memory management is based on the JVM. So JVM memory management includes two methods:

* On-Heap memory management: 

Objects are allocated on the JVM heap and bound by GC.

* Off-Heap memory management: 

Objects are allocated in memory outside the JVM by serialization, managed by the application, and are not bound by GC. This memory management method can avoid frequent GC, but the disadvantage is that you have to write the logic of memory allocation and memory release.

1. Storage Memory: It's mainly used to store Spark cache data, such as RDD cache, Broadcast variable, Unroll data, and so on.
2. Execution Memory: It's mainly used to store temporary data in the calculation process of Shuffle, Join, Sort, Aggregation, etc.
3. User Memory: It's mainly used to store the data needed for RDD conversion operations, such as the information for RDD dependency.
4. Reserved Memory: The memory is reserved for system and is used to store Spark's internal objects.


![spark_memory_1](https://github.com/zhangruiskyline/system/blob/main/images/spark_memory_1.png)
![spark_memory_2](https://github.com/zhangruiskyline/system/blob/main/images/spark_memory_2.png)

* Balance

- ```spark.memory.fraction```

 expresses the size of M as a fraction of the (JVM heap space - 300MiB) (default 0.6). The rest of the space (40%) is reserved for user data structures, internal metadata in Spark, and safeguarding against OOM errors in the case of sparse and unusually large records.
- ```spark.memory.storageFraction```

 expresses the size of R as a fraction of M (default 0.5). R is the storage space within M where cached blocks immune to being evicted by execution.

If we have too high storage faction, application memory may have pressure like  cache, but if we have too low storage fraction, like broadcast will be impacted



### GC Tuning

https://databricks.com/blog/2015/05/28/tuning-java-garbage-collection-for-spark-applications.html

GC Tunning Parameters:

https://www.oracle.com/technical-resources/articles/java/g1gc.html#:~:text=The%20default%20value%20is%2060%20percent%20of%20your%20Java%20heap.&text=XX%3ADefaultMaxNewGenPercent%20setting.-,This%20setting%20is%20not%20available%20in%20Java%20HotSpot%20VM%2C%20build,of%20the%20STW%20worker%20threads

For the long GC time, maybe we could try to reduce the ```ParallelGCThreads``` setting from default(23) to a lower number. Since there are multiple JVMs running on one machine and with less thread, GCs of different JVMs won't interfere with each other. I'm running a job with ```ParallelGCThreads=8``` and it has much less GC time(about 10%)

### Other Tuning

Spark automatically monitors cache usage on each node and drops out old data partitions in a least-recently-used (LRU) fashion. If you would like to manually remove an RDD instead of waiting for it to fall out of the cache, use the ```RDD.unpersist()``` method.

1. Storage fraction: too low, broadcast(model) will have issue since Spark internal memory will be not sufficient,  too High, then application has OOM risk like our cache usage will be high

2. Concurrency: too low , then our processing delay could be accumulated, but too high then OOM risk







# Spark Component


## Data Input


### Manage Input Schema

- Base class to implement Interface as

```JAVA
package org.apache.kafka.common.serialization;

import java.io.Closeable;
import java.util.Map;

public interface Deserializer<T> extends Closeable {
    void configure(Map<String, ?> var1, boolean var2);

    T deserialize(String var1, byte[] var2);

    void close();
}
```

```JAVA
public interface MyStreamGen<T extends BaseEventClass> {

    JavaDStream<T> fetchStreams(Config myConfig, JavaStreamingContext jssc);

}

```
- Example of Stream Gen 

```JAVA
public JavaDStream<T> fetchStreams(Config myConfig, JavaStreamingContext jssc) {

    MyDataInput myConf = getConf();

    JavaInputDStream<MyDataType> inputStream = MyDataInputUtils.createDirectStream(jssc, myConf);

    storeOffset(inputStream, myConf);

    Deserializer<T[]> myDeserializer = getDeserializer();

    JavaDStream<T[]> contentEventStream = inputStream
            .map(t -> myDeserializer.deserialize(null, t.getBytes()))
            .filter(t -> t != null && t.length > 0);
    JavaDStream<T> contentEventDStream = contentEventStream.flatMap(t -> Arrays.asList(t).iterator());
    return contentEventDStream;
}
```


### Kafka Dstream

https://spark.apache.org/docs/1.3.1/streaming-kafka-integration.html

```JAVA
import java.util.*;
import org.apache.spark.SparkConf;
import org.apache.spark.TaskContext;
import org.apache.spark.api.java.*;
import org.apache.spark.api.java.function.*;
import org.apache.spark.streaming.api.java.*;
import org.apache.spark.streaming.kafka010.*;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;
import scala.Tuple2;


private JavaInputDStream<ConsumerRecord<byte[], T[]>> createLogDstream(TopicAndMetadata topicMetadata, JavaStreamingContext jssc, Map<String, Object> kafkaParams) {

	JavaInputDStream<ConsumerRecord<byte[], T[]>> stream;
	Map<String, Object> kafkaParams = new HashMap<>();
	kafkaParams.put("bootstrap.servers", "localhost:9092,anotherhost:9092");
	kafkaParams.put("key.deserializer", StringDeserializer.class);
	kafkaParams.put("value.deserializer", getDeserializerClass());
	kafkaParams.put("group.id", "use_a_separate_group_id_for_each_stream");
	kafkaParams.put("auto.offset.reset", "latest");
	kafkaParams.put("enable.auto.commit", false);

	Collection<String> topics = Arrays.asList("topicA", "topicB");

	stream =
	  KafkaUtils.createDirectStream(
	    streamingContext,
	    LocationStrategies.PreferConsistent(),
	    ConsumerStrategies.Subscribe(topics, kafkaParams)
	  );

	//stream.mapToPair(record -> new Tuple2<>(record.key(), record.value()));

	return stream
}
```



```JAVA
public abstract Class<V> getDeserializerClass();

//V is java.io.Serializable
```

We can ramp this function into 

```JAVA
@Override
public JavaDStream<T> fetchStreams(TopicAndMetadata topicMetadata, JavaStreamingContext jssc) {
    Map<String, Object> kafkaParams = TopicsSchema.createKafkaParams(topicMetadata);
    logger.info(kafkaParams);
    logger.info(System.getProperties());
    kafkaParams.put("key.deserializer", ByteArrayDeserializer.class);
    kafkaParams.put("value.deserializer", getDeserializerClass());

    JavaInputDStream<ConsumerRecord<byte[], T[]>> dStream = createLogDstream(topicMetadata, jssc, kafkaParams);
    JavaDStream<T> dStream =
            logDstream.filter(t -> t.value() != null && t.value().length > 0)
                    .flatMap(t -> Arrays.asList(t.value()).iterator());

    return dStream;
}
```


#### Manage Offset

https://blog.cloudera.com/offset-management-for-apache-kafka-with-apache-spark-streaming/

#### Manage Offset in Data Store

https://spark.apache.org/docs/2.0.2/streaming-kafka-0-10-integration.html#storing-offsets

```JAVA
// The details depend on your data store, but the general idea looks like this

// begin from the the offsets committed to the database
Map<TopicPartition, Long> fromOffsets = new HashMap<>();
for (resultSet : selectOffsetsFromYourDatabase)
  fromOffsets.put(new TopicPartition(resultSet.string("topic"), resultSet.int("partition")), resultSet.long("offset"));
}

//Retrieve Offset 

JavaInputDStream<ConsumerRecord<String, String>> stream = KafkaUtils.createDirectStream(
  streamingContext,
  LocationStrategies.PreferConsistent(),
  ConsumerStrategies.<String, String>Assign(fromOffsets.keySet(), kafkaParams, fromOffsets)
);

stream.foreachRDD(new VoidFunction<JavaRDD<ConsumerRecord<String, String>>>() {
  @Override
  public void call(JavaRDD<ConsumerRecord<String, String>> rdd) {
    OffsetRange[] offsetRanges = ((HasOffsetRanges) rdd.rdd()).offsetRanges();
    
    Object results = yourCalculation(rdd);

    // begin your transaction

    // update results
    // update offsets where the end of existing offsets matches the beginning of this batch of offsets
    // assert that offsets were updated correctly

    // end your transaction
  }
});
```





## Data output

- We should use factory design pattern to get Client to connect to remote lib
- Use ```getOrCreate``` 

### Client Factory

- K/V Client

```JAVA
public class MyClientFactory {

    public static ConcurrentMap<String, MyClient> clients = new ConcurrentHashMap<>();

    public static <K extends Ktype, V extends Vtype> MyClient<K, V> getOrCreate(Config myConfig){
    	//Execute 
    	return (myClient<K, V>) clients.computeIfAbsent(generateKey()), (key) -> {
            myClient<K, V> client = null;
            try {
                client = new myClient<>(myConfig);
            } catch (Exception e) {
                throw new RuntimeException("Failed to create my client", e);
            }
            return client;
        });
    }

}
```

- Kafka Client

```JAVA
public class KafkaProducerFactory {
    private static volatile Map<String, Producer> producers = new ConcurrentHashMap<>();
    private static String BOOTSTRAP_SERVERS_KEY = "bootstrap.servers";

    public static <K, V> Producer<K, V> getKafkaProducer(Properties props) {
        if (props.containsKey(BOOTSTRAP_SERVERS_KEY)) {
            return producers.computeIfAbsent(props.getProperty(BOOTSTRAP_SERVERS_KEY), x -> {
                Producer<K, V> producer;
                try {
                    producer = new KafkaProducer<>(props);
                } catch (Exception e) {
                    throw new RuntimeException("failed to create kafka producer", e);
                }
                return producer;
            });
        } else {
            throw new IllegalArgumentException("no bootstrap servers specified");
        }
    }
```

### Data Sender

1.  For each different Sink option, we can define Abstract Class, and invoke the client lib  

    - Example of kafka sender

```JAVA
protected void send(ProducerRecord<K, V> record) {
    try {
        Producer<K, V> producer = KafkaProducerFactory.getKafkaProducer(this.properties);
        producer.send(record, this.producerCallback);
        MdmLogger.log(appConfig, KAFKA_STATUS, true,100, this.appName, this.topicProvided, this.envType.name(), jobName, this.environment, this.producerTopic, "sent", name, "unknown");
    } catch (Exception e) {
        throw new RuntimeException("Failed to send to kafka", e);
    }
}
```



2. For Data sender, we should define abstract class and add metrics collection in abstract class


## Utils

### Log

1. Log4j

2. After configure the log, we can have logger to be retrievd in each Class

```JAVA
private static Logger logger = Logger.getLogger(MyClass.class.getSimpleName());
```

### Reflect

https://docs.oracle.com/javase/tutorial/reflect/member/ctorInstance.html

In Java, we generally create objects using the new keyword or we use some DI framework e.g. Spring to create an object which internally use Java Reflection API to do so. In this Article, we are going to study the reflective ways to create objects.

There are two methods present in Reflection API which we can use to create objects
```
Class.newInstance() → Inside java.lang package
Constructor.newInstance() → Inside java.lang.reflect package
```

By name, both methods look same but there are differences between them which we are as following

1. Class.newInstance() can only invoke the no-arg constructor,
   Constructor.newInstance() can invoke any constructor, regardless of the number of parameters.

2. Class.newInstance() requires that the constructor should be visible,
   Constructor.newInstance() can also invoke private constructors under certain circumstances.

3. Class.newInstance() throws any exception (checked or unchecked) thrown by the constructor,
   Constructor.newInstance() always wraps the thrown exception with an InvocationTargetException.

Due to above reasons Constructor.newInstance() is preferred over Class.newInstance(), that’s why used by various frameworks and APIs like Spring, Guava, Zookeeper, Jackson, Servlet etc.

```JAVA
public class ReflectUtils {
    protected static Logger logger = Logger.getLogger(ReflectUtils.class.getSimpleName());

    public static <T> T extractComponent(String className) throws Exception {

        Class<?> c = Class.forName(className);
        Constructor<?> cons = c.getConstructor();
        Object object = cons.newInstance();

        return (T) object;
    }
}
```


## Cache

### Guava Cache

https://github.com/google/guava/wiki/CachesExplained

- Implement Abstract Method

```JAVA
public Map<K, V> loadAll(Iterable<? extends K> keys) throws Exception {
    throw new CacheLoader.UnsupportedLoadingOperationException();
}
```

## Data Store

### Hbase

https://www.baeldung.com/hbase

https://www.corejavaguru.com/bigdata/hbase-tutorial/configuration

http://blog.asquareb.com/blog/2015/01/01/configuration-parameters-that-can-influence-hbase-performance/

## Config

### Type Safe Config

https://github.com/lightbend/config/blob/master/config/src/main/java/com/typesafe/config/impl/ConfigParser.java

#### Example

https://dzone.com/articles/typesafe-config-features-and-example-usage


1. default.conf
```
conf {
  name = "default"
  title = "Simple Title"
  nested {
    whitelistIds = [1, 22, 34]
  }
  
  combined = ${conf.name} ${conf.title} 
}

featureFlags {
  featureA = "yes"
  featureB = true
}
```

2. override.conf

```
conf {
  name = "overrides"
}

redis {
  ttl = 5 minutes
}

uploadService {
  maxChunkSize = 512k
  maxFileSize = 5G
}
```

#### Loading from ConfigFactory

Loading a Typesafe Config from a resource on the classpath is a simple one liner. Typesafe Config has several default configuration locations it looks at when loading through ```ConfigFactory.load()```; but we are big fans of making everything explicit. It's preferred if the configs that were being loaded are listed right there when we call it.

```JAVA
Config defaultConfig = ConfigFactory.parseResources("defaults.conf");
```


#### Loading with FallBack

https://www.stubbornjava.com/posts/environment-aware-configuration-with-typesafe-config

A great use case for this is environment specific overrides. You can set up one defaults file and have an override for local, dev, and prod. This can be seen in our environment aware configurations. In this case, we first load our ```overrides.conf``` , then set a fallback of our ```defaults.conf``` from above. When searching for any property, it will first search the original config, then traverse down the fallbacks until a value is found. This can be chained many times. We will come back to the ```resolve() method a little later.

```JAVA
Config fallbackConfig = ConfigFactory.parseResources("overrides.conf")
                                     .withFallback(defaultConfig)
                                     .resolve();
```

#### Loading String

```JAVA
log.info("name: {}", defaultConfig.getString("conf.name"));
log.info("name: {}", fallbackConfig.getString("conf.name"));
log.info("title: {}", defaultConfig.getString("conf.title"));
log.info("title: {}", fallbackConfig.getString("conf.title"));
```

```
2017-08-15 09:07:14.461 [main] INFO  TypesafeConfigExamples - name: default
2017-08-15 09:07:14.466 [main] INFO  TypesafeConfigExamples - name: overrides
2017-08-15 09:07:14.466 [main] INFO  TypesafeConfigExamples - title: Simple Title
2017-08-15 09:07:14.466 [main] INFO  TypesafeConfigExamples - title: Simple Title

```

#### Config Value Resolution

Sometimes, it's useful to reuse a config value inside of other config values. This is achieved through configuration resolution by calling the ```resolve()``` method on a Typesafe Config. An example of this use case could be if all hostnames are scoped per environment ({env}.yourdomain.com). It would be nicer to only override the env property and have all other values reference it. Let's use the above name and title again just for an example. If you look at the example configs, we have the following ```combined = ${conf.name} ${conf.title}```.

```JAVA
log.info("combined: {}", fallbackConfig.getString("conf.combined"));
```

```
2017-08-15 09:07:14.466 [main] INFO  TypesafeConfigExamples - combined: overrides Simple Title
```

#### Duration/Memory/List/Boolean Helper

```JAVA
// {{start:durations}}
log.info("redis.ttl minutes: {}", fallbackConfig.getDuration("redis.ttl", TimeUnit.MINUTES));
log.info("redis.ttl seconds: {}", fallbackConfig.getDuration("redis.ttl", TimeUnit.SECONDS));
// {{end:durations}}

// {{start:memorySize}}
// Any path in the configuration can be treated as a separate Config object.
Config uploadService = fallbackConfig.getConfig("uploadService");
log.info("maxChunkSize bytes: {}", uploadService.getMemorySize("maxChunkSize").toBytes());
log.info("maxFileSize bytes: {}", uploadService.getMemorySize("maxFileSize").toBytes());
// {{end:memorySize}}

// {{start:whitelist}}
List<Integer> whiteList = fallbackConfig.getIntList("conf.nested.whitelistIds");
log.info("whitelist: {}", whiteList);
List<String> whiteListStrings = fallbackConfig.getStringList("conf.nested.whitelistIds");
log.info("whitelist as Strings: {}", whiteListStrings);
// {{end:whitelist}}


// {{start:booleans}}
log.info("yes: {}", fallbackConfig.getBoolean("featureFlags.featureA"));
log.info("true: {}", fallbackConfig.getBoolean("featureFlags.featureB"));
```

### Full TypeSafe Config Example

https://www.stubbornjava.com/posts/environment-aware-configuration-with-typesafe-config

```JAVA
public class Configs {
    private static final Logger log = LoggerFactory.getLogger(Configs.class);

    private Configs() { }

    /*
     * I am letting the typesafe configs bleed out on purpose here.
     * We could abstract out and delegate but its not worth it.
     * I am gambling on the fact that I will not switch out the config library.
     */

    // This config has all of the JVM system properties including any custom -D properties
    private static final Config systemProperties = ConfigFactory.systemProperties();

    // This config has access to all of the environment variables
    private static final Config systemEnvironment = ConfigFactory.systemEnvironment();

    // Always start with a blank config and add fallbacks
    private static final AtomicReference<Config> propertiesRef = new AtomicReference<>(null);

    public static void initProperties(Config config) {
        boolean success = propertiesRef.compareAndSet(null, config);
        if (!success) {
            throw new RuntimeException("propertiesRef Config has already been initialized. This should only be called once.");
        }
    }

    public static Config properties() {
        return propertiesRef.get();
    }

    public static Config systemProperties() {
        return systemProperties;
    }

    public static Config systemEnvironment() {
        return systemEnvironment;
    }

    public static Configs.Builder newBuilder() {
        return new Builder();
    }

    // This should return the current executing user path
    public static String getExecutionDirectory() {
        return systemProperties.getString("user.dir");
    }

    public static <T> T getOrDefault(Config config, String path, BiFunction<Config, String, T> extractor, T defaultValue) {
        if (config.hasPath(path)) {
            return extractor.apply(config, path);
        }
        return defaultValue;
    }

    public static <T> T getOrDefault(Config config, String path, BiFunction<Config, String, T> extractor, Supplier<T> defaultSupplier) {
        if (config.hasPath(path)) {
            return extractor.apply(config, path);
        }
        return defaultSupplier.get();
    }

    public static Map<String, Object> asMap(Config config) {
        return Seq.seq(config.entrySet())
                  .toMap(e -> e.getKey(), e -> e.getValue().unwrapped());
    }

    public static class Builder {
        private Config conf = ConfigFactory.empty();

        public Builder() {
            log.info("Loading configs first row is highest priority, second row is fallback and so on");
        }

        public Builder withResource(String resource) {
            Config resourceConfig = ConfigFactory.parseResources(resource);
            String empty = resourceConfig.entrySet().size() == 0 ? " contains no values" : "";
            conf = conf.withFallback(resourceConfig);
            log.info("Loaded config file from resource ({}){}", resource, empty);
            return this;
        }

        public Builder withSystemProperties() {
            conf = conf.withFallback(systemProperties);
            log.info("Loaded system properties into config");
            return this;
        }

        public Builder withSystemEnvironment() {
            conf = conf.withFallback(systemEnvironment);
            log.info("Loaded system environment into config");
            return this;
        }

        public Builder withOptionalFile(String path) {
            File secureConfFile = new File(path);
            if (secureConfFile.exists()) {
                log.info("Loaded config file from path ({})", path);
                conf = conf.withFallback(ConfigFactory.parseFile(secureConfFile));
            } else {
                log.info("Attempted to load file from path ({}) but it was not found", path);
            }
            return this;
        }

        public Builder withOptionalRelativeFile(String path) {
            return withOptionalFile(getExecutionDirectory() + path);
        }

        public Builder withConfig(Config config) {
            conf = conf.withFallback(config);
            return this;
        }

        public Config build() {
            // Resolve substitutions.
            conf = conf.resolve();
            if (log.isDebugEnabled()) {
                log.debug("Logging properties. Make sure sensitive data such as passwords or secrets are not logged!");
                log.debug(conf.root().render());
            }
            return conf;
        }
    }

    public static void main(String[] args) {
        log.debug(ConfigFactory.load().root().render(ConfigRenderOptions.concise()));

        //newBuilder().withSystemEnvironment().withSystemProperties().build();
    }
}
```

### DB Connection Pool via TypeSafe Config

https://www.stubbornjava.com/posts/database-connection-pooling-in-java-with-hikaricp

* Constantly opening and closing connections can be expensive. Cache and reuse.
* When activity spikes you can limit the number of connections to the database. This will force code to block until a connection is available. This is especially helpful in distributed environments.
* Split out common operations into multiple pools. For instance you can have a pool designated for OLAP connections and a pool for OLTP connections each with different configurations.


1. Here is our config

```
pools {
    default {
        jdbcUrl = "jdbc:hsqldb:mem:testdb"
        maximumPoolSize = 10
        minimumIdle = 2
        username = "SA"
        password = ""
        cachePrepStmts = true
        prepStmtCacheSize = 256
        prepStmtCacheSqlLimit = 2048
        useServerPrepStmts = true
    }

    // This syntax inherits the config from pools.default.
    // We can then override or add additional properties.
    transactional = ${pools.default} {
        poolName = "transactional"
    }

    processing = ${pools.default} {
        poolName = "processing"
        maximumPoolSize = 30
    }
}
```

```JAVA
public class ConnectionPools {
    private static final Logger logger = LoggerFactory.getLogger(ConnectionPools.class);
    /*
     * Normally we would be using the app config but since this is an example
     * we will be using a localized example config.
     */
    private static final Config conf = new Configs.Builder()
                                                  .withResource("examples/hikaricp/pools.conf")
                                                  .build();
    /*
     *  This pool is made for short quick transactions that the web application uses.
     *  Using enum singleton pattern for lazy singletons
     */
    private enum Transactional {
        INSTANCE(ConnectionPool.getDataSourceFromConfig(conf.getConfig("pools.transactional"), Metrics.registry(), HealthChecks.getHealthCheckRegistry()));
        private final HikariDataSource dataSource;
        private Transactional(HikariDataSource dataSource) {
            this.dataSource = dataSource;
        }
        public HikariDataSource getDataSource() {
            return dataSource;
        }
    }
    public static HikariDataSource getTransactional() {
        return Transactional.INSTANCE.getDataSource();
    }

    /*
     *  This pool is designed for longer running transactions / bulk inserts / background jobs
     *  Basically if you have any multithreading or long running background jobs
     *  you do not want to starve the main applications connection pool.
     *
     *  EX.
     *  You have an endpoint that needs to insert 1000 db records
     *  This will queue up all the connections in the pool
     *
     *  While this is happening a user tries to log into the site.
     *  If you use the same pool they may be blocked until the bulk insert is done
     *  By splitting pools you can give transactional queries a much higher chance to
     *  run while the other pool is backed up.
     */
    private enum Processing {
        INSTANCE(ConnectionPool.getDataSourceFromConfig(conf.getConfig("pools.processing"), Metrics.registry(), HealthChecks.getHealthCheckRegistry()));
        private final HikariDataSource dataSource;
        private Processing(HikariDataSource dataSource) {
            this.dataSource = dataSource;
        }
        public HikariDataSource getDataSource() {
            return dataSource;
        }
    }

    public static HikariDataSource getProcessing() {
        return Processing.INSTANCE.getDataSource();
    }

    public static void main(String[] args) {
        logger.debug("starting");
        DataSource processing = ConnectionPools.getProcessing();
        logger.debug("processing started");
        DataSource transactional = ConnectionPools.getTransactional();
        logger.debug("transactional started");
        logger.debug("done");
    }
}
```

# Spark Jobs

All jobs compoenent needs to be serializable, so that they can be transfered from Master to executor

```JAVA
public class BaseComponent implements Serializable {

}
```

## Spark Stream vs Structure Stream

https://dzone.com/articles/spark-streaming-vs-structured-streaming

Fan of Apache Spark? I am too. The reason is simple. Interesting APIs to work with, fast and distributed processing, and, unlike MapReduce, there's no I/O overhead, it's fault tolerance, and much more. With this, you can do a lot in the world of big data and fast data. From "processing huge chunks of data" to "working on streaming data," Spark works flawlessly. In this post, we will be talking about the streaming power we get from Spark.

Spark provides us with two ways of working with streaming data:

Structured Streaming (introduced with Spark 2.x)
Let's discuss what these are exactly, what the differences are, and which one is better.


### Spark RDD(DStream) Streaming

Spark Streaming is a separate library in Spark to process continuously flowing streaming data. It provides us with the DStream API, which is powered by Spark RDDs. DStreams provide us data divided into chunks as RDDs received from the source of streaming to be processed and, after processing, sends it to the destination. Cool, right?!

https://spark.apache.org/docs/latest/rdd-programming-guide.html



### Structured Streaming

From the Spark 2.x release onwards, Structured Streaming came into the picture. Built on the Spark SQL library, Structured Streaming is another way to handle streaming with Spark. This model of streaming is based on Dataframe and Dataset APIs. Hence, with this library, we can easily apply any SQL query (using the DataFrame API) or Scala operations (using DataSet API) on streaming data.

Okay, so that was the summarized theory for both ways of streaming in Spark. Now we need to compare the two.

### Distinctions

#### Real Streaming
What does real streaming imply? Data which is unbounded and is being processed upon being received from the source. This definition is satisfiable (more or less).

If we talk about Spark Streaming, this is not the case. Spark Streaming works on something we call a micro batch. The stream pipeline is registered with some operations and Spark polls the source after every batch duration (defined in the application) and then a batch is created of the received data, i.e. each incoming record belongs to a batch of DStream. Each batch represents an RDD.


![dstream](https://github.com/zhangruiskyline/system/blob/main/images/dstream.png)


Structured Streaming works on the same architecture of polling the data after some duration, based on your trigger interval, but it has some distinction from the Spark Streaming which makes it more inclined towards real streaming. In Structured Streaming, there is no batch concept. The received data in a trigger is appended to the continuously flowing data stream. Each row of the data stream is processed and the result is updated into the unbounded result table. How you want your result (updated, new result only, or all the results) depends on the mode of your operations (Complete, Update, Append).

![sstream](https://github.com/zhangruiskyline/system/blob/main/images/sstream.png)

Winner of this round: Structured Streaming.

#### RDD vs. DataFrames/DataSet

Another distinction can be the use case of different APIs in both streaming models. In summary, we read that Spark Streaming works on the DStream API, which is internally using RDDs and Structured Streaming uses DataFrame and Dataset APIs to perform streaming operations. So, it is a straight comparison between using RDDs or DataFrames. There are several blogs available which compare DataFrames and RDDs in terms of `performance` and `ease of use.` This is a good read for RDD v/s Dataframes. All those comparisons lead to one result: that DataFrames are more optimized in terms of processing and provide more options for aggregations and other operations with a variety of functions available (many more functions are now supported natively in Spark 2.4).

So Structured Streaming wins here with flying colors.

#### Processing With the Vent Time, Handling Late Data

One big issue in the streaming world is how to process data according to the event-time. Event-time is the time when the event actually happened. It is not necessary for the source of the streaming engine to prove data in real-time. There may be latencies in data generation and handing over the data to the processing engine. There is no such option in Spark Streaming to work on the data using the event-time. It only works with the timestamp when the data is received by the Spark. Based on the ingestion timestamp, Spark Streaming puts the data in a batch even if the event is generated early and belonged to the earlier batch, which may result in less accurate information as it is equal to the data loss. On the other hand, Structured Streaming provides the functionality to process data on the basis of event-time when the timestamp of the event is included in the data received. This is a major feature introduced in Structured Streaming which provides a different way of processing the data according to the time of data generation in the real world. With this, we can handle data coming in late and get more accurate results.

With event-time handling of late data, Structured Streaming outweighs Spark Streaming.

#### End-to-End Guarantees

Every application requires fault tolerance and end-to-end guarantees of data delivery. Whenever the application fails, it must be able to restart from the same point where it failed in order to avoid data loss and duplication. To provide fault tolerance, Spark Streaming and Structured Streaming both use the checkpointing to save the progress of a job. But this approach still has many holes which may cause data loss.

Other than checkpointing, Structured Streaming has applied two conditions to recover from any error:

1. The source must be replayable.
2. The sinks must support idempotent operations to support reprocessing in case of failures.
Here's a link to the docs to learn more.

With restricted sinks, Spark Structured Streaming always provides end-to-end, exactly once semantics. Way to go Structured Streaming!

#### Restricted or Flexible
Sink: The destination of a streaming operation. It can be external storage, a simple output to console, or any action

With Spark Streaming, there is no restriction to use any type of sink. Here we have the method ```foreachRDD``` to perform some action on the stream. This method returns us the RDDs created by each batch one-by-one and we can perform any actions over them, like saving to storage or performing some computations. We can cache an RDD and perform multiple actions on it as well (even sending the data to multiple databases).

But in Structures Streaming, until v2.3, we had a limited number of output sinks and, with one sink, only one operation could be performed and we could not save the output to multiple external storages. To use a custom sink, the user needed to implement ForeachWriter. But here comes Spark 2.4, and with it we get a new sink called foreachBatch. This sink gives us the resultant output table as a DataFrame and hence we can use this DataFrame to perform our custom operations.

With this new sink, the `restricted` Structured Streaming is now more `flexible` and gives it an edge over the Spark Streaming and other over flexible sinks.

#### Throughput/Concurrency

Dstream can enable Concurrency, so multiple micro batches can run concurrently, throughput could be higher, but we may also face out of order, since micro batches could be processed in concurrently, late arrived micro batch could be processed beforehand.

For structure stream, we can not leverage concurrency, if kafka has limited partition, job delay will be built up

#### Conclusion
We saw a fair comparison between Spark Streaming and Spark Structured Streaming. We can clearly say that Structured Streaming is more inclined to real-time streaming but Spark Streaming focuses more on batch processing. The APIs are better and optimized in Structured Streaming where Spark Streaming is still based on the old RDDs.

So to conclude this post, we can simply say that Structured Streaming is a better streaming platform in comparison to Spark Streaming.

Please make sure to comment your thoughts on this!

References:

Structured Streaming Programming Guide
Streaming Programming Guide

## Dstream(RDD) Job

### Dstream Generator

```JAVA
// This uses factory design pattern to return the appropriate DStreams generator class for each topic.
public class MyDStreamGenFactory {
    public static MyDStreamGen<? extends BaseEventClass> getStreamGenerator(String className) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        Class<?> c = Class.forName(className);
        Constructor<?> cons = c.getConstructor();
        return  (MyDStreamGen) cons.newInstance();
    }

    public static MyDStreamGen getStreamGenerator(Config componentConfig, Config jobConfig, Config appConfig) throws Exception {
        Class<?> c = Class.forName(jobConfig.getString("streaming.queue.fetchClass"));
        Constructor<?> cons = c.getConstructor();
        BaseComponent component = (BaseComponent) cons.newInstance();
        component.initialize(componentConfig, jobConfig, appConfig, "streamGen");
        return  (MyDStreamGen) component;
    }
}
```

In the config, we have 

```
mydomain.myorg.MyJob {

  streaming.queue {
    type = kafka
    TOPIC = MY_TOPIC
    BROKERS = ${BROKERS_LIST}
    STREAMDURATIONINSECONDS = XX
    fetchClass = com.mydomain.myorg.spark_common.stream.deserializer.my_event.KafkaMyEventJsonStreamGen
  }

}
```

### Dstream Job

```JAVA
abstract public class SparkStreamingJob<I extends BaseEventClass, O extends Object> extends SparkBaseJob<O> {

    public void run(SparkSession ss) throws Exception {

        try {
            JavaStreamingContext jssc = createJavaStreamingContext(ss);
            jssc.start();
            jssc.awaitTermination();
        } catch (Throwable e) {
            logger.error("Failed to running streaming", e);
            if (e.getCause() instanceof InvalidOffsetException
                    || e.getCause() instanceof IllegalArgumentException ) {
                logger.info("Removing out of range's offset path.");
                FilePathHandler.withThrowingHadoopFilesystem(offsetPath, fs -> {
                    fs.delete(new Path(offsetPath), true);
                });
                logger.info("Removed out of range's offset path.");
            }
            throw new Exception(e);
        }
    }

    public JavaStreamingContext createJavaStreamingContext(SparkSession ss) throws Exception {

        ConfigUtils.disableSecurityManager();
        JavaSparkContext jsc = new JavaSparkContext(ss.sparkContext());
        this.numOfRecords = ss.sparkContext().longAccumulator("numOfRecords");

        JavaStreamingContext jssc = new JavaStreamingContext(jsc, Durations.seconds(getConf("StreamDuration")));
        // The app name will appear in the upper right hand side of Spark Streaming UI
        tm.applicationName = jsc.appName();

        // use DStream. Get stream generator from Factory pattern
        MyDStreamGen streamGenerator = MyDStreamGenFactory.getStreamGenerator(componentConfig, jobConfig, appConfig);
        if (streamGenerator == null) {
            throw new IllegalArgumentException(String.format("illegal LogFetch class %s provided", tm.streamFetchClass));
        }
        JavaDStream<I> dStream = streamGenerator.fetchStreams(tm, jssc);
        processDStream(dStream, ss);
        return jssc;

    }

    public void processDStream(JavaDStream<I> dStream, SparkSession ss) {
        logger.info("starting streaming job");
        JavaDStream<O> transformStream = transform(dStream);
        transformStream.foreachRDD(rdd -> {
            if (!rdd.partitions().isEmpty()) {
                if (jobConfig.hasPath(STREAMING_INCOMING_PARTITIONER)) {
                	//getInstance via ReflectUtils.extractComponent
                    MyPartitioner<O> partitioner = NyPartitioner.getInstance(jobConfig.getString(STREAMING_INCOMING_PARTITIONER),
                            ConfigUtils.getOrDefault(jobConfig, STREAMING_INCOMING_NUM_OF_PARTITIONs, rdd.partitions().size()));
                    rdd = rdd.mapToPair(x -> new Tuple2<>(partitioner.getParitionKey(x), x))
                            .partitionBy(partitioner)
                            .map(x -> x._2);
                }
                this.numOfRecords.reset();
		        processRDD(rdd, ss);
		        rdd.unpersist(false);
		            }
		        });
    }

    public JavaDStream<O> transform(JavaDStream<I> dStream) {
        return dStream.map(x -> (O)x);
    }
}
```

## Structure Stream Job

https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html

### StreamingQueryListener

https://jaceklaskowski.gitbooks.io/spark-structured-streaming/content/spark-sql-streaming-StreamingQueryListener.html

StreamingQueryListener is the contract of listeners that want to be notified about the life cycle events of streaming queries, i.e. start, progress and termination.

#### onQueryStarted

```JAVA
onQueryStarted(event: QueryStartedEvent): Unit
```

Informs that DataStreamWriter was requested to start execution of the streaming query (on the stream execution thread)

#### onQueryProgress

```JAVA
onQueryProgress(event: QueryProgressEvent): Unit
```
Informs that MicroBatchExecution has finished triggerExecution phase (the end of a streaming batch)

we can get all information while
https://jaceklaskowski.gitbooks.io/spark-structured-streaming/content/spark-sql-streaming-StreamingQueryProgress.html

Example
```JAVA
Long processedTime = event.progress().durationMs().get("triggerExecution"); // the whole micro-batch processed time
```

#### onQueryTerminated

```JAVA
onQueryTerminated(event: QueryTerminatedEvent): Unit
```
Informs that a streaming query was stopped or terminated due to an error


### Base Job

```JAVA
abstract public class SparkDataFrameJob extends SparkBaseJob {
    private static final Logger logger = Logger.getLogger(SparkBaseJob.class.getSimpleName());

    public void initialize(Config jobConfig, Config appConfig) throws Exception {
        super.initialize(jobConfig, appConfig);
        initializeProperties();
    }

    public void initializeProperties() {
    	//Init
    }

    @Override
    public void run(SparkSession ss) throws Exception {
        Dataset<Row> dataset = read(ss);
        Dataset<Row> transformedDataset = transform(dataset);
        write(transformedDataset);
    }

    abstract public Dataset<Row> transform(Dataset<Row> dataset);

    abstract public Dataset<Row> read(SparkSession ss);

    abstract public void write(Dataset<Row> dataset);
 }
```

### Structure Stream Job


```JAVA
public abstract class SparkStructureStreamingJob extends SparkDataFrameJob {
    private static final Logger logger = Logger.getLogger(SparkStructureStreamingJob.class.getSimpleName());
    // number of partitions if repartition needed
    protected int partitions;
    protected String dataColumn;
    protected String outputPath;


    @Override
    public Dataset<Row> read(SparkSession ss) {
        listeners.forEach(x -> ss.streams().addListener(x));
        //ss.conf().set("spark.sql.streaming.metricsEnabled", "true");
        if (ss.conf().contains("spark.streaming.kafka.maxRatePerPartition")) {
            Dataset<Row> dataset = ss
                    .readStream()
                    .format(format)
                    .options(inputOptions)
                    .option(MAX_OFFSET_PER_TRIGGER_KEY, ss.conf().get("spark.streaming.kafka.maxRatePerPartition"))
                    .load();
            return dataset;
        }
        Dataset<Row> dataset = ss
                .readStream()
                .format(format)
                .options(inputOptions)
                .load();
        return dataset;
    }

    public UserDefinedFunction transformColumns() {
        UserDefinedFunction transformColumns = udf(
                (byte[] bytes, String topic) -> {
                    long startTs = Instant.now().toEpochMilli();
                    List rows = this.deserializer.getRows(bytes, topic);
                    long endTs = Instant.now().toEpochMilli();
                    return rows.toArray(new Row[]{});
                }, this.deserializer.getDataType()
        );
        return transformColumns;
    }

    public Dataset<Row> transform(Dataset<Row> dataset) {
        return dataset.selectExpr(dataColumn, "topic", "myTopic")
                .withColumn("myColumn", transformColumns().apply(col(dataColumn), col("topic")))
                .withColumn("myColumn", explode(col("myColumn")))
                .selectExpr("someColumn", "myColumn.*");
    }


    @Override
    public void write(Dataset<Row> dataset) {
        try {
            if (this.outputPath != null) {
                dataset
                        .writeStream()
                        .partitionBy(paritionColumns)
                        .format(outputFormat)
                        .options(outputOptions)
                        .start(outputPath)
                        .awaitTermination();
            } else {
                dataset
                        .writeStream()
                        .partitionBy(paritionColumns)
                        .format(outputFormat)
                        .options(outputOptions)
                        .start()
                        .awaitTermination();
            }
        } catch (StreamingQueryException e) {
            throw new RuntimeException("Failed to write", e);
        }
    }

    /**
     * initialize the listeners if there are any
     *
     * @return
     * @throws Exception
     */
    public List<NativeAdsSparkStreamingListener> initializeListeners() throws Exception {
        List<NativeAdsSparkStreamingListener> listeners = new ArrayList<>();
        if (jobConfig.hasPath(LISTENER_KEY)) {
            List<String> list = jobConfig.getStringList(LISTENER_KEY);
            for (String listener : list) {
                NativeAdsSparkStreamingListener streamingQueryListener = ReflectUtils.extractComponent(listener);
                streamingQueryListener.initialize(jobConfig, appConfig);
                listeners.add(streamingQueryListener);
            }
        }
        return listeners;
    }
}
```

#### Convert from Dataset<Row> to Dataset<T>


```JAVA
public interface IMyDeserializer<T> extends Serializable, Deserializer<T[]> {
    Encoder<T> getEncoder();
}

//In our Class
private IMyDeserializer<T> deserializer;

public Dataset<T> toDataset(Dataset<Row> dataset) {
        if (numOfPartitions != 0) {
            UserDefinedFunction udf = functions.udf((byte[] bytes) -> {
                String key = new String(bytes);
                int partition = myCustomerPartitioner(key, numOfPartitions);
                return partition;
            }, DataTypes.IntegerType);
            dataset = dataset.repartition(numOfPartitions, udf.apply(functions.col("key")));
        }
        dataset = dataset.selectExpr(dataColumn, "column1", "column2");
        Dataset<T> transformedDataset = dataset.mapPartitions((MapPartitionsFunction<Row, T>) rows -> {
            List<T> events = new ArrayList<>();
            rows.forEachRemaining(x -> {
                events.addAll(Arrays.asList(deserializer.deserialize(x.getAs("column1").toString(), x.getAs(dataColumn))));
            });
            List<T> filteredEvents = myFilter(events);
            return filteredEvents.iterator();
        }, deserializer.getEncoder());

        return transformedDataset;
    }
```


