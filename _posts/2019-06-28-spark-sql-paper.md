---
layout: post
title:  "[Spark-Paper] Spark SQL: Relational Data Processing in Spark"
date: 2019-06-28 17:05:12
categories: Spark-Paper 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## Spark SQL: Relational Data Processing in Spark

Spark SQL이란 ?
 - Shark에 대한 경험을 토대로 Spark 프로그래머가 관계형 처리, Relational Processing(e.g. 선언적 쿼리 및 최적화 된 스토리지) 이점을 활용할 수 있게 해줌
 - SQL 사용자가 Spark에서 복잡한 분석 라이브러리를 호출 할 수 있도fhr  ka

이전 시스템과 비교해, Spark SQL에 추가된 두가지 기능
1. DataFrame API를 통해 Procedural processing, relational processing을 통합해 사용할 수 있음  
2. Scala로 정의된 Optimizer, Catalyst를 쉽게 규칙을 추가하고 확장할 수 있음

---

## 1. Introduction

- 데이터 어플리케이션은 다양한 프로세싱 기술과, Source, Storage format 등이 필요함
- 이를 위해 개발된 MapReduce 같은 프레임워크는 low-level로 procedural programming interface를 제공하였음
- 하지만 다음과 같이 고려해야할 사항이 존재함
> - 개발하는 것도 번거롭고, 고성능을 위해서는 개발자가 직접 수동으로 최적화해야함

- 그래서 새로운 많은 시스템에는 큰 데이터에 대한 Relational interface를 제공하였음
> - Pig, Hive, Dremel, Shark 모두 declartive query를 활용하여 자동 최적화 작업을 추가함

그런데..
- relational system의 인기도 높아지고 활용도도 높지만, 데이터 어플리케이션에서는 좀 부족한게 있음
1. 데이터는 대부분 반,비정형 데이터여서 이에 맞는 코드를 짜야함
2. ML이나 Graph Processing 같은 분석은 relational system에서 작업하기가 어려움
> - 논문에 따르면 대부분의 파이프라인은 relational query과 procedural 알고리즘을 이루어져 있다고 함 

근데.. relational과 procedural 시스템은 분리가 되어있어 개발자는 하나의 패러다임을 골라야함 
> - 그래서 이 논문에서는 relational, procedural을 함께사용할 수 있는 **Spark SQL**에 대해 설명하고자 함

그렇다면 어떻게 두 모델(relational, procedural)의 gap을 어떻게 극복했는가?

1. DataFrame API
> - R의 Dataframe과 비슷하지만, lazy evaluation으로 relational 연산이 가능
> - procedural, relation api를 모두 사용가능하며, optimizer가 최적화해줌
2. Catalyst
> - 다양한 data source와 algorithm을 지원하는 optimizer
> - 사용자가 쉽게, optimization rule, data type 등을 추가할 수 있음
> - Scala의 pattern-matching으로 optimization rule을 계산함 

---

## 2. Background and Goals 

### Shark : Previous Relational Systems on Spark

Shark 
> - Hive 시스템으로 변경해서 Spark에서 돌 수 있게 하고, RDBMS의 Opmization을 사용함
> - Columnar processing 같은 Optimization

물론, Shark도 좋은 성능을 내고 Spark에서도 잘 동작했으나 중요한 3가지 해결과제가 있었음
1. Hive catalog안에 저장된 query external data 만 활용할 수 있음
> - 그래서, spark에서 만든 RDD같이 Spark Program안에 있는 데이터를 처리하는데 유용하지 않음

2. spark에서 shark을 호출하려면, SQL String만 이용해야함
> - 불편함

3. Hive Optimizer은 Mapreduce 맞춤형임
> - 확장이 어렵고, ML 또는 새로운 Data source에 대한 data type을 확장하기 어려움 

### Goals for Spark SQL 

위와 같은 Shark의 해결과제를 해결하고 Spark SQL이 나왔고 다음과 같은 목표가 있음
1. Spark의 Native RDD와 외부 data source에 대한 relational processing을 제공함
2. DBMS 기반 high performance를 제공함
3. semi-structed, external data source 등 새로운 data source을 쉽게 지원하고자 함
4. ML, graph 같은 분석 알고리즘에도 확장 가능하게 설계함

---

## 3. Programming Interface

<img width="672" alt="image" src="https://user-images.githubusercontent.com/12586821/60395948-0727b600-9b76-11e9-83f8-1a084fc6994c.png">

### DataFrame API
- distributed collections of rows with same schema
- RDB의 데이터블과 같은 개념이고, RDD와 같이 조작가능함 
> - 그러나, RDD와 다르게 Dataframe의 스키마 정보를 가지고 있고, 다양한 relational operation이 가능하고, 이에 따라 최적화가 진행됌

Dataframe Operations

<img width="631" alt="image" src="https://user-images.githubusercontent.com/12586821/60396062-abf6c300-9b77-11e9-98f6-f438b1778cc4.png">

