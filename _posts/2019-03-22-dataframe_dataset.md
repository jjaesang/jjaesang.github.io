---
layout: post
title:  "[HP-Spark] DataFrame/DataSet "
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

#### 2. JDBC
#### 3. Parquet
#### 4. Hive Table
#### 5. RDD
#### 6. Local Collections
#### 7. Etc.. 

  
### Dataset

### UDF/UDAF

### 질의 옵티마이저
