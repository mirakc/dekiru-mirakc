# チューナー設定

チャンネル設定と同様に，チューナー設定もMirakurunと一定の互換性があります．

Mirakurunの`tuners.yml`の例:

```yaml
- name: gr1
  types: [GR]
  command: recpt1 --device /dev/px4video2 <channel> - -
  decoder: /usr/local/bin/decoder
```

mirakcの`config.yml`の例:

```yaml
tuners:
  - name: gr1
    types: [GR]
    command: recpt1 --device /dev/px4video2 {{{channel}}} - -
```

ここで，`tuners[].command`にはテンプレート文字列が指定されており，`{{{`と`}}}`（Triple Mustache）ま
たは`{{`と`}}`（Double Mustache）で囲われた識別子（上記の例では`channel`）は，テンプレートパラメータ
ーと呼ばれる特別な変数です．mirakcは，テンプレート言語として[Mustache]を採用しており，リスト型のテン
プレートパラメーターの展開もサポートしています．`tuners[].command`以外にもいくつかの設定項目でテンプ
レート文字列が指定可能で，設定項目毎に使用可能なテンプレートパラメーターは異なります．

> [!IMPORTANT]
> Double MustacheとTriple Mustacheでは動作が異なる点に注意してください．
> 詳細は[こちら](https://mustache.github.io/mustache.5.html)に書かれています．

mirakcのチューナー設定には`decoder`プロパティは存在しません．その代わりに，後述するフィルター設定を
使用します．

Dockerを使用している場合，以下のように`compose.yaml`に使用するデバイスファイルを記述する必要がありま
す．

```yaml
services:
  mirakc:
    image: mirakc/mirakc:alpine
    container_name: mirakc
    init: true
    restart: unless-stopped
    # `devices`を追加
    devices:
      # チューナーコマンドで利用するデバイスファイルを列挙
      - /dev/px4video2
    ports:
      - 40772:40772
    volumes:
      - ./config.yml:/etc/mirakc/config.yml:ro
    environment:
      TZ: Asia/Tokyo
      RUST_LOG: info
```

ここまで設定すると，EPGデータの取得が可能になります．動くか試してみましょう．

```console
# mirakcコンテナーの起動（Ctrl+Cで停止）
$ sudo docker compose up
...
... scan-services: performing...
...
... scan-services: Done successfully, 13s 714ms 157us 639ns elapsed
...
^CGracefully stopping... (press Ctrl+C again to force)
Stopping mirakc ... done
```

ここで，各行の始めの`...`は実際に表示されるテキストではなく，ログの省略を意味します．

たくさんのログが表示されますが，正しく動作していれば，上記のようなログを見つけることができます．

## 既に稼働しているMirakurun/mirakcをチューナーとして利用する {#mirakurun}

ちょっとした動作確認のために，わざわざチューナーのデバイスドライバーをインストールするのは面倒な作業
です．このような場合には，以下のような設定を行い，既に稼働しているMirakurunまたはmirakcをチューナー
として利用することをお勧めします（同様のことはMirakurunでも可能です）．

```yaml
tuners:
  - name: mirakurun
    types: [GR, BS]
    command: >-
      curl 'http://mirakurun:40772/api/channels/{{{channel_type}}}/{{{channel}}}/stream?decode=0' -sG
```

上記の`http://mirakurun:40772`の部分は接続先のMirakurunまたはmirakcサーバーのURLに置き換えてくださ
い．

Mirakurunとは異なり，mirakcの`/api/channels/{{{channel_type}}}/{{{channel}}}/stream?decode=0`APIは，
チューナーからの出力をそのまま送出します．NULLパケットすらドロップしません．そのため，TSパケットを処
理するツールのテストなどにも使えて便利です．

```shell
curl 'http://mirakc:40772/api/channels/GR/27/stream?decode=0' -sG | \
  MIRAKC_ARIB_LOG=debug mirakc-arib scan-services
```

## 設定例 {#examples}

一般的に利用されているチューナーコマンドに対する設定例を以下に記載します．

```yaml
# typesプロパティ及びデバイスファイルのパスなどは，自分の環境に合わせて書き換え
# る必要があります
tuners:
  - name: recpt1
    types: [GR]
    command: >-
      recpt1 --device /path/to/dev {{{channel}}} - -

  - name: recdvb
    types: [GR]
    command: >-
      recdvb --dev 1 {{{channel}}} - -

  - name: dvbv5-zap
    types: [GR]
    command: >-
      dvbv5-zap -a 0 -c /path/to/conf -r -P {{{channel}}} -o -
```

## チャンネル制限 {#excluded-channels}

> [!IMPORTANT]
> チャンネル制限は3.3.0以降でのみ利用可能です

複数のチューナーがあり，[何らかの理由](https://github.com/mirakc/mirakc/issues/2156)で特定のチューナ
ーで利用可能なチャンネルを制限したい場合，`excluded-channels`プロパティで設定可能です．

```yaml
channels:
  # このチャンネルはtuner-1では何故か開けない．．．
  # excluded-channelsを指定しないとtuner-1が使われた場合に必ず処理が失敗する
  - name: channel-1
    type: GR
    channel: '99'

tuners
  - name: tuner-1
    types: [GR]
    command: ...
    excluded-channels:
      # channel-1がユニークなチャンネル名であるなら，nameで指定できる
      - name: channel-1
```

上記の例では，`channel-1`を`tuner-1`で開かないように設定しています．`channel-1`を名前に持つチャンネ
ルが１つ以下（ユニークなチャンネル名）の場合，`name`プロパティを使って指定できます．

`channel-1`を名前に持つチャンネルが複数存在する場合は`name`プロパティは使えません．代わりに`params`
プロパティを使って指定します．

```yaml
tuners
  - name: tuner-1
    types: [GR]
    command: ...
    excluded-channels:
      # channel-1をnameに持つチャンネルが複数ある場合，paramsで指定しなければならない
      - params:
          channel-type: GR
          channel: '99'
```

[Mustache]: https://mustache.github.io/
