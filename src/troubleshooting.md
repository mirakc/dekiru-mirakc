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
      RUST_LOG: info,mirakc=debug
```

フィルター等の外部プログラムが`stderr`に出力するログを確認したい場合は，環境変数
`MIRAKC_DEBUG_CHILD_PROCESS`を以下のように定義してください．


```yaml
# docker-compose.ymlからの抜粋
...
    environment:
      MIRAKC_DEBUG_CHILD_PROCESS: ''
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

## GitHub Issuesの検索

同様の問題が他の人の環境でも発生している場合，[GitHub Issues]に既に報告済みかも
しれません．思いつくキーワードを指定して，報告済みの問題を検索することで，解決方
法が見つかるかもしれません．

問題が既に解決済みである場合もあります．`is:closed`での検索してみてください．

もし同様の問題が見つからない場合は，問題を登録してください．既に問題の回避・解決
方法が分かっている場合は，その方法も一緒に登録しましょう．同様の問題に悩んでいる
人の手助けになります．

[GitHub Issues]: https://github.com/masnagam/mirakc/issues
