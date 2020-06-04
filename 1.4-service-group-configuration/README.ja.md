# 演習 1.4 - Service-Groupの構成

## 目次

- [本演習の目的](#本演習の目的)
- [Service-Groupを構成するPlaybookの作成](#Service-Groupを構成するPlaybookの作成)
- [Service-Groupを構成するPlaybookの実行](#Service-Groupを構成するPlaybookの実行)

# 本演習の目的

本演習では、`a10_slb_service_group`モジュールを利用し、サーバー負荷分散の対象となるService-Groupの設定を行います。

# Service-Groupを構成するPlaybookの作成

Service-Groupを設定するために、Ansible実行用サーバーのplaybookディレクトリで、`a10_slb_service_group_create.yaml`という名前でPlaybookを作成します。
このPlaybookでは、Ansibleモジュールとして`a10_slb_service_group`を利用します。

```
[root@ansible playbook]# vi a10_slb_service_group_create.yaml
```

1つのService-Groupに2台のServerの`port 80 tcp`を割り当て、構成変更後に構成の保存をするためのタスクを追加して連続実行するようにします。

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
  - name: Configure service group
    a10_slb_service_group:
      ansible_host: "{{ ansible_host }}"
      ansible_port: "{{ ansible_port }}"
      ansible_username: "{{ ansible_username }}"
      ansible_password: "{{ ansilbe_password }}"
      name: "sg1"
      protocol: "tcp"
      lb_method: "round-robin"
      member_list:
        - name: "s1"
          port: "80"
        - name: "s2"
          port: "80"
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

- `name: "sg1"`は、モジュールのパラメーターで、`a10_slb_service_group`で設定するService-Groupの名前を指定します。
- `protocol: "tcp"`は、モジュールのパラメーターで、`a10_slb_service_group`で設定するService-Groupのプロトコルを指定します。
- `lb_method: "round-robin"`は、モジュールのパラメーターで、`a10_slb_service_group`で設定するService-Groupの負荷分散方法を指定します。ここではラウンドロビン形式を指定しています。
- `member_list:`は、リスト形式のモジュールのパラメーターで、`a10_slb_service_group`で設定するService-Groupの負荷分散対象とするServer名を`name`、ポート番号を`port`で指定します。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# Service-Groupを構成するPlaybookの実行

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_service_group_create.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Configure service group] ********************************************************************************************************************
changed: [192.168.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

2つのタスクが連続実行され、1つ目のタスク（Configure　service group）でService-Groupが構成され、2つ目のタスク（Write memory）で変更した構成が起動用の設定として保存されたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 385 bytes
!Configuration last updated at 15:44:51 IST Thu Sep 12 2019
!Configuration last saved at 15:44:54 IST Thu Sep 12 2019
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
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

新たにslb service-groupとしてsg1が設定され、tcpプロトコルのトラフィックに対し、Server s1とs2のport 80に負荷分散する設定になっています。

ここで、`show slb service-group`をvThunderで実行します。

```
vThunder#show slb service-group
Total Number of Service Groups configured: 1
                   Current = Current Connections, Total = Total Connections
                   Fwd-p = Forward packets, Rev-p = Reverse packets
                   Peak-c = Peak connections
Service Group Name
Service                         Current    Total      Fwd-p     Rev-p     Peak-c
-----------------------------------------------------------------------------------
*sg1                  State: All Up
s1:80                           0          0          0         0         0
s2:80                           0          0          0         0         0
```

ヘルスチェックの結果、どちらのサーバーもそのポートもStateがUpになっているため、Service-GroupとしてもAll Upの状態にあることがわかります。

再度同じPlaybookを実行してみます。
```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_service_group_create.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Configure service group] ********************************************************************************************************************
ok: [192.168.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

Service-Groupを設定する部分は冪等性が保たれていることがわかります。

これで、Service-Groupの追加が完了しました。
次の演習では、サーバー負荷分散のためのNAT poolの設定を行います。

本演習は以上となります。  [トレーニングガイドに戻る](../README.ja.md)
