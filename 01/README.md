Ansible を試す
==============

Qiitaにも[投稿済み](http://qiita.com/nmatayoshi/items/f64bb474d2896b45cfa3)

# 1. 環境

VagrantとVirtualBoxは事前にインストールしておいてください。

- Max OS X 10.10
- Homebrew 0.9.5
- VirtualBox 4.3.18
- Vagrant 1.6.5

# 2. Vagrant 環境の準備

~~CentOS 6.5 x86_64 を使います。~~  
CentOS Linux 6/x86_64を使います。

```bash
$ mkdir ansible_test
$ cd ansible_test
$ vagrant box add centos/6
$ vagrant init centos/6
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

# 4. Playbook の作成

簡単な Playbook を作ってみます。
yum update all と httpd をインストールします。

## イベントリファイル

先ほどの hosts ファイルを上書きします。[]でグループを定義します。

    [web_server]
    192.168.33.10
    [db_server]
    192.168.33.10

## Playbook

いわゆるChefのレシピ。

```bash
$ vi playbook.yml
```

```yml:playbook.yml
- hosts: all
  user: vagrant
  sudo: yes
  tasks:
    - name: upgrade all packages
      yum: name=* state=latest

- hosts: web_server
  user: vagrant
  sudo: yes
  tasks:
    - name: install httpd package
      yum: name={{ item }} state=present
      with_items:
        - lynx
        - httpd
    - name: Enable httpd server
      service: name=httpd state=running enabled=yes
```

- yum の state について

    | state   | 意味     |
    |---------|---------|
    | present | まだインストールされていなければインストールする。<br />インストール済みなら何もしない。(default) |
    | latest  | まだインストールされていなければインストールする。<br />インストール済みのバージョンより最新版が見つかればアップデートする。 |
    | absent  | パッケージをアンインストールする。 |

    詳細は [yum - Manages packages with the yum package manager — Ansible Documentation](http://docs.ansible.com/yum_module.html)

# 5. Playbook のテスト

## 文法チェック

```bash
$ ansible-playbook -i hosts playbook.yml --syntax-check

playbook: playbook.yml
```

エラーがなければplaybook名が出力されます。

## task一覧

指定した Playbook で実行される task の一覧が出力されます。

```bash
$ ansible-playbook -i hosts playbook.yml --list-tasks

playbook: playbook.yml

  play #1 (all):
    upgrade all packages

  play #2 (web_server):
    Install httpd package
    Enable httpd server
```

## 対象host の確認

指定した Playbook で対象となる host の一覧が出力されます。

```bash
$ ansible-playbook -i hosts playbook.yml --list-hosts

playbook: playbook.yml

  play #1 (all): host count=1
    192.168.33.10

  play #2 (web_server): host count=1
    192.168.33.10
```

# 6. Playbook の実行

## dry-run

まずは dry-run で実行時の動きを確認します。

```bash
$ ansible-playbook -i hosts playbook.yml --check

PLAY [all] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [upgrade all packages] **************************************************
ok: [192.168.33.10]

PLAY [web_server] *************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [Install httpd package] *************************************************
changed: [192.168.33.10] => (item=lynx,httpd)

TASK: [Enable httpd server] ***************************************************
failed: [192.168.33.10] => {"failed": true}
msg: cannot find 'service' binary or init script for service,  possible typo in service name?, aborting

FATAL: all hosts have already failed -- aborting

PLAY RECAP ********************************************************************
           to retry, use: --limit @/Users/nobuho/playbook.retry

192.168.33.10              : ok=4    changed=1    unreachable=0    failed=1
```

dry-run なので init.d スクリプトがインストールされず service の起動に失敗しているようです。

## 実行

```bash
$ ansible-playbook -i hosts playbook.yml

PLAY [all] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [upgrade all packages] **************************************************
ok: [192.168.33.10]

PLAY [web_server] *************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [Install httpd package] *************************************************
changed: [192.168.33.10] => (item=lynx,httpd)

TASK: [Enable httpd server] ***************************************************
changed: [192.168.33.10]

PLAY RECAP ********************************************************************
192.168.33.10              : ok=5    changed=2    unreachable=0    failed=0   
```

上手くいきました。
ちなみにもう一回実行すると……

```bash
$ ansible-playbook -i hosts playbook.yml

PLAY [all] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [upgrade all packages] **************************************************
ok: [192.168.33.10]

PLAY [web_server] *************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [Install httpd package] *************************************************
ok: [192.168.33.10] => (item=lynx,httpd)

TASK: [Enable httpd server] ***************************************************
ok: [192.168.33.10]

PLAY RECAP ********************************************************************
192.168.33.10              : ok=5    changed=0    unreachable=0    failed=0  
```

httpd などはすでにインストール済みのため何もしません。(changedが0)


長くなったので今回はここまで。  
別で serverspec によるテストとか、今回はパッケージで入れましたけどソースコードからのビルドとか設定ファイルの作成を試します。

# 参考

- [Ansible チュートリアル | Ansible Tutorial in Japanese](http://yteraoka.github.io/ansible-tutorial/)
- [docker でちょっとだけ試す Ansible - ようへいの日々精進 XP](http://inokara.hateblo.jp/entry/2014/02/26/012144)
- Software Design 2014/11号
- [yum - Manages packages with the yum package manager — Ansible Documentation](http://docs.ansible.com/yum_module.html)
