---
layout: post
title:  "[NiFi] NiFi을 이용한 Database 연동 "
date: 2019-03-06 23:15:12
categories: NiFi 
author : Jaesang Lim
tag: NiFi
cover: "/assets/instacode.png"
---

- NiFi 내에서 처리된 데이터를 Kafka, Hdfs, Local FileSystem, DB , Elasticsearch 등 다양한 sink에 넣어야하는 케이스가 있음
- Elasticsearch 는 PutElasticsearchHttpRecord 프로세스 하나로 충분히 가능함
- 하지만 DB에 대해서는 Controller Service를 이용하여 DB Connection Pool를 관리하고, 이와 관련된 프로세서도 몇 개 있음
- 그 중 3개에 대한 프로세서를 정리하고자함.

### Database Connection Pooling Service
- DBCPConnectionPool의 Controller serivce를 설정해야함 ( PutSQL, ExecuteSQL에서 꼭 해줘야함 )
- JDBC 접근할 때와 마찬가지로 
  1. Database Connection URL	
    - e.g) jdbc:phoenix:${zk_quorum}:2181/hbase-unsecure
  2. Database Driver Class Name	
    - e.g) org.apache.phoenix.jdbc.PhoenixDriver
  3. Database Driver Location(s)	
    - e.g) file:///usr/hdp/current/phoenix/phoenix-client.jar
    
- 등등 설정할 수 있음

### ExecuteSQL
- select query를 하기 위한 프로세서
- query 결과는 Avro 포맷으로 return 
- [ExecuteSQL](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.5.0/org.apache.nifi.processors.standard.ExecuteSQL/index.html)

### PutSQL
- insert, update query를 하기위한 프로세서
- flowfile의 content에 실제 실행할 query문이 있어야함
  - 여기서 중요한건 query문 끝에 ; 넣으면 오류난다. 빼야함 
- flowfile의 query 문에 preparedStatement 처럼 ? 을 통한 query문을 적고 들어갈 값들에 대해서는 attributes로 설정가능
- preparedStatement 처럼 ? 설정시, attribute에는 sql.args.N.type / sql.args.N.value 값이 꼭 있어야함!
- [PutSQL](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.6.0/org.apache.nifi.processors.standard.PutSQL)

### ConvertJSONToSQL
- JSON 데이터을 Statement Type( Update, Insert, Delete)의 SQL문으로 변경시켜주는 프로세서
- JSON 데이터는 Nested JSON일 경우, 그냥 String으로 취급하기에 Flat한 JSON 권장
- [ConvertJSONToSQL](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.6.0/org.apache.nifi.processors.standard.ConvertJSONToSQL/index.html)
 
--- 
- 사실상, 내가 다루는 데이터도 JSON 데이터이기에 ConvertJSONToSQL로 데이터 변경 후 , PutSQL로 사용하려했음
- 하지만 Phoenix 자체는 insert 대신 upsert문을 사용하기에 ConvertJSONToSQL 한 후에도, ReplaceText로 insert를 upsert로 바꿔야하는 상황
- 그래서 아예 ReplaceText를 사용하여 flowfile의 content에 query statement를 적용했음

1) Replacement Value	
  - upsert into table_name values ('${'id'}','${'time''}',..,'${value}' )
  - attribute에 있는 값들을 넣어줌

2) Replacement Strategy	
  - Always Replace

3) Evaluation Mode	
  - Entire text	
  - 하나의 flowfile은 하나의 record로 분리해서 사용해서 line-by-line 대신 entire text로 사용
