---
layout: post
title:  "dashmap <1> - concurrent hashmap"
date:   2022-08-28 23:01:00 +0930
categories: rust dashmap concurrent hashmap
---

Concurrent hashmap의 사용 동기는 익히 알려진 대로, map 전체의 lock을 잡는 대신 
critical section 을 최소한으로 설계하여 lock의 사용 기간과 범위를 
극단적으로 줄이도록 하여, 겹치지 않는 영역에 대한 자료 구조 연산들을 최대한 동시적으로
진행될 수 있도록 하겠다는 것이다. 물론, lock의 사용을 사용자로부터 감추어 deadlock/race-condition 
등의 concurrency bug 걱정을 덜어 줄 수 있다는 것도 큰 덤이 될 것이다.

그런데, RUST 에서는 한가지 이유가 더 있다. 바로, **mutable data structure의 
쓰레드간 공유를 "컴파일러의 허락"하에 사용하고 싶다는 것.** Concurrent hashmap의
대표 주자인 [dashmap 문서](https://docs.rs/dashmap/latest/dashmap/struct.DashMap.html) 에 보면,
아래와 같은 문구가 눈에 띈다.

> DashMap tries to be very simple to use and to be a direct replacement 
> for RwLock<HashMap<K, V, S>>. To accomplish these all methods take &self 
> instead modifying methods taking &mut self. This allows you to put a 
> DashMap in an Arc<T> and share it between threads while being able to 
> modify it.

RUST에서는, struct 필드값을 변경시키는 fn 의 경우 "&mut self"를 첫번째 인자로 가져오게 
되어 있는데, 이 경우 `Mutex`나 `RwLock`으로 보호되어야 쓰레드간 공유가 가능하도록 되어있다.
이 때문에, `HashMap`을 여러 쓰레드에서 쓰고 싶으면, `Arc<RwLock<HashMap<K, V, S>>>`와
같은 형태로 사용되어야 컴파일이 된다는 이야기. 게다가, 컴파일러를 만족시킨다 하더라도 concurrent
writes가 들어오면 `RwLock`, `Mutex` 어느걸 쓰더라도 single-threaded 성능 혹은 그 이하의 성능으로
떨어질 수 밖에 없을 것.

`DashMap`을 쓰면 문제가 간단해진다. 마법과도 같이, [insert, remove, clear, ...](https://github.com/xacrimon/dashmap/blob/master/src/lib.rs#L403)
이 `&self` 로 인자로 받기 때문에 `Arc<DashMap<K, V, S>>`의 형태로 곧바로 사용할 수 있고,
별도의 lock을 획득하는 과정이 필요가 없어진다. 다만, 그 마법은 일종의 [unsafe](https://github.com/xacrimon/dashmap/blob/459db7ac6f3d0b46f507cb724dd9bb0ce389f4ae/src/lib.rs#L850) 를 
이용한 흑마법의 일종(!)으로 컴파일러를 어느 정도 회피하는 것과 연관이 있는 듯하니 참고는 해둘 필요가 있겠다.

Concurrent hashmap의 사용처로 metrics library를 꼽을 수 있겠다.
Multi-threaded application에서 다양한 thread에서 metric이 emit될 수 
있을 텐데, metric library의 책임은 이들을 thread-safe 하게 잘 저장해두었다가
외부의 crawler (e.g., prometheus)에 정해진 포맷으로 전달해주는 것.
Concurrent hashmap을 사용할 좋은 기회이고, `Dashmap`의 fn 중에 흥미로운 것이 있어
한가지 소개하면 [alter()](https://docs.rs/dashmap/latest/dashmap/struct.DashMap.html#method.alter) 라는
것이다. 가령, 특정 key에 해당하는 value를 1 증가시키거나 다른 변형을 가하고 싶은 경우,
아무리 concurrent hashmap을 쓴다고 해도 단순치 않은데, 왜냐면 사용자가 여러개의 
insert/get 연산들을 atomic하게 처리하고 싶다는 의도를 알릴 방법이 없기 때문이다. 
다행히, `alter`를 사용하면 사용자가 단지 값을 읽는 것이 아니라 읽은 다음 곧바로 변경시키고자 한다는
의도를 전달할 수 있어, metric 구현에 유용하게 사용할 수 있다.

아래의 예제는,
5개의 message sender와 2개의 message receiver를 만들어 한개의 channel로 연결한 뒤,
channel에 message를 보낼 때마다 `num_sends`를 1씩 증가시키고
message를 받을 때마다 `num_recvs`를 1씩 감소시키고,
1초에 한번씩 metric 내용 전체를 출력하는 예제이다.
Channel을 통해 보내지는 총 메시지 개수는 5 * 1000000 이 되기 때문에, 
Dashmap으로 concurrency control이 잘 되었다면, 
적당한 시간이 흐른 뒤에는 `num_sends`과 `num_recvs`가 모두 5000000으로 출력이
되어야 한다.



```rust
use std::sync::Arc;
use std::thread;
use std::time::Duration;
use crossbeam::channel::{Receiver, RecvError, Sender, SendError, unbounded};
use dashmap::DashMap;
use rand::Rng;

const NUM_SENDERS: i32 = 5;
const NUM_RECEIVERS: i32 = 2;

fn main() {
    let counters  = {
        let m = DashMap::new();
        m.insert("num_sends", 0);
        m.insert("num_recvs", 0);
        Arc::new(m)
    };

    let (tx, rx) = unbounded();
    for _ in 0..NUM_SENDERS {
        let tx = tx.clone();
        let counters = counters.clone();
        run_msg_sender(tx, counters);
    }

    for _ in 0..NUM_RECEIVERS {
        let rx = rx.clone();
        let counters = counters.clone();
        run_msg_receiver(rx, counters);
    }

    loop {
        println!("{:?}", counters);
        thread::sleep(Duration::from_secs(1));
    }
}

fn run_msg_sender(tx: Sender<i32>, counters: Arc<DashMap<&'static str, i32>>) {
    thread::spawn(move || {
        let mut rng = rand::thread_rng();
        for _ in 0..1000000 {
            let val = rng.gen_range(0..10000);
            match tx.send(val) {
                Ok(_) => {
                    counters.alter("num_sends", |_, v| v + 1);
                }
                Err(_) => {
                    break;
                }
            };
        }
    });
}

fn run_msg_receiver(rx: Receiver<i32>, counters: Arc<DashMap<&'static str, i32>>) {
    thread::spawn(move || {
        loop {
            match rx.recv() {
                Ok(_) => {
                    counters.alter("num_recvs", |_, v| v + 1);
                }
                Err(_) => {
                    break;
                }
            }
        }
    });
}
```

위를 실행하면, 다음과 같은 결과가 출력된다:
```bash
$ cargo run
{"num_sends": 40, "num_recvs": 44}
{"num_sends": 843597, "num_recvs": 843597}
{"num_sends": 1667305, "num_recvs": 1667294}
{"num_sends": 2489836, "num_recvs": 2489839}
{"num_sends": 3316906, "num_recvs": 3316905}
{"num_sends": 4145870, "num_recvs": 4145863}
{"num_sends": 4994430, "num_recvs": 4994021}
{"num_sends": 5000000, "num_recvs": 5000000}
{"num_sends": 5000000, "num_recvs": 5000000}
...
```