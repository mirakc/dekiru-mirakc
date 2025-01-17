# イベント通知

mirakcはServer-Sent Events ([SSE]) によるイベント通知をサポートしています．これを
利用することで，特定のイベントが発生したときにスクリプトを実行できます．

すでにmirakcが起動している場合，以下のコマンドを実行してみてください．通知された
イベントが表示されるはずです．番組情報の取得が完了していない場合は表示されません．
番組情報の取得が完了するまで待ちましょう．

```console
$ curl http://localhost:40772/events -sG
event:epg.programs-updated
data:{"serviceId":400101}

event:epg.programs-updated
data:{"serviceId":3273601025}

event:epg.programs-updated
data:{"serviceId":3273601024}
```

それでは，`events.epg.programs-updated`イベントを使って，簡単なルール・ベースの
自動録画予約機能を動かしてみます．

`compose.yaml`

```yaml
services:
  recman:
    container_name: recman
    # arm32環境ではlukechannings/denoは動作しない
    # arm32環境では別のイメージを使うこと
    image: lukechannings/deno
    init: true
    restart: unless-stopped
    command: >-
      run --allow-net=mirakc
      https://raw.githubusercontent.com/mirakc/contrib/main/recording/simple-rules.js
      -b http://mirakc:40772 --folder videos
    environment:
      TZ: Asia/Tokyo
    depends_on:
      - mirakc
```

`sample-rules.js`の実行には[`deno`]を使用します．`node`ではないので注意してください．

`recman`コンテナーを起動しログを確認します．

```console
$ docker logs -f recman
...
Connect to http://mirakc:40772/events...
EPG programs updated: ＮＨＫ総合１・東京: 3273601024
EPG programs updated: ＮＨＫ総合２・東京: 3273601025
EPG programs updated: ＮＨＫＢＳ１: 400101
Added: scheduled: ＮＨＫ総合１・東京: 327360102422829: ...
...
```

スクリプトの実行完了をしばらく待って以下を実行すると，録画予約が自動登録されてい
ることを確認できます．

```console
$ curl http://localhost:40772/api/recording/schedules -sG | jq '.[].program.id'
327360102422829
327360102423293
327360102423762
327360102424202
327360102424706
327360102425360
327360102426083

$ curl http://localhost:40772/api/programs/327360102422829 -sG | \
    jq -r '.name | test("ＮＨＫニュース７")'
true
```

上記のスクリプトでは`epg.programs-updated`イベントのみを使っていますが，
放送中の番組の追跡機能を有効化した場合に通知される`onair.program-changed`イベン
トを使えば，放送中の番組（及び次に放送される番組）に対しても同様の自動録画ルール
を適用できます．

今回の`compose.yaml`の例では，GitHub上のスクリプトを直接指定していますが，
通常はローカルのスクリプトを実行します．`deno`には`--watch`オプションがあります．
このオプションを指定すると，`deno`は実行中のスクリプトの変更を監視し，自動で再起
動します．ルールを書き換えた場合に自動で反映されるようになり，利便性が増します．

> [!TIP]
> 外部の共有フォルダーに保存されているスクリプトをコンテナーにマウントしている場
> 合，スクリプトを修正しても`deno`が変更を検出しない場合があります．そのような場
> 合は`docker`を実行しているマシンにログインし，`touch`などでスクリプトのタイム
> スタンプを更新するか，コンテナーを再起動する必要があります．

通知されるイベントの詳細については，[mirakc/docs/events.md]を参照してください．

[SSE]: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
[`deno`]: https://deno.land/
[mirakc/docs/events.md]: https://github.com/mirakc/mirakc/blob/main/docs/events.md
