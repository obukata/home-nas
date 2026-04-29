# 3. HDD マウントとディレクトリ階層の設計

前: [2. OMV 初期設定](02-initial-config.md)

## 目的

1TB HDD を OMV でマウントしてデータ領域として利用可能にし、SSD/HDD それぞれの役割に沿ったディレクトリ階層を作る。後続ステップ (Nextcloud / バックアップ) で使うパスをここで先に確定させる。

> **SMB は今回使わない**: 本構成ではファイル共有は Nextcloud に集約する。SMB (Samba) を使うと LAN からの直接アクセスが可能になる一方、外出先アクセスには VPN 必須・権限とバックアップが別管理になる等、2 人運用では手間の方が大きい。Nextcloud 単独なら同じフォルダが家でも外でも同じように見える。
>
> 将来「家で大容量アーカイブを Finder からドラッグ&ドロップしたい」等の要求が出たら、この章の旧版を参考に SMB を後付けで有効化すればよい。

## 前提

- [02-initial-config.md](02-initial-config.md) までが完了している
- SSH で `<ADMIN_USER>@<OMV_IP>` にログインできる
- 内蔵 HDD (1TB) はまだマウントされていない (OMV インストール時は SSD だけを選択した)

## 手順

### 3-1. HDD のフォーマットとマウント

Web UI: `Storage → Disks` で内蔵 HDD (1TB, `APPLE HDD HTS541` 等) が認識されていることを確認。

元 Fusion Drive の残骸 (Apple の APFS/HFS+ パーティション) が残っていると `File Systems → Create` の Device ドロップダウンに出てこない。その場合は先に `Storage → Disks` で HDD を選択して **Wipe** (Quick で十分) を実行する。SSD (121GB) は絶対に Wipe しない。

次に `Storage → File Systems → Create`:

| 項目 | 値 |
|---|---|
| Device | 1TB HDD (`/dev/sda` 等) |
| Type | `ext4` (無難。XFS でも可) |
| Label | `datahdd` (任意) |

`Create` → 完了まで数分〜 10 分ほどかかる (1TB ext4 の inode 初期化)。完了したら `Mount` で同じ HDD を選んでマウント。

マウント後、`Storage → File Systems` で表示されるマウントポイントをメモする。これが **`<HDD_ROOT>`**。通常以下のような形:

```
/srv/dev-disk-by-uuid-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

以降の手順書はこのパスを `<HDD_ROOT>` として参照する。

SSD 側は OS 直下の `/srv` をそのまま使うので、`<SSD_ROOT>` = `/srv`。

### 3-2. ディレクトリ階層の設計

SSD と HDD を役割分担させる。

**SSD (`<SSD_ROOT>` = `/srv`)** — 高速だが容量小。OS と同居:

```
/srv/
├── appdata/              ... Docker コンテナ設定・DB (小)
│   ├── nextcloud/html/   ... Nextcloud の PHP 本体
│   ├── mariadb/          ... Nextcloud の DB
│   ├── redis/
│   ├── npm/              ... Nginx Proxy Manager 設定
│   └── wireguard/        ... WG 鍵・ピア conf
├── compose/              ... docker-compose.yml 置き場
│   ├── nextcloud/
│   ├── npm/
│   └── wireguard/
└── docker/               ... Docker data-root (イメージ層)
```

**HDD (`<HDD_ROOT>`)** — 大容量。Nextcloud のユーザーデータとバックアップ:

```
<HDD_ROOT>/
├── appdata/
│   └── nextcloud/
│       └── data/         ... Nextcloud のユーザーデータ (全ユーザー分)
└── shares/
    └── backups/          ... バックアップ出力先 (snapshots)
```

SSH で一気に作る:

```bash
# SSD 側
sudo mkdir -p /srv/appdata/{nextcloud/html,mariadb,redis,npm,wireguard}
sudo mkdir -p /srv/compose/{nextcloud,npm,wireguard}
sudo mkdir -p /srv/docker

# HDD 側 (<HDD_ROOT> は実際のマウントパスに置換)
sudo mkdir -p <HDD_ROOT>/appdata/nextcloud/data
sudo mkdir -p <HDD_ROOT>/shares/backups
```

所有権は後続の章で適宜変更する (Nextcloud コンテナが `www-data` で書くため) が、この段階では root 所有のままで OK。

### 3-3. OMV で Shared Folder として登録

OMV の「Shared Folder」は OMV 内部でパスに名前を付けるだけの登録。SMB で公開するわけではない (公開は Services → SMB で有効にして初めて発生)。Nextcloud の Docker ボリュームとしてもここで登録したパスを流用できるので、先に登録しておく。

`Storage → Shared Folders → Create`:

| Name | Device (File System) | Relative Path | 用途 |
|---|---|---|---|
| nextcloud-data | HDD (`<HDD_ROOT>`) | `appdata/nextcloud/data/` | Nextcloud のユーザーデータ格納先 |
| backups | HDD (`<HDD_ROOT>`) | `shares/backups/` | バックアップ出力先 |

Permissions は両方とも `Administrator: read/write, Users: no access` で OK (Nextcloud コンテナは root 権限で必要な chown を後で行う)。

`appdata` / `compose` / `docker` (SSD 側) は Docker コンテナから直接使うので OMV の Shared Folder としては登録不要。

### 3-4. SMB/CIFS サービスは有効化しない

`Services → SMB/CIFS → Settings` は **Enabled を OFF のまま** にする。今回は LAN からのファイルアクセスはすべて Nextcloud (Web / Desktop Client / モバイルアプリ) 経由で行う。

将来 SMB が欲しくなった場合:
1. この Settings を Enabled に
2. 公開したい Shared Folder を `Shares → Create` で追加
3. ユーザーに SMB 用パスワードを設定

## 確認

- [ ] `<HDD_ROOT>` が OMV の File Systems で Mounted 表示
- [ ] `/srv/appdata/`, `/srv/compose/`, `/srv/docker/` が存在する
- [ ] `<HDD_ROOT>/appdata/nextcloud/data/` が存在する
- [ ] `<HDD_ROOT>/shares/backups/` が存在する
- [ ] OMV の `Storage → Shared Folders` に `nextcloud-data` と `backups` が登録されている
- [ ] `df -h` で SSD (/) と HDD (`<HDD_ROOT>`) の両方が表示される

## トラブルシュート

- **HDD がマウントできない / File System Create のドロップダウンに HDD が出ない**: 元 Fusion Drive の残骸で古い APFS パーティションが残っている。`Storage → Disks` から HDD を選んで **Wipe (Quick)** を実行してから `Create` をやり直す
- **ext4 の Create がなかなか終わらない**: 1TB HDD の inode 初期化に数分〜 10 分かかるのは正常。`ps aux | grep mkfs` でプロセスが動いていれば進行中

## 次へ

次: [4. omv-extras 導入と Docker / compose プラグイン有効化](04-omv-extras-docker.md)
