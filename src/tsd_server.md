# `tsd`サーバーの詳しい話

> 実運用での`tsd`サーバーの使用はお勧めしません．理由は以下を参照してください．

私が動作検証用に利用しているTSストリームを処理するためのサーバーです．サーバー化
することで，TSストリーム処理機能を開発環境と動作検証環境（SBC）に個別に
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
FROM alpine AS b25-build
RUN apk add --no-cache cmake git g++ libtool make pkgconf pcsc-lite-dev
RUN git clone --depth=1 https://github.com/tsukumijima/libaribb25.git /src
RUN cmake -S /src -B /build -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/opt/libaribb25
RUN make -C /build -j $(nproc)
RUN make -C /build install

FROM alpine
COPY --from=b25-build /opt/libaribb25 /usr/local/
COPY ./main.sh /
RUN apk add --no-cache ccid libgcc libstdc++ musl pcsc-lite-libs socat tzdata
EXPOSE 40773
ENTRYPOINT ["/bin/sh", "/main.sh"]
```

`main.sh`:

```shell
if [ -z "$BCAS_SERVER" ]
then
  echo "BCAS_SERVER must be defined" >&2
  exit 1
fi

rm -rf /var/run/pcscd
mkdir -p /var/run/pcscd

echo "Create a UNIX-domain socket to communicate with a remote BCAS server listening on $BCAS_SERVER"
socat unix-listen:/var/run/pcscd/pcscd.comm,fork tcp-connect:$BCAS_SERVER &

echo "Start a descrambling server on tcp 40773"
socat tcp-listen:40773,fork,reuseaddr system:'arib-b25-stream-test -v 0'
```

## `bcas`コンテナー

`Dockerfile`:

```Dockerfile
FROM alpine
COPY ./main.sh /
RUN apk add --no-cache ccid musl pcsc-lite-libs socat tzdata
EXPOSE 40774
ENTRYPOINT ["/bin/sh", "/main.sh"]
```

`main.sh`:

```shell
rm -rf /var/run/pcscd
mkdir -p /var/run/pcscd

echo "Start pcscd"
if [ -n "$BCAS_DEBUG" ]
then
  pcscd -f --debug &
else
  pcscd -f &
fi

echo "Start a pcscd proxy on tcp 40774"
socat tcp-listen:40774,fork,reuseaddr unix-connect:/var/run/pcscd/pcscd.comm
```

## compose.yaml

あとはこれらを起動するだけ．

```yaml
services:
  b25:
    container_name: b25
    depends_on:
      - bcas
    image: b25
    init: true
    restart: unless-stopped
    ports:
      - 40773:40773
    environment:
      B25_BCAS_SERVER: bcas:40774
      TZ: Asia/Tokyo

  bcas:
    container_name: bcas
    image: bcas
    init: true
    restart: unless-stopped
    devices:
      - /dev/bus/usb
    ports:
      - 40774:40774
    environment:
      #BCAS_DEBUG: 1
      TZ: Asia/Tokyo
```
