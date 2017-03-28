# Triển khai Tacker VNF Forwarding Graph

## Đặt vấn đề
- NFV được trông đợi vào khả năng điều phối và quản lý lưu lượng đi qua các VNFs, hay còn gọi là Service Function Chaining (SFC). Người dùng trong hệ thống NFV không chỉ muốn tạo ra VNFs, mà còn muốn định nghĩa nên SFCs để điều khiển lưu lượng đi qua các VNFs. Theo định nghĩa về khối chức năng MANO (Management and Orchestration) của ETSI, SFC là một đồ hình gồm các VNFs kết nối với nhau một cách logic bởi các Network Forwarding Paths(NFPs) - đường đi của lưu lượng mạng đi qua xuyên suốt đồ hình mạng ấy

- Mục đích hướng tới ở đây là làm thế nào định nghĩa một đồ hình mạng với cấu trúc mang tính logic, trừu tượng có khả năng điều phối được, đồng thời đảm bảo có thể render đồ hình logic đó xuống hệ thống overlay networks. Bước tiếp theo là khả năng phân loại lưu lượng mạng của tenant để lưu chuyển chúng qua SFC. Khái niệm VNFFG trong tacker là sự kết hợp của VNFs, SFC và việc phân loại lưu lượng để cân nhắc xem có áp dụng SFC cho lưu lượng đó hay không.

- Một VNFFG có thể sẽ phức tạp với nhiều đường đi qua đồ thị, tuy nhiên Tacker VNFFG hiện tại mới chỉ hỗ trợ tạo SFCs với một đường đi duy nhất. VNFFG giải quyết vấn đề này bằng việc tạo ra một đồ hình đa đường bằng nhiều SFCs đơn đường khác nhau. Tính năng này có thể coi là giải pháp "tối ưu hóa chuỗi chức năng mạng" (chain optimization) bằng việc thu thập thông tin về tất cả các đường đi đã định nghĩa trong đồ hình mạng. Đây cũng là một phần chủ đạo của vấn đề điều phối VNFFG.

- VNF cũng có thể tái phân loại lưu lượng trong đồ hình. Ví dụ như với một VNF như L7 Deep Packet Inspection (DPI), nó sẽ xác định từ payload của packet và có thể đưa ra quyết định chuyển hướng gói tin đi qua đồ hình mạng.

## Đề xuất thay đổi
- VNFFG tab được bổ sung trên tacker-horizon cho phép user tạo ra đồ hình mạng từ các VNFs có sẵn, cũng như sub-tab Classification để khai báo classifier. Các tham số này có hai cách để nhập: sử dụng drop down menu để lựa chọn các thành phần hoặc định nghĩa trước trong VNFFG Descriptor chuẩn TOSCA template (các network function trong VNFFG và kết nối giữa chúng trong VNFFG).

- Driver tạo 'vnffg' bao gồm hai hướng triển khai: neutron networking-sfc hoặc opendaylight sfc. Trong đó driver hỗ trợ cho VNFFG plugin là networking-sfc driver. OpenDaylight driver tương tác với OpenDaylight SFC là triển khai riêng áp dụng cho OPNFV trên các phiên bản Brahmaputra và Colorado, trong đó tacker sẽ tương tác trực tiếp với SFC provider là OpenDaylight nhờ OpenDaylight driver mà không thông qua networking-sfc. Còn với triển khai chính thức, networking-sfc driver sẽ tương tác với port-chain plugin của project networking-sfc. Networking SFC sau đó sẽ tương tác với SFC provider với hai tùy chọn: Open vSwitch Agent hoặc SDN Controller như OpenDaylight (triển khai này không cho các SFC provider tương tác trực tiếp với tacker mà buộc phải thông qua networking-sfc).
```sh
+---------------------------------------------+
|              Client Application             |
+-----------+---------------------+-----------+
            | Tacker VNFFG API    | Tacker VNFM API
+-----------|---------------------|-----------+
|           v                     v           |
|  +-----------------+    +----------------+  |
|  |      Tacker     |    |    Tacker      |  |
|  | NFVO Extension  |<-->| VNFM Extension |  |
|  |   Plugin/DB     |    |   Plugin       |  |
|  +--------+--------+    +----------------+  |
|           |                                 |
|         +==========================+        |
|         |     networking-sfc       |        |
|         |     Port Chain Driver    |        |
|         +==========================+        |
| Tacker Server        |                      |
+----------------------|----------------------+
                       | Port Chain API
+----------------------|----------------------+
| Neutron Server       v                      |
|            +-------------------+            |
|            | networking-sfc    |            |
|            | Port Chain Plugin |            |
|            +-------------------+            |
+---------------------------------------------+
```

