# Simple IME module for GNU Emacs (tr-emacs-ime-module)

Windows 用 (MinGW/Cygwin) GNU Emacs でダイナミックモジュールの機構を利用し、
IME パッチ無しの公式バイナリなどでも、
IME による日本語入力を使いやすくする試みです。

## はじめに

GNU Emacs 26.2 から、
IME パッチがなくても MS-IME などの IME (input method editor)
による日本語入力が「とりあえず」できるようになりました。
しかし IME パッチ無しだと、Emacs 自前の
かな漢字変換 (IM: input method) と IME が連動しないため、

* IME を ON/OFF してもモードラインのかな漢字変換状態表示が変わらない
* IME ON で使っているとき、
  状況に応じて自動的に IME OFF してくれる機能が無く、
  直接入力したいキーが IME に吸われて未確定文字になってしまう
    * ミニバッファでの y/n 入力
    * M-x によるミニバッファでのコマンド名入力
    * など
* C-\\ (toggle-input-method) すると IME ではなくて、
  IM による Emacs 独自の日本語入力モードになってしまい、
  他の Windows アプリと操作感や変換辞書が異なるため使いにくい

という問題があり使いにくくなってしまいます
（他にもあると思いますが、個人的に困った部分のみ列挙しています）。
IME パッチの方が便利なのは間違いありませんが、
Emacs リリースのたびに煩雑なことをしなければならなかったり、
場合によっては IME パッチによって Emacs が不安定になってしまうこともあります。

そこで、全世界のユーザが使っている IME パッチ無しで安定した
Emacs バイナリを使い、上記のような問題を解消できる最小限を目指します。
幸い、GNU Emacs 27.1 からダイナミックモジュールが
デフォルトで有効になったので、これを使って IME 関連の実装を追加します。

