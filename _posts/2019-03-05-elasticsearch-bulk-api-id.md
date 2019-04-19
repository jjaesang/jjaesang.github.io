---
layout: post
title:  "[ELASTIC SEARCH] bulk api 사용시 고유id값 설정 "
date: 2019-03-05 12:24:12
categories: elasticsearch 
author : Jaesang Lim
tag: elasticsearch
cover: "/assets/instacode.png"
---

### Elasticsearch bulk api 사용시 고유 id 설정

- filebeat와 logstash로 처리하던 프로세스의 불안정성과 비효율성이 있어, 로그를 직접 파싱하고 Bulk API를 통해 색인을 진행하려고 한다.
- 이 과정에서 index의 _id 필드가 중복의 발생할 경우, 제대로 데이터가 입력되지 않은 케이스가 있는 것을 발견
> - 같은 인덱스, 같은 데이터에 대한 id 값을 주었을 때, 처음은 문서수가 정확하게 색인되는 것을 확인
> - 같은 일을 반복했을 시에는 docs.count는 동일하고, doc.delete만 늘어가는 것을 확인.
> - 이 이유에 대해서는 나중에 한번 정리해야겠다. 

- logstash나, elasticsearch의 default _id 생성법을 찾고 임의의 값을 넣어주고 색인해본 결과, 정상적으로 모든 문서가 색인되는 것을 확인하였다.
- 기본 _id 생성 알고리즘
  - Auto-generated ids are random base64-encoded UUIDs. 
  - The base64 algorithm is used in URL-safe mode hence - and _ characters might be present in ids.
  
### 결론 : UUID 설정해서 _id값에 넣자 ! 

---

#### 문제가 되었던 코드

- 사실상, count를 증분해서 _id를 설정하면 되지만, 나는 멀티 프로세스 환경에서 실행해야하고, 그렇게 되면 문서수의 count는 중복될 가능성이 매우크다.

```java
    BulkRequest request = new BulkRequest();
    request.timeout(TimeValue.timeValueMinutes(2));

    int doc_id =0;
    for (String json : jsons) {
        String _id = UUID.randomUUID().toString();
        request.add(new IndexRequest(index, "doc", String.valueOf(doc_id))
                .source(json, XContentType.JSON));
        doc_id++;
    }
    BulkResponse bulkResponse = client.bulk(request);
```


#### count 대신 UUID 사용


```java
    BulkRequest request = new BulkRequest();
    request.timeout(TimeValue.timeValueMinutes(2));

    for (String json : jsons) {
        String _id = UUID.randomUUID().toString();
        request.add(new IndexRequest(index, "doc", _id)
                .source(json, XContentType.JSON));
    }
    BulkResponse bulkResponse = client.bulk(request);
```


