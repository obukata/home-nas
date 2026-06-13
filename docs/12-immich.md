# 12. Immich の導入 (写真・動画管理)

前: [11. n8n](11-n8n.md)

## 目的

Google Photos ライクなセルフホスト写真・動画管理システム Immich を導入する。
スマートフォンからの自動バックアップ、顔認識、AI 検索、アルバム管理が使える。

アクセスは LAN 内のみ。外出先からは WireGuard VPN 経由でアクセスする。

## 構成

| コンテナ | 役割 |
|---|---|
| `immich_server` | API サーバ + Web UI (port 2283) |
| `immich_machine_learning` | 顔認識・AI 検索用 ML モデル実行 |
| `immich_redis` | キャッシュ (Nextcloud の Redis とは別) |
| `immich_postgres` | DB (pgvecto-rs、Nextcloud の MariaDB とは別) |

## ストレージ配置

| 用途 | パス |
|---|---|
| compose ファイル | `/srv/compose/immich/` |
| PostgreSQL データ (SSD) | `/srv/appdata/immich-db/` |
| 写真・動画アップロード先 (HDD) | `<HDD_ROOT>/appdata/immich/` |

## 手順

### 12-1. ディレクトリ作成

```bash
sudo -i
mkdir -p /srv/compose/immich
mkdir -p /srv/appdata/immich-db
mkdir -p /srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5/appdata/immich
exit
```

### 12-2. .env の作成

```bash
sudo nano /srv/compose/immich/.env
```

以下を貼り付け、`<DB_PASSWORD>` を `openssl rand -base64 24` で生成した乱数に差し替える:

```dotenv
# Immich バージョン (latest に追従。特定バージョンに固定したい場合は "v1.x.x" に)
IMMICH_VERSION=release

# アップロードデータの保存先 (HDD)
UPLOAD_LOCATION=/srv/dev-disk-by-uuid-3a9a7624-afbb-4993-a1dd-38e2bd1546f5/appdata/immich

# PostgreSQL データの保存先 (SSD)
DB_DATA_LOCATION=/srv/appdata/immich-db

# DB 接続情報 (A-Za-z0-9 のみ使用すること)
DB_PASSWORD=<強固な乱数>
DB_USERNAME=immich
DB_DATABASE_NAME=immich

# タイムゾーン
TZ=Asia/Tokyo
```

パーミッションを 600 に:

```bash
sudo chmod 600 /srv/compose/immich/.env
```

### 12-3. docker-compose.yml の作成

```bash
sudo nano /srv/compose/immich/docker-compose.yml
```

以下を貼り付け:

```yaml
name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    restart: always
    env_file:
      - .env
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "2283:2283"
    depends_on:
      - redis
      - database
    networks:
      - internal
      - proxy-net
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    restart: always
    env_file:
      - .env
    volumes:
      - model-cache:/cache
    networks:
      - internal
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: redis:6.2-alpine
    restart: always
    networks:
      - internal
    healthcheck:
      test: redis-cli ping || exit 1

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14
    restart: always
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    networks:
      - internal

volumes:
  model-cache:

networks:
  internal:
    driver: bridge
  proxy-net:
    external: true
```

### 12-4. 起動

#### (推奨) OMV UI から

OMV 管理画面 → **Services → Compose → Files** → `immich` → **Up**

#### SSH から直接

```bash
sudo -i
cd /srv/compose/immich
docker compose up -d
docker compose logs -f immich-server
# "Immich Server is listening" が出たら完了 → Ctrl+C
exit
```

初回は ML モデルのダウンロードがあるため数分かかる。

### 12-5. AGH に DNS エントリを追加

AdGuard Home の管理画面（`http://<OMV_IP>:3000`）:

1. **Filters → DNS rewrites** → **Add DNS rewrite**
2. `immich.home.lan` → `<OMV_IP>` を追加

### 12-6. NPM にプロキシホストを追加

Nginx Proxy Manager の管理画面（`http://<OMV_IP>:81`）:

1. **Proxy Hosts** → **Add Proxy Host**
2. 設定:

| 項目 | 値 |
|---|---|
| Domain Names | `immich.home.lan` |
| Scheme | `http` |
| Forward Hostname / IP | `immich_server` |
| Forward Port | `2283` |
| Websockets Support | ON |

3. SSL タブ → 既存の自己署名証明書を選択（または新規作成）

### 12-7. 初回セットアップ

ブラウザで `https://immich.home.lan/` を開く:

1. 管理者アカウントを作成（メールアドレス + パスワード）
2. 夫婦分の追加アカウントを作成: Administration → Users → Create user
3. スマートフォンアプリ (iOS/Android) をインストール → サーバ URL に `https://immich.home.lan` を入力

## バックアップへの組み込み

`/usr/local/sbin/homenas-backup.sh` に以下を追加（`docs/09-backup.md` も更新）:

```bash
# Immich DB ダンプ
docker exec immich_postgres pg_dump -U immich immich > "${BACKUP_DATE_DIR}/immich-db.sql"

# Immich アップロードデータは HDD 内で rsync 不要 (バックアップ宛先が同一 HDD のため)
# 外付けストレージ追加後に別途対応する
```

> **Note**: Immich の写真データ（HDD）は、Nextcloud データと同様にバックアップ宛先が同一 HDD のため、外付けストレージ追加まではバックアップなし。DB のみ SSD → HDD にバックアップされる。

## 確認チェックリスト

- [ ] `sudo docker ps` で 4 コンテナが `Up`
- [ ] `https://immich.home.lan/` でログイン画面が表示される
- [ ] スマホアプリから接続・写真アップロードができる
- [ ] アップロードした写真が `<HDD_ROOT>/appdata/immich/` 以下に保存されている

## トラブルシュート

- **502 / 接続できない**: `immich_server` が起動しきるまで数分かかる。`docker compose logs -f immich-server` で確認
- **ML が遅い / 動かない**: Mac mini 2014 は GPU なし CPU 推論のため、初回インデックスに時間がかかる。メモリ 16GB あるので問題なく動く
- **ストレージパスが見つからない**: `.env` の `UPLOAD_LOCATION` / `DB_DATA_LOCATION` を再確認

## 次へ

Nextcloud からのデータ移行は `docs/13-immich-migration.md` 参照（移行実施後に作成）。
