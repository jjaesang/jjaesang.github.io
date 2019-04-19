---
layout: post
title:  "[ELASTIC SEARCH] Too many open file Error & Index read-only problem "
date: 2019-04-18 10:15:12
categories: elasticsearch 
author : Jaesang Lim
tag: elasticsearch
cover: "/assets/instacode.png"
---

## Too many open file Error

#### 1. nofile과 fs.file-max 값을 확인하고 수정하자
#### 2. 프로세서의 open file descriptor는 'lsof -p ${PID}'로 확인할 수 있다
#### 3. 과거의 사용하지않은 index는 close 하자

#### 상황 정리
- Elasticsearch 클러스터 3대로 운영하던 중 1대가 'Too many open files'로 ssh 접속이 안되는 상황이 발생하였다..
> - 한대만.. 

- 클러스터의 index 개수는 약 1500개 정도 이고.. 
- 실제로 'lsof -p ${ES_PID}'로 서버의 Open File 수를 보면, 하나의 index에 대해서 Lucene 인덱스을 위한 약 10개이상의 file open하고 있다.
```bash
 /data/nodes/0/indices/${INDEX_UUID}/0/index/_1.cfs
 /data/nodes/0/indices/${INDEX_UUID}/0/index/_mm_Lucene8_9.tim
 /data/nodes/0/indices/${INDEX_UUID}/0/index/_mm_Lucene8_9.doc
 /data/nodes/0/indices/${INDEX_UUID}/0/index/_mm_Lucene8_9.dvd
 .... 
 등등 이런 file descriptor들이 있는 것을 확인할 수 있다.
```


- 이렇게 많은 file descriptor을 오픈하기 때문에 Elasticsearch의 'Important System Configuration' 에서는 다음과 같이 설정해야한다고 나와있다 (Must로!)
> - ulimit -n 65535 
> - nofile 65535 
> - nofile은 soft와 hard로 나눠서 설정이 가능한데 root 계정과 non-root 계정에 대한 프로세서가 최대로 띄울 수 있는 file 개수이다.


- 사실 위의 설정은 이미 처음 클러스터 구축할 때 했는데.. 왜 한대만 그런가 봤더니만
- 서버 자체의 최대 open할 수 있는 file 개수 설정값 ( /proc/sys/fs/file-max) 이 다른 서버에 비해 작게 설정되어 있었다.
> - 사실상, nofile과 같이 65535로 세팅되어도 되지만 과거의 정리하지 않은 index을 다시 메모리에 올려 색인하는 과정에서 설정값 이상의 file open이 발생한 것 같다.

- nofile은 processor 단위의 최대 openfile수 이고, fs.file-max는 kernel 레벨의 최대 openfile수라고 한다.
> - [ulimit-vs-file-max](https://unix.stackexchange.com/questions/447583/ulimit-vs-file-max)

- 일단 'Too many open files' 해결하기 위해 시스템 세팅 (/proc/sys/fs/file-max) 값을 수정하였고
- 사용하지 않은 index에 대해 close를 통해 색인을 하지않도록 설정하였다.
- close는 index의 metadata는 유지하는 것을 제외하고, 클러스터의 overhead가 없으며, read/write가 blocking된다.
- 과거의, 사용하지 않은 index을 위한 file open 수가 줄어드는 것을 'lsof -p ${ES_PID}'로 확인하였다

---

## [FORBIDDEN/12/index read-only / allow delete (api)] ERROR

### index의 setting값에 "index.blocks.read_only_allow_delete":"false" 로 설정
### 상황 정리

- 위의 오류를 찾아보면, 서버의 data dir의 용량이 부족한지 확인해보라고 하는데, 사실상 내 상황에서는 용량이 가득차지 않았다
> - 가득찼다면 시스템 알람을 받았을 거고, 실제로도 매일 df -h로 확인할 때, 넉넉했으니깐
- 내 생각에는 too many open file에러로 elasticseach 프로세서가 비정상적으로 내려가면서 작업중이던 일부 index를 read/write block 해놓은 것 같다.
- 일단 index의 settings값에 blocks/read_only_allow_delete 필드가 있으면 해당 인덱스에 read/write 작업이 안된다.
> - 정상적인 index는 아예 blocks/read_only_allow_delete 필드가 없음

- read/write를 하기 위해서는 다음과 같은 명령어로 settings의 값으 변경해준다
```bash
curl -XPUT -H "Content-Type: application/json" http://xx.xx.xxx.xxx:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete":"false"}'
```

- _all은 모든 index에 대해 적용하는 것이고, 일부 index만 실행하려면 _all 위치에 index이름을 넣어주면 된다.
- 에러나면 왜 에러났는지 안보일 수 도 있기때문에 나는 왠만하면 curl 마지막에 -v로 verbose모드로 모든 로그를 확인한다.

- _all로 했을 때 'process_cluster_event_timeout_exception'가 날 수 있다.
- 에러의 원인은 마스터가 클러스터의 상태값을 업데이트하는 과정에서의 부하가 있어서 그렇다고 한다.
- 이럴 때는 master_timeout 값을 늘려준다
 
```bash
curl -XDELETE http://xx.xx.xxx.xxx:9200/index?master_timeout=60s
```


