# 演習 1.6 - HTTPのVirtual-Serverの構成

## 目次

- [本演習の目的](#本演習の目的)
- [Virtual-Serverを構成するPlaybookの作成](#Virtual-Serverを構成するPlaybookの作成)
- [Virtual-Serverを構成するPlaybookの実行](#Virtual-Serverを構成するPlaybookの実行)
- [Virtual-Serverの動作確認](#Virtual-Serverの動作確認)

# 本演習の目的

本演習では、`a10_slb_virtual_server`モジュールを利用し、サーバー負荷分散を行うVirtual-Serverの設定を行います。
設定後、クライアントからVirtual-Serverにアクセスし、Webサーバーの負荷分散が正しく動作しているかの確認を行います。

# Virtual-Serverを構成するPlaybookの作成

Virtual-Serverを設定するために、Ansible実行用サーバーのplaybookディレクトリで、`a10_slb_virtual_server_http_create.yaml`という名前でPlaybookを作成します。
このPlaybookでは、Ansibleモジュールとして`a10_slb_virtual_server`を利用します。

```
[root@ansible playbook]# vi a10_slb_virtual_server_http_create.yaml
```

Virtual-Serverに仮想IP（VIP）として10.0.1.100、HTTPでのListenポート80番を割り当て、NATプールとService-Groupを紐づけます。
構成変更後に構成の保存をするためのタスクを追加して連続実行するようにします。

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
  - name: Configure virtual server
    a10_slb_virtual_server:
      ansible_host: "{{ ansible_host }}"
      ansible_port: "{{ ansible_port }}"
      ansible_username: "{{ ansible_username }}"
      ansible_password: "{{ ansilbe_password }}"
      name: "vip1"
      ip_address: "10.0.1.100"
      port_list:
        - port_number: "80"
          protocol: "http"
          service_group: "sg1"
          pool: "p1"
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

- `name: "vip1"`は、モジュールのパラメーターで、`a10_slb_virtual_server`で設定するVirtual-Serverの名前を指定します。
- `ip_address: "10.0.1.100"`は、モジュールのパラメーターで、`a10_slb_virtual_server`で設定するVirtual-Serverの仮想IPアドレス（Virtual IP； VIP）を指定します。
- `port_list:`は、リスト形式のモジュールのパラメーターで、`a10_slb_virtual_server`で設定するVirtual-ServerがListenするポートの番号を`port_number`、プロトコルを`protocol`、紐づけるService-Groupを`service_group`、SNATのために紐づけるNATプールを`pool`で指定します。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# Virtual-Serverを構成するPlaybookの実行

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_virtual_server_http_create.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Configure virtual server] *******************************************************************************************************************
changed: [192.168.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

2つのタスクが連続実行され、1つ目のタスク（Configure　virtual server）でVirtual-Serverが構成され、2つ目のタスク（Write memory）で変更した構成が起動用の設定として保存されたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 441 bytes
!Configuration last updated at 16:30:35 IST Thu Sep 12 2019
!Configuration last saved at 16:30:38 IST Thu Sep 12 2019
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
ip nat pool p1 10.0.2.100 10.0.2.100 netmask /24
!
ip route 0.0.0.0 /0 10.0.1.254
!
slb server s1 10.0.2.11
  port 80 tcp
!
slb server s2 10.0.2.12
  port 80 tcp
!
slb service-group sg1 tcp
  member s1 80
  member s2 80
!
slb virtual-server vip1 10.0.1.100
  port 80 http
    source-nat pool p1
    service-group sg1
!
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

新たにslb virtual-serverとしてvip1が設定され、port 80/httpのトラフィックに対し、NATプールp1を使ってSNATしながら、Service-Group　sg1を用いてサーバー負荷分散する設定になっています。

ここで、`show slb virtual-server`をvThunderで実行します。

```
vThunder#show slb virtual-server
Total Number of Virtual Services configured: 1
Virtual Server Name      IP              Current    Total      Request  Response Peak
Service-Group            Service         connection connection packets  packets  connection
----------------------------------------------------------------------------------------
*vip1 10.0.1.100         All Up

   port 80  http                         0          0          0        0        0
sg1                      80/http         0          0          0        0        0
Total received conn attempts on this port: 0

```

ヘルスチェックの結果、Service-GroupがUpしているため、Virtual-ServerとしてもAll Upの状態にあることがわかります。

再度同じPlaybookを実行してみます。
```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_virtual_server_http_create.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Configure virtual server] *******************************************************************************************************************
ok: [192.168.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

Virtual-Serverを設定する部分は冪等性が保たれていることがわかります。

これで、Virtual-Serverの設定が完了しました。

# Virtual-Serverの動作確認

Virtual-Serverの動作を確認するために、クライアントからVirtual-ServerのVIPにHTTPでアクセスします。

Windows 10クライアントからは、ブラウザ（Chrome、Firefox、Edgeなど）で、http://10.0.1.100/にアクセスします。
10秒程度間隔をあけて再読み込みを実行すると、実行するたびに負荷分散されてアクセスするWebサーバーが変わることを確認できます。

vThunderで負荷分散の状況を確認するには、これまで実行してきた`show slb virtual-server`、`show slb service-group`、`show slb server`などを実行します。

```
vThunder#show slb virtual-server
Total Number of Virtual Services configured: 1
Virtual Server Name      IP              Current    Total      Request  Response Peak
Service-Group            Service         connection connection packets  packets  connection
----------------------------------------------------------------------------------------
*vip1 10.0.1.100         All Up

   port 80  http                         0          4          28       24       0
sg1                      80/http         0          4          24       16       0
Total received conn attempts on this port: 4

vThunder#show slb service-group
Total Number of Service Groups configured: 1
                   Current = Current Connections, Total = Total Connections
                   Fwd-p = Forward packets, Rev-p = Reverse packets
                   Peak-c = Peak connections
Service Group Name
Service                         Current    Total      Fwd-p     Rev-p     Peak-c
-----------------------------------------------------------------------------------
*sg1                  State: All Up
s1:80                           0          2          12        8         0
s2:80                           0          2          12        8         0

vThunder#show slb server
Total Number of Servers configured: 2
Total Number of Services configured: 2
                   Current = Current Connections, Total = Total Connections
                   Fwd-pkt = Forward packets, Rev-pkt = Reverse packets
Service                   Current    Total      Fwd-pkt    Rev-pkt    Peak-conn  State
---------------------------------------------------------------------------------------
s1:80/tcp                 0          2          12         8          0          Up
s1: Total                 0          2          12         8          0          Up

s2:80/tcp                 0          2          12         8          0          Up
s2: Total                 0          2          12         8          0          Up

```

Virtual-Serverへのアクセスがround-robinで均等に負荷分散されていることがわかります。
次の演習では、2台あるサーバーの1台を切り離した場合の動作を確認します。

本演習は以上となります。  [トレーニングガイドに戻る](../README.ja.md)
