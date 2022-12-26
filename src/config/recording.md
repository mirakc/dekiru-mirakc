# 録画予約

mirakcの録画予約機能を有効にするには，以下の設定を`config.yml`に追加する必要があ
ります．

```yaml
recording:
  records-dir: /var/lib/mirakc/recording
  contents-dir: /var/lib/mirakc/recording/contents
```

`records-dir`には登録した録画予約の内容など管理情報を保持するためのファイルが作
成されます．

`contents-dir`は指定しなくても録画予約機能は有効になりますが，通常は録画データの
みを共有するので，指定することを推奨します．

`docker-compose.yml`でフォルダーをマウントします．

```yaml
services:
  mirakc:
    ...
    volumes:
      ...
      - mirakc-recording:/var/lib/mirakc/recording
      - ./videos:/var/lib/mirakc/recording/contents

volumes:
  ...
  mirakc-recording:
    name: mirakc_recording
    driver: local
```

mirakcコンテナーを再起動し，録画予約関係のWebエンドポイントが有効になっているか
確認します．

```console
# 録画予約一覧の取得
# 何も予約していないので当然空のリストが返ってきます
$ curl http://localhost:40772/api/recording/schedules -sG
[]
```

試しに，現在放送中の番組を録画予約してみましょう．

```console
# ＮＨＫ総合１・東京の番組表から，現在放送中の番組のProgram IDを取得
# Program IDは，実際に返ってきた値に読み替えてください
$ curl http://localhost:40772/api/services/3273601024/programs -sG | \
    jq '.[] | select((.startAt / 1000) <= now and ((.startAt + .duration) / 1000) > now) | .id'
327360102407046

# 録画予約を登録
# もうすぐ終了する番組でなければ，登録に成功します
# add-recording-schedule.shは後述
$ sh add-recording-schedule.sh http://localhost:40772 327360102407046 | \
    jq '.programId'
327360102407046
```

録画予約の登録に成功しました．放送中の番組を録画予約したので，録画はすぐに開始されます．

```console
# すぐに録画が開始されるので，空のリストが返されます
$ curl http://localhost:40772/api/recording/schedules -sG
[]

# 番組がまだ放送中なら録画中のリストに含まれています
$ curl http://localhost:40772/api/recording/recorders -sG | jq '.[].programId'
327360102407046
```

`contentPath`に指定したファイルが，`videos`フォルダー内に作成されていることが確認できます．

```
$ ls videos
'Ｄｅａｒにっぽん「それでもいま　一歩前へ　〜核大国アメリカでの対話〜」[字][再].m2ts'
```

ここで使用した`add-recording-schedule.sh`は，以下のようなシェル・スクリプトです．

```sh
BASEURL=$1
PROGRAM_ID=$2

# 番組名をファイル名に使用
TITLE=$(curl $BASEURL/api/programs/$PROGRAM_ID -sG | jq -Mr '.name')
JSON=$(cat <<EOF | jq -Mc '.'
{
  "programId": $PROGRAM_ID,
  "contentPath": "$TITLE.m2ts",
  "tags": ["manual"]
}
EOF
)

# 録画予約を登録
curl $BASEURL/api/recording/schedules -s \
  -X POST \
  -H 'Content-Type: application/json' \
  -d "$JSON"
```

## 録画失敗時の動作

何らかの理由で録画予約した番組の放送時間が急に変更された場合，mirakcは録画に失敗
します．EpgStationなど他の多くのアプリケーションも同様に失敗するでしょう．

mirakcは，このような場合の録画失敗を救済するための機能として，録画リトライ機能を
実装しています．録画リトライ機能は，録画に失敗した番組の開始を検出するために，
チューナーを定期的に使用します．そのため，チューナーに空きがない場合，正常に機能
しません．

録画リトライ機能を擬似的に体験してみましょう．

まず，録画リトライ機能を有効にするために`config.yml`に
`recording.max-start-delay`を設定します．

```yaml
recording:
  ...
  max-start-delay: 3h
```

今回の例では，番組開始時間の遅延を最大３時間まで許容するように設定します．２４時
間（`24h`）以内の任意の値を指定可能です．
書式については[こちら](https://github.com/tailhook/humantime)参照してください．

コンテナーを再起動し，３時間以内に始まるが，現在放送中でも次に放送される番組でも
ない番組を探します．

```console
# 次の次に始まる番組のProgram IDを取得
$ curl http://localhost:40772/api/services/3273601024/programs -sG | \
    jq '.[] | select((.startAt / 1000) > now) | .id' | head -n 2
327360102407064
327360102407065
```

`327360102507065`の録画（録画予約ではありません）を開始します．

```console
# start-recording.shは後述
$ sh start-recording.sh http://localhost:40772 327360102407065 | jq '.programId'
327360102407065
```

この番組はまだ始まっていないので，当然録画はすぐに停止します．

```console
# 録画はすぐに停止するので，空のリスト
$ curl http://localhost:40772/api/recording/recorders -sG
[]
```

以下のようなログが見つかります．

```
... WARN ... Recording stopped before the program starts program_quad=7FE07FE004001B99
... INFO ... On-air program observer added service_triple=7FE07FE004000000 name="recording"
... INFO ... Retry recording program_quad=7FE07FE004001B99
```

現在放送中の番組が終了し，次の番組が始まるころに，以下のようなログが表示されます．

```
... INFO ... Added schedule.program_quad=7FE07FE004001B99
... INFO ... Rescheduled recording program.quad=7FE07FE004001B99 program.start_at=2022-12-26 14:00:00 +09:00
```

録画予約を確認すると，先程録画に失敗した番組が追加されていることを確認できます．

```console
$ curl http://localhost:40772/api/recording/schedules -sG | jq '.[].programId'
327360102407065
```

ここで使用した`start-recording.sh`は，以下のようなシェル・スクリプトです．

```sh
BASEURL=$1
PROGRAM_ID=$2

# 番組名をファイル名に使用
TITLE=$(curl -sG $BASEURL/api/programs/$PROGRAM_ID | jq -Mr '.name')
JSON=$(cat <<EOF | jq -Mc '.'
{
  "programId": $PROGRAM_ID,
  "contentPath": "$TITLE.m2ts",
  "tags": ["manual"]
}
EOF
)

# 録画を開始
curl $BASEURL/api/recording/recorders -s \
  -X POST \
  -H 'Content-Type: application/json' \
  -d "$JSON"
```

## 録画した番組の視聴

mirakcは，録画した番組を視聴するためのWebエンドポイントを機能を提供していません．
これは，Kodiなどの優れたメディア・センター・アプリケーションがすでに存在している
ためです．これらを使用すれば，mirakcで視聴するよりも良い体験を得られます．

別アプリケーションをどうしても使用できないという場合には，`contents-dir`にしてい
たフォルダーをmirakc上の特定URLにマウントすることができます．

`config.yml`

```yaml
server:
  ...
  mounts:
    # `recording.contents-dir`に指定したパスを/videosにマウント
    /videos:
      path: /var/lib/mirakc/recording/contents
      listing: true
```

ブラウザーで`http://localhost:40772/videos/`を読み込むと，録画したファイルの一覧
が表示されます．

## ルール・ベースの自動録画予約

mirakc自体は，EPGStationなどがサポートしているようなルール・ベースの自動録画予約
機能を提供していません．しかし，後述するイベント通知機能を利用することで，番組表
更新時にルール・ベースの自動録画予約を行うことが可能です．詳細については
[イベント通知](./events.md)を参照してください．
