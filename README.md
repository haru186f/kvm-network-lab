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

## 学んだこと
- KVM / QEMU / libvirt の役割
  - KVM: Linux カーネルに組み込まれた仮想化技術（Linux を Type 1 ハイパバイザーとして動作させる）
  - QEMU: マシンエミュレーター（KVM と組み合わせることで完全仮想化環境を提供）
  - libvirt: KVM/QEMU などの仮想化基盤を統一的に管理・操作するためのインターフェース
- NAT / bridge / isolated ネットワークの違い
  - NAT: 内部ネットワークと外部ネットワークを区別し、アドレス変換を行う（ルータなど内外の境界に設定する）
  - bridge: 同一セグメントで接続される
  - isolated: 外部ネットワークから完全に分離される（外部との通信はルータを経由する）
- nmcli によるネットワーク設定は永続的だが、ip は一時的で再起動で消えてしまう
  - そのため、nmcli は設定変更に、ip は状態確認に利用する
- インターフェースの設定適用に `nmcli con up eth0` を使用すると SSH 接続が切れる可能性があるので、`nmcli dev reapply eth0` を利用する
- Linux は `net.ipv4.ip_forward` パラメータを有効にすることで、ルータとして動作させることができる
  - `/etc/sysctl.conf` に `net.ipv4.ip_forward = 1` を追記し、再起動することで有効化できる
- SELinux はなぜ無効にするのか
  - SELinux は、万が一侵入された場合に被害を最小限にする仕組みであり、その制御は root 権限ユーザーにも及ぶ
  - そのため、設定や構築に問題がなくても、SELinux のアクセス制御によって本質的でないエラーが発生してしまう
  - そのような理由から、検証環境では無効化することがあるが、本番環境では無効化するデメリットを理解した上や、他で防御を行う場合に限って無効化する
- draw.io によるネットワーク構成図の作成や、Markdown による手順書の作成など、実務で使用する機会の多いツールや記法について学ぶことができた

## 苦労したこと
- 最初から SSH 接続のみに限定しようとすると設定がうまくいかず、VM を何度も作り直すことになった
  - そこで、最初はパスワード認証を許可したまま作業を進め、全ての設定が完了して SSH での接続確認が取れた後に、`sshd_config` を編集してパスワード認証を無効化することで、問題なく構築できた

## 今後について
- cloud-init や libvirt などドキュメントは基本的に英語で書かれているので、英語を学ぶ必要があると感じた
- 今回学んだ KVM / QEMU / libvirt による仮想化基盤の構築や、cloud-init による VM 作成を、今後のサーバ構築に活かしていきたい
- Docker などのコンテナ技術についても学習し、仮想マシンとの違いや用途に応じた使い分けを理解していきたい
- Ansible などの IaC（Infrastructure as Code）ツールを使って、VM 作成や各種設定を自動化することにも挑戦してみたい
