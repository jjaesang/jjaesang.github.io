---
layout: post
title:  "Spark Streaming With Kafka Direct API"
date: 2018-05-30 13:05:12
categories: Spark Streaming
author : Jaesang Lim
tag: Spark Streaming
cover: "/assets/spark.png"
---

### Spark Streaming With Kafka Direct API


### 로직 설명 
`
 KafkaUtils.createDirectStream[String, String](
      sparkStreamContext,
      PreferConsistent,
      Subscribe[String, String](ESTAT_TOPIC, kafkaParams) )
      `

#### 1. createDirectStream
- Return Type : InputDStream[ConsumerRecord[K, V]] using DirectKafkaInputDStream
`new DirectKafkaInputDStream[K, V](ssc, locationStrategy, consumerStrategy, perPartitionConfig)`

#### 2. LocationStragegy
> Kafka에서 받아온 파티션을 Exeuctor에게 어떻게 규칙으로 할당할 것인가에 대한 전략 !

1. PreferBrokers
> - Kakfa 브로커와 동일한 노드에 Executor를 실행할 때
> - 나는 Kafka 클러스터가 따로 구성되어 있기 때문에, 이 옵션을 쓸 일은 없다

2. PreferConsistent
> - Executor에게 일관되게 Parition을 분배
> - 현재 내가 사요하는 전략이며 대부분 이렇게 한다고 코드 주석에도 적혀 있다.

3. PreferFixed
> - 파티션을 특정 Executor에가 할당하는 법 

코드 주석에 이미 잘 설명되어 있음 
```


 * Choice of how to schedule consumers for a given TopicPartition on an executor.
 * See [[LocationStrategies]] to obtain instances.
 * Kafka 0.10 consumers prefetch messages, so it's important for performance
 * to keep cached consumers on appropriate executors, not recreate them for every partition.
 * Choice of location is only a preference, not an absolute; partitions may be scheduled elsewhere.
 
@Experimental
object LocationStrategies {
  /**
   *  :: Experimental ::
   * Use this only if your executors are on the same nodes as your Kafka brokers.
   */
  @Experimental
  def PreferBrokers: LocationStrategy =
    org.apache.spark.streaming.kafka010.PreferBrokers

  /**
   *  :: Experimental ::
   * Use this in most cases, it will consistently distribute partitions across all executors.
   */
  @Experimental
  def PreferConsistent: LocationStrategy =
    org.apache.spark.streaming.kafka010.PreferConsistent

  /**
   *  :: Experimental ::
   * Use this to place particular TopicPartitions on particular hosts if your load is uneven.
   * Any TopicPartition not specified in the map will use a consistent location.
   */
  @Experimental
  def PreferFixed(hostMap: collection.Map[TopicPartition, String]): LocationStrategy =
    new PreferFixed(new ju.HashMap[TopicPartition, String](hostMap.asJava))

  /**
   *  :: Experimental ::
   * Use this to place particular TopicPartitions on particular hosts if your load is uneven.
   * Any TopicPartition not specified in the map will use a consistent location.
   */
  @Experimental
  def PreferFixed(hostMap: ju.Map[TopicPartition, String]): LocationStrategy =
    new PreferFixed(hostMap)
}
```

#### 3. ConsumerStrategy[K,V] ( subscribe / subscribepattern / Assgin)
> * Choice of how to create and configure underlying Kafka Consumers on driver and executors.
> * `Subscribe[String, String](TOPIC_NAME, kafkaParams) )`







## Optimizing Spark Streaming applications reading data from Apache Kafka

- Spark Streaming , Flink, Storm , Kafka Stream과 같이 널리 사용되는 framework in real time processing

- 하지만 성능 문제, 이벤트 마다 처리가 아닌 Time Window로 처리 = 그로 인한 딜레이 존재 


### Kafka direct implementation

- Recevier Based
> - Receiver에 기반한 구현은 병렬화가 덜되고 TLS 보안과 호환되지 않음
> - 프로세스를 병렬화하려면 여러 주제를 읽는 여러 개의 DStream을 만들어야함
> - kafka의 데이터가 오직 하나의 executor에 의해서만 수신된다면,이 데이터는 Spark의 Block Manager에 저장 될 것이고 executor가 한 Transformation에서 그 데이터를 사용할 것..
> - 스트리밍 응용 프로그램의 HA를 얻으려면 Checkpointing을 활성화해야함

- Direct API
> - 모든 Spark Executor는 Kafka로부터 데이터를받을 수 있기 때문에 직접 구현은 완전히 병렬화가 가능!
> - 정보가 하나 이상의 Topic에서 오는 것인지는 중요하지 않음
> - Kafka에 자체적인 오프셋 관리하기 때문에 ,Checkpointing을 활성화 할 필요가 없음


[참고자료](https://www.stratio.com/blog/optimizing-spark-streaming-applications-apache-kafka/)
