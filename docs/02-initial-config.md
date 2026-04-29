# 2. OMV 初期設定

前: [1. OMV インストーラ USB 作成と Mac mini へのインストール](01-install-omv.md)

## 目的

OMV 初期状態から、本番運用に耐える最低限の安全性・接続性を整える。具体的には管理者パスワード変更、固定 IP、OS アップデート、管理ユーザー作成、SSH 鍵認証化まで。

## 前提

- [01-install-omv.md](01-install-omv.md) が完了し、OMV にブラウザで到達できる
- ルータの管理画面にアクセスできる (DHCP 予約を行うため)

## 手順

### 2-1. 初回 Web UI ログインと管理者パスワード変更

ブラウザで `http://<DHCP_IP>/` を開き、初期認証でログイン。

| 項目 | 値 |
|---|---|
| ユーザー名 | `admin` |
| パスワード | `openmediavault` |

ログイン後すぐに右上 (または `System → General Settings → Web Administrator Password`) で Web 管理者パスワードを強固なものに変更する。初期パスワードは公開情報なので**必ず最初に変更する**。

### 2-2. 固定 IP の設定

OMV 側で static を組むよりも、**ルータの DHCP 予約で MAC アドレスに IP を固定する方式**を推奨。理由:
- Mac mini 側の /etc/network/interfaces を素の DHCP のままにできる
- ルータ引っ越し時にルータ側だけ見直せば済む

手順:

1. OMV Web UI の `System → Network → Interfaces` で現在の MAC アドレスと IP をメモ
2. ルータ管理画面で DHCP 予約に MAC と希望 IP (`<OMV_IP>`) を登録
3. Mac mini を再起動 (`reboot`) して IP が希望値になっていることを確認

OMV 側で static を強制したい場合:
- `System → Network → Interfaces` で該当 IF を編集し、IPv4 を Static にして IP/Mask/Gateway/DNS を入力
- Save → Apply

### 2-3. タイムゾーンと日時

`System → Date & Time`:

- Time zone: `Asia/Tokyo`
- NTP を有効 (デフォルト `pool.ntp.org` でよい)

### 2-4. OS アップデート

Web UI で: `System → Update Management → Updates` で `Check` → 一覧が出たら `Install all`。

SSH でやる場合 (初期 root ログインで):

```bash
apt update
apt full-upgrade -y
reboot
```

### 2-5. 管理ユーザーの作成

root 直接運用は避けたいので、sudo 可能な管理ユーザーを作る。

Web UI で: `Users → Users → Create`。
- Name: `<ADMIN_USER>`
- Shell: `/usr/bin/bash` (Debian 13 の usrmerge で `/bin/bash` は `/usr/bin/bash` への symlink。UI のドロップダウンに `/usr/bin/bash` しか出ないのが正常)
- Groups: `_ssh`, `sudo` を付ける
  - `_ssh` は OMV 8 / 新しい Debian で SSH ログイン許可グループとして使われている (`/etc/ssh/sshd_config` に `AllowGroups root _ssh` が書かれている)。名前の似た `ssh` (アンダースコアなし) は **別物**。間違って `ssh` グループを自作・付与しても SSH は通らない
  - `users` は一般ユーザーのプライマリグループとして自動付与される。放置で OK

> **OMV 8 Web UI での注意**: `Users → Users → Create` ではホームディレクトリが自動作成されないケースがある (`/etc/passwd` には `/home/<ADMIN_USER>` と記載されるが、実ディレクトリは未作成)。作成後に root SSH or コンソールで確認:
> ```bash
> ls -la /home/<ADMIN_USER>
> # No such file or directory なら手動で作る
> sudo mkdir -p /home/<ADMIN_USER>
> sudo cp -a /etc/skel/. /home/<ADMIN_USER>/
> sudo chown -R <ADMIN_USER>:users /home/<ADMIN_USER>
> sudo chmod 700 /home/<ADMIN_USER>
> ```
> これをやらないと後の `ssh-copy-id` が `Could not chdir to home directory` で失敗する

SSH でやる場合:

```bash
adduser <ADMIN_USER>
usermod -aG sudo,_ssh <ADMIN_USER>
```

### 2-6. SSH 有効化と鍵認証

Web UI: `Services → SSH` で `Enabled` にチェック、`Permit root login` はオフ (最終的に)、`Password authentication` も鍵設定後にオフ。

まず鍵を配置する。作業用 Mac 側で鍵がなければ生成:

```bash
ssh-keygen -t ed25519 -C "homenas admin"
```

公開鍵を Mac mini に転送:

```bash
ssh-copy-id <ADMIN_USER>@<OMV_IP>
```

鍵で入れることを確認:

```bash
ssh <ADMIN_USER>@<OMV_IP>
```

入れたら Web UI で:
- `Services → SSH → Password authentication` をオフ
- `Permit root login` をオフ
- Save → Apply

(念のため SSH を開いたまま別セッションで接続テストしてから切る)

### 2-7. ファイアウォール (任意)

初期の OMV は原則、アクセス元を LAN に限定する前提で、OS ファイアウォールは最小限。以降 WireGuard を導入したら、外部は UDP 51820 のみに絞る。ここではまだ何も設定しない。

## 確認

- [ ] Web UI に新しい管理者パスワードでログインできる
- [ ] `ping <OMV_IP>` が通り、再起動後も同じ IP
- [ ] `date` で `Asia/Tokyo` の現在時刻
- [ ] `apt list --upgradable` が空 (または security のみ)
- [ ] `ssh <ADMIN_USER>@<OMV_IP>` が鍵のみで通る
- [ ] `ssh root@<OMV_IP>` は拒否される

## トラブルシュート

- **Web UI に入れない**: 再起動直後はサービス起動に 30 秒ほどかかる。それでも `502 Bad Gateway` なら `systemctl status openmediavault-engined`
- **sudo で `<ADMIN_USER> is not in the sudoers file`**: `usermod -aG sudo <ADMIN_USER>` 後、再ログインが必要
- **鍵認証が通らない**: `~/.ssh/authorized_keys` のパーミッションが 600 でないとはじかれる。`chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys`
- **パスワードは合っているのに `Permission denied` (publickey,password)**: `AllowGroups` と実際のグループ名の不一致が原因のことが多い。`grep -iE "^(AllowGroups|AllowUsers)" /etc/ssh/sshd_config` で許可グループを確認し、`id <ADMIN_USER>` でそのグループに属しているか確認。OMV 8 は `_ssh` (アンダースコア付き)。無印の `ssh` グループに入れても弾かれる

## 次へ

次: [3. HDD マウントとディレクトリ階層の設計](03-shared-folders-smb.md)
