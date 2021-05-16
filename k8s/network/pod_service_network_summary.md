# K8s Network 정리

CRI로 docker를 가정했을 때의 Pod, service network에 대한 간략한 정리


## Pod network

Host network를 공유하지 않는 이상 Pod 간의 networking이 필요함

1. Pod 내에서는 container끼리 network namespace를 공유하며, 외부 통신을 위한 veth 인터페이스가 있음
- pause container를 통해 모든 container들에게 동일 namespace를 제공

2. 한 노드의 Pod끼리는 linux bridge를 통해 서로 연결됨
- linux bridge는 소프트웨어로 구현된 switch
- 각 Pod의 veth와 pair를 이루는 veth가 linux bridge에 추가됨
- linux bridge는 docker가 만듦

3. 다른 노드의 Pod과 통신을 하려면 route 정보가 추가되어 있어야 됨
- 노드A의 Pod1(10.244.1.10)과 노드B의 Pod2(10.244.2.10)
- Pod1에서 Pod2로 패킷을 보내려면 아래와 같은 순서를 거쳐야 됨
  - Pod1의 bridge
  - 노드A의 eth 인터페이스
  - 노드B의 eth 인터페이스
  - Pod2의 bridge
  - Pod2의 veth
- Pod1에서 노드A의 eth0까지는 문제없지만, Pod2가 노드B에 있다는 정보는 노드A에 추가돼야 함
  - `ip route add 10.244.2.10 via eth0` 이런 식으로 해당 route 정보 추가

4. 이러한 작업을 모든 노드의 모든 Pod마다 직접 할 수 없으므로 CNI 플러그인이 이 과정을 자동화해줌
- 디테일한 방법이 다를 수 있는데, 공통으로 해야 하는 작업들을 CNI 규격으로 정의하고, 각 플러그인은 이를 구현
- WeaveNet 같은 경우 각 노드에 agent가 있어서, 이 agent들끼리 목적지 Pod을 찾아서 패킷을 해당 노드로 보내줌
- kubelet에 CNI 플러그인의 binary와 config 경로를 입력하여 Pod이 생성될 때 자동으로 네트워크를 설정해줌
  - config 같은 경우 IP Address Management(IPAM) 정책 등이 있음
  - Pod에 할당할 IP 대역대는 K8s bootstrap 시, calico의 경우 플러그인 설치 시 입력해야함

> 참고사항
K8s bootstrap 시 시작되는 control-plane은 static Pod들로써 host의 IP로 뜸


## Service network

Pod은 언제든 지워질 수 있으므로 Pod 간의 통신에 있어서는 IP/port와 달리 정적인 endpoint가 필요하고, K8s는 이를 Service로 제공

1. Deployment 등을 Service로 다른 Pod에게 노출
- 다른 Pod들은 해당 Deployment의 Pod과 통신할 떄 해당 Service의 endpoint를 이용함
- Service에 할당할 IP 대역대는 K8s bootstrap 시, calico의 경우 플러그인 설치 시 입력해야 함

2. 각 노드에서 kube-proxy가 해당 Service endpoint와 목적지 Pod 주소를 맵핑함
- kube-proxy는 DaemonSet으로 K8s bootstrap 시 모든 node에서 작동함
- 노드에 iptables rule을 내려서 Service-Pod 맵핑을 관리함
- kube-proxy Pod들도 host의 IP로 뜸

3. coredns를 이용하여 각 Pod에서 Service의 이름으로 endpoint를 resolve함
- coredns는 Deployment로 뜨며 모든 Pod의 DNS server로 작동
  - K8s bootstrap 시 kube-dns라는 Service가 생성됨
  - 모든 Pod들의 `/etc/resolve.conf`에는 nameserver의 endpoint로 kube-dns의 endpoint가 지정돼있음
