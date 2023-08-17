# easy-bhyve - お手軽BHyVeスクリプト

BhyVe assistant script. 

This is useful for bhyve.
But all documents are in Japanese. Sorry!

https://docs.google.com/presentation/d/13FuFBqcL_SBt3qbMX5LYciEKdePLlzn5RJpdQnVbEII/edit?usp=sharing

## お手軽BHyVeスクリプトの概要

このプログラムはBHyVeなVMをユーザーがコマンド感覚で扱うスクリプトです

通常のコマンドであれば、`/usrl/ocal/bin/XXXX`等に配置しますが、easy-bhyveと本体を`~/bin`に配置するのが前提になっています。利用ユーザーは`doas`または`sudo`を使ったroot権限が利用できることが必要です。

新しいVMを作るとき、最小で変数を2個設定するだけ
別のサーバーでの同じ名前のVMも同じスクリプト内で設定
自分の ~/bin をCVSで共有している関係
HDDイメージを複数台追加できるように改良(当初2台まで、2017年6月) 
ようやくVM側で複数のNetwork IFを利用できるようになり(2017年7月)
自由に仮想ネットワークが組めます  ＼(^o^)／
