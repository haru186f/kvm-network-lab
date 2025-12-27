# KVM Network Lab (AlmaLinux)

## 概要
LPIC-102の nmcli / ip コマンドの理解を深めるために、
自宅ミニPC上のKVMを使って仮想ネットワーク環境を構築しました。

## 構成
- Host OS: AlmaLinux
- Hypervisor: KVM + libvirt
- Guest OS: Almalinux (cloud-init)
- VM:
  - host00: Router
  - host01-04: Host
- Network
  - network1: 172.16.0.0/16
  - network2: 172.17.0.0/16
  - host00(Router)が2つのネットワークを中継

## ネットワーク構成図
![network](architecture/network-diagram.drawio.png)


## 実施内容
- KVM / QEMU / libvirt を用いた仮想化基盤の構築
- cloud-init を利用したVMの作成
- libvirt NAT ネットワークの作成（network1 / network2）
- nmcli による IP アドレスおよびデフォルトルートの設定
- ルータ VM における IP フォワーディングの有効化
- ping / traceroute を用いた異なるネットワーク間での疎通確認

## 苦労したこと・学び


## 今後の課題
