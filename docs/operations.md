# 運用ガイド (構築完了後)

構築手順 ([§0](00-overview.md)-[§11](11-n8n.md)) が完了した後の定常運用メモ。

- [1. 新しい端末を追加する](#1-新しい端末を追加する)
- [2. 新しい家族アカウントを追加する](#2-新しい家族アカウントを追加する)
- [3. 端末を紛失/売却した時の無効化](#3-端末を紛失売却した時の無効化)
- [4. 端末の DNS バイパスを無効化する (AGH を効かせるため)](#4-端末の-dns-バイパスを無効化する-agh-を効かせるため)
- [5. 日次のヘルスチェック](#5-日次のヘルスチェック)
- [6. コンテナのアップデート](#6-コンテナのアップデート)
- [7. 定期メンテナンス](#7-定期メンテナンス)
- [8. トラブル対応の基本フロー](#8-トラブル対応の基本フロー)
  - [8-A. 特定サイトだけ繋がらない時の切り分け](#8-a-特定サイトだけ繋がらない時の切り分け)

---

## 1. 新しい端末を追加する

### 1-1. 新しいスマホ (iPhone / Android)

所要 5〜10 分。

1. App Store / Google Play から **WireGuard** 公式アプリをインストール
2. 同じく **Nextcloud** 公式アプリをインストール
3. サーバで空いている peer を確認
   ```bash
   ls /srv/appdata/wireguard/
   # peer1〜peer3 の使用状況を把握
   ```
4. peer が足りなければ枠を増やす
   ```bash
   sudo -i
   nano /srv/compose/wireguard/.env
   # WG_PEERS=3 → 4 などに増やす
   cd /srv/compose/wireguard
   docker compose up -d
   exit
   ```
   既存 peer の鍵は保持されたまま、新 peer が追加される
5. サーバで QR を表示
   ```bash
   sudo docker exec -it wireguard /app/show-peer 3   # 割当先の番号
   ```
6. スマホ WireGuard アプリ → 右上 `+` → QR スキャン → 名前を付けて保存 → トンネル ON
7. **モバイル回線に切り替えて** (Wi-Fi OFF) Nextcloud アプリを開く
   - サーバ URL: `https://nextcloud.home.lan`
   - 自己署名警告は信頼する
   - アカウントでログイン
8. Nextcloud アプリ → Settings → **自動アップロード** を有効化 (写真を上げたい場合)
9. Wi-Fi に戻して最終動作確認

### 1-2. 新しい PC (Mac / Windows)

1. WireGuard 公式アプリを [wireguard.com/install](https://www.wireguard.com/install/) からインストール (Mac は App Store も可)
2. サーバ側で conf の読み取り権限を一時的に付与
   ```bash
   sudo chmod 644 /srv/appdata/wireguard/peer3/peer3.conf
   ```
3. 新しい PC から scp で取得
   ```bash
   scp <ADMIN_USER>@192.168.3.19:/srv/appdata/wireguard/peer3/peer3.conf ~/
   ```
4. サーバ側で権限を戻す
   ```bash
   sudo chmod 600 /srv/appdata/wireguard/peer3/peer3.conf
   ```
5. WireGuard アプリ → **Import Tunnel(s) from File** → `peer3.conf` を指定
6. トンネル ON → テザリング等で外部ネットに切り替えて疎通確認
7. [Nextcloud デスクトップクライアント](https://nextcloud.com/install/) をインストール
   - サーバ URL: `https://nextcloud.home.lan`
   - 同期フォルダを選択

---

## 2. 新しい家族アカウントを追加する

1. 管理者で Nextcloud Web UI にログイン (`https://nextcloud.home.lan/`)
2. 右上アイコン → **ユーザ**
3. `+ 新しいユーザー` → ユーザー名 / 表示名 / パスワード / メール / グループ
4. 共有フォルダ (`Family` / `Work` 等) を既存ユーザーから新ユーザーに共有
   - 自分のアカウントで該当フォルダを開く → 共有アイコン → 新ユーザー名を検索して追加
5. 新ユーザーに伝える情報: URL (`https://nextcloud.home.lan`) / ユーザー名 / 初期パスワード
6. 新ユーザーのスマホ/PC セットアップは §1 を参照

---

## 3. 端末を紛失/売却した時の無効化

### 3-1. WireGuard peer を失効させる

紛失した端末に対応する peer 番号を無効化:

```bash
sudo rm -rf /srv/appdata/wireguard/peer<番号>
sudo -i
cd /srv/compose/wireguard
docker compose restart wireguard
exit
```

以後、その peer の鍵では接続できない。他の端末には影響なし。

### 3-2. Nextcloud アプリトークンを失効させる

該当ユーザーで Nextcloud Web にログイン → **個人設定 → セキュリティ → デバイス＆セッション**:

- 紛失端末に対応するセッションの **「取消」** を押す
- n8n 等で作った App password も同画面でいつでも取消可能

### 3-3. (念のため) パスワードを変更

同じく個人設定 → セキュリティ → パスワード変更。

---

## 4. 端末の DNS バイパスを無効化する (AGH を効かせるため)

Wi-Fi の DNS を AGH (`<OMV_IP>`) に向けても、端末側の以下の機能が有効だとそれらが**優先**され、AGH を素通りしてしまう。AGH のクエリ・ログに端末の IP が出ない・ブロックが効かない時はまずここを疑う。

### 4-1. Android (端末側システム設定)

**プライベート DNS を OFF にする** (これが最重要):

1. 設定 → **ネットワークとインターネット** → **プライベート DNS**
2. `OFF` を選択 (初期値は `自動` = Google の DoT を直接使うので AGH を通らない)

> 機種によっては 設定 → 接続 → その他の接続設定 → プライベート DNS の場所にある。

### 4-2. Chrome (Android / PC 共通)

**セキュア DNS を OFF にする**:

1. Chrome 右上 `︙` → 設定
2. **プライバシーとセキュリティ** → **セキュア DNS を使用**
3. `OFF` にする (または `現在のサービス プロバイダ` を選択)

初期値で ON になっていると、Chrome だけ独自に DoH で外部解決するため、AGH のブロックが Chrome だけ効かないという状態になる。Firefox にも同等の設定 (`about:preferences` → `DNS over HTTPS`) がある。

### 4-3. iOS (参考)

iOS は Android ほど強い DNS バイパスはないが、**構成プロファイル**で DoH/DoT を入れている場合や、**iCloud Private Relay** を有効にしている場合は同様のバイパスが発生する。

- 設定 → Apple ID → iCloud → **プライベートリレー** を OFF (Safari のみ影響)
- 設定 → 一般 → VPN とデバイス管理 にあやしい DNS プロファイルがないか確認

### 4-4. 動作確認

スマホで広告が多めのサイトを開いた後、AGH 管理画面 (`http://<OMV_IP>:8082`) の **クエリ・ログ** でスマホの IP (`192.168.3.x`) からのクエリが流れていれば OK。ブロック件数が Mac とそろって増えていれば完全に効いている。

---

## 5. 日次のヘルスチェック

不安な日や遠出前に回せばよい。5 分で終わる。

### 5-1. コンテナが全て起動しているか

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
```

以下が全て `Up` で `healthy` (該当するもの) なら OK:

- `adguardhome`
- `nextcloud-app` / `nextcloud-db` / `nextcloud-redis`
- `npm`
- `wireguard`

### 5-2. バックアップが昨夜回ったか

```bash
ls -lt <HDD_ROOT>/shares/backups/ | head
tail -30 /var/log/homenas-backup.log
```

直近の `snapshot-YYYYMMDD-HHMMSS/` が今朝のタイムスタンプであること、`log` にエラーがないこと。

### 5-3. DuckDNS が生きているか

```bash
tail -5 /srv/appdata/duckdns/duck.log
```

末尾 5 行が全て `OK` であること。`KO` が続いていたらトークン失効や cron 停止を疑う。

### 5-4. グローバル IP と DuckDNS が一致しているか

```bash
dig +short <FQDN>.duckdns.org
curl -4 -s https://ifconfig.me
```

両者一致が望ましい。ズレていれば DuckDNS の更新が遅れているか、cron が止まっている。

---

## 6. コンテナのアップデート

月 1 回程度。事前に §5-2 でバックアップが取れていることを確認してから実行。

```bash
sudo -i
for dir in /srv/compose/*/; do
  echo "== Updating ${dir}"
  cd "${dir}"
  docker compose pull
  docker compose up -d
done
docker image prune -f
exit
```

アップデート後に §5-1 で全コンテナが `Up` になっていることを確認。

もしアップデートで問題が出た場合:

```bash
# 直近のイメージに戻す (例: Nextcloud)
sudo -i
cd /srv/compose/nextcloud
docker compose down
# docker-compose.yml の image タグを前バージョンに固定し直す
nano docker-compose.yml
docker compose up -d
exit
```

> `latest` タグでデプロイしているため、ロールバックは「前バージョンのタグを明示」して上書き起動する形になる。よく使うバージョン (例: `nextcloud:29.0.5-apache`) をメモしておくと安心。

---

## 7. 定期メンテナンス

### 7-1. 自己署名証明書の更新 (825 日ごと)

証明書の有効期限が近づいたら再発行:

```bash
sudo -i
cd /srv/appdata/npm/custom-certs
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout nextcloud.key \
  -out nextcloud.crt \
  -days 825 \
  -subj "/CN=nextcloud.home.lan" \
  -addext "subjectAltName=DNS:nextcloud.home.lan"
exit
```

Mac に scp で取得し、NPM UI の Custom Certificate を **同名で上書きアップロード** (Edit → キーと証明書の両方を差し替え) → Save。

### 7-2. 外付け HDD 追加時のバックアップ宛先移行

[§9-6](09-backup.md) 参照。`homenas-backup.sh` の `DEST_ROOT` を外付けパスに書き換えて初回フル同期。

### 7-3. Nextcloud ユーザーの個人フォルダ整理

`Documents` / `Photos` 等の初期フォルダを後から消す場合は、対象ユーザーでログイン → ファイルアプリから削除 → ゴミ箱からも削除。

### 7-4. OS アップデート (OMV + Debian)

OMV Web UI → System → Update Management → Updates でパッケージ一覧を確認し、**Upgrade** を押す。再起動を要するカーネルアップデート時は、WireGuard コンテナが正常に上がってくるか起動後に確認。

---

## 8. トラブル対応の基本フロー

何かが動かなくなった時はまず順番に切り分ける。

### Step 1. 何が落ちているか把握

```bash
sudo docker ps -a --format 'table {{.Names}}\t{{.Status}}'
```

`Exited` のコンテナがあれば、それが原因の可能性が高い。

### Step 2. 該当コンテナのログを見る

```bash
sudo docker logs <container名> --tail 100
```

エラーメッセージから原因を推定。典型:

- `bind: address already in use` → ポート衝突
- `permission denied` → ボリュームの所有者/パーミッション
- `no such image` → pull 失敗 or タグが存在しない
- DB 接続エラー → `nextcloud-db` が起動しきる前に `app` が起動した可能性 → `restart app`

### Step 3. まず再起動で試す

```bash
sudo docker restart <container名>
```

### Step 4. compose ごと作り直す

```bash
sudo -i
cd /srv/compose/<サービス名>
docker compose down
docker compose up -d
exit
```

### Step 5. バックアップから復元

上記で直らなければ、[§9 復元テスト](09-backup.md) の手順で直近のバックアップから compose と appdata を戻す。

---

## 8-A. 特定サイトだけ繋がらない時の切り分け

「他のサイトは開くのにこのドメインだけ `DNS_PROBE_POSSIBLE` / 名前解決エラー」となった時の手順。原因は大体 (1) AGH のフィルタ誤爆 / (2) AGH 上流 DNS がそのドメインを引けない / (3) サイト側の障害、のどれか。

### Step 1. AGH 経由と別 resolver で同じ引きができるか比較

Mac / Linux のターミナルで:

```bash
# AGH 経由
dig <対象ドメイン> @<OMV_IP>

# Cloudflare の public resolver
dig <対象ドメイン> @1.1.1.1

# Google の public resolver
dig <対象ドメイン> @8.8.8.8
```

結果のパターン:

| AGH | 1.1.1.1 | 8.8.8.8 | 原因 | 対処 |
|---|---|---|---|---|
| 🔴 NXDOMAIN / ブロック済み | 🟢 | 🟢 | AGH フィルタ誤爆 | Step 2 へ |
| 🔴 SERVFAIL | 🟢 | 🔴 SERVFAIL | AGH 上流 (Google) がそのドメインだけ引けない | Step 3 へ |
| 🔴 SERVFAIL | 🔴 | 🔴 | サイト側 DNS の障害 | 向こうの復旧待ち |
| 🟢 (IP 返る) | 🟢 | 🟢 | DNS は健全。ブラウザキャッシュ / ルーティングの問題 | Step 4 へ |

### Step 2. AGH フィルタ誤爆の対処

AGH 管理画面 (`http://<OMV_IP>:8082`) → **クエリ・ログ** で検索ボックスに対象ドメインを入れる。赤表示の行をクリック → **「このドメインのブロックを解除する」** でホワイトリスト化。

### Step 3. AGH 上流 DNS の見直し

AGH 管理画面 → **設定 → DNS 設定 → アップストリームDNSサーバー** に、異なるプロバイダを複数入れておく:

```
tls://1dot1dot1dot1.cloudflare-dns.com
tls://dns.google
https://dns.google/dns-query
```

先頭 Cloudflare、Google をフォールバックにすると、片側で SERVFAIL を返すドメインもどちらかで引ける。「並列要求」にチェックを入れると同時に複数の上流に投げて最初に返った応答を採用する。

保存後、Mac のキャッシュも落として再確認:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
dig <対象ドメイン> @<OMV_IP>
```

### Step 4. ブラウザキャッシュが汚れている場合

Chrome: `chrome://net-internals/#dns` → **Clear host cache** / `chrome://net-internals/#sockets` → **Flush socket pools**

Safari: 開発メニュー → キャッシュを空にする

---

## 参考: よく使うコマンド早見

```bash
# 全コンテナ状態
sudo docker ps -a

# あるコンテナのログを流し見
sudo docker logs -f <container名>

# コンテナ内シェルに入る
sudo docker exec -it <container名> bash

# Nextcloud occ コマンド
sudo docker exec -u www-data nextcloud php occ <subcommand>

# ディスク使用量
df -h
sudo du -sh /srv/appdata/* <HDD_ROOT>/shares/* 2>/dev/null

# 直近のログイン履歴
last -n 20
```
