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
    - ゲートウェイ: 172.16.255.254
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
  <ip address='172.16.255.254' netmask='255.255.0.0'/>
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
  <ip address='172.17.255.254' netmask='255.255.0.0'/>
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
本章では cloud-init を利用して、<br>
ホスト VM（host01–host04）および ルータ VM (host00) を作成する。

### 5.1 meta-data 作成
```bash
cd /var/lib/libvirt/images/cloud-init
vim meta-data.yaml
```
```bash
instance-id: host01
local-hostname: host01
```
※ host02〜host04、host00 作成時は<br>
`instance-id` と `local-hostname` をそれぞれ変更する。

### 5.2 ホスト用 user-data 作成 (host01-04)
```bash
vim user-data-host.yaml
```
```bash
#cloud-config

hostname: host01
fqdn: host01.knowd.co.jp

timezone: Asia/Tokyo
locale: en_US.UTF-8

users:
  - default
  - name: haru
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: [wheel]
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 ...

disable_root: true
ssh_pwauth: false

package_update: true
package_upgrade: false
packages:
  - vim
  - chrony
  - iproute
  - iputils
  - tcpdump
  - traceroute
  - bind-utils

runcmd:
  - systemctl enable --now chronyd
  - systemctl disable --now firewalld
  - systemctl set-default multi-user.target
```
※ host02〜host04 作成時は<br>
`hostname` と `fqdn` をそれぞれ変更する。

### 5.3 ルータ用 user-data 作成 (host00)
```bash
vim user-data-router.yaml
```
```bash
#cloud-config

hostname: host00
fqdn: host00.knowd.co.jp

timezone: Asia/Tokyo
locale: en_US.UTF-8

users:
  - default
  - name: haru
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: [wheel]
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 ...

disable_root: true
ssh_pwauth: false

package_update: true
package_upgrade: false
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
  - systemctl enable --now chronyd
  - systemctl enable --now nftables
  - systemctl disable --now firewalld
  - systemctl set-default multi-user.target
  - echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/99-router.conf
  - sysctl --system
```

### 5.4 Cloud-init seed ISO 作成
```bash
cd /var/lib/libvirt/images/cloud-init
cloud-localds host01-seed.iso user-data-host.yaml meta-data.yaml
```
※ host02〜host04、host00 作成時は<br>
seed ISO 名 と user-data を変更する。

### 5.5 VM 作成 (host01-04, host00)
```bash
cd /var/lib/libvirt/images

# Cloud Image をベースにディスク作成
cp AlmaLinux-9-GenericCloud-latest.x86_64.qcow2 host01.qcow2

# cloud-init seed ISO 作成
cd /var/lib/libvirt/images/cloud-init
cloud-localds host01-seed.iso user-data-host.yaml meta-data.yaml

# VM 作成
virt-install \
  --name host01 \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/host01.qcow2,format=qcow2 \
  --disk path=/var/lib/libvirt/images/cloud-init/host01-seed.iso,device=cdrom \
  --os-variant almalinux9 \
  --network network=network1,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --import
```
※ host02 は network1 を指定する。<br>
※ host03・host04 は network2 を指定する。<br>
※ host00 は `--network` を 3 回指定し、`default → network1 → network2` の順で接続する。

## 6. VM 設定

## 7. 疎通確認
