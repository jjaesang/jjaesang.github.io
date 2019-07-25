---
layout: post
title:  "BlockingQueue 인터페이스 정리"
date: 2019-07-25 20:05:12
categories: JAVA
author : Jaesang Lim
tag: JAVA
cover: "/assets/instacode.png"
---

해당 내용은 BlockingQueue 인터페이스 javadoc을 보고 작성함 
- [링크](https://docs.oracle.com/javase/8/docs/api/?java/util/concurrent/BlockingQueue.html)

## BlockingQueue 

java.util.concurrent 패키지에 있는 인터페이스로 Thread safe한 Queue로 다음과 같은 구현체가 있음
- ArrayBlockingQueue
- LinkedBlockingQueue
- PriorityBlockingQueue
- SynchronousQueue
- DelayQueue
- TransferQueue

여기서 각 구현체의 특성을 다 다루지는 않고, BlockingQueue 인터페이스에 대한 내용을 다룸
> - 궁극적으로 인터페이스에 정의된 함수이 BlockingQueue의 핵심기능이니깐.. 


## BlockingQueue 특징

1. null을 허용하지 않음, null값을 add/put/offer을 하면 NullPointerException 발생
> - null값은 poll 작업의 실패를 나타낼 때 사용함

2. capacity bound
> - BlockingQueue는 용량,capacity에 제한을 둘 수 있음 
> - capacity 제약을 두지 않으면 Integer.MAX_VALUE

3. Collection 인터페이스 지원 
> - BlockingQueue는 Producer-Consumer 패턴의 대기열 목적으로 디자인되었음
> - 하지만, 그래도 Collection 인터페이스를 지원해서, remove(x)와 같이 임의의 데이터를 Queue에서 지울 수 있음 
> - 근데 이 작업은 매우 비효율적이고, 대기중인 메세지가 취소되는 경우와 같이 가끔씩 사용하기 위해 만듬

4. Thread-safe하게 구현 
> - 모든 Queue 메소드들은 내부적으로 locks 또는 동시성 제어를 사용하여 원자성을 보장함 
> - 하지만 Bulk Collection Operations (addAll, containsAll, retainAll, removeAll 등)은 원자성을 보장하지 않음
> - 원자성을 보장하려면 따로 구현을 해야함
> - addAll(c)로 100개의 데이터를 queue에 넣을 때, 일부는 넣고 실패할 수 있음 (throwing Exception)


5. 더 이상 데이터가 추가되지 않는 것을 나타내기 위한 close,shutdown같은 작업을 지원하지 않음   
> - 목적에 맞춰서 구현해야함
> - 일반적으로 producer가 특수한 end-of-stream 또는 poison 객체를 삽입하고 , consumer가 판단하는 패턴으로 구현해야함



## BlockingQueue 메소드들

BlockingQueue는 총 4가지 방법을 지원함 

1. Throw exception
> - 해당 메소드가 즉시 가능하지 않을 경우, Exception을 발생시키도록 하는 방법

2. Special value
> - null이나 Boolean 값을 반환하는 방법

3. Blocks
> - 해당 메소드가 즉시 가능하지 않을 경우, 해당 작업을 수행할 때까지 현재의 Thread가 무한히 대기(block)

4. Time out
> - 주어진 시간 만큼만 block

각 구현된 메소들

![image](https://user-images.githubusercontent.com/12586821/61859757-b486b300-af03-11e9-9ad7-57f00107d003.png)

---

추가로 보면 좋을 것 같은 메소드들 

- int remainingCapacity()
- boolean contains(Object o)
- int drainTo(Collection<? super E> c, int maxElements)
- int drainTo(Collection<? super E> c)


```bash
 
    /**
     * Returns the number of additional elements that this queue can ideally
     * (in the absence of memory or resource constraints) accept without
     * blocking, or {@code Integer.MAX_VALUE} if there is no intrinsic
     * limit.
     *
     * <p>Note that you <em>cannot</em> always tell if an attempt to insert
     * an element will succeed by inspecting {@code remainingCapacity}
     * because it may be the case that another thread is about to
     * insert or remove an element.
     *
     * @return the remaining capacity
     */
    int remainingCapacity();
    
    /**
     * Returns {@code true} if this queue contains the specified element.
     * More formally, returns {@code true} if and only if this queue contains
     * at least one element {@code e} such that {@code o.equals(e)}.
     *
     * @param o object to be checked for containment in this queue
     * @return {@code true} if this queue contains the specified element
     * @throws ClassCastException if the class of the specified element
     *         is incompatible with this queue
     *         (<a href="../Collection.html#optional-restrictions">optional</a>)
     * @throws NullPointerException if the specified element is null
     *         (<a href="../Collection.html#optional-restrictions">optional</a>)
     */
    public boolean contains(Object o);
    

    /**
     * Removes all available elements from this queue and adds them
     * to the given collection.  This operation may be more
     * efficient than repeatedly polling this queue.  A failure
     * encountered while attempting to add elements to
     * collection {@code c} may result in elements being in neither,
     * either or both collections when the associated exception is
     * thrown.  Attempts to drain a queue to itself result in
     * {@code IllegalArgumentException}. Further, the behavior of
     * this operation is undefined if the specified collection is
     * modified while the operation is in progress.
     *
     * @param c the collection to transfer elements into
     * @return the number of elements transferred
     * @throws UnsupportedOperationException if addition of elements
     *         is not supported by the specified collection
     * @throws ClassCastException if the class of an element of this queue
     *         prevents it from being added to the specified collection
     * @throws NullPointerException if the specified collection is null
     * @throws IllegalArgumentException if the specified collection is this
     *         queue, or some property of an element of this queue prevents
     *         it from being added to the specified collection
     */
    int drainTo(Collection<? super E> c);


```

