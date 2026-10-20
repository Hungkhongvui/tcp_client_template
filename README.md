# Đo thông lượng TCP trên Zynq với lwIP

Workspace AMD Vitis chứa hai ứng dụng C sử dụng **lwIP RAW API** để đo tốc độ truyền TCP qua Ethernet. Các ứng dụng chạy bare-metal trên `ps7_cortexa9_0`, sử dụng platform `platformtest` dựa trên phần cứng `zed.xsa` (ZedBoard, Zynq-7000).

| Ứng dụng | Vai trò của board | Hướng truyền dữ liệu |
| --- | --- | --- |
| `lwip_tcp_perf_client1` | TCP client, kết nối tới máy tính chạy iperf server | Board → máy tính |
| `lwip_tcp_perf_server1` | TCP server, nhận kết nối từ máy tính chạy iperf client | Máy tính → board |

Thông tin kết nối, lượng dữ liệu truyền và băng thông được in ra console UART. Mỗi lần thử nghiệm chỉ cần chạy một trong hai ứng dụng trên board.

## Cấu trúc repo

```text
.
├── lwip_tcp_perf_client1/
│   ├── src/
│   │   ├── main.c                 # Khởi tạo platform, lwIP và mạng
│   │   ├── tcp_perf_client.c      # Kết nối, gửi dữ liệu và thống kê
│   │   ├── tcp_perf_client.h      # IP server, cổng và thời gian đo
│   │   ├── CMakeLists.txt         # Cấu hình build ứng dụng
│   │   └── README.txt             # Tài liệu mẫu gốc
│   ├── _ide/                     # Cấu hình chạy/debug và khởi tạo PS
│   └── vitis-comp.json
├── lwip_tcp_perf_server1/
│   ├── src/
│   │   ├── main.c
│   │   ├── tcp_perf_server.c      # Lắng nghe, nhận dữ liệu và thống kê
│   │   ├── tcp_perf_server.h      # Cổng và chu kỳ báo cáo
│   │   ├── CMakeLists.txt
│   │   └── README.txt
│   ├── _ide/
│   └── vitis-comp.json
└── platformtest/
    ├── hw/zed.xsa                 # Mô tả phần cứng
    ├── ps7_cortexa9_0/            # Domain standalone, BSP và lwIP
    ├── zynq_fsbl/                 # First Stage Boot Loader
    └── vitis-comp.json
```

Các file `platform*.c`, `i2c_access.c`, `iic_phyreset.c`, `sfp.c` và `si5324.c` cung cấp phần khởi tạo và hỗ trợ phần cứng theo cấu hình build.

## Yêu cầu

- Board ZedBoard phù hợp với `platformtest/hw/zed.xsa`, nguồn cấp và cáp Ethernet.
- Kết nối JTAG để nạp/debug và UART để xem log.
- AMD Vitis cùng bộ công cụ biên dịch ARM và thư viện embedded tương ứng. Metadata trong repo tham chiếu bản **2026.1.1** trên Windows.
- Máy tính chạy **iperf 2**. Tài liệu mẫu đi kèm sử dụng iperf **2.0.5**; các lệnh dưới đây dùng chương trình `iperf` thuộc dòng này.
- Terminal UART được cấu hình theo baud rate của BSP/phần cứng.

Các ứng dụng liên kết với `xilstandalone`, `xiltimer`, `lwip220` và `xil`. Đây là dự án embedded phụ thuộc BSP và toolchain của Vitis; chạy CMake thông thường trên máy tính chưa đủ để build.

## Cấu hình mạng

Hai ứng dụng dùng chung các giá trị mặc định sau:

| Thông số | Giá trị | Nơi cấu hình |
| --- | --- | --- |
| IPv4 của board khi không dùng được DHCP | `192.168.1.10` | `src/main.c` của từng ứng dụng |
| Subnet mask | `255.255.255.0` | `src/main.c` |
| Gateway | `192.168.1.1` | `src/main.c` |
| MAC của board | `00:0a:35:00:01:02` | `mac_ethernet_address` trong `src/main.c` |
| IP máy tính mà board client kết nối tới | `192.168.1.100` | `tcp_perf_client.h`: `TCP_SERVER_IP_ADDRESS` |
| Cổng TCP | `5001` | `tcp_perf_client.h` / `tcp_perf_server.h`: `TCP_CONN_PORT` |
| Chu kỳ báo cáo | `5` giây | Cả hai header: `INTERIM_REPORT_INTERVAL` |
| Thời gian gửi của board client | `300` giây | `tcp_perf_client.h`: `TCP_TIME_INTERVAL` |

