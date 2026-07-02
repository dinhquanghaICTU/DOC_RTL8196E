# Phân tích cấu trúc source `hacking-lidl-silvercrest-gateway`

## 1. Thông tin source được phân tích

Repository:

```text
/home/quanghaictu/WORK/dumpTUYA/hacking-lidl-silvercrest-gateway
```

Upstream:

```text
https://github.com/jnilo1/hacking-lidl-silvercrest-gateway.git
```

Snapshot đang phân tích:

```text
Commit : f04f0da748578ea6f93617d96613d48324ced694
Date   : 2026-06-11
Title  : chore: drop stale release-notes files from the repo root
```

Quy mô source tại thời điểm phân tích:

- Khoảng 1.015 file.
- Khoảng 187 MiB nếu tính cả `.git`.
- Source thực tế ngoài `.git` khoảng 34 MiB.
- Thành phần lớn gồm tài liệu phần cứng, firmware EFR32, bootloader/kernel/rootfs/userdata cho RTL8196E và các binary firmware dựng sẵn.

## 2. Repository này giải quyết bài toán gì?

Lidl Silvercrest Gateway nguyên bản phụ thuộc Tuya cloud. Repository này thay thế firmware của cả hai chip để biến gateway thành một hub local có thể dùng cho Zigbee, Thread/Matter hoặc Zigbee Router.

Board có hai bộ xử lý độc lập:

```text
                     Ethernet/TCP/IP
 Home Assistant  <--------------------------+
 Zigbee2MQTT                                  |
 OTBR host                                   |
                                              v
 +----------------------------------------------------------------+
 | Lidl Silvercrest Gateway                                      |
 |                                                                |
 |  +-------------------------+       UART      +---------------+  |
 |  | Realtek RTL8196E        | <------------> | Silabs EFR32  |  |
 |  | Lexra/MIPS, Linux       |   ttyS1        | Cortex-M4     |  |
 |  | 400 MHz, 32 MiB RAM     |                | 802.15.4 RF   |  |
 |  +-------------------------+                +---------------+  |
 |             |                                      |           |
 |             v                                      v           |
 |       SPI NOR 16 MiB                         Zigbee/Thread      |
 +----------------------------------------------------------------+
```

Vai trò của từng chip:

- **RTL8196E** chạy Linux, Ethernet, SSH, quản lý cấu hình, bridge UART ↔ TCP, OTBR và quy trình flash/update.
- **EFR32MG1B** xử lý radio IEEE 802.15.4; tùy firmware có thể chạy Zigbee NCP, radio RCP, OpenThread RCP hoặc Zigbee Router độc lập.

Vì vậy repository được chia thành hai nhánh firmware lớn:

```text
2-Zigbee-Radio-Silabs-EFR32/       firmware cho chip radio
3-Main-SoC-Realtek-RTL8196E/      Linux system cho SoC chính
```

## 3. Bản đồ thư mục cấp cao

```text
hacking-lidl-silvercrest-gateway/
├── .github/                         CI và workflow GitHub
├── 0-Hardware/                      tài liệu phần cứng, pinout, datasheet
├── 1-Build-Environment/             toolchain và môi trường build
├── 2-Zigbee-Radio-Silabs-EFR32/     firmware radio Zigbee/Thread
├── 3-Main-SoC-Realtek-RTL8196E/     bootloader + Linux system
├── docs-assets/                     CSS/assets cho website tài liệu
├── lib/                             hàm shell dùng chung
├── backup_gateway.sh                backup toàn bộ SPI NOR
├── restore_gateway.sh               restore full flash
├── build_fullflash.sh               ghép bốn partition thành 16 MiB
├── create_fullflash.sh              ghép và hỗ trợ flash thủ công
├── flash_install_rtl8196e.sh        cài/upgrade Linux firmware hoàn chỉnh
├── flash_efr32.sh                   flash radio EFR32 qua mạng
├── mkdocs.yml                       cấu hình website MkDocs
└── README.md                        tài liệu vào dự án
```

Tên thư mục được đánh số để thể hiện thứ tự đọc và dependency:

1. Hiểu hardware.
2. Chuẩn bị compiler/tool.
3. Build/flash chip EFR32.
4. Build/flash hệ thống RTL8196E.

## 4. `0-Hardware`: tài liệu phần cứng

