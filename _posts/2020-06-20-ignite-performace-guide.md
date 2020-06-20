---
layout: post
title:  "[Ignite] Performace and Trobleshooting Guide"
date: 2020-06-20 19:05:12
categories: Ignite
author : Jaesang Lim
tag: Ignite
cover: "/assets/instacode.png"
---

# Ignite(GridGain) Performace and Trobleshooting Guide 
- [링크](https://www.gridgain.com/docs/latest/perf-troubleshooting-guide/general-perf-tips)
 
 
## Memory and JVM tunning

1. Tune Swappiness Setting
 - 전체 RAM 사용량이 특정 threshold값을 도달하면 OS는 RAM에 있던 Page를 디스크로 swap함
 - Swap은 클러스터 성능에 영향을 줄 수 있고, GC pause 시간도 늘어날수있음
 > - GC로그상에 'low user time, high system time, long GC pause'로그가 남는다면 java heap page를 swap하면서 발생할 수 도 있음 
 
 - vm.swappiness 변수를 10 또는 **persistence을 활성화할 경우 0**으로 설정할 것 
 - sysctl -w vm.swappiness=0
 
 
2. Share RAM with OS and Apps
 - 서버의 RAM은 OS, Gridgain, 다른 app과 공유함
 - Pure Memory Mode
  > - 전체 RAM의 90%를 넘지않게 설정
 - Persistence mode
  > - OS가 데이터를 disk에 sync하기 위한 page cache를 사용함 
  > - 그래서, page cache가 활성화되어있다면, 전체 RAM의 70%을 넘지않게 설정할 것 
  > - Native Persistence는 page cache 사용이 높아져서, kswapd 데몬은 백그라운드에서 page cache가 사용하는 page reclamation과 동일하지앟을 수 있음
  > - 그래서, 
결과적으로 직접 페이지 교정으로 인해 대기 시간이 길어지고 GC 일시 정지가 길어질 수 있습니다.
 add extra bytes between wmark_min and wmark_low with /proc/sys/vm/extra_free_kbytes:
 
 sysctl -w vm.extra_free_kbytes=1240000

Refer to this resource for more insight into the relationship between page cache settings, high latencies, and long GC pauses.
https://events.static.linuxfound.org/sites/events/files/lcjp13_moriya.pdf


3. Java Heap and GC Tuning

Gridgain은 데이터를 offheap에 가지고 있긴하지
java 힙은 여전히 ​​애플리케이션 워크로드에서 생성 된 오브젝트에 사용됩니다. 
예를 들어, GridGain 클러스터에 대해 SQL 쿼리를 실행할 때마다 쿼리는 offheap 메모리에 저장된 데이터 및 인덱스에 액세스하지만 
이러한 쿼리의 결과 세트는 애플리케이션이 결과 세트를 읽을 때까지 Java 힙에 유지됩니다.
 따라서 처리량 및 작업 유형에 따라 Java 힙을 여전히 많이 사용할 수 있으므로 워크로드에 대한 JVM 및 GC 관련 조정이 필요할 수 있습니다.
 
 
 For JDK 1.8+ deployments you should use G1 garbage collector. T
```
-server
-Xms10g
-Xmx10g
-XX:+AlwaysPreTouch
-XX:+UseG1GC
-XX:+ScavengeBeforeFullGC
-XX:+DisableExplicitGC

-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/heapdump
-XX:OnOutOfMemoryError=“kill -9 %p”
-XX:+ExitOnOutOfMemoryError

-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+PrintGCDateStamps
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=100M
-Xloggc:/path/to/gc/logs/log.txt

-XX:+PrintAdaptiveSizePolicy
   -XX : + PrintGCDetails 설정에 의도적으로 포함되지 않은 많은 추가 세부 사항을 제공합니다.
   
-XX:+UnlockCommercialFeatures
-XX:+FlightRecorder
-XX:+UnlockDiagnosticVMOptions
-XX:+DebugNonSafepoints
   - Performance Analysis With Flight Recorder

```

ignite Persistence 할 경우, 'MaxDirectMemorySize' = walSegmentSize * 4  (default 256)


4. Advance Memory Setting

Linux 및 Unix 환경에서 애플리케이션은 커널 특정 설정으로 인해 I / O 또는 메모리 부족으로 인해 긴 GC 일시 정지 또는 성능 저하에 직면 할 수 있습니다.
 이 섹션에서는 긴 GC 일시 중지를 극복하기 위해 커널 설정을 수정하는 방법에 대한 지침을 제공합니다.

If GC logs show low user time, high system time, long GC pause then most likely memory constraints are triggering swapping or scanning of a free memory space.

1. swap 확인할것
2.  -XX:+AlwaysPreTouch to JVM settings on startup
3. Disable NUMA zone-reclaim optimization. 
  sysctl -w vm.zone_reclaim_mode=0
 
4. Turn off Transparent Huge Pages if RedHat distribution is used.
  
  echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled
  echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag


5. Advance I/O Setting

If GC logs show low user time, low system time, long GC pause then GC threads might be spending too much time in the kernel space being blocked by various I/O activities. For instance, this can be caused by journal commits, gzip, or log roll over procedures.

 can try changing the page flushing interval from the default 30 seconds to 5 seconds:
 

sysctl -w vm.dirty_writeback_centisecs=500
sysctl -w vm.dirty_expire_centisecs=500


---

Persistence Tunning


1. Adjusting Page Size
DataStorageConfiguration.pageSize


SSD의 pagesize와 OS의 pagesize보다 작게 설정하면 안되고, default는 4KB

2. Keep WALs seperately

- data file과 WAL을 분리하자.
- Gridgain은 data, wal

또한 Point-in-Time-Recovery와 같은 다른 기능도 WAL 파일에 쓸 수 있으므로 더 많은 리소스가 필요합니다. 따라서 각각에 대해 별도의 물리 디스크 장치를 사용함으로써 전체 쓰기 처리량을 두 배로 늘릴 수 있습니다.

3. Increasing WAL Segment Size
(512K ~ 2G)
기본 WAL 세그먼트 크기 (64MB)는로드가 많은 시나리오에서 WAL이 세그먼트를 너무 자주 전환하고 전환 / 회전이 비용이 많이 드는 작업이기 때문에 비효율적 일 수 있습니다. 
세그먼트 크기를 더 큰 값 (최대 2GB)으로 설정하면 전환 작업 수를 줄이는 데 도움이 될 수 있습니다. 그러나 이렇게하면 미리 쓰기 로그의 전체 볼륨이 증가한다는 단점이 있습니다.

https://www.gridgain.com/docs/latest/developers-guide/persistence/native-persistence#changing-wal-segment-size

4. Changing WAL ahem 

기본 모드의 대안으로 다른 WAL 모드를 고려하십시오.
 각 모드는 노드 장애시 서로 다른 신뢰도를 제공하며 속도에 반비례합니다.
  즉, WAL 모드의 안정성이 높을수록 속도가 느립니다. 따라서 사용 사례에 높은 안정성이 필요하지 않은 경우 안정성이 낮은 모드로 전환 할 수 있습니다.
https://www.gridgain.com/docs/latest/developers-guide/persistence/native-persistence#wal-modes


5. Disabling WAL
https://www.gridgain.com/docs/latest/developers-guide/persistence/native-persistence#disabling-wal

6. Page Write Throttling

GridGain은 주기적으로 더티 페이지를 메모리에서 디스크로 동기화하는 검사 점 프로세스를 시작합니다. 더티 페이지는 RAM으로 업데이트되었지만 해당 파티션 파일에 기록되지 않은 페이지입니다 (업데이트는 WAL에 추가 된 것입니다). 이 프로세스는 응용 프로그램의 논리에 영향을주지 않고 백그라운드에서 발생합니다.

그러나 검사 점에 예약 된 더티 페이지가 디스크에 기록되기 전에 업데이트되면 이전 상태는 검사 점 버퍼라는 특수 영역에 복사됩니다. 버퍼가 오버플로되면 GridGain은 체크 포인트가 끝날 때까지 모든 업데이트 처리를 중지합니다. 결과적으로, 체크 포인트주기가 완료 될 때까지이 다이어그램에 표시된대로 쓰기 성능이 0으로 떨어질 수 있습니다.

검사 점이 진행되는 동안 더티 페이지 임계 값에 다시 도달하면 동일한 상황이 발생합니다. 그러면 GridGain은 하나 이상의 검사 점 실행을 예약하고 첫 번째 검사 점주기가 끝날 때까지 모든 업데이트 작업을 중단합니다.

두 가지 상황은 일반적으로 디스크 장치가 느리거나 업데이트 속도가 너무 심할 때 발생합니다. 이러한 성능 저하를 완화하고 방지하려면 페이지 쓰기 제한 알고리즘을 활성화하십시오. 이 알고리즘은 검사 점 버퍼가 너무 빨리 채워지거나 더티 페이지의 백분율이 빠르게 급등 할 때마다 업데이트 작업의 성능을 디스크 장치 속도로 낮 춥니 다.


  <bean class="org.apache.ignite.configuration.DataStorageConfiguration">

            <property name="writeThrottlingEnabled" value="true"/>

        </bean>
        
        
 7. Adjusting Checkpointing Buffer Size
 
  To keep write performance at the desired pace while the checkpointing is in progress, consider increasing DataRegionConfiguration.checkpointPageBufferSize and enabling write throttling to prevent performance​ drops:
  
8. Enabling Direct I/O


  
일반적으로 응용 프로그램이 디스크에서 데이터를 읽을 때마다 OS는 데이터를 가져 와서 파일 버퍼 캐시에 먼저 넣습니다. 마찬가지로 모든 쓰기 작업에 대해 OS는 먼저 캐시에 데이터를 쓰고 나중에 디스크로 전송합니다. 이 프로세스를 제거하기 위해 파일 버퍼 캐시를 무시하고 디스크에서 직접 데이터를 읽고 쓰는 직접 I / O를 활성화 할 수 있습니다.

GridGain의 Direct I / O 모듈은 체크 포인트 프로세스의 속도를 높이는 데 사용되어 더티 페이지를 RAM에서 디스크로 기록합니다. 쓰기 집약적 인 워크로드에 Direct I / O 플러그인 사용을 고려하십시오.



직접 I / O 및 WAL
WAL 파일에 대해서는 직접 I / O를 사용할 수 없습니다. 그러나 Direct I / O 모듈을 활성화하면 WAL 파일에 대해서도 약간의 이점이 있습니다. WAL 데이터는 OS 버퍼 캐시에 너무 오랫동안 저장되지 않습니다. 다음 페이지 캐시 스캔시 (WAL 모드에 따라) 플러시되고 페이지 캐시에서 제거됩니다.

IGNITE_DIRECT_IO_ENABLED=true




## Thread Pools Tunning


1. System Pool

시스템 풀은 SQL 및 쿼리 풀로 이동하는 다른 유형의 쿼리를 제외한 모든 캐시 관련 작업을 처리합니다. 또한이 풀은 컴퓨팅 작업의 취소 작업을 처리합니다.

기본 풀 크기는 max (8, 총 코어 수)입니다. 풀 크기를 변경하려면 프로그래밍 언어의 IgniteConfiguration.setSystemThreadPoolSize (…) 또는 유사한 API를 사용하십시오.


2. Query Pool

쿼리 풀은 클러스터에서 전송 및 실행되는 모든 SQL, Scan 및 SPI 쿼리를 처리합니다.

기본 풀 크기는 max (8, 총 코어 수)입니다. 풀 크기를 변경하려면 프로그래밍 언어에서 IgniteConfiguration.setQueryThreadPoolSize (…) 또는 유사한 API를 사용하십시오.

3. Public Pool

퍼블릭 풀은 컴퓨팅 그리드의 핵심입니다. 이 계산은 모든 계산을 받고 처리합니다.

기본 풀 크기는 max (8, 총 코어 수)입니다. 풀 크기를 변경하려면 프로그래밍 언어의 IgniteConfiguration.setPublicThreadPoolSize (…) 또는 유사한 API를 사용하십시오.

4. Service Pool

서비스 그리드 호출은 서비스의 스레드 풀로 이동합니다. 서비스 및 컴퓨팅 구성 요소에 대한 전용 풀을 사용하면 서비스 구현에서 계산을 호출하거나 그 반대의 경우 스레드 기아 및 교착 상태를 피할 수 있습니다.

기본 풀 크기는 max (8, 총 코어 수)입니다. 풀 크기를 변경하려면 프로그래밍 언어에서 IgniteConfiguration.setServiceThreadPoolSize (…) 또는 유사한 API를 사용하십시오.

5. Striped Pool


스트라이프 풀은 리소스에 대해 서로 경쟁하지 않는 여러 스트라이프에 작업 실행을 분산시켜 기본 캐시 작업과 트랜잭션을 가속화합니다.

기본 풀 크기는 max (8, 총 코어 수)입니다. 풀 크기를 변경하려면 프로그래밍 언어에서 IgniteConfiguration.setStripedPoolSize (…) 또는 유사한 API를 사용하십시오.

6. Data Streamer Pool
데이터 스 트리머 풀은 IgniteDataStreamer 및 IgniteDataStreamer를 내부적으로 사용하는 다양한 스트리밍 어댑터에서 오는 모든 메시지와 요청을 처리합니다.

기본 풀 크기는 max (8, 총 코어 수)입니다. 풀 크기를 변경하려면 프로그래밍 언어의 IgniteConfiguration.setDataStreamerThreadPoolSize (…) 또는 유사한 API를 사용하십시오.
