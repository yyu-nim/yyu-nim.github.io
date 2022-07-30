---
layout: post
title:  "actix <4> - supervising without supervisor"
date:   2022-07-30 22:30:00 +0900
categories: actix rust selfsupervising
---

Actix의 [supervisor 기능](https://actix.rs/actix/actix/struct.Supervisor.html)은 제한적인데,
가령, supervised actor의 lifecycle event (예를 들면 restart)에 대한
notification을 외부의 actor에 message로 보내는 쉬운 방법이 없다는 점이 그 예이다.
직접적으로 말하면, 내 actor가 unwrap()으로 인한 실패, I/O failure로 인한 실패,
혹은 어떤 상태 오류로 인해 `ctx.stop()`을 호출해야 하는 경우등에, supervisor에게
에러 보고를 "쉽게" 하고 싶다! 현재, actix actor는 `getSender()` 와 같은 기능이 없기 
때문에, 수동으로 actor address를 injection 해주어야 하는 불편함이 존재한다.
아래의 시나리오를 생각해보자.
```text
MySupervisor -> MyChild: "child1"을 생성함
MySupervisor -> MyChild: "child1"을 멈춤
MyChild -> MyChild : 상태 정리하고 멈춤
MyChild -> MySupervisor : "child1"이 멈추었다는 이벤트가 보고됨
MySupervisor -> MySupervisor: 적당한 에러 처리
MySupervisor -> MyChild : "child1"을 다시 생성함
```
위와 같은 구현을 하려면, 
1) MySupervisor는 MyChild의 address를 기억하고 있어야 하고,
2) MyChild는 MySupervisor의 address를 기억하고 있어야 한다.

별일 아니라 생각할 수 있지만, actor 구성이 계층적으로 이루어지는 상황에서는 bolierplate
코드 작성이 늘어날 수 있다. 어쨌든, actix::Supervisor에는 위와 같은 
기능이 없으니, 직접 actor를 정의해서 비슷한 시도를 해본다면, 아래와 같다. 다음은,
MySupervisor actor 1개와 MyChild actor 2개를 띄우고, MyChild actor 각각을
1초 간격으로 3번씩 Stop 시켜보면서 MySupervisor가 제대로 restart를 해주는지
확인해보는 예제이다. 

