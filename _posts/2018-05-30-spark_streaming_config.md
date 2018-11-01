---
layout: post
title:  "Spark Streaming Configuration"
date: 2018-05-30 13:05:12
categories: Spark Streaming
author : Jaesang Lim
tag: Spark Streaming
cover: "/assets/spark_log.png"
---
### Spark Streaming Configuration

- Streaming Job은 Yarn Cluster에서 실행이 되면, 일부로 중지시키지 않는 한, 영원히 실행
- 중단이 된다는 것은 데이터 처리에 대한 Delay , Loss , Duplicate 될 가능성이 크다 
> - 개인적인 생각이지만 Kafka와 같이 사용한다면 중단 시, Delay는 있을 수 있지만, Loss와 Duplicate는 피할 수 있을 것 같다. 

- 사실상, Yarn 과 Spark 모두 long-running 서비스를 위해 디자인 된 것이 아님

#### 1. Fault Tolerance를 위한 Configuration
> - 부분 실패가 있더라도, 계속 작업할 수 있게 !! 
> 
1. spark.yarn.maxAppAttempts=4
2. spark.yarn.am.attemptFailuresValidityInterval=1h
3. spark.yarn.max.executor.failures=( 8 * ${NUM_EXECUTORS} ) 
4. spark.yarn.executor.failuresValidityInterval=1h 
5. spark.task.maxFailures=8


##### 1_1. spark.yarn.maxAppAttempts=4 ( Driver 재시도 )
> - Spark Driver 와 AM은 단일 JVM을 공유하기 때문에, Spark Driver 오류로 잡이 종료
> - Yarn 설정을 통해 , Driver가 죽어도 다시 실행하도록 하는 최대 시도 횟수를 증가 
> - default : 2 
> - 블로그 저자에 따르면 4가 적당! 
> - 그 이상 설정할 시, 일시적인 오류가 아닌 것들에 대해서 불필요하게 재시작을 할 수 있음


##### 1_2. spark.yarn.am.attemptFailuresValidityInterval=1h
> - 위에서 설정한 4번의 최대 횟수를 측정하는 기간이 , 일주일이라면 크게 의미가 없음
> - maxAppAttempts 의 주기를 reset하는 것이 필요 
> - 1시간이 지나면 최대 시도 횟수는 초기화


##### 1_3. spark.yarn.max.executor.failures=( 8 * ${NUM_EXECUTORS} )  ( Executor 재시도 )
> - 응용프로그램이 최종 실패를 리턴하기 전에 App의 실패 최대 개수를 산정
> - default : MAX( 2 * ${NUM_EXECUTORS}, 3 )

##### 1_4. spark.yarn.executor.failuresValidityInterval=1h 
##### 1_5. spark.task.maxFailures=8
> - 장기 실행 작업의 경우 작업을 실패하기 전에 어플리케이션의 최대 실패 횟수를 늘림
> - default : 4

#### 2. Performance를 위한 Configuration

##### 2_1. spark.speculation=true

- Streaming 작업은 처리 시간을 안정적이고 예측 가능하게 유지
- Stable 하게 유지하는 것이 중요 ( Processing Time < Batch Interval )
- 많은 Job을 처리하는 Cluster 환경에서는 Speculative execution이 Batch processing에 도움
- ** 개인적으로, speculation=true로 했을 때 큰 장점이 없어서 False로 처리했음 **


#### 3. Graceful Stop 

- 지금 처리하려고 들고 있는 데이터 까지 다 처리하고 작업을 종료하자
- 경험 상, GracefulStop가 호출이 된 후부터는 Kafka로 부터 데이터를 더 이상 받지않음
- Spark Streaming Queue에 쌓여있는 Job을 모두 처리하면 그 때 종료!


##### 3_1. JVM Shutdown Hook
 - sys.ShutdownHookThread { ssc.stop(true,true) } 
 - 1.4 버전 이상 버전에서 작동 안함 ( deadlock 발생 가능성 있음 )

##### 3_2. spark.streaming.stopGracefullyOnShutdown = true
 - default : false 
 - 코드 내에서 ssc.stop() 할 필요없음!
 - 대신에 Driver에게 SIGTERM signal를 줘야함 
 - Spark UI에서 Driver가 실행중인 서버를 찾아 AM의 PID로 SIGTERM(15)
 > - 나는 기본적으로 kill -9 PID 이렇게 사용했음 .. 15는 정상 종료 / 9는 강제 종료의 다른 SIG 였다..

```

17/02/02 01:31:35 ERROR yarn.ApplicationMaster: RECEIVED SIGNAL 15: SIGTERM
17/02/02 01:31:35 INFO streaming.StreamingContext: Invoking stop(stopGracefully=true) from shutdown hook
...
17/02/02 01:31:45 INFO streaming.StreamingContext: StreamingContext stopped successfully
17/02/02 01:31:45 INFO spark.SparkContext: Invoking stop() from shutdown hook
...
17/02/02 01:31:45 INFO spark.SparkContext: Successfully stopped SparkContext
...
17/02/02 01:31:45 INFO util.ShutdownHookManager: Shutdown hook called
```

