# やりたいこと（バックログ）

<!--
使い方:
- 思いついたら Claude に伝えれば、ここに追記されます（優先度付け・状態変更・完了もお任せ）。
- 状態の見出し(##)は「今やる / 近いうち / いつか / 完了」の4つ。
- 各項目は「### タイトル」の下に 優先度(高/中/低) / 背景 / 次の一歩 を書く。
- ダッシュボード dashboard.home.lan の「やりたいこと」ページに自動表示される。
-->

## 今やる

## 近いうち

### バイブコーディングをマルチエージェント開発に進化させる
- 優先度: 中
- 背景: 今は技術スタックや実装方法まで指定するスタイル。ゴールだけ伝えて、プランナー/エンジニア/デザイナー的なエージェントに分担させて自律開発する手法を試したい。
- 次の一歩: Claude Code のサブエージェント／Workflow(マルチエージェント・オーケストレーション)／Plan→実装の分業を活用。小さめの題材(下の過去問サービスMVPが好適)で「ゴールだけ渡す→プラン→実装→レビュー」を1回通してみる。

### Obsidian をセルフホスティングする
- 優先度: 中
- 背景: ノート/ナレッジを自前管理したい。クラウド任せにせず homenas でホストする。
- 次の一歩: 2026-07-02 以降に着手。まず方式を確定（端末間同期=Self-hosted LiveSync(CouchDB) / ブラウザ閲覧用 Web フロント / 単なる vault 同期・バックアップ のどれか）→ docker で構築 → 公開せず Tailscale 等の内部アクセス前提。

## いつか

### Mermaid 構成図の見た目を調整する
- 優先度: 低
- 背景: dashboard.home.lan/infra の構成図、初期サイズ・横長レイアウト・文字/配色などの見た目をもう少し良くしたい（一旦は「表示＆ズームできればOK」とした）。
- 次の一歩: homelab.mmd の構造見直し（横長→縦方向に寄せる/サブグラフ整理）、Mermaid テーマ微調整、初期ズーム/フィットの調整。

### ターミナルを“オタク環境”にする（nvim / yazi / starship 等）
- 優先度: 低
- 背景: ターミナルを“テンションの上がる見た目”にしたい。Ghostty・WezTerm を試したが日本語IME(変換)が弱く断念し、iTerm2 に定着。emulator は iTerm2 のままで、中身(nvim/yazi/starship/配色テーマ)で作り込む方針（中身はターミナル非依存）。
- 次の一歩: やる気が出たら nvim(LazyVim 等)から段階的に。Mac側の Claude Code(導入済) で進めるのが楽。

### 国家資格の過去問出題Webサービス（広告収入）
- 優先度: 中
- 背景: 国家資格の受験者向けに過去問を出題する Web サービスを作りたい。広告収入が目的。
- 次の一歩: ①対象資格を1つ決める(過去問の入手性・受験者数・競合で判断) ②MVP を定義(出題→解答→正誤→解説のループ) ③スタック(既存の Next.js + Vercel 構成を流用可)。まず「資格1つ＋問題データの入手方法」の確定が最初の関門。※上のマルチエージェント開発の最初の題材に最適。

### Mac の Homebrew を arm64(/opt/homebrew) に移行
- 優先度: 低
- 背景: Mac(Apple Silicon)の Homebrew が x64(/usr/local・Rosetta)のまま。ネイティブ拡張を持つツールが arm64 を取れず壊れる原因(Claude Code 導入時に発覚)。当面は個別にネイティブ導入で回避できるが、根治には arm64 brew への移行が必要。
- 次の一歩: 現状の `brew list` を棚卸し → /opt/homebrew に arm64 Homebrew を導入 → パッケージを arm64 で入れ直し → PATH を /opt/homebrew 優先に → x64 brew を撤去。既存ツール(gh/rclone 等)に影響するので計画的に。

### 健康な外付けHDDでクロスドライブ・バックアップへ再移行
- 優先度: 中
- 背景: 現在バックアップは内蔵HDD内のみ（同一ドライブ）で、HDD故障時に元データと共倒れのリスク。別筐体に退避してクロスドライブ保護にしたい。前回用意した外付けはSMART故障で断念。
- 次の一歩: 健康な外付け（購入後にまずSMART確認）を入手 → 既定手順（mkfs lazy init → OMVマウント → homenas-backup.sh の DEST_ROOT 差し替え → ダッシュボードの BACKUPS_PATH）で移行

## 完了

### Tailscale を導入して VPN を快適化（非公開アクセスの土台）
- 優先度: 中
- 背景: 外出先から Immich/Nextcloud アプリ・dashboard に繋ぎたい＆WireGuardのフルトンネルで帰宅時にネットが切れる問題を解消したかった。
- 次の一歩: 完了（2026-06-27）。サーバ＋スマホ/PC に Tailscale 導入(squib02@gmail.com)。スプリットトンネルで常時ONでも通常ネットOK・外出先から接続OKを実証。Tailscale DNS=AdGuard(100.72.254.73)に「Override local DNS」→全端末で広告ブロック＆内部名解決(バイパスは Tailscale OFF)。AdGuardに `*.obukata.uk`→100.72.254.73 のrewrite、NPMで `*.obukata.uk` のLet's Encryptワイルドカード証明書(CloudflareトークンでDNS-01)＋Proxy Host(immich/nextcloud/dashboard.obukata.uk)。Nextcloudは override.yml の OVERWRITEHOST 削除で多ドメイン対応。アプリは `https://〜.obukata.uk` で設定・動作確認済み。subnet router / WireGuard撤去 は任意で後日。

### Cloudflare Tunnel で自宅サーバを公開（セキュリティ最優先）
- 優先度: 中
- 背景: ポート開放せず外部公開＋ガッチガチ認証をやりたかった。
- 次の一歩: 完了（2026-06-26）。独自ドメイン **obukata.uk** を Cloudflare で取得、Zero Trust チーム `obukata`、トンネル `homelab` を作成、cloudflared を Docker(proxy-net)で起動、**dash.obukata.uk → newdash** を公開し **Cloudflare Access(Allow: squib02@gmail.com のみ)** で施錠まで検証OK。ただし方針転換：Immich/Nextcloud のアプリは Access で壊れる＆管理系は非公開が安全なので、当面は **VPN(Tailscale)前提の非公開運用**に。Cloudflare/ドメインは将来の公開サービス用に温存（cloudflared コネクタは稼働したまま＝いつでも再利用可）。

### スマホで仕事できる体制を作る
- 優先度: 中
- 背景: スマホ(Termius)→ホームサーバ→tmux でこのセッション、という流れ。権限コマンドの実行・コピペが大変で、内容確認もできずブルシット感が強かった。
- 次の一歩: 完了（2026-06-25）。sudoスコープ化で権限コマンドは Claude が直接実行(手打ち/確認の手間ゼロ)、tmuxコピーは OSC52 整備、dashboard.home.lan も iPad から閲覧可(Wi-Fi の DNS を 192.168.3.19 に手動設定)。日本語IMEは Ghostty/WezTerm 等を検討したが Termius が最もマシと確定し許容。

### tmux のコピーモードを直す（キーボードで選択コピー）
- 優先度: 高
- 背景: tmux 上の文字をうまくコピーできない。Shift＋ドラッグ頼みで、キーボードでの選択コピーが効かなかった。スマホ作業の土台にもなる。
- 次の一歩: 完了（2026-06-25）。~/.tmux.conf に mode-keys vi / mouse on / set-clipboard on / terminal-features clipboard / v・y・C-v バインドを設定。SSH越しのMacクリップボードは Warp が OSC52 非対応でNGだったため Ghostty に変更して解決（サーバに xterm-ghostty terminfo も導入）。

### サーバのSMART監視をSlack通知化
- 優先度: 中
- 背景: ドライブ故障を事前に察知したい（外付けが壊れていた件の再発防止）
- 次の一歩: 完了（smartd + smart-slack.sh、2026-06-24 稼働）
