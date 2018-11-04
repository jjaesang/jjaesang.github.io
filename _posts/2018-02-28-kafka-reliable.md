---
layout: post
title:  "Reliable Data Delivery In Kafka"
date: 2018-02-28 13:05:12
categories: Kafka
author : Jaesang Lim
tag: Kafka
cover: "/assets/kafka_log.png"
---

### Reliable Data Delivery In Kafka

#### 다룰 내용들~
1. Reliability Guarantees
2. Replication
3. Broker Configuration
4. Using Producers in a Reliable System
5. Using Consumers in a Reliable System

#### 1. Reliability Guarantees

- Reliable System ?
	- 데이터의 유실 없이 최소화되어
	- 안정적인 전달이 보장되는 시스템

- Guarantee ? 
	- partition 내의 messages의 순서
	- commit된 messages는 partition의 모든 in-sync replicas에 쓰여짐
		-acks (producer, default = 1)
		-min.insync.replicas (broker/topic, default = 1)
	-하나 이상의 replica가 살아있는 한, messages는 손실되지 않음
	-consumers는 commit된 message만 읽을 수 있음


- 이러한 기본적인 guarantees로 어느정도 reliable system을 구성할 수 있음
- 하지만, 완벽하지 않으며 configuration parameter를 이용하여 시스템에서 필요한 reliability를 결정해야 함
- reliability ↑ : durability ↑ throughput↓ latency↑ hardware costs↑

<hr/>

#### 2. Replication
- Multiple Replicas를 가지는 구조는 Kafka의 일부에 장애가 발생했을 때도,
- Reliability와 durability를 제공

- In-Sync Replica
	1. leader replica
	2. 아래의 조건을 만족하는 followers
		- Zookeeper에 n초안에 heartbeat를 보냄 (zookeeper.session.timeout.ms = 6s)
		- n초안에 messages를 leader로부터 fetch (replica.lag.time.max.ms = 10s)
		- n초안에 leader에서 가장 최신 데이터를 fetch

- In-Sync Replica와 성능의 관련성

- in-sync replicas가 조금씩 뒤쳐져 있다면?
	- producer와 consumer를 느리게함
- in-sync replicas의 수가 적으면 ?
	- partition의 장애나 데이터 손실에 대한 위험성이 더 높아짐

* 참고
* out-of-sync replica가 뒤쳐져 있다면 producer와 consumer의 performance에 영향이 없음
* producer와 consumer가 out-of-sync replica가 messages를 얻기까지 기다리지를 않기 때문

<hr/>

#### 3. Broker Configuration

1. Replication Factor
2. Unclean Leader Election
3. Minimum In-Sync Replicas

##### 3_1. Replication Factor

- topic을 생성할 때 기본 replication 개수
- higher replication factor
	- availability ↑ reliability ↑ disaster ↓ disk space ↑
	- availability가 중요한 경우에는 3이상을 추천
- replicas의 위치
	- 기본적으로 Kafka는 partition의 replicas가 각각 다른 broker에 있는지 확인
	- replicas를 다수의 rack에 배치해야 하고 rack이름을 설정
		- broker.rack (broker, default = null)

##### 3_2. Unclean Leader Election

- Clean Leader Election
	- partition의 leader가 더이상 available할 수 없을 때, in-sync replicas 중 하나가 leader로 선출되는 경우

- Unclean Leader Election 
	- 모든 replica가 죽으면?
	- leader외에 in-sync replicas가 존재하지 않을 때, leader가 사용불가능 하게 되어 new leader를 선출해야 하는 경우에 Unclean Leader Election을 허용할지 말지를 결정할 수 있음

- out-of-sync replica를 leader로 선출할지의 여부
- Disable
	- data의 quality와 consistency와 durability가 중요한 경우
	- bank system
- Able
	- availability가 더 중요한 경우
	- real-time click analysis


##### 3_3. Minimum In-Sync Replicas
- in-sync 되어야 하는 replica의 최소 개수
- 이를 충족시키지 못하면
	- produce request를 거절 후 ,NotEnoughReplicasException을 수신
	- consumer는 존재하는 data를 계속 읽을 수 있음

