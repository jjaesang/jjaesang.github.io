---
layout: post
title:  "BlockingQueue VS ConcurrentLinkedQueue"
date: 2019-07-22 20:05:12
categories: JAVA
author : Jaesang Lim
tag: JAVA
cover: "/assets/instacode.png"
---


# BlockingQueue VS ConcurrentLinkedQueue

[논문-Simple, fast, and practical non-blocking and blocking concurrent queue algorithms](https://dl.acm.org/citation.cfm?doid=248052.248106)

java.util.concurrent 패키지에 Queue에 대한 클래스 중 Thread Safe한 클래스가 있는데, 왜 BlockingQueue 중 BlockingLinkedQueue를 사용했는지에 대해 정리

ConcurrentLinkedQueue
 - non-blocking lock-free queues.
 
BlockingQueue
 - Blocking & Lock Queue
 
 
일단 Blocking을 선택한 이유?




---



LinkedBlockingQueue는
> - 생성자의 인자에 큐의 용량 capacity 를 명시하여 사이즈를 지정할 수 있음

ConcurrentLinkedQueue는 
> - 큐의 사이즈를 지정할 수 없을 뿐만 아니라 size 메서드는 상수 시간에 호출되지 않아서 큐에 들어있는 원소의 개수를 파악하는 것이 어려움

 ConcurrentLinkedQueue는 큐에 꺼낼 원소가 없다면 즉시 리턴하고 다른 일을 수행하러 간다. 
 따라서, ConcurrentLinkedQueue는 생산자-소비자 producer-consumer 모델에서 소비자가 많고 생산자가 하나인 경우에 사용하면 좋다.

여러 개의 쓰레드에서 하나의 Queue 객체에 들어있는 데이터를 꺼내기 위해 queue.poll() 메서드를 호출할 경우, 동일한 실행 결과가 나타날 수 있다. 
예를 들어, Queue에 [1, 2, 3]과 같은 데이터가 들어있을 경우, 스레드 3개가 critical section에서 poll() 메서드를 호출하면 각 스레드들은 모두 1이라는 데이터를 가져오고 Queue에는 [2, 3]만 남게 된다. 
큐가 비어있을 경우 null을 리턴한다.


LinkedBlockingQueue 은 이름에서도 알 수 있듯이 각각의 블로킹 큐가 링크드 노드로 연결된 큐이다. 
큐에서 꺼내갈 원소가 없을 경우 해당 쓰레드는 wait 상태에 들어간다.
 따라서, LinkedBlockingQueue는 생산자가 많고 하나의 소비자일 경우에 사용하면 좋다.
  또한 이 글의 서두에서 언급한 것처럼, LinkedBlockingQueue은 큐의 폭발을 막기 위해 생성자에 큐의 사이즈를 명시할 수 있도록 설계되었다.
LinkedBlockingQueue 내에 있는 데이터를 가져오기 retrieve 위해 poll()과 take() 메소드를 제공한다. 
이 두 메소드의 차이점은 큐가 비어있을 때, poll 메소드는 null을 리턴하거나 Timeout을 설정할 수 있는 반면에, take 메소드는 꺼낼 수 있는 원소가 있을 때까지 기다린다(waiting).


https://stackoverflow.com/questions/1426754/linkedblockingqueue-vs-concurrentlinkedqueue


 락(lock) 메커니즘은 Mutex(mutual-exclusion) 입니다.
기본적인 Mutex의 사용방법은 공유 자원에 접근하기 전에 락을 걸고(lock), 공유 자원의 사용을 마치면 락을 풀어주는(unlock) 것입니다

Thread 가 Critical Section에 접근할때 (또는 processer로 부터 resource를 할당받을때) 해당 Thread는 세마포어의 카운트를 감소시키고 수행이 종료된 후엔 세마포어의 카운트를 원래대로 증가시킨다.
다시말하면 세마포어는 일종의 신호등의 역할을 하는것이다.

세마포어는 한정된 수의 사용자만을 지원할 수 있는 공유 자원에 대한 접근을 통제하는데 유용하다. MFC에서는 지금까지 알아본 Critical Section, Mutex, Semaphore에 대한 클래스를 제공해 준다. 이 클래스들은 모두 CSyncObject에서 파생되었으며, 따라서 사용법도 비슷하다.


세마포어(Semaphore) : 공유된 자원의 데이터를 여러 프로세스가 접근하는 것을 막는 것

뮤텍스(Mutex) : 공유된 자원의 데이터를 여러 쓰레드가 접근하는 것을 막는 것



** 뮤텍스란(Mutex)? **

“Mutual Exclusion 으로 상호배제라고도 한다. Critical Section을 가진 쓰레드들의 Runnig Time이 서로 겹치지 않게 각각 단독으로 실행되게 하는 기술입니다. 다중 프로세스들의 공유 리소스에 대한 접근을 조율하기 위해 locking과 unlocking을 사용합니다. 

즉, 쉽게 말하면 뮤텍스 객체를 두 쓰레드가 동시에 사용할 수 없다는 의미입니다.

 

** 세마포어란?(Semaphore) **

” 세마포어는 리소스의 상태를 나타내는 간단한 카운터로 생각할 수 있습니다. 일반적으로 비교적 긴 시간을 확보하는 리소스에 대해 이용하게 되며, 유닉스 시스템의 프로그래밍에서 세마포어는 운영체제의 리소스를 경쟁적으로 사용하는 다중 프로세스에서 행동을 조정하거나 또는 동기화 시키는 기술입니다. 

세마포어는 운영체제 또는 커널의 한 지정된 저장장치 내 값으로서, 각 프로세스는 이를 확인하고 변경할 수 있습니다. 확인되는 세마포어의 값에 따라, 그 프로세스가 즉시 자원을 사용할 수 있거나, 또는 이미 다른 프로세스에 의해 사용 중이라는 사실을 알게 되면 재시도하기 전에 일정 시간을 기다려야만 합니다. 세마포어는 이진수 (0 또는 1)를 사용하거나, 또는 추가적인 값을 가질 수도 있습니다. 세마포어를 사용하는 프로세스는 그 값을 확인하고, 자원을 사용하는 동안에는 그 값을 변경함으로써 다른 세마포어 사용자들이 기다리도록 해야합니다.

 

( 차이점들!! )

<Mutex vs Semaphore>

1) Semaphore는 Mutex가 될 수 있지만 Mutex는 Semaphore가 될 수 없습니다.

(Mutex 는 상태가 0, 1 두 개 뿐인 binary Semaphore)

2) Semaphore는 소유할 수 없는 반면, Mutex는 소유가 가능하며 소유주가 이에 대한 책임을 집니다. (Mutex 의 경우 상태가 두개 뿐인 lock 이므로 lock 을 ‘가질’ 수 있습니다.)

3) Mutex의 경우 Mutex를 소유하고 있는 쓰레드가 이 Mutex를 해제할 수 있습니다. 하지만 Semaphore의 경우 이러한 Semaphore를 소유하지 않는 쓰레드가 Semaphore를 해제할 수 있습니다.

4) Semaphore는 시스템 범위에 걸쳐있고 파일시스템상의 파일 형태로 존재합니다. 반면 Mutex는 프로세스 범위를 가지며 프로세스가 종료될 때 자동으로 Clean up된다.



출처: https://jwprogramming.tistory.com/13 [개발자를 꿈꾸는 프로그래머]