- 그런데 Yarn 모드에선 spark.yarn.maxAppAttempts ( = yarn.resourcemanager.am.max-attempts =2 ) 여서 
- 2번 SIGTERM를 줘야함
- maxAppAttempts를 1로 줄일 수 는 있는데.. 그렇게 Graceful Shutdown이 Driver의 재시작보다 중요한지 고민해야하암
- 그러면.. 또 어느 노드에서 Driver가 떴는지 확인해서 죽여함.. 


-** yarn application -kill <applicationId> 추천하지 않음**
- SIGTERM을 Container에게 전달하지만, 즉각적으로 SIGKILL이 먼저도착한다 .
- SIGTERM을 보내고 일정 시간 후에 SIGKILL를 보냄 
- SIGETEM과 SIGKILL의 시간 차이는 yarn.nodemanager.sleep-delay-before-sigkill.ms ( default 250ms )
- 이 설정값을 바꿀 수는 있지만 1분으로 늘려도 작동하지 않음
- 즉각적으로 죽는 것을 볼 수 있음.. 정상 로그도 출력되지 않음
```
17/02/02 12:12:27 ERROR yarn.ApplicationMaster: RECEIVED SIGNAL 15: SIGTERM
17/02/02 12:12:27 INFO streaming.StreamingContext: Invoking stop(stopGracefully=true) from shutdown hook
```


##### 3_3. HDFS Marker file ( 내가 사용한 방법 ! )
 - Streaming App 에서 HDFS 에 저장된 Marker File를 주기적으로 체크하는 것
 - File이 존재하면 scc.stop(true, true )
 - 위의 stop를 Batch 처리하는 코드 내부에서 처리하면 안댐 ( deadlock  )
 - scc.stop하는 다른 Thread에서 처리해야함
 - ssc.start 다음에 호출
 - 시작할 때 FILE 제거 및 종료할 때 Touchz

```
	// Streaming Job 실행
    sparkStreamContext.start()

    val checkIntervalMillis = batch_interval.toInt * 1000 * 10 * 2
    var isStopped = false

    while (!isStopped) {
      isStopped = sparkStreamContext.awaitTerminationOrTimeout(checkIntervalMillis)
      if (isStopped)
        logger.warn("Process Stop Check - Confirmed! The streaming context is stopped. Exiting application.")
      else {
        logger.warn("Process Stop Check - Streaming App is still running")
        }
      }
      
      // Marker File이 있는지 없는지 조회하는 함수 호출 
      checkShutdownMarker()
      if (!isStopped && stopFlag) {
        logger.warn("Graceful stopping sparkStreamContext right now")
        sparkStreamContext.stop(true, true)
        logger.warn("Graceful sparkStreamContext is stopped!")
      }
    }
```

참고 
```
  /**
   * Stop the execution of the streams, with option of ensuring all received data
   * has been processed.
   *
   * @param stopSparkContext if true, stops the associated SparkContext. The underlying SparkContext
   *                         will be stopped regardless of whether this StreamingContext has been
   *                         started.
   * @param stopGracefully if true, stops gracefully by waiting for the processing of all
   *                       received data to be completed
   */
   
   def stop(stopSparkContext: Boolean, stopGracefully: Boolean): Unit = {
   
   }
```

#### 4. YARN Configuration

- spark.yarn.driver.memoryOverhead=512  ( driver-memory < 5GB )
- spark.yarn.executor.memoryOverhead=1024 ( executor-memory < 10G ) 
- default : min( 384, executorMemory * 0.10 ) 
- 10G / 5G 보다 작을 시 , 늘리자

#### 5. Spark Delay Scheduling

- spark.locality.wait = 10ms

-  Driver Wait Strategy ( default : 3s )
> - Process-local: 3s to launch the task in the receiver executor
> - Node-local: 3s to launch the task in an executor on the receiver host
> - Rack-Local: 3s to launch the task in an executor on a host in the receiver rack

- - 10ms로 줄이면 처리 기반 3배 단축 
- Kafka Streaming일 때는  큰 차이를 못 느낄 수 있음 


#### 6. BackPressure

[concept 관련 링크](http://www.reactive-streams.org/ )
- queue에 계속 쌓이면 Delay + OOM 가능 
- backPressure를 활성화하여 , Drvier가 현재 Scheduling Delay 와 Processing Delay를 모니터링하여
  Receiver의 maximum rate를 조절함 
```
2016-12-06 08:27:02,572 INFO org.apache.spark.streaming.receiver.ReceiverSupervisorImpl Received a newrate limit: 51.
```

- 작업이 안들어오거나 하는 등의 문제를 정확하게 파악하지 못할 수 있음
- 2가지 방법은 BackPressure를 조절할 수 있음

1. Minimum Rate
- 왜 ? 100 record 이하로 떨어지지 않을까?
- 내부적으로 Spark는 PID 기반 Back pressure를 실행
- spark.streaming.backpressure.pid.minRate : default 100 per second ( PID RateEstimator 구현체에 있음 )
- 필요하면 줄여라 


2. Initital Rate 
- 이전 batch의 Processing Time 기반, backPressure Rate 계산
- 즉, 처음 시작했을시 , BackPressure은 바로 실행되지 않음 
- 처음 배치는 큰 사이즈로 들어올 수 있기 떄문에 이 것을 Smooth 하게 처리하는 Config 있음
	- spark.streaming.backpressure.initalRate 조정 
	- spark.streaming.kafka.maxRatePerPartition 조정 ( Kafka ) 