## Tacker với networking-sfc
   - Theo mô hình ở trên có thể thấy project Tacker triển khai SFC driver để giao tiếp với port-chain API của project Neutron networking-sfc. Tacker API được cung cấp hỗ trợ các thao tác quản lý các VNFs và tạo VNFFGs.
   - Port-chain API có thể sử dụng để tạo nên một hay nhiều Network Forwarding Paths trên đồ hình mạng.    
     
     - __Vấn đề đặt ra__
       - Tacker VNFFG và Neutron networking-sfc hoạt động ở hai cấp độ trừu tượng khác nhau. Nếu như Neutron networking-sfc tạo nên một SFC bằng việc kết nối một danh sách các neutron ports thì Tacker VNFFG thao tác với các service instance hoặc thậm chí là với service type (ví dụ load balancer, firewall,...).
       - Để có thể render ra các Networking Forwarding Paths của VNFFG, networking-sfc phải cung cấp khả năng tạo ra SFCs và Classifiers cho Tacker.
     
     - __Giải quyết__
       - Networking-sfc driver của Tacker sẽ map từ mô tả mức trừu tượng ở VNFFG sang Neutron port, hay port chain. 
       - NFVO plugin sẽ gửi các yêu cầu thực hiện CRUD (Create, Read, Update, Delete) VNFFG tới networking-sfc driver. Driver này sẽ map các thao tác vận hành đó thành các thao tác CRUD với port-chain và gọi port-chain API của networking SFC port chain plugin.
       - NFVO plugin cũng giao tiếp với Tacker VNF Manager để thu thập thông tin về các VNF instance và ingress/exgress interfaces nếu VNFFG chỉ mô tả kiểu VNF.
       - Khả năng mở rộng là tính năng nâng cao trong tương lai của Tacker, tuy nhiên VNF scaling hiện tại đã được hỗ trợ bởi networking-sfc. Nếu VNFM trả lại nhiều VNF instances thì NFVO sẽ lựa chọn một VNF instance để tạo port-pair-group với một port-pair. Khi NFVO plugin hỗ trợ scaling, nó có thể   tạo ra một port-pair-group bao gồm tất cả các VNF instances mà VNF Manager trả vể.

       Networking-sfc driver có những tính năng sau:

       - Map định nghĩa của VNFFG chain sang định nghĩa của một port-chain trong networking-sfc.
       - Driver sẽ định dạng lại các thao tác CRUD và gọi port-chain API
       - Nếu một VNFFG được chỉ định tính chất đối xứng (symmetrical), driver sẽ thiết lập giá trị __symmetric=true__ trong thuộc tính chain parameters.
       - Map VNFFG classifier thành flow-classifier cho port-chain.
       - Driver mặc định của Tacker VNFFG là __networking-sfc__

       Tacker NFVO workflow để định nghĩa nên VNFFG như sau:
       ```sh
       tacker vnffg-create --name myvnffg --vnfm_mapping VNF1:testVNF2,VNF2:testVNF1
                      --symmetrical True
       ```
       Networking SFC workflow tương ứng sẽ như sau:
       ```sh
       neutron port-pair-create --ingress <port-id> --egress <port-id>
       
       neutron port-pair-group-create --port-pairs <port-pair-id>
       
       neutron flow-classifier-create --protocol tcp --destination-port 80:80
       
       neutron port-chain-create --port-pair-group <port-pair-group-id>
                                 --flow-classifier <classifier-id> <name>
       ```
       Các thao tác tương ứng từ Tacker VNFFG APIs tới Neutron networking-sfc client sẽ như sau:
       ```sh
       +---------------------------------------------------------------------+
       |   Tacker VNFFG API          |   networking-sfc client CLI           |
       +-----------------------------+---------------------------------------+
       |                             |                                       |
       |       vnffg-create          |   neutron port-pair-create            |
       |                             |       --ingress [Neutron port]        |
       |                             |       --egress [Neutron port]         |
       |                             |                                       |
       |                             |   neutron port-pair-group-create      |
       |                             |       --port-pairs [port pair id]     |
       |                             |                                       |
       |                             |   neutron flow-classifier-create      |
       |                             |       [parameters]                    |
       |                             |                                       |
       |                             |   neutron port-chain-create           |
       |                             |       --port-pair-group <id>          |
       |                             |       --flow-classifier <fc-id>       |
       |                             |                                       |
       +-----------------------------+---------------------------------------+
       ```

## Data model
...to be continued

## Tham khảo
- [Tacker VNFFG specs](https://github.com/openstack/tacker-specs/blob/master/specs/newton/tacker-vnffg.rst)
- [Tacker & networking-sfc integration](https://github.com/openstack/tacker-specs/blob/master/specs/newton/tacker-networking-sfc.rst)