```text
0-Hardware/
├── README.md
├── datasheet/
└── media/
```

### `README.md`

Mô tả board vật lý và các linh kiện chính:

- RTL8196E: Lexra RLX4181, kiến trúc tương thích MIPS32 big-endian, 400 MHz.
- SPI NOR GD25Q127/GD25Q128 dung lượng 16 MiB.
- SDRAM 32 MiB.
- Module Tuya TYZS4 dùng EFR32MG1B232F256GM48.
- Header debug J1 kết hợp UART của RTL8196E và SWD của EFR32.
- UART debug RTL8196E dùng TTL 3,3 V, 38400 8N1.

### `datasheet/`

Chứa datasheet PDF của SoC, SPI flash, SDRAM, module Tuya và EFR32. Đây là nguồn dùng để xác định:

- Memory map và thanh ghi RTL8196E.
- Kích thước/erase block của SPI NOR.
- Pin UART, GPIO, SWD và radio.
- Giới hạn RAM/flash của EFR32.

### `media/`

Ảnh PCB, vị trí chip, header debug và hình minh họa cho tài liệu. Không tham gia quá trình build.

## 5. `1-Build-Environment`: môi trường build thống nhất

```text
1-Build-Environment/
├── Dockerfile
├── install_deps.sh
├── 10-lexra-toolchain/
├── 11-realtek-tools/
└── 12-silabs-toolchain/
```

Thư mục này giải quyết việc repository phải build cho hai kiến trúc khác nhau:

- RTL8196E: Lexra/MIPS Linux.
- EFR32: ARM Cortex-M4 bare-metal.

### `Dockerfile`

Tạo môi trường build có khả năng tái lập. Container cài dependency, build Lexra toolchain, Realtek tools và Silicon Labs tools. Source project được mount vào `/workspace`.

### `install_deps.sh`

Dùng khi build native trên Ubuntu 22.04 hoặc WSL2. Script cài package host rồi gọi các installer/build script con.

### `10-lexra-toolchain/`

```text
10-lexra-toolchain/
├── build_toolchain.sh
├── crosstool-ng.config
├── TOOLCHAIN_UPDATE.md
└── patches/
```

Chức năng:

- Dựng cross compiler `mips-lexra-linux-musl` bằng crosstool-NG.
- Toolchain hiện dùng GCC 15.2, binutils 2.45.1 và musl 1.2.6.
- Các patch xử lý đặc điểm Lexra không có instruction truy cập unaligned `lwl/lwr/swl/swr` như MIPS thông thường.
- Output nằm ở `<project>/x-tools/mips-lexra-linux-musl/`.

Không nên dùng một MIPS compiler bất kỳ thay thế, vì code sinh ra có thể chứa instruction mà Lexra không hỗ trợ.

### `11-realtek-tools/`

```text
11-realtek-tools/
├── build_tools.sh
├── cvimg/
├── lzma-4.65/
└── lzma-loader/
```

Chức năng:

- `cvimg`: đóng Realtek header/signature cho bootloader, kernel, rootfs hoặc userdata để bootloader biết loại image và flash offset.
- `lzma-4.65`: compressor dùng cho kernel/boot image.
- `lzma-loader`: loader giải nén LZMA kiểu cũ; tài liệu ghi luồng mới đã chuyển dần sang zboot.
- `build_tools.sh`: build các tool nói trên và `flash_erase`, đặt output vào thư mục `bin/`.

### `12-silabs-toolchain/`

Chứa `install_silabs.sh`, dùng để tải/cài:

- `slc-cli`.
- Gecko SDK.
- `arm-none-eabi-gcc`.
- Simplicity Commander.

Output nằm trong `<project>/silabs-tools/`. Các script build EFR32 tự dò và source môi trường này.

## 6. `2-Zigbee-Radio-Silabs-EFR32`: firmware chip radio

```text
2-Zigbee-Radio-Silabs-EFR32/
├── build_efr32.sh
├── make-all-bauds.sh
├── 20-EZSP-Reference/
├── 21-Simplicity-Studio/
├── 22-Backup-Flash-Restore/
├── 23-Bootloader-UART-Xmodem/
├── 24-NCP-UART-HW/
├── 25-RCP-UART-HW/
├── 26-OT-RCP/
└── 27-Router/
```

### `build_efr32.sh`

