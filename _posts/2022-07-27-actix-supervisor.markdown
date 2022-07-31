---
layout: post
title:  "actix <3> - supervisor strategy"
date:   2022-07-27 22:30:00 +0900
categories: actix rust supervisor
---

Actix의 supervisor는 자신이 생성한 actor가 동작을 멈출 경우, 
`restart` 해주는 역할을 한다. "무조건 restart하는 것"이 현재 지원되는 유일한
failure handling 전략으로 보이며, 추후에는 escalation, restart conditionally 등의
전략을 지원할 수 있기를. 
아래의 예제는 `MyActor`를 의도적으로 3번 stop 시키고, 
마지막에는 몇번 restart되었는지를 확인하여 표준 출력하는 프로그램이다.

```mermaid
sequenceDiagram
    participant SystemRunner
    participant MyActor
    SystemRunner->>MyActor: PoisonPill
    MyActor->>MyActor: Restarted
    SystemRunner->>MyActor: PoisonPill
    MyActor->>MyActor: Restarted
    SystemRunner->>MyActor: PoisonPill
    MyActor->>MyActor: Restarted
    SystemRunner->>MyActor: ReportNumRestarts
    MyActor->>SystemRunner: 3
```

{% mermaid %}
graph TD;
A-->B;
A-->C;
B-->D;
C-->D;
{% endmermaid %}

`actix::Supervised::restarting()` 에서 `self.num_restarted` 의
값이 reset되지 않고 이전 값이 남아 있음을 관심있게 볼 것. 
즉 actor를 재시작한다고 해서 가지고 있던 상태를 초기화하는 것이 아니고, 
재사용하되 execution context만 초기화된다는 점. 


```rust
use actix::prelude::*;
use actix::{Actor, Context, Handler, System};

#[derive(Message)]
#[rtype(result = "()")]
struct PoisonPill;

#[derive(Message)]
#[rtype(result = "u32")]
struct ReportNumRestarts;

struct MyActor {
    num_restarted: u32,
}

impl Actor for MyActor {
    type Context = Context<Self>;
}

impl actix::Supervised for MyActor {
    fn restarting(&mut self, ctx: &mut Context<MyActor>) {
        self.num_restarted += 1; // 여기에서 값이 초기화되지 않고 누적됨!
        println!("restarting {} times...", self.num_restarted);
    }
}

impl Handler<PoisonPill> for MyActor {
    type Result = ();

    fn handle(&mut self, _: PoisonPill, ctx: &mut Context<MyActor>) {
        ctx.stop(); // 독약을 먹여서 (!), 의도적으로 중단시킴
    }
}

impl Handler<ReportNumRestarts> for MyActor {
    type Result = u32;

    fn handle(&mut self, msg: ReportNumRestarts, ctx: &mut Self::Context) -> Self::Result {
        self.num_restarted
    }
}

fn main() {
    let mut sys = System::new();
    sys.block_on(async {
        // 현재 Supervisor의 전략은 restart가 유일한 듯.
        let addr = actix::Supervisor::start(|_| MyActor { num_restarted: 0 });
        addr.do_send(PoisonPill);
        addr.do_send(PoisonPill);
        addr.do_send(PoisonPill);
        let num_restarts = addr.send(ReportNumRestarts).await.unwrap();
        println!("MyActor has restarted {} times", num_restarts);
    });

    sys.run();
}
```

실행하면, 다음과 같다.
```bash
$ cargo run
restarting 1 times...
restarting 2 times...
restarting 3 times...
MyActor has restarted 3 times
```