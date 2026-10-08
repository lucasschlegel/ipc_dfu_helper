# ipc_dfu_helper

Minimal NCS (nRF Connect SDK) application for the nRF9160 that performs
full modem DFU using the official `nrf_modem_bootloader` API, communicating
with the ESP32/RP2040 programmer over a UART tunnel.

## Why this exists

`POWER.LTEMODEM.STARTN` requires a **CPU bus transaction** to propagate.
Debug AP (SWD) writes are accepted without SLVERR but the POWER state machine
never propagates the change — the same applies to `CLOCK.TASKS_HFCLKSTART`.
`nrf_modem_lib_bootloader_init()` runs on the APP CPU and handles all of:

- HFXO startup
- `POWER.LTEMODEM` sequencing (STARTN / FORCEOFF)
- IPC peripheral configuration
- Shared memory setup
- Loading the modem ROM into DFU mode
- Receiving the root key digest via IPC

This app replaces the failing SWD-based approach in `probe_dap_glue.c`
for devices where `ERASEPROTECT=0x0` blocks `CTRL-AP ERASEALL` (so the old
TF-M/SPM is still intact in APP flash).

## Hardware

```
ESP32  ←USB/SPI→  RP2040 (VPROG/TUN)
                     │ UART1 (tunnel)
                     │ via MUX GPIO10 → Bank A
                     ↓
              nRF9160 P0.28(RXD) / P0.29(TXD)
              nRF9160 P0.27(CTS) / P0.26(RTS)
```

Baud rate: 115200, 8N1, hardware flow control.

## Build

```bash
# NCS v2.x with TF-M (nrf9160dk_nrf9160_ns board target)
west build -b nrf9160dk_nrf9160_ns \
    -- -DOVERLAY_CONFIG=prj.conf

# Output: build/merged.hex  (TF-M + helper app combined)
```

For a custom board, create `boards/<your_board>_ns.overlay` following the
pattern in `boards/nrf9160dk_nrf9160_ns.overlay`, then:
```bash
west build -b <your_board>_ns
```

## Flash (via RP2040 SWD)

`ERASEPROTECT` blocks `CTRL-AP ERASEALL` but **not** individual page erases
via `NVMC.ERASEPAGE`.  The RP2040 can page-erase + write the helper app hex:

```c
// In probe_dap_glue.c, old_spm_in_flash branch:
//
// 1. Parse build/merged.hex into (addr, data) segments
// 2. For each segment:
//    a. page-erase all 4KB pages it covers via nrf91_erase_page()
//    b. write via nrf91_write_words() (NVMC WEN + word writes)
// 3. CTRL-AP RESET to boot helper app
// 4. Run hdfu_wait_ready() to confirm boot
// 5. Run the DFU protocol (see ipc_dfu_host_protocol.h)
```

## Protocol

All communication over the UART tunnel.

### Request frame (RP2040 → nRF9160)
```
[0xAA][0x55][CMD][LEN_3][LEN_2][LEN_1][LEN_0][PAYLOAD...][CRC8]
              └─1B─┘ └────── 4B little-endian ──────┘
```

### Response frame (nRF9160 → RP2040)
```
[0x55][0xAA][CMD][STATUS][LEN_3][LEN_2][LEN_1][LEN_0][PAYLOAD...][CRC8]
              └─1B─┘ └─1B─┘ └────────── 4B little-endian ──────────┘
STATUS: 0x00=OK, 0xFF=ERROR (PAYLOAD=int32_t error code)
```

CRC: CRC-8/SMBUS (poly=0x07, init=0x00, no reflection) over all bytes
     before the CRC byte.

### Boot banner (nRF9160 → RP2040, spontaneous after init)
```
CMD=0x00, STATUS=0x00, PAYLOAD=first 4 bytes of root key digest
```
Wait up to 10 seconds for this after CTRL-AP RESET.

### Commands

| CMD  | Name          | Request payload         | Response payload          |
|------|---------------|-------------------------|---------------------------|
| 0x01 | PING          | (none)                  | (none)                    |
| 0x02 | GET_DIGEST    | (none)                  | 32 bytes (8×uint32_t LE)  |
| 0x03 | BL_WRITE      | raw ihex text bytes     | (none)                    |
| 0x04 | BL_UPDATE     | (none)                  | (none)                    |
| 0x05 | FW_WRITE      | 4B addr LE + data bytes | (none)                    |
| 0x06 | FW_UPDATE     | (none)                  | (none)                    |
| 0x07 | RESET         | (none)                  | device resets (no reply)  |

