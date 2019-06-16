---
layout: post
title:  "[HP-Spark] 파티셔너와 키/값 데이터 "
date: 2019-06-14 18:45:12
categories: High-Performance-Spark 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## 파티셔너와 키/값 데이터

- 스파크의 파티션 하나는 하나의 태스크와 연관되는 병렬 실행의 한 단위
- 명시적 파티셔너가 없는 RDD는 **데이터 크기**와 **파티션 크기**에 의해서만 데이터를 파티션들에 할당함

RDD 파티션되는 방식을 변경하는 메소드

1. repartition / coalesce
> - RDD의 값에 개의치 않고 파티션 개수를 바꾸는데 쓸 수 있음
> - repartition은 RDD를 해시파티셔너와 주어진 파티션 개수와 함께 셔플함
> - coalesce는 요구하는 파티션 개수보다 현재 파티션 개수가 적을 경우, 전체 셔플링을 회피하기 위한 repartition은 최적화 버전

2. partitionBy
> - 파티션 개수가 아니라, 파티셔너 객체를 받아 새로운 파티셔너로 RDD를 셔플
> - 파티셔너는 키의 값을 기반으로 파티션을 레코드에 할당

- repartition / coalesce는 명시적으로 파티셔너를 RDD에 할당하지 않음

---

### 스파크 파티셔너 객체 사용하기

파티셔너란?
- 레코드를 어떻게 분산하고 그로 인해 각 태스크가 어떤 레코드들을 처리할 것인가를 정의하는 것
- numPartitions, getPartition 두 함수를 가지는 추상 클래스
> - numPartitions 
> > - 파티셔닝이 끝난 후 RDD의 파티션 개수를 정의
> - getPartition(key)
> > - 키에 대해 키와 연계된 레코드가 전송되는 파티션의 정수 인덱스와의 매핑을 정의

---

### 해시 파티셔닝

- 페어 RDD 연산의 기본 파티셔너, HashPartitioner
- 키의 해시값을 기반으로 자식 파티션의 인덱스를 결정
- 인자로 partitions 사용, RDD의 파티션 개수와 해싱 함수에 쓰일 버킷 개수를 결정
- partition를 정의안하면, spark.default.parallelism 값 사용

```scala
/**
 * A [[org.apache.spark.Partitioner]] that implements hash-based partitioning using
 * Java's `Object.hashCode`.
 *
 * Java arrays have hashCodes that are based on the arrays' identities rather than their contents,
 * so attempting to partition an RDD[Array[_]] or RDD[(Array[_], _)] using a HashPartitioner will
 * produce an unexpected or incorrect result.
 */
class HashPartitioner(partitions: Int) extends Partitioner {
  require(partitions >= 0, s"Number of partitions ($partitions) cannot be negative.")

  def numPartitions: Int = partitions

  def getPartition(key: Any): Int = key match {
    case null => 0
    case _ => Utils.nonNegativeMod(key.hashCode, numPartitions)
  }

  override def equals(other: Any): Boolean = other match {
    case h: HashPartitioner =>
      h.numPartitions == numPartitions
    case _ =>
      false
  }

  override def hashCode: Int = numPartitions
}

```

---

### 레인지 파티셔닝

- 동일한 범위의 키들에 대한 레코드들을 주어진 파티션에 할당
- 레코드들을 정렬해서 해당 파티션 안에 있는지 확인할 수 있도록 **정렬**을 요구하므로, 전체 RDD가 정렬될 것 
- 샘플링과 전체 파티션들에 레코드를 균등하게 분배시키는 최적화에 의해 각 파티션에 지정될 범위를 결정
- 한 키에 대한 모든 레코드 중 중복 키가 너무 많으면 해시 파티셔닝 처럼 메모리 에러를 발생시킬 수 있음
- 인자로 partitions 뿐 아니라, 샘플을 위한 실제RDD를 필요로함
> - RDD는 튜플이여야하며, 키들은 정의된 순서를 갖고 있어야함

- 샘플링은 실제로 RDD를 부분적으로 평가하며, 실행 그래프를 끊게 됨
> - 그래서, 트랜스포메이션이면서 액션이기도 함

```scala

/**
 * A [[org.apache.spark.Partitioner]] that partitions sortable records by range into roughly
 * equal ranges. The ranges are determined by sampling the content of the RDD passed in.
 *
 * @note The actual number of partitions created by the RangePartitioner might not be the same
 * as the `partitions` parameter, in the case where the number of sampled records is less than
 * the value of `partitions`.
 */
class RangePartitioner[K : Ordering : ClassTag, V](
    partitions: Int,
    rdd: RDD[_ <: Product2[K, V]],
    private var ascending: Boolean = true)
  extends Partitioner {

    // too long..
 }
```

---


### 사용자 파티셔

- 키의 해시값, 순서에 의한 것이 아닌 데이터 파티셔닝을 위한 특별한 함수 정의할 떄 사용

다음과 같은 메소드를 구현해야함

1. numPartitions
> - 파티션의 개수를 돌려주는 메소드 

2. getPartition(key : Any)
> - 키를 받아 키의 레코드가 속한 파티션의 인덱스 리턴

3. equals
> - 파티셔너 간의 동질성을 정의하는 메서드
> - HashPartitioner는 파티션 개수가 같으면 true 리턴
> - RangePartitioner는 범위 구분이 같다면 true 리턴
> - 조인이나 cogroup에서 RDD가 어떤 파티셔너에 의해 파티셔닝이 되어있다면, 재파티셔닝을 하지 않기 때문에 중요함

4. hashcode
> - HashPartitioner은 파티션의 개수
> - RangePartitioner은 각 범위 구분값의 타입에서 가져온 해시 함수를 그대로 사용
> > - 모든 범위 경계값에 대해 해시함수 호출하고 이들을 합쳐서 하나의 값으로 만듬
> > - 같은 범위들을 지닌다면 동일한 hashcode가 나오고, 범위가 하나라도 달라지면 역시 달라짐


---


### 파티셔닝을 유지하는 좁은 트랜스포메이션들

키/값 쌍의 값 부분만 바꾸는 종류가 아니라면, 결과 RDD는 파티셔너를 가지지 않음
- map/flatMap
> - 키를 바꿀 가능성이 있으므로, 키를 변경하지 않더라고 결과 RDD는 파티셔너를 가지지 않ㄷ음
- mapValues
> - 파티셔너 보존
- mapPartitions
> - perservesPartitioning 플래그가 true로 설정하면 보존함

```scala
  /**
   * Return a new RDD by applying a function to each partition of this RDD.
   *
   * `preservesPartitioning` indicates whether the input function preserves the partitioner, which
   * should be `false` unless this is a pair RDD and the input function doesn't modify the keys.
   */
  def mapPartitions[U: ClassTag](
      f: Iterator[T] => Iterator[U],
      preservesPartitioning: Boolean = false): RDD[U] = withScope {
    val cleanedF = sc.clean(f)
    new MapPartitionsRDD(
      this,
      (context: TaskContext, index: Int, iter: Iterator[T]) => cleanedF(iter),
      preservesPartitioning)
  }
```

```scala

val sortedData = data.sortByKey()
val mapValues: RDD[(Double,Strring)] = sortedData.mapValues(_.toString)

assert(mapValues.partitioner.isDefined, " Using Map Values perserves partitioning")

val map = sortedData.map(pair => (pair._1, pair._2.toString))
assert(map.partitioner.isEmpty, " Using map deoest not perserve partitioning")

```




