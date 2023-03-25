---
layout: post
title:  "nvmf <6> - end-to-end integrity check"
date:   2023-03-25 22:00:00 +0930
categories: rust nvmf chatgpt integrity dif dix
---

호스트가 NVMe-oF 타겟에 특정 주소 A로 데이터를 적었다고 하자. 
추후 호스트가 주소 A의 데이터를 읽었는데, 뭔가 알수 없게 깨져 있어서 data corruption이
의심되는 상황이라 해보자. 이 경우, 어디서 문제가 생겼던 것일까?
호스트는 타겟에게 정말 의도한 데이터를 주었을까?
타겟은 정말 호스트에게 받은 데이터를 변경 없이 제대로 저장 장치에 썼을까?
저장 장치는 정말 bit rot/corruption 없이 데이터를 잘 보관하고 있었을까?

이런 문제가 발생하면, data path의 모든 부분을 의심할 수 밖에 없는데, 이 때 
도움을 줄 수 있는 feature가 바로 end-to-end data integrity check 이라 
볼 수 있다. 데이터 복사/변경이 발생할 수 있는 모든 레이어에, 데이터 무결성 (integrity)를 체크하도록 하여,
의도치 않은 데이터가 도달하거나 처리된 경우, 즉각적인 에러 핸들링으로 데이터의 재전송을 유도하던지 (transient error일 경우),
error propagation을 막는 전략 (fault isolation을 위해) 으로 볼 수 있다. NVMe-oF 애플리케이션 개발 시, 호스트의 data corruption 디버깅에서 
억울한 누명을 쓰지 않기 위해서라도 꼭 포함되면 좋을 feature로 볼 수 있을 것이다.

NVMe/NVMe-oF 에서는 어떤 방법으로 end-to-end data integrity check을 구현할 수 있을까?
ChatGPT 에게 물어보도록 하자:

> How to enforce end-to-end data integrity in my NVMe-oF application?
> > ChatGPT: Enforcing end-to-end data integrity is critical to ensure the reliability and security of data transmission in an NVMe-oF (Non-Volatile Memory Express over Fabrics) application. Here are some steps you can take to enforce end-to-end data integrity:
> > ...
> > Enable and configure the data integrity features of your NVMe SSDs and NVMe-oF target devices. These features include the Protection Information (PI) field, which adds a checksum or cryptographic hash to each block of data transferred between the host and the target device. Make sure that your NVMe SSDs and NVMe-oF target devices support the same data integrity features and use the same settings.
>
> Could you elaborate more on the Protection Information approach?
> > ChatGPT: The Protection Information feature is a part of the NVMe protocol that allows the host and the NVMe device to ensure the integrity of the data being transferred. The Protection Information feature adds a checksum or a cryptographic hash to each block of data that is transferred between the host and the device. The checksum or hash is calculated over the data block and stored in a Protection Information (PI) field, which is then transmitted along with the data.
>
> What about DIF/DIX? Do they offer additional protection?
> > ChatGPT: Yes, DIF/DIX offers additional protection beyond what's offered by Protection Information. 
> > DIF (Data Integrity Field) and DIX (Data Integrity Extension) are two other methods of ensuring data integrity in storage systems. They are similar to Protection Information in that they add a checksum or a cryptographic hash to each block of data, but they also include additional fields that can provide more information about the data being transferred.

```rust
use nvmeadm::{NvmeCmd, NvmeError, NvmeResult};
use nvmeadm::command::{Command, OpCode};
use nvmeadm::controller::Controller;
use nvmeadm::data::{Data, WriteData};

// Define the DIF/DIX settings
const DIF_ENABLE: bool = true; // Enable DIF/DIX
const DIF_TYPE: u8 = 0x01; // 0x01 = CRC-16
const DIF_INSERT: bool = true; // Insert DIF/DIX on write

// Define the Protection Information settings
const PI_ENABLE: bool = false; // Disable Protection Information
const PI_TYPE: u8 = 2; // 2 = SHA-256
const PI_LOC: u8 = 0; // 0 = Protection Information is transferred with the data

// Define the write handler function
pub fn write_with_data_integrity(controller: &Controller, namespace_id: u32, lba: u64, data: &[u8]) -> NvmeResult<()> {
    let block_size = controller.get_namespace(namespace_id)?.get_block_size() as u32;
    let data_len = data.len() as u32;
    let metadata_len = if DIF_ENABLE {
        let dif_size = 8; // DIF metadata size
        let dix_size = 4; // DIX metadata size
        let dif_stride = 1; // DIF/DIX is inserted for each data block
        (data_len + block_size - 1) / block_size * (dif_size + dif_stride * dix_size)
    } else {
        0 // No DIF/DIX metadata
    };
    let pi_length = if PI_ENABLE {
        match PI_TYPE {
            0 => 8, // CRC32C checksum
            1 | 2 => 32, // SHA-1 or SHA-256 hash
            _ => return Err(NvmeError::InvalidParameter),
        }
    } else {
        0 // No PI metadata
    };

    // Build the write command
    let mut write_cmd = Command::new(OpCode::Write);
    write_cmd.set_namespace_id(namespace_id);
    write_cmd.set_cdw10(lba as u32);
    write_cmd.set_cdw11((lba >> 32) as u32);
    write_cmd.set_cdw12(metadata_len << 16 | PI_TYPE as u32 << 8 | PI_LOC as u32);
    write_cmd.set_cdw13(if DIF_ENABLE {
        (DIF_TYPE as u32) << 16 | (DIF_INSERT as u32) << 15
    } else {
        0 // No DIF/DIX
    });
    write_cmd.set_data(Data::Write(WriteData::new(data.to_vec(), metadata_len + pi_length)));

    // Send the write command to the controller
    let mut response = controller.send_command(&write_cmd)?;

    // Check the status of the write command
    if response.status() != 0 {
        return Err(NvmeError::CommandError(NvmeCmd::Write, response.status()));
    }

    Ok(())
}
```

