# カスタムイメージの作成

多くの人は，ここまでの説明と`mirakc/mirakc`イメージを使って録画システムを構築
できたと思います．しかし，配布しているDockerイメージに含まれていないツール，例え
ば`recpt1`と`recdvb`以外のチューナーコマンドを使っている人は，このDockerイメージ
だけでは録画システムを構築できません．自力でカスタムイメージを作成する必要があり
ます．

カスタムイメージの作成はそれほど難しい作業ではありません．試しに`HEALTHCHECK`を
追加してみます．

まず，以下のような`Dockerfile`を作成します．

```Dockerfile
FROM mirakc/mirakc:alpine-arm32v7

# 短時間でテストできるように，チェック間隔を１０秒に設定しています
HEALTHCHECK --interval=10s --timeout=3s \
  CMD curl -fsSL http://localhost:40772/api/status || exit 1
```

ここでポイントとなるのが，`FROM`に指定したDockerイメージのタグ`alpine-arm32v7`で
す．最終的にmirakcコンテナーを動作させるSBC上でカスタムイメージをビルドする場合
は，マルチアーキイメージである`alpine`タグを使用しても問題ありません．しかし，一
般的にSBC上でのビルドは大変時間がかかります．このビルド時間の問題は，より高速な
PC上でSBC用のDockerイメージをクロスビルドすることで解決できます．ただ，クロスビ
ルドでマルチアーキイメージを使用すると，最終的にコンテナーを実行するSBC
（Raspberry Pi 3だとarm32v7）用のDockerイメージではなく，クロスビルドを実行する
マシン（多くの場合，amd64）用のDockerイメージを取得してしまいます．そのため，上
記では`alpine-arm32v7`を指定しています．

> 今回の例では不要ですが，異なるアーキテクチャ用のDockerイメージ上で何からのコマ
> ンドを実行する場合，QEMUユーザーモードエミュレーションのための準備が必要です．
> 具体的には，[こちら](https://github.com/mirakc/mirakc/blob/master/.github/workflows/docker.yml#L36)
> を参照してください．

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

* [tv-recorder/mirakc](https://github.com/collelog/tv-recorder/tree/master/mirakc)
* [docker-bon-mirakurun/dockerfiles/mirakc](https://github.com/68fpjc/docker-bon-mirakurun/blob/master/dockerfiles/mirakc)

Dockerイメージのビルド時に，追加するソフトウェアをクロスビルドする場合などは，以
下を参考にしてください．

* [mirakc/docker/templates](https://github.com/mirakc/mirakc/tree/master/docker/templates)
