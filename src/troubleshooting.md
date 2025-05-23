# トラブルシューティング

最後に，問題が発生したときの対処方法について説明を行い，本記事を終了したいと思い
ます．

## ログメッセージの確認 {#logs}

問題発生時にまず行うべき事柄の１つは，システムから出力されるログを確認することで
す．以下のコマンドは，`mirakc`コンテナーの最新1000行のログを表示します．

```shell
sudo docker logs --tail=1000 mirakc
```

特定のログレベルのメッセージのみ表示させたい場合は，`grep`を使います．

```shell
sudo docker logs --tail=1000 mirakc 2>&1 | grep -e WARN -e ERROR
```

例えば，受信できないチャンネル番号が指定されていた場合，以下のようなエラーが出力
されます．

```console
... scan-services: Failed: JSON error: EOF while parsing a value at line 1 column 0
... sync-clocks: Failed: JSON error: EOF while parsing a value at line 1 column 0
```

ログレベルを変更することで，より多くの情報を得られるようになります．

```yaml
# compose.yamlからの抜粋
...
    environment:
      # mirakcのログレベルをdebugに
      RUST_LOG: info,mirakc=debug
```

フィルター等の外部プログラムが`stderr`に出力するログを確認したい場合は，環境変数
`MIRAKC_DEBUG_CHILD_PROCESS`を以下のように定義してください．


```yaml
# compose.yamlからの抜粋
...
    environment:
      MIRAKC_ARIB_LOG: info
      MIRAKC_ARIB_LOG_NO_TIMESTAMP: 1
      MIRAKC_DEBUG_CHILD_PROCESS: 1
      RUST_LOG: info,mirakc=debug
```

環境変数`MIRAKC_ARIB_LOG_NO_TIMESTAMP=1`を定義すると，`mirakc-arib`のログメッセ
ージからタイムスタンプを削除できます．

デフォルト設定では，`mirakc-arib`は何もログを出力しません．必ず，環境変数
`MIRAKC_ARIB_LOG`でログレベルを指定してください．`RUST_LOG`のように，コマンド毎
にログレベルを指定することも可能です．

```yaml
# compose.yamlからの抜粋
...
    environment:
      # filter-serviceとfilter-programはdebug，それ以外はinfo
      MIRAKC_ARIB_LOG: info,filter-service=debug,filter-program=debug
```

`MIRAKC_ARIB_LOG`に指定可能なログレベルの一覧が
[こちら]([mirakc/mirakc-aribのREADME.md])にあります．

## パイプラインの動作確認 {#pinelines}

mirakcは，TSストリームに関する処理のほぼ全てを外部プログラムで実行します．この設
計には，余分なリソースを消費するデメリットがある一方で，以下のようなメリットがあ
ります．

* 既存のTSストリーム処理を行うプログラムを再利用できる
* スクリプト等でTSストリーム処理をカスタマイズできる
* TSストリーム処理部のみ取り出しでデバッグできる

