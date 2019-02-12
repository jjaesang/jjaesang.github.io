---
layout: post
title:  " WebHDFS REST API를 통한 Active NameNode 확인"
date: 2019-02-12 20:51:12
categories: Hadoop
author : Jaesang Lim
tag: Hadoop
cover: "/assets/hadoop_log.png"
---

## WebApi를 이용한 Hadoop Active NameNode 확인하는 법
---

- 때로는 외부 서버에서 하둡 클러스터에 접근하여 HDFS 파일을 읽어 오거나, 업로드를 해야할 일이 생긴다
> 사실상, 해당 하둡 클러스터에 접속하여 hadoop cli로 접근하면 되지만.. 접근하고 내려받고 scp로 전송하고.. 귀찮으니깐.. 
 
- 사실 매우 간단하게, 네임노드 서버 주소와 HDFS 경로를 입력하여 데이터를 받아올 수 있다
```
 $hadoop fs -text hdfs://master01.xxx.xxx.xxx.com/path_1/path_2/filename.txt > hdfs_file.txt 
```


---
- 하지만!!, 이렇게 네임노드 서버의 주소를 하드코딩해버리면.. 언젠가 저 명령어를 포함한 shell script는 에러가 날 것이다..
> 어떤 장애, 문제로 인해 네임노드는 failover 될 수 있기 때문에.. 

- 그래서 이와 같이, 외부에서 hdfs WebApi에 접근할 시에는, Active 네임노드인지 확인해야하는 것이 필요하다 !!

```
#!/usr/bin/env bash

hadoop = ${HADOOP_HOME}/bin/hadoop

NN_RESPONSE=`curl -I "http://master01.xxx.xxx.xxx.com:50070/webhdfs/v1/${hdfs_home_path}?op=GETFILESTATUS" 2>/dev/null | head -n 1 | awk -F" " '{print $2}'`
if [${NN_RESPONSE} == "200" ]; then
  echo "access!"
else
  echo "denied!"
fi
```

--- 

- Active NameNode인 경우의 Response

```
[jaesang@test02 ~]$  curl -I "http://master01.xxx.xxx.xxx.com:50070/webhdfs/v1/user/jaesang?op=GETFILESTATUS"

HTTP/1.1 200 OK
Cache-Control: no-cache
Expires: Tue, 12 Feb 2019 12:06:40 GMT
Date: Tue, 12 Feb 2019 12:06:40 GMT
Pragma: no-cache
Expires: Tue, 12 Feb 2019 12:06:40 GMT
Date: Tue, 12 Feb 2019 12:06:40 GMT
Pragma: no-cache
Content-Type: application/json
Content-Length: 0
Server: Jetty(6.1.26)
```

- StandBy NameNode인 경우 경우의 Response

```
[jaesang@test02 ~]$  curl -I "http://master02.xxx.xxx.xxx.com:50070/webhdfs/v1/user/jaesang?op=GETFILESTATUS"

HTTP/1.1 403 Forbidden
Cache-Control: must-revalidate,no-cache,no-store
Date: Tue, 12 Feb 2019 12:06:43 GMT
Pragma: no-cache
Date: Tue, 12 Feb 2019 12:06:43 GMT
Pragma: no-cache
Content-Type: text/html; charset=iso-8859-1
Content-Length: 1386
Server: Jetty(6.1.26)
```

- 이 결과를 봤을 때, HTTP Response Code만 중요하기 때문에 Response Code만 추출하는 bash를 추가한다 !

```
NN_CHECK_URL='curl -I "http://master01.xxx.xxx.xxx.com:50070/webhdfs/v1/${hdfs_home_path}?op=GETFILESTATUS"'
NN_RESPONSE=`${NN_CHECK_URL} 2>/dev/null | head -n 1 | awk -F" " '{print $2}'`
echo ${NN_RESPONSE} # 200 
```

- 끝!
- 확인하고 땡기자.. 언제 에러날지 모르니.. 
