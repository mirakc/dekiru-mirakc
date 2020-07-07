# Dockerのインストール

今回利用したRaspbianのAPTリポジトリーには，`docker.io`および`docker-compose`パッ
ケージが既に含まれています．しかし，

* `docker`: 18.09.1
* `docker-compose`: 1.21.0

と一年以上前のバージョンです．そこで今回は，Raspbian APTリポジトリーを使わずに，
これらのツールをインストールします．

まずは`docker`をインストールします．[Docker公式ドキュメント]では[get.docker.com]
スクリプトの使用は推奨されていませんが，Raspbianでのインストール手順が公式ドキュ
メントに記載されていないため，Raspbianでも動作する`get.docker.com`スクリプトを使
用します．

```shell
# Dockerのインストール
#
# 動作確認のため使用した`get.docker.com`スクリプトのコミットハッシュは
# `442e66405c304fa92af8aadaa1d9b31bf4b0ad94`
curl -sSL https://get.docker.com | sudo sh

# 念の為，動くことを確認
docker version
```

次に`docker-compose`をインストールします．残念ながら，公式リポジトリー
[docker/compose]ではamd64バイナリーしか配布されていません．そこで，私が作成した
[masnagam/sbc-scripts]の`get-docker-compose`スクリプトを使用します．このスクリプ
トを使用すると，Docker Hubにアップロード済みの[masnagam/docker-compose]イメージ
からバイナリーを抽出できます．

```shell
# Dockerイメージからdocker-composeコマンドを抽出
curl -fsSL https://raw.githubusercontent.com/masnagam/sbc-scripts/master/get-docker-compose | \
  sh | sudo tar -x -C /usr/local/bin

# バージョンが表示されればOK
docker-compose version
```

上記で利用したDockerイメージはDebian/Busterベースですが，今回使用している
RaspbianのバージョンもDebian/Busterをベースとして作成されているため，問題なく動
作します．

上記で利用した`masnagam/docker-compose`イメージは，
[GitHub Actions](https://github.com/masnagam/sbc-scripts/actions)と公式リポジト
リーに含まれている
[Dockerfile](https://github.com/docker/compose/blob/master/Dockerfile)を使ってビ
ルドしたマルチアーキイメージです．現時点では，以下のアーキテクチャーのみサポート
しています．

* amd64
  * 一般的なデスクトップPCに搭載されているIntel系やRyzen系
* arm32v7
  * Raspberry Pi 2以降のRaspbianやZeroPiなど
* arm64v8
  * ROCK64向けArmbianなど
  * Raspberry Pi 2B v1.2以降の64bits OS

残念ながら，古いRaspberry PiやRaspberry Pi Zero系（arm32v5）では利用できません．

[Docker公式ドキュメント]: https://docs.docker.com/engine/install
[get.docker.com]: https://get.docker.com/
[docker/compose]: https://github.com/docker/compose
[masnagam/sbc-scripts]: https://github.com/masnagam/sbc-scripts
[masnagam/docker-compose]: https://hub.docker.com/r/masnagam/docker-compose
