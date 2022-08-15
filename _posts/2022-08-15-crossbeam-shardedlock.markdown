---
layout: post
title:  "crossbeam <4> - sharded lock"
date:   2022-08-15 21:15:00 +0930
categories: actix rust crossbeam shardedlock
---

State machine을 구현하다 보면, 이런 경우에 고민이 될 수 있다:
*"State A 에서 State B로 전환을 해야 하는데, State A에서 외부 Event가 들어오지 않도록
잠깐 막고 싶다"*. 

일단은, 프로젝트에 Actor framework의 도입이 되었거나 예정이거나 
큰 부담없이 도입할 수 있는 상태라면, 
[akka become/unbecome](https://doc.akka.io/docs/akka/current/actors.html#become-unbecome) 
을 사용하거나 유사한 패턴으로 actor 기반의 state machine을 구현하는 것이, 
여러 고민을 덜어줄 수 있는 좋은 방법이라 본다. 
Actor 도입이 당장 어렵고 shared memory 기반의 communication을 해야만 한다면
(즉 lock을 써야 한다면), `crossbeam`이 제공하는 [thread synchronization](https://docs.rs/crossbeam/latest/crossbeam/#thread-synchronization)
 을 살펴볼 필요가 있다.

이름만 봐서는 그 의도를 알기 어려운 `ShardedLock`이 오늘 포스트의 주제이다.
SharedLock이 아니고 ShardedLock 임에 주의할 것.
특징을 간단히 살펴보면, 

* `std::sync::RwLock`와 기능적으로 동일하다. 즉 reader-writer lock의 구현체이다.
* thread_id 로 샤딩하여 thread_local reader lock을 운영한다. 아마도 이 때문에,
[문서](https://docs.rs/crossbeam/latest/crossbeam/sync/struct.ShardedLock.html) 에서 read operation이 RwLock보다 빠르다고 언급하는 것으로 보여진다. 
샤드(shard)의 개수는 8로 고정되어 있다 (crossbeam-utils 0.8.10 기준).
* 위에 언급한 reader lock 최적화의 반대 급부로, writer lock은 성능이 떨어진다고 한다.
* Writer lock이 획득된 상태에서, lock owner thread가 `panic`으로 죽게 되면,
그 lock의 상태는 `poisoned`로 정의되고, `read()` 호출의 결과로 Error가 리턴되므로, 
이에 대한 처리가 필요하다.

`ShardedLock`을 써서 포스트 초반에 언급한 문제를 해결해 볼 수 있을 것 같다.
Read-Write 라는 단어 자체의 뜻에 휘둘리지 말고, `RwLock`이 가진 속성을 활용하면 되는데, 
바로 shared access와 exclusive access를 사용자가 직접 선택하여 쓸 수 있다는 것이다.
즉, 위의 문제에서 State가 변하지 않는 상황에서는 shared access lock (reader lock)을
획득하도록 하고, State가 변하는 상황에서는 exclusive access lock (writer lock)을
획득하도록 lock을 디자인 하면, 원하는 바를 이룰 수 있지 않을지. 

아래의 예제는, `Stage1`에서 `Stage2`로 transition을 하려고 하는 상황에서,
`concurrency` 만큼의 reader threads 들이 read lock을 반복적으로 획득했다가
놓으면서 `IORequest`라는 이벤트를 보내고, 그와 동시에 1개의 writer thread가
write lock을 획득하여 state transition을 실행하는 코드 예제이다.

```rust
use std::sync::{Arc, LockResult};
use std::{env, thread};
use std::time::{Duration, Instant};
use crossbeam::sync::{ShardedLock, WaitGroup};
use crate::Event::IORequest;

#[derive(Debug)]
enum State {
    Stage1,
    Stage2,
}

enum Event {
    IORequest,
}

struct IoHandler {
    state: State,
}

impl IoHandler {
    fn handle(&self, _e: Event) {
        // do something with the Event
    }
    fn change_state(&mut self, next_state: State) {
        println!("Changing state from {:?} to {:?}", self.state, next_state);
        self.state = next_state;
    }
}

fn main() {
    let args: Vec<String> = env::args().collect();
    let concurrency : u32 = (&args[1]).parse().unwrap();
 
    // custom struct 를 ShardedLock에 건네주어 초기화 하는 예제
    let io_handler = Arc::new(ShardedLock::new(IoHandler { state: State::Stage1 }));

    // 사용자에게 입력받은 수만큼, reader threads 들을 생성
    for worker_num in 0..concurrency {
        let io_handler = io_handler.clone();
        thread::spawn(move || {
            for _io_num in 0..100000000 {
                // IORequest의 read/write가 아니고,
                // State의 변경이 없으면 read,
                // State의 변경이 있으면 write, 라고 자체적으로 정의한 lock
                let read_lock = io_handler.read();
                match read_lock {
                    Ok(handle) => {
                        handle.handle(Event::IORequest);
                    }
                    // handle a case where the lock is poisoned (i.e., write-lock had been acquired but its owner died)
                    Err(_e) => todo!("not implemented yet")
                };
            }
            println!("Worker {worker_num} has finished I/O processing...");
        });
    }

    {
        // add 1 second of jitter in order for the readers to be ready
        thread::sleep(Duration::from_secs(1));
        let io_handler = io_handler.clone();
        let handle = thread::spawn(move || {
            let now = Instant::now();
            // State 변경을 원하므로, writer lock을 얻도록 한다.
            // 일단 reader threads 들이 모두 lock을 놓을 때까지
            // 대기해야 하므로, 획득 시간이 길어질 수 있다.
            // 그래서, 확인차 획득 시간을 출력해보도록 하자.
            let write_lock = io_handler.write();
            match write_lock {
                Ok(mut handler) => {
                    let elapsed = now.elapsed().as_micros();
                    println!("It took {elapsed} us to acquire write lock (vs. num_readers = {concurrency})");
                    handler.change_state(State::Stage2);
                }
                // handle a case where the lock is poisoned (i.e., write-lock had been acquired but its owner died)
                Err(_e) => todo!("not implemented yet")
            }
        });
        handle.join();
    }
}
```

`concurrency`를 바꾸어 가며 실행해보면 아래와 같다:
```bash
% cargo run --bin sharded-lock -- 1
It took 4 us to acquire write lock (vs. num_readers = 1)
Changing state from Stage1 to Stage2

% cargo run --bin sharded-lock -- 10
It took 53 us to acquire write lock (vs. num_readers = 10)
Changing state from Stage1 to Stage2

% cargo run --bin sharded-lock -- 100
It took 255405 us to acquire write lock (vs. num_readers = 100)
Changing state from Stage1 to Stage2

% cargo run --bin sharded-lock -- 1000
It took 3342588 us to acquire write lock (vs. num_readers = 1000)
Changing state from Stage1 to Stage2
```
예상한 것과 같이, reader threads의 숫자가 많아질 수록, writer thread가
exclusive access를 획득하기까지 시간이 더 걸리는 것을 볼 수 있다.
애초에 `RwLock` 타입의 lock을 쓴다는 것 자체가, read가 write보다 훨씬 많고
중요하다는 일종의 워크로드의 특성이 반영된 선택의 결과이므로, 받아들일 만하다.