위와 같은 코드 예제를 받았다. NVMe-oF 로 오는 write command로부터 data, metadata (여기에
DIF/DIX checksum이 포함되어 있다고 ChatGPT 대답함)를 확인 후 다시 write command에 전달하고 
이걸 `controller.send_command()`를 통해 실제 NVMe 디바이스에 보내는 코드이다. 

만약 여기서 `data`에 checksum 계산을 직접 해서 `metadata`의 DIF/DIX 와 비교해보는
코드를 넣어보고 싶다면? 

> Can we add the checksum validation code in the above example? 
> Let's assume that both initiator and target use CRC-16 algorithm.

```rust
use nvme_rs::{
    nvme_admin_opc,
    nvme_admin_cmd::{NvmeAdminCommand, NvmeAdminOpcode},
    nvme_core::{NvmeError, NvmeNamespace, NvmeController},
    nvme_queue::{NvmeCompletionQueue, NvmeSubmissionQueue},
    nvme_util::NvmeResult,
    nvme_ioctl::{DifCheckFlags, DifRefTagType, DifLocType, DifType},
};

fn handle_write_command(
    controller: &mut NvmeController,
    ns: &NvmeNamespace,
    data: &[u8],
    metadata: &[u8],
    lba: u64,
) -> NvmeResult<()> {
    // Calculate the expected DIF checksums
    let dif_type = DifType::Crc16;
    let mut expected_dif_checksums = Vec::new();
    let block_size = ns.get_sector_size() as usize;
    for block_data in data.chunks_exact(block_size) {
        let crc = nvme_rs::nvme_util::crc16(block_data, block_size);
        expected_dif_checksums.extend_from_slice(&crc.to_be_bytes());
    }

    // Validate the DIF checksums in the metadata
    let metadata_len = metadata.len();
    let expected_ref_tag = [0u8; 8];
    let dif_check_flags = DifCheckFlags::empty();
    let dif_ref_tag_type = DifRefTagType::App;
    let dif_loc = DifLocType::Last;
    let dif_type = DifType::Crc16;
    let mut actual_dif_checksums = vec![0; metadata_len / 2];
    let mut cmd = NvmeAdminCommand::new(
        nvme_admin_opc::NVME_ADMIN_OPC_GET_LOG_PAGE,
        NvmeNamespace::build_get_log_page_command(
            0x10,
            0,
            metadata_len as u32,
            dif_check_flags,
            dif_ref_tag_type,
            dif_loc,
            dif_type,
            Some(expected_ref_tag),
        ),
    )?;
    cmd.add_data_out(&mut actual_dif_checksums)?;
    controller.admin_cmd(&mut cmd)?;

    if actual_dif_checksums != expected_dif_checksums {
        return Err(NvmeError::CommandExecutionError {
            opcode: NvmeAdminOpcode::NVME_ADMIN_OPC_GET_LOG_PAGE,
            status: cmd.status(),
        });
    }

    // Calculate the expected DIX checksums
    let mut expected_dix_checksums = Vec::new();
    if let Some(dix_metadata) = metadata.get(block_size..) {
        let dix_type = DifType::Crc32C;
        for block_data in data.chunks_exact(block_size) {
            let crc = nvme_rs::nvme_util::crc32c(block_data, block_size);
            expected_dix_checksums.extend_from_slice(&crc.to_be_bytes());
        }

        // Validate the DIX checksums in the metadata
        let expected_ref_tag = [0u8; 8];
        let dif_check_flags = DifCheckFlags::empty();
        let dif_ref_tag_type = DifRefTagType::App;
        let dif_loc = DifLocType::First;
        let mut actual_dix_checksums = vec![0; dix_metadata.len() / 4];
        let mut cmd = NvmeAdminCommand::new( (잘림)
```

출력이 자꾸 잘려, 전체 코드를 다 볼 수가 없었고 매번 re-generate 할 때마다 코드가
많이 바뀌어서, 그 중 나름 간결하게 표현된 버전을 copy&paste로 올렸다. 추후 필요시,
이 코드를 약간 변형해서 사용하면 될 것 같다.