Build orchestrator cho toàn bộ firmware EFR32. Có thể build tất cả hoặc chọn từng target như bootloader, NCP, RCP, OT-RCP và Router.

### `make-all-bauds.sh`

Tạo ma trận firmware theo nhiều UART baud rate. Filename `.gbl` mang baud để script flash chọn đúng binary và đồng bộ `radio.conf` phía Linux.

### `20-EZSP-Reference/`

Tài liệu khái niệm, không phải source build. Giải thích:

- EmberZNet là Zigbee stack của Silicon Labs.
- SoC mode: stack và app cùng chạy trên EFR32.
- NCP mode: Zigbee stack chạy trên EFR32, host điều khiển bằng EZSP.
- RCP mode: EFR32 chủ yếu xử lý PHY/MAC, stack chạy trên host.
- Quan hệ giữa EmberZNet version, EZSP version và adapter Zigbee2MQTT/ZHA.

### `21-Simplicity-Studio/`

Hướng dẫn dùng Simplicity Studio V5 để tạo/build/flash project EFR32 bằng GUI. Đây là con đường thay thế cho build tự động bằng `slc-cli`.

Thư mục `media/` chứa ảnh hướng dẫn thao tác IDE.

### `22-Backup-Flash-Restore/`

Hướng dẫn backup, probe, flash và restore EFR32 bằng:

- J-Link/SWD và Simplicity Commander.
- Gecko Bootloader/UART.
- `universal-silabs-flasher` thông qua bridge của RTL8196E.

Thư mục `media/` chứa ảnh kết nối và thao tác. Đây là tài liệu recovery cho chip radio.

### `23-Bootloader-UART-Xmodem/`

```text
23-Bootloader-UART-Xmodem/
├── build_bootloader.sh
├── patches/
└── firmware/
```

Chức năng:

- Build Gecko Bootloader stage 2 hỗ trợ UART Xmodem.
- `patches/*.slcp`, `.slpb` và header cấu hình pin/UART là input cho `slc-cli`.
- `firmware/*.gbl` là image update bootloader qua bootloader hiện tại.
- `firmware/*combined.s37` dùng cho flash toàn bộ bằng SWD/J-Link.

Bootloader này là nền tảng để các firmware application sau có thể update từ xa mà không cần SWD.

### `24-NCP-UART-HW/`

NCP = Network Co-Processor.

```text
24-NCP-UART-HW/
├── build_ncp.sh
├── patches/
├── firmware/
├── docker/
└── media/
```

Chức năng:

- Zigbee stack EmberZNet chạy trên EFR32.
- Linux/Zigbee2MQTT/ZHA giao tiếp với NCP bằng EZSP qua UART/TCP bridge.
- Đây là lựa chọn đơn giản nhất cho Zigbee coordinator.
- Firmware dựng sẵn dùng EmberZNet 7.5.1 ở các baud 115200, 230400, 460800, 691200 và 892857.
- `patches/` chứa `.slcp`, source app/main và cấu hình USART/PTI được chép vào project do SDK sinh ra.
- `docker/docker-compose.yml` cung cấp ví dụ chạy Zigbee2MQTT phía host.

### `25-RCP-UART-HW/`

RCP = Radio Co-Processor dùng CPC.

```text
25-RCP-UART-HW/
├── build_rcp.sh
├── patches/
├── firmware/
├── cpcd/
├── zigbeed-7.5.1/
├── zigbeed-8.2.2/
├── rcp-stack/
├── docker/
└── measure_uart_overruns.sh
```

Chức năng:

- EFR32 chạy phần radio thấp; Zigbee stack chạy phía host.
- `cpcd/` build CPC daemon và patch transport TCP để host có thể kết nối qua gateway.
- `zigbeed-*` build Zigbee daemon host-side theo EmberZNet 7.5.1 hoặc 8.2.2.
- Có thể nâng stack phía host mà không reflash EFR32.
- `docker/` cung cấp compose cho cpcd + zigbeed + Zigbee2MQTT.
- `measure_uart_overruns.sh` đo lỗi/overrun UART khi thử baud và tải cao.
- Firmware dựng sẵn giới hạn 115200, 230400, 460800 vì `cpcd` dùng baud POSIX.

### `26-OT-RCP/`

OpenThread Radio Co-Processor sử dụng Spinel/HDLC.

