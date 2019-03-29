---
layout: post
title:  "[Nifi] Nifi 권장 설정값 "
date: 2019-03-29 16:15:12
categories: Nifi 
author : Jaesang Lim
tag: Spark
cover: "/assets/instacode.png"
---

- Nifi로 엄청 삽질했다.. 프로세서 구현은 매우 간단하고 고민해야할 사항이 한정적이였는데
- 실제 클러스터 환경에서 돌려보니, 재시작만 해도 JVM Heap 사용량이 100% 올라가고.. 
- LogLevel도 TRACE로 설정해도 딱히 나오는 것도 없고.. 
> - 이 상황에서 유용한 JVM GC 모니터링하는 명령어를 배웠다 ㅎㅎ
> - jstat -gcutil ${PID} -t -h20 1s
> - -t는 타임값이 나오고, h20은 20줄마다 header를 찍어준다 

- 일단 그래서 모든 설정값을 default로 변경하고 hortonworks에서 권장하는 설정값을 차근차근 정리하고, 적용해서 해결할 것이다!

> - [Apache NiFi Configuration Best Practices](https://docs.hortonworks.com/HDPDocuments/HDF3/HDF-3.3.1/nifi-configuration-best-practices/hdf-nifi-configuration-best-practices.pdf)
> - [NiFi System Administrator’s Guide](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html)
> - [HDF/NIFI Best practices](https://community.hortonworks.com/articles/7882/hdfnifi-best-practices-for-setting-up-a-high-perfo.html)


### Nifi 설정값 

##### nifi.flow.configuration.file 
 > - 화면에 보이는 그래프 저장 경로
 > - default : ./conf/flow.xml.gz

##### nifi.flow.configuration.archive.dir
 > - 그래프 백업 저장 경로
 > - default : ./conf/archive
 > - 30일, 500메가로 기본 설정되고 바꿀 수 있음
 
##### nifi.bored.yield.duration 
  > - 프로세서가 할 일 있는 지 확인하는 주기, 
  > - default :10ms
  > - deafult보다 작으면 latency 줄어드나, cpu많이 씀. 더 길면 cpu 아껴서 사용할 수 있음

##### content Repository 경로와 flowfile Repository 경로는 다른 곳으로 권장

##### nifi.provenance.repository.max.storage.size 
  > - 주어진 기간이나 사이즈(rollover.time / size) provenance 정보 저장 파일 만드는 최대 크기, 
  > - default : 1G
  > - 프로덕션에서는 1~2T해도 무방이라고 적혀있는데 nifi.provenance.repository.index.shard.size를 고려했을 때 20-30G가 적당해 보임 
  
##### nifi.provenance.repository.rollover.size 
  > - 단일event file 사이즈
  > - default : 100MB
  > - 프로덕션에서는 1G정도가 적당
  
##### nifi.provenance.repository.index.threads
  > - Provenace Repository location 당 1-2개 적당, 
  > - 4개 넘지 않도록 할 것
  
##### nifi.provenance.repository.index.shard.size 
  > - nifi.provenance.repository.max.storage.size의 반보다 작게
  > - 프로덕션에서는 4-8G로 권장
  
##### nifi.provenance.repository.concurrent.merge.threads
  > - number of merge threads * the number of storage locations =< index.threads 
  > - 예를 들어 인덱스 스레드가 8개고 스토리지가 2개면 머지 스레드는 4개 이하로
  
##### nifi.queue.swap.threshold 
  > - 큐 스왑하는 임계치 ( 개별 큐에 해당 ) 
  > - 스왑이 발생하면, flowfile_repository/swap 폴더에서 확인할 수 있음
  > - default : 20000
  > - 큐가 자주 넘친다면 조금 키우는 것 고려. 이거 올리면 Heap도 같이 올려줄 것
  > - 왜냐면 swap되지 않은 큐에 있는 flowfile은 Heap에 상주하기 때문
  
##### nifi.provenance.repository.implementation=org.apache.nifi.provenance.WriteAheadProvenanceRepository 
  > - 이거 사용할 거면 bootstrap.conf에서 G1GC는 주석처리할 것
  > - 이거 관련해서 저번 Nifi 관련 블로그에 올렸었는데, G1GC에서의 문제점이 있다는 hortonworks 글을 보았음... ㅠㅠ
  > - G1GC 대신, CMS GC로 선택했음 
  > - java.arg.13=‐XX:+UseConcMarkSweepGC


### 해당 리눅스 서버 설정

#### /etc/security/limits.conf 
##### 최대 파일 핸들러
 > - 파일 많이 여니까 limit 상향
 > - * hard nofile 50000 
 > - * soft nofile 50000

##### 최대 프로세스 포크
 > - 쓰레드 많이 생성하니까 상향
 > - * hard nproc 10000 
 > - * soft nproc 10000

##### TCP 소켓 포트 수 상향
> - sudo sysctl -w net.ipv4.ip_local_port_range="10000 65000" 

##### TIME_WAIT 설정 (빨리빨리 teardown 시킴)
> -sudo sysctl -w net.ipv4.netfilter.ip_conntrack_tcp_timeout_time_wait="1" 

#### etc/sysctl.conf 
##### 스왑하지 않게 설정 
> - m.swappiness = 0

##### 저장소 파티션에 noatime 옵션 지정
> - /etc/fstab 에서 수정


#### 서버 사양
![nii](https://user-images.githubusercontent.com/12586821/55220899-a7a2cd00-524b-11e9-96c3-b7e533ee1321.PNG)

- 처리량 관점에서 볼 때 프로덕션에서는 최소 하드디스크 6개/각 2TB 에 5대 급으로 운영하는게 적당해 보임
- 100MB/s, 10000이벤트/s 정도~

