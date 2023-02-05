# Dockerのインストール

[Docker公式ドキュメント]の記述に従い，Dockerをインストールします．

```shell
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 念の為，動くことを確認
sudo docker version
sudo docker compose version
```

一行のコマンドでインストールしたいという人は，以下を使ってください．

```shell
SETUP_TARGET=debian curl -fsSL \
  https://raw.githubusercontent.com/masnagam/setup/main/scripts/docker.sh | sh -s
```

上記スクリプトを使うと，スクリプト内で`sudo usermod -aG docker $(whoami)`が実行されるため，`sudo`な
しで`docker`を実行可能となる点に注意してください．ユーザーを`docker`グループに追加したくない場
合は，上記スクリプトを使わず，Docker公式ドキュメントの手順に従ってインストールしてください．

[Docker公式ドキュメント]: https://docs.docker.com/engine/install
