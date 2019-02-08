---
layout: post
title:  "스파크 병렬 연산 모델 : RDD "
date: 2019-02-08 18:05:12
categories: Spark
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## 스파크의 병렬 연산 모델: RDD
---

### 스파크의 구성 요소

1. Driver (= Master Node)
    -  병렬 데이터 처리를 수행할 수 있는 클러스터 시스템 (HDFS / YARN )위에서 구동되는 Driver를 위한 프로그램을 개발해야함
  
2. Executor (= Slave Node)
    - RDD는 Executor (= Slave Node)에 저장된다.
    - RDD 구성하는 객체를 ' 파티션 ( Partition ) '
    - 파티션은 경우에 따라 저장된 노드가 아닌 다른 노드에서 계산될 수 도 있음
    - 스파크의 대용량 데이터서트를 표현하는 것 : RDD ( Resilient Distributed Dataset)
    - 변경할 수 없는 형태의 분산 객체 모임
    - RDD = fault-tolerant collection of elements that can be operated on in parallel. 

3. 스파크 클러스터 매니저
    - 스파크 어플리케이션에서 설정한 파라미터에 따라 분산 시스템에 Executor를 실행하고, 분산시키는 역할
    
4. 스파크 실행 엔진
    - 연산을 위해 Executor에게 데이터를 분산해주고 실행을 요청 


--- 

### 스파크 어플리케이션의 간단한 흐름

1. Driver 프로그램이 RDD의 각 데이터의 처리 단계를 결정 ( Tranformation )
2. 바로 실행되는 것이 아니라 연산 시점을 뒤로 미루어, 실제로 최종 RDD를 계산해야하는 시점에 RDD의 변형을 수행 ( Action / Lazy Evaludation )
  - 대개, 저장 장치에 쓰던가, Driver에게 결과데이터를 보낼 때
  - 이 과정에서 더 빠른 접근과 반복 연산을 위해 스파크 어플리케이션이 동작하는 동안 로드된 RDD를 Executor 노드들의 메모리에 가지고 있을 수 있음 ( Persist or Cache )
3. RDD는 Immutable하도록 구현되어, 데이터 변형 결과는 기존 RDD가 아닌 새로운 RDD를 리턴 

- 액션은 스케줄러를 시작하게 함
- 스케줄러는 RDD 프랜스포메이션 간의 종속성을 바탕으로 DAG 생성
= 스파크는 최종 분산 데이터세트 ( 각 파티션 )의 각 객체를 생성하기 위해 취해야할 일련의 각 단계를 정의하기 위해 역으로 거슬러 올라가는 방식으로 액션을 평가
- 이 실행 계획이라 부르는 각 단계의 순서에 따라 결과를 내놓을 때 까지 스케줄러는 가 단계마다 존재하지 않는 파티션을 계산해서 만듬 
  
--- 

## 지연 평가 

- 대개 In-memory 저장소를 쓰는 다른 시스템들은 변경 가능한 객체를 '잘게 쪼개어' 업데이트하는 방식에 기초를 두고 있음
- e.g) 중간 결과를 저장하면서 테이블의 특정 셀을 업데이트하기 위해 호출하는 것 
- 그에 반해 RDD의 연산은 '지연평가' 방식을 태책
- 액션이 호출되기 전까지는 계산되지 않음 
  > 모든 프렌스포메이션이 100% 지연은 아님
  > sortByKey는 데이터의 범위를 결정하기 위해, RDD를 우선 평가할 필요가 있기 때문에 Transformation과 Action에 둘다 속함

### 지연 평가의 성능과 사용성에서의 장점 
1.  지연 평가는 드라이버와 통신할 필요 없는 연산을 통합하여 데이터가 여러 단계를 거치지 않게함
  > e.g) map -> filter 
  > 스파크는 각 Executor에 map, filter을 수행하는 명령을 합쳐서 보낼 수 있고, 각 파티션에서 map, filter를 한번에 수행할 수 있음
  - Executor에 두번의 명령을 보내고 각 파티션에서 두 번의 연산을 수행하지 않고, 레코드에 한번에 접근해도 되므로 이론적으로 '연산 복잡성을 반으로 줄일 수 있다'

2. 개발자가 직접 연산 통합 작업을 해야하는 MR같은 프레임워크에 비해 동일한 로직을 구현하기 쉬움
  > 개발자는 직접 연관성있는 연산들을 함께 엮어 놓기 하면 이들을 합쳐주는 작업은 스파크 평가 엔진에서 처리


### 지연 평가와 장애 내구성 ( immutable의 이유 ?.. )
- 스파크는 장애에 강하다
- 하드웨어나 네트워크 장애에도 작업이 완전히 실패하지 않고, 데이터 유실이 일어나거나 잘못된 결과를 변환하지 않음
> 거짓말~~ 이 아니라 그렇다면 왜 이렇게 Fault Tolerance 할까 

