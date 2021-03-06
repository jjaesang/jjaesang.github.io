---
layout: post
title:  "[HP-Spark] 내가 생각하는 스파크"
date: 2019-03-16 18:15:12
categories: High-Performance-Spark 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## 스파크 란?
- 공식문서 
> - Apache Spark is a fast and general-purpose cluster computing system. 
- 스파크 완벽 가이드 (책)
> - 빅데이터를 위한 통합 컴퓨팅 엔진과 라이브러리의 집합
- 하이 퍼포먼스 스파크 (책)
> - 범용 목적의 고성능 분산처리 시스템

---

##  '내가 생각하는' 스파크 란? 
#### 범용 목적의(=통합) 컴퓨팅 엔진을 가진 분산 처리시스템

- 범용 목적 (= 통합 ) ?
> - 데이터 분석의 배치, 스트리밍 및 머신러닝 등 수행할 수 있음 
  > - 머신러닝만, 스트리밍만, 배치만 하는 프레임워크는 아니다. (sklearn / storm / MR 등이 될 수 있을 듯)
> - 위의 다양한 분석 작업을 같은 연산 엔진 Spark Core로 수행함 

- 컴퓨팅 엔진?
> - 저장소 역할이 아닌 데이터 연산만 수행 
> - 하둡과 같이, hdfs와 mr간의 종속성이 없이, 특정 저장소에 종속되지 않고, 저장소 역할이 아닌, 데이터 연산만 수행한다는 의미

- 여기서 스스로에 던지는 질문
1. 왜 범용이야 ? 범용이 좋아?
> - 같은 작업을 수행하는 함수 자체를 배치작업, 스트리밍 작업, 머신러닝작업( = 여기서는 데이터 전처리에 대해 한정된다고 생각함)에서 동일하게 활용할 수 있음
> - 같은 작업을 수행하는 함수를 배치/스트리밍/머신러닝 마다 새롭게 코드를 개발하고 하면 생산성 측면에서 좋지 않을 것 같음 

2. 왜 컴퓨팅 엔진이라는 것에 국한했는가, 저장소도 같이 하면 좋은거 아니야?
> - 아니야. 일단 컴퓨팅엔진이라고 해서 다양한 소스(=저장소)에 접근하는 것이 어렵다는 것을 의미하지않음 ( = kafka , jdbc , filesystem, distributedFileSystem 등등)
 > - 스파크 완벽 가이드에서도 정의했듯이, APMLab실의 스파크팀에서도 라이브러리 자체가 강력한 부가기능이라는 것을 확인하고, 다양한 라이브러리들 만듬 ( ML,Streaming 등 )
 > - 스파크는 표준 라이브러리와 서드파티 라이브러리로, 다양한 입력 소스에 대한 라이브러리가 있음
> - 저장소가 같이 있으면 Hadoop 처럼 Hdfs와 MR간의 종속성이 있어, 둘 중 하나만 단독적으로 사용하려면 어려움이 따름 
 > - 사실 이 부분, 연산에만 집중하는 프레임워크라는 것이 스파크 개발팀에서 하둡과 기존 빅데이터 플랫폼과의 차별점이라고함

---

##  '스파크 완벽 가이드' 에서의 스파크 란? 
#### 빅데이터를 위한 통합 컴퓨팅 엔진과 라이브러리의 집합 

- 통합 ?
  - 스파크는 '빅데이터 애플리케이션 개발에 필요한 통합 플랫폼을 제공하자'는 핵심 목표를 가짐 
  - 간단한 데이터 읽기~ SQL, ML, 스트림 처리까지 다양한 데이터 분석 작업을 같은 연산 엔진과 일관성있는 API로 수행할 수 있게 설계
  - 현실 세계의 데이터 분석 작업이 다양한 처리 유형과 라이브러리를 결합해 수행된다는 통찰에서 비롯 

- 컴퓨팅 엔진?
  - 저장소 시스템의 데이터를 연산하는 역할만 수행할 뿐, 영구 저장소 역할은 수행하지 않음
  - 대신, Azure, S3 Hadoop, Cassandra, Kafka 등 지원
  - 스파크는 내부에 데이터를 오랜 시간 저장하지 않으며, 특정 저장소 시스템을 선호하지 않음
  - 왜 ? 연구 결과 대부분 데이터는 여러 저장소에 혼재되어 저장되어 있고, 데이터 이동은 높은 비용 유발(네트워크)
  - 그래서 데이터 저장 위치에 상관없이 처리에 집중하도록 만들어지고, API는 서로 다른 저장소 시스템을 매우 유사하게 볼 수 있게 만듬
  - **연산에 초점을 맞추면서 하둡과 기존 빅데이터 플랫폼과 차별화**
  - 하둡은 범용 서버 클러스터 환경에서 저비용 저장 장치를 사용하도록 HDFS와 컴퓨팅 시스템(MR)을 가지고 있고 서로 매우 연관성이 있음
  - 서로의 의존성이 있어, 둘 중 하나의 시스템만 단독으로 사용하기 어려움
 
- 라이브러리
  - 궁극의 스파크 컴포넌트는 데이터 분석 작업에 필요한 통합 API를 제공하는 통합 엔진 기반의 자체 라이브러리다.  
  - 엔진에서 제공하는 표준 라이브러리와 서드파티 패키지 형태의 외부 라이브러리 지원 
  - 사실상, 표준 라이브러드는 여러 오픈소스의 집합체
  > - SQL, ML, Streaming, Structued Streaming, GraphX
  - 서브파티 패키지
  > - 다양한 외부 저장소를 위한 connector부터 ML 라이브러리
  > - https://spark-packages.org/
 
 ---
 
## '하이 퍼포먼스 스파크' 에서의 스파크 란?
#### 범용 목적의 고성능 분산처리 시스템
 
 - 스파크는 하나의 서버에서 처리할 수 있는 수준 이상의 대용량 데이터처리가 가능 + 쓰기 편한 고수준 API 제공
 - 사용자들이 직접 데이터 변형 로직과 머신러닝 알고리즘의 병렬 실행을 작성할 수 있도록 지원하면서, 시스템 종류에 구해받지 않음
 - 그 덕으로 서로 다른 규모와 종류의 시스템이 섞여 있는 분산 저장 시스템에서도 빠르게 연산할 수 있는 코드를 작성하는 것이 가능
 - 간단하게 구현하더라고, 최적화하지 않으면 매우 느리고 불안정할 가능성이 크다
 - **매우 큰 용량의 데이터와 연계되는 경우가 많음 -> 성능 최적화에 의해 얻는 이득 (시간, 리소스)가 어마어마**
 - 성능은? 빨리 실행이 끝나는 것 + 대용량 데이터를 다루어야 하는 상황에서의 실행 가능 여부 자체를 의미할 수 있음
 - 즉, 수 GB 단위를 처리하는데 실패한 쿼리를 데이터 구조의 철저한 파악 및 요구사항 분석을 통해 최적화하고, 동일 시스템에서 수 TB도 문제없이 돌아가게 만드는 것이 가능하다는 말
 - **사용하는 데이터의 구조와 형태 ( 데이터 크기, 키 분포)에 대해 잘 파악하는 것이 많은 성능**


