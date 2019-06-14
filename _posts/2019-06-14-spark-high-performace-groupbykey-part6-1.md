---
layout: post
title:  "[HP-Spark] groupByKey 함수는 왜 그렇게 위험한가? "
date: 2019-06-14 18:45:12
categories: High-Performance-Spark 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## GroupByKey 함수는 왜 그렇게 위험한가? 

- 각 키에 대해 반복자(Iterable)를 되돌려 주는 함수 groupByKey의 확장성에 대해 경고함
- groupByKey가 크기와 관련해 문제를 발생시킬 수 있으며
- groupByKey 대신 사용할 수 있는 방법에 대한 제안과 조언을 설명할 것 


---

### 골디락스 GroupByKey 해결책 

[기존 소스 및 방법](https://github.com/jjaesang/Spark-Training/blob/master/src/main/scala/com/study/goldilocks/GoldilocksWhileLoop.scala)
> - Map으로 그룹처리

```scala

  def findRankStatistics(dataFrame: DataFrame,
                         ranks: List[Long]): Map[Int, Iterable[Double]] = {
    require(ranks.forall(_ > 0))

    val pairRDD: RDD[(Int, Double)] = mapToKeyValuePairs(dataFrame)

    val groupColumns: RDD[(Int, Iterable[Double])] = pairRDD.groupByKey()
    groupColumns.mapValues(
      iter => {
        val sortedIter = iter.toArray.sorted

        sortedIter.toIterable.zipWithIndex.flatMap({
          case (colValue, index) =>
            if (ranks.contains(index + 1)) {
              Iterator(colValue)
            } else {
              Iterator.empty
            }
        })
      }).collectAsMap()
  }
  
```


groupByKey는 각 키에 대한 값들을 반복자로 돌려주므로, 키별로 값들을 정렬하려면 배열로 변환하고 정렬해야함
> - 반복자는 오직 한번만 값을 순회할 수 있음

장점
1. 근사값이 아닌 정확한 결과를 내놓는다 ( 당연한 소리를..? )
2. 짧고 이해하기 쉬움
> - 스파크와 스칼라의 기본 함수를 사용해, 예외적인 경우가 별로 없고 테스트하기도 쉬움 
3. 입력 데이터에 칼럼은 많지만, 레코드 수가 매우적다면, groupByKey에서 단 한 번의 셔플을 요구함
> - 이그제규터들 내에서 정렬 단계는 좁은 트랜스포메이션으로 수행되므로 상대적으로 효과적 

결과
- 환경 : 1만 개 레코드와 수천 개의 칼럼
- groupByKey가 [기존 방법](https://github.com/jjaesang/Spark-Training/blob/master/src/main/scala/com/study/goldilocks/GoldilocksWhileLoop.scala) 보다 자릿수가 다를 정도로 빠름
- 그러나, **수백만 개**의 레코드로 테스트 했을때, 다수의 노드 환경에서 지속적으로 메모리 에러를 냈음.. 

---

### 골디락스 GroupByKey.. 왜 실패했는가?

- groupByKey에 의해 만들어진 group은 항상 반복자가 되며, 분산될 수 없음
- 결국, 스파크는 모든 셔플 데이터를 메모리에 읽어 들어야하는 고비용의 **Shuffle Read** 단계가 필요로 함 
> - e.g) 입력 데이터가 200MB 인데, Shuffle Read는 86MB ..
- 스파크는 거의 모든 셔플 데이터를 메모리에 읽어 들어야함 
- 키들의 해시값으로 파티셔닝하고, 결과를 반복자로 그룹 지어 계속 메모리에 올리게 되므로, 키별로 중복 레코드가 많으면 이그제큐터의 **OOM** 자주 발생
> - 즉, 키가 값은 해시값을 가지는 레코들은 하나의 머신의 메모리에 모두 모여야함 
> - 키 중에 하나라도, 단일 이그제큐터의 메모리에 들어가지 못할 만큼 많은 레코드라면, 전체 연산이 실패함.. ㅠㅠ

해결법
1. 셔플 이전에 키당 레코드 개수를 줄일 수 있는 **맵사이드 리듀스** 수행하는 집계연산을 선택
> - aggregateByKey 또는 reduceByKey
2. 한 키와 연관된 모든 키값이 메모리에 올라오지 않아도 되는 넓은 트랜스포메이션 사용
> -  **보조 정렬과 repartitionAndSortWithinPartitions 함수** 방법 사용, 다음에 다룰 것임 
3. groupByKey를 꼭 써야한다면, mapPartitions로 수행하는 **반복자-반복자 트랜스포메이션** 사용
> - 이어지는 연산을 반복자-반족자 트랜스포메이션으로 하는 것이 최선

