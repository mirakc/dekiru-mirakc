# タイムシフト録画（全録）

Mirakurunには無い機能として，mirakcはタイムシフト録画（いわゆる全録）を実装しています．機能の細かい
説明は後回しにして，動かしてみましょう．

まず，タイムシフト録画用のフォルダーとファイルを作成します．

```console
$ mkdir timeshift
$ fallocate -l 1540096000 timeshift/nhk-bs.timeshift.m2ts
$ ls -lh timeshift
total 3.1G
-rw-r--r-- 1 pi pi 1.5G Mar 18 07:21 nhk-bs.timeshift.m2ts
```

ディスク容量が足りない場合，ファイル作成に失敗します．十分な空き容量を作ってから再チャレンジしてく
ださい．

フォルダーとファイルの作成に成功したら，`compose.yaml`を修正し，これを`mirakc`コンテナーにマ
ウントします．

```yaml
# 追加部分のみ抜粋
services:
  mirakc:
    volumes:
      - ./timeshift:/var/lib/mirakc/timeshift
```

NHK BSをタイムシフト録画するように`config.yml`を修正してみます．

```yaml
# 追加部分のみ抜粋
tuners:
  - name: nhk-bs-timeshift
    types: [BS]
    command: recpt1 --device /dev/px4video1 {{{channel}}} - -

jobs:
  # 通常，タイムシフト録画では以下のジョブは不要
  # 無効化すると`/api/programs`などが機能しなくなる
  sync-clocks:
    disabled: true
  update-schdules:
    disabled: true

timeshift:
  recorders:
    nhk-bs:
      service-id: 400101
      ts-file: /var/lib/mirakc/timeshift/nhk-bs.timeshift.m2ts
      data-file: /var/lib/mirakc/timeshift/bhk-bs.timeshift.json
      num-chunks: 10
      uses:
        tuner: nhk-bs-timeshift
        channel-type: BS
        channel: BS15_0
```

準備は整いました．mirakcを起動してください．

```console
$ sudo docker compose up -d
```

タイムシフト録画が動いていれば，以下のように表示されます．

```console
$ curl -sG http://localhost:40772/api/timeshift | jq .[].name
"nhk-bs"
```

起動直後は何も録画されていないので何も視聴できませんが，１，２分程度待つと視聴可能な状態になります．

```console
# 直後は0で視聴できないが，そのうち0以外が表示されるようになる
$ curl -sG http://localhost:40772/api/timeshift/nhk-bs | jq .duration
75420
```

`ffplay`がインストールされているデスクトップ環境上で以下のコマンドを実行すると，最初の番組からタイ
ムシフト再生されます．

```shell
# raspberrypi上ではなく，別のデスクトップ環境上で実行すること
curl -sG http://raspberrypi.local:40772/api/timeshift/nhk-bs/stream | ffplay -
```

上記のWeb APIはタイムシフト録画をライブ視聴相当でストリーミングします．そのため，シークはできません
が，視聴中の番組が終了したら次の番組が自動で再生されます．

特定の番組のみを視聴したい場合は，以下のWeb APIを使います．

```shell
ID=$(curl -sG http://raspberrypi.local:40772/api/timeshift/nhk-bs/records | jq .[0].id)
curl -sG http://raspberrypi.local:40772/api/timeshift/nhk-bs/records/$ID/stream | ffplay -
```

こちらはWeb API呼び出し時に録画されているところまでが再生可能範囲になります．シーク可能ですが，録
画中の番組を再生した場合，途中までしか再生されません．普通は録画完了した番組の再生に使います．

## タイムシフト録画とは？

冒頭でも書きましたが，mirakcのタイムシフト録画は，いわるゆ全録機能です．以下のような特徴を持ちま
す．

* 固定サイズのファイルをリングバッファーとして，指定サービスのTSストリームを録画します
  * サイズは`config.yml`で指定します
