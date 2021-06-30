# フロントエンドアプリケーションとの連携

mirakcの準備が整ったので，フロントエンドアプリケーションから利用できるように設定
を変更してみましょう．

## EPGStation v1

EPGStationのセットアップ方法は，すでに色々なところに書かれているので省略します．
以下，既にセットアップ済みの環境が存在すると仮定して説明を行います．

もし，実験目的で設定変更しようとしているのでしたら，以下を実行する前にデータベー
スのバックアップを作成しておいてください．設定を変更し，EPGStationを再起動する
と，データベースに保存されているEPGデータは上書きされます．Mirakurunとmirakcでは
利用している文字コード変換処理が異なるため，録画ルールで使用している正規表現など
が機能しなくなります．

それでは設定を変更します．変更箇所は以下の１箇所のみです．

```json
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

## EPGStation v2

EPGStation v2では，設定ファイルの形式がJSONからYAMLに変わりました．しかし，
Mirakurunからの移行時に書き換える箇所は変わりません．

```yaml
# 変更不要な設定は省略
mirakurunPath: http://raspberrypi.local:40772/
```

## TVTest

原因はよく分かっていませんが，Mirakurun用の`BonDriver_Mirakurun`ではうまく動かな
いようです．代わりに，epgdatacapbonさんが作成した[BonDriver_mirakc]を使ってくだ
さい．

`SERVICE_SPLIT=1`とすることでmirakcと`BonDriver_mirakc`間の通信量を減らすことが
できます．

[BonDriver_mirakc]: https://github.com/epgdatacapbon/BonDriver_mirakc

## Kodi

[PVR IPTV Simple Client]アドオンをインストールすれば，Kodiからライブ視聴が可能と
なります．

アドオンの設定画面から，以下を設定してください．

* General | M3U Play List URL
  * http&#58;//`HOST`:`PORT`/api/iptv/playlist
* EPG Settings | XMLTV URL
  * http&#58;//`HOST`:`PORT`/api/iptv/epg

`HOST`および`PORT`は，自分の環境に合わせて書き換えてください．

スキン設定からフォントを`Arial based`に変更しても日本語が正しく表示されない場合
があります．そのような場合は，Kodi内の`arial.ttf`を日本語グリフを含んだUnicode対
応フォントに置き換えることで改善します．以下にmacOSでの設定例があります．

* https://gist.github.com/masnagam/e79a1e5e02c8343f5117dd6572be8031

[タイムシフト・ファイルシステム](./config/timeshift.md#タイムシフト・ファイルシステム)をSambaで公開
している場合，`smb://<host>/<folder>`をビデオソースとして登録することで，録画ファイルをKodi
で再生できます．以下のオプションがお薦めです．

* Viewtype: WideList
* Sort by: Date
* Order: Descending

Sambaを使った場合，サムネイル画像の生成などのために集中的なファイルアクセスが発生し，
mirakc-timeshift-fsおよびSambaを稼働させているマシンが一時的に高負荷状態になることがあります．この
負荷状態は，サムネイル画像の生成をやめるなどKodiの設定を見直すことでを回避できます．しかし，他にも
問題があるようで，タイムシフト録画ファイルが大量に存在すると，Kodiがハングすることがありました．そ
のため，Kodiでタイムシフト録画を視聴する場合，後述のMiniDLNA経由にすることを推奨します．

[PVR IPTV Simple Client]: https://kodi.wiki/view/Add-on:PVR_IPTV_Simple_Client

## MiniDLNA (ReadyMedia)

タイムシフト録画をTVで視聴しようと思っても，TVがSambaをサポートしていない場合があります．このような
場合でも，もしTVがDLNA/UPnPをサポートしているなら，[MiniDLNA]を使えばTVで視聴可能です．

MiniDLNAは[inotify]を使って録画ファイルの更新を検出できますが，残念ながら以下のような状況では機能し
ません．

* [タイムシフト・ファイルシステム](./config/timeshift.md#タイムシフト・ファイルシステム)の実装で利
  用しているFUSEでは，inotifyがサポートされていません
  * https://github.com/libfuse/libfuse/wiki/Fsnotify-and-FUSE
* `mount.cifs`でマウントしたファイルシステムはinotifyをサポートしていません
  * https://lists.samba.org/archive/linux-cifs-client/2009-April/004318.html
* Dockerコンテナーにマウントしたホストファイルシステム上では，inotifyが動作しない場合があります
  * https://github.com/moby/moby/issues/18246

そこで，MiniDLNAにパッチを当てたDockerイメージ[mirakc/minidlna]を用意してあります．これを使えば
DLNA/UPnPをサポートしているTVでタイムシフト録画を視聴可能です．

使用方法については，[README.md](https://github.com/mirakc/docker-minidlna)を参照してください．

[MiniDLNA]: https://sourceforge.net/projects/minidlna/
[inotify]: https://ja.wikipedia.org/wiki/Inotify
[mirakc/minidlna]: https://hub.docker.com/r/mirakc/minidlna
