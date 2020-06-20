---
layout: post
title:  "[Ignite] Configurations"
date: 2020-06-20 16:05:12
categories: Ignite
author : Jaesang Lim
tag: Ignite
cover: "/assets/instacode.png"
---

# Ignite(GridGain) Configurations 

최신 버전으로 업그레이드 전, 우리의 목적, 상황에 맞게 최적의 Configuration값을 설정하자.

모든 설정값은 해당 javadoc에서 볼수있음
- [링크](https://www.gridgain.com/sdk/ce/8.7.19/javadoc/constant-values.html#org.apache.ignite.configuration.DataStorageConfiguration.DFLT_CHECKPOINT_FREQ)

## Understanding Configuration
- [링크](https://www.gridgain.com/docs/latest/developers-guide/understanding-configuration)
 

설정방법
- Spring XML Configuration
- Code 
 > - IgniteConfiguration
 > - DataRegionConfiguration
 > - CacheConfiguration
 

## Configuring Data Regions

-GridGain은 데이터 영역 개념을 사용하여 캐시 또는 캐시 그룹에 사용 가능한 RAM의 양을 제어합니다. 
- 데이터 영역은 캐시 된 데이터가 상주하는 RAM의 논리적 확장 가능 영역입니다. 
영역의 초기 크기와 해당 영역이 차지할 수있는 최대 크기를 제어 할 수 있습니다.
 크기 외에도 데이터 영역은 캐시의 지속성 설정을 제어합니다.
- default로 RAM의 20%사용하고 모든 캐시는 해당 region에 들어감 
- Region을 여러개 설정할 수 있음 

1. Region 설정

<bean class="org.apache.ignite.configuration.IgniteConfiguration" id="ignite.cfg">
    <property name="dataStorageConfiguration">
        <bean class="org.apache.ignite.configuration.DataStorageConfiguration">
            <!--
            Default memory region that grows endlessly. Any cache will be bound to this memory region
            unless another region is set in the cache's configuration.
            -->
            <property name="defaultDataRegionConfiguration">
                <bean class="org.apache.ignite.configuration.DataRegionConfiguration">
                    <property name="name" value="Default_Region"/>
                    <!-- 100 MB memory region with disabled eviction. -->
                    <property name="initialSize" value="#{100 * 1024 * 1024}"/>
                </bean>
            </property>
        </bean>
    </property>
    <!-- other properties -->
</bean>

2. pageEvictaionMode 





---

https://www.gridgain.com/docs/latest/developers-guide/memory-configuration/eviction-policies


By default, off-heap memory eviction is disabled, which means that the used memory constantly grows until it reaches its limit. 
To enable eviction, specify the page eviction mode in the data region configuration. Note that off-heap memory eviction is configured per data region. 

By default, eviction starts when the overall RAM consumption by a region gets to 90%.
 Use the DataRegionConfiguration.setEvictionThreshold(…​) parameter if you need to initiate eviction earlier or later.
