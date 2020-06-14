---
layout: post
title:  "[Ignite] Architecture deep dive Intro"
date: 2020-06-14 17:05:12
categories: Ignite
author : Jaesang Lim
tag: Ignite
cover: "/assets/instacode.png"
---

# Ignite Architecture deep dive 

Apache Ignite는 Open-source memory-centric distributed database, caching, computing platform


## 1. Understanding the cluster topology: Shared-nothing acrhitecture

Grid Computing
 - Cluster Computing과 비슷하지만, Cluster Computing은 homogeneous resources로 구성되어있지만, grids는 heterogeneous로 구성
 - grid에 있는 컴퓨터는 다른 OS, hardware에서 돌수있지만, cluster OS,hardware가 같은 것으로 구성 

Shared-nothing Architecture 

- Apache Ignite는 여러 개의 동일한 노드가 하나의 마스터 또는 코디네이터없이 클러스터를 형성하는 Shraed-nothing Architecture 
- Shraed-nothing Architecture에 모든 노드는 동일한 프로세스를 실행
- 중단없이, 노드를 추가/제거하여 RAM양을 늘릴수있음 
- 노드는 peer-to-peer로 메세지를 전달하여 통신하고, resilient하여 일 노드 또는 여러 노드의 중단없는 자동 감지 및 복구가 가능합니다.

## 2. Client And Server Node 

- Ignite 노드는 JVM에서 동작하는 하나의 프로세스

Server Node 
- data를 store하고, Computing하는 Container역할
- data를 포함하고 caching, computing, streaming에 참여함
- Standalione Java 프로세스로 시작함

Client Node
- cache에 put/get과 같은 operation을 수행하기 위한 entry point
- near cache에 저장된 데이터의 일부분을 저장할수있어, 자주 접근한 데이터에대한 local cache역할도 가능
- 대개 application 내에 embedded로 시작함 


- 모든 노드는 default로 Server노드로 시작하고 client노드는 명시적으로 설정해야함
- Client노드는 thick client( e.g. Oracle OCI8)
- Client노드는 Ignite Grid에 연결되어 grid topology(=각 노도의 data partitiotn)를 알고있어, 데이터를 가진 특정 서버에게 요청을 보낼 수있음

Compute Node
- 또 다른 특별 타입의 local node
- 비지니스 로직을 수행하는데 참여하는 node로 기본적으로 서버노드로 동작 
- 하지만, client노드도 computing에 참여할 수 있음 

Server Node / Data Node
- 데이터를 저장하고 computing task를 수행
- client노드는 server cache를 조작할수있고, local data도 저장할수있고 때로는 computing task를 수행할 수 있음
- 그런데 대게, client노드는 caceh를 get/put하는데만 사용

그렇다면 언제? 왜? client노드에서 computing task를 돌릴까?
- 서버 노드에서 많은 양의 transaction이 있어, 내가 돌리고자하는 작업을 서버노드에서 작업을 안하게 하고싶음

Cluster group이란?
 - 작업을 수행하기 위해 server 또는 client노드가 모여있는 logical unit
 
Ingnite 노드는 배포관점에서 두가지 Major Group이 있음
1. Embedded with application
2. Standalone server node

---

Ignite는 standalone java application 또는 application server같은 java web application에 내부적으로 실행가능

하나의 jVM에서도 여러개의 Node 실행가능 - unit test할때 유

## 3. Real cluster topology

- Client와 Server node가 다른 host에서 실행되는 경우.
- production 환경에서 사용되는 방법으로, 개별 Ignite 서버 노드를 down/restart하더라도 전체 클러스터에 영향을 주지않음

## 4. Data partitioning in Ignite 

- Data Partitioning과 distribution 기술은 대용량의 데이터를 다수의 데이터센터에서 다룰 수 있음 

data distribution Model
1. Sharding
 - horizontal partitioning
 - 각 서버는 data의 subset만 가지고 있음
 
2. Replication
- 복제는 여러 서버에 데이터를 복사하므로 데이터의 각 부분을 여러 위치에서 찾을 수 있습니다.
- 각 파티션을 복제하면 단일 파티션 실패 가능성이 줄어들고 데이터 가용성이 향상됩니다.

## 5. Understanding data distribution: DHT

Ignite Shards는 partition이라고 불리고, partition은 대용량의 dataset을 포함한 memory segment 
- partition은 load를 분산하여 경합을 줄이고 성능향상시킴


### Distributed Hash Table ( DHT )

- 클러스터에서 데이터를 파티셔닝하기 위한 distributed scalable system에 사용되는 알고리즘 
- web caching, p2p system, distributed databse에 사용

