---
layout: post
title:  "Mastering Spark in Scalable Algorithms"
date: 2018-07-02 13:05:12
categories: Spark
author : Jaesang Lim
tag: Spark
cover: "/assets/spark_log.png"
---

### Mastering Spark in Scalable Algorithms
- Spark의 아키텍쳐와, 문제상황에서 어떻게 해결할까에 대한 디자인패턴에 대해 다룰 것~~

#### Spark job을 어떻게 만들 것인가에 대한 기본 원칙
1. 가능한 data locality를 지킬 것
	- Spark에서 기본적으로 해주므로 걱정할 필요 없지만 확인할 것
	- 각 stage에서 불필요하게 데이터를 이동시키지 않는가?
2. 데이터의 균등 분배를 보장할 것
	- 놀고 있는 executor 없도록 적절하게 데이터 분배 
3. 더 빠른 저장소를 선호
	- L1 cache > L2 cache > Main memory > Disk(random seek)
	- Spark에서 in-memory 처리 기능도 제공
	- 불필요하게 main memory나 disk 쓰지말고 빠른 cache 활용해도 좋음(Project Tungsten)
4. 지켜본 후 최적화할 것
	- Donald Knuth왈, “어설픈 최적화는 모든 악의 근원이다.”
	- evidence-based로 접근하라
	- 작게 시작해서 크게 늘릴 것

<hr/>

#### Spark Arcitecture
<center><img src="https://user-images.githubusercontent.com/12586821/47960850-d1611a80-e044-11e8-8062-4bba39801331.png" /></center>
1. Driver
	- Spark의 main entry point
	- single JVM
	- job 시작 및 모든 operation 제어
	- 너무 큰 데이터셋을 driver로 가져오지 말것 (rdd.collect) -> OOM
	- 필요하다면 JVM heap 사이즈를 늘려줄 것 (--driver-memory)
2. SparkSession
	- driver가 시작할 때 SparkSession class가 초기화됨
	- SQLContext, SparkContext, StreamContext class 등을 통해 모든 Spark service들에 접근 제공
	- Spark runtime performance-related 설정 튜닝하는 위치

3. Resilient distributed datasets (RDDs)
	- 동종 데이터의 분산 셋에 대한 추상화된 표현
	- 실제 데이터는 클러스터 내 여러 노드에 저장되지만 분석 시 실제 위치를 알 필요는 없음. RDD활용하면 됨
	- RDD는 케이크 조각처럼 partition들로 구성되어 있고,  각 partition들은 복제됨(Spark가 data locality를 고려하여 결정), 
	-[newHadoopRDD Github](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/rdd/NewHadoopRDD.scala) 
	- RDD는 HDFS같은 block storage로 부터 데이터가 제대로 cache되는 것을 보장할 책임이 있음

4. Executor
	- 일꾼 노드
	- 각 executor는 driver와 연결되어 있고 데이터에 대한 작업을 실행하기 위한 지침을 기다림

5. Shuffle operation
	- executor간 물리적 데이터 전송
	- 주로 같은 키로 이루어진 데이터 그룹 이동
	- 전략적으로 더 높은 병렬도를 위한 data repartition
	- 네트워크를 통한 데이터 이동, 데이터를 디스크에 저장해야 하므로 매우 느림
	- scalability에 매우 중요한 부분이라 할 수 있음

6. Cluster manager
	- Spark 외부에서 cluster의 리소스 교섭자(negotiator) 역할을 함
	- executor의 코어 개수나 메모리 할당
	- cluster manager는 여러가지 있으나 algorithmic 성능에 크게 영향을 줄 가능성은 낮음

7. Task
	- Data의 single partition에 대한 일련의 작업을 실행하기 위한 instruction
	- Processing을 데이터로 이동시킨 것

8. DAG
	- Action에 대한 모든 transformations의 논리적 실행 계획(execution plan)
	- SparkSQL, Datasets의 최적화는 catalyst optimizer가 대신함

9. DAG scheduler
	- physical plan 만듦
	- DAG에서 stage로 나누고, 각 stage는 그와 대응되는 일련의 task(각 partition당 하나)를 생성함

