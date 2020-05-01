# フロントエンドアプリケーションとの連携

mirakcの準備が整ったので，フロントエンドアプリケーションから利用できるように設定
を変更してみましょう．

## EPGStation

EPGStationのセットアップ方法は，すでに色々なところに書かれているので省略します．
以下，既にセットアップ済みの環境が存在すると仮定して説明を行います．

もし，実験目的で設定変更しようとしているのでしたら，以下を実行する前にデータベー
スのバックアップを作成しておいてください．設定を変更し，EPGStationを再起動する
と，データベースに保存されているEPGデータは上書きされます．Mirakurunとmirakcでは
利用している文字コード変換処理が異なるため，録画ルールで使用している正規表現など
が機能しなくなります．

それでは設定を変更します．変更箇所は以下の１箇所のみです．

```jsonc
{
  // 変更不要な設定は省略
  "mirakurunPath": "http://raspberrypi.local:40772/"
}
```

EPGStationを動かしているサーバーから`raspberrypi.local`をmDNSを使って名前解決で
きることを前提としています．mDNSが使えない場合は，IPアドレスもしくは名前解決可能
なホスト名に置き換えてください．

EPGStationを再起動すると置き換えは完了します．チャンネル設定で指定したチャンネル
の番組表が表示されていれば，置き換え成功です．

## TVTest

原因はよく分かっていませんが，Mirakurun用の`BonDriver_Mirakurun`ではうまく動かな
いようです．代わりに，epgdatacapbonさんが作成した[BonDriver_mirakc]を使ってくだ
さい．

`SERVICE_SPLIT=1`とすることでmirakcと`BonDriver_mirakc`間の通信量を減らすことが
できます．

[BonDriver_mirakc]: https://github.com/epgdatacapbon/BonDriver_mirakc
