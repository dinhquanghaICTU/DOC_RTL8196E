# RTL8196E — quy trình flash và ghi chú porting Yocto

Tài liệu này ghi lại quy trình đã thực hiện thành công trên Lidl/Silvercrest
Gateway dùng RTL8196E, đồng thời tổng hợp layout SPI NOR, RAM và định dạng image
từ source `hacking-lidl-silvercrest-gateway`.

> **Cảnh báo:** đây không phải flow U-Boot chuẩn. Bootloader hiện tại là monitor
> của Realtek (prompt `<RealTek>`). Kernel không phải một `zImage + dtb` tách rời:
> source custom tạo `vmlinuz` zboot, nhúng DTB vào kernel, rồi bọc bằng header
> Realtek `cvimg` có signature `cs6c`.

## 1. Phần cứng liên quan

- SoC: Realtek RTL8196E, Lexra RLX4181, big-endian, 400 MHz.
- RAM: 32 MiB, physical `0x00000000..0x01ffffff`.
- SPI NOR: 16 MiB GD25Q127/128, erase block 64 KiB.
- Console RTL8196E: TTL 3.3 V, `38400 8N1`, không flow control.
- Bootloader/TFTP IP mặc định: `192.168.1.6`.
- Linux custom IP đã chọn khi flash: `192.168.1.88/24`.
- Radio: EFR32MG1B nối RTL8196E qua UART1, được bridge ra TCP `8888`.

J1 dùng khi flash RTL8196E:

| J1 | Chức năng | USB–UART |
|---:|---|---|
| 1 | 3.3 V/VREF | Không cấp nguồn từ adapter |
| 2 | GND | GND |
| 3 | RTL8196E TX | RX adapter |
| 4 | RTL8196E RX | TX adapter |
| 5 | EFR32 SWDIO | Không dùng cho RTL8196E |
| 6 | EFR32 SWCLK | Không dùng cho RTL8196E |

Gateway dùng nguồn riêng; USB–UART phải là logic 3.3 V.

## 2. Chuẩn bị Windows + WSL2

Đã dùng Ubuntu 22.04 trong WSL2:

```powershell
wsl --install -d Ubuntu-22.04
```

WSL Settings:

- `Networking mode = Mirrored`.
- Khi TFTP bị timeout, tạm tắt `Hyper-V Firewall`.
- Tạm tắt Windows Defender Firewall cho đúng Ethernet network; bật lại sau khi
  hoàn tất.

Đặt IPv4 tĩnh cho Ethernet Windows:

```text
Address: 192.168.1.2
Mask:    255.255.255.0
Gateway: để trống
```

Khởi động lại WSL networking:

```powershell
wsl --shutdown
wsl -d Ubuntu-22.04
```

Clone repo vào filesystem Linux để tránh CRLF (`/bin/bash^M`) và lỗi permission
trên `/mnt/c`:

```bash
cd ~
sudo apt update
sudo apt install -y git build-essential tftp-hpa netcat-openbsd \
  iproute2 squashfs-tools mtd-utils fakeroot python3-venv
git clone https://github.com/jnilo1/hacking-lidl-silvercrest-gateway.git
cd ~/hacking-lidl-silvercrest-gateway
```

Kiểm tra WSL đi trực tiếp ra Ethernet:

```bash
ip route get 192.168.1.6
```

Kết quả đúng có dạng sau và **không có** `via 172.x.x.x`:

```text
192.168.1.6 dev eth0 src 192.168.1.2
```

## 3. Quy trình flash RTL8196E đã thực hiện

### 3.1 Vào bootloader

1. Mở PuTTY/Tera Term đúng COM, `38400 8N1`, flow control `None`.
2. Nối Ethernet trực tiếp PC ↔ gateway.
3. Rút/cắm nguồn và nhấn `ESC` liên tục.
4. Dừng tại:

```text
<RealTek>
```

Không gõ lại chuỗi `<RealTek>`; nó chỉ là prompt.

### 3.2 Build và upload full image

Trong WSL:

```bash
cd ~/hacking-lidl-silvercrest-gateway
./flash_install_rtl8196e.sh
```

Cấu hình đã chọn:

```text
Network: Static
IP:      192.168.1.88
Netmask: 255.255.255.0
Gateway: 192.168.1.1
Radio:   Zigbee / NCP 115200
```

