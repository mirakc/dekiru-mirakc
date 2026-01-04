# カスタムイメージの作成

多くの人は，ここまでの説明と`mirakc/mirakc`イメージを使って録画システムを構築できたと思います．しか
し，配布しているDockerイメージに含まれていないツール，例えば`recpt1`と`recdvb`以外のチューナーコマン
ドを使っている人は，このDockerイメージだけでは録画システムを構築できません．自力でカスタムイメージを
作成する必要があります．

## カスタムイメージのネイティブビルド {#native-build}

カスタムイメージの作成はそれほど難しい作業ではありません．試しに`HEALTHCHECK`を追加してみます．

まず，以下のような`Dockerfile`を作成します．

```Dockerfile
FROM mirakc/mirakc:alpine

# 短時間でテストできるように，チェック間隔を１０秒に設定しています
HEALTHCHECK --interval=10s --timeout=3s \
  CMD curl http://localhost:40772/api/status -sG || exit 1
```

次にカスタムイメージのビルド．

```shell
sudo docker build -t custom/mirakc .
```

あとは起動して動作確認．

```shell
sudo docker run -d --rm --name custom-mirakc \
  -v $(pwd)/config.yml:/etc/mirakc/config.yml custom/mirakc

# １０秒ごとにログが表示される
sudo docker events

sudo docker stop custom-mirakc
```

より実用的な例としては，以下のDockerfileを参考にしてください．

* [yude/tv-recorder-dockerfile](https://github.com/yude/tv-recorder-dockerfile/tree/master/mirakc)
* [octarect/tvrec](https://github.com/octarect/tvrec/blob/master/docker/mirakc/Dockerfile)
* [No292nukegara/mirakc-recisdb](https://github.com/No292nukegara/mirakc-recisdb/blob/main/Dockerfile)
* [docker-bon-mirakurun/dockerfiles/mirakc](https://github.com/68fpjc/docker-bon-mirakurun/blob/master/dockerfiles/mirakc)

## カスタムイメージのクロスビルド {#cross-build}

ほとんどの場合，カスタムイメージを特定の実行環境上でビルド（ネイティブビルド）すると思います．しか
し，カスタムイメージをクロスビルドすべき幾つかの状況が存在します．

* Multi-Archイメージを作成したい
* コンパイルが必要がツールをカスタムイメージに追加したいが，SBCでは時間がかかる

このような場合，ターゲットプラットフォームを指定したカスタムイメージのクロスビルドを行うことになりま
す．

クロスビルドを行う場合，クロスビルドを実行するマシンとは異なるCPUアーキテクチャー用のプログラムを実
行する場合があます．例えば，aptで追加のパッケージをインストールする場合です．このような場合，クロス
ビルド前にQEMUユーザーモードエミュレーションのための準備が必要です．macOS向けdocker desktopでは最初
から有効になっているようですが，Linux環境では以下を実行する必要があります．

```shell
docker run --privileged --rm tonistiigi/binfmt --install all
```

詳細については，[tonistiigi/binfmt]を参照してください．

QEMUの準備ができたら，後はクロスビルドするだけです．試しに，`linux/arm/v7`向けイメージをmacOS（amd64
）上でクロスビルドしてみます．

```shell
# 「カスタムイメージの作成」で使用したDockerfile
cat <<EOF >Dockerfile
FROM mirakc/mirakc:alpine

# 短時間でテストできるように，チェック間隔を１０秒に設定しています
HEALTHCHECK --interval=10s --timeout=3s \
  CMD curl http://localhost:40772/api/status -sG || exit 1
EOF

docker build -t custom/mirakc:arm32v7 --platform=linux/arm/v7 .
```

クロスビルドに成功したか確認してみます．

```console
$ docker image inspect custom/mirakc:arm32v7 | \
    jq '.[] | [.Os, .Architecture, .Variant] | join("/")'
"linux/arm/v7"
```

[tonistiigi/binfmt]: https://hub.docker.com/r/tonistiigi/binfmt
