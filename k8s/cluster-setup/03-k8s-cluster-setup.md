# 쿠버네티스 클러스터 구축하기

## 목차

- [사전 준비](#사전-준비)
- [공통 설정(전체 노드)](#공통-설정-전체-노드)
  - [스왑 비활성화](#스왑-비활성화)
  - [커널 모듈 및 파라미터 설정](#커널-모듈-및-파라미터-설정)
  - [컨테이너 런타임 설치(containerd)](#컨테이너-런타임-설치-containerd)
  - [kubeadm, kubelet, kubectl 설치](#kubeadm-kubelet-kubectl-설치)
- [Control Plane 초기화](#Control-Plane-초기화)
- [kubectl 사용 설정](#kubectl-사용-설정)
- [CNI 설치 - Calico](#CNI-설치--Calico)
- [Worker Node 클러스터 등록](#Worker-Node-클러스터-등록)
- [클러스터 상태 확인](#클러스터-상태-확인)

---

## 사전 준비

[01-virtualbox-setup-manual.md](./01-virtualbox-setup-manual.md) 또는
[02-virtualbox-setup-vagrant.md](./02-virtualbox-setup-vagrant.md)를 참고하여 VM 4대가 실행 중인지 확인한다.

| 노드          | hostname | IP           |
| ------------- | -------- | ------------ |
| Control Plane | cp       | 192.168.56.x |
| Worker Node 1 | worker1  | 192.168.56.x |
| Worker Node 2 | worker2  | 192.168.56.x |
| Worker Node 3 | worker3  | 192.168.56.x |

MobaXterm으로 각 노드에 SSH 접속한 뒤 아래 과정을 진행한다.

## 공통 설정(전체 노드)

공통 설정 부분은 Control Plane 과 Worker1~3 모든 노드에서 진행한다.

### 스왑 비활성화

Kubernetes는 swap 켜진 상태를 허용하지 않음.
노드 에이전트인 kubelet이 swap이 켜져 있으면 기본적으로 실행을 거부하거나 노드를 NotReady로 두기 때문이다.

swap: RAM 부족 시 디스크를 메모리처럼 쓰는 기능.

Kubernetes에서 문제되는 부분

1. Pod 메모리 제한 무너짐

- swap 켜짐으로 디스크로 밀려나고 제한 의미가 없어짐.

2. 성능 예측 불가능

- swap 사용시 느려짐
- Pod 응답시간이 튐

3. OOM 관리가 깨짐

- Kubernetes는 메모리 부족 시 Pod Kill을 해야 정상임.
- swap이 있으면 안 죽고 디스크로 이동함.

따라서 swap 비활성화를 모든 노드에서 실행해준다.

```bash
sudo swapoff -a
```

swap이 비활성화인지 확인

```bash
free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       416Mi       3.3Gi       1.1Mi       312Mi       3.4Gi
Swap:             0B          0B          0B
```

또는

```bash
cat /etc/fstab

/swapfile none swap sw 0 0 # 이런 내용이 없으면 됨
```

### 커널 모듈 및 파라미터 설정

K8s는 Pod 간 네트워크 통신을 위해 Linux 커널 기능을 활성화해야 한다.
기본 Ubuntu 설정만으로는 부족하기 때문에 아래 두 가지를 추가로 설정한다.

1. 커널 모듈 로드
   커널에 추가로 기능을 붙이는 것 (기본값으로 꺼져있는 기능을 켜는 것)

```bash
# overlay: containerd가 사용하는 파일시스템 레이어 기능
# br_netfilter: 브리지 네트워크 트래픽을 iptables가 처리할 수 있게 해줌
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 커널 모듈 로드 확인
lsmod | grep br_netfilter
lsmod | grep overlay

# br_netfilter           32768  0
# bridge                425984  1 br_netfilter
# overlay               212992  0

```

2. 커널 파라미터 설정
   켜진 기능의 세부 동작 방식을 조정하는 것

```bash
# net.bridge.bridge-nf-call-iptables  = 브리지 트래픽을 iptables로 처리
# net.bridge.bridge-nf-call-ip6tables = 위와 동일 (IPv6)
# net.ipv4.ip_forward                 = 노드 간 패킷 전달 허용
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 재부팅 없이 즉시 적용
sudo sysctl --system

# 커널 파라미터 적용 확인
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
# net.bridge.bridge-nf-call-iptables = 1
# net.ipv4.ip_forward = 1

```

<details>
  <summary>왜 필요한가?</summary>

```
  [ Pod A ] ──브리지──▶ [ Pod B ]
                          ↑
                iptables가 여기서 네트워크 정책/라우팅 처리
                br_netfilter 없으면 iptables가 브리지 트래픽을 못 봄
```

K8S는 Pod간 통신을 브리지 네트워크로 처리한다.
기본 Linux 설정에서는 브리지 트래픽이 iptables를 우회한다.
이것을 통과하도록 강제하는 것이다.

`ip_forward`는 노드가 패킷을 다른 노드로 전달할 수 있게 해주는 설정으로, 클러스터 내 노드 간 통신에 필수이다.

- 안 하면 생기는 문제
- br_netfilter 없으면 Pod 간 통신이 iptables를 우회해서 K8s 네트워크 정책이 적용 안 됨
- CNI(Flannel, Calico 등) 설치해도 Pod 간 통신 안 됨
- ip_forward 없으면 노드 간 패킷 전달이 안 돼서 다른 노드의 Pod에 접근 불가

</details>

### 컨테이너 런타임 설치(containerd)

```bash
# 패키지 목록 업데이트
sudo apt-get update

# containerd 설치
sudo apt-get install -y containerd

# containerd 설치 확인
containerd --version

# containerd 기본 설정 파일 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# SystemdCgroup 활성화 (K8s가 요구하는 cgroup 드라이버)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# containerd 재시작 및 부팅 시 자동 시작 등록
sudo systemctl restart containerd
sudo systemctl enable containerd

# containerd 실행 중인지 확인
sudo systemctl status containerd
```

K8S의 kubelet과 containerd가 같은 cgroup 드라이버를 써야 충돌이 안난다.
Ubuntu 24.04는 systemd 기반이라 `SystemdCgroup = true`로 맞춰줘야 한다.
안하면 나중에 `kubeadm init`에서 에러난다.

### kubeadm, kubelet, kubectl 설치

| 도구    | 역할                                            |
| ------- | ----------------------------------------------- |
| kubeadm | 클러스터 초기화/설정 도구(설치할때만 주로 사용) |
| kubelet | 각 노드에서 Pod를 실제로 실행시키는 에이전트    |
| kubectl | 클러스터에 명령 내리는 CLI 도구                 |

```bash
# 필요한 패키지 설치
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Google 패키지 저장소 키 등록
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# K8s 패키지 저장소 추가
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


# 패키지 목록 업데이트 후 설치
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 버전 고정 (자동 업데이트 방지)
sudo apt-mark hold kubelet kubeadm kubectl
```

## Control Plane 초기화

여기서 부터는 controlplane에서만 진행한다.

먼저 controlplane IP를 확인한다.

```bash
ip a
# 192.168.56.102
```

다른 worker 노드에서 해당 IP에 접근 가능한지 확인한다.

```bash
pin 192.168.56.102
```

그 다음 kubadm init을 한다.

```bash
# Calico 기본 CIDR과 대역이 겹치지 않게 피해서 pod-network-cidr을 설정한다.
sudo kubeadm init \
--apiserver-advertise-address=192.168.56.102 \
--pod-network-cidr=10.244.0.0/16

# ...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.102:6443 --token <token> --discovery-token-ca-cert-hash sha256:<HASH>

```

**kubeadm join 192.168.56.102:6443 --token <token> --discovery-token-ca-cert-hash sha256:<HASH>** 잘 기억해둔다!. 이따가 워커노드 조인할 때 필요하다

까먹어도 Control Plane에서 재발급이 가능하다

````bash
kubeadm token create --print-join-command
``

## kubectl 사용 설정

`kubeadm init`이 끝나면 kubeconfig파일이 `/etc/kubernetes/admin.conf`에 생성된다.
이걸 `$HOME/.kube/config`에 복사해야 `kubectl` 명령어를 쓸 수 있다.

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

````

## CNI 설치 - Calico

CNI 종류는 여러가지가 있는데 Calico가 설치가 비교적 쉽고 Pod 간 통신뿐 아니라 보안정책까지 실습하기 괜찮다고 하여 Clico를 선택했다. Flannel보다 기능이 많으면서도 Cilium보다는 진입장벽이 낮다고 한다.

Calico 기본 CIDR이 `192.168.0.0/16`인데, VM IP 대역이 `192.168.56.x`라 충돌한다.

혹시라도 VM의 Host-Only 네트워크 대역이랑 Calico IP 대역이 겹친다면 kubeadm reset 후 다시 init을 진행한다

```bash
# 클러스터 리셋
sudo kubeadm reset
sudo rm -rf $HOME/.kube

# 다시 init
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.102 \
  --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Calico CNI를 설치한다.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.5/manifests/calico.yaml
```

## Worker Node 클러스터 등록

`kubeadm init`완료 했을 때 출력됐던 join 명령어를 각 워커 노드들에서 실행한다.

```bash
kubeadm join 192.168.56.102:6443 --token <token> --discovery-token-ca-cert-hash sha256:<HASH>
```

join 명령어를 잊어버렸다면 Control Plane에서 재발급한다.

```bash
kubeadm token create --print-join-command
```

sudo 권한이 없을 때 오류가 날 수 있으므로 `sudo`를 붙여준다.

```bash
kubeadm join 192.168.56.102:6443 --token <token> --discovery-token-ca-cert-hash sha256:<HASH>
[preflight] Running pre-flight checks
[preflight] Some fatal errors occurred:
        [ERROR IsPrivilegedUser]: user is not running as root
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
error: error execution phase preflight: preflight checks failed
To see the stack trace of this error execute with --v=5 or higher


sudo kubeadm join 192.168.56.102:6443 --token <token> --discovery-token-ca-cert-hash sha256:<HASH>
```

## 클러스터 상태 확인

Control Plane에서 워커노드들이 잘 join 되었는지 확인한다.

```bash
kubectl get nodes
NAME               STATUS     ROLES           AGE   VERSION
k8s-controlplane   Ready      control-plane   45m   v1.35.4
k8s-worker1        Ready      <none>          12m   v1.35.4
k8s-worker2        Ready      <none>          12m   v1.35.4
k8s-worker3        Ready      <none>          1m    v1.35.4
```

nginx Pod 배포 후 정상 실행 확인한다.

```bash
# nginx Pod 배포
kubectl run nginx --image=nginx

# Pod 상태 확인 (Running 이면 정상)
kubectl get pods -o wide
```

Pod간 통신도 확인한다.(CNI 정상 동작 확인)

```bash
# busybox Pod 하나 더 띄우기
kubectl run busybox --image=busybox --rm -it --restart=Never -- sh

# busybox 쉘 안에서 nginx Pod IP로 ping
ping <nginx Pod IP>
```
