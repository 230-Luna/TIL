# setup 에러

## 목차

- [kubaadm init 하다가 난 에러](#kubaadm-init-하다가-난-에러)

## kubaadm init 하다가 난 에러

```bash
sudo kubeadm init \
--apiserver-advertise-address=192.168.56.102 \
--pod-network-cidr=192.168.0.0/16

[init] Using Kubernetes version: v1.35.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-controlplane kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.56.102]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-controlplane localhost] and IPs [192.168.56.102 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-controlplane localhost] and IPs [192.168.56.102 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 1.510560715s
[control-plane-check] Waiting for healthy control plane components. This can take up to 4m0s
[control-plane-check] Checking kube-apiserver at https://192.168.56.102:6443/livez
[control-plane-check] Checking kube-controller-manager at https://127.0.0.1:10257/healthz
[control-plane-check] Checking kube-scheduler at https://127.0.0.1:10259/livez
[control-plane-check] kube-apiserver is not healthy after 4m38.376821078s

[control-plane-check] kube-controller-manager is not healthy after 4m38.456753845s
[control-plane-check] kube-scheduler is not healthy after 4m38.484889333s

A control plane component may have crashed or exited when started by the container runtime.
To troubleshoot, list all containers using your preferred container runtimes CLI.
Here is one example how you may list all running Kubernetes containers by using crictl:
        - 'crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps -a | grep kube | grep -v pause'
        Once you have found the failing container, you can inspect its logs with:
        - 'crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock logs CONTAINERID'

error: error execution phase wait-control-plane: failed while waiting for the control plane to start: [kube-apiserver check failed at https://192.168.56.102:6443/livez: Get "https://192.168.56.102:6443/livez?timeout=10s": dial tcp 192.168.56.102:6443: connect: connection refused, kube-controller-manager check failed at https://127.0.0.1:10257/healthz: Get "https://127.0.0.1:10257/healthz": dial tcp 127.0.0.1:10257: connect: connection refused, kube-scheduler check failed at https://127.0.0.1:10259/livez: Get "https://127.0.0.1:10259/livez": dial tcp 127.0.0.1:10259: connect: connection refused]
To see the stack trace of this error execute with --v=5 or higher


```

### 문제

control plane 컨테이너가 뜨지 못해서 kubeadn init 실패

### 원인

### 해결

`running` 상태인지 확인

```bash
sudo crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps -a
WARN[0000] Config "/etc/crictl.yaml" does not exist, trying next: "/usr/bin/crictl.yaml"
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD                                        NAMESPACE
d66af3cc15059       a0eecd9b69a38       2 minutes ago       Running             kube-scheduler            1                   c9d376cd7fac6       kube-scheduler-k8s-controlplane            kube-system
59c41cd754dbe       96ce7469899d4       2 minutes ago       Running             kube-controller-manager   1                   eb3fd8cc563f2       kube-controller-manager-k8s-controlplane   kube-system
8c4ba7486a70e       840f22aa169cc       8 minutes ago       Running             kube-apiserver            0                   9319d7a46dca0       kube-apiserver-k8s-controlplane            kube-system
ded6a1459ade6       0a108f7189562       8 minutes ago       Running             etcd                      0                   e0fd67d36dab2       etcd-k8s-controlplane                      kube-system


```

containerd 설정에서 SystemdCgroup = true로 되어 있는가?

```bash
grep SystemdCgroup /etc/containerd/config.toml
SystemdCgroup = false # true가 정상

# true로 수정
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' \
/etc/containerd/config.toml

# 재시작
sudo systemctl restart containerd
```

init 실패했으므로 reset 후 다시 init

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo systemctl restart containerd


sudo kubeadm init \
--apiserver-advertise-address=192.168.56.102 \
--pod-network-cidr=192.168.0.0/16
```
