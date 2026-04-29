# 7. Nginx Proxy Manager の導入と Nextcloud の https 化

前: [6. AdGuard Home の導入 (LAN 内 DNS + 広告ブロック)](06-adguard-home.md)

## 目的

Nginx Proxy Manager (以降 NPM) を Docker で立て、`https://nextcloud.home.lan` で Nextcloud にアクセスできる状態を作る。
外部公開しないため、証明書は自己署名とする。将来サービスを増やした場合も NPM の GUI から追加するだけで済むようにしておく。

## 前提

- [5. Nextcloud コンテナの構築](05-nextcloud.md) で Nextcloud が `http://<OMV_IP>:8080/` で動いている
- [6. AdGuard Home の導入](06-adguard-home.md) で `nextcloud.home.lan` → `<OMV_IP>` が解決できる状態になっている

## 手順

### 7-1. NPM の compose ファイル

`/srv/compose/npm/.env`:

```dotenv
SSD_ROOT=/srv
```

`/srv/compose/npm/docker-compose.yml`:

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - ${SSD_ROOT}/appdata/npm/data:/data
      - ${SSD_ROOT}/appdata/npm/letsencrypt:/etc/letsencrypt
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```

NPM のデータは設定 + 証明書でサイズ小なので SSD 側に置く。

### 7-2. 起動

```bash
cd /srv/compose/npm
docker compose up -d
docker compose logs -f npm
```

### 7-3. 初回ログインと管理者の変更

ブラウザで `http://<OMV_IP>:81/` を開く。初期認証:

| 項目 | 値 |
|---|---|
| Email | `admin@example.com` |
| Password | `changeme` |

**即座に** プロフィールから Email / パスワードを変更する。

### 7-4. 自己署名証明書の作成

NPM の UI: `SSL Certificates → Add SSL Certificate → Custom` …と行きたいところだが、簡単なのは Let's Encrypt 以外のユースケースで `Custom` を使う方式。
もしくは NPM コンテナ内で openssl で自己署名を作り、`Custom` に読み込ませる。

SSH で作る例:

```bash
cd /srv/appdata/npm
mkdir -p custom-certs
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout custom-certs/nextcloud.key \
  -out custom-certs/nextcloud.crt \
  -days 825 \
  -subj "/CN=nextcloud.home.lan" \
  -addext "subjectAltName=DNS:nextcloud.home.lan"
```

生成した `.crt` と `.key` を NPM の `SSL Certificates → Add SSL Certificate → Custom` からアップロード。

> **Note**: 自己署名なので各クライアントでブラウザ/OS に警告が出る。常用するなら [mkcert](https://github.com/FiloSottile/mkcert) で家庭内 CA を作り、家族端末に CA を信頼させる方式の方が快適

### 7-5. プロキシホストの追加

NPM UI: `Hosts → Proxy Hosts → Add Proxy Host`

| 項目 | 値 |
|---|---|
| Domain Names | `nextcloud.home.lan` |
| Scheme | `http` |
| Forward Hostname/IP | `nextcloud` (同一ネットワーク `proxy-net` 上のコンテナ名) |
| Forward Port | `80` |
| Block Common Exploits | On |
| Websockets Support | On |

`SSL` タブ:
- SSL Certificate: 7-4 で作った自己署名を選択
- Force SSL: On
- HTTP/2 Support: On

`Advanced` タブに Nextcloud 推奨の追加設定:

```
client_max_body_size 10G;
proxy_request_buffering off;

location = /.well-known/carddav {
    return 301 $scheme://$host/remote.php/dav;
}
location = /.well-known/caldav {
    return 301 $scheme://$host/remote.php/dav;
}
```

Save。

この時点で `https://nextcloud.home.lan/` はログイン画面までは開くはず。ただし Nextcloud 内部の canonical URL はまだ `http://<OMV_IP>:8080` のままなので、リンクや OCS 応答が壊れる。次ステップで切り替える。

### 7-6. Nextcloud の canonical URL を https に切り替え

§5 でわざと保留していた `OVERWRITEPROTOCOL` / `OVERWRITEHOST` / `OVERWRITECLIURL` を有効化する。compose override で足すので、元の `docker-compose.yml` には触らない。

`/srv/compose/nextcloud/docker-compose.override.yml` を新規作成:

```bash
sudo -i
nano /srv/compose/nextcloud/docker-compose.override.yml
```

内容:

```yaml
services:
  app:
    environment:
      OVERWRITEPROTOCOL: https
      OVERWRITEHOST: nextcloud.home.lan
      OVERWRITECLIURL: https://nextcloud.home.lan
```

反映 (app コンテナが再作成される):

```bash
cd /srv/compose/nextcloud
docker compose up -d
exit
```

以降、Nextcloud が生成する絶対 URL はすべて `https://nextcloud.home.lan/...` になる。`http://<OMV_IP>:8080/` に直接アクセスしても `https://nextcloud.home.lan/` にリダイレクトされるので、以後の常用ルートは https 側一本になる。

### 7-7. Nextcloud 側の trusted_proxies 更新

NPM コンテナの IP を確認:

```bash
sudo docker inspect npm -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}'
```

Nextcloud で `trusted_proxies` に NPM の IP (またはサブネット) を設定:

```bash
sudo docker exec -u www-data nextcloud php occ config:system:set trusted_proxies 0 --value="<NPM_IP>"
```

### 7-8. 動作確認

ブラウザで `https://nextcloud.home.lan/` にアクセス。

- 自己署名なので警告が出たら例外許可
- ログイン画面が出て、Nextcloud 管理者で入れる
- `Settings → Administration → Overview` の Security & setup warnings が減っていることを確認

また、`http://<OMV_IP>:8080/` への直アクセスは開いたままでもよいが、常用しないなら Nextcloud compose の `ports` を落として閉じてもよい (トラブル切り分け用にしばらく残すのを推奨)。

## 確認

- [ ] `https://nextcloud.home.lan/` でログイン画面が出る
- [ ] Nextcloud のセキュリティ警告が NPM 導入前より減っている
- [ ] `trusted_proxies` が設定されている: `docker exec -u www-data nextcloud php occ config:system:get trusted_proxies`

## トラブルシュート

- **プロキシホストが 502**: NPM が Nextcloud コンテナに到達できていない。両方が `proxy-net` に所属しているか `docker network inspect proxy-net`
- **ブラウザで CN 不一致の警告**: `subjectAltName` が入っていない証明書は現代ブラウザで拒否される。7-4 の `-addext` を確認
- **Nextcloud ログに `X-Forwarded-Host` 関連の警告**: `OVERWRITEHOST` と `trusted_proxies` の両方が必要

## 次へ

次: [8. WireGuard の導入とクライアント配布](08-wireguard.md)
