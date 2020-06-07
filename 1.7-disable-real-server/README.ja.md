# 演習 1.7 - Serverの切り離しと切り戻し

## 目次

- [本演習の目的](#本演習の目的)
- [Serverを切り離すPlaybookの作成](#Serverを切り離すPlaybookの作成)
- [Serverを切り離すPlaybookの実行](#Serverを切り離すPlaybookの実行)
- [Serverを切り戻すPlaybookの作成](#Serverを切り戻すPlaybookの作成)
- [Serverを切り戻すPlaybookの実行](#Serverを切り戻すPlaybookの実行)

# 本演習の目的

本演習では、`a10_slb_server`モジュールを利用し、サーバー負荷分散の対象となっているWebサーバーs2の切り離しを切り戻しを行います。
具体的にはServerのDisable/Enableを実行します。
例えば、Webサーバーのメンテナンスが必要な場合やWebサーバーを更改する際に、Disableすることで通信を停止しサーバー負荷分散の対象から外すことで、サービスを継続したままのメンテナンスが可能になります。

# Serverを切り離すPlaybookの作成

Server s2をDisableにするために、Ansible実行用サーバーのplaybookディレクトリで、`a10_slb_server_s2_disable.yaml`という名前でPlaybookを作成します。
このPlaybookでは、Ansibleモジュールとして`a10_slb_server`を利用します。

```
[root@ansible playbook]# vi a10_slb_server_s2_disable.yaml
```

Server s2に対して`action: "disable"`を設定することでWebサーバーを負荷分散対象から切り離します。

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
  - name: Disable real server s2
    a10_slb_server:
      ansible_host: "{{ ansible_host }}"
      ansible_port: "{{ ansible_port }}"
      ansible_username: "{{ ansible_username }}"
      ansible_password: "{{ ansible_password }}"
      name: "s2"
      host: "10.0.2.12"
      action: "disable"
      state: present
```

- `action: "disable"`は、モジュールのパラメーターで、`a10_slb_server`で設定するServerをDisableにします。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# Serverを切り離すPlaybookの実行

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_server_s2_disable.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Disable real server s2] *********************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

タスクDisable real server s2でServer s2をDiableにしたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 441 bytes
!Configuration last updated at 17:03:03 IST Thu Sep 12 2019
!Configuration last saved at 16:36:47 IST Thu Sep 12 2019
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
  disable
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

slb server s2がDisableになっています。
この状態になると、s2はサーバー負荷分散対象から外されます。

ここで、`show slb virtual-server`、`show slb service-group`、`show slb server`をvThunderで実行します。

```
vThunder#show slb virtual-server
Total Number of Virtual Services configured: 1
Virtual Server Name      IP              Current    Total      Request  Response Peak
Service-Group            Service         connection connection packets  packets  connection
----------------------------------------------------------------------------------------
*vip1 10.0.1.100         Functional Up

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
*sg1                  State: Functional Up
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

s2:80/tcp                 0          2          12         8          0          Down
s2: Total                 0          2          12         8          0          Disabled

```

Server s2がDisableになっているため、Virtual-ServerもService-GroupもFunctional Upの状態になります。

クライアントからcurlでアクセスすると、Server s2がDisableになっているため、Server s1にのみトラフィックが転送されていることがわかります。

# Serverを切り戻すPlaybookの作成

Server s2を再度有効化（Enable）するために、Ansible実行用サーバーのplaybookディレクトリで、`a10_slb_server_s2_enable.yaml`という名前でPlaybookを作成します。
このPlaybookでも、Ansibleモジュールとして同じく`a10_slb_server`を利用します。

```
[root@ansible playbook]# vi a10_slb_server_s2_enable.yaml
```

Server s2に対して`action: "enable"`を設定することでWebサーバーを負荷分散対象に戻します。

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
  - name: Enable real server s2
    a10_slb_server:
      ansible_host: "{{ ansible_host }}"
      ansible_port: "{{ ansible_port }}"
      ansible_username: "{{ ansible_username }}"
      ansible_password: "{{ ansible_password }}"
      name: "s2"
      host: "10.0.2.12"
      action: "enable"
      state: present
```

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# Serverを切り戻すPlaybookの実行

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_server_s2_enable.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Enable real server s2] **********************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

タスクEnable real server s2でServer s2がEnableになったことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 441 bytes
!Configuration last updated at 17:15:09 IST Thu Sep 12 2019
!Configuration last saved at 16:36:47 IST Thu Sep 12 2019
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

slb server s2から`disable`が消えています。
この状態になると、s2はサーバー負荷分散対象に再び組み入れられます。

ここで、`show slb virtual-server`、`show slb service-group`、`show slb server`をvThunderで実行します。

```
vThunder#show slb virtual-server
Total Number of Virtual Services configured: 1
Virtual Server Name      IP              Current    Total      Request  Response Peak
Service-Group            Service         connection connection packets  packets  connection
----------------------------------------------------------------------------------------
*vip1 10.0.1.100         All Up

   port 80  http                         0          8          56       48       0
sg1                      80/http         0          8          48       32       0
Total received conn attempts on this port: 8

vThunder#show slb service-group
Total Number of Service Groups configured: 1
                   Current = Current Connections, Total = Total Connections
                   Fwd-p = Forward packets, Rev-p = Reverse packets
                   Peak-c = Peak connections
Service Group Name
Service                         Current    Total      Fwd-p     Rev-p     Peak-c
-----------------------------------------------------------------------------------
*sg1                  State: All Up
s1:80                           0          6          36        24        0
s2:80                           0          2          12        8         0

vThunder#show slb server
Total Number of Servers configured: 2
Total Number of Services configured: 2
                   Current = Current Connections, Total = Total Connections
                   Fwd-pkt = Forward packets, Rev-pkt = Reverse packets
Service                   Current    Total      Fwd-pkt    Rev-pkt    Peak-conn  State
---------------------------------------------------------------------------------------
s1:80/tcp                 0          6          36         24         0          Up
s1: Total                 0          6          36         24         0          Up

s2:80/tcp                 0          2          12         8          0          Up
s2: Total                 0          2          12         8          0          Up

```

Server s2がEnableに戻ったため、Virtual-ServerもService-GroupもAll Upの状態になります。

ここで、再度クライアントからcurlでアクセスすると、Server s1とs2に均等にトラフィックが転送されていることがわかります。

基本編の演習はここまでになります。時間があれば応用編にお進みください。

本演習は以上となります。  [トレーニングガイドに戻る](../README.ja.md)
