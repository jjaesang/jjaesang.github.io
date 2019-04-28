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
설정값이 의미하는 것과, 카프카 내부가 어떻게 동작하는지 알 수 있어서 매우 의미있는 논문임~~

### Deciding which Service Goads to Optimize

- 4가지의 이루고자하는 서비스의 목표가 있으며, 이 4가지의 목표는 서로 tradeoff 관계가 있음
1. Throughput 
2. Latency
3. Durability
4. Availability
- 위의 4가지 목표를 모두 달성하는 것은 불가능하므로, 서비스의 목적,목표에 맞춰 설정해야함

#### 1. High Throughput
- Rate that data is moved from producer to brokers or brokers to consumer
- 데이터 이동하는 비율, 즉 주어진 시간 내 처리하는 데이터 양을 늘리는 것

#### 2. Low Latency
- Elpased time moving message end-to-end
- e.g) chat application / interactive web / real-time stream processing
- 데이터의 유입부터 처리까지, end-to-end의 지연시간을 최소로 줄이자

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
- High Throughput을 얻기 위해서는 데이터가 이동하는 비율을 최대화해야하며, 이 데이터 전송률은 가능한 한 빨라야함
- Topic의 partition은 parallelism의 단위이기 때문에 partition이 많을수록 throughput이 높아짐
- partition 수가 많으면 좋다고 막 늘리는 것 보다는 producer와 consumer의 throghput를 고려해야함
[https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster](how-choose-number-topics-partitions-kafka-cluster) 좋은 블로그임

#### Producer 

