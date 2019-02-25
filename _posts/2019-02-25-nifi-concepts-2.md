---
layout: post
title:  "[Nifi] Nifi Concept #2 "
date: 2019-02-25 20:15:12
categories: Nifi 
author : Jaesang Lim
tag: Nifi
cover: "/assets/instacode.png"
---

### NIFI 
- 분산 환경에서 대량의 데이터를 수집하고 처리하기 위해 만들어졌음
- 실시간 처리에 강점을 보임 
 > - 디렉토리에 파일이 생성되면, 바로 DB, HDFS, Hbase, Elasticsearch, Kafka 등에 전달 완료  
- 클러스터 환경에서는 장애가 나도 복귀 될때까지, 데이터 처리는 못하지만, 잃어버리지 않음 
- Zero Master 클러스터 환경 제공 
 > -  0.x 버전에는 Master-slave구조 였으나, SPOF의 문제를 해결하기 위해 1.x버전부터 Zookeeper의 Auto-Election 방법을 사용 
- 웹 인터페이스가 있어서 쉽게 사용하기 쉬움
 > - Flexible하고 쉽게 Data Pipeline를 구성할 수 있음 ( Drag & Drop ) 
- 데이터의 이동 경로를 추적할 수 있음
- 배치 작업은.. 잘 못함. 
 > - HDFS에게 대량의 파일을 빠르게 할 때는 DistCp 효율

### 구성요소
![nifi](https://user-images.githubusercontent.com/12586821/53334879-5b5b2900-393d-11e9-8fb3-9bc67f14f4a4.PNG)

#### FlowFile
- Attribute 와 Content로 구성
##### 1. Attribute
  > - Key/value 형태로, 데이터 이동, 변환, 저장에 필요한 정보 
  > - FlowFile에 대한 metadata값  
##### 2. Content
  > - 데이터 값
- Processor와 Processor의 이동간에도 **복사본** 을 만들어 추적가능 
  > - 내용은 복사하지않고,  어디에 있는지 포인터 정보만 복사 = RDD의 lineage 
  

#### Repository
- 위에서 설명한 FlowFile의 복사본은 어디에 저장되며, 추적이 되는건가? 
- 모든게 immutable함
![nifi2](https://user-images.githubusercontent.com/12586821/53334944-86457d00-393d-11e9-8b87-73db3a05ab5a.PNG)

##### 1. Content_repository
> - FlowFile의 Content 저장
 
##### 2. Flowfile_repository
> - FlowFile에 대한 속성값과 내용이 어디에 있는지 저장

##### 3. Provenance_repository
> - Processor가 flowfile을 처리할 때마다 이력( 이벤트)저장 )
> - FlowFile의 History  

- 자세한 내용은 여기서! https://nifi.apache.org/docs/nifi-docs/html/nifi-in-depth.html#intro

#### Processor
- flowFile 처리기라고 생각하자
- 다 적진 못하지만, 지금 생각나는 것들만 해도 이정도 있다.
![nifi3](https://user-images.githubusercontent.com/12586821/53334985-a2e1b500-393d-11e9-9162-765927f46854.PNG)

- Kafka에서 데이터 Consume / Produce 
- Avro Schema 기반 validation 및 데이터 변형
- Attribute 값 변경 및 추가
- Content 내용 변경 및 추가 
- 포맷 변경 (csv to json with Avro 등등)
- elasticsearch에 저장
- ExecuteScript로 Groovy, Python으로 flowfile 처리 
- Attribute 기반 라우팅
- Attribute 값에서 Expression Language 및 RecordPath, JsonPath를 이용하여 데이터 처리 


#### Connection
![nifi4](https://user-images.githubusercontent.com/12586821/53335000-ad9c4a00-393d-11e9-9c54-8267b4d62c7b.PNG)

- FlowFile의 Queue 로써 아래와 같은 기능 제공
1. Prioritization 
  > - FlowFile을 어떤 우선 순위로 처리할 것인가?
2. Expiration 
  > - Queue에 얼마동안 보관할 것 인가?
3. Back Pressure 
  > - Queue 얼만큼 차면 FlowFile의 생성을 조절할 것인가?