```rust
use std::borrow::{Borrow, BorrowMut};
use std::collections::HashMap;
use std::time::Duration;
use actix::prelude::*;
use actix::{Actor, Context, Handler, System};

#[derive(Message)]
#[rtype(result = "()")]
struct PoisonPill;

#[derive(Message)]
#[rtype(result = "()")]
struct PoisonPillSwallowed {
    name: String,
}

#[derive(Message, Debug)]
#[rtype(result = "()")]
struct SpawnChild {
    name: String,
}

#[derive(Message, Debug)]
#[rtype(result = "()")]
struct StopChild {
    name: String,
}

#[derive(Message, Debug)]
#[rtype(result = "String")]
struct Report;

struct MySupervisor {
    name_to_addr: HashMap<String, Addr<MyChild>>,
}

struct MyChild {
    name: String,
    supervisor: Addr<MySupervisor>,
}

impl Actor for MySupervisor {
    type Context = Context<Self>;
}

impl Actor for MyChild {
    type Context = Context<Self>;

    fn stopping(&mut self, ctx: &mut Self::Context) -> Running {
        self.supervisor.do_send(PoisonPillSwallowed {
            name: self.name.to_string() }
        );

        Running::Stop
    }
}

impl Handler<PoisonPill> for MyChild {
    type Result = ();

    fn handle(&mut self, _: PoisonPill, ctx: &mut Context<MyChild>) {
        println!("MyChild {} is stopping...", self.name);
        ctx.stop();
    }
}

impl Handler<SpawnChild> for MySupervisor {
    type Result = ();

    fn handle(&mut self, msg: SpawnChild, ctx: &mut Self::Context) -> Self::Result {
        println!("MySupervisor has received {:?}...", msg);
        let name_to_addr = self.name_to_addr.borrow_mut();
        if name_to_addr.contains_key(&msg.name) {
            println!("The child {} exists already. Ignored.", msg.name);
        } else {
            let addr = MyChild {
                name: msg.name.to_string(),
                supervisor: ctx.address(),
            }.start();
            name_to_addr.insert(msg.name.to_string(), addr);
            println!("The child {} has been spawned.", msg.name);
        }
    }
}

impl Handler<StopChild> for MySupervisor {
    type Result = ();

    fn handle(&mut self, msg: StopChild, ctx: &mut Self::Context) -> Self::Result {
        println!("MySupervisor has received {:?}", msg);
        match self.name_to_addr.get(&msg.name) {
            None => {
                println!("MySupervisor is not aware of MyChild {}", msg.name);
            }
            Some(addr) => {
                addr.do_send(PoisonPill);
            }
        }
    }
}

impl Handler<PoisonPillSwallowed> for MySupervisor {
    type Result = ();

    fn handle(&mut self, msg: PoisonPillSwallowed, ctx: &mut Self::Context) -> Self::Result {
        match self.name_to_addr.remove(&msg.name) {
            None => {
                println!("MySupervisor is not aware of MyChild {}", msg.name);
            }
            Some(_) => {
                println!("MySupervisor is restarting MyChild {}", msg.name);
                let addr = MyChild {
                    name: msg.name.to_string(),
                    supervisor: ctx.address(),
                }.start();
                self.name_to_addr.insert(msg.name, addr);
            }
        }
    }
}

impl Handler<Report> for MySupervisor {
    type Result = String;

    fn handle(&mut self, msg: Report, ctx: &mut Self::Context) -> Self::Result {
        format!("{:?}", self.name_to_addr)
    }
}

fn main() {
    let mut sys = System::new();
    sys.block_on(async {
        let addr = MySupervisor { name_to_addr: HashMap::new() }.start();
        addr.do_send( SpawnChild { name: "child1".to_string() } );
        addr.do_send( SpawnChild { name: "child2".to_string() } );
        for _ in 0..3 {
            actix::clock::sleep(Duration::from_secs(1)).await;
            addr.do_send( StopChild { name: "child1".to_string() });
        }
        for _ in 0..3 {
            actix::clock::sleep(Duration::from_secs(1)).await;
            addr.do_send( StopChild { name: "child2".to_string() });
        }
        let report = addr.send(Report).await.unwrap();
        println!("MySupervisor report = {}", report);
    });

    sys.run();
}
```

실행하면, 다음과 같다:
```bash
MySupervisor has received SpawnChild { name: "child1" }...
The child child1 has been spawned.
MySupervisor has received SpawnChild { name: "child2" }...
The child child2 has been spawned.
MySupervisor has received StopChild { name: "child1" }
MyChild child1 is stopping...
MySupervisor is restarting MyChild child1
MySupervisor has received StopChild { name: "child1" }
MyChild child1 is stopping...
MySupervisor is restarting MyChild child1
MySupervisor has received StopChild { name: "child1" }
MyChild child1 is stopping...
MySupervisor is restarting MyChild child1
MySupervisor has received StopChild { name: "child2" }
MyChild child2 is stopping...
MySupervisor is restarting MyChild child2
MySupervisor has received StopChild { name: "child2" }
MyChild child2 is stopping...
MySupervisor is restarting MyChild child2
MySupervisor has received StopChild { name: "child2" }
MyChild child2 is stopping...
MySupervisor is restarting MyChild child2
MySupervisor report = {"child1": Addr { tx: AddressSender { sender_task: Mutex { data: SenderTask { task: None, is_parked: false } }, maybe_parked: false } }, "child2": Addr { tx: AddressSender { sender_task: Mutex { data: SenderTask { task: None, is_parked: false } }, maybe_parked: false } }}
```