---
layout: post
title:  "dashmap <3> - capacity/shards and parameter sweep"
date:   2022-09-11 19:01:00 +0930
categories: rust dashmap capacity shards parameter sweep
---

Dashmap의 팩토리 메서드 중에, [with_capacity_and_hasher_and_shard_amount()](https://docs.rs/dashmap/latest/dashmap/struct.DashMap.html#method.with_capacity_and_hasher_and_shard_amount)
라는 것이 있다. 주의해서 사용하라는 경고 문구가 없으니, 개발자가 마음껏 사용해도 되는 API 일 터. 
권장되는 좋은 값들이 예시로 있다면 좋았을 텐데 아쉽다. 그래서, 인자로 들어가는 값들의 다양한 조합들에 대해
`insert()` 메서드의 성능 차이가 있는지 여부를 **직접** 확인해 보도록 한다.

`capacity`는 dashmap의 rebuild와 관련한 영향을 줄 것이라는 합리적인 의심을 가질 수 있다.
dashmap의 capacity에 가깝게 엔트리들이 많아지게 되면, hash collision이 빈번하게 생겨 O(1)의 
time complexity를 달성치 못하게 될 가능성이 많아져 Hashmap 사용의 원래 목적이 사라지게 되므로,
capacity를 늘려 rehashing을 수행해야 하는 시점이 온다. 따라서, parameter sweep을 할 때에 
capacity를 변화해 가면서 `insert()`가 rehashing을 발생시키는 지점이 얼마나 자주 오는지, 
그때 시간이 어느 정도 걸리는지 확인해보면 좋을 것이다. 
이 때 시간을 측정하는 방법이 여럿 있을 수 있는데, 만약 "딱 하나의 숫자"로만 성능을 대표해서 설명해야 한다면, 
average insertion time을 측정하는 것이 대체적인 의견일 텐데, 이는 너무 많은 정보를 놓칠 우려가 있다.
여기에서는 Histogram을 사용해서 "각종 대표 통계치 + percentile" 정보를 요약해보도록 하자.
[HdrHistogram](https://crates.io/crates/hdrhistogram) 이 3.5M downloads 에 사용하기도 쉽고
평도 좋은 듯 하니, 이를 사용해 보도록 한다.

`shard`는 dashmap의 내부에서 concurrency를 극대화하고 lock contention을 줄이는 용도의 값일 것이라
추정해 볼 수 있다. 이를 고려한다면, 실험 설계에 concurrent insert가 발생할 수 있도록 하여 시간을 측정해
보아야 할텐데, concurrent insert()는 고려되지 않은 실험 설계를 해 두었고, 예상과 달리 
`shard` 값이 single-threaded workload에서도 차이를 보인 점 미리 언급해 둔다.
`hasher`는 사용자의 key를 dashmap 내부의 address space에 골고루 mapping 시킬 수 있도록 하는
무언가로 의심해볼 수 있고, `RandomState` 이외의 struct를 사용한 예제를 보기 힘드므로, 고정시켜 사용하도록 한다.

[Criterion](https://crates.io/crates/criterion) 과 같은 microbenchmark 툴을 사용해서 
plotting 결과까지 확인해보는 것도 재밌겠으나, 추후 포스팅에서 다뤄보기로 한다. 
외부 의존성을 최대한 줄이고, 본 포스팅의 목적대로 예제를 작성해보면 아래와 같다:

```rust
use dashmap::DashMap;
use std::collections::hash_map::RandomState;
use std::time::Instant;
use hdrhistogram::Histogram;
use rand::{Rng, thread_rng};

fn main() {
    let num_inserts = 100000;
    let map_init_capacities = [1, 10, 100, 1000, 10000, 100000].to_vec();
    let map_shards = [2, 4, 8, 16, 32, 64, 128].to_vec();
    parameter_sweep(num_inserts, map_init_capacities, map_shards);
}

fn parameter_sweep(num_inserts: u64, map_capacities: Vec<usize>, map_shards: Vec<usize>) {
    for capacity in &map_capacities {
        for shard in &map_shards {
            let mut hist = Histogram::<u64>::new(2).unwrap();

            // 여기에서의 생성 인자가, insert()의 성능에 어떻게 영향을 주는지가 궁금.
            let map = DashMap::with_capacity_and_hasher_and_shard_amount(
                *capacity, RandomState::new(), *shard);
            for _ in 0..num_inserts {
                let v = thread_rng().gen_range(0..u64::MAX);
                // insert() 전후로 시간을 기록해서 histogram에 저장.
                let now = Instant::now();
                map.insert(v, ());
                let elapsed_us = now.elapsed().as_nanos() as u64;
                hist += elapsed_us;
            }

            // Histogram의 결과를 확인해보자.
            println!("capacity: {}, shard: {}, 50-perc: {}, 99-perc: {}, 99.99-perc: {}",
                     capacity, shard,
                     hist.value_at_percentile(50 as f64),
                     hist.value_at_percentile(99 as f64),
                     hist.value_at_percentile(99.99));
        }
    }
}
```

이를 실행하면,
```bash
capacity: 1, shard: 2, 50-perc: 627, 99-perc: 3087, 99.99-perc: 378879
capacity: 1, shard: 4, 50-perc: 583, 99-perc: 959, 99.99-perc: 1028095
capacity: 1, shard: 8, 50-perc: 583, 99-perc: 1003, 99.99-perc: 1032191
capacity: 1, shard: 16, 50-perc: 543, 99-perc: 959, 99.99-perc: 1019903
capacity: 1, shard: 32, 50-perc: 543, 99-perc: 959, 99.99-perc: 514047
capacity: 1, shard: 64, 50-perc: 543, 99-perc: 1003, 99.99-perc: 260095
capacity: 1, shard: 128, 50-perc: 543, 99-perc: 3087, 99.99-perc: 134143
capacity: 10, shard: 2, 50-perc: 583, 99-perc: 1003, 99.99-perc: 255999
capacity: 10, shard: 4, 50-perc: 543, 99-perc: 959, 99.99-perc: 1019903
capacity: 10, shard: 8, 50-perc: 583, 99-perc: 919, 99.99-perc: 1032191
capacity: 10, shard: 16, 50-perc: 583, 99-perc: 1003, 99.99-perc: 1019903
capacity: 10, shard: 32, 50-perc: 583, 99-perc: 959, 99.99-perc: 511999
capacity: 10, shard: 64, 50-perc: 627, 99-perc: 4191, 99.99-perc: 282623
capacity: 10, shard: 128, 50-perc: 583, 99-perc: 4127, 99.99-perc: 136191
capacity: 100, shard: 2, 50-perc: 627, 99-perc: 3471, 99.99-perc: 274431
capacity: 100, shard: 4, 50-perc: 627, 99-perc: 1167, 99.99-perc: 1097727
capacity: 100, shard: 8, 50-perc: 583, 99-perc: 1003, 99.99-perc: 1028095
capacity: 100, shard: 16, 50-perc: 583, 99-perc: 959, 99.99-perc: 1019903
capacity: 100, shard: 32, 50-perc: 627, 99-perc: 3759, 99.99-perc: 544767
capacity: 100, shard: 64, 50-perc: 583, 99-perc: 1087, 99.99-perc: 258047
capacity: 100, shard: 128, 50-perc: 543, 99-perc: 1959, 99.99-perc: 132095
capacity: 1000, shard: 2, 50-perc: 583, 99-perc: 959, 99.99-perc: 266239
capacity: 1000, shard: 4, 50-perc: 583, 99-perc: 3087, 99.99-perc: 1019903
capacity: 1000, shard: 8, 50-perc: 583, 99-perc: 1167, 99.99-perc: 1023999
capacity: 1000, shard: 16, 50-perc: 583, 99-perc: 959, 99.99-perc: 1019903
capacity: 1000, shard: 32, 50-perc: 583, 99-perc: 959, 99.99-perc: 516095
capacity: 1000, shard: 64, 50-perc: 543, 99-perc: 959, 99.99-perc: 259071
capacity: 1000, shard: 128, 50-perc: 543, 99-perc: 1047, 99.99-perc: 130559
capacity: 10000, shard: 2, 50-perc: 543, 99-perc: 1003, 99.99-perc: 7711
capacity: 10000, shard: 4, 50-perc: 543, 99-perc: 959, 99.99-perc: 1019903
capacity: 10000, shard: 8, 50-perc: 543, 99-perc: 959, 99.99-perc: 1028095
capacity: 10000, shard: 16, 50-perc: 543, 99-perc: 919, 99.99-perc: 1019903
capacity: 10000, shard: 32, 50-perc: 543, 99-perc: 959, 99.99-perc: 514047
capacity: 10000, shard: 64, 50-perc: 543, 99-perc: 919, 99.99-perc: 259071
capacity: 10000, shard: 128, 50-perc: 543, 99-perc: 959, 99.99-perc: 132095
capacity: 100000, shard: 2, 50-perc: 583, 99-perc: 919, 99.99-perc: 6047
capacity: 100000, shard: 4, 50-perc: 543, 99-perc: 1047, 99.99-perc: 6399
capacity: 100000, shard: 8, 50-perc: 583, 99-perc: 3423, 99.99-perc: 6431
capacity: 100000, shard: 16, 50-perc: 583, 99-perc: 1003, 99.99-perc: 4927
capacity: 100000, shard: 32, 50-perc: 543, 99-perc: 875, 99.99-perc: 3583
capacity: 100000, shard: 64, 50-perc: 583, 99-perc: 1047, 99.99-perc: 6847
capacity: 100000, shard: 128, 50-perc: 583, 99-perc: 875, 99.99-perc: 8703
```

이번에는 결과 해석을 위해, 출력을 생략하지 않고 적어두었다.
일단 `insert()` latency 분포의 50th-percentile 값을 보면, 대략 500 ~ 600 ns 수준으로 
주어진 capacity/shard 의 변화가 크지 않은 것을 볼 수 있다. 하지만, 99.99th-percentile 값은
제법 차이가 나는데, 가령 capacity가 "1" 정도로 매우 작은 경우, insert() 한번 호출하는데 
1 밀리초 정도의 매우 높은 latency를 보이는 경우가 발생한다는 것이고 이는 dashmap 자체를 rehashing하여
크기를 늘리는 작업에 걸린 운나쁜 insert() 였을 것이라 추정할 수 있을 것이다.

`shard`의 값이 늘어남에 따라 99.99-perc와 같은 tail latency가 감소되는 것도 흥미로운 부분이다.
애초에, shard 값은 lock contention을 줄이고 concurrency를 극대화하기 위해 map 내부적으로 
다수의 별도의 작은 map들을 두는 방식의 구현이 아닐까 추측했었는데, 그렇다면 위 예제와 같은 single threaded
workload에서는 별 차이가 없었어야 했는데 예측이 빗나갔다. 내부적인 구현은 추후 파악해보기로 하고,
일단은 "capacity가 작은 경우 (그래서 rehashing이 발생하는 경우) 에는 shard 값이 클 수록
tail latency가 줄어든다" 정도의 관찰 결과만 기록해두고 가면 되겠다.
