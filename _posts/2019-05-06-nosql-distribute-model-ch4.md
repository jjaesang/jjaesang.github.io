---
layout: post
title:  "[NoSQL] Ch4. 분산 모델"
date: 2019-05-06 20:15:12
categories: NoSQL 
author : Jaesang Lim˚
tag: NoSQL
cover: "/assets/instacode.png"
---

## 분산 모델 

'빅데이터 세상으로 떠나는 간결한 안내서 NoSQL'를 읽고 정리하고자함

---

### 요점 정리
- 데이터를 분산하는 형식은 '샤딩'과 '분산' 두가지 있음
> 1. 샤딩
> > - 여러 서버에 데이터를 나눠 분산
> > - 각 서버는 전체 데이터의 부분 집합을 저장하며 같은 데이터는 한 서버에서만 찾을 수 있음
> 2. 복제
> > - 여러 서버에 데이터를 복사하므로, 같은 데이터를 여러 곳에서 찾을 수 있음

- 두 기법 중 하나를 사용하거나 두가지 모두 사용할 수 있음

- 복젝는 두가지 형태가 있음
> 1. 마스터-슬레이브 복제
> > - 마스터는 데이터의 원본 출처가 되어 쓰기를 처리하고,
> > - 슬레이브는 마스터와 데이터 동기화하고 읽기를 처리
> 2. 피어-투-피어 복제
> > - 모든 노드에 쓰기를 허용
> > - 노드는 데이터 복사를 동기화하기 위해 조율함

- 마스터-슬레이브 복제는 업데이트 충돌 발생을 줄임
- 피어-투-피어 복제는 한 노드에 모든 쓰기 부담을 지우지 않도록 단일 실패 지점이 생기지 않도록 함 

---

- NoSQL이 관심을 끈 주된 이유는 '대규모 클러스터'에서 데이터베이스르 실행할 수 있는 능력
- 집합은 분산의 자연스러운 단위로 사용할 수 있어, 집합 지향은 수평 확장과 잘 맞음

분산 모델의 장점

1. 더 많은 양의 데이터를 처리할 수 있는 데이터 저장소를 만들 수 있음
2. 많은 읽기, 쓰기 트패픽을 처리하도록 할 수 있음
3. 네트워크 속도가 느려지거나, 장애가 발생하는 상황에서도 가용성이 더 높아짐

분산 모델의 단점
1. 클러스터에서 실행하는데 복잡성이 따름

데이터 분산하는 방법
1. 복제 (replication)
> - 같은 데이터를 복사해 여러 노드에 분산하는 방법
2. 샤딩 (sharding)
> - 각 노드마다 다른 데이터를 놓음

### 1. 단일 서버 
- 가장 단순한 분산 방법은 분산을 하지 않는 것 
- 많은 NoSQL이 클러스터 환경을 고려해 설계되었지만, 단일 서버에서 사용한다해서 문제가 되지 않음
- 데이터 사용이 대부분 집합 구조를 처리하는 것이라면, 단일 서버에서 문서 저장소 또는 키-값 저장소를 사용하는 것도 괜찮음

### 2. 샤딩
- 바쁜 데이터 저장소의 이유는 '많은 사람들이 데이터 집합의 다른 부분에 접근하기 때문'
- 데이터의 다른 부분을 다른 서버에 두어 수평확장하는 것이 '샤딩'
- 함께 접근되는 데이터는 같은 노드에 모여 있게 하고, 이런 데이터 무더기를 여러 노드에 잘 배치해 최적의 데이터 접근성을 제공해야함

문제
1. 데이터를 어떻게 뭉쳐놓으면 한 사용자가 한 서버로부터 데이터를 대부분 얻게할 것인가
> - 집합은 '함께 접근되는 빈도가 높은 데이터'를 모이는 것이므로, 집합은 분산의 아주 좋은 단위
2. 부하를 균등하게 분배하는 것 
> - 집합을 잘 배치해 모든 노드에 균등하게 분산하고 같은 양의 부하가 걸리도록 해야함
> - 이런 경우에는 순차적으로 읽힐 집합을 함께 두는 것이 유리하고
> - 빅테이블은 행을 사전 순으로 유지하고 웹 주소는 역도메인네임(com.zum)으로 정렬 
> - 이 방법으로 여러 페이지에 대한 데이터가 함께 접근되어 처리효율이 높아짐

