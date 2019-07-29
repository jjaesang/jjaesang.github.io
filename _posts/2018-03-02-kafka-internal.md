---
layout: post
title:  "Kafka Internal"
date: 2018-03-02 13:05:12
categories: Kafka
author : Jaesang Lim
tag: Spark
cover: "/assets/kafka_log.png"
---


### Kafka Internal

#### 다룰 내용들~
1. Kafka Controller
2. Kafka Replication work
3. Request Processing
4. Kafka Physical Storage


#### 1. Kafka Controller

 - 일반적인 broker 기능 + partition 관리자
 - partition과 replica들의 상태 관리
 - partition 재할당 관리
 - zookeeper /controller, /controller_epoch
 
 
broker는 고유한 id을 가지고 있고, 이 id기반으로 주키퍼에 연결하여, ephemeral node로 연결
> - emphemeral node여서 어떤 이유로 connection이 끊어지면 삭제되고, watch에 의해 broker가 사라진 것을 다른 브로커들도 알게됌
> - 해당 broker을 다시 재시작하면, broker ephemeral node에느 사라지지만, isr같은 다른 자료구조에서 해당 노드가 기존에 있던 브로커임을 인지하고 재시작함

controller는 먼저 등록한 Broker로 할당
> - 만약 broker가 사라지면, controller의 watch로 인지하고 다른 Broker이 controller로 연결
> - 이미 할당되었다면 node already exist로 1대만 controller로 split brain을 막음
> - 모든 broker는 controller epoch number로 관리

 - partition 재할당 관리
	- broker 삭제
		- 삭제된 broker가 가지고 있던 leader partition들의 새로운 leader 필요
		- 새로운 leader가 필요한 모든 partition 검사,  어떤 partition이 새로운 leader가 될 지 결정
		- 모든 broker에 해당 정보를 포함한 request 요청

	- 새로운 broker 추가
		- 새로운 broker가 추가됐음을 기존의 broker들에게 알림
		- broker id를 통해 broker에 replica가 있는지 check
		- 만약 replica가 있다면 새로운 broker는 기존에 존재하는 leader로부터 message를 복제

<hr/>
#### 2. Kafka Replication work

- two type of replicas가 존재함
	1. leader replica
		- 모든 request는 leader를 통해 수행 (일관성 보장)
	2. follower replica
		- leader가 아닌 모든 replica
		- request 처리 x
		- 오직 leader로부터 message 복사만 수행

- fetch request
	- follower가 leader와 sync 상태를 유지하기 위해 보내는 request
	- consumer request와 동일
	- 다음에 받기를 원하는 message의 offset이 포함
		- leader는 offset을 보고 follower가 어디까지 message를 받았는지 확인 가능


- replica가 isr인지 판단하는 것은 replica가 leader에게 요청함 Fetch request
> - 이게 일정 시간동안 안오거나, 최신 Offset이 아닌 값에 대한 fetch request할 시, out-of-sync로 간주
> - replica.lag.time.max.ms -> 10초

- preferred leader
> - 토픽 만들고 처음에 할당한 leader partition
> - 이 정보를 계속 가지고 있는 이유는, 초기에 토픽을 만들 때 브러커당 partition 분배를 균형있게 해놔서
> - auto.leader.rebalance.enable = true
> - prefered leader가 현재 리더가 아닌데, isr이면 prefered-leader가 current leader로 변경
> - isr의 첫번째 partition


#### 3. Request Processing

broker은 여러 thread와 두개의 queue를 가짐
1. acceptor thread
2. processing thread

1. request queue
2. response queue

client의 acceptor thread을 통해 연결을 맺고, processor thread가 Request을 Request Queue에 저장
> - 또 processor thread는 request queue에 있는 요청을 받아 처리하고 response queue에 저장



- Request Protocol
	- Kafka request의 형식 및 응답 형식에 대한 binary protocol
	- request includes standard header: 
		1. Request Type
		2. Request version
		3. Correlation ID (=request 고유 번호)
		4. Client ID

- 대표적인 Request

	1. Producer Request
		- producer가 broker에 보내는 요청
		- client가 작성한 message를 포함
			
			2. Fetch Request
		- consumer와 follower replica가 보내는 요청
		- broker의 message를 fetch

##### 3_1. Metadata Request

- client가 어디에 요청을 보낼 지 어떻게 알 수 있을까?
- metadata request는 client가 관심있는 topic list를 포함
- metadata request를 받은 서버는 그 topic에 해당하는 partition의 정보(leader 위치 등)를 반환
- client는 metadata를 cache, request 실패 시 갱신
	- 모든 broker가 metadata를 cache, 어떤 broker로든 요청 보낼 수 있음
	- cache refresh interval은 metadata.max.age.ms로 조정 가능
<center><img src="https://user-images.githubusercontent.com/12586821/47949422-fe9ac380-df85-11e8-9094-fd9e3f0d951a.png"/></center>

##### 3_2. Producer Request

1. validation check
	- leader partition이 request를 받으면, request 전에 선행
	- client가 해당 topic에 쓰기 권한이 있는가?
	- request에 지정된 acks(ch3)가 유효한가?(0, 1, all만이 허락됨)

