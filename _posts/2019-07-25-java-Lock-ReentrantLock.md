---
layout: post
title:  "Lock & ReentrantLock"
date: 2019-07-27 20:05:12
categories: JAVA
author : Jaesang Lim
tag: JAVA
cover: "/assets/instacode.png"
---

## Lock

멀티쓰레드, 동시성 프로그래밍에서는 가장 중요한 개념은 **Thread**와 **Lock** 

Lock
- 하나의 자원에 대해 여러 Thread가 동시에 접근하는 것을 도구
- Synchronized, 동기화하거나, 아니면 접근 자체를 직렬화한다고 표현함 
- 멀티쓰레드 환경에서는 여러 Thread가 Heap 메모리에 있는 객체나 자원을 접근할 때 동기화를 통해 접근에 제약을 두어야함
> - 여러 Thread가 어떤 순서로, 어떻게 동작할지 보장할 수 없기 때문에..

- 즉, Lock은 **공유하는 자원을 잠그는 자물쇠 역할**
> - 좀 이해를 쉽게하자면, 자원 접근하는데 필요한 관문, 문지기 역할임 
- 관문을 통과하지 못하면, 대기해야함 

Lock의 주요 2기능
1. 특정 조건에 따라 정확하게 지정한 수의 Thread만 자원에 접근해야함 
2. 접근을 못한 Thread를 **줄 세워**, 들어갈 수 있을 때, 다시 깨워서 통과하게 해줘야함 
> - 접근하지 못한 Thread는 sleep상태이며, sleep 상태인 Threadf 어떤 순서로 깨워서 (notifity) 해서 통과시켜줄 지에 대해 구현해야함


Lock의 2가지 락
- synchronized의 경우 암묵적 락
> - synchronized는 동기화를 하고자하는 블럭이나 메소도를 synchronized로 감싸서 락을 겁니다.
> - 어느 부분이 락인지 명확하지 않아서 암묵적인 락

- ReentrantLock은 명시적인 락
> - 다중 스레드가 공유하는 주요 컬랙션 대신 완전히 독립적인 락을 채택하여 구현하는 것

RenntrantLock VS  synchronized
- synchronized의 경우 Thread간의 lock을 획득하는 순서를 보장해주지 않음
> - 이 것을 unFair라고 하고, ReentrantLock은 Fair, unFair 모두 지원 
- 코드가 단일 블록의 형태를 넘어 여러가지 컬랙션이 얽혀 있을때 명시적으로 락을 실행시킬 수 있음
- 대기상태의 락에 대한 인터럽트를 걸어야 할 경우
- 락을 획득하려고 대기중인 스레드들의 상태를 받아야 할 경우에 쓸 수 있습니다.

ReentrantLock 쓸 경우
- 락을 모니터링 해야 할 때
- 락을 획득하려는 쓰레드의 개수가 많을 때(4개 이상)
- 위에서 이야기한 복잡한 동기화 코드를 작성해야 할 때

단점 ..?
- 기본 키워드인 synchronized과 달리 java.util.concurrent를 import해야 되고 
   try/fianlly block이 무조건 들어가기 때문에 코드가 지저분




---


java.utils.concurrent 패키지에는 Lock에 대한 인터페이스 구현체는 다음과 같이 있음

Interface
- Lock 
> - 1개의 Thread만 공유 자원을 접근할 수 있음

- ReadWriteLock 
> - 여러 Thread가 공유 자원을 접근할 수 있음

- Condition 

Class
- ReentrantLock 
- ReentrantReadWriteLock 

---


## ReentrantLock 클래스 구조 

Lock의 기능은 AbsractQueueSynchronizer를 상속받은 Sync 클래스
 > - 이 Sync 클래스를 상속받은 FairSync와 NonFairSync를 통해 수행

ReentrantLock의 전체적인 코드 구성 
```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;

    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        // 생략 ...
    }

    /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
      // 생략 ...
    }

    /**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
       // 생략 ...
    }


    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

```

Lock을 만들어야 할때마다 고려해야 사항
- Thread를 대기 
- 대기 시킨 Thread를 깨움
- 대기 Timeout을 처리해야 하는 
- 등등.. 
AbstractQueuedSynchronizer는 위의 고려사항을 모두 처리해 줌 


---

## ReentrantLock 클래스 메소드
 
[링크](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantLock.html)
Lock 인터페이스를 구현함
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();


- tryLock()
> - Lock을 얻기 위해 Thread가 대기 상태로 가지 않음
> - Lock를 선점하고 있는 Thread가 없을 때만 Lock을 얻어오고, 못가져오더라도 대기 상태로 가지 않음 
> - trylock()이라는 메서드 입니다. trylock은 락을 선점한 스레드가 없을 때만 락을 얻으려고 시도하는 메서드





