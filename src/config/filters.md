# フィルター設定

> 以下の`config.yml`の記述例は，`0.12.0`以降でのみ有効です．これより古いバージョ
> ンでのフィルターの記述方法については，GitHubの履歴を確認してください．

EPGデータの取得が完了すると，特定サービスのTSストリーミングが可能となります．

```console
# mirakcコンテナーをバックグラウンドで起動
$ sudo docker compose up -d
Starting mirakc ... done

# 最初のサービス（ＮＨＫ総合１）のIDを表示
$ curl -s http://localhost:40772/api/services | jq .[0].id
3273601024

# ＮＨＫ総合１のTSストリーミングを実行（Ctrl+Cで停止）
$ curl http://localhost:40772/api/services/3273601024/stream >/dev/null
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.8M    0 16.8M    0     0  1578k      0 --:--:--  0:00:10 --:--:-- 1725k
```

しかし，今の設定のままでは，TSストリームを`ffmpeg`で変換したりすることはできませ
ん．なぜならば，フィルターが設定されていないためです．

フィルターは，TSストリームを標準入力から受け取り，処理後のTSストリームを標準出力
に出力する外部プログラムです．mirakc自体にはこのような機能は実装されておらず，
フィルター設定で指定された外部プログラムにTSストリームの処理を委譲します．

フィルターの動作を確認するため，以下のような設定を行ってみます．

`config.yml`:

```yaml
filters:
  decode-filter:
    command: >-
      echo "{{{channel_type}}}/{{{channel}}} SID#{{{sid}}}"
```

設定反映のためmirakcコンテナーを再起動し，動作確認します．

```console
# mirakcコンテナーを再起動
$ sudo docker compose restart

# post-filter付きTSストリーミングを実行
$ curl -s http://localhost:40772/api/services/3273601024/stream?decode=1
GR/27 SID#1024
```

TSストリームをチャンネル情報に置き換えることに成功しました．

なお，`filters.decode-filter`は，Mirakurunの`tuners.yml`での`decoder`に相当する設
定です．

フィルターの動作確認も終了したので，実用的なフィルターを設定してみます．

```yaml
filters:
  # Chinachu/Mirakurunの設定例より
  decode-filter:
    command: >-
      arib-b25-stream-test
```

また，以下のように外部サーバーでTSストリームを処理し，その結果をクライアントに返
すことも可能です．

```yaml
filters:
  decode-filter:
    # 標準入力から入力されるTSストリームをリモートホストtsdのTCP 40773ポートに転
    # 送し，tsdから返されたTSストリームを標準出力に出力
    command: >-
      socat - tcp-connect:tsd:40773
```

[socat]は，配布している`mirakc/mirakc`イメージにも含まれているツールです．詳し
い説明はリンク先を見るかGoogle先生に教えてもらってください．

`filters`プロパティーはチューナーごとに設定するのではなく，すべてのチューナーに
対して共通で利用される設定です．そのため，特定のチューナーに対して特別な処理を行
いたい場合は，渡されたテンプレートパラメーターを利用して処理を分岐するようなスク
リプトを作成し，これをフィルターとして設定する必要があります．

試しに，以下のような簡単な`bs-not-supported`スクリプトを作って，動作を確認してみ
ます．

```shell
#!/bin/sh

CHANNEL_TYPE="$1"
CHANNEL="$2"

if [ "$CHANNEL_TYPE" = "GR" ]; then
  echo "$CHANNEL"
else
  echo "BS not supported"
fi
```

`config.yml`:

```yaml
# 変更部分のみ記載
filters:
  decode-filter:
    command: >-
      bs-not-supported {{{channel_type}}} {{{channel}}}
```

`docker-compose.yml`:

```yaml
# 変更部分のみ記載
...
    volumes:
      - mirakc-epg:/var/lib/mirakc/epg
      - ./config.yml:/etc/mirakc/config.yml:ro
      # 忘れずに`chmod +x bs-not-supported`しておくこと
      - ./bs-not-supported:/usr/local/bin/bs-not-supported:ro
...
```

以下のように表示されれば，`bs-not-supported`スクリプトは機能しています．

```console
$ curl http://localhost:40772/api/channels/GR/27/stream?decode=1
27

$ curl http://localhost:40772/api/channels/BS/BS15_0/stream?decode=1
BS not supported
```

[socat]: http://www.dest-unreach.org/socat/doc/socat.html
