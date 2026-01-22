# KVM Network Lab (AlmaLinux)

## 概要
LPIC-102 の nmcli / ip コマンドの理解を深めるために、<br>
自宅ミニPC上の KVM を用いて仮想ネットワーク環境を構築しました。

あわせて、LPIC-202 で扱われるサーバ構築分野の学習に向けた準備として、<br>
KVM / libvirt による仮想化基盤や cloud-init を用いた VM 作成手法の学習も目的としています。

## 構成
- Host OS: AlmaLinux
- Hypervisor: KVM + libvirt
- Guest OS: Almalinux (cloud-init)
- VM:
  - host00: Router
  - host01-04: Host
- Network
  - network1: 172.16.0.0/24
  - network2: 172.17.0.0/24
  - host00(Router)が2つのネットワークを中継

## ネットワーク構成図
![network](architecture/network-diagram.drawio.png)

## 実施内容
- KVM / libvirt を用いた仮想化基盤の構築
- cloud-init による VM の自動作成・初期設定
- libvirt isolated ネットワーク（network1 / network2）の作成
- nmcli を用いた IP アドレス・デフォルトルートの設定
- ip によるインタフェース・ルーティング情報の確認
- ルータ VM（host00）における IP フォワーディングの有効化
- ping / traceroute を用いた異なるネットワーク間の疎通確認

## 苦労したこと・学び
- EC2 などでも利用される cloud-init を用いた VM 作成手順が分かった
- NAT / bridge / isolated ネットワークの違いを理解できた
  - NAT: 内部ネットワークと外部ネットワークを区別し、アドレス変換を行う（ルータなど内外の境界に設定する）
  - bridge: 同一セグメントで接続される
  - isolated: 外部ネットワークから完全に分離される
- nmcli によるネットワーク設定は永続的だが、ip は一時的で再起動で消えてしまう
  - そのため、nmcli は設定変更に、ip は状態確認に利用する
- インターフェースの設定適用に `nmcli con up eth0` を使用すると SSH 接続が切れる可能性があるので、`nmcli dev reapply eth0` を利用する
- Linux では ipv4_forwarding パラメータを有効にすることで、ルータとして動作する

## 今後の課題
- cloud-init や libvirt などドキュメントは基本的に英語で書かれているので、英語を学ぶ必要があると感じた
- 今回学んだ KVM / libvirt による仮想化基盤の構築や、cloud-init による VM 作成を、今後のサーバ構築に活かしていきたい
