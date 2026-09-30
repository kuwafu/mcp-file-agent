# mcp-file-agent

FastMCPとWebDAV APIを活用し、Nextcloud上のファイル操作・自律仕分けを行うLLM連携エージェント基盤。

---

## 概要

講義資料やPDF等のドキュメントを、Proxmox上の自前Nextcloudストレージへ自動分類・配置するためのインフラおよびMCP（Model Context Protocol）エージェント構成です。
ローカルLLMおよびLibreChatからの自律的なツール呼び出し（Tool Calling）により、安全かつ高速なファイル操作を実現します。

---

## アーキテクチャ

* **ストレージ基盤:** Nextcloud (WebDAV API)
* **エージェント・プロトコル:** FastMCP (Python, SSEトランスポート)
* **UI / クライアント:** LibreChat
* **推論バックエンド:** llama.cpp (OpenAI互換API, Qwen2.5 / DeepSeek-V2)
* **ネットワーク:** Proxmox VE (VM/LXC分離), Tailscaleメッシュネットワーク (MagicDNS)

```text
[LibreChat (LXC)]
│
▼ (SSE / HTTP)
[FastMCP Server (LXC)]
│
▼ (WebDAV API)
[Nextcloud (VM / Tailscale)]

```
---

## 主な機能と設計方針

1. **セキュアな権限分離 (Read-Onlyの担保)**
   * MCPサーバー（LLMが叩く窓口）には閲覧専用のNextcloudアプリパスワードを付与し、LLMの誤動作による破壊的変更を物理的に防止。
   * ファイル移動・整理を実行するバッチ（`sorter_worker`）のみに書き込み権限を付与する二重防御構造を採用。

2. **堅牢なWebDAVパイプライン**
   * 日本語ファイル名・階層パスの `urllib.parse.quote` による適切なURLエンコード。
   * Nextcloud WebDAVのヘッダー制限（RFC 4918準拠）に対応した、確実なXMLパース（`PROPFIND`）および `MOVE` 処理。

3. **常時稼働デーモン化**
   * FastMCPサーバーは `systemd` サービスとしてデーモン化。
   * 仮想環境（`venv`）を明示した起動構成により、OS依存のバージョン衝突を回避。

## 講義音声 自動文字起こし & 構造化要約パイプライン

スマホからNextcloudへ講義音声（AAC）をアップロードするだけで、オンプレミスのXeon計算基盤上で文字起こし（CTranslate2 / faster-whisper）とLLM要約（llama.cpp / Qwen2.5）を自律実行し、講義ノートをWebDAV経由で自動配置するパイプライン。

### 1. 全体データフロー

```text
[ スマホ (Nothing Phone / Nextcloud App) ]
         │ (.aac 音声アップロード)
         ▼
[ NextcloudPi (LXC) : /00_Inbox ]
         │
         │ (WebDAV MOVE: 01_Processing へ排他隔離)
         ▼
[ ArcLight (Ubuntu VM) ]
   ├── systemd timer (OnUnitInactiveSec=5min 定期巡回・排他制御)
   ├── ffmpeg (生AAC/ADTS パケットサニタイズ -> 16kHz mono WAV 変換)
   ├── faster-whisper / CTranslate2 (large-v3 / CPU int8 / AVX-512 VNNI / beam_size=1)
   └── llama-server / llama.cpp (Qwen2.5 GGUF / /v1/chat/completions 要約生成)
         │
         │ (WebDAV PUT: Markdown要約 & 全文テキスト)
         ▼
[ NextcloudPi (LXC) : /講義ノート/<科目名>/第<XX>回/ ]
   ├── 講義ノート_要約.md
   └── 文字起こし_全文.txt
         │
         │ (WebDAV MOVE: 元音声の退避)
         ▼
[ NextcloudPi (LXC) : /99_Archive/<科目名>/<元音声>.aac ]
```
