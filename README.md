study-ansible
=====

# 準備
## インストール
-  VirtualBox
  - https://www.virtualbox.org/wiki/Downloads
- Vagrant
  - https://www.vagrantup.com/downloads.html
- packer
  - brew tap homebrew/binary
  - brew install packer
- ansible
  - brew install ansible

## boxファイル作成
```sh
wget https://github.com/shiguredo/packer-templates/archive/develop.zip
unzip develop.zip
rm develop.zip
cd packer-templates-develop/ubuntu-14.04/
packer build -only=virtualbox-iso -parallel=true template.json
box add ubuntu-14.04 ./ubuntu-14-04-x64-virtualbox.box
vagrant box list
```

## 仮想マシン起動
```sh
cd study-ansible/tutorial
vagrant init ubuntu-14.04
vagrant up
```

## 仮想マシンの操作おさらい
```sh
# 確認
vagrant status
# 停止
vagrant halt
# 一時停止
vagrant suspend
# 再開
vagrant resume
# 破棄
vagrant destroy
# 設定再読込
vagrant reload
```

## saharaプラグインのインストール
```sh
# プラグインのインストール
vagrant plugin install sahara
# プラグインの一覧
vagrant plugin list
# プラグインのアンインストール
vagrant plugin uninstall sahara
# Sandboxモード開始
vagrant sandbox on
# 状態確認
vagrant sandbox status
# ロールバック
vagrant sandbox rollback
# コミット
vagrant sandbox commit
# Sandboxモード終了
vagrant sandbox off
```


# ansible入門

- http://www.slideshare.net/iwashi86/chefansibleansible-50941335
- http://yteraoka.github.io/ansible-tutorial/
  - node1にansibleをいれて、node2にwordpressのサーバを立てる
```
[node1] --> [node2]
```

```Vagrantfile
config.vm.define :node1 do |node|
  node.vm.box = "ubuntu-14.04"
  node.vm.network :forwarded_port, guest: 22, host: 2001, id: "ssh"
  node.vm.network :private_network, ip: "192.168.33.11"
end

config.vm.define :node2 do |node|
  node.vm.box = "ubuntu-14.04"
  node.vm.network :forwarded_port, guest: 22, host: 2002, id: "ssh"
  node.vm.network :forwarded_port, guest: 80, host: 8000, id: "http"
  node.vm.network :private_network, ip: "192.168.33.12"
end
```


```
vagrant up
```

## Ansible で node1 から node2 へ ssh するため、Vagrant 用の秘密鍵をコピー

- node1の`ssh_config`を使って`(scp -F ssh_config)`、ssh2の秘密鍵をnode1にscp

```
vagrant ssh-config node1 > ssh_config
scp -F ssh_config .vagrant/machines/node2/virtualbox/private_key node1:.ssh/id_rsa
```


## node1にansibleをインストール
```
vagrant ssh node1
sudo apt-get install -y software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install -y ansible
```

## node1からnode2を操作してみる
```sh
vagrant ssh node1
ansible 192.168.33.12 -m ping # エラー、inventryの指定が必要
```

## inventryとは
- 実行対象のホストを指定するファイル
- ここに書いたホスト以外ansibleは操作しない
- `-i`で指定できる
- ini形式
- デフォルトは`/etc/ansible/hosts`
- デフォルトのinventryの指定は`ansible.cfg`に書いてある

```ini
# シングルホスト
green.example.com
192.168.100.1

# グループ名をつけてまとめて処理を実行できる
[webservers]
alpha.example.org
beta.example.org
192.168.1.100
192.168.1.110

# 簡略表記
www[001:006].example.com

```

## ansible.cfgとは
- ansibleの設定
- 探す順
  - 1. カレントディレクトリのansible.cnf
  - 2. 環境変数の `ANSIBLE_CONFIG` or `~/ansible.cfg`
  - 3. `/etc/ansible/ansible.cfg`


