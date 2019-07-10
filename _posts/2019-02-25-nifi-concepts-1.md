---
layout: post
title:  "[NiFi] NiFi Concept #1 "
date: 2019-02-25 11:15:12
categories: NiFi 
author : Jaesang Lim
tag: NiFi
cover: "/assets/instacode.png"
---

## NIFI 

- Apache NiFi supports powerful and scalable directed graphs of data routing, transformation, and system mediation logic.  
- Apache NiFi는 강력하고 확장 가능한 데이터 라우팅, 변환 및 시스템 중재 로직의 방향 그래프를 지원

### Why NIFI? 
- 시스템 간 data flow 자동화 및 관리 목적
- Create, Run, View, Start, Stop, Change, Fix, Dataflows in Real-time
- 드래그 앤 드롭, 스톱/스타트 버튼, 에러 view, 통계량, health 

```
- 호튼웍스 왈,
  - Hortonworks DataFlow는 수많은 소스에서 데이터를 수집하고 전송하는 것과 관련된 실시간 과제를 해결하는 통합 플랫폼이며 Apache NiFi로 지원되는 자동화된 데이터의 출처를 통해 실시간 흐름에 대한 대화형 명령 및 제어를 제공합니다.
  - Hortonworks DataFlow는 모든 유형의 데이터 흐름을 쉽게 자동화하고 모든 종류의 데이터 흐름을 안전하게 보호하는 한편 모든 데이터 및 모든 소스에서 파생된 실시간 비즈니스 인사이트와 작업을 수집, 수행 및 조정하기 위해 설계되었습니다.
```


### Challenges (데이터 흐름 관리 시 어려운 점)

1. System fail
  > - 네트워크 안되고, 디스크 고장나고, 소프트웨어는 충돌 나고, 사람은 사고침
2. Data access exceeds capacity to consume
  > - 데이터 소스가 데이터 처리나 전송 부분에서 오버페이스 할 수 있음
3. Boundary conditions are mere suggestions
  > - 너무 크거나, 작거나, 빠르거나 느리거나, 잘못된 데이터는 항상 들어올 수 있음
4. What is noise one day becomes signal the next
  > - 조직에 우선순위에 따라 data flow를 빨리 만들거나 바꾸거나 해줘야 함
5. Systems evolve at different rates
  > - 시스템들의 발전 속도가 다 다름
6. Compliance and security
  > - 법이나 규정, 정책 등이 계속 바뀜
7. Continuous improvement occurs in production
  > - 프로덕션 환경과 비슷하게 실험하고 싶은데 그렇게 만들기가 어려움

### 특징

- 시스템 간 데이터 전달을 효율적으로 처리, 관리, 모니터링
- 복잡한 시스템 간의 데이터 이동에 NiFi를 이용하여 쉽고, 안전하게 개발 할 수 있다.
- Dataflow를 쉽게 개발할 수 있고, 시스템 간의 데이터 이동과 내용을 볼 수 있는 기능과 UI를 제공한다.
- 실시간 데이터 전송에 필요한 유용한 기능을 제공한다.
- 강력한 자원과 권한 관리를 통해 멀티테넌시(Multi-tenant, 여러 조직이 자원을 공유해서 사용)를 지원한다.
- 데이터가 어느 시스템으로부터 왔는지 추적할 수 있다.
- 국내에는 크게 사용하는 곳이 없지만, 해외에서는 충분한 사례가 있다.
- 오픈 소스이면서, 호튼웍스의 기술 지원을 받을 수 있다.
- 여러 NiFi 시스템 간 통신을 지원한다(Site-to-site)

### High Level Overview of Key NiFi Features

#### 1. Flow Management 
- Guaranteed Delivery
  - Write ahead log와 Content repository 제공

- Data Buffering w/ Back Pressure and Pressure Release
  - 대기중(queued)인 모든 데이터를 버퍼링 할 수있을뿐 아니라 지정된 대기 시간에 도달하면 백 프레셔를 제공하거나 지정된 시간 (데이터가 소멸 된 시간)에 도달하면 데이터를 제거 할 수 있음

- Prioritized Queuing
  - 기본은 오래된 데이터 먼저 들고 오는 것이지만, 경우에 따라 최신 데이터나 가장 큰 데이터를 우선으로 들고 올 수도 있음

- Flow Specific QoS (latency v throughput, loss tolerance, etc.)
  - 절대 유실되어서는 안되거나 혹은 수 초 이내 처리, 전송되어야 할 경우 세분화된 flow 특수 구성이 있음


#### 2. Ease of Use

- Visual Command and Control
  - 복잡해진 dataflow를 시각화하여 단순화시킴. 실시간 수정하고 즉시 적용됨. Dataflow변경을 위해 flow를 중지할 필요 없음

- Flow Templates
  - Dataflow는 패턴지향적인 경향이 있는데 잘한 거 템플릿 제공함.