BSP hiện có `LWIP_DHCP=1` và `LWIP_IPV6=0`. Board sẽ xin địa chỉ DHCP trước; nếu hết thời gian chờ mà chưa có địa chỉ, ứng dụng chuyển sang IPv4 mặc định. Luôn lấy địa chỉ thực tế từ dòng `Board IP` trên UART.

Để thử bằng IPv4 mặc định, đặt card Ethernet của máy tính thành `192.168.1.100/24`, rồi nối với board trực tiếp hoặc qua switch. Nếu mạng đang cấp DHCP, bảo đảm địa chỉ máy tính khớp `TCP_SERVER_IP_ADDRESS` khi chạy board client.

Khi dùng nhiều board trên cùng mạng, đặt MAC và IP riêng cho từng board.

## Build và nạp chương trình

1. Clone repo hoặc mở thư mục repo đã có trên máy.
2. Mở AMD Vitis và chọn thư mục gốc repo làm workspace. Nếu các component chưa xuất hiện, import các component hiện có: `platformtest`, `lwip_tcp_perf_client1` và `lwip_tcp_perf_server1`.
3. Kiểm tra platform dùng phần cứng `platformtest/hw/zed.xsa`, processor `ps7_cortexa9_0` và domain `standalone_ps7_cortexa9_0`.
4. Kiểm tra lại các đường dẫn theo máy. `platformtest/vitis-comp.json`, BSP và `compile_commands.json` chứa tham chiếu tới thư mục cài Vitis trên máy tạo dự án. Nếu Vitis báo thiếu đường dẫn, chọn lại XSA/toolchain và tái sinh BSP hoặc metadata build trong Vitis.
5. Build platform `platformtest` trước, sau đó build ứng dụng cần chạy.
6. Cấp nguồn board, kết nối JTAG, UART và Ethernet; mở terminal UART.
7. Chọn cấu hình Run/Debug phần cứng của ứng dụng, kiểm tra target JTAG và khởi tạo PS7, rồi nạp và chạy ELF tương ứng.

Tên ELF được định nghĩa trong CMake:

```text
lwip_tcp_perf_client1.elf
lwip_tcp_perf_server1.elf
```

Cấu hình launch đi kèm tham chiếu ELF trong thư mục `build/` của từng ứng dụng. Các thư mục build/export bị bỏ qua bởi `.gitignore`, vì vậy cần tạo lại bằng Vitis sau khi clone.

## Thử nghiệm 1: board gửi dữ liệu tới máy tính

```text
Board: lwip_tcp_perf_client1  ── TCP/5001 ──>  Máy tính: iperf server
```

1. Đặt IP Ethernet của máy tính thành `192.168.1.100`, hoặc sửa `TCP_SERVER_IP_ADDRESS` trong [tcp_perf_client.h](lwip_tcp_perf_client1/src/tcp_perf_client.h) theo IP thực tế rồi build lại.
2. Trên máy tính, khởi động iperf server trước:

   ```sh
   iperf -s -p 5001 -i 5 -w 2M
   ```

3. Nạp và chạy `lwip_tcp_perf_client1` trên board.
4. Theo dõi UART và terminal iperf. Board gửi dữ liệu trong `300` giây và báo cáo định kỳ mỗi `5` giây theo cấu hình mặc định.

Để đo lại, chạy lại ứng dụng client trên board trong khi iperf server vẫn đang lắng nghe.

## Thử nghiệm 2: máy tính gửi dữ liệu tới board

```text
Máy tính: iperf client  ── TCP/5001 ──>  Board: lwip_tcp_perf_server1
```

