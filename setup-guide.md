# KVM Network Lab 構築手順書

## 1. 目的
本手順書は、AlmaLinux をホストOSとした KVM 環境上に<br>
複数の仮想ネットワークおよびルータVMを構築し、<br>
異なるネットワーク間の疎通確認を行うことを目的とする。

## 2. 前提条件
- Host OS: AlmaLinux 9
- CPU: Intel VT-x 対応
- メモリ: 16GB 以上
- ネットワーク: Host OS は Wi-Fi 接続
- 管理方法: SSH（Tailscale 経由）

## 3. ホストOS準備

### 3.1 仮想化支援機能の確認
CPU が仮想化支援機能（Intel VT-x）に対応していることを確認する。
```bash
lscpu | grep Virtualization
```
以下のように表示されれば問題ない。
```bash
Virtualization: VT-x
```

### 3.2 KVM / libvirt 関連パッケージのインストール
```bash
sudo dnf -y install \
  qemu-kvm \
  libvirt \
  libvirt-daemon \
  libvirt-daemon-config-network \
  libvirt-daemon-driver-qemu \
  virt-install \
  bridge-utils
```

### 3.3 libvirtd の起動と自動起動設定
```bash
sudo systemctl enable --now libvirtd
```

### 3.4 ユーザーを libvirt グループへ追加
通常ユーザーで `virsh` や `virt-install` を実行できるようにする。
```bash
sudo usermod -aG libvirt haru
reboot
```
再起動後、以下のコマンドが実行できることを確認する。
```bash
virsh list --all
```

## 4. 仮想ネットワークの作成
本構成では、libvirt NAT ネットワークを 2 つ作成する。

### ネットワーク構成
- network1
  - セグメント: 172.16.0.0/24
  - ゲートウェイ: 172.16.0.254
  - 接続VM: host01, host02
- network2
  - セグメント: 172.17.0.0/24
  - ゲートウェイ: 172.17.0.254
  - 接続VM: host03, host04
- ルータ VM (host00)
  - network1 と network2 を中継

### 4.1 network1 の作成
network1 用の設定ファイルを作成する。
```bash
cd /var/lib/libvirt/network

cat > network1.xml << EOF
<network>
  <name>network1</name>
  <bridge name='br-net1' stp='on' delay='0'/>
  <forward mode='nat'/>
  <ip address='172.16.0.254' netmask='255.255.255.0'/>
</network>
EOF
```
ネットワークを定義・起動する。
```bash
virsh net-define network1.xml
virsh net-start network1
virsh net-autostart network1
```
確認：
```bash
virsh net-list --all
```

### 4.2 network2 の作成
network2 用の設定ファイルを作成する。
```bash
cat > network2.xml << EOF
<network>
  <name>network2</name>
  <bridge name='br-net2' stp='on' delay='0'/>
  <forward mode='nat'/>
  <ip address='172.17.0.254' netmask='255.255.255.0'/>
</network>
EOF
```
ネットワークを定義・起動する。
```bash
virsh net-define network2.xml
virsh net-start network2
virsh net-autostart network2
```
確認：
```bash
virsh net-list --all
```

## 5. VM の作成（cloud-init）
本章では cloud-init を利用して、<br>
ホスト VM（host01～host04）および ルータ VM (host00) を作成する。

### 5.1 meta-data の作成
```bash
sudo mkdir -p /var/lib/libvirt/images/cloud-init
cd /var/lib/libvirt/images/cloud-init

cat > meta-data-host00 << EOF
instance-id: host00
local-hostname: host00
EOF
```
同様の手順で host01 ～ host04 用の meta-data を作成する。

### 5.2 user-data の作成
```bash
cat > user-data << EOF
#cloud-config

timezone: Asia/Tokyo
locale: en_US.UTF-8

users:
  - name: haru
    groups: wheel
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 ...

ssh_pwauth: false
disable_root: true

package_update: true
package_upgrade: true
packages:
  - vim
  - chrony
  - iproute
  - iputils
  - nftables
  - tcpdump
  - traceroute
  - bind-utils

runcmd:
  - setenforce 0
  - sed -i -e 's/^\SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
  - systemctl stop firewalld
  - systemctl disable firewalld
  - systemctl restart rsyslog
EOF
```

### 5.3 AlmaLinux Cloud Image の取得
```bash
sudo mkdir -p /var/lib/libvirt/images/base
cd /var/lib/libvirt/images/base

curl -LO \
https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
```

### 5.4 VM 用ディスク の作成
```bash
sudo mkdir -p /var/lib/libvirt/images/vm

sudo qemu-img create \
  -f qcow2 \
  -b /var/lib/libvirt/images/base/AlmaLinux-9-GenericCloud.qcow2 \
  /var/lib/libvirt/images/vm/host00.qcow2
```
同様の手順で host01～host04 用のディスクを作成する。

### 5.5 cloud-init seed ISO の作成
```bash
cd /var/lib/libvirt/images/cloud-init

cloud-localds seed-host00.iso user-data meta-data-host0
```
同様の手順で host01～host04 用の seed ISO を作成する。

### 5.5 VM の作成（virt-install）
virt-install を用いて VM を作成する。<br>
VM の役割に応じて、接続する仮想ネットワークを指定する。

#### 5.5.1 ルータ VM (host00)
ルータVM (host00) は以下の3つのネットワークに接続する。
- `default` (管理用)
- `network1`
- `network2`
```bash
sudo virt-install \
  --name host00 \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/vm/host00.qcow2,format=qcow2 \
  --disk path=/var/lib/libvirt/images/cloud-init/seed-host00.iso,device=cdrom \
  --os-variant almalinux9 \
  --network network=default,model=virtio \
  --network network=network1,model=virtio \
  --network network=network2,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --import
```

#### 5.5.2 ホストVM (network1 側))
host1 および host2 は network1 に接続する。
```bash
sudo virt-install \
  --name host01 \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/vm/host01.qcow2,format=qcow2 \
  --disk path=/var/lib/libvirt/images/cloud-init/seed-host01.iso,device=cdrom \
  --os-variant almalinux9 \
  --network network=network1,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --import
```
同様の手順で host02 を作成する。

#### 5.5.3 ホスト VM (network2 側)
host03 および host04 は network2 に接続する。
```bash
sudo virt-install \
  --name host03 \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/vm/host03.qcow2,format=qcow2 \
  --disk path=/var/lib/libvirt/images/cloud-init/seed-host03.iso,device=cdrom \
  --os-variant almalinux9 \
  --network network=network2,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --import
```
同様の手順で host04 を作成する。

#### 5.5.4 作成確認
作成した VM の一覧を確認する。
```bash
virsh list --all
```

## 6. VM 設定

## 7. 疎通確認
