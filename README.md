# easy-bhyve - お手軽BHyVeスクリプト <!-- omit in toc -->

## easy-bhyveの概要 <!-- omit in toc -->

easy-bhyveは実験やテスト用途でVMを利用するために、できる限り簡単な設定でVMの作成、起動を行なうことを目標に作られたbhyveのマネージコマンドです。easy-bhyveで作られたVMは、通常のコマンドと同様の感覚で起動や停止等を行えます。またZFSのボリュームと組み合わせることで、VMのクローンやスナップショットが簡単に実現でき、テストサーバーを便利に扱えます。

## easy-bhyveの特徴

- 

- [easy-bhyveの使い方](#easy-bhyveの使い方)
  - [ホスト側の変数](#ホスト側の変数)
  - [各ファイルのパス](#各ファイルのパス)
  - [各VMのid](#各vmのid)
  - [VM用のディスクイメージのルートパスの準備](#vm用のディスクイメージのルートパスの準備)
  - [FreeBSD用VMの設定](#freebsd用vmの設定)
  - [Debian、Ubuntu系のVMの設定](#debianubuntu系のvmの設定)
  - [NetBSDのVMの作成](#netbsdのvmの作成)
  - [ディスク関連の変数](#ディスク関連の変数)
  - [ネットワーク関連の変数](#ネットワーク関連の変数)
- [VM制御用シンボリックリンクの作成](#vm制御用シンボリックリンクの作成)
- [リソースの確認とディスクイメージの作成](#リソースの確認とディスクイメージの作成)
- [OSのインストールと起動](#osのインストールと起動)
- [Grubを使うOSでのインストールと起動](#grubを使うosでのインストールと起動)
- [-d デバッグオプション](#-d-デバッグオプション)
- [ZFS Volume専用コマンド](#zfs-volume専用コマンド)
- [リファレンス](#リファレンス)
  - [easy-bhyveが作成するコマンド一覧](#easy-bhyveが作成するコマンド一覧)
  - [ホスト側の環境を設定する変数](#ホスト側の環境を設定する変数)
  - [省略可能な変数](#省略可能な変数)
  - [VMで個別に設定できる変数一覧](#vmで個別に設定できる変数一覧)

## easy-bhyveの使い方

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
debian1-clean           debian1-snapshot        freebsd0-install
debian1-clone           debian1-ttyboot         freebsd0-resources
debian1-console         easy-bhyve.conf         freebsd0-rollback
debian1-history         freebsd0-boot           freebsd0-shutdown
debian1-install         freebsd0-clean          freebsd0-snapshot
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

easy-bhyveはシェルスクリプトで、設定ファイルではシェル変数に値を指定します。シェル変数はホスト側の環境の設定と、VM毎のリソースの設定があります。easy-bhyveを使う場合に最低限必要なホスト側の環境を設定する変数は次の3つです。

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

### 各VMのid

easy-bhyveでは各VMに個別のidを割り当てる必要があり、0以上の適当な数字を設定します。idを基準にVMのtapインターフェースやnmdmデバイスを割り当てるため、VM毎に違うものである必要があります。たとえばあるVMのidを**3**にした場合、ネットワークインターフェースは**tap3**、nmdmデバイスは **/dev/nmdm3A**、 **/dev/nmdm3B**となります。

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

### FreeBSD用VMの設定

IDを 0、VM名をfreebsd0とした場合の設定です。

| 変数の設定例 | 説明 |
|--------------|------|
| freebsd0_id=0 | id の設定 |
| freebsd0_cd0=/ISOFILEPATH/FreeBSD-14.0-RELEASE-amd64-disc1.iso | インストール用ISOイメージのパス |
| freebsd0_freebsd=YES | FreeBSDの場合bhyveloadを使うため(Grubを使わないOS) |

個別にCPUやメモリ量を設定する場合はVMNAME_で名前を指定します。

| 変数の設定例 | 説明 |
|--------------|------|
| freebsd0_cpu=2 | freebsd0のCPU数 |
| freebsd0_mem=512M | freebsd0に割当るメモリ量 |

デフォルトのディスクイメージは`volroot`変数で指定したパスの直下にVM名のボリューム、つまり`zroot/vm/freebsd0`（デバイス名としては`/dev/zvol/zroot/vm/freebsd0`）となるので設定する必要はありません。デフォルトとは違うものを個別に指定する場合は、`freebsd0_hd0`の変数で指定します。

### Debian、Ubuntu系のVMの設定

| 変数の設定例 | 説明 |
|--------------|------|
| debian1_id=1 | id の設定 |
| debian1_bootpart=msdos1 | VMのディスクイメージの起動パーティション |
| deboam1_cd0=/ISOFILEPATH/debian-12.4.0-amd64-netinst.iso | Debianのインストール用ISOのパス |

### NetBSDのVMの作成

IDを 2、VM名をnetbsd2とした場合の設定です。

| 変数の設定例 | 説明 |
|--------------|------|
| netbsd2_id=2 | id の設定 |
| netbsd2_cd0=/ISOFILEPATH/NetBSD-7.0.1-amd64.iso | NetBSDのインストール用ISOファイル |
| netbsd2_grub_install="knetbsd -h -r cd0a (cd0)/netbsd" | grub2-bhyveでNetBSDをインストールするコマンド |
| netbsd2_grub_bootcmd="knetbsd -h -r /dev/rld0a (hd0,netbsd1)/netbsd" | grub2-bhyveでNetBSDを起動するコマンド |

ここで設定した VMNAME_grub_install と VMNAME_grub_bootcmd、VMNAME_grub_bootpartはOS毎に設定する必要があります。

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

## VM制御用シンボリックリンクの作成

変数の設定が終わったら、同じidを別のVMに使っていたりしないか確認します。これにはeasy-bhyveコマンドにlistの引数を指定して実行します。

```command
$ ls bin
easy-bhyve      easy-bhyve.conf
$ easy-bhyve list
0       freebsd0
1       debian1$
```

easy-bhyveではidの重複を確認する機能は用意していないので、ユーザー自身でidの重複を確認する必要があります。idの重複が無ければ、各VMの制御コマンド用のシンボリックリンクを作成します。これには引数にsetupを指定します。

```command
$ easy-bhyve setup
$ ls bin
easy-bhyve      easy-bhyve.conf eb
$ ls bin/eb
debian1-boot            debian1-shutdown        freebsd0-history
debian1-clean           debian1-snapshot        freebsd0-install
debian1-clone           debian1-ttyboot         freebsd0-resources
debian1-console         easy-bhyve.conf         freebsd0-rollback
debian1-history         freebsd0-boot           freebsd0-shutdown
debian1-install         freebsd0-clean          freebsd0-snapshot
debian1-resources       freebsd0-clone          freebsd0-ttyboot
debian1-rollback        freebsd0-console
$
```

なお`easy-bhyve setup`はebディレクトリが存在する場合、一旦ebディレクトリ内のファイルを削除してから作り直します。ですのでebディレクトリにユーザーが任意のファイルを置くことはできませんので注意してください。

## リソースの確認とディスクイメージの作成

VMNAME-resources を実行して、変数の設定が正しいことを確認します。

```command
$ bin/eb/debian1-resources
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

リソース設定を確認できたら、VMNAME-resourceコマンドを使ってディスクイメージを作成します。

```command
$ bin/eb/freebsd0-resources -V 30g -s
$
```

-Vと-sはzfs createのオプションで、それぞれボリュームサイズの指定とスパースにするかどうかとなります。-sを指定すればストレージ消費は実際に使用した分だけのになるので、少々大き目のサイズを指定しても影響は無いでしょう。

## OSのインストールと起動

ディスクイメージを作成したら、OSをインストールします。これにはVMNAME-installコマンドを利用します。

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

VMNAME-bootコマンドを使って起動した場合、bhyveプロセスはバックグランドで起動しコンソールはnmdmデバイスにリダイレクトされ、そのままではブートメッセージを確認できません。そのためコンソールへ接続するにはVMNAME-consoleを利用します。

```command
$ ~/bin/eb/freebsd0-console
```

VMNAME-consoleでは内部でcuコマンドを利用しているので、終了には、改行を入力した直後に「~.」(チルダとピリオド)を入力します。

VMNAME-ttybootでは、コンソールの入出力がそのままコマンドラインの標準入出力に接続されます。

## Grubを使うOSでのインストールと起動

grub2-bhyveを使うOSのうち、Debian、Ubuntu等は

- FreeBSD 同様、インストールして起動する
- grub-bhyveはmapファイルの設定が必要なのでは？
  - /tmp に一時的(mktemp)にmapファイルを作りgrub終了後に削除
  - その後bhyveを実行
  
## -d デバッグオプション

内部でsudoを利用している部分を全て echo で代用し、コマンド列が見えて挙動が分かりやすい

- sudo kldload ...
- sudo sysctl ...
- sudo zfs create ... 
- sudo ifconfig ...
- sudo grub-bhyve ...
- sudo bhyve ...
- etc ...

## ZFS Volume専用コマンド

*-snapshot

- 指定した引数の空白を “_”に置換し「日時分」を加えたsnapshotを作成
  - 例 ~/bin/eb/freebsd0-snapshot hoge fuga
  - スナップショット名  zroot/vm/freebsd0@_201711261353-hoge_fuga
- 引数を指定ない場合は、Volume の snapshot 一覧が表示される

*-rollback

- 指定したsnapshot (@の後ろ部分)まで zfs 的 rollback行う
  - zroot/vm/freebsd0@_201711261353-hoge_fuga
- 引数を指定しない場合は、最新のsnapshotにrollback

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
| VMNAME-clean | VMの各リソースのクリア |

ZFS Volume専用コマンド

| コマンド名 | 説明 |
|------------|------|
| VMNAME-snapshot | VM用ZFSボリュームのスナップショットの作成 |
| VMNAME-rollback | 最後のスナップショットへのロールバック |
| VMNAME-clone | VMのクローンの作成 |
| VMNAME-history | VMのクローンやスナップショットの履歴確認 |

### ホスト側の環境を設定する変数

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
| VMNAME_id | id の設定 |
| VMNAME_cdX | インストール用ISOイメージのパス |
| VMNAME_hdX | インストール用ISOイメージのパス |
| VMNAME_ifX	| タップIFの名前を指定する。デフォルトはtap${${VMNAME_id}}|
| VMNAME_ifX_bridge | タップIFが接続されるホスト側のブリッジIFの名前。デフォルトは${bridge}|
| VMNAME_freebsd=YES | FreeBSDの場合に指定する(Grubではなくbhyveloadを使うため) |
| VMNAME_cpu| VMのCPU数 |
| VMNAME_mem | VMNに割当るメモリ量 |
| VMNAME_grub_bootpart | VM用を起動するストレージイメージのパーティション番号<br>MBRの場合はmsdosX、GPTの場合はgptX (Xは1, 2, 3, ...)|
| VMNAME_grub_install | grub2-bhyveでOSのインストール時に起動するコマンド文字列 |
| VMNAME_grub_bootcmd | grub2-bhyveでOSを起動するコマンド文字列 |
