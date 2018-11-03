---
layout: post
title:  "Building Data Pipeline In Kafka"
date: 2018-03-21 13:05:12
categories: Kafka
author : Jaesang Lim
tag: Spark
cover: "/assets/kafka_log.png"
---

### Building Data Pipeline In Kafka

* Kafka Connect를 사용하여 파이프라인을 구축하는 방법에 대한 챕터지만,
* Kafka Connect에 대한 내용은 제외하고 파이프라인의 구축 시 고려해야할 사항에 대해서만 정리하고자 한다~

<hr/>
#### 데이터 파이프라인 구축 시 고려할 점

##### Timeline 적시 

- 시스템에서 어떨 때는 하루에 한번 벌크로 데이터가 올 때도 있고, 수 millisecond에 데이터가 올 수도 있음
- 좋은 data integration 시스템은 다양한 timeliness 요구사항을 지원해야 함
- kafka는 근접 실시간 부터 hourly batch까지 다 지원가능함
- producer는 빈번하게 혹은 빈번하지 않게 kafka에게 write 가능함
- consumer는 마지막 event가 도착하자마자 읽을수도 있고 batch로 읽을 수도 있음. 매 시간 읽거나 이전 시간의 event를 누적할 수도 있음
- back-pressure 적용이 매우 하찮은 일이 됨. consumer에서 batch로 읽으면 됨.

##### Reliability (신뢰)
- 단일고장점 ( Single Point Failure ) 피하고 빠른 자동 복구 원함.
- 데이터 손실 없어야 함
- at-least-once 손실 없지만 중복 있음 (kafka 보장)
- exactly-once 손실없고 중복없음 (외부 데이터 소스가 transactional model 이거나 unique key 있거나)
- 대신 kafka connect API가 제공되어서 좀 쉽게 exactly-once 구현할 수 있다고 함

##### High and Varying Throughput (높고 다양한 처리량)
- 확장 가능해야 하며, 갑자기 처리량 커져도 처리할 수 있어야함
- kafka가 producer와 consumer간의 버퍼 역할을 하므로 producer 처리량에 꼭 consumer 처리량을 결부시켜 생각할 필요 없음
- consumer와 producer를 독립적이며 동적으로 추가하면 되므로 kafka의 scale 능력은 좋음
- kafka는 별로 크지 않은 클러스터에서도 초당 100메가 정도는 처리함
- kafka Connect API에는 병렬처리에 관한 것도 있음(작업 분배, 멀티쓰레드, 가용 CPU 자원 활용 등)

##### Data Format
- XML or db에서 kafka로 보낼때 avro 사용
- ElasticSearch에 write할 때는 json으로 변환
- HDFS로 write할때는 Parquet, S3로 write할 때는 CSV로 변환할 수 있음
- Kafka, kafka Connect API는 데이터 포맷이 무엇인 지 완전 상관없음. 

##### Transformations (변환)

- ETL vs ELT
- ETL 
	- 추출해서 변환하고 적재, 데이터 파이프라인이 변환의 책임이 있음, 단점은 만약에 변환해서 날라간 데이터가 나중에 필요해 지면 데이터 파이프라인 재구축해야됨.

- ELT 
	- 추출해서 적재하고 변환, high-fidelity pipeline (원본이랑 거의 비슷한), data lake architecture
	- 모든 데이터 접근 가능하므로 매우 유연함,  트러블 슈팅 쉬움
	- 단점은 target 시스템에 CPU, 저장 리소스 많이 듦 

##### Failure Handling

- 모든 데이터가 완벽하다고 가정하는 것은 위험하므로 실패 처리 어떻게 할 지 미리 계획하는 것이 매우 중요함
- 파싱이 안되거나, 오타, 실수, 중복 등 복구할 수 있는가?
- kafka는 긴 기간의 모든 event를 저장하고 있으므로 해당 시점으로 돌아갈 수 있고 필요할 때 에러를 복구 할 수 있음

##### Coupling and Agility

- 데이터 소스와 target의 decoupling이 데이터 파이프라인의 매우 중요한 목표
- 실수로 coupling이 발생할 수 있는 몇가지 방법이 있음

1. Ad-hoc pipeline (임시변통의 파이프라인)
	- 새로운 시스템에 추가적인 파이프라인 필요하다면 새로운 기술 적용에 비용이 많이 듦

2. Loss of metadata
	- 만약 데이터 파이프라인이 schema metadata 저장하고 있지 않거나 schema evolution 안될 경우 
	- 데이터 소스와 사용하는 소프트웨어 간 강한 결함 발생을 끊어야 함
	- Oracle -> HDFS에서 Oracle에서 필드 추가하면 HDFS 데이터 활용하는 소프트웨어는 동시에 다 업그레이드 하거나 꺼지거나 할텐데. 이러면 agile 하지 못함

3. Extreme processing
	- 미리 processing 해버리면 이 data를 사용하는 downsteam system의 결과는 모두 같아지므로 최대한 데이터를 보존해야 함

