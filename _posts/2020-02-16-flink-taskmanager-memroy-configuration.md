---
layout: post
title:  "Flink 1.10.0 Taskmanager Memory Concepts & Configuration"
date: 2020-02-16 20:05:12
categories: Flink
author : Jaesang Lim
tag: Flink
cover: "/assets/instacode.png"
---


# Flink 1.10.0 Taskmanager Memory Concepts & Configuration

[Flink 1.10.0 TaskManager Memory Configurtion](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_setup.html)

개요

- Flink 1.10.0부터 TaskManager Memory 설정을 fine-grained하게 설정하도록 변경하였고, 그 이유는 다음과 같음
> 1. 모든 운영환경에 합리적인 default값을 설정하는 것이 어려움 
> 2. 1.10.x 이하 버전에 사용하던 taskmanager.heap.size or taskmanager.heap.mb 설정은 On-Heap 뿐아니라, Off-Heap의 구성요소를 포함하는 설정값으로 이름의 혼동이 있음

---

그래서.. 먼저 Flink TaskManager의 메모리 구조를 보고, 어떻게 메모리 설정하도록 바뀌었는지 알아보고자함

<img width="359" alt="image" src="https://user-images.githubusercontent.com/12586821/74604722-e2cc1a00-5103-11ea-9cb0-092319eeba36.png">


복잡하지만 크게 보면 2가지로 나눠져있음
## 1. Total Process Memory
## 2. Total Flink Memory

정리하면 **Total Process Memory = Total Flink Memory + (JVM Metaspace + JVM Overhead)**

---

**Total Flink Memory**를 자세히 보면 크게 또 2가지로 나눠져있음
1. JVM Heap Memory
2. Off-Heap Memory

### JVM Heap Memory 

- Framework Heap 영역
> - Flink 자체에서 필요한 전용 JVM 메모리

- Task Heap 영역
> - 우리가 개발한 코드, 즉 task와 operator가 실행되는 JVM heap memory

### Off-Heap Memory

- Managed Memory
> - Flink에 관리되는 메모리로, native memory 즉, off-heap에 할당
> - Managed Memory을 사용하는 워크로드
1. Streaming - RocksDBBackend
2. Batch - sorting, hash tables, caching of intermediate results.

- Network
> - Task 간에 data를 교환, exchange을 위한 메모리 공간 
> - e.g. 네트워크 전송을 위한 버퍼링 공간 

- JVM Overhead
> - JVM overhead: e.g. thread stacks, code cache, garbage collection space etc

---

이제 어떻게 메모리르 설정할 수 있게 하는가에 대해 알아보자 

셋 중 하나의 설정값을 이용하여 Flink TaskManager 메모리 설정하기를 권장
1. taskmanager.memory.flink.size
2. taskmanager.memory.process.size
3. taskmanager.memory.task.heap.size and taskmanager.memory.managed.size


참고 

- Default flink-conf.yaml은 기본 메모리 구성의 일관성을 유지하기 위해 taskmanager.memory.process.size를 설정함
- 1)taskmanager.memory.flink.size 와 2)taskmanager.memory.process.size 둘다 설정하는 것은 추천하지않음
- 둘다 설정할 경우, 메모리 구성 충돌로 인해 배포가 실패 할 수 있음 

---

각 설정값을 정리하자면

### 1. taskmanager.memory.flink.size
> - Total Flink Memory size
> - JVM Metaspace and JVM Overhead을 제외한 메모리
> - Framework Heap Memory, Task Heap Memory, Task Off-Heap Memory, Managed Memory, and Network Memory.
 
### 2. taskmanager.memory.process.size
Total Process Memory size for the TaskExecutors. 
> - taskmanager.memory.flink.size + JVM Metaspace + JVM Overhead 이 포함된 설정값 

### 3-1. taskmanager.memory.task.heap.size
> - Task Heap Memory size for TaskExecutors. 
> - task실행을 위한 JVM heap memory
> - 설정안할시, taskmanager.memory.flink.size - (Framework Heap Memory+ Task Off-Heap Memory+ Managed Memory + Network Memory)

### 3-2. taskmanager.memory.managed.size 
> - Managed Memory size for TaskExecutors.
> - Memory Manger에 의해 관리되는 Off-heap memory 
> - sorting, hash tables, caching of intermediate results and RocksDB state backend에 사용되는 메모리 구긴
> - 설정안할시 taskmanager.memory.flink.size값에서 taskmanager.memory.managed.fraction (default 0.4)


설정 팁
Standalone 환경
- Total Flink memory (taskmanager.memory.flink.size)

Configure memory for standalone deployment


1. total Flink memory (taskmanager.memory.flink.size)
2. how much memory is given to Flink
3. you can adjust JVM metaspace if it causes problems.

The total Process memory is not relevant
because JVM overhead is not controlled by Flink or deployment environment,  only physical resources of the executing machine matter in this case.

총 프로세스 메모리는 관련이 없습니다.
JVM 오버 헤드는 Flink 또는 배치 환경에 의해 제어되지 않으므로이 경우 실행중인 시스템의 실제 자원 만 중요합니다.



(YARN/Mesos) 환경 
2. Total process memory (taskmanager.memory.process.size)
> - 이 메모리 JVM heap, managed memory size , direct memory로 분할되어 설정
> - 컨테이너화 배포환경(YARN/Mesos)일 경우, Container 사이즈에 해당 


---

Configure memory for state backends

Heap state backend
- Stateless job 이거나, 
1. running a stateless job 이거나 heap state backend (MemoryStateBackend or FsStateBackend을 사용시, Managed memory는 0으로 설정
> - This will ensure that the maximum amount of memory is allocated for user code on the JVM.
 
 
RocksDB state backend
-  RocksDBStateBackend은 native memory영역을 사용하고 RocksDB는 default로 managed memory 사이즈로 native memory를 설정 

how to tune RocksDB memory 
https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/large_state_tuning.html#tuning-rocksdb-memory

state.backend.rocksdb.memory.managed

