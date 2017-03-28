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

- Chuyển xuống tab __PXE Setup__. Bước này lựa chọn PXE interface để từ đó __Fuel Master__ cài đặt các máy __Controller__ và __Compute__, ở đây lựa chọn card eth0 tương ứng __admin_network__. Chỉnh sửa lại DHCP pool cho phép cấp dải địa chỉ giới hạn, ở đây lựa chọn dải __10.20.0.31-10.20.0.60__. Fuel Master node đóng vai trò như DHCP server cấp địa chỉ trong dải đó cho card mạng thuộc __admin_network__ (eth0) của các máy __Controller__ và __Compute__:
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

- Sau khi cài đặt xong, truy cập vào fuel master (tài khoản: `root/r00tme`) thực hiện cài đặt một số plugin cần thiết:

```sh
cd /opt/opnfv
fuel plugins --install fuel-plugin-ovs-0.9-0.9.0-1.noarch.rpm
fuel plugins --install opendaylight-0.9-0.9.0-1.noarch.rpm
fuel plugins --install tacker-0.2-0.2.0-1.noarch.rpm
```

- Truy cập fuel master node trên trình duyệt: `https://<fuel_master_ip>:8443` với tài khoản `admin/admin`:
![fuel-gui](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_login.png?raw=true)

- Thực hiện tạo môi trường OpenStack mới theo các bước sau:
  - Click `New OpenStack Environment`:
  ![new ops env](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_create_env.png?raw=true)
  - Đặt tên môi trường:
  ![env name](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_create_env_1.png?raw=true)
  - Chọn hypervisor sử dụng cho các máy ảo trên môi trường OpenStack là `QEMU-KVM`:
  ![hyper](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_create_env_2.png?raw=true)
  - Lựa chọn tùy chọn network cho tenant network là tunel network:
  ![tunnel](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_create_env_3.png?raw=true)
  - Lựa chọn storage backends, ở đây sử dụng LVM, có thể lựa chọn CEPH tùy theo nhu cầu:
  ![lvm](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_create_env_4.png?raw=true)
  - Lựa chọn cài đặt thêm một số dịch vụ khác, ở đây không chọn thêm, tuy nhiên nếu đủ tài nguyên có thể tích vào tùy chọn cài thêm Ceilometer:
  ![additional](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_create_env_5.png?raw=true)
  - Kết thúc tiến trình tạo môi trường, click `Create`:
  ![finish](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_create_env_6.png?raw=true)

- Trên giao diện `webvirtmgr`, truy cập vào hai máy ảo `OPNFV-CTL`(Controller Node) và `OPNFV-COM` (Compute Node), bật hai máy ảo này lên, lúc này hai máy ảo sẽ boot lên nhờ bootstrap image từ `Fuel Master` node. Truy cập vào tab `Access` sẽ thấy giao diện chuẩn bị boot như sau:
![bootstrap](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_bootstrap.png?raw=true)

- Sau khi bootstrap image cài đặt xong, trên giao diện của fuel master sẽ có thông báo, kiểm tra danh sách các node:
![node list](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_add_node_list_nodes.png?raw=true)

- Kiểm tra danh sách các node trên giao diện dòng lệnh: `fuel nodes`:
![fuel nodes](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_nodes.png?raw=true)

- Đổi tên hostname cho các OpenStack node như sau:

```sh
fuel node --node 1 --hostname compute
fuel node --node 2 --hostname controller
```

Trong đó 1, 2 là id của nodes lấy từ lệnh `fuel nodes`. Kết quả tương tự như sau:
![change hostname](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_change_hostname.png?raw=true)

- Kiểm tra danh sách các plugins đã cài:
![plugins](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_list_plugins.png?raw=true)

- Chuyển sang tab `Settings -> Compute`, lựa chọn hypervisor type là KVM:
![hypervisor](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_setting_compute.png?raw=true)

- Chuyển xuống tab `Other` lựa chọn plugins sẽ cài đặt, bao gồm OVS với NSH và OpenDaylight plugins, tích chọn như hình:
![ovs-odl](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_setting_ovs_nsh_n_odl.png?raw=true)

- Chuyển sang tab `OpenStack Services`, tích chọn cài đặt `Tacker` cho môi trường OPNFV:
![tacker](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_setting_tacker.png?raw=true)

- Chuyển sang tab `Networks -> default`, cấu hình lại các tham số về network cho phù hợp như sau:
  - Public network, cấp dải từ `192.168.2.60-192.168.2.99`:
![public](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_network_public.png?raw=true)
  - Storage, management, private, chú ý đặt dải nằm ngoài dải dhcp tương ứng các dải mạng đã thiết lập lúc ban đầu (đọc các file xml để biết thêm dải dhcp nằm trong khoảng nào để đặt tránh bị chồng lấn):
![others](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_network_others.png?raw=true)

- Chuyển xuống tab `Neutron L3`, đặt dải `Floating IP` nằm ngoài dải cấp cho public network ở phía trên, tương tự như hình dưới:
![floating IP](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_network_l3.png?raw=true)

- Chuyển xuống tab `Other`, tích vào tùy chọn `Assign public network to all nodes`:
![assign](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_network_assign_public_all_node.png?raw=true)

- Chuyển sang mục `Nodes`, tiến hành gán roles cho các node như sau:
  - Chuyển sang mục `Nodes`, click `Add Nodes`:
  ![nodes](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_add_node.png?raw=true)
  - Gán role `Compute` cho node `Compute` rồi click `Apply Change`:
  ![compute role](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_add_node_compute.png?raw=true)
  - Gán role `Controller, OpenDaylight, Tacker, Cinder` cho node `Controller`, click `Apply Change`:
  ![controller roles](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_add_node_controller.png?raw=true)
  - Sau khi gán thành công ta có 2 node với các roles tương ứng như sau:
  ![roles](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_add_node_nodes_with_roles.png?raw=true)

- Vẫn tại mục `Nodes`, lúc này click `Select All` rồi click `Configure Interfaces`:
![cfg ifs](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_node_cfg_inf.png?raw=true)

- Kéo thả các roles vào các card mạng phù hợp như hình, ở đây theo thứ tự như sau: eth0 (admin) - eth1 (public) - eth2 (storage) - eth3 (management) - eth4 (private) sau đó lưu lại thay đổi.
![ad](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_node_cfg_inf_1.png?raw=true)
![other](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_node_cfg_inf_2.png?raw=true)

- Chuyển qua tab `Networks -> Connectivity Check`, click `Verify` để tiến hành kiểm tra các kết nối, két quả thành công tương tự như sau:
![verify](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/fuel_network_verify.png?raw=true)

- Chuyển về mục `Dashboard`, bắt đầu triển khai hệ thống theo các thao tác sau:
![deploy](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/opnfv_deploy.png?raw=true)
![deploy](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/opnfv_deploy_1.png?raw=true)
![deploy](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv-gui/opnfv_deploy_2.png?raw=true)

- Đợi khoảng nửa tiếng tới 45 phút cho tới khi hệ thống deploy xong, kết quả thành công sẽ hiển thị tương tự như sau:
![success](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/fuel_deploy_success.png?raw=true)

- Truy cập thử vào horizon của OpenStack (tài khoản `admin/admin`):
![horizon](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv_gui.png?raw=true)
![horizon](https://github.com/thaihust/ghichep-nfv/blob/master/opnfv/opnfv-setup-images/opnfv/opnfv_gui_1.png?raw=true)


## <a name="verify"></a>4. Kiểm tra chức năng



## <a name="ref"></a>5. Tham khảo
