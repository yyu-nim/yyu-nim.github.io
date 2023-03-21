---
layout: post
title:  "nvmf <3> - read/write command"
date:   2023-03-21 22:00:00 +0930
categories: rust nvmf chatgpt read write command
---

이번에는 실제 사용자 데이터의 read/write를 어떻게 처리하면 좋을지가 궁금해서, 
ChatGPT에게 여쭤보았다:

> Let's get back to the example that we handle read/write I/O 
> command over NVMeoF in RUST. Could you write it again please?

Opcode 별로 match cases 들을 죽 나열해주었는데, 0x06, 0x07 일 경우 각각
write/read 커맨드에 해당한다고 한다. 실제 구현은 빼주었길래, 그 부분 요청하였다:

> Could you elaborate more on the write/read commands 
> where the opcodes are 0x06 and 0x07 respectively?

오늘은 ChatGPT가 network error 라며 중간에 출력을 자꾸 중단시켜서, 
소스 코드 마지막 몇줄은 수동으로 완성하였다:

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use byteorder::{ByteOrder, LittleEndian};

// Define the NVMeoF command header
#[repr(C)]
struct NvmeofCmdHeader {
    opcode: u8,
    flags: u8,
    command_id: u16,
    nvmf_specific: u32,
    data_len: u32,
    metadata_len: u32,
    padding: [u8; 2],
}

fn handle_io_command(mut stream: &TcpStream, header: NvmeofCmdHeader) -> Result<(), std::io::Error> {
    match header.opcode {
        // Write command
        0x06 => {
            let mut data = vec![0; header.data_len as usize];
            stream.read_exact(&mut data)?;
            // Do something with the data
            println!("Received write command with data {:?}", data);
        },
        // Read command
        0x07 => {
            let mut data = vec![0; header.data_len as usize];
            // Do something to fill the data
            stream.write_all(&data)?;
            println!("Sent read command response with data {:?}", data);
        },
        _ => {
            println!("Unsupported opcode: {}", header.opcode);
        }
    }
    Ok(())
}

fn handle_connection(mut stream: TcpStream) -> Result<(), std::io::Error> {
    let mut header_buf = [0; std::mem::size_of::<NvmeofCmdHeader>()];
    stream.read_exact(&mut header_buf)?;

    let header = NvmeofCmdHeader {
        opcode: header_buf[0],
        flags: header_buf[1],
        command_id: LittleEndian::read_u16(&header_buf[2..4]),
        nvmf_specific: LittleEndian::read_u32(&header_buf[4..8]),
        data_len: LittleEndian::read_u32(&header_buf[8..12]),
        metadata_len: LittleEndian::read_u32(&header_buf[12..16]),
        padding: [header_buf[16], header_buf[17]],
    };

    match header.opcode {
        // Connect command
        0x01 => {
            let mut buf = vec![0; header.data_len as usize];
            stream.read_exact(&mut buf)?;

            let hostid = String::from_utf8_lossy(&buf[0..16]);
            let cntlid = String::from_utf8_lossy(&buf[16..32]);
            let subnqn = String::from_utf8_lossy(&buf[32..(32 + header.data_len as usize - 32)]);

            println!("Received connect command: hostid={}, cntlid={}, subnqn={}", hostid, cntlid, subnqn);

            // Do something with hostid, cntlid, and subnqn
            // ...

            // Send a response
            let response = b"OK";
            stream.write_all(response)?;
            println!("Sent connect command response: {:?}", response);
        },
        // Read/write commands
        0x06 | 0x07 => {
            handle_io_command(&stream, header)?;
        }
    }
}
```

`data`에는 사용자 데이터가 이미 있거나, 곧 쓰여질 버퍼일 것이다.
만약 이 데이터를 로컬 NVMe 디바이스에서 읽거나 쓸 수 있는 몇가지 방법이 있을텐데 그 중,
커널의 [io_uring](https://en.wikipedia.org/wiki/Io_uring) 인터페이스를 사용해 본다면? 
검색해 볼 시간에 ChatGPT에게 물어보자:

> Could you write an example that we read user data 
> from NVMeoF command and write to local NVMe device through io_uring?

출력이 잘리긴 했지만, 생각보다 디테일을 살려 예제를 출력해주었다 (+ io_uring write만 물어봤는데
io_uring read 까지 그냥 알려주었다):

```rust
use std::io::{self, Read, Write};
use std::net::{TcpListener, TcpStream};
use byteorder::{ByteOrder, LittleEndian};
use log::{error, info};
use io_uring::{opcode, types, IoUring};

