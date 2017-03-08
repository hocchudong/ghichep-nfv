# OpenStack Neutron networking-sfc

## 1. Giới thiệu
Khái niệm Service Function Chaining về cơ bản gắn với SDN policy-based routing. SFC thường liên quan tới vấn đề về bảo mật mặc dù nó có thể sử dụng cho nhiều mục đích khác.

Về cơ bản, SFC định tuyến các gói tin đi qua một hoặc nhiều chức năng mạng thay vì các định tuyến truyền thống dựa vào địa chỉ IP đích. Các chức năng mạng trong SFC liên kết với nhau về mặt logic như thể có dây cáp vật lý kết nối chúng với nhau vậy. Mục đích là để tạo ra một chuỗi chức năng mạng để đáp ứng nhu cầu của khách hàng khi cung cấp dịch vụ (đảm bảo về bảo mật, chất lượng trải nghiệm, chất lượng dịch vụ,...) 

## 2. Kiến trúc Neutron networking-sfc
Nhắc lại một số kiến thức liên quan OpenStack Networking service (neutron) và OpenStack Compute service, các compute instance kết nối vào một mạng ảo thông qua các port. Điều này cho phép tạo sử dụng mô hình điều khiển lưu lượng đối với SFC sử dụng các port. Việc kết nối các port này trong một chuỗi các port cho phép điều khiển lưu lượng đi qua một hoặc nhiều compute instance đóng vai trò là chức năng mạng (service function - SF).

Một chuỗi các port tương đương khái niệm Service Function Path (SFP) bao gồm:
 - Một tập các port định nghĩa nên trình tự các service function
 - Một tập các flow classifiers (bộ phân loại các luồng) chỉ định các traffic flows đi vào chuỗi

Nếu một SF gắn với một cặp port thì phải chỉ rõ ingress port và exgress port của SF đó. SF cho phép gắn với một port và port đó đóng cả hai vai trò, coi như là một port cho phép lưu lượng đi theo hai chiều vào và ra.

Một chuỗi các port (port chain) được coi là một service chain vô hướng. Một SFC bao gồm hai port chain vô hướng là SFC hai chiều.

Một flow classifier chỉ thuộc về một port chain để tránh việc bối rối khi hệ thống quyết định xem chain nào sẽ xử lý các gói tin. Một port chain có thể gắn với nhiều classifier vì nhiều loại lưu lượng có thể yêu cầu cùng một SFP.

Project networking-sfc là project con của neutron, triển khai port chain plug-in với Open vSwitch driver và SDN Controller drivers ([networking-odl](https://github.com/openstack/networking-odl.git) hoặc [networking-onos](https://github.com/openstack/networking-onos.git)) cho phép tương tác với các SFC provider khác nhau (Open vSwitch agent hoặc SDN Controller như OpenDaylight hoặc ONOS).
Project này cũng cung cập một driver API chung để hỗ trợ các driver khác nhau đó nhằm cung cấp các giải pháp khác nhau triển khai một SFP.

![port-chain-plugin](https://docs.openstack.org/ocata/networking-guide/_images/port-chain-architecture-diagram.png)

## 3. Một số khái niệm
- __Port chain__: một port chain là một tập hợp có trình tự của các port pair groups. Mỗi port pair group là một chặng của port chain. Một port pair group đại diện một tập các service function có chức năng tương đương nhau. Ví dụ một tập hợp các firewall là một port pair group trong đó mỗi port pair gắn với một firewall instance. Một flow classifier sẽ chứng thực cho một flow. Một port chain có thể gắn với nhiều flow classifiers. Flow classifier thực hiện chứng thực một flow bằng thao tác đóng gói (encapsulation), sử dụng nhãn mpls (hiện tại là lựa chọn mặc định) hoặc NSH (network service header - hỗ trợ trong các bản phát hành OpenStack tương lai, hiện tại có thể triển khai với bản vá Open vSwitch không chính thức từ repo [ovs_nsh_patches](https://github.com/yyang13/ovs_nsh_patches.git) và yêu cầu sử dụng kết hợp với SFC provider là OpenDaylight hoặc ONOS, chưa hỗ trợ bởi Open vSwitch agent).

    Việc sử dụng mpls hay NSH định nghĩa trong thuộc tính _chain_parameters_

    Các thuộc tính của port chain:
     - id - port chain id
     - tenant_id - project id
     - name - tên chuỗi
     - description - mô tả chuỗi
     - port_pair_groups - danh sách các port pair group id
     - flow_classifiers - danh sách các flow classifier id
     - chain_parameters - một từ điển (tập hợp các phần tử định dạng _key:value_) các tham số của chuỗi

- __Port pair group__: định nghĩa xem lại ở trên. Một port pair group có thể chứa một hoặc nhiều port pair (đại diện cho một hoặc nhiều service function). Nhiều port pair cho phép cân bằng tải hoặc phân bố lưu lượng xử lý trên một tập các service function instance cùng chức năng.

- __Port pair__: một port pair đại diện cho một service function instance bao gồm ingress port và exgress port (hoặc một port đảm nhận hai chức năng).
    
    Các thuộc tính của port pair:
    - id - Port pair ID
    - tenant_id - Project ID
    - name - tên port pair
    - description - mô tả port pair
    - ingress - Ingress port
    - egress - Egress port
    - service_function_parameters - một từ điển chứa các tham số của service function

    Trong đó thuộc tính _service_function_parameters_ chỉ định một service function có phải thuộc dạng truyền thống (với giá trị thiết lập bằng _none_) hay có hỗ trợ NSH. Nếu giá trị thiết lập bằng _none_ thì service function đó không thể xử lý các gói tin đóng gói NSH và cần phải có một __service function proxy__ để gỡ NSH khi gói tin đưa từ __service function forwarder__ tới service function để xử lý và đóng gói lại NSH cho gói tin đã được __service function__ xử lý trước khi gửi trả lại __service function forwarder__. 

- __Flow classifier__: Tập hợp các thuộc tính định nghĩa nên flow, gồm cả các thuộc tính về nguồn và đích của flow. 

## 4. Tham khảo

- [OpenStack Networking SFC document](https://docs.openstack.org/ocata/networking-guide/config-sfc.html)
