---
layout: post
title:  "YARN 아키텍처_ResourceManager"
date: 2016-11-28 13:05:12
categories: YARN
author : Jaesang Lim
tag: YARN
cover: "/assets/hadoop_log.png"
---

# YARN ResourceManager 아키텍처 

<img src="https://user-images.githubusercontent.com/12586821/47788808-26a3e000-dd56-11e8-86e2-11a110216742.PNG"/>
### 1. 클라이언트간의 상호 작용

##### 1) ClientService

- 리소스매니저의 클라이언트 인터페이스
- Client에서 ResourceManager로 향하는 모든 RPC통신을 처리하는 인터페이스
- 애플리케이션 제출/ 종료 애플리케이션과 queue, 클러스터 통계, 사용자 ACL, 보안 정보 제공

##### 2) AdminService

- YARN의 클러스터 관리자는 종종 실행해야할 목록이 있음
- 이 관리 요청이 일반 사용자의 빈번한 요청때문에 서비스를 받지 못하는 경우가 없도록, 그리고 더 높은 우선순위를 가지도록 하기 위해서 노드-목록 갱신
- 큐의 설정 등의 관리 명령은 AdminService 인터페이스를 통하여 수행

<hr/>

### 2. 노드와의 상호 작용 

##### 1) ResourceTrackerService

- 모든 노드에서 오는 RPC에 응답
- 새 노드의 등록, 유효하지 않거나 퇴역한 노드로부터의 연결 거부, 노드의 하트비트 획득과 스케쥴러로의 전달을 책임
- 이 요소는 다음의 NMLivelinessMonitor와 매우 밀접하게 연관되어 있음


##### 2) NMLivelinessMonitor ( = 노드매니저의 무결점 동작 모니터 ) 

- 운영되고 있는 노드들을 추적하고 정지한 노드를 명확히 하기 위해서 이 구성요소는 각 노드의 마지막 하트비트 시간을 추적
- 미리 정의된 간격의 시간내에 하트비트를 보내지 않는 어떠한 노드든 정지한 것으로 여기고 RM에 의해서 만료
- 만료된 노드에서 현재 수행중인 모든 컨테이너는 정지한 것으로 표시되고 이후로 만료된 노드에는 어떠한 새 컨테이너도 배정하지 않습니다.


##### 3) NodesListManager

- 유효한 노드와 제외된 노드 모두 NodeListManager의 메모리 내에서 관리. 
- yarn.resourcemanager.nodes.include-path와 yarn.resourcemanager.nodes.exclude-path에 설정된 호스트 설정 파일을 읽고 그 파일들에 기반하여 초기 노드 목록 생성
- 시간이 진행됨에 따라 해제된 노드에 대한 추적을 유지


<hr/>

### 3. 애플리케이션간의 상호 작용

##### 1) ApplicationMasterService

- 모든 ApplicationMaster과의 RPC를 책임, 응답
- 새로운 ApplicationMaster의 등록과 종료된 ApplicationMaster으로부터 보내오는 종료/해제 요청
- 현재 동작중인 모든ApplicationMaster으로부터의 컨테이너 배치/해제 요청을 받아 YarnScheduler로 전송하는 역할


##### 2) AMLivelinessMonitor

- 동작, 정지/응답없는 ApplicationMaster들의 관리를 돕기 위해서 각 ApplicationMaster의 추적과 마지막 하트비트 시간을 유지하고, 이 기록을 바탕으로 RM에 의해 만료
- 만료된 AM에서 현재 수행중인 모든 컨테이너는 정지(dead)한 것으로 표시
- RM은 동일한 AM을 새 컨테이너에서 동작시키기 위해 스케쥴 ( 최대 4번 ) 

<hr/>

### 4. ResourceManager의 핵심 ( Scheduler )

##### 1) ApplicationsManager

- 제출된 응용프로그램의 집합을 관리 후, 스케줄러에게 전달
- 애플리케이션이 사용 가능한 자원 요청 여부 검증
- 완료된 응용프로그램의 캐시를 유지 ( ApplicationSummary )하여 웹 UI나 명령행을 통한 사용자의 요청에 대해 응답함.


