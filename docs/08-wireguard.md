# 8. WireGuard の導入とクライアント配布

前: [7. Nginx Proxy Manager の導入と Nextcloud の https 化](07-nginx-proxy-manager.md)

## 目的

Mac mini 上で WireGuard サーバを立て、外出先から VPN で家庭 LAN にアクセスできるようにする。
クライアント (iPhone/Android/Mac/Windows) に設定を配布し、VPN 経由で Nextcloud にアクセスできる状態を作る。

## 前提

- [7. Nginx Proxy Manager の導入と Nextcloud の https 化](07-nginx-proxy-manager.md) までが完了している
- [6. AdGuard Home](06-adguard-home.md) が LAN 内で稼働している
- ルータで UDP ポート開放ができる

> **グローバル IP について**: 日本の家庭回線はほぼ動的 IP (再起動や ISP 側のリース更新で変わる)。WireGuard の接続先が変わると外出先から繋がらなくなるため、**DDNS (Dynamic DNS) で FQDN を発行して運用する**のが実質必須。本手順では §8-1 で DuckDNS を設定してから §8-3 の `.env` に使う。

## 手順

### 8-1. DuckDNS で FQDN を用意

[DuckDNS](https://www.duckdns.org/) は無料・メール認証不要・更新が `curl` 一発の軽量 DDNS サービス。家庭用途で事実上の定番。

#### 8-1-1. アカウント作成と FQDN 取得

1. <https://www.duckdns.org/> にアクセス
2. 右上で GitHub / Google / Twitter / Reddit のいずれかでログイン
3. トップページ `domains` 欄に希望のサブドメイン (例: `mikehome`) を入れて `add domain`
4. 発行される FQDN (例: `mikehome.duckdns.org`) を控える
5. ページ上部に表示される **token** (UUID 形式) を控える。以後 IP 更新の認証に使う

#### 8-1-2. OMV 側に IP 自動更新スクリプトを設置

グローバル IP が変わったら DuckDNS に通知する cron を仕掛ける。

```bash
sudo -i
mkdir -p /srv/appdata/duckdns
cat > /srv/appdata/duckdns/duck.sh <<'EOF'
#!/bin/bash
DOMAIN="<DuckDNS サブドメイン>"       # 例: mikehome (.duckdns.org は不要)
TOKEN="<DuckDNS token>"
LOG=/srv/appdata/duckdns/duck.log
echo -n "$(date -Is) " >> "${LOG}"
curl -s "https://www.duckdns.org/update?domains=${DOMAIN}&token=${TOKEN}&ip=" >> "${LOG}"
echo "" >> "${LOG}"
EOF
chmod 700 /srv/appdata/duckdns/duck.sh
nano /srv/appdata/duckdns/duck.sh   # DOMAIN と TOKEN を実値に書き換える
exit
```

cron を登録 (5 分おき)。OMV は `root` の crontab を Salt で管理しているため、混ざらないよう `/etc/cron.d/` に独立ファイルで置く:

```bash
sudo tee /etc/cron.d/duckdns >/dev/null <<'EOF'
*/5 * * * * root /srv/appdata/duckdns/duck.sh >/dev/null 2>&1
EOF
sudo chmod 644 /etc/cron.d/duckdns
```

> `/etc/cron.d/` 方式は `crontab -e` と違い、**時刻フィールドの直後にユーザー名 (`root`) を書く**点に注意。

#### 8-1-3. 動作確認

手動で 1 回叩き、結果が `OK` になることを確認:

```bash
sudo /srv/appdata/duckdns/duck.sh
sudo cat /srv/appdata/duckdns/duck.log
# 末尾が "OK" なら成功。"KO" はトークン or ドメイン間違い
```

FQDN が今のグローバル IP を指しているかチェック:

```bash
dig +short <FQDN>.duckdns.org
curl -s https://ifconfig.me
```

両者が一致すれば DNS レコード反映済み。反映に最大 1 分ほどかかる場合あり。

### 8-2. 導入方式の選択

2 案:

- **(A) OMV プラグイン `openmediavault-wireguard`**: GUI ベースで設定。シンプル。プラグイン依存
- **(B) Docker コンテナ `linuxserver/wireguard`**: compose 一式で管理可能。他コンテナ構成と統一感

本手順では **(B) Docker コンテナ方式** で書く (再現性と移植性が高い)。プラグインを使う場合は Web UI の `Services → WireGuard` で同等の設定を行えばよい。

### 8-3. compose ファイル

`/srv/compose/wireguard/.env`:

```dotenv
SSD_ROOT=/srv
WG_HOST=<FQDN>.duckdns.org
WG_PORT=51820
WG_PEERS=3
WG_INTERNAL_SUBNET=10.10.0.0/24
WG_PEERDNS=<OMV_IP>
```

`/srv/compose/wireguard/docker-compose.yml`:

```yaml
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=100
      - TZ=Asia/Tokyo
      - SERVERURL=${WG_HOST}
      - SERVERPORT=${WG_PORT}
      - PEERS=${WG_PEERS}
      - PEERDNS=${WG_PEERDNS}   # AdGuard Home (§6) に向ける
      - INTERNAL_SUBNET=10.10.0.0
      - ALLOWEDIPS=192.168.1.0/24,10.10.0.0/24
      - LOG_CONFS=true
    volumes:
      - ${SSD_ROOT}/appdata/wireguard:/config
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

> **ALLOWEDIPS の考え方**
> - ここに入れた CIDR が「VPN 経由で通る範囲」になる (クライアント側の AllowedIPs)
> - `192.168.1.0/24` は家 LAN。家 LAN の IP レンジに合わせて置き換える
> - `10.10.0.0/24` は WireGuard 内部サブネット
> - インターネットも VPN に流したい場合は `0.0.0.0/0` を入れる (フルトンネル)。今回はスプリットトンネルで LAN のみ

### 8-4. 起動

```bash
cd /srv/compose/wireguard
docker compose up -d
docker compose logs -f wireguard
```

起動するとコンテナ内 `/config/` に `peer1`〜`peer3` のディレクトリが生成され、各ピアの `.conf` と QR コード (`peer1.png`) ができる。
ホスト側から参照:

```bash
ls /srv/appdata/wireguard/
# -> peer1/, peer2/, peer3/, server/, wg0.conf, ...
```

### 8-5. ルータのポートフォワード

ルータ管理画面で:

- Protocol: UDP
- External Port: `51820`
- Internal Host: `<OMV_IP>`
- Internal Port: `51820`

### 8-6. クライアントへの配布

**iPhone / Android**:
1. 公式 WireGuard アプリをインストール
2. アプリで `+` → `QR コードからスキャン`
3. Mac mini のコンソールで QR を表示させる:
   ```bash
   docker exec -it wireguard /app/show-peer 1
   ```
   または生成された `peer1.png` を任意の方法で端末に表示
4. スキャン後、トンネルを ON

**Mac**:
1. App Store から WireGuard アプリ
2. `Import Tunnel(s) from File` で `peer2.conf` を開く

**Windows**:
1. <https://www.wireguard.com/install/> からクライアント
2. `Import tunnel(s) from file` で `peer3.conf`

### 8-7. DNS 対応の確認 (VPN 経由の名前解決)

本手順では `PEERDNS=<OMV_IP>` を環境変数に指定してあるため、生成される peer conf の `[Interface]` に自動的に `DNS = <OMV_IP>` が入る。これにより VPN 接続中のクライアントは **AdGuard Home ([§6](06-adguard-home.md))** 経由で `nextcloud.home.lan` を解決する。

生成された conf を念のため目視確認:

```bash
sudo cat /srv/appdata/wireguard/peer1/peer1.conf
```

`DNS = <OMV_IP>` の行があれば OK。なければ `.env` の `WG_PEERDNS` を直し、`docker compose up -d` でコンテナを作り直してから peer を再生成する。

### 8-8. 接続テスト

スマホを **モバイル回線に切り替えて** (Wi-Fi を切る) VPN トンネルを ON。

- `https://nextcloud.home.lan/` がブラウザで開く
- Nextcloud Android アプリから同 URL でログインできる
- `ping <OMV_IP>` が通る (ping アプリ等で確認)

## 確認

- [ ] `docker compose ps` で wireguard が running
- [ ] ルータ管理画面でポート 51820/UDP が Mac mini に転送されている
- [ ] モバイル回線のスマホから VPN 接続成立
- [ ] VPN 経由で Nextcloud にログインできる

## トラブルシュート

- **トンネルは張れるが Nextcloud に届かない**: `ALLOWEDIPS` に家 LAN のサブネットが入っているか確認
- **DNS が引けない**: 生成された peer conf の `[Interface]` に `DNS = <OMV_IP>` があるか。AGH 側で該当ドメインが `Filters → DNS rewrites` に登録されているか ([§6-5](06-adguard-home.md))
- **グローバル IP が変わって繋がらなくなった**: DuckDNS の更新が止まっている。`/srv/appdata/duckdns/duck.log` 末尾が連続で `OK` か確認。`KO` ならトークン/ドメインの typo、そもそもログが更新されていないなら `ls -l /etc/cron.d/duckdns` と `systemctl status cron` で cron が動いているか確認
- **モジュールが読み込めないエラー**: Debian の kernel と一致する `wireguard` カーネルモジュールが必要。`apt install wireguard-dkms linux-headers-amd64` をホスト側に入れる

## 次へ

次: [9. バックアップ設定](09-backup.md)
