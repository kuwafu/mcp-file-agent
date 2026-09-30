# mcp-file-agent

Proxmox上のNextcloudを中心とした、ローカルLLM（llama.cpp）およびFastMCP連携による自律型ファイル操作・講義ノート生成インフラ。

---

## 概要

オンプレミス環境（Xeon計算基盤）上で、以下の2つの自動化パイプラインを統合運用するためのリポジトリです。

1. **Nextcloud MCP Agent:** LibreChatおよびローカルLLMから、WebDAV経由で安全にファイル操作・仕分けを行うMCPサーバー基盤。
2. **講義音声 自動パイプライン:** スマホから投入された講義音声を検知し、CPU推論（faster-whisper + llama.cpp）により文字起こしと構造化ノート生成を完全自律実行するシステム。

---

## システムアーキテクチャ

```text
[ スマホ / LibreChat ]
        │
        ├── (WebDAV: 講義音声アップロード) ──┐
        └── (SSE: MCPツール呼び出し) ──┐      │
                                     ▼      ▼
                [ Proxmox VE (voyager) ]
                ├── [Nextcloud (VM)]  <-- ストレージ基盤 (WebDAV API)
                ├── [ArcLight (VM)]   <-- 推論・処理基盤
                │    ├── llama.cpp (Qwen2.5 / DeepSeek-V2)
                │    ├── faster-whisper (AVX-512 VNNI / INT8)
                │    └── systemd timer (自律巡回デーモン)
                └── [FastMCP (LXC)]   <-- 権限分離型MCPサーバー
