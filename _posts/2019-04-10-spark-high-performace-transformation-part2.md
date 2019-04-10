---
layout: post
title:  "[HP-Spark] 효율적인 트랜스포메이션 : Part 2"
date: 2019-04-10 20:05:12
categories: High-Performance-Spark 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## 트랜스포메이션으로 어떤 타입의 RDD가 반환하는가

- RDD는 두가지면에서 추상적인 개념
  > 1. 거의 모든 타입의 레코드를 가질 수 있음 ( e.g) String, Row, Tuple )
  > 2. RDD 인터페이스에 다양한 속성을 추가한 여러 구현체 중의 하나가 될 수 있음 
  
1. 일부 트랜스포메이션은 특정한 레코드 타입을 가지는 RDD에만 적용할 수 있음
2. 각 트랜스포메이션이 RDD의 여러 구현체 중 하나를 반환하기 때문에 중요
> - 즉, 다른 RDD 구현체에 동일한 트랜스포메이션이 호출되어도 다르게 평가될 수 있음 ( mappedRDD / CoGroupedRDD 같은..)

- RDD란 scala, java 같은 강타입 언어들과 매우 유사한 collection 타입이며, collection 멤버들의 타입을 가리키는 타입으로 인스턴스화 진행
- 타입정보를 잃어버린 RDD는 RDD[Any] 나 RDD[(Any,Int)] 같다면 sortByKey가 내부적으로 순서가 정의된 키를 가진 키/쌍 RDD에 대해서만 호출가능하기 때문에
컴파일 되지 않음

- 입력과 반환 타입을 구체적으로 지정하고, 제너릭 타입은 쓰지않는 것이 최선인 경우가 대부분임

- DataFrame을 RDD로 작업할 때, 타입 정보를 잃어버리는 경우가 많음
- DataFrame은 암묵적인 변환에 의해 Row의 RDD로 변환할 수 있지만, Row객체가 강타입이 아니므로, 컴파일러는 Row 객체를 만들기 위해 쓰인 값의 타입을 기억하지 못함
- Row 객체에 인덱스로 접근하면 Any 타입의 객체로 반환해주고, 필요에 따라 casting 해줘야함
- DataSet의 장점 중 하나가 강타입이라서, RDD로 변환한 뒤에도 타입 정보가 유지가 된다.

---

## 객체 생성 최소화하기
- GC는 더 이상 쓰이지 않는 객체에 할당된 메모리를 정리하는 과정
- Spark는 JVM에서 실행되는데, JVM은 자동으로 메모리 관리를 하고 많은 자료 구조들을 가지고 있어 스파크 작업에서 GC는 부담이 될 수 있음
- GC의 오버헤드가 작업 실행을 끊지는 않더라도 직렬화에 걸리는 시간을 늘리면서 느려지기도 함 
- 그래서, 우리는 개발할 때 객체의 크기나 숫자를 줄임으로써 GC에 대한 비용을 최소화할 수 있음
- 즉 ! 기존 객체를 재활용하고 메모리 공간을 덜 쓰는 자료구조, primitive 타입을 쓰는 등 객체 크기나 개수를 줄일 수 있음

### 기존 객체 재활용하기
- 어떤 RDD 트랜스포메이션은 새로운 객체를 반환하지않고, 람다식에서 인자를 수정할 수 있게 해준다
> - aggregate나 aggregateByKey 같은 집계 연산 함수를 위한 Sequence / Combine 활용할 수 있음

- 필요 함수
> 1. 초기값 
  > - 최초 누적 변수의 값 설정 
> 2. Sequence 함수
  > - 누적 변수와 값 하나를 받아 누적 변수를 업데이트 하는 함수
> 3. Combine 함수
  > - 누적 변수 2개를 받아 어떻게 합쳐서 결과 누적 함수를 만드는 함수 

- 객체 사용을 줄이면서 this.type Annotation을 사용해 기존 객체를 돌려주는 방법에 대한 코드 및 내용 정리
[객체 재사용하지 않음](https://github.com/jjaesang/Spark-Training/blob/master/src/main/scala/com/study/transform/MetricsCalculator.scala)
[객체 재사용 + this.type](https://github.com/jjaesang/Spark-Training/blob/master/src/main/scala/com/study/transform/MetricsCalculatorReuseObject.scala)

- 더 작은 자료구조를 사용할 수 있음
> - case class나 tuple보다는 배열을 쓰는 것이 GC오버헤드를 줄이는데 효과적
> - scala의 배열은 내부적으로 java배열과 동일하며, scala Collections 중 메모리 효율이 가장 뛰어남
> - scala tuple은 객체이므로 비용이 큰 연산에 대해서는 tuple보다는 두세개짜리 배열 쓰는 것이 더 나을 수 도 있음


---

다음은 mapPartitions , 셋업 오버헤드를 줄이는 브로드캐스트, 어큐물레이터에 대해 다룰 예정
