# 9. バックアップ設定

前: [8. WireGuard の導入とクライアント配布](08-wireguard.md)

## 目的

重要データ (Nextcloud のデータとコンフィグ + DB + compose 一式) を、HDD 上の `<HDD_ROOT>/shares/backups/` に日次で rsync スナップショットする。

## 注意 (重要)

ストレージ構成の都合で、ドライブ故障耐性は部分的:

- **SSD 側のデータ** (DB、compose、NPM 設定、WG 鍵、Nextcloud の PHP 本体): 宛先が HDD なので **クロスドライブ保護される**。SSD 故障で完全消失しない
- **HDD 側のデータ** (Nextcloud のユーザーデータ): 宛先が同じ HDD 内なので **同一ドライブ内バックアップ**。HDD 故障で消失する。誤削除・ランサムウェア対策としての位置付け

> **いずれも外付けドライブを追加した時点で、宛先パスを差し替えて本来のバックアップ運用に移行する前提**。

## 前提

- [8. WireGuard の導入とクライアント配布](08-wireguard.md) までが完了している
- `<HDD_ROOT>/shares/backups/` が空の状態で存在する
- SSH で `<ADMIN_USER>@<OMV_IP>` にログイン可能

## 手順

### 9-1. バックアップ対象の整理

対象と除外:

| 対象パス | ドライブ | 内容 | 備考 |
|---|---|---|---|
| `<HDD_ROOT>/appdata/nextcloud/data/` | HDD | Nextcloud 実データ (全ユーザー) | |
| `/srv/appdata/nextcloud/html/config/` | SSD | Nextcloud 設定 | |
| `/srv/compose/` | SSD | compose と .env | |
| `/srv/appdata/mariadb/` | SSD | DB のファイルレベル | **cold backup: mysqldump の方が安全** (9-2 で別建て) |
| `/srv/appdata/npm/` | SSD | NPM 設定・証明書 | |
| `/srv/appdata/wireguard/` | SSD | WG 鍵・ピア conf | |

除外:
- `<HDD_ROOT>/shares/backups/` (自己参照)
- `/srv/appdata/redis/` (揮発キャッシュ)
- `/srv/docker/` (イメージ層。再取得可能)

### 9-2. rsync スナップショットスクリプト

宛先を変数化しておき、将来外付けドライブに差し替えやすくする。

`/usr/local/sbin/homenas-backup.sh` を作成:

```bash
#!/bin/bash
set -euo pipefail

# === 設定 ===
SSD_ROOT="/srv"
HDD_ROOT="<HDD_ROOT>"                      # 実パスに置換
DEST_ROOT="${HDD_ROOT}/shares/backups"     # 将来: /mnt/backup-hdd/snapshots
RETENTION_DAYS=14

TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
DEST="${DEST_ROOT}/snapshot-${TIMESTAMP}"
LATEST="${DEST_ROOT}/latest"

mkdir -p "${DEST}/ssd" "${DEST}/hdd"

# rsync: --link-dest で前回スナップショットとのハードリンクを張り、世代間で未変更ファイルを重複保存しない
LINK_DEST_SSD=""
LINK_DEST_HDD=""
if [ -L "${LATEST}" ]; then
  LATEST_REAL="$(readlink -f "${LATEST}")"
  [ -d "${LATEST_REAL}/ssd" ] && LINK_DEST_SSD="--link-dest=${LATEST_REAL}/ssd"
  [ -d "${LATEST_REAL}/hdd" ] && LINK_DEST_HDD="--link-dest=${LATEST_REAL}/hdd"
fi

# --- DB は cold dump してから rsync に含める ---
DUMP_DIR="${SSD_ROOT}/appdata/mariadb/_dumps"
mkdir -p "${DUMP_DIR}"
docker exec nextcloud-db sh -c '
  exec mariadb-dump --single-transaction -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases
' > "${DUMP_DIR}/all-$(date +%Y%m%d).sql"
# 古い dump を 7 日で削除
find "${DUMP_DIR}" -name 'all-*.sql' -mtime +7 -delete

# --- SSD 側のソースを DEST/ssd/ に ---
rsync -aH --delete --numeric-ids \
  --exclude='appdata/redis/' \
  --exclude='docker/' \
  ${LINK_DEST_SSD} \
  "${SSD_ROOT}/appdata/" \
  "${SSD_ROOT}/compose/" \
  "${DEST}/ssd/"

# --- HDD 側のソースを DEST/hdd/ に (自己参照しないよう backups を除外) ---
rsync -aH --delete --numeric-ids \
  --exclude='shares/backups/' \
  ${LINK_DEST_HDD} \
  "${HDD_ROOT}/shares/" \
  "${HDD_ROOT}/appdata/" \
  "${DEST}/hdd/"

# latest シンボリックリンクを更新
ln -sfn "${DEST}" "${LATEST}"

# 古いスナップショットを削除
find "${DEST_ROOT}" -maxdepth 1 -type d -name 'snapshot-*' -mtime +${RETENTION_DAYS} -exec rm -rf {} +

echo "Backup complete: ${DEST}"
```

