---
layout: post
title:  "actix <6> - looking up addr from SystemRegistry"
date:   2022-08-01 01:00:00 +0900
categories: actix rust actor service registry
---

Actix의 문서에는 중요한 기능들에 대한 설명이 상당 부분 빠져 있는데,
그 중 하나가 [System Registry](https://docs.rs/actix/latest/actix/registry/struct.SystemRegistry.html) 이다.
[소스 코드](https://github.com/actix/actix/blob/master/actix/src/registry.rs)
 로부터 이해해보면, Actor를 `SystemService`라는 형태로 생성할 수 있는 방법이
있음을 알게 되는데, 이 경우 다음의 두 가지 혜택이 자동으로 따라온다:

1. Actor가 자동으로 `Supervised` 된다.
2. Actor의 (Type, Addr)이 자동으로 `SystemRegistry`에 등록이 된다.

1번에 대해서는 이전 두 포스트를 통해 설명되었으니 생략하도록 하고,
2번의 활용에 대한 설명으로 아래의 Chain Replication을 예제로 만들어 본다.

```text
NodeService ---> NodeActor1 ---> NodeActor2 ---> NodeActor3
    ↑                                                 |
    ---------------------------------------------------
```

`NodeService`로 (key, value)가 들어오면 NodeActor1, NodeActor2, NodeActor3
에 그 정보를 복제하고, 최종적으로는 NodeService에게 write success에 대한
ack를 리턴해주는 actor topology를 구성하고 싶다고 가정해보자.
쓸 수 있는 전략은 일반적으로, actor 간 message에 `Addr<NodeService>`를 계속
달고 다니면서, 마지막 NodeActor3에 이르렀을 때 한번만 사용하는 것이다. 당연히
동작하겠지만, NodeActor1, NodeActor2에서는 필요없을 정보를 들고 다니면서 
message를 이동시키는 것이 마음에 들지 않을 수 있다. 
두번째는, NodeActor3의 생성 시점에 `Addr<NodeService>`를 어떻게든 주입시키는 
것이다. Actor의 생성시점에 필요한 정보가 모두 있다는 가정하에, 무난한 방법이지만
`NodeService`가 Supervisor에 의해 재시작된 경우에도 NodeActor3가 기존
`Addr<NodeService>`를 그대로 쓸 수 있을지 명확치 않다.

이럴 때 사용할 수 있는 새로운 방법이 `SystemRegistry`에 등록된 `Addr<NodeService>`를
lookup 해서 write ack를 보내는 것이다. Actix의 내부 구현상, SystemRegistry를
직접 참조할 수는 없고, 다만 `SystemService`로 구현된 actor의 경우, 
`<ActorType>::from_registry()`의 형태로 Addr을 구해올 수 있다.
사용법에서 간접적으로 파악할 수 있듯, SystemService는 type별로 1개만 존재할 수 있으니, 
기존 c++ 등의 구현에서 singleton으로 디자인될 수 있는 객체는 SystemService actor로
만들어 볼 수 있다고 보면 되겠다.

이를 활용한 예제 코드는 아래와 같다.

```rust
use std::collections::HashMap;

use actix::{Actor, ActorContext, Addr, ArbiterService, Context, Handler, Message, Supervised, System, SystemRegistry, SystemService};

#[derive(Message, Debug)]
#[rtype(result = "()")]
struct WriteRequest {
    key: String,
    val: String,
}

#[derive(Message, Debug)]
#[rtype(result = "()")]
struct WriteResponse {
    key: String,
    val: String,
}

#[derive(Message, Debug)]
#[rtype(result = "()")]
struct AddFirstNode {
    node_actor: Addr<NodeActor>,
}

struct NodeService {
    first_node: Option<Addr<NodeActor>>,
}
impl Actor for NodeService {
    type Context = Context<Self>;
}
impl Supervised for NodeService {}
impl Default for NodeService {
    fn default() -> Self {
        NodeService {
            first_node: None,
        }
    }
}
impl SystemService for NodeService {
    fn service_started(&mut self, ctx: &mut Context<Self>) {
        println!("NodeService has started...");
    }
}
impl Handler<AddFirstNode> for NodeService {
    type Result = ();

    fn handle(&mut self, msg: AddFirstNode, ctx: &mut Self::Context) -> Self::Result {
        self.first_node = Some(msg.node_actor);
    }
}
impl Handler<WriteRequest> for NodeService {
    type Result = ();

    fn handle(&mut self, msg: WriteRequest, ctx: &mut Self::Context) -> Self::Result {
        if let Some(n) = &self.first_node {
            n.do_send(msg);
        } else {
            println!("{:?} cannot be replicated!", msg);
        }
    }
}
impl Handler<WriteResponse> for NodeService {
    type Result = ();

    fn handle(&mut self, msg: WriteResponse, ctx: &mut Self::Context) -> Self::Result {
        println!("{:?} has been fully replicated through the chain! You could return a success to the user", msg);
    }
}

struct NodeActor {
    name: String,
    map: HashMap<String, String>,
    next_actor: Option<Addr<NodeActor>>,
}

impl Actor for NodeActor {
    type Context = Context<Self>;
}

impl Handler<WriteRequest> for NodeActor {
    type Result = ();

    fn handle(&mut self, msg: WriteRequest, ctx: &mut Self::Context) -> Self::Result {
        let map = &mut self.map;
        map.insert(msg.key.clone(), msg.val.clone());
        println!("{} has replicated (key = {}, val = {}). Forwarding the same to the next actor...", self.name, msg.key, msg.val);
        match &self.next_actor {
            None => {
                println!("Reached the end of replication chain. Sending an ack to the first node (NodeService)");

                // 이런식으로, SystemRegistry에 자동으로 등록된 곳에서 Addr를 얻어올수 있다!
                let node_service_addr = NodeService::from_registry();
                node_service_addr.do_send(
                    WriteResponse { key: msg.key, val: msg.val });
            }
            Some(next_actor) => {
                next_actor.do_send(msg);
            }
        }
    }
}

fn main() {
    let mut sys = System::new();
    sys.block_on(async {
        let node3 = NodeActor {
            name: "node3".to_string(),
            map: Default::default(),
            next_actor: None
        }.start();
        let node2 = NodeActor {
            name: "node2".to_string(),
            map: Default::default(),
            next_actor: Some(node3),
        }.start();
        let node1 = NodeActor {
            name: "node1".to_string(),
            map: Default::default(),
            next_actor: Some(node2),
        }.start();

        // 아래와 같이 NodeService actor를 생성하면 자동으로 supervised 되면서 SystemRegistry에 등록된다.
        let node_service = NodeService::from_registry();
        node_service.do_send(AddFirstNode { node_actor: node1 });
        node_service.do_send(WriteRequest { key: "k1".to_string(), val: "v1".to_string() });
        node_service.do_send(WriteRequest { key: "k2".to_string(), val: "v2".to_string() });
        node_service.do_send(WriteRequest { key: "k3".to_string(), val: "v3".to_string() });
    });

    sys.run();
}
```
이를 실행하면 다음과 같다.
```bash
NodeService has started...
node1 has replicated (key = k1, val = v1). Forwarding the same to the next actor...
node1 has replicated (key = k2, val = v2). Forwarding the same to the next actor...
node1 has replicated (key = k3, val = v3). Forwarding the same to the next actor...
node2 has replicated (key = k1, val = v1). Forwarding the same to the next actor...
node2 has replicated (key = k2, val = v2). Forwarding the same to the next actor...
node2 has replicated (key = k3, val = v3). Forwarding the same to the next actor...
node3 has replicated (key = k1, val = v1). Forwarding the same to the next actor...
Reached the end of replication chain. Sending an ack to the first node (NodeService)
node3 has replicated (key = k2, val = v2). Forwarding the same to the next actor...
Reached the end of replication chain. Sending an ack to the first node (NodeService)
node3 has replicated (key = k3, val = v3). Forwarding the same to the next actor...
Reached the end of replication chain. Sending an ack to the first node (NodeService)
WriteResponse { key: "k1", val: "v1" } has been fully replicated through the chain! You could return a success to the user
WriteResponse { key: "k2", val: "v2" } has been fully replicated through the chain! You could return a success to the user
WriteResponse { key: "k3", val: "v3" } has been fully replicated through the chain! You could return a success to the user
```