# `tsd`サーバーの詳しい話

> 実運用ので`tsd`サーバーの使用はお勧めしません．理由は以下を参照してください．

私が動作検証用に利用しているTSストリームを処理するためのサーバーです．サーバー化
することで，TSストリーム処理機能を開発環境（macOS）と動作検証環境（SBC）に個別に
準備する手間が省けます．また，TSストリームに対する処理は，SBCにとって軽い処理で
はないため，複数のTSストリームを同時に処理する場合，１台のSBCで全てを処理するこ
とが困難であるという事情もあります．

`tsd`サーバーでは，以下の２つのDockerコンテナーが動いています．

* `b25`コンテナー
* `bcas`コンテナー

`b25`コンテナーは，TCP経由で入力されるTSストリームを，`bcas`コンテナーを利用して
処理し，処理結果をTCP接続元に返します．

`bcas`コンテナーは，TCP経由でのスマートカードリーダーへのアクセスを提供します．
このコンテナーを使うことで，複数のサーバーから１つのスマートカードリーダーを使用
可能となります．

どちらのコンテナーもTCP接続のために`socat`を使っています．以下を見ると分かります
が，`socat`を使うとUNIXソケットの共有が簡単にできます．

`tsd`サーバーとして機能を分離することで，

* `mirakc`コンテナーの構築が簡単になる
* 分離した各コンテナーの機能を他のサーバー/コンテナーから再利用できる

いう利点が得られる一方，分離したことによる以下のような欠点も存在します．

* ネットワークトラフィックの増大
* CPU使用量が増加する可能性

特に，`b25`コンテナーによるデコード処理の分離は影響が大きいです．`socat`によるES
パケットの転送処理により，デコード処理を分離した恩恵（CPU使用量の低下）を打ち消
してしまう可能性があります．

## `b25`コンテナー

`Dockerfile`:

```Dockerfile
FROM alpine:latest

RUN set -eux \
 && apk add --no-cache \
      ccid \
      musl \
      pcsc-lite-libs \
      socat \
      tzdata \
 && apk add --no-cache --virtual .build-deps \
      gcc \
      g++\
      make \
      musl-dev \
      nodejs \
      npm \
      pcsc-lite-dev \
      pkgconf \
 # Use arib-b25-stream-test instead of stz2012/libarib25.
 # Because stz2012/libarib25 doesn't support STDIN/STDOUT.
 # stz2012/libarib25 supports NEON, but it doesn't improve the performance
 # significantly.
 && (cd /tmp; npm i arib-b25-stream-test) \
 && cp /tmp/node_modules/.bin/arib-b25-stream-test /usr/local/bin/b25 \
 # cleanup
 && apk del --purge .build-deps \
 && rm -rf /tmp/*

COPY b25-server /usr/local/bin/

EXPOSE 40773
ENTRYPOINT ["b25-server"]
```

`b25-server`:

```shell
#!/bin/sh -eu

B25_BCAS_SERVER=${B25_BCAS_SERVER:-}

if [ -z "$B25_BCAS_SERVER" ]; then
    echo "B25_BCAS_SERVER must be defined" >&2
    exit 1
fi

rm -rf /var/run/pcscd
mkdir -p /var/run/pcscd

echo "Create a UNIX-domain socket to communicate with a remote BCAS server listening on $B25_BCAS_SERVER"
socat unix-listen:/var/run/pcscd/pcscd.comm,fork \
      tcp-connect:$B25_BCAS_SERVER &

echo "Start a descrambling server on tcp 40773"
socat tcp-listen:40773,fork,reuseaddr system:b25
```

## `bcas`コンテナー

`Dockerfile`:

```Dockerfile
FROM alpine:latest

RUN set -eux \
 && apk add --no-cache \
      ccid \
      musl \
      pcsc-lite-libs \
      socat \
      tzdata

COPY bcas-server /usr/local/bin/

EXPOSE 40774
ENTRYPOINT ["bcas-server"]
```

`bcas-server`:

```shell
#!/bin/sh -eu

BCAS_DEBUG=${BCAS_DEBUG:-}

rm -rf /var/run/pcscd
mkdir -p /var/run/pcscd

echo "Start pcscd"
if [ -n "$BCAS_DEBUG" ]; then
    pcscd -f --debug &
else
    pcscd -f &
fi

echo "Start a pcscd proxy on tcp 40774"
socat tcp-listen:40774,fork,reuseaddr \
      unix-connect:/var/run/pcscd/pcscd.comm
```

## docker-compose.yml

あとはこれらを起動するだけ．

```yaml
version: '3'

services:
  b25:
    container_name: b25
    ports:
      - 40773:40773
    environment:
      B25_BCAS_SERVER: bcas:40774
      TZ: Asia/Tokyo
    depends_on:
      - bcas

  bcas:
    container_name: bcas
    devices:
      - /dev/bus/usb
    ports:
      - 40774:40774
    environment:
      #BCAS_DEBUG: 1
      TZ: Asia/Tokyo
```