- employees : DataFrame
- employees("depId") : expression 

expression object
> - 다양한 operator가 있을 수 있음
> - 모든 operator은 AST(Abstract Syntax Tree)로 만들어지고, AST는 catalyst에게 최적화를 위해 넘겨짐

### Dataframes VS Relational Query Language

dataframe은 relational query (SQL, pig)와 같은 operation을 제공

한 사용자는 DataFrame API가 "중간 결과를 명명 할 수있는 것을 제외하고는 SQL과 같이 간결하고 선언적"이라고 말함
> - “concise and declarative like SQL, except I can name intermediate results”
> - 연산을 구조화하고 중간 단계를 디버그하는 것이 더 쉬운 방법을 언급함

DataFrames에서의 프로그래밍을 단순화하기 위해 API는 logical result을 열심히 계산함
> - 즉, 표현식에 사용 된 열 이름이 기본 테이블에 있는지 여부나 해당 데이터 유형이 적절한 지 여부를 식별함

따라서 Spark SQL은 사용자가 실행될 때까지 기다리지 않고 유효하지 않은 코드 행을 입력하자마자 오류를 냄
> - 긴 SQL 문보다 작업하기가 쉽게 함

### Querying Native Datasets

real-world pipeline
> - 다양한 이종의 데이터 소스에서 데이터를 추출하고 다른 프로그래밍 알고리즘을 이용해 연산함 

Spark SQL은 기존의 procedural Spark Code와 상호운요ㅇ하기 위해, RDD을 Dataframe으로 변환할 수 이음
> - Spark SQL은 reflection을 이용해, object의 스키마를 추론하고, JavaBeans, Scala Case class로 매핑할 수 있음 

내부적으로 Spark SQL은 RDD를 가리키는 논리적 데이터 스캔 연산자를 생성함 (logical data scan operator)
- 이는 native object의 필드에 액세스하는 physical 연산자로 컴파일됌


이것은  객체 관계형 매핑 (ORM)과 매우 다름
> - ORM은 종종 전체 개체를 다른 형식으로 변환하는 값 비싼 변환임
> - 하지만 Spark SQL은 각 쿼리에 사용 된 필드만 추출하여 기본 Object에 접근함

네이티브 데이터 세트를 쿼리하는 기능을 사용하면 기존 Spark 프로그램 내에서 최적화 된 관계형 작업을 실행할 수 있음

RDD와 외부 구조화 된 데이터를 간단하게 결합 할 수 있음
> - RDD와 Hive의 테이블을 결합


### In-Memory Caching

Spark SQL은 Shark와 마찬가지로 Columnar storage를 사용하여 메모리에서 데이터를 캐시할 수 있음 

데이터를 JVM 객체로 저장하는 Spark의 기본 캐시와 약간 다름
- Columnar storage 캐시는 dictionary encoding 및 run-length encoding 같은 Columnar 압축 스키마를 적용함
> - 이는 메모리 풋 프린트를 한 단계 줄임

---


## 4. Catalyst Optimizer

Spark SQL을 위한 확장가능한 optimizer인 Catalyst를 scala로 구현함
1. 쉽게 최적화 기능을 추가할 수 있음
2. 외부의 데이터 저장소로 부터의 기능도 확장할 수 있음
> - push filtering 또는 aggregation을 외부 저장소에 적용할 수 있음

과거에는 Optimizer을 확장하기 위해서는 rule을 만들기 위한 도메인 지식과, 해당 코드를 translate할 optimizer complier에 대한 지식이 필요
> - 유지보수와 learning curve가 있음

하지만 Catalyst는 scala의 Pattern-matching을 통해 쉽게 구현할 수 있음

총 4단계를 통해 Catalyst는 최적화하며, operator를 tree로 구현하고, rule을 통해 tree를 조작하는 형태
1. analysis
2. logical optimization
3. physical planning
4. code generation (= query를 complie해서 Java bytecode로 변)

### Trees

Catalyst의 data type는 node object로 구성된 tree
각 노드는 
 > - 각 노드의 타입을 가지고, 자식을 가질 수 도, 안가질 수 도 있음 
 > -  Scala의 TreeNode Class의 subClass로 구현되어있음
 
<img width="561" alt="image" src="https://user-images.githubusercontent.com/12586821/60751400-dbca1e80-9fee-11e9-92b7-ce286e335907.png">
<img width="705" alt="image" src="https://user-images.githubusercontent.com/12586821/60751434-4da26800-9fef-11e9-8f06-a1dd65432eb8.png">

### Rules

Scala Object인 Tree는 Rule에 의해 조작되어새로운 Tree로 변환됌
> - Rule이 적용되는 방법은 일반적으로 Pattern matching으로 진행되며, 트리를 변환함 (transform)

Catalyst에서 tree는 transform 함수를 통해 tree의 모든 노드를 재귀로 순회하면서 pattern matching을 진행함 

