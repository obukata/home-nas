# 5. Nextcloud コンテナの構築

前: [4. omv-extras 導入と Docker / compose プラグイン有効化](04-omv-extras-docker.md)

## 目的

Docker で Nextcloud 本体 + MariaDB + Redis を起動し、LAN 内から Web UI にアクセスできる状態を作る。
https 化と外部ドメイン設定は次ステップ (NPM) でやるため、ここでは `http://<OMV_IP>:8080` の直接アクセスまで。

## 前提

- [04-omv-extras-docker.md](04-omv-extras-docker.md) が完了し、`proxy-net` 作成済み
- SSD 側: `/srv/compose/nextcloud/`, `/srv/appdata/nextcloud/html/`, `/srv/appdata/mariadb/`, `/srv/appdata/redis/` が存在する
- HDD 側: `<HDD_ROOT>/appdata/nextcloud/data/` が存在する

## 手順

### 5-1. .env の作成

まず必要な値を手元で用意する:

1. **`<HDD_ROOT>`**: `df -h | grep srv` で HDD のマウントポイント (`/srv/dev-disk-by-uuid-...`) を確認
2. **強固な乱数を 3 つ**生成してパスワードマネージャーに控える (MariaDB root, Nextcloud DB, Nextcloud admin 用):
   ```bash
   openssl rand -base64 24
   openssl rand -base64 24
   openssl rand -base64 24
   ```

ファイルを作成 (nano で):

```bash
sudo nano /srv/compose/nextcloud/.env
```

以下を貼り付けて、`<...>` のプレースホルダを自分の値で埋める:

```dotenv
SSD_ROOT=/srv
HDD_ROOT=<HDD_ROOT>

MYSQL_ROOT_PASSWORD=<強固な乱数1>
MYSQL_PASSWORD=<強固な乱数2>
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud

NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=<強固な乱数3>

NEXTCLOUD_TRUSTED_DOMAINS=nextcloud.home.lan <OMV_IP>
```

> **Note**: `OVERWRITEPROTOCOL` / `OVERWRITEHOST` / `OVERWRITECLIURL` は [§7](07-nginx-proxy-manager.md) (NPM と LAN 内 DNS が整ってから) で追加する。この段階で入れてしまうと `http://<OMV_IP>:8080/` にアクセスしても `https://nextcloud.home.lan/` へリダイレクトされ、まだ解決できないホスト名で失敗する。

保存: `Ctrl+O` → Enter → `Ctrl+X`。

パーミッションを 600 に (パスワードが入っているので必須):

```bash
sudo chmod 600 /srv/compose/nextcloud/.env
sudo ls -l /srv/compose/nextcloud/.env   # -rw------- 表示を確認
```

> **コメントの扱い**: ロケールが UTF-8 未設定だと日本語コメントが文字化けする。上記は ASCII のみにしてある。ロケールを直したければ `sudo dpkg-reconfigure locales` → `en_US.UTF-8` (と必要なら `ja_JP.UTF-8`) を有効化してから日本語コメントを入れる。

### 5-2. docker-compose.yml の作成

ファイルを作成:

```bash
sudo nano /srv/compose/nextcloud/docker-compose.yml
```

以下を貼り付け:

```yaml
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    restart: unless-stopped
    command:
      - --transaction-isolation=READ-COMMITTED
      - --log-bin=binlog
      - --binlog-format=ROW
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
    volumes:
      - ${SSD_ROOT}/appdata/mariadb:/var/lib/mysql
    networks:
      - internal

  redis:
    image: redis:7-alpine
    container_name: nextcloud-redis
    restart: unless-stopped
    volumes:
      - ${SSD_ROOT}/appdata/redis:/data
    networks:
      - internal

  app:
    image: nextcloud:29-apache
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      - db
      - redis
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      REDIS_HOST: redis
      NEXTCLOUD_ADMIN_USER: ${NEXTCLOUD_ADMIN_USER}
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: ${NEXTCLOUD_TRUSTED_DOMAINS}
      # OVERWRITEPROTOCOL / OVERWRITEHOST / OVERWRITECLIURL は §6 で override ファイルから追加
    volumes:
      # PHP 本体は SSD (速度重視)
      - ${SSD_ROOT}/appdata/nextcloud/html:/var/www/html
      # ユーザーアップロードデータは HDD (容量重視)
      - ${HDD_ROOT}/appdata/nextcloud/data:/var/www/html/data
    ports:
      - "8080:80"
    networks:
      - internal
      - proxy-net

networks:
  internal:
    driver: bridge
  proxy-net:
    external: true
```

