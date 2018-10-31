---
layout: post
title:  "YARN 아키텍처_NodeManager"
date: 2016-11-28 13:05:12
categories: YARN
author : Jaesang Lim
tag: YARN
cover: "/assets/hadoop_log.png"
---

# NodeManger의 핵심 기능

### 1. 컨테이너 시작


 - 컨테이너 시작을 가능하게 하기 위해서 NM은 컨테이너 명세의 일부로서 컨테이너의 실행시간에 대한 자세한 정보를 수신하기를 기대
 - 이 정보는 컨테이너의 커맨드 라인 명령, 환경 변수, 컨테이너가 필요로 하는 (파일) 리소스들의 목록과 보안 토큰을 포함

 - 컨테이너시작 요청을 받게 되었을 때 보안이 활성화 되어 있다면, NM은 먼저 사용자, 올바른 자원 할당 등을 검증
 - 그 후 NM은 컨테이너를 시작하기 위해서 다음의 단계를 수행한다.

	1. 명세된 모든 자원들의 로컬 복사본을 생성(DistributedCache).
	2. 컨테이너가 사용하기 위해 생성한 워킹 디렉터리를 분리하고, 로컬 자원들을 이 디렉터리에서 사용가능한 상태로 생성
	3. 실제의 컨테이너를 시작하기 위해서 실행 환경과 커맨드 라인을 사용

### 2. 로그 취합

 - NM은 응용프로그램이 완료된 후 로그들을 안전하게 HDFS등의 파일 시스템으로 옮길 수 있는 옵션을 제공
 - 단일 응용프로그램에 속하고 이 NM에서 실행한 모든 컨테이너들의 로그들은 취합되어 파일 시스템의 지정된 위치에 단일 로그 파일
 - 가능하다면 압축된 상태로 저장
 - 사용자들은 이 로그들을 YARN 커맨드 라인 툴, 웹UI 또는 FS에서 직접 접근
 `yarn logs -applicationId <applicationId>`

### 3. 맵리듀스 셔플 보조

- 맵리듀스 응용프로그램을 실행하기 위해서 필요한 셔플 기능은 보조서비스로 구현
- 이 서비스는 Netty 웹 서버를 시작하고 리듀스 태스크의 구체적인 MR 셔플 요청을 어떻게 처리해야 하는지 알고 있음
- MR ApplicaionMaster는 필요할 수 있는 보안 토큰과 함께 셔플 서비스를 위한 서비스 id 를 특정
- NM은 리듀스 태스크로 전달하는 셔플 서비스가 관계되는 Port와 함께 ApplicaionMaster를 제공한다.

<hr/>

# YARN NodeManager 아키텍처 
<center>
<img src="https://user-images.githubusercontent.com/12586821/47789555-2f95b100-dd58-11e8-8bdc-b35fda1f67e4.PNG"/>
</center>

#### 1. NodeStatusUpdater

 - 시작할 때, RM에 등록을 수행하고, 노드에 사용 가능한 자원들의 정보를 송신
 - NM-RM 통신은 노드에서 동작 중인 새 컨테이너, 완료된 컨테이너 등의 컨테이너 상태에 대한 업데이트를 제공
 - RM은 이미 동작중인 컨테이너를 종료하기 위해서 NodeStatusUpdater에게 신호를 줄 수 있음
<hr/>

#### 2. ContainerManager

 - ContainerManager는 NM의 핵심으로 노드에서 동작 중인 컨테이너를 관리하기 위한 기능 제공


##### 1) RPC server

 - AM로부터 새로운 컨테이너를 시작하거나 동작중인 컨테이너를 정지시키는 요청을 받는 역할
 - ContainerManager는 ContainerTokenSecretManager, NMTokenSecretManager를 이용하여 모든 요청을 승인, 인증
 - 이 노드에서 수행중인 컨테이너에 대한 모든 연산은 보안 도구에 의해 후처리될 수 있는 로그에 기록


