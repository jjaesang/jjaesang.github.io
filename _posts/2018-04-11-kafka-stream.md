---
layout: post
title:  "Stream Processing In Kafka"
date: 2018-04-11 13:05:12
categories: Kafka
author : Jaesang Lim
tag: Spark
cover: "/assets/kafka_log.png"
---

### Stream Processing In Kafka

#### 다룰 내용들~
1. What is Stream Processing 
2. Stream Processing Concept
3. Stream Processing Design Pattern
4. Kafka Streams
5. How to Choose a Stream-Processing Framework

#### 1. What is Stream Processing 

##### 1_1. Data Stream 

- Event Stream 또는 Streaming Data 로 부르기도 함
- 속성
	1. Unbounded Dataset
		- 시간의 흐름에 따라 무한히 계속 증가하는 데이터
	2. Event Streams are Ordered
	3. Immutable data record
	4. Event Streams are replayable
		- 하루, 한 달 전 데이터를 다시 처리할 수 있어야함
		- Kafka가 Event Stream에 대해 Capturing & Replaying 할 수 있으니 걱정 노!

##### 1_2. Programming Paradigm

1. Request - response
	- Low Latency
	- Blocking
	- OTLP ( Online Transaction Processing )
	- POS 시스템, 신용카드 처리 
	
2. Batch Processing
	- High Latency / High throughput
	- Data warehouse / BI 
	
3. Stream Processing
	- Non Blocking
	- Request-Response 와 Batch Processing의 중간

<hr/>

#### 2. Stream Processing Concept

##### 2_1. Time
Streaming에서의 Time을 선정할 수 있는 요소 및 가장 좋은 Time의 개념에 대한 설명 

- Event Time ( Good ! ) 
	- 이벤트가 발생한 시간 
	- 이벤트 발생 후 데이터베이스 레코드를 기반으로 Kafka 레코드가 생성 된 경우에는레코드에 대한 Event Time을 기록
	- Stream 처리시에서 가장 중요하게 여겨지는 Time 개념

- Log Append Time ( So So.. )
	- Kafka Broker에 도착된 시간
	- Event Time에 비해 Stream 처리 시, 크게 관련성이 적음

- Processing Time ( Bad ! )
	- Stream Processor가 Event를 받은 시간 
	- Event 발생 후, 1초 , 1분, 하루 이상이 될 수 도 있음
	- 가장 피하는 것이 좋음 

##### 2_2. State
이벤트간에 저장된 정보 state!

- State
	- Event 들 간에 저장된 정보 
	- 이벤트를 개별적으로 처리하는 것은 매우 간단
	- 여러 개의 이벤트에 대해 처리 할 때는 각 이벤트에 대한 State 관리를 해야함
	- ( e.g counting, moving average,  join two stream )  
	
- Local or Internal State
	- Stream application 내부에서 관리 ( e.g ) Hash Table )
	- 메모리상에 관리하기 때문에 빠른 접근 
	- 양이 많아지면 메모리 이슈 발생 가능
	
- External State
	- 외부 DB에서 관리 NoSQL ( Hbase / Cassandra )
	- 안정적인 state 관리 및 외부, 다른 Application에서 State를 참조할 수 있음 
	- 외부 DB인 만큼 Latency 발생 및 복잡성

##### 2_3. Stream-Table Duality
Table & Stream 을 동시에 관리하자

- Table
	- 현재시점의 데이터 
- Stream 
	- 이벤트의 지속적인 변화
- Table -> Stream
	- CDC ( Change Data Capture ) 	
	- Insert / Update / Delete 연산의 히스토리
	- e.g) Kafka Connect 

- Stream -> Table 
	- materializing
	<center><img src="https://user-images.githubusercontent.com/12586821/47949851-4ec95400-df8d-11e8-9b2c-51ee45e74997.png"/></center>

##### 2_4. Time Window
대부분의 Stream 처리는 Window 연산 
Window 연산에서 두가지 설정해야하는 것이 있음
1. Size of the window
2. How Often the window moves ( = advance Interval ) 

- tumbling window
	- window size == advance interval
	<center><img src="https://user-images.githubusercontent.com/12586821/47949859-50931780-df8d-11e8-91f1-ac06de84c6c0.png"/></center>

- Hopping Window  ( Overlap )
	- window size > advance interval 
	<center><img src="https://user-images.githubusercontent.com/12586821/47949853-4ec95400-df8d-11e8-91ff-67b898f787bf.png"/></center>
    