- 과거에는 샤딩을 애플리케이션 로직에서 처리
- e.g) 성의 앞 글자가 A~D , E~G 로 나워 다른 샤드에 넣을 수 있음
> - 이러면 애플리케이션 코드가 복잡해지고, 샤딩의 균형을 다시 맞춰야할 경우 코드 수정 및 데이터의 전환이 필요함

- 걱정마! NoSQL은 '자동 샤딩'을 제공
> - 즉, 데이터를 각 샤드에 할당하고 데이터에 접근할 때 올바른 샤드에 접근하도록 보장함

성능 측면
- 읽기/쓰기 모두 향상 시킬 수 있음
- 복제를 캐시와 함께 사용하면, 읽기 성능을 높일 수 있지만, 쓰기가 많은 애플리케이션에는 큰 성능향상이 없음
- 샤딩은 쓰기에 대한 수평 확장 방법을 제공 

- 샤딩만 단독적으로 사용하면 복원력은 거의 개선되지 않음 ( resilience )
> - 데이터가 다른 노드에 있지만, 해당 노드가 실패하면 데이터 접근할 수 없음 
> - 그래서 샤딩을 단도적으로 사용하지 않음 

- 단일 서버에서 싲가해 부하가 서버 용량을 초과할 것 같을 때만 샤딩을 추가하는 것이 더 좋음
- 하지만, 샤딩을 새롭게하는 것은 어려운 일이니, 샤딩을 충분히 미리 적용하고 샤딩을 수행할 여유가 있을 때 적용하는 것을 추천

### 3. 마스터-슬레이브 복제 

- 마스터는 데이터의 출처가 되고 데이터에 대한 업데이틀 처리할 책임을 가짐
- 다른 노드는 슬레이브 또는 secondary 노드
- 복제 프로세스는 슬레이브를 마스터와 동기화하는 작업

장점
1. 마스터-슬레이브 복제는 '읽기'가 많이 발생하는 데이터 집합을 가진 경우 확장할 때 큰 도움을 줌
> - 슬레이브 노드를 추가하고 모든 읽기 요청을 슬레이브에서 처리하면 더 많은 읽기 요청을 처리해 수평적 확장이 가능
> - 업데이터를 하고 이를 전파하는데 여전히 마스터의 능력에 제한이 있음
> - 쓰기 트래픽이 많은 데이터 집합에서는, 좋은 구성이라고 할 수 없음

2. 읽기 복원력 
> - 마스터가 실패해도 슬레이브가 읽기 요청을 처리할 수 있음 
> - 마스터가 죽어도, 슬레이브에 복제되어있다면 바로 마스터로 할당되어 빠르게 복구할 수 있음

- 모든 읽기/쓰기 트래픽은 마스터가 처리하고 슬레이브는 핫 백업으로 동작할 수 있음
> - 서버 실패에 적절히 대응할 수 있기를 원할 때 유용

- 읽기 복원력을 얻으려면 읽기와 쓰기 경로를 다르게 해야함

단전
- 비일관성이 있음
> - 변경사항이 슬레이브에 전파되기 전에, 다른 클라이언트가 슬레이브에서 읽을 시 다른 값을 볼 수 있음
> - 이 문제의 해결법은 다음 블로그에서 다룰 예정 (ch5. 일관성_Consistency)

### 4. 피어-투-피어 복제

- 마스터 슬레이브 복제는 읽기 확장성에 도움이 되지만, 쓰기에는 도움이 되지 않음
- 슬레이브 실패에 대해 복원력을 제공하지만, 마스터에 대해 복원력을 제공하지 못함
- 즉, 마스터는 병족이고 SPOF 상태 

- 그래서 마스터를 두지 않는 구조 '피어-투-피어' 등장
> - 모든 노드에서 쓰기 요청을 처리하고, 노드 중 어느 것이 실패하더라도 데이터 저장소에 대한 접근이 중지되지 않음

단점
- 복잡함 = 일관성
- 두 사람이 동시에 동일한 레코드에 업데이트할 때, 쓰기 충돌 위험이 있고 이는 읽기에서 비일관성 문제가 발생할 수 있지만 
- 읽기 측면에서는 일시적 문제지만 쓰기 측면에서는 영구적인 문제가 될 수 있음 

쓰기 비일관성을 처리하는 방법
1. 트랜잭션 처리
2. 병합하는 정책

### 5. 샤딩과 복제의 결합

- 피어-투-피어 복제와 샤딩을 사용하는 것은 '칼럼 패밀리 데이터베이스'의 흔한 전략
- 복제 인수를 3으로 설정하는 등 