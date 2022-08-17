---
layout: post
title:  "crossbeam <5> - parker/unparker"
date:   2022-08-17 20:15:00 +0930
categories: rust crossbeam parker tcp
---

Multi-threaded 애플리케이션을 작성하다 보면, 두 thread 상태를 
정해진 프로토콜대로 변경하고 싶을 때가 생긴다. 이 때는 마치 악수를 하듯, 
서로의 상태를 확인하면서 메시지를 주고 받는 식의 구현이 필요한데, 기본적인 해법으로는 언어에서 제공하는
shared memory channel 혹은 network transport channel 등을 활용하여 
직접 메시지와 프로토콜을 구현하는 방법이 있겠다.

그런데, 둘 간에 주고 받아야 하는 메시지가 단지 "나는 준비되었으니 너도 움직여라" 
정도의 컨트롤 시그널만 담고 있다면, 메시지를 정의하고 채널을 연결하는 등의 
귀찮은 작업들을 대신 도와줄 라이브러리를 찾아보는 것을 고려해봄직 하다. 
`crossbeam`의 `Parker/Unparker`가 여기에서 도움을 줄 수 있다. `park()/unpark()`라는 단 두 개의 API만으로, 두 thread를 원하는 시점에
움직이게 하거나 기다리게 할 수 있다. 

[crossbeam 문서](https://docs.rs/crossbeam/0.8.2/crossbeam/sync/struct.Parker.html) 에서
기본적인 사용법은 숙지가 가능할 것이다. 본 포스트에서는, 
[TCP state transition](https://users.cs.northwestern.edu/~agupta/cs340/project2/TCPIP_State_Transition_Diagram.pdf) 을
`Parker/Unparker`로 시뮬레이션 하는 예제를 공유한다. 

![TCP](https://flylib.com/books/3/223/1/html/2/files/18fig12.gif)

위 그림은, TCP server/client가 메시지를 주고 받으며,
state transition을 하면서 최초 접속하는 것부터 해제까지의 흐름을 보여주는데,
실제 TCP에서는 메시지를 주고 받으며 "서로의 상태를 확인"하지만, 
본 시뮬레이션 예제에서는 `Parker/Unparker`를 통해 비슷한 기능을 구현해 본다.
어디까지나 Parker의 기능 이해를 돕기 위한 예제임에 유의할 것.

```rust
use std::thread;
use std::time::Duration;
use crossbeam::sync::{Parker, Unparker, WaitGroup};

fn main() {
    let client_parker = Parker::new();
    let client_unparker = client_parker.unparker().clone();
    let server_parker = Parker::new();
    let server_unparker = server_parker.unparker().clone();
    let finish_line = WaitGroup::new();

    {
        let finish_line = finish_line.clone();
        thread::spawn(move || {
            tcp_server(server_parker, client_unparker);
            finish_line.wait();
        });
    }

    {
        let finish_line = finish_line.clone();
        thread::spawn(move || {
            // server가 떠있어야 client가 접속할 수 있으므로, 1초 delay를 주자.
            thread::sleep(Duration::from_secs(1)); 
            tcp_client(client_parker, server_unparker);
            finish_line.wait();
        });
    }

    finish_line.wait();
    println!("TCP connection disconnected");
}


fn tcp_client(p: Parker, u: Unparker) {
    // CLOSED           -> SYN_SENT -> ESTABLISHED -> FIN_WAIT1 -> FIN_WAIT2 -> TIME_WAIT -> (timeout) -> CLOSED
    println!("[Client] CLOSED");
    println!("[Client] SYN_SENT");
    u.unpark();
    p.park();
    println!("[Client] ESTABLISHED");
    u.unpark();
    p.park();
    println!("[Client] FIN_WAIT_1");
    u.unpark();
    p.park();
    println!("[Client] FIN_WAIT_2");
    u.unpark();
    p.park();
    println!("[Client] TIME_WAIT");
    u.unpark();
    p.park_timeout(Duration::from_secs(1)); // 2MSL timeout
    println!("[Client] CLOSED");
}

fn tcp_server(p: Parker, u: Unparker) {
    // CLOSED -> LISTEN -> SYN_RCVD -> ESTABLISHED -> CLOSE_WAIT -> LAST_ACK                           -> CLOSED
    println!("[Server] CLOSED");
    println!("[Server] LISTEN");
    p.park();
    println!("[Server] SYNC_RCVD");
    u.unpark();
    p.park();
    println!("[Server] ESTABLISHED");
    u.unpark();
    p.park();
    println!("[Server] CLOSE_WAIT");
    u.unpark();
    p.park();
    println!("[Server] LAST_ACK");
    u.unpark();
    p.park();
    println!("[Server] CLOSED");
}
```

이를 실행하면, 아래와 같은 결과가 나온다:

```bash
$ cargo run
[Server] CLOSED
[Server] LISTEN
[Client] CLOSED
[Client] SYN_SENT
[Server] SYNC_RCVD
[Client] ESTABLISHED
[Server] ESTABLISHED
[Client] FIN_WAIT_1
[Server] CLOSE_WAIT
[Client] FIN_WAIT_2
[Server] LAST_ACK
[Client] TIME_WAIT
[Server] CLOSED
[Client] CLOSED
TCP connection disconnected
```

위와 같은 handshake를 어기면서 state transition이 발생하는 경우가 생긴다면 그것은
본 예제의 버그일 것. 
예를 들면, Client가 `SYN_SENT` -> `ESTABLISHED` 로 상태 전환을 하기 위해서는,
반드시 Server의 `SYNC_RCVD` 상태가 존재해야만 한다.
