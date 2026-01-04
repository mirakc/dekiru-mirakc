# フロントエンドアプリケーション

mirakcの準備が整ったので，フロントエンドアプリケーションから利用できるように設定を変更してみましょ
う．

## EPGStation v1 {#epgstation-v1}

EPGStationのセットアップ方法は，すでに色々なところに書かれているので省略します．以下，既にセットアッ
プ済みの環境が存在すると仮定して説明を行います．

もし，実験目的で設定変更しようとしているのでしたら，以下を実行する前にデータベースのバックアップを作
成しておいてください．設定を変更し，EPGStationを再起動すると，データベースに保存されているEPGデータ
は上書きされます．Mirakurunとmirakcでは利用している文字コード変換処理が異なるため，録画ルールで使用
している正規表現などが機能しなくなります．

それでは設定を変更します．変更箇所は以下の１箇所のみです．

```json
{
  // 変更不要な設定は省略
  "mirakurunPath": "http://raspberrypi.local:40772/"
}
```

EPGStationを動かしているサーバーから`raspberrypi.local`をmDNSを使って名前解決できることを前提として
います．mDNSが使えない場合は，IPアドレスもしくは名前解決可能なホスト名に置き換えてください．

EPGStationを再起動すると置き換えは完了します．チャンネル設定で指定したチャンネルの番組表が表示されて
いれば，置き換え成功です．

## EPGStation v2 {#epgstation-v2}

EPGStation v2では，設定ファイルの形式がJSONからYAMLに変わりました．しかし，Mirakurunからの移行時に書
き換える箇所は変わりません．

```yaml
# 変更不要な設定は省略
mirakurunPath: http://raspberrypi.local:40772/
```

## miraview {#miraview}

[miraview]はmirakc用のWeb UIです．コンテナー利用者であれば，README.mdに書かれている手順で簡単に導入
できます．

`curl`と`jq`さえあれば何でもできるけど，さすがに番組表はGUIで見たいという人は利用を検討しましょう．

## mirakc-ui {#mirakc-ui}

[mirakc-ui]はmirakc用のWeb UIです．番組表の表示以外に録画管理をサポートしています．README.mdに記載さ
れている手順で簡単に導入できます．

`miraview`とは異なり，`mirakc-ui`はWeb UI専用のサーバーを起動します．

## TVTest {#tvtest}

原因はよく分かっていませんが，Mirakurun用の`BonDriver_Mirakurun`ではうまく動かないようです．代わり
に，epgdatacapbonさんが作成した[BonDriver_mirakc]を使ってください．

`SERVICE_SPLIT=1`とすることでmirakcと`BonDriver_mirakc`間の通信量を減らすことができます．

[BonDriver_mirakc]: https://github.com/epgdatacapbon/BonDriver_mirakc

## Kodi {#kodi}

