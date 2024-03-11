# easy-bhyve - お手軽BHyVeスクリプト <!-- omit in toc -->

**このドキュメントは現在更新中です。当初と仕様を変更した関係で、古いままの記述や誤った記述があります**

- [easy-bhyveの概要](#easy-bhyveの概要)
  - [easy-bhyveが作成するコマンド一覧](#easy-bhyveが作成するコマンド一覧)
- [初期設定](#初期設定)
  - [各ファイルのパス](#各ファイルのパス)
  - [easy-bhyveでのid](#easy-bhyveでのid)
  - [VM用のZFS Volume(あるいはファイルシステム)のルートパスの用意](#vm用のzfs-volumeあるいはファイルシステムのルートパスの用意)
  - [ホスト側の変数](#ホスト側の変数)
  - [CPUとメモリのデフォルト](#cpuとメモリのデフォルト)
  - [FreeBSD用VMの設定](#freebsd用vmの設定)
  - [Debian、Ubuntu系のVMの設定](#debianubuntu系のvmの設定)
  - [NetBSDのVMの作成](#netbsdのvmの作成)
- [インストール用ISOイメージのパスの扱い](#インストール用isoイメージのパスの扱い)
- [ディスク関連の変数](#ディスク関連の変数)
- [ネットワーク関連の変数](#ネットワーク関連の変数)
- [設定したらコマンドのシンボリックリンクを作成](#設定したらコマンドのシンボリックリンクを作成)
- [シンボリックリンクができたらやること](#シンボリックリンクができたらやること)
- [OSのインストールと起動](#osのインストールと起動)
- [-d デバッグオプション](#-d-デバッグオプション)
- [ZFS Volume専用コマンド](#zfs-volume専用コマンド)

## easy-bhyveの概要

easy-bhyveは、BHyVeのVMの起動や終了等をユーザーがコマンド感覚で扱うシェルスクリプトで、シェルスクリプトの本体と同じディレクトリに設定ファイル(シェル変数を記述)を用意します。

一般的なコマンドであれば、`/usr/local/bin/XXXX`等に配置しますが、easy-bhyveでは本体と設定ファイルを個人の実行ファイルのディレクトリ(多くは`~/bin`)に配置するのを前提に作ってあります。そしてVMの起動停止等の制御コマンドをeasy-bhyveを置いたディレクトリ下の`eb`ディレクトリ(つまり`~/bin/eb`、変更可)を作成してそれぞれに応じたシンボリックリンクを作成し、それらのシンボリック経由でコマンドを利用します。ユーザーは、`doas`または`sudo`を使ったroot権限が利用できることが必要です。

文章で表現するより、次のeasy-bhyveの初期設定の例を見ればピンとくると思います。

$HOME/binディレクトリ内の内容の確認と設定ファイルの内容

```command
$ ls -l ~/bin
total 17
-rwxr-xr-x  1 minmin  minmin  17779 Mar 11 07:14 easy-bhyve
-rw-r--r--  1 minmin  minmin    597 Mar 11 07:20 easy-bhyve.conf
$ cat bin/easy-bhyve.conf
#
#  default リソース
#
bridge=bridge0          # デフォルトのbridge if
netif=em0               # bridge if を使う IF名
volroot=zroot/vm        # VM用ZFS Volumeのprefix

# ISO Images
iso_path=/archive/ISO   # ISO image のあるPathのprefix
iso_freebsd14=${iso_path}/FreeBSD/FreeBSD-14.0-RELEASE-amd64-disc1.iso
iso_debian12=${iso_path}/Debian/debian-12.4.0-amd64-netinst.iso

# FreeBSD
freebsdvm_id=0
freebsdvm_cpu=2
freebsdvm_mem=256m
freebsdvm_freebsd=YES
freebsdvm_cd0=${iso_freebsd14}

# Debian GNU/Linux
debianvm_id=1
debianvm_cpu=2
debianvm_mem=256m
debianvm_cd0=${iso_debian12}
debianvm_grub_root=msdos1
$
```

easy-bhyveのイニシャライズ

```command
$ easy-bhyve setup
$
```

~/binディレクトリ下を確認すると、ebディレクトリが作成されて多数のシンボリックリンクが作られているのがわかります。

```command
$ ls -lR ~/bin
total 26
-rwxr-xr-x  1 minmin  minmin  17779 Mar 11 07:14 easy-bhyve
-rw-r--r--  1 minmin  minmin    597 Mar 11 07:20 easy-bhyve.conf
drwxr-xr-x  2 minmin  minmin     25 Mar 11 07:26 eb

bin/eb:
total 12
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-boot -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-clean -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-clone -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-console -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-history -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-install -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-resources -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-rollback -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-shutdown -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-snapshot -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 debianvm-ttyboot -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  18 Mar 11 07:26 easy-bhyve.conf -> ../easy-bhyve.conf
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-boot -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-clean -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-clone -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-console -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-history -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-install -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-resources -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-rollback -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-shutdown -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-snapshot -> ../easy-bhyve
lrwxr-xr-x  1 minmin  minmin  13 Mar 11 07:26 freebsdvm-ttyboot -> ../easy-bhyve
$
```

初期化後、例えばfreebsdvmのインストールを行う場合、次のコマンドを実行します。

```command
$ ~/bin/eb/freebsdvm-install
````

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

## 初期設定

### 各ファイルのパス

easy-bhyveを利用する場合、~/binディレクトリに次のファイルを用意します。もちろん~/binではないディレクトリに配置しても問題ありませんが、ユーザー権限で書き込めることが必要です。

| ファイル | パス |
|----------|------|
| easy-bhyve本体 | ~/bin/easy-bhyve |
| 共通設定ファイル | ~/bin/easy-bhyve.conf |
| 個別設定ファイル | ~/bin/easy-bhyve-SVRNAME.conf |

設定ファイルは、共通設定ファイル、個別設定ファイルのいずれかが必要です。両方ある場合は最初に共通設定ファイルを読み込み、次に個別設定ファイルを読み込みます。ここでSVRNAMEはbhyveを動かすホスト名のローカルパート (FQDNの先頭の「.」の前まで)です。

### easy-bhyveでのid

easy-bhyveでは各VMに個別のidを割り当てる必要があり、0以上の適当な数字を設定します。idを基準にVMのtapインターフェースやnmdmデバイスを割り当てるため、VM毎に違うものである必要があります。たとえばあるVMのidを**3**にした場合、ネットワークインターフェースは**tap3**、nmdmデバイスは **/dev/nmdm3A**、 **/dev/nmdm3B**となります。

### VM用のZFS Volume(あるいはファイルシステム)のルートパスの用意

ZFSの場合

```command
*$ zfs create -o mountpoint=none zroot/vm
```

USFの場合

```command
$ mkdir /vm
```

ルートパスは変数で設定するため、任意のものが利用できます(後述)。

### ホスト側の変数

easy-bhyveはシェルスクリプトで、設定はシェル変数を設定します。シェル変数はホスト側の設定と、VM毎の設定の2種類があります。

| 変数の設定例 | 説明 |
|--------------|------|
| bridge=bridge0 | tapインターフェースを割り当てホスト側のブリッジ名 |
| netif=igb0 | ブリッジを割り当てる物理IF |
| volroot=zroot/vm | ZFS Volumeのprefix (UFSの場合 /vm 等) |
| ebdir=eb | シンボリックリンクを作成するディレクトリでeasy-bhyveがあるディレクトリ下に作成される|

### CPUとメモリのデフォルト

| 変数の設定例 | 説明 |
|--------------|------|
| cpu=1 | VMのデフォルトのCPU数 |
| mem=256M | VMのデフォルトの割当メモリ |

### FreeBSD用VMの設定

IDを 1、VM名をfreebsd0とした場合の設定です。

| 変数の設定例 | 説明 |
|--------------|------|
| freebsd0_id=1 | id の設定 |
| freebsd0_cd0=/isopath/FreeBSD-14.0-RELEASE-amd64-disc1.iso | インストール用ISOイメージのパス |
| freebsd0_freebsd=YES | FreeBSDの場合bhyve-loadを使うため(grub2-bhyveを使わない) |

個別にCPUやメモリ量を設定する場合はVMNAME_で名前を指定します。

| 変数の設定例 | 説明 |
|--------------|------|
| freebsd0_cpu=2 | freebsd0のCPU数 |
| freebsd0_mem=512M | freebsd0に割当るメモリ量 |

デフォルトの`zroot/vm/freebsd0`となるので設定する必要はありません。
個別に指定する場合は、`freebsd0_hd0`の変数で指定します。

### Debian、Ubuntu系のVMの設定

| 変数の設定例 | 説明 |
|--------------|------|
| debian0_id=2 | id の設定 |
| deboam0_cd0=/isopath/debian-12.4.0-amd64-netinst.iso | Debianのインストール用ISOのパス |

### NetBSDのVMの作成

IDを 3、VM名をnbsd0とした場合の設定です。

| 変数の設定例 | 説明 |
|--------------|------|
| nbsd0_id=3 | id の設定 |
| nbsd0_cd0=/isopath/NetBSD-7.0.1-amd64.iso | NetBSDのインストール用ISOファイル |
| nbsd0_grub_inst="knetbsd -h -r cd0a (cd0)/netbsd" | grub2-bhyveでNetBSDを起動するコマンド |
| nbsd0_grub_boot="knetbsd -h -r /dev/rld0a (hd0,netbsd1)/netbsd" | grub2-bhyveでNetBSDを起動するコマンド |
    - _grub_inst と _grub_boot はOS毎に調べる必要あり(後述)

## インストール用ISOイメージのパスの扱い

- 個別設定
  - iso_path=/isofile_dir/ISO
  - freebsd0_cd0=${iso_path}/FreeBSD-11.1-RELEASE-amd64-disc1.iso
  - debian0_cd0=${iso_path}/debian-9.1.0-amd64-netinst.iso
- 既存VMをクローンしたVMを作った場合は、インストールが不要となるためISOイメージの設定は不要です。

## ディスク関連の変数 

- _zvol	ZFS Volume名
	- デフォルト ${volroot}/VMNAME
- _hd0	1台目のディスクイメージのパス
  - ZFSのデフォルト		/dev/zvol/${VMNAME_zvol}/VMNAME
  - UFSのデフォルト	${volroot}/VMNAME
- _hd1	2台目のディスクイメージのパス (デフォルトは未設定)
- _hdX	X=2, 3, …  (X+1)台目のディスクイメージのパス

注) 実際の変数名は VMNAME_zvol, VMNAME_hd0, …

## ネットワーク関連の変数

- _if0			1本目のtapを指定
  - デフォルト	tap${${VMNAME_id}}
- _if0_bridge	1本目のtapが接続されるブリッジIF
  - デフォルト	${bridge}
- _if1			2本目のtap	デフォルトは未設定
  - _if1_bridge	2本目のtapが接続されるブリッジIF
- _if2, _if2_bridge, _if3, _if4_bridge, …

注) 実際の変数名は VMNAME_if0, VMNAME_if0_bridge, …

## 設定したらコマンドのシンボリックリンクを作成

- easy-bhyve setupを実行する (~/bin にはPATHが設定されているはず)
  - $ easy-bhyve setup
- 内部処理
  - 自分自身のidを定義している行で行頭から
    - fbsd0_id=1
    - deb0_id=2
  - fbsd0 や deb0 の VM名とコマンド名を組み合わせてリンクを作成
  - 制限：1行に2つ定義してあるとうまく検索できません(笑)
    - fbsd0_id=1   deb0_id=2
  - ~/bin/eb/* にコマンドのシンボリックリンクがゾロゾロ出来上がる

## シンボリックリンクができたらやること

- VMNAME-resources を実行して、変数の設定が正しいことを確認
- ディスクイメージの作成
  - $ ~/bin/eb/fbsd0-resources -V 20g -s
    - -V：size指定   -s：スパースにするかどうか (zfs volume作成と同じ)

## OSのインストールと起動

OS が FreeBSD の場合

- インストールして起動する
  - $ ~/bin/eb/fbsd0-install
  - $ ~/bin/eb/fbsd0-boot

grub-bhyveを使うOSのうち、Ubuntu、Debian

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
  - 例 ~/bin/eb/fbsd0-snapshot hoge fuga
  - スナップショット名  zroot/vm/fbsd0@_201711261353-hoge_fuga
- 引数を指定ない場合は、Volume の snapshot 一覧が表示される

*-rollback

- 指定したsnapshot (@の後ろ部分)まで zfs 的 rollback行う
  - zroot/vm/fbsd0@_201711261353-hoge_fuga
- 引数を指定しない場合は、最新のsnapshotにrollback
