---
layout: post
title:  "BlockingQueue 구현체 정리"
date: 2019-07-26 20:05:12
categories: JAVA
author : Jaesang Lim
tag: JAVA
cover: "/assets/instacode.png"
---

해당 내용은 BlockingQueue 인터페이스를 구현한 클래스에 대한 내용을 다루고자 함
> - 모두 Javadoc을 보고 작성

## BlockingQueue 구현 클래스들
- [각 구현체들은 인터페이스 javadoc에 다 있음](https://docs.oracle.com/javase/8/docs/api/?java/util/concurrent/BlockingQueue.html)

- LinkedBlockingQueue
- ArrayBlockingQueue
- PriorityBlockingQueue
- SynchronousQueue
- DelayQueue
- LinkedTransferQueue
- LinkedBlockingDeque


## LinkedBlockingQueue
[링크](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedBlockingQueue.html)

- FIFO 순서를 가지며, linked nodes을 기반
- Queue의 head는 오래된 데이터, queue에서 꺼낼 때는 head값을 가져옴
- Queue의 tail은 최신 데이터, queue에 넣을 때는 tail에 붙임 
- capacity를 지정할 수 있고, 하지 않으면 Interger.MAX_VALUE
- collections framework에 속하며, Collection과 Iterator 인터페이스를 가짐
> - iterator() return Iterator<E>
> - toArray(), return Object[]
**배열 기반 Queue보다 처리량이 높음 ( high Throughput)**
> - 하지만, 동시프로그램에서는 예측할 수 있는 성능이 낮다 (?) - 무슨말이지..
> - Linked queues typically have higher throughput than array-based queues but less predictable performance in most concurrent applications.

---

## ArrayBlockingQueue
[링크](https://docs.oracle.com/javase/8/docs/api/?java/util/concurrent/ArrayBlockingQueue.html)

- FIFO로 순서를 가지며, Array 기반
- LinkedBlockingQueue와 같은 기능을 제공하지만, Array 기반이기에 **당연히 capacity를 지정해야함**
> - 큐가 꽉 찬 상태에서 put하면 넣을 수 있을 때까지 Block 
> - 빈 큐에 take 해도 가져올 때까지 Block

- Fairness policy를 제공함 
> - Producer와 Consumer를 순서하기 위한 정책 제공
> - 기본적으로는 보장하지 않고, true로 설정하면 Thread의 접근을 FIFO 순서로 허용함
> - Throughput을 감소시키지만, 데이터의 변동성과 starvation을 피할 수 있음 
```java
    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and default access policy.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and the specified access policy.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
    
```

---

## PriorityBlockingQueue
[링크](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/PriorityBlockingQueue.html)

- PriorityQueue 클래스와 같은 순서롤 보장하는 BlockingQueue
- 범위 제한이 없어, 계속 추가하면 OutofMemeoryError 발생할 수 있음 
- **(중요)** iterator() 함수로 반환된 Iterator는 PriorityBlockingQueue의 순서를 보장하지 않음
> - 순서를 보장하기 위해서는 **Arrays.sort(pq.toArray())** , 이렇게 지정해야함
- **(중요)** 같은 우선순위를 가지는 element에 대해서는 순서를 보장하지 않음
> - 이를 위해서 custom class, comparator, secondary key를 구성해야함 
> - 
  ```java
   class FIFOEntry<E extends Comparable<? super E>>
       implements Comparable<FIFOEntry<E>> {
     static final AtomicLong seq = new AtomicLong(0);
     final long seqNum;
     final E entry;
     public FIFOEntry(E entry) {
       seqNum = seq.getAndIncrement();
       this.entry = entry;
     }
     public E getEntry() { return entry; }
     public int compareTo(FIFOEntry<E> other) {
       int res = entry.compareTo(other.entry);
       if (res == 0 && other.entry != this.entry)
         res = (seqNum < other.seqNum ? -1 : 1);
       return res;
     }
   }
  ```

- drainTo 메소드로 데이터를 삭제하고, 다른 collection을 넣을 수 있음
- 초기 capacity는 11로 되어있고 자료구조는 BalancedBinaryHeap을 사용함

```java
    /**
     * Default array capacity.
     */
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    
    /**
     * Priority queue represented as a balanced binary heap: the two
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * priority queue is ordered by comparator, or by the elements'
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * lowest value is in queue[0], assuming the queue is nonempty.
     */
    private transient Object[] queue;

    /**
     * Creates a {@code PriorityBlockingQueue} with the specified initial
     * capacity that orders its elements according to the specified
     * comparator.
     *
     * @param initialCapacity the initial capacity for this priority queue
     * @param  comparator the comparator that will be used to order this
     *         priority queue.  If {@code null}, the {@linkplain Comparable
     *         natural ordering} of the elements will be used.
     * @throws IllegalArgumentException if {@code initialCapacity} is less
     *         than 1
     */
    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }

```

---

## SynchronousQueue
[링크](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/SynchronousQueue.html)

- 1개만 가지는 큐..?
 

--- 
## DelayQueue
[링크](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/DelayQueue.html)

- Delay을 가진 큐로, 소스보니 PriorityQueue을 가짐


SynchronousQueue / DelayQueue는 추가예정 

- 그리고 모든 Queue을 보다보면, Thread-safe를 위해 Lock을 사용하는데, ReentrantLock을 사용함
> - ReentrantLock에 대해서도 다음 블로그에서 다 파헤칠 예정
