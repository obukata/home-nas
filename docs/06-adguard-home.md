# 6. AdGuard Home の導入 (LAN 内 DNS + 広告ブロック)

前: [5. Nextcloud コンテナの構築](05-nextcloud.md)

## 目的

家庭内に DNS サーバを立て、以下 2 つの役割を担わせる:

1. **LAN 内プライベートドメインの解決**: `nextcloud.home.lan` `npm.home.lan` `n8n.home.lan` 等を家庭内サブネット上の Mac mini (`<OMV_IP>`) に向ける
2. **広告/トラッカードメインのブロック**: 家中のデバイスの DNS 問い合わせをこのサーバ経由にし、広告/解析系ドメインに嘘の応答を返して通信させない

本章を通すと、以降の章 (§7 NPM, §8 WireGuard, §11 n8n) がすべて `*.home.lan` の名前で動かせるようになる。

## 背景: なぜ自前 DNS が必要か

通常は「ルータの DNS 静的エントリ機能」で済ませるのが最小構成。しかし ISP のレンタルルータ (SoftBank 光 BB ユニット E-WMTA2.3 / 一部の NTT HGW 等) はこの機能が削られていることが多く、その場合は家庭内に 1 台 DNS サーバを立てるしかない。今回は「どうせ立てるなら広告ブロックもやらせよう」という発想で **AdGuard Home (以下 AGH)** を採用する。

> **採用理由**
> - 素の `dnsmasq` より GUI がある (家族に説明しやすい)
> - Pi-hole より設定がシンプル (Go 製 1 バイナリ)
> - LAN 内静的エントリ / ブロックリスト / DoH/DoT upstream が標準装備
> - 将来 Pi-hole 等に移行したくなっても静的エントリは同じ概念で再設定できる

## 前提

- [5. Nextcloud コンテナの構築](05-nextcloud.md) まで完了している
- `<OMV_IP>` (Mac mini の LAN 内固定 IP) が固定されている ([§2](02-initial-config.md) 参照)
- ルータが「DHCP で配布する DNS サーバ IP」を変更できるなら有利 (後述)。できなくても問題ない

## 手順

### 6-1. compose と .env の作成

`/srv/compose/adguardhome/.env`:

```bash
sudo mkdir -p /srv/compose/adguardhome
sudo nano /srv/compose/adguardhome/.env
```

```dotenv
SSD_ROOT=/srv
TZ=Asia/Tokyo
```

`/srv/compose/adguardhome/docker-compose.yml`:

```bash
sudo nano /srv/compose/adguardhome/docker-compose.yml
```

```yaml
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    network_mode: host   # DNS 用の :53 とクライアント IP 可視化のため host モード
    environment:
      TZ: ${TZ}
    volumes:
      - ${SSD_ROOT}/appdata/adguardhome/work:/opt/adguardhome/work
      - ${SSD_ROOT}/appdata/adguardhome/conf:/opt/adguardhome/conf
```

> **Note: `network_mode: host` の選択理由**
> - AGH は :53 で DNS 要求を受ける。コンテナ側 NAT を通すと、家中のクライアントが全員「Docker ブリッジの IP」に見え、統計や個別クライアントルールが機能しない
> - ホストモードでは AGH がホスト NIC に直接バインドするので、クライアント IP がそのまま残る
> - トレードオフは「コンテナ分離性が低下する」だが、家庭内信頼境界では許容範囲

### 6-2. ホスト側で :53 が空いているか確認

Debian/OMV では `systemd-resolved` が `127.0.0.53:53` を占めていることがある。これは問題にならない (AGH は `0.0.0.0:53` にバインドする)。ただし稀に `0.0.0.0:53` でリッスンしている場合、起動が失敗する:

```bash
sudo ss -ulnp | grep ':53'
```

`127.0.0.53:53` だけなら OK。`0.0.0.0:53` や `<OMV_IP>:53` が出ていたら、そのプロセスを止めるか `DNSStubListener=no` にする:

```bash
# 必要な場合のみ
sudo mkdir -p /etc/systemd/resolved.conf.d
echo -e "[Resolve]\nDNSStubListener=no" | sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf
sudo systemctl restart systemd-resolved
```

