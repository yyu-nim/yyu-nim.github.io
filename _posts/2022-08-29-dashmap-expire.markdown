---
layout: post
title:  "dashmap <2> - retain() and LoadingCache (GuavaR?)"
date:   2022-09-04 22:01:00 +0930
categories: rust dashmap retain cache 
---

DashMap의 API 문서를 읽다보면, [retain()](https://docs.rs/dashmap/latest/dashmap/struct.DashMap.html#method.retain)
이라는 함수가 흥미를 끈다. 문서의 설명과 예제에 있는대로, `(key, value) -> bool` 타입의
함수를 파라미터로 건네줘서, DashMap의 entry들중에 위의 함수를 적용한 결과가 true인 것만을
남기고 false인 것들은 지우는 기능이다. 이 기능을 어디에 사용하면 좋을지?

Java 백그라운드가 있는 분 중에는, Google Guava 라이브러리에 포함된 `LoadingCache`를
떠올릴 수도 있겠다. 캐시를 생성할 때, 옵션중의 하나로 cache entry가 캐시안에 얼마 동안
머무를 수 있는지를 지정할 수 있는 옵션이 있다! 보통의 경우라면, cache가 가득 찬 상황에서
LRU 등의 eviction 알고리즘에 의해 cache entry의 유지 기간이 정해질 텐데, 그것이 아니고
"시간"에 따라 eviction이 아닌 곧바로 expiration에 의해 삭제될 수 있도록 하는 기능이다.
이처럼 워크로드의 특성이 temporality와 관련이 있는 경우, retain() 기능을 사용하여
cache invalidation 구현을 할 수 있을 것이다.

[Guava LoadingCache](https://github.com/google/guava/wiki/CachesExplained) 
의 사용법을 보면 아래와 같다.
```java
// RUST 코드 아님. Java임.
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(1000)
       .expireAfterWrite(10, TimeUnit.MINUTES)
       .removalListener(MY_LISTENER)
       .build(
           new CacheLoader<Key, Graph>() {
             @Override
             public Graph load(Key key) throws AnyException {
               return createExpensiveGraph(key);
             }
       });
```
의미가 명확하니 자세한 설명은 생략하고, 여기에서 Builder 패턴 간략화하고, 
bounded capacity 기능 생략하고,
removal notification 기능 생략하고, 
Key/Value 제너릭 생략해서,
간단한 RUST 버전 구현인 GuavaR `LoadingCacheR`을 만들어 보려 한다.
추후에, GuavaR 완성을 목표로 시리즈 연재물을 올리는 것도 좋을 듯.

아래의 예제는, `LoadingCacheR`을 생성해서 `NUM_THREADS`가 
concurrent 하게 get()를 `NUM_REQS`만큼 호출하는 코드이다. 
캐시에 put()을 하는 코드를 사용자가 (즉, main) 직접 호출하지 않는 점에 주목하자.
get()의 결과로 cache miss가 발생한 시점에, `cache_loader`가 주어진 key로
value를 얻어와서 캐시에 entry를 넣어주는 구조이고, 따라서 이를 "**Loading**"Cache라
부른다.
앞서 언급한 retain()의 기능을 시험하기 위해, `build()` 시점에 새로운 쓰레드를
생성하여 매 7초마다, 10초이상 캐시에 머무른 entry들을 expire 시키도록 한다.
`map`의 value는 `(Instant, u64)`처럼 pair를 구성해서 timestamp를 바탕으로
 expiration 여부를 결정할 수 있도록 한다. 
정석적인 Builder pattern은 [문서1](https://rust-unofficial.github.io/patterns/patterns/creational/builder.html), 
[문서2](https://doc.rust-lang.org/1.0.0/style/ownership/builders.html) 를
참고하도록 하고, 여기서는 type 수를 최소화하고 싶어 helper struct 없이 
`LoadingCacheR`에 전부 밀어넣었다.

```rust
#[macro_use]
extern crate log;

use std::borrow::Borrow;
use std::fmt::{Display, Formatter};
use std::sync::Arc;
use std::thread;
use std::time::Duration;
use std::time::Instant;

use crossbeam::atomic::AtomicCell;
use crossbeam::queue::ArrayQueue;
use dashmap::DashMap;
use env_logger::{Builder, Target};
use rand::Rng;

const NUM_THREADS: usize = 5;
const NUM_REQS: usize = 1000;
const DUMMY_VALUE: u64 = 10;
const PURGE_INTERVAL_SEC: u64 = 7;
const PRINT_INTERVAL_SEC: u64 = 1;
const EXPIRE_AFTER_WRITE_SEC: u64 = 10;

fn main() {
    env_logger::init();
    let cache = LoadingCacheR::newBuilder()
        .expireAfterWrite(Duration::from_secs(EXPIRE_AFTER_WRITE_SEC))
        .build(|key| -> u64 {
            debug!("Loading a value for key {}...", key);
            // 실제 시스템에서는, 값을 읽어오거나 계산하는데 오래 걸린다고 가정하자.
            // 예를 들면 DB table을 join하거나 analytic query를 처리해야 얻을 수 있는 결과들의 경우가 그럴듯.
            // 여기선 thread::sleep() 으로 단지 delay 만 주도록 한다.
            let mut rng = rand::thread_rng();
            let jitter_ms = rng.gen_range(0..100);
            thread::sleep(Duration::from_millis(jitter_ms));
            debug!("Loaded a value for key {}...", key);
            DUMMY_VALUE
        });

    let cache = Arc::new(cache);
    for _ in 0..NUM_THREADS {
        let cache = cache.clone();
        thread::spawn(move || {
            let mut rng = rand::thread_rng();
            for _ in 0..NUM_REQS {
                let key = format!("Key{}", rng.gen_range(0..1000) as u64);
                cache.get(key.as_str());
                thread::sleep(Duration::from_millis(10));
            }
        });
    }

    loop {
        thread::sleep(Duration::from_secs(PRINT_INTERVAL_SEC));
        info!("{}", cache);
    }
}

struct LoadingCacheR {
    map: Arc<DashMap<String, (Instant, u64)>>,
    retention: Option<Duration>,
    cache_loader: Option<fn(String) ->u64>,
    num_cache_hit: AtomicCell<u64>,
    num_cache_miss: AtomicCell<u64>,
}

impl LoadingCacheR {
    fn newBuilder() -> LoadingCacheR {
        LoadingCacheR {
            map: Arc::new(Default::default()),
            retention: None,
            cache_loader: None,
            num_cache_hit: AtomicCell::new(0),
            num_cache_miss: AtomicCell::new(0),
        }
    }

    fn expireAfterWrite(mut self, retention: Duration) -> LoadingCacheR {
        self.retention = Some(retention);
        self
    }

    fn build(mut self, cache_loader: fn(String) -> u64) -> LoadingCacheR {
        self.cache_loader = Some(cache_loader);
        if let Some(dur) = self.retention {
            let cloned = self.map.clone();
            thread::spawn(move || {
                loop {
                    thread::sleep(Duration::from_secs(PURGE_INTERVAL_SEC));
                    let mut num_expired = 0;
                    cloned.retain(|k, (t, v)| {
                        if t.elapsed() < dur {
                            true
                        } else {
                            debug!("Expiring ({}, ({:?}, {})", k, t, v);
                            num_expired += 1;
                            false
                        }
                    });
                    info!("Expired {} entries...", num_expired);
                }
            });
        }
        self
    }

    fn get(&self, key: &str) -> Option<u64> {
        if let Some(entry) = self.map.get(key) {
            debug!("Cache hit for key {}", key);
            self.num_cache_hit.fetch_add(1);
            let (_written_time, v) = entry.value();
            return Some(v.clone());
        }
        self.num_cache_miss.fetch_add(1);

        match &self.cache_loader {
            None => {
                None
            }
            Some(f) => {
                let v = f(key.to_string());
                let now = Instant::now();
                self.map.insert(String::from(key), (now, v));
                Some(v)
            }
        }
    }
}

impl Display for LoadingCacheR {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "# of cache entries: {}, # of cache hits: {}, # of cache misses: {}",
               self.map.len(), self.num_cache_hit.load(), self.num_cache_miss.load());
        Ok(())
    }
}
```

이를 실행하면, 매 1초마다 캐시의 상태를 출력해주게 된다. 
로그 레벨을 INFO로 하도록 인자를 주어야 하는 점에 유의할 것. 
실험의 후반부에는 캐시의 모든 entry들이 expire되어, 캐시 사이즈가 0으로 출력됨을
확인할 수 있다.
```bash
$ RUST_LOG=info cargo run
[2022-09-04T14:17:13Z INFO  expiry_cache] # of cache entries: 76, # of cache hits: 1, # of cache misses: 79
[2022-09-04T14:17:14Z INFO  expiry_cache] # of cache entries: 147, # of cache hits: 14, # of cache misses: 151
[2022-09-04T14:17:15Z INFO  expiry_cache] # of cache entries: 223, # of cache hits: 31, # of cache misses: 227
[2022-09-04T14:17:16Z INFO  expiry_cache] # of cache entries: 300, # of cache hits: 55, # of cache misses: 301
[2022-09-04T14:17:17Z INFO  expiry_cache] # of cache entries: 372, # of cache hits: 95, # of cache misses: 375
[2022-09-04T14:17:18Z INFO  expiry_cache] # of cache entries: 440, # of cache hits: 146, # of cache misses: 444
[2022-09-04T14:17:19Z INFO  expiry_cache] Expired 0 entries...
[2022-09-04T14:17:19Z INFO  expiry_cache] # of cache entries: 507, # of cache hits: 210, # of cache misses: 513
[2022-09-04T14:17:20Z INFO  expiry_cache] # of cache entries: 568, # of cache hits: 304, # of cache misses: 574
[2022-09-04T14:17:21Z INFO  expiry_cache] # of cache entries: 620, # of cache hits: 402, # of cache misses: 626
[2022-09-04T14:17:22Z INFO  expiry_cache] # of cache entries: 678, # of cache hits: 494, # of cache misses: 685
[2022-09-04T14:17:23Z INFO  expiry_cache] # of cache entries: 730, # of cache hits: 619, # of cache misses: 739
[2022-09-04T14:17:24Z INFO  expiry_cache] # of cache entries: 785, # of cache hits: 764, # of cache misses: 792
[2022-09-04T14:17:25Z INFO  expiry_cache] # of cache entries: 827, # of cache hits: 957, # of cache misses: 837
[2022-09-04T14:17:26Z INFO  expiry_cache] Expired 296 entries...
[2022-09-04T14:17:26Z INFO  expiry_cache] # of cache entries: 572, # of cache hits: 1174, # of cache misses: 879
[2022-09-04T14:17:27Z INFO  expiry_cache] # of cache entries: 632, # of cache hits: 1263, # of cache misses: 940
[2022-09-04T14:17:28Z INFO  expiry_cache] # of cache entries: 686, # of cache hits: 1360, # of cache misses: 994
[2022-09-04T14:17:29Z INFO  expiry_cache] # of cache entries: 741, # of cache hits: 1481, # of cache misses: 1050
[2022-09-04T14:17:30Z INFO  expiry_cache] # of cache entries: 789, # of cache hits: 1627, # of cache misses: 1100
[2022-09-04T14:17:31Z INFO  expiry_cache] # of cache entries: 835, # of cache hits: 1809, # of cache misses: 1146
[2022-09-04T14:17:32Z INFO  expiry_cache] # of cache entries: 868, # of cache hits: 2055, # of cache misses: 1178
[2022-09-04T14:17:33Z INFO  expiry_cache] Expired 433 entries...
[2022-09-04T14:17:33Z INFO  expiry_cache] # of cache entries: 464, # of cache hits: 2346, # of cache misses: 1209
[2022-09-04T14:17:34Z INFO  expiry_cache] # of cache entries: 533, # of cache hits: 2415, # of cache misses: 1276
[2022-09-04T14:17:35Z INFO  expiry_cache] # of cache entries: 603, # of cache hits: 2497, # of cache misses: 1346
[2022-09-04T14:17:36Z INFO  expiry_cache] # of cache entries: 662, # of cache hits: 2586, # of cache misses: 1408
[2022-09-04T14:17:37Z INFO  expiry_cache] # of cache entries: 718, # of cache hits: 2682, # of cache misses: 1462
... (생략)
[2022-09-04T14:17:50Z INFO  expiry_cache] # of cache entries: 238, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:51Z INFO  expiry_cache] # of cache entries: 238, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:52Z INFO  expiry_cache] # of cache entries: 238, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:53Z INFO  expiry_cache] # of cache entries: 238, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:54Z INFO  expiry_cache] Expired 236 entries...
[2022-09-04T14:17:54Z INFO  expiry_cache] # of cache entries: 2, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:55Z INFO  expiry_cache] # of cache entries: 2, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:56Z INFO  expiry_cache] # of cache entries: 2, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:57Z INFO  expiry_cache] # of cache entries: 2, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:58Z INFO  expiry_cache] # of cache entries: 2, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:17:59Z INFO  expiry_cache] # of cache entries: 2, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:18:00Z INFO  expiry_cache] # of cache entries: 2, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:18:01Z INFO  expiry_cache] Expired 2 entries...
[2022-09-04T14:18:01Z INFO  expiry_cache] # of cache entries: 0, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:18:02Z INFO  expiry_cache] # of cache entries: 0, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:18:03Z INFO  expiry_cache] # of cache entries: 0, # of cache hits: 3306, # of cache misses: 1694
[2022-09-04T14:18:04Z INFO  expiry_cache] # of cache entries: 0, # of cache hits: 3306, # of cache misses: 1694
```