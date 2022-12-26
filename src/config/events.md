# イベント通知

`config.yml`の`events`の各プロパティにコマンドを指定することで，特定のイベントが
発生したときに指定したコマンドを実行することができます．

ここでは，`events.epg.programs-updated`を使って，簡単なルール・ベースの自動録画
予約機能を実装してみます．

`config.yml`

```yaml
events:
  epg:
    programs-updated: >-
      sh /var/lib/mirakc/scripts/epg-programs-updated.sh
```

`docker-compose.yml`

```yaml
services:
  mirakc:
    ...
    volumes:
      ...
      - ./scripts:/var/lib/mirakc/scripts:ro
    environment:
      # スクリプトの動作確認のため
      RUST_LOG: 'info,mirakc=debug'
      MIRAKC_DEBUG_CHILD_PROCESS: 1
```

`scripts/epg-programs-updated.sh`

```sh
log() {
  echo "$1" >&2
}

error() {
  log "ERROR: $1"
  exit 1
}

BASE_URL=http://localhost:40772
TAG='rule-nhk'

# Service IDの取得
SERVICE_ID=$(cat | jq -Mr '.serviceId')

if [ $SERVICE_ID != 3273601024 ]
then
  log "No rule is defined for $SERVICE_ID"
  exit 0
fi

# `/api/services/{id}/programs`は終了した番組も含んでいるので，それらを排除
BASE_FILTER='select(.startAt >= (now + 10000))'  # now +10s

# ルールをここに記述
RULE_FILTER='select(.name | test("ＮＨＫニュース７"))'

COLLECTED=$(curl $BASE_URL/api/services/$SERVICE_ID/programs -sG | \
             jq -Mc ".[] | $BASE_FILTER | $RULE_FILTER | .")

# 以前自動登録した録画予約を削除
#
# 対象番組を集める前に削除しないこと
# そうしてしまうと，登録できない番組が出てきてしまう
log "Removing scheduled tagged with $TAG..."
TARGET=$(echo "$TAG" | jq -Rr @uri)  # percent-encoding
curl "$BASE_URL/api/recording/schedules?target=$TARGET" -s -X DELETE

for PROGRAM_JSON in $COLLECTED
do
  PROGRAM_ID=$(echo "$PROGRAM_JSON" | jq -Mr '.id')
  # TODO: ここで'/'などファイル名で使用できない文字を除去したりする必要がある
  TITLE=$(echo "$PROGRAM_JSON" | jq -Mr ".name")
  START_AT=$(echo "$PROGRAM_JSON" | jq -Mr '.startAt')
  DATE=$(date --date="@$(expr $START_AT / 1000)" +%Y%m%d%H%M)
  SCHEDULE_JSON=$(cat <<EOF | jq -Mc '.'
{
  "programId": $PROGRAM_ID,
  "contentPath": "${DATE}_${TITLE}.m2ts",
  "tags": ["$TAG"]
}
EOF
)
  log "Adding $SCHEDULE_JSON..."
  curl $BASE_URL/api/recording/schedules -s \
    -X POST \
    -H 'Content-Type: application/json' \
    -d "$SCHEDULE_JSON"
done
```

スクリプトの実行完了をしばらく待って以下を実行すると，録画予約が自動登録されてい
ることを確認できます．

```console
$ curl http://localhost:40772/api/recording/schedules -sG | jq '.[].programId'
327360102407088
327360102407531
327360102407957
327360102408432
327360102408944
327360102409528
327360102410344
327360102410933

$ curl http://localhost:40772/api/programs/327360102407088 -sG | \
    jq -r '.name | test("ＮＨＫニュース７")'
true
```

イベント・スクリプトのプロトコルの詳細については，[mirakc/docs/config.md]を参照
してください．

## Server-Sent Event (SSE)

mirakcは，イベント通知のためのWebエンドポイントを提供しています．通知にはSSEを使
用しているため，これをサポートしているウェブ・ブラウザウーで以下のURLを開くだけ
で動作確認が可能です．

* `http://$HOST:$PORT/events`

これを利用すれば，ルール・ベースの自動録画予約機能などをウェブ・アプリケーション
側で実装することが可能です．

[mirakc/docs/config.md]: https://github.com/mirakc/mirakc/blob/main/docs/config.md
