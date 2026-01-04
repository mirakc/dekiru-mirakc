# 局ロゴの表示

番組表の表示が可能なテレビなどでは，番組表に局ロゴが表示されます．局ロゴはTSストリーム上で定期的に配
信されており，テレビなどはこれを抽出・表示しています．

残念ながら，mirakcは起動中の局ロゴ抽出をサポートしていません．しかし，`resource.logos`プロパティーを
設定することで，任意の画像を局ロゴとして利用することが可能です．

`config.yml`:

```yaml
# NHK総合のロゴ画像を指定
resource:
  logos:
    - service-id: 3273601024
      image: /var/lib/mirakc/logos/nhk.png
```

ここで`nhk.png`は局ロゴとして使用する画像です．この画像はどこかから拾ってくるか（ライセンスに注意！
），自力でTSストリームから抽出する必要があります．今回は`mirakc/mirakc`イメージに含まれている
`mirakc-arib`を使って，TSストリームからロゴを抽出することにします．

```shell
# mirakcは起動済みとする
mkdir -p logos

# `.data`はdata URL文字列
# Base64文字列の前に`data:image/png;base64,`が付与されている
# Base64文字列内に`,`は存在しないので，これを区切り文字とできる
curl 'http://localhost:40772/api/channels/GR/27/stream?decode=0' -sG | \
  sudo docker run --rm -i --entrypoint=/usr/bin/env mirakc/mirakc:alpine \
    mirakc-arib collect-logos | head -1 | jq -Mr '.data' | \
    cut -d ',' -f 2 | base64 -d >logos/nhk.png
```

局ロゴは頻繁には流れてこないので，抽出には数分（NHKの場合は１０分以下）かかります．お茶でも飲んでし
ばらく待ちましょう．

抽出が完了したら，`compose.yaml`を以下のように書き換えて，コンテナーに`logos`フォルダーをマウントし
ます．

```yaml
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
$ curl http://localhost:40772/api/services/3273601024/logo -I
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
$ curl http://localhost:40772/api/iptv/playlist -sG | grep tvg-logo
... tvg-logo="http://localhost:40772/api/services/3273601024/logo" ...

$ curl http://localhost:40772/api/iptv/epg -sG | xmllint --format - | grep icon
    <icon src="http://localhost:40772/api/services/3273601024/logo"/>
```