##### 2) ResourceLocalizationService

 - 컨테이너가 필요한 다양한 파일 리소스를 안전하게 다운로드하고 관리와 접근 제어 제한 적용
 1. 지역화
	> LocalFileSystem에 원격지 자원을 복사하고 다운로딩하는 과정 
 2. 로컬리소스
	> 컨테이너 실행에 필요한 파일/라이브러리 나타냄
 3. 삭제서비스
	> 명령이 발생할 때 NM의 내부에서 실행되고 local 경로를 삭제를 위한 서비스
 4. 로컬라이저 ( Localizer) 
	> 지역화를 실행하는 프로세스 또는 thread
	> PUBLIC(PublicLocalizer),PRIVATE, APPLICATION(ContainerLocalizer) 이 있음
 5. 로컬캐시 (LocalCache) 



##### 3) ContainersLauncher

- 컨테이너를 가능한 빠르게 준비하고 시작하기 위해서 스레드 풀을 유지
- RM이나 AM에서 보내진 요청이 있다면 컨테이너의 프로세스들을 정리한다.


##### 4) AuxServices
 
- NM는 보조서비스들을 설정하여 NM의 기능을 확장하기 위한 프레임워크를 제공
- 특정한 프레임워크들이 필요로 하는 노드 별 커스텀 서비스 사용을 허가하면서도 여전히 NM의 다른 부분으로부터 분리
- 이러한 서비스들은 NM이 시작되기전에 설정
- 예비 서비스들은 노드에서 응용프로그램의 첫번째 컨테이너가 시작되었을때와 응용프로그램이 완료되었다고 여겨질 때 통보

##### 5) ContainerMonitor

- 컨테이너가 시작된 후 이 구성요소는 컨테이너가 실행되는 동안의 자원 사용 관찰
- 메모리같은 자원의 공평한 공유와 격리를 강제하기 위해서 각 컨테이너는 RM에 의해서 자원들의 일부를 할당
- ContainerMonitor는 각 컨테이너의 사용을 지속적으로 모니터링 
- 컨테이너가 할당을 넘어선다면, 컨테이너를 종료시키기 위해 신호를 보냄
- 이 것은 탈주자 컨테이너가 동일한 노드에서 실행중인 다른 정상 동작하는 컨테이너들에게 영향을 주는 것을 막는다.


##### 6) LogHandler

- 컨테이너의 로그들을 로컬 디스크에 유지하거나 압축하여 파일 시스템에 업로드할 수 있는 플로그인 기능

<hr/>
#### 3. ContainerExecutor

- 컨테이너가 필요로 하는 파일들과 디렉터리들을 안전하게 배치
- 컨테이너에 상응하는 프로세스들을 시작하거나 정리하기 위해서 운영체제와 상호작용한다.

<hr/>
#### 4. NodeHealthCheckerService

- 미리 구성된 스크립트를 주기적으로 실행하여 노드의 상태를 점검하는 기능을 제공
- 디스크에 가끔 임시파일을 생성하여 디스크의 상태를 모니터링
- 시스템의 모든 상태 변경은 정보를 차례차례 RM으로 전송하는 NodeStatusUpdater로 보고된다.


<hr/>
#### 5. Security


##### 1) ApplicationACLsManager

- NM은 허가된 사용자만 접근해야 하는 ‘웹UI의 컨테이너 로그’같은 사용자 API 선별
- 각 application 마다 ACL을 관리하며 요청을 받았을 때 ACL 적용

##### 2) ContainerTokenSecretManager

- 수신된 다양한 요청을 확인해 시작 컨테이너의 모든 요청이 RM에게 제대로 승인됐음을 보장

<hr/>
#### 6. WebServer

- 응용프로그램, 주어진 시점에서 실행 중인 컨테이너들, 노드 상태에 관련된 정보들 
- 컨테이너에 의해서 생성된 로그들의 목록을 보여줌
