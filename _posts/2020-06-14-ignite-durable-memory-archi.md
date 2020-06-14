---
layout: post
title:  "[Ignite] Durable Memory Architecture"
date: 2020-06-14 21:05:12
categories: Ignite
author : Jaesang Lim
tag: Ignite
cover: "/assets/instacode.png"
---

# Ignite Durable Memory Architecture

Ignite의 새로운 메모리 아키텍처 'Durable memory architecture'

- Durable memory architecture와 native persistence는 2.0버전 이후 베포 
- 메모리와 디스크의 데이터는 동일한 Binary representation을 가져 메모리에서 디스크로 이동하는 동안 데이터를 추가로 변환 할 필요가 없음.

Ignite의 새로운 메모리 아키텍처는 
 1. Page Format으로 Off-heap을 제공 
 2. 고정 된 크기의 Page로 분할되는 page-based memory 아키텍처라고도 함
 
- page는 RAM의 관리되는 off-heap에 할당되며 특정 계층 구조로 구성
 
해당 아키텍쳐는 **Region <- Segment <- Page**로 추상화되어있음

 
---
 
## 1. Page 

- 실제 data 또는 metadata를 포함한 기본이되는 저장단위.
- 각 page에는 fixed length와 unique identifier인 FullPageId를 가짐 
- page는 heap 밖에 저장되며, Page Memory Abstraction를 사용하여 메모리와 상호 작용함

- 메모리에서 디스크로 푸시되는 것도 page단위로 발생하기때문에, page 크기는 성능에 중요함 
> -  그렇지 않으면 스와핑 효율성이 심각하게 저하

- 페이지 크기가 작으면 단일 페이지에 맞지 않는 대용량 레코드를 저장하는데 문제가 있음
> - 대용량 레코드를 읽기위해 Ignite는 10-15 개의 레코드가 있는 작은 페이지를 얻기 위해 운영 체제를 많이 호출해야함

레코드가 단일 페이지에 맞지 않으면 여러 페이지에 분산되고 각 페이지에는 일부 레코드 조각만 저장함
> - 이렇게되면 Ignite가 전체 레코드를 얻기 위해 여러 페이지를 조회해야하고, 이를 위해 메모리 페이지 크기를 설정할 수 있음 

- 저장 장치 (SSD, Flash 등)와 운영 체제의 캐시 페이지 크기와 같거나 그보다 작은 페이지 크기를 사용하는 것이 좋음 
> - 운영 체제의 캐시 페이지 크기를 파악하기 어려운 경우 4KB를 페이지 크기로 사용할 것


Page는 2개의 section으로 나눠짐
1. Header
2. Page Data(Memory Page)
 > 1. Data Page
 > 2. Index Pages

