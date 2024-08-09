# easy-bhyve - お手軽BHyVeスクリプト <!-- omit in toc -->

**このドキュメントは現在書きかけで、一部不完全な部分があります。**

## easy-bhyveの概要

easy-bhyveはできる限り少ない設定でbhyveを利用できるように設計した、bhyveのVMのマネージコマンドです。bhyveの細かいオプションを知らなくても、少しの設定で新たなVMを用意して利用できます。easy-bhyveで作られたVMは、通常のコマンドと同様の感覚で起動や停止等を行えます。またZFSのボリュームと組み合わせることで、VMのクローンやスナップショットが実現でき、開発やテスト用のサーバーを柔軟に扱えます。

## easy-bhyveの特徴

easy-bhyveの特徴を並べてみます。

- コマンド感覚でのVMの起動・停止
- ホスト側の事前設定不要
- 新しいVMを作る場合、最小で変数を3個設定するだけ
- ZFSと組み合わせることで
  - UFSに比べてVMのディスクパフォーマンスの向上
  - VMイメージのクローンやスナップショットを簡単に実現

## 目次 <!-- omit in toc -->

- [easy-bhyveの概要](#easy-bhyveの概要)
- [easy-bhyveの特徴](#easy-bhyveの特徴)
- [easy-bhyveの初期設定](#easy-bhyveの初期設定)
  - [ホスト側の変数](#ホスト側の変数)
  - [各ファイルのパス](#各ファイルのパス)
  - [VM用のディスクイメージのルートパスの準備](#vm用のディスクイメージのルートパスの準備)
  - [各VMのID](#各vmのid)
- [FreeBSDのVMへのインストールと起動](#freebsdのvmへのインストールと起動)
  - [FreeBSD用VMの設定](#freebsd用vmの設定)
  - [VM制御用シンボリックリンクの作成](#vm制御用シンボリックリンクの作成)
  - [リソースの確認とディスクイメージの作成](#リソースの確認とディスクイメージの作成)
  - [OSのインストールと起動](#osのインストールと起動)
- [DebianやUbuntuのインストールと起動](#debianやubuntuのインストールと起動)
  - [Debian、Ubuntu系のVMの設定](#debianubuntu系のvmの設定)
  - [NetBSDのVMの作成](#netbsdのvmの作成)
- [ZFS Volume専用コマンド](#zfs-volume専用コマンド)
- [easy-bhyveコマンド](#easy-bhyveコマンド)
- [リファレンス](#リファレンス)
  - [easy-bhyveが作成するコマンド一覧](#easy-bhyveが作成するコマンド一覧)
  - [ホスト側の環境を設定する変数 (すべて必須)](#ホスト側の環境を設定する変数-すべて必須)
  - [省略可能な変数](#省略可能な変数)
  - [VMで個別に設定できる変数一覧](#vmで個別に設定できる変数一覧)
  - [ディスク関連の変数](#ディスク関連の変数)
  - [ネットワーク関連の変数](#ネットワーク関連の変数)

## easy-bhyveの初期設定

easy-bhyveはシェルスクリプトで、本体と同じディレクトリに設定ファイル(シェル変数を記述)を用意します。

一般的なコマンドであれば、`/usr/local/bin/XXXX`等に配置しますが、easy-bhyveでは本体と設定ファイルを個人の実行ファイルのディレクトリ(多くは`~/bin`)に配置するのを前提に作ってあります。そしてVMの起動停止等の制御コマンドをeasy-bhyveを置いたディレクトリ下の`eb`ディレクトリ(つまり`~/bin/eb`、変更可)を作成してそれぞれに応じたシンボリックリンクを作成し、それらのシンボリック経由でコマンドを利用します。ユーザーは、`doas`または`sudo`を使ったroot権限が利用できることが必要です。

文章で表現するより、次のeasy-bhyveの初期設定の例を見ればピンとくると思います。

$HOME/binディレクトリ内の内容の確認と設定ファイルの内容

```command
$ ls -l ~/bin
total 17
-rwxr-xr-x  1 minmin  minmin  17779 Mar 11 07:14 easy-bhyve
-rw-r--r--  1 minmin  minmin    597 Mar 11 07:20 easy-bhyve.conf

$ cat bin/easy-bhyve.conf
#  system config
bridge=bridge0          # デフォルトのbridge if
netif=em0               # bridge if を使う IF名
volroot=zroot/vm        # VM用ZFS Volumeのprefix

# FreeBSD
freebsd0_id=0
freebsd0_cpu=2
freebsd0_mem=512M
freebsd0_freebsd=YES
freebsd0_cd0=/ISOFILEPATH/FreeBSD/FreeBSD-14.0-RELEASE-amd64-disc1.iso

# Debian GNU/Linux
debian1_id=1
debian1_cpu=2
debian1_mem=512M
debian1_cd0=/ISOFILEPATH/Debian/debian-12.4.0-amd64-netinst.iso
debian1_grub_root=msdos1
$
```

設定ファイルが用意できたら、easy-bhyveのセットアップを行います。

```command
$ easy-bhyve setup
$
```

~/binディレクトリ下を確認すると、ebディレクトリが作成され、その下には多数のシンボリックリンクが作られているのがわかります。

```command
$ ls -l ~/bin
total 26
-rwxr-xr-x  1 minmin  minmin  17779 Mar 11 07:14 easy-bhyve
-rw-r--r--  1 minmin  minmin    597 Mar 11 07:20 easy-bhyve.conf
drwxr-xr-x  2 minmin  minmin     25 Mar 11 07:26 eb

$ ls bin/eb
debian1-boot            debian1-shutdown        freebsd0-history
debian1-clear           debian1-snapshot        freebsd0-install
debian1-clone           debian1-ttyboot         freebsd0-resources
debian1-console         easy-bhyve.conf         freebsd0-rollback
debian1-history         freebsd0-boot           freebsd0-shutdown
debian1-install         freebsd0-clear          freebsd0-snapshot
debian1-resources       freebsd0-clone          freebsd0-ttyboot
debian1-rollback        freebsd0-console

$ easy-bhyve list
0       freebsd0
1       debian1
$
```

セットアップ後、例えばfreebsd0のインストールを行う場合、次のコマンドを実行します。

```command
$ ~/bin/eb/freebsd0-install
````

### ホスト側の変数

easy-bhyveはシェルスクリプトで、設定はシェル変数に値を指定します。シェル変数はホスト側の環境の設定と、VM毎のリソースの設定があります。easy-bhyveを使う場合に最低限必要なホスト側の環境を設定する変数は次の3つです。

| 変数の設定例 | 説明 |
|--------------|------|
| bridge=bridge0 | VMが使うtapインターフェースを割り当てホスト側のブリッジ名 |
| netif=igb0 | ブリッジを割り当てるホストの物理IF |
| volroot=zroot/vm | ZFS Volumeのprefix (UFSの場合 /vm 等) |


### 各ファイルのパス

easy-bhyveを利用する場合、`~/bin`ディレクトリに次のファイルを用意します。もちろん他のディレクトリに配置しても問題ないですが、ユーザーの書き込み権限があるディレクトリであることが必要です。

| ファイル | パス |
|----------|------|
| easy-bhyve本体 | ~/bin/easy-bhyve |
| 共通設定ファイル | ~/bin/easy-bhyve.conf |
| ホスト個別設定ファイル | ~/bin/easy-bhyve-SVRNAME.conf |

設定ファイルは、共通設定ファイル、個別設定ファイルのいずれかが必要です。両方ある場合は最初に共通設定ファイルを読み込み、次に個別設定ファイルを読み込みます。ここでSVRNAMEはbhyveを動かすホスト名のローカルパート (FQDNの先頭の「.」の前まで)です。共通設定ファイルと個別設定ファイルがあるのは、ホームディレクトリをNFSで複数のサーバーから共有している場合に設定を分けるのに便利だからです。実際作者の環境は、ホームディレクトリを複数のサーバーで共有していて、2台のサーバーでそれぞれbhyveのVMを利用しています。

### VM用のディスクイメージのルートパスの準備

ZFSの場合 (root権限で設定)

```command
$ zfs create -o mountpoint=none zroot/vm
```

easy-bhyveを使う場合はZFSでの利用を推奨します。UFSでも利用できますが、VMイメージのスナップショットやクローンを使うにはZFSが必須です。

USFの場合 (root権限で設定)

```command
$ mkdir /vm
```

作成したルートパスを設定ファイルのvolrootに設定します。

```shell
volroot=zroot/vm
```

### 各VMのID

easy-bhyveではtapインターフェースやnmdmデバイスの割り当てのために、各VMに個別のID(0以上の数字)を設定します。例えばVMのIDを**3**に設定した場合、ネットワークインターフェースは**tap3**、nmdmデバイスは **/dev/nmdm3A**、 **/dev/nmdm3B**となります。

## FreeBSDのVMへのインストールと起動

### FreeBSD用VMの設定

IDを0、VM名をfreebsd0とした場合の設定です。最小で次の3つの変数を設定します。

| 変数の設定例 | 説明 |
|--------------|------|
| freebsd0_id=0 | ID の指定 |
| freebsd0_cd0=/ISOFILEPATH/FreeBSD-14.0-RELEASE-amd64-disc1.iso | インストール用ISOイメージのパス |
| freebsd0_freebsd=YES | FreeBSDの場合bhyveloadを使うためその識別 |

個別にCPUやメモリ量を設定する場合はVMNAME_で始まる変数に値を指定します。

| 変数の設定例 | 説明 |
|--------------|------|
| freebsd0_cpu=2 | freebsd0のCPU数 |
| freebsd0_mem=512M | freebsd0に割当るメモリ量 |

デフォルトのディスクイメージは`volroot`変数で指定したパスの直下にVM名のボリューム、つまり`zroot/vm/freebsd0`（デバイス名としては`/dev/zvol/zroot/vm/freebsd0`）となるので設定する必要はありません。デフォルトとは違うものを個別に指定する場合は、`freebsd0_hd0`の変数に直接**デバイス名**を指定します。

### VM制御用シンボリックリンクの作成

変数の設定が終わったら、同じIDを別のVMに使っていたりしないか確認します。これには`easy-bhyve list`コマンドを使います。

```command
$ ls bin
easy-bhyve      easy-bhyve.conf
$ easy-bhyve list
0       freebsd0
1       debian1
$
```

`easy-bhyve list`ではIDの重複を確認する機能は用意していないので、ユーザー自身でIDの重複を確認する必要があります。IDの重複が無ければ、各VMの制御コマンド用を生成するために`easy-bhyve setup`コマンドを実行します。

```command
$ easy-bhyve setup
$ ls bin
easy-bhyve      easy-bhyve.conf eb
$ ls bin/eb
debian1-boot            debian1-shutdown        freebsd0-history
debian1-clear           debian1-snapshot        freebsd0-install
debian1-clone           debian1-ttyboot         freebsd0-resources
debian1-console         easy-bhyve.conf         freebsd0-rollback
debian1-history         freebsd0-boot           freebsd0-shutdown
debian1-install         freebsd0-clear          freebsd0-snapshot
debian1-resources       freebsd0-clone          freebsd0-ttyboot
debian1-rollback        freebsd0-console
$
```

`easy-bhyve setup`は、すでにebディレクトリが存在する場合、**一旦ebディレクトリ内のファイルをすべて削除してから作るため、ebディレクトリに任意のファイルを置くことはできません**。

### リソースの確認とディスクイメージの作成

**VMNAME-resources**を実行して、各リソースの設定を確認します。

```command
$ ~/bin/eb/freebsd0-resources
host=brunehaut
system=freebsd0
con=/dev/nmdm0A
cpu=2
mem=512M
zvol=zroot/vm/freebsd0
cd0=/isopath/FreeBSD/FreeBSD-14.0-RELEASE-amd64-disc1.iso
host_netif=em0
host_bridge=bridge0
if0=tap0 if0_bridge=bridge0
hd0=/dev/zvol/zroot/vm/freebsd0
ifopt=" -s 2:0,virtio-net,tap0"
hdopt=" -s 3:0,virtio-blk,/dev/zvol/zroot/vm/freebsd0"
$
```

リソースを確認できたら、**VMNAME-resource**コマンドを使ってディスクイメージを作成します。

```command
$ ~/bin/eb/freebsd0-resources -V 30g -s
$
```

-Vと-sはzfs createのオプションで、それぞれボリュームサイズの指定とスパースにするかどうかとなります。-sを指定すればストレージ消費は実際に使用した分だけになるので、少々大き目のサイズを指定しても影響は無いでしょう。

### OSのインストールと起動

ディスクイメージが作成できたらOSをインストールします。これには**VMNAME-install**コマンドを利用します。

```command
$ ~/bin/eb/freebsd0-install
```

あとはインストーラの指示に従って作業します。インストールが終わったらOSを起動します。

```command
$ ~/bin/eb/freebsd0-boot
```

または

```command
$ ~/bin/eb/freebsd0-ttyboot
```

**VMNAME-boot**コマンドを使って起動した場合、bhyveプロセスはバックグランドで起動しコンソールはnmdmデバイスにリダイレクトされ、そのままではブートメッセージを確認できません。コンソールへ接続するには**VMNAME-console**を利用します。

```command
$ ~/bin/eb/freebsd0-console
```

**VMNAME-console**では内部でcuコマンドを利用しているので、終了するには**改行を入力した直後**に「~.」(チルダとピリオド)を入力します。

**VMNAME-ttyboot**では、bhyveプロセスはフォアグラウンドのままになり、コンソールの入出力がそのまま標準入出力に接続されます。

## DebianやUbuntuのインストールと起動

Debian、Ubuntu等のLinuxやOpenBSD、NetBSD等は、OSの起動にはbhyve用のGrubの実装であるgrub2-bhyveを使うため、あらかじめgrub2-bhyveをインストールする必要があります。

```command
$ pkg install -y grub2-bhyve
```

### Debian、Ubuntu系のVMの設定

Debianの場合の設定例は次のようになります。

| 変数の設定例 | 説明 |
|--------------|------|
| debian1_id=1 | ID の設定 |
| debian1_grub_bootpart=msdos1 | VMのディスクイメージの起動パーティション |
| deboam1_cd0=/ISOFILEPATH/debian-12.4.0-amd64-netinst.iso | Debianのインストール用ISOファイルのパス |

grub2-bhyveでMBR形式のディスクパーティションは、順にmsdos1、msdos2として識別します。Debianでは起動パーティションはmsdos1となるようです。またUbuntuでは、ディスクパーティションをGPTで設定するため、起動パーティションにはgpt2を設定します。

| 変数の設定例 | 説明 |
|--------------|------|
| ubuntu0_id=2 | ID の設定 |
| ubuntu0_grub_bootpart=gpt2 | VMのディスクイメージの起動パーティション |
| ubuntu0_cd0=/ISOFILEPATH/ubuntu-24.04-live-server-amd64.iso | Ubuntuのインストール用ISOファイルのパス |

起動パーティションがどれであるかは、実際にはOSのインストール後でないとわからないこともあります。そのためeasy-bhyveでは、**VMNAME-boot**コマンドで **-p**オプションを指定することで、一旦grub2-bhyveのコマンドプロンプトで起動を中断できます。あらためてgrub2-bhyve のlsコマンドを使って、パーティションの内容を確認し、VMNAME_grub_bootpart変数を設定します。次は 実際にVMNAME-bootで **-p**オプションを指定してgrubのコマンドプロンプトで `ls`コマンドでインストールしたファイルシステムの内容を確認している例です。

```text
                             GNU GRUB  version 2.00

   Minimal BASH-like line editing is supported. For the first word, TAB
   lists possible command completions. Anywhere else TAB lists possible
   device or file completions.


grub> ls
(hd0) (hd0,msdos5) (hd0,msdos1) (host)
grub> ls (hd0,msdos1)
        Partition hd0,msdos1: Filesystem type ext* - Last modification time
2024-06-18 13:10:50 Tuesday, UUID 8defe964-1517-428d-b7cd-c9c7ab61520c -
Partition start at 2048 - Total size 60911616 sectors
grub> ls (hd0,msdos1)/
lost+found/ etc/ media/ vmlinuz.old var/ usr/ lib lib64 sbin bin boot/ dev/ hom
e/ proc/ root/ run/ sys/ tmp/ mnt/ srv/ opt/ initrd.img.old vmlinuz initrd.img
grub> exit
```

### NetBSDのVMの作成

IDを 3、VM名をnetbsd3とした場合の設定です。

| 変数の設定例 | 説明 |
|--------------|------|
| netbsd3_id=2 | ID の設定 |
| netbsd3_cd0=/ISOFILEPATH/NetBSD-7.0.1-amd64.iso | NetBSDのインストール用ISOファイル |
| netbsd3_grub_install="knetbsd -h -r cd0a (cd0)/netbsd" | grub2-bhyveでNetBSDをインストールするコマンド |
| netbsd3_grub_bootcmd="knetbsd -h -r /dev/rld0a (hd0,netbsd1)/netbsd" | grub2-bhyveでNetBSDを起動するコマンド |

ここで設定した VMNAME_grub_install と VMNAME_grub_bootcmd、VMNAME_grub_bootpartはOS毎に設定する必要があります。

変数名の設定ができれば、あとはFreeBSDの場合と同様に**VMNAME-install**を実行してOSをインストールし、その後は**VMNAME-boot**で起動します。

## ZFS Volume専用コマンド

ZFSではスナップショットとロールバック、クローンを自由に利用できるので、VMのイメージに対してこれらを簡単に扱えるコマンドを用意してあります。

**VMNAME-snapshot** は指定した引数の空白を “_”に置換し「日時分」を加えたsnapshotを作成します。たとえば`~/bin/eb/freebsd0-snapshot hoge hoge`を実行すると`zroot/vm/freebsd0@_202407121353-hoge_hoge`という名前のスナップショットが作成されます。また引数を指定ない場合は、該当Volumeのスナップショットの一覧を表示します。

```text
$ ~/bin/eb/debian10-snapshot
NAME                                           VOLSIZE   USED  REFER
zroot/vm/debian10                                  30G  4.31G  3.17G
zroot/vm/debian10@_202206150938-10.12_updated      30G   477M  2.75G
zroot/vm/debian10@_202307311452-backup             30G   236M  3.07G
```

**VMNAME-rollback**は、指定したsnapshot名(@の後ろ部分)まで ZFSのロールバックを行います。
引数を指定しない場合は、最新のスナップショットまでロールバックします。該当のVMが終了した状態で利用します。

**VMNAME-clone**はディスクイメージにsnapshotを作成し、引数名でイメージのクローンを作成します。同じ設定のVMを新たに作る場合等に利用します。

```text
$ ~/bin/eb/debian10-clone
Usage: easy-bhyve [snapshot] New_VM [New_VM]
$ ~/bin/eb/debian10-clone deb10test
NAME                                                    VOLSIZE   USED  REFER
zroot/vm/deb10test                                          30G     1K  3.17G
zroot/vm/deb10test@_202408091502-debian10_202408091502      30G     0B  3.17G
$
```

**VMNAME-history**は**VMNAME-clone**で作成したイメージの履歴を表示します。

```text
$ ~/bin/eb/deb10test-history
zroot/vm/debian10
zroot/vm/debian10@_202206150938-10.12_updated
zroot/vm/debian10@_202307311452-backup
zroot/vm/debian10@_202408091502-clone_base
zroot/vm/deb10test
zroot/vm/deb10test@_202408091502-debian10_202408091502
$
```

## easy-bhyveコマンド

easy-bhyveを引数を指定せずに実行すると、稼働中のVMのpsコマンドの結果と`/dev/vmm/`の七ランを表示します。**-a**オプションを指定すると、さらにVMに設定したCPU数とメモリ量を表示します。

```text
$ easy-bhyve -a
USER   PID %CPU %MEM     VSZ    RSS TT  STAT STARTED     TIME COMMAND
root 82398 12.1  1.2 1129340 203652  -  SCs  07:47    0:06.48 bhyve: debian10 (bhyve)
root 30261  0.0  4.6 1105164 750720  -  SCs   3Jul24 20:51.28 bhyve: debian12 (bhyve)

crw-------  1 root wheel 0x125 Jul 12 07:47 /dev/vmm/debian10
crw-------  1 root wheel 0x108 Jul  3 12:49 /dev/vmm/debian12

82398  debian10      2  1024m
30261  debian12      4  1024m
$
```

debian10をVM内部でshutdownコマンド等で停止すると、次のように動いていないのにdebian10のリソースが残ります。

```text
$ easy-bhyve -a
USER   PID %CPU %MEM     VSZ    RSS TT  STAT STARTED     TIME COMMAND
root 30261  0.0  4.6 1105164 750720  -  SCs   3Jul24 20:51.28 bhyve: debian12 (bhyve)

crw-------  1 root wheel 0x125 Jul 12 07:47 /dev/vmm/debian10
crw-------  1 root wheel 0x108 Jul  3 12:49 /dev/vmm/debian12
$
```

このような場合は、debian10に割り当てたネットワークインターフェースも残ったままの状態となります。これらのリソースを開放する場合は**VMNAME-clear**コマンドを使います。

```text
$ ~/bin/eb/debian10-clear
$
```

使ったリソースを残さないようにするには、VM内部で停止するのではなく、ホスト側で**VMNAME-shutdown**コマンドを使えばVMの停止後にリソースを開放します。

## リファレンス

### easy-bhyveが作成するコマンド一覧

ZFS、UFS共通コマンド

| コマンド名 | 説明 |
|------------|------|
| VMNAME-install | VMへのOSのインストール |
| VMNAME-boot | インストール済みVMの起動 |
| VMNAME-ttyboot | コンソールをttyで起動 (*-boot -t でも同じ) |
| VMNAME-console | NULLモデム経由でコンソールへ接続 |
| VMNAME-resources | リソースの表示 |
| VMNAME-shutdown | VMのシャットダウン |
| VMNAME-clear | VMの各リソースのクリア |

ZFS Volume専用コマンド

| コマンド名 | 説明 |
|------------|------|
| VMNAME-snapshot | VM用ZFSボリュームのスナップショットの作成 |
| VMNAME-rollback | 最後のスナップショットへのロールバック |
| VMNAME-clone | VMのクローンの作成 |
| VMNAME-history | VMのクローンやスナップショットの履歴確認 |

### ホスト側の環境を設定する変数 (すべて必須)

| 変数の設定例 | 説明 |
|--------------|------|
| netif | ブリッジを割り当てるホストの物理IF |
| bridge | tapインターフェースを割り当てホスト側のブリッジ名 |
| volroot | ZFS Volumeのprefix (UFSの場合 /vm 等) |

### 省略可能な変数

| 変数の設定例 | 説明 |
|--------------|------|
| cpu | VMのデフォルトのCPU数・省略した場合はホストのCPU数の半分 |
| mem | VMのデフォルトの割当メモリ・省略した場合は 512MB |
| ebdir | シンボリックリンクを作成するディレクトリ名を指定する。デフォルトは`eb`でeasy-bhyveがあるディレクトリ下に作成される|

### VMで個別に設定できる変数一覧

| 変数の設定例 | 説明 |
|--------------|------|
| VMNAME_id | ID の設定 (必須)|
| VMNAME_cdX | インストール用ISOイメージのパス (インストールする場合必須) |
| VMNAME_hdX | インストール用ISOイメージのパス |
| VMNAME_ifX	| タップIFの名前を指定する。デフォルトはtap${${VMNAME_id}}|
| VMNAME_ifX_bridge | タップIFが接続されるホスト側のブリッジIFの名前。デフォルトは${bridge}|
| VMNAME_freebsd=YES | FreeBSDの場合に指定する(Grubではなくbhyveloadを使うため) |
| VMNAME_cpu| VMのCPU数 |
| VMNAME_mem | VMNに割当るメモリ量 |
| VMNAME_grub_bootpart | VM用を起動するストレージイメージのパーティション番号<br>MBRの場合はmsdosX、GPTの場合はgptX (Xは1, 2, 3, ...)|
| VMNAME_grub_install | grub2-bhyveでOSインストール時の起動するコマンド文字列 |
| VMNAME_grub_bootcmd | grub2-bhyveでOSを起動するコマンド文字列 |


### ディスク関連の変数

| 変数名 | 説明 |
|--------------|------|
| VMNAME_hd0 | 1台目のディスクイメージのパスで省略するとデフォルト値となる<br>ZFSのデフォルト /dev/zvol/${volroot}/VMNAME<br>UFSのデフォルト	${volroot}/VMNAME |
| VMNAME_hd1 | 2台目のディスクイメージのパス (デフォルトは未設定) |
| VMNAME_hdX	X=2, 3, …  | (X+1)台目のディスクイメージのパス |

### ネットワーク関連の変数

| 変数名 | 説明 |
|--------------|------|
| VMNAME_if0	| タップIFの名前を指定する。デフォルトはtap${${VMNAME_id}}|
| VMNAME_if0_bridge|tapが接続されるブリッジIFの名前。デフォルトは${bridge}|
| VMNAME_if1	| 2番目のタップIF	デフォルトは未設定|
| VMNAME_if1_bridge|	2番目のタップIFが接続されるブリッジIF|

ネットワークインターフェースも、ディスクと同様に複数設定できます。
