---
layout: post
title:  "crossbeam <6> - mpmc channel"
date:   2022-08-19 01:00:00 +0930
categories: rust crossbeam mpmc channel k8s
---

[crossbeam 문서](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html) 에 보면,
`std::sync::mpsc`에는 없는 multi-producer multi-consumer channel 기능이 있다고 한다.
`std`에 포함된 채널은 multi-producer single-consumer channel을 구현하고 있는데, 
thread간 message passing을 구현하기 위한 primitive로 볼 수 있고, 가장 대중적인 사용법이라
볼 수 있겠다. Message를 보낼 때에는, 내가 원하는 thread를 특정하여 보내는 경우가 대부분일 것이기
때문이다.

Consumer가 여럿 있어서, 내가 보내는 메시지가 누구든 처리만 해주면 좋겠다 이런 use-case가
어떤 것이 있을지? 본 포스팅에서는, mpmc channel을 활용해서 쿠버네티스 (k8s)의 pod을 시뮬레이션하는
예제를 공유하려 한다. [k8s ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 에서는
`spec.replicas`라는 필드에 적힌 값으로 (혹은 kubectl을 사용해서 온디맨드로), 동일한 `pod`를
여러개 띄우고, 트래픽을 load balancing (LB) and/or high availability (HA)를 달성할 수 있다.
이를 mpmc channel의 관점에서 생각해보면, 각 `pod`가 `channel receiver`를 clone해서 가진 뒤,
누구든 먼저 message를 receive하는 쪽이 처리를 해주도록 하면 될 것이다.

아래의 예제에서는, 3개의 pod replica를 생성하면서 모두에게 동일한 channel receiver를 
전달해준다. 그리고,

* pod replica 3개가 `VALID_MSG`를 처리하도록 한다.
* pod replica 1개를 죽여, 나머지 2개가 `VALID_MSG`를 처리하도록 한다.
* pod replica 1개를 더 죽여, 나머지 1개가 `VALID_MSG`를 처리하도록 한다.
* pod replica 1개를 더 죽여, 아무도 `VALID_MSG`를 처리하지 못하도록 한다.
* pod replica 3개를 살려, 바로 이전의 없어진 5개의 `VALID_MSG`가 다시 처리되도록 한다.

```rust
use std::collections::HashMap;
use std::thread;
use std::thread::JoinHandle;
use std::time::Duration;
use crossbeam::channel::{Receiver, RecvError, SendError, unbounded};
use crossbeam::sync::WaitGroup;

enum Msg {
    POISONPILL,
    VALID_MSG,
}

fn main() {
    let replica_count = 3;
    let (s, r) = unbounded();
    let thread_handles = bringup_pod(&r, replica_count);

    for _ in 0..5 {
        s.send(Msg::VALID_MSG).expect("replica 3개 있으므로, 메시지 처리 가능해야 함");
    }

    s.send(Msg::POISONPILL);
    for _ in 0..5 {
        s.send(Msg::VALID_MSG).expect("replica 2개 있으므로, 메시지 처리 가능해야 함");
    }

    s.send(Msg::POISONPILL);
    for _ in 0..5 {
        s.send(Msg::VALID_MSG).expect("replica 1개 있으므로, 메시지 처리 가능해야 함");
    }

    s.send(Msg::POISONPILL);
    for _ in 0..5 {
        s.send(Msg::VALID_MSG).expect("replica 0개 있으므로, 메시지 처리 불가능해야 함");
    }

    for h in thread_handles {
        h.join();
    }

    println!("We will bring up 3 pod replica again and those 5 missing VALID_MSG will continue to be processed...");
    let thread_handles = bringup_pod(&r, replica_count);

    for h in thread_handles {
        h.join();
    }
}

fn bringup_pod(r: &Receiver<Msg>, replica_count: u32) -> Vec<JoinHandle<()>> {
    let start_line = WaitGroup::new();
    let mut thread_handles = Vec::new();
    for _ in 0..replica_count {
        let r = r.clone();
        let start_line = start_line.clone();
        let h = thread::spawn(move || {
            bringup_pod_replica(r, start_line);
        });
        thread_handles.push(h);
    }
    start_line.wait();
    thread_handles
}

fn bringup_pod_replica(r: Receiver<Msg>, start_line: WaitGroup) {
    println!("Bringing up pod replica: {:?}", thread::current().id());
    start_line.wait();

    loop {
        // 여기서 message를 받아서 처리 함. 각 pod가 아래 코드를 수행할 것이고, 
        // 누군가 1개의 pod이라도 남아있으면 메시지 처리가 가능해짐
        let msg = r.recv();
        match msg {
            Ok(msg) => {
                match msg {
                    Msg::POISONPILL => {
                        println!("Killing the pod replica: {:?}", thread::current().id());
                        break;
                    }
                    Msg::VALID_MSG => {
                        println!("Handling the msg by the pod replica: {:?}", thread::current().id());
                        thread::sleep(Duration::from_secs(2));
                    }
                };
            }
            Err(e) => {
                println!("Breaking the loop because of {}", e);
                break;
            }
        };
    }
}
```

이를 실행하면 아래와 같다:

```bash
$ cargo run
Bringing up pod replica: ThreadId(3)
Bringing up pod replica: ThreadId(2)
Bringing up pod replica: ThreadId(4)
Handling the msg by the pod replica: ThreadId(3)
Handling the msg by the pod replica: ThreadId(2)
Handling the msg by the pod replica: ThreadId(4)
Handling the msg by the pod replica: ThreadId(2)
Killing the pod replica: ThreadId(4)
Handling the msg by the pod replica: ThreadId(3)
Handling the msg by the pod replica: ThreadId(2)
Handling the msg by the pod replica: ThreadId(3)
Handling the msg by the pod replica: ThreadId(2)
Handling the msg by the pod replica: ThreadId(3)
Handling the msg by the pod replica: ThreadId(2)
Killing the pod replica: ThreadId(3)
Handling the msg by the pod replica: ThreadId(2)
Handling the msg by the pod replica: ThreadId(2)
Handling the msg by the pod replica: ThreadId(2)
Handling the msg by the pod replica: ThreadId(2)
Handling the msg by the pod replica: ThreadId(2)
Killing the pod replica: ThreadId(2)
We will bring up 3 pod replica again and those 5 missing VALID_MSG will continue to be processed...
Bringing up pod replica: ThreadId(5)
Bringing up pod replica: ThreadId(6)
Bringing up pod replica: ThreadId(7)
Handling the msg by the pod replica: ThreadId(7)
Handling the msg by the pod replica: ThreadId(5)
Handling the msg by the pod replica: ThreadId(6)
Handling the msg by the pod replica: ThreadId(5)
Handling the msg by the pod replica: ThreadId(6)
```

간단히 설명하면, 

1. 초반에는 ThreadId 2, 3, 4가 생성되어 메시지 처리
2. ThreadId 4가 죽고, 나머지 두개의 pod replica가 메시지 처리
3. ThreadId 3이 죽고, 나머지 한개의 pod replica가 메시지 처리
4. ThreadId 2가 죽고, 아무도 5개의 `VALID_MSG`를 처리하지 못함
5. ThreadId 5, 6, 7이 새로 생성되고, 채널에 남아있었던 5개의 `VALID_MSG`를 처리함.
