# チャンネル設定

mirakcには，利用可能なチャンネルをスキャンする機能は備わっていません．しかし，Mirakurunのチャンネル
設定と一定の互換性があるため，Mirakurunの設定を流用可能です．

Mirakurunの`channels.yml`の例:

```yaml
# NHK総合
- name: NHK
  type: GR
  channel: '27'

# NHK BS
- name: NHK-BS
  type: BS
  channel: BS15_0
  serviceId: 101
```

mirakcの`config.yml`の例:

```yaml
# `channels`プロパティの要素としてチャンネルを登録
channels:
  # NHK総合
  - name: NHK
    type: GR
    channel: '27'

  # NHK BS
  - name: NHK-BS
    type: BS
    channel: BS15_0
    # プロパティ名が違う
    # 配列で複数指定可能
    services: [101]
```

基本的には，以下のような置き換えでMirakurunの`channels.yml`から変換可能です．

| Mirakuruのプロパティ名 | mirakcのプロパティ名       |
|------------------------|----------------------------|
| serviceId              | services （配列）          |
| satelite               | extra-args （文字列）      |
| space                  | extra-args （文字列）      |
| （なし）               | excluded-services （配列） |

チューナーコマンドとして`recpt1`を使用する場合，以下のようなMirakurun用のチャンネルスキャンスクリプ
トを利用可能です．

* [mcconfig.pl](https://www.jifu-labo.net/2019/02/mirakurn_channels_config/)
* [uru2/scan_ch_mirak.sh](https://gist.github.com/uru2/479b7da80063e39d1ca2cf467e40e290)

mirakc専用のスクリプトもあります．

* [uru2/scan_ch_mirakc_docker.sh](https://gist.github.com/uru2/7f738d864c2789b35c35e6bb7be9d0cb)

どちらも`epgdump`を使用していますが，現時点では`mirakc/mirakc`イメージには含まれていません．
`mirakc/mirakc`イメージからカスタムイメージを作成する方法については後述します．