- producer는 같은 파티션에 대한 데이터를 batch로 처리함. 즉, 여러 message를 하나의 request로 묶어서 전송함
> - 한국 카프카 사용자 모임 발표때 들어보니, 배치로 처리하는 것은 네트워크의 ack, header등을 아낄 수 있다고 함 ( Naggle's algorithm 기반 )
- 그래서, producer 측면에서는 다음과 같은 configuration을 고려해야함

###### 1. batch.size = 100,000~200,000 (default : 16384) / linger.ms = 10~100 (default : 0)

- 한번의 batch request의 데이터 사이즈 / 배치사이즈가 다 채우질 때까지 기다리는 시간
- broker서버의 요청하는 request수가 적어, Producer의 load 및 broker CPU overhead를 줄일 수 있음

- tradeoff 
> - message을 받은 즉시 전송하지 않기 때문에 Latency가 증가함

###### 2. compression.type = lz4 (default : none)
- lz4, snappy, gzip을 지원하며, compression은 전송전 batch처리에 적용되므로, 압축률에 batch 처리의 효율성에 영향이 미침 
- 즉, 배치 비율이 높을수록 압축률이 향상됩

###### 3. ack = 1 (dafault 1)
- batch request는 leader partition에 요청을 보내고, producer는 broker서버에게 ack를 기다림
- ack를 빨리 받으면 받을 수 록, throughput이 증가함

- tradeoff
> -  low duarablity  ( ack =1 ) 

###### 4. retries = 0 (default 0)
- producer가 전송 실패시 재시도 횟수
- reties =0로 설정시 데이터 유실이 있을 수 있음

###### 5. max.block.ms / buffer.memory
- Producer는 보내지 않은 메시지를 메모리 buffer에 저장해둠 ( producer의 accumulator )
- 보내지 않은 메세지를 담은 buffer가 가득차면, max.block.ms가 지나거나, 메모리가 해제되기 전까지 전송을 하지 않음
- partition이 많지 않다면, 이 설정값을 수정할 필요는 없음
- 하지만, partition수가 많으면, buffer.memory, linger time, partition수 등을 고려해야하여, memory를 늘린다면 전송 로직이 멈추지 않아 throughput 증가

#### Consumer 

###### 1. fetch.min.bytes = ~ 100,000 / fetch.max.waits.ms

- leader partition을 가진 broker에게 fetch request시, 얼만큼 데이터를 가져올 것인가로 조정할 수 있음
- 이 값을 올린다면, fetch request 횟수는 줄어드므로 , broker CPU overhead는 감소함 
> - increase Througput / decrease Latency
- fetch request에 fetch.min.bytes를 충족시킬만큼의 메시지가 있거나 fetch.max.wait.ms이 만료 될 때까지 broker는 consumer에게 message를 전송하지 않음
 
##### 2. consumer 개수
- consumer 수가 많으면 parallelism 증가, load 분산, 동시에 여러개 partition 땡기면서 Througput 증가

##### 3. GC 튜닝
- [kafka Recommended JVM Setting](https://docs.confluent.io/current/kafka/deployment.html#jvm)


--- 

### Optimizing for Latency

#### Producer
###### 1. linger.ms = 0
###### 2. compression = none
###### 3. ack = 1

#### Consumer
###### 1. fetch.min.bytes = 1 


#### Broker

###### 1. num.replica.fetchers = increase
- partition이 많으면 많을 수록, throughput 증가하면서 latency도 증가함
- broker는 single thread로 다른 broker에게 데이터를 복제
- 그래서, 하나의 브로커에 파티션이 많을 수록, 복제하는데 시간이 오래걸리고, 그렇게 되면 commit의 delay가 있어 Latency가 증가
- 해결책
> - partition 수를 줄이거나, broker 서버를 증설해서 broker 서버당 할당되는 partition 개수를 줄이자
> - follower partition이 leader partition을 못 따라가면, 하나씩 증가하면서 테스트해봐야함

---


### Optmimizing for Durability

- Kafka는 replication을 통해, message를 여러 broker서버에 복제해 데이터 유실 방지
- 브러커 서버가 문제가 발생해도, 데이터는 다른 브로커 서버에 있어 데이터 접근할 수 있음

#### Producer 
###### 1. replication.factor = 3 ( Topic Level )
###### 2. ack = all
###### 3. retries = 1 /  max.in.flight.requests.per.connection = 1
 - 클러스터의 일시적 오류인해, retry 하게 되면 데이터의 중복이 발생할 수 있음 
  > - 이런 경우에는, consumer에서 중복을 제거하는 로직이 있어야함
 - 동시에 여러개의 send request를 전송하고, retry 하게되면 데이터의 순서가 바뀌는 문제가 발생할 수 있음 
 - max.in.flight.requests.per.connection = 1로 설정하여, 한번에 하나의 request만 보내게 설정하여, 순서 보장할 수 있음

#### consumer
###### 1. auto.commit.enable
 - 어떻게, 얼마나 자주 commit함에 따라 duarlity를 높일 수 있음 
 - commitAsync() , commitSync, 또는 consumer의 RebalanceLinstener를 상속받아 구현 할 것 


#### broker
###### 1. default.replication.factor = 3 / auto.create.topics.enable = false
 - producer가 존재하지 않은 topic에 producing시, auto.create.topic.enable = true 시 토픽이 자동생성
 - 자동 생성된 Topic은 dafault.replication.factor에 따라 복제수가 생김
 - auto.create.topics.enable = false로 하던지, 아니면 default.replication.factor을 3으로 설정할 것 

###### 2. offsets.topic.replication.factor = 3
 - __consumser__offests 토픽의 repliaction.factor에 대한 값

###### 3. min.insync.replicas = 2
 - ack이 all일 경우, min.insync.replica 의 partition들이 offset이 복제완료가 되어야 producer에게 ack를 보냄

###### 4. unclean.leader.election.enable = false
 - broker 서버가 내려가면, 해당 broker의 문제를 파악하고, 해당 broker서버가 가진 leader partition을 다른 partition으로 변경함
 - 새로운 leader를 선정하는 방법은, 해당 partition의 ISR 리스트 중 하나를 leader로 선정하고 나머지 follower들은 새로운 leader partition 정보를 복제 시작
 - unclean.leader.election.enable은 ISR가 아닌 broker 서버를 leader로 선정할 것 인가에 대한 설정값 
 - 이 값은, commit은 되었지만 복제되지 않은 데이터 유실을 방지해 durablity는 증가시키지만, replica가 sync될 때 까지 기다리는 시간이 생겨 availablity 측면에서 손해를 봄
 - no live in-sync replicas면 에러가 나면서 partition을 쓸 수없는 상태가 된다. 즉, data에 대한 durability가 증가한다.


###### 5. broker.rack
 - rack 정보를 입력시 , partition replica를 다른 rack으로 분배함

###### 6. log.flush.interval.message/ log.flush.interval.ms
 - kafka는 replication을 통해 durabliity를 높이면서, OS는 page cache의 데이터를 Disk로 flush할 수 있게 되어 있음
 - 일반적으로 변경할 일은 없겠지만, 토픽이 throughput이 매우 작으면 OS가 disk로 flush하는 시간이 오래걸린다면, 해당 값을 낮게 설정해야함

---

### Optimizing for Avaliablity

- 파티션이 많을 수록 병렬성이 높아지지만, 브로커가 문제가 발생시 복구시간이 더 오래걸림
- 리더가 선정되기 전까지는 producer /consumer 모두 멈춤 

#### broker

###### 1. acks = all / min.insync.replica =2 
- producer가 min.insync.replica를 충족을 못한다면 exepction이 발생함
- 만일 해당 브로커 서버가 내려가 shrink ISR( 축소된, 즉 하나가 없는 상황) 에서 min.insync.replica를 높이 설정 시, producer는 계속 실패를 할 것 이고, 이 것은 해당 파티션의 avaliablty를 감소시킴
- 반면에 너무 낮게 설정하면, durablity 측면에서 손해를 봄


###### 2. unclean.leader.election.enable = true


###### 3. num.recovery.threads.per.data.dir
 - 브로커가 재시작시 , 다른 브러커와 sync를 맞추기 위해 log data file를 scan함, log recovery라고 함 
 - 각 data dir마다 recovery하는 , shutdown시 log를 flush하는 thread개수를 설정하는 값 
 - RAID로 묶여있다면, 이 값을 늘리면 로딩 타임을 줄일 수 잇음 

#### Consumer

###### 1.seesion.timeout.ms
 - 해당 consumer가 문제가 발생했는지 체크하는 시간 
 - 짧을 수 록 빨리 rebalance하여 처리
 - consumer failure은 두가지 타입이 있음
1. soft failure 
> 1. poll 하는 주기가 길어질 경우 ( max.poll.interval.ms )
> 2. JVM GC Long Pause
2. hard failure
> - SIGKILL









