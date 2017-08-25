# Ceph Architecture and Components
Trong bài viết này, chúng ta sẽ đi tìm hiểu về kiến trúc cũng như các thành phần trong hệ thống lưu trữ Ceph

 - Ceph storage architecture
 - Ceph RADOS
 - Ceph Object Storage Device (OSD)
 - Ceph monitors (MON)
 - librados
 - The Ceph block storage
 - Ceph Object Gateway

# Ceph storage architecture
Ceph là giải pháp mã nguồn mở để xây dựng hạ tâng lưu trữ phân tán, ổn định, độ tin cậy và hiệu năng cao cũng như dễ dàng mở rộng. Với hệ thống lưu trữ được điều khiển bằng phần mềm, Ceph cung cấp giải pháp lưu trữ theo object, block và file trong một nền tảng đơn nhất. 

Dưới đây là sơ đồ mô tả kiến trúc  và các thành phần chính trong hệ thống Ceph.

![enter image description here](http://i.imgur.com/yIvDY8M.png)

**Reliable Autonomic Distributed Object Store (RADOS)** là nền tảng của Ceph storage cluster. Mọi dữ liệu trong Ceph đều được lưu dưới dạng các **object** và RADOS sẽ chịu trách nhiệm lưu trữ các **object** này, bất kể kiểu dữ liệu của chúng. RADOS layer đảm bảo dữ liệu sẽ luôn ở trong trạng thái nhất quán và tin cậy. Để làm được điều này, nó thực hiện replicate dữ liệu, phát hiện lỗi và hồi phục cũng như data migration và rebalancing giữa các node trong cluster. 

**Object Storage Device (OSD)**: đây là thành phần duy nhất trong Ceph cluster có nhiệm vụ store data và retrieve khi có một hành động write hoặc read data. Thông thường, một OSD deamon được gắn với một ổ đĩa vật lí.

**Ceph monitors (MONs)** theo dõi trạng thái của toàn bộ cluster bằng cách duy trì một **cluster map** bao gồm OSD, MON, PG, CRUSH và MDS **map**. Tất cả các node trong cluster gửi thông tin về cho monitor node khi có thay đổi trạng thái. Một monitor duy trì thông tin của mỗi thành phần trong một **map** riêng biệt. Monitor node không có nhiệm vụ lưu trữ dữ liệu, đây là nhiệm vụ của OSD.

**librados**: là một thư viện cung cấp các API truy cập tới RADOS hỗ trợ các ngôn ngữ PHP, Ruby, Java, C và C++. Nó cung cấp một native interface tới Ceph storage cluster, RADOS và các services khác như RBD, RGW VÀ CephFS. 

**Ceph Block Device** hay còn gọi là **RADOS block device (RBD)** cung cấp giai pháp lưu trữ block storage, có khả năng mapped, formated và mounted như là một ổ đĩa tới server. RDB được trang bị các tính năng lưu trữ doanh nghiệp như **thin provisioning** và **snapshot**.

**Ceph Object Gateway** hay còn gọi là **RADOS gateway (RGW)** cung cấp RESTful API interface, tương thích với Amazon S3 (Simple Storage Service) và Openstack Object Storage API (Swift). RGW cũng hỗ trợ dịch vụ xác thực Openstack Keystone.

# Ceph RADOS
**RADOS (Reliable Autonomic Distributed Object Store)** là thành phần trung tâm trong hệ thống Ceph storage, cũng được gọi như **Ceph storage cluster**. RADOS cung cấp tất cả các tính năng quan trọng trong Ceph, bao gồm distributed object store, high availability, reliablity, no single point of failure, self-healing, self-managing... Các phương thức truy cập dữ liệu của Ceph như RBD, CephFS, RADOSGW và librados , tất cả đều nằm trên top của RADOS layer.

Khi Ceph cluster nhận được yêu cầu **write** từ phía client, thuật toán CRUSH tính toán và quyết định vị trí dữ liệu nên lưu trữ trong cluster. Thông tin này sau đó được truyền tới RADOS layer để xử lí tiếp. Dựa vào các ruleset của CRUSH, RADOS phân phối dữ liệu tới tất cả các node trong cluster dưới định dạng các **object**. Cuối cùng, các **object** này được lưu trong các OSD.

RADOS đảm bảo dữ liệu luôn tin cậy. Tại cùng một thời điểm, RADOS replicate các object, tạo ra các bản copy và lưu chúng trong các failure zone khác nhau, tránh lưu chúng trên cùng một failure zone. Ngoài lưu trữ và replicate các object trên toàn bộ cluster, RADOS cũng đảm bảo các object luôn trong trạng thái nhất quán. Trong trường hợp các object không nhất quán, hành động **recover** được thực hiện trên các bản sao object còn lại. Hoạt động này được thực hiện tự động và trong suốt với người dùng, cung cấp khả năng self-managing và self-healing cho hệ thống Ceph. 

## Ceph Object Storage Device
Ceph OSD là một trong những thành phần quan trọng nhất trong Ceph storage cluster. Nó lưu trữ **actual data** trong các ổ đĩa vật lí trên mỗi node của cluster dưới dạng các object. Phần lớn các công việc bên trọng Cẹph cluster được thực hiện bới Ceph OSD daemon. Chúng ta sẽ đi thảo luận về vai trò và trách nhiệm của Ceph OSD daemon.

Ceph OSD lưu trữ dữ liệu người dùng và trả về cùng một dữ liệu đó khi có yêu cầu của người dùng. Một Ceph cluster bao gồm nhiều OSDs.  Bất kì hành động **read** hoặc **write **, client đều yêu cầu một **cluster map** từ monitors, và sau đó client sẽ trực tiếp tương tác với các OSD trong các hành động **read/write** mà không cần sự can thiệp của monitor. Điều này khiến qúa trình trao đổi dữ liệu diễn ra nhanh hơn, client có thế lưu trực tiếp tới OSD mà không cần thêm bất kì lớp xử lí dữ liệu nào.

Ceph cung cấp độ tin cậy cao bằng cách replicate mỗi object trên các node trong cluster, làm chúng có tính sẵn sàng cao và khả năng chống lỗi. Mỗi object trong OSD có một bản **primary copy** và một số **secondary copy** được phân bố đều trên các OSD. Mỗi OSD vừa đóng vai trò là primary OSD của một số object này, vừa có thể là secondary OSD của một số object khác. Bình thường các secondary OSD nằm dưới sự điều khiển của primary OSD, tuy nhiên chúng vẫn có khả năng trở thành primary OSD.

Trong trường hợp disk failure, Ceph OSD daemon sẽ so sánh với các OSD khác để thực hiện hành động **recovery**. Trong thời gian này, secondary OSD giữ bản sao của object bị lỗi sẽ được đẩy lên làm primary, đồng thời các bản sao mới sẽ được tạo trong qúa trình recover. Qúa trình này là trong suốt với người dùng.

## Ceph monitors
Như tên gọi, Ceph monitor chịu trách nhiệm giám sát tình trạng của toàn bộ cluster. Chúng lưu trữ các thông tin quan trọng của cluster, trạng thái của các node và thông tin cấu hình cluster. Các thông tin này được lưu trong **cluster map**, bao gồm monitor, OSD, PG, CRUSH và MDS **map**. 

 - **Monitor map:** Lưu thông tin end-to-end của node monitor như Ceph cluster ID, monitor hostname, địa chỉ IP và port, thời gian cuối cùng có sự thay đổi. Để check monitor map, thực hiện command sau:
	```# ceph mon dump
	```
 - **OSD map:** Lưu các thông tin chung như cluster ID, thời điểm OSD map được tạo và last-changed. Các thông tin liên quan tới **pool** như poll name, poll ID, type, placement group. Nó cũng lưu thông tin của OSD như count, state, weight, last clean interval... Để check OSD map, thực hiện command sau:
    ```# ceph osd dump
	```
 - **PG map:** Lưu PG version, time stamp, last OSD map epoch, full ratio và chi tiết của mỗi placement group như PG id, state của PG... Để check PG map, thực hiện command sau:
	```# ceph pg dump
	```
 - **CRUSH map:** Lưu thông tin về các storage devices, failure domain ( host, rack, row, room, device) và các quy tắc khi lưu trữ dữ liệu .Để check CRUSH  map, thực hiện command sau:
	```# ceph osd crush dump
	```
 - **MDS map:** Lưu thông tin về MDS map epoch, map creation và modification time, data and metadata pool ID, cluster MDS count, MDS state .Để check MDS map, thực hiện command sau:
    ```# ceph mds dump
	```
	