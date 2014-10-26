AnsibleでDockerのコンテナを構築する
===================================

Qiitaにも[投稿済み](http://qiita.com/nmatayoshi/items/58183cba4d98a151b806)です。

今回は Ansible を使って Docker 上のコンテナの環境を構築します。
Macの環境に Boot2Docker を使って構築しても良いのですが、今回は Vagrant 内に Docker 環境を構築することにします。

※1 ~ 3は[前回](../01/)と一緒です。

# 1. 環境

VagrantとVirtualBoxは事前にインストールしておいてください。

- Max OS X 10.10
- Homebrew 0.9.5
- VirtualBox 4.3.18
- Vagrant 1.6.5

# 2. Vagrant 環境の準備

CentOS 6.5 x86_64 を使います。

```bash
$ mkdir ansible_test
$ cd ansible_test
$ vagrant box add matayoshi/centos6.5.x86_64
$ vagrant init matayoshi/centos6.5.x86_64
$ vi Vagrantfile
$ diff Vagrantfile Vagrantfile.org 
27c27
<   config.vm.network "private_network", ip: "192.168.33.10"
---
>   # config.vm.network "private_network", ip: "192.168.33.10"
$ vagrant up
$ vagrant ssh-config --host 192.168.33.10 >> ~/.ssh/config
```

# 3. Ansible をインストール

```bash
$ brew install ansible
$ ansible --version
ansible 1.7.2
```

## ゲスト環境への接続確認

```bash
$ echo "192.168.33.10" > hosts
$ ansible -i hosts 192.168.33.10 -m ping
192.168.33.10 | success >> {
    "changed": false, 
    "ping": "pong"
}
```

# 4. Docker のインストール

Playbook については [github](https://github.com/matayoshi/ansible_tutorial/tree/master/02) にありますので、そちらを確認して下さい。
今回は role を使って Playbook を構成してます。

```bash
$ cat <<_EOT_ > hosts
[docker_host]
192.168.33.10
_EOT_
```

```bash
$ ansible-playbook -i hosts docker.yml
```

仮想環境の CentOS 内に Docker のインストールとイメージの build が出来ました。
build したイメージに対して[前回](../01/)の Playbook と[同等のもの](docker/playbook.yml)を実行します。

# 5. Docker に対して Ansible を実行する

CentOS 内にログインします。

```bash
$ vagrant ssh
```

Host の Playbook は /vagrant の下にマウントされてますので、そちらに移動します。

```bash
$ cd /vagrant/docker
```

Playbook で作成されている docker のイメージを実行します。

```bash
$ docker run -d --name web_test docker/ansible
```

実行した Docker のIPアドレスを調べます。

```bash
$ docker inspect -f "{{ .NetworkSettings.IPAddress }}" web_test
172.17.0.15
```

この結果は環境によって異なります。
このIPアドレスを Inventory File に書き出します。

```bash
$ cat <<_EOT_ > hosts
[web_server]
172.17.0.15
_EOT_
```

もしくは先ほどのコマンドの結果を直接流し込みます。

```bash
$ cat <<_EOT_ > hosts
[web_server]
$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" web_test)
_EOT_
$ cat hosts
[web_server]
172.17.0.15
```

Ansible を実行します。

```bash
$ export ANSIBLE_HOST_KEY_CHECKING=False
$ ansible-playbook -i hosts playbook.yml
 [WARNING]: The version of gmp you have installed has a known issue regarding
timing vulnerabilities when used with pycrypto. If possible, you should update
it (ie. yum update gmp).


PLAY [all] ******************************************************************** 

GATHERING FACTS *************************************************************** 
ok: [172.17.0.30]

TASK: [upgrade all packages] ************************************************** 
ok: [172.17.0.30]

PLAY [web_server] ************************************************************* 

GATHERING FACTS *************************************************************** 
ok: [172.17.0.30]

TASK: [Install httpd package] ************************************************* 
changed: [172.17.0.30] => (item=lynx,httpd)

TASK: [Enable httpd server] *************************************************** 
changed: [172.17.0.30]

PLAY RECAP ******************************************************************** 
172.17.0.30                : ok=5    changed=2    unreachable=0    failed=0   
```

`export ANSIBLE_HOST_KEY_CHECKING=False` については Ansible を実行する前に一度 docker に ssh でログインしておけば不要です。

```bash
$ ssh docker@172.17.0.15 -i ~/.ssh/id_rsa
The authenticity of host '172.17.0.15 (172.17.0.15)' can't be established.
RSA key fingerprint is
xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.17.0.15' (RSA) to the list of known hosts.
```

はい。docker に対して Ansible を実行できました!
別で serverspec によるテストとか、今回はパッケージで入れましたけどソースコードからのビルドとか設定ファイルの作成を試します。


# 参考

- [Ansibleのroleを使いこなす - Qiita](http://qiita.com/hnakamur/items/63f2d94badf89246e04a)
- [Playbook Roles and Include Statements — Ansible Documentation](http://docs.ansible.com/playbooks_roles.html#roles)
- [Ansible でファイル内に変数を使う場合は copy ではなく template で。 - べにやまぶろぐ](http://beniyama.hatenablog.jp/entry/2014/05/28/004953)
- [docker上のコンテナをansibleで構成管理する - とあるSEの学習日記](http://cross-black777.hatenablog.com/entry/2014/06/15/163130)
- [Docker で SSH 接続可能なコンテナ (CentOS) を作成する - Qiita](http://qiita.com/comutt/items/1251cc19885947cd6d3d)