Ignite memory page structure
![image](https://user-images.githubusercontent.com/12586821/84591420-2009a400-ae79-11ea-89a9-4167d70c3b24.png)

### 1-1. Header

1. Type
 > - size 2 bytes, defines the class of the page implementation (ex. DataPageIO, BplusIO).
2. Version
 > - size 2 bytes, defines the version of the page.
3. CRC
 > - size 4 bytes, defines the checksum.
4. PageId
 > - unique page identifier.
5. Reserved
 > - size 3*8 bytes.

### 1-2. Memory Page

Memory Page는 2가지로 나뉨
1. Data pages
2. Index Pages

![image](https://user-images.githubusercontent.com/12586821/84591452-77a80f80-ae79-11ea-9398-b37a1eb4dedc.png)


#### 1-2-1. Data pages
- Ignite cache에 저장한 데이터를 저장 
- single record가 하나의 data page에 할당되지않으면, 여러 page에 나눠서 저장 

- 하나의 Data page에는 메모리 defragmentation를 피하기 위해 가능한 효율적으로 메모리를 활용하기 위해 여러 키-값 entry이 있음
> - 새로운 키-값 entry가 캐시에 추가 될 때 전체 키-값 쌍에 맞는 최적의 Data page를 찾음

- entry update중, 사이즈가 data page에서 사용가능한 여유공간을 초과하면, 새로운 data page를 찾고 값을 이동 

Data page도 내부적으로 두가지로 나눰
1. Header
    1. Free space, refers to the max row size, which is guaranteed to fit into this data page.
    2. Direct count.
    3. Indirect count.
  
2. Page Data


#### 1-2-2. Index Pages and B+trees
- Index Page는 B+trees에 저장되며 각 페이지는 여러 페이지에 분산 될 수 있습니다.
> - 모든 SQL 및 캐시 index는 B+trees에 저장 

- SQL 스키마에 선언된 모든 인덱스에 대해 Ignite는 전용 B+trees 인스턴스를 초기화 및 관리  
- Data Pages와 달리 Index Pages는 항상 메모리에 저장되어 빠르게 액세스할수있음 


---


## 2. Segments

- 연속 된 물리적 메모리 블록으로, 할당 된 메모리의 원자 단위
> - 할당 된 메모리가 부족하면 OS에 추가로 세그먼트가 요청
-  세그먼트는 고정 된 크기의 페이지로 나뉘고, 모든 페이지 유형에는 Data Page와 Index Page로 Segment에 상주
![image](https://user-images.githubusercontent.com/12586821/84591528-16347080-ae7a-11ea-890b-7a864ab03090.png)


- 현재 버전에서 세그먼트 크기가 최소 256MB 인 하나의 전용 메모리 영역에 최대 16 개의 메모리 세그먼트를 할당 할 수 있음
- 메모리 세그먼트에서 현재 사용 가능한 페이지 및 LoadedPagesTable이라는 영역 주소에 대한 페이지 ID 매핑에 대한 정보를 관리하기 위해 특정 구성 요소를 사용
> - LoadedPagesTable 또는 PageIdTable은 페이지 ID에서 상대 메모리 세그먼트 청크로의 매핑을 관리
> - LoadedPagesTable은 Ignite 버전 2.5부터 FullPageId의 HashMap을 유지하기 위해 Robin Hood Hashing 알고리즘을 사용

---

## 3. Region

The top level of the Ignite durable memory storage architecture is the data Region, a logical expandable area.
![image](https://user-images.githubusercontent.com/12586821/84591710-87c0ee80-ae7b-11ea-9419-ec64c29c9a9d.png)


- Region에는 몇 개의 메모리 Segment가 있을 수 있으며 설정, 제약 조건 등 단일 스토리지 영역을 공유하는 세그먼트를 그룹화 할 수 있음
- 크기, 제거 정책이 다양하고 디스크에 유지 될 수있는 여러 데이터 region을 구성 될 수 있음



## 4. Data Eviction (제거)

Eviction과 Expiration은 사용되지 않는 캐시의 entry를 정리하여 OOM으로부터 메모리를 보호하는 수단

1. Page Based Eviction 
 - off-heap메모리에서 memory page를 제거

2. Entry based Eviction
 - Java Heap 또는 On-heap heap메모리에서 Entry를 제거 

### page based eviction


RAM이 page로 가득차있고, 새로운 page을 할당해야할 경우 메모리에서 Eviction해야함
- 클러스터 전체에서 아닌 노드별로 발생함 
각 노드에서 eviction모드를 추적하고, 메모리에 상주한 data page 사이즈를 분석하여 어느 페이지에 제거가 필요한지를 결정함

- Ignite Native Persistence가 활성화 된 경우, Page eviction은 자동으로 메모리공간이 충분하지않을때, 자동으로 오래된 page을 eviction하도록 동작하고, region에서 page based eviction 정책은 무시댐 
- index page와 metapage는 eviction되지않음

### entry based eviction

heap에 저장할수있는 entry 개수를 제어함 
nearCache나, setOnheapCacheEnable된 캐시는 on-heap 최대 size가 도달할 경우 evication 함

- persistence할 경우, 디스크에 저장되지만, on-heap에서 제거된 entry Ignite SQL 쿼리를 통해 반환되지 않음(?)

---

## 5. Data Expiration (만료)

캐시 Eviction와 Expiration의 기본 차이점
 - Expiration 정책은 항목이 Expiration 된 것으로 간주되기 전에 통과해야하는 '시간'을 지정한다는 것

캐시 Entry가 Expiration되면 캐시에서 제거 될 수 있음 (TTL)

Apache Ignite Expiration 정책은 기본 노드 당이 아닌 '클러스터 전체'의 작업입니다. 

![image](https://user-images.githubusercontent.com/12586821/84591779-1897ca00-ae7c-11ea-9675-137f4537e335.png)


EgarTTL가 활성화된 캐시가 하나라도 있으면, 백그라운드로 expired된 항목을 정리하기위해 노드당 단일 쓰레드 생성하여 작업 진행
 - PARTITIONED 및 REPLICATED 캐시에 따라 다름