const NVME_ADMIN_CMD_OPCODE: u8 = 0x01;
const NVME_IO_CMD_WRITE_OPCODE: u8 = 0x06;
const NVME_IO_CMD_READ_OPCODE: u8 = 0x07;

fn handle_io_cmd(
    mut stream: &TcpStream,
    cmd_buf: &[u8],
    user_data: &[u8],
    io_uring: &mut IoUring,
) -> io::Result<()> {
    let opcode = cmd_buf[0];
    let nsid = LittleEndian::read_u32(&cmd_buf[4..]);

    if opcode == NVME_IO_CMD_WRITE_OPCODE {
        let lba = LittleEndian::read_u64(&cmd_buf[8..]);
        let length = LittleEndian::read_u32(&cmd_buf[16..]);

        let mut sqe = opcode::Write::new(types::Fd(stream.as_raw_fd()), user_data)
            .offset(lba * 512)
            .build();

        unsafe {
            sqe.prep_flags(io_uring::squeue::Flags::FIXED_FILE);
        }

        io_uring.submit_sqes()?;
        io_uring.wait_for_cqe()?;
        let cqe = io_uring.peek_cqe().unwrap();
        io_uring
            .push_sqes()
            .map_err(|e| io::Error::new(io::ErrorKind::Other, e))?;

        if cqe.result() < 0 {
            error!("Error occurred while writing to local NVMe device.");
        }
    } else if opcode == NVME_IO_CMD_READ_OPCODE {
        let lba = LittleEndian::read_u64(&cmd_buf[8..]);
        let length = LittleEndian::read_u32(&cmd_buf[16..]);

        let mut sqe = opcode::Read::new(types::Fd(stream.as_raw_fd()), user_data)
            .offset(lba * 512)
            .build();

        unsafe {
            sqe.prep_flags(io_uring::squeue::Flags::FIXED_FILE);
        }

        io_uring.submit_sqes()?;
        io_uring.wait_for_cqe()?;
        let cqe = io_uring.peek_cqe().unwrap();
        io_uring
            .push_sqes()
            .map_err(|e| io::Error::new(io::ErrorKind::Other, e))?;

        if cqe.result() < 0 {
            error!("Error occurred while reading from local NVMe device.");
        } else {
            let buf = cqe.user_data();
            stream.write_all(&buf[..length as usize])?;
        }
    }

    Ok(())
}

fn handle_nvmeof_cmd(mut stream: TcpStream, io_uring: &mut IoUring) -> io::Result<()> {
    let mut header_buf = [0u8; 28];
    stream.read_exact(&mut header_buf)?;

    let opcode = header_buf[0];
    let nsid = LittleEndian::read_u32(&header_buf[4..]);

    if opcode == NVME_ADMIN_CMD_OPCODE {
        // handle admin command
    } else if opcode == NVME_IO_CMD_WRITE_OPCODE || opcode == NVME_IO_CMD_READ_OPCODE {
        let mut data_buf = vec![0u8; 4096];
        let data_len = stream.read (잘림)
```

예제로부터 새로 배운 사실은, 
1) `io_uring`이라는 crate이 존재한다는 것,
2) sqe (send queue entry?)를 준비할 때 `unsafe`가 필요하다는 것,
3) `submit_sqes()`, `wait_for_cqe()` 라는 API로 async I/O를 수행하고 대기할 수 있다는 점 이었다.

`sqe`를 커널에 던지자 마자 곧바로 대기하는 현재의 방식은, read/write system call을
호출하는 것과 크게 다르지 않나 생각이 들었다. 그만두기 전에 한번 더 물어보았다:

> In the example, you submit two reqs one by one, 
> waiting for the previous one to finish, don't you? 
> I wonder how to submit those two reqs asynchronously.

(오늘 ChatGPT 연결 상태가 좋지 않아 이것으로...)