---
layout: post
title:  "[Spark-Paper] RDD : A Fault-Tolerant Abstraction for In-Memory Cluster Computing"
date: 2019-04-19 17:05:12
categories: Spark-Paper 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## Resilient Distributed Datasets : A Fault-Tolerant Abstraction for In-Memory Cluster Computing


### 1. 무엇을 다루고자 하는가
- MapReduce와 Dryad가 대용량의 데이터를 처리하는 분산 프로그래밍 모델로 널리 사용중이였음
- 하지만 기존 프로그래밍모델에서 사용되는 acyclic data flow model이 적합하지 않은 두가지 application이 있음
- ml, graph application에 사용되는 iterative algorithm과 interactive data mining tool은 중간결과를 디스크에 저장하면서 
- 데이터를 iteractive, interactive하게 활용시, overhead와 latency가 크면서, 데이터를 재사용(reuse)할 수 있는 Distributed memory abstraction을 소개함
- 재사용 뿐 아니라, 기존 acyclic data flow model이 가지는 중요 특징을 가진 Distributed memory abstraction, RDD를 소개하고 함

---
좀 자세히 알아보자면 ..

### 2. RDD, 넌 왜 이 세상에서 나타난거야?
- 이미 널리 사용중인 Mapreduce와 Dryad 같은 분산 프로그래밍 모델이 있었음
- 위의 두 프로그래밍 모델은 이미 분산 환경에서의 다음과 같은 문제를 해결하고 구현해놓았음! 최고!
1. locality-aware scheduling
2. fault-tolerance
3. load balancing
4. commodity clusters에서의 대용량 데이터 분석 능력
- 일반적으로 cluster computing system은 acyclic data flow model를 기반으로 함
> - acyclic data flow model를 간단히 이해하자만, 특정 분산 스토리지에서 데이터를 읽어와, deterministic operator로 이루어진 DAG를 구축해, 결과를 다시 persist한 storage에 저장해놓지.
- 하지만 이 acyclic data flow model이 효율적으로 작동하지 못하는 application 타입이 있음
- 바로 !! parallel operation에서의 중간 또는 결과 데이터를 **재사용(reuse)** 하는 경우
- 예를 들자면 
> - 1. iterative algorithm
  > > - machine learning, graph application
> - 2. interactive data mining tool

- 왜냐고? 기존 분산 프로그래밍 모델들은 결과를 disk에 저장하고, 다시 읽는 즉, reuse하는 경우에는 다시 disk에서 읽어와 latency와 overhead가 생기기 때문
- 좋아, 이런 acyclic data flow model로 해결하기 어려운 데이터를 reuse하는 경우에는 효율적인 Distribute memory abstraction가 필요하겠어
- 바로 그것이 RDD(Resilient Distributed Datasets)!

---

### 3. RDD의 정의 및 특징
- RDD는 read-only, partitioned collections of records

- acyclic data model이 가진 매력적인 특성은 이미 가지고 있음
> 1. automatic fault-tolerance
> 2. locality-aware scheduling
> 3. scalablity
- 추가로, RDD는 사용자가 명시적으로 cache를 통해 데이터셋을 디스크가 아닌, 메모리상에 올려둘 수 있는 기능이 생겼음
> - reuse 목적으로 추후에 같은 데이터셋에 접근시 low-latency와 low-overhead 보장

---

### 4. RDD의 fault-tolerance
- 위에서 설명했듯이 acyclie data model의 매력적인 특징 3가지를 다 가지고 있음
- 하지만 그 중, 'fault-tolerance'의 특징을 효율적으로 지원하는 것이 가장 어려웠음..
- 왜? 일반적으로 distribute dataset의 fault-tolerance를 보장하기 위해서는 두가지 방법이 있었음
> 1. disk에 checkpointing
> 2. 변경된 데이터의 상태값을 logging하는 법

- 앞 서 설명했듯이, 대용량의 데이터를 분석하고자하는 목표가 있는 상황에서는 checkpointing하는 것은 매우 비싼 작업임
> 1. 디스크 용량이 필요함
> 2. 여러 서버에 데이터를 복제(replicate)하면, 복제해야하는 데이터의 사이즈 보다 작은 network-bandwidth로 복제해야함