mirakcでは，TSストリーム処理プログラムをパイプで連結したものを「パイプライン」と
呼んでいます．パイプラインの基本的構成については
[こちら](https://github.com/mirakc/mirakc/blob/main/docs/inside-mirakc.md)を参
照してください．

ログレベルを`debug`に変更すると，パイプラインに関するログが出力されるようになり
ます．

```
... DEBUG pipeline{id=0.0 label="tuner"}: ...
... DEBUG pipeline{id=0.0.1 label="epg.scan-services"}: ...
... DEBUG pipeline{id=0.1.1 label="epg.sync-clocks"}: ...
```

上記のように，パイプラインには一意の`id`と処理毎に`label`が付与されています．こ
れらを使いログを`grep`することで，特定のパイプラインの動作を確認できます．

各パイプラインの最初のログには，以下のように実行したコマンドの情報が出力されます．

```
... Spawned command="<実行したコマンド>" pid=<そのPID>
```

動作確認対象のパイプラインを構成するコマンドを集めた後，これらを直接実行すること
で問題の発生源を切り分けることが可能です．以下は`GET /api/channels/GR/27/stream`
で実行されるパイプラインを直接実行する場合の例です．

```shell
# チューナーコマンドで`curl`を，`decode-filter`に`socat`を使用しています
curl -fsSL http://remote:40772/api/channels/GR/27/stream?decode=0 | \
  socat -,cool-write tcp-connect:tsd:40773
```

通常のコマンド実行と同じなので，各プログラムのログレベルを変更することも，`gdb`
などをアタッチすることも可能です．

## フィルターの動作確認 {#filters}

フィルターで問題が発生している場合，フィルターに指定した外部プログラムを単独実行
することで，問題の切り分けが可能です．

例えば，以下のコマンドで`mirakc-arib filter-service`の動作を確認できます．

```shell
sudo docker exec -it mirakc sh -c \
  'curl -s http://localhost:40772/api/channels/GR/27/stream?decode=0 | \
     MIRAKC_ARIB_LOG=debug mirakc-arib filter-service --sid=1024 >/dev/null'
```

本記事では説明しませんでしたが，ジョブ設定（`jobs`）に指定されている外部プログラ
ムも上記と同様の方法で動作を確認できます．

## EPGStationの番組表が更新されない {#epgstation}

稀にこのような状況が発生します．考えられる原因が複数存在するため，１つづつ確認し
てください．

### 空きチューナーが存在しない {#epgstation-tuner-unavailable}

EPG関連のジョブの優先度は低く設定されているため，ストリーミングにより全てのチュ
ーナーが使用されていると，番組表更新のためのジョブが実行できません．その結果，い
つまで経っても番組表が更新されないという状況が発生します．

以下のコマンドでチューナーの使用状況が確認できます．

```shell
curl -s http://mirakc:40772/api/tuners | jq .[].isFree
```

全てのチューナーが使用されている場合，`false`のみが表示されます．

この状況では，少なくとも１つのチューナーを開放しないと，番組表は更新されません．

### EPG関連ジョブが終了しない {#epgstation-job-not-finished}

EPG関連ジョブは多くても１つだけ実行されるようになっています．そのため，あるEPG関
連ジョブが終了しなくなると，他のEPG関連ジョブを実行することができず，番組表が更
新されない状態に陥ります．

この状態に陥った場合，以下のようなログが繰り返し出力されます．

```
... WARN mirakc::job: update-schedules: Already running, skip
```

`update-schedules`ジョブは，進捗しない状態が30秒続くと停止するようになっています．
この値は，`config.jobs.update-schedules.command`のコマンドラインオプションとして
指定可能です．詳しくは`mirakc-arib -h collect-eits`を参照してください．

ジョブ停止時間を過ぎても停止しない場合，特定セクションのデータが到着しないような
状況に陥っています．この場合は，`update-schedules`ジョブを停止するしかありません．
`ps`コマンドでジョブのPIDを調べて`kill`してください．ジョブは登録してある時間に
なったら再び実行されるので，これを`kill`しても問題ありません．

いちいちPIDを調べて`kill`するのが面倒だという人は，`timeout`をジョブのコマンドに
付けておきましょう．以下の例では，`update-schedules`ジョブの最大実行時間を10分に
設定しています（[Mirakurun互換の動作](https://github.com/Chinachu/Mirakurun/blob/master/doc/Configuration.md)）．

```yaml
jobs:
  update-schedules:
    command: >-
      timeout 600 mirakc-arib collect-eits
      {{#sids}} --sids={{{.}}}{{/sids}}{{#xsids}} --xsids={{{.}}}{{/xsids}}
    schedule: '0 7,37 * * * * *'
```

`timeout`によりジョブが中断された場合，番組表の一部が未取得・未更新の状態になり
ます．前述のように，ジョブは再実行されるため，時間が経てば全ての番組情報が取得さ
れます．

## No space for tnuer#x.x.x, drop the chunk {#no-space}

mirakcはチューナー共有をサポートしており，チューナーから読み込んだデーターをクライアント毎のキュー
に書き込みます．このキューが溢れた場合，以下のようなワーニングメッセージが出力されます．

```
...: No space for tuner#0.6.1, drop the chunk
```

本事象に関する問題がいくつか報告されています．

* [EPGStation ライブ視聴時の再生が約5秒で止まる](https://github.com/mirakc/mirakc/issues/18)
* [No space in chunk queues for a few minutes](https://github.com/mirakc/mirakc/issues/236)

解析の結果，環境依存問題であることがわかっています．まずはログレベルを上げてログメッセージから何が
起きているのか確認してください．その後の対処は，発生している事象に依存するため，ここで明確に対処法
を記述することはできません．上記報告済み不具合のやり取りを確認の上，自分で問題を解決してください．
もちろん，mirakcの不具合と思われる事象については，問題を登録していただけると助かります．

## mirakc/mirakcイメージでTSストリームがデコードできない {#decode}

mirakc/mirakcイメージにはTSストリームをデコードするためのコマンドが**含まれていません**．
そのため，このイメージに対して`arib-b25-stream-test`を設定すると，以下のようなエラーが表示されて，
TSストリームを取得できなくなります．

```console
$ curl -sG http://localhost:40772/api/channels/GR/27/stream
{"code":500,"reason":null,"errors":[]}

# actix-webのログレベルをdebug以上に設定しておく必要があります
$ docker logs mirakc | grep arib-b25-stream-test
... DEBUG ... CommandFailed(UnableToSpawn("arib-b25-stream-test", ...
```

デコードするためには，以下のいずれかの対応が必要です．

* `filters.decode-filter.command`に指定したコマンドを含むカスタムイメージを作成する
* `filters.decode-filter.command`に指定したコマンドを`docker run`の`-v`オプションでホスト上のコマン
  ドをコンテナーにマウントする
  * ただし，コンテナー内で実行可能なコマンドに限る
* `socat`を使って外部サーバー上でデコードする

## mirakc-timeshift-fsが起動しない {#timeshift-fs}

`mirakc-timeshift-fs`起動時に以下のようなエラーメッセージが表示され起動できない場合，

```
Error: IoError(Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

`fusermount`（もしくは`fusermount3`）が環境にインストールされていないか，インス
トール先が`PATH`に含まれていない可能性があります．

[docker/Dockerfile.debian](https://github.com/mirakc/mirakc/blob/main/docker/Dockerfile.debian)
などを参考に，不足パッケージをインストールし，`PATH`を適切に設定してください．

`-o allow_other`などのオプションを使用する場合は，起動前に`/etc/fuse.conf`を適切
に設定しておく必要があります．詳細は`man fuse`を参照してください．

## GitHub Issuesの検索 {#github-issues}

同様の問題が他の人の環境でも発生している場合，[GitHub Issues]に既に報告済みかも
しれません．思いつくキーワードを指定して，報告済みの問題を検索することで，解決方
法が見つかるかもしれません．

問題が既に解決済みである場合もあります．`is:closed`での検索してみてください．

もし同様の問題が見つからない場合は，問題を登録してください．既に問題の回避・解決
方法が分かっている場合は，その方法も一緒に登録しましょう．同様の問題に悩んでいる
人の手助けになります．

[GitHub Issues]: https://github.com/mirakc/mirakc/issues