## node1からnode2をもう一度操作してみる
```sh
echo 192.168.33.12 > hosts

ansible -i hosts 192.168.33.12 -m ping
192.168.33.12 | success >> {
    "changed": false,
    "ping": "pong"
}

ansible -i hosts 192.168.33.12 -a 'uname -r'
192.168.33.12 | success | rc=0 >>
3.19.0-25-generic



ansible -i hosts 192.168.33.12 -m apt -s -a name=vim

192.168.33.12 | success >> {
  "changed": true,
    "stderr": "",
    "stdout": "Reading package lists...\nBuilding dependency tree...\nReading state information...\nThe following extra packages will be installed:\n  libgpm2 libpython2.7 vim-runtime\nSuggested packages:\n  gpm ctags vim-doc vim-scripts\nThe following NEW packages will be installed:\n  libgpm2 libpython2.7 vim vim-runtime\n0 upgraded, 4 newly installed, 0 to remove and 3 not upgraded.\nNeed to get 6899 kB of archives.\nAfter this operation, 31.7 MB of additional disk space will be used.\nGet:1 http://us.archive.ubuntu.com/ubuntu/ trusty/main libgpm2 amd64 1.20.4-6.1 [16.5 kB]\nGet:2 http://us.archive.ubuntu.com/ubuntu/ trusty-updates/main libpython2.7 amd64 2.7.6-8ubuntu0.2 [1039 kB]\nGet:3 http://us.archive.ubuntu.com/ubuntu/ trusty/main vim-runtime all 2:7.4.052-1ubuntu3 [4888 kB]\nGet:4 http://us.archive.ubuntu.com/ubuntu/ trusty/main vim amd64 2:7.4.052-1ubuntu3 [956 kB]\nFetched 6899 kB in 22s (313 kB/s)\nSelecting previously unselected package libgpm2:amd64.\n(Reading database ... 59837 files and directories currently installed.)\nPreparing to unpack .../libgpm2_1.20.4-6.1_amd64.deb ...\nUnpacking libgpm2:amd64 (1.20.4-6.1) ...\nSelecting previously unselected package libpython2.7:amd64.\nPreparing to unpack .../libpython2.7_2.7.6-8ubuntu0.2_amd64.deb ...\nUnpacking libpython2.7:amd64 (2.7.6-8ubuntu0.2) ...\nSelecting previously unselected package vim-runtime.\nPreparing to unpack .../vim-runtime_2%3a7.4.052-1ubuntu3_all.deb ...\nAdding 'diversion of /usr/share/vim/vim74/doc/help.txt to /usr/share/vim/vim74/doc/help.txt.vim-tiny by vim-runtime'\nAdding 'diversion of /usr/share/vim/vim74/doc/tags to /usr/share/vim/vim74/doc/tags.vim-tiny by vim-runtime'\nUnpacking vim-runtime (2:7.4.052-1ubuntu3) ...\nSelecting previously unselected package vim.\nPreparing to unpack .../vim_2%3a7.4.052-1ubuntu3_amd64.deb ...\nUnpacking vim (2:7.4.052-1ubuntu3) ...\nProcessing triggers for man-db (2.6.7.1-1ubuntu1) ...\nSetting up libgpm2:amd64 (1.20.4-6.1) ...\nSetting up libpython2.7:amd64 (2.7.6-8ubuntu0.2) ...\nSetting up vim-runtime (2:7.4.052-1ubuntu3) ...\nProcessing /usr/share/vim/addons/doc\nSetting up vim (2:7.4.052-1ubuntu3) ...\nupdate-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vim (vim) in auto mode\nupdate-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vimdiff (vimdiff) in auto mode\nupdate-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rvim (rvim) in auto mode\nupdate-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rview (rview) in auto mode\nupdate-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vi (vi) in auto mode\nupdate-alternatives: using /usr/bin/vim.basic to provide /usr/bin/view (view) in auto mode\nupdate-alternatives: using /usr/bin/vim.basic to provide /usr/bin/ex (ex) in auto mode\nProcessing triggers for libc-bin (2.19-0ubuntu6.6) ...\n"
}

```

## Playbookとは
- どのホストに対して何を実行するかを指定する
- 何を実行するか＝どのモジュールをどのオプションで
- YAML形式
- playbookでは、記述の解釈と指定のみ行い、実際の処理はモジュールに任せている


## モジュールとは
- サーバ設定の実務を行うのがモジュール
- プラグイン形式になっている
- ansibleに最初から入っているモジュールも多くある
- 標準入出力さえできればどんな言語でもモジュール作成可能らしい
- モジュールはリモートへ転送されて実行されるので、phpで書いたモジュールの実行にはリモートにもphpが必要となる
- ansible-doc モジュール名

## 簡単なPlaybook

```
cat <<_EOD_ > hosts
[test-servers]
192.168.33.12
_EOD_
```

```yml
cat <<_EOD_ > simple-playbook.yml
---
- hosts: test-servers
  become: yes
  tasks:
    - name: be sure httpd is installed
      apt: name=apache2 state=installed

    - name: be sure httpd is running and enabled
      service: name=apache2 state=started enabled=yes
_EOD_
```

- hosts
  - 対象となるホストまたはグループ名
  - カンマ区切りorYAMLのリスト指定で複数も可能
- become
  - リモートでsudoを使って実行する
  - デフォルトはrootでの実行だが、`become_user`を指定することでほかのユーザとして実行できる
- tasks
  - 実行する処理


## ansible-playbookコマンドのオプション
```
ansible-playbook --help
```

```
# シンタックスチェック
ansible-playbook -i hosts simple-playbook.yml --syntax-check

# タスクの一覧を確認
ansible-playbook -i hosts simple-playbook.yml --list-tasks

# dry-run
ansible-playbook -i hosts simple-playbook.yml --check
```

