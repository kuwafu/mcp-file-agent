
# Nextcloud × LibreChat × llama.cpp 自宅インフラ自動化・連携仕様書



---

## 1. プロジェクト概要・背景



* **背景・目的：** 往復3時間の通学およびアルバイトによる可処分時間の枯渇に対応するため、Proxmox VE上に大学講義資料の整理・参照を半自動化するパイプラインを構築する。


* **基本アーキテクチャ方針：**
* 自律型エージェント（ReActループ）は推論回数増大によるCPU帯域圧迫やファイル操作のハルシネーションリスクがあるため不採用。


* 制御フローはPythonで決定論的に固定し、曖昧な自然言語分類のみをローカルLLMに委譲する「パイプライン型」を採用。


* NextcloudのDBメタデータ整合性を担保するため、ストレージ実体の直接操作を禁止し、入出力はすべて公式WebDAV API経由に統一。


* 「裏方の自動仕分け（バッチ・書き込み権限）」と「LibreChat連携（リアルタイム対話・閲覧専用権限）」を疎結合に分離。





---

## 2. インフラ環境・ネットワーク構成



### ノードおよびコンテナ構成



* **推論サーバー（ArcLight / Ubuntu VM）：**
* デュアルIntel Xeon（AVX-512、vNUMA最適化）。


* `model_proxy.py` 経由で `llama-server`（ポート8080）を単一インスタンス・FIFO直列キューイングで稼働。


* モデル：`Qwen2.5-7B-Instruct-Q4_K_M.gguf`（CPUメモリ帯域およびTool Calling精度を両立）。




* **ストレージ・同期ハブ（NextcloudPi / NCP LXC）：**
* 独立したLXCコンテナで稼働。


* WebDAVエンドポイント：`/remote.php/dav/files/ncp/`。


* FastMCPベースのMCPサーバー（ポート8000）を常駐。




* **クライアント・UI（LibreChat LXC）：**
* Dockerコンテナ群（`api` サービス等）として稼働。


* OpenAI互換エンドポイントとしてArcLightを参照し、MCP経由でNextcloudPiと連携。





### ネットワーク仕様



* **Tailscaleメッシュネットワークへの固定：**
* LANインターフェース（DHCP）の不安定化を回避するため、通信経路をTailscaleに一本化。


* NextcloudPi IP：`100.82.15.31`。


* NextcloudPi MagicDNS（FQDN）：`nextcloudpi.taildeb88d.ts.net`（`trusted_domains` 登録済み）。





---

## 3. 実装コード・設定ファイル一覧



### 3.1 共通WebDAV通信モジュール（`nc_client.py`）

配置先：NextcloudPi `~/ncp-automation/nc_client.py`

* 依存関係を `requests` のみに絞り込み。


* 自己署名SSLの検証無効化（`verify=False`）。


* WebDAVの `Destination` ヘッダー転送におけるlatin-1制限を回避するため、日本語パスを `urllib.parse.quote(..., safe="/")` でURLエンコード処理。



```python
import xml.etree.ElementTree as ET
from urllib.parse import quote, unquote, urlparse
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class NextcloudWebDAV:
    def __init__(self, base_url: str, user: str, app_password: str):
        self.base_url = base_url.rstrip("/")
        self.user = user
        self.webdav_root = f"{self.base_url}/remote.php/dav/files/{self.user}"
        self.auth = (user, app_password)

    def _url(self, path: str) -> str:
        clean_path = "/" + path.strip("/") if path.strip("/") else ""
        quoted_path = quote(clean_path, safe="/")
        return f"{self.webdav_root}{quoted_path}"

    def list_dir(self, path: str = "/") -> list[str]:
        url = self._url(path) + "/"
        res = requests.request("PROPFIND", url, auth=self.auth, headers={"Depth": "1"}, verify=False)
        if res.status_code != 207:
            return []
        root = ET.fromstring(res.text)
        items = []
        target_path = unquote(urlparse(url).path).rstrip("/") + "/"
        for elem in root.findall(".//{DAV:}response"):
            href = unquote(elem.find("{DAV:}href").text)
            clean_href = href.rstrip("/")
            if clean_href == target_path.rstrip("/"):
                continue
            item_name = clean_href.split("/")[-1]
            if item_name:
                items.append(item_name)
        return items

    def mkdir_p(self, path: str) -> bool:
        parts = [p for p in path.strip("/").split("/") if p]
        cur = ""
        for p in parts:
            cur += "/" + p
            res = requests.request("MKCOL", self._url(cur), auth=self.auth, verify=False)
            if res.status_code not in [201, 405]:
                return False
        return True

    def move(self, src_path: str, dest_path: str, overwrite: bool = False) -> bool:
        src_url = self._url(src_path)
        dest_url = self._url(dest_path)
        headers = {
            "Destination": dest_url,
            "Overwrite": "T" if overwrite else "F"
        }
        res = requests.request("MOVE", src_url, auth=self.auth, headers=headers, verify=False)
        return res.status_code in [201, 204]

    def read_text(self, path: str) -> str:
        res = requests.get(self._url(path), auth=self.auth, verify=False)
        res.raise_for_status()
        return res.text

    def write_text(self, path: str, content: str) -> bool:
        res = requests.put(self._url(path), data=content.encode("utf-8"), auth=self.auth, verify=False)
        return res.status_code in [201, 204]

```

