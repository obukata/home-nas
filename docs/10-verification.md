# 10. 各クライアントからの接続確認

前: [9. バックアップ設定](09-backup.md)

## 目的

構築した家庭内 NAS が、想定した全クライアント (Mac / Windows / Android / iOS) から、LAN 内と VPN 経由の両方で期待通りに動作することを確認する。

## 前提

- [9. バックアップ設定](09-backup.md) までが完了している
- 各クライアント端末が手元にある
- Nextcloud の夫婦 2 アカウント (自分・配偶者) が作成済み

## 確認項目

### 10-1. LAN 内: Nextcloud Web UI (Mac / Win)

- ブラウザで `https://nextcloud.home.lan/`
- 自己署名の警告 → 例外許可 (CA 配布する場合はそれで解決)
- 自分のアカウントでログイン
- ファイルのアップロード・ダウンロード
- `Family` / `Work` が共有フォルダとして見える
- 配偶者のアカウントで別ブラウザ (またはプライベートウィンドウ) からログインし、同じ `Family` / `Work` が見えることを確認
- 片方で入れたファイルがもう片方で見えることを確認

### 10-2. Nextcloud デスクトップクライアント (Mac)

1. <https://nextcloud.com/install/#install-clients> から Mac 版をダウンロード
2. サーバ URL に `https://nextcloud.home.lan`
3. 自己署名を信頼する (初回のみ)
4. 自分のアカウントでログイン
5. ローカルの同期フォルダを選択 (例: `~/Nextcloud`)
6. 初回同期完了まで待つ (Family / Work / 個人フォルダがローカルに展開される)
7. Finder で `~/Nextcloud/Family/` にファイルを入れる → 配偶者側の同期にも反映されることを確認 (配偶者の Mac/PC にクライアントが入っていればそちらにも降りる)
8. サーバ側 (Web UI) で 1 ファイル作成 → Mac の Finder に降りてくることを確認

### 10-3. Nextcloud デスクトップクライアント (Windows)

- 同上の Windows 版
- サーバ URL / 自己署名信頼 / ローカルフォルダ設定
- 双方向同期の確認

### 10-4. Nextcloud Android アプリ

1. Google Play から公式 Nextcloud アプリをインストール
2. サーバ URL `https://nextcloud.home.lan`
3. 自己署名の信頼 (アプリ内ダイアログ)
4. 自分のアカウントでログイン
5. **写真自動アップロードを有効化**:
   - `Settings → Auto upload`
   - アップロード先フォルダを `/Photos` (自分の個人フォルダ内) 等に設定
   - Wi-Fi 時のみ / 充電中のみ等の条件を好みで
6. カメラで 1 枚撮影 → Nextcloud サーバに上がることを確認

### 10-5. Nextcloud iOS アプリ (該当時)

- App Store から公式 Nextcloud アプリ
- 同様の手順で設定、写真自動アップロードを確認

### 10-6. VPN 経由 (スマホ)

- スマホを **モバイル回線に切り替え** (Wi-Fi を OFF)
- WireGuard アプリでトンネル ON
- Nextcloud アプリで継続アクセス可能か確認
- ブラウザでも `https://nextcloud.home.lan` を開けること
- 1 枚写真を撮影 → 自動アップロードが VPN 経由で走ることを確認

### 10-7. VPN 経由 (Mac)

- Wi-Fi でテザリング等、家の LAN ではないネットワークに接続
- Mac 版 WireGuard アプリでトンネル ON
- `https://nextcloud.home.lan` がブラウザで開く
- Nextcloud Desktop Client が VPN 経由で同期を継続できる (しばらく待つと新規ファイルが降りる/上がる)

### 10-8. バックアップの初回実行確認

- `ls <HDD_ROOT>/shares/backups/`
- `snapshot-YYYYMMDD-HHMMSS` ディレクトリが翌朝までに作成されていること
- `/var/log/homenas-backup.log` にエラーがないこと

### 10-9. セキュリティ確認

- 外部から `https://<ルータのグローバル IP>/` や `http://<グローバル IP>/` にアクセスしても **何も応答しない** こと (VPN 以外は閉じている)
- `nmap -p- <グローバル IP>` を外部から実行して、開いているのが UDP 51820 のみであることを確認 (UDP スキャンは tcp より時間がかかる)

## 最終チェックリスト

- [ ] 自分のアカウントと配偶者アカウントの両方で Nextcloud Web にログインできる
- [ ] `Family` / `Work` が両アカウントから見える (共有が正しく設定されている)
- [ ] Mac/Win デスクトップクライアントで双方向同期 OK
- [ ] Android/iOS アプリで写真自動アップロード OK
- [ ] スマホの VPN 経由アクセス OK
- [ ] Mac の VPN 経由アクセス OK
- [ ] 日次バックアップが生成されている
- [ ] 外部からは UDP 51820 以外アクセス不可

## トラブルシュート

- **VPN 接続しているのに名前解決できない**: DNS 設定を見直し ([08-wireguard.md](08-wireguard.md) の 8-6 / [06-adguard-home.md](06-adguard-home.md) の 6-5 参照)
- **Android で自動アップロードが走らない**: 電池最適化から Nextcloud を除外。バックグラウンド制限を解除
- **共有フォルダが配偶者側に現れない**: Nextcloud の Web UI で共有設定を再確認。ユーザー名のスペルミスに注意。`occ files:scan --all` で再スキャン

## 完了

ここまで通れば家庭用ファイルサーバの基本運用は完成。

運用開始後に検討するもの:

- 外付け HDD 追加とバックアップ宛先移行
- クラウドへの二次バックアップ
- Nextcloud のチューニング (プレビュー生成、フルテキスト検索)
- 監視 / 通知 (Uptime Kuma 等)
- OS とコンテナの定期アップデート運用
- SMB を追加したくなったら [docs/03-shared-folders-smb.md](03-shared-folders-smb.md) 末尾の「SMB/CIFS サービスは有効化しない」節の指示に従って後付け有効化
- Slack / メール / カレンダー 等の外部サービスと Nextcloud を連携させたくなったら [11. (オプション) n8n で家庭用自動化ハブを立てる](11-n8n.md)

前: [9. バックアップ設定](09-backup.md) / 次: [11. (オプション) n8n で家庭用自動化ハブを立てる](11-n8n.md) / 戻る: [README](../README.md)
