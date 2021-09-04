# トラブルシューティング

最後に，問題が発生したときの対処方法について説明を行い，本記事を終了したいと思い
ます．

## ログメッセージの確認

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
# docker-compose.ymlからの抜粋
...
    environment:
      # mirakcのログレベルをdebugに
      RUST_LOG: 'info,mirakc=debug'
```

フィルター等の外部プログラムが`stderr`に出力するログを確認したい場合は，環境変数
`MIRAKC_DEBUG_CHILD_PROCESS`を以下のように定義してください．


```yaml
# docker-compose.ymlからの抜粋
...
    environment:
      MIRAKC_DEBUG_CHILD_PROCESS: ''
      RUST_LOG: 'info,mirakc=debug'
      MIRAKC_ARIB_NO_TIMESTAMP: ''
      MIRAKC_ARIB_LOG: 'info'
```

環境変数`MIRAKC_ARIB_LOG_NO_TIMESTAMP`を定義すると，`mirakc-arib`のログメッセー
ジからタイムスタンプを削除できます．

デフォルト設定では，`mirakc-arib`は何もログを出力しません．必ず，環境変数
`MIRAKC_ARIB_LOG`でログレベルを指定してください．`RUST_LOG`のように，コマンド毎
にログレベルを指定することも可能です．

```yaml
# docker-compose.ymlからの抜粋
...
    environment:
      # filter-serviceとfilter-programはdebug，それ以外はinfo
      MIRAKC_ARIB_LOG: 'info,filter-service=debug,filter-program=debug'
```

## フィルターの動作確認

フィルターで問題が発生している場合，フィルターに指定した外部プログラムを単独実行
することで，問題の切り分けが可能です．

例えば，以下のコマンドで`mirakc-arib filter-service`の動作を確認できます．

```shell
sudo docker exec -it mirakc sh -c \
  'curl -s http://localhost:40772/api/channels/GR/27/stream | \
     MIRAKC_ARIB_LOG=debug mirakc-arib filter-service --sid=1024 >/dev/null'
```

本記事では説明しませんでしたが，ジョブ設定（`jobs`）に指定されている外部プログラ
ムも上記と同様の方法で動作を確認できます．

## EPGStationの番組表が更新されない

稀にこのような状況が発生します．考えられる原因が複数存在するため，１つづつ確認し
てください．

### 空きチューナーが存在しない

EPG関連のジョブの優先度は低く設定されているため，ストリーミングにより全てのチュ
ーナーが使用されていると，番組表更新のためのジョブが実行できません．その結果，い
つまで経っても番組表が更新されないという状況が発生します．

以下のコマンドでチューナーの使用状況が確認できます．

```shell
curl -s http://mirakc:40772/api/tuners | jq .[].isFree
```

全てのチューナーが使用されている場合，`false`のみが表示されます．

この状況では，少なくとも１つのチューナーを開放しないと，番組表は更新されません．

### EPG関連ジョブが終了しない

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

## No space for tnuer#x.x.x, drop the chunk

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

## GitHub Issuesの検索

同様の問題が他の人の環境でも発生している場合，[GitHub Issues]に既に報告済みかも
しれません．思いつくキーワードを指定して，報告済みの問題を検索することで，解決方
法が見つかるかもしれません．

問題が既に解決済みである場合もあります．`is:closed`での検索してみてください．

もし同様の問題が見つからない場合は，問題を登録してください．既に問題の回避・解決
方法が分かっている場合は，その方法も一緒に登録しましょう．同様の問題に悩んでいる
人の手助けになります．

[GitHub Issues]: https://github.com/mirakc/mirakc/issues