10. Transformation
	- 기존의 RDD data를 변경하여 새로운 RDD data를 생성해내는 것. 
	- filter와 같이 특정 data만 뽑아 내거나 map 함수 처럼, data를 분산 배치 하는 것 등을 들 수 있다.
	[참고](http://bcho.tistory.com/1027)

11. Action
	- RDD 값을 기반으로 무엇인가를 계산해서(computation) 결과를 (셋이 아닌) 생성해 내는 것
	- count()와 같은 operation들이 있음

12. Stage
	- 물리적으로 task가 매핑될 수 있는 작업의 그룹 (partition 당 하나)
	- 연속된 narrow transformation은 single stage. 동일한 executor에서 작업이 가능하므로 shuffle이 필요없음
	- wide transformation을 만나면 stage boundary가 발생하고, 2개의 stage가 존재할 때 첫번째 stage가 완료될 때까지 두번째 stage는 시작될 수 없음 

13. Task scheduler
	- 일련의 task(DAG scheduler에 의해 결정된)를 전달받고(partition당 하나)
	- data locality와 함께 적절한 executor에서 각각 실행되도록 스케줄함  

<hr/>
#### Tune your analytic

- Spark UI 확인할 것
- 리소스 병목 확인, 코드에서 어느 부분이 시간을 많이 잡아먹고 있는가

1. Input Size or Shuffle Read Size/Records
	- task별 data read 양 (remote나 local 관계없이)
	- 너무 많다면 executor를 늘리거나 repartition 고려할 것

2. Duration
	- task가 실행된 시간
	- 만약 input size가 적은데 duration이 길다면, CPU-bound일 수 있음
	- Thread dump 활용해서 어디서 시간 잡아먹는 지 확인
	- Spark UI에서 보면 min, 25%, median, max 등등 수치 있는데 그 수치의 variance 확인할 것
	- [CPU-bound, I/O-bound, Memory-Bound 에 대한 설명](https://code.i-harness.com/ko/q/d40d8)

3. Shuffle Write Size/Records
	- 가능한 적게

4. Locality Level
	- Stage 페이지에 있는 data locality 수치
	- PROCESS_LOCAL 제일 좋음
	- 보통은 도움 안되지만 narrow transformation에서 NODE_LOCAL 이나 RACK_LOCAL이 많다면 executor의 수를 늘려봐라

5. GC Time
	- 각 task 별로 GC에 소요되는 시간
	- 전체 시간에 10프로 미만 권장
	- 너무 높다면 뭔가 근본적인 문제가 있다는 것임
	- GC문제라기 보다는 data distribution과 관련된 부분을 review해 볼 것 
	- executor 수, JVM heap size, 파티션 수, 병렬도, 데이터 치우침(skew) 등등 

6. Thread dump (per executor)
	- Executor 페이지 확인
	- executor 내부 작업 엿보기
	- 정렬, interested thread를 상단에 리스팅(Executor task launch worker)

7. Skipped Stages
	- 실행할 필요없는 stage
	- RDD가 cache되어 있어서 재 연산할 필요없다는 의미임
	- 일반적으로 good caching strategy의 sign

8. Event Timeline
	- Stages 페이지에서 실행중인 task의 timeline 제공
	- 병렬도 수준 확인, 주어진 시간에 executor당 몇개의 task가 실행되는 지 확인 

<hr/>

### Design patterns and techniques

#### 1. Spark APIs
- Problem
	- API와 function들이 너무 많아서 어떤 것이 가장 성능이 좋은 지 알기 어려움
- Solution
	- Off-heap explicit memory management, cache-miss improvement, dynamic stage generation 같은 최근 최적화(project tungsten) 기술은 DataFrame과 Dataset만 지원됨.
	-  RDD는 안됨
	- 새롭게 소개된 Encoder 역시 Kryo Serialization이나 Java serialization 보다 빠르고 공간 효율성도 좋음

```
personDS.groupBy($"name").count.sort($"count".desc).show
// DS 36 seconds
personRDD.map(p => (p.person,1)).reduceByKey(_+_).sortBy(_._2.false)
// RDD 99 seconds
```
#### 2. Summary pattern
- Problem
	- 엄격한 SLA(service level agreements)가 있을 경우 전체 데이터를 가지고 계산할 시간적 여유가 없음
- Solution
	- Summary pattern은 two-pass algorithms
	- 전체 데이터를 직접 처리하지 않지만, aggregation의 결과가 전체 데이터에서 실행했을 때와 같도록 만듦
	- 적절한 구간의 summary를 계산(분 당, 일 당, 주 당 등등)
	- 나중에 사용하기 위해 summary데이터 저장
	- 더 큰 구간에 대한 aggregate를 계산
	- 스트리밍 분석을 위한 incremental 혹은 online algorithm에 유용함

#### 4. Expand and Conquer pattern
- Problem
	- 상대적으로 적은 수의 task, 각 task는 높은 input/Shuffle Size
	- task 완료시간이 오래걸리며, 때로는 idle executor가 생기는 경우도 있음
- Solution
	- flatMap을 써서 shuffle시킴
		- 각 row당 많은 record가 생기므로(tokenized) 시간 복잡도 증가할 수 있음
	- repartition을 써서 task 수 늘리고 task당 처리하는 data 양을 줄임
		- OOM도 줄일 수 있음 


#### 5. Lightweight shuffle
- Problem
	- 전체 시간 중 Shuffle Read Blocked Time의 비중이 높을 때(>5%)
	- 어떻게 shuffle 완료를 기다리는 것을 피할 수 있나?
- Solution
	- Spark에서 data compression, merge file consolidation 등 많은 테크닉을 쓰지만 그래도 성능 병목이 되는 2가지 근본적인 문제가 있음
		1. I/O intensive함 (집중적으로 사용함)
			- 네트워크 타고 데이터 이동, target machine에 데이터 쓰기
			- cache나 local partition에 비해 120배 느림
		2. 동시성을 위한 동기화 포인트임
			- stage 내 각 task가 모두 끝나야 다음 stage로 넘어갈 수 있음
	- shuffle은 가능한 피하거나 줄이거나

- Solution
	- Dataset이나 DataFrame API를 쓴다면 실행 계획 만들 때 50가지 이상의 최적화 기술 들어감
		- 안쓰는 컬럼 이나 partition 자동 pruning
		- [Spark Sql Catalyst Optimizer github](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala)

	- RDD일 경우 몇가지 테크닉
		- shuffle 하기 전에 map을 이용해서 안쓰는 데이터 제거
		- key-value pair를 가지고 있다면, rdd대신 rdd.keys 활용 검토
			- count나 membership 테스트일 경우 key만 있어도 충분함
		- stage의 순서 조정
			- join한 뒤 group by 할 것인가 group by한 뒤 join할 것인가
		- transformation 전 후의 레코드 수 비교하여 비용산정해봐야 함

	- Filter first
		- shuffle전에 filter할 거 있으면 먼저 해야함

	- CoGroup 사용
		- 두개 이상의 RDD있을 때, 같은 키끼리 모으고 싶다면 CoGroup 활용
		- 같은 타입의 K를 키로 사용하는 RDD[(K,V)]를 HashPartitioner를 이용하여 group지어서 항상 같은 노드에 가도록 함

	- 다른 codec 활용
		- lz4, lzf, snappy 사용해보고 어떤 게 좋은지 결정

#### 6. Broadcast variables pattern
- Problem
	- 분석에 작은 크기의 reference dataset들과 dimension table들이 값비싼 shuffle을 유발함

- Solution
	- transaction log나 트윗같이 무한히 증가하는 데이터 말고 UK postcode 같은 유한한 데이터(bounded datasets) - 종종 바뀌지만 유한한 크기
	- bounded dataset일 경우 join하지 말고 broadcast variable을 활용해서 모든 executor에 다 쏴라 

#### 7. Optimized cluster
- Problem
	- 어떻게 설정을 해야 클러스터의 전체 리소스를 사용할 수 있는지 잘 모르겠음
- Solution
	- 일반적으로 큰 executor보다는 많은 executor 선호
	- YARN-기반 클러스터에서 리소스 분배 측정법
		1. number of executors = (total cores – cluster overhead) / cores per executor
			- ((T – (2*N + 6)) / 5)
				- T = 클러스터의 총 코어 수, N = 클러스터의 총 노드 수
				- 2 = HDFS와 YARN의 오버헤드 (노드에 Datanode, NodeManager있다고 가정)
				- 6 = HDFS와 YARN의 master process
					-  NameNode, ResourceManager, SecondaryNameNode, ProxyServer, HistoryServer 등 (주키퍼, HA등 때문에 더 추가될 수 있음)
				- 5 = executor 당 코어수, I/O 경합없는 최적의 task concurrency (입증할 수 없지만)

		2. mem per executor = (mem per node / number of executors per node) * safety fraction
			- (64 / E)* 0.9 => 57.6 / E
			- E = 노드 당 executor 수
			- 0.9 = off-heap overhead(기본 10%)를 제외한 heap에 할당된 실제 메모리 비율
			- 일반적으로 executor에 많은 메모리 할당하는 것이 좋음(cache, sort space)
			- GC오래 걸릴 수 있음 / Spark UI보면서 조정할 것


#### 8. Redistribution Pattern
- Problem
	- 항상 적은 수의 executor만 실행됨
- Solution
	- repartition 혹은 파티션 수 명시하면 re-partition함
	- 반대로 너무 많은 task(10000+)가 뜬다면 coalesce



#### 9. Salting Key Pattern
- Problem
	- Task의 대부분은 괜찮은 시간에 끝나는 데 한 두개가 오래걸림. repartition해봐도 별 효과없음

- Solution
	- 데이터 분포가 한 쪽으로 치우침(skew)
	- 몇몇 task가 너무 오래 걸리거나 input 혹은 output이 너무 많음
	- rdd.keys.count 해서 RDD의 키가 executor의 수 보다 너무 적다면 키 전략 다시 고려해 볼 것
	- salting key한 후 예전 key로 re aggregation
```
rdd filter {
case (k,v) => isPopular(k)
}
.map {
case (k,v) => (k+ r.nextInt(n),v)
}
```

#### 10. Secondary sort Pattern
- Problem
	- 키 별로 grouping하고 sorting할 때 OOM
```
rdd.reduceByKey(_+_).sort(_._2,false) // inefficient for large groups
```
- Solution
	- Composition key
		- 그룹화할 요소와 정렬할 요소 포함
	- Grouping partitioner
		- Composition key의 어느 부분이 grouping과 관련있는 지 이해
	- Composite key ordering
		- Composition key의 어느 부분이 sorting과 관련있는 지 이해


- Example 
```
case class Mention(name:String, ariticle:String,Published: Long)
//Composite key :
cas class SortKey(name:String,pulished :Long)
```
최근에 멘션 날린 이름 순으로 정렬하고 싶을 때
이름과 멘션 날짜로 composite key 생성

- 이름 별로 grouping하는 Grouping partitioner 구현 
 
```
class GroupingPartitioner(paritions: Int) extends Pratitioner {

override def numParitions:Int = partitions

override def getParition(key: Any): Int = {
val groupBy = key.asInstanceOf[SortKey]
groupBy.name.hashcode() % numParitions
}
}
```

- 발행시점으로 정렬하는 Composite key ordering 구현
- 주어진 partitioner로 repartition 하면서 repartition 결과의 키가 정렬 됨
- 지금 예제에서는 implicit에서 sortBy를 published로 바꾼 것임 
- Keyby 는 map의 특별한 case로 Composite key 생성한 것을 키로 하고 m을 value로 만듦 


```
implicit val sortBy: Ordering[SortKey] = Ordering.by(m => m.published
val pais = mentions.rdd.keyBy(m => SortKey(m.name,m.published))
pairs.repartiionAndSortWithinParitions(new GroupingParitioner(n))
```


#### 11. Probability Algorithms
- Problem
	- 데이터셋 너무 커서 통계량 구하는 데 너무 오래 걸림
	- 조금 틀려도 좀 더 빨리 받고 싶음
- Solution
	- 시간이 문제될 때
		- rdd.countApprox(), rdd.countByValueApprox(), rdd.countApproxDistinct()
		- org.apache.spark.sql.DataFrameStatFunction의 df.stst

	- 공간이 문제될 때
		- Bloom Filter
			- membership test
			- 있는 건 없다고 안하지만 없는 건 있다고 할 수 있음
		- HyperLogLog
			- 컬럼의 distinct 값
		- CountMinSketch
			- 데이터 스트림에서 이벤트 발생 빈도 테이블 제공(Spark Stream에서 유용)


#### 12 .Selective caching
- Problem
	- cache사용하니까 이전보다 느려짐

- Solution
	- RDD를 한번 이상 쓸 때 cache가 효율적임
		- 데이터가 여러 stage 교차적으로 사용될 때
		- 데이터가 다수의 child 데이터셋의 lineage에 있을 때
		- iterative process일 때 (stochastic gradient descent) 

	- 재활용안하면 문제됨
		- cache overhead(만들고 업데이트 하고 flush하고 안쓰면 GC하고)

	- 너무 크면 문제됨
		- OOM 혹은 디스크로 swap data
		- ds.cache -> ds.count 한 후 spark UI의 Storage에서 확인

		- Solution
			- OOM날 경우 MEMORY_AND_DISK 활용 검토
			- GC시간 길 경우 MEMORY_AND_SER 검토
			- Fraction Cached가 100%이면서 Size on Disk가 최대한 작게 만드는 것이 목표 

#### 13. Garbage collection
- Problem
	- GC time이 전체 processing time에서 상당한 비율을 차지할 때(> 15%)
- Solution
	- Spark의 GC는 자체적으로 꽤 효율적으로 작동하므로, 확실할때만 조정해야함 (내 잘못)
	- 과도한/비정상적인 메모리 소비의 근원이 아닌지 확인
		- cache 전략 확인
		- 안쓰는 RDD는 명시적으로 unpersist 
	- 객체 할당 너무 많이 하는지 확인
		- domain model 단순화
		- 객체 재사용
		- 가능한 primitive Type선호
		- map -> mapPartitions

- Solution
	- G1 GC사용 권장	
	- XX:UseG1GC 
	- InitiatingHeapOccupancyPercent
		- 45% 미만이면 pause 횟수 좀 줄일 수 있음
		- -XX:InitiatingHeapOccupancyPercent
 	- ConcGCThread
		- GC Thread 많을 수록 GC 빨리함
		- CPU 많이 씀
		- XX:ConcGCThread