##### 2) ApplicationACLsManager

- RM은 클라이언트와 어드민에 권한이 있는 사용자만 접근을 허락
- ACL(Access-Control-List)들을 응용프로그램별로 관리
- 응용프로그램의 중단, 응용프로그램의 상태 요청등이 있을 때 권한을 강제한다.


##### 3) ApplicationMasterLauncher

- ApplicationMaster 실행
- 종료된 이전의 AM 시도들과 함께 새로 제출된 AM을 실행하기 위한 노드메니저와 통신하는 스레드 풀을 관리
- 또한 응용프로그램이 정상적으로 혹은 강제로 종료되었을 경우 AM을 청소


##### 4) YarnScheduler

- 커패시티와 큐 등의 제약에 따라 실행 중인 다양한 애플리케이션에 할당
- 메모리, CPU, 디스크, 네트워크 등, 응용프로그램의 자원 요구사항에 기반


##### 5) ContainerAllocationExpirer

- 할당된 컨테이너들이 AM들에 의해서 사용되고 있으며 차후에 컨테이너가 해당되는 노드매니저에서 실행되는지 확인

- AM들은 신뢰할 수 없는 사용자 코드로 실행하고 컨테이너를 사용하지 않고 할당만을 유지할 수 경우 클러스터 사용률이 저하
- 이 문제를 해결하기 위해서, ContainerAllocationExpirer는 해당하는 노드에서 사용되고 있지 않는 할당된 컨테이너들의 목록을 유지

- 어떠한 컨테이너이든 해당하는 노드매니저가 정해진 시간(기본 10분)안에 RM으로 컨테이너가 동작을 시작하였다고 보고하지 않으면 컨테이너는 정지했다고 가정하고 RM에 의해 만료된다.


<hr/>

### 5. TokenSecretManagers ( 보안 )

- 리소스매니저는 토큰, 다양한 RPC 인터페이스에서 인증/허가를 위해 사용되는 비밀키를 관리하기 위해 몇개의 SecretManager를 운영


##### 1) ContainerTokenSecretManager

- AM가 Container 시작 요청의 승인을 위해 사용
- Container를 사용하기 위해서 RM가 AM에게 발급한 특별한 토큰 세트와 ContainerToken 관리
- RM이 사용하는 보안 도구
- RM은 이 토큰을 사용해 Container 시작에 필요한 중요 정보를 AM를 통해 NM에게 전송 
 > 직접 NM에게 전송할 수 없음 .. 왜? 
	> - ContainerToken은 Container 할당 이후에 encoding 되기 때문
	> - RM은 AM에 보내기 전에 CotainerToken에 Contain값 암호화
	> - { ContainerID , NM Address, Application launcher, Resource, expireTimestamp, masterKeyIdentifier, RM identifier }


##### 2) AMRMTokenSecretMananger

- AM형태를 모방해 RM에게 자원, 스케줄링 요청하는 경우를 방지하고자 RM은 ApplicationAttempt마다 AMRMToken 사용



##### 3) NMTokenSecretManager

- AM은 NM마다 하나의 커넥션을 관리하기 위해 NMToken사용하여 모든 요청을 노드에 사용
- RM은 NM마다 ApplicationAttempt 당 하나의 NMToken 발생
- 새로운 Container가 생성될 때마다 RM은 NMToken을 AM에게 발행
- 네트워크 최적화를 위해 MToken을 수집할 때마다 NMTokenCache 관리 
- AM은 항상 최신 NMToken 사용하기 위해 NM은 AM로부터 하나의 NMToken만 수신


##### 4) DelegationTokenRenewer

- 보안 모드에서 RM은 Kerberos로 인증
- 응용프로그램을 대신하여 파일 시스템 토큰을 갱신하는 서비스를 제공해야 힘
- 이 구성요소는 응용프로그램이 동작하는 동안 그리고 토큰이 갱신할 수 없기 전까지 제출된 응용프로그램의 토큰을 갱신

