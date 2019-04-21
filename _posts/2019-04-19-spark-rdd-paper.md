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


### 무엇을 다루고자 하는가
- MapReduce와 Dryad가 대용량의 데이터를 처리하는 분산 프로그래밍 모델로 널리 사용중이나, 약간 부족한 문제가 있음
- 이 문제를 해결하고자 RDD라는 Distributed memory abstraction를 소개하고 함
- RDD의 정의, 목적, RDD를 활용하여 위에서 언급한 문제를 어떻게 해결했는지에 대한 설명이 있음

---

### RDD, 넌 왜 이 세상에서 나타난거야?
- 이미 널리 사용중인 Mapreduce와 Dryad 같은 분산 프로그래밍 모델이 있었음
- 위의 두 프로그래밍 모델은 이미 분산 환경에서의 다음과 같은 문제를 해결하고 구현해놓았음! 최고!
1. locality-aware scheduling
2. fault-tolerance
3. load balancing
4. commodity clusters에서 대용량 데이터 분석 능력
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
바로 그것이 RDD(Resilient Distributed Datasets)!

### 자기소개, RDD
- acyclic data model이 가진 매력적인 특성은 이미 가지고 있음
> 1. automatic fault-tolerance
> 2. locality-aware scheduling
> 3. scalablity
- 추가로, RDD는 사용자가 명시적으로 cache를 통해 데이터셋을 디스크가 아닌, 메모리상에 올려둘 수 있는 기능이 생겼음

- 기존 Distributed Shared Memory와는 다르게 제한 사항이 있는데, 바로 RDD는 read-write가 아닌, read-only, 즉 immutable이라는 특징을 가지고 있음
- 이 read-only, immutable한 특징으로 low-overhead fault-tolerance를 가능하게 했음
- 기존 Distributed Shared Memory는 fault-tolerance하기 위해, disk에 checkpoint를 하는 비싼 작업을 진행했지만,
- RDD는 lineage을 통해 작업에 실패한 partition이나 rdd를 low-overhead로 복원할 수 있음
- lineage에는 이 RDD가 어떤 다른 RDD에서 왔으며, 어떤 Iterator를 통해 만들어져있는지에 대한 다양한 정보가 있음

- 그래서 RDD가 두 문제를 해결하고자 하는 최초의 시도야? iterative / interactive을 해결하기 위해? 
- 아니야, Google Pregel(specialized programming model for iterative graph algorithms) / Twister,HaLoop(iterative MapReduce model)가 있었음
- 하지만 위에 두 모델은 특정 애플리케이션에 제한되는 사항이 있어, 그래서 우리는 RDD를 좀 더 범용적으로 사용할 수 있게 Spark라는 프로그래밍 모델에 구현해 놓았음

### RDD의 fault-tolerance

- 위에서 설명했듯이 acyclie data model의 매력적인 특징 3가지를 다 가지고 있음
- 하지만 그 중, 'fault-tolerance'의 특징을 효율적으로 지원하는 것이 가장 어려웠음..
- 왜? 일반적으로 distribute dataset의 fault-tolerance를 보장하기 위해서는 두가지 방법이 있었음
> 1. disk에 checkpoint하기
> 2. 변경된 데이터의 상태값을 logging하는 법

- 앞 서 설명했듯이, 대용량의 데이터를 분석하고자하는 목표가 있는 상황에서는 checkpoint하는 것은 매우 비싼 작업임
> 1. 디스크 용량이 필요함
> 2. 여러 서버에 데이터를 복제(replicate)하면, 복제해야하는 데이터의 사이즈 보다 작은 network-bandwidth로 복제해야함


- 그래서 두번 째 방법, log update으로 fault-tolerance 문제를 해결하였음
> - 물론, logging할게 많아지면 이것 또한 비쌈..
> - 하지만, RDD의 transformation은 coarse-grained transformation만을 지원하기 때문에 fine-grained만큼 로깅할 양이 적음
> - lineage에는 RDD가 어떻게 만들어졌는지애 대한 transformation을 기록해두고, parition의 문제가 발생했을시 lineage 정보를 통해 복구가 가능하게 하였음


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