- Hash Table
 - key, value, hash function이 필요하고, hash function은 key를 value가 위치할 slot에 매핑시킴
 - 각 element의 unique한 slot을 계산하는 과정을 Hashing이라고 
 

Hash Table에 구현에는 Memory overhead가 있고, 많은 메로리를 사용함

Hash Table은 하나의 서버에서 데이터를 저장하는데 용이함
하지만! 많은 수백만키를 저장할때는 DHT가 나타남 


DHT는 단지 클러스터의 여러 노드에 분산 된 키-값 저장소
키의 subset으로 bucket에 할당하고, 이 bucket은 각각 노드에 상주함 

bucket는 별도의 HashTable로 볼수있음 
DHT에서 해시 함수의 또 다른 주요 목적은 키를 소유 한 노드에 키를 맵핑하여 올바른 노드에 요청을 할 수 있도록하는 것입니다.
따라서 DHT의 클러스터에서 키 값을 조회하기위한 두 가지 해시 함수가 있습니다.

첫 번째 해시 함수는 키에 대한 적절한 버킷 매핑을 검색하고 두 번째 해시 함수는 노드에있는 키 값의 슬롯 번호를 반환합니다.



Bucket, Partition 테이블이라고하면, key가 어떤 서버, bucket에 있는지 알수있
 - 이 테이블에는 partitionID와 ID에 해당되는 node 정보를 가지고 있음 
 
가장 큰 문제는 클러스터의 node의 개수를 고정시키는것이 효율적이라는 것 
 - add/remove할때마다 hashing function이 변경되고, data을 재분배함 
 
 
 그래서 이 문제를 해결하고자 **Consistence Hashing or Rendezvous hashing**을 다룸
 
 Rendezvous hashing은 Highest Random Weight(HRW) Hashing이라고 부름
Apache Ignite는 Rendezvous 해싱을 사용하여 토폴로지가 변경 될 때 최소 크기의 파티션 만 이동하여 클러스터를 확장합니다.

Consistence Hashing은 Cassandra, Hazelcast 등과 같은 다른 분산 시스템에서도 매우 인기가 있습니다. 초기 단계에서 Apache Ignite는 일관된 해싱을 사용하여 다른 노드로 이동하는 파티션 수를 줄였습니다. 그럼에도 불구하고 Apache Ignite 코드베이스에서 Consistent Hashing 구현과 관련하여 Java 클래스 GridConsistentHash를 찾을 수 있습니다.


## Rendezvous hashing
= highest random weight hashing 

- Rendezvous 해싱 ³⁶ (일명 최고 랜덤 웨이트 (HRW) 해싱)은 1996 년 미시간 대학교에서 David Thaler와 Chinya Ravishankar에 의해 소개되었습니다.
  인터넷에서 멀티 캐스트 클라이언트가 분산 방식으로 랑데부 지점을 식별 할 수 있도록 처음 사용되었습니다.
  몇 년 후 분산 캐시 조정 및 라우팅을 위해 Microsoft Corporation에서 사용되었습니다.
  랑데부 해싱은 링 기반의 일관된 해싱의 대안입니다. 이를 통해 클라이언트는 주어진 키를 배치 할 노드에 대한 분산 계약을 달성 할 수 있습니다.
  
- 
알고리즘은 노드가 해시로 숫자로 변환되는 일관된 해싱 ³의 유사한 아이디어를 기반으로합니다.
알고리즘의 기본 개념은 알고리즘이 원에 노드와 복제본을 투영하는 대신 가중치를 사용한다는 것입니다.

노드 (N)와 키 (K)의 각 조합에 대해 표준 해시 함수 해시 (Ni, K)를 사용하여 숫자 값을 생성하여 주어진 키를 저장해야하는 노드를 찾습니다.

  선택된 노드가 가장 높은 노드입니다. 이 알고리즘은 복제 기능이있는 시스템에서 특히 유용합니다
  


Consistent hashing (CH) 보다 장점 
1. 노드가 HRW 해싱을위한 원을 만들기 위해 토큰을 미리 정의 할 필요는 없습니다.

2. HRW 해싱의 가장 큰 장점은 노드를 추가하거나 제거하는 동안에도 클러스터 전체에 키를 균등하게 분배 할 수 있다는 것입니다. CH의 경우 작은 크기의 클러스터에 키를 균등하게 분배하려면 각 노드에 많은 가상 노드 (Vnode)를 작성해야합니다.

3. HRW 해싱은 데이터 배포를위한 추가 정보를 저장하지 않습니다.

4. 일반적으로 HRW 해싱은 주어진 키 K에 대해 서로 다른 N 서버를 제공 할 수 있습니다. 이는 중복 데이터 저장을 지원하는 데 매우 유용합니다.

