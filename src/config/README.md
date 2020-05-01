# mirakcの設定

mirakcを起動できるようになったので，チャンネルやチューナーの設定を行っていきま
す．

Mirakurunでは，設定する項目ごとにYAMLファイルが存在します．例えば，チャンネル設
定なら`channels.yml`という感じです．一方，mirakcでは，`config.yml`という１つの
YAMLファイルにすべての設定を記述します．

本記事では，最低限動かすために必要な設定項目についてのみ説明を行います．他の設定
項目については，[mirakc/docs/config.md]を参照してください．

また，ログに関する設定は，`config.yml`ではなく，いくつかの環境変数を用いて設定し
ます．詳細については，[mirakc/docs/logging.md]を参照してください．

[mirakc/docs/config.md]: https://github.com/masnagam/mirakc/blob/master/docs/config.md
[mirakc/docs/logging.md]: https://github.com/masnagam/mirakc/blob/master/docs/logging.md
