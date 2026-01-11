# KVM Network Lab 構築手順書

## 1. 目的
本手順書は、AlmaLinux をホストOSとした KVM 環境上に<br>
複数の仮想ネットワークとルータVMを構築し、<br>
ネットワーク間疎通を確認することを目的とする。

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

### 3.2 KVM / QEMU / libvirt 関連パッケージのインストール
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
再起動後、以下が実行できることを確認する。
```bash
virsh list --all
```

## 4. 仮想ネットワークの作成
本構成では、libvirt の NAT ネットワークを 2 つ作成する。
- network1
  - セグメント: 172.16.0.0/24
  - ゲートウェイ: 172.16.0.254
  - ホスト: host01, host02
- network2
  - セグメント: 172.17.0.0/24
  - ゲートウェイ: 172.17.0.254
  - ホスト: host03, host04
- ルータ VM（host00）が両ネットワークを中継する

### 4.1 network1の作成
network1.xmlを作成する。
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

### 4.2 network2の作成
network2.xmlを作成する。
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

## 5. VM 作成（cloud-init）
本章では cloud-init を利用して、<br>
ホスト VM（host01–host04）および ルータ VM (host00) を作成する。

### 5.1 meta-data の作成
cloud-init 用の meta-data を作成する。
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
すべての VM で共通の user-data を作成する。
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
cloud-init 対応の AlmaLinux Cloud Image を取得する。
```bash
sudo mkdir -p /var/lib/libvirt/images/base
cd /var/lib/libvirt/images/base

curl -LO https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
```

### 5.4 VM用ディスク の作成
ベースイメージを backing file とした qcow2 形式の差分ディスクを作成する。
```bash
sudo mkdir -p /var/lib/libvirt/images/vm

sudo qemu-img create \
  -f qcow2 \
  -b /var/lib/libvirt/images/base/AlmaLinux-9-GenericCloud.qcow2 \
  /var/lib/libvirt/images/vm/host00.qcow2
```
同様の手順で host01～host04 用のディスクを作成する。

### 5.5 cloud-init seed ISO の作成
meta-data および user-data から seed ISO を作成する。
```bash
cd /var/lib/libvirt/images/cloud-init

cloud-localds seed-host00.iso user-data meta-data-host0
```
同様の手順で host01～host04 用の seed ISO を作成する。

### 5.5 VM の作成（virt-install）
VM の作成（virt-install）
```bash
virt-install \
  --name host00 \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/vm/host00.qcow2,format=qcow2 \
  --disk path=/var/lib/libvirt/images/cloud-init/seed-host00.iso,device=cdrom \
  --os-variant almalinux9 \
  --network network=network1,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --import
```
確認：
```bash
virsh list
```
- host02：`network1`を指定する
- host03、host04：`network2`を指定する
- host00 ：
  - `--network` を 3 回指定する
  - `default → network1 → network2` の順で接続する

## 6. VM 設定

## 7. 疎通確認