Script build `fullflash.bin` đúng 16 MiB và TFTP tới `192.168.1.6`. Upload chỉ
hoàn tất khi UART báo đủ `0x01000000` byte:

```text
**TFTP Client Upload File Size = 01000000 Bytes at 80500000
Success!
<RealTek>
```

Lúc này image **chỉ nằm trong RAM** tại `0x80500000`. Rút nguồn trước `FLW` sẽ
làm mất image nhưng firmware cũ trong flash vẫn còn.

### 3.3 Ghi RAM xuống toàn bộ SPI NOR

Bootloader stock trên board thực tế hiển thị cú pháp có tham số SPI chip count:

```text
FLW <dst_ROM_offset> <src_RAM_addr> <length_Byte> <SPI cnt#>
```

Lệnh dùng để ghi toàn bộ 16 MiB:

```text
FLW 00000000 80500000 01000000 0
```

Nếu hỏi thì trả lời `Y`. Dãy dấu `.` dài là tiến trình erase/write. Không rút
nguồn cho đến khi `<RealTek>` quay lại. Sau đó boot image mới:

```text
J BFC00000
```

Nếu chỉ TFTP rồi power-cycle mà chưa `FLW`, máy sẽ boot lại Tuya Linux cũ.

### 3.4 Kiểm tra Linux mới

```bash
ssh root@192.168.1.88
```

Tài khoản mặc định là `root`; mật khẩu mặc định thường là `root`. Đổi ngay:

```sh
passwd
uname -a
cat /proc/mtd
```

## 4. Flash và kiểm tra EFR32 Zigbee

Nạp firmware NCP 7.5.1 ở 115200 baud:

```bash
cd ~/hacking-lidl-silvercrest-gateway
./flash_efr32.sh -g 192.168.1.88 ncp
```

Trong lần thực hiện, tool phát hiện firmware cũ EZSP `6.5.5.0 build 432`, sau
đó flash thành công `ncp-uart-hw-7.5.1-115200.gbl`.

Kiểm tra trên gateway:

```sh
netstat -lnt | grep 8888
dmesg | grep -i uart
cat /userdata/etc/radio.conf
```

Kỳ vọng:

```text
FIRMWARE=ncp
FIRMWARE_BAUD=115200
0.0.0.0:8888 ... LISTEN
```

Kiểm tra từ host:

```bash
nc -vz 192.168.1.88 8888
```

Zigbee2MQTT:

```yaml
serial:
  port: tcp://192.168.1.88:8888
  adapter: ember
```

## 5. Layout SPI NOR custom 16 MiB

Nguồn chuẩn là `build_fullflash.sh` và fixed partitions trong
`rtl8196e.dts`.

| MTD | Nội dung | Offset đầu | Kích thước | Offset cuối (exclusive) | Dạng lưu trong full flash |
|---|---|---:|---:|---:|---|
| mtd0 | `boot+cfg` | `0x000000` | `0x020000` (128 KiB) | `0x020000` | Raw bootloader, bỏ header cvimg 16 byte |
| mtd1 | `linux` | `0x020000` | `0x1e0000` (1920 KiB) | `0x200000` | Giữ header `cs6c` 16 byte |
| mtd2 | `rootfs` | `0x200000` | `0x200000` (2 MiB) | `0x400000` | Raw SquashFS, bỏ header cvimg 16 byte |
| mtd3 | `jffs2-fs` | `0x400000` | `0xc00000` (12 MiB) | `0x1000000` | Raw big-endian JFFS2, bỏ header cvimg 16 byte |

Sơ đồ:

```text
SPI NOR 0x000000 ------------------------------------------------ 0x1000000
        | boot+cfg |       linux       | rootfs |     userdata      |
        0x000000   0x020000            0x200000 0x400000      0x1000000
          128 KiB      1920 KiB          2 MiB        12 MiB
```

Magic được `build_fullflash.sh` kiểm tra:

| Offset | Magic bytes | Ý nghĩa |
|---:|---|---|
| `0x000000` | `0b f0 00 04` | Raw bootloader |
| `0x020000` | `63 73 36 63` | ASCII `cs6c`, kernel cvimg |
| `0x200000` | `68 73 71 73` | `hsqs`, SquashFS big-endian representation |
| `0x400000` | `19 85` | JFFS2 big-endian magic |

`fullflash.bin` được fill `0xff`, sau đó đặt từng payload tại đúng offset. Đây
là layout cần tái tạo bằng Yocto image recipe/custom image class; WIC mặc định
không tự sinh đúng định dạng này.