> [!IMPORTANT]
> Kodiでタイムシフト録画を視聴する場合は **21.1** 以降を使用してください．
> これよりも古いバージョンには問題があり，タイムシフト録画を快適に再生できません．
> 詳細については
> [こちら](https://github.com/mirakc/docker-timeshift-x/tree/main/gerbera#kodi-gets-stuck-when-starting-playback-solved)
> を参照してください．

[PVR IPTV Simple Client]アドオンをインストールすれば，Kodiからライブ視聴が可能となります．

アドオンの設定画面から，以下を設定してください．

* General | M3U Play List URL
  * http&#58;//`HOST`:`PORT`/api/iptv/playlist
* EPG Settings | XMLTV URL
  * http&#58;//`HOST`:`PORT`/api/iptv/epg

`HOST`および`PORT`は，自分の環境に合わせて書き換えてください．

スキン設定からフォントを`Arial based`に変更しても日本語が正しく表示されない場合があります．そのよう
な場合は，Kodi内の`arial.ttf`を日本語グリフを含んだUnicode対応フォントに置き換えることで改善します．
以下にmacOSでの設定例があります．

* https://gist.github.com/masnagam/e79a1e5e02c8343f5117dd6572be8031

タイムシフト録画もKodiで視聴可能です．`Settins / Services`から`UPnP / DLNA`を有効にしましょう．後述の
`mirakc/timeshift-gerbera`を実行すれば，`video source`の選択ダイアログに`UPnP devices`が表示されま
す．これを選択するとGerberaのアイコンと共に`Timeshift`が表示されるので，これを追加します．

各タイムシフト・レコーダーのフォルダーを選択すると，録画一覧が表示されます．以下の表示オプションがお
薦めです．

* Viewtype: WideList
* Sort by: Date
* Order: Descending

[Samba]などで共有されたフォルダー内に大量の動画が存在する場合，サムネイル画像の生成などのために集中
的なファイルアクセスが発生し，Samba（および`mirakc-timeshift-fs`）を稼働させているマシンが一時的に高
負荷状態になることがあります．この負荷状態は，サムネイル画像の生成をやめるなどKodiの設定を見直すこと
で回避できます．しかし，他にも問題があるようで，タイムシフト録画ファイルが大量に存在すると，Kodiがハ
ングすることがありました．そのため，Kodiでタイムシフト録画を視聴する場合，UPnP/DLNAの使用を推奨しま
す．

[PVR IPTV Simple Client]: https://kodi.wiki/view/Add-on:PVR_IPTV_Simple_Client

### バージョンの固定

Kodiを更新すると，PVR IPTV Simple Clientが動作しなくなったり，PVR IPTV Simple Clientが原因で起動直後
にクラッシュが発生することが度々あります．そのため，Kodiの更新は慎重に行いましょう．

[`flatpak`]を使用しているなら，以下の方法で`flatpak update`による自動更新からKodiを除外できます．

```shell
# See https://unix.stackexchange.com/questions/673741

# バージョンを固定する機能はないが，自動更新を停止することは可能
flatpak mask tv.kodi.Kodi
```

発生している問題に依存しますが，以下の方法で過去バージョンにダウングレードできます．

```shell
# See https://github.com/flatpak/flatpak/issues/3097

# 以下のコマンドで表示される一覧から戻したいバージョンのコミットハッシュを探す
flatpak remote-info --log flathub tv.kodi.Kodi

# 指定バージョンにダウングレード
sudo flatpak update --commit=<commit-hash> tv.kodi.Kodi
```

[`flatpak`]: https://flatpak.org/

## mirakc/timeshift-gerbera {#mirakc-timeshift-gerbera}

多くのTVはUPnP/DLNAをサポートしているため，[Gerbera]を使えばタイムシフト録画を視聴できます．Gerbera
の設定は多少面倒なので，専用のDockerイメージ[mirakc/timeshift-gerbera]を用意してあります．

このDockerイメージでは，コンテナー内で`mirakc-timeshift-fs`を実行するため，自動アンマウントやタイム
シフト・ファイルシステムのファイル所有権の問題が発生しません．そのため，`mirakc/timeshift-fs`を直接
使用するのではなく，`mirakc/timeshift-*`の使用を推奨します．

類似のDockerイメージを自分で作成したいという方は，[GitHub](https://github.com/mirakc/docker-timeshift-x)
にソースをアップロードしてあるので参考にしてください．

VLCはGerberaから供給されるタイムシフト録画を問題なく再生できることを確認済みです．

DLNA/UPnPをサポートしていない機器でもSambaをサポートしているなら，代わりに[mirakc/timeshift-samba]を
使うことができます．

## mirakc-searchとmirakc-rec {#mirakc-contrib}

コマンド操作に慣れている人は，[mirakc/contrib]にあるスクリプトの利用をお勧めします．

> [!TIP]
> タイムシフト録画用の`timeshift/timeshift.sh`も追加しました

`mirakc-search`

```sh
#!/bin/sh
export MIRAKC_SEARCH_CUSTOM_PROGRAM_JQ_DIR=/path/to/program-jq
export MIRAKC_SEARCH_BASE_URL=http://mirakc:40772
sh /path/to/mirakc/contrib/search/search.sh "$@"
```

`mirakc-rec`

```sh
#!/bin/sh
export MIRAKC_REC_BASE_URL=http://mirakc:40772
sh /path/to/mirakc/contrib/recording/recording.sh "$@"
```

変数と`/path/to/...`は，自分の環境に合わせて書き換えてください．

この２つのスクリプトを使って，BSPの映画を録画予約してみます．

```console
# BSPの映画を検索
# ビルトインのフィルター以外にも，直接フィルターをインラインで指定することも可能
$ mirakc-search movie gt-1h msid 'map(select(.msid == 400103))'
ID           START             END               MINS  ...
40010309234  2023-06-23 13:00  2023-06-23 14:45  105   ...
...

# 録画予約
$ mirakc-rec add 40010309234
ID           STATE      START             END               MINS  ...
40010309234  scheduled  2023-06-23 13:00  2023-06-23 14:45  105   ...
```

`-h`オプションで各スクリプトのヘルプが表示されます．

[miraview]: https://github.com/maeda577/miraview
[mirakc-ui]: https://github.com/ansanloms/mirakc-ui
[Gerbera]: https://gerbera.io/
[Samba]: https://en.wikipedia.org/wiki/Samba
[mirakc/contrib]: https://github.com/mirakc/contrib
[mirakc/timeshift-gerbera]: https://hub.docker.com/r/mirakc/timeshift-gerbera
[mirakc/timeshift-samba]: https://hub.docker.com/r/mirakc/timeshift-samba
