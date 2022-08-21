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

여기까지 간략히 설명을 마치고, 아래의 예제에 대해 생각해 보자.
RocksDB의 Write-Ahead-Log (WAL) entries를 담는 파일이 있다고 가정해 보자.
WAL entries들은 여러 threads에 의해 생성되어 저장되어야 하고, 
저장된 WAL entries들은 역시 또다른 여러 threads에 의해 읽히고 지워진다고 가정해보자.
이를 구현하기 위해 ArrayQueue와 SegQueue중 어느 것을 선택해야 할지?

*(To Be Continued...)*