- sliding window ( Overlap )
	- 이벤트가 들어올 때 마다 처리
	<center><img src="https://user-images.githubusercontent.com/12586821/47949856-4f61ea80-df8d-11e8-90b4-408ab0e2941e.png"/></center>

<center><img src="https://user-images.githubusercontent.com/12586821/47949852-4ec95400-df8d-11e8-837c-c8b3d6926757.png" /></center>


<hr/>


#### 3. Stream Processing Design Pattern

1. Single-Event Processing
2. Processing With Local State
3. Multiphase Processing/Repartitioning
4. Processing with External Lookup : Stream-Table Join
5. Streaming Join
6. Out-of-Sequence Event
7. Reprocessing


##### 3_1. Single-Event Processing

- Stream Event를 독립적으로 처리 ( = map / filter ) 
<center><img src="https://user-images.githubusercontent.com/12586821/47949857-4ffa8100-df8d-11e8-80ce-ed5990690b00.png"/></center>

##### 3_2. Processing with Local State
- Stream Processing 은 Time Window 기반 Aggregation
- Stream의 State 관리 
- 매일 최소, 최대 값을 계산한다면,현 시간까지의 최소,최대값을 저장 후 스트림의 결과와 비교 해야함

<center><img src="https://user-images.githubusercontent.com/12586821/47949854-4f61ea80-df8d-11e8-92b0-3ed381be0bc7.png"/></center>

- 이 모든 작업은 로컬 상태 (공유 상태가 아닌)를 사용하여 수행 할 수 있음.
- Kafka 파티셔너를 사용하여 동일한 주식 기호가 있는 모든 이벤트가 동일한 파티션에 기록할 수 있음
- 할당 된 파티션에서 모든 이벤트를 가져와서 처리가능
- 즉, 응용 프로그램의 각 인스턴스는 할당 된 파티션에 쓰여지는 주식 기호의 하위 집합에 대한 상태를 유지할 수 있음


##### 3_3. Multiphase Processing/Repartitioning
- e.g) 하루동안 거래량 많은 Top 10의 주식을 구할 때
<center><img src="https://user-images.githubusercontent.com/12586821/47949855-4f61ea80-df8d-11e8-94b1-3ea75361b0d5.png"/></center>

##### 3_4. Processing with External Lookup : Stream-table Join
- Stream 처리시, 다른 외부 데이터 참조할 때
- Click 로그가 Stream으로 들어오고, Click한 사용자의 정보를 외부 데이터를 참조할 경우, 성능을 위해 cache

<center><img src="https://user-images.githubusercontent.com/12586821/47949858-4ffa8100-df8d-11e8-949c-7f1e358d1761.png"/></center>


</hr>

#### 4. Kafka Streams

1. Real time , Stateful Stream Processing 
2. Library, not a framework
3. low-level Processor API / high-level Stream DSL 
4. KStreamBuilder를 이용하여 Stream의 topology ( = DAG ) 생성
5. KafkaStreams Object 생성하여 , 위에서 선언한 topology 실행


##### 4.1 WordCount example 
<center><img src="https://user-images.githubusercontent.com/12586821/47950040-124b2780-df90-11e8-9c38-f1690cc0e91f.png"/></center>
##### 4.2 ClickStream example 
<center><img src="https://user-images.githubusercontent.com/12586821/47950039-124b2780-df90-11e8-92e1-9d44a5e350d4.png"/></center>

#### 6. How to Choose a Stream-Processing Framework

1. Ingest
	- 시스템에서 타깃 시스템으로 데이터를 전달하는 경우, 타깃 시스템에 적합한가

2. Low milliseconds actions
	- Fraud Detection 같은 즉각적인 반응이 필요한가

3. Asynchronous microservices
	- Updating the inventory of Store ( Local Cache ) 

4. Near real-time data analytics
	- Aggregation과 Join를 통한 빠르게 데이터로 부터 Insight를 얻고 싶은가 

5. Operability of the System
	- 배포 및 모니터링를 쉽게 할 수 있는가? 사용 중인 플랫폼과 연동할 수 있는가? 실수가 있을 시, 재분석을 할 수 있는가?

6. Usability of APIs and ease of debugging
	- 같은 Framework여도 버전에 따라 많은 update와 변화가 생긴다.. 
 
7. Makes hard things easy
	- Advanced Window 연산 및 local cache 관리 등 쉽게 할 수 있도록 제공하는가?

8. Community
	- Coummunity가 활성화되어 있는가?