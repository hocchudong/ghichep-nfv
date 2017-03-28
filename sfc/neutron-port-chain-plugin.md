# Networking SFC - System design and workflow

## System architecture
Hình vẽ dưới đây thể hiện kiến trúc chung của Port Chain plugin. Theo như sơ đồ này, port chain plugin có thể triển khai nhiều loại driver để tương tác với các service provider khác nhau, ví dụ OVS driver và các SDN Controller driver. Thông qua "Common driver API", các driver khác nhau có thể cung cấp các triển khai khác nhau để render nên service chain path. 
```sh
 Port Chain Plugin With Different Types of Drivers
+-----------------------------------------------------------------+
|  +-----------------------------------------------------------+  |
|  |                        Port Chain API                     |  |
|  +-----------------------------------------------------------+  |
|  |                        Port Chain Database                |  |
|  +-----------------------------------------------------------+  |
|  |                        Driver Manager                     |  |
|  +-----------------------------------------------------------+  |
|  |                        Common Driver API                  |  |
|  +-----------------------------------------------------------+  |
|                                   |                             |
|  +------------+------------------------+---------------------+  |
|  | OVS Driver |   Controller Driver1   |  Controller Driver2 |  |
|  +------------+------------------------+---------------------+  |
+-------|------------------|-------------------------|------------+
        |                  |                         |
   +-----------+   +-----------------+      +-----------------+
   | OVS Agent |   | SDN Controller1 |      | SDN Controller2 |
   +-----------+   +-----------------+      +-----------------+
```

Hình thứ hai dưới đây mô tả kiến trúc triển khai với OVS driver. Hình vẽ mô tả các thành phần trong Neutron Server và compute nodes để hỗ trợ các tính năng của Neutron Based SFC. OVS driver và OVS agent được mở rộng để hỗ trợ tính năng service chain. OVS Driver sẽ giao tiếp với mỗi OVS agent thông qua lời gọi RPC để lập trình nên bảng chuyển tiếp OVS phù hợp, do đó traffic flow của một tenant có thể được điều khiển thông qua trình tự Neutron port mà người dùng định nghĩa để đảm bảo thực hiện các thao tác mong muốn với dịch vụ. Giao tiếp giữa OVS Agent và OVS thông qua giao thức OVSDB hoặc OpenFlow.

```sh
 Port Chain Plugin With OVS Driver
+-------------------------------+
|  +-------------------------+  |
|  |    Port Chain API       |  |
|  +-------------------------+  |
|  |    Port Chain Database  |  |
|  +-------------------------+  |
|  |    Driver Manager       |  |
|  +-------------------------+  |
|  |    Common Driver API    |  |
|  +-------------------------+  |
|              |                |
|  +-------------------------+  |
|  |        OVS Driver       |  |
|  +-------------------------+  |
+-------|----------------|------+
        |                |
   +-----------+   +-----------+
   | OVS Agent |   | OVS Agent |
   +-----------+   +-----------+
```

## Port chain creation workflow
Xem xét ví dụ sử dụng neutron CLI commands để tạo port-chain bao gồm hai service VM là vm1 và vm2. User có thể là admin hoặc tenant.

Traffic flow đi vào port chain sẽ đi từ địa chỉ nguồn __22.1.20.1 TCP port 23__ tới địa chỉ IP đích __171.4.5.6 TCP port 100__. Luồng dữ liệu này cần xử lý bởi SF1 trên vm1 (xác định bởi Neutron port-pair [p1,p2]), SF2 chạy trên vm2 (xác định bởi Neutron port-pair [p3,p4]) và SF3 chạy trên vm3 (xác định bởi Neutron port-pair [p5,p6]).

Mạng __net1__ phải có sẵn trước khi tạo neutron port sử dụng neutron API. SFC traffic có thể vận chuyển qua bất kì giao thức transport nào của mạng __net1__ (ethernet, VXLAN, GRE,...). Bởi SFc traffic sẽ được mang theo thông qua các kênh chuyển tải đã thiết lập sẵn giữa các vSwitches. Thông thường, Open vSwitch triển khai đường hầm vận chuyển các gói tin đi qua VLXAN tunnel. 