## 6. Header Realtek cvimg

Header dài 16 byte, khai báo trong `cvimg.c`:

```c
struct img_header {
    unsigned char signature[4];
    uint32_t start_addr;
    uint32_t burn_addr;
    uint32_t len;
} __attribute__((packed));
```

Các trường số được ghi network byte order/big-endian. Payload có checksum 16-bit
ở cuối. `start_addr` là địa chỉ chạy/load, `burn_addr` là offset SPI NOR.

Các image component:

- Kernel: signature `cs6c`, burn address `0x00020000`, **giữ header trong
  flash**.
- Rootfs: build riêng có signature `r6cr`, start `0x80c00000`, burn
  `0x00200000`; khi ghép fullflash thì bỏ 16-byte header và đặt raw SquashFS ở
  `0x200000`.
- Userdata: signature `r6cr`, start `0x80c00000`, burn `0x00400000`; khi ghép
  fullflash thì bỏ header và đặt raw JFFS2 ở `0x400000`.
- Bootloader `boot.bin`: khi ghép fullflash bỏ header 16 byte, raw code bắt đầu
  tại flash offset zero.

Do đó không được đặt thẳng một Yocto `zImage`, `uImage`, DTB hoặc rootfs vào
flash mà không tái tạo đúng header/magic/layout bootloader đang mong đợi.

## 7. RAM map và địa chỉ boot quan trọng

### 7.1 Physical RAM và MIPS aliases

DTS khai báo:

```text
Physical RAM: 0x00000000..0x01ffffff (32 MiB)
KSEG0 cached: 0x80000000..0x81ffffff
KSEG1 uncached alias: 0xa0000000..0xa1ffffff
```

Các vùng quan trọng lấy từ bootloader docs/source:

| Vùng | Địa chỉ | Ghi chú |
|---|---|---|
| MIPS exception vectors | `0x80000000..0x800001ff` | Không được dùng làm TFTP load address |
| RAM-test bootloader | `0x80100000..0x80200000` | Vùng test tương đối an toàn |
| Stage-2 bootloader | khoảng `0x80400000..0x80421600` | Không ghi đè khi monitor đang chạy |
| TFTP/default kernel buffer | từ `0x80500000` | Fullflash 16 MiB đã được upload tại đây |
| Kernel loaded region | `0x80500000..~0x81e00000` | Tránh dùng làm scratch lúc kernel boot |
| Boothold reserved page | physical `0x01ffe000..0x01ffefff` | DTS `reserved-memory` |
| Boothold flag alias | quanh `0xa1ffeffc` | KSEG1 uncached |
| Reset/flash boot jump | `0xbfc00000` | `J BFC00000` |

`0x80500000` là KSEG0 alias của physical `0x00500000`; 16 MiB upload chiếm tới
`0x81500000`, vẫn nằm trong 32 MiB RAM nhưng đè lên vùng kernel runtime, nên chỉ
dùng trong bootloader/download mode.

### 7.2 Kernel load/entry và DTB

Kernel source dùng:

```text
arch/mips/boot/compressed/ (zboot)
Platform load address: 0xffffffff80000000 (32-bit alias 0x80000000)
```

Quy trình `build_kernel.sh`:

1. Build `vmlinux` và compressed zboot ELF `vmlinuz`.
2. DTB Lidl được chọn bằng `CONFIG_DTB_RTL8196E_GEN=y` và **built into kernel**.
3. Đọc entry point thực tế từ ELF `vmlinuz` bằng `readelf`.
4. `objcopy -O binary vmlinuz vmlinuz.bin`.
5. Bọc bằng `cvimg`:

```bash
cvimg \
  -i vmlinuz.bin \
  -o kernel-6.18.img \
  -s cs6c \
  -e <ENTRY_LAY_TU_VMLINUZ_ELF> \
  -b 0x00020000 \
  -a 4k
```

Vì entry được lấy từ ELF của đúng build, không nên hard-code một entry suy đoán
trong Yocto. Recipe phải dùng `readelf` tương tự.

Bootargs hiện nằm trong built-in DTS:

```text
console=ttyS0,38400 loglevel=7 root=/dev/mtdblock2 rootfstype=squashfs
```

Rootfs vì thế phải nằm ở mtd2/offset `0x200000` và là SquashFS tương thích.

## 8. Rootfs và userdata