### 6-3. 起動

```bash
sudo -i
cd /srv/compose/adguardhome
docker compose up -d
docker compose logs -f adguardhome
# ログが流れ終わったら Ctrl+C、exit
exit
```

起動確認:

```bash
sudo docker ps | grep adguardhome
sudo ss -ulnp | grep ':53'   # :53 を AdGuardHome (pid) が持っていれば OK
```

### 6-4. 初回セットアップウィザード

ブラウザで `http://<OMV_IP>:3000/` を開く。(初回セットアップ時のみこのポートで UI が起動する)

ウィザードに従う:

1. **Admin Web Interface**: Listen interface = `All interfaces`、Port = **`8082`** (後で NPM で :80 を使うので `80` は避ける)
2. **DNS server**: Listen interface = `All interfaces`、Port = `53` のまま
3. **管理者ユーザー作成**: Username と強固なパスワードを設定 (パスワードマネージャーに控える)
4. **Configuration saved** の画面で設定を保存

完了すると案内が `http://<OMV_IP>:8082/` に切り替わるので、以降の管理はそちらで。

### 6-5. LAN 内プライベートドメインの登録

AGH 管理画面: `Filters → DNS rewrites → Add DNS rewrite`

以下のエントリを追加 (最低限は `nextcloud.home.lan` だけでよいが、以降の章で出てくる名前もまとめて入れておくと楽):

| Domain | Answer |
|---|---|
| `nextcloud.home.lan` | `<OMV_IP>` |
| `npm.home.lan` | `<OMV_IP>` |
| `n8n.home.lan` (§11 で使う場合) | `<OMV_IP>` |

ワイルドカードも使える (`*.home.lan` → `<OMV_IP>` で「全部 Mac mini に向ける」ことも可)。家庭内サービスを全部 Mac mini に集約する設計なら、ワイルドカード 1 行で済む:

| Domain | Answer |
|---|---|
| `*.home.lan` | `<OMV_IP>` |

### 6-6. Upstream DNS の設定

AGH 自身がプライベート以外の名前 (`google.com` 等) を解決するために使う上流 DNS。

管理画面: `Settings → DNS settings → Upstream DNS servers`

推奨 (どれか 1 つ以上、複数書けば冗長化):

```
https://dns.google/dns-query
https://cloudflare-dns.com/dns-query
https://dns.quad9.net/dns-query
```

いずれも DoH (DNS over HTTPS) 対応で、上流との通信が暗号化される (ISP からドメインが丸見えにならない)。

`Apply` を押して保存。その後の **Test upstreams** で全部 `OK` が出れば反映成功。

### 6-7. 広告ブロックリストの有効化

管理画面: `Filters → DNS blocklists`

デフォルトで `AdGuard DNS filter` が有効になっているのを確認。追加で以下を入れると効果が伸びる (任意):

- `AdAway Default Blocklist`
- `OISD Blocklist Full` (大規模・誤爆少なめ)
- 日本のモバイルアプリ広告を重点ブロックしたければ `280blocker domain list` (検索して URL を貼る)

Disable / Enable は個別に切り替えられる。**誤爆で特定サイトが開けなくなった場合**は `Query log` で原因ドメインを特定し、`Filters → Custom filtering rules` で許可 (`@@||example.com^`) するのが定石。

### 6-8. クライアント側の DNS 切り替え

AGH を**使わせる**には、各端末で DNS サーバを `<OMV_IP>` に向ける必要がある。光 BB ユニットが DHCP で配る DNS を変更できないので、**端末ごとに手動設定**する。

#### ルータで一括設定できる場合 (参考)

もしルータ管理画面に「DHCP で配る DNS サーバ」の設定欄があれば、そこを `<OMV_IP>` に変えるのが最強 (端末個別設定不要)。光 BB ユニットはこの機能もないので、下の端末別設定で進める。

#### Mac (LAN 内)

1. システム設定 → ネットワーク → Wi-Fi (または Ethernet) → 詳細 → **DNS**
2. 左下 `+` で `<OMV_IP>` を追加、既存のエントリより**上**に置く
3. `OK` → 適用

確認:

