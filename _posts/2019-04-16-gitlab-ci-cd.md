---
layout: post
title:  "[ETC] GitLab CI/CD 설정 "
date: 2019-04-16 17:15:12
categories: etc 
author : Jaesang Lim
tag: etc
cover: "/assets/instacode.png"
---

## GitLab CI/CD 구성법
> - 이미 gitlab을 사용하고 있으며, git 서버에 gitLab-runner가 설치되어있다는 가정하에 runner 등록 및 .gitlab-ci.yml에 대한 설명

1. gitlab-runner 등록
2. .gitlab-ci.yml 정의

- 총 두 단계만 진행하도록 하겠음!
---

## 1. gitlab-runner 등록

![333](https://user-images.githubusercontent.com/12586821/56197008-e5d52480-6072-11e9-8af0-f44d72ed7735.png)

```bash
$ gitlab-runner register

        Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
        # 위에 있는 URL
        Please enter the gitlab-ci token for this runner:
        # 위에 있는 Token
        Please enter the gitlab-ci description for this runner:
        [xxxx]:
        Please enter the gitlab-ci tags for this runner (comma separated):
        
        nifi-bundle    : 실제로 구별하는 tag 아주 중요한 이름 잘짖기
        Registering runner... succeeded                     runner=Emp_iyNFxa
        Please enter the executor: docker, shell, ssh, virtualbox, docker-ssh, parallels, docker+machine, docker-ssh+machine, kubernetes:
        # shell로 Job 정의할 꺼니깐
        shell
        Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

$ gitlab-runner restart
  
```
- 위의 작업하면 CI/CD 탭의 runner가 초록불이 들어오면 정상!


---

## 2. .gitlab-ci.yml 포맷

### 1. stage
- 작업할 Stage 정의
- test , build, deploy 등등 .. 

### 2. variables
- .gitlab-ci.yml에서 Global하게 사용할 변수 정의

### 3. 각 스테이지 별 Job 정의
- only : 
- stage 
 > - 1번 stage 정의한 stage 중 하나 입력
- scipts 
 > - 해당 스테이지에서 작업할 shell 작업 정의
 > - mvn test 등등
- tags 
 > - git-runner 등록 시, 사용한 tags 내용을 그대로 입력 
 > - 안그러면 오류남 
- variables:
 > - GIT_STRATEGY: none / clone / fetch
 > - Job 마다 정의할 수 있으며, Global하게 정의하면 모든 Job에 동일하게 GIT_STRATEGY 작용
- when :
 > - 해당 시작은 언제 실행할 것인가?
 > > - default는 on_success로 이전 stage가 끝나면 바로 실행
 > > - 직접 수동으로 실행하고자 할 때는 'when: manual'로 정의

- 전체 예시


```yaml
stages :
  - test
  - build
  - deploy
  - restart

variables:
  NIFI_SERVERS: "jaesang@xx.xx.xx.00 jaesang@xx.xx.xx.01 jaesang@xx.xx.xx.02"
  NIFI_NAR_PATH: "/usr/hdf/current/nifi/lib"
  NIFI_NAR_NAME: "nifi-custom-bundle-nar-1.0.nar"
  MVN_HOME: "/usr/local/apache-maven-3.5.3"


test:
  only:
    - master
  stage: test
  script:
    - ${MVN_HOME}/bin/mvn test
  tags:
    - nifi-bundle


build:
  only:
    - master
  stage: build
  script:
    - echo $(pwd)
    - ${MVN_HOME}/bin/mvn -Dmaven.test.skip=true clean package
  tags:
    - nifi-bundle


deploy:
  only:
    - master
  stage: deploy
  variables:
    GIT_STRATEGY: none
  script:
    - >
     for SERVER in ${NIFI_SERVERS};
       do
        echo "SERVER : ${SERVER}"
       rsync -av -e "ssh -pXX -ljaesang" --delete nifi-custom-bundle-nar/target/${NIFI_NAR_NAME} ${SERVER}:${NIFI_NAR_PATH}
       done
  when: manual
  tags:
    - nifi-bundle

restart:
  only:
    - master
  stage: restart
  variables:
      GIT_STRATEGY: none
  script:
   - echo "RESTART NiFi Services for update latest custom nar.. "
   - ssh -p6879 ${AMBARI_SERVER} "echo ${NIFI_PASSWORD} | sudo -S sh ~/nifi_rest_api.sh restart "
  when: manual
  tags:
    - nifi-bundle
```
