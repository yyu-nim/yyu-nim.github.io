---
layout: post
title:  "actix-broker - graceful shutdown <1>"
date:   2022-07-21 21:30:00 +0900
categories: actix broker rust
---


```markdown
TestDriver ------ GracefulShutdowner ------- PosAccessor
                                         |
                                         --- PeerAccessor
                                         |
                                         --- DbAccessor
```
위와 같이 actor 들을 구성해볼 수 있다.
* `TestDriver` -> `GracefulShutdowner` : 3 초후 graceful shutdown 하라는 메시지 보냄
* `GracefulShutdowner` -> `PosAccessor`, `PeerAccessor`, `DbAccessor`: shutdown을 준비하라는 메시지 보냄
* `PosAccessor`, `PeerAccessor`, `DbAccessor` -> `GracefulShutdowner`: shutdown 준비되었다는 메시지 보냄
* `GracefulShotdowner`: Actor System을 종료시킴

```rust
use std::thread;
use actix::prelude::*;
use actix_broker::{Broker, BrokerIssue, BrokerSubscribe, SystemBroker};
use std::time::Duration;

type BrokerType = SystemBroker;

/***
Actors
 */
// Broadcast "Prepare-Shutdown"
// Aggregate 3 * "I'm ready to be shut down"
// Once three ready messages have arrived, Stop the actor system
struct GracefulShutdowner {
    num_of_workers_to_wait: u32,
    num_received_ready_for_shutdown: u32,
}

struct PosAccessor {
    shutdown_prepared: bool,
}
struct PeerAccessor {
    shutdown_prepared: bool,
}
struct DbAccessor {
    shutdown_prepared: bool,
}
struct TestDriver;

/***
Actor Messages
 */
#[derive(Debug, Clone, Message)]
#[rtype(result = "()")]
struct GracefulShutdownMsg;
#[derive(Debug, Clone, Message)]
#[rtype(result = "()")]
struct PrepareShutdownMsg;
#[derive(Debug, Clone, Message)]
#[rtype(result = "()")]
struct ReadyForShutdownMsg {
    who_am_i: String,
}

/***
Customizing Actor Init logic
 */
impl Actor for GracefulShutdowner {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        println!("GracefulShutdowner started");
        self.subscribe_sync::<BrokerType, GracefulShutdownMsg>(ctx);
        self.subscribe_sync::<BrokerType, ReadyForShutdownMsg>(ctx);
    }
}
impl Actor for PosAccessor {
    type Context = Context<Self>;
    fn started(&mut self, ctx: &mut Self::Context) {
        println!("PosAccessor started");
        self.subscribe_sync::<BrokerType, PrepareShutdownMsg>(ctx);
    }
}
impl Actor for PeerAccessor {
    type Context = Context<Self>;
    fn started(&mut self, ctx: &mut Self::Context) {
        println!("PeerAccessor started");
        self.subscribe_sync::<BrokerType, PrepareShutdownMsg>(ctx);
    }
}
impl Actor for DbAccessor {
    type Context = Context<Self>;
    fn started(&mut self, ctx: &mut Self::Context) {
        println!("DbAccessor started");
        self.subscribe_sync::<BrokerType, PrepareShutdownMsg>(ctx);
    }
}
impl Actor for TestDriver {
    type Context = Context<Self>;
    fn started(&mut self, ctx: &mut Self::Context) {
        println!("TestDriver started. After 3 seconds, we inject shutdown message");
        thread::sleep(Duration::from_secs(3));
        println!("TestDriver is injecting graceful shutdown request");
        self.issue_async::<BrokerType, _>(GracefulShutdownMsg);
    }
}

/***
Actor Message Handlers
 */
impl Handler<GracefulShutdownMsg> for GracefulShutdowner {
    type Result = ();

    fn handle(&mut self, msg: GracefulShutdownMsg, _ctx: &mut Self::Context) {
        println!("GracefulShutdowner is starting to shut down... msg = {:?}", msg);
        self.num_received_ready_for_shutdown = 0;
        self.issue_async::<BrokerType, _>(PrepareShutdownMsg);
    }
}
impl Handler<ReadyForShutdownMsg> for GracefulShutdowner {
    type Result = ();

    fn handle(&mut self, msg: ReadyForShutdownMsg, ctx: &mut Self::Context) -> Self::Result {
        println!("GracefulShutdowner has received a ready message from {}", msg.who_am_i);
        self.num_received_ready_for_shutdown += 1;
        if self.num_received_ready_for_shutdown < self.num_of_workers_to_wait {
            println!("GracefulShutdowner is still waiting for more ready message... {}/{}",
                     self.num_received_ready_for_shutdown, self.num_of_workers_to_wait);
        } else {
            println!("GracefulShutdowner has received ready messages from all participants... {}/{}. Stopping actor system...",
                     self.num_received_ready_for_shutdown, self.num_of_workers_to_wait);
            System::current().stop();
            println!("Bye!");
        }
    }
}

impl Handler<PrepareShutdownMsg> for PosAccessor {
    type Result = ();

    fn handle(&mut self, msg: PrepareShutdownMsg, ctx: &mut Self::Context) -> Self::Result {
        if !self.shutdown_prepared {
            println!("PosAccessor is preparing for a shutdown... msg = {:?}", msg);
            println!("PosAccessor has finished its all outstanding I/Os to POS");
            self.issue_async::<BrokerType, _>(
                ReadyForShutdownMsg { who_am_i : "PosAccessor".to_string() }
            );
            self.shutdown_prepared = true;
        } else {
            println!("PosAccessor is already prepared for a shutdown... msg = {:?} will be discarded", msg);
        }
    }
}

impl Handler<PrepareShutdownMsg> for DbAccessor {
    type Result = ();

    fn handle(&mut self, msg: PrepareShutdownMsg, ctx: &mut Self::Context) -> Self::Result {
        if !self.shutdown_prepared {
            println!("DbAccessor is preparing for a shutdown... msg = {:?}", msg);
            println!("DbAccessor has finished its all outstanding I/Os to DB");
            self.issue_async::<BrokerType, _>(
                ReadyForShutdownMsg { who_am_i : "DbAccessor".to_string() }
            );
            self.shutdown_prepared = true;
        } else {
            println!("DbAccessor is already prepared for a shutdown... msg = {:?} will be discarded", msg);
        }
    }
}

impl Handler<PrepareShutdownMsg> for PeerAccessor {
    type Result = ();

    fn handle(&mut self, msg: PrepareShutdownMsg, ctx: &mut Self::Context) -> Self::Result {
        if !self.shutdown_prepared {
            println!("PeerAccessor is preparing for a shutdown... msg = {:?}", msg);
            println!("PeerAccessor has finished its all outstanding I/Os to Peer Replicator");
            self.issue_async::<BrokerType, _>(
                ReadyForShutdownMsg { who_am_i : "PeerAccessor".to_string() }
            );
            self.shutdown_prepared = true;
        } else {
            println!("PeerAccessor is already prepared for a shutdown... msg = {:?} will be discarded", msg);
        }
    }
}

fn main() {
    println!("Starting");
    let sys = System::new();

    sys.block_on(async {
        let shutdowner = GracefulShutdowner {
            num_of_workers_to_wait: 3,
            num_received_ready_for_shutdown: 0,
        };
        shutdowner.start();
        PosAccessor { shutdown_prepared: false }.start();
        PeerAccessor { shutdown_prepared: false }.start();
        DbAccessor { shutdown_prepared: false }.start();
        TestDriver.start();
    });
    sys.run().unwrap();
    println!("Done");
}
```