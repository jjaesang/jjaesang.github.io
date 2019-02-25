---
layout: post
title:  "[Nifi] Nifi Concept #1 "
date: 2019-02-25 11:15:12
categories: Nifi 
author : Jaesang Lim
tag: Nifi
cover: "/assets/instacode.png"
---

## NIFI 
> 참고 : https://community.hortonworks.com/content/kbentry/7882/hdfnifi-best-practices-for-setting-up-a-high-perfo.html

### Databasae Repository ( H2 Setting )
```nifi.database.directory=./database_repository```
- Nifi는 2개의 H2 DB를 사용함 
  > - 1. User DB - keep track of user login
  > - 2. History DB - keep track of all change made on the graph
- Nifi 설치한 root 디렉토리에 설치
- 다른 디렉토리로 옮긴다 해서 성능상 차이는 없음
- 하지만 Nifi 업그레이드 후 사용자 및 구성 요소 히스토리 정보를 유지하기에 용이하게 이동하는 것을 추천

### FlowFile Repository
- Nifi UI의 모든 FlowFile들의 상태를 유지 관리
- FlowFile Repo가 문제가 생길시, Nifi가 현재 작업하고 있는 일부 또는 모든 파일에 대한 Access 권한을 잃을 수 도 있음
- 대부분의 문제가 발생하는것은 Disk 공간 부족
- 고성능 시스템(?)에서는 FlowFile Repo를 Content Repo와 Provenace Repo와 같은 디스크에 위치시키면 안됌 

- Nifi는 프로세스로 Physical file(content)을 직접 넘기지 않음
- FlowFile은 한 프로세서에서 다음 프로세서로의 전송 단위 역할

- FlowFile은 JVM 메모리에 있음
- 프로세스 간의 Connection 사이의 queue에 데이터가 계속 쌓인다면 JVM OOM 에러가 발생할 수 있음
- 그래서 Nifi에서는 일정한 flowfile 개수를 디스크로 swap 하는 설정값이 존재함
```nifi.queue.swap.threshold=20000```
- Queue에 들어가는 flowfile이 2만개가 넘어가면 디스크로 swap이 발생하고 얼마나 swap이 발생함에 따라 성능에 영향을 줌


### Content Repository
- 실제 데이터(content)가 저장되는 곳 
- Nifi 단일 instance 내에 여러가의 repo 설정 가능
```
nifi.content.repository.directory.contS1R1=/cont-repo1/content_repository
nifi.content.repository.directory.contS1R2=/cont-repo2/content_repository
nifi.content.repository.directory.contS1R3=/cont-repo3/content_repository
```
- *.contS1R1 은 내가 원하는 걸로~ ( 기본은 default라고 적혀있음 )
- 여러 디스크에 걸쳐 I/O로드를 나누면, 장애 발생시 상당한 성능 향상과 내구성을 얻을 수 있음
- 클러스터 환경에서 구축 시에는 Content Repo이름을 모두 다르게 설정하게 disk 사용률를 디버깅할 때 용이하게 하자

### Provenance Repository
- Content Repo와 비슷하게 많은 양의 provencance event를 Read/Write 작업을 진행함
- Dataflow의 모든 transaction이 기록 



