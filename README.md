# mcp-file-agent

オンプレミス仮想化基盤（Proxmox VE）上で稼働する Nextcloud を中核とし、ローカル LLM（`llama.cpp`）および LibreChat の Model Context Protocol (MCP) を統合した、学術データ自律仕分け・音声文字起こし・講義ノート生成インフラです[cite: 10, 13]。

---

## 1. システム概要 (Overview)

本リポジトリは、以下の2つの自律パイプラインを密結合して運用するための統合バックエンドです[cite: 10, 13]。

1. **Nextcloud FastMCP Agent (閲覧・検索・仕分け基盤):**[cite: 10, 13]
   * LibreChat の MCP クライアント機能から、WebDAV 経由で安全に Nextcloud 上の学術資料や講義ノートを横断検索・参照する FastMCP (SSE) サーバー[cite: 10, 13]。
   * アップロードされた資料のファイル名から科目・回数をローカル LLM が自律推論し、該当ディレクトリへ自動分類・移動する受動的ワーカー[cite: 10]。
2. **講義音声 文字起こし & 構造化要約パイプライン (推論バッチ基盤):**[cite: 10, 13]
   * スマホから Nextcloud の受信トレイ（`00_Inbox`）へ投入された音声ファイルを検知[cite: 10, 13]。
   * オンプレミスの Xeon 計算基盤（Cascade Lake）上で CPU 推論（`faster-whisper` + `llama.cpp`）を実行し、文字起こし全文と試験対策用 Markdown 講義ノートを自律生成して Nextcloud へ再配置[cite: 10, 13]。

---

## 2. システムアーキテクチャ (Architecture)

物理ホスト（Proxmox VE）内の VM / LXC コンテナ間を Tailscale メッシュネットワークで結び、宅内 LAN の IP 変動や瞬断に影響されない閉域通信を確立しています[cite: 10, 13]。

```text
======================== [ オンプレミス物理ホスト (Proxmox VE) ] ========================
       │
       ├── [ ArcLight (Ubuntu VM) ] ── 推論ノード
       │    ├── CPU/RAM: デュアル Intel Xeon Gold（AVX-512 VNNI 最適化）[cite: 10]
       │    ├── llama-server (Qwen2.5-32B, コンテキスト長 32k)[cite: 10]
       │    ├── llama-proxy.service (FastAPI 動的プロキシ / ポート 8000)[cite: 10]
       │    └── lecture-pipeline (systemd timer 巡回 / faster-whisper large-v3)[cite: 10]
       │
       ├── [ NextcloudPi (Debian LXC) ] ── ストレージ & MCP ハブ
       │    ├── Nextcloud WebDAV API (ポート 443 / remote.php/dav/files/<user>/)[cite: 10]
       │    ├── nextcloud-mcp.service (FastMCP SSE サーバー / ポート 8000)[cite: 10]
       │    └── sorter_worker.py (LLM 連携受動的ファイル自動仕分け)[cite: 10]
       │
       └── [ LibreChat (Docker LXC) ] ── オーケストレーション UI
            └── mcpServers 連携 (Nextcloud 内のノートを対話形式で参照・RAG 活用)[cite: 10]
========================================================================================