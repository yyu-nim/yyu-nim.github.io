---
layout: post
title:  "actix <5> - recipient vs. addr"
date:   2022-07-31 10:00:00 +0900
categories: actix rust recipient actor
---

Actix에서 actor에게 메시지를 보낼 때, 두 종류의 addressing을 사용할 수 있다.
한 가지는 Addr<T> 이고, 다른 한 가지는 Recipient<R> 이다. 둘 모두
`do_send`, `send` 를 사용하여 메시지를 보내기 때문에 유사해 보이는데,
결정적인 차이는, Addr의 generic type "T"는 actor type이 들어가고,
Recipient의 generic type "R"은 message type이 들어간다는 점이다.
즉, message를 보낼 때 받고자 하는 Actor의 정확한 type을 항상 알 수 있는 경우라면
`Addr`을 쓰면 되고, 특정 메시지의 핸들러만 구현 여부만 중요하다면 
`Recipient`를 쓰면 된다.

아래의 예제는, NVMe ANA multipath를 모사하여 path 변경을 Recipient를 이용하여
표현한 경우이므로 참고해볼 수 있다.

```rust
use actix::{Actor, Addr, Context, Handler, Message, Recipient, System};
use actix::dev::RecipientRequest;

#[derive(Message, Debug)]
#[rtype(result = "()")]
struct IoRequest;

#[derive(Message, Debug)]
#[rtype(result = "()")]
struct ChangePath;

struct MultipathActor {
    activated: Recipient<IoRequest>,
    use_optimized: bool,
}
struct OptimizedPathActor {
    num_received: u32,
}
struct NonOptimizedPathActor {
    num_received: u32,
}

impl Actor for MultipathActor {
    type Context = Context<Self>;
}

impl Actor for OptimizedPathActor {
    type Context = Context<Self>;
}

impl Actor for NonOptimizedPathActor {
    type Context = Context<Self>;
}

impl Handler<IoRequest> for MultipathActor {
    type Result = ();

    fn handle(&mut self, msg: IoRequest, ctx: &mut Self::Context) -> Self::Result {
        self.activated.do_send(msg);
    }
}

impl Handler<IoRequest> for OptimizedPathActor {
    type Result = ();

    fn handle(&mut self, _msg: IoRequest, _ctx: &mut Self::Context) -> Self::Result {
        self.num_received += 1;
        println!("Optimized path has received {} requests...", self.num_received);
    }
}

impl Handler<IoRequest> for NonOptimizedPathActor {
    type Result = ();

    fn handle(&mut self, _msg: IoRequest, _ctx: &mut Self::Context) -> Self::Result {
        self.num_received += 1;
        println!("Non-optimized path has received {} requests...", self.num_received);
    }
}

impl Handler<ChangePath> for MultipathActor {
    type Result = ();

    fn handle(&mut self, msg: ChangePath, ctx: &mut Self::Context) -> Self::Result {
        let new_path = {
            // Flip the path
            if self.use_optimized {
                self.use_optimized = false;
                NonOptimizedPathActor { num_received: 0 }.start().recipient()
            } else {
                self.use_optimized = true;
                OptimizedPathActor { num_received: 0 }.start().recipient()
            }
        };
        self.activated = new_path;
    }
}

fn main() {
    let mut sys = System::new();
    sys.block_on(async {
        let optimized_path_actor = OptimizedPathActor { num_received: 0 }.start();
        let addr = MultipathActor {
            activated: optimized_path_actor.recipient(),
            use_optimized: true
        }.start();

        for _ in 0..5 {
            addr.do_send(IoRequest);
        }

        // Change the path
        addr.do_send(ChangePath);

        for _ in 0..5 {
            addr.do_send(IoRequest);
        }

        // Change the path again
        addr.do_send(ChangePath);

        for _ in 0..5 {
            addr.do_send(IoRequest);
        }
    });

    sys.run();
}
```

이를 실행하면 다음과 같다.
```rust
Optimized path has received 1 requests...
Optimized path has received 2 requests...
Optimized path has received 3 requests...
Optimized path has received 4 requests...
Optimized path has received 5 requests...
Non-optimized path has received 1 requests...
Non-optimized path has received 2 requests...
Non-optimized path has received 3 requests...
Non-optimized path has received 4 requests...
Non-optimized path has received 5 requests...
Optimized path has received 1 requests...
Optimized path has received 2 requests...
Optimized path has received 3 requests...
Optimized path has received 4 requests...
Optimized path has received 5 requests...
```