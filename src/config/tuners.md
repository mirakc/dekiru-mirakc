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

ここで，`tuners[].command`にはテンプレート文字列が指定されており，`{{{`と`}}}`
（Triple Mustache）または`{{`と`}}`（Double Mustache）で囲われた識別子（上記の例
では`channel`）は，テンプレートパラメーターと呼ばれる特別な変数です．mirakcは，テ
ンプレート言語として[Mustache]を採用しており，リスト型のテンプレートパラメーター
の展開もサポートしています．`tuners[].command`以外にもいくつかの設定項目でテンプ
レート文字列が指定可能で，設定項目毎に使用可能なテンプレートパラメーターは異なり
ます．

> Double MustacheとTriple Mustacheでは動作が異なる点に注意してください．
> 詳細は[こちら](https://mustache.github.io/mustache.5.html)に書かれています．

mirakcのチューナー設定には`decoder`プロパティは存在しません．その代わりに，後述
するフィルター設定を使用します．

Dockerを使用している場合，以下のように`compose.yaml`に使用するデバイスファ
イルを記述する必要があります．

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

ここで，各行の始めの`...`は実際に表示されるテキストではなく，ログの省略を意味し
ます．

たくさんのログが表示されますが，正しく動作していれば，上記のようなログを見つける
ことができます．

## 既に稼働しているMirakurun/mirakcをチューナーとして利用する

ちょっとした動作確認のために，わざわざチューナーのデバイスドライバーをインストー
ルするのは面倒な作業です．このような場合には，以下のような設定を行い，既に稼働し
ているMirakurunまたはmirakcをチューナーとして利用することをお勧めします（同様の
ことはMirakurunでも可能です）．

```yaml
tuners:
  - name: upstream
    types: [GR, BS]
    command: >-
      curl -s http://upstream:40772/api/channels/{{{channel_type}}}/{{{channel}}}/stream?decode=0
```

上記の`http://upstream:40772`の部分は接続先のMirakurunまたはmirakcサーバーのURL
に置き換えてください．

Mirakurunとは異なり，mirakcの`/api/channels/{{{channel_type}}}/{{{channel}}}/stream?decode=0`
APIは，チューナーからの出力をそのまま送出します．NULLパケットすらドロップしま
せん．そのため，TSパケットを処理するツールのテストなどにも使えて便利です．

```shell
curl -s http://mirakc:40772/api/channels/GR/27/stream?decode=0 | \
  MIRAKC_ARIB_LOG=debug mirakc-arib scan-services
```

## 設定例

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

[Mustache]: https://mustache.github.io/
