# Raspberry Piのセットアップ

Raspberry Piのセットアップ手順については，「[Raspberry Pi 4で構築する録画マシン]」に記載されているた
め詳細な説明を行いません．

以降の説明は，

* Raspberry Pi 3B
* Raspberry Pi OS Lite (64-bit)
* PX-Q3U4 + [tsukumijima/px4_drv]

で動作確認をしましたが，以下のような環境でもmirakcは動作します．

* Raspberry Pi 2B以上 + Raspberry Pi OS Lite (32-bit)
* ZeroPi + [Armbian] (arm32v7)
* ROCK64 + Armbian (arm64v8)

## デバイスの準備

デバイスにログインしないことには何も始まりません．Raspbianを書き込んだマイクロSDカードに以下の修正を
適用します．

```shell
# SSHの有効化
touch /mnt/boot/ssh

# WiFiの無効化（使わない場合）
echo "dtoverlay=disable-wifi" >>/mnt/boot/config.txt

# Bluetoothの無効化（使わない場合）
echo "dtoverlay=disable-bt" >>/mnt/boot/config.txt
```

必要に応じて以下のカーネルパラメーターを設定してください．

```text
# cmdline.txt
... coherent_pool=4M ...
```

マイクロSDカードをRaspberry Piに挿入し，電源をオン．有効化してある`ssh`を使ってログインします．

```shell
# 必要に応じてホストを追加
cat <<EOF >>~/.ssh/config
Host pi
  HostName raspberrypi.local
  User pi
EOF

# パスワード入力が面倒なので公開鍵をコピー
ssh-copy-id pi

# ログイン
ssh pi
```

`raspberrypi.local`に接続できない場合，何らかの理由でmDNSが機能していないことを意味します．

ログインしたら，安全のために以下を実行しておきます．

```shell
# パスワード変更
passwd

# rootをロック
sudo passwd -l root
```

SDカードの寿命が気になる人は，以下の設定に追加してtmpfsの設定も行っておきましょう．

```shell
# swapの無効化
sudo systemctl stop dphys-swapfile
sudo systemctl disable dphys-swapfile
sudo rm -f /var/swap
free -h

# ntpの有効化
echo 'NTP=ntp.nict.jp' | sudo tee -a /etc/systemd/timesyncd.conf >/dev/null
sudo systemctl restart systemd-timesyncd
systemctl status systemd-timesyncd | grep nict
```

ソフトウェアを更新し，再起動します．

```shell
sudo apt-get update
sudo apt-get dist-upgrade
sudo reboot
```

[Raspberry Pi 4で構築する録画マシン]: https://medium.com/chinachu/c98v-raspberry-pi-4-49c9692f5c09
[tsukumijima/px4_drv]: https://github.com/tsukumijima/px4_drv
[Armbian]: https://en.wikipedia.org/wiki/Armbian