|オプション|説明|
|:--|:--|
|-k, --ask-pass|SSH のパスワードを尋ねる(プロンプトが出る)|
|-K, --ask-sudo-pass|sudo のパスワードを尋ねる(プロンプトが出る)|
|-C, --check|インストールなどの変更は行わないが、条件の確認などは実行する|
|-c CONNECTION, --connection=CONNECTION|local, ssh, paramiko から選択。デフォルトは paramiko。古い OpenSSH で ssh を指定する場合は ANSIBLE_SSH_ARGS="" と環境変数を空で上書きする必要がある|
|-D, --diff|file や template の差分(diff)を表示する。--check と一緒に使うと便利|
|-e EXTRA_VARS, --extra-vars=EXTRA_VARS|追加の変数を key=value で指定する。playbook に書かれている変数の上書きはされない|
|-f FORKS, --forks=FORKS|並列実行する数(デフォルトは5)|
|-h, --help|このヘルプメッセージを表示して終了する|
|-i INVENTORY, --inventory-file=INVENTORY|インベントリファイルを指定する(デフォルトは /etc/ansible/hosts)|
|-l SUBNET, --limit=SUBNET|対象サーバーを指定のものだけに制限する|
|--list-hosts|それぞれの playbook の対象ホスト一覧を表示して終了する。playbook の実行はされない|
|--list-tasks|各 playbook のタスク一覧を表示して終了する|
|-M MODULE_PATH, --module-path=MODULE_PATH|モジュールファイルのディレクトリを指定する(デフォルトは /usr/share/ansible)|
|--private-key=PRIVATE_KEY_FILE|SSH の秘密鍵ファイルを指定する|
|--start-at-task=START_AT|指定の task から開始する|
|--step|ひとつの task ごとに "Perform task: タスク名 (y/n/c):" と確認される。y: (yes) 実行する、n: (no) 実行しない、c: (continue) 以降を確認なしで実行する|
|-s, --sudo|対象サーバーでの task を sudo で実行する|
|-U SUDO_USER, --sudo-user=SUDO_USER|sudo での実行ユーザーを指定する(デフォルトは root)|
|--syntax-check|playbook の文法チェックだけを行う|
|-t TAGS, --tags=TAGS|指定の tag が付けられた task のみを実行する|
|-T TIMEOUT, --timeout=TIMEOUT|SSH のタイムアウトを指定する(デフォルトは10秒)|
|-u REMOTE_USER, --user=REMOTE_USER|SSH で接続するユーザー名を指定する|
|-v, --verbose|冗長モード (-vvv でより冗長な出力になる)|
|--version|バージョンを表示して終了する|


## playbookを--checkしたあと、実行してみる
```sh
vagrant@ubuntu-1404:~$ ansible-playbook -i hosts simple-playbook.yml --check

PLAY [test-servers] ***********************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.12]

TASK: [be sure httpd is installed] ********************************************
changed: [192.168.33.12]

TASK: [be sure httpd is running and enabled] **********************************
failed: [192.168.33.12] => {"failed": true}
msg: no service or tool found for: apache2

FATAL: all hosts have already failed -- aborting

PLAY RECAP ********************************************************************
           to retry, use: --limit @/home/vagrant/simple-playbook.retry

192.168.33.12              : ok=2    changed=1    unreachable=0    failed=1



vagrant@ubuntu-1404:~$ ansible-playbook -i hosts simple-playbook.yml

PLAY [test-servers] ***********************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.12]

TASK: [be sure httpd is installed] ********************************************
changed: [192.168.33.12]

TASK: [be sure httpd is running and enabled] **********************************
ok: [192.168.33.12]

PLAY RECAP ********************************************************************
192.168.33.12              : ok=3    changed=1    unreachable=0    failed=0


vagrant@ubuntu-1404:~$ ansible-playbook -i hosts simple-playbook.yml

PLAY [test-servers] ***********************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.12]

TASK: [be sure httpd is installed] ********************************************
ok: [192.168.33.12]

TASK: [be sure httpd is running and enabled] **********************************
ok: [192.168.33.12]

PLAY RECAP ********************************************************************
192.168.33.12              : ok=3    changed=0    unreachable=0    failed=0
```

- ローカルからnode2にapache2がインストールされたことを確認
  - http://localhost:8000/
- 何回実行しても同じ = 冪等性


## GATHERING FACTS
- 対象サーバから情報を集めるタスク
- setup モジュール
- 集める必要が無い場合には、playbook.ymlに`gather_facts: no`
- `ansible -i hosts 192.168.33.12 -m setup`
- 集めた情報はtaskのオプションやテンプレート(jinja2)で使える
  - `{{ ansible_eth0["ipv4"]["address"] }}`
  - `{{ ansible_eth0.ipv4.address }}`
  - `{{ ansible_processor_count * 5 }}`

```
tasks:
  - name: gathering data task example
    command: echo {{ ansible_eth0.ipv4.address }}

tasks:
  - name: "shutdown Debian flavored systems"
    command: /sbin/shutdown -t now
    when: ansible_os_family == "Debian"
```


# Best Practiceに沿った構成
