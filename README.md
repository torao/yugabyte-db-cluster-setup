# YugabyteDB クラスタを構築する

このリポジトリは、動作確認やテストを想定した最小構成の YugabyteDB クラスタを迅速に構築することを目的としています。ローカルの Docker コンテナや VirtualBox のような仮想化環境、または物理的なノードを想定しています。簡便さを優先し、セキュリティは非優先ですので本番環境向きではありません。

配備されている Ansible Playbook は、レプリケーション係数 3 の Yugabyte クラスタとなる YB Master ×3 + YB TServer ×3 の 6 ノードを構成します。

## 準備するもの

次の用に構成されたノードを用意します。

- AlmaLinux 9 (または同等の RHEL 系 Linux) がインストールされている。
- SSH でリモート接続でき、`sudo` 権限を持つユーザが作成されている。
- 次の 2 つのネットワークインターフェースを持つ。
   1. `enp0s3`: `yb_net`: クラスタ内の他のノードと通信したり、`dnf` などで外部のネットワークと接続する (VirtualBox の NAT-Network)。
   2. `enp0s8`: `host_net`: Ansible を実行する構成ノード (ホスト OS) との通信に使用する (VirtualBox の Host-only Network)。

また、このノードが使用するデフォルトゲートウェイの IP アドレスを調べておいてください。

## YugabyteDB のインストール

新規に構成する場合や、新しいノードを追加する場合は、次のように行います。

1. まずは初期状態の IP アドレスを持つノードを [inventory.yml](inventory.yml) に記述します。ここで、固定する IPv4 アドレスもしています。
2. [init.yml](init.yml) を実行しします。
3. 再び [inventory.yml](inventory.yml) を編集して、接続先を固定 IP アドレスに変更します。
4. 以後は設定した固定 IP アドレスを使用します。

構成ノードから macOS, Ubuntu, WSL2 等で以下のコマンドを実行します。

```shell
# 構成ノード上で Ansible や必要なライブラリをインストールします。
sudo apt update && sudo apt install -y ansible openssh-client sshpass wget

# デプロイユーザで各ノードに接続できることを確認します。
ansible -i inventory.yml all -m ping -u yb-deploy -k

# 初期構成を行います。
ansible-playbook -i inventory.yml init.yml -u yb-deploy -k --ask-become-pass

# inventory.yml の接続先を新しい IP アドレスに変更して、再び各ノードに接続できることを確認します。
# ここで一度再起動しても良いかもしれません。
# ansible -i inventory.yml all -a "shutdown -r now" --become --ask-become-pass -u yb-deploy -k
ansible -i inventory.yml all -m ping -u yb-deploy -k

# YugabyteDB をダウンロードして各ノードにインストールを実行します。
wget https://software.yugabyte.com/releases/2024.2.3.2/yugabyte-2024.2.3.2-b6-linux-x86_64.tar.gz
ansible-playbook -i inventory.yml yb-install.yml -u yb-deploy -k -e yb_tarball=yugabyte-2024.2.3.2-b6-linux-x86_64.tar.gz -e yb_version=yugabyte-2024.2.3.2-b6

# 監視系 (Prometheus + Grafana) のインストールを実行します。
ansible-playbook -i inventory.yml monitoring.yml -u yb-deploy -k
```

クラスタが正しく起動すれば、host_net の IP アドレスを使って構成ノードから各ノードと通信できます。

- http://192.168.56.xxx:7000/ (YB Master)
- http://192.168.56.yyy:9000/ (YB TServer)
- http://192.168.56.210:9090/ (Prometheus)
- http://192.168.56.210:3000/ (Grafana; id/pass = admin/admin)

## ネットワーク構成

2 つのネットワークインターフェースは DHCP で構成されていても良いが、Ansible の実行によって最終的に固定 IP アドレスで設定されます。また、両方の機能を持つ 1 つのネットワークインターフェースのみが設置されている場合は、名前を同じに変更してもかまいません。

このリポジトリでは OS をインストールしたばかりの初期状態の環境、特に VirtualBox で新規作成したばかりのノードを想定しているため、すでにネットワーク設定などが行われている場合や、ネットワークポリシーの制約がある場合は以下のファイルを修正する必要があります。

- [group_vars/yugabyte.yml](group_vars/yugabyte.yml)
- [inventory.yml](inventory.yml)
- [init.yml](init.yml)

まずは初期状態の IP アドレスを [inventory.yml](inventory.yml) にして [init.yml](init.yml) を実行し、再び [inventory.yml](inventory.yml) を編集してそれ以降は

| ホスト名     | Host-only Network (enp0s8) | NAT Network (enp0s3) |
|:------------|:---------------|:-----------|
| yb-master1  | 192.168.56.200 | 10.0.2.200 |
| yb-master2  | 192.168.56.201 | 10.0.2.201 |
| yb-master3  | 192.168.56.202 | 10.0.2.202 |
| yb-tserver1 | 192.168.56.205 | 10.0.2.205 |
| yb-tserver2 | 192.168.56.206 | 10.0.2.206 |
| yb-tserver3 | 192.168.56.207 | 10.0.2.207 |
