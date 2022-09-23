---
layout: post
title:  "lazy_static <1> - global event channel"
date:   2022-09-24 01:10:00 +0930
categories: rust lazy_static event crossbeam
---

Thread-safe concurrent application을 작성할 때 자주 사용되는 방법은,
두 쓰레드간 channel을 만들어 message를 통해서만 상태 변환을 하는 것이다. 
메모리 주소를 다수의 쓰레드간 공유하는 방식과 비교해보면 lock의 획득 순서라던지, critical section 구간을
줄이기 위한 최적화 등의 노력을 덜해도, 혹은 아예 하지 않아도 되기 때문에 프로그래머의 개발 스트레스를 
상당히 줄여줄 수 있다.

문제는, "데이터를 보내는 쪽"과 "데이터를 받는 쪽"에 channel을 공유해주기 위해 수반되는 통합 작업이
생각보다 많은 노력이 들어 오히려 디자인적인 스트레스가 발생할 가능성이 있을 수 있다. 
만약, channel의 Sender/Receiver 위치가 서로 논리적인 거리가 먼, 그리고 소프트웨어 스택상 매우 깊은 위치에 
존재하고 있다면, 다수의 layer를 통과하면서 channel에 대한 참조를 유지할 수 밖에 없는 상황이 생기고 
이 경우 기존 디자인된 API들에 영향을 줄 수 있기 때문이다.

만약, channel간 주고받는 message의 성격이 외부 API 수준으로 노출될 정도의 중요도를 갖는게 아니라고 판단이
되어서, 코드상 어디에서나 쉽게 참조 가능한 형태로 만들고 싶은 경우에는 어떤 방법이 있을지? 
구체적으로 예를 들면, global event channel 같은 것을 만들어서, 소프트웨어 스택상 어느 위치에서라도 손쉽게 
정의한 이벤트를 보내거나 받아볼 수 있을지?

`lazy_static`이 여기서 역할을 해 줄 수 있다. 아래의 예제는 `EVENT_CHANNEL`라는 mpmc channel을 
static scope으로 선언한 뒤, 5개의 메시지 보내는 쓰레드와 1개의 메시지 받는 쓰레드를 생성하여, 위에 생성한
채널로 그것들을 연결한 것이다.
여기서 눈여겨 볼 부분은, sender thread와 receiver thread간에     "**명시적인 
channel 공유 및 의존성 주입을 하는 부분이 없고**", 그 대신 static scope에 정의된 channel을 `clone()`해서 
해당 시점에 즉시 event를 전달할 수 있게 되었다는 점이다.   


```rust
use std::borrow::{Borrow, BorrowMut};
use std::sync::{mpsc, Mutex};
use std::thread;

use crossbeam::channel::{Receiver, RecvError, Sender, unbounded};
use crossbeam::sync::WaitGroup;
use lazy_static::lazy_static;

lazy_static!{
    static ref EVENT_CHANNEL: Mutex<(Sender<EventType>, Receiver<EventType>)> = {
        let chan = unbounded();
        let tx : Sender<EventType> = chan.0;
        let rx : Receiver<EventType> = chan.1;
        Mutex::new((tx, rx))
    };
}

pub enum EventType {
    DO_WORK,
    REPORT_WORK,
    POISONPILL,
}

pub fn event_sender() -> Sender<EventType> {
    let event_channel = &EVENT_CHANNEL;
    event_channel.lock().unwrap().borrow_mut().0.clone()
}

pub fn event_receiver() -> Receiver<EventType> {
    let event_channel = &EVENT_CHANNEL;
    event_channel.lock().unwrap().borrow_mut().1.clone()
}

fn main() {
    let wg_receivers = WaitGroup::new();
    spawn_msg_receivers(1, &wg_receivers);

    let wg_senders = WaitGroup::new();
    spawn_msg_senders(5, 1000, &wg_senders);
    wg_senders.wait();

    let mut sender = event_sender();
    sender.send(EventType::REPORT_WORK);
    sender.send(EventType::POISONPILL);
    sender.send(EventType::POISONPILL);
    wg_receivers.wait();
}

fn spawn_msg_senders(num_senders: u32, num_reqs: u32, wait_group: &WaitGroup) {
    for id in 0..num_senders {
        let wait_group = wait_group.clone();
        thread::spawn(move || {
            let sender = event_sender();
            for _req_num in 0..num_reqs {
                sender.send(EventType::DO_WORK);
            }
            println!("Sender {} has sent {} works", id, num_reqs);
            wait_group.wait();
        });
    }
}

fn spawn_msg_receivers(num_receivers: u32, wait_group: &WaitGroup) {
    for id in 0..num_receivers {
        let wait_group = wait_group.clone();
        thread::spawn(move || {
            let receiver = event_receiver();
            let mut work_done = 0;
            loop {
                match receiver.recv() {
                    Ok(EventType::DO_WORK) => {
                        work_done += 1;
                        continue
                    },
                    Ok(EventType::REPORT_WORK) => {
                        println!("Receiver {} has processed {} works", id, work_done);
                        continue;
                    }
                    Ok(EventType::POISONPILL) => break,
                    Err(_e) => break,
                };
            }
            wait_group.wait();
        });
    }
}
```

실행 결과는 아래와 같다.

```bash
$ cargo run
Sender 4 has sent 1000 works
Sender 2 has sent 1000 works
Sender 1 has sent 1000 works
Sender 3 has sent 1000 works
Sender 0 has sent 1000 works
Receiver 0 has processed 5000 works
```