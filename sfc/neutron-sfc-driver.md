# Service Function Chaining Driver với OpenDaylight

## Đặt vấn đề
- OpenStack SFC project (networking-sfc) cung cấp các APIs chung để có thể triển khai kết hợp với nhiều networking service provider khác nhau hỗ trợ Service Function Chaining (SFC). Các APIs này cung cáp cách thức để tích hợp OpenStack SFC với bất kì SFC providers nào. Trong đó OpenDaylight SFC project là một project đã trưởng thành để cung cấp SFC, nhưng chưa có một mô hình tích hợp tiêu chuẩn nào để sử dụng project này như một SFC provider cho Neutron networking-sfc.

- Project Tacker gần đây đã phát hành với khả năng đáp ứng một số NFV usecase (trong đó có SFC) sử dụng OpenStack platform. Việc cung cấp một mô hình tích hợp đầy đủ và chuẩn mực giữa OpenStack và OpenDaylight đối với SFC usecase sẽ giúp cho người dùng NFV tận dụng dụng được giải pháp kết hợp OpenStack, Tacker và OpenDaylight. Phiên bản thử nghiệm của giải pháp tích hợp này đã được triển khai bởi Tim Rozet, nhưng trong bản triển khai này, Tacker giao tiếp trực tiếp với OpenDaylight SFC và classifier providers mà không thông qua OpenStack SFC APIs (networking-sfc). Triển khai này đòi hỏi Tacker phải viết thêm các driver phụ để tương tác với OpenDaylight SFC và NetVirt để tạo SFC và classifier, không phải chức năng chính của một NFVOchestrator và VNFManager.

## Đề xuất thay đổi
Networking-sfc driver cho OpenDaylight Controller được triển khai trong project networking-odl và gửi lời gọi networking-sfc API tới OpenDaylight.

## Thiết kế chi tiết

Để đảm bảo tích hợp đầy đủ giữa OpenStack SFC và OpenDaylight, yêu cầu đặt ra là phải có SFC driver cho OpenDaylight. OpenDaylight SFC driver được coi như một lớp đệm giữa OpenStack và OpenDaylight đảm nhận hai tác vụ sau:
- Chuyển đổi OpenStack SFC classifier API sang OpenDaylight SFC classifier yang models.
- Chuyển đổi OpenStack SFC API sang OpenDaylight Neutron Northbound SFC models

SFC providers (NetVirt, Group-based Policy, SFC) trong OpenDaylight lắng nghe các OpenDaylight Neutron Northbound SFC models và chuyển đổi chúng sang yang model của classifier/sfc. Sơ đồ dưới đây mô tả quá trình tích hợp ở mức cao giữa OpenStack và OpenDaylight SFC provider:
```sh
            +---------------------------------------------+
            | OpenStack Network Server (networking-sfc)   |
            |            +-------------------+            |
            |            | networking-odl    |            |
            |            |   SFC Driver      |            |
            |            +-------------------+            |
            +----------------------|----------------------+
                                   | REST Communication
                                   |
                         -----------------------
OpenDaylight Controller |                       |
+-----------------------|-----------------------|---------------+
|            +----------v----+              +---v---+           |
| Neutron    | SFC Classifier|              |SFC    | Neutron   |
| Northbound |    Models     |              |Models | Northbound|
| Project    +---------------+              +-------+ Project   |
|               /        \                      |               |
|             /           \                     |               |
|           /               \                   |               |
|     +-----V--+        +---V----+          +---V---+           |
|     |Net-Virt|  ...   |   GBP  |          |  SFC  |  ...      |
|     +---------+       +--------+          +-------+           |
+-----------|----------------|------------------|---------------+
            |                |                  |
            |                |                  |
+-----------V----------------V------------------V---------------+
|                     Network/OVS                               |
|                                                               |
+---------------------------------------------------------------+
```

OpenStack SFC APIs là port-pair based API trong khi đó OpenDaylight SFC API dựa trên IETF SFC yang models, do đó có những thao tác chuyển đổi yêu cầu cải thiện API, đảm bảo khả năng tận dụng được các backends providers khác nhau.  

## Tham khảo
- [Neutron SFC driver](https://docs.openstack.org/developer/networking-odl/specs/sfc-driver.html)
