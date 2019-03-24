---
layout: post
title:  "[HP-Spark] DataFrame/DataSet와 Catalyst Optimizer "
date: 2019-03-22 18:15:12
categories: High-Performance-Spark 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

### 데이터 적재/저장 함수들

- 스파크 SQL의 데이터 적재/저장하는 방식이 스파크 Core와는 다름
- 특정 타입들의 연산을 저장 계층까지 내려서 수행할 수 있도록, 스파크 SQL은 자체적인 데이터 소스 API를 가지고 있음
> - https://databricks.com/blog/2015/01/09/spark-sql-data-sources-api-unified-data-access-for-the-spark-platform.html

- 데이터 소스는 해당 데이터 소스까지 내려가게되는 형태의 연산들을 지정하고 제어하는 것이 가능 ( filter pushDown )
- DataFrameWriter 
  > - ds/df.write() 
- DataFrameReader
  > - spark.read()


### 지원하는 포맷 format 
- JDBC,Parquet,Hive Table,RDD,Local Collections 등을 지원
 
#### 1. JSON
  - 스파크 SQL은 데이터를 샘플링하여 스키마 추측함
  - 스키마 정보를 판단하기 위해 데이터를 읽어야하므로, 다른 데이터 소스들에 비해 더 많은 비용이 듬
  > - .schema() 로 스키마 정보를 넣으면, 추론하는 작업은 스킵함
  - 레코드 사이의 스키마 차이가 크다면, 샘플 레코드 수가 너무 적으면 sampleRatio를 더 높게 설정하여 추측에 필요한 레코드 수를 늘릴 수 있음
  
  ```scala
   val df = spark.read.format("json").option("sampleRatio","1.0").load()
  ```
  
  - 스키마 추론 과정 ( org.apache.spark.sql.execution.datasources.json.InferSchema.scala )
    1. Infer the type of each record
    2. Merge types by choosing the lowest type necessary to cover equal keys
    3. Replace any remaining null fields with string, the top type
    
  - 입력 데이터에서 필터링 해야하는 경우 ( 잘못된, 불필요한 JSON 데이터)에는 Text로 읽고,필터 후 JSON을 다시 로드 할 수 있음
  - DataFrameReader의 기존 json 함수 실행 
  
  ```scala
    val rdd: RDD[String] = input.filter(_.contains("error"))
    val df = spark.read.json(rdd)
  ```

---


### 질의 옵티마이저

- 카탈리스트는 스파크 SQL의 질의 옵티마이저
> - 스파크 SQL의 핵심은 카탈리스트 옵티마이저이다.
> - 질의 계획을 받아 스파크가 수행할 수 있는 실행 계획으로 변환 
- DataFrame/Dataset에 관계형/함수형 트랜스포메이션을 적용할 때, 스파크 SQL은 논리 계획이라는 질의 계획을 표현하는 트리 그래프를 만듬
> - RDD에 대한 트랜스포메이션이 DAG를 만드는 것 처럼..
- 논리적 계획에 여러 최적화를 적용할 수 있으며, 비용 기반 모델을 써서 동일한 논리 계획에 대해 여러 물리적 계획을 세워 선택할 수 있음

![Catalyst-Optimizer-diagram](https://user-images.githubusercontent.com/12586821/54874875-39b76980-4e37-11e9-8fc3-c5d6ce8ec877.png)
![Spark-SQL-Optimization-2](https://user-images.githubusercontent.com/12586821/54877777-a72ebe80-4e66-11e9-993c-de5cd4d707d0.jpg)


- 카탈리스트에서는 4개 부분으로 나누어 Tree에 Transformation을 수행한다.
1. Analysis
  > - Unresolved Logical Plan -> (Analysis Rule + catalog_schema) -> Logical Plan
2. Logical Plan Optimization
  > - Logical Plan -> (Optimization Rule) -> Optimizer Logical Plan
3. Physical Planning
  > - Optimizer Logical Plan -> (Cost Model + Rule-based physical optimizations ) -> Physical Plan
4. Code Generation
  > - Physical Plan -> (Janino) -> RDDs 

#### 1. Analysis

- Spark SQL은 SQL Parser에서 반환한 Dataframe 객체의 Relation을 계산하는 것으로부터 시작
- Spark SQL은 Catalyst Rule과 Catalog object(Data source의 모든 Table을 Tracking하는 객체)을 이용하여 Attribute를 분석한다
> - Attribute란 ? Dataset/DataFrame의 컬럼 혹은 데이터 연산에 의해 새롭게 생성된 컬럼을 의미
> - select id, name from user_info 
> - id, name의 컬럼값의 타입과, 컬럼 이름이 맞는지 확인한다.

##### Unresolved Logical Plan -> (Analysis Rule + catalog_schema) -> Logical Plan으로 변경된다. 

#### 2. Logical Plan Optimizations 
> - [spark sql catalyst Optimizer](https://github.com/apache/spark/blob/5264164a67df498b73facae207eda12ee133be7d/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/expressions.scala)

- Logical Plan에 'Rule Based Optimization'을 적용
> 1. Constant Folding
  > - 상수, 리터럴로 표현된 표현식을 Complie Time에 계산하는 것 ( runtime 시 계산하지 않고 )
> 2. Predicate Pushdown
  > - subQuery 밖에 있는 where 절을 subQuery로 밀어넣는 것
  > - Join 후 filter가 아닌 , filter 후 Join하는 형태로 바꾸는 것으로 생각하면 될 듯 
> 3. Projection Pruning
  > - 연산에 필요한 컬럼만 가져옴
> 4. Null Propagation
> 5. Boolean Expression Simplification 
> etc .. 

##### Logical Plan -> (Optimization Rule) -> Optimizer Logical Plan으로 변경된다. 

#### 3. Physical Planning
- Optimized Logical Plan을 기반으로 1개 이상의 Spark Execution engine에서 수행할 수 있는 Physical Plan으로 바꿈 
- 이렇게 생성된 여러개의 Physical Plan을 Cost Model로 하나 선정한다. 
> - cost-based Optimization은 Join Algorithm을 선택

- Cost Model 이외에도, Rule-based physical optimizations도 수행
> - projection , filter , 저장소에 대한 predicate or projection pushdown도 수행 

##### Optimizer Logical Plan -> (cost Model + rule-based physical optimizations ) -> Physical Plan으로 변경

#### 4. Code Generation
- 최적화된 Physical Plan을 Java Bytecode로 변환
- Java 코드를 컴파일 하기 위해 자니노(Janino)를 사용
> - 초기 버전은 스칼라의 콰지쿼트(Quasiquote) 썼으나, 작은 데이터세의 코드 생성에도 오버헤드가 너무 컸음..
- Janino
> - Janino is a super-small, super-fast Java compiler.
> - Janino can not only compile a set of source files to a set of class files like JAVAC, 
> - but also compile a Java expression, block, class body or source file in memory, load the bytecode and execute it directly in the same JVM.
> - JANINO is integrated with Apache Commons JCI ("Java Compiler Interface") and JBoss Rules / Drools

##### Physical Plan -> (Janino) -> RDDs 으로 변경






