* サイズ上限に達すると，古い録画から順番に消えていきます
  * ファイルをある固定サイズのチャンクに分割し，この単位で消していきます
* mirakcを停止しても録画は消えません
  * 再起動しても前の録画を残し，続きから録画を再開します

固定サイズのファイルを使って録画を行うため，タイムシフト録画を行ない続けても，ディスクを際限なく消
費してしまうことはありません．

タイムシフト録画が設定されると，mirakcは指定されたサービスの番組をひたすら録画し続けます．このよう
な性質の機能であることから，タイムシフト録画を有効にしたmirakcを起動する場合は，以下の条件を満たし
ておく必要があります．

* タイムシフト録画の数以上のチューナーを用意する
  * mirakcを起動することは可能ですが，タイムシフト録画数はチューナー数を超えません
* 十分なディスク容量を用意する
  * 容量が足りないと書き込みエラーが発生して機能しないため，予め`fallocate`などで`ts-file`を最大サ
    イズで確保しておくことが望ましいです
  * `ts-file`の最大ファイルサイズは，チャンクサイズ（既定値は`154009600`で約154MB）に`num-chunks`
    を掛けた数
  * `data-file`の最大ファイルサイズは計算不可能ですが，多くても数MB程度
* 常時起動
  * NASに書き込むのはお勧めしません（NASのファームアップデート時に停止する必要があるため）

このような条件を満たす必要があるため，タイムシフト録画用のmirakcは専用の独立サーバーとして運用する
ことをお勧めします．その上で，タイムシフト録画の対象となるチャンネルのみを`config.yml`に定義します．
こうしておけば，EPG関連データの取得に失敗することはなくなります．なぜなら，常時チューナーが開いて
いるためです．一方，タイムシフト録画の対象になっていないチャンネルを`config.yml`に定義していると，
そのチャンネルのEPG関連データの取得のためチューナーを開こうとするため，チューナーの数がタイムシフ
ト録画の数以上存在しないとエラーが発生します．

## タイムシフト録画に必要なチャンク数の計算方法

先に説明したように，タイムシフト録画用の`ts-file`は，チャンクサイズ（`chunk-size`）とチャンク数（
`num-chunks`）で決まります．

```
ts-file-max-size = chunk-size * num-chunks
```

タイムシフト録画に使用するディスクサイズが既に決まっている場合には，それを超えないようにチャンク数
を決めることになります．例えば，既定値のチャンクサイズで１チャンネルを最大1TBだけタイムシフト録画し
たい場合は，

```
num-chunks = ts-file-max-size / chunk-size = 1000000000000 / 154009600 = 6493
```

となります．

録画期間からチャンク数を計算する場合は，少し面倒な計算が必要です．例えば，NHK BSを一週間分タイムシフト
録画しようと考えた場合，

* NHK BSのTSストリームのビットレートは約20Mbps
* 一分間の録画に必要なサイズは`20M bits * 60 / 8`で約150MB（既定値の１チャンクで大体１分間録画でき
  る）
  * タイムシフト録画対象のサービスに依存しますが，既定値の１チャンクは約１〜３分に相当するサイズで
    す
* `150 * 60 * 24 * 7 = 1512000`で約1.5TB

となります．これらを計算式に当てはめると，

```
num-chunks = 1512000000000 / 154009600 = 9817
```

となるため，`num-chunks`に`10000`を指定しておけば，余裕を持って一週間分タイムシフト録画できると期待
できます．この場合の`ts-file`の最大ファイルサイズは，`1540096000000`（約1.5TB）です．

TSストリームのビットレートはサービスごとに異なるため，タイムシフト録画の期間を決めてからチャンク数
を求めたい場合には，サービスごとにビットレートを計測する必要があります．また，ビットレートは固定で
はなく時間変動するため，上記計算は目安程度と考えてください．

以下は実績値です．参考にしてください．