|| broker configuration parameter  | topic configuration parameter  | default value  |
|--------|--------|--------|--------|
|    Replication Factor    		|   default.replication.factor     |    replication.factor    |      1  |
|    Unclean Leader Election   |    unclean.leader.election.enable    |unclean.leader.election.enable    |   true (0.11.0이후는 false)    |
|    Minimum In-Sync Replicas    |     min.insync.replicas   |    min.insync.replicas |     1   |


<hr/>

#### 4. Using Producers in a Reliable System

1. Send Acknowledgments
2. Error Handling : Configuring Producer Retries
3. Error Handling : Additional Error Handling

- Producer가 broker에게 Produce Request를 보내면 성공/Error가 반환
- Configuring Producer Retriesprducer가 자동적으로 retry함으로써 error handle
- Additional Error Handlingdeveloper가 producer library를 사용하여 처리

##### 4_1. Send Acknowledgments

- acks = 0
	- producer가 network를 통해 message를 보내기만 하면 성공으로 간주
	- latency↓  throughput↑ reliability↓
- acks = 1
	- message가 leader replica에 쓰이면 성공으로 간주
	- message가 성공적으로 leader에 쓰여서 produce request가 성공으로 간주되었지만, follower에게 복제되기전에 leader에 장애가 발생하면 데이터가 유실됨
- acks = all
	- min.insync.replica 값의 in-sync replicas에서 message를 복제하면 성공으로 간주
	- latency↑ throughput↓ reliability↑

##### 4_2. Configuring Producer Retries

- Produce Request를 하였을 때 Retriable Error가 발생한다면?
- messages를 손실하지 않기 위해서 messages를 계속해서 재전송하도록 producer를 configure

- Retires
	- producer configuration
	- default value : 1
	- Retriable Error가 발생했을 때, producer가 자동으로 retry하는 횟수

- Retry 횟수의 선정 방법
	- producer가 n번 retry 후에 throw exception을 하였을 때, 수행할 계획에 따라 달라짐
	-  exception을 catch한 후에 또다시 retry할 것이라면 retries를 늘려야 함

##### 4_3. Additional Error Handling
- Nonretriable broker errors
	- Messages를 Broker에 보내기 전에 발생하는 errors
		- Nonretriable broker errors (message size, authorization errros 등) broker에 mssages를 보내기 전에 발생하는 errors (serialization error 등)
	- Producer가 Retry Attempt를 모두 소비했을 때, Available Memory가 Limit에 도달하는 경우
	- developer가 직접 처리해야 함

- error handlers를 작성하여 application의 목표에 맞도록 error를 처리
	- messages를 drop?
	- error를 log?
	- local disk의 directory에 message를 store?
	- another application으로 callback을 trigger? 등

##### 4_4. Message delivery guarantees

- At most once
	- message가 손실되더라도 재배달되지 않음
- At least once
	- message가 손실되지 않지만 재배달 될 수 있음
- Exactly once
	- message는 딱 한번만 배달 됨

- Message Retry가 발생하는 경우
	- broker에 message를 쓰지 못한 경우
	- broker가 producer에게 응답하는 과정에서 실패한 경우 
	- ( = message가 여러번 broker에 쓰여짐 )
	- broker에 message가 쓰여지는것이 실패한 경우 외에도 broker가 producer에 응답하는 과정에서 실패할 때도 message를 재전송하게 됨

- At least once (Kafka 0.11.0 version 이전)
	- consumer에서 처리
	- message를 idempotent(멱등원)로 생성
	- Retries 및 careful error handling으로 각 message가 적어도 한 번 저장되는것은 guarantee되지만, message가 정확하게 한 번 저장되는 것을 보장할 수 는 없음
	- message마다 unique한 identifier를 사용하여 messages를 consume할 때 중복을 감지하고 정리
	-  message를 idempotent로 만들어서 같은 message가 2번 보내지더라도 부정적인 영향이 없도록 함


- Exactly once (Kafka 0.11.0 version 이후)
	- producer가 idempotent delivery option을 지원
	- broker는 각 producer에게 ID를 assing하고, producer가 모든 message를 sequence number와 함께 전송하여 duplicate를 제거
	- producer는 transcation과 같은 semantics를 이용하여 multiple topic partitions에 messages를 보내는 기능을 지원하므로, 모든 messages가 write되거나 write되지 않음

<hr/>

#### 5. Using Consumer in a Reliable System

##### 5_1. Commited Offsets
	
