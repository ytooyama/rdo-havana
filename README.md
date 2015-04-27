rdo-havana
==========

###これはなに
RDO packstackでOpenStack Havanaの色々な環境を作る手順書のようなものです。
Havanaについてはもうこれ以上更新しません。

###環境について
以下の環境でpackstackコマンドを実行して環境を作ります。OpenStackの中ではそのほかのLinuxも動作します。

- RHEL 6.4以降
- CentOS 6.4以降
- Scientific Linux 6.4以降

以下はRDO公式サイトでは動作すると書かれていますが、2014/9/12時点で動作しません。
- Fedora 19

以下は標準リポジトリーにパッケージがありますが、[horizonの問題](https://bugzilla.redhat.com/show_bug.cgi?id=1108333)と、ファイアウォールがうまく開かない問題、Nova-Computeの追加の処理がうまく行われないなどの問題があり、2014/9/12時点で動作しません。
- Fedora 20 (標準リポジトリーのパッケージを利用)

###RDOってなに？

[公式サイト](http://jp-redhat.com/openstack/rdo/)をご覧ください。
