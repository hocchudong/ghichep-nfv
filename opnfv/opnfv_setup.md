# Cài đặt OPNFV Colorado 3.0 sử dụng Fuel Installer
# Mục lục
### [1. Yêu cầu](#pre)
### [2. Mô hình cài đặt](#topo)
### [3. Tiến hành cài đặt](#steps)
### [4. Kiểm tra chức năng](#verify)
### [5. Tham khảo](#ref)

---
*Chú ý: Bài lab sau là kịch bản triển khai trên nền tảng ảo hóa KVM. Hệ thống thực tế nên sử dụng hạ tầng vật lý để đảm bảo hiệu năng.*

## <a name="pre"></a>1. Yêu cầu
- Server cấu hình tối thiểu: 32GB RAM, CPU 8 - 16 cores, 1TB HDD.
- Server cài đặt ubuntu server 16.04 (hoặc 14.04) và môi trường ảo hóa KVM ([webvirt-kvm](https://github.com/retspen/webvirtmgr)), tạo tối thiểu 3 dải mạng cho cài đặt:
  - **ADMIN** network: 10.20.0.0/24 - là dải để máy installer quản trị, PXE boot, cấu hình và kiểm thử các kịch bản trên môi trường OPNFV (yêu cầu tắt DHCP). 
  - **PUBLIC** network: 192.168.2.0/24 - là dải để kết nối ra internet cho hệ thống OPNFV.
  - **MANAGEMENT + STORAGE + PRIVATE** network: 10.10.0.0/24 - là dải phục vụ chung cho các lưu lượng liên quan tới quản lý, giao tiếp giữa các tiến trình, giao tiếp giữa các instance (yêu cầu giới hạn dải DHCP). Dải mạng này có thể tách ra thành 3 dải mạng riêng biệt theo hướng dẫn cài đặt bên dưới.

## <a name="topo"></a>2. Mô hình cài đặt
- Mô hình cài đặt ở đây là mô hình noha, bao gồm 3 node (ảo hóa) như sau:
  - **FUEL MASTER**:
    - Cài đặt Fuel Installer (dựa trên CentOS) và các plugins cho phép kích hoạt các tính năng cần thiết của môi trường (ví dụ: tacker, ovs-nsh, opendaylight plugin,etc.)
    - Cấu hình: 4vCPUs, 4GB RAM, 80GB ổ cứng.
    - Ba card mạng: **ADMIN**, **PUBLIC**, **MANAGEMENT**.

  - **CONTROLLER**:
    - Đóng vai trò tương tự như **CONTROLLER** node trong OpenStack, ngoài các project core của OpenStack còn cài đặt thêm các thành phần cần thiết khác như: OpenDaylight, Tacker, Ceilometer, Heat, OVS-NSH 
    - Cấu hình: 8vCPUs, 16 - 18GB RAM, 400GB ổ cứng.
    - Ba hoặc năm card mạng: ADMIN, PUBLIC, STORAGE, MANAGEMENT, PRIVATE.
  
  - **COMPUTE**:
    - Đóng vai trò tương tự như **COMPUTE** node trong OpenStack
    - Cấu hình: 6vCPUs, 6 - 8GB RAM, 80 - 100GB ổ cứng.
    - Ba hoặc năm card mạng: ADMIN, PUBLIC, STORAGE, MANAGEMENT, PRIVATE.
  
![OPNFV](http://i.imgur.com/rpT0EB3.png)


## <a name="steps"></a>3. Tiến hành cài đặt



## <a name="verify"></a>4. Kiểm tra chức năng



## <a name="ref"></a>5. Tham khảo