- 특정 partition의 특정 offset까지의 모든 messages를 받고 처리했음을 의미
- offsets을 commit하는 시기와 방법에 주의해야 함
	- 읽고 아직 처리하지 않은 offsets을 commit하면 데이터 유실이 발생할 수 있음

##### 5_2. Consumer Configuration

- group.id (“”)
	- 해당 topic의 모든 messages는 전체 group에 의해 나뉘어져 읽어짐

- offset.reset (latest)
	- earlist : 중복처리의 가능성↑ reliable↑
	- latest : 중복처리의 가능성↓ reliable↓

- enable.auto.commit (true)
	- autocommit을 사용하면 : 중복처리의 가능성 존재, 데이터 처리가 보장되지 않음
	- consumer가 일부 records를 처리하고 중단이 되면 commit이 되기 전이기 때문에 처리된 일부 records가 재처리됨
	- consumer poll loop안에서 consume된 모든 record를 처리하면 처리하지 않은 offset이 commit되지 않음

- auto.commit.interval.ms (5000)
	- commit frequency가 증가하면 : overhead↑, 중복처리 데이터 개수↓


- Commit은 언제하나?
	- 데이터가 처리된 후에 offsets을 commit
	- Consumer가 종료하기 전에 partitions의 offset을 commit하고, 해당 partitions이 reassign될 때는 기존의 상태를 가져와야 함

- Commit의 빈도는?
	- Commit의 빈도를 performance와 중복처리간의 trade-off를 고려하여 선정
	- commit의 빈도가 높아지면 성능이 줄어들지만, 장애발생시 중복처리되는 데이터의 개수가 줄어듬
	- commit의 빈도가 낮아지면 성능은 높아지지만, 장애발생시 중복처리되는 데이터의 개수가 늘어남
	- poll loop안에서 여러번 commit하거나, 여러개의 loops에 대해 한번만 commit하도록 구성할 수 있음

- Consumer의 Retry가 필요한 경우
	- buffer를 이용하여 retry
	- retry topic을 이용하여 retry

- Consumer는 상태를 유지해야 할 수도 있음
	- offset을 commit하는 동시에 state를 별도의 topic에 기록
	- consumer가 재시작될 때 해당 tpic에서 state를 가져와서 중단된 부분부터 재시작
	- Kafka 0.11.0 이전에서는 transaction이 제공되지 않기 때문에 데이터 중복처리 혹은 유실 가능성 존재 
	- (Kafka 0.11.0 이후에서는) transcation을 지원

- 긴 데이터 처리 시간이 걸리는 경우
	- Kafka 0.10.1 이전에서는 broker에게 heartbeats를 보내기 위해 poll을 계속 해야함
	- multiple threads로 병렬 처리
	- worker threads로 data를 전달 한 후 work thread가 끝날때까지 pause, 끝나면 resume

- Exactly-once delivery
	- Kafka 0.11.0 이전에서는 지원되지 않으므로 직접 처리
	- Unique key를 지원하는 시스템에 결과를 저장
	- transaction이 있는 시스템에서 records와 offsets을 저장


<hr/>
* 참고
	- Kafka 0.10.1 이전 version에서는 추가적인 records를 처리하지 않으려고 하더라도, broker에게 heartbeats를 보내기 위해 poll을 계속 해야 함 
	-  데이터 처리시간이 너무 오래 걸리면 broker에게 heartbeats를 보낼 수 없어 rebalance가 trigger됨
 	- consumer를 pause하면 additional data의 fetch없이 poll이 유지됨

	- Kafka 0.11.0 이전 version에서는, exactly-once를 지원을 제공하지는 않지만 Kafka가 정확하게 한 번 외부 시스템에 쓰여지도록 보장하는 few trick을 사용할 수 있음
		1. unique keys를 지원하는 system에 results를 write (idempotent(멱등원) write)
			-  record를 unique key와 함께 value로 쓰고, 나중에 실수로 같은 record를 다시 consume하면 정확하게 같은 key와 value를 write하게 됨.
			- data store는 기존의 것이 override되고 중복처리되지 않은 경우와 동일한 결과를 얻음
	
		2. transactions이 있는 system에서 records와 offsets을 write
			- 동일한 transcation에서 records와 offsets을 write하여 in-sync되도록 함
			- 시작 시, external store에 write되어 있는 최신 records의 offsets를 retrieve(검색)한 후 consumer.seek()를 사용하여 해당 offset에서 consume을 시작













