---
layout: post
title:  "Exactly Once Semantics in Apache Kafka"
date: 2019-07-08 18:05:12
categories: Kafka
author : Jaesang Lim
tag: Kafka
cover: "/assets/kafka_log.png"
---

## Introducing Exactly Once Semantics in Apache Kafka

우연치 않게 카프카 서버 점검도 할겸, 이리저리 찾아보다가 아주 재밌는 글이 있어서 정리하고자 함

지금은 Exactly Once semantics을 고민해서 문제를 풀만한 사항이 없어서..
At least once semantics로 중복은 sink단에서 또는 애플리케이션 로직 상에서 처리하고 있음 ㅠㅠ

기회가 된다면 Exactly Once semantic으로 문제를 풀어내고, 거기서 느끼고 경험한 것을 제대로 정리해보고 싶다!

관련 참고 자료
[Introducing Exactly Once Semantics in Apache Kafka](https://www.confluent.io/kafka-summit-nyc17/introducing-exactly-once-semantics-in-apache-kafka/)
[Exactly once Semantics are Possible: Here’s How Kafka Does it](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)
[Transactions in Apache Kafka](https://www.confluent.io/blog/transactions-apache-kafka/)


### At least once in order delivery per partition
- 기본적으로 producer/consumer를 사용하면 At least once 로 데이터 중복이 발생할 수 있음
> - producer가 ack를 못받거나, 네트워크 이슈 등으로 인해 데이터가 정상적으로 들어가지 않았다고 판단하면, 동일 데이터를 retry해서 중복 발생 
- Kafka는 기본적으로 동일 파티션 내에서는 메세지의 순서를 보장함 !

![image](https://user-images.githubusercontent.com/12586821/60793840-72d7d780-a1a3-11e9-8d09-20ca26b45ac0.png)

위의 그림이 Existing Semantics로 At least once, 데이터 중복이 발생하는 케이스임
> - 저 그림은 최종 데이터가 중복이 발생했다는 뜻이고, 왜 저렇게 발생했는지 알아보면

1. Producer는 send("x","y")을 leader partition에 요청함
2. leader는 받은 메세지("x","y")를 log에 기록함
3. producer에게 전달완료라고 ack을 보냄
> - 여기서 ack보내는 것은 당연히 옵션에 따라 다르다 (all, 1 등등)
> - 여기까지가 우아한 상황
4. Producer는 다른 메세지 send("a","b")을 leader partition에 요청함
5. leader는 받은 메세지("a","b")를 log에 기록함
6. ack을 못보냄.. 어떤 한 이유로 인해..
7. Producer는 **다시** 메세지 send("a","b")을 leader partition에 요청함 ( Retry 설정에 따라서 !)
8. leader는 받은 메세지("a","b")를 log에 기록함
9. producer에게 전달완료라고 ack을 보냄

응..? 데이터 중복이 발생

이 상황이 At least once semantic으로 발표자료에는 Apache Kafka's existing semantics라고 정의함

어떻게 극복하나!! 

---

### 1. Exactly once in order delivery per partition -> idempotent producer

위의 참고자료에서는 idempotent operation을 다음과 같이 정의함
 - An idempotent operation is one which can be performed many times without causing a different effect than only being performed once.
 - 즉, 한번만 보내는 것을 의미하는 것이 아닌, 여러번 수행해도 다른 효과를 발생시키지 않는 연산을 의미함
 - Kafka에 비유하면 producer가 retry에 의해 같은 메세지를 여러번 보내더라도, kafka log에는 한번만 기록된다는 뜻이다

그러면 어떻게 idempotent producer을 구현했는가? .. 
> - 허무하지만 send할 때, 해당 메세지에 대해 seqNo만 부여해서 처리했음.. 
> - 참고자료에 자세히 나오지만, seqNo만 추가한거라서, 사실상 오버헤드는 거의 없다고함 

자세히 알아보자

![image](https://user-images.githubusercontent.com/12586821/60794539-f47c3500-a1a4-11e9-966a-4393a768c0ce.png)



1. Producer는 send("a","b")을 leader partition에 요청할때, ProducerID(PID=100)와 메세지번호(seqNo=0)를 같이 전송함
2. leader는 받은 메세지("a","b")와 PID, SeqNo를 log에 기록함
3. producer에게 전달완료라고 ack을 보냄
> - 여기까지가 우아한 상황
4. Producer는 다른 메세지 send("x","y")을 leader partition에 요청할때, ProducerID(PID=100)와 메세지번호(seqNo=1)를 같이 전송함
5. leader는 받은 메세지("x","y")PID, SeqNo를 log에 기록함
6. ack을 못보냄.. 어떤 한 이유로 인해..
7. Producer는 **다시** 메세지 send("x","y")을 leader partition에 요청할때, ProducerID(PID=100)와 메세지번호(seqNo=1)를 같이 전송함
8. leader는 log에 기록하지 않고 producer에게 **ack-depulicate**로 반환해, 메세지를 저장하지 않음 
> - seqNo가 1인 메세지는 이미 기록되어있으니깐!

---
### 2. Multi Partition writes -> transaction

producer는 transaction 정보를 transaction coordinator에 요청해서 transaction 처리를 함~!
transaction coordinator와 transaction log는 transaction의 state를 관리

![image](https://user-images.githubusercontent.com/12586821/60795704-15458a00-a1a7-11e9-8221-53ecea3cf79b.png)
- producer에서 transaction 처리하는 로직 (매우 간단)

![image](https://user-images.githubusercontent.com/12586821/60795752-373f0c80-a1a7-11e9-8670-15761eaf0d9a.png)
- initTransactions으로 Coordinator에게 해당 작업의 transactionId를 설정함 


![image](https://user-images.githubusercontent.com/12586821/60795763-3d34ed80-a1a7-11e9-9060-6acf288e2170.png)
- beingTransaction 과 send 메소드로 해당 transaction에 저장할 파티션 리스트를 transaction log에 저장

![image](https://user-images.githubusercontent.com/12586821/60795772-432ace80-a1a7-11e9-9812-b8e107563002.png)
- 메세지를 send함! 

![image](https://user-images.githubusercontent.com/12586821/60795782-48881900-a1a7-11e9-8a20-981c0de00bdf.png)
- commitTransaction()으로 해당 transaction을 commit함
- transaction log에는 transaction Id (Tid)을 Prepare 상태로 변경 

![image](https://user-images.githubusercontent.com/12586821/60795836-5ccc1600-a1a7-11e9-9a30-744b5ec3a378.png)
- prepare이 되면, coordinator는 data log에 commit된 정보를 marking 함
- 그 다음 coordinator는 producer에게 SUCCESS 했다고 ack을 보냄

![image](https://user-images.githubusercontent.com/12586821/60795884-72d9d680-a1a7-11e9-8df9-afcc4ce8d14c.png)
- consumer는 isolation_level 설정값인 read_committed 일때, Commit된 메세지만 읽음!

---

### 3. Performance Boost 

- Up to +20% producer throughput
- Up to +50% consumer throughput
- Up to -20% disk utilization.. 
- batch로 전송시, 더 줄일 수 있음

**record랑 batch record format 바뀐 것 꼭 추가할 것**
---

### Configuration

#### Producer
enable.idempotence = true
> - max.infilight.request.per.connection =1 
> - acks = all
> - retires > 1 (preferably MAX_INT)

transaction.id = '유니크한거'
> - enable.idempotence = true

#### Consumer 
isolation.level = read_committed or read_uncommitted 

