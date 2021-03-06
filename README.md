# 64bit版 GNU Emacs 27.1 for Windows（w/ IMEパッチ）

GNU Emacs 27.1を、Windows向けに64bitでビルドしたものです。[公式のWindowsビルド](http://ftpmirror.gnu.org/emacs/windows/)の同等物に、いわゆるIMEパッチを当てています。よってストレス無く日本語入力できると思いますが、いきなり落ちたりする可能性もありますので、注意してお使いください。

26.3から日本語入力時に未確定文字のインライン表示ができるようになったので、ただ日本語が入力したいというだけであればパッチあては必要なくなったのですが、依然としてw32-ime.elは含まれていませんので一応バイナリを提供します。今後は[tr-emacs-ime-module](https://github.com/trueroad/tr-emacs-ime-module)などが進むべき道かもしれません…。

ダウンロードは[こちら](https://github.com/mhatta/emacs-27-x86_64-win-ime/raw/master/emacs-27.1_x86_64-win-ime-20200914.zip)から。

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [64bit版 GNU Emacs 27.1 for Windows（w/ IMEパッチ）](#64bit版-gnu-emacs-271-for-windowsw-imeパッチ)
    - [実行方法](#実行方法)
        - [init.elの設定](#initelの設定)
    - [ビルド方法](#ビルド方法)
        - [MSYS2のインストール](#msys2のインストール)
        - [必要なパッケージのインストール](#必要なパッケージのインストール)
        - [Emacsのソースコードをダウンロード](#emacsのソースコードをダウンロード)
        - [IMEパッチを当てる](#imeパッチを当てる)
        - [configureとmake](#configureとmake)
        - [DLLのコピー](#dllのコピー)
            - [おまけ C/Migemo](#おまけ-cmigemo)
    - [バグなど](#バグなど)
        - [configureのエラーでビルドできない](#configureのエラーでビルドできない)
        - [なぜか処理が途中で止まる](#なぜか処理が途中で止まる)
        - [日本語入力がオンにならない](#日本語入力がオンにならない)
		- [ewwでhttpsなサイトが開けない](#ewwでhttpsなサイトが開けない)
        - [ImageMagick7への対応](#imagemagick7への対応)

<!-- markdown-toc end -->

## 実行方法

ダウンロードしたzipアーカイヴを展開し、`emacs-27.1\bin\runemacs.exe` を実行してください。

### init.elの設定

私は以下の設定で使っています。

``` emacs-lisp
  ;; Windows IME
  (setq default-input-method "W32-IME")
  (setq-default w32-ime-mode-line-state-indicator "[--]")
  (setq w32-ime-mode-line-state-indicator-list '("[--]" "[あ]" "[--]"))
  (w32-ime-initialize)
  ;; 日本語入力時にカーソルの色を変える設定
  (add-hook 'w32-ime-on-hook '(lambda () (set-cursor-color "coral4")))
  (add-hook 'w32-ime-off-hook '(lambda () (set-cursor-color "black")))

  ;; ミニバッファに移動した際は最初に日本語入力が無効な状態にする
  (add-hook 'minibuffer-setup-hook 'deactivate-input-method)

  ;; isearch に移行した際に日本語入力を無効にする
  (add-hook 'isearch-mode-hook '(lambda ()
                                  (deactivate-input-method)
                                  (setq w32-ime-composition-window (minibuffer-window))))
  (add-hook 'isearch-mode-end-hook '(lambda () (setq w32-ime-composition-window nil)))

  ;; helm 使用中に日本語入力を無効にする
  (advice-add 'helm :around '(lambda (orig-fun &rest args)
                               (let ((select-window-functions nil)
                                     (w32-ime-composition-window (minibuffer-window)))
                                 (deactivate-input-method)
                                 (apply orig-fun args))))

```

## ビルド方法

ビルド方法は基本的に[https://github.com/chuntaro/NTEmacs64](https://github.com/chuntaro/NTEmacs64)を踏襲しています。大体一式で3～4GB程度の空き容量が必要のようです。なお、当方は64bit版の Windows 10 Pro 1909（Build 18363.1082）上でビルドしています。

なお、画像に関してはGIF, JPEG, PNG, SVG, TIFF, XPMに対応しています（後述しますが、ImageMagickは組み込んでいません）。27.1からはImageMagickに頼らない画像の操作に対応したようです。また、GNUTLSにも対応していますので、Emacs Lispで書かれたウェブブラウザewwも動作します。

### MSYS2のインストール

Windows向けUnix開発環境のMSYS2が必要です。[MSYS2 homepage](https://www.msys2.org/)から`msys2-x86_64-（日付）.exe`をダウンロードしてインストールしてください。64bit用ですので、i686ではなくx86_64のほうです。

### 必要なパッケージのインストール

スタートメニューから「MSYS2 64bit」→「MSYS2 MinGW 64-bit」、あるいは`c:\msys64\mingw64.exe`を実行してMSYS2のシェルを起動します。MSYS2はArch Linuxのパッケージマネージャである[Pacman](https://wiki.archlinux.jp/index.php/Pacman)を採用していますので、すでにインストールされているパッケージを`$ pacman -Syu`でアップデートした上で、

```
$ pacman -S --needed base-devel \
  mingw-w64-x86_64-toolchain \
  mingw-w64-x86_64-xpm-nox \
  mingw-w64-x86_64-libtiff \
  mingw-w64-x86_64-giflib \
  mingw-w64-x86_64-libpng \
  mingw-w64-x86_64-libjpeg-turbo \
  mingw-w64-x86_64-librsvg \
  mingw-w64-x86_64-lcms2 \
  mingw-w64-x86_64-jansson \
  mingw-w64-x86_64-libxml2 \
  mingw-w64-x86_64-gnutls \
  mingw-w64-x86_64-zlib
```

を実行して開発に必要なパッケージを追加インストールします。Emacsのバージョンが上がるとたまに必要なパッケージが増えるので（たとえば26.1ではlcms2が追加された）、[公式のINSTALL.W64](https://git.savannah.gnu.org/cgit/emacs.git/tree/nt/INSTALL.W64)を適宜参照したほうがよいでしょう。

### Emacsのソースコードをダウンロード

MSYS2のシェルから`$ wget http://ftpmirror.gnu.org/emacs/emacs-27.1.tar.xz`を実行してEmacsのソースコードをダウンロードし、`$ tar xvf emacs-27.1.tar.xz`で展開します。`c:\` の直下に展開（`c:\emacs-27.1` のように）するのがよいでしょう。

### IMEパッチを当てる

本レポジトリから`emacs-27.1-windows-ime-（日付）.patch`をダウンロードし、

```
$ cd /c/emacs-27.1
$ patch -p1 < /path/to/emacs-27.1-windows-ime-（日付）.patch
```

でパッチを当てます。その上で、`$ ./autogen.sh`を実行して`configure`スクリプトを更新してください。なお、このパッチは[rzl24oziさんが整理してくださったもの](https://gist.github.com/rzl24ozi/008d32c1f0742d3d2901295bf0366efa)をベースにしています。

### configureとmake

`configure`を実行します。

```
$ CFLAGS='-O2 -march=x86-64 -mtune=generic -static -s -g0' LDFLAGS='-s' ./configure --prefix=/c/emacs-27.1 --without-dbus --without-compress-install
```

[chuntaroさんのビルド](https://github.com/chuntaro/NTEmacs64)ではCFLAGSに`-Ofast -march=x86-64 -mtune=corei7`を与えてコンパイルしていますが、[このあたりの議論](https://www.reddit.com/r/emacs/comments/7gex1q/emacs_64bit_for_windows_with_imagemagick_7/)を見るとEmacsの場合は`-O2`のほうがパフォーマンスが良いらしいので、`-O2`に戻しました。なお、27.1からダイナミック・モジュールはデフォルトでサポートされるようになったので、`--with-modules`は不要です。

`configure`が終わったら、

```
$ make bootstrap; make install-strip
```

で`c:\emacs-27.1`以下にインストールされます。

### DLLのコピー

ビルドしたバイナリをMSYS2がインストールされていないマシンで使うには、MSYS2から必要なDLLを`c:\emacs-27.1\bin`以下にコピーして持っていく必要があります。このレポジトリにある `msys2-dll-copy.sh` を使うと良いでしょう。配付しているzipアーカイヴには必要なDLLを入れたつもりですが、抜けがあるかもしれませんので、その場合は自分でコピーしてください。DLLの依存関係に関しては、[Dependency Walker](http://www.dependencywalker.com/)を使うと分かります。

#### おまけ C/Migemo ####

ローマ字で日本語のインクリメンタル検索ができる[C/Migemo](https://github.com/koron/cmigemo)は大変便利なツールですが、GitHubにある最新版の64bit Windows用バイナリがないようなので用意しました。ダウンロードは[こちら](https://github.com/mhatta/emacs-26-x86_64-win-ime/raw/master/cmigemo-1.3-mingw64-20180629.zip)から。適当なところ（たとえば`C:\`）に展開した上で、`C:\cmigemo-mingw64\bin`にPATHを通し、MELPAから`migemo(-el)`をインストールした上で、

``` emacs-lisp
(require 'migemo)
(setq migemo-command "cmigemo")
(setq migemo-options '("-q" "-e"))
(setq migemo-coding-system 'utf-8-unix)
(setq migemo-dictionary "C:/cmigemo-mingw64/share/migemo/utf-8/migemo-dict")
```

などと設定してやれば使えます。

## バグなど

このビルドに関して何かお気づきの点があれば、[issues](https://github.com/mhatta/emacs-26-x86_64-win-ime/issues)で報告してください。

### configureのエラーでビルドできない

`configure`で`emacs does not support 'x86_64-pc-msys' systems`というようなエラーが出てビルドできない場合、WindowsのPATHがMSYS2に引き継がれておかしくなっている可能性があります。Windowsの環境変数で`MSYS2_PATH_TYPE=inherit`を指定している場合は、一時的にinheritではなくstrictやnone（inherit以外なら何でもよい）にしておくと良いでしょう。

### なぜか処理が途中で止まる

エラーや警告は出ないのにビルドが途中でフリーズしてしまう場合、アンチウイルスやランサムウェア対策のソフトウェアが悪さをしていることがあります。私の場合、Acronis Active Protectionが原因でした。Org-modeやAUC-TeXのような大規模なEmacs Lispパッケージをコンパイルする時も、一時的に膨大なファイルの書き込み、読み込みが行われるせいか、ランサムウェアと勘違いされて止められてしまうことがあるようです。これらはEmacsのビルドやパッケージのインストールの際には停止しておいたほうがよいでしょう。

### 日本語入力がオンにならない

以前のWindows 10の更新（1803?）で何かおかしくなったらしく、半角/全角キー等を押してIMEをオンにしようとしても日本語入力が有効にならないという問題が発生していました。一度マウスでEmacsのウィンドウを移動したり、リサイズすると直るようです。[ここでの議論](https://github.com/chuntaro/NTEmacs64/issues/3)を参照してください。

どうやらWindowsのほうで何か直したらしく、1909以降ではこの問題は発生していません。

現在でも、Google日本語入力を使うと、日本語入力は普通にできるのですがウィンドウを動かしたりしないとサジェスト等が表示されないという問題があるようです（Microsoft IMEやATOK 2017では起こらない）。そもそも根本原因がよく分かりません…。

### ewwでhttpsなサイトが開けない

GNUTLS（libgnutls-30.dll）が3.6.3以降だと、ewwでhttpsなサイトが開けないようです。MSYS2のGNUTLSは2019年5月現在すでに3.6.7.1になっているので、[GitlabのGNUTLSのサイトから落とした3.6.3のDLL](https://gitlab.com/gnutls/gnutls/builds/artifacts/gnutls_3_6_3/download?job=MinGW64.DLLs)に入れ替えてあります。

26.3以降ではMSYS2最新のGNUTLSのままでも動くようなので、入れ替えていません。

### ImageMagick7への対応

Emacsは26.3の時点でもImageMagick6までの対応で、バージョン7には対応していません。よってこのビルドにもImageMagickは組み込んでいません。

一応ImageMagick7に対応させるパッチは用意しましたが、思ったように動かないので、パッチのみの提供とします。興味のある方は試してみてください。

ちなみに、問題はEmacsではなく、MSYS2のImageMagick7がまともに動作していないことにあるようです。[このあたりの議論](https://github.com/Alexpux/MINGW-packages/issues/1885)を参照してください。なお、pdf-toolsを使うのにEmacs側のImageMagickサポートは必要ありません。
