---
layout: post
title:  "Garbage Collection Monitoring"
date: 2019-04-02 14:55:12
categories: Etc
author : Jaesang Lim
tag: GC
cover: "/assets/instacode.png"
---

## Garbage Collection Monitoring


- Nifi 모니터링 하면서, TRACE 레벨의 로그도 남지 않고 계속 Heap 메모리가 증가하는 현상이 있어, GC 모니터링을 진행하였다
> - 메모리 누수의 문제를 의심하고 시작했지만, 사실상 KafkaConsumer 프로세서의 구현의 버그가 있는 듯 하다.
- 하이튼, 이번에 Nifi 프로세스의 GC 모니터링을 하면서 사용해본 모니터링 툴(?), 방법에 대해 정리하고자 한다. 


### jstat
- HotSpot JVM에 있는 모니터링 도구
- 다양한 옵션(gc, gccapacity)등이 있지만, gcutil이 가장 확인하기 쉬웠다.

1. gc
	> - 각 힙(heap) 영역의 현재 크기와 현재 사용량(Eden 영역, Survivor 영역, Old 영역등), 총 GC 수행 횟수, 누적 GC 소요 시간을 보여 준다.
2. gccapactiy
	> - 각 힙 영역의 최소 크기(ms), 최대 크기(mx), 현재 크기, 각 영역별 GC 수행 횟수를 알 수 있는 정보를 보여 준다.
	> - 단, 현재 사용량과 누적 GC 소요 시간은 알 수 없다.
3. gcutil
	> - 각 힙 영역에 대한 사용 정도를 백분율로 보여 준다. 아울러 총 GC 수행 횟수와 누적 GC 시간을 알 수 있다.

#### gcutil
jstat -gcutil -t -h20 ${PID} 1s
 
- '-t' 맨 앞 필드에 시간 출력
- '-h20' 20줄에 한번씩 header 찍음
- '1s' 1초마다 결과 출력

![gcutil](https://user-images.githubusercontent.com/12586821/55393255-e43a3580-5577-11e9-922f-6b5c60c324dd.png)

- YGC 
 	> - Young Generation의 GC 이벤트 발생 횟수
- YGCT 
  > - Yong Generation의 GC 수행 누적 시간	 ( 초 단위)
- FGC 
  > - Full GC 이벤트가 발생한 횟수	
- FGCT 
  > - Full GC 수행 누적 시간	 ( 초 단위)
- GCT 
  > - 전체 GC 수행 누적 시간 (초 단위)

- YGCT / YGC 하면 평균 Young GC 시간을 알 수 있음!
- 하지만 GC의 수행 시간의 편차가 심할 수 있어 평균보다는 개별 GC 시간을 파악하려면 자바 프로세스 시작할 때 -verbosegc 옵션을 넣어 확인하는 것이 더 유리

### jmap
자바 어플리케이션의 메모리 맵을 확인할 수 있음

- jmap  -heap ${PID}
> - JVM Heap 상태를 확인 할 수 있음
![jmap-heap](https://user-images.githubusercontent.com/12586821/55393214-cff63880-5577-11e9-96a4-aaf54a6086e3.png)

- jmap -histo:live ${PID}
> - 클래스별 객체 수와 메모리 사용량 확인

- jmap -dump:format=b,file=myDump.hprof ${PID}
> - heap dump 생성 후, visualVM으로 확인 가능

### jvmtop

- 실행중인 JVM의 Top 형태로 메모리 정보 및 CPU 프로파일링도 가능함
- https://github.com/patric-r/jvmtop
- JVM 메모리 정보
![jvmtop](https://user-images.githubusercontent.com/12586821/55393212-cf5da200-5577-11e9-94c1-4d1704e31852.png)
- JVM CPU 프로파일링
![jvmtop-profile](https://user-images.githubusercontent.com/12586821/55393213-cf5da200-5577-11e9-8f9c-eda804bbbe54.png)

### visualVM ( with jstatd )

- JVM을 모니터링하는 GUI툴
- Visual GC를 이용하면 jstatd 실행을 통해 GC 정보를 자세히, UI로 확인할 수 있음
- 자바 프로세서는 외부 서버에 실행중이고, visualVM으로 로컬에서 확인하려고 하면 원격서버에 다음과 같이 설정해야함 

- rmiregistry / jstatd 데몬 실행켜놓아야함
> 1. rmiregistry ${PORT} &
> 2. jstatd -p ${PORT} -J-Djava.security.policy=/home/test/tools.policy &
> - /home/test/tools.policy
  ```bash
  grant codebase "file:${java.home}/../lib/tools.jar" {
     permission java.security.AllPermission;
  };
  ```
  
- VisualVM에서 서버를 등록하고, 해당 서버의 jstatd Connection 포트를 위에서 설정한 ${PORT}로 변경

![jvm](https://user-images.githubusercontent.com/12586821/55393215-cff63880-5577-11e9-8c9e-fcacee8ce83c.png)


- 앞서 만든 heap dump ( myDump.hprof )을 시각화 할 수 있음
- 덤프 파일을 생성할 수 도 있고, Sampler를 통해, CPU/Memory 사용중인 Thread 정보도 알 수 있음!
- VisualGC 탭에서는 다음과 같이 Heap 메모리가 이쁘게 보임
![aa](https://user-images.githubusercontent.com/12586821/55393240-dd132780-5577-11e9-9a2d-b57b75400708.png)