```text
26-OT-RCP/
├── build_ot_rcp.sh
├── patches/
├── firmware/
├── docker/
├── range-testing/
├── OT-CTL-CHEATSHEET.md
└── THREAD-MATTER-PRIMER.md
```

Một firmware OT-RCP được dùng cho ba mô hình:

1. Zigbee-on-Host: Zigbee stack chạy trong Zigbee2MQTT phía host.
2. OTBR-on-host: gateway chỉ bridge UART ↔ TCP, OTBR chạy trên PC/server.
3. OTBR-on-gateway: `otbr-agent` chạy trực tiếp trên RTL8196E.

Thư mục con:

- `docker/`: compose cho cả ba use case.
- `range-testing/`: báo cáo/qui trình đo range và radio.
- `OT-CTL-CHEATSHEET.md`: lệnh vận hành Thread dataset/network.
- `THREAD-MATTER-PRIMER.md`: kiến thức nền Thread/Matter.
- `patches/`: source/config cho UARTDRV, flow control và project OT-RCP.

Firmware mặc định chạy 460800 baud với RTS/CTS. Hardware flow control rất quan trọng để tránh RX FIFO overrun khi Spinel traffic burst.

### `27-Router/`

Firmware Zigbee 3.0 Router dạng SoC:

- Zigbee stack và ứng dụng router cùng chạy trên EFR32.
- Không cần host Zigbee stack.
- Tự network steering để join mạng đang permit join.
- Có thể làm range extender và hỗ trợ sleepy child.
- Mini CLI qua UART/TCP port 8888 dùng để xem network hoặc vào bootloader.
- `patches/zap-*` là code sinh/cấu hình cluster từ Zigbee Application Framework.
- `firmware/` chứa `.gbl` dựng sẵn ở 115200 baud.

## 7. `3-Main-SoC-Realtek-RTL8196E`: Linux firmware

```text
3-Main-SoC-Realtek-RTL8196E/
├── build_rtl8196e.sh
├── flash_remote.sh
├── 30-Backup-Restore/
├── 31-Bootloader/
├── 32-Kernel/
├── 33-Rootfs/
├── 34-Userdata/
└── 35-Migration/
```

### Phân vùng SPI NOR

Hệ thống custom dùng bốn partition:

| MTD | Offset | Kích thước | Nội dung |
|---|---:|---:|---|
| `mtd0` | `0x000000` | 128 KiB | bootloader + config |
| `mtd1` | `0x020000` | 1920 KiB | kernel |
| `mtd2` | `0x200000` | 2 MiB | SquashFS rootfs |
| `mtd3` | `0x400000` | 12 MiB | JFFS2 userdata |

Thiết kế tách rootfs read-only và userdata persistent giúp:

- Base system khó bị hỏng do mất điện.
- Cấu hình, SSH key, password, Thread credentials tồn tại qua upgrade.
- Có thể thay kernel/rootfs mà vẫn giữ userdata.

### `build_rtl8196e.sh`

Build orchestrator cho SoC chính. Có thể build `bootloader`, `kernel`, `rootfs`, `userdata` hoặc tổ hợp. Script tìm Lexra toolchain rồi gọi build script trong từng thư mục con.

### `flash_remote.sh`

Flash từng partition từ xa theo luồng:

```text
SSH vào Linux
  -> lưu cấu hình nếu flash userdata
  -> chạy boothold + reboot
  -> đợi SSH tắt
  -> đợi bootloader xuất hiện ở BOOT_IP
  -> TFTP image
  -> bootloader tự nhận dạng và flash partition
```

Script này chỉ dùng được với custom firmware có `boothold`, không dùng trực tiếp cho stock Tuya.

### `30-Backup-Restore/`

Chứa tài liệu và script backup/restore SPI NOR theo ba cấp độ:

1. SSH khi Linux còn chạy.
2. Bootloader `FLR/FLW` + TFTP khi Linux hỏng.
3. CH341A khi bootloader cũng hỏng.

Các script đáng chú ý:

- `split_flash.sh`: tách full flash 16 MiB thành partition custom hoặc layout Tuya.
- `scripts/restore_mtd_via_ssh.sh`: restore MTD qua SSH cho firmware gốc.

`media/` chứa ảnh đấu nối SPI programmer và quy trình recovery.

### `31-Bootloader/`

Đây là bootloader open-source riêng cho RTL8196E, không phải U-Boot chuẩn.