5. 마지막으로 HRW 해싱은 이해하고 코딩하기가 쉽습니다.

Consistent hashing (CH) 보다 단점

1. HRW 해싱에는 키를 노드에 매핑하기 위해 키당 둘 이상의 해시 계산이 필요합니다. 느린 해싱 기능을 사용하는 경우 큰 차이를 만들 수 있습니다.

2. CH 알고리즘으로 단 한 번이 아니라 각 키 노드 조합에 대해 해시 함수를 실행하는 데 HRW 해싱이 느려질 수 있습니다.


RendezvousAffinityFunction 장점 기본 Rendezvous

1. 전체 클러스터 재시작시에도 노드에 대한 캐시 키 선호도가 유지됩니다. 즉, 전체 클러스터를 다시 시작하고 디스크에서 데이터를 다시로드 할 경우 모든 키가 여전히 동일한 노드에 매핑되어야합니다.

2. Ignite RendezvousAffinityFunction은 파티션 대 노드 맵핑 단계가 거의 일정하고 클러스터 토폴로지가 변경 될 때마다 계산합니다 (클러스터에서 노드 추가 또는 제거).

RendezvousAffinityFunction 단점
 - 골고루 분산되지않음 ( 5-10%)
 
 
option
1. 파티션-그리드에 분산 될 여러 파티션으로, 기본값은 1024입니다.

2. excludeNeighbors – 기본 및 백업 파티션이 동일한 노드에 있지 않도록합니다. true로 설정되면 기본 및 백업 사본을 저장하기 위해 이웃과 동일한 호스트를 제외합니다.

3. backupFilter – 백업 노드에 대한 선택적 필터. 이 필터 만 통과 한 노드는 제공된 경우 백업 사본으로 선택됩니다. 파티션의 기본 및 백업 복사본을 다른 랙에 저장하려는 경우에 편리합니다.



Peer-to-peer replication


장점 :

1. 단일 실패 지점이 없습니다.

2. 클러스터에서 노드를 빠르게 추가하거나 제거 할 수 있습니다.

단점 :

1. 다른 노드의 사본에 대한 변경 사항의 느린 전파로 인해 데이터가 일치하지 않을 수 있습니다.

2. 두 명의 사용자가 동시에 다른 노드에 저장된 동일한 레코드의 다른 사본을 업데이트하려고 할 때 쓰기 / 쓰기 충돌의 가능성이 있습니다.

Apache Ignite uses the combination of peer-to-peer replication and sharding strategies,

In PARTITION mode, nodes to which the keys are assigned to are called primary node. Optionally, you can also configure any number of backup copies for cached data. Ignite will automatically assign backup nodes for each key if the number of backup copies is higher than 0.



1. Partitioned mode
The goal of this cache mode is to get extreme scalability.
극단적 인 확장 성을 확보하십시오. 이 모드에서 Ignite는 캐시 된 데이터를 투명하게 분할하여 전체 클러스터에로드를 분산시킵니다. 캐시 크기와 처리 능력은 데이터를 균등하게 분할하여 클러스터 크기와 선형으로 증가합니다. 데이터 관리 책임은 클러스터 전체에서 자동으로 공유됩니다. 클러스터의 모든 노드 또는 서버에는 정의 된 경우 백업 사본과 함께 기본 데이터가 포함됩니다.

A partitioned mode is ideal when working with large datasets and updates are very frequent. To get better query performance, i


2. Replicated mode
The goal of this approach is to get extreme read performance.

3. local mode

This is a very primitive version of cache mode located in the Ignite server node. No data is distributed to other nodes in the cluster and does not have any replica or backup copies of the cache with this approach. As far as the cache with the Local mode does not have any replication or partitioning process, data fetching is very inexpensive and fast. It provides zero latency access to recently and frequently used data. 
The local cache mode is mostly used in read-only operations. It also works very well for read-through behavior, where data is loaded from the data sources on cache misses.

4. near cache 
Near cache can be located on client nodes. Near cache stores the most recently or frequently updated partitioned data and usually sited in front of the partitioned cache. It can improve query performance by caching frequently used data entries on the client node locally. N

, if your remote server nodes located on the separate machine and you want to get a performance boost, you should configure your client node to use near cache.

Near cache can only be used with PARTITIONED cache

Near cache are fully transactional and get updated or invalidated automatically whenever the cache changes on the server node.



Ignite Visor CLI supports two more new commands from the Apache Ignite 2.5 version: cache -slp and cache – rlp for displaying and reset lost partition for a cache. Apache Ignite Web console also delivers these functionalities through monitoring that shows the partition loss metrics.
