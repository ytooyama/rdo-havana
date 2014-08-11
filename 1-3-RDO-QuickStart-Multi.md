#RDO Neutron Quickstart Plus マルチノード編

最終更新日: 2014/8/11

##この文書について
この文書はとりあえず2台構成のOpenStack Havana環境をGRE構成で構築する場合の手順を説明しています。
本例はNeutron関連を別のノードとして、OpenStack本体とコンピュートは同じノード上に構築する例で説明します。

この文書は以下の公開資料を元にしています。

RDO Neutron Quickstart

<http://openstack.redhat.com/Neutron-Quickstart>
<http://openstack.redhat.com/Neutron_with_existing_external_network>

##Step 0: 要件

Software:
- Red Hat Enterprise Linux (RHEL) 6.4以降
- CentOS, Scientific Linux 6.4以降
- Fedora 19

Hardware:
- CPU 3Core以上
- メモリー6GB以上
- 最低1つのNIC
※All-in-oneの構成を作る場合は、Privateネットワーク用はloインターフェイスを利用できます。

- ネットワーク
本書では次のネットワーク構成を利用します。

Private Network | Public Network
--------------  | -------------
192.168.2.0/24  | 192.168.1.0/24

- OpenStackコントローラ・コンピュートノード

eth0            | eth1(gw)
--------------  | --------------
192.168.0.100/24 | 192.168.1.100/24

- OpenStack ネットワーク(Neutron)ノード

eth0            | eth1(gw)
--------------  | --------------
192.168.0.101/24 | 192.168.1.101/24


- カーネルパラメータの設定
以下のように設定を変更します。

````
# vi /etc/sysctl.conf

net.ipv4.ip_forward = 1              #変更
net.ipv4.conf.default.rp_filter = 0  #変更

net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0

net.ipv4.conf.all.rp_filter = 0     #追記
net.ipv4.conf.all.forwarding = 1    #追記

# sysctl -e -p /etc/sysctl.conf
（設定を反映）
````

##Step 1: SELinuxの設定変更
SELinuxの設定を変更します｡

下記公式サイトのコメントにあるようにSELinuxをpermissiveに設定する<br />
↓<br />
ただしdisableにするとpackstackが想定通り動かない模様。

<http://openstack.redhat.com/SELinux_issues>

Note: Due to the quantum/neutron rename, SELinux policies are currently broken for Havana, so SELinux must be disabled/permissive on machines running neutron services, edit /etc/selinux/config to set SELINUX=permissive.

##Step 2: ソフトウェアリポジトリの追加

ソフトウェアパッケージのインストールとアップデートを行う｡
Neutron環境の構築にはSELinuxの設定変更が必要なので設定完了後、一旦再起動する｡

次のコマンドを実行:

````
# yum install -y http://rdo.fedorapeople.org/openstack-havana/rdo-release-havana.rpm
````

システムアップデートの実施:

````
# yum -y update
# reboot
````

##Step 3: Packstackおよび必要パッケージのインストール

以下のようにコマンドを実行します｡

````
# yum install -y openstack-packstack python-netaddr libguestfs-tools
````


##Step 4:アンサーファイルを生成

以下のようにコマンドを実行してアンサーファイルを作成します｡

````
# packstack --gen-answer-file=answer.txt
(answer.txtという名前のファイルを作成する場合)
````

アンサーファイルを使うことで定義した環境でOpenStackをデプロイできます｡

作成したアンサーファイルは1台のマシンにすべてをインストールする設定が行われています｡IPアドレスや各種パスワードなどを適宜設定します｡

##Step 5:アンサーファイルを自分の環境に合わせて設定

OpenStack環境を作るには最低限以下のパラメータを設定します。

- __CONFIG_NOVA_COMPUTE_HOSTSにコンピュートノードを設定__

複数のコンピュートノードを追加するにはコンマでIPアドレスを列挙します｡

- __1つ指定する例__

CONFIG_NOVA_COMPUTE_HOSTS=192.168.1.100

- __複数指定する例__

CONFIG_NOVA_COMPUTE_HOSTS=192.168.1.100,192.168.1.101

- __NeutronノードのIPアドレスを設定する__

CONFIG_NEUTRON_SERVER_HOST=192.168.1.100 #On the Controller

CONFIG_NEUTRON_L3_HOSTS=192.168.1.101

CONFIG_NEUTRON_DHCP_HOSTS=192.168.1.101

CONFIG_NEUTRON_METADATA_HOSTS=192.168.1.101

CONFIG_NEUTRON_OVS_TENANT_NETWORK_TYPE=gre

CONFIG_NEUTRON_OVS_TUNNEL_RANGES=1:1000

CONFIG_NEUTRON_OVS_TUNNEL_IF=eth1

- __NICを利用したいものに変更する__

（例）eth1がゲートウェイに接続されている場合

CONFIG_NOVA_COMPUTE_PRIVIF=eth0

