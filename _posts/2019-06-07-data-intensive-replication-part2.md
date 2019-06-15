---
layout: post
title:  "[Data-Intensive] Ch5. 복제 (Multi-leader based Replication)"
date: 2019-06-07 19:15:12
categories: Data-Intensive-Application 
author : Jaesang Lim˚
tag: Data-Intensive-Application 
cover: "/assets/instacode.png"
---

## 복제
'데이터 중심 애플리케이션 설계'를 읽고 정리하고자함

- 복제는 3가지 접근 방식으로 나눠서 정리할 예정
1. 도입 및 단일 리더 복제
**2. 다중 리더 복제**
3. 리더 없는 복제

- 이 글은 복제에 대한 **다중 리더 복제(Multi-leader based Replication)** 방법에 대해 다룰 것

---

## 다중 리더 복제 

리더 기반 복제의 한계
 - 리더가 하나만 존재해, 모든 쓰기는 해당 리더를 거쳐야함
 - 어떠한 이유로 리더와 연결할 수 없으면 데이터베이스에 쓸 수 없음
 
해결책
 - 쓰기를 허용하는 노드를 하나 이상 두자
 - 쓰기 처리를 하는 각 노드는 데이터 변경을 다른 모든 노드에 전달함
 - 이 방식을 **다중 리더** , **마스터 마스터**, **액티브/액티브 복제** 라고 함
 - 각 리더는 다른 리더의 팔로워가 되는 구조 
  
---

### 다중 리더 복제 사용 사례

#### 1. 다중 데이터센터 운영
![d1](https://user-images.githubusercontent.com/12586821/59100262-78ea5680-8960-11e9-805e-46012f869d69.PNG)

- 각 데이터센터마다 리더가 있음
- 각 데이터센터 내에는 보통의 '리더-팔로워 복제' 수행하고, 데이터센터 간에는 각 리더가 다른 데이터센터의 리더에게 변경사항 복제

- 자동 증가 키, 트리거, 무결성 제약은 다중 리더 복제의 위험한 영역으로 간주됌

#### 2. 오프라인 작업을 하는 클라이언트

- 인터넷 연결이 끊어진 동안 애플리케이션이 계속 동작해야하는 상황
- e.g) 휴대전화의 캘린더앱
> - 읽기 요청
> > - 인터넷의 상관없이 언제든 회의 일정을 확인할 수 있어야함
> - 쓰기 요청
> > - 새로운 회의를 등록할 수 있어야함
> - 오프라인 상태에서 데이터를 변경하면 온라인 상태가 될 때, 서버와 동기화해야함
- 이 경우 모든 디바이스에는 리더로 동작하는 로컬 데이터베이스가 있음 (쓰기 요청을 받기 위함)
- 모든 디바이스 상에서 캘린더 복제 서버 간의 다중 리더 복제를 '비동기'으로 프로세스가 있음 (동기화)


---

### 쓰기 충돌 다루기

