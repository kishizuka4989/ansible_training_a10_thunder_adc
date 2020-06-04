# 演習 1.2 - データポートのenableとルーティングの設定

## 目次

- [本演習の目的](#本演習の目的)
- [データポートをenableするPlaybookの作成](#データポートをenableするPlaybookの作成)
- [データポートをenableするPlaybookの実行](#データポートをenableするPlaybookの実行)
- [スタティックルートを設定するPlaybookの作成](#スタティックルートを設定するPlaybookの作成)
- [スタティックルートを設定するPlaybookの実行](#スタティックルートを設定するPlaybookの実行)
- [クライアントとWebサーバーからのvThunderへの疎通確認](#クライアントとWebサーバーからのvThunderへの疎通確認)

# 本演習の目的

本演習では、`acos_command`モジュールを利用してvThunderのデータポートを利用可能にする（enableする）設定と、`a10_ip_route_rib`モジュールを利用してvThunderのスタティックルートの設定を行います。
データポートをenableにすることで、クライアントとVE 10、WebサーバーとVE 20との間でネットワークの疎通確認をすることができます。

データポートがenableになっていないことを確認するには、vThunderで以下のCLIコマンドを実行します。

```
vThunder#show interfaces brief
Port    Link  Dupl  Speed  Trunk Vlan MAC             IP Address          IPs  Name
------------------------------------------------------------------------------------
mgmt    Up    Full  1000   N/A   N/A  2cc2.6064.4ec4  192.168.0.1/24        1
1       Disb  None  None   none  10   2cc2.603c.b0de  0.0.0.0/0             0
2       Disb  None  None   none  20   2cc2.6044.0a45  0.0.0.0/0             0
ve10    Down  N/A   N/A    N/A   10   2cc2.603c.b0de  10.0.1.1/24           1
ve20    Down  N/A   N/A    N/A   20   2cc2.6044.0a45  10.0.2.1/24           1

Global Throughput:0 bits/sec (0 bytes/sec)
Throughput:0 bits/sec (0 bytes/sec)
```

Port 1も2もDisb（disable）になっており、ve10もve20もDownしていることが確認できます。


# データポートをenableするPlaybookの作成

vThunderのデータポートを利用可能にするために、Ansible実行用サーバーのplaybookディレクトリで、`a10_interfaces_ethernet_enable.yaml`という名前でPlaybookを作成します。
この操作がaXAPIでPUTを利用するコマンドに相当することから、aXPIをベースとするモジュールではこの操作がサポートされていません（2020年6月現在）。
そのため、このPlaybookではSSHでvThunderに接続してCLIでコマンドを実行する`acos_command`を利用します。
CLIを利用するモジュールで利用する変数が異なることから、新たなインベントリファイルを`hosts_cli`として作成します。

```
[root@ansible playbook]# vi hosts_cli
```

`hosts_cli`に以下の内容を記述し保存します。
```
[vThunder]
192.168.0.1

[vThunder:vars]
ansible_connection=network_cli
ansible_user=admin
ansible_password=a10
ansible_network_os=acos
ansible_become_password=""
```

`[vThunder:vars]`にSSHを利用するモジュールでのPlaybook全般で利用する以下の変数を記述しています。
- ansible_connection: vThunderへのアクセスの仕方（SSH）を指定します。Ansibleでネットワーク機器を操作する際に共通で利用される`network_cli`を利用します。
- ansible_user: vThunderにSSHでアクセスするためのユーザー名
- ansible_password: vThunderにSSHでアクセスするためのパスワード
- ansible_network_os: Ansibleでどのネットワーク機器を操作するかを指定します。A10の機器はACOSというOSを利用しているので`acos`を指定します。
- ansible_become_password: vThunderでConfigモードに入る際のパスワードを指定します。本演習ではパスワードを設定していないので空文字にしています。

`[vThunder]`にはPlaybookで操作対象にするThunder（本演習では1台のみ）のIPアドレスを記述しています。

次に、`a10_interfaces_ethernet_enable.yaml`という名前で`acos_command`を利用するPlaybookを作成します。

```
[root@ansible playbook]# vi a10_interfaces_ethernet_enable.yaml
```

vThunderの持つ2つのデータポートを連続してenableし、設定を保存にするようにPlaybookを記述します。

``` 
---
- hosts: 192.168.0.1
  connection: local
  gather_facts: no
  become: True
  collections:
    - a10.acos_cli

  tasks:
    - name: Enable Interface Ethernet
      acos_command:
        commands:
          - command: "configure"
          - command: "interface ethernet 1"
          - command: "enable"
          - command: "exit"
          - command: "interface ethernet 2"
          - command: "enable"
          - command: "exit"
          - command: "write memory all-partitions"
```

- `commands:`は、モジュールのパラメーターで、`command:`で設定するCLIコマンドを上から順番に実施します。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# データポートをenableするPlaybookの実行

このPlaybookを実行すると、以下のようになります。`hosts_cli`をインベントリファイルとして利用することにご注意ください。

```
[root@ansible playbook]# ansible-playbook -i hosts_cli a10_interfaces_ethernet_enable.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Enable Interface Ethernet] *****************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

タスクが実行され、ethernet 1と2がenableになり、write memoryで変更した構成が起動用の設定として保存されたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 385 bytes
!Configuration last updated at 14:44:38 IST Thu Sep 12 2019
!Configuration last saved at 14:44:41 IST Thu Sep 12 2019
!64-bit Advanced Core OS (ACOS) version 4.1.4-GR1, build 78 (Jan-18-2019,16:02)
!
multi-config enable
!
terminal idle-timeout 0
!
vlan 10
  untagged ethernet 1
  router-interface ve 10
!
vlan 20
  untagged ethernet 2
  router-interface ve 20
!
interface management
  ip address 192.168.0.1 255.255.255.0
!
interface ethernet 1
  enable
!
interface ethernet 2
  enable
!
interface ve 10
  ip address 10.0.1.1 255.255.255.0
!
interface ve 20
  ip address 10.0.2.1 255.255.255.0
!
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

interface ethernet 1と2がenableになっています。

ここで、CLIコマンド`show interfaces brief`をvThunderで実行します。

```
vThunder#show interfaces brief
Port    Link  Dupl  Speed  Trunk Vlan MAC             IP Address          IPs  Name
------------------------------------------------------------------------------------
mgmt    Up    Full  1000   N/A   N/A  2cc2.6064.4ec4  192.168.0.1/16        1
1       Up    Full  10000  none  10   2cc2.603c.b0de  0.0.0.0/0             0
2       Up    Full  10000  none  20   2cc2.6044.0a45  0.0.0.0/0             0
ve10    Up    N/A   N/A    N/A   10   2cc2.603c.b0de  10.0.1.1/24           1
ve20    Up    N/A   N/A    N/A   20   2cc2.6044.0a45  10.0.2.1/24           1

Global Throughput:0 bits/sec (0 bytes/sec)
Throughput:0 bits/sec (0 bytes/sec)
```
ethernet 1と2がUpになり、ve10とve20もUpになっていることが確認できます。

再度同じPlaybookを実行してみます。
```
[root@ansible playbook]# ansible-playbook -i hosts_cli a10_interfaces_ethernet_enable.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Enable Interface Ethernet] ******************************************************************************************************************
ok: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

冪等性が保たれていることがわかります。

# スタティックルートを設定するPlaybookの作成

実習の環境では、クライアントとvThunderとの間にルーターがあるため、クライアント側に通信を戻す場合にはルーターに対するスタティックルートを設定しないとクライアントに通信が正しく戻りません。
このスタティックルートを設定にするために、Ansible実行用サーバーのplaybookディレクトリで、`a10_ip_route_rib_create.yaml`という名前でPlaybookを作成します。

```
[root@ansible playbook]# vi a10_ip_route_rib_create.yaml
```

スタティックルートを設定して設定を保存するようにPlaybookを記述します。

``` 
---
- hosts: 192.168.0.1
  connection: local
  gather_facts: no
  collections:
    - a10.acos_axapi

  vars:
    ansible_host: "192.168.0.1"
  tasks:
  - name: Configure Static Route
    a10_ip_route_rib:
      ansible_host: "{{ ansible_host }}"
      ansible_port: "{{ ansible_port }}"
      ansible_username: "{{ ansible_username }}"
      ansible_password: "{{ ansilbe_password }}"
      ip_dest_addr: "0.0.0.0"
      ip_mask: "/0"
      ip_nexthop_ipv4:
        - ip_next_hop: "10.0.1.254"
      state: present

  - name: Write memory
    a10_write_memory:
      ansible_host: "{{ ansible_host }}"
      ansible_port: "{{ ansible_port }}"
      ansible_username: "{{ ansible_username }}"
      ansible_password: "{{ ansilbe_password }}"
      state: present
      partition: all
```

- `ip_dest_addr: "0.0.0.0"`は、宛先アドレス帯を指定するパラメーターで、`"0.0.0.0"`は全ての宛先アドレス帯を指します。
- `ip_mask:`は、宛先アドレス帯のサブネットマスクを指定するパラメーターで、`"/0"`で全てのサブネットマスクを指します。
- `ip_nexthop_ipv4:`は、リスト形式のパラメーターで、`ip_next_hop:`で宛先のルーターを指定します。ここでは、`"10.0.1.254"`として、クライアントとvThunderの間にあるルーターを指すようにしています。

上記の設定により、Webサーバー側（10.0.2.0/24）以外への通信は全て10.0.1.254に転送されるようになります。
ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# スタティックルートを設定するPlaybookの実行

このPlaybookを実行すると、以下のようになります。ここでは`hosts`をインベントリファイルとして利用することにご注意ください。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_ip_route_rib_create.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Configure Static Route] *********************************************************************************************************************
changed: [192.168.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

タスクが実行され、スタティックルートが設定されて、write memoryで変更した構成が起動用の設定として保存されたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 385 bytes
!Configuration last updated at 14:44:38 IST Thu Sep 12 2019
!Configuration last saved at 14:44:41 IST Thu Sep 12 2019
!64-bit Advanced Core OS (ACOS) version 4.1.4-GR1, build 78 (Jan-18-2019,16:02)
!
multi-config enable
!
terminal idle-timeout 0
!
vlan 10
  untagged ethernet 1
  router-interface ve 10
!
vlan 20
  untagged ethernet 2
  router-interface ve 20
!
interface management
  ip address 192.168.0.1 255.255.255.0
!
interface ethernet 1
  enable
!
interface ethernet 2
  enable
!
interface ve 10
  ip address 10.0.1.1 255.255.255.0
!
interface ve 20
  ip address 10.0.2.1 255.255.255.0
!
!
ip route 0.0.0.0 /0 10.0.1.254
!
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

スタティックルートの部分が設定されていることがわかります。
（注： このモジュールは2020年6月現在冪等性が保たれていません）

# クライアントとWebサーバーからのvThunderへの疎通確認

データポートが利用可能になったので、クライアントからvThunderへの疎通確認を行うことができます。
Windows10のクライアントにログインし、vThunderのve10（10.0．1.1）に対してコマンドプロンプトから`ping`を実行すると応答が返ってくるので、疎通を確認できます。

```
C:>ping 10.0.1.1

10.0.1.1 に ping を送信しています 32 バイトのデータ:
10.0.1.1 からの応答: バイト数 =32 時間 =131ms TTL=63
10.0.1.1 からの応答: バイト数 =32 時間 =104ms TTL=63
10.0.1.1 からの応答: バイト数 =32 時間 =102ms TTL=63
10.0.1.1 からの応答: バイト数 =32 時間 =81ms TTL=63

10.0.1.1 の ping 統計:
    パケット数: 送信 = 4、受信 = 4、損失 = 0 (0% の損失)、
ラウンド トリップの概算時間 (ミリ秒):
    最小 = 81ms、最大 = 131ms、平均 = 104ms
```

同様に、Webサーバーにログインし、vThunderのve20（10.0．2.1）に対して`ping`を実行すると応答が返ってくるので、疎通を確認できます。

```
[root@server01 ~]# ping 10.0．2.1
PING 10.0．2.1 (10.0．2.1) 56(84) bytes of data.
64 bytes from 10.0．2.1: icmp_seq=1 ttl=64 time=23.0 ms
64 bytes from 10.0．2.1: icmp_seq=2 ttl=64 time=17.2 ms
64 bytes from 10.0．2.1: icmp_seq=3 ttl=64 time=15.0 ms
64 bytes from 10.0．2.1: icmp_seq=4 ttl=64 time=13.0 ms
^C
--- 10.0．2.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 13.073/17.118/23.055/3.740 ms
```

vThunderからも`ping`を実行すると応答が返ってくるので、疎通を確認できます（Windows 10クライアントは応答を返さない設定になっていますのでご注意ください）。

```
vThunder#ping 10.0.2.11
PING 10.0.2.11 (10.0.2.11) 56(84) bytes of data.
64 bytes from 10.0.2.11: icmp_seq=1 ttl=64 time=15.5 ms
64 bytes from 10.0.2.11: icmp_seq=2 ttl=64 time=21.9 ms
64 bytes from 10.0.2.11: icmp_seq=3 ttl=64 time=12.9 ms
64 bytes from 10.0.2.11: icmp_seq=4 ttl=64 time=15.6 ms
64 bytes from 10.0.2.11: icmp_seq=5 ttl=64 time=17.7 ms

--- 10.0.2.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 12.916/16.758/21.935/3.009 ms
```

これで、データポートのenableが完了しました。
次の演習では、サーバー負荷分散のためのServerの設定を行います。

本演習は以上となります。  [トレーニングガイドに戻る](../README.ja.md)
