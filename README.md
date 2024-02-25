# easy-bhyve - お手軽BHyVeスクリプト <!-- omit in toc -->

**このドキュメントは現在更新中です。当初と仕様を変更した関係で、古いままの記述や誤った記述があります**

- [easy-bhyveの概要](#easy-bhyveの概要)
- [特徴1: コマンド感覚でVMの起動、停止](#特徴1-コマンド感覚でvmの起動停止)
- [特徴2: フォアグラウンド・バックグラウンドのVM](#特徴2-フォアグラウンドバックグラウンドのvm)
- [特徴3: とにかく手軽にVMを起動](#特徴3-とにかく手軽にvmを起動)
- [特徴4: ZFSとの相性抜群](#特徴4-zfsとの相性抜群)
- [特徴5: ホスト側の事前設定不要](#特徴5-ホスト側の事前設定不要)
- [初期設定](#初期設定)
- [新しいFreeBSD VMの作成](#新しいfreebsd-vmの作成)
- [Debian、Ubuntu系の新しいVMの作成](#debianubuntu系の新しいvmの作成)
- [CPUとメモリの設定](#cpuとメモリの設定)
- [NetBSDの新しいVMの作成](#netbsdの新しいvmの作成)
- [インストール用ISOイメージのパスの扱い](#インストール用isoイメージのパスの扱い)
- [ディスク関連の変数](#ディスク関連の変数)
- [ネットワーク関連の変数](#ネットワーク関連の変数)
- [設定したらコマンドのシンボリックリンクを作成](#設定したらコマンドのシンボリックリンクを作成)
- [シンボリックリンクができたらやること](#シンボリックリンクができたらやること)
- [OSのインストールと起動](#osのインストールと起動)
- [-d デバッグオプション](#-d-デバッグオプション)
- [ZFS Volume専用コマンド](#zfs-volume専用コマンド)

## easy-bhyveの概要

easy-bhyveは、BHyVeのVMの起動や終了等をユーザーがコマンド感覚で扱うスクリプトです

一般的なコマンドであれば、`/usr/local/bin/XXXX`等に配置しますが、easy-bhyveでは本体と設定ファイルを個人の実行ファイルのディレクトリ(多くは`~/bin`)に配置するのが前提になっています。そしてVMの起動停止等の制御コマンドを`~/bin/eb`ディレクトリ下に、それぞれに応じたシンボリックリンクを作成して、それらのシンボリック経由でコマンドを利用します。ユーザーは、`doas`または`sudo`を使ったroot権限が利用できることが必要です。

easy-bhyveが用意するコマンド名一覧

- 標準で用意されるコマンド
  - VMNAME-install : VMへのインストール
  - VMNAME-boot : インストール済みVMの起動
  - VMNAME-ttyboot : consoleをttyで起動 (*-boot -t でも同じ)
  - VMNAME-console : null-modem経由でコンソールへ接続
  - VMNAME-resources : リソースの表示
  - VMNAME-shutdown : VMのシャットダウン
  - VMNAME-clean : VMの各リソースのクリア
- ZFS Volume専用のコマンド
  - VMNAME-snapshot : zvolへのsnapshot
  - VMNAME-rollback : 最後のsnapshotへのrollback
  - VMNAME-clone : VMのclone
  - VMNAME-history : VMのcloneやsnapshotの履歴確認

## 特徴1: コマンド感覚でVMの起動、停止

- 新しいVMを作るとき、最小で変数を2個設定するだけ
- 実験用途向けのVM運用を前提にVMの起動と停止をユーザーがコマンド感覚でホイホイできる
  - 本当の理由 : ホストのパフォーマンス(CPU、メモリ)に影響するので、VMは常時動かしたく無いけど必要な時にすぐ起動したい
- 特権が必要なオペレーションについては内部で doas または sudo を使用
- スクリプト本体を[VMNAME]-[コマンド名]のシンボリックリンクで利用する
  - シンボリックリンクも自動作成
- VMサービスとして root権限を外部に提供する用途には不向き (後述)

## 特徴2: フォアグラウンド・バックグラウンドのVM

フォアグラウンド

- コンソール操作が必要な場合とかVM側の初期設定とか

バックグラウンド

- daemon(8)を利用し独立プロセス化
  - コントロールttyが残らないのでpsで見てスッキリ
  - コンソールへの接続は nmdm(4) 経由
- 副作用としてVM内部からのリブート操作が不可能
  - リブートするとbhyveプロセスそのものが終了した後に立ち上げる術がない
  - root権限を外部に提供するVMサービスが不向きな理由

## 特徴3: とにかく手軽にVMを起動

- DebianやUbuntuはGrub (grub-bhyve) 経由で素直にboot
  - TIPS: インストール時にLVMを利用しないのが良さそう
- Grubでboot時にコマンド列を指定する必要があるOSも、コマンド一発でのbootをサポート
  - OpenBSDとかNetBSDの起動も簡単
- CentOSはGrubで自動起動できるはずなのだができていない
  - 現状カーネルがアップデートされると、設定変更が必要

## 特徴4: ZFSとの相性抜群

- VMのディスクパフォーマンスでもZFS Volumeは有利
- 特に開発環境用VMでのVMイメージのsnapshot、cloneは強力
- ZFS環境で無くても一応スクリプトは使える
 
## 特徴5: ホスト側の事前設定不要

事前設定不要とは言えdoasやsudoとgrub2-bhyveは事前にインストール & doas(またはsudo)の設定が必要

- pkg install doas(sudo) grub2-bhyve

必要なkernel moduleは実行時にロードする

- vmm.ko とか if_tap.koとか if_bridge.ko等
  - 終了しても解放しない(さすがにそこまでしなくて良いはず)
- bridge if や tap if は必要に応じて作成
  - 一度作ったbridge ifはVMをクリアしても削除しない
- tap ifはvmのクリア時に削除

## 初期設定

 | ファイル | パス |
 |----------|------|
 | スクリプト本体 | ~/bin/easy-bhyve |
 | 共通設定ファイル | ~/bin/easy-bhyve.conf |
 | 個別設定ファイル | ~/bin/easy-bhyve-SVRNAME.conf |

VM用のZFS Volume(あるいはファイルシステム)のルートの用意

- ZFS zfs create -o mountpoint=none zroot/vm
 	- VM用のZFS Volume zroot/vm/VMNAME 
- UFS	mkdir /vm
 	- VM用のDiskImage	/vm/VMNAME 
- 名前はこの通りで無くても可  (下記参照)

初期設定の変数

- bridge=bridge0
  - tap IF のデフォルトbridge IF名
- netif=igb0
  - デフォルトbridgeの物理IF
- volroot=zroot/vm
 	- ZFS Volumeのprefix (volroot=/vm UFS用の DiksImageのprefix)
- ebdir=eb
 	- シンボリックリンクを作成するディレクトリ

## 新しいFreeBSD VMの作成

- IDとVMの名前を決める
	- ID			1
	- VM名		fbsd0
- 設定する変数
	- fbsd0_id=1
	- fbsd0_cd0=/isopath/FreeBSD-11.1-RELEASE-amd64-disc1.iso
	- fbsd0_freebsd=YES	(FreeBSDはgrub-bhyve不要のため)

## Debian、Ubuntu系の新しいVMの作成

- IDとVMの名前を決める
	- ID			2
	- VM名		deb0
- 設定する変数
  - deb0_id=2
  - deb0_cd0=/isopath/debian-9.2.1-amd64-netinst.iso

## CPUとメモリの設定

- デフォルトリソース
  - cpu=1			VMのデフォルトのCPU数
  - mem=256M		VMのデフォルトのメモリ
    - スクリプト内で定義
- 個別に値を設定する場合
  - VMNAME_cpu=4
  - VMNAME_mem=2048M
  - 具体例
    - deb0_cpu=2
    - deb0_mem=1024M

## NetBSDの新しいVMの作成

- IDとVMの名前を決める
  - ID			3
  - VM名		nbsd0
- 設定する変数
  - nbsd0_id=3
  - nbsd0_cd0=/isopath/NetBSD-7.0.1-amd64.iso
  - nbsd0_grub_inst="knetbsd -h -r cd0a (cd0)/netbsd"
  - nbsd0_grub_boot="knetbsd -h -r /dev/rld0a (hd0,netbsd1)/netbsd"
    - _grub_inst と _grub_boot はOS毎に調べる必要あり(後述)

## インストール用ISOイメージのパスの扱い

- 個別設定
  - iso_path=/isofile_dir/ISO
  - freebsd0_cd0=${iso_path}/FreeBSD-11.1-RELEASE-amd64-disc1.iso
  - debian0_cd0=${iso_path}/debian-9.1.0-amd64-netinst.iso
- インストールが不要なら、ISOイメージの設定は不要
  - 既存VMをクローンした場合とか

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