> **Note**
> - `SSD_ROOT` / `HDD_ROOT` は `.env` に実パスで定義する (`HDD_ROOT=/srv/dev-disk-by-uuid-xxxx` のように)
> - DB (MariaDB) と Redis は SSD 側に置き、I/O を速くする
> - Nextcloud のユーザーデータは HDD 側、大容量が安全
> - 画像の `29-apache` はこの手順書作成時点の例。運用開始時に最新安定版の tag を確認

### 5-3. 起動

起動方法は2通り。どちらでもよい。

#### (推奨) OMV UI の Compose プラグインから

1. OMV 管理画面 → **Services → Compose → Files** を開く
2. `nextcloud` を選択して **Up** ボタン
3. ログは同画面の **Logs** から確認できる

#### SSH から直接

`/srv/compose/` は `root:root` のまま運用する方針 ([04-omv-extras-docker.md](04-omv-extras-docker.md) §4-4) なので、一般ユーザーのまま `cd` すると Permission denied になる。root シェルに入ってから実行する:

```bash
sudo -i
cd /srv/compose/nextcloud
docker compose up -d
docker compose logs -f app
# 確認後、Ctrl+C でログ終了 → exit で root シェルを抜ける
exit
```

初回起動時、Nextcloud がデータディレクトリの初期化を行うため数分かかる。
ログに `apache2 -DFOREGROUND` が出て安定すれば完了。

### 5-4. 初回アクセス確認

ブラウザで `http://<OMV_IP>:8080/` を開く。

- `.env` に書いた `NEXTCLOUD_ADMIN_USER` / `NEXTCLOUD_ADMIN_PASSWORD` で自動セットアップされているはず
- ログインして管理画面に入れることを確認
- 右下の `Settings → Overview` に警告が出る場合は後ステップで潰す (NPM 導入で解決するものが多い)

### 5-5. Nextcloud 設定の微調整

電話番号入力のデフォルト地域を日本に:

```bash
sudo docker exec -u www-data nextcloud php occ config:system:set default_phone_region --value="JP"
```

> 運用ユーザーは `docker` グループに入れない方針 ([§4-4](04-omv-extras-docker.md)) なので、`docker` コマンドは `sudo` 経由で叩く。`sudo -i` で root シェルに入ってから裸の `docker ...` を連発するのも可。

> `trusted_proxies` もここで設定したくなるが、値となる NPM コンテナの IP はまだ存在しない (§6 で作る)。§6 側の手順で設定するので、ここではスキップしてよい。

### 5-6. 夫婦 2 アカウントの作成と共有フォルダ設定

今回の構成では、ファイル共有はすべて Nextcloud に集約する。夫婦 2 人分のアカウントを作り、「夫婦で共有したいフォルダ」と「個人のフォルダ」を Nextcloud の共有機能で作る。

#### アカウントの設計

| アカウント | 用途 |
|---|---|
| `admin` (`.env` の `NEXTCLOUD_ADMIN_USER`) | Nextcloud 管理者。個人利用はしない |
| `<自分の名前>` (例: `obukata`) | 夫用。日常使い |
| `<配偶者の名前>` (例: `spouse`) | 妻用。日常使い |

管理者アカウントを日常使いしないのは、権限事故を防ぐため (誤って他ユーザーのデータを消せる権限を普段持ちたくない)。

#### 5-6-1. アカウント作成

