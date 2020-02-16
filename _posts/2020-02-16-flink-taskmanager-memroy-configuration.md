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


Flink 1.10.0부터 TaskManager Memory 설정을 fine-grained하게 설정하도록 변경하였고, 이유는 다음과 같음
> 1. 모든 운영환경에 합리적인 default값을 설정하는 것이 어려움 
> 2. 1.10.x 이하 버전에 사용하던 taskmanager.heap.size or taskmanager.heap.mb 설정은 On-Heap 뿐아니라, Off-Heap의 구성요소를 포함하는 설정값으로 이름의 혼동이 있음

---

## Taskmanager 메모리구조

그래서.. 먼저 Flink TaskManager의 메모리 구조를 보고, 어떻게 메모리 설정하도록 바뀌었는지 알아보고자함

<img src="https://user-images.githubusercontent.com/12586821/74604722-e2cc1a00-5103-11ea-9cb0-092319eeba36.png" width="250" height="600">

복잡하지만 크게 보면 2가지로 나눠져있음
 - **Total Process Memory**
 - **Total Flink Memory**

정리하면 **Total Process Memory = Total Flink Memory + (JVM Metaspace + JVM Overhead)**

---

**Total Flink Memory**를 자세히 보면 크게 또 2가지로 나눠져있음
1. JVM Heap Memory
2. Off-Heap Memory

**JVM Heap Memory**

- Framework Heap 영역
> - Flink 자체에서 필요한 전용 JVM 메모리

- Task Heap 영역
> - 우리가 개발한 코드, 즉 task와 operator가 실행되는 JVM heap memory

**Off-Heap Memory**

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

## Taskmanager 메모리 설정값

셋 중 하나의 설정값을 이용하여 Flink TaskManager 메모리 설정하기를 권장
1. taskmanager.memory.flink.size
2. taskmanager.memory.process.size
3. taskmanager.memory.task.heap.size and taskmanager.memory.managed.size


참고 
- taskmanager.memory.flink.size와 taskmanager.memory.process.size 둘다 설정하는 것은 추천하지않음
- 둘다 설정할 경우, 메모리 구성 충돌로 인해 배포가 실패 할 수 있음 

---

#### 1. taskmanager.memory.flink.size
- Total Flink Memory size
- JVM Metaspace and JVM Overhead을 제외한 메모리 
- Framework Heap Memory, Task Heap Memory, Task Off-Heap Memory, Managed Memory, and Network Memory.
 
#### 2. taskmanager.memory.process.size
- Total Process Memory size for the TaskExecutors. 
- taskmanager.memory.flink.size + JVM Metaspace + JVM Overhead 이 포함된 설정값 

#### 3-1. taskmanager.memory.task.heap.size
- Task Heap Memory size for TaskExecutors. 
- task실행을 위한 JVM heap memory
- 설정안할시, taskmanager.memory.flink.size - (Framework Heap Memory+ Task Off-Heap Memory+ Managed Memory + Network Memory)

#### 3-2. taskmanager.memory.managed.size 
- Managed Memory size for TaskExecutors.
- Memory Manger에 의해 관리되는 Off-heap memory 
- sorting, hash tables, caching of intermediate results and RocksDB state backend에 사용되는 메모리 영역
- 설정안할시 taskmanager.memory.flink.size값에서 taskmanager.memory.managed.fraction (default 0.4)


---

### Standalone 환경 메모리 설정 권장사항
- Total Flink memory (taskmanager.memory.flink.size)

### YARN/Mesos 환경 메모리 설정 권장사항
- Total process memory (taskmanager.memory.process.size)
> - 이 메모리 JVM heap, managed memory size , direct memory로 분할되어 설정
> - 컨테이너화 배포환경(YARN/Mesos)일 경우, Container 사이즈에 해당 

### State Backends에 따른 메모리 설정 

1. Heap state backend
- Stateless job 이거나 heap state backend (MemoryStateBackend or FsStateBackend)을 사용시, Managed memory는 0으로 설정)
 
 
2. RocksDB state backend
-  RocksDBStateBackend은 native memory영역을 사용하고 RocksDB는 default로 managed memory 사이즈로 native memory를 설정 
- [how to tune RocksDB memory](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/large_state_tuning.html#tuning-rocksdb-memory)


---

참고자료

[Flink 1.10.0 TaskManager Memory Configurtion](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_setup.html)
