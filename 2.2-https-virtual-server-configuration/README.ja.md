# 演習 2.2 - HTTPSのVirtual-Serverの構成

## 目次

- [本演習の目的](#本演習の目的)
- [Virtual-Serverを構成するPlaybookの作成](#Virtual-Serverを構成するPlaybookの作成)
- [Virtual-Serverを構成するPlaybookの実行](#Virtual-Serverを構成するPlaybookの実行)
- [Virtual-Serverの動作確認](#Virtual-Serverの動作確認)

# 本演習の目的

本演習では、既存のHTTP通信に対するサーバー負荷分散に加えて、`a10_slb_virtual_server`モジュールを利用し、HTTPS通信に対してSSL/TLSを終端し、サーバー負荷分散を行うVirtual-Serverの設定を追加します。
設定後、クライアントからVirtual-Serverにアクセスし、HTTPS通信の終端と負荷分散が正しく動作しているかの確認を行います。

本構成では、クライアントとvThunder間ではHTTPS通信、vThunderとWebサーバー間ではHTTP通信を行う構成になります。

# Virtual-Serverを構成するPlaybookの作成

Virtual-Serverを設定するために、Ansible実行用サーバーのplaybookディレクトリで、`a10_slb_virtual_server_http_https_create.yaml`という名前でPlaybookを作成します。
このPlaybookでは、Ansibleモジュールとして`a10_slb_virtual_server`を利用します。

```
[root@ansible playbook]# vi a10_slb_virtual_server_http_https_create.yaml
```

既存のVirtual-Serverの設定に加えて、HTTPSでのListenポート443番を割り当て、NATプールとService-Group、SSL/TLSテンプレートを紐づけます。
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
  - name: Configure virtual server and HTTPS port
    a10_slb_virtual_server:
      ansible_host: "{{ ansible_host }}"
      ansible_port: "{{ ansible_port }}"
      ansible_username: "{{ ansible_username }}"
      ansible_password: "{{ ansible_password }}"
      name: "vip1"
      ip_address: "10.0.1.100"
      port_list:
        - port_number: "80"
          protocol: "http"
          service_group: "sg1"
          pool: "p1"
        - port_number: "443"
          protocol: "https"
          service_group: "sg1"
          pool: "p1"
          template_client_ssl: "ssl1"
      state: present

  - name: Write memory
    a10_write_memory:
      ansible_host: "{{ ansible_host }}"
      ansible_port: "{{ ansible_port }}"
      ansible_username: "{{ ansible_username }}"
      ansible_password: "{{ ansible_password }}"
      state: present
      partition: all
```

- `port_list:`は、リスト形式のモジュールのパラメーターで、`a10_slb_virtual_server`で設定するVirtual-ServerがListenするポートでSSL/TLSを終端するために紐づけるSSL/TLSテンプレートを`template_client_ssl`で指定します。

既存のport_list設定（`port 80 http`）を残す場合は、必ずPlaybookに記述する必要があります。
記述がない場合は、既存のport設定が消えて新規の（Playbook内に記述された）port設定のみが設定される形になります。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# Virtual-Serverを構成するPlaybookの実行

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_virtual_server_http_https_create.yaml

PLAY [192.168.0.1] ********************************************************************************************************************************

TASK [Configure virtual server and HTTPS port] ****************************************************************************************************
changed: [192.168.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [192.168.0.1]

PLAY RECAP ****************************************************************************************************************************************
192.168.0.1                : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

2つのタスクが連続実行され、1つ目のタスク（Configure virtual server and HTTPS port）でHTTPとHTTPSの双方でListenするVirtual-Serverが構成され、2つ目のタスク（Write memory）で変更した構成が起動用の設定として保存されたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 508 bytes
!Configuration last updated at 18:31:04 IST Thu Sep 12 2019
!Configuration last saved at 18:31:07 IST Thu Sep 12 2019
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
slb template client-ssl ssl1
  cert server.crt
  key server.key
!
slb virtual-server vip1 10.0.1.100
  port 80 http
    source-nat pool p1
    service-group sg1
  port 443 https
    source-nat pool p1
    service-group sg1
    template client-ssl ssl1
!
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

新たに`slb virtual-server`に、port 443/httpsのトラフィックに対してSSL/TLSテンプレートssl1を使ってSSL/TLS通信を終端し、NATプールp1を使ってSNATしながら、Service-Group sg1を用いてサーバー負荷分散する設定が追加されています。

ここで、`show slb virtual-server`をvThunderで実行します。

```
vThunder# show slb virtual-server
Total Number of Virtual Services configured: 2
Virtual Server Name      IP              Current    Total      Request  Response Peak
Service-Group            Service         connection connection packets  packets  connection
----------------------------------------------------------------------------------------
*vip1 10.0.1.100         All Up

   port 80  http                         0          12         84       72       0
sg1                      80/http         0          12         72       48       0
Total received conn attempts on this port: 12

   port 443  https                       0          0          0        0        0
sg1                      443/https       0          12         72       48       0
Total received conn attempts on this port: 0

```

ヘルスチェックの結果、Service-GroupがUpしているため、ポート80と443ともに動作しており、Virtual-ServerとしてもAll Upの状態にあることがわかります。

これで、Virtual-Serverの設定が完了しました。

# Virtual-Serverの動作確認

Virtual-Serverの動作を確認するために、クライアントからVirtual-ServerのVIPにHTTPSでアクセスします。

CentOSクライアントからは、curlで`https://10.0.1.100/`にアクセスします。
サーバー証明書に自己証明書を用いているため、curlの証明書の正当性に関する警告を無視するオプションとして-kを付与してください。

```
-bash-4.2$ curl -k https://10.0.1.100/
<HTML>
<HEAD>
<TITLE>A10 Networks AX Training</TITLE>
</HEAD>
<BODY>
<h1>It works!</h1>
<h2>You are on Server S1</h2>
<p></p>
</body>
</html>
-bash-4.2$ curl -k https://10.0.1.100/
<HTML>
<HEAD>
<TITLE>A10 Networks AX Training</TITLE>
</HEAD>
<BODY>
<h1>It works!</h1>
<h2>You are on Server S2</h2>
<p></p>
</body>
</html>
-bash-4.2$
```

ブラウザ（Firefox）で、`https://10.0.1.100/`にアクセスすることもできます。
サーバー証明書に自己証明書を用いているため、初回のアクセスでは警告が出ますが、警告を無視してアクセスしてください。
ブラウザを再起動して再読み込みを実行すると、実行するたびに負荷分散されてアクセスするWebサーバーが変わることを確認できます。

vThunderで負荷分散の状況を確認するには、`show slb virtual-server`、`show slb service-group`、`show slb server`などを実行します。

以上で全ての演習が終了となります。ありがとうございました。

[トレーニングガイドに戻る](../README.ja.md)
