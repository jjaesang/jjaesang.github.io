---
layout: post
title:  "[NiFi] DistributedMapCache 프로세서 "
date: 2019-07-10 17:53:12
categories: NiFi 
author : Jaesang Lim
tag: NiFi
cover: "/assets/instacode.png"
---

NiFi 관련 사내 세미나도 마무리되었고, 이제 NiFi로 유연하게 문제를 해결해야할 때인 것 같다 

그래서 이번에는 NiFi에서 처리한 데이터를 Enrich하는 방법으로 DistributedMapCache Processor를 사용해서 해결하고자한다

매번 DB가서 해당 메타정보를 가져온다던지, 아니면 배치로 테이블로 Join해서 enrich하는 방법도 있겠지만, Data Flow을 한 곳에서 처리하고 싶어서 ! 


# DistributedMapCache Processor 
---

## DistributedMapCache를 사용하기 위해 필요한 Processor 및 Controller Services
Processor
- PutDistributedMapCache 
- FetchDistributedMapCache

Controller Service
- DistributedMapCacheClientService 
- DistributedMapCacheServer
---

### PutDistributedMapCache Processor
- 캐시할 key/value을 넣는 Processor
- key는 flowfile의 Attribute값 
> - Processor의 'Cache Entry Identifier'에 명시한 Attribute
- value는 flowfile의 Content값
- [github](https://github.com/apache/nifi/blob/master/nifi-nar-bundles/nifi-standard-bundle/nifi-standard-processors/src/main/java/org/apache/nifi/processors/standard/PutDistributedMapCache.java)
---

### FetchDistributedMapCache Processor
- key,flowfile의 Attribute값으로 값을 가져와서 새로운 Attribute로 붙임 
> - key는 Processor의 'Cache Entry Identifier'에 명시한 Attribute
> - cache된 정보를 가져와 추가할 Attribute 이름 'Put Cache Value In Attribute'
- [github](https://github.com/apache/nifi/blob/master/nifi-nar-bundles/nifi-standard-bundle/nifi-standard-processors/src/main/java/org/apache/nifi/processors/standard/FetchDistributedMapCache.java)
---

### DistributedMapCacheClientService (Controller Service)
- DistributedMapCacheServer와 통신하는 Client
- Cluster끼리 Cache된 Map 공유하기 위함 
- 설정값은 DistributedMapCacheServer의 hostname과 port만 설정함
> - Cluster 환경에서는 hostname을 localhost로 설정
- [github](https://github.com/apache/nifi/tree/master/nifi-nar-bundles/nifi-standard-services/nifi-distributed-cache-services-bundle/nifi-distributed-cache-client-service/src/main/java/org/apache/nifi/distributed/cache/client)

---

### DistributedMapCacheServer (Controller Service)

- Socket으로 캐시된 맵에 접근할 수 있게함 
- DistributedMapCacheClientService과 상호작용
- 실행하고 꼭 확인해보자 
> - netstat -plnt 로 해당 port가 열렸는지.. (default : 4557)

- Persistence Directory
> - 해당 경로를 지정하면 캐시된 정보를 디스크에 저장함 
> - 설정하지 않으면, 메모리에만 저장되고 관리 

- Maximum Cache Entries 설정 가능
> - Cache에 최대 몇개의 데이터를 가지고 있을 것인가
> - 소스코드를 보니, 여기서 설정한 값으로 Map 만들더라..

당연히 Eviction 전략도 있음
- Eviction Strategy
> - FIFO
> - Least Recently Used (LRU)
> - Least Frequently Used (LFU)

- [github](https://github.com/apache/nifi/tree/master/nifi-nar-bundles/nifi-standard-services/nifi-distributed-cache-services-bundle/nifi-distributed-cache-server/src/main/java/org/apache/nifi/distributed/cache/server)


---

모 여기까지는 사용법이고, 내가 재밌게 봤던 것은 Cache하는 자료구조는 무엇이고, Eviction 전략을 어떻게 구현했는가 궁금했음

캐시에 사용되는 Map은 HashMap을 사용하고, Eviction을 처리하기 위해서는 ConcurrentSkipListMap 사용함
> - ConcurrentSkipListMap에 Eviction 전략에 따라서 각 Comparator를 구현해놓았음!

간단하게 Map 만들고, Evict하는 함수만 보면 다음과 같다
- 생각보다 소스코드가 간결해서 좋았음!


```java
public class SimpleMapCache implements MapCache {

    private static final Logger logger = LoggerFactory.getLogger(SimpleMapCache.class);

    private final Map<ByteBuffer, MapCacheRecord> cache = new HashMap<>();
    private final SortedMap<MapCacheRecord, ByteBuffer> inverseCacheMap;

    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    private final String serviceIdentifier;

    private final int maxSize;

    public SimpleMapCache(final String serviceIdentifier, final int maxSize, final EvictionPolicy evictionPolicy) {
        // need to change to ConcurrentMap as this is modified when only the readLock is held
        inverseCacheMap = new ConcurrentSkipListMap<>(evictionPolicy.getComparator());
        this.serviceIdentifier = serviceIdentifier;
        this.maxSize = maxSize;
    }

    @Override
    public String toString() {
        return "SimpleMapCache[service id=" + serviceIdentifier + "]";
    }

    // don't need synchronized because this method is only called when the writeLock is held, and all
    // public methods obtain either the read or write lock
    private MapCacheRecord evict() {
        if (cache.size() < maxSize) {
            return null;
        }

        final MapCacheRecord recordToEvict = inverseCacheMap.firstKey();
        final ByteBuffer valueToEvict = inverseCacheMap.remove(recordToEvict);
        cache.remove(valueToEvict);

        if (logger.isDebugEnabled()) {
            logger.debug("Evicting value {} from cache", new String(valueToEvict.array(), StandardCharsets.UTF_8));
        }

        return recordToEvict;
    }
    //.....
    //....
    }
    
```


   
