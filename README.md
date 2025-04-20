# Kali Live with Encryption Storage
## はじめに - Live環境の有用性
Kali Live USB/DVDは、プライバシーやセキュリティを重視する上で非常に有効な方法です。RAMメモリ上にOSを展開し動作する事で、シャットダウン後すべてのデータは消去されます。開いたWebサイトやOSの操作履歴といった痕跡を一切残しません。またこのような特性上、しばし公共のPCや信頼できないコンピュータの操作以外にもアクセス権を持たないコンピュータの操縦を可能とする他、OSINTや即応的な攻撃にも転用できる事を踏まえてもOPSECやOffensiv Securityの観点から見て望ましいと言えます。

しかしこのような揮発性の高い特性から永続的な記録を行う事ができず、Live領域以上のデータを持ち歩く事もできません。そこでTailsから着想を得る形で、データ侵害に強い暗号化された永続化領域を備えたKali Live USBを作成する方法とLive環境を初期化して日本語入力を可能にするシェルスクリプトを本リポジトリで提供します。また暗号化済み領域を復号して認識させることにより、普段使用しているコンピュータとUSBの永続化領域の間でファイルのやり取りが行なえます。Live環境は備え付けられた暗号化領域への読み書きをスタンドアロンで行う為の存在と言えます。そこにたまたまMetasplotなど侵入テストツールが入っているだけとも言えます。

## 用意するもの
- KaliLinux Live版 iso(執筆時点の最新版は[2024.3](https://cdimage.kali.org/kali-2024.3/))
- 7.0GB以上のUSBメモリー
- 何らかのLinux環境(ROM焼きとcryptsetup実行に必要)

## Live USBと暗号化済みストレージの作成
まず7.0GB以上のUSBをフォーマットしてパーティションを切ります。今回は7.2GBのUSBを例として解説するので2.2GBを記憶領域に割り当てとりあえずフォーマットします。すると5GBの領域が自然に生まれるのでそちらも同じくフォーマットします。KaliのLive版isoイメージのファイルサイズは4.3GBなのでLive領域は5GBで十分です。

### フォーマットと領域作成
作業が整い次第、次の手順で2.2GB領域に暗号化済み領域を作成します。この作業はLive USBの紛失や盗難などによって保存領域のデータが侵害される事を防ぐ為の処置です。

USBのデバイス名は周辺機器や環境によって`/dev/sdb`であったり`/dev/sdc`など変動するので、ここでは`/dev/sdX`と呼称します(lsblk -fで確認できます)。

```bash
cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --hash sha512 --iter-time 2000 --key-size 512 --pbkdf argon2id --use-urandom --verify-passphrase /dev/sdX2
```

このコマンドを実行すると次のような出力結果を得ると思いますが、以下の手順に従ってください。

```bash
WARNING!
========
This will overwrite data on /dev/sdX2 irrevocably.

Are you sure? (Type uppercase yes): YES #筆者注:大文字で入力
Enter passphrase for /dev/sdX2: #筆者注:パスワード設定入力しても表示されない
Verify passphrase: #パスワードの確認。もう一度入力
```

### 暗号化領域のマッピングとマウント

```bash
## 領域復号とマッピング
cryptsetup luksOpen /dev/sdX2 kali-encryption-storage

## マウント
mount /dev/mapper/kali-encryption-storage /mnt/kali-storage

## 領域のフォーマット(初回のみ)
mkfs.ext4 /dev/mapper/kali-encryption-storage
```

アンマウントおよびマッピングの終了は以下のとおりです。

```bash
## アンマウント
umount /mnt/kali-storage

## マッピング終了
cryprsetup luksClose kali-encryption-storage
```

### データ保存領域のとKali Live USB本体のセットアップ
`/mnt/kali-storage`に移りターミナルから以下のコマンドを実行してください。まっさらなKali Live環境を初期化する`1n17_ka11.sh`がクローンされます。

```bash
git clone https://github.com/Extr3m4R4710/create-kali-live-usb-with-encryption-storage live_init
```



その後5GB領域(/dev/sdX1)へ執筆時点で最新のkali-linux-2024.3-live-amd64.isoを`cpコマンドでROM焼き`してください。`USB全体が上書きされてしまう為、ddやその他イメージライターは絶対に使わないのでください。`
```bash
cp -v /path/to/kali-linux-2024.3-live-amd64.iso /dev/sdX1
```

Live領域作成後、図のようになっていれば成功です。この図では一旦Live USBの記憶領域を復号化してマウントしています。
![](./img/kali-live-disk-part.png)

## 作成したLive USBの使い方
BIOSかUEFIをUSBブート可能な状態にセットアップする必要があります。特にここでは解説しませんが、必要な場合`Live USB ブート 方法`で検索してください。すべてそこにあります。

ユーザー名、パスワード共に`kali`でログインできます。

デスクトップに表示された暗号化済み領域(ここでは2.3GB Encrypted)をクリックしてください。これは先程作成した暗号領域です。環境によっていくつか領域が表示されますので、作成した暗号領域のサイズを覚えておきましょう。パスフレーズ認証が始まるので、設定したパスフレーズを入力すると自動的に領域がマウントされます。そこからファイルマネージャーの右クリックでもいいのでターミナルを立ち上げ、復号化した領域内の`live_init`以下の`1n17_ka11.sh`を実行しセットアップします。初期化にはネットワークが必要です。事前にWiFi等に接続している必要があります。`必ずroot権限で実行してください`。

```bash
sudo bash ./live_init/1n17_ka11.sh
```

初期化シェルスクリプトを実行する事で`ノーログVPNやtor`の他`日本語入力IME、gpg、keepassxc(パスワードマネージャー)、sn0int、proxychains-ng`が自動でインストールされます。その他、個人プロジェクトで開発している`script_vox`ツールキットが自動でクローンされます。

本Live環境はRAM上で動作します。永続化されたUSBと一体のストレージを除き、コンピュータを使用した痕跡をセッション終了後に破棄し、秘匿性を高めます。しかし通信データは保護されません。通信の保護・秘匿が必要な場合は初期化と同時にインストールされた`Riseup-VPN`を使用しRiseupチームが運営するネットワークを経由するか、`script_vox`に収録された`NetSpectre`で全ての通信でTorを経由させる必要があります。またproxychainsでプログラム単位でtorを使用することもできます。

初期化したLive環境は再度ログインすることで全て利用可能となります。日本語入力の利用には最低1回のログアウトが必要です。
![](./img/kali-live-desktop-screenshot.png)

## 余談
今回はKaliLinuxを使っていますが、`1n17_ka11.sh`は主に`curl`、`apt`、`git`で構成されています。パッケージ名の整合性さえ確保できればParrotOSやBackBoxでも動作します。そのような意味でLive領域はOSを問いません。例えばLive環境にFedoraを使う場合は一連のaptコマンドをdnfに置き換えてそのパッケージ名を適切に変更し、初期化スクリプトに出現するLiveユーザー名(/home/以下に記載)をkaliからliveuserに変更するだけです。Arch系の場合も同様にaptをpacmanに変更し、相当するパッケージ名に変更し、Liveユーザ名をkaliから規定のユーザー名に変更します。
