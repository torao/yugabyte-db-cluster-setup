# Yugabyte DB Cluster Setup for Ansible

このリポジトリは、YugabyteDB クラスタを複数ノード上に構築するための Ansible Playbook 一式です。
AlmaLinux 9 などの RHEL 系ディストリビューションで動作確認を行っています。

## Assumptions

各ノードは VirtualBox 上のゲスト OS として存在する想定です。
各ノードは、ノード間の通信用の NAT-Network `enp0s3` (yb_net) と、ホスト OS と通信するための
Host-only Network `enp0s8` (host_net) の 2 つのネットワークインターフェースを持ちます。
マシン環境の違いは [group_vars/yugabyte.yml](group_vars/yugabyte.yml) と [inventory.yml](inventory.yml)
を修正する必要があるでしょう。

## Operations

macOS, Ubuntu, WSL2 等の制御端末で以下のコマンドを実行します。

```shell
sudo apt update && sudo apt install -y ansible openssh-client sshpass wget

ansible -i inventory.yml all -m ping -u username -k
ansible-playbook -i inventory.yml init.yml -u username -k --ask-become-pass
ansible -i inventory.yml all -a "shutdown -r now" --become --ask-become-pass -u username -k
ansible -i inventory.yml all -m ping -u username -k

wget https://software.yugabyte.com/releases/2024.2.3.2/yugabyte-2024.2.3.2-b6-linux-x86_64.tar.gz

ansible-playbook -i inventory.yml yb-install.yml -u username -k
```

クラスタが起動すれば host_net の IP アドレスを使って各ノードと通信できるだろう。

http://192.168.56.xxx:7000/