```text
31-Bootloader/
├── boot/
├── btcode/
├── doc/
├── build_bootloader.sh
├── flash_bootloader.sh
└── boot.bin
```

#### `boot/`

Phần monitor/bootloader chính:

- `head.S`, `inthandler.S`: startup và interrupt entry.
- `arch.c`, `irq.c`: init kiến trúc/interrupt.
- `uart.c`: console 38400.
- `flash.c`: SPI NOR read/write/erase.
- `swCore.c`, `swNic.c`, `swTable.c`: switch/Ethernet/NIC để phục vụ TFTP.
- `monitor.c`: command prompt `<RealTek>` và các command như FLR/FLW.
- `main.c`: init và quyết định boot/download mode.
- `ld.script`: memory layout/link address.

#### `btcode/`

Bootstrap/LZMA stage:

- Startup assembly.
- LZMA decoder.
- `piggy.S` nhúng payload.
- Code load/decompress và jump sang bootloader chính.

#### `doc/`

Tài liệu nội bộ về toolchain, command, reset vector, test RAM-only và cơ chế reboot từ Linux về bootloader.

#### Build/flash output

- `boot.bin`: image để flash.
- `btcode/build/test.bin`: image test chạy từ RAM, tránh brick trong lúc phát triển.

Bootloader custom còn hỗ trợ:

- TFTP.
- Ping.
- Auto nhận dạng image theo header.
- Flash fullflash hoặc partition.
- UDP `OK/FAIL` để script host tự động biết kết quả.
- Nhận magic `HOLD` và BOOT_IP từ Linux qua DRAM.

### `32-Kernel/`

Port Linux 6.18 cho RTL8196E/Lexra.

```text
32-Kernel/
├── build_kernel.sh
├── flash_kernel.sh
├── config-6.18-realtek.txt
├── files-6.18/
├── patches-6.18/
├── scripts/
└── kernel-6.18.img
```

#### `build_kernel.sh`

Tải/chuẩn bị Linux upstream, apply patch, copy file overlay, cấu hình, build kernel + DTB rồi đóng Realtek image header.

#### `patches-6.18/`

Các patch nối platform RTL8196E vào Linux upstream:

- Kconfig/Makefile cho MIPS platform.
- CPU probing và Lexra exception/entry/cache handling.
- IRQ/timer core adjustment.
- Hook Kconfig/Makefile cho clocksource, irqchip, GPIO, LED, Ethernet, SPI, UART và watchdog.
- Một số patch network/timer phục vụ SoC/driver đặc thù.

#### `files-6.18/`

Các file mới không có trong upstream:

- `arch/mips/realtek/`: platform setup, PROM, IRQ dispatch, IMEM.
- `arch/mips/mm/c-lexra.c`: cache implementation cho Lexra.
- `arch/mips/boot/dts/realtek/rtl819x.dtsi`: SoC-level device tree.
- `rtl8196e.dts`: board-level device tree, partition, GPIO, UART, Ethernet và radio wiring.
- `timer-rtl819x.c`: clocksource/timer.
- `irq-rtl819x.c`: interrupt controller.
- `gpio-rtl819x.c`: GPIO controller.
- `spi-rtl819x.c`: SPI controller/NOR access.
- `8250_rtl819x.c`: UART 16550 variant.
- `rtl819x_wdt.c`: watchdog.
- `leds-gpio-pwm.c`: LED brightness bằng software PWM.
- `rtl8196e-eth/`: Ethernet driver gồm ring/descriptor, hardware ops, DT parsing và netdev core.
- `rtl8196e-uart-bridge/`: in-kernel UART ↔ TCP bridge cho EFR32.

#### UART bridge

Driver `rtl8196e-uart-bridge` thay userspace `serialgateway` cũ. Nó:

- Mở `/dev/ttyS1` phía EFR32.
- Expose TCP port 8888.
- Cho đổi baud, flow control, enable và bind address qua module parameter/sysfs.
- Được `S50uart_bridge` quản lý ở Zigbee mode.
- Được tắt để `otbr-agent` sở hữu UART trực tiếp ở Thread mode.
- Được chuyển sang chế độ flash khi `flash_efr32.sh` upload GBL.

### `33-Rootfs/`

Base root filesystem nhỏ, read-only:

```text
33-Rootfs/
├── busybox/
├── dropbear/
├── skeleton/
├── build_rootfs.sh
├── flash_rootfs.sh
└── rootfs.bin
```

#### `busybox/`

- `build_busybox.sh`: build BusyBox bằng Lexra/musl.
- `busybox.config`: chọn applet cho init, shell, network, mount, watchdog và tool cơ bản.
- BusyBox cung cấp `/bin/sh`, init, syslog, udhcpc, ip, mount và phần lớn command hệ thống.

#### `dropbear/`

Build Dropbear SSH server/client nhỏ gọn, phù hợp thiết bị 32 MiB RAM và rootfs 2 MiB.

#### `skeleton/`

Filesystem template gồm:

- `/init`, `/etc/inittab`, `/etc/init.d/rcS` cho boot.
- BusyBox symlink/app command.
- Dropbear binary.
- Cấu hình hostname, network, NTP, timezone, passwd.
- Mount point `/userdata`.

#### `build_rootfs.sh`

Assemble skeleton + BusyBox + Dropbear, tạo SquashFS và dùng `cvimg` đóng header thành `rootfs.bin`.

### `34-Userdata/`

Partition JFFS2 read-write chứa cấu hình và ứng dụng lớn/thay đổi thường xuyên:

```text
34-Userdata/
├── skeleton/
├── boothold/
├── keepalive/
├── s40button/
├── nano/
├── ethtool/
├── iperf3/
├── ot-br-posix/
├── build_userdata.sh
├── flash_userdata.sh
└── userdata.bin
```

#### Các component

- `boothold/`: ghi magic và BOOT_IP vào DRAM để reboot vào bootloader download mode.
- `keepalive/`: giám sát/giữ dịch vụ hoạt động.
- `s40button/`: xử lý nút trước mặt gateway, long-press để reset/recover EFR32.
- `nano/`: build nano và ncurses/terminfo.
- `ethtool/`: tool chẩn đoán Ethernet.
- `iperf3/`: đo throughput mạng.
- `ot-br-posix/`: build `otbr-agent`, `ot-ctl` và thành phần OpenThread Border Router cho Lexra.

#### `skeleton/etc/init.d/`

Trình tự init chính:

| Script | Chức năng |
|---|---|
| `S05syslog` | khởi động log |
| `S10network` | cấu hình Ethernet static/DHCP |
| `S11leds` | áp brightness/mode LED |
| `S15hostname` | đặt hostname |
| `S20time` | timezone/NTP |
| `S25watchdog` | watchdog phần cứng |
| `S26panicrec` | ghi/khôi phục thông tin kernel panic |
| `S30dropbear` | SSH server |
| `S40button` | nút nhấn/recovery EFR32 |
| `S50uart_bridge` | Zigbee mode: UART ↔ TCP bridge |
| `S70otbr` | Thread mode: chạy otbr-agent |
| `S90checkpasswd` | cảnh báo password mặc định |

`radio.conf` quyết định ai sở hữu `/dev/ttyS1`:

- Không có `MODE=otbr`: `S50uart_bridge` chạy.
- Có `MODE=otbr`: `S50uart_bridge` bỏ qua, `S70otbr` chạy.

### `35-Migration/`

Tài liệu migration từ Tuya/firmware cũ sang layout custom. Mô tả:

- First flash qua serial + bootloader.
- Upgrade qua SSH và bảo toàn cấu hình.
- Migration `radio.conf` giữa các version.
- Fullflash install và per-partition remote flash.
- Điều kiện recovery/rollback.

## 8. Các script ở repository root

### `build_fullflash.sh`

Ghép chính xác bốn image vào buffer 16 MiB đã fill `0xFF`:

```text
boot.bin          -> 0x000000, bỏ cvimg header
kernel-6.18.img   -> 0x020000, giữ cs6c header
rootfs.bin        -> 0x200000, bỏ cvimg header
userdata.bin      -> 0x400000, bỏ cvimg header
```

Sau khi ghép, script kiểm tra kích thước và magic tại từng offset:

- Bootloader code.
- `cs6c` kernel header.
- SquashFS `hsqs`.
- JFFS2 magic `0x1985`.

### `create_fullflash.sh`

Wrapper tương tác cũ/tiện dụng hơn: hỏi network/radio config, build userdata, ghép fullflash và tùy chọn upload TFTP. Với bootloader cũ, người dùng vẫn nhập `FLW` trên UART.

