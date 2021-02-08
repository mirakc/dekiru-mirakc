# Dockerを使ったインストール

本記事では，Dockerを使ったmirakcのインストール手順についてのみ説明します．

CPUパワーが貧弱なSBCでDocker使って大丈夫なのかと危惧する人もいるかも知れません
が，実際に動かしてみた経験から問題ないことが分かっています．Dockerコンテ
ナーとして実行する場合，ホストOS上で直接実行する場合に比べてオーバーヘッドが存在
します．しかし，このオーバーヘッドは問題になるほど大きくありません．さらに，この
オーバーヘッドに対して，以下の費用対効果が十分にあると私は考えています．

* インストールの容易さ
* 環境依存度の低減
* ホストOSアップデートの影響を受けにくい

もちろん，Dockerを使わず，ホストOSにmirakcを直接インストールすることも可能です．
その手順は，Dockerイメージのビルド手順を記述したDockerfileにすべて記述されていま
す．[このフォルダー](https://github.com/mirakc/mirakc/tree/master/docker)に
DebianおよびApline用のDockerfileがそれぞれ格納されています．ビルド環境の違いによ
り多少手順が異なりますが，基本的には同様の手順を実行することで，任意のLinux環境
にインストールできると思います．

ただ，mirakcをホストOSに直接インストールする場合は，必要なソフトウェアを個別にビ
ルドする必要があるため，初心者向きとは言えません．というのも，一般的に，SBCのCPU
は普段皆さんが使っているようなデスクトップPC向けCPUに比べて非常に非力であるた
め，必然的に必要なソフトウェアをデスクトップPC上などでクロスビルドすることになる
からです．例えば，過去にROCK64上でmirakcをリリースビルドしたことがありますが，１
時間では終わらなかったと記憶しています．

このような事情もあり，mirakcではDockerを使ったインストールを推奨しています．

[dockerfile-gen]: https://github.com/mirakc/mirakc/blob/master/docker/dockerfile-gen