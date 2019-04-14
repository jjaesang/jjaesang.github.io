---
layout: post
title:  "[HP-Spark] 효율적인 트랜스포메이션 : Part 3"
date: 2019-04-14 20:45:12
categories: High-Performance-Spark 
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

## mapPartitions로 수행한 반복자-반복자 트랜스포메이션

- RDD의 mapPartitions 함수는 레코드들의 iterator를 받아 다른 iterator로 출력하는 함수를 인자로 받음
> - 한 파티션 안의 레코드들을 순회할 수 있는 iterator를 받음

- mapParitions 트랜스포메이션은 사용자가 한 파티션의 데이터를 대상으로 임의의 코드를 정의할 수 있게 해줌
- filter, map, flatMap 등 의 트랜스포메이션도 mapPartitions을 써서 작성할 수 있음
> - mapPartitions 코드를 최적화하는 것은 복잡하면서도 뛰어난 성능을 발휘하는 만드는데 중요한 부분을 차지함

- 레코드 일부가 디스크에 쓰일 수 있도록 하는 유연성을 스파크에 허용하기 위해서는,
- 직접 작성한 mapPartitions를 위한 함수가 전체 파티션을 메모리에 올리지 않도록 주의해함 (암시적으로 리스트로 변한한단던지.. )
**이거 무슨말이지..**

### 반복자-반복자 트랜스포메이션이란?
 - iterator를 받아서 다른 컬렉션으로 강제적인 변환없이 iterator를 되돌려주는 것
 - iterator를 다른 collection으로 변환하지 않고
 - iterator 액션 중 하나로 iterator를 평가해서 새로운 collection을 만드는 것이 아니라, 
 - iterator 트랜스포메이션을 써서 새로운 iterator를 되돌려주는 것
 
 - while loop로 새로운 collection을 만드는 것은 반복자-반복자 트랜스포메이션이 아님
 - iterator를 collection으로 변환하고 처리한 뒤 다시 iterator로 변환해서 돌려주는 것도 반복자-반복자 트랜스포메이션이 아님
 
 - 실질적으로 mapPartitions의 iterator인자를 collection 객체로 변환하는 것은 반복자-반복자 트랜스포메이션의 이점을 다 없애버림
 > - iterator는 한번에 하나의 아이템에 접근하므로, 전체 아이템을 메모리에 올리지 않는 등의 이득이 있음
 > - 하지만, iterator를 collection으로 변환하는 순간 모든 아이템이 메모리에 올라가므로 이득이 사라짐
 > - RDD 트랜스포메이션에 비유하자면, RDD collect해 모두 데이터르 처리하고 새로운 RDD로 만드는 것은 트랜스포메이션이라고 할 수 없는 것과 똑같음
 
  
 
#### iterator란 무엇인가
 1. 스칼라의 iterator 객체는 collection이 아님
 > - 스파크 평가시, java.util.Iterator와 동일한 장단점을 가짐 
 
 2. 그저, collection내의 원소들에 하나씩 접근할 수 있는 방식을 정의해놓은 함수
 3. iterator는 immutable하고, 동일 원소에 두 번 이상 접근할 수 없음( 딱 한번만! )
 - 즉, iterator란? '한 번만 순회하면서 접근할 수 있으며 스칼라의 TraversableOnce Interface를 구현'
 - iterator는 immutable한 스칼라 collection이 가지는 메소드 있음
  > 1. 매핑( map, flatMap) 
  > 2. 추가 (++) 
  > 3. 합치기 (foldLeft, reduceRight, reduce)
  > 4. 원소상태(forAll, exists)
  > 5. 순회(next,foreach) 
 - iterator는 한번만 순회 가능하기 때문에 반복자의 전체 아이템들을 보아야하는 종류의 메소그는 원래의 iterator를 텅비게 만듬
 
- 주의 사항
> - iterator에 대해 size나 암시적 변환을 작동해서 자기도 모르게 iterator를 순회하는 객체를 호출하는 방식은, iterator를 모두 소비해버릴 수 있음
> - iterator는 어떤 스칼라 collection 타입으로든 변환할 수 있음
> - 하지만, 타입으로 변환시키는 것은 모든 아이템을 접근할 수 밖에 없으며, 새로운 collection 타입으로 변환하고나면 
> - iterator는 마지막 아이템 위치를 가리케게 되는 '빈 상태'가 된다
 
 ---
 
 - RDD는 일종의 평가 명령의 집합기 때문에 트랜스포메이션이나 액션처럼 '반복자 메소드'를 개념화하는 것이 도움이 될 수 있음
 > - 반복자 메소드란? next,size,foreach 등등
 > - next,size,foreach는 액션과 유사하게 iterator를 순회한 다음 평가를 한다
 > - map, flatMap은 새로운 반복자를 반환 
 
- **iterator 트랜스포메이션은 스파크 프랜스포메이션과 달리, 병렬적으로 실행되지 않고, 한번에 한개씩 순차적으로 실행**
> - iterator를 느리게 만들지만 병렬로 실행하는 것 보다 사용하기가 더 쉬움
> - 일대일로 실행되는 함수들은 iterator 연산에서 서로 연계되어 실행되지 않으므로, map을 세번호출하면 iterator 내에서 각 아이템에 세번씩 접근을 의미

 
 
### 시간적/공간적인 이득
- 반복자-반복자 트랜스포메이션을 사용하느 것의 주된 이점은 **'스파크가 선택적으로 데이터를 디스크에 쓸 수 있다는 것'**
- 개념적은 반복자-반복자 트랜스포메이션은 아이템을 한번에 하나씩 평가하는 절차를 정의하는 것을 의미
- 그러므로, 스파크는 전체 파티션을 메모리에 읽어 들이거나, 모든 결과 레코드를 메모리의 collection에 담아 되돌려 줄 필요가 없이 **'그런 절차를 레코드의 batch 처리로 적용'**
- 즉, 반복자-반복자 트랜스포메이션은 스파크가 하나의 executor에서 메모리에 담기에는 너무 큰 파티션들을 memory에러없이 처리할 수 있음
- 파티션을 iterator로 유지하면, 스파크가 디스크 공간을 선택적으로 사용할 수 있음
- 전체 파티션이 메모리 공간에 들어가기 곤란한 경우, 반복자-반복자 트랜스포메이션은 파티션을 모두 디스크에 쓰지 않고 필요한 레코드만큼만 사용하여 디스크 IO나 재연산에 들어가는 비용을 아낌
- 마지막으로 iterator에 정의된 메소드를 사용하는 것은 중간 데이터용 자료 구조를 정의하지 않아도 된다는 뜻
> - 큰 크기의 중간 자료 구조를 쓰는 횟수를 줄이는 것은 필용없는 객체 생성을 피하는 방법


