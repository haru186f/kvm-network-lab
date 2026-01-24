# KVM Network Lab

## 概要
LPIC-102 の nmcli / ip コマンドの理解を深めるために、<br>
自宅ミニPC上の KVM を用いて仮想ネットワーク環境を構築しました。

あわせて、LPIC-202 で扱われるサーバ構築分野の学習に向けた準備として、<br>
KVM / QEMU / libvirt による仮想化基盤や cloud-init を用いた VM 作成手法の学習も目的としています。

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
- KVM / QEMU / libvirt を用いた仮想化基盤の構築
- cloud-init による VM の自動作成・初期設定
- libvirt isolated ネットワーク（network1 / network2）の作成
- nmcli を用いた IP アドレス・デフォルトルートの設定
- ip によるインタフェース・ルーティング情報の確認
- ルータ VM（host00）における IP フォワーディングの有効化
- ping / traceroute を用いた異なるネットワーク間の疎通確認

## 苦労したこと・学び
- KVM / QEMU / libvirt の役割を理解できた
  - KVM: Linux カーネルに組み込まれた仮想化技術（Linux を Type 1 ハイパバイザーとして動作させる）
  - QEMU: マシンエミュレーター（KVM と組み合わせることで完全仮想化環境を提供）
  - libvirt: KVM/QEMU などの仮想化基盤を統一的に管理・操作するためのインターフェース
- EC2 などでも利用される cloud-init を用いた VM 作成手順を学ぶことができた
- NAT / bridge / isolated ネットワークの違いを理解できた
  - NAT: 内部ネットワークと外部ネットワークを区別し、アドレス変換を行う（ルータなど内外の境界に設定する）
  - bridge: 同一セグメントで接続される
  - isolated: 外部ネットワークから完全に分離される（外部との通信はルータを経由する）
- nmcli によるネットワーク設定は永続的だが、ip は一時的で再起動で消えてしまう
  - そのため、nmcli は設定変更に、ip は状態確認に利用する
- インターフェースの設定適用に `nmcli con up eth0` を使用すると SSH 接続が切れる可能性があるので、`nmcli dev reapply eth0` を利用する
- Linux では `net.ipv4.ip_forward` パラメータを有効にすることで、ルータとして動作する
  - `echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf` を実行し再起動することで有効化できる
- SELinux をなぜ無効にするのかを理解できた
  - SELinux は、万が一侵入された場合に被害を最小限にする仕組みで、制御範囲は root 権限ユーザにも及ぶ
  - 設定・構築に問題がなくても、SELinux の制御によって本質的でないエラーが発生してしまう
  - 検証環境なら無効にしても問題ないが、本番環境では無効にするデメリットを理解した場合や、他で防御を行う場合に無効にすることがある
  - `sestatus` で現在のステータスを確認でき、`/etc/selinux/config` 内の `SELINUX=disabled` に変更し再起動することで無効化できる
- draw.io を用いたネットワーク構成図の作成や、Markdown による手順書の作成など、実務で使用する機会の多いツールや記法についても学ぶことができた

## 今後の課題
- cloud-init や libvirt などドキュメントは基本的に英語で書かれているので、英語を学ぶ必要があると感じた
- 今回学んだ KVM / QEMU / libvirt による仮想化基盤の構築や、cloud-init による VM 作成を、今後のサーバ構築に活かしていきたい
- Docker などのコンテナ技術についても学習し、仮想マシンとの違いや用途に応じた使い分けを理解していきたい
- Ansible などの IaC（Infrastructure as Code）ツールを使って、VM 作成や各種設定を自動化することにも挑戦してみたい
