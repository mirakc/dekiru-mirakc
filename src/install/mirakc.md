# mirakcのインストール

細かい設定は後回しにして，mirakcを起動してみましょう．

まず，以下のような`docker-compose.yml`を作成します．

```yaml
version: '3.7'

services:
  mirakc:
    image: masnagam/mirakc:alpine
    container_name: mirakc
    init: true
    restart: unless-stopped
    ports:
      - 40772:40772
    volumes:
      - ./config.yml:/etc/mirakc/config.yml:ro
    environment:
      TZ: Asia/Tokyo
      RUST_LOG: info
```

次に，同じフォルダー内に`config.yml`を作成します．

```yaml
server:
  addrs:
    - http: 0.0.0.0:40772
```

最後に，mirakcを起動します．

```console
# mirakcコンテナーをバックグラウンドで起動
$ sudo docker-compose up -d
Starting mirakc ... done

# バージョン文字列の取得
$ curl -s http://localhost:40772/api/version
"0.9.0"

# mirakcコンテナーのシャットダウン
$ sudo docker-compose down
Stopping mirakc ... done
Removing mirakc ... done
Removing network pi_default
```

mirakcの起動に成功していれば，`"0.9.0"`のようなバージョン文字列が表示されます．

どうでしょうか？思っていたより簡単ですよね．
