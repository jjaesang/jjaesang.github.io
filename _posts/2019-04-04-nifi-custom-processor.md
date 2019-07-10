---
layout: post
title:  "[NiFi] NiFi Custom Processor 만들기 "
date: 2019-04-04 17:53:12
categories: NiFi 
author : Jaesang Lim
tag: NiFi
cover: "/assets/instacode.png"
---

- 시작은 KafkaConsumer를 디버깅하기 위해 만들었지만...
- 현재는 nifi에서 제공하는 KafkaConsumer와 동작은 같지만, Consumer Config를 좀 더 유연하게 관리할 수 있도록 Custom Processor를 만들어서 사용중이다.
- 그래서 이 기회에 NiFi Consumer Processor를 빌드하는 방법에 대해 정리하고자 한다. 

# NiFi Custom Processor 

## Requirements
- Maven
- Intellij
- Terminal / Git bash 


## Step for Creating Custom Apache NiFi Processor
#### 1. Maven Repository에서 미리 정의된 maven project template(=archetype)을 불러옴
```bash
$ mvn archetype:generate
```

#### 2. nifi 입력
```bash
Choose a number or apply filter (format: [groupId:]artifactId, case sensitive contains): 1347: nifi
```

#### 3. 가져올 archetype 입력 ( processor-bundle-archetype )
```bash
Choose archetype:
1: remote -> org.apache.nifi:nifi-processor-bundle-archetype (-)
2: remote -> org.apache.nifi:nifi-service-bundle-archetype (-)
Choose a number or apply filter (format: [groupId:]artifactId, case sensitive contains): : 2
```

#### 4. 선택한 archetype 버전 입력 
```bash
Choose org.apache.nifi:nifi-service-bundle-archetype version:
1: 0.3.0
2: 0.4.0
...
26: 1.8.0
27: 1.9.0
28: 1.9.1
Choose a number: 28: 26
```

#### 5. Maven 프로젝트 입력
![new](https://user-images.githubusercontent.com/12586821/55542993-22179500-5703-11e9-8ee7-23e8a88be1c0.PNG)

#### 6. 빌드 성공 
![new2](https://user-images.githubusercontent.com/12586821/55542995-22179500-5703-11e9-87bc-9c859023ca07.PNG)

#### 7. IntelliJ에서 읽어와, 필요한 프로세서 개발 
   - 프로세서 개발 후, nar파일 로드시, java.lang.NoClassDefFoundError: org/apache/nifi/ssl/SSLContextService 에러 발생할 수 있음

  ```text
          2019-04-03 12:35:38,762 ERROR [main] org.apache.nifi.NiFi Failure to launch NiFi due to java.util.ServiceConfigurationError: org.apache.nifi.processor.Processor: Provider com.zum.nifi.processors.kafka.MyKafkaConsumer could not be instantiated
          java.util.ServiceConfigurationError: org.apache.nifi.processor.Processor: Provider com.zum.nifi.processors.kafka.MyKafkaConsumer could not be instantiated
  
          Caused by: java.lang.NoClassDefFoundError: org/apache/nifi/ssl/SSLContextService
          ...
          ...
          Caused by: java.lang.ClassNotFoundException: org.apache.nifi.ssl.SSLContextService
          ...
          ...
  ```
   
   - 프로젝트에 xxx-xxx-nar , xxx-xxx-processors 두개의 디렉토리가 존재
   - xxx-xxx-nar 디렉토리의 POM.xml dependency 추가하면 해결
   
   ```xml
      <dependency>
            <groupId>org.apache.nifi</groupId>
            <artifactId>nifi-standard-services-api-nar</artifactId>
            <version>1.8.0</version>
            <type>nar</type>
      </dependency>
   ```
    
   - [관련 링크](https://cwiki.apache.org/confluence/display/NIFI/Maven+Projects+for+Extensions#MavenProjectsforExtensions-LinkingProcessorsandControllerServices)
    
#### 8. 개발한 프로세서 빌드 
```bash
$ mvn clean install
```


#### 9. 완성된 .nar을 NiFi의 lib에 넣고 NiFi 재시작
> - 재시작 시, nifi-app.log에서 몇개의 Nar파일이 있고, 업로드한다는 로그가 남음

#### 10. NiFi Processor 탭에서 정상적으로 Import되는지 확인
