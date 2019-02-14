---
layout: post
title:  "[Streaming] Spark structured Streaming : 기본편"
date: 2019-02-14 15:15:12
categories: Spark-Structured-Streaming 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## 스트리밍 프로세싱에서 필요하다고 생각하는 것들은 .. 
---

- 스트리밍 처리에 있어서 필요한 기능들이 무엇이 있을지에 대해 고민을 먼저 해보자.
- **1) 데이터를 받아야하는 Source** 가 필요할 것이고,  **2) 데이터를 처리하는 처리 엔진**, **3) 처리된 결과를 저장하는 Sink**가 필요할 것이다.
- 그렇다면, **4) Source에서 데이터는 언제 받아와야할까?** 시간마다? 받아온 데이터를 다 처리 직후 ? 이런 궁금증 생길 것이다
- 또 **5) 처리된 데이터를 Sink에 어떻게 저장, 또는 처리를 할 것인가?** 증분할 것인가? 업데이트인가? 아니면 매번 전체 결과를 덮어 쓸 것인가?

- 위와 같이 생각만 해도 필요한 5가지 기능은 모두 Spark가 우리에게 쉽게 개발할 수 있게 해준다 ( 최고! ) 

- 위에서 고민한 내용들을 이제 스파크 구조적 스트리밍에 하나씩 대입해보고자 한다.

--- 

## 구조적 스트리밍의 핵심 개념 및 요소들
 - **스트림 데이터를 데이터가 계쏙해서 추가되는 테이블 처럼 다루는 것** 이 것이 핵심이다. 끝 
 > 근데 여기서, 계속해서 추가되면.. 시간이 지나면 지날 수 록 데이터의 사이즈가 커지고 그러면 터지지 않을까?... 이 것에 대한 해답도 찾아야한다.
 
 
#### 1) Source ( org.apache.spark.sql.streaming.DataStreamReader )
 - 스트리밍 방식으로 데이터를 읽어오는 방법 
    > - Apache Kafka 0.10 버전 
    > - HDFS 같은 분산파일시스템의 파일 ( 디렉토리의 신규 파일을 계속 읽어오는 방법 )
    > > - Parquet, text, json, csv 등등 
 - 모든 파일은 스트리밍 작업에서 바라보고 있는 디렉토리에 원자적으로 추가 되어야함
 > - 부분적으로 파일이 쓰인다면, 파일을 소스로 할 시 데이터를 부분만 처리하는 문제가 있다
 
 
#### 2) 처리하는 엔진 ( 구조적 스트리밍 엔진 )
 - 구조적 스트리밍은 SQL 엔진 기반의 스트림 처리 프레임워크
 - 구조적 API ( DataFrame/Dataset, SQL)를  사용하며, 배치 연산과 동일하게 표현
 - 코드와 목적지를 설정해주면 구조적 스트리밍 엔진에서 신규 데이터에 대한 증분 및 연속형 쿼리를 실행
 - 코드 생성, 쿼리 최적화 등의 기능을 지원하는 카탈리스트 엔진을 사용하여, 우리가 개발한 로직의 논리적 명령을 물리적 실행으로 변경해준다
 - 체크포인트와 WAL도 지원하여 fault-Tolerance 보장
 
#### 3) Sink ( org.apache.spark.sql.streaming.DataStreamWriter )
 - 스트림 결과를 저장할 목적을 명시
     > - Apache Kafka 0.10 버전 
     > - 거의 모든 파일 포맷
     > - 출력 레코드를 임의 연산을 수행하는 foreach ( 나 같은 경우에는, DB에 upsert하는 경우가 많아 대부분 이 sink를 사용한다)
     > - 테스트용 콘솔 싱크 
     > - 디버깅용 메모리 싱크 
    
#### 4) Trigger 
 - 데이터의 출력 시점을 정의
 - 구조적 스트리밍에서 언제 신규 데이터를 확인하고 결과를 갱신할 지 정의
 - 기본 : 마지막 입력 데이터를 처리한 직후에 신규 입력 데이터를 조회해 최단 시간내 새로운 처리 결과를 만들어냄
 - File Sink 경우, 매우 작은 크기의 파일이 다량으로 생성될 수 있음
 - 그래서 **처리 시간_고정된 주기로만 신규 데이터 탐색** 기반 Trigger 지원 ( 이건 모, DStream Batch_interval과 동일하다)
```scala
  df.writeStream.trigger(ProcessingTime("10 seconds"))
```

#### 5) OutputMode ( 출력모드 )
 - Sink를 정의하기 위해, **데이터를 출력하는 방법**에 대한 정의
 - 예를 들어보자면
     > - 신규 정보만 추가 ( = 증분 )
     > - 바뀐 정보로 기존 로우를 업데이트 (= 업데이트 ) 
     > - 매번 전체 결과를 덮어 쓰는 정유
 - 이런 상황에 대응하기 위해 OutputMode를 설정해야함
 - append : 싱크에 신규 레코드만 추가
 - udpate : 변경 대상 레코드를 자체 갱신
 - complete : 전체 출력 내용 재작성 
 


--- 


### Sink 중 가장 많이 사용할 것 같은 Foreach Sink에 대한 내용

- Dataset API에 있는 foreachPartitions와 비슷
- 각 파티션에서 임의의 연산을 병렬로 수행

#### ForeachWriter 인터페이스 (org.apache.spark.sql.ForeachWriter)
  - foreach 싱크를 사용하려면 ForeachWriter 인터페이스를 구현 해야함 
  - open, process, close 3가지 메소드가 있고, 모두 트리거 후 출력을 생성할 때 마다 호출
  - 반드시 알아야할 사항
    - writer는 UDF나 Dataset 맵 함수처럼 반드시 Seriablizable 인터페이스를 구현해야함
    - open, process,closs 모두 각 익스큐터에서 실행
    - open : 연결을 맺거나 트랙잭션을 시작하는 등 초기화 작업
    > - open 메소드가 아닌 다른 부분에서 초기화할 경우, 드라이버에서 초기화되는 오류가 발생할 수 있음
  - 임의의 코드이기 때문에, 내고장성을 고려하여 사용해야함
  
##### 1) open
 - 처리하려는 로우를 식별하는 고유값 처리하는 2가지 파라미터 있음
 - Trigger 마다 호출
 - Boolean Type 
##### 2) process
 - open에서 true일 때, process 메소드는 데이터의 레커드 마다 호출
 - 데이터를 처리하거나 저장하는 용도
##### 3) close
 - open 메소드가 호출되면, 장애가 발생하지 않는 한 close 메서드도 호출 ( open의 true/false와 상관없음 )
 - 만약 스트림 처리 도중에 오류가 발생하면 close에서 그 오류를 받음
    
```scala
     import org.apache.spark.sql.ForeachWriter
     
     datasetOfString.write.foreach(new ForeachWriter[String] {
       def open(partitionId: Long, version: Long): Boolean = {
         // open a database connection
       }
       def process(record: String) = {
         // write string to connection
       }
       def close(errorOrNull: Throwable): Unit = {
         // close the connection
       }
     })
    
```



