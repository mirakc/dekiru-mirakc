# Dockerイメージ

Docker Hubで配布している[mirakc/mirakc]イメージには，mirakcと`recpt1`などの主
要なツールがすでに含まれています．多くの場合，各自でツールをソースからビルドした
りインストールする必要はありません．

[mirakc/docs/docker.md]に記載されているように，`mirakc/mirakc`イメージには複数
のタグがあり，この中から自分の目的に合ったタグを選択します．以下に代表的なタグを
列挙します．

* latest
  * debianのエイリアス
* debian
  * 最新リリースのDebian/Busterベースのイメージ
* alpine
  * 最新リリースのAlpine/3.12ベースのイメージ
* master
  * master-debianのエイリアス
* master-debian
  * GitHub masterブランチの最新コミットから作成したDebian/Busterベースのイメージ
* master-alpine
  * GitHub masterブランチの最新コミットから作成したAlpine/3.12ベースのイメージ

> `0.11.0`より古いAlpine系イメージは，Alpine/3.11をベースとしています．

これらのイメージは全てマルチアーキイメージです．以下のアーキテクチャーをサポート
しています．

* amd64
* arm32v6
* arm32v7
* arm64v8

Raspberry PiなどのSBCで実行する場合は，先の例で使用した`mirakc/mirakc:alpine`
を選択することを推奨しています．詳しい理由については[mirakc/docs/notes.md]を見て
ください．

[mirakc/mirakc]: https://hub.docker.com/repository/docker/mirakc/mirakc
[mirakc/docs/docker.md]: https://github.com/mirakc/mirakc/blob/master/docs/docker.md#pre-built-images-in-dockerhub
[mirakc/docs/notes.md]: https://github.com/mirakc/mirakc/blob/master/docs/notes.md#mirakc-leaks-memory
