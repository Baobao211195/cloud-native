### Not    
+ K8S cung cap cho cac pod 1 IP va 1 single DNS name va set vao Pod va co the loadbalance cho chung 
## Service without selector.(use case) 
 + có 1 db bên ngoài trên môi trường prod. nhưng trên môi trường test thì là db của bạn.
 + Bạn muốn muốn từ service của bạn đi tới service khác ở 1 namespace khác hoặc ở cluster khác.
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```
- Vì service không có selector nên Endpoint object tưong ứng không được tạo tự động. Vậy phải map = tay service tới địa chỉ và port trong mạng như sau:
```
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```  
 ### Virtual IPs và và service proxies.  
- Tất cả các node trong cluster thì đều có 1 proxy gọi là ***kube-proxy***,  
nó có nhiệm vụ tạo ra một định dạng virtural ip dành cho các kiểu service khác nhau ngoại trừ kiểu service là ***ExternalName***.  
## Chế độ user space proxy.  

- kube-proxy theo dõi k8s control plane để thêm hoặc remove của service và các Endpoint.  
 Đối với mỗi Service nó mở 1 port(lựa chọn ngẫu nhiên) trên local node.  
 Bất kỳ các kết nối tới ***"proxy port"*** này để bị chặn lại tới một trong các pods của Service backend.  
 Kube-proxy nhận thuộc tính ***SessionAffinity*** dành cho việc thiết lập Service vào 1 account khi đó sẽ quyết định backend pod nào được sử dụng.  
- Cuối cùng với chế độ này, sẽ thiết lập 1 ***iptables rules*** để bắt mọi traffic tới các ***clusterIP***(ảo) của Service và ***port***.  
Những rules này sẽ redirect tới cái proxy port mà được proxies các pods.  
Mặc định thì khi ở chế độ này thì kube-proxy sẽ chọn 1 pod theo thuật toán round-robin.  

## Chế độ iptables.  
- Tương tự vs chế độ user, nhưng **iptable** sẽ được cài ở **Service**  
và cũng cài ở mỗi ***Enpoint*** object. Tại Service thì sẽ capture và redirect tới một trong các backend pods đã được thiết lập.  
Nhưng ở trên Endpoint thì sẽ lựa chọn 1 backend pod.  
- Mặc định sẽ lựa chọn ngẫu nhiên 1 backend pod.  
- Sử dụng chế độ này để handle traffic thì sẽ kéo việc overload hệ thống xuống thấp.  
vì lúc này traffic được handle bởi linux netfilter mà không phải cần chuyển giữ userspace và kernel space.  
Với cách này thì đảm bảo tin cậy hơn.  
- Giả sử trong trường hợp mà pod đầu tiên được lựa chọn ko phản hồi,  
kết nối thất bại. kube-proxy sẽ detect kết nối tới pod đầu tiên đó bị thất bại  
và tự động retry vs một backend pod khác.  

## Chế độ  IPVS proxy.(stable trong k8s ver 1.111[stable]). (tốt hơn 2 thằng trên)  
- IPVS cung cấp rất nhiều lựa chọn cho việc cân bằng tải tới các Backend Pods.
  + rr: round robin
  + lc: least connection
  + dh: destination hasing
  + sh: source hasing
  + sed:  shortest expected delay.
  + nq: never queu.
- Khi run kube-proxy ở chế độ này thì phải kiểm tra IPVS kernel đã có chưa , trước khi run ở chế độ này.  
 nếu chưa có , kube proxy sử dụng chế độ thứ 2 là ***iptable***.  
  
 # Setting nhiều port trên một service.  
  

Có thể thiết lập nhiều port cho một service như ví dụ dưới đây.  
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```  
# Lựa chọn Ip addresses cho riêng bạn.  
- Bạn có thể chỉ định cụ thể 1 ip address cho 1 ***Service*** bằng cách thiết lập giá trị cho ***spec.clusterIp***.  
Nếu bạn muộn chọn Ip trong 1 khoảng thì dùng ***service-cluster-ip-range***.  

# Traffic Policies.  
## External traffic policy

- Bạn có thể thiết lập giá trị ***spec.externalTrafficPolicy*** để thiết lập việc controll traffic như thế nào từ các   
resources từ bên ngoài đã được định tuyến vào bên trong , nếu = ***Cluster*** thì có nghĩa là định tuyết từ tất cả các  
traffice từ bên ngoài tới các endpoint và nếu = ***Local*** thì sẽ định tuyến chỉ tới các endpoint ở bên trong. Nếu   
không có bất cứ endpoint nào ở bên trong thì kube-proxy sẽ không forward tới bất kỳ service nào liên quan.  

## Internal traffic policy.

- Bạn có thể thiết lập giá trị ***spec.internalTrafficPolicy*** để thiết lập việc controll traffic như thế nào từ các   
resources ở bên trong  đã được định tuyến , nếu = ***Cluster*** thì có nghĩa là định tuyết từ tất cả các  
traffice từ bên trong tới các endpoint và nếu = ***Local*** thì sẽ định tuyến chỉ tới các endpoint ở bên trong. Nếu   
không có bất cứ endpoint nào ở bên trong thì kube-proxy sẽ không forward tới bất kỳ service nào liên quan.  

# Discovering services .  
K8s có 2 cách để tìm một service đó là dùng biến môi trường và DNS.  
## Biến môi trường.  
- Khi một pod được run trên node, ***kubelet*** sẽ thực hiện thiết lập các biến môi trường cho các service được active.  
Nó support các ***Docker link compatible*** và các biến đơn giản như ***{SVCNAME}_SERVICE_HOST and {SVCNAME}_SERVICE_PORT***.  
Trong đó các service name được viết hoa, các dấu ***-*** được chuyển thành dấu gạch dưới ***_***. 
Dưới đây là 1 ví dụ về service là redis-master được expose tại port 6739 và được chỉ định ip ***10.0.0.11*** thì sẽ sinh ra những biến mt như sau:  
```  
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379

REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```  
---
**NOTE**
Nếu dùng cách này để discovering service.
Khi một pods cần access và 1 service thì phải tạo service trước pods biết đến sự tồn tại của chúng.   
Nếu không các pods sẽ không biết được những biến môi trường nào đang được hoạt động.

---  
## DNS.  
- Nếu dùng DNS thì phải cần sử dụng ***ad on*** là ***coreDNS***. ***coreDNS*** sẽ watch  Kubenete API khi mà các servive mới được tạo ra và  
sẽ thiết lập cho nó 1 tập các DSN records cho mỗi service. Nếu DNS được enable thông qua cluster thì tất cả các pods sẽ được tự động resolve các  
Service bằng các DNS name.  
Ví dụ có một ***my-service*** trong 1 name space là ***ns** thì khi này các pods trong name space này sẽ tìm được service theo ***my-service***.  
Còn các pods ở service khác thì có thể tìm kiếm theo ***my-service.ns***.
Thêm nữa k8s còn support việc look up theo cả giao thức như sau ***_http._tcp.my-service.ns***.  

# Headless Services.  
