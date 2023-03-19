---
layout: post
title:  "nvmf <2> - connect command"
date:   2023-03-19 23:00:00 +0930
categories: rust nvmf chatgpt connect command
---

ChatGPT는 매번 같은 코드를 주지 않는다. 마치 타로 카드와 같이, 쿼리를 하는 시점에 인간의 지각(?)을
넘어서는 거대한 상태에 의존하여 대답을 해주는 것 같다. 개발자 언어를 쓰자면, 결과값이
non-deterministic 하게 나온다고 느껴진다. 따라서, ChatGPT가 내어주는 부분적인 코드를 
그대로 복붙해서 전체 큰 프로그램을 완성하는 것은 당장은 가능해보이지 않고, 다만 다양한 방식으로
질문을 해서 핵심 정보를 추출하고 변형해 내도록 유도를 해내는 기술이 필요해 보인다.

이전 포스팅으로부터, `Connect`라는 OpCode를 담은 NVMe-oF 커맨드가 TCP 를 통해 도착할
수 있음을 배웠다. 그렇다면, `Connect`라는 커맨드는 어떤 상태 변화를 기대하면서 요청된 것일까?
NVMe-oF 스펙을 공부하면 알 수도 있겠지만, ChatGPT를 활용해서 최대한 시간을 아껴보자:

> Hi there, let's get back to where we left off. 
> Could you write a handler for NVMeoF connect command in RUST?

뭔가 코드를 짜주었다. 그런데 
```rust
// TODO: Handle the Connect command payload
```
이런식으로 제일 궁금한 부분한 빈칸으로 남겨두고 code snippet을 리턴해주었다. 집요하게 다시 물어보았다.
> Okay fine. Then, could you write an example of how to handle the Connect
> command payload?

이번 code snippet에서는 TCP stream에서 `hostid`, `cntlid`, `subnqn`이라는 걸
추출해서 stdout으로 출력해주기만 하였다. 이 값들이 왜 필요했을까? 깊은 고민은 접고, 물어보았더니
ChatGPT가 실토하였다.
> What should I do with hostid, cntlid, and subnqn? 
> Should I remember those for future I/O commands?
>> ChatGPT: Yes, you will need to remember hostid, 
>> cntlid, and subnqn for future I/O commands that may 
>> come from the host. These fields uniquely identify the initiator (host), controller, and subsystem namespace that are associated with the NVMeoF connection.

오호라. 이미 답을 알고 있으면서, 예제에는 일부러 생략을 하고 있었던 것이다.
아마도 대화가 고팠던지 아니면 free tier user에게 아쉬움을 주는 전략이던지. 
그래서 이 내용으로 // TODO를 채워달라 하였다. 그랬더니 아래와 같았다:

```rust
use std::io::{Read, Write};

const NVMEOF_COMMAND_CONNECT: u8 = 0x00;
const NVMEOF_COMMAND_CAPSULE: u8 = 0x01;
// ... other NVMeoF command opcodes ...

struct NvmeofConnection {
    hostid: u64,
    cntlid: u16,
    subnqn: String,
    // ... other connection-specific fields ...
}

impl NvmeofConnection {
    fn new(hostid: u64, cntlid: u16, subnqn: String) -> Self {
        Self {
            hostid,
            cntlid,
            subnqn,
            // ... initialize other connection-specific fields ...
        }
    }

    // ... define methods for I/O operations ...
}

fn handle_nvmeof_connect(stream: &mut std::net::TcpStream) -> Result<(), std::io::Error> {
    let mut buf = [0u8; 4096];
    let size = stream.read(&mut buf)?;

    // Parse the Connect command payload
    let connect_command = &buf[0..size];
    let hostid = u64::from_le_bytes(connect_command[8..16].try_into().unwrap());
    let cntlid = u16::from_le_bytes(connect_command[16..18].try_into().unwrap());
    let subnqn_len = connect_command[20] as usize;
    let subnqn = String::from_utf8_lossy(&connect_command[21..(21 + subnqn_len)]).to_string();

    // Store the connection information for future I/O commands
    let connection = NvmeofConnection::new(hostid, cntlid, subnqn);

    // Send the Connect Response
    let mut response = [0u8; 48];
    response[0] = NVMEOF_COMMAND_CONNECT;
    response[1] = 0x01; // Connect Response status: Success
    stream.write(&response)?;

    Ok(())
}
```

앞서 언급했던 것처럼, 이전 포스트와 구조가 약간 다르기 때문에 그대로 복사해서 사용할 수는 없겠지만
가장 핵심적인 정보인 `hostid`, `cntlid`, `subnqn`을 추출해서 `connection`이란
변수에 저장해두는 코드를 "알려주지도 않았는데" 스스로 작성하였다. 이 배움을 바탕으로, 추후
코드 통합시에 아마도 connection을 다른 어딘가로 잘 전달해서 NVMe-oF read/write I/O를
처리할 때 사용할 수 있을 것이라 예상된다. 이렇게 `OpCode == 0x00` 케이스에 대한 간단한
구현을 마친다. 이번 포스팅 작성은 24분 걸렸다.