CONFIG_NOVA_NETWORK_PRIVIF=eth0

CONFIG_NOVA_NETWORK_PUBIF=eth1

- __Dashboardにアクセスするパスワード__

CONFIG_KEYSTONE_ADMIN_PW=admin

- __テスト用demoユーザーとかネットワークを作らないようにする__

CONFIG_PROVISION_DEMO=n

##Step 6: Packstackを実行してOpenStackのインストール

設定を書き換えたアンサーファイルを使ってOpenStackを導入するには、次のようにアンサーファイルを指定して実行します。

````
# packstack --answer-file=/root/answer.txt
````

packstackコマンドによるデプロイが完了したら、両ノードのneutron/plugin.iniにOVSの設定が行われていることを確認します。

````
# vi /etc/neutron/plugin.ini

[OVS]
vxlan_udp_port=4789
tunnel_type=gre
tunnel_id_ranges=1:1000
tenant_network_type=gre
local_ip=192.168.1.100
enable_tunneling=True
integration_bridge=br-int
tunnel_bridge=br-tun

[AGENT]
polling_interval=2

[SECURITYGROUP]
firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
````

確認後、両ノードを再起動します。
再起動後、ovs-vsctl showでトンネルができていることを確認してください。

````
# ovs-vsctl show
54ab38b9-9762-4444-ac7a-742e8b230709
    Bridge br-int
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "tap1251d3cd-3b"
            tag: 1
            Interface "tap1251d3cd-3b"
                type: internal
        Port "qr-d3288ac2-77"
            tag: 1
            Interface "qr-d3288ac2-77"
                type: internal
        Port br-int
            Interface br-int
                type: internal
    Bridge br-tun
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "gre-1"
            Interface "gre-1"
                type: gre
                options: {in_key=flow, local_ip="192.168.1.101", out_key=flow, remote_ip="192.168.1.100"}
        Port br-tun
            Interface br-tun
                type: internal
    Bridge br-ex
        Port "qg-70b4cede-9e"
            Interface "qg-70b4cede-9e"
                type: internal
        Port "eth0"
            Interface "eth0"
        Port br-ex
            Interface br-ex
                type: internal
    ovs_version: "1.11.0"
````


##Step 7: ネットワーク設定の変更

次に外部と通信できるようにするための設定を行います。
http://openstack.redhat.com/Neutron_with_existing_external_network

###◆public用として使うNICの設定を確認
コマンドを実行して、アンサーファイルに設定したpublic用NICを確認します。
以降の手順ではeth1であることを前提として解説します。

````
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_NETWORK_PUBIF
CONFIG_NOVA_NETWORK_PUBIF=eth1
````

###◆public用として使うNICの設定ファイルを修正
packstack実行後に__ネットワークノード__ヘログインして、eth1をbr-exにつなぐように設定をします(※BOOTPROTOは設定しない)

eth1からIPアドレス、サブネットマスク、ゲートウェイの設定を削除して次の項目だけを記述し、br-exの方に設定を書き込みます｡

````
# vi /etc/sysconfig/network-scripts/ifcfg-eth1

DEVICE=eth1
ONBOOT=yes
HWADDR=xx:xx:xx:xx:xx:xx # Your eth1's hwaddr
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
NM_CONTROLLED=no
````

###◆ブリッジインターフェイスの作成
br-exにeth1のIPアドレスを設定します。

````
# vi /etc/sysconfig/network-scripts/ifcfg-br-ex

DEVICE=br-ex
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
OVSBOOTPROTO=none
OVSDHCPINTERFACES=eth1
IPADDR=192.168.1.101
NETMASK=255.255.255.0  # netmask
GATEWAY=192.168.1.1    # gateway
DNS1=8.8.8.8           # nameserver
DNS2=8.8.4.4
NM_CONTROLLED=no
````

###◆プライベートインターフェイスの設定
プライベートインターフェイスとしてanswer.txtに指定したNICの設定を変更します。loデバイスを指定した場合は特に設定変更する必要はありません。
ただし、loデバイスを指定した場合はall-in-one構成のみしか構成できません。

- answer.txtファイルの設定を確認します。

````
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_NETWORK_PRIVIF
CONFIG_NOVA_NETWORK_PRIVIF=eth0
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_COMPUTE_PRIVIF
CONFIG_NOVA_COMPUTE_PRIVIF=eth0
````

- IP設定を行います。IPアドレスとサブネットマスクの設定を行います。

````
# vi /etc/sysconfig/network-scripts/ifcfg-eth0

DEVICE=eth0
HWADDR=xx:xx:xx:xx:xx:xx # Your eth0's hwaddr
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPADDR=192.168.0.101
NETMASK=255.255.255.0
NM_CONTROLLED=no
````

ここまでできたらいったんホストを再起動します。

````
# reboot
````

以降は単体構成と同様のセットアップ手順で構築できます。
