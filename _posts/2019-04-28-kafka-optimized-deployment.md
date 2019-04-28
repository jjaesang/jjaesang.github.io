---
layout: post
title:  "Optimizing Kafka Deployment Paper 정리"
date: 2019-04-28 14:05:12
categories: Kafka
author : Jaesang Lim
tag: Spark
cover: "/assets/kafka_log.png"
---

[optimizing-your-apache-kafka-deployment](https://www.confluent.io/white-paper/optimizing-your-apache-kafka-deployment/) 논문 리뷰

서비스의 목적에 맞춰 카프카 클러스터의 설정값을 튜닝해야함

### Deciding which Service Goads to Optimize

- 4가지의 이루고자하는 서비스의 목표가 있으며, 이 4가지의 목표는 tradeoff 관계가 있음
1. throughput 
2. latency
3. durability
4. availability
- 위의 4가지 목표를 모두 달성하는 것은 불가능하므로, 서비스의 목적,목표에 맞춰 설정해야함

#### 1. High Throughput
- rate that data is moved from producer to brokers or brokers to consumer
- 데이터 이동하는 비
- 초당 처리량을 늘리자

#### 2. Low Latency
- elpased time moving message end-to-end
- e.g) chat application / interactive web / real-time stream processing
- 지연시간을 최대로 줄이자

#### 3. High Durability
- Guarantees that messages that have been committed will not be lost
- e.g) event-driven microservices pipeline / integration between a streaming source and permanent Storage
- 한번 커밋된 데이터는 유실되지 않아야함 ( 데이터 유실 없애자 )

#### 4. High Acailability
- Minimizes downtime in case of unexpected failures
- kafka는 분산시스템으로 fault-tolerance이 구현되어 있음
- downTime을 최소화하자

---

### Optimizing for Throughput

- Throughput를 최적화하기 위해서는 producer, broker, consumer는 주어진 시간 내에 최대한 많은 데이터를 이동해야함
- High Throughput을 얻기 위해서는 데이터가 이동하는 비율을 최대화해야하며 ,이 데이터 전송률은 가능한 한 빨라야함
- Topic의 parition은 parallelism의 단위이기 때문에 partition이 많을수록 throughput이 높아짐
- partition 수가 많으면 좋다고 막 늘리는 것 보다는 producer와 consumer의 throghput를 고려해야함

#### 1. Producer 측면에서의 High Throughput

- producer는 같은 파티션에 대한 데이터를 batch로 처리함. 즉, 여러 message를 하나의 request로 묶어서 전송함
> - 한국 카프카 사용자 모임 발표때 들어보니, 배치로 처리하는 것은 네트워크의 ack, header등을 아낄 수 있다고 함 ( naggle's algorithm 기반)
- 그래서, producer측면에서는 다음과 같은 configuration을 고려해야함

1. batch.size / linger.ms
> - 한번의 batch request의 데이터 사이즈 / 배치사이즈가 다 채우질 때까지 기다리는 시간
  - 장점
  > - broker서버의 request 수가 적어, Producer의 load 및 broker CPU overhead를 줄일 수 있음
  - tradeoff 
  > - message을 받은 즉시 전송하지 않기 떄문에 Latency가 증가함

2. compression.type
> - lz4, snappy, gzip을 지원하며, compression은 전송전 batch처리에 적용되므로, 압축률에 batch 처리의 효율성에 영향이 미침 
> - 즉, 배치 비율이 높을수록 압축률이 향상됩

3. ack
> - 위에서 모인 batch request는 leader partition에 요청을 보내고, producer는 broker서버에게 ack를 기다림
> - ack를 빨리 받으면 받을 수 록, throughput이 증가함
- tradeoff
> -  low duarablity  ( ack =1 ) 

4. retries
- producer가 전송 실패시 재시도 횟수
- reties =0로 설정시 데이터 유실이 있을 수 있음

5. max.block.ms / buffer.memory
- Producer는 보내지 않은 메시지를 메모리 buffer에 저장해둠 ( producer의 accumulator)
- 보내지 않은 메세지를 담은 buffer가 가득차면, max.block.ms가 지나거나, 메모리가 해제되기 전까지 전송을 하지 않음
- partition이 많지 않다면, 이 설정값을 수정할 필욥는 없음
- 하지만, partition수가 많으면, buffer.memory, linger time, parition count 등을 고려해야하며, memory를 늘린다면 전송하는 동작이 멈추지 않아 throughput은 증가

#### 2. Consumer 측면에서의 High Throughput

1. fetch.min.bytes / fetch.max.waits.ms

- leader partition을 가진 broker에게 fetch request시, 얼만큼 데이터를 가져올 것인가로 조정할 수 있음
- 이 값을 올린다면, fetch request 횟수는 줄어드므로 , broker CPU overhead는 감소함 -> increase Througput / decrease Latency
fetch 요청에 fetch.min.bytes를 충족시킬만큼의 메시지가 있거나 대기 시간 (구성 매개 변수 fetch.max.wait.ms)이 만료 될 때까지 브로커가 사용자에게 새 메시지를 보내지 않기 때문입니다.


- consumer 수가 많으면 parall 증가, load 분산, 동시에 여러개 parition 땡김


- GC - CMS 쓰쏌



--- 

### Optimizing for Latency

- partition이 많으면 많을 수록, througput 증가하면서 latency도 증가함
- 브러커는 single thread로 다른 브러커에데 데이터를 복제함, 그래서 파티션이 많을 수록, 복제하는데 시간이 오래걸리고, 그렇게 되면 commit의 delay가 있어 Latency가 증가

1. num.replica.fetchers
2. linger.ms
3. compression
4. ack

### Optminizing for Durability

1. repliacation.factor
2. acks
3. retries
4. max.in.flight.requests.per.connection

1. default.replication.factor
2. auto.create.topics.enable
3. min.insync.replicas
4. unclen.leader.election.enable
5. broker.rack
6. log.flush.interval.message/ log.flush.interval.ms

1. auto.commit.enable

### Optimizing for Avaliablity

1. unclean.leader.election.enable
2. min.insync.replicas
3. num.recovery.threads.per.data.dir

1.seesion.timeout.ms










