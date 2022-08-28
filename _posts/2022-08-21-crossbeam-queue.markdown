---
layout: post
title:  "crossbeam <7> - non-blocking concurrent queues"
date:   2022-08-22 00:01:00 +0930
categories: rust crossbeam concurrent queue
---

코드 리뷰 중에, 누군가가 만약 queue에 아이템을 넣고 빼는 연산의 전후에 lock을 잡고 획득하고
풀어주는 패턴으로 작성한 것을 보았다면, 보통은 아래와 같은 코멘트를 달게 될 것이다:
> 본 PR에서 사용된 queue가 thread-safe 하지 않기 때문에, lock을 사용하신 것으로 이해하면 될까요?
> Lock이 coarse-grained 하게 사용되어서 critical section이 너무 커진 것 같고
> 그래서 concurrency로 얻을 수 있는
> 이득이 크게 줄어들 수 있을 것 같은데, concurrent queue를 사용하는 것은 어떨까요? 

PR author가 이같은 피드백을 받아들여 검색을 해본다면, crossbeam의 [ArrayQueue](https://docs.rs/crossbeam/latest/crossbeam/queue/struct.ArrayQueue.html)
/ [SegQueue](https://docs.rs/crossbeam/latest/crossbeam/queue/struct.SegQueue.html) 를
금방 찾을 수 있을 것이다. 그리고는 금방 고민에 빠지게 될 것이다. **둘 중 무엇을 사용해야 한단 말인가?**

crossbeam의 문서에 적힌, 그 둘의 one liner summary는 아래와 같다:

* **ArrayQueue**: a bounded MPMC queue that allocates a fixed-capacity buffer on construction
* **SegQueue**: an unbounded MPMC queue that allocates small buffers, segments, on demand.

개발자가 코드 개발에 집중하느라 문서에 상대적으로 신경을 덜 쓴 것인지 알 수는 없으나, 
좋은 구현에 걸맞는 좀 더 친절하고 목적을 잘 설명해줄 수 있는 설명이었으면 좋았겠다고 생각한다. 
일단 RUST라는
언어가 후발 주자이므로, 시장에서의 지배적인 위치에 있는 언어를 사용하다 온 사람들을 위해 간단히
비교 설명을 해 줄 수 있으면 좋았을 것이고, 또한 구현 디테일을 굳이 one liner에 설명할 필요가
없지 않았을까 생각해본다. RUST 사용자는 이런 one liner로부터 의도를 파악하고 알맞은 선택을
알아서 할 것이다 라는 기대로 의도가 있었다면, 그것은 불친절이라 볼 수 있다. 

[java.util.concurrent](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/package-summary.html) 패키지에
설명된 concurrent queue들을 보면 다음과 같이 설명되어 있다:
> * ArrayBlockingQueue: A bounded blocking queue backed by an array
> * ConcurrentLinkedQueue: An unbounded thread-safe queue based on linked nodes
> * DelayQueue: An unbounded blocking queue of Delayed elements, in which an element can only be taken when its delay has expired.
> * LinkedBlockingQueue: An optionally-bounded blocking queue based on linked nodes. 
> * LinkedTransferQueue: An unbounded TransferQueue based on linked nodes.
> * PriorityBlockingQueue: An unbounded blocking queue that uses the same ordering rules as class PriorityQueue and supplies blocking retrieval operations.
> * SynchronousQueue: A blocking queue in which each insert operation must wait for a corresponding remove operation by another thread, and vice versa.

각각의 용법에 대한 설명은 목적을 벗어나므로 생략하도록 하고 대신 [Baeldung](https://www.baeldung.com/java-concurrent-queues) 의
훌륭한 문서를 참고하자. 여기서 중요한 것은, queue의 속성이 "bounded냐 unbounded냐", 
"blocking이냐 non-blocking"이냐 이다. Java 문서에서는 bounded/unbounded 둘 중에 하나는
반드시 one liner에 표시가 되고, blocking의 경우 one liner에 표시가 된다 (지금보니, 
`LinkedTransferQueue`의 경우는 blocking임에도 불구하고 one liner에 표시가 안되어 있기는
함;). One liner 요약과 더불어 `BlockingQueue` [인터페이스](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/package-summary.html) 
의 구현 여부로도 queue의 속성을 파악할 수 있다.

이를 비추어 crossbeam의 concurrent queues 를 "다시" 설명하면, 아래와 같을 것이다:

* **ArrayQueue**: a bounded non-blocking thread-safe queue backed by an array
* **SegQueue**: an unbounded non-blocking thread-safe queue based on linked nodes

이제, 각 속성이 사용자에게 의미하는 바가 무엇일지를 살펴보면,

* *bounded*
  * Queue의 생성 시점에 그 최대 크기를 지정하는 경우임. 메모리의 최대 사용량을 제한하는 효과가
  있기 때문에, 프로그램 실행 도중 의도치 않게 heap objects 들을 지나치게 많이 생성하면서 
  OS anonymous page들의 swap thrashing이 발생하는 등의 side effect들이 생기지 않도록
  방지하는 효과가 있음. `array` 라는 용어에, bounded라는 속성이 내재해있다고 봐도 무방할 것. 
* *unbounded*
  * Queue의 크기가 논리적으로는 무제한인 경우임. 다만, 물리적인 메모리의 크기 제한이 있기 때문에
  이를 오해하여 queue에 아이템들을 쌓아두기만 하고 뽑아가지를 않는다면, 어느 시점 부터는 
  swap thrashing과 같은 side effect가 생기게 될 것이므로, producer-consumer 간에
  flow control 혹은 back pressure 등의 구현이 있어야 이슈없이 동작할 수 있을 것.
  `linked nodes`라는 용어에, unbounded라는 속성이 내재해있다고 봐도 무방할 것.
* *blocking*
  * Queue가 꽉 찼을 경우에 producer가 push(혹은 put) 동작중에 대기를 하게 된다던지, 
  Queue가 비어있을 경우에 consumer가 pop(혹은 put) 동작중에 대기를 하게 된다던지 하는 
  속성임.
* *non-blocking*
  * Queue가 꽉 찼을 경우에는 `Result`로 producer의 push 연산 성공 여부를, 
  Queue가 비었을 경우에는 `Option`으로 consumer의 pop 연산 성공 여부를 표현함.
  Producer/consumer 입장에서는 원하는 연산이 실패했을 경우의 handling을 반드시
  수행해주어야 하고, 이 과정에서 busy loop이 되는 것을 피할 수 있도록 필요하면
  thread sleep 혹은 async yield를 수행해줄 필요가 생김. 
* *thread-safe*
  * `Send + Sync` trait을 구현하고 있기 때문에, Arc::new()를 사용하여 여러 thread간에
  자료 구조의 공유가 가능함.
* *thread-unsafe*
  * `Send + Sync` trait을 구현하지 않고 있기 때문에, 다른 lock이나 mutex를 활용하여
  단일 thread만 자료구조에 접근 가능하도록 강제한 상황에서만 여러 thread간에 자료 구조의
  공유가 가능함 (예를 들면, Arc::new(Mutex::new())).

여기까지 간략히 설명을 마치고, Host와 Device간의 통신을 concurrent queue를 이용하는 
예제에 대해 생각해보도록 하자.
목적은, `Host`가 `Device`에게 I/O request를 보내고 그 결과값에 대한 notification을 받도록 하는 것이다.
대략, 아래와 같은 두가지 방식을 생각해볼 수 있다.

```text
1) Host --> Device (push)
   Device --> Host (push)
2) Host --> Send Queue (push)
   Device <-- Send Queue (pull)
   Device --> Completion Queue (push)
   Host <-- Completion Queue (pull)
```
첫번째는 Host가 Device에게 직접 요청을 보내고, 요청이 완료되면 interrupt를 통해 Host는
이를 알아채고, 등록해둔 handler를 호출하는 방식으로 볼 수 있다. ATA/IDE 같은 오래된 
스토리지 IO path에서 사용된 방식이고, AHCI/SCSI 등에서 command queueing 프로토콜의
도입으로 interrupt 발생 시, 어떤 요청에 대한 completion인지를 알 수 있도록 변화가 생기기는
했지만, 여전히 수십~수백 수준의 낮은 concurrency 만을 지원할 수 있었다.

위와 같은 프로토콜의 한계를 극복하고자 Device와의 통신을 (상대적으로 매우 큰) queue를 통해
concurrency를 극대화하도록 프로토콜이 변화해 왔는데, 스토리지 스택에서는 NVMe 이 등장이
이와 연관이 있다(고 생각한다). 두번째 방식에 언급된 것처럼, Host/Device간의 직접적인 통신에
의존하는 대신, 중간에 I/O 요청을 보내는데 사용되는 Send Queue (SQ), I/O 요청의 종료를
알리는데 사용되는 Completion Queue (CQ)를 사용해서, Host와 Device는 handshake의
필요성을 최소화하고 대신 pub/sub 스타일로 요청을 **massively-parallel** 하게 처리할 수 
있게 되었다.

이걸, concurrent queue를 이용해서 시뮬레이션을 해보도록 하자. 일단은, 아래와 같은
multi-threaded application을 만들고 싶다고 해보자:

```text
       |-- IO dispatcher          ---|                
       |-- IO dispatcher          ---|     |--- SQ ---|
Host --|-- ...                    ---|-----|          |----  Device
       |-- IO dispatcher          ---|     |--- CQ ---|
       |-- IO completion handler  ---|
```

간단히 말해, N개의 `I/O dispatcher`를 만들어서 I/O 요청을 동시에 최대한 보내고 싶고,
1개의 `I/O completion handler`를 만들어서 I/O 요청을 마무리하여 user application에게
ack를 보내고 싶다 (이걸 M개로 늘려도 상관은 없겠지만). 
Device에서는 `SQ`에 쌓인 요청을 한개씩 빼내어서 처리하고 싶고, 완료되면 그것을 
비동기적으로 `CQ`에 쌓아서 Host에게 알리고 싶다.

**SQ/CQ를 구현하기 위해서는 어떤 RUST 라이브러리를 사용해야 할까?** 

일단, SQ는 Host의 여러 thread에 의해 동시에 "push"가 일어날 것이다. 그리고 Device도 
Host의 thread와 같이 경쟁하여 "pop"을 성공시켜야 한다. 따라서 (user mutex 없이) 
thread-safe queue를 쓰는 것이 합리적일 것이다. Coarse-grained mutex를 써서 thread-safety를 만족시키면,
애초에 이런 queueing model로 구현을 할 이유가 없어지므로 취할 수 없는 방식.
두번째로, device의 성능 때문에
Host에 I/O 요청이 무한정 많이 쌓이게 되면, memory pressure가 생길 수 있으므로 (아마
Host보다는 Device가 느려서 그럴 가능성이 다분하므로), 어느 정도 bound를 두는 것이 좋을 
것이다. `crossbeam`의 두 concurrent queue는 모두 non-blocking이므로, 여기서는
선택의 여지가 없고, 사용성의 측면에서 blocking을 해줄 수 있도록 간단한 wrapper function을
만들어 두면 좋을 것이다. blocking을 할 수 있는 방법은 몇가지가 있는데, 본 예제에서는 
매우 단순하게 (busy-wait 할 수 있는 리스크를 안고) `thread::yield_now()`를 하여
OS에게 스케줄링 권한을 내어주는 전략을 취할 것이다. 

그래서 `ArrayQueue`를 사용하기로 하였다. 앞서 언급한 간략한 디자인을 바탕으로, 
아래와 같은 구현이 가능할 것이다:

```rust
use std::sync::Arc;
use std::thread;
use std::time::Duration;

use crossbeam::channel::{bounded, Receiver, RecvError, Sender, unbounded};
use crossbeam::queue::ArrayQueue;
use rand;
use rand::Rng;

const Q_SIZE : usize = 65536;
const CONCURRENT_DISPATCHERS : u8 = 8;
const MAX_SECTORS : i64 = 1000000000;
const PRINT_EVERY_N_SECONDS : u64 = 1;

fn main() {
    let submission_queue = Arc::new(ArrayQueue::new(Q_SIZE));
    let completion_queue = Arc::new(ArrayQueue::new(Q_SIZE));

    run_host(submission_queue.clone(), completion_queue.clone(), CONCURRENT_DISPATCHERS);
    run_device(submission_queue.clone(), completion_queue.clone());

    loop {
        println!("sq_len = {}, cq_len = {}", submission_queue.len(), completion_queue.len());
        thread::sleep(Duration::from_secs(PRINT_EVERY_N_SECONDS));
    }
}

fn run_host(sq: Arc<ArrayQueue<i64>>, cq: Arc<ArrayQueue<i64>>, num_dispatchers: u8) {

    fn start_io_dispatcher(sq: Arc<ArrayQueue<i64>>) {
        let mut rng = rand::thread_rng();
        loop {
            let lba = rng.gen_range(0..MAX_SECTORS);
            push_blocking(&sq, lba);
        }
    }

    fn start_io_completion_handler(cq: Arc<ArrayQueue<i64>>) {
        loop {
            let _lba = pop_blocking(&cq);
        }
    }

    for _ in 0..num_dispatchers {
        let sq = sq.clone();
        thread::spawn(move || {
            start_io_dispatcher(sq);
        });
    }

    thread::spawn(move || {
        start_io_completion_handler(cq);
    });
}

fn run_device(sq: Arc<ArrayQueue<i64>>, cq: Arc<ArrayQueue<i64>>) {

    fn start_io_processor(sq: Arc<ArrayQueue<i64>>, tx: Sender<i64>) {
        loop {
            let lba = pop_blocking(&sq);
            // do some processing (noop op.)
            // send the completion signal to other thread
            tx.send(lba);
        }
    }

    fn start_io_completer(cq: Arc<ArrayQueue<i64>>, rx: Receiver<i64>) {
        loop {
            let lba = match rx.recv() {
                Ok(lba) => { lba }
                Err(_) => {
                    break;
                }
            };

            push_blocking(&cq, lba);
        }
    }

    let (tx, rx) = unbounded(); // alternatively, we could use bounded channel.
    thread::spawn(move || {
        start_io_processor(sq, tx);
    });

    thread::spawn(move || {
        start_io_completer(cq, rx);
    });
}

fn push_blocking(q: &Arc<ArrayQueue<i64>>, v: i64) {
  loop {
    match q.push(v) {
      Ok(_) => {
        return;
      }
      Err(_) => {
        // the queue is full. waiting.
        thread::yield_now(); // the alternatives would be "channel, Mutex, or Condvar"
      }
    }
  }
}

fn pop_blocking(q: &Arc<ArrayQueue<i64>>) -> i64 {
  loop {
    match q.pop() {
      None => {
        thread::yield_now(); // the alternatives would be "channel, Mutex, or Condvar"
      }
      Some(v) => {
        return v;
      }
    };
  }
}
```

이를 실행하면, 아래와 같다:

```bash
$ cargo run
sq_len = 123, cq_len = 0
sq_len = 0, cq_len = 205
sq_len = 268, cq_len = 616
sq_len = 17, cq_len = 2
sq_len = 1902, cq_len = 122
sq_len = 1212, cq_len = 972
sq_len = 22, cq_len = 1283
sq_len = 1017, cq_len = 13
sq_len = 271, cq_len = 67
sq_len = 0, cq_len = 453
sq_len = 26, cq_len = 179
sq_len = 4751, cq_len = 49
sq_len = 33, cq_len = 213
sq_len = 1556, cq_len = 0
```

`sq_len`은 Host가 Device에 보냈으나 아직 처리되지 않은 I/O 요청들의 개수,
`cq_len`은 Device가 Host에게 완료를 알렸으나 아직 Host가 가져가지 않은 I/O 요청들의 개수라고 보면 되겠다.
본 예제의 포인트는, Mutex의 사용 없이 multi-threads 간에 안전하게/효율적으로 요청을 
주고 받을 수 있다는 점 (물론 ArrayQueue 내부적으로는 아주 작은 단위로 Mutex가 쓰였을 수 있지만).
 prometheus/grafana로 그래프로 보면 더 좋을 것 같은데, 그 부분은 추후 포스팅에 다룰 기회가 있기를.


(이번에는 [coolbear](https://fasterthanli.me)의 블로그에 영감을 받아 나름대로 만연체로 작성)


