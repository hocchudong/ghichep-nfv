# Cài đặt OPNFV Colorado 3.0 sử dụng Fuel Installer
# Mục lục
### [1. Yêu cầu](#pre)
### [2. Mô hình cài đặt](#topo)
### [3. Các bước cài đặt](#steps)
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
    - Ba hoặc năm card mạng: ADMIN, PUBLIC, STORAGE, MANAGEMENT, PRIVATE (DATA NETWORK).
  
![OPNFV](http://i.imgur.com/rpT0EB3.png)


## <a name="steps"></a>3. Các bước cài đặt
### 3.1 Chuẩn bị
- Cài webvirtmgr (bao gồm cả môi trường kvm và giao diện đồ họa trên trình duyệt) lên server theo hướng dẫn [tại đây](https://github.com/retspen/webvirtmgr).
- Thiết lập các network:
  - Thêm các bridge cần thiết như sau (chú ý: card eno1 của server là card kết nối internet):
    - Bridge external:

      ```sh
      brctl addbr br-ex 
      brctl addif br-ex eno1
      ifconfig eno1 0
      ifconfig br-ex 192.168.2.50/24
      route add default gw 192.168.2.1
      ```

    - Lưu lại cấu hình vào file `/etc/network/interfaces` với nội dung tương tự như sau:
     
      ```sh
      # PUBLIC NETWORK
      auto br-ex
      iface br-ex inet static
          bridge_ports eno1 
              address 192.168.2.50/24
      	gateway 192.168.2.1
      	dns-nameservers 8.8.8.8
      
      iface eno1 inet manual
        up ifconfig $IFACE 0.0.0.0 up
        up ip link set $IFACE promisc on
        down ip link set $IFACE promisc off
        down ifconfig $IFACE down
      ```

  - Tải các tệp định nghĩa network [ở đây](https://github.com/thaihust/ghichep-nfv/tree/master/opnfv/opnfv-setup-network-config).
  - Thực hiện tạo các network như sau, sau đây là mẫu tạo network ADMIN, các network khác tạo tương tự:

    ```sh
    virsh net-define 1_admin-net.xml
    virsh net-start admin-net
    virsh net-autostart admin-net
    ```

    *Chú ý: __1_admin-net.xml__ là tên tệp định nghĩa network, __admin-net__ là tên network chỉ định trong section __\<name\>\</name\>__ của tệp đó.*

### 3.2 Cài đặt Fuel Master node
- Tải iso của Fuel Installer [tại đây](http://artifacts.opnfv.org/fuel/colorado/opnfv-colorado.3.0.iso).
- Truy cập giao diện webvirtmgr trên server, đảm bảo đã tạo hai vùng lưu trữ (__default__ - lưu trữ image các máy ảo và __iso__ - lưu trữ các file iso) tương tự như sau:

![opnfv storages](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/server_storage.png?raw=true)

- Truy cập vào vùng lưu trữ __iso__ và tải lên iso của fuel installer như hình bên dưới:

![opnfv iso](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/server_iso.png?raw=true)

- Truy cập vào vùng lưu trữ các image máy ảo, tạo trước 3 images để chuẩn bị tạo máy ảo cho các máy Fuel Master Installer, Controller và Compute, tương tự như sau:

![opnfv images](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/server_images.png?raw=true)

- Đảm bảo các network cần thiết đã cấu hình và hiển thị như sau:

![networks](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/networks.png?raw=true)

- Chuyển sang tab __Instances__, tạo ra 3 máy ảo Fuel Master, Controller và Compute với cấu hình tương tự như sau:
  - Fuel master node:
  ![opnfv master](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_properties.png?raw=true)

  - Controller node:
  ![opnfv controller](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv_ctl_properties.png?raw=true)

  - Compute node:
  ![opnfv compute](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv_com_properties.png?raw=true)

  - *__Chú ý__: Các máy ảo __Controller__ và __Compute__ trên tab __Settings__/__Media__ không được kết nối với bất kì file iso nào, riêng với __Fuel Master__ node thì kết nối với iso __opnfv-colorado.3.0.iso__ như sau:*
  ![connect fuel iso](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_iso_connect.png?raw=true)

- Truy cập máy ảo Fuel Master, chuyển sang tab __Power__, click __Start__ để bắt đầu cài đặt. Sau đó chuyển sang tab __Access__, click __Console__ để theo dõi, giao diện sẽ tương tự như sau:
![fuel first time](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup.png?raw=true)
    
Lựa chọn __1. Fuel Install (Static IP)__ để bắt đầu cài đặt.

- Đợi Fuel Install cài đặt cơ bản xong hệ điều hành và các gói phụ thuộc ban đầu (mất khoảng 20 phút), đến khi xuất hiện giao diện cấu hình môi trường như sau:
![fuel installation config](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_1.png?raw=true)
Bước này yêu cầu thay đổi mật khẩu đăng nhập vào Fuel GUI, tuy nhiên có thể để mặc định là __admin/admin__ không cần thay đổi.
- Chuyển xuống tab __Networks__, cấu hình cho 3 card mạng của Fuel Master Node như sau:
  - Card __eth0__ - card tương ứng __admin_network__, giữ nguyên cấu hình như sau:
  ![admin card](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_2.png?raw=true)
  - Card __eth1__ - card tương ứng __public_network__, cấu hình với địa chỉ phù hợp với dải mạng public lựa chọn với gateway chính xác:
  ![public card](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_3.png?raw=true)     
  
  Di chuyển con trỏ xuống tùy chọn __Apply__ để áp dụng thay đổi.
  - Card __eth2__ - card tương ứng __management_network__, cấu hình với địa chỉ __10.10.0.30__ như hình dưới (chú ý nên đặt địa chỉ tránh dải dhcp của mạng __management_network__ nằm trong 10.10.0.2-10.10.0.29):
  ![mgmt card](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_4.png?raw=true)
  
  Di chuyển con trỏ xuống tùy chọn __Apply__ để áp dụng thay đổi.

- Chuyển xuống tab __Security Setup__, giữ nguyên tùy chọn:
![security setup](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_5.png?raw=true)

- Chuyển xuống tab __PXE Setup__. Bước này lựa chọn PXE interface để từ đó __Fuel Master__ cài đặt các máy __Controller__ và __Compute__, ở đây lựa chọn card eth0 tương ứng __admin_network__. Chỉnh sửa lại DHCP pool cho phép cấp dải địa chỉ giới hạn, ở đây lựa chọn dải __10.20.0.31-10.20.0.60__. Fuel Master node đóng vai trò như DHCP server cấp địa chỉ trong dải đó cho card mạng thuộc __admin_network__ (eth0) của các máy __Controller__ và __Compute:
![pxe setup](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_6.png?raw=true)

  Di chuyển con trỏ xuống tùy chọn __Check__ để kiểm tra.

- Chuyển xuống tab __DNS & Hostname__ đổi __External DNS__ thành __8.8.8.8__, sau đó chuyển con trỏ xuống tùy chọn __Apply__ để áp dụng thay đổi và tùy chọn __Check__ để kiểm tra kết nối internet thông suốt:
![dns](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_7.png?raw=true)

- Chuyển xuống tab __Bootstrap image__, sau đó chuyển con trỏ xuống tùy chọn __Check__ để kiểm tra các repo cho cho bootstrap image. Các repo này trỏ tới các máy chủ nguồn cần thiết để cài đặt các gói cho môi trường OPNFV trên các máy __Controller__ và __Compute__:
![bootstrap](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_8.png?raw=true)

- Chuyển xuống tab __Time Sync__ để chỉ định các NTP server (cho mục đích đồng bộ về thời gian giữa các node trong môi trường OPNFV). Chú ý đổi lại các ntp server sang châu Á như hình bên dưới:
![ntp](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_9.png?raw=true)

- Chuyển xuống tab __Quit and Setup__, chuyển con trỏ sang tùy chọn __Save and Quit__ để __Fuel Master__ hoàn thành quá trình cài đặt, tạo các bootstrap image:
![quit](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/fuel_master_setup_10.png?raw=true)

## <a name="verify"></a>4. Kiểm tra chức năng



## <a name="ref"></a>5. Tham khảo




