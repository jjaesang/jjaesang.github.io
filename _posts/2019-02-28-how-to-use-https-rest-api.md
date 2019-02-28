---
layout: post
title:  "[ETC] curl 인증서와 함께 https rest-api 실행 "
date: 2019-02-25 11:15:12
categories: etc 
author : Jaesang Lim
tag: etc
cover: "/assets/instacode.png"
---

### Https / SSL로 인증이 필요한 웹서비스의 curl로 rest-api 요청하기

- 브라우저에 등록한 pkcs 파일을 이용하여 client cert, ca cert, client key 파일을 추출한다
> - pkcs 파일은 .pem파일의 암호화된 버전 같다.. (?)
- 추출한 인증서와 키를 옵션을 주고 curl 요청을 보낸다

#### 인증서 및 키 추출 스크립트
- config에서 keyStorePassword를 추출할 때마다 입력해준다. 
```bash
openssl pkcs12 -in keystore.pkcs12 -nocerts -nodes -out clientcert.key 
openssl pkcs12 -in keystore.pkcs12 -clcerts -nokeys -out clientcert.cer
openssl pkcs12 -in keystore.pkcs12 -cacerts -nokeys -chain -out cacerts.cer

# 각각 암호 입력
```

#### curl 옵션

- 암호키를 가지고 curl로 테스트하면 끝!
```bash
curl --cert clientcert.cer \
     --key clientcert.key \
     --cacert cacerts.cer \
     -X GET -v -i 'https://xxx.com:9091/nifi-api/processors/xxx'
```


- 이와 관련된 인증서, 키 값에 대해서는 나중에 정리한번 해봐야겠다.

