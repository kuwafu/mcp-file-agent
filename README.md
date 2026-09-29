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
