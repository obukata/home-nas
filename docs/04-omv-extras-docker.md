# 4. omv-extras 導入と Docker / compose プラグイン有効化

前: [3. HDD マウントとディレクトリ階層の設計](03-shared-folders-smb.md)

## 目的

OMV の拡張パッケージ集 `omv-extras` を導入し、Docker とその compose プラグインを有効化する。
以降の Nextcloud / Nginx Proxy Manager / WireGuard はこの上で動かす。

## 前提

- [03-shared-folders-smb.md](03-shared-folders-smb.md) までが完了している
- `/srv/appdata/`, `/srv/compose/`, `/srv/docker/` (SSD 側) と `<HDD_ROOT>/shares/`, `<HDD_ROOT>/appdata/nextcloud/data/` が作成済み

## 手順

### 4-1. omv-extras のインストール

SSH で Mac mini に入り、omv-extras 公式 wiki (<https://wiki.omv-extras.org/>) に掲載されている導入コマンドを実行する。OMV のバージョン (7/8) を自動検出して対応する deb を取得・インストールしてくれる。

```bash
sudo wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | sudo bash
```

もしくは root シェルで:

```bash
sudo -i
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
exit
```

スクリプトが行う処理:
1. OMV のバージョンを検出
2. omv-extras の GPG 署名鍵をインストール
3. 対応する `openmediavault-omvextrasorg_xxx_all8.deb` をダウンロード
4. `apt install` で導入

> **中身を先に確認したい場合**:
> ```bash
> wget -O /tmp/install https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install
> less /tmp/install       # 中身を読む
> sudo bash /tmp/install  # 納得してから実行
> ```

インストール後、Web UI を再読み込みすると `System → omv-extras` メニューが増える。

### 4-2. Docker プラグインのインストール

OMV 8 では `openmediavault-compose` プラグインを使う (旧 portainer ベースではなく、GUI から compose を直接扱う方式)。

Web UI: `System → Plugins` → プラグイン一覧から `openmediavault-compose` を検索しインストール。

インストール後、`Services → Compose` メニューが現れる。

### 4-3. Compose の初期設定

`Services → Compose → Settings`:

| 項目 | 値 |
|---|---|
| Shared folder (Compose files) | `/srv/compose/` を指す OMV の Shared Folder を作成して指定 (compose プラグインの内部用途。外部公開はされない) |
| Docker storage | `/srv/docker` (SSD 側、Docker data-root 用) |

> **Docker storage (data-root) の配置方針**
> - デフォルトの `/var/lib/docker` は OS パーティションに書かれるため、明示的にパスを決めておく
> - 今回は SSD 側 `/srv/docker/` に置く。イメージ層は頻繁に読み書きされるため SSD の方が快適
> - 容量が気になる場合は `docker system prune` で掃除可能

`Save → Apply` するとプラグインが `daemon.json` の `data-root` を書き換えて Docker を再起動する。

### 4-4. Compose ファイル配置の設計

`/srv/compose/` 以下をサービスごとに分ける:

```
/srv/compose/
├── nextcloud/
│   ├── docker-compose.yml
│   └── .env
├── npm/
│   ├── docker-compose.yml
│   └── .env
└── wireguard/
    ├── docker-compose.yml
    └── .env
```

各 `.env` は機微情報 (パスワード等) を入れるので、個別に `chmod 600` しておく (後ステップの Nextcloud / NPM / WireGuard 章で都度やる)。Git にコミットしない方針にする (将来リポジトリ運用する場合)。

`/srv/compose/` 自体の所有権は **`root:root` のまま** にする。OMV の Compose プラグインが Apply 時に Owner/Group を `root` に再適用するため、ここでユーザー所有に変えても上書きされる。SSH から編集したい時は `sudo nano /srv/compose/nextcloud/docker-compose.yml` のように `sudo` 経由で開けばよい。

### 4-5. 外部 Docker ネットワークの作成

Nextcloud と NPM を同じネットワークに置き、プロキシから内部名で通せるようにする。あらかじめ外部ネットワークとして作っておく。

```bash
sudo docker network create proxy-net
```

compose ファイル側では `external: true` でこの `proxy-net` を参照する。

## 確認

- [ ] `systemctl status docker` が `active (running)`
- [ ] `docker info | grep "Docker Root Dir"` の値が `/srv/docker` (または設定した値) になっている
- [ ] `docker network ls` に `proxy-net` がある
- [ ] Web UI で `Services → Compose` が利用可能

## トラブルシュート

- **`apt install` で omv-extras の依存エラー**: `apt update` を先に実行。署名鍵が古い場合は `wget https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/openmediavault-omvextrasorg.gpg -O /etc/apt/trusted.gpg.d/omv-extras.gpg`
- **Docker が起動しない**: `journalctl -u docker -n 100` でログを確認。`daemon.json` の JSON 構文エラーが多い
- **`docker network create` で既に存在と言われる**: 問題なし。スキップ

## 次へ

次: [5. Nextcloud コンテナの構築](05-nextcloud.md)
