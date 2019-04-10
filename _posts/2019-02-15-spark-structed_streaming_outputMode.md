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

### Complete 모드 ( OutputMode.Complete() )

- 결과 테이블의 전테 상태를 싱크로 출력
- 모든 데이터가 계속해서 변경될 수 있는 일부 상태 기반 데이터를 다룰 떄 유용

### Update 모드 ( OutputMode.Update() )

- 이전 출력 결과에서 변경된 로우만 싱크로 출력 
- 쿼리에서 집계 연산을 하지않는다면 append와 모드와 같음

---

### WordCount 예제를 통한 Complete/Update 모드의 결과 차이 확인하기

- 소스 코드 ( 매우 간단 )
![aaaaa](https://user-images.githubusercontent.com/12586821/52842809-97270f00-3143-11e9-9bb7-ba36e565ec35.PNG)

- 인풋 ( 소켓을 통한 총 10개의 라인을 입력)
![update](https://user-images.githubusercontent.com/12586821/52842952-fc7b0000-3143-11e9-9e04-9c19c30e06bc.png)

- 최종 결과 비교 
![bd](https://user-images.githubusercontent.com/12586821/52845885-9003ff00-314b-11e9-80f4-2f385b335eda.PNG)

![ba](https://user-images.githubusercontent.com/12586821/52845884-9003ff00-314b-11e9-9a90-6e46b66429f4.PNG)
