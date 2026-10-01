# mcp-file-agent

Proxmox上のNextcloudを中心とした、ローカルLLM（llama.cpp）およびLibreChatのMCP機能連携によるファイル操作・講義ノート自動生成インフラ。

---

## 概要

オンプレミス仮想化基盤（Proxmox VE）上で稼働する、以下の2つのパイプラインを統合運用するためのリポジトリです。

1. **Nextcloud MCP Agent:** LibreChat標準のMCPクライアント機能（`mcpServers`）から、WebDAV経由で安全にNextcloud上のファイル操作・仕分けを行うエージェント基盤。
2. **講義音声 文字起こし & 要約パイプライン:** スマホからNextcloudへ手動アップロードされた講義音声をトリガーとし、サーバー側でCPU推論（faster-whisper + llama.cpp）による文字起こしとMarkdown構造化ノート生成を自律実行するシステム。

---

## システムアーキテクチャ

```text
[ スマホ (手動アップロード) ] ──(Nextcloud App)──┐
                                              ▼
            [ Proxmox VE ]
            ├── [Nextcloud (VM/LXC)] <-- ストレージ基盤 (WebDAV API)
            │        ▲
            │        │ (WebDAV API)
            ├── [LibreChat (LXC)]    <-- UI & MCPクライアント基盤
            │    └── [mcpServers (Nextcloud Agent)]
            │             │ (OpenAI互換 API / Tool Calling)
            │             ▼
            └── [ArcLight (Ubuntu VM)] <-- 推論・音声処理基盤
                 ├── llama.cpp (Qwen2.5 / DeepSeek-V2)
                 ├── faster-whisper (AVX-512 VNNI / INT8)
                 └── systemd timer (講義音声自律巡回デーモン)
```

---

## ドキュメント & 詳細仕様

* **[講義音声 自動文字起こし & 構造化要約パイプライン仕様](docs/lecture-pipeline.md)**
  * ステートマシン設計、生AAC破損のffmpegサニタイズ、PyAV前方互換性対策、AVX-512 VNNI × beam_size=1 チューニング、systemd timer排他制御。
* **[Nextcloud MCP 連携仕様](docs/nextcloud-mcp.md)**
  * FastMCP、WebDAVクライアント仕様、LibreChat SSRF回避設定、ファイル自動仕分け。
* **[インフラ・ネットワーク構成仕様](docs/infrastructure.md)**
  * Proxmox VE、Tailscale MagicDNS固定、Xeon CPU/NUMA最適化。

---

## ディレクトリ構成

```text
.
├── README.md
├── docs/
│   ├── infrastructure.md     # Proxmox / ネットワーク構成
│   ├── nextcloud-mcp.md      # FastMCP / WebDAV 連携仕様
│   └── lecture-pipeline.md   # 音声文字起こし・要約パイプライン仕様 & 知見
├── src/                      # スクリプト本体
│   ├── pipeline.py
│   └── nc_client.py
└── systemd/                  # ユニット定義
    ├── lecture-pipeline.service
    └── lecture-pipeline.timer
```