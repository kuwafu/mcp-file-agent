# Nextcloud MCP & 自動仕分け連携仕様書

LibreChatから安全に講義資料を参照・検索するためのFastMCPサーバー基盤、およびローカルLLMを用いた受動的ファイル自動仕分けパイプラインの設計仕様。

---

## 1. アーキテクチャ方針と設計思想

* **パイプライン型アーキテクチャの徹底:**
  自律型エージェント（ReActループ）は推論回数増大によるCPU帯域圧迫やファイル操作のハルシネーション（誤削除・誤移動）リスクがあるため不採用。制御フローはPythonで決定論的に固定し、曖昧な自然言語分類のみをローカルLLMに委譲。
* **WebDAV APIによるメタデータ整合性担保:**
  NextcloudのDBメタデータ整合性を保護するため、ストレージ実体（ローカルパス）の直接操作を禁止。ファイル操作はすべて公式WebDAV API経由に統一。
* **権限の疎結合分離:**
  「裏方の自動仕分け（バッチ・書き込み権限）」と「LibreChat連携（リアルタイム対話・閲覧専用権限）」を分離し、LLM経由の破壊的操作を物理的に排除。

---

## 2. コンポーネント構成

```text
[ LibreChat (LXC) ] ──(SSE / ポート8000)──> [ FastMCP Server (NextcloudPi) ]
        │                                                │
        │ (OpenAI互換 API)                               │ (WebDAV API)
        ▼                                                ▼
[ ArcLight (llama-server) ]                    [ Nextcloud ストレージ ]
        ▲                                                ▲
        │ (分類プロンプト)                                │ (MOVE / PROPFIND)
        └────────── [ 自動仕分けワーカー (sorter_worker.py) ]
```

---

## 3. 実装・設定ファイル

### ① 共通WebDAV通信モジュール (`src/nc_client.py`)
`requests` のみで構成した軽量WebDAVクライアント。WebDAVの `Destination` ヘッダー転送におけるlatin-1制限を回避するため、日本語パスを `quote(..., safe="/")` でURLエンコード処理。

### ② 自動仕分けワーカー (`sorter_worker.py`)
* **配置先:** NextcloudPi `~/ncp-automation/sorter_worker.py`
* **機能:** `/_Inbox` をスキャンし、ファイル名から科目名および講義回を判定。`講義ノート/[科目名]/第[XX]回/` へ自律移動。
* **分類プロンプト:** `response_format={"type": "json_object"}` を指定し、厳密なJSON構造を出力。チャット推論との競合に耐えるよう `timeout=300` を設定。

### ③ MCPサーバー実装 (`mcp_server.py`)
* **配置先:** NextcloudPi `~/ncp-automation/mcp_server.py`
* **機能:** FastMCP（1系固定）を用いたSSEトランスポート（`0.0.0.0:8000`）。パストラバーサル防止ガードを組み込んだ閲覧専用ツールを提供。
  * `list_course_notes(subpath)`: 講義フォルダ・ファイル一覧取得。
  * `read_course_note(relative_file_path)`: 講義ノート・要約Markdownの読み出し。

### ④ LibreChat設定 (`librechat.yaml`)
SSRF保護回避のため、`mcpSettings.allowedAddresses` にTailscale IPを明示登録。

```yaml
version: 1.1.5

endpoints:
  custom:
    - name: "ArcLight Local (Xeon 6242)"
      apiKey: "dummy"
      baseURL: "http://<arclight-local-ip>:8080/v1"
      models:
        default: ["Qwen2.5-7B-Instruct-Q4_K_M.gguf"]
        fetch: true
      titleConvo: true

mcpSettings:
  allowedAddresses:
    - "<nextcloud-tailscale-ip>:8000"
    - "<nextcloud-local-ip>:8000"
  allowedDomains:
    - "<nextcloud-tailscale-ip>"
    - "<nextcloud-local-ip>"

mcpServers:
  nextcloud-notes:
    type: sse
    url: http://<nextcloud-tailscale-ip>:8000/sse
```

### ⑤ systemd サービス定義 (`/etc/systemd/system/nextcloud-mcp.service`)
SSH切断後のゾンビプロセス化を防止し、バックグラウンドで永続常駐稼働。

```ini
[Unit]
Description=Nextcloud Academic Notes MCP Server
After=network.target tailscaled.service

[Service]
Type=simple
User=root
WorkingDirectory=/root/ncp-automation
ExecStart=/root/ncp-automation/venv/bin/python3 /root/ncp-automation/mcp_server.py
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

---

## 4. トラブルシューティング知見

| 現象・エラー | 根本原因 | 解決策 |
| :--- | :--- | :--- |
| WebDAV MOVE 時にクラッシュ | HTTPヘッダー（`Destination`）に日本語文字列を直接渡した（latin-1制限） | `quote(path, safe="/")` でURLエンコード処理を実装 |
| FastMCP 構文・仕様エラー | `mcp 2.x` の仕様破壊、`run()` 引数不正、ループバック拘束 | `pip install 'mcp<2'` で固定し、`host="0.0.0.0", port=8000` を明示 |
| `Domain is not allowed` | LibreChatのSSRF防止セキュリティによるローカルIP遮断 | `librechat.yaml` の `mcpSettings.allowedAddresses` にTailscale IPを登録 |
| `EHOSTUNREACH`（LAN接続断） | NextcloudPi側のローカル `eth0` インターフェースDOWN・IP消失 | 通信経路をTailscaleメッシュIPへ完全移行 |
| LLMがツールを実行せずコード捏造 | `llama-server` に `--jinja` がなく、tools引数がドロップ | サーバー起動オプションに `--jinja` を追加 |
| `[Errno 98] Address already in use` | SSH切断に伴う手動プロセスのゾンビ残存 | `fuser -k 8000/tcp` で解放後、systemdデーモンへ移行 |