> 上記の `<HDD_ROOT>` は実パスに置換する。`--exclude` のパスは rsync のソースから見た相対パス。SSD 側と HDD 側で rsync を 2 回走らせ、同じ snapshot ディレクトリ配下に `ssd/` と `hdd/` を作る構造にしている。

パーミッション (作成した `/usr/local/sbin/homenas-backup.sh` を root 専有・実行可に):

```bash
sudo chmod 700 /usr/local/sbin/homenas-backup.sh
sudo chown root:root /usr/local/sbin/homenas-backup.sh
```

> スクリプトをホームディレクトリ等で編集してから配置する場合は `sudo install -m 700 -o root -g root ~/homenas-backup.sh /usr/local/sbin/` でも同じ結果になる。

### 9-3. 手動実行と Dry-run

いきなり cron に入れず、まず dry-run で挙動を確認。`rsync` 行に `--dry-run -v` を一時的に加えて手動実行:

```bash
sudo /usr/local/sbin/homenas-backup.sh 2>&1 | tee /tmp/backup.log
```

最初の 1 回で:
- `<HDD_ROOT>/shares/backups/snapshot-YYYYMMDD-HHMMSS/{ssd,hdd}/` ができる
- `<HDD_ROOT>/shares/backups/latest` シンボリックリンクができる

### 9-4. cron で日次実行

OMV の Web UI で `System → Scheduled Tasks → Add`:

| 項目 | 値 |
|---|---|
| Enabled | Yes |
| User | root |
| Schedule | Daily, Minute 0, Hour 3 |
| Command | `/usr/local/sbin/homenas-backup.sh >> /var/log/homenas-backup.log 2>&1` |

Web UI を使わず直接 crontab でも可:

```bash
sudo crontab -e
# 以下を追記
0 3 * * * /usr/local/sbin/homenas-backup.sh >> /var/log/homenas-backup.log 2>&1
```

### 9-5. 復元テスト

**必ずやる**。復元できないバックアップは無価値。

1. 適当なファイルを Nextcloud で 1 つアップロードする (例: `restore-test.txt`)
2. サーバ側で実体パスを確認: `sudo find <HDD_ROOT>/appdata/nextcloud/data -name 'restore-test.txt'`
3. バックアップを手動で 1 回走らせる
4. `<HDD_ROOT>/shares/backups/snapshot-.../hdd/appdata/nextcloud/data/.../restore-test.txt` があることを確認
5. Nextcloud 上で元ファイルを削除 (さらにゴミ箱からも)
6. バックアップから戻す (対応するユーザーディレクトリへ):
   ```bash
   sudo cp <HDD_ROOT>/shares/backups/latest/hdd/appdata/nextcloud/data/<ユーザー>/files/restore-test.txt \
       <HDD_ROOT>/appdata/nextcloud/data/<ユーザー>/files/
   sudo chown www-data:www-data <HDD_ROOT>/appdata/nextcloud/data/<ユーザー>/files/restore-test.txt
   # Nextcloud に再スキャンさせる
   docker exec -u www-data nextcloud php occ files:scan <ユーザー>
   ```
7. Nextcloud 上で戻っていることを確認

DB 復元テスト:

```bash
# コンテナ内で dump を流し込む (テスト用に別 DB に入れて確認する方が安全)
docker exec -i nextcloud-db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < <HDD_ROOT>/shares/backups/latest/ssd/appdata/mariadb/_dumps/all-YYYYMMDD.sql
```

### 9-6. 外付けドライブ移行時のプラン (将来)

USB 外付け HDD を追加した時:

1. OMV で外付けをマウント (`/srv/dev-disk-by-uuid-YYYY/` 等)
2. OMV の Shared Folders には登録しない (バックアップ専用)
3. `homenas-backup.sh` の `DEST_ROOT` を外付けのパスに書き換える
4. 初回は `rsync` が丸ごとコピーするため時間がかかる (1TB ほぼ全データのコピー)
5. 完了後、旧宛先 `<HDD_ROOT>/shares/backups/` は保持しつつ、退役を判断

## 確認

- [ ] `/usr/local/sbin/homenas-backup.sh` が存在し、root 専有 (700)
- [ ] 手動実行で `snapshot-*/` が作られる
- [ ] `latest` が最新スナップショットを指している
- [ ] cron (OMV Scheduled Tasks) が毎日 03:00 に登録されている
- [ ] 復元テストが成功
- [ ] `/var/log/homenas-backup.log` にエラーがない

## トラブルシュート

- **`mariadb-dump: not found`**: 古い MariaDB イメージ (10.x 以前) では `mysqldump` のみ。スクリプト内の `mariadb-dump` を `mysqldump` に戻す。MariaDB 11.x 以降は `mariadb-dump` が正
- **`mariadb-dump` 失敗**: `.env` の `MYSQL_ROOT_PASSWORD` と起動時の環境変数が一致しているか
- **rsync がパーミッションエラー**: root で実行しているか (`sudo` 経由)
- **空き容量不足**: `du -sh <HDD_ROOT>/shares/backups/*` で各世代のサイズを確認。`RETENTION_DAYS` を下げる
- **`--link-dest` が効かずサイズが増え続ける**: 前回の `latest` シンボリックリンクが正しく張れているか確認

## 次へ

次: [10. 各クライアントからの接続確認](10-verification.md)