- 그래서 두번째 방법, log update으로 fault-tolerance 문제를 해결하였음
> - 물론, logging할게 많아지면 이것 또한 비쌈..
> - 하지만, RDD의 transformation은 coarse-grained transformation만을 지원하기 때문에 fine-grained만큼 로깅할 양이 적음
> - lineage에는 RDD가 어떻게 만들어졌는지애 대한 transformation을 기록해두고, parition의 문제가 발생했을시 lineage 정보를 통해 복구가 가능하게 하였음
> - 그 결과, RDD는 lineage을 통해 작업에 실패한 partition이나 rdd를 low-overhead로 복원할 수 있음

---

### RDD VS DSM(Distributed Shared Memory)

- DSM을 활용한 응용 프로그램은 global address의 임의의 위치를 읽고 쓰는 과정
- 하지만 클러스터 환경에서의 DSM는 효율적이고 fault-tolerance하게 구현하는 것이 어려움

1. DSM은 어떤 공유되는 메모리공간에 값을 read/write할 수 있지만, RDD는 coarse-grained transformations 방법으로만 write할 수 있음
> - Write 측면에서는 DSM은 'fine-grained', RDD는 'coarse-grained'
> - Read 측면에서는 DSM은 'fine-grained', RDD는 'fine or coarse-grained'
> - RDD를 하나의 lookup table로 사용할 수 있기 때문

2. Stragglers(다른 Task에 비해 뒤쳐진, 느린 Task)에 대한 작업이 가능, 즉 투기적 실행이 가능
> - DSM은 두개의 작업 복사본이 동이한 메모리 위치를 읽으면서, 서로 update를 하는 등의 문제가 있어 구현하기 어려움
> - RDD가 **Immutable**한 장점 중 하나라고 할 수 있음

3. Data locality 기반으로 작업을 자동 할당
4. 메모리가 부족할 경우
> - RDD는 Disk에 저장해 MR과 같은 성능을 낼 수 있음
> - DSM는 swap과 같은 작업은 performance가 저하될 수 있음

5. 장애복구
> - RDD는 lineage를 통해 low-overhead로 장애 복구할 수 있음
> - DSM은 checkpointing을 하거나 애플케이션 자체를 rollback해야함


### RDD

- RDD는 read-only, partitioned collections of records
1. HDFS같은 Storage에서 읽어와 만들 수 있음
2. 이미 존재하는 RDD를 이용하여 만들 수 있음
- RDD를 만드는 과정을 Transformation이라 하며, Lazy-Evaluation를 통해 action이 발생하기 전까지는 계속 Lineage에 Transformation을 기록만 해둠

Spark에서의 RDD의 구현법
- 우리는 2가지 측면으로 RDD를 controll할 수 있음
1. cache
> - cache하면 runtime동안 해당 서버 메모리에 저장되고, cache사이즈가 되면 disk로 spill함

2. partitioning
> - RDD의 각 record의 key기반으로 partitioning을 할 수 있음 ( hash or range )


### RDD가 적합하지 않은 애플리케이션

- fine-grained 형태로 공유하는 상태값을 비동기적으로 업데이트, 처리하는 웹 애플리케이션 등에는 적합하지 않음
- 이런 애플리케이션에는 transaction를 보장하며, checkpoint하는 DB가 더 효율적

- RDD는 모든 record에 동일한 operation을 수행하는 배치 애플리케이션에 적합함 ( coarse-grained transformation을 제공하니깐 )
- 각 Transformation lineage를 통해 효율적으로 저장해, 많은 양의 로그를 기록할 필요없이 fault-tolerance할 수 있음


### RDD가 Checkpointing도 제공함!

- lineage를 통해 RDD의 장애를 복구할 수 있으나, 매우 긴~ Lineage일 경우, wide transformation인 경우 모든 partition을 다시 다 재계산해야하므로,
- 복구하는 시간이 lineage를 통해 재계산하는 것보다 , checkpointing하는 것이 빠를 수 도 있음

- RDD의 immutable 특성은  일반적인 shared memory보다 checkpointing을 더 간단하게 만듭니다.
프로그램 일시 중지 또는 분산 스냅 샷 구성없이 RDD를 백그라운드에서 작성할 수 있습니다.


