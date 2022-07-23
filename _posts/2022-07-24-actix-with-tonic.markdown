---
layout: post
title:  "actix-broker <2> - grpc tonic integration"
date:   2022-07-24 08:00:00 +0900
categories: actix broker rust tonic
---

actix runtime에서 grpc tonic을 사용할 수 있는 방법의 핵심은, `Box::pin()`을 사용하는 것.
다음의 예제는, 10개의 grpc client를 사용해서 HelloRequest를 총 1만개 보내고, 최종적으로
각 actor로부터 report를 수집해서 출력하는 예제임.
예제 코드는 [tonic 공식 repo](https://github.com/hyperium/tonic/blob/master/examples/helloworld-tutorial.md)
를 참조하였고, Box::pin() 을 사용한 async execution 해결책은 [async-actix](https://kelvinfan001.github.io/async-actix/)를 
참조하였음. broker는 여기서 사용하지 않았음. topology는 다음과 같음.

```text
main runtime ---- HelloworldClient 0  ---
              |-- HelloworldClient 1  --|
              |-- HelloworldClient 2  --|------ (grpc channel) --->   GreeterServer
              |-- ...                 --|
              --- HelloworldClient 9  ---
```


#### src/helloworld-client-actor.rs

```rust
use actix::prelude::*;
use hello_world::greeter_client::GreeterClient;
use hello_world::HelloRequest;
use rand::Rng;

pub mod hello_world {
    tonic::include_proto!("helloworld");
}

/***
Actors
 */
struct HelloworldClient {
    greeter_client: GreeterClient<tonic::transport::Channel>,
    num_reqs_processed: u64,
}

/***
Actor Messages
 */

#[derive(Debug, Clone, Message)]
#[rtype(result = "()")]
struct ReqForSayHello;

#[derive(Debug, Clone, Message)]
#[rtype(result = "u64")]
struct ReqForReport;

/***
Customizing Actor Init logic
 */
impl Actor for HelloworldClient {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        println!("HelloworldClient started");
    }
}

/***
Actor Handlers
 */
impl Handler<ReqForSayHello> for HelloworldClient {
    type Result = ();

    fn handle(&mut self, msg: ReqForSayHello, ctx: &mut Self::Context) -> Self::Result {
        let addr = ctx.address();
        println!("{:?} has received ReqForSayHello. Issuing gRPC to server...", addr);
        let mut grpc_client = self.greeter_client.clone();

        // This enables "async execution" within the actix's actor runtime (backed by tokio)
        let fut = Box::pin(
            async move {
                let request = tonic::Request::new(HelloRequest{name: "tonic".to_string()});
                let response = grpc_client.say_hello(request).await;
                match response {
                    Ok(_) => {},
                    Err(e) => {
                        println!("failed to invoke say_hello(), caused by {:?}", e);
                    }
                }
            }
        );
        let actor_future = fut.into_actor(self);
        ctx.spawn(actor_future);
        self.num_reqs_processed += 1; // ideally, this should be Arc::clone()'d and passed to the actor future
    }
}

impl Handler<ReqForReport> for HelloworldClient {
    type Result = u64;

    fn handle(&mut self, msg: ReqForReport, ctx: &mut Self::Context) -> Self::Result {
        self.num_reqs_processed
    }
}

fn main() {
    let sys = System::new();
    sys.block_on(async {
        let mut client = GreeterClient::connect("http://[::1]:50051").await.unwrap();
        let mut actor_addrs = Vec::new();

        // Spawn 10 grpc actors
        for _ in 0..10 {
            let grpc_cloned = client.clone();
            let hello_client = HelloworldClient {
                greeter_client: grpc_cloned,
                num_reqs_processed: 0,
            };
            let addr = hello_client.start();
            actor_addrs.push(addr);
        }

        // Deliver 10000 messages to those random actors
        for _ in 0..10000 {
            let rand_idx = rand::thread_rng().gen_range(0..actor_addrs.len());
            let addr = actor_addrs.get(rand_idx);
            match addr {
                None => { panic!("failed to pick a random actor addr!") }
                Some(addr) => {
                    addr.do_send(ReqForSayHello);
                }
            }
        }

        // Synchronously "ask" those actors for the reports
        let mut report = Vec::new();
        for (i, actor_addr) in actor_addrs.iter().enumerate() {
            let res = actor_addr.send(ReqForReport).await;
            report.push(format!("actor= {:?}, num_reqs_processed = {:?}", i, res.unwrap()));
        }
        let r = report.join("\n").to_string();

        // Print out the report
        println!("{}", r);
    });
    sys.run().unwrap();
}
```

#### src/helloworld-server.rs
```rust
use tonic::{transport::Server, Request, Response, Status};

use hello_world::greeter_server::{Greeter, GreeterServer};
use hello_world::{HelloReply, HelloRequest};

pub mod hello_world {
    tonic::include_proto!("helloworld"); // The string specified here must match the proto package name
}

#[derive(Debug, Default)]
pub struct MyGreeter {}

#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>, // Accept request of type HelloRequest
    ) -> Result<Response<HelloReply>, Status> { // Return an instance of type HelloReply
        println!("Got a request: {:?}", request);

        let reply = hello_world::HelloReply {
            message: format!("Hello {}!", request.into_inner().name).into(), // We must use .into_inner() as the fields of gRPC requests and responses are private
        };

        Ok(Response::new(reply)) // Send back our formatted greeting
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    let greeter = MyGreeter::default();

    Server::builder()
        .add_service(GreeterServer::new(greeter))
        .serve(addr)
        .await?;

    Ok(())
}
```

#### proto/helloworld.proto
```protobuf
syntax = "proto3";
package helloworld;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

#### cargo.toml
```toml
[package]
name = "actix-broker-test"
version = "0.1.0"
edition = "2021"

[dependencies]
actix = "0.13"
actix-broker = "0.4.3"
tonic = "0.7"
prost = "0.10"
rand = "0.8.5"
tokio = { version = "1.0", features = ["macros", "rt-multi-thread"] }

[build-dependencies]
tonic-build = "0.7"
```

#### Executions
```bash
# terminal 1
$ cargo run --bin helloworld-server
```

```bash
# terminal 2
$ cargo run --bin helloworld-client-actor
...
actor= 0, num_reqs_processed = 1024
actor= 1, num_reqs_processed = 983
actor= 2, num_reqs_processed = 999
actor= 3, num_reqs_processed = 1040
actor= 4, num_reqs_processed = 1013
actor= 5, num_reqs_processed = 951
actor= 6, num_reqs_processed = 1051
actor= 7, num_reqs_processed = 963
actor= 8, num_reqs_processed = 965
actor= 9, num_reqs_processed = 1011
```