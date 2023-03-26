---
layout: post
title:  "crossbeam <8> - parker/unparker (feat. ChatGPT)"
date:   2023-03-25 22:15:00 +0930
categories: rust crossbeam chatgpt parker tcp
---

이전의 [crossbeam-parker/unparker](/rust/crossbeam/parker/tcp/2022/08/17/crossbeam-parker-tcp.html) 포스팅에서 
TCP state transition을 Parker/Unparker API를 사용해서 시뮬레이션 하는 예제를 
공유하였었다. 이걸 ChatGPT에게 시키면 어떻게 예제를 작성해줄 수 있을까 문득 궁금해졌다.

> Can you use parker/unparker from crossbeam crate to simulate a TCP state transition?
> > ChatGPT: Here's an example implementation of a simplified TCP 
> > state machine that uses Parker and Unparker to simulate the blocking behavior of TCP state transitions:

```rust
use crossbeam::{Parker, Unparker};
use std::thread;

#[derive(Debug, PartialEq)]
enum TcpState {
    Closed,
    Listen,
    SynReceived,
    SynSent,
    Established,
    CloseWait,
    LastAck,
    FinWait1,
    FinWait2,
    TimeWait,
    Closing,
}

struct TcpConnection {
    state: TcpState,
    parker: Parker,
    unparker: Unparker,
}

impl TcpConnection {
    fn new(state: TcpState) -> Self {
        let (parker, unparker) = Parker::new();
        Self {
            state,
            parker,
            unparker,
        }
    }

    fn send_syn(&mut self) {
        match self.state {
            TcpState::Listen => {
                self.state = TcpState::SynReceived;
                self.parker.park();
            }
            _ => panic!("invalid state for sending SYN"),
        }
    }

    fn receive_syn(&mut self) {
        match self.state {
            TcpState::Closed => {
                self.state = TcpState::SynSent;
                self.parker.park();
            }
            TcpState::SynReceived => {
                self.state = TcpState::Established;
                self.unparker.unpark();
            }
            _ => panic!("invalid state for receiving SYN"),
        }
    }

    fn send_fin(&mut self) {
        match self.state {
            TcpState::Established => {
                self.state = TcpState::FinWait1;
                self.parker.park();
            }
            TcpState::CloseWait => {
                self.state = TcpState::LastAck;
                self.parker.park();
            }
            _ => panic!("invalid state for sending FIN"),
        }
    }

    fn receive_fin(&mut self) {
        match self.state {
            TcpState::Established => {
                self.state = TcpState::CloseWait;
                self.parker.park();
            }
            TcpState::FinWait1 => {
                self.state = TcpState::FinWait2;
                self.parker.park();
            }
            TcpState::FinWait2 => {
                self.state = TcpState::TimeWait;
                self.parker.park();
            }
            TcpState::CloseWait => {
                self.state = TcpState::LastAck;
                self.parker.park();
            }
            _ => panic!("invalid state for receiving FIN"),
        }
    }
}

fn main() {
    let mut conn = TcpConnection::new(TcpState::Closed);

    // send SYN
    thread::spawn(move || {
        conn.send_syn();
        println!("SYN sent, state: {:?}", conn.state);
    });

    // receive SYN
    thread::spawn(move || {
        thread::sleep(std::time::Duration::from_secs(1));
        conn.receive_syn();
        println!("SYN received, state: {:?}", conn.state);
    });

    // send FIN
    thread::spawn(move || {
        thread::sleep(std::time::Duration::from_secs(2));
        conn.send_fin();
        println!("FIN sent, state: {:?}", conn.state);
    });

    // receive FIN
    thread::spawn(move || {
        thread::sleep(std::time::Duration::from_secs(3));
        conn.receive_fin();
        println!("FIN received, state: {:?}", conn.state);
    });

    thread::sleep(std::
    (잘림)

```

역시, 출력 도중 잘리기는 했지만, 훌륭하다! enum 을 이용한 TCP 상태의 정의라던가,
thread spawn후 약간의 jitter를 주어 원하는 상태를 만들때까지 일부러 딜레이를 주는
일종의 휴리스틱까지 (이런 건 인간적인 느낌도 나서 신기함), 조금만 다듬어서 쓰면 원 포스팅의
예제보다 훨씬 좋은 퀄리티로 블로그 작성이 가능할 것 같다.