### Bước 1: Tạo các neutron port và boot VM gắn vào các port đó
- Tạo các neutron port trên __net1__:
```sh
neutron port-create --name p1 net1
neutron port-create --name p2 net1
neutron port-create --name p3 net1
neutron port-create --name p4 net1
neutron port-create --name p5 net1
neutron port-create --name p6 net1
```
- Boot vm1 từ nova kết nối với 2 port p1 và p2:
```sh
nova boot --image cirros --nic port-id=p1-id --nic port-id=p2-id vm1 --flavor m1.tiny
```
- Boot vm2 từ nova kết nối với 2 port p3 và p4:
```sh
nova boot --image cirros --nic port-id=p3-id --nic port-id=p4-id vm2 --flavor m1.tiny
```
- Boot vm3 từ nova kết nối với 2 port p5 và p6:
```sh
nova boot --image cirros --nic port-id=p5-id --nic port-id=p6-id vm3 --flavor m1.tiny
```

### Bước 2: Tạo flow classifier
Tạo ra classifier để lọc các gói tin với địa chỉ nguồn __22.1.20.1__, địa chỉ đích __171.4.5.6__ với kết nối TCP, port nguồn 23, port đích 100:
```sh
neutron flow-classifier-create \
 --ethertype IPv4 \
 --source-ip-prefix 22.1.20.1/32 \
 --destination-ip-prefix 172.4.5.6/32 \
 --protocol tcp \
 --source-port 23:23 \
 --destination-port 100:100 FC1
```

### Bước 3: Tạo các port pair
Tạo các port-pair tương ứng với các service VM:
```sh
neutron port-pair-create \
       --ingress=p1 \
       --egress=p2 PP1

neutron port-pair-create \
       --ingress=p3 \
       --egress=p4 PP2

neutron port-pair-create \
       --ingress=p5 \
       --egress=p6 PP3
```

### Bước 4: Tạo các port pair group
```sh
neutron port-pair-group-create \
       --port-pair PP1 --port-pair PP2 PG1 \
neutron port-pair-group-create \
       --port-pair PP3 PG2
```

### Bước 5: Tạo port chain
```sh
neutron port-chain-create \
       --port-pair-group PG1 --port-pair-group PG2 --flow-classifier FC1 PC1
```

Như vậy workflow tạo nên một service chain sẽ như sau:
```sh
PortChainAPIParsingAndValidation: create_port_chain
               |
               V
PortChainPlugin: create_port_chain
               |
               V
PortChainDbPlugin: create_port_chain
               |
               V
DriverManager: create_port_chain
               |
               V
portchain.drivers.OVSDriver: create_port_chain
```

## SFC Encapsulation

### Khái niệm
SFC Encapsulation cung cấp các thông tin về định danh của Service Function Path, và sử dụng bởi SFC-aware functions, ví dụ như Service Function Forwarder và SFC-aware Service Function. SFC encapsulation không phải sử dụng cho việc chuyển tiếp các gói tin. Bên cạnh định danh của Service Function Path (SFP id) thì SFC Encapsulation cũng mang theo metadata bao gồm thông tin về data-plance. Metadata đóng vai trò quan trọng trong SFC encapsulation.

SFC Encapsulation triển khai trong networking sfc hiện tại khi làm việc với Open vSwitch agent chỉ hỗ trợ MPLS, tương lai sẽ hỗ trợ NSH encapsulation. MPLS không thực sự là một giải pháp cho SFC encapsulation bởi nó không bao đóng gói tin một cách hoàn toàn, không mang theo metadata mà chỉ sử dụng để định danh cho SFP.

### Sử dụng
Để tạo port-chain với các port-pairs sử dụng tương quan MPLS bằng việc thiết lập tham số __correlation__ của port-pair hay service function bằng __mpls__:
```sh
service_function_parameters: {correlation: 'mpls'}
```
Nhãn MPLS thực chất được chèn vào giữa Ethernet header và giao thức lớp 3. Mặc định, port-chain luôn luôn thiết lập tham số __correlation__ bằng __mpls__:
```sh
chain_parameters: {correlation: 'mpls'}
``` 






