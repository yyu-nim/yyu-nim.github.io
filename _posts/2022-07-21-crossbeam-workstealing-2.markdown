---
layout: post
title:  "crossbeam <2> - work-stealing queue II"
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
                let work = worker.steal();
                match work {
                    Steal::Empty => {
                        println!("Worker {worker_num} has processed {num_processed} works");
                        break;
                    }
                    Steal::Success(_) => {
                        num_processed += 1;
                    }
                    Steal::Retry => continue,
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

`worker.steal()`의 결과로 `Steal::Retry`가 나올 경우가 있음에 유의하고 반드시 처리해주어야 함.

```bash
$ cargo run
Worker 1 has processed 107729 works
Worker 5 has processed 93090 works
Worker 0 has processed 99753 works
Worker 3 has processed 96996 works
Worker 8 has processed 103692 works
Worker 2 has processed 100445 works
Worker 6 has processed 97571 works
Worker 4 has processed 101253 works
Worker 9 has processed 100601 works
Worker 7 has processed 98870 works
The number of remaining works: 0
```