2. message durability guarantee
	- linux에서 kafka message는 filesystem cache에 쓰이며,
	- disk에 쓰이는 것이 보장되지 않음
		- 시스템 down 시 메시지 휘발
	- Kafka는 replica를 통해 message durability 보장

* 참고
 - leader partition에 message가 기록되면, broker는 acks configuration을 검사
 - acks가 0또는 1로 설정되어있다면 broker는 즉시 응답
 - all로 설정되어 있다면 request는 leader가 follower가 message를 복제한 것을 관찰할 때까지 purgatory(사전적 의미는 일시적 장소)라 불리는 buffer에 저장
 - 든 follower가 message를 복제한 시점에 client에 응답

##### 3_3. Fetch Request

- 특정 topic의 partition과 offset을 지정하여 message 요청

1. validation check
	- producer request validation check와 유사하게 처리
	- offset이 특정 partition에 대해 존재하는가?
	- 요청 message가 삭제될 만큼 너무 오래 되진 않았는가?

2. zero-copy message return
	- message를 file(linux file cache)에서 중간 buffer 없이 네트워크 채널로 직접 전송
	- bytes 복사 및 buffer 관리 overhead 감소, 성능 향상

3. message size upper/lower bound
	- upper bound
	- lower bound
		- traffic이 적은 topic을 읽을 때 CPU와 네트워크 사용량 감소
		- 다른 batch와 마찬가지로 size+time을 기준으로 전송 

<hr/>

#### 4. Physocal Storage

- kafka 기본 저장단위 : partition replication
	- 한 partition은 여러 개의 broker로 쪼개질 수 없음
	- 한 broker 내의 partition은 여러 디스크에 나눠 저장될 수 없음

#### 4_1. Parition Allocation
- 목표
	- broker 간의 균등한 replica 분산
	- 한 partition의 replica들은 각각 다른 broker에 존재(같은 partition은 한 broker 내에 x)

- partition allocation process
	- random하게 broker 시작, round-robin 방법으로 partition leader 할당
	- replica는 leader가 있는 broker와 일정한 간격의 broker에 배치

- broker 추가 시, auto partition rebalancing 없음
	- replica reassignment tool 사용 
	- bin/kafka-reassign-partitions.sh ( kafka manager로 편하게 할 수 잇음!)
- 참고
	- 5개의 브로커에 레플리카 3의 파티션 5개를 을 할당한다고 할 때, 0번 리더가 2번에 할당되고, 1번 리더가 4번에 할당되고, 2번 리더가 0번 브로커에 할당되고, and so on …
 	- 파티션 0의 리더가 브로커 4에 있으면 첫번째 팔로워는 브로커 0에 있고 두번째 팔로워는 브로커 1에 있고..and so on

#### 4_2. File Management

- Retention
	- Kafka는 message를 영구히 저장하지 않음
	- consumer가 메시지를 읽을 때까지 기다리지 않음
	- retention period에 따라 message 삭제
		- log.retention.hours: message 수명
		- log.retention.bytes: partition의 최대 물리적 크기 

- Segment
	- 대용량 파일에서 삭제할 메시지를 찾고 삭제하는 것은 overhead가 큼
	- 각 partition을 segment로 분할
	- segment가 한도에 도달하면 현재 쓰고있는 파일을 닫고, 새 파일을 열어 작업 수행
	- 현재 작성중인 활성(active) 세그먼트는 절대 삭제되지 않음
		- retention기간보다 오래된 데이터가 삭제되지 않을 수 있음에 주의

- Index
	- 어떻게 broker는 유효한 offset의 message를 빠르게 찾을 수 있을까?
	- 주어진 offset의 메시지를 빠르게 찾기 위해 각 partition에 index를 유지/관리
	- 세그먼트 파일과 파일 내의 위치에 대한 offset을 mapping
	<center><img src="https://user-images.githubusercontent.com/12586821/47949501-964ce180-df87-11e8-9cab-58cc2ed5fc2a.png"/></center>
- Retention Compaction
	- 오래된 message를 삭제하는 것이 아닌, 가장 최신의 값만 저장
	- key/value message를 생성하는 응용프로그램에서만 가능
		- null key가 있다면 compaction 실패

	- two log sectio:
		1. Clean Section
			- 이전에 압축된 message
			- 각 key에는 압출될 때 가장 최신이였던 하나의 값만이 포함
		2. Dirty Section
			- 마지막 압축 이후 쓰여진 messages
<center><img src="https://user-images.githubusercontent.com/12586821/47949500-964ce180-df87-11e8-9e96-bae1e51ea1ea.png"/></center>

- Log Compaction Details
    1. 전체 partition에서 dirty message의 비율이 가장 큰 partition을 선택, 압축 수행
    2. dirty message들에 대해 offset map 작성
        - map entry는 key에 16byte hash, message에 8byte offset으로 구성
        - segment 압축에 오직 24byte per entry 사용
    3. clean message를 오래된 순서로 읽으며, offset map과 비교/검사
        - clean message의 key가 map에 없다면, 그 message이후로 들어온 같은 key의 message가 없다는 것이므로 message를 replacement segment에 복사
        - 반대로, key가 map에 있다면, 이후에 해당 key에 신규 message가 입력됐다는 것이므로 그 message는 무시
    4. message 복사가 모두 끝나면 원본 segment와 replace segment를 swap하고 다음 segment로 이동



