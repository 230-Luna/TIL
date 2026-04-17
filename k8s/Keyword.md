## CNI(Container Network Interface)

Kubernetes에서 Pod들이 서로 통신할 수 있게 해주는 네트워크 플러그인.

Kubernetes는 Pod 생성만 하고 Pod IP 할당, Pod 간 통신, Node 간 통신은 CNI가 담당한다.

### 종류

- Calico: 많이 쓰는 범용 CNI, NetworkPolicy 강력 지원, BGP 기반
- Cilium: 요즘 핫함. eBPF 기반, 성능 죄강, kube-proxy 대체가능, 고성능
- Flannel: 단순 overlay network, NetworkPolicy 없음, 기능 부족.

### CNI가 하는 일

- Pod IP 할당
- Pod <-> Pod 간 통신
- Node <-> Node 간 통신
- iptables 규칙 설정
- 라우팅 테이블 설정
- NetworkPolicy 적용
  ...등등

## 컨테이너 런타임

컨테이너를 실제로 생성하고 실행하는 프로그램.
Kubernetes는 '이 컨테이너 실행해'라고만 지시하고 실제 실행하는 건 런타임이 담당한다.

### 종류

- containerd
- CRI-O
- Docker(예전 방식)

### 컨테이너 런타임이 하는 일

- 이미지 pull
- 컨테이너 생성
- 컨테이너 시작/중지
- 컨테이너 삭제
- 컨테이너 상태 관리

### CRI-O

## kubeadm

클러스터 생성 도구
클러스터 설치용 1회성 도구임.

### kubeadm이 하는 일

- control plane 구성
- 인증서 생성
- kube-apiserver 생성
- etcd 생성
- scheduler 생성
- worker join 설정

## kubelet

각 노드에서 항상 실행되는 에이전트
노드에서 Pod 관리하는 상시 데몬

### kubeadm이 하는 일

1. Pod 생성 요청 받음
2. container runtime 호출(containerd)
3. 컨테이너 실행
4. 상태체크
5. 죽으면 재시작

## BGF(Border Gateway Protocol)

네트워크 경로를 서로 알려주는 프로토콜
-> '이 Pod IP는 이 노드에 있어'라고 노드끼리 공유

## eBPF(extended Berkeley Packet Filter)

리눅스 커널 안에서 직접 네트워크 처리하는 기술

- 기존 Kubernetes 네트워크 흐름: packet > iptables > kube-proxy > routing > pod
- eBPF 사용: eBPF(커널에서 바로 처리) > pod
