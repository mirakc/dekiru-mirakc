# 録画予約

mirakcの録画予約機能を有効にするには，以下の設定を`config.yml`に追加する必要があ
ります．

```yaml
recording:
  basedir: /var/lib/mirakc/recording
```

`basedir`には，登録した録画予約の内容など管理情報を保持するためのファイル
（`schedules.json`）や，録画データが保存されます．

`docker-compose.yml`でフォルダーをマウントします．

```yaml
services:
  mirakc:
    ...
    volumes:
      ...
      - ./recording:/var/lib/mirakc/recording
```

mirakcコンテナーを再起動し，録画予約関係のWebエンドポイントが有効になっているか
確認します．

```console
# 録画予約一覧の取得
# 何も予約していないので当然空のリストが返ってきます
$ curl http://localhost:40772/api/recording/schedules -sG
[]
```

以降の説明では[`recording.sh`]というシェル・スクリプトを使用しています．

## 録画予約の登録・削除

試しに，現在放送中の番組を録画予約してみましょう．

```console
# ＮＨＫ総合１・東京の番組表から，現在放送中の番組のProgram IDを取得
# Program IDは，実際に返ってきた値に読み替えてください
$ curl http://localhost:40772/api/services/3273601024/programs -sG | \
    jq '.[] | select((.startAt / 1000) <= now and ((.startAt + .duration) / 1000) > now) | .id'
327360102407046

# 録画予約を登録
# もうすぐ終了する番組でなければ，登録に成功します
$ sh recording.sh -r add 327360102407046 | jq '.program.id'
327360102407046
```

録画予約の登録に成功しました．放送中の番組を録画予約したので，録画はすぐに開始されます．

```console
# すぐに録画が開始されるので，空のリストが返されます
$ sh recording.sh -r list | jq '.[] | .program.id, .state'
327360102407046
"recording"

# 番組がまだ放送中なら録画中のリストに含まれています
$ curl http://localhost:40772/api/recording/recorders -sG | jq '.[].programId'
327360102407046
```

`options.contentPath`に指定したファイルが，`recording/videos`フォルダー内に作成
されていることが確認できます．

```console
$ ls recording
schedules.json  videos

$ ls recording/videos
'<datetime>_<program-id>.m2ts'  # <datetime>および<program-id>は実際の値に読み替えること
```

このように，`options.contentPath`に存在しないディレクトリーが含まれている場合，
録画時に自動でディレクトリーが作成されます．

録画予約を削除すれば，録画も停止します．

```console
$ sh recording.sh delete 327360102407046

$ sh recording.sh -r list
[]

$ curl http://localhost:40772/api/recording/recorders -sG
[]
```

録画予約を削除しても録画データは消えません．

```console
$ ls recording/videos
'<datetime>_<program-id>.m2ts'
```

基本的に録画予約は，番組表に記載されている時刻に録画を開始します．多くの場合，こ
れでも問題はないのですが，ごく稀に，前番組の放送延長などが原因で録画予約した番組
の放送時間が変更されることがあります．このような場合，番組表に記載された時刻には
番組は開始されないため，当然の結果として番組全体を正しく録画することはできません．

mirakcは，録画予約した番組の放送時間の変更を追跡する機能を実装しています．

## 放送中の番組の追跡

mirakcは，現在放送中の番組（及び次に放送される番組）を追跡する機能を持っています．
この機能を利用することで，録画予約した番組の放送時間が変更された場合でも，適切に
録画予約情報を更新し，変更後の番組開始時間から録画を開始するようになります．

本機能が安定的に機能するためには，本機能のために専用のチューナーを割り当てる必要
があります．割り当てたチューナーは本機能以外では使用できなくなります．そのため，
複数のチューナーが存在する環境以外で本機能を使用するのは困難でしょう．

本機能を有効化するために，`config.yml`を以下のように修正します．

```yaml
tuners:
  ...
  # 放送中の番組追跡専用のチューナーを追加
  - name: tracker
    types: [GR, BS]
    command: ...
    # このプロパティでチューナーを独占的に使用するトラッカーを指定
    dedicated-for: tracker

onair-program-trackers:
  tracker:
    local:
      channel-types: [GR, BS]
```

mirakcコンテナーを再起動してください．以下のようなログが毎分出力されるようになります．

```console
$ docker logs -f mirakc | grep OnairProgramTracker
... INFO ... Use dedicated tuner tuner.index=1 channel=GR/27 user.info=OnairProgramTracker(tracker)
... INFO ... Activate tuner.index=1 channel=GR/27 user.info=OnairProgramTracker(tracker)
... INFO ... Subscribed subscription.id=tuner#1.0.1 user.info=OnairProgramTracker(tracker)
... INFO ... Unsubscribed subscription.id=tuner#1.0.1 user.info=OnairProgramTracker(tracker)
```

本機能を有効化すると，毎分チューナーを開いて，各サービスで現在放送中の番組（及
び次に放送される番組）を確認するようになります．

注意点としては，毎分上記のようなログがサービスの数だけ出力されるため，場合によっ
ては大量のログが出力されます．また，CPUやメモリー使用量も当然増えるため，すでに
リソースが逼迫している環境での使用は困難です．

