---
layout: post
title:  "Spark Join에 대한 정리"
date: 2018-12-23 13:05:12
categories: Spark
author : Jaesang Lim
tag: YARN
cover: "/assets/spark_log.png"
---

### Spark Join

#### 1. 조인 표현식
- 스파크는 왼쪽과, 오른쪽 데이터셋에 있는 하나 이상의 키값을 비교함
- 왼쪽 데이터셋과 오른쪽 데이터셋의 결합 여부를 결정하는 조인 표현식 ( join Expression)의 평가 결과에 따라 두개의 데이터셋을 조인함

#### 2. 조인 타입
- 조인 표현식은 두 로우의 조인 여부를 결정
- But, 조인 타입은 결과 데이터셋에 어떤 데이터가 잇어야 하는지 결정

1. 내부 조인 ( Inner Join )
	- 왼쪽과 오른쪽 데이터셋에 키가 있는 로우 유지
2. 외부 조인 ( Outer Join )
	- 왼쪽이나, 오른쪽 데이터셋에 키가 있는 로우 유지 
3. 왼쪽 외부 조인 ( Left Outer Join )
	- 왼쪽 데이터셋에 키가 있는 로우 유지
4. 오른쪽 외부 조인 ( Right Outer Join )
	- 오른쪽 데이터셋에 키가 있는 로우 유지
5. 왼쪽 세미 조인 ( Left Semi Join )
	- 왼쪽 데이터셋의 키가 오른쪽 데이터셋에 있는 경우에는 키가 일치하는 왼쪽 데이터셋만 유지
6. 왼쪽 안티 조인 ( Left Anti Join )
	- 왼쪽 데이터셋의 키가 오른쪽 데이터셋에 없는 경우에는 키가 일치하지 않은 왼쪽 데이터셋만 유지
7. 자연 조인 ( Natural Join )
	- 두 데이터셋에서 동일한 이름을 가진 컬럼을 암시적으로 결합하는 조인
8. 교차 조인 ( Cross Join / Cartesian Join )
	- 왼쪽 데이터셋의 모든 로우와 오른쪽 데이터셋의 모든 로우를 조합 


```

// 임시 데이터 셋 
val person = Seq(
    (0, "Bill Chambers", 0, Seq(100)),
    (1, "Matei Zaharia", 1, Seq(500, 250, 100)),
    (2, "Michael Armbrust", 1, Seq(250, 100)))
  .toDF("id", "name", "graduate_program", "spark_status")
val graduateProgram = Seq(
    (0, "Masters", "School of Information", "UC Berkeley"),
    (2, "Masters", "EECS", "UC Berkeley"),
    (1, "Ph.D.", "EECS", "UC Berkeley"))
  .toDF("id", "degree", "department", "school")
val sparkStatus = Seq(
    (500, "Vice President"),
    (250, "PMC Member"),
    (100, "Contributor"))
  .toDF("id", "status")
```



#### 3. 내부 조인 
- Dataframe이나 테이블에 존재하는 키를 평가하고, True로 평가되는 로우만 결합

```
val joinExpression = person.col("graduate_program") === graduateProgram.col("id")

val innerjoinResult = person.join(graduateProgram, joinExpression)
innerJoinResult.show()

val joinType = "inner"

val innerjoinResult = person.join(graduateProgram, joinExpression, joinType)
innerJoinResult.show()

```

#### 4. 외부 조인 / 왼쪽 / 오른쪽
- Dataframe이나 테이블에 존재하는 키를 평가 ( true Or false )한 로우를 포함시킴
- 왼쪽 외부 조인일 경우, 왼쪽의 모든 로우와, 왼쪽 DataFrame과 오른쪽 DataFrame의 로우를 함께 포함
- 오른쪽 외부 조인의 경우, 오른쪽의 모든 로우와, 오른쪽 DataFrame과 왼쪽 DataFrame의 로우를 함께 포함
- 일치하는 로우가 없다면 null

```
val joinExpression = person.col("graduate_program") === graduateProgram.col("id")

val joinType = "outer" or "left_outer" OR "right_outer"

val outerJoinResult = person.join(graduateProgram, joinExpression, joinType)
innerJoinResult.show()

```

#### 5. 왼쪽 세미 조인
- 오른쪽 Dataframe의 어떤 값도 포함하지 않기 때문에, 다른 조인타입과는 약간 다름
- 단지 오른쪽 Dataframe은 값이 존재하는지 확인하귀 위해 값 비교 용도로 사용
- 기존 조인과 달리 Dataframe의 필터 정도로 볼 수 있음 

#### 6. 왼쪽 안티 조인 
- 왼쪽 세미 조인과 반대, 비교키가 없는 로우만 반환 
- SQL의 NOT IN 

---

#### 7. 조인 사용 시 문제점

