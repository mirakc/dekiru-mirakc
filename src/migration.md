# v1からv2への移行

ここでは，v1からv2への移行について説明します．

## EPGキャッシュの再構築

内部のデータ構造が変更されたことに伴い，v1のEPGキャッシュをv2では再利用できませ
ん．そのため，v2移行時にEPGキャッシュが再構築されます．

チャンネル数に依存しますが，EPGキャッシュの再構築は時間がかかります．例えば，地
デジ12チャンネル＋BS12チャンネルのEPGキャッシュの再構築には40分以上必要です．そ
のため，録画予約が詰まっていない時間帯を選んでv2移行を行ってください．

## config.yml

[mirakc内部のサービスID及び番組IDからTSIDを削除したことに伴い](https://github.com/mirakc/mirakc/issues/690)，
`timeshift.recorders[].service-triple`が`timeshift.recorders[].service-id`に変更
されました．

```yaml
# v1
timeshift:
  recorders:
    bs1:
      service-triple: [4, 16625, 101]

# v2
timeshift:
  recorders:
    bs1:
      service-id: 400101  # Mirakurun Service ID
```

`resource.logos[].service-triple` も同様です．

```yaml
# v1
resource:
  logos:
    - service-triple: [4, 16625, 101]

# v2
resource
  logos:
    - service-id: 400101  # Mirakurun Service ID
```

これに伴い，Web APIのいくつかのJSON形式が変更されましたが，本文書の説明対象外な
ので，詳細は省略します．

[mirakc内部でopenapi.jsonを生成するようになったため](https://github.com/mirakc/mirakc/issues/623)，`mirakurun.openapi-json`は削除されました．

```yaml
# v1
mirakurun:
  openapi-json: /path/to/mirakurun.openapi.json

# v2: Removed
```

## タイムシフト録画

チャンクサイズの既定値が変更されました．

* v1
  * 163840000
  * `8192`の倍数でなければならない
* v2
  * 154009600
  * `8192`の倍数でなければならない
    * `8192 * 188`の倍数を推奨

`ts-file`のサイズも変わるため，再計算してください．

`data-file`のJSON形式も変更されたため，mirakcを実行すると以下のようなエラーが表
示され，タイムシフト録画は開始されません．

```
ERROR mirakc_core::timeshift: Failed to load records err=JSON \
  error: missing field `id` at line 1 column 233 recorder.name="nhk"
ERROR mirakc_core::timeshift: Recover or simply remove <data-file> recorder.name="nhk"
```

エラーメッセージで提案されているように，以下のどちらかを実行してください．

1. `mirakc rebuild-timeshift`を実行し，タイムシフト録画ファイルを再構築する
2. `data-file`を削除する

`mirakc rebuild-timeshift`によるタイムシフト録画ファイルの再構築には時間がかかります
（詳細は[こちら](https://github.com/mirakc/mirakc/issues/690#issuecomment-1384949952)）．
当然，再構築が完了するまでタイムシフト録画を実行することは不可能なので，この方法
はお勧めしません．mirakc (v1)を起動し，必要なレコードのみを別ファイルに保存．
その後，`data-file`を削除することを推奨します．

レコードの保存は，`mirakc-timesift-fs`を利用しているならファイルをコピーすれば
良いだけです．`curl`とWeb APIを使っても保存できますが，説明は省略します．
