---
layout: post
title:  "[Nifi] Throughtput 향상을 위한 작업들 "
date: 2019-02-27 16:15:12
categories: Nifi 
author : Jaesang Lim
tag: Spark
cover: "/assets/instacode.png"
---

> Nifi에서 초당 처리량을 보장하면서, 시스템 리소스(CPU/Memory)를 효율적으로 사용하고자 진행했던 내용들 정리

### 요구사항
- 초당 약 5000건 이상의 로그데이터를 안정성이 있게 처리
- 안정성 : 초당 5000건 이상 처리하면서도, 시스템 리소스 효율적 사용 

### 문제 상황 
- Kafka에서 로그 Consume하고, Validation 과 Parsing 하는 과정에서 Queue와 lag가 지속적으로 증가함 
 > - 처리량 측면에서 실패... 
- 유입되는 데이터양에 비해, 처리량이 늦어지면서 시스템 리소스( CPU / Memory )의 비효율적 사용

### 해결방안
- Nifi는 사이즈가 작은 다수의 flowfile보다 사이즈가 큰 적은 flowfile를 처리하는데 더 효율적
- swap 설정값 조정 - nifi.queue.swap.threshold
- Nifi Provenance Repository 구현체를 WriteAheadProvenanceRepsitory로 설정

### 결과
- JVM Heap Memory 
  - 80% → 48% 
- CPU 사용량 
  - 70% →  20% 
- Kafka Lag  
  - 무한히 증가 →  0
- 전반적인 시스템 사용률 및 처리량 증가

### 해결방안 - PROCESSOR 측면 

#### 1. Processor의  Concurrent Task 를 늘림 ( default : 1 )  - 프로세서 마다 최대 3개까지 설정

- 프로세서 안에서 Concurrent Task를 아무리 늘려도 최대 10개의 Threads만 할당
- Controller Settings에서 ‘Maximum Timer Driven Thread Count’로 설정해줘야 가능
- 클러스터의 모든 Thread가 아닌 각 노드 당 최대 수로 설정 
- e.g ) 서버당 16개의 Thread 있을 때, 3개의 노드로 구성된 클러스터라고 48이 아닌 16으로 설정
  > - Event Driven Thread Count도 있으나, 현재 Experimental 이기 때문에 사용하지 않았음
  > - Event / Timer Driven 둘 다 사용할 시에는 적정선을 조정해야함 


- 모든 프로세서를 실행해 놓고 Queue가 쌓이는 특정 프로세스에게 많은 Concurrent Task 할당
- 30개까지 할당했으나, 2~3개 할당했을 때보다 처리량이 더 적어짐 
- 많은 Thread 할당으로 Threads 간의 race condition이 발생한 것으로 추정

- 처리량을 보고 2~3으로 설정 완료

        
#### 2. Run Duration을 통한 CPU 사용량 감소

#### 3. ExecuteScript Processor 의 구현 방법 변경 ( 배치로 )
- 하나씩 처리하는 것이 아닌, 하나의 ProcessSession에서 다수의 flowfile 처리
- 기존방식
```groovy
def flowFile = session.get()
// do something
```

- 변경방식
```groovy
List<FlowFile> flowFileList = session.get(BATCH_SIZE);
fileFileList.each { flowfile -> 
	… // do something
}
```
#### 4. ConsumeKafka_1_0 Processor ‘ Message Demarcator ‘ 설정

- ‘Message Demarcator’ 설정하지 않을 시, 한개의 flowfile의 content에는 하나의 로그메세지가 담겨있음
- 즉, 100개의 flowfile이라면 총 100개의 로그를 의미
- 하지만 Nifi는 Performance 측면에서 사이즈가 작은 많은 flowfile 보다는 사이즈가 큰 적은 flowfile 가 더 효율적
- Message Demarcator 설정 후, 하나의 flowfile 내 다수의 로그가 포함되어 있음
- 이에 따라 ValidateRecord Process의 처리속도가 향상

---

### 해결방안 - CONFIG 측면 

#### 1. Nifi Queue Swap threshold 변경 ( default 20,000 -> 100,000 )

