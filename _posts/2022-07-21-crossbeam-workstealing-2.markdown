---
layout: post
title:  "Crossbeam work-stealing queue <2>"
date:   2022-07-21 21:30:00 +0900
categories: crossbeam rust
---

100만개의 일을 만들어서, 10개의 worker들에게 나눠주기. 방식은 work-stealing으로
worker들이 일을 빼앗아 감.
```rust
use std::thread;
use std::time::Duration;
use crossbeam::deque::{Steal, Worker};

fn main() {
    let master = Worker::new_fifo();
    for work in 0..1000000 {
        master.push(work);
    }
    let mut worker_handle = Vec::new();
    let stealer = master.stealer();
    for worker_num in 0..10 {
        let worker = stealer.clone();
        let handle = thread::spawn(move || {
            let mut num_processed = 0;
            loop {
                if let Steal::Success(_) = worker.steal() {
                    num_processed += 1;
                } else {
                    println!("Worker {worker_num} has processed {num_processed} works");
                    break;
                }
            }
        });
        worker_handle.push(handle);
    }

    for handle in worker_handle {
        handle.join();
    }
    println!("The number of remaining works: {}", master.len());
}
```

```bash
$ cargo run
Worker 2 has processed 0 works
Worker 0 has processed 0 works
Worker 3 has processed 253 works
Worker 5 has processed 0 works
Worker 1 has processed 60 works
Worker 4 has processed 0 works
Worker 7 has processed 0 works
Worker 6 has processed 424 works
Worker 9 has processed 0 works
Worker 8 has processed 999263 works
The number of remaining works: 0
```

실행 결과를 보면, 특정 worker들만 일들을 가져가고 있는 현상들이 있는데
OS의 스케줄링 문제인지 crossbeam을 잘못 사용하고 있는 것인지 
확인이 필요함.
