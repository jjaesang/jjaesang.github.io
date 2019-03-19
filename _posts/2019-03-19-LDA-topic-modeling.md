---
layout: post
title:  "[ML] LDA Topic Modeling"
date: 2019-03-19 21:25:12
categories: Machine_Learning
author : Jaesang Lim
tag: ML
cover: "/assets/instacode.png"
---

## LDA (Latent Dirichlet Allocation)
> - 한국말로는 '잠재 디리클레 할당' 

### 1. 개념
![그림1](https://user-images.githubusercontent.com/12586821/54606389-12574a00-4a8f-11e9-9778-3e0d2d04dec2.png)

- 주어진 문서에 대하여 각 문서에 어떤 토픽들이 존재하는지를 서술하는 대한 확률적 토픽 모델 기법 중 하나
> - 쉽게 말해, 문서 집합에서 어떤 토픽이 있는지 추론하는 Topic Model
- 미리 알고 있는 토픽별 단어수 분포를 바탕으로, 주어진 문서에서 발견된 단어수 분포를 분석
- 이 분석을 통해 해당 문서가 어떤 토픽들을 함께 다루고 있을지를 예측할 수 있음


### 2. 방법론
1. Unsupervised Learning 
  - Dirichlet Distribution 기반으로 Document와 Topic를 모델링하는 방법
  - 문서 집합을 제공하고, LDA는 주제를 Output
  - 문서는 각 토픽가 있고, 각 토픽는 일련의 단어와 연관되어 있다라는 컨셉으로 시작 

2. Iterative Algorithm
  - 첫번째 Iteration은 각 문서의 모든 단어는 random topic 할당 
  - 문서에 대한 Topic Distribution와 토픽에 걸친 Word Distribution이 계산 

### 2. 하이퍼 파라미터 

- 최적의 K(토픽수)와 Max Iteration(학습 횟수)등을 설정해야함
> - Optimizer도 설정해야함

1. K = 토픽 숫자 - 동적으로 선정
> - 문서 집합을 몇 개의 클러스터 군, 즉 몇 개의 토픽으로 나눌 것인가
> - 문서 집합에 사실상 몇 개의 토픽이 존재하지는 알 수 없음
> - 최적의 K를 찾기 위한 방법을 모색 필요

2. Max Iteration(학습 횟수)
> - 최적의 K가 선정된 후, 일정 학습 시간이 지나면 logPerplexity와 logLikelihood 값은 일정하게 수렴함
> - Max Iteration은 모델 학습 시간을 선형적으로 증가시킴
> -수렴하기 시작하면 그 뒤의 학습은 결과에 큰 영향이 끼치지 않음 

3. Optimizer ( Spark 2.1.2 기준 ML)
- EMLDAOptimizer 
> - DistributedLDAModel
- OnlineLDAOptimizer 
> - LocalLDAModel


### 3. 모델 학습 결과

- 토픽 클러스터링 결과로, 토픽 키워드와 문서에 할당된 토픽 정보를 알 수 있음
- 각 토픽 별, 키워드와 문서 정보를 활용하여 토픽 추출
> - Topic 1, 2, .. K 이 아닌 , Topic의 이름을 할당해주자
![그림2](https://user-images.githubusercontent.com/12586821/54606390-12efe080-4a8f-11e9-8369-e7d6ebc55c7f.png)
![그림4](https://user-images.githubusercontent.com/12586821/54606391-12efe080-4a8f-11e9-82d6-55ef45f3c596.png)

