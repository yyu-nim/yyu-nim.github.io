---
layout: post
title:  "nvmf <5> - reconnected"
date:   2023-03-23 22:00:00 +0930
categories: rust nvmf chatgpt reconnect seqdiagram
---

NVMe는 SSD를 circular queue 인터페이스를 써서 parallelism을 극대화할 수 있도록
돕는 프로토콜이라 본다면, NVMe-oF는 NVMe를 unreliable network을 통해 원격으로
연결시켜 줄 수 있는 프로토콜이라 볼 수 있을 것이다 (라고 난 생각한다). 여기에서의 핵심은,
네트워크 연결이 끊어졌을 때, 이를 Initiator에서 돌고 있는 애플리케이션으로부터 최대한
감추어 availability를 높이는 것이다. 네트워크의 동적인 상태가 애플리케이션에게 너무 
직접적으로 전달이 되면, 어려운 네트웍 에러 핸들링 부담이 애플리케이션에게 전가될 것이므로
사용하기가 어려워질 것이고, 반대로 애플리케이션에게 네트워크의 변화를 전혀 알리지 않으면
hang 문제를 피하기 어려울 것이다. 따라서 이 둘의 균형을 적절히 잡아줄 수 있는 timeout이
필요할 텐데, 모든 상황에 적합한 단 하나의 magic value가 있을리는 없고, 결국 Target이
자신의 사용 목적을 염두에 두고, 적절한 값을 Initiator에게 알려주어야 할 것이다. 
이러한 timeout 정보의 교환은 어떻게 이루어지게 될까, 
이를 ChatGPT&RUST로 어떻게 작성할 수 있을지가 오늘의 포스팅 주제이다.

일단, 간단한 시퀀스 다이어그램을 요청해서 Initiator/Target간의 reconnection이 어떻게
발생하는지 확인해보자:

> Can you draw a sequence diagram that describes 
> the interaction between an initiator and a target 
> to handle a case where the target is restarted and the connection is recovered?

```text
Initiator                        Target
--------                         ------
   |                                |
   |  Connect to target             |
   |------------------------------->|
   |                                |
   |  Create queue pair             |
   |------------------------------->|
   |                                |
   |  Send I/O request              |
   |------------------------------->|
   |                                |
   |  Receive I/O completion        |
   |<-------------------------------|
   |                                |
   |  Target restarts               |
   |                                |
   |  Attempt to reconnect to target|
   |------------------------------->|
   |                                |
   |  Re-establish queue pair       |
   |------------------------------->|
   |                                |
   |  Check for pending I/O requests|
   |                                |
   |  Reissue pending I/O requests  |
   |------------------------------->|
   |                                |
   |  Send new I/O request          |
   |------------------------------->|
   |                                |
   |  Receive new I/O completion    |
   |<-------------------------------|

```

대략 위와 같은 흐름에서, Target이 어떤 방식으로 timeout 값을 Initiator에게 전달할 수 있을까?
> How does the target advertise its preferred timeout
> value to the Initiator? Could you write the example in RUST?

```rust
use std::net::{TcpListener, TcpStream};

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:4420")?;

    // Wait for a connection from an NVMeoF initiator
    for stream in listener.incoming() {
        let mut stream = stream?;

        // Send the NVMeoF discovery log page to the initiator
        let log_page = build_discovery_log_page();
        stream.write_all(&log_page)?;

        // Handle I/O requests from the initiator
        // ...
    }

    Ok(())
}

fn build_discovery_log_page() -> Vec<u8> {
    let subnqn = b"nvme:example-subsystem-name:timeout_ms=5000\0";
    let subnqn_len = subnqn.len() as u32;
    let log_page_len = 4096;

    let mut log_page = Vec::with_capacity(log_page_len as usize);
    log_page.resize(log_page_len as usize, 0);

    // Set the subtype field to 1 to indicate a discovery log page
    log_page[0] = 1;

    // Set the subnqn field to advertise the preferred timeout value
    let subnqn_offset = 16;
    let subnqn_slice = &mut log_page[subnqn_offset..];
    subnqn_slice[..subnqn.len()].copy_from_slice(subnqn);

    // Set the length field to the length of the log page
    let length_offset = 4;
    let length_slice = &mut log_page[length_offset..];
    length_slice.copy_from_slice(&(log_page_len as u32).to_le_bytes());

    log_page
}

```

ChatGPT에 따르면, Discovery 커맨드에 subnqn을 적어보낼 때, 뒤에 `:timeout_ms`를
같이 붙여서 timeout을 전달할 수 있다고 한다. 이를 Initiator에 사용하고 있는지는 
추가 확인이 필요할 것 같다. 어쨌든, 이런 깨알 정보들을 직접 찾는 수고를 덜어주는 것만으로도
충분히 ChatGPT는 가치를 입증한다고 생각한다.