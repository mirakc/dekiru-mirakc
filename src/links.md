# リンク集

リンク先のページの記述は，対象バージョンが古かったり，期待通り動作しない場合があります．参考情報とし
て活用しましょう．

## 録画サーバーを作ろう

読み応えのある記事．

* [mirakc / PX-S1UD / Raspberry Pi 3B で録画鯖 - Qiita](https://qiita.com/motiakoron/items/fdd76590a44461ca061e)
* [極小TVチューナーマシンを組む、あるいはpodman play kube周りについて - あずんひの日](https://aznhe21.hatenablog.com/entry/2023/01/18/mirakc-podman)
  * `docker compose`ではなく，`podman`を使っています
* Rock5BでTV録画サーバを作る
  * [前編](https://zenn.dev/kuruton/articles/293a9cb4f062da)
  * [後編](https://zenn.dev/kuruton/articles/2080e1464155d2)
* [Proxmox VEでdocker-mirakc-konomitvを使ってテレビ視聴サーバーを構築する - Qiita](https://qiita.com/heisannn/items/46c850f78f6104142a43)
  * [Proxmox VE](https://en.wikipedia.org/wiki/Proxmox_Virtual_Environment)という仮想環境管理ツール
    を使っています
* [mirakc+Amatsukaze+juicefsの組み合わせでテレビ録画を見る - mat2uken-blog](https://blog.mat2uken.blog/posts/mirakc-amatsukaze-juicefs/)
  * 自動録画予約を自力実装したり，想定通りの使用法

## compose.yaml

`compose.yaml`の書き方について参考になりそうなもの．

* [5ym/docker-mirakc-epgstation](https://github.com/5ym/docker-mirakc-epgstation)
* [tea-red/tv-recorder](https://github.com/tea-red/tv-recorder)
* [yude/tv-recorder-dockerfile](https://github.com/yude/tv-recorder-dockerfile)
  * 多分，削除されてしまった`collelog/tv-recorder-dockerfile`のfork

## Tips

ちょっとしたテクニックとか．

* [簡単に mirakc を Docker なしで使用する | wktk's blog](https://wktk.jp/entry/use-mirakc-without-docker-easily/)
  * `glibc`とかの互換性（ABI）の問題で動かないこともあるので，自分が何をやっているのか理解している人
    のみ実行しましょう

## 古典

以下は古すぎて，実用的な情報はあまり含まれていません．

* [MirakurunクローンをRustで実装しました - Qiita](https://qiita.com/masnagam/items/e71a234f2e66b1246a37)
  * 多分mirakcに関する最初の記事
* [Mirakurunクローンのmirakcを動かしてみた - にゃののん日記](https://nyanonon.hatenablog.com/entry/20191114/1573731890)
  * 多分私以外が書いたmirakcに関する最初の記事
