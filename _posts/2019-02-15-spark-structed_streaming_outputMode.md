---
layout: post
title:  "[Streaming] Spark structured Streaming : OutputMode"
date: 2019-02-15 18:15:12
categories: Spark-Structured-Streaming 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## OutputMode 편
---

### Append 모드 ( OutputMode.Append() )

- 새로운 로우가 결과 테이블에 추가되면 사용자가 명시한 트리거에 맞춰 싱크로 출력
- 신규 결과에 대해서만 Sink를 수행함
- Map, Filter, Join 와 같은 작업에서 사용됨

### Complete 모드 ( OutputMode.Complete() )

- 결과 테이블의 전테 상태를 싱크로 출력
- 모든 데이터가 계속해서 변경될 수 있는 일부 상태 기반 데이터를 다룰 떄 유용
- 기존 결과, 갱신된 결과, 신규 결과 모두 Sink를 수행함
- 전체 데이터를 다루는 작업에서 사용됨 (ex. 정렬)

### Update 모드 ( OutputMode.Update() )

- 이전 출력 결과에서 변경된 로우만 싱크로 출력 
- 쿼리에서 집계 연산을 하지않는다면 append와 모드와 같음
- 갱신된 결과, 신규 결과에 대해 Sink를 수행함
- Aggregation 와 같은 작업에서 사용됨
---

### WordCount 예제를 통한 Complete/Update 모드의 결과 차이 확인하기

- 소스 코드 ( 매우 간단 )
![aaaaa](https://user-images.githubusercontent.com/12586821/52842809-97270f00-3143-11e9-9bb7-ba36e565ec35.PNG)

- 인풋 ( 소켓을 통한 총 10개의 라인을 입력)
![update](https://user-images.githubusercontent.com/12586821/52842952-fc7b0000-3143-11e9-9e04-9c19c30e06bc.png)

- 최종 결과 비교 
![bd](https://user-images.githubusercontent.com/12586821/52845885-9003ff00-314b-11e9-80f4-2f385b335eda.PNG)

![ba](https://user-images.githubusercontent.com/12586821/52845884-9003ff00-314b-11e9-9a90-6e46b66429f4.PNG)

### Structed Streaming 제약 사항

- Aggregation
> - Multiple Aggregation이 불가능
> - 주로 Update Output Mode 사용

- Join
> - Join 하기 전 Aggregation이 불가능
> -  Join은 Append Output Mode만 가능함

- 그 외
> - 정렬은 Complete Output Mode 일때만 가능함
> - RDD에 Take() 같은 function은 Structured Streaming에서 지원하지 않음

- 즉, 배치에서 수행될 수 있는 모든 기능들이 Structured Streaming에서는 제약이 있음


### WaterMark

- 오래된 데이터를 제거하는 기능
- 스트리밍 처리는 신규 데이터 + 상태 정보(기존 결과)를 바탕으로 결과를 출력함
- 하지만, 상태 정보를 무한정 가지고 있을 필요가 없음



