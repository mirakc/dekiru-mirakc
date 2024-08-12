# mirakcの設定

> 本記事の設定ファイルはすべてYAML形式で記述しますが，TOML形式で記述することも可能です．
> 記述例は[こちら](https://github.com/mirakc/mirakc/blob/main/docs/config.md#configuration)
> に記載されています．

mirakcを起動できるようになったので，次はチャンネルやチューナーなどの設定を行いま
す．

Mirakurunでは，設定する項目ごとにYAMLファイルが存在します．例えば，チャンネル設
定なら`channels.yml`という感じです．一方，mirakcでは，`config.yml`という１つの
設定ファイルにすべての設定を記述します．

本記事では，最低限動かすために必要な設定項目についてのみ説明を行います．他の設定
項目については，[mirakc/docs/config.md]を参照してください．

また，ログに関する設定は，`config.yml`ではなく，いくつかの環境変数を用いて設定し
ます．詳細については，[mirakc/docs/logging.md]を参照してください．

[mirakc/docs/config.md]: https://github.com/mirakc/mirakc/blob/main/docs/config.md
[mirakc/docs/logging.md]: https://github.com/mirakc/mirakc/blob/main/docs/logging.md