|サービス           |chunk-size       |num-chunks|録画可能期間|
|------------------|-----------------|----------|----------|
|ＮＨＫ総合１・東京  |154009600（既定値）|14000     |二週間程度 |
|ＮＨＫＥテレ１東京  |154009600（既定値）|14000     |二週間程度 |
|ＮＨＫＢＳ１       |154009600（既定値）|18000     |二週間程度 |
|ＮＨＫＢＳプレミアム|154009600（既定値）|18000     |二週間程度 |

上記各タイムシフト録画の`data-file`はどれも2MBを超えません．そのため，上記設定で４チャンネルのタイ
ムシフト録画を10TBのHDDで行うことが可能です．

`config.yml`の設定値に基いて必要なサイズのファイルを確保するスクリプトを用意してあります．

```console
deno run -A \
  https://raw.githubusercontent.com/mirakc/contrib/main/timeshift/allocate-ts-file.js \
  -c /path/to/config.yml
```

詳細については`-h`で表示されるヘルプを見るか，[スクリプトファイル](https://github.com/mirakc/contrib/blob/main/timeshift/allocate-ts-file.js)
の内容を直接確認してください．

## データ管理方法

mirakcのタイムシフト録画では，録画データ（TSパケット）ファイル（`ts-file`）をチャンクという塊で管理
します．そのため，番組の録画が進んでもチャンクがいっぱいになるまで録画済みとは扱われません．

```
ts-file
+-------------------------------------------------
| Chunk#0 | Chunk#1 | Chunk#2 | Chunk#3 | ...
|xxxxxxxxx|xxx______|_________|xxxxxxxxx|x...
+-------------------------------------------------
|         |   |               |
+-番組#100-+   +-番組#10だがもう-+-番組#10（途中から)---
                見ることはできない
```

上記では`Chunk#0`はデータで埋まっているため録画済みとして扱われます．一方，`Chunk#1`は現在書き込み
を行っているチャンクで，データで埋まるまで録画画済みとは扱われません．`Chunk#1`には過去の録画データ
が記録されている場合がありますが，データは上書きされていきます．

`Chunk#2`にも録画データが書き込まれているかもしれませんが，このデータは古くなったものとして扱われま
す．その結果，古い番組のデータが消えていきます．

TSパケットの他に，録画済み番組情報などを記録しておく必要があり，`data-file`はそのために使用されま
す．`data-file`を定期的に更新する必要がありますが，これがチャンク単位で行われるようになっています．
`data-file`を削除すると，`ts-file`があっても，どこにどの番組が録画されているのかわからなくなり，録
画された番組はないものとして扱われます．

## タイムシフト・ファイルシステム

> [`mirakc/timeshift-gerbera`]もしくは[`mirakc/timeshift-samba`]を使えば，以下の説明に出てくるよう
> な複雑なマウント処理は必要ありません．開発処理にはこれらのDockerイメージは提供していなかったとい
> う歴史的経緯により，以下の説明では２つのコンテナー間でタイムシフト・ファイルシステムを共有するた
> めに複雑なマウント処理を行っています．

mirakcはバックエンド機能しか提供しません．そのため，タイムシフト録画した番組を再生する場合，`curl`
や`jq`などを使って，必要な情報を取得する必要があります．Web UIなどのフロントエンドを実装すればよい
のでしょうが，今の所はその計画はありません．

タイムシフト録画した番組を再生できるとはいえ，流石にこのままでは不便なので（作った本人である私も）
使いたくはありません．そこで，[FUSE]を使ってタイムシフト録画した番組を普通のファイルとして扱えるフ
ァイルシステムを用意しました．

以下を`compose.yaml`に追記してください．

```yaml
services:
  ...
  mirakc-timeshift-fs:
    container_name: mirakc-timeshift-fs
    image: mirakc/timeshift-fs
    init: true
    restart: unless-stopped
    cap_add:
      - SYS_ADMIN
    devices:
      - /dev/fuse
    volumes:
      # タイムシフト録画しているmirakcと同じ設定を使うこと
      - ./config.yml:/etc/mirakc/config.yml:ro
      # タイムシフト録画ファイルをコンテナーにマップ
      - ./timeshift:/var/lib/mirakc/timeshift
      # mirakc/timeshift-fsはコンテナー内の/mntにタイムシフト・ファイルシステムをマウントする
      # 下記はこれをDockerホストの./timeshift-fsに反映させる
      - type: bind
        source: ./timeshift-fs
        target: /mnt
        bind:
          propagation: rshared
    environment:
      TZ: Asia/Tokyo
      RUST_LOG: info
      # タイムシフト・ファイルシステムの所有者の設定
      # ログイン・ユーザーのUID/GIDを指定
      MIRAKC_TIMESHIFT_UID: 1000
      MIRAKC_TIMESHIFT_GID: 1000
      MIRAKC_TIMESHIFT_MOUNT_OPTIONS: ALLOW_OTHER
      # ファイル名にレコードの開始時間を含める場合には以下を設定
      # ファイル作成日時のファイル属性をサポートしないファイルシステムで有用
      #MIRAKC_TIMESHIFT_FS_START_TIME_PREFIX: true
```

`timeshift-fs`フォルダーを作成し，追加したコンテナーを起動します．

```
mkdir timeshift-fs
sudo docker compose up -d mirakc-timeshift-fs
```

起動に成功すれば，`timeshift-fs`にタイムシフト・ファイルシステムがマウントされます．

```console
$ ls timeshift-fs
nhk-bs

$ ls timeshift-fs/nhk-bs
6052B1C8.ＢＳニュース.m2ts
```

タイムシフト・ファイルシステムは読み取り専用なので書き込みはできません．録画が進むとファイルが増え
たり減ったりします．

普通のファイルとして扱われるので，あとは煮るなり焼くなり好きにできます．試しにSambaでファイル共有
してみましょう．

`compose.yaml`を書き換え

```yaml
services:
  ...
  samba:
    depends_on:
      # ./timeshift-fsへのタイムシフト・ファイルシステムのマウント完了後に起動する
      - mirakc-timeshift-fs
    container_name: samba
    image: dperson/samba
    command:
      # LAN（192.168.0.0/16）からのアクセスのみ許可
      - '-g'
      - 'hosts allow = 192.168.'
      # timeshiftフォルターとして公開（ゲスト・アクセス可）
      - '-s'
      - 'timeshift;/mnt'
    init: true
    restart: unless-stopped
    ports:
      - '139:139'
      - '445:445'
    volumes:
      # タイムシフト・ファイルシステムをコンテナー内にマップ
      - ./timeshift-fs:/mnt:ro
    environment:
      TZ: Asia/Tokyo
```

`samba`コンテナーを起動します．

```shell
sudo docker compose up -d samba
```

エクスプローラーやFinderでSambaに接続し，MPEG2-TSを再生可能なメディアプレーヤーでファイルを開けば，
番組が再生されるはずです．

macOSユーザーには，Finderのプレビューを無効にすることをお勧めします．これを有効にしていると，動画
のサムネイル画像の生成処理が開始され，Finderが操作不能になる場合があります．また，アプリケーション
のメニューからファイルを開くと，Finderのプレビューを無効にしているのに，なぜかサムネイル画像を生成
しようとすることがあるようなので，m2tsファイルをメディアプレーヤーに関連付けて，`open`コマンドで開
くことをお勧めします．

どうもタイムシフト・ファイルシステムのアンマウントがうまくできないようで，それが原因で
`mirakc-timeshift-fs`コンテナーの再起動に失敗するようです．そのような場合は，以下のようにアンマウン
ト後に`mirakc-timeshift-fs`コンテナーを起動してください．

```shell
sudo umount ./timeshift-fs
```

## 多重音声（デュアルモノラル）について

タイムシフト録画は，単にTSパケットをファイルに書き込むだけなので，多重音声（デュアルモノラル）な録
画を再生する場合，メディアプレーヤーがこれをサポートしていないと２つの音声（例えば，日本語と英語）
が同時に再生されます．

多重音声（デュアルモノラル）をサポートしているメディアプレーヤーは数が限られています．例えば，日本
向けのTVなどでは音声切り替え可能ですが，Kodiなどはサポートしていません．

非サポートのメディアプレーヤーで正しく再生するためには，再生前にトランスコードする必要があります．
[フィルター設定](./filters.md)で説明しましたが，mirakcのフィルターを使えば，わざわざトランスコード済
みのファイルを作成する必要はありません．

以下の`split-dual-mono`フィルターは，ステレオ音声を２つの音声に分離します．

```yaml
# ffmpegがインストール済みのカスタムイメージを使うこと
filters:
  post-filters:
    # See https://trac.ffmpeg.org/wiki/AudioChannelManipulation
    split-dual-mono:
      command: >-
        ffmpeg -i - -hide_banner -vcodec copy
        -filter_complex "[0:a]channelsplit=channel_layout=stereo"
        -f mpegts pipe:1
```

音声トラックを切り替え可能なメディアプレーヤーで再生すれば，目的の言語の音声のみを再生できます．

```shell
# ステレオ音声が２つのモノラル音声に分離されることを確認
# 実際は多重音声（デュアルモノラル）の録画にフィルターを適用する
STREAM=http://raspberrypi.local:40772/api/timeshift/nhk-bs/stream
curl -sG "$STREAM?post-filters[]=split-dual-mono" | ffprobe
```

コンソール上のログから，ステレオ音声が分離されていることを確認できます．

```console
Input #0, mpegts, from 'pipe:':
  Duration: N/A, start: 1.400000, bitrate: N/A
  Program 1
    Metadata:
      service_name    : Service01
      service_provider: FFmpeg
    Stream #0:0[0x100]: Audio: mp2 ([3][0][0][0] / 0x0003), 48000 Hz, mono, fltp, 384 kb/s
    Stream #0:1[0x101]: Audio: mp2 ([3][0][0][0] / 0x0003), 48000 Hz, mono, fltp, 384 kb/s
    Stream #0:2[0x102]: Video: mpeg2video (Main) ([2][0][0][0] / 0x0002), yuv420p(tv, bt709, top first), 1440x1080 [SAR 4:3 DAR 16:9], 29.97 fps, 29.97 tbr, 90k tbn, 59.94 tbc
    Side data:
      cpb: bitrate max/min/avg: 20000000/0/0 buffer size: 9781248 vbv_delay: N/A
```

欠点として，post-filterでストリームを変換した場合，シークできなくなります．変換によりストリームが動
的に生成されるため，HTTPレスポンスは`Content-Length`ヘッダーを含まず，HTTPレスポンス・ボディにはチ
ャンク・エンコーディングが適用されるためです．

## タイムシフト録画の再実行

タイムシフト録画が何らかの理由で停止した場合，自動でタイムシフト録画を再実行します．

```
INFO mirakc_core::timeshift: Recording stopped recorder.name="nhk-bs"
...
INFO mirakc_core::timeshift: Recording started recorder.name="nhk-bs"
```

タイムシフト録画が停止している間も，録画データにアクセス可能です．

放送休止中にタイムシフト録画が停止することをユーザーが報告しています．

* https://github.com/mirakc/mirakc/issues/693

[FUSE]: https://en.wikipedia.org/wiki/Filesystem_in_Userspace
[`mirakc/timeshift-gerbera`]: https://github.com/mirakc/docker-timeshift-x/tree/main/gerbera
[`mirakc/timeshift-samba`]: https://github.com/mirakc/docker-timeshift-x/tree/main/samba