![d2](https://user-images.githubusercontent.com/12586821/59100263-78ea5680-8960-11e9-8a66-1d720d183680.PNG)
- 사용자 1은 페이지 제목을 A->B로 변경함
- 같은 시각, 사용자 2는 A->C로 변경함
- 각 사용자의 변경 내역은 로컬 리더에 성공적으로 적용
- 하지만, 비동기로 복제할 때 충돌을 감지하는 상황


#### 1. 동기 대 비동기 충돌 감지
- 다중 리더 설정은 두 쓰기는 모두 성공하며, 충돌은 이후 특정 시점의 비동기로만 감지됨
- 이론적으로 충돌 감지는 **동기식**으로 만들 수 있음
- 즉, 쓰기가 성공한 사실을 사용자에게 말하기 전, 모든 복제 서버에 복제되기를 기다림
- 하지만, 이렇게 되면 다중 복제 리더의 장점인 **'각 복제 서버가 독립적으로 쓰기를 허용'**이 사라짐
- 동기식으로 충돌을 감지하려면, 단일 리더 복제만 사용해야할 수 있음

#### 2. 충돌 회피
- 충돌을 처리하는 가장 간단한 방법은 아예 충돌을 피하는 것
- 특정 레코드의 모든 쓰기가 동일한ㄴ 리더를 거치도록 애플리케이션에서 보장한다면, 충돌은 발생하지 않음
- 많은 다중 리더 복제 구현 사례에서 충돌을 잘 처리하기 어려워서, 충돌을 피하도록 권장함

#### 3. 일관된 상태 수렴
- 다중 리더은 쓰기 순서가 정해지지 않아 최종값이 무엇인지 명확하지 않음
- 단순하게 각 복제 서버가 쓰기를 본 순서대로 적용한다면, 데이터베이스는 결국 일관성이 없는 상태가 됌 ( 망한거지 )
- 모든 복제 계획은 모든 복제 서버가 최종적으로는 동일하는 것을 보장해야함
- 따라서, 데이터베이스는 수렴(convergent) 방식으로 충돌을 해소해야함
- 모든 변경이 모든 복제 서버에 동일한 최종값이 전달되게 해야함

#### 방법

1. 각 쓰기에 ID를 부여하고 가장 높은 ID를 가진 쓰기를 고르고 다른 쓰기 요청은 버림
> - 타임스탬프를 사용한다면, 최종 쓰기 승리(last write wins, LWW)라고 함
> - 대중적인 방법이지만, 데이터 유실의 위험이 있음

2. 각 복제 서버에 고유 ID를 부여하고 높은 숫자를 가진 복제 서버의 값을 최종값으로 선정
> - 역시, 데이터 유실 위험 있음

3. 어떻게든 값을 병합해버림

4. 충돌을 기록해 모든 정보를 보존함
> - 나중에 사용자에게 보여줘, 충돌을 해소하도록 애플리케이션에서 처리

#### 4. 사용자 정의 충돌 해소 로직
- **대부분** 다중 리더 복제는 애플리케여선 코드르 사용해 충돌 해소 로직을 작성함

쓰기 수행 중
> - 복제된 변경 사항 로그에서 데이터베이스 시스템이 충돌을 감지하자마자 충돌 핸들러 호출
> - 이 핸들러는 사용자에게 충돌 내용을 표시하지 않고, 백드라운드 프로세스에서 빠르게 실행되어야함

읽기 수행 중
> - 충돌을 감지하면 모든 충돌 쓰기를 저장
> - 다음 번 데이터 읽을 때, 여러 버전의 데이터가 애플리케이션에 반환되고, 사용자가 처리
> - 처리된 결과를 다시 데이터베이스에 기록 (카우치DB가 이렇게 작동)

- 충돌 해소는 보통 전체 트랜잭션이 아닌 **개별 로우나 문서 수준**에서 적용
- 원자적으로 여러 다른 쓰기를 수행하는 트랜잭션이라면 , 각 쓰기는 충돌 해소를 위해 여전히 별도로 간주 

#### 5. 자동 충돌 해소

- 충돌 해소 규칙은 빠르게 복잡해질 수 있고 코드에서 오류가 발생할 수 있음
- 아마존의 장바구니 충돌 해소 로직은 일정 시간 동안 추가된 상품을 보관하지만, 삭제한 상품은 보존하지 않음
- 즉, 전에 상품을 이미 삭제했어도 가끔 구매자의 장바구니에 다시 해당 상품이 보이기도 함

동시에 데이터를 수정할 때 발생하는 충돌을 자동으로 해소하는 연구들
1. 충돌 없는 복제 데이터타입(conflict-free replicated datatype, CDRT)
> - Set,Map, 정렬 목록, 카운터 등을 위한 데이터 구조의 집합으로 
> - 동시에 여러 사용자가 편집할 수 있고 충돌을 자동으로 해소

2. 병합 가능한 영속 데이터 구조(mergeable persistent data structure)
> - Git, 버전 제어시스템과 유사함
> - 히스트리를 유지하고, 삼중 병합 함수를 사용(three-way merge function)
> - CDRT는 two-way merge function 사용

3. 운영 변환(operational transformation)
> - 구글 독스 같은 협업 편집 애플리케이션의 충돌 해소 알고리즘
> - 문서를 구성하는 문자 목록과 같은 정렬된 항목 목록의 동시 편집을 위해 설계

---

### 다중 리더 복제 토폴로지

- 복제 토폴로지란?
> - 쓰기를 한 노드에서 다른 노드로 전달하는 통신 경로

- 위의 첫번째 그림에서는 두 리더가 있다면 가능한 토폴리지는 하나뿐
> - 리더 1은 모든 쓰기를 리더 2에 전송해야하고, 반대도 마찬가지
- 리더가 둘 이상이라면, 다양한 토폴로지가 가능함

![d3](https://user-images.githubusercontent.com/12586821/59100260-78ea5680-8960-11e9-9a5a-76940eb268be.PNG)
다중 리더 복제를 설정하는 토폴로지 예제


#### 1. 전체 연결 토폴로지 (all-to-all)
> - 모든 리더가 각자의 쓰기를 다른 모든 리더에게 전송

#### 2. 원형 토폴로지(circular topology)
> - mysql이 사용함
> - 각 노드가 하나의 노드로부터 쓰기르 받고, 이 쓰기는 다른 한 노드에 전달함

#### 3. 별 토폴로지
> - 지정된 루트 노드 하나가 다른 모든 노드에 쓰기를 전달
> - 별 모양은 트리로 일반화 가능

 원형, 별 토폴로지에서의 **쓰기**는 모든 복제 서버에 도달하기 전에 여러 노드를 거침
 - 노드들은 다른 노드로부터 받은 데이터 변경 사항을 전달해야함
 - 무한 루프를 방지하기 위해 각 노드별 고유 식별자가 있고, 복제로그에서 각 쓰기는 거치는 모든 노드의 식별자가 태깅됌
 - 노드가 데이터 변경 사항을 받았을 때, 자신의 식별자가 태깅된 경우는 노드가 이미 처리한 사실을 알기 때문에 무시해버림

원형, 별 토폴로지의 문제점
 - 하나의 노드에서 장애 발생하면 다른 노드간에 복제 메세지 흐름에 방해를 줌
 - 즉, 해당 노드가 복구될 때까지 통신할 수 없음

전체 연결 토폴로지의 문제점
![d4](https://user-images.githubusercontent.com/12586821/59100261-78ea5680-8960-11e9-9425-55fa36db9ea1.PNG)
> - 일부 네트워크 연결이 다른 연결보다 빠르다면 일부 복제 메세지가 다른 메세지를 추월해버릴 수 있음

- 이런 이벤트를 정렬하는 위해 **버전 벡터(version vector)**를 사용할 수 있음
- 많은 다중 리더 복제 시스템에는 충돌 감지 기법이 제대로 구현되어 있지 않음 


### 결론
- 다중 리더 복제 시스템을 하려면 위에 다룬 문제를 인지하고 문서를 주의깊게 읽은 다음
- 데이터베이스를 철저히 테스트해 실제로 믿을 만한 보장을 제공하는지 확인하자