---

### 3.2 自動仕分けワーカー（`sorter_worker.py`）

配置先：NextcloudPi `~/ncp-automation/sorter_worker.py`

* `_Inbox` をスキャンし、LLMに科目・講義回を判定させ `大学/2026_秋/[科目名]/第[XX]回/` へ移動。


* チャット推論との競合でタイムアウト死しないよう `timeout=300` を設定。



```python
import os
import json
import requests
from nc_client import NextcloudWebDAV

NCP_HOST = "https://nextcloudpi.taildeb88d.ts.net"
NCP_USER = "ncp"
NCP_PASS = "kTiZe-FB3XS-cLdLN-rXijZ-rccWG"

LLM_HOST = "http://192.168.0.158:8080"
LLM_API_URL = f"{LLM_HOST}/v1/chat/completions"

INBOX_DIR = "/_Inbox"
TARGET_ROOT = "/大学/2026_秋"

SUBJECTS = [
    "細胞組織学",
    "遺伝子工学Ⅰ",
    "生物分子科学実験Ⅳ",
    "生物分子科学実験Ⅴ",
    "物理化学Ⅰ",
    "その他"
]

def classify_file(filename: str) -> dict:
    system_prompt = f"""あなたは大学の講義資料を分類する整理エンジンです。
入力されたファイル名から該当する「科目名」と「講義の第何回か」を推測し、必ず以下のJSON形式のみで回答してください。前置きや解説は一切出力しないでください。
【候補科目】
{", ".join(SUBJECTS)}
【JSONフォーマット】
{{"subject": "科目名", "week": 整数（不明な場合は0）}}"""

    payload = {
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"ファイル名: {filename}"}
        ],
        "temperature": 0.1,
        "response_format": {"type": "json_object"}
    }

    res = requests.post(LLM_API_URL, json=payload, timeout=300)
    res.raise_for_status()
    content = res.json()["choices"][0]["message"]["content"]
    return json.loads(content)

def process_inbox(nc: NextcloudWebDAV):
    files = nc.list_dir(INBOX_DIR)
    if not files:
        return

    for filename in files:
        try:
            result = classify_file(filename)
            subject = result.get("subject", "その他")
            week = int(result.get("week", 0))

            dest_dir = f"{TARGET_ROOT}/{subject}/第{week:02d}回" if week > 0 else f"{TARGET_ROOT}/{subject}"
            nc.mkdir_p(dest_dir)
            nc.move(f"{INBOX_DIR}/{filename}", f"{dest_dir}/{filename}")
        except Exception as e:
            print(f"[ERROR] {filename} skip: {e}")

if __name__ == "__main__":
    nc = NextcloudWebDAV(NCP_HOST, NCP_USER, NCP_PASS)
    process_inbox(nc)

```

---

### 3.3 ArcLight 推論プロキシ起動コマンド（`/home/wtf/model_proxy.py`）

`llama-server` 起動ブロックの修正内容：

* 過去のRPC分散並列オプション（`--rpc`, `-ts` 等）を排除し、単一スタンドアロン構成へ単純化。


* OpenAI互換Tool Calling用のJinja2テンプレートを有効化するため `"--jinja"` フラグを追加。



```python
    cmd = [
        LLAMA_BIN,
        "-m", str(model_path),
        "--host", "127.0.0.1",
        "--port", str(INTERNAL_PORT),
        "-c", "16384",
        "-t", "16",
        "--jinja"
    ]
    server_process = subprocess.Popen(cmd)

```

---

### 3.4 MCPサーバー実装（`mcp_server.py`）

配置先：NextcloudPi `~/ncp-automation/mcp_server.py`

* FastMCP（1系固定）を用いたSSEトランスポート（`0.0.0.0:8000`）。


* 外部からの不正なパストラバーサルを弾くバリデーションを実装。



```python
import os
from mcp.server.fastmcp import FastMCP
from nc_client import NextcloudWebDAV

NCP_HOST = "https://nextcloudpi.taildeb88d.ts.net"
NCP_USER = "ncp"
NCP_PASS = "kTiZe-FB3XS-cLdLN-rXijZ-rccWG"

mcp = FastMCP("Nextcloud-Academic-Notes", host="0.0.0.0", port=8000)
nc = NextcloudWebDAV(NCP_HOST, NCP_USER, NCP_PASS)

TARGET_BASE = "/大学/2026_秋"

@mcp.tool()
def list_course_notes(subpath: str = "") -> list[str]:
    """Nextcloud上の秋学期講義フォルダやファイルの一覧を取得します。
    引数 subpath: '細胞組織学' や '細胞組織学/第01回' などの相対パス。空なら講義一覧を返します。
    """
    clean_sub = subpath.strip("/")
    target = f"{TARGET_BASE}/{clean_sub}" if clean_sub else TARGET_BASE
    return nc.list_dir(target)

@mcp.tool()
def read_course_note(relative_file_path: str) -> str:
    """指定された講義ノートや要約ファイルの中身を読み出します。
    引数 relative_file_path 例: '細胞組織学/第01回/要約.md'
    """
    clean_path = relative_file_path.strip("/")
    target = f"{TARGET_BASE}/{clean_path}"

    if not clean_path or ".." in clean_path:
        return "エラー: 不正なパス指定です。"

    try:
        return nc.read_text(target)
    except Exception as e:
        return f"ファイル読み出し失敗: {e}"

if __name__ == "__main__":
    mcp.run(transport="sse")

```

