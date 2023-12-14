---
title: "Rosetta for Linux を使って Arch Linux ARM で x86_64 バイナリを動かす"
emoji: "🖥"
type: "tech"
topics: ["mac", "rosetta2", "vm", "archlinux", "docker"]
published: true
published_at: 2023-12-14 19:30
publication_name: team_soda
---

:::message
＼スニダンを開発しているSODA inc.の [Advent Calendar](https://qiita.com/advent-calendar/2023/soda-inc) 14日目の記事です!!!／
:::

## はじめに

SODA ではローカル開発環境の構築に Docker を使っています。
MySQL や Redis など、イチから構築しようとすると面倒な開発環境をコード化してメンバー間で共有できるため、大変便利です。

しかし macOS で Docker を使おうとすると [Docker Desktop](https://docs.docker.com/desktop/) や [lima](https://lima-vm.io/) を使うことになります。
これらのツールは i/o パフォーマンスが遅かったり、macOS と Linux の差異に悩まされることが多く、macOS で Docker を使う開発の体験が良くないと感じていました[^orbstack]。

[^orbstack]: [OrbStack](https://orbstack.dev/) は筆者が未使用なため未知数です。

Windows では WSL2 で作った環境に引き籠もることでそこそこの開発体験を得られていたので、macOS でも Linux の仮想マシンを動かしてその中に引きこもって開発すれば快適なのでは？と思い至り、この1年の開発で実践してきました。

その中で Apple Silicon の macOS をホストマシンとし、ゲスト OS で x86_64 のバイナリが動く Arch Linux ARM の環境を構築できたので、その手順を備忘録として共有します。

## 準備

本家の Arch Linux であればインストールメディアの ISO が公開されているため、これをマウントするだけで簡単にインストールすることができます。

しかし Arch Linux ARM はインストールメディアが公開されていません。 そこで Ubuntu のインストールメディアを使って仮のシェル環境を用意し、そこから手動でインストールを行います。

### Ubuntu Server for ARM のインストールメディアの入手

Ubuntu for ARM のダウンロードページから Ubuntu 22.04 の ISO イメージを入手します。

@[card](https://ubuntu.com/download/server/arm)

### UTM のインストール

macOS で仮想マシンを動かすために UTM をインストールします。

@[card](https://getutm.app/)

@[card](https://github.com/utmapp/UTM)

今回は Homebrew でインストールします。

```sh
brew install utm
```

## UTM に Arch Linux ARM をインストールする

公式ドキュメントを参考に、仮想マシン上に Arch Linux ARM をインストールしていきます。

@[card](https://archlinuxarm.org/platforms/armv8/generic)

### 仮想マシンの作成

UTM で仮想マシンを作成します。

UTM 起動直後の画面で `新規仮想マシンを作成` を選択します。

![utm-home](/images/arch-linux-arm-with-rosetta-2/utm-home.png)

`仮想化` を選択します。

![utm-create-vm-start](/images/arch-linux-arm-with-rosetta-2/utm-create-vm-start.png)

`Linux` を選択します。

![utm-create-vm-os](/images/arch-linux-arm-with-rosetta-2/utm-create-vm-os.png)

以下のように設定し `続ける` を選択します。

- `Apple 仮想化を使用` にチェックを入れる
- `Rosetta を有効にする` にチェックを入れる
- `起動 ISO イメージ` に Ubuntu インストールメディアの ISO を指定する

![utm-create-vm-linux](/images/arch-linux-arm-with-rosetta-2/utm-create-vm-linux.png)

仮想マシンに割り当てるメモリサイズを設定します。今回は 8192MB を設定し、 `続ける` を選択します。

![utm-create-vm-hw](/images/arch-linux-arm-with-rosetta-2/utm-create-vm-hw.png)

仮想マシンに割り当てるストレージサイズを設定します。今回は 128GB を設定し、 `続ける` を選択します。

![utm-create-vm-storage](/images/arch-linux-arm-with-rosetta-2/utm-create-vm-storage.png)

共有ディレクトリは設定せず、デフォルトのまま `続ける` を選択します。

![utm-create-vm-shared-directory](/images/arch-linux-arm-with-rosetta-2/utm-create-vm-shared-directory.png)

確認画面が出ます。
適当な名前をつけてから `仮想マシン設定を開く` にチェックを入れ `保存` を選択します。

![utm-create-vm-summary](/images/arch-linux-arm-with-rosetta-2/utm-create-vm-summary.png)

自動で仮想マシン設定が開きます。デフォルトだと解像度が高すぎて操作しづらいため、解像度を適当に調整して `保存` を選択します。

![utm-edit-vm-display](/images/arch-linux-arm-with-rosetta-2/utm-edit-vm-display.png)

作成した仮想マシンをサイドバーから選択し ▶️ ボタンを押して仮想マシンを起動します。

![utm-home-after-vm-creation](/images/arch-linux-arm-with-rosetta-2/utm-home-after-vm-creation.png)

別ウィンドウが開いて GRUB の画面が表示されるはずなので、そのまま Ubuntu のインストーラーが起動して言語選択の表示になるまで進めます。

![utm-display-grub](/images/arch-linux-arm-with-rosetta-2/utm-display-grub.png)

言語選択の表示になったら `F2` キーを押し、シェルを起動します。

![utm-display-ubuntu-language](/images/arch-linux-arm-with-rosetta-2/utm-display-ubuntu-language.png)

![utm-display-ubuntu-shell](/images/arch-linux-arm-with-rosetta-2/utm-display-ubuntu-shell.png)

:::details SSH Server の設定

このままインストール作業を進めても良いのですが、UTM の仮想ディスプレイではコピーやペーストができず不便なので、SSH 接続できるように設定します。

`openssh-server` をインストールし `sshd.service` を起動します。

```sh
apt update
apt install openssh-server
systemctl start sshd.service
```

`root` ユーザーの `authorized_keys` に公開鍵を登録します。

```sh
mkdir -p ~/.ssh
curl https://github.com/koyashiro.keys > ~/.ssh/authorized_keys
```

仮想マシンの IP アドレスを確認します。

```sh
ip a | grep 'inet '
```

```shell-session
root@ubuntu-server:~# ip a | grep 'inet '
    inet 127.0.0.1/8 scope host lo
    inet 192.168.65.11/24 metric 100 brd 192.168.65.255 scope global dynamic enp0s1
```

今回は `192.168.65.11` でした。

`192.168.65.11` に `root` ユーザーとして SSH します。

```sh
ssh root@192.168.65.11
```

無事に接続できたら Arch Linux ARM をインストール作業に入ります。
:::

### Arch Linux ARM のインストール

仮想ストレージのデバイス名を確認します。

```sh
lsblk
```

```shell-session
root@ubuntu-server:~# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 132.7M  1 loop /rofs
loop1    7:1    0 308.3M  1 loop
loop2    7:2    0   467M  1 loop
loop3    7:3    0 123.4M  1 loop
loop4    7:4    0 132.7M  1 loop /media/minimal
loop5    7:5    0 308.3M  1 loop /media/full
loop6    7:6    0  59.2M  1 loop /snap/core20/1977
loop7    7:7    0  18.7M  1 loop /snap/subiquity/5010
loop8    7:8    0  46.4M  1 loop /snap/snapd/19459
loop9    7:9    0 109.6M  1 loop /snap/lxd/24326
loop10   7:10   0  68.5M  1 loop /snap/core22/861
sda      8:0    0   1.9G  1 disk
├─sda1   8:1    0   1.9G  1 part /cdrom
└─sda2   8:2    0   5.8M  1 part
vda    252:0    0    64G  0 disk
```

`sda` は `/cdrom` にマウントされているため、これが Ubuntu のインストールメディアであることがわかります。

消去法で `vda` が仮想ストレージであることがわかったので、ここにファイルシステムを作っていきます。

```sh
fdisk /dev/vda
```

今回は以下のようなパーティションレイアウトを採用をします。

```plaintext
Device       Start       End   Sectors   Size Type
/dev/vda1     2048    616447    614400   300M EFI System
/dev/vda2   616448   2713599   2097152     1G Linux swap
/dev/vda3  2713600 268435422 265721823 126.7G Linux root (ARM-64)
```

:::message
`fdisk` の使い方は割愛します。
Arch Linux Wiki の `fdisk` のページが詳しいです。以下をご参照ください。
@[card](https://wiki.archlinux.org/title/fdisk)
:::

ファイルシステムを作成します。

```sh
mkfs.fat -F 32 /dev/vda1
mkswap /dev/vda2
mkfs.ext4 /dev/vda3
```

それぞれマウントします。

```sh
mount /dev/vda3 /mnt
mkdir -p /mnt/boot
mount /dev/vda1 /mnt/boot
```

最終的に以下のような構成になりました。

```shell-session
[koyashiro@alarm ~]$ lsblk -f | grep vda
vda
|-vda1 vfat    FAT32                                            6AF7-BC96                             154.3M    48% /boot
|-vda2 swap    1                                                95d6e400-8c13-44ee-97dc-3e51ee2f319a                [SWAP]
`-vda3 ext4    1.0                                              a5f2ffc6-2019-4875-9ce6-eb0968134bda  115.8G     2% /
```

Arch Linux ARM のインストールに必要なパッケージをインストールします。

```sh
apt install arch-install-scripts libarchive-tools
```

:::details インストールするパッケージについて

#### `arch-install-scripts`

後述する `arch-chroot` や `genfstab` が含まれています。

#### `libarchive-tools`

後述する `bsdtar` が含まれています。
:::

最新の Arch Linux ARM のイメージをダウンロードします。

```sh
curl -O https://jp.mirror.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
```

ダウンロードした Arch Linux ARM のイメージを `bsdtar` で `/mnt` に展開します。

```sh
bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
```

`/mnt` に最低限の環境ができたので、 `genfstab` で現在のマウント設定を `/mnt/etc/fstab` に書き込んでおきます。

```sh
genfstab -U /mnt > /mnt/etc/fstab
```

`arch-chroot` で `/mnt` に展開した Arch Linux ARM に chroot します。

```sh
arch-chroot /mnt
```

### Arch Linux ARM のセットアップ

ここからは chroot 環境で Arch Linux ARM の初期設定を行います。
本家の Arch Linux とほとんど同じです。

#### pacman の初期設定

`pacman-key` で `pacman` の鍵を初期化します。

```sh
pacman-key --init
pacman-key --populate archlinuxarm
```

`pacman` が使えるようになったので、インストールされているパッケージをすべて最新化しておきます。

```sh
pacman -Syu
```

#### ブートローダーの設定

Arch Linux のドキュメントを参考に `systemd-boot` のセットアップを行います。

@[card](https://wiki.archlinux.org/title/Systemd-boot)

ルートパーティションの UUID を調べます。

```sh
lsblk -f | grep vda3 | awk '{print $4'}
```

```shell-session
[root@ubuntu-server /]# lsblk -f | grep vda3 | awk '{print $4'}
a5f2ffc6-2019-4875-9ce6-eb0968134bda
```

今回は `a5f2ffc6-2019-4875-9ce6-eb0968134bda` でした。

`/boot/loader/entries/arch.conf` に Arch Linux ARM のブート設定を書き込みます。

```sh
cat > /boot/loader/entries/arch.conf <<EOF
title Arch Linux ARM
linux /Image
> initrd /initramfs-linux.img
> initrd /initramfs-linux-fallback.img
> options root="UUID=a5f2ffc6-2019-4875-9ce6-eb0968134bda" rw
> EOF
```

最低限のセットアップが完了したので `exit` で chroot 環境から抜けます。

```sh
exit
```

Ubuntu のインストーラーでの作業は終わったので `reboot` して Arch Linux ARM を起動します。

```sh
reboot
```

UTM の仮想ディスプレイで Arch Linux ARM のログイン画面が出れば成功です。

![utm-display-alarm-login](/images/arch-linux-arm-with-rosetta-2/utm-display-alarm-login.png)

### Arch Linux ARM へのログイン

#### `root` ユーザーでの作業

`root` ユーザーでログインします。デフォルトのパスワードは `root` です。

初期状態で `alarm` という一般ユーザーが作成されていますが、削除して一般ユーザーを作り直します。

```sh
userdel alarm
useradd -m koyashiro
passwd koyashiro
```

作成した一般ユーザーで `sudo` コマンドが使えるように設定します。

```sh
pacman -S --noconfirm sudo
usermod -aG wheel koyashiro
```

`visudo` で以下のコメントアウトを外します。

```sh
visudo
```

```diff
-# %wheel ALL=(ALL:ALL) ALL
+%wheel ALL=(ALL:ALL) ALL
```

仮想ディスプレイでの作業はつらいため `openssh` をインストールして、SSH サーバーを有効にしておきます。

```sh
pacman -S --noconfirm openssh
systemctl enable --now sshd.service
```

```sh
mkdir -p -m 700 /home/koyashiro/.ssh
curl https://github.com/koyashiro.keys > /home/koyashiro/.ssh/authorized_keys
chown -R "$(id -u koyashiro):$(id -g koyashiro)" /home/koyashiro/.ssh
```

IP アドレスも確認しておきます。

```shell-session
[koyashiro@alarm ~]$ ip a | grep 'inet '
    inet 127.0.0.1/8 scope host lo
    inet 192.168.65.11/24 metric 1024 brd 192.168.65.255 scope global dynamic enp0s1
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```

#### 一般ユーザーでログイン

SSH でログインします。

```sh
ssh 192.168.65.11
```

`sudo` できるか確認します。

```sh
sudo echo
```

問題なければあとはお好みで Arch Linux の初期設定を行います。

@[card](https://wiki.archlinux.org/title/installation_guide)

### Rosetta 2 for Linux

Arch Linux ARM をセットアップしただけなので、まだ aarch64 ネイティブのバイナリしか動きません。

試しに x86_64(amd64) の Go のコンパイラを実行しようとすると

```shell-session
[koyashiro@alarm ~]$ curl -L https://go.dev/dl/go1.21.5.linux-amd64.tar.gz | tar xz
[koyashiro@alarm ~]$ ./go/bin/go version
-bash: ./go/bin/go: cannot execute binary file: Exec format error
```

`Exec format error` となり、実行に失敗します。

UTM の Rosetta のページを参考に binfmt の設定をします(一部アレンジしています)。

@[card](https://docs.getutm.app/advanced/rosetta/)

```sh
sudo mkdir -p /media/rosetta
sudo mount -t virtiofs rosetta /media/rosetta
echo 'rosetta /media/rosetta virtiofs ro,nofail 0 0' | sudo tee -a /etc/fstab
echo ":rosetta:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00:\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/media/rosetta/rosetta:F" | sudo tee /lib/binfmt.d/rosetta.conf
sudo systemctl enable --now systemd-binfmt.service
```

上記を済ませてからもう一度実行してみると

```shell-session
[koyashiro@alarm ~]$ ./go/bin/go version
go version go1.21.5 linux/amd64
```

無事に Arch Linux ARM で x86_64 の Go コンパイラが実行できました。

## Docker で x86_64 のコンテナを動かす

Docker で x86_64 (linux/amd64) のコンテナが動かせるかも確認します。

`pacman` で `docker` をインストールし、サービスを開始します。

```sh
sudo pacman -S --noconfirm docker
sudo systemctl enable --now docker.service
```

`--platform` フラグで `linux/amd64` を指定して `hello-world` イメージを動かしてみます。

```shell-session
[koyashiro@alarm ~]$ sudo docker run --rm --platform linux/amd64 hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
719385e32844: Pull complete
Digest: sha256:3155e04f30ad5e4629fac67d6789f8809d74fea22d4e9a82f757d28cee79e0c5
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

無事に実行できました。

## おわりに

x86_64 のバイナリが動く Arch Linux ARM のセットアップ手順は以上です。

筆者はこの環境に SSH して Neovim で Go や TypeScript を書いていますが、使い慣れた Arch Linux 環境で開発できるためかなり体験がよいです。

SSH さえできれば Visual Studio Code の Remote SSH 拡張機能も使えるので、Vim/Neovim ユーザー以外にもおすすめできます。

Windows の WSL2 ほどカジュアルに使えるものではありませんが、macOS と Linux の差異に悩まされている方は是非ともお試しください。
