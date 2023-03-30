---
layout: post
title:  "nvmf <9> - Identify"
date:   2023-03-30 22:15:00 +0930
categories: rust crossbeam copilot identify nvmf
---

Copilot/ChatGPT의 도움으로 NVMe-oF 패킷으로부터 헤더를 추출하여 
원하는 커맨드 핸들러로 분기하는 코드를 작성해 보았다. 그렇다면, 이제는 커맨드를 하나씩
구현해 볼 차례이다. 일단 쉬워보이는 `Identify`를 골라, ChatGPT에게 차근차근
물어보도록 하자:

> How to implement Identify command handler for NVMe-oF protocol in RUST?
> > ChatGPT: Implementing an Identify command handler for NVMe-oF protocol in Rust involves a few steps. ...

```rust
struct Identify {
    stream: TcpStream,
    header_vec: Vec<u8>,
}

impl Identify {
    fn new(stream: TcpStream, header: &[u8]) -> Identify {
        Identify {
            stream,
            header_vec: Vec::from(header),
        }
    }

    pub async fn handle(&mut self) -> Result<(), Box<dyn Error>> {
        let (_opcode, identify_params) = self.parse_command().unwrap();
        let cns = identify_params[0];
        let csi = u32::from_le_bytes(identify_params[1..5].try_into().unwrap());
        let nvmsetid = u32::from_le_bytes(identify_params[5..9].try_into().unwrap());
        let uuidindex = u32::from_le_bytes(identify_params[9..13].try_into().unwrap());
        let uuidlist = &identify_params[13..];
        self.populate_identify_controller(cns, csi, nvmsetid, uuidindex, uuidlist);

        let mut response = Vec::new();
        response.push(0); // status code
        response.push(0); // status code specific
        response.push(0); // reserved
        response.push(0); // reserved
        response.push(0); // reserved
        response.push(0); // reserved
        response.push(0); // reserved
        response.push(0); // reserved

        self.stream.write(response.as_slice());

        Ok(())
    }

    fn parse_command(&self) -> Result<(u8, &[u8]), String> {
        let command = self.header_vec.as_slice();

        // Check that the command has at least 4 bytes
        if command.len() < 4 {
            return Err("Command buffer too small".to_owned());
        }

        // Extract the opcode and parameter length fields
        let opcode = command[0];
        let plen = u16::from_le_bytes(command[2..4].try_into().unwrap()) as usize;

        // Check that the command buffer is large enough to contain the parameters
        if command.len() < plen + 4 {
            return Err("Command buffer too small".to_owned());
        }

        // Return the opcode and parameters
        Ok((opcode, &command[4..(plen + 4)]))
    }

    fn populate_identify_controller(&self, cns: u8, csi: u32, nvmsetid: u32, uuidindex: u32, uuidlist: &[u8]) -> Vec<u8> {
        match cns {
            NVME_CTRLR_IDENTIFY_CNS_CTRLR => {
                let controller = IdentifyController {
                    vendor_id: 0x1234,
                    subsystem_vendor_id: 0x5678,
                    serial_number: "SN1234".to_owned(),
                    model_number: "Model1234".to_owned(),
                    firmware_revision: "1.0".to_owned(),
                    recommended_aen_bitmap: 0x01,
                    controller_multi_path_io_and_namespace_attach: 0,
                    maximum_data_transfer_size: 0x4000,
                    controller_id: 1,
                };
                bincode::serialize(&controller).unwrap()
            }
            NVME_CTRLR_IDENTIFY_CNS_NS => {
                // TODO: implement namespace information based on csi, uuidindex, and uuidlist
                let ns_data = vec![0; 4096];
                let mut response = vec![0; 4096 + 4];
                byteorder::LittleEndian::write_u32(&mut response[0..4], 4096);
                response[4..].copy_from_slice(&ns_data);
                response
            }
            _ => {
                let response = vec![0; 4];
                byteorder::LittleEndian::write_u32(&mut response[..4], 0xffffffff);
                response
            }
        }
    }
}
```

ChatGPT로부터 뼈대를 받아와, Copilot으로 완성한 코드는 위와 같았다. 
간단히 요약하면, NVMe-oF 패킷의 헤더로부터 `Identify` 커맨드를 위한
파라미터를 추출해야 한다. ChatGPT가 써준 `parse_command`에 따르면,
파라미터의 경우 고정된 길이가 아니고 가변길이로 디자인되어 있기 때문에,
`command[2..4]`에 있는 `plen`이라는 값을 추출해서,
`command[4..(plen + 4)])`를 통해 파라미터들을 얻어온다.
그 다음에는, NVMe controller의 binary encoding을 수행하여, 이를
TcpStream를 통해 해당 구조체를 호스트에게 보내준다.
`vendor_id, subsystem_vendor_id, serial_number, model_number, 
firmware_revision, recommended_aen_bitmap, 
controller_multi_path_io_and_namespace_attach, 
maximum_data_transfer_size, controller_id`들은 현재는 임의의 값을
채워넣었지만, 추후 cns, csi, nvmsetid, uuidindex, uuidlist를 활용하여
의미있는 로직을 추가할 수 있을 것이다.

코드는 작성하였으나, 아직 테스트가 없다. `nvme-cli`를 사용하면 쉽게 NVMe-oF 커맨드를
만들어서 보내는 방식으로 system test를 작성할 수 있을텐데, 문제는 리눅스 전용툴인 관계로 
Mac에서는 사용할 수 없다는 점이 단점이고, 따라서 별도의 가상화 솔루션을 사용해야 한다.
이에 대해서는 다음 포스팅에서 이어가겠다. 본 블로그 작성은 예제 포함 30분+ 정도 소요되었다. 
