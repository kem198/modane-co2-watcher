# modane-co2-watcher

![CO2濃度計の写真](images/co2_watcher.jpg)

Raspberry Pi で動作する二酸化炭素濃度計アプリケーションです。  
2022 年当時、C 言語と Linux の学習の過程で制作しました。

## 仕様

- 周辺の CO2 濃度を 10 分毎に取得し、グラフとして表示します。
  - グラフは 20 時間前まで、10 時間前まで、3 時間前まで の 3 つの表示を切り替えられます。
- 取得した CO2 濃度は `/logs/co2_conces.csv` へログとして蓄積されます。
  - このログファイルは約 30 日分の記録を保存し、制限に達したらログファイルをローテートします。
  - ログファイルは最大 12 ファイル分を保持します。
- 現在の日時や天気を一定間隔毎に表示します。
  - 天気を取得する場所は環境変数で指定できます。
- キャラクターの表情が一定間隔で変わります。
- シェルのウィンドウ幅に応じてレスポンシブに描画します。

## 動作環境

下記の環境で動作確認しています。

### ハードウェア

- Raspberry Pi 4 Model B Rev 1.5
- MH-Z19 (CO2 測定センサ)

### ソフトウェア

- Raspberry Pi OS 64-bit
- Python 3.13
- LXTerminal
- 等幅フォント ([HackGen Console](https://github.com/yuru7/HackGen))

## セットアップ ～ 初回起動

### 1. ハードウェアを接続

Raspberry Pi の電源を入れる前に、MH-Z19 を本体の UART ポートへ接続します。

接続例については [CO2 濃度の測定](#co2-濃度の測定) に記載している参考資料を確認してください。

完了後、Raspberry Pi に電源を接続して起動します。

### 2. 必要なパッケージをインストール

コンパイルと Python パッケージのインストールに必要なパッケージをインストールします。

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
  build-essential \
  libncurses-dev \
  python3-full \
  swig \
  liblgpio-dev
```

### 3. UART を有効化

MH-Z19 と Raspberry Pi が UART で通信できるようにします。

```bash
sudo raspi-config
```

`3 Interface Options` → `I6 Serial Port`

```text
Serial console → No
Serial port hardware → Yes
```

設定後、Raspberry Pi を再起動します。

```bash
sudo reboot
```

UART デバイスを確認します。

```bash
ls -l /dev/serial* /dev/ttyAMA* /dev/ttyS* 2>/dev/null
```

`/dev/serial0` が存在することを確認します。

### 4. リポジトリを取得

プログラムのソースコードを取得します。

```bash
cd ~
git clone https://github.com/kem198/modane-co2-watcher.git
cd ~/modane-co2-watcher
```

### 5. Python 仮想環境を作成

system Python とは分離した Python 環境を作成します。

```bash
python3 -m venv ~/modane-co2-watcher-venv
source ~/modane-co2-watcher-venv/bin/activate
```

### 6. Python パッケージをインストール

`requirements.txt` に記載したバージョンの Python パッケージをインストールします。

```bash
python -m pip install -r requirements.txt
```

### 7. プログラムをコンパイル

C のソースコードから実行ファイルを作成します。

```bash
gcc ModaneCO2Watcher.c -lncursesw -o ModaneCO2Watcher.out
```

## 動作確認

### 1. MH-Z19 の通信を確認

MH-Z19 から CO2 濃度を取得できることを確認します。

```bash
python3 -c "import mh_z19; print(mh_z19.read_all(serial_console_untouched=True))"
```

以下のように `co2` が表示されれば正常です。

```text
{'co2': 883, 'temperature': 35, 'TT': 75, 'SS': 0, 'UhUl': 1280}
```

### 2. プログラムを起動

(任意) 天気表示の地域に設定します。

```bash
# 例: 福岡の場合
export WTTR_LOCALE="Fukuoka"
```

※ 未設定の場合、東京 (Tokyo) がデフォルトで使用されます。

アプリケーションを起動します。

```shell
./ModaneCO2Watcher.out
```

## (任意) ワンコマンドで実行できるようにする

`.bashrc` を利用してコマンドのエイリアスを作成し、Raspberry Pi の再起動後にアプリケーションを実行できるようにします。

```sh
cat >> "$HOME/.bashrc" <<'EOF'

# modane-co2-watcher
alias mcw='cd "$HOME/modane-co2-watcher" && \
export WTTR_LOCALE="Fukuoka" && \
export PATH="$HOME/modane-co2-watcher-venv/bin:$PATH" && \
./ModaneCO2Watcher.out'
EOF
```

`.bashrc` へ追記された内容を確認します。

```sh
cat ~/.bashrc
```

Raspberry Pi を再起動します。

```bash
sudo reboot
```

エイリアスを実行し、アプリケーションが起動できることを確認します。

## (任意) 推奨フォントで表示する

推奨のフォントである [HackGen](https://github.com/yuru7/HackGen) を 表示に利用できるようにします。

### 1. フォントファイルの設定

フォントファイルを取得・展開し、フォントとして利用できるようにします。

```sh
mkdir -p "$HOME/.local/share/fonts"

cd /tmp
wget -q https://github.com/yuru7/HackGen/releases/download/v2.10.0/HackGen_v2.10.0.zip
unzip -o HackGen_v2.10.0.zip -d HackGen_v2.10.0

find HackGen_v2.10.0 -name "*.ttf" -exec cp {} "$HOME/.local/share/fonts/" \;
fc-cache -f
```

フォントファイルが登録されていることを確認します。

```sh
fc-list | grep -i "HackGen"
```

```sh
# 期待値
/home/ユーザ名/.local/share/fonts/HackGen-Regular.ttf: HackGen:style=Regular
/home/ユーザ名/.local/share/fonts/HackGen-Bold.ttf: HackGen:style=Bold
/home/ユーザ名/.local/share/fonts/HackGen35Console-Regular.ttf: HackGen35 Console:style=Regular
/home/ユーザ名/.local/share/fonts/HackGenConsole-Regular.ttf: HackGen Console:style=Regular
/home/ユーザ名/.local/share/fonts/HackGenConsole-Bold.ttf: HackGen Console:style=Bold
/home/ユーザ名/.local/share/fonts/HackGen35-Regular.ttf: HackGen35:style=Regular
/home/ユーザ名/.local/share/fonts/HackGen35-Bold.ttf: HackGen35:style=Bold
/home/ユーザ名/.local/share/fonts/HackGen35Console-Bold.ttf: HackGen35 Console:style=Bold
```

LXTerminal の Edit > Preference から HackGen Console Regular を設定します。

## 参考文献

### Raspberry Pi

#### セットアップ

- [ラズパイで遊ぼう！ - YouTube](https://www.youtube.com/playlist?list=PLZv220voQQ_OYkVoim13CA91R_iLospsR)
- [Raspberry Pi4で使えるタッチパネル付き5インチディスプレイを買ってみた！ – すいラボ](https://sui-lab.info/archives/3222)
- [【ラズパイ】Raspberry Piをディスプレイなしでセットアップする - 車輪日記](https://bowmiow.net/garage/raspi-first/#toc12)

#### 設定

- [ラズベリーパイでフォントを簡単に追加削除する | けいきゅん ヽ(^◇^\*)/♪ でおじゃる](https://ameblo.jp/anima-ameblo/entry-12398046009.html)
- [ラズパイを起動したら、ターミナル開いてシェルを実行する方法 - Qiita](https://qiita.com/tonosamart/items/f59daa481f90c85a8a99)
- [電源入れたらRaspberryPiのターミナルがGUIで全画面表示されるようにする - 知見（・・）！](https://amiq11.hatenablog.com/entry/2018/09/12/230201)
- [vim :: readonly のファイルを sudo で強制的に保存する [Tipsというかメモ]](https://tm.root-n.com/unix:command:vim:readlonly_write)
- [Raspberry Piの起動時にターミナルが立ち上がり、「Hello world」と表示される機能を実装しようと思いましたが上手くいきません.](https://teratail.com/questions/334030)
- [RasiPiでブラウザを自動起動してキオスク端末にする方法 | 映像とその周辺](https://www.kalium.net/image/2021/03/11/rasipiでブラウザを自動起動してキオスク端末にする/)

#### CO2 濃度の測定

- [Raspberry Pi 4とMH-Z19Bで二酸化炭素濃度を計測してみた | DevelopersIO](https://dev.classmethod.jp/articles/raspberry-pi-4-b-mh-z19b-co2/)
- [【Python】Raspberry Pi + mh-z19でCO2濃度取得してみた - BFT名古屋 TECH BLOG](https://bftnagoya.hateblo.jp/entry/2021/08/25/120844)

### C 言語

#### 画面描画

- [curses による端末制御](https://www.kushiro-ct.ac.jp/yanagawa/ex-2017/2-game/01.html)
- [cursesライブラリの超てきとー解説](https://www.kushiro-ct.ac.jp/yanagawa/pl2b-2018/curses/about.html)
- [[linux] cursesライブラリのインストール --- undefined reference 'initscr'|Debugging as Usual](http://debuggingasusual.blogspot.com/2011/12/curses.html)
- [C言語でシンプルすぎるブロック崩しを書いた - Qiita](https://qiita.com/pokohide/items/a246045f3ccaf540a375)
- [文字列の長さの取得(C言語) - 超初心者向けプログラミング入門](https://programming.pc-note.net/c/mojiretsu2.html)

#### 配列・メモリ操作

- [【C言語】文字列を連結・結合する【strcatの危険性とsnprintfの安全性】 | MaryCore](https://marycore.jp/prog/c-lang/concat-c-string/#snprintf関数による文字列結合)
- [【C言語】sprintf 関数と snprintf 関数（お手軽に文字列を生成する関数） | だえうホームページ](https://daeudaeu.com/c-sprintf/#sprintf-3)
- [【C言語】malloc関数（メモリの動的確保）について分かりやすく解説 | だえうホームページ](https://daeudaeu.com/c_malloc/)
- [配列を自由自在に作る - 苦しんで覚えるC言語](https://9cguide.appspot.com/19-01.html)
- [C言語の引数に多次元配列を渡す - Qiita](https://qiita.com/Hiraku/items/babed27bc1d750c2e12d)
- [配列の要素数を求める | Programming Place Plus　Ｃ言語編　逆引き](https://programming-place.net/ppp/contents/c/rev_res/array000.html)

#### その他

- [C言語ケーススタディ　時計の作り方1](http://www.orchid.co.jp/computer/cschool/clock1.html)
- [popenでコマンドの出力を読み込む - C言語入門](https://kaworu.jpn.org/c/popenでコマンドの出力を読み込む)
- [C言語のソースからバックグラウンドでシェルを実行したい](https://teratail.com/questions/29960)
- [C言語によるCSVファイルの読み込み方法 - なるぽのブログ](https://yu-nix.com/archives/c-read-csv/)
- [ファイルの存在を確認する | Programming Place Plus　Ｃ言語編　逆引き](https://programming-place.net/ppp/contents/c/rev_res/file000.html)

### ログローテーション

- [[Python] ログのファイル出力とログローテーション | HIROs.NET Blog](https://blog.hiros-dot.net/?p=10297)
- [Raspberry Piで記録する温度、湿度などのログをローテーションする - Qiita](https://qiita.com/mashi0727/items/d19186759c52cbba60e9)

### 天気情報の取得

- [curl で wttr.in に問い合わせて ターミナル上で天気予報を確認する - ブログ](https://gouf.hatenablog.com/entry/2018/06/29/174028)
- [天気を呟くbot｜シェルスクリプトで作る Twitter bot 作成入門](https://zenn.dev/mattn/books/bb181f3f4731920f29a5/viewer/cc50c48272963c206d34)

### アスキーアート

- [PythonとOpenCVで画像をアスキーアート化してみる（トレースAAへの道） | ねほり.com](https://nehori.com/nikki/2021/04/04/post-27881/)
- [アスキーアート - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%A2%E3%82%B9%E3%82%AD%E3%83%BC%E3%82%A2%E3%83%BC%E3%83%88)