```bash
dig nextcloud.home.lan
# ANSWER SECTION に <OMV_IP> が返れば OK
```

#### Windows (LAN 内)

1. 設定 → ネットワークとインターネット → Wi-Fi/イーサネット → アダプター設定の変更
2. 該当アダプタを右クリック → プロパティ → `インターネット プロトコル バージョン 4 (TCP/IPv4)` → プロパティ
3. `次の DNS サーバーのアドレスを使う` で **優先**: `<OMV_IP>`、**代替**: `1.1.1.1` 等
4. OK で保存

#### iPhone (LAN 内)

1. 設定 → Wi-Fi → 接続中のネットワークの `ⓘ` → `DNS を構成` → `手動`
2. 既存のサーバーを削除 (必要に応じて)、`サーバーを追加` で `<OMV_IP>`
3. 保存

#### Android (LAN 内)

端末により UI が異なる。代表的な 2 方式:

- **Wi-Fi 詳細から静的 IP に変更** → DNS1 に `<OMV_IP>`、DNS2 に `1.1.1.1` 等
- **Private DNS (Android 9+)**: `設定 → ネットワーク → プライベート DNS` — ただしこれは DoT サーバ名指定のみで、IP 直指定不可。LAN 内では使いにくい

Android で手動 DNS 設定を避けたいなら、**自宅でも WireGuard を ON にする運用**が一番楽 (§8 完了後)。

#### 外出中の端末 (WireGuard 経由)

§8 で WireGuard クライアント conf を生成する際、`[Interface]` セクションに以下を記述する:

```
DNS = <OMV_IP>
```

これで VPN 接続中は自動的に AGH 経由の名前解決になる (具体的な conf 生成手順は §8 側に記載)。

### 6-9. Nextcloud の trusted_domains を確認

§5-1 で `NEXTCLOUD_TRUSTED_DOMAINS=nextcloud.home.lan <OMV_IP>` を入れてあるので、名前でも IP でも Nextcloud が受け入れる状態になっている。この章で `nextcloud.home.lan` が実際に引けるようになるので、ブラウザで `http://nextcloud.home.lan:8080/` を試すと直接アクセスできる (まだ https ではない、それは §7 で)。

## 確認

- [ ] `sudo docker ps` で `adguardhome` が `Up`
- [ ] 管理画面 `http://<OMV_IP>:8082/` にログインできる
- [ ] Mac で `dig nextcloud.home.lan` を打つと `<OMV_IP>` が返る
- [ ] ブラウザで `http://nextcloud.home.lan:8080/` が開く (Nextcloud のログイン画面が出る)
- [ ] 管理画面 `Query log` に、自分の端末からの問い合わせが流れてくる
- [ ] 適当な広告サイト (例: `doubleclick.net`) を `dig` すると `0.0.0.0` が返る

## トラブルシュート

- **起動直後に `bind: address already in use` でコンテナが落ちる**: ホストの `:53` が他プロセスに握られている。§6-2 参照
- **管理画面 `http://<OMV_IP>:3000/` が開かない**: 既に初期セットアップ済みの可能性。`http://<OMV_IP>:8082/` を試す
- **`dig nextcloud.home.lan` が SERVFAIL**: クライアントの DNS がまだ AGH を見ていない。§6-8 の設定を再確認
- **広告はブロックされるが Nextcloud も開けない**: AGH の `Query log` で該当ドメインを探し、誤爆していれば `Custom filtering rules` で `@@` で許可
- **家全体が DNS で止まる**: AGH コンテナが落ちると家中の名前解決が停止する。`restart: unless-stopped` は付いているが、ホスト自体が落ちた時に備えて **フォールバック用の代替 DNS (`1.1.1.1` 等) を各端末の DNS2 に入れておく** と停電復旧時に慌てずに済む

## バックアップへの追加

AGH の設定は `/srv/appdata/adguardhome/conf/AdGuardHome.yaml` に集約される。[§9](09-backup.md) のバックアップ対象が `/srv/appdata/` 全体を拾うなら自動で含まれる。

## 次へ

次: [7. Nginx Proxy Manager の導入と Nextcloud の https 化](07-nginx-proxy-manager.md)