本文書の例ではチャンネル数（サービス数）が少ないので問題ありませんが，多くのチャ
ンネル（サービス）を使用している環境では，現在放送中の番組の取得処理が１分以内
（安全を考えると50秒以内）に終了するようにトラッカー数を調整する必要があります．

```yaml
onair-program-trackers:
  gr-tracker:
    local:
      channel-types: [GR]
  bs-tracker1:
    local:
      channel-types: [BS]
      # 期限内に処理が終了するように，対象サービスを制限
      services: [...]
  bs-tracker2:
    local:
      channel-types: [BS]
      # bs-tracker1で処理しないサービスを列挙
      services: [...]
```

本機能がどのように動作するのか見るために，後続の２つの番組の録画予約を行ってみま
しょう．

まず，対象番組のIDを取得します．

```console
$ curl http://localhost:40772/api/services/3273601024/programs -sG | \
    jq '.[] | select((.startAt / 1000) > now) | .id' | head -n 2
327360102415793  # 番組表情，次の番組
327360102415795  # その次の番組
```

まず，番組表で次の次に始まる番組予定を予約します．

```console
$ sh recording.sh -r add 327360102415795 | jq '.program.id, .state'
327360102415795
"scheduled"
```

状態`scheduled`とは，番組表の情報を元に録画予約された状態です．録画予約された番
組が次に放送予定の番組（または現在放送中の番組）になると，状態が`tracking`へと変
わります．これを確認するために，番組表で次に始まる予定の番組を録画予約します．

```console
$ sh recording.sh -r add 327360102415793 | jq '.program.id, .state'
327360102415793
"tracking"
```

## 録画失敗時の動作

例え録画予約した番組の放送時間の変更を追跡していたとしても，番組が放送されなけれ
ば録画予約は失敗します．録画に失敗した場合，録画予約の状態は`rescheduling`となり
ます．

> **注意**: 少しでも番組を録画できた場合は，録画失敗とは扱われません

録画に失敗した録画予約はすぐには削除されず残り続けます．この間に，番組表や現在放
送中の番組が更新されるなどして，録画に失敗した番組の放送時間が変更されると，再度
録画予約が有効化されます．ただし，録画予約時に指定した番組ID（`programId`）を使
って放送時間の更新を検出するため，対象としていた番組のIDが変わってしまうと，別の
番組として扱われます．このような場合には，一旦録画予約を削除し，再度新しい番組ID
で録画予約を登録し直す必要があります．

> 現状，予定されていた番組の開始時刻から１５時間まで録画予約を保持します．
> 実際の削除は録画予約の実行直前に行うため，録画予約がなければ残り続けます．

録画失敗の状況を擬似的に体験するため，すでに終了している番組の録画を録画予約して
みます．

```console
$ curl http://localhost:40772/api/services/3273601024/programs -sG | \
    jq '.[] | select((.startAt / 1000) <= now) | .id' | tail -n 2
327360102415795  # 番組表上，終了している番組
327360102415796  # 番組表上，現在放送中の番組

$ sh recording.sh -r add 327360102415795 | jq '.program.id, .state'
327360102415795
"scheduled"
```

追加した録画予約に対する録画はすぐに開始されますが，番組は終了しているので当然失
敗します．

```console
$ docker logs mirakc | grep WARN
...  WARN ... Need rescheduling schedule.program.quad=7FE07FE004003DB3
```

その後，録画予約は`rescheduling`状態に．

```console
$ sh recording.sh -r list | jq '.[] | .program.id, .state'
327360102415795
"rescheduling"
```

この番組はすでに終了していて放送時間が更新される可能性はないので，実験後は録画予
約を削除してください．

## 録画した番組の視聴

mirakcは，録画した番組を視聴するための機能を提供していません．これは，Kodiなどの
優れたメディア・センター・アプリケーションがすでに存在しているためです．これらを
使用すれば，mirakcで視聴するよりも良い体験を得られます．

別アプリケーションをどうしても使用できないという場合には，録画データを含むフォル
ダーをmirakc上の特定URLにマウントすることで，録画データをmirakc経由でダウンロー
ドできるようになります．

> ファイルシステム上の録画データに直接（またはsambaなどを使って間接的に）アクセ
> スできるので，普通はこのようなことをする必要はありません

`config.yml`

```yaml
server:
  ...
  mounts:
    /videos:
      path: /var/lib/mirakc/recording/videos
      listing: true
```

ブラウザーで`http://localhost:40772/videos/`を読み込むと，録画したファイルの一覧
が表示されます．

## ルール・ベースの自動録画予約

mirakc自体は，EPGStationなどがサポートしているようなルール・ベースの自動録画予約
機能を提供していません．しかし，後述するイベント通知機能を利用することで，番組表
更新時にルール・ベースの自動録画予約を行うことが可能です．詳細については
[イベント通知](../events.md)を参照してください．

[`recording.sh`]: https://github.com/mirakc/contrib/blob/main/recording/recording.sh
