# KVM Network Lab 構築手順書

## 1. 目的
本手順書は、AlmaLinux をホストOSとした KVM 環境上に
複数の仮想ネットワークとルータVMを構築し、
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
再起動後、以下が実行できることを確認する。
```bash
virsh list --all
```

### 3.5 AlmaLinux Cloud Image 取得
```bash
sudo mkdir -p /var/lib/libvirt/images/cloud-init
sudo chown -R haru:haru /var/lib/libvirt/images/cloud-init

cd /var/lib/libvirt/images
curl -O https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
```

## 4. 仮想ネットワークの作成
本構成では、libvirt の NAT ネットワークを 2 つ作成する。
- network1
    - セグメント: 172.16.0.0/16
    - ゲートウェイ: 172.168.255.254
    - 用途: host01, host02
- network2
    - セグメント: 172.17.0.0/16
    - ゲートウェイ: 172.17.255.254
    - 用途: host03, host04
- ルータ VM（host00）が両ネットワークを中継する

### 4.1 network1の作成
network1.xmlを作成する。
```bash
<network>
  <name>network1</name>
  <bridge name='br-net1' stp='on' delay='0'/>
  <forward mode='nat'/>
  <ip address='172.16.255.254' netmask='255.255.0.0'>
    <dhcp>
      <range start='172.16.0.1' end='172.16.255.253'/>
    </dhcp>
  </ip>
</network>
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
<network>
  <name>network2</name>
  <bridge name='br-net2' stp='on' delay='0'/>
  <forward mode='nat'/>
  <ip address='172.17.255.254' netmask='255.255.0.0'>
    <dhcp>
      <range start='172.17.0.1' end='172.17.255.253'/>
    </dhcp>
  </ip>
</network>

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

## 6. ルータVM（host00）設定

## 7. 疎通確認
