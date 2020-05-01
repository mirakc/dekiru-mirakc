# masnagam/mirakcイメージ

Docker Hubで配布している[masnagam/mirakc]イメージには，起動に必要なツールがすべ
て含まれています．各自でツールをソースからビルドしたりインストールする必要はあり
ません．

[mirakc/docs/docker.md]に記載されているように，`masnagam/mirakc`イメージには複数
のタグがあり，この中から自分の目的に合ったタグを選択します．以下に代表的なタグを
列挙します．

* latest
  * debianのエイリアス
* debian
  * Debian/Busterベースの最新リリースイメージ
* alpine
  * Alpine/3.11ベースの最新リリースイメージ
* master
  * master-debianのエイリアス
* master-debian
  * Debian/BusterベースのGitHub masterブランチの最新コミットから作成したイメージ
* alpine
  * Alpine/3.11ベースのGitHub masterブランチの最新コミットから作成したイメージ

これらのイメージは全てマルチアーキイメージです．以下のアーキテクチャーをサポート
しています．

* amd64
* arm32v6
* arm32v7
* arm64v8

Raspberry PiなどのSBCで実行する場合は，先の例で使用した`masnagam/mirakc:alpine`
を選択することを推奨しています．詳しい理由については[mirakc/docs/notes.md]を見て
ください．

[masnagam/mirakc]: https://hub.docker.com/repository/docker/masnagam/mirakc
[mirakc/docs/docker.md]: https://github.com/masnagam/mirakc/blob/master/docs/docker.md#pre-built-images-in-dockerhub
[mirakc/docs/notes.md]: https://github.com/masnagam/mirakc/blob/master/docs/notes.md#mirakc-leaks-memory