##### 7.1. 복합 데이터 타입의 조인 
- Boolean을 변환하는 모든 표현식은 조인 표현식으로 간주
- 배열 값이 포함되는가?
```
person.withColumnRenamed("id","persionId")
	.join(sparkStatus, exprt("array_cotains(spark_status,id)"))
    .show()
```

##### 7.2. 중복 컬럼명 처리
- Dataframe의 각 컬럼은 스파크 SQL 엔진인 카탈리스트 내에 고유 ID가 있음
- 고유 ID는 카탈리스트 내부에서만 사용할 수 있으며, 직접 참조할 수 는 없음 

###### 문제상황
1. 조인에 사용할 DataFrame의 특정 키가 동일한 이름을 가지며, 키가 제거되지 않도록 조인 표현식에 명시하는 것
2. 조인 대상이 아닌 두 개의 컬럼이 동일한 이름을 가진 경우 


```
val gradProgramDupe = graduateProgram.withColumnRenamed("id", "graduate_program")

val joinExpr = gradProgramDupe.col("graduate_program") === person.col(
  "graduate_program")
  
person.join(gradProgramDupe, joinExpr).select("graduate_program").show()
// 오류 발생 - Reference 'graduate_program' is ambigous ... 

```

###### 해결방안 
1. 다른 조인 표현식 사용 
 - 불리언 형태의 조인 표현식 대신, 문자열이나 시권스 형태로 바꾸는 일 
 - 이렇게 하면 조인을 할 때, 두 컬럼 중 하나는 자동으로 제거 
2. 조인 후 컬럼 제거
3. 조인 전 컬럼명 변경

```
// 다른 조인 표현식 사용 예제 코드
val joinExpr = gradProgramDupe.col("graduate_program") === person.col(
  "graduate_program")
  
person.join(gradProgramDupe, joinExpr).show()
// 위의 joinExpr 대신 문자열로 넣자
person.join(gradProgramDupe, "graduate_program").select("graduate_program").show()
```

```
// 조인 후 컬럼 제거 예제 코드
person.join(gradProgramDupe, joinExpr).drop(person.col("graduate_program"))
.select("graudate_program").show()

```



#### 8. 스파크의 조인 수행 방식

- 스파크가 조인을 수행하는 방식을 이해하기 위해서 실행에 필용한 두가지 핵심 전략 

1. 노드간 네트워크 통신 전략
2. 노드별 연산 전략 

##### 8.1. 네트워크 통신 전략

- 스파크는 조인 시 두가지 클러스터 통신 방식을 활용
	1. 셔플 조인 ( Shuffle Join ) 
		- 전체 노드 간 통신을 유발 
	2. 브로드캐스트 조인 ( Broadcast Join )
		- 통신을 유발하지 않음 

##### 8.1 큰 테이블과 큰 테이블 조인  ( Shuffle Join )
- 셔플 조인은 전테 노드 간 통신이 발생
- 그리고 조인에 사용하는 특정 키나, 키 집합을 어떤 노드가 가졌는지에 따라 해당 노드와 데이터를 공유
- 네트워크 복잡해지고 많은 자원 사용 
- 즉, 전체 조인 프로세스가 진행되는 동안 모든 워커 노드에서 통신이 발생

##### 8.2 큰 데이블과 작은 테이블 조인 ( Broadcast Join )
- 테이블이 단일 워커 노드의 메모리 크기에 적합할 정도로 충분히 작다면, 조인 연산 최적화가능
- 브로드케이스 조인으로, 작은 Dataframe을 클러스터의 전체 워커노드에 복제
- 자원을 많이 사용하는 것 처럼 보일 수 있지만, 조인 프로세스 내내 전체 노드가 통신하는 현상을 방지 할 수 있음
- 조인 시작 시, 단 한번만 복제가 수행되고, 그 이후로는 개별 워커가 다른 워커를 기다리거나 통신할 필요없이 작업 수행 
- 브로드케이스 조인은 이전 조인과 마찬가지로 대규모 노드 간 통신 발생
- 하지만 그 이후로는 노드사이의 추가적인 통신 발생하지 않음
- 따라서 모든 단일 노드에서 개별적으로 조인이 수행되므로, CPU가 큰 병목구간이 됌
- DataFrame API 수행하면 옵티마이저에서 브로드케이스트 조인을 할 수 있게 Hint 줄 수 있음 

```
import org.apache.spark.sql.functions.broadcast

val joinExpr = person.col("graduate_program") === graduateProgram.col("id")
person.join(broadcast(gradudateProgram), joinExpr).show()
```

##### 정리

- 조인 전에 데이터를 적절하게 분할하면 셔플이 계획되어 있더라도 동일한 머신에 두 DataFrame이 있을 수 잇음
- 따라서 Shuffle을 피할 수 있음