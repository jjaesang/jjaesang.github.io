---
layout: post
title:  "Kafka consumer Offset 체크 bash"
date: 2019-04-01 19:05:12
categories: Kafka
author : Jaesang Lim
tag: Spark
cover: "/assets/kafka_log.png"
---

- Nifi를 하면서 사용한 ConsumeKafka_1_0 프로세서를, 사용하면서 큰 실수를 했다.
- 해당 프로세서가 정상작동하겠구나.. 이렇게하면 되겠지.. 하고 제대로 확인하지 않았고, 의도한 설정값기반으로 정상적으로 데이터를 가져오지 않았다.
- 그래서 Kafka에서 데이터를 땡겨올 때, 꼭 정말 의도한 대로 데이터가 들어오고 있는지를 확인해야겠다..
- 그래서 간단하지만 모니터링하기 쉬운 CURRENT-OFFSET의 합을 계속 보여주는 bash를 정리하고자한다.

```
while true ; 
  do echo -e -n "$(date)\t"; ./kafka-consumer-groups.sh --bootstrap-server xxx.xxx.com:9092 
  --group consumer-group-id 
  --describe 2> /dev/null  
  | sed 's/\s\+/\t/g' | cut -f 3 | grep  '^[0-9]\+$' | jq -s add | grep -o '^[0-9]\+$'  ;  done
  
```

- --describe 2> /dev/null 
 > - 2> /dev/null 을 하지 않으면 'Note: This will not show information about old Zookeeper-based consumers.' 이 메세지가 계속 나오니 조심!

- cut -f 3 은 세번째 필드값을 가져오자 했다
 > 1. TOPIC
 > 2. PARTITION
 > 3. CURRENT-OFFSET
 > 4. LOG-END-OFFSET
 > 5. LAG
 > 6. CONSUMER-ID
 > 7. HOST
 > 8. CLIENT-ID

- lag에 대한 합을 구하고자 하면 그냥 'cut -f 5' 만 하면 끝

- grep  '^[0-9]\+$' 
> - 이것은 헤더값 , 위에 적은 총 8개의 헤더는 보고싶지 않아서 넣었다


- 이거 하기 전에, Nifi 프로세서를 모니러링하기 위한 jstat, jmap, jvmtop, jstatd, jvisualVM 등을 활용했는데, 이 것도 꼭 정리할 것이다. 
- 화이팅, 삽질하면서 아주 많은 것을 공부해보자..
