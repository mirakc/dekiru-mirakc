# EGPデータキャッシュ

mirakcの先程のログの中に，以下のようなワーニングが見つかります．

```console
... WARN ... No epg.cache-dir specified, skip to save services
```

これは，まだEPGデータキャッシュを指定していないため，せっかく取得したEPGデータを
保存することができないことを意味しています．この状態でmirakcを停止すると，すべて
のEPGデータは消えてしまいます．

今の設定のままでは，mirakc起動直後はEPGデータが存在しません．その結果，EPGデータ
の取得が完了するまで，番組表取得や番組録画はできません．そこで，EPGデータキャッ
シュを設定します．

もちろん，EPGデータキャッシュは必要ないという人は，以下の設定を行う必要はありま
せん．

`config.yml`:

```yaml
# EPGデータの保存先のフォルダーを指定
epg:
  cache-dir: /var/lib/mirakc/epg
```

`docker-compose.yml`:

```yaml
version: '3'

services:
  mirakc:
    image: mirakc/mirakc:alpine
    container_name: mirakc
    init: true
    restart: unless-stopped
    devices:
      - /dev/px4video2
    ports:
      - 40772:40772
    volumes:
      # epg.cache-dirで指定したパスにボリュームをマウント
      - mirakc-epg:/var/lib/mirakc/epg
      - ./config.yml:/etc/mirakc/config.yml:ro
    environment:
      TZ: Asia/Tokyo
      RUST_LOG: info

volumes:
  # EPGデータキャッシュ用ボリューム
  mirakc-epg:
    name: mirakc_epg
    driver: local
```

データボリュームの代わりにDockerホスト上のフォルダーをマウントすることも可能です
が，特別な理由がない限りデータボリュームを使用することをお勧めします．

例えば，現在の設定では，mirakcコンテナーではコマンドをすべて`root`権限で実行する
ため，ボリューム内に作成されるファイルの所有者は`root`になります．Dockerホスト上
のフォルダーをマウントする場合には，この辺にも注意が必要です．

設定が機能しているか，確認してみます．

```console
# mirakcコンテナーの起動（５分程度待ってからCtrl+Cで停止）
$ sudo docker compose up
...
... ERROR ... Failed to load services: ...
... ERROR ... Failed to load clocks: ...
... ERROR ... Failed to load schedules: ...
...
... Saved schedules for 3 services
```

先程のワーニングが出ていなければ正しく機能しています．

設定したデータボリュームはmirakcコンテナーをシャットダウンしても残り続けます．

```console
# mirakcコンテナーのシャットダウン
$ sudo docker compose down
Removing mirakc ... done
Removing network pi_default

# データボリュームが存在することを確認
$ sudo docker volume ls
DRIVER              VOLUME NAME
local               mirakc_epg
```

では次に，キャッシュが機能しているか確認するため，再度mirakcコンテナーを起動しま
す．

```console
# mirakcコンテナーの起動（Ctrl+Cで停止）
$ sudo docker compose up
```

今度はEPGデータ読み込みエラーが表示されなくなっているはずです．キャッシュは正し
く機能しています．

EPGデータが正しく取得できているのか確認してみましょう．

```console
# jqをインストール
$ sudo apt-get install -y jq

# mirakcコンテナーをバックグラウンドで起動
$ sudo docker compose up -d

$ curl -s http://localhost:40772/api/services | jq .[0]
{
  "id": 3273601024,
  "serviceId": 1024,
  "networkId": 32736,
  "type": 1,
  "logoId": 0,
  "remoteControlKeyId": 1,
  "name": "ＮＨＫ総合１・東京",
  "channel": {
    "type": "GR",
    "channel": "27"
  },
  "hasLogoData": false
}

# アクセスログを確認
$ sudo docker logs --tail=1 mirakc
... 172.19.0.1:59498 "GET /api/services HTTP/1.1" 200 564 ...

# mirakcコンテナーをシャットダウン
$ sudo docker compose down
```

なお，以下のコマンドを実行すると，シャットダウン時にデータボリュームを削除しま
す．

```console
$ sudo docker compose down -v
...
Removing volume mirakc_epg

$ sudo docker volume ls
DRIVER              VOLUME NAME
```

データボリュームを削除すると，当然EGPデータキャッシュもなくなります．そのため，
次回起動時にEGPデータの取得が完了するまで，番組表取得や番組録画はできなくなりま
す．
