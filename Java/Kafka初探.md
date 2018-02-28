# Kafka初探

本文档基于 [Apache Kafka官网](http://kafka.apache.org/) 进行翻译和整理，以梳理知识点，便于读者更快地了解与学习Kafka。

## 1. Kafka是什么，用来做什么

> Kafka is used for building real-time data pipelines and streaming apps. It is horizontally scalable, fault-tolerant, wicked fast, and runs in production in thousands of companies.
>
> Kafka用于构建实时数据管道和流式应用程序。它水平可扩展、具备容错性、速度快成一匹野马，并被数以千计的公司运用于生产。

Kafka是一个*分布式流平台 (distributed streaming platform)*，它具备以下三项关键功能：

1. **发布和订阅记录流**。在这方面，它类似于消息队列或企业消息传递系统。
2. 以容错的方式**存储记录流**。
3. 在发生记录时**处理记录流**。

Kafka主要有以下两类应用场景：

1. 在系统或应用程序之间构建可靠的获取数据的实时流数据管道
2. 构建对数据流进行转换或响应的实时流式应用程序

## 2. 服务端部署

### 2.1 下载并解压

下载地址：[kafka_2.11-1.0.0.tgz](https://www.apache.org/dyn/closer.cgi?path=/kafka/1.0.0/kafka_2.11-1.0.0.tgz)

下载完成后解压：

```shell
# 本文档中压缩包所在路径为'/usr/local'
> tar -xzf kafka_2.11-1.0.0.tgz
> cd kafka_2.11-1.0.0
```

### 2.2 准备Zookeeper

Kafka运行依赖于Zookeeper。如果仅作测试，可以选择使用Kafka自带的单节点Zookeeper实例，启动方式：

```shell
> bin/zookeeper-server-start.sh config/zookeeper.properties
```

本文档中使用搭建在其他服务器上的Zookeeper，故在配置阶段需要注意配置Zookeeper地址。

### 2.3 配置

- `kafka_2.11-1.0.0/config/server.properties` 

  注意以下三项配置：

  ```properties
  listeners=PLAINTEXT://:9092
  advertised.listeners=PLAINTEXT://168.33.66.151:9092
  zookeeper.connect=168.33.130.230:2181,168.33.130.231:2181,168.33.130.232:2181
  ```

  其中 `listeners` 与 `advertised.listeners` 配置项默认被注释，去掉注释并修改 `advertised.listeners` 为kafka服务器IP。

  **注意：如果kafka需要对外提供服务，请务必将 `advertised.listeners` 的IP设置为对外IP（不要设置为"localhost"或"127.0.0.1"）。**

  修改 `zookeeper.connect` 为你的Zookeeper地址，多个服务器地址用英文半角逗号隔开。

### 2.4 启动

```shell
> bin/kafka-server-start.sh config/server.properties
```

在启动时遇到了一条 `sed` 命令的语法错误（笔者操作系统版本为：SUSE Linux Enterprise Server 11 SP4  (x86_64)），经检查是 `kafka_2.11-1.0.0/bin/kafka-run-class.sh` 脚本中的以下语句报错：

```shell
# the first segment of the version number, which is '1' for releases before Java 9
# it then becomes '9', '10', ...
JAVA_MAJOR_VERSION=$($JAVA -version 2>&1 | sed -E -n 's/.* version "([^.-]*).*"/\1/p')
if [[ "$JAVA_MAJOR_VERSION" -ge "9" ]] ; then
 # 略...
else
 # 略...
fi
```

提示 `sed` 命令不支持 `-E` 的选项。为消除这个错误，笔者将对 `JAVA_MAJOR_VERSION` 的赋值部分改为了：

```shell
JAVA_MAJOR_VERSION=1
```

此处改动仅供参考。

### 2.5 创建topic

Kafka的记录流以topic进行分类存储于处理，由于客户端无法进行创建topic的操作，需要预先调用服务端脚本创建topic，这里作为示例，创建一个单分区的topic - `test` ：

```shell
> bin/kafka-topics.sh --create --zookeeper 168.33.130.230:2181 --replication-factor 1 --partitions 1 --topic test
Created topic "test".
```

### 2.6 本地测试发送与接收

可以使用Kafka自带的命令行客户端 `producer` 与 `consumer` 测试消息的发送与接收， `producer` 接收文件或标准输入，并将其作为消息发送到卡夫卡群集， `consumer` 读取消息并以标准输出形式输出。

使用 `producer` 发送消息，默认情况下每行将作为单独的一条消息发出：

```shell
> bin/kafka-console-producer.sh --broker-list 168.33.66.151:9092 --topic test
This is a message
This is another message
```

使用 `consumer` 接收消息：

```shell
> bin/kafka-console-consumer.sh --bootstrap-server 168.33.66.151:9092 --topic test --from-beginning
This is a message
This is another message
```

## 3. 客户端连通性测试

在Java中配置Kafka客户端，需要首先引入相关依赖，这里从maven引入依赖：

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>1.0.0</version>
</dependency>
```

本文档仅作客户端连通性测试，为方便展示，将 `producer` 与 `consumer` 的实例集成到同一个类 `KafkaUtils.java` 中：

```java
import java.util.Arrays;
import java.util.Date;
import java.util.Properties;
import java.util.concurrent.Future;

import org.apache.kafka.clients.consumer.Consumer;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.alibaba.fastjson.JSON;
import com.sunveee.template.ssm.util.LogUtil;

/**
 * Kafka工具类
 * <p>封装了生产者、消费者的配置和一些基本操作</p>
 * 
 * @author 51
 * @version $Id: KafkaUtils.java, v 0.1 2018年2月28日 上午10:03:42 51 Exp $
 */
public class KafkaUtils {
    private final static Logger             logger            = LoggerFactory.getLogger(KafkaUtils.class);

    /**
     * 生产者
     */
    private static Producer<String, String> producer;
    /**
     * 消费者
     */
    private static Consumer<String, String> consumer;
    /**
     * 服务器地址，多个地址以英文半角逗号隔开
     */
    private static final String             BOOTSTRAP_SERVERS = "168.33.66.151:9092";
    /**
     * 所使用的topic，注意：该topic无法在客户端创建，需要在服务器端通过脚本创建
     */
    private static final String             TOPIC_NAME        = "test";

    private KafkaUtils() {
    }

    /** 
     * 生产者配置
     */
    static {
        Properties props = new Properties();
        props.put("bootstrap.servers", BOOTSTRAP_SERVERS);
        props.put("acks", "all");
        props.put("retries", 0);
        props.put("batch.size", 16384);
        props.put("linger.ms", 1);
        props.put("buffer.memory", 33554432);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        producer = new KafkaProducer<>(props);
    }

    /** 
     * 消费者配置
     */
    static {
        Properties props = new Properties();
        props.put("bootstrap.servers", BOOTSTRAP_SERVERS);
        props.put("group.id", "test");
        props.put("enable.auto.commit", "true");
        props.put("auto.commit.interval.ms", "1000");
        props.put("session.timeout.ms", "30000");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList(TOPIC_NAME)); // 订阅相关主题
    }

    /**
     * 发送消息
     * 
     * @param topic
     * @param key
     * @param msg
     * @return
     */
    public static Future<RecordMetadata> sendMsgToKafka(String topic, String key, String msg) {
        // 发送消息，这之前可能需要对输入的topic进行校验：该topic是否配置在了TOPIC_NAMES中
        Future<RecordMetadata> result = producer.send(new ProducerRecord<String, String>(topic, String.valueOf(new Date().getTime()), msg));

        LogUtil.info(logger, "发送消息返回:{0}", JSON.toJSONString(result));
        return result;
    }

    /** 
     * 接收消息
     */
    public static void getMsgFromKafka() {
        // 建议使用线程池方式实现，这里仅作示例
        while (true) {
            ConsumerRecords<String, String> records = KafkaUtils.getKafkaConsumer().poll(100);
            if (records.count() > 0) {
                for (ConsumerRecord<String, String> record : records) {
                    LogUtil.info(logger, "从kafka接收到消息：" + record.value());
                }
            }
        }
    }

    public static Consumer<String, String> getKafkaConsumer() {
        return consumer;
    }

    public static void closeKafkaProducer() {
        producer.close();
    }

    public static void closeKafkaConsumer() {
        consumer.close();
    }
}
```

相应的发送消息测试类 `TestSend.java` ：

```java
import java.util.Calendar;

/**
 * 测试发送消息至Kafka
 * 
 * @author 51
 * @version $Id: TestSend.java, v 0.1 2018年2月28日 上午10:15:51 51 Exp $
 */
public class TestSend {
    public static void main(String[] args) {
        KafkaUtils.sendMsgToKafka("test", String.valueOf(Calendar.getInstance().getTimeInMillis()), "message body");
        KafkaUtils.closeKafkaProducer();
    }
}
```

接受消息测试类 `TestReceive.java` ：

```java
/**
 * 从Kafka接受消息
 * 
 * @author 51
 * @version $Id: TestReceive.java, v 0.1 2018年2月28日 上午10:22:25 51 Exp $
 */
public class TestReceive {
    public static void main(String[] args) {
        KafkaUtils.getMsgFromKafka();
    }
}
```

首先运行 `TestReceive.java` ，控制台显示：

```
[INFO] 2018-02-28 14:30:34 [org.apache.kafka.clients.producer.ProducerConfig] ProducerConfig values: 
	(略)
[INFO] 2018-02-28 14:30:34 [org.apache.kafka.common.utils.AppInfoParser] Kafka version : 1.0.0 
[INFO] 2018-02-28 14:30:34 [org.apache.kafka.common.utils.AppInfoParser] Kafka commitId : aaa7af6d4a11b29d 
[INFO] 2018-02-28 14:30:34 [org.apache.kafka.clients.consumer.ConsumerConfig] ConsumerConfig values: 
	(略)
[INFO] 2018-02-28 14:30:34 [org.apache.kafka.common.utils.AppInfoParser] Kafka version : 1.0.0 
[INFO] 2018-02-28 14:30:34 [org.apache.kafka.common.utils.AppInfoParser] Kafka commitId : aaa7af6d4a11b29d 
[INFO] 2018-02-28 14:30:35 [org.apache.kafka.clients.consumer.internals.AbstractCoordinator] [Consumer clientId=consumer-1, groupId=test] Discovered coordinator 168.33.66.151:9092 (id: 2147483647 rack: null) 
[INFO] 2018-02-28 14:30:35 [org.apache.kafka.clients.consumer.internals.ConsumerCoordinator] [Consumer clientId=consumer-1, groupId=test] Revoking previously assigned partitions [] 
[INFO] 2018-02-28 14:30:35 [org.apache.kafka.clients.consumer.internals.AbstractCoordinator] [Consumer clientId=consumer-1, groupId=test] (Re-)joining group 
[INFO] 2018-02-28 14:30:35 [org.apache.kafka.clients.consumer.internals.AbstractCoordinator] [Consumer clientId=consumer-1, groupId=test] Successfully joined group with generation 16 
[INFO] 2018-02-28 14:30:35 [org.apache.kafka.clients.consumer.internals.ConsumerCoordinator] [Consumer clientId=consumer-1, groupId=test] Setting newly assigned partitions [test-0] 
```

然后运行 `TestSend.java` ，其控制台显示：

```
[INFO] 2018-02-28 14:32:49 [org.apache.kafka.clients.producer.ProducerConfig] ProducerConfig values: 
	(略)
[INFO] 2018-02-28 14:32:49 [org.apache.kafka.common.utils.AppInfoParser] Kafka version : 1.0.0 
[INFO] 2018-02-28 14:32:49 [org.apache.kafka.common.utils.AppInfoParser] Kafka commitId : aaa7af6d4a11b29d 
[INFO] 2018-02-28 14:32:49 [org.apache.kafka.clients.consumer.ConsumerConfig] ConsumerConfig values: 
	(略)
[INFO] 2018-02-28 14:32:49 [org.apache.kafka.common.utils.AppInfoParser] Kafka version : 1.0.0 
[INFO] 2018-02-28 14:32:49 [org.apache.kafka.common.utils.AppInfoParser] Kafka commitId : aaa7af6d4a11b29d 
[INFO] 2018-02-28 14:32:50 [com.sunveee.template.ssm.kafka.KafkaUtils] 发送消息返回:{"cancelled":false,"done":true} 
[INFO] 2018-02-28 14:32:50 [org.apache.kafka.clients.producer.KafkaProducer] [Producer clientId=producer-1] Closing the Kafka producer with timeoutMillis = 9223372036854775807 ms. 
```

此时， `TestReceive.java` 的控制台有以下输出，表示 `consumer` 收到了 `producer` 发送到Kafka的消息：

```
[INFO] 2018-02-28 14:32:49 [com.sunveee.template.ssm.kafka.KafkaUtils] 从kafka接收到消息：message body
```

至此客户端与服务端连通性测试完毕。