# Dockerイメージ

> [!NOTE]
> 何らかの修正があった場合，毎週土曜日の早朝に自動でタグを付けて，イメージを
> Docker Hubにアップロードするようになっています．

Docker Hubで配布している[mirakc/mirakc]イメージには，mirakcと`recpt1`などの主
要なツールがすでに含まれています．多くの場合，各自でツールをソースからビルドした
りインストールする必要はありません．

[mirakc/docs/docker.md]に記載されているように，`mirakc/mirakc`イメージには複数
のタグがあり，この中から自分の目的に合ったタグを選択します．以下に代表的なタグを
列挙します．

* latest
  * debianのエイリアス
* debian
  * 最新リリースのDebianベースのイメージ
* alpine
  * 最新リリースのAlpineベースのイメージ
* main
  * main-debianのエイリアス
* main-debian
  * GitHub mainブランチの最新コミットから作成したDebianベースのイメージ
* main-alpine
  * GitHub mainブランチの最新コミットから作成したAlpineベースのイメージ

> [!WARNING]
> `0.17.0`以降のAlpineイメージは，32ビットCPUでは動作しない可能性があります．
> 詳しくは[こちら](https://wiki.alpinelinux.org/wiki/Release_Notes_for_Alpine_3.13.0#musl_1.2)を見てください．

これらのイメージは全てマルチアーキイメージです．以下のプラットフォームをサポート
しています．

* linux/386
* linux/amd64
* linux/arm/v7
* linux/arm64/v8

> [!CAUTION]
> リンクエラーを回避するため，Alpineのlinux/386に含まれる`mirakc-arib`はSSPが無効化されています．

Raspberry PiなどのSBCで実行する場合は，先の例で使用した`mirakc/mirakc:alpine`
を選択することを推奨しています．詳しい理由については[mirakc/docs/notes.md]を見て
ください．

[mirakc/mirakc]: https://hub.docker.com/r/mirakc/mirakc
[mirakc/docs/docker.md]: https://github.com/mirakc/mirakc/blob/main/docs/docker.md#pre-built-images-in-dockerhub
[mirakc/docs/notes.md]: https://github.com/mirakc/mirakc/blob/main/docs/notes.md#mirakc-leaks-memory
