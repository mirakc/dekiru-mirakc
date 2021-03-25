# タイムシフト録画（全録）

> タイムシフト録画は，mirakc `0.17.0`以降で利用可能です．

Mirakurunには無い機能として，mirakcはタイムシフト録画（いわゆる全録）を実装しています．機能の細かい
説明は後回しにして，動かしてみましょう．

まず，タイムシフト録画用のフォルダーとファイルを作成します．

```console
$ mkdir timeshift
$ fallocate -l 1638400000 timeshift/nhk.timeshift.m2ts
$ fallocate -l 1638400000 timeshift/bs1.timeshift.m2ts
$ ls -lh timeshift
total 3.1G
-rw-r--r-- 1 pi pi 1.6G Mar 18 07:21 bs1.timeshift.m2ts
-rw-r--r-- 1 pi pi 1.6G Mar 18 07:19 nhk.timeshift.m2ts
```

ディスク容量が足りない場合，ファイル作成に失敗します．十分な空き容量を作ってから再チャレンジしてく
ださい．

フォルダーとファイルの作成に成功したら，`docker-compose.yml`を修正し，これを`mirakc`コンテナーにマ
ウントします．

```yaml
# 追加部分のみ抜粋
services:
  mirakc:
    volumes:
      - ./timeshift:/var/lib/mirakc/timeshift
```

NHK総合とBS1をタイムシフト録画するように`config.yml`を修正してみます．

```yaml
# 追加部分のみ抜粋
timeshift:
  recorders:
    nhk:
      service-triple: [32736, 32736, 1024]
      ts-file: /var/lib/mirakc/timeshift/nhk.timeshift.m2ts
      data-file: /var/lib/mirakc/timeshift/nhk.timeshift.json
      num-chunks: 10
    bs1:
      service-triple: [4, 16625, 101]
      ts-file: /var/lib/mirakc/timeshift/bs1.timeshift.m2ts
      data-file: /var/lib/mirakc/timeshift/bs1.timeshift.json
      num-chunks: 10
```

準備は整いました．mirakcを起動してください．

```console
$ sudo docker-compose up -d
```

タイムシフト録画が動いていれば，以下のように表示されます．

```console
$ curl -sG http://localhost:40772/api/timeshift | jq .[].name
"nhk"
"bs1"
```

起動直後は何も録画されていなので何も視聴できませんが，１，２分程度待つと視聴可能な状態になります．

```console
# 直後は0で視聴できないが，そのうち0以外が表示されるようになる
$ curl -sG http://localhost:40772/api/timeshift/bs1 | jq .duration
75420
```

`ffplay`がインストールされているデスクトップ環境上で以下のコマンドを実行すると，最初の番組からタイ
ムシフト再生されます．

```shell
# raspberrypi上ではなく，別のデスクトップ環境上で実行すること
curl -sG http://raspberrypi.local:40772/api/timeshift/bs1/stream | ffplay -
```

上記のWeb APIはタイムシフト録画をライブ視聴相当でストリーミングします．そのため，シークはできません
が，視聴中の番組が終了したら次の番組が自動で再生されます．

特定の番組のみを視聴したい場合は，以下のWeb APIを使います．

```shell
ID=$(curl -sG http://raspberrypi.local:40772/api/timeshift/bs1/records | jq .[0].id)
curl -sG http://raspberrypi.local:40772/api/timeshift/bs1/records/$ID/stream | ffplay -
```

こちらはWeb API呼び出し時に録画されているところまでが再生可能範囲になります．シーク可能ですが，録
画中の番組を再生した場合，途中までしか再生されません．普通は録画完了した番組の再生に使います．

## タイムシフト録画とは？

冒頭でも書きましたが，mirakcのタイムシフト録画は，いわるゆ全録機能です．以下のような特徴を持ちま
す．

* 固定サイズのファイルをリングバッファーとして録画を行ないます
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
  * `ts-file`の最大ファイルサイズは，チャンクサイズ（既定値は`163840000`で約160MB）に`num-chunks`
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

チャンクサイズの既定値は`163840000`で約160MBです．タイムシフト録画対象のサービスに依存しますが，お
およそ１〜３分に相当するサイズです．例えば，BS1を一週間分タイムシフト録画しようと考えた場合，

* BS1のTSストリームのビットレートは約20Mbps
* 一分間の録画に必要なサイズは`20M bits * 60 / 8`で約150MB
* `150 * 60 * 24 * 7 = 1512000`で約1.5TB

となります．１チャンクで大体１分間録画できるので，`num-chunks`には`1512000`を指定すればよいという
計算になります．

TSストリームのビットレートはサービスごとに異なるため，タイムシフト録画の期間を決めてからチャンク数
を求めたい場合には，サービスごとにビットレートを計測する必要があります．一方，タイムシフト録画に使
用するディスクサイズが既に決まっている場合には，それを超えないようにチャンク数を決めることになりま
す．この場合はビットレートの計測は不要で，単に割り当てる最大ファイルサイズをチャンクサイズで割れば
よいだけです．

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
す．`data-file`を削除すると，`ts-file`があっても，どこにどの番組が録画されているのかわからなくな
り，録画された番組はないものとして扱われます．そのため，`data-file`を定期的に更新する必要があります
が，これがチャンク単位で行われるようになっています．

## タイムシフト・ファイルシステム

mirakcはバックエンド機能しか提供しません．そのため，タイムシフト録画した番組を再生する場合，`curl`
や`jq`などを使って，必要な情報を取得する必要があります．Web UIなどのフロントエンドを実装すればよい
のでしょうが，今の所はその計画はありません．

タイムシフト録画した番組を再生できるとはいえ，流石にこのままでは不便なので（作った本人である私も）
使いたくはありません．そこで，[FUSE]を使ってタイムシフト録画した番組を普通のファイルとして扱えるフ
ァイルシステムを用意しました．

以下を`docker-compose.yml`に追記してください．

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
```

`timeshift-fs`フォルダーを作成し，追加したコンテナーを起動します．

```
mkdir timeshift-fs
sudo docker-compose up -d mirakc-timeshift-fs
```

起動に成功すれば，`timeshift-fs`にタイムシフト・ファイルシステムがマウントされます．

```console
$ ls timeshift-fs
bs1 nhk

$ ls timeshift-fs/bs1
6052B1C8_ＢＳニュース.m2ts
```

タイムシフト・ファイルシステムは読み取り専用なので書き込みはできません．録画が進むとファイルが増え
たり減ったりします．

普通のファイルとして扱われるので，あとは煮るなり焼くなり好きにできます．試しにSambaでファイル共有
してみましょう．

`docker-compose.yml`を書き換え

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
      RUST_LOG: info
```

`samba`コンテナーを起動します．

```shell
sudo docker-compose up -d samba
```

エクスプローラーやFinderでSambaに接続し，MPEG2-TSを再生可能なメディアプレーヤーでファイルを開けば，
番組が再生されるはずです．

macOSユーザーには，Finderのプレビューを無効にすることをお勧めします．これを有効にしていると，動画
のサムネイル画像の生成処理が開始され，Finderが操作不能になる場合があります．また，アプリケーション
のメニューからファイルを開くと，Finderのプレビューを無効にしているのに，なぜかサムネイル画像を生成
しようとすることがあるようなので，m2tsファイルをメディアプレーヤーに関連付けて，`open`コマンドで開
くことをお勧めします．

[FUSE]: https://en.wikipedia.org/wiki/Filesystem_in_Userspace
