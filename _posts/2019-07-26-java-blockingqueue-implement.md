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
[각 구현체들은 인터페이스 javadoc에 다 있음](https://docs.oracle.com/javase/8/docs/api/?java/util/concurrent/BlockingQueue.html)

- LinkedBlockingQueue
- ArrayBlockingQueue
- PriorityBlockingQueue
- SynchronousQueue
- DelayQueue
- LinkedTransferQueue
- LinkedBlockingDeque

---

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

Item을 내부적으로 저장할 공간이 없기 때문에, 큐에서 Item을 가져가려는 Thread는 물론이고 Queue에 Item을 저장하려는 Thread도 상대방 Thread가 없을 때는 대기함
- Item을 저장하고 관리하는 리스트 같은 장치가 없음
- Item을 건네는 Thread와 Item을 가져가려는 Thread간에 Item을 전달시켜주는 랑데뷰 채널으로서 작동
 
SyhchronousQueue는 java.util.concurrent.Exchanger와 비슷해보이지만 차이가 있음
- Exchanger는 Thread의 종류에 상관없이 먼저 만나는 두 Thread간에 값을 교환(Exchange)시켜줌
- SynchronousQueue는 Item을 건네는 Thread와 Item을 가져가려는 Thread의 두 종류의 Thread간에 값을 전달(Transfer)하는 역할
- 즉, Exchanger는 교환자이고 SynchronousQueue는 전달자

- Item을 저장하고 관리하는 리스트 같은 장치는 존재하지 않지만, 대기하는 Thread를 관리함 
> - 즉, SynchronousQueue는 Item 리스트 대신에 대기 Thread 리스트를 유지하면서 Item을 건네려는 Thread와 Item을 가져가려는 Thread를 매칭시켜주는 역할을 함
 
SynchronousQueue는 내부에서 Transferer라는 이름의 객체를 사용해서 이 대기 Thread를 관리함
> - Stack기반으로 관리 (fair 정책을 true로 설정시 )
> - Queue기반으로 관리 

[출처] java.util.concurrent.SynchronousQueue 분석(1)|작성자 쫌조

```java
    /**
     * Creates a {@code SynchronousQueue} with the specified fairness policy.
     *
     * @param fair if true, waiting threads contend in FIFO order for
     *        access; otherwise the order is unspecified.
     */
    public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }
```

**SynchronousQueue는 대기 Thread 리스트를 관리**하는데.. Producer와 Consumer 두 종류의 Thread에 대한 2개의 대기 리스트를 관리해야할까?
- 결론적으로는 아님
- Thread가 대기하는 상황은 해당 Item에 대해 매칭될 상대방 Thread가 없는 경우
> - 즉, 같은 종류의 Thread가 모인 경우에만 대기 Thread 리스트가 생기고, 다른 종류의 Thread가 도착할 때, 대기중인 것 중 Stack / Queue 기반으로 매칭 시켜줌 


### TransferStack

대기 Thread를 Stack으로 구현할 때 문제가 있을 수 있음 ( Lock-Free로 구현되어 있음 )
> - Item을 가져오는 것과 건네오는 위치가 동일함
> - 즉, insert도 Stack에 Top에 하고, 꺼낼 때도 Stack에 Top에 함
> - Item을 꺼내려고 Stack에 요청했는데, 그 도중에 새로운 Item이 들어오면.. ? 
> - Lock-Free로 구현되어 있기 때문에 이것을 방지하기 위한 방법이 필요함 

- Item을 가져오는 request를 Request Mode
- Item을 건네는 request를 Data Mode

만약, Stack에 Request Mode가 쌓여있다고 가정해보고
- Stack에 같은 mode의 Thread 들만 쌓여있는 상태에서 다른 mode의 Thread가 도착하게 되면 **fulfill**이 일어남
> - 이 fulfill을 시도하는 Thread는 mode를 fulfill mode로 해서 Stack에 자신을 삽입
> - 즉, 자신을 fulfill mode로 해서 Stack에 쌓아두고 바로 아래의 Thread와 fulfill 동작을 수행

- fulfill이 수행되는 동안에는 Top에는 fulfill mode의 Thread가 있는 상태가 되고 새로운 Thread가 SynchronousQueue에 도착했을 때도 Top에는 fulfill mode의 Thread가 존재하는 것을 확인하므로
 Thread는 자신을 Stack에 삽입하는 동작을 보류함
- 그리고 fulfill이 끝나면 자신을 포함해서 바로 아래에 있는 fulfill을 수행한 상대방 Thread까지 같이 Stack에서 제거하는 

TransferStack의 transfer 메소드

```java

        /**
         * Puts or takes an item.
         */
        @SuppressWarnings("unchecked")
        E transfer(E e, boolean timed, long nanos) {
            /*
             * Basic algorithm is to loop trying one of three actions:
             *
             * 1. If apparently empty or already containing nodes of same
             *    mode, try to push node on stack and wait for a match,
             *    returning it, or null if cancelled.
             *
             * 2. If apparently containing node of complementary mode,
             *    try to push a fulfilling node on to stack, match
             *    with corresponding waiting node, pop both from
             *    stack, and return matched item. The matching or
             *    unlinking might not actually be necessary because of
             *    other threads performing action 3:
             *
             * 3. If top of stack already holds another fulfilling node,
             *    help it out by doing its match and/or pop
             *    operations, and then continue. The code for helping
             *    is essentially the same as for fulfilling, except
             *    that it doesn't return the item.
             */

            SNode s = null; // constructed/reused as needed
            int mode = (e == null) ? REQUEST : DATA;

            for (;;) {
                SNode h = head;
                if (h == null || h.mode == mode) {  // empty or same-mode
                    if (timed && nanos <= 0) {      // can't wait
                        if (h != null && h.isCancelled())
                            casHead(h, h.next);     // pop cancelled node
                        else
                            return null;
                    } else if (casHead(h, s = snode(s, e, h, mode))) {
                        SNode m = awaitFulfill(s, timed, nanos);
                        if (m == s) {               // wait was cancelled
                            clean(s);
                            return null;
                        }
                        if ((h = head) != null && h.next == s)
                            casHead(h, s.next);     // help s's fulfiller
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    }
                } else if (!isFulfilling(h.mode)) { // try to fulfill
                    if (h.isCancelled())            // already cancelled
                        casHead(h, h.next);         // pop and retry
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                        for (;;) { // loop until matched or waiters disappear
                            SNode m = s.next;       // m is s's match
                            if (m == null) {        // all waiters are gone
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                break;              // restart main loop
                            }
                            SNode mn = m.next;
                            if (m.tryMatch(s)) {
                                casHead(s, mn);     // pop both s and m
                                return (E) ((mode == REQUEST) ? m.item : s.item);
                            } else                  // lost match
                                s.casNext(m, mn);   // help unlink
                        }
                    }
                } else {                            // help a fulfiller
                    SNode m = h.next;               // m is h's match
                    if (m == null)                  // waiter is gone
                        casHead(h, null);           // pop fulfilling node
                    else {
                        SNode mn = m.next;
                        if (m.tryMatch(h))          // help match
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }
```


### TransferQueue

TransferQueue는 오히려 TransferStack보다 간단함
 > - Queue는 Item의 삽입과 추출이 반대편에서 일어나므로 삽입과 추출 동작이 서로 간섭을 하지 않음
 
TransferQueue에 도착한 Thread는 tail에 있는 노드를 살펴보고 자신과 같은 mode이면 tail쪽에 삽입을 하고, tail이 자신과 다른 mode이면 head에 있는 Node와 fulfill을 시도함

이때 fulfill을 하는 Thread는 자신을 Queue에 삽입하거나 하지 않음
 > -  fulfill을 하는 동안에 Queue의 tail쪽에 새로운 노드가 삽입이 되더라도 문제가 없기 때문
 
그런데.. 만약 한 Thread가 fulfill을 하는 동안에 역시나 다른 Thread가 도착해서 같은 과정으로 tail을 확인한 후에 head에서 fulfill을 시도한다면..?
- 즉 fulfill에 대한 경합이 발생할 수 있음
> - 이 경우에는 head에 있는 노드의 item 필드에 대해 CAS(compare and swap)연산을 통해 fulfill을 수행 CAS연산에 성공하는 Thread는 하나 뿐임을 보장함 
> - CAS연산에 실패한 Thread는 다시 큐에서 꺼내는 작업을 시작함
 
TransferStack과 다르게 조건이 2개뿐!

```java
 
    /**
          * Puts or takes an item.
          */
         @SuppressWarnings("unchecked")
         E transfer(E e, boolean timed, long nanos) {
             /* Basic algorithm is to loop trying to take either of
              * two actions:
              *
              * 1. If queue apparently empty or holding same-mode nodes,
              *    try to add node to queue of waiters, wait to be
              *    fulfilled (or cancelled) and return matching item.
              *
              * 2. If queue apparently contains waiting items, and this
              *    call is of complementary mode, try to fulfill by CAS'ing
              *    item field of waiting node and dequeuing it, and then
              *    returning matching item.
              *
              * In each case, along the way, check for and try to help
              * advance head and tail on behalf of other stalled/slow
              * threads.
              *
              * The loop starts off with a null check guarding against
              * seeing uninitialized head or tail values. This never
              * happens in current SynchronousQueue, but could if
              * callers held non-volatile/final ref to the
              * transferer. The check is here anyway because it places
              * null checks at top of loop, which is usually faster
              * than having them implicitly interspersed.
              */
 
             QNode s = null; // constructed/reused as needed
             boolean isData = (e != null);
 
             for (;;) {
                 QNode t = tail;
                 QNode h = head;
                 if (t == null || h == null)         // saw uninitialized value
                     continue;                       // spin
 
                 if (h == t || t.isData == isData) { // empty or same-mode
                     QNode tn = t.next;
                     if (t != tail)                  // inconsistent read
                         continue;
                     if (tn != null) {               // lagging tail
                         advanceTail(t, tn);
                         continue;
                     }
                     if (timed && nanos <= 0)        // can't wait
                         return null;
                     if (s == null)
                         s = new QNode(e, isData);
                     if (!t.casNext(null, s))        // failed to link in
                         continue;
 
                     advanceTail(t, s);              // swing tail and wait
                     Object x = awaitFulfill(s, e, timed, nanos);
                     if (x == s) {                   // wait was cancelled
                         clean(t, s);
                         return null;
                     }
 
                     if (!s.isOffList()) {           // not already unlinked
                         advanceHead(t, s);          // unlink if head
                         if (x != null)              // and forget fields
                             s.item = s;
                         s.waiter = null;
                     }
                     return (x != null) ? (E)x : e;
 
                 } else {                            // complementary-mode
                     QNode m = h.next;               // node to fulfill
                     if (t != tail || m == null || h != head)
                         continue;                   // inconsistent read
 
                     Object x = m.item;
                     if (isData == (x != null) ||    // m already fulfilled
                         x == m ||                   // m cancelled
                         !m.casItem(x, e)) {         // lost CAS
                         advanceHead(h, m);          // dequeue and retry
                         continue;
                     }
 
                     advanceHead(h, m);              // successfully fulfilled
                     LockSupport.unpark(m.waiter);
                     return (x != null) ? (E)x : e;
                 }
             }
         }
```
 
 
--- 

## DelayQueue
[링크](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/DelayQueue.html)

- Delay을 가진 큐로, Item관리는 PriorityQueue로 진행
- Delay가 expired된 Item만 return 받을 수 있음
> - Queue의 head는 Delay가 가장 오랜 지난 Item

- size 메소드는 expired 된 요소와 expired되지 않은 요소의 수를 반환

```java
  /**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element with an expired delay is available on this queue,
     * or the specified wait time expires.
     *
     * @return the head of this queue, or {@code null} if the
     *         specified waiting time elapses before an element with
     *         an expired delay becomes available
     * @throws InterruptedException {@inheritDoc}
     */
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null) {
                    if (nanos <= 0)
                        return null;
                    else
                        nanos = available.awaitNanos(nanos);
                } else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    if (nanos <= 0)
                        return null;
                    first = null; // don't retain ref while waiting
                    if (nanos < delay || leader != null)
                        nanos = available.awaitNanos(nanos);
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            long timeLeft = available.awaitNanos(delay);
                            nanos -= delay - timeLeft;
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }

```

---