Rootfs custom:

- SquashFS read-only.
- Compression XZ, block size 256 KiB.
- Giới hạn partition: 2 MiB.
- `/dev/console`, `/dev/null`, `/dev/zero` được tạo trong image.

Userdata:

- JFFS2 big-endian (`mkfs.jffs2 -b`).
- Erase block `0x10000` (64 KiB).
- Zlib-only; kernel không bật mọi compressor JFFS2.
- Partition 12 MiB tại `0x400000`.
- Chứa network config, SSH keys/password, init scripts và `radio.conf`.

Đối với Yocto, cấu hình hợp lý là rootfs SquashFS read-only và một recipe/image
riêng tạo JFFS2 persistent data. Không đưa toàn bộ rootfs writable vào JFFS2 nếu
muốn giữ boot time và độ bền flash.

## 9. Checklist porting sang Yocto

1. Tạo `meta-rtl8196e/conf/machine/rtl8196e.conf`.
2. Dùng `ARCH=mips`, big-endian và Lexra cross-toolchain
   `mips-lexra-linux-musl-`; GCC MIPS generic chưa chắc sinh code an toàn cho
   RLX4181.
3. Tạo kernel recipe áp toàn bộ `patches-6.18`, overlay `files-6.18` và config
   RTL8196E.
4. Giữ built-in DTB, hoặc sửa cả bootloader/kernel protocol nếu muốn DTB tách.
5. Build zboot `vmlinuz`, lấy entry từ ELF, objcopy và bọc `cvimg/cs6c`.
6. Giới hạn kernel image cả header trong `0x1e0000` byte.
7. Tạo SquashFS XZ không vượt `0x200000` byte.
8. Tạo big-endian JFFS2 eraseblock 64 KiB tại offset `0x400000`.
9. Tạo custom deploy task ghép đúng `fullflash.bin` 16 MiB và kiểm tra magic.
10. Test kernel/rootfs từng partition trước; chỉ thay mtd0/bootloader khi có
    backup và SPI programmer.

## 10. Các lỗi thực tế và bài học

- Repo trên `/mnt/c` bị CRLF gây `/bin/bash^M`; clone vào `~/...` trong WSL.
- WSL NAT tạo route `via 172.x`; phải dùng Mirrored mode để thấy trực tiếp
  `192.168.1.6`.
- Hyper-V/Windows Firewall có thể làm TFTP timeout hoặc lặp WRQ.
- Chỉ coi TFTP thành công khi UART báo đủ `01000000 Bytes`.
- TFTP chỉ đưa image vào RAM; `FLW` mới thực sự ghi SPI NOR.
- Bootloader stock của board này cần biến thể `FLW ... 0` có SPI count.
- UART monitor dễ mất ký tự nếu paste nhanh; nên gõ chậm hoặc dùng transmit
  delay 50–100 ms/ký tự.
- Gọi `FLW` thiếu tham số từng gây `Undefined Exception`; không thử command ghi
  flash với thiếu/sai đối số.
- Không trả lời `y` cho script trước khi UART thực sự hoàn tất `FLW`.
- Không power-cycle trong lúc dãy dấu chấm erase/write đang chạy.
- Sau khi flash EFR32, `radio.conf`, baud và firmware thật trên chip phải khớp.

## 11. Source-of-truth trong repo

- Flash assembly: `build_fullflash.sh`.
- DTS/layout: `3-Main-SoC-Realtek-RTL8196E/32-Kernel/files-6.18/arch/mips/boot/dts/realtek/rtl8196e.dts`.
- SoC DTSI/RAM/MMIO: `.../rtl819x.dtsi`.
- Kernel packaging: `3-Main-SoC-Realtek-RTL8196E/32-Kernel/build_kernel.sh`.
- Header generator: `1-Build-Environment/11-realtek-tools/cvimg/cvimg.c`.
- Rootfs: `3-Main-SoC-Realtek-RTL8196E/33-Rootfs/build_rootfs.sh`.
- Userdata/JFFS2: `3-Main-SoC-Realtek-RTL8196E/34-Userdata/build_userdata.sh`.
- Bootloader RAM notes: `3-Main-SoC-Realtek-RTL8196E/31-Bootloader/doc/TESTING.md`.
- Boot flow/boothold: `3-Main-SoC-Realtek-RTL8196E/31-Bootloader/doc/REBOOT_TO_BOOTLOADER.md`.