1. Nạp và chạy `lwip_tcp_perf_server1` trên board.
2. Đợi UART hiển thị địa chỉ board và thông báo lắng nghe cổng TCP.
3. Trên máy tính, chạy lệnh sau; thay IP nếu board nhận địa chỉ khác qua DHCP:

   ```sh
   iperf -c 192.168.1.10 -p 5001 -i 5 -t 300 -w 2M
   ```

4. Đọc thống kê trên UART và terminal iperf. Trong chế độ này, tham số `-t` của iperf trên máy tính quyết định thời gian gửi.

`Transfer` thể hiện lượng dữ liệu đã truyền; `Bandwidth` thể hiện tốc độ truyền trong khoảng đo. Repo chưa cung cấp số liệu benchmark thực tế để đối chiếu.

## Thay đổi cấu hình

- **IP, subnet, gateway, MAC:** sửa `main.c` của ứng dụng cần chạy.
- **Đích kết nối và thời gian gửi của client:** sửa `TCP_SERVER_IP_ADDRESS`, `TCP_CONN_PORT`, `TCP_TIME_INTERVAL` trong `tcp_perf_client.h`.
- **Cổng server:** sửa `TCP_CONN_PORT` trong `tcp_perf_server.h`, đồng thời cập nhật `-p` ở lệnh iperf.
- **Chu kỳ thống kê:** sửa `INTERIM_REPORT_INTERVAL` trong header tương ứng.
- **DHCP, IPv6 và bộ đệm TCP:** chỉnh cấu hình thư viện lwIP trong BSP/domain bằng Vitis, tái sinh BSP rồi build lại ứng dụng. File `bsp/include/lwipopts.h` phản ánh cấu hình đã sinh và có thể bị ghi đè khi tái sinh.

### IPv6

Mã nguồn có nhánh hỗ trợ IPv6 nhưng BSP hiện tắt tính năng này. Sau khi bật IPv6 trong cấu hình lwIP và build lại:

- Board tạo địa chỉ IPv6 link-local từ MAC và in địa chỉ lên UART.
- Với board client, cập nhật `TCP_SERVER_IPV6_ADDRESS` trong `tcp_perf_client.h` thành địa chỉ của máy tính.
- Với board server, dùng địa chỉ trên UART và chỉ định interface để xác định phạm vi link-local.

Ví dụ với iperf 2 trên máy tính:

```sh
# Máy tính làm server IPv6
iperf -V -s -p 5001 -i 5 -w 2M

# Máy tính làm client IPv6; thay địa chỉ và interface theo thực tế
iperf -V -c fe80::20a:35ff:fe00:102%eth0 -p 5001 -i 5 -t 300 -w 2M
```

## Xử lý lỗi thường gặp

| Hiện tượng | Kiểm tra |
| --- | --- |
| Không thấy log UART | Nguồn board, cổng serial, baud rate và ứng dụng đã chạy qua bước khởi tạo |
| `DHCP request timed out` | DHCP trên mạng; sau đó xem board có chuyển về `192.168.1.10` hay không |
| Client không kết nối được | Server đã chạy, IP đích, cổng `5001`, subnet, dây Ethernet và firewall cho phép TCP trên cổng thử nghiệm |
| iperf trên máy tính không kết nối được board | Ứng dụng server đã chạy và IP trong lệnh đúng với `Board IP` trên UART |
| Build báo thiếu BSP, toolchain hoặc file | Đường dẫn cài Vitis/XSA, platform và domain đã chọn; tái sinh BSP rồi build lại |
| Kết quả khác nhau giữa các lần đo | Tốc độ link Ethernet, lưu lượng nền, cấu hình bộ đệm TCP và cấu hình build |

## Tài liệu và bản quyền

- [README mẫu TCP client](lwip_tcp_perf_client1/src/README.txt)
- [README mẫu TCP server](lwip_tcp_perf_server1/src/README.txt)

Mã nguồn mẫu và các thư viện đi kèm có thông báo bản quyền, điều khoản giấy phép riêng của AMD/Xilinx và các thành phần bên thứ ba. Xem phần đầu từng file và giấy phép đi kèm thư viện khi sử dụng hoặc phân phối lại.