`admin` でログインした状態で右上のアイコン → `Users` → `+ New user`。

自分用と配偶者用の 2 アカウントを作成:

| 項目 | 値 |
|---|---|
| Username | `<自分の名前>` (例: `obukata`) |
| Password | 強固なパスワード |
| Groups | (何も追加しない。デフォルトの一般ユーザーで OK) |

配偶者アカウントも同様に作成。

#### 5-6-2. 夫婦共有フォルダの作成と共有

まず **自分のアカウントでログインし直す** (Web UI 右上からサインアウト → 作成したユーザーで入る)。

Files 画面で `+` → `New folder`:

- `Family` (夫婦で共有するプライベートなファイル)
- `Work` (夫婦の仕事ファイル)

それぞれのフォルダで共有アイコン (人型マーク) をクリック → 検索欄に配偶者のユーザー名を入力 → 候補から選択 → 共有ボタン。

共有後のパーミッション確認: 共有したユーザー名の横に歯車アイコンが出るので、`allow editing` がチェックされていることを確認 (デフォルト ON)。

配偶者アカウントでログインし直すと、`Family` と `Work` が「他者からの共有」としてルートに現れる (Nextcloud 28 以降は自動で展開される)。

#### 5-6-3. 個人フォルダ

自分のアカウントのルート直下にある他のフォルダ (共有していないもの) が自動的に個人フォルダになる。例:

- `Documents` (Nextcloud のデフォルト)
- `Photos` (同)
- 自分で作った共有なしフォルダ

共有していないフォルダは **自分しか見えない** のが Nextcloud のデフォルト。明示的に何もしなくて OK。

#### 5-6-4. ストレージ消費の注意

Nextcloud の仕様上、共有フォルダは **所有者のディスククォータ** から消費される (共有された側はカウントしない)。`Family` と `Work` は自分が所有者なら自分側で容量が効く。

家庭運用で容量は HDD 1TB の範囲で回すので、個別クォータは無制限にしておいて OK:
- Admin の Users 管理画面 → 各ユーザーの `Quota` → `Unlimited`

### 5-7. デスクトップクライアントとモバイルアプリ

Nextcloud のアクセス方法はステップ 9 (動作確認) で一度にテストするが、概要だけここで:

- **Mac/Win**: 公式 Nextcloud Desktop Client → ローカルにフォルダを作って自動同期。Finder/Explorer から普通のフォルダとして使える
- **iOS/Android**: 公式 Nextcloud アプリ → ブラウジング、写真自動アップロード

NPM 経由の https URL (次章で構築) を各クライアントに設定する。家の中なら直接、外出先なら WireGuard VPN 経由でアクセス。

## 確認

- [ ] `sudo docker ps` で `nextcloud`, `nextcloud-db`, `nextcloud-redis` が `Up` (または `sudo docker compose -f /srv/compose/nextcloud/docker-compose.yml ps`)
- [ ] `http://<OMV_IP>:8080/` でログイン画面が出る
- [ ] 管理者でログインでき、ファイルのアップロード/ダウンロードができる
- [ ] `<HDD_ROOT>/appdata/nextcloud/data/` 以下に実ファイルが生成されている
- [ ] `/srv/appdata/mariadb/` 以下に DB ファイルが生成されている (SSD 上)

## トラブルシュート

- **502 / 503 が返る**: 初回は初期化に時間がかかる。`docker compose logs app` で `apache2 -DFOREGROUND` を待つ
- **`Access through untrusted domain`**: `.env` の `NEXTCLOUD_TRUSTED_DOMAINS` に現在アクセスしているホストを追加し再起動
- **DB 接続エラー**: `db` コンテナが起動しきる前に `app` が立ち上がるケース。`docker compose restart app`
- **パフォーマンスが出ない**: `occ config:app:set previewgenerator` や `memcache.local: OC\Memcache\APCu` の設定は後日チューニング

## 次へ

次: [6. AdGuard Home の導入 (LAN 内 DNS + 広告ブロック)](06-adguard-home.md)