### `flash_install_rtl8196e.sh`

Entry point chính để cài hoặc upgrade toàn hệ thống RTL8196E:

- Không truyền Linux IP: giả định board đang ở bootloader, thực hiện first install.
- Có Linux IP: SSH vào gateway, lưu cấu hình, chạy `boothold`, reboot, TFTP fullflash.
- Với bootloader mới: image được auto-flash và gửi UDP notification.
- Với bootloader Tuya/cũ: script hướng dẫn thao tác serial.
- Hỗ trợ `-y` để chạy không tương tác và `--boot-ip` cho subnet khác.

### `backup_gateway.sh`

Tự phát hiện gateway đang ở:

- Custom Linux.
- Tuya Linux.
- Bootloader.

Nếu SSH được thì đọc `/dev/mtdX`; nếu chỉ còn bootloader thì hướng dẫn FLR/TFTP. Output gồm `fullflash.bin`, từng partition và log.

### `restore_gateway.sh`

Xác minh fullflash đúng 16 MiB, phát hiện loại bootloader rồi:

- Bootloader custom: upload fullflash để auto-flash.
- Bootloader cũ: hướng dẫn LOADADDR/TFTP/FLW.

### `flash_efr32.sh`

Entry point flash chip radio qua mạng:

1. Chọn bootloader/NCP/RCP/OT-RCP/Router và baud.
2. Chuẩn bị Python venv + `universal-silabs-flasher`.
3. SSH vào RTL8196E, dừng daemon đang sở hữu radio.
4. Cấu hình in-kernel bridge và pulse `nRST`.
5. Probe app hiện tại bằng EZSP/CPC/Spinel hoặc Gecko Bootloader.
6. Chuyển EFR32 vào bootloader và upload `.gbl` bằng Xmodem qua `socket://IP:8888`.
7. Ghi `FIRMWARE`, version, baud và `MODE` vào `/userdata/etc/radio.conf`.
8. Reboot gateway.

### `lib/ssh.sh`

Thư viện shell dùng chung cho các script flash:

- SSH retry và timeout.
- ControlMaster để không hỏi password nhiều lần.
- Hỗ trợ `SSH_PASSWORD` + `sshpass` cho CI.
- Validate IPv4 và cleanup socket.

## 9. Cách các image được build và liên kết với nhau

```text
Lexra toolchain
   |
   +--> bootloader source ------------> boot.bin
   |
   +--> Linux 6.18 + patches/files ---> kernel-6.18.img
   |
   +--> BusyBox + Dropbear + skeleton -> rootfs.bin
   |
   +--> apps + config + init scripts --> userdata.bin
                                            |
                                            v
                                  build_fullflash.sh
                                            |
                                            v
                                      fullflash.bin
                                      (16 MiB NOR)

Silabs toolchain
   |
   +--> Gecko Bootloader -------------> bootloader*.gbl
   +--> NCP project -------------------> ncp*.gbl
   +--> RCP project -------------------> rcp*.gbl
   +--> OpenThread RCP ---------------> ot-rcp*.gbl
   +--> Zigbee Router ----------------> z3-router*.gbl
                                            |
                                            v
                                      flash_efr32.sh
                                            |
                                   UART bridge + Xmodem
                                            |
                                            v
                                         EFR32
```

Hai firmware liên kết ở ba điểm:

1. UART vật lý giữa RTL8196E và EFR32.
2. In-kernel bridge TCP port 8888 hoặc `otbr-agent` sở hữu trực tiếp UART.
3. `/userdata/etc/radio.conf` là nguồn cấu hình baud/mode/firmware identity.

## 10. Vai trò của các định dạng file

| Định dạng | Vai trò |
|---|---|
| `.gbl` | firmware EFR32 có thể upload qua Gecko Bootloader |
| `.s37` | image Motorola S-record dùng SWD/Commander |
| `.slcp` | project/config đầu vào cho Silicon Labs Configurator |
| `.patch` | thay đổi Linux upstream hoặc SDK-generated source |
| `.dts/.dtsi` | mô tả SoC, board, pin, partition và peripheral |
| `boot.bin` | bootloader RTL8196E đã đóng gói |
| `kernel-6.18.img` | kernel + DTB/compression + Realtek header |
| `rootfs.bin` | SquashFS base system + Realtek header |
| `userdata.bin` | JFFS2 persistent/apps + Realtek header |
| `fullflash.bin` | raw image toàn bộ SPI NOR 16 MiB |

