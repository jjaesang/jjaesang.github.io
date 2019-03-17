---
layout: post
title:  "Shuffle에 대해 알아보기 "
date: 2019-03-17 18:44:02
categories: Spark
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

# Spark Shuffle의 중간결과는 local disk에 저장한다

- Shuffle은 디스크에 많은 수의 중간 파일을 생성
> - 정확히 어떤 데이터가 저장되는지 확인 필요

- Spark 1.3에서이 파일들은 해당 RDD가 더 이상 사용되지 않고 GC 발생할 때까지 보존
- 이는 리니지가 다시 계산 될 때 셔플 파일을 다시 만들 필요가 없도록하기 위해 수행
- GC는 응용 프로그램이 이러한 RDD에 대한 참조를 유지하거나 GC가 자주 시작하지 않는 경우 오랜 시간이 지난 후에 만 ​​발생할 수 있음
- 즉, 장기 실행 스파크 작업은 많은 양의 디스크 공간을 소비 할 수 있음
- 임시 저장소 디렉토리는 Spark 컨텍스트를 구성 할 때 spark.local.dir 구성 매개 변수에 의해 지정
- 참고 ) [Why does Spark save Map phase output to local disk?](https://stackoverflow.com/questions/35479876/why-does-spark-save-map-phase-output-to-local-disk)
- 참고 ) [공식문서](https://spark.apache.org/docs/2.1.2/programming-guide.html#shuffle-operations)

#### 위 내용 기반으로 내 스스로의 답을 만들자 ! 이번주에 꼭