- nifi.queue.swap.threshold
- 초당 약, 4000~5000개의 flowfile이 들어올 시, 몇 초만 지나도 queue에 쌓인 flowfile 들이 swap이 발생하여 처리 속도에 현저히 느려짐
- 최대한 swap이 발생하지 않도록 프로세서들의 수정하고, 유입되는 flowfile양이 많기 때문에 threshold를 올려 swap을 피함
- swap threshold를 올렸기 때문에, JVM Heap 메모리에 영향이 갈 수 있음

#### 1. ProvenanceRepository Implementation = WriteAheadProvenanceRepsitory로 변경 

##### 위에 내용을 다 했는데도 아직도 해결 못했었음.. 

- primary node를 제외하고 다른 노드에서의 데이터 처리량이 현저히 차이남 ( 사실상 100-200 처리하는 다른 node / primary node 3000) 약 30배 이상 처리함
- 이렇게 되다 보니 kafka consumer thread = nifi 에서는 concurrnet task 마다 파티션을 할당해줌
- primary node 를 제외한 다른 노드의 consumer는 lag가 계속 쌓임 ( 정작 primary node는 lag가 0 / 그냥 다 먹어버림)
- 서버 사양의 차이인가.. 봤음.. 근데 사실상 아님
- 그리고 다른 노드의 consumer들은 commit하는, 즉 poll 후 commit하는 시간이 계속 증가해 rebalance 발생해서 전반적으로 lag가 쌓이게 됌
- 일단 max.poll.interval.ms을 늘림 (기본 5분인데 10분으로 ) <- 사실상 이건 큰 영향이 없음
- primary node가 아닌 node의 몬가 문제가 잇을 것 같아서 로그를 확인해봄 
```
2019-02-26 17:09:12,083 WARN [Timer-Driven Process Thread-13]
 o.a.n.p.PersistentProvenanceRepository The rate of the dataflow is exceeding the provenance recording rate.
 Slowing down flow to accommodate.
 Currently, there are 96 journal files (946825875 bytes) and threshold for blocking is 80 (1181116006 bytes)
```

- primary node에서는 어떤 WARN도 발생하지 않음..

- 그래서 찾아본 해결 방안!! 

- nifi.properties 
- nifi.provenance.repository.implementation = default : org.apache.nifi.provenance.PersistentProvenanceRepository
  > - org.apache.nifi.provenance.WriteAheadProvenanceRepository로 변경
  > - WARN 로그도 안뜨고 현재 CPU, JVM Heap, 서버당 처리량이 균등함


---

#### ProvenanceRepository에 대한 이해
 
##### 일단 ProvenanceRepository는 무엇인가 ? 
  - flowfile의 history를 저장하는 곳
  - history는 각 데이터에 대한 data의 lineage를 기록함 
  - > 디렉토리보니, [0-9]{8}.prov.gz  /[0-9]{8}.prov <- 계속 생성됌 / index 디렉토리 / toc 디렉토리가 있음
  - flowfile에 대한 event가 발생될 때마다 새로운 provenance event가 생성되고, 이 이벤트는 FlowFile의 스냅 샷으로, 그 시점에 존재했던 흐름에 적합하고 적합합니다.
  - 이 이벤트 생길 때, flowfile의 Attribute , content에 대한 pointer , stat 정보를 복사해 Provenance Repository에 저장함
  - Provenance Repository는 모든 Provenance 이벤트를 보관함 ( retention은 있음 )
  
  - Flowfile의 attribute와 Content의 Pointer를 모두 Provenance Repository에 가지고 있어, 우리가 데이터의 Lineage 또는 처리 내역을 볼 수 있고
  - 나중에 데이터 자체를 보고 흐름의 어느 지점에서나 데이터르 재실행할 수 있음
  - 여기서 기억해야할 점
    - Provenance는 Content Repository에 있는 content를 복사하는 것이 아닌기 때문에, ( content에 대한 pointer만 저장) 
    - content Repository에 데이터가 없어지면 재실행할 수 없음
    > - 재실행만 안되고, lineage는 볼 수 있음 
    
  
  - 참고
    - Since provenance events are snapshots of the FlowFile, as it exists in the current flow, 
    - changes to the flow may impact the ability to replay provenance events later on. 
    - For example, if a Connection is deleted from the flow, the data cannot be replayed from that point in the flow, 
    - since there is now nowhere to enqueue the data for processing.
  
