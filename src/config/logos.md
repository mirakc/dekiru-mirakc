# 局ロゴの表示

> 以下の内容は`0.18.0`以降でのみ有効です．

番組表の表示が可能なテレビなどでは，番組表に局ロゴが表示されます．局ロゴはTSスト
リーム上で定期的に配信されており，テレビなどはこれを抽出・表示しています．

残念ながら，mirakcは起動中の局ロゴ抽出をサポートしていません．しかし，
`resource.logos`プロパティーを設定することで，任意の画像を局ロゴとして利用するこ
とが可能です．

`config.yml`:

```yaml
# NHK総合のロゴ画像を指定
resource:
  logos:
    - service-triple: [32736, 32736, 1024]
      image: /var/lib/mirakc/logos/nhk.png
```

ここで`nhk.png`は局ロゴとして使用する画像です．この画像はどこかから拾ってくるか
（ライセンスに注意！），自力でTSストリームから抽出する必要があります．今回は
`mirakc/mirakc`イメージ含まれている`mirakc-arib`を使って，TSストリームからロゴを
抽出することにします．

```shell
# mirakcは起動済みとする
mkdir -p logos
curl -sG http://localhost:40772/api/channels/GR/27/stream | \
  sudo docker run --rm -i --entrypoint=/usr/bin/env mirakc/mirakc:alpine \
    mirakc-arib collect-logos | head -1 | jq -Mr '.data' | \
    base64 -d >logos/nhk.png
```

局ロゴは頻繁には流れてこないので，抽出には数分（NHKの場合は１０分以下）かかりま
す．お茶でも飲んでしばらく待ちましょう．

抽出が完了したら，`docker-compose.yml`を以下のように書き換えて，コンテナーに
`logos`フォルダーをマウントします．

```yaml
version: '3'

services:
  mirakc:
    ...
    volumes:
      ...
      # logosフォルダーを読み取り専用でマウント
      - ./logos:/var/lib/mirakc/logos:ro
```

コンテナーを再起動して動作を確認します．

```console
$ curl -I http://localhost:40772/api/services/3273601024/logo
HTTP/1.1 200 OK
content-length: 760
server: mirakc-core/0.18.0
accept-ranges: bytes
last-modified: Sat, 19 Jun 2021 07:26:36 GMT
content-disposition: inline; filename="nhk.png"
content-type: image/png
etag: "e830d5818f5490f2:2f8:60cd9c2c:0"
date: Sat, 19 Jun 2021 07:29:46 GMT
```

ブラウザーを使えば，実際に表示される画像を確認できます．

設定した局ロゴは，`/api/iptv/*`にも反映されます．

```console
$ curl -sG http://localhost:40772/api/iptv/playlist | grep tvg-logo
... tvg-logo="http://localhost:40772/api/services/3273601024/logo" ...

$ curl -sG http://localhost:40772/api/iptv/epg | xmllint --format - | grep icon
    <icon src="http://localhost:40772/api/services/3273601024/logo"/>
```
