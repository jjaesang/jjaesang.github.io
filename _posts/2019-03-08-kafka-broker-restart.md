---
layout: post
title:  "Kafka 브로커 재시작 프로세스 및 변화"
date: 2019-03-08 13:05:12
categories: Kafka
author : Jaesang Lim
tag: Spark
cover: "/assets/kafka_log.png"
---

- 카프카서버의 랙이전 작업을 하면서, 브로커 서버를 중지 후, 재시작하면서 작업한 내용 

### 1. kafka 프로세스 중지


- 해당 브로커 서버에 있는 Leader Partition은 ISR의 Follower Partition로 leadership이 넘어가면서, producing / consuming 요청 처리에 문제 없음
- Leader에서 Follower로 이전하는 과정에서는 Exception이 떨어질 수 있음
- 브로커 서버가 내려가자마자, 즉시 leadership이 변경되면서 Delay 없음
- 리더가 변경되면서 일부 Connection Error 가 발생할 수 있다고하나, 현재 시점에서 발생하지 않음

### 2. 랙 정보 수정 및 kafka 프로세스 재실행

- Broker 재시작될 때, 모든 Partition은 Preference Leader를 자동으로 선출 
- auto.leader.rebalance.enable 설정시, restart시 자동으로 시작 ( default : true)
- leader.imbalance.check.interval.seconds ( default : 5m )
- leader.imbalance.per.broker.percentage ( default : 10 )
- 5분 기다려야 leadership이 재할당 된다고 하나, 재할당 스크립트 실행

### 3. partition leadership 재분배


- 
```
sh kafka-preferred-replica-election.sh --zookeeper zklist
```

- JSON 파일 없을 시, ZKClient로 모든 토픽의 Partition 정보를 다 가져옴
- 장애 발생한 브로커가 정상화되고 난 후 수동으로 파티션의 리더를 복원 
- 실행한지 약 3초만에 기존의 1번 브로커 서버가 leader로 있던 partition이 다시 1번 브로커서버의 leader로 재할당 