---

### 3.5 LibreChat設定（`/root/LibreChat/librechat.yaml`）

* SSRF保護回避のため `mcpSettings.allowedAddresses` にTailscale IPを登録。



```yaml
version: 1.1.5

endpoints:
  custom:
    - name: "ArcLight Local (Xeon 6242)"
      apiKey: "dummy"
      baseURL: "http://192.168.0.158:8080/v1"
      models:
        default: ["Qwen2.5-7B-Instruct-Q4_K_M.gguf"]
        fetch: true
      titleConvo: true

mcpSettings:
  allowedAddresses:
    - "100.82.15.31:8000"
    - "192.168.0.116:8000"
  allowedDomains:
    - "100.82.15.31"
    - "192.168.0.116"

mcpServers:
  nextcloud-notes:
    type: sse
    url: http://100.82.15.31:8000/sse

```

---

### 3.6 MCP常駐化サービス（`/etc/systemd/system/nextcloud-mcp.service`）

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

## 4. 発生した問題と解決プロトコル一覧



| フェーズ | 発生した現象・エラー | 根本原因 | 適用した解決策

 |
| --- | --- | --- | --- |
| **仕分け** | WebDAV MOVE時にクラッシュ | HTTPヘッダー（`Destination`）に日本語文字列を直接渡したため（latin-1制限）

 | `urllib.parse.quote(path, safe="/")` でヘッダー値をURLエンコード

 |
| **MCP構築** | `FastMCP` の構文・仕様エラー | `mcp` 2.x の仕様破壊、`run()` 引数の不正、内部ループバック拘束

 | `pip install 'mcp<2'` でバージョン固定。コンストラクタで `host="0.0.0.0", port=8000` を明示

 |
| **連携設定** | `Domain is not allowed` | LibreChatの内部SSRF防止セキュリティによるプライベートIP遮断

 | `.env` ではなく `librechat.yaml` の `mcpSettings.allowedAddresses` にIPを追加

 |
| **ルーティング** | `EHOSTUNREACH 192.168.0.116` | NextcloudPi側の `eth0` インターフェースがDOWN・消失

 | 接続先をTailscale IP（`100.82.15.31`）およびMagicDNSへ完全移行

 |
| **推論** | LLMがツールを実行せずPHPを捏造 | `llama-server` に `--jinja` オプションがなく、tools引数がドロップされた

 | `model_proxy.py` の起動引数に `"--jinja"` を追加

 |
| **推論** | モデルがテキストで雑談する | `Qwen2-57B-A14B`（無印MoE）のTool Calling追従性不足

 | モデルを `Qwen2.5-7B-Instruct-Q4_K_M.gguf` に変更

 |
| **認証** | SabreDAV `401 NotAuthenticated` | 使用していたアプリパスワードの失効・文字列不一致

 | WebUIから `kTiZe-FB3XS-cLdLN-rXijZ-rccWG` を再発行し設定更新

 |
| **プロセス** | `[Errno 98] Address already in use` | SSH切断に伴う手動実行プロセスのゾンビ残存

 | `fuser -k 8000/tcp` で解放後、systemdデーモン化して永続管理

 |

---

## 5. 現在の到達状況・積み残しタスク



### 完成・稼働中



* [x] PythonによるNextcloud WebDAV API操作基盤（`nc_client.py`）


* [x] ローカルLLM判定によるファイル自動仕分けスクリプト（`sorter_worker.py`）


* [x] ArcLight推論VMのOpenAI互換Tool Calling環境（`--jinja` + Qwen2.5-7B）


* [x] Tailscale MagicDNS経由の閲覧専用MCPサーバー（`mcp_server.py`）


* [x] LibreChat ⇔ MCP ⇔ Nextcloud WebDAV のエンドツーエンド自律ツール呼び出し疎通


* [x] MCPサーバーのsystemd常駐自動起動化（`nextcloud-mcp.service`）



### 未着手・保留・今後のタスク



* [ ] **音声の自動文字起こし：** faster-whisper等を用いた音声バッチ処理の構築。


* [ ] **Markdown要約の自動生成：** 抽出プロンプトを用いた要約生成処理の `sorter_worker.py` への組み込み。


* [ ] **仕分けワーカーの自動巡回化：** `sorter_worker.py` の cron / inotify（フォルダ監視）常駐化。


* [ ] **実戦運用テスト：** 講義ノートを投入し、LibreChatの `read_course_note` 経由で試験対策・質疑応答を行うプロンプト検証。