- 각 파티션이 자신을 재계산하는데 필요한 종속성 정보 등의 데이터를 갖고 있기 때문에 가능!
- 변경 가능한 객체를 사용자에게 제공하는 대부분 분산 프레임워크은 데이터의 변경이력을 일일이 로깅해놓거나 노드들에게 데이터를 복제해 놓는 방식으로 장애에 대비함
- 반면 스파크는 각 파티션이 복제에 필요한 모든 정보를 갖고 있으므로, 각 RDD에서 데이터 변경 내역 로그를 유지하거나, 실제 중간 단계들을 로깅할 필요없음
- 만일 파티션이 유실되면 RDD는 재계산에 필요한 종속성 그래프에 대한 충분한 정보를 가지고 있으므로 더 빠른 복구를 위해 병렬 연산을 수행할 수 있음 

### 지연 평가와 디버깅 - 어려움;; ( stackTrace 자체가 Action에 찍힘 )
- 지연평가는 오직 액션을 수행한 시점에서만 문제가 발생하게 되므로, 디버깅에 중요한 영향을 가져온다.
> e.g) wordcount 실행 시, 

---

## 불변성과 RDD 인터페이스
- RDD의 각 타입이 구현해야만 하는 속성들을 RDD Interface에 정의해 놓음
- 이 속성들이란 실행 엔진이 RDD를 계산하는데 필요한 데이터 위치에 대한 정보나, RDD의 종속성 같은 것이 포함된 정보
- 내부적으로 스파크는 RDD를 표현하는 다섯 가지의 속성이 있음 
- 위의 3가지가 필수적인 정보, 아래 2개는 부가적 

1. RDD를 구성하는 파티션의 리스트
2. 각 파티션에 대한 반복 연산을 담당하는 함수들
3. 다른 RDD들을 가르키는 종속성 목록
4. 파티셔너 ( = 스칼라 튜플로 표시되는 키/쌍 쌍을 레코드로 가지는 RDD를 위함 )
5. 기본 위치 목록 (HDFS을 위함)

--- 
- partitions()
  - 분산 데이터세트의 부분들을 구성하는 파티션 객체들의 배열을 리턴
  - 파티셔너를 가진 RDD 라면 각 파티션의 인덱스는 그 파티션의 데이터가 가진 키에 getPartition()을 호출했을 때의 결과와 같음

  ```
  /**
   * Get the array of partitions of this RDD, taking into account whether the
   * RDD is checkpointed or not.
   */
  final def partitions: Array[Partition] = {
    checkpointRDD.map(_.partitions).getOrElse {
      if (partitions_ == null) {
        partitions_ = getPartitions
        partitions_.zipWithIndex.foreach { case (partition, index) =>
          require(partition.index == index,
            s"partitions($index).partition == ${partition.index}, but it should equal $index")
        }
      }
      partitions_
    }
  }
  
  ```


- iterator(p,parentlter)
  - 각각의 부모 파티션을 순회하는 Iterator가 주어지면, 파티션 p의 구성요소들을 재 계산한다
  - 이 함수는 RDD에서 각각의 파티션을 계산하기 위해 호출
  - 사용자가 직적 호출 X , 액션이 수행할 때, 스파크에 의해 호출되는 용도
  
  ```
    /**
     * Internal method to this RDD; will read from cache if applicable, or otherwise compute it.
     * This should ''not'' be called by users directly, but is available for implementors of custom
     * subclasses of RDD.
     */
    final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
      if (storageLevel != StorageLevel.NONE) {
        getOrCompute(split, context)
      } else {
        computeOrReadCheckpoint(split, context)
      }
    }

  ```

- dependencies()
  - 종속성 객체의 목록을 리턴
  - 종속성 정보는 스케줄러가 현재의 RDD가 어떤식으로 다른 RDD에 종속될지에 알려준다
  - Narrow dependency
      >  - 부모의 파티션 중에서 한개나 작은 부분 집합만 필요로 할 때
  - Wide dependenct
      > - 파티션이 반드시 부모의 모든 데이터를 재배치하고 연산해야만 되는 경우 

  ```
   /**
     * Get the list of dependencies of this RDD, taking into account whether the
     * RDD is checkpointed or not.
     */
    final def dependencies: Seq[Dependency[_]] = {
      checkpointRDD.map(r => List(new OneToOneDependency(r))).getOrElse {
        if (dependencies_ == null) {
          dependencies_ = getDependencies
        }
        dependencies_
      }
    }
  ```

- paritionor()
  - hashPartitioner같이 element와 partition 사이에 연관되는 함수를 갖고 있는 RDD라면 Option타입으로 partitioner 객체 리턴
  ```
    /** Optionally overridden by subclasses to specify how they are partitioned. */
    @transient val partitioner: Option[Partitioner] = None
  ```
 
- preferredLocation(p)
  - 파티션 p의 데이터 지역성에 대한 정보를 리턴
  - p가 저장된 각 노드의 정보를 문자열로 표한한 Seq를 리턴
  - HDFS 경우, preferredLocation 결과의 각 문자열이 파티션이 저장된 노드의 하둡 이름이 된다?
  ```
    protected def getPreferredLocations(split: Partition): Seq[String] = Nil
  ```
