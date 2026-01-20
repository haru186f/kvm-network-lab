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
- 実行ユーザ: root

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
dnf update

dnf -y install \
  qemu-kvm \
  libvirt \
  libvirt-daemon \
  libvirt-daemon-config-network \
  libvirt-daemon-driver-qemu \
  virt-install
```

### 3.3 libvirtd の起動と有効化
```bash
systemctl enable --now libvirtd
```

### 3.4 libvirt グループの追加
一般ユーザでも VM の操作（起動・停止・一覧表示）を行えるようになる。
```bash
usermod -aG libvirt haru
```

## 4. 仮想ネットワークの作成

### ネットワーク構成
- default（NAT）
  - セグメント: 192.168.122.0/24
  - ゲートウェイ: 192.168.122.1
  - 接続VM: host00
- network1（isolated）
  - セグメント: 172.16.0.0/24
  - ゲートウェイ: 172.16.0.254
  - 接続VM: host00, host01, host02
- network2（isolated）
  - セグメント: 172.17.0.0/24
  - ゲートウェイ: 172.17.0.254
  - 接続VM: host00, host03, host04

### 4.1 network1 の作成
```bash
cd /var/lib/libvirt/network

cat > network1.xml << EOF
<network>
  <name>network1</name>
  <bridge name='virbr-net1'/>
</network>
EOF

virsh net-define network1.xml
virsh net-start network1
virsh net-autostart network1
```

### 4.2 network2 の作成
```bash
cat > network2.xml << EOF
<network>
  <name>network2</name>
  <bridge name='virbr-net2'/>
</network>
EOF

virsh net-define network2.xml
virsh net-start network2
virsh net-autostart network2
```

## 5. 仮想マシンの作成（cloud-init）

### 5.1 meta-data の作成
```bash
mkdir -p /var/lib/libvirt/images/cloud-init
cd /var/lib/libvirt/images/cloud-init

for i in {00..04}; do
  cat > meta-data-host${i} << EOF
instance-id: host${i}
local-hostname: host${i}
EOF
done
```

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
  - sed -i -e 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
EOF
```

### 5.3 AlmaLinux Cloud Image の取得
```bash
mkdir -p /var/lib/libvirt/images/base
cd /var/lib/libvirt/images/base

curl -LO \
https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
```

### 5.4 VM 用差分ディスク の作成
```bash
mkdir -p /var/lib/libvirt/images/vm

for i in {00..04}; do
  qemu-img create \
    -f qcow2 \
    -b /var/lib/libvirt/images/base/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2 \
    /var/lib/libvirt/images/vm/host${i}.qcow2
    20G
done
```

### 5.5 cloud-init seed ISO の作成
```bash
cd /var/lib/libvirt/images/cloud-init

for i in {00..04}; do
  cloud-localds seed-host${i}.iso user-data meta-data-host${i}
done
```

### 5.6 ルータ VM の作成（host00）
```bash
virt-install \
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

### 5.7 ホスト VM の作成（host01-04）
```bash
for i in {01..04}; do
  if [[ $i -le  2 ]]; then
    NET=network1
  else
    NET=network2
  fi

  virt-install \
    --name host${i} \
    --memory 1024 \
    --vcpus 1 \
    --disk path=/var/lib/libvirt/images/vm/host${i}.qcow2,format=qcow2 \
    --disk path=/var/lib/libvirt/images/cloud-init/seed-host${i}.iso,device=cdrom \
    --os-variant almalinux9 \
    --network network=${NET},model=virtio \
    --graphics none \
    --console pty,target_type=serial \
    --import
done
```

## 6. VM 設定

## 7. 疎通確認