## 11. Nên bắt đầu đọc source từ đâu?

### Muốn hiểu toàn hệ thống

1. `README.md` ở root.
2. `0-Hardware/README.md`.
3. `3-Main-SoC-Realtek-RTL8196E/README.md`.
4. `2-Zigbee-Radio-Silabs-EFR32/README.md`.

### Muốn hiểu quá trình boot RTL8196E

1. `31-Bootloader/boot/head.S`.
2. `31-Bootloader/boot/main.c`.
3. `31-Bootloader/boot/monitor.c`.
4. `32-Kernel/files-6.18/arch/mips/realtek/setup.c`.
5. `32-Kernel/files-6.18/arch/mips/boot/dts/realtek/rtl8196e.dts`.
6. `33-Rootfs/skeleton/init` và `etc/init.d/rcS`.
7. `34-Userdata/skeleton/etc/init.d/rcS`.

### Muốn hiểu Zigbee bridge

1. `rtl8196e-uart-bridge/README.md` và `DESIGN.md`.
2. `rtl8196e_uart_bridge_main.c`.
3. `34-Userdata/skeleton/etc/init.d/S50uart_bridge`.
4. `flash_efr32.sh`.
5. `24-NCP-UART-HW/README.md` hoặc `25-RCP-UART-HW/README.md`.

### Muốn hiểu Thread/Matter

1. `26-OT-RCP/THREAD-MATTER-PRIMER.md`.
2. `26-OT-RCP/README.md`.
3. `34-Userdata/ot-br-posix/README.md`.
4. `34-Userdata/skeleton/etc/init.d/S70otbr`.

### Muốn build/flash

1. `1-Build-Environment/README.md`.
2. `build_fullflash.sh`.
3. `flash_install_rtl8196e.sh`.
4. `flash_efr32.sh`.
5. `30-Backup-Restore/README.md` trước khi ghi flash.

## 12. Nhận xét kiến trúc

Điểm mạnh:

- Phân chia rõ hardware, toolchain, EFR32 và Linux SoC.
- Có binary dựng sẵn nhưng vẫn giữ source/build script.
- Layout partition tách rootfs read-only và userdata persistent.
- Có nhiều tầng recovery: SSH, bootloader/TFTP, SWD và SPI programmer.
- Cấu hình radio tập trung trong `radio.conf`.
- Device-tree hóa board-specific data, thuận tiện port sang RTL8196E board khác.
- Script flash có validate, preserve config và automation khá đầy đủ.

Điểm cần lưu ý khi phát triển:

- Lexra không hoàn toàn tương đương MIPS thông thường; bắt buộc dùng đúng toolchain và patch architecture.
- SPI NOR chỉ 16 MiB và RAM chỉ 32 MiB, nên kernel/rootfs/app phải nhỏ.
- Kernel 6.18 dựa trên tập patch/overlay lớn; khi nâng kernel phải audit lại architecture core và driver.
- `rootfs.bin` read-only, thay đổi persistent phải đặt trong userdata.
- Baud/flow-control hai đầu UART phải khớp tuyệt đối.
- Flash bootloader hoặc fullflash luôn có nguy cơ brick; cần backup 16 MiB trước.
- NCP, RCP, OT-RCP và Router không chỉ khác file firmware; daemon/config phía RTL8196E cũng phải chuyển đúng mode.

## 13. Kết luận

Repository này là một BSP/firmware distribution hoàn chỉnh chứ không chỉ là một bản hack đơn lẻ. Nó bao phủ:

- Hardware reverse engineering.
- Toolchain cho Lexra và ARM.
- Bootloader riêng cho RTL8196E.
- Linux 6.18 port và driver board-specific.
- SquashFS rootfs tối giản.
- JFFS2 userdata chứa app/config.
- Nhiều firmware Zigbee/Thread cho EFR32.
- Build, backup, migration, recovery và remote flash.

Ý tưởng trung tâm là biến RTL8196E thành Linux host/bridge có thể bảo trì từ xa, còn EFR32 trở thành radio có firmware thay đổi theo use case. Hiểu được ranh giới hai chip, bốn partition SPI NOR và ownership của UART `/dev/ttyS1` là đủ để lần theo phần còn lại của repository.