transform 함수는 partial function으로, input에 대한 가능한 연산을 모두 처리함 
> - 그래서, Rule은 같은 transform을 호출하더라도, 다양한 pattern을 매칭시킬 수 있고, 여러 transformation을 간결하게 구현할 수 있음

<img width="625" alt="image" src="https://user-images.githubusercontent.com/12586821/60751467-1bddd100-9ff0-11e9-858c-de859301f36a.png">


rule은 tree을 모두 transform하는 동안 여러번 실행되고, Catalyst는 rule을 모아 batch로 처리함
> - 각 batch는 더 이상 변형할 것이 없을 때까지 연산함(fixed point 라고함)

e.g. 첫번째 Batch는 모든 expression을 분석해, 각각 타입을 할당함
다음 batch는 각 타입별로 constant folding을 연산함 
그 이후, 새로운 tree에 대한 sanity check을 실행함 ( 모든 attribute가 data type이 할당되었는가? )

rule의 조건은 모두 scala code이며, immutable한 tree에 functional transformation은 debug + 병렬적으로 실행이 가능함

### Using Catalysts in Spark SQL

총 4단계로 진행
<img width="1328" alt="image" src="https://user-images.githubusercontent.com/12586821/60751533-aecb3b00-9ff1-11e9-94a9-7c8624db1262.png">

1. Analyzing a Logical Plan
2. Logical Plan Optimization
3. Physical Planning
4. Code Generation, complie query to Java Bytecode

3번째, Physical Planning 단계에서는 여러개의 plan을 만들고 cost 기반으로 비교해 선택함

3번쨰를 제외하고는 모두 rule-based로 연산함

#### 1. Analysis

spark SQL은 Catalyst Rule과 Catalog object을 사용해 attribute을 해석함 (약 1000줄 정도 )
즉, unresolved logical plan, SQL Parser로 부터 반환된 AST트리 또는 API를 통해 구성된 DataFrame에 대해 attribute을 가져오거나, 각 data type을 추론함

e.g. SELECT col FROM sales
<img width="662" alt="image" src="https://user-images.githubusercontent.com/12586821/60751611-29488a80-9ff3-11e9-95b1-3a23e88153e6.png">

#### 2. Logical Optimization
Rule-based optmization으로 동작함  ( 약 800줄 정도)
> - constant folding
> - predicate pushdown
> - projection prunning
> - null propagation
> - boolean expression simplification
> - etc..

<img width="1328" alt="image" src="https://user-images.githubusercontent.com/12586821/60751539-c4406500-9ff1-11e9-889e-c28f8b239178.png">

#### 3. Physical Planning
Logical Plan기반으로, Spark Execution Engine을 통해 한개 이상의 Physical Plan을 만듬 ( 약 500줄 정도 )
> - cost기준으로 하나 선택함, Only! Join 알고리즘을 선택함
> - relation이 작으면, Spark SQL은 peer-to-peer broadcast로 구현된 Broadcast Join을 선택함
> > - 테이블이 cache되어 있거나, external file로 부터 읽을 때나, limit같은 subquery가 있을 떄 테이블의 사이즈를 측정함

Rule based optimization을 진행함
> - pipelining projection 또는 filter 
> - logcial Plan에 predicate or projection pushdown이 가능하다면 강제할 수 있음 

#### 4. Code Generation
각 머신에서 실행될 수 있게 Java bytecode로 변환
> - Spark SQL 작업은 인메모리의 데이터셋의 CPU-Bound 작업이 종종 있어, 빠르게 Code Generation을 하고자 했음
> - 그래서 Scala의 특수 기능이라고 하는 Quasiquotes을 사용함

Quasiquotes는 스칼라 (Scala) 언어에서 AST (Abstract Syntax Tree)를 프로그래밍 방식으로 생성 할 수 있음
- 런타임시 스칼라 컴파일러에 전달되어 바이트 코드를 생성함

Catalyst를 사용하여 SQL의 표현식을 나타내는 트리를 스칼라 코드의 AST로 변환하여 해당 표현식을 평가 한 다음 생성 된 코드를 컴파일하고 실행함
<img width="1328" alt="image" src="https://user-images.githubusercontent.com/12586821/60751543-d3bfae00-9ff1-11e9-8677-35fa6393c205.png">

Quasiquotes는 컴파일시, type check을 진행하고 적절한 AST 또는 Literal만 대체되어 문자열 연결보다 훨씬 유용하게 사용함
> - 런타임에 Scala 파서를 실행하는 대신 직접 스칼라 AST로 변환
> - Catalyst가 누락 한 표현식 수준 최적화가있는 경우 결과 코드가 Scala 컴파일러에 의해 추가로 최적화 진행
Quasiquotes는 손으로 튜닝 한 프로그램과 비슷한 성능의 코드를 생성 할 수 있음!

## 5. Advanced Analytic Features

## 6. Evaluation