- Deeper View: Provenance Log Files

  - 각 prov evnet는 2개의 Map를 가짐 ( evnet 발생 전 attribute, 후 udpate된 attribute )
  - process에서 relation으로 나간 후 저장하지되지 않고, seession이 commit 될 때, attribute 값을 저장
  - Nifi가 실행중일 때는, 16개의 Provenance log file이 롤링되는 그룹으로 지정됌 ( 16개를 늘린다면 throughtput 증가)
  - log 파일은 30초마다 롤링 = 새롭게 생성된 provenace 이벤트는 새로운 16개의 로그파일 그룹에 쓰기 시작하고 원본 로그는 저장
  

참고 : https://alopresto.github.io/ewapr/

---

##### 1. PersistentProvenanceRepository
- Provenance event ( 즉, flowfile의 히스토리)를  Sequential하게 저장하고, 압축함, 그 다음에 index와 query 정보를 추가함 
- 대량의 작은 FlowFiles를 처리하는 NiFi 인스턴스에서 사용되는 경우 PersistentProvenanceRepository가 곧 병목 현상이 될 수 있습니다. 

- nifi.provenance.repository.index.threads
> - Provenance events를 인덱싱하는 쓰레드 개수 ( dafault : 2 )
> - 많은 양의 flowfile 있을 때는 indexing 하는 것이 bottleneck이 되기 때문에, 늘리는 것이 좋음
> - "The rate of the dataflow is exceeding the provenance recording rate. Slowing down flow to accommodate."  이 WARN이 뜬다면

- nifi.provenance.repository.always.sync
> - default : false  
> - 저장소에 대한 변경 사항은 디스크에 동기화됩니다. 즉, NiFi는 정보를 캐시하지 않도록 운영 체제에 요청합니다.
> - false하면 data loss 가능성이 있지만 성능은 없다 

- nifi.provenance.repository.journal.count
> - default : 16
> - Provenance Event 데이터를 serialize하는 데 사용해야하는 저널 파일 수
> - 이 값을 늘리면 더 많은 작업이 저장소를 동시에 업데이트 할 수는 있지만 나중에 저널 파일을 병합하는 것이 더 비쌉니다.
> - 이 값은 이상적으로 리포지토리를 동시에 업데이트 할 것으로 예상되는 스레드 수와 같아야하지만 필수 환경에서는 잘 작동하는 경향이 있습니다

##### 2. VolatileProvenanceRepository
- 메모리에 저장

##### 3. WriteAheadProvenanceRepository
- persistent 처럼 저널파일에 history을 위한 event를 직접 적지 않고, EventStore에 write-ahead log를 저장
- EventIndex (Apache Lucene 기반)로 작성된 데이터를 찾기 위해 EventIdentifier를 단순히보고하는 EventStore를 결합하여
  - 두 구성 요소를 함께 사용하여 높은 처리량과 효율적인 질의 및 검색 기능을 제공 할 수 있습니다 
- EventIndex를 사용하면 즉각적으로 질의할 수 있음
- PersistentProvenanceRepository은 배치 처리를 해서 기다려야함
- WriteAheadProvenanceRepository은 PersistentProvenanceRepository보다 3배 빠르며 더 많은 디스크 및 CPU 리소스를 사용할 수있게되면 더 확장 가능


- nifi.provenance.repository.concurrent.merge.threads
> - Lucene은 index를 위해 여러개의 segment파일을 만듬
> - 이 segment는 빠른 query를 위해 하나로 합쳐지는데, 합치는 threads의 개수 ( 각 디렉토리당, 즉 3개라해서 하나씩 주기 위해 3개가 아닌 1개로 설정해야함)
> - high throughtput 환경에서는 merge threads < index threads / number of directory
> - 2개 디렉 index thread 8 -> merge thread 4보다 작게
> - 크게 설정하면 모든 쓰레드가 병합에 사용되면서, indexing이 pending되어 latency가 증가할 수 있음

4. EncryptedWriteAheadProvenanceRepository

----


##### 추가 느껴본 문제 ConnectionLoss 에러

```
nifi.zookeeper.connect.timeout 3s -> 30s
nifi.zookeeper.session.timeout 3s -> 30s
nifi.cluster.node.connection.timeout 30s 
nifi.cluster.node.read.timeout  30 s
nifi.cluster.node.protocol.threads 10 -> 30
```

https://community.hortonworks.com/questions/108501/zookeeper-connection-error-in-nifi-version-nifi-12.html
https://issues.apache.org/jira/browse/CURATOR-209


