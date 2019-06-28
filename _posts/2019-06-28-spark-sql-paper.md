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
 - Shark에 대한 경험을 토대로 Spark 프로그래머가 관계형 처리, Relational Processing(e.g) 선언적 쿼리 및 최적화 된 스토리지) 이점을 활용할 수 있게 해줌
 - SQL 사용자가 Spark에서 복잡한 분석 라이브러리를 호출 할 수 있도fhr  ka

이전 시스템과 비교해, Spark SQL에 추가된 두가지 기능
1. DataFrame API를 통해 Procedural processing, relational processing을 통합해 사용할 수 있음  
2. Scala로 정의된 Optimizer, Catalyst를 쉽게 규칙을 추가하고 확장할 수 있음

## Introduction

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

근데.. relational과 precedural 시스템은 분리가 되어있어 개발자는 하나의 패러다임을 골라야함 
