# Cách vào bootloader và flash RTL8196E

## 1. Mục đích

Tài liệu này ghi lại hai cách đưa RTL8196E vào dấu nhắc bootloader:

```text
<RealTek>
```

Tại đây có thể truyền firmware qua Ethernet/TFTP và ghi vào SPI NOR. Hai cách gồm:

1. Nhấn `ESC` qua UART khi cấp nguồn hoặc reset.
2. Chạy `boothold` trong Linux rồi reboot.

> `boothold` là chức năng của bootloader tùy biến trong project jnilo1. Firmware và bootloader gốc có thể không hỗ trợ cách này.

## 2. Chuẩn bị

- USB-to-UART mức điện áp **3.3 V**.
- Ethernet nối máy tính với gateway.
- TFTP client trên máy tính.
- File cần nạp như `boot.bin`, kernel, rootfs hoặc `fullflash.bin`.
- Backup SPI NOR và máy nạp SPI để cứu thiết bị khi sửa bootloader.

Cấu hình UART của bootloader:

```text
Baud rate : 38400
Data bits : 8
Parity    : none
Stop bits : 1
Flow ctrl : none
```

Không nối chân nguồn 5 V của USB-to-UART vào board. Chỉ nối `GND`, `TX` và `RX`, đồng thời đấu chéo `TX ↔ RX`.

## 3. Cách 1: nhấn ESC qua UART

Cách này dùng khi Linux không chạy nhưng bootloader trong SPI NOR vẫn còn hoạt động.

### Các bước

1. Kết nối UART 3.3 V và mở terminal ở `38400 8N1`.
2. Nhấn hoặc giữ `ESC` liên tục.
3. Cấp nguồn hoặc reset gateway.
4. Tiếp tục nhấn `ESC` trong cửa sổ khởi động, khoảng 3 giây.
5. Khi thành công sẽ xuất hiện:

```text
<RealTek>
```

Trong source jnilo1, bootloader kiểm tra ký tự `ESC` trong quá trình tìm và nạp kernel. Khi nhận được `ESC`, nó bỏ luồng boot Linux và chuyển sang download mode.

### Khi ESC không hoạt động

Kiểm tra lần lượt:

- UART có đúng `38400 8N1` không.
- TX/RX có đấu chéo không.
- Adapter có dùng mức 3.3 V không.
- Có nhấn `ESC` từ trước thời điểm reset không.
- Bootloader trong NOR có còn nguyên và có hỗ trợ ngắt boot bằng `ESC` không.

Nếu bootloader gốc không cho vào prompt theo cách thông thường, phương pháp kéo chân SPI NOR `SCLK` xuống GND mà tài liệu phần cứng của board hướng dẫn chỉ nên dùng như **recovery lần đầu**. Không dùng thường xuyên vì thao tác sai thời điểm có thể gây chập hoặc làm hỏng giao tiếp flash.

## 4. Cách 2: boothold từ Linux

Cách này thuận tiện nhất sau khi đã cài bootloader jnilo1 và Linux vẫn hoạt động. Không cần mở UART hay kéo chân SPI NOR.

Đăng nhập SSH rồi chạy:

```bash
boothold && reboot
```

Với bootloader V2.7 trở lên, có thể truyền cả địa chỉ IP dùng trong download mode:

```bash
boothold 192.168.1.6 && reboot
```

Luồng thực hiện:

```text
Linux chạy boothold
        |
        +-- ghi HOLD magic vào vùng RAM dành riêng
        +-- reboot
                |
                v
Bootloader đọc HOLD magic
        |
        +-- xóa cờ one-shot
        +-- không boot kernel
        +-- dừng tại <RealTek> và chạy TFTP server
```

Cờ là one-shot: sau lần vào download mode, lần reboot tiếp theo sẽ boot Linux bình thường nếu không ghi lại cờ.

### Điều kiện để boothold hoạt động trong image Yocto

Khi tự port sang Yocto phải giữ đồng bộ:

- Chương trình `/usr/bin/boothold` trong rootfs.
- Bootloader có logic kiểm tra HOLD magic.
- Vùng `reserved-memory` trong DTS để kernel không sử dụng vùng RAM chứa cờ.
- Địa chỉ và giá trị magic của bootloader phải trùng với chương trình `boothold`.

Nếu tự viết lại công cụ, có thể đổi tên thành `hnn-recovery`; nguyên lý vẫn là ghi magic vào vùng RAM dành riêng rồi reboot.

## 5. Flash qua TFTP sau khi đã vào `<RealTek>`

Bootloader mặc định sử dụng IP:

```text
192.168.1.6
```

Đặt máy tính cùng subnet, ví dụ `192.168.1.1/24`, rồi kiểm tra kết nối:

```bash
ping 192.168.1.6
```

Nạp một image bằng TFTP:

```bash
tftp -m binary 192.168.1.6 -c put boot.bin
```

Với bootloader jnilo1 V2.5 trở lên, có thể gửi full image 16 MiB:

```bash
tftp -m binary 192.168.1.6 -c put fullflash.bin
```

Bootloader nhận dạng loại image bằng header hoặc magic tại các offset rồi ghi vào vùng tương ứng của SPI NOR.

## 6. Nên dùng cách nào

| Trạng thái thiết bị | Cách nên dùng |
|---|---|
| Linux và SSH còn chạy | `boothold [IP] && reboot` |
| Linux hỏng, bootloader còn chạy | UART `38400 8N1`, nhấn `ESC` khi reset |
| Bootloader gốc không bắt được ESC | Recovery phần cứng SCLK theo đúng hướng dẫn board, chỉ để cài lần đầu |
| Bootloader hoặc vùng đầu NOR đã hỏng | Máy nạp SPI NOR |

Không nên cập nhật `fullflash.bin` cho OTA thông thường vì nó chứa cả bootloader. Khi mất điện lúc ghi vùng bootloader, thiết bị có thể brick. Trong vận hành nên chỉ cập nhật kernel/rootfs hoặc dùng thiết kế OTA A/B.

## 7. Source tham chiếu

```text
/home/quanghaictu/WORK/dumpTUYA/hacking-lidl-silvercrest-gateway/3-Main-SoC-Realtek-RTL8196E/31-Bootloader/boot/main.c
/home/quanghaictu/WORK/dumpTUYA/hacking-lidl-silvercrest-gateway/3-Main-SoC-Realtek-RTL8196E/31-Bootloader/doc/REBOOT_TO_BOOTLOADER.md
/home/quanghaictu/WORK/dumpTUYA/hacking-lidl-silvercrest-gateway/3-Main-SoC-Realtek-RTL8196E/31-Bootloader/doc/TESTING.md
/home/quanghaictu/WORK/dumpTUYA/hacking-lidl-silvercrest-gateway/flash_install_rtl8196e.sh
```