同様の試みに
[w32-imeadv](https://github.com/maildrop/w32-imeadv)
がありますが、
これは IME パッチの機能を完全に再現して置き換えることを目指しているようで、
C++ 実装のモジュールによって、それなりに複雑で大がかりな機構
（外部プロセスを経由したスレッド間通信など）を備えています。
一方で IME パッチで UI や設定をつかさどる要の部分ともいえる Lisp 実装
[w32-ime.el](https://github.com/trueroad/w32-ime.el)
が使えるわけでは無いので、こうした部分は異なってしまっています。

本プロジェクトでは、安定性を重視するため、
モジュールの実装は必要最小限として複雑な実装をしません。
そのため、IME パッチの全機能を再現することはできません
（それだとやっぱり使いにくいから実装を増やしていく、
なんてことになるかもしれません）。
逆に UI や設定など従来の IME パッチと同じようにしたいため、
[w32-ime.el](https://github.com/trueroad/w32-ime.el)
を、ほとんどそのまま使えるようにします。

## 環境

### 動作環境

* Windows 用 (MinGW/Cygwin) GNU Emacs
    * IME パッチは不要です
    * GNU Emacs 26.2 以降が必要です
        * IME パッチ無しでも「とりあえず」IME が使える必要があります
    * ダイナミックモジュールが有効になっている必要があります
        * GNU Emacs 27.1 からデフォルトで有効です
    * Cygwin
      [emacs-w32](https://cygwin.com/packages/summary/emacs-w32.html)
      27.1-1 で動作確認しています
    * MinGW でダイナミックモジュールをビルドすれば、
      GNU 公式バイナリでも動作するかもしれません（未確認）

### ビルド環境

動作環境に加えて以下が必要になります。
Cygwin の Emacs で使うならば Cygwin 環境で、
GNU 公式バイナリなどの MinGW の Emacs で使うなら MinGW 環境で、
それぞれ揃えてください。

* C99 対応コンパイラ
    * 普通の GCC など
* emacs-module.h
    * Emacs についているはずです
    * ビルド時に使った emacs-module.h の Emacs バージョンが新しくて、
      動作環境の Emacs バージョンが古い場合は、動作しません
* Autotools (autoconf, automake, libtool)
    * [本リポジトリ](https://github.com/trueroad/tr-emacs-ime-module)
      のソースを使ってビルドする場合に必要です
    * Autotools が無ければ手動でビルド・インストールすることも一応できます

## ビルド

### Autotools

以下のようにすればビルドできます。

```
$ ./autogen.sh
$ mkdir build
$ cd build
$ ../configure --prefix=/usr
$ make
```

### 手動

[tr-ime-module.c](./src/tr-ime-module.c)
のあるディレクトリで、

```
$ touch config.h
$ gcc -shared -o tr-ime-module.dll tr-ime-module.c -limm32
```

などのようにすればできると思います。
config.h は空で構いません。gcc のオプションなどは適宜調整してください。

## インストール

### Autotools

以下のようにしてインストールできます。

```
$ make install
```

### 手動

ビルドした tr-ime-module.dll と、
[tr-ime-module-helper.el](./lisp/tr-ime-module-helper.el),
[w32-ime-for-tr-ime-module.el](./w32-ime/w32-ime-for-tr-ime-module.el)
の 3 ファイルを Emacs の load-path が通っているディレクトリに置いてください。

## 設定

以下のようにするとロードできます。
~/.emacs.d/init.el などに追加しておくとよいと思います。

```el
;; IME パッチ無しモジュール有りならばモジュールをロードする
(when (and (eq window-system 'w32)
           (not (symbol-function 'ime-get-mode))
           (string= module-file-suffix ".dll")
           (locate-library "tr-ime-module-helper"))
  (require 'tr-ime-module-helper)
  (require 'w32-ime "w32-ime-for-tr-ime-module"))
```

一部を除き IME パッチと同じような設定ができます。
ただし `(global-set-key [M-kanji] 'ignore)` はしないでください。
モジュール環境か IME パッチ環境かで設定を分けるならば、
以下のようにしてください。（分ける必要が無ければ不要です。）

```el
;; IME パッチ環境とモジュール環境で別々の設定をする
(if (featurep 'tr-ime-module-helper)
    (progn
      ;; ダイナミックモジュール環境用
      (global-set-key [M-kanji] 'toggle-input-method))
    ;; IME パッチ環境用
    (global-set-key [M-kanji] 'ignore))
```

あとはお好みで以下のような設定をします。

```el
;; IM のデフォルトを IME に設定
(setq default-input-method "W32-IME")
;; IME のモードライン表示設定
(setq-default w32-ime-mode-line-state-indicator "[--]")
(setq w32-ime-mode-line-state-indicator-list '("[--]" "[あ]" "[--]"))
;; IME 初期化
(w32-ime-initialize)
;; IME 制御（yes/no などの入力の時に IME を OFF にする）
(wrap-function-to-control-ime 'universal-argument t nil)
(wrap-function-to-control-ime 'read-string nil nil)
(wrap-function-to-control-ime 'read-char nil nil)
(wrap-function-to-control-ime 'read-from-minibuffer nil nil)
(wrap-function-to-control-ime 'y-or-n-p nil nil)
(wrap-function-to-control-ime 'yes-or-no-p nil nil)
(wrap-function-to-control-ime 'map-y-or-n-p nil nil)
```

## 制約

わかっているだけで以下のような制約があります。

* Alt + 半角/全角キー（もしくは C-\\）以外の IME ON/OFF に対応できない
    * 現代的には Alt 無しの半角/全角キー単独で IME の ON/OFF
      操作をするようですが、これを IM 側に反映できません
        * 個人的には古いスタイルの Alt + 半角/全角キーでの操作が
          身についてしまっているので特に困っていませんが…
    * マウスなど他の方法で IME の ON/OFF 操作をした場合も
      IM 側に反映できません
    * C-\\ か Alt + 半角/全角キーを押せば同期します
        * モードラインを見ながら 1, 2 回押せば意図した状態で同期します
    * すべての IME 切り替え方法に対応するには、
      UI スレッドのメッセージループへの WM_IME_NOTIFY メッセージ到着を
      何らかの方法で Lisp へ通知する必要があり、これがかなり困難です
        * IME パッチは C 実装でメッセージ処理を追加して、
          kanji キーのイベントという形で通知しているようです
        * w32-imeadv はメッセージ処理をサブクラス化で奪い取り、
          さらに別プロセスを経由して通知するという、
          かなり大がかりで複雑な機構を採用しています
* IME ON の状態で C-x, C-c, C-h などをしても自動的に IME OFF にならず、
  そのあとのキー入力が通常文字だと IME に吸われ未確定文字になってしまう
    * M-x などは wrap-function-to-control-ime で設定することにより
      自動的に IME OFF にできますが、C-x などについてはできません
    * この目的に使えそうなフックが見つからず、対応はかなり困難です
        * IME パッチは直接 C 実装に手を入れて対応しているようです
        * w32-imeadv は、やはり実現困難なので、メッセージ処理を奪い取った上で
          C-x が押されたら IME OFF するという
          「ad-hoc な方法」が実装されています
* IME ON の状態で C-s などの検索開始をすると、
  未確定文字列がミニバッファではなくて、元々入力していた位置に表示される
    * IME パッチ無しのときの IME 関連メッセージ処理に
      問題があるのではないかと思いますが、
      この処理を置き換えるのはかなり複雑です
        * IME パッチは C 実装で IME 関連メッセージ処理を
          完全に置き換えています
        * w32-imeadv はメッセージ処理を奪い取って置き換えています
* 未確定文字列のフォントが指定できず、
  ☃や🍣のような文字がおかしな表示（いわゆるトーフ）になる
    * 候補一覧の選択ウィンドウが出れば、その中では正しい表示になります
    * 確定してしまえば（そのグリフを持ったフォントが
      Emacs に設定されていれば）Emacs 上でも正しく表示されます
    * 対応するには少なくともフォント設定機構を何とかした上で、
      メッセージ処理を奪い取るなどの、かなり複雑な処理が必要です
* 再変換 (RECONVERSION, DOCUMENTFEED) には対応できない
    * 対応するにはメッセージ処理を奪い取り、
      UI スレッドのメッセージループから Lisp への通知が必要になるなど、
      かなり困難な処理が必要です
* IME ON/OFF 制御や状態取得に MSDN
  などの公式なドキュメントに記載されていない非公式のメッセージを使っている
    * 最悪の場合は Windows のアップデートなどにより、
      いきなり動かなくなる可能性もあります
    * 対応するにはメッセージ処理を奪い取るなどの複雑な処理が必要です

## ライセンス

Copyright (C) 2020 Masamichi Hosoda

Simple IME module for GNU Emacs (tr-emacs-ime-module)
is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

Simple IME module for GNU Emacs (tr-emacs-ime-module)
is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with tr-emacs-ime-module.
If not, see <https://www.gnu.org/licenses/>.

[COPYING](./COPYING)
