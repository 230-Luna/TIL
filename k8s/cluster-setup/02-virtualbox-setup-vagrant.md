# Vagrant를 활용해서 VirtualBox VM 자동 생성하기

## 목차

- [Vagrant란?](#Vagrant란?)
- [VirtualBox, Vagrant 설치](#VirtualBox-Vagrant-설치)
- [Vagrant 환경 설정](#Vagrant-환경-설정)
- [Vagrant 실행](#Vagrant-실행)
- [SSH 클라이언트 접속](#SSH-클라이언트-접속)
- [Vagrantfile 내용 추가](#Vagrantfile-내용-추가)

---

## Vagrant란?

Hashicorp에서 개발한 VirtualBox, VMware 등 가상화 소프트웨어를 코드로 관리하여 개발 환경을 자동 생성하는 도구이다.
VirtualBox + VM 생성 + 네트워크 설정 + SSH 설정을 자동화 해준다.

## VirtualBox, Vagrant 설치

터미널을 열어 virtualbox와 varant를 설치한다.

```bash
  winget install --id Oracle.VirtualBox -v 7.2.6 -e
  winget install --id Hashicorp.Vagrant -v 2.4.9 -e
```

> VirtualBox 및 MobaXterm 설치는 [01-virtualbox-setup-manual.md](./01-virtualbox-setup-manual.md)를 참고한다.

## Vagrant 환경 설정

Vagrant 프로젝트를 위한 디렉토리를 생성한다.

```bash
mkdir vagrant-k8s-cluster
cd vagrant-k8s-cluster
```

Vagrant 환경을 초기화한다.

```bash
vagrant init
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```

Vagrantfile 파일이 생성된다.

생성된 Vagrantfile을 열어 수정한다.

```ruby
# Worker Node 개수 (이 숫자만 바꾸면 worker 수 조절 가능)
N = 3

# "2" = Vagrantfile API 버전 (현재 표준, 1은 구버전)
# do ... end 문법 아래 end 까먹지 말것.
# |변수이름|
Vagrant.configure("2") do |config|

  # Control Plane
  config.vm.define "k8s-cluster-control-plane" do |cp|
    # Vagrant box = 미리 만들어지 VM이미지(OS + 기본 설정)
    # https://vagrantcloud.com/search 여기서 검색 가능
    # Canonical will no longer publish Vagrant images directly starting with Ubuntu 24.04 LTS (Noble Numbat).
    cp.vm.box = "bento/ubuntu-24.04"
    cp.vm.hostname = "cp"
    cp.vm.provider "virtualbox" do |vb|
      vb.name = "k8s-cluster-control-plane"
      vb.cpus = 2
      vb.memory = 4096
      # VBoxManage CLI를 통해 VM 옵션을 세부 조정하는 명령어
      # "modifyvm" = VM 설정 변경, "--groups" = VirtualBox UI에서 그룹으로 묶어서 표시
      # k8s-cluster-group 그룹 추가해서 vm을 해당 그룹 내에 생성하게 함
      vb.customize ["modifyvm", :id, "--groups", "/k8s-cluster-group"]
    end
    # Host-Only 네트워크로 고정 IP 할당
    # 192.168.56.x 는 VirtualBox Host-Only 어댑터의 기본 대역
    cp.vm.network "private_network", ip: "192.168.56.10"
    # 호스트 PC 폴더와 VM 내부 폴더를 서로 공유(동기화)하는 기능
    # Vagrant는 기본적으로 호스트 프로젝트 폴더를 VM의 /vagrant에 자동 마운트함.
    # 이 박스는 VirtualBox Guest Additions가 없어 마운트 시 에러가 발생하므로,
    # disabled: true로 해당 기본 동기화를 명시적으로 비활성화함.
    cp.vm.synced_folder "../data", "/vagrant", disabled: true
  end

  # Worker Nodes

  # 1부터 N까지 반복 (N=3이면 1,2,3 순서로 worker VM 3개 생성)
  (1..N).each do |i|
    config.vm.define "k8s-cluster-worker#{i}" do |worker|
      worker.vm.box = "bento/ubuntu-24.04"
      worker.vm.hostname = "worker#{i}"
      worker.vm.provider "virtualbox" do |vb|
        vb.name = "k8s-cluster-worker#{i}"
        vb.cpus = 2
        vb.memory = 1024
        vb.customize ["modifyvm", :id, "--groups", "/k8s-cluster-group"]
      end
      # ip 끝자리: 10+i → worker1=.11, worker2=.12, worker3=.13
      worker.vm.network "private_network", ip: "192.168.56.#{10+i}"
      worker.vm.synced_folder "../data", "/vagrant", disabled: true
    end
  end
end


```

<details>
<summary>
VirtualBox Guest Additions이란?
</summary>
VM 내부에 설치하는 드라이버/도구 모음으로, 호스트 PC와 VM이 더 잘 협력하도록 해주는 소프트웨어
</details>

## Vagrant 실행

터미널을 열어 Vagrantfile이 있는 위치로 이동한다.
현재 디렉터리에서 Vagrantfile 기준으로 VM 생성한다

```bash
# Vagrantfile을 읽어 정의된 VM을 모두 생성하고 시작
vagrant up
```

## SSH 클라이언트 접속

Virtualbox에 vm이 생성된 것을 확인한다.

MobaXterm으로 해당 vm에 접속한다.
(MobaXterm = Windows용 SSH 클라이언트)

기본 비밀번호는 `vagrant`이다.

```bash
username: vagrant
password: vagrant
```

> 실습용 설정이므로 기본 비밀번호를 그대로 사용했다.
> 실제 운영할 때는 패스워드 로그인을 차단하고 SSH 키 인증만 허용해야 한다.

## Vagrantfile 내용 추가

추후에 내용 추가할 예정.
쿠버네티스 설치 스크립트 추가하면 vm 생성과 동시에 쿠버네티스 클러스터 구축까지 한번에 끝낼 수 있다.

VM 생성 시 자동으로 실행할 초기화 스크립트를 지정한다.

예시:

```ruby
cfg.vm.provision "shell", path: "scripts/install.sh"
# path: Vagrantfile 기준 상대경로로 스크립트 파일 지정
```

`vagrant up` 또는 `vagrant provision` 실행 시 해당 스크립트가 root 권한으로 자동 실행된다.

`vagrant up` 실행 시 VM 최초 생성과 함께 스크립트가 자동 실행된다.
이미 VM이 켜져 있는 상태에서 스크립트만 다시 실행하려면 `vagrant provision`을 사용한다.

> 스크립트 파일은 Vagrantfile과 같은 디렉터리 기준 상대경로로 지정한다.

### 디렉터리 구조 예시

```
vagrant-k8s-cluster/
├── Vagrantfile
└── scripts/
    ├── common.sh     # 모든 VM 공통 설정 (패키지 설치 등)
    └── cp.sh         # Control Plane 전용 설정

```
