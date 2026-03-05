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

## 3. ホスト OS の準備

### 3.1 仮想化支援機能の確認
CPU が仮想化支援機能（Intel VT-x）に対応していることを確認する。
```bash
lscpu | grep Virtualization
```
以下のように表示されれば問題ない。
```bash
Virtualization: VT-x
```

### 3.2 パッケージのインストール
```bash
dnf update -y

dnf install -y \
  qemu-kvm \
  libvirt \
  libvirt-daemon \
  libvirt-daemon-config-network \
  libvirt-daemon-driver-qemu \
  virt-install \
  genisoimage
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
cat > /var/lib/libvirt/network/network1.xml << EOF
<network>
  <name>network1</name>
  <bridge name='virbr-net1'/>
</network>
EOF

# ネットワークを定義・起動する
virsh net-define /var/lib/libvirt/network/network1.xml
virsh net-start network1
virsh net-autostart network1

# 確認
virsh net-list --all
```

### 4.2 network2 の作成
```bash
cat > /var/lib/libvirt/network/network2.xml << EOF
<network>
  <name>network2</name>
  <bridge name='virbr-net2'/>
</network>
EOF

# ネットワークを定義・起動する
virsh net-define /var/lib/libvirt/network/network2.xml
virsh net-start network2
virsh net-autostart network2

# 確認
virsh net-list --all
```

## 5. 仮想マシンの作成（cloud-init）

### ディレクトリ構成
```bash
/var/lib/libvirt/images
├── base
│   └── AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
└── vms
    ├── host00
    │   ├── disk.qcow2
    │   ├── seed.iso
    │   ├── user-data
    │   └── meta-data
    ├── host01
    │   ├── disk.qcow2
    │   ├── seed.iso
    │   ├── user-data
    │   └── meta-data
    ├── host02
    │   ├── disk.qcow2
    │   ├── seed.iso
    │   ├── user-data
    │   └── meta-data
    ├── host03
    │   ├── disk.qcow2
    │   ├── seed.iso
    │   ├── user-data
    │   └── meta-data
    └── host04
        ├── disk.qcow2
        ├── seed.iso
        ├── user-data
        └── meta-data
```

### 5.1 VM用ディレクトリの作成
```bash
mkdir -p /var/lib/libvirt/images/vms
mkdir -p /var/lib/libvirt/images/base
```

### 5.2 meta-data / user-data の作成
```bash
for i in {00..04}; do
  DIR=/var/lib/libvirt/images/vms/host${i}

  # vm 用ディレクトリを作成
  mkdir -p $DIR

  # meta-data を作成
  cat > $DIR/meta-data << EOF
instance-id: host${i}
local-hostname: host${i}
EOF

  # user-data を作成
  cat > $DIR/user-data << EOF
#cloud-config

timezone: Asia/Tokyo
locale: en_US.UTF-8

users:
  - name: haru
    groups: wheel
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-ed25519 ...

ssh_pwauth: false
disable_root: true

package_update: true

packages:
  - vim
  - git
  - iproute
  - iputils
  - nftables
  - tcpdump
  - traceroute
  - bind-utils

runcmd:
  - setenforce 0
  - sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
EOF
done
```

### 5.3 AlmaLinux Cloud Image の取得
```bash
curl -L -o /var/lib/libvirt/images/base/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2 \
https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
```

### 5.4 VM用差分ディスク / ISOイメージの作成
```bash
for i in {00..04}; do

  BASE_DIR=/var/lib/libvirt/images/base
  VMS_DIR=/var/lib/libvirt/images/vms/host${i}

  # VM用差分ディスクを作成
  qemu-img create -f qcow2 \
    -b $BASE_DIR/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2 \
    $VMS_DIR/disk.qcow2 20G

  # ISOイメージを作成
  genisoimage \
    -output $VMS_DIR/seed.iso \
    -volid cidata \
    -joliet \
    -rock \
    $VMS_DIR/user-data \
    $VMS_DIR/meta-data
done
```

### 5.5 VM の作成
```bash
for i in {00..04}; do

  DIR=/var/lib/libvirt/images/vms/host${i}

  if [[ $i -eq 0 ]]; then
    virt-install \
      --name host00 \
      --memory 1024 \
      --vcpus 1 \
      --disk path=$DIR/disk.qcow2,format=qcow2,bus=virtio \
      --disk path=$DIR/seed.iso,device=cdrom \
      --os-variant almalinux9 \
      --network network=default,model=virtio \
      --network network=network1,model=virtio \
      --network network=network2,model=virtio \
      --import \
      --autostart \
      --noautoconsole
  else
    if [[ $i -le  2 ]]; then
      NET=network1
    else
      NET=network2
    fi

    virt-install \
      --name host${i} \
      --memory 1024 \
      --vcpus 1 \
      --disk path=$DIR/disk.qcow2,format=qcow2,bus=virtio \
      --disk path=$DIR/seed.iso,device=cdrom \
      --os-variant almalinux9 \
      --network network=default,model=virtio \
      --network network=${NET},model=virtio \
      --import \
      --autostart \
      --noautoconsole
  fi
done

# 確認
virsh list --all
```

## 6. VM の設定

## 7. 疎通確認
