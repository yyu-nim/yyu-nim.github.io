---
layout: post
title:  "nvmf <4> - persisting qpair"
date:   2023-03-22 22:00:00 +0930
categories: rust nvmf chatgpt qpair persistence
---

ChatGPT의 한계를 계속 시험해보고 싶어, 어려운 질문들을 던져 보았다.
소프트웨어 엔지니어링에서 중요하면서도 어려운, 정형화된 답이 없는 failover 분야이다.
NVMe-oF 타겟을 "아무때나" 종료시켰을 때, 재시작된 NVMe-oF 타겟이 마치 아무일도
없었다는 듯 운영을 재개할 수 있을까?

이전 포스팅까지의 정보를 바탕으로, NVMe-oF 타겟은 queue pair 라는 것을 생성하여
호스트 (소위 initiator)와 통신한다는 것을 알 수 있었고, 설명에 포함하지는 않았지만
head/tail 포인터를 두어 약속된 offset에 해당하는 entry들을 추가하거나 삭제하는
것으로 보인다. 

그렇다면, 재시작된 NVMe-oF는 적어도 queue pair와 관련된 모든 상태를 저장하고 
있어야, 마치 failure가 없었던 것처럼 재개될 수 있을 것이다. 어떻게 구현할 수 있을까?
ChatGPT의 답변은 이랬다:

> Can you write an example of NVMeoF target in RUST 
> that can restore its previous connection queue information after getting restarted?
> > ChatGPT: Sure! Here's an example of an NVMeoF target 
> > in Rust that saves its connection queue information to a file, and restores it when the application is restarted:
> 
> Don't we need to remember the qpair information like head, tail pointers and etc?
> > ChatGPT: Yes, you are correct. In addition to the connection queue, 
> > you will also need to save and restore the queue pairs (qpairs) information, including the head and tail pointers, when restarting the NVMeoF target application.
> 
> Awesome. But, what about the memory address for the queue pair? 
> Is it something that persists across the app restart? 
> Storing just head and tail would not help to locate the actual data address right.
> > ChatGPT: You are correct. The head and tail pointers alone are not enough
> > to locate the actual data address of the queue pair.
> > The actual memory address of the queue pair would be lost after
> > the application restarts.
> > To solve this problem, you can store the queue pair data
> > in a shared memory region that can be accessed by the NVMeoF
> > target application even after it restarts.
> > One way to do this is to use a library like mmap in Rust,
> > which allows you to map a file to a memory region that
> > can be shared between processes.

내가 생각하고 있던 답과 비슷하게 나왔다! head/tail의 속도를 고려해보았을 때
`mmap`을 사용하여, 
in-memory access 속도를 가지면서도 process restart가 되어도 정보가
유지될 수 있는 좋은 방법이라 생각한다. 물론, 이 경우 노드/OS 자체가 죽는 경우에는
queue pair 정보를 잃게 될 것이다 (다른 대안은, head/tail의 변경점을
저널링 하는 것).

아래는 mmap을 사용하여 process restart 수행 시 qpair 정보를 로드하는
예제를 나타낸다.

```rust
use std::fs::{File, OpenOptions};
use std::io::{BufReader, BufWriter, Read, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;
use std::os::unix::io::AsRawFd;
use std::mem::size_of;
use std::slice;
use libc::{c_void, MAP_SHARED, PROT_READ, PROT_WRITE};

const CONNECTION_QUEUE_FILE: &str = "connection_queue.txt";
const QPAIRS_FILE: &str = "qpairs.txt";
const QPAIR_DATA_FILE: &str = "qpair_data.bin";
const QPAIR_DATA_SIZE: usize = 4096;

// Struct to represent a QPair
struct QPair {
    head: usize,
    tail: usize,
    // Add any other necessary fields here
}

fn mmap_qpair_data_file() -> *mut c_void {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .open(QPAIR_DATA_FILE)
        .unwrap();
    let size = QPAIR_DATA_SIZE * size_of::<QPair>();
    unsafe {
        libc::ftruncate(file.as_raw_fd(), size as i64);
        libc::mmap(
            std::ptr::null_mut(),
            size,
            PROT_READ | PROT_WRITE,
            MAP_SHARED,
            file.as_raw_fd(),
            0,
        )
    }
}

// Function to handle incoming client connections
fn handle_client(mut stream: TcpStream) {
    // Handle client request here
    // ...
}

fn main() {
    // Load previous connection queue from file
    let mut connection_queue: Vec<TcpStream> = Vec::new();
    if let Ok(file) = File::open(CONNECTION_QUEUE_FILE) {
        let reader = BufReader::new(file);
        for stream in reader.lines() {
            if let Ok(stream_str) = stream {
                if let Ok(stream) = TcpStream::connect(stream_str) {
                    connection_queue.push(stream);
                }
            }
        }
    }

    // Load previous QPairs from file
    let mut qpairs: Vec<QPair> = Vec::new();
    if let Ok(file) = File::open(QPAIRS  (잘림))

```

Queue pair에 데이터가 존재하는 상태에서 failure가 발생할 경우,
정책적으로 해당 데이터들을 복구 포기하는 것이 더 나은 방법일 수 있고, 이에 대해서는
추후 블로그 포스트에서 다룰 수 있을 것 같다.