### Typical DFU session

```
Host                           nRF9160 helper
 │                                │
 │ [CTRL-AP RESET]                │
 │                   ←[BANNER]    │  CMD=0x00, digest prefix (spontaneous)
 │ CMD_PING          →            │
 │                   ←OK          │
 │ CMD_GET_DIGEST    →            │
 │                   ←OK+32B      │  full digest
 │   (host selects 72B3D7C.ipc_dfu.signed_1.1.0.ihex from ZIP)
 │ CMD_BL_WRITE+ihex →            │
 │                   ←OK          │
 │ CMD_BL_UPDATE     →            │
 │                   ←OK          │  (may take ~10s; loader starts in modem)
 │                                │
 │  [for each firmware segment]   │
 │ CMD_FW_WRITE+addr+data →       │
 │                   ←OK          │
 │ CMD_FW_UPDATE     →            │
 │                   ←OK          │  (modem flash programmed)
 │                                │
 │ CMD_RESET         →            │
 │                   (device resets, real firmware boots)
```

### Digest format

`GET_DIGEST` returns 8 × `uint32_t` values in little-endian byte order.
The first 7 hex nibbles of the concatenated bytes (lowercase) form the
filename prefix for the IPC DFU loader:
```
e.g.  digest bytes = 72 b3 d7 c1 ...
      ihex file    = 72B3D7C.ipc_dfu.signed_1.1.0.ihex
```

## Integration into probe_dap_glue.c

In the `old_spm_in_flash` branch of `nrf91_ipc_init()`, replace the current
SYSRESETREQ / HFXO / SRAM-stub code with:

```c
#include "ipc_dfu_host_protocol.h"

/* 1. Flash merged.hex to nRF9160 APP flash via SWD */
if (!flash_helper_app_to_nrf91()) {
    vprog_log("NRF91_IPC_INIT: failed to flash helper app");
    return false;
}

/* 2. CTRL-AP RESET */
ctrlap_reset();
busy_wait_ms(200);

/* 3. Wait for helper app banner */
uint8_t digest_prefix[4];
if (!hdfu_wait_ready(digest_prefix, 10000)) {
    vprog_log("NRF91_IPC_INIT: helper app did not start");
    return false;
}
vprog_logf("NRF91_IPC_INIT: helper ready, digest prefix: %02x%02x%02x%02x",
           digest_prefix[0], digest_prefix[1],
           digest_prefix[2], digest_prefix[3]);

/* 4. Get full digest */
if (!hdfu_get_digest(digest_out)) {
    vprog_log("NRF91_IPC_INIT: GET_DIGEST failed");
    return false;
}
vprog_log("NRF91_IPC_INIT: digest retrieved");
return true;   /* caller proceeds with bl_write / bl_update / fw_write etc. */
```

The `ipc_fmfu.c` on the ESP32 side already has the ihex loader and firmware
data — it just needs to send them to the RP2040 using the new tunnel protocol
instead of the VPROG 'i' command.

## Memory layout

```
nRF9160 APP flash (1 MB, 0x00000000–0x000FFFFF):
  0x00000000  TF-M (Secure partition, ~56 KB)
  0x0000E000  MCUboot or SPM NS vector table
  0x00010200  helper app NS image

nRF9160 APP SRAM (256 KB, 0x20000000–0x2003FFFF):
  0x20000000  nrf_modem_bootloader shmem (0x2018 bytes = 8216 bytes)
  0x20002018  TF-M NS data + heap + stack
```

## Kconfig notes

If you see linker errors about insufficient RAM, reduce:
- `CONFIG_HEAP_MEM_POOL_SIZE` in prj.conf
- Send firmware in smaller chunks (e.g. 8 KB per CMD_FW_WRITE)

If `nrf_modem_lib_bootloader_init()` returns `-NRF_ENOMEM`, increase
`CONFIG_NRF_MODEM_LIB_SHMEM_CTRL_SIZE` or check that the partition manager
has allocated the modem shmem region correctly.

west build -b nrf9160dk_nrf9160_ns

  arm-none-eabi-objcopy -I ihex -O binary build/merged.hex ipc_dfu_helper.bin

west build -b nrf9160dk_nrf9160_ns ipc_dfu_helper/
  arm-none-eabi-objcopy -I ihex -O binary build/merged.hex ipc_dfu_helper.bin
  xxd -i ipc_dfu_helper.bin ipc_dfu_helper_hex.h