- Data Provenance
  - 데이터 왔다 갔다 하거나 변환되는 이력을 자동으로 저장하고 인덱싱함. 문제 해결이나 최적화 등에 매우 유용한 정보가 됨

- Recovery / Recording a rolling buffer of fine-grained history
  - 데이터는 content repository가 부족할 때만 지워지므로, 데이터 다운로드나 replay 등 기능을 object lifecycle의 모든 구간에 걸쳐 할 수 있음.

#### 3. Extensible Architecture
- Extension
  - 확장 가능한 것: processors, Controller Services, Reporting Tasks, Prioritizers, and Customer User Interfaces 

- Classloader Isolation
  - 클래스로더 격리로 extension간 외존성 최소화

- Site-to-Site Communication Protocol
  - NiFi 인스턴스 간 통신 가능

#### 4. Flexible Scaling Model
- Scale-out (Clustering)
  - 단일 노드가 초당 수백 메가, 혹은 평범한 클러스터가 초당 기가 이상 데이터 처리할 경우 로드 밸런싱, 페일오버 등 문제 생김. 
  - 그때는 kafka같은 것이 도움이 됨. NiFI의 site-to-site 기능(여러 NiFi간 통신) 또한 괜찮음.

- Scale-up & down
  - 처리량 늘리고 싶으면 프로세서에서 concurrent task 수 증가시키면 됨. 


### Some Use case
- Building Ingestion and Delivery layers in IoT Solutions 
- Ingestion tier in Lambda Architecture (for feeding both speed and batch layers) 
- Ingestion tier in Data Lake Architectures 
- Cross Geography Data Replication in a secure manner 
- Integrating on premise system to on cloud system (Hybrid Cloud Architecture) 
- Simplifying existing Big Data architectures which are currently using Flume, Kafka, Logstash, Scribe etc. or custom connectors. 
- Developing Edge nodes for Trade repositories. 
- Enterprise Data Integration platform 
- And many more…

---


###  다른 블로그 참고 내용

> 참고 : https://community.hortonworks.com/content/kbentry/7882/hdfnifi-best-practices-for-setting-up-a-high-perfo.html

### Databasae Repository ( H2 Setting )

```scala
nifi.database.directory=./database_repository
```

- NiFi는 2개의 H2 DB를 사용함 
  > - 1. User DB - keep track of user login
  > - 2. History DB - keep track of all change made on the graph
- NiFi 설치한 root 디렉토리에 설치
- 다른 디렉토리로 옮긴다 해서 성능상 차이는 없음
- 하지만 NiFi 업그레이드 후 사용자 및 구성 요소 히스토리 정보를 유지하기에 용이하게 이동하는 것을 추천

### FlowFile Repository
- NiFi UI의 모든 FlowFile들의 상태를 유지 관리
- FlowFile Repo가 문제가 생길시, NiFi가 현재 작업하고 있는 일부 또는 모든 파일에 대한 Access 권한을 잃을 수 도 있음
- 대부분의 문제가 발생하는것은 Disk 공간 부족
- 고성능 시스템(?)에서는 FlowFile Repo를 Content Repo와 Provenace Repo와 같은 디스크에 위치시키면 안됌 

- NiFi는 프로세스로 Physical file(content)을 직접 넘기지 않음
- FlowFile은 한 프로세서에서 다음 프로세서로의 전송 단위 역할

- FlowFile은 JVM 메모리에 있음
- 프로세스 간의 Connection 사이의 queue에 데이터가 계속 쌓인다면 JVM OOM 에러가 발생할 수 있음
- 그래서 NiFi에서는 일정한 flowfile 개수를 디스크로 swap 하는 설정값이 존재함

```scala
nifi.queue.swap.threshold=20000
```

- Queue에 들어가는 flowfile이 2만개가 넘어가면 디스크로 swap이 발생하고 얼마나 swap이 발생함에 따라 성능에 영향을 줌


### Content Repository
- 실제 데이터(content)가 저장되는 곳 
- NiFi 단일 instance 내에 여러가의 repo 설정 가능

```scala
nifi.content.repository.directory.contS1R1=/cont-repo1/content_repository
nifi.content.repository.directory.contS1R2=/cont-repo2/content_repository
nifi.content.repository.directory.contS1R3=/cont-repo3/content_repository
```

- *.contS1R1 은 내가 원하는 걸로~ ( 기본은 default라고 적혀있음 )
- 여러 디스크에 걸쳐 I/O로드를 나누면, 장애 발생시 상당한 성능 향상과 내구성을 얻을 수 있음
- 클러스터 환경에서 구축 시에는 Content Repo이름을 모두 다르게 설정하게 disk 사용률를 디버깅할 때 용이하게 하자

### Provenance Repository
- Content Repo와 비슷하게 많은 양의 provencance event를 Read/Write 작업을 진행함
- Dataflow의 모든 transaction이 기록 



