This file is a merged representation of the entire codebase, combined into a single document by Repomix.

# File Summary

## Purpose
This file contains a packed representation of the entire repository's contents.
It is designed to be easily consumable by AI systems for analysis, code review,
or other automated processes.

## File Format
The content is organized as follows:
1. This summary section
2. Repository information
3. Directory structure
4. Repository files (if enabled)
5. Multiple file entries, each consisting of:
  a. A header with the file path (## File: path/to/file)
  b. The full contents of the file in a code block

## Usage Guidelines
- This file should be treated as read-only. Any changes should be made to the
  original repository files, not this packed version.
- When processing this file, use the file path to distinguish
  between different files in the repository.
- Be aware that this file may contain sensitive information. Handle it with
  the same level of security as you would the original repository.

## Notes
- Some files may have been excluded based on .gitignore rules and Repomix's configuration
- Binary files are not included in this packed representation. Please refer to the Repository Structure section for a complete list of file paths, including binary files
- Files matching patterns in .gitignore are excluded
- Files matching default ignore patterns are excluded
- Files are sorted by Git change count (files with more changes are at the bottom)

# Directory Structure
````
docs/
  infrastructure.md
  lecture-pipeline.md
  nextcloud-mcp.md
src/
  mcp_server.py
  nc_client.py
  pipeline.py
  sorter_worker.py
systemd/
  lecture-pipeline.service
  lecture-pipeline.timer
  nextcloud-mcp.service
.gitignore
LICENSE
README.md
requirements.txt
````

# Files

## File: docs/infrastructure.md
````markdown
# 自宅インフラ & ネットワーク構成仕様書

オンプレミス仮想化基盤（Proxmox VE）上で稼働する、推論基盤・ストレージ基盤・UI基盤のトポロジおよびネットワーク設計。

---

## 1. 物理・仮想化アーキテクチャ

単一の物理サーバーノード上に、役割に応じたVMおよびLXCコンテナを分離展開。

```text
======================== [ 物理ホスト (Proxmox VE) ] ========================
       │
       ├── [ ArcLight (Ubuntu VM) ]
       │    ├── 役割: 推論サーバー（LLM要約 / 音声文字起こし）
       │    ├── CPU/RAM: デュアル Intel Xeon Gold（AVX-512, vNUMA最適化）
       │    ├── ポート 8000: llama-proxy.service (FastAPI動的プロキシ)
       │    └── ポート 8080: llama-server (Qwen2.5-32B, -c 32768)
       │
       ├── [ NextcloudPi (LXC) ]
       │    ├── 役割: 学術データ同期ハブ / WebDAVストレージ / MCPサーバー
       │    ├── ポート 443: Nextcloud WebDAV API (/remote.php/dav/files/<user>/)
       │    └── ポート 8000: nextcloud-mcp.service (FastMCP SSE)
       │
       └── [ LibreChat (LXC) ]
            ├── 役割: クライアントUI & MCPオーケストレーション
            └── 構成: Docker Compose (LibreChat API / MongoDB / Web)
=============================================================================
```

---

## 2. ネットワーク仕様 & ルーティング方針

### ① Tailscale メッシュネットワークへの一本化
* **設計意図:** 宅内LAN（DHCP）のインターフェース瞬断やIP変動によるノード間不通（`EHOSTUNREACH`）を物理的に排除するため、サービス間通信をTailscaleメッシュ網へ固定。
* **MagicDNS活用:** Tailscale FQDN（`<node-name>.<tailnet-name>.ts.net`）をNextcloudの `trusted_domains` に設定し、証明書警告およびIP直打ち依存を解消。

### ② 固定アドレス・エンドポイント設計
* **NextcloudPi:**
  * Tailscale IP: `100.x.x.x`
  * WebDAV Root: `https://<nextcloud-host>.ts.net/remote.php/dav/files/<user>/`
  * FastMCP Endpoint: `http://<nextcloud-tailscale-ip>:8000/sse`
* **ArcLight (推論ノード):**
  * OpenAI互換 API (外部・LibreChat向け): `http://<arclight-local-ip>:8080/v1`
  * 動的プロキシ API (内部・パイプライン向け): `http://127.0.0.1:8000/v1`

---

## 3. 計算リソース & 推論最適化

### ① 音声処理（faster-whisper / CTranslate2）
* **ハードウェアアクセラレーション:** Cascade Lake世代の `AVX-512 VNNI`（INT8積和演算命令）を活用。
* **メモリ帯域バウンド対策:** `large-v3` において `beam_size=1, best_of=1`（Greedy探索）を採用。自己回帰デコーダのメモリアクセス回数を削ぎ落とし、リアルタイム係数（RTF）約0.28を達成。

### ② LLM要約（llama.cpp / llama-proxy）
* **大容量コンテキスト:** 長時間の講義テキスト（3万文字超）を処理するため、コンテキスト長を `-c 32768`（32kトークン）へ拡張。
* **単一インスタンス・FIFO直列化:** `model_proxy.py` によりメモリ競合を防止。複数リクエストをキューイング制御し、CPU帯域を単一推論に全振りする構成を採用。
````

## File: docs/lecture-pipeline.md
````markdown
# 講義音声 自動文字起こし & 構造化要約パイプライン仕様書

スマホからNextcloudへ講義音声（AAC）をアップロードするだけで、オンプレミスのXeon計算基盤上で文字起こし（CTranslate2 / faster-whisper）とLLM要約（llama.cpp / Qwen2.5-32B）を自律実行し、講義ノートをWebDAV経由で自動配置するパイプライン。

---

## 1. 全体データフロー

```text
[ スマホ (手動アップロード) ]
         │ (.aac 音声投入)
         ▼
[ NextcloudPi : /00_Inbox ]
         │
         │ (WebDAV MOVE: 01_Processing へ排他隔離)
         ▼
[ ArcLight (Ubuntu VM) ]
   ├── systemd timer (OnUnitInactiveSec=5min 定期巡回・排他制御)
   ├── ffmpeg (生AAC/ADTS パケットサニタイズ -> 16kHz mono WAV 変換)
   ├── faster-whisper / CTranslate2 (large-v3 / CPU int8 / AVX-512 VNNI / beam_size=1)
   │     │ (文字起こし完了後、Nextcloudへ即座に先行PUT保存)
   │     ▼
   └── llama-proxy (FastAPI ポート8000) ──> llama-server (Qwen2.5-32B, -c 32768)
         │
         │ (WebDAV PUT: Markdown要約)
         ▼
[ NextcloudPi : /講義ノート/<科目名>/第<XX>回/ ]
   ├── 文字起こし_全文.txt (先行保存ガードにより100%保護)
   └── 講義ノート_要約.md   (Qwen2.5-32B による構造化ノート)
         │
         │ (WebDAV MOVE: 処理完了後に元音声を退避)
         ▼
[ NextcloudPi : /99_Archive/<科目名>/<元音声>.aac ]
```

---

## 2. ディレクトリ状態遷移と非破壊設計

ファイルのライフサイクルに応じたディレクトリ分離（ステートマシン）を採用し、元音声データを一切破壊・上書きしない非破壊設計を担保する。

* `00_Inbox/`: 音声ファイルの投入ポスト。手動アップロード先。
* `01_Processing/`: 処理中ディレクトリ。スクリプト検知と同時にWebDAV `MOVE` で即座に隔離し、次回巡回の多重処理を防止。
* `講義ノート/<科目名>/第<XX>回/`: 生成された成果物の格納先。
  * `文字起こし_全文.txt`: Whisper large-v3 が書き起こした生テキスト（数万文字）。
  * `講義ノート_要約.md`: Qwen2.5-32B が整形した試験対策用のMarkdown講義ノート。
* `99_Archive/<科目名>/`: 処理完了後の元音声ファイル（`.aac`）を無加工のまま退避。万が一の再要約や聞き直しに対応。

---

## 3. 推論基盤と処理性能実績

### ① faster-whisper（CTranslate2 / AVX-512 VNNI）
* **モデル:** `large-v3`（約15.5億パラメータ, INT8量子化）
* **探索モード:** `beam_size=1, best_of=1`（Greedy探索）
* **実測性能:**
  * **専門講義A:** 約57分音声 ──> 約16分で処理完了（20,662文字）
  * **総合講義B:** 約101分音声 ──> 約29分で処理完了（33,819文字）
  * **RTF（リアルタイム係数）:** 約 **0.28**（音声実時間の3割未満の猛烈な速度でCPU完走）

### ② llama.cpp（動的プロキシ ＋ Qwen2.5-32B）
* **アーキテクチャ:** FastAPI製動的プロキシ（`llama-proxy.service`, ポート8000）を介して、バックエンドの `llama-server`（ポート8080）をオンデマンド制御。
* **モデル:** `Qwen2.5-32B-Instruct-Q4_K_M.gguf`（Cascade Lake CPU Dual / 12ch メモリ帯域の性能をフル活用）
* **コンテキスト長:** `-c 32768`（32kトークンへ拡張し、100分超の講義テキストも余裕で呑み込める設計）

---

## 4. トラブルシューティングと技術知見（ハマりどころ）

### ① スマホ生AAC（ADTS）のパケット破損と ffmpeg サニタイズ
* **課題:** スマホ標準ボイスレコーダーが出力する生AACはフレーム境界の乱れがあり、Pythonの `av`（PyAV）で直接デコードすると `InvalidDataError` が発生して音声長 `0.0秒` 判定で即死する。
* **解決策:** OSネイティブの `ffmpeg` を呼び出し、事前に `16kHz 16bit モノラル WAV` へサニタイズ変換してからWhisperに投入。破損パケットを自動スキップして確実にデコードを完遂。

### ② PyAV 19.x 系と faster-whisper の世代間ギャップ
* **課題:** PyAV 19.0.0 では `metadata_errors="ignore"` 引数が削除されており、faster-whisper 内部の `audio.py` で `TypeError` が発生する。
* **解決策:** `audio.py` 内の該当引数を `sed` でワンライナー削除、または `av<14,>=12.0.0` の安定世代に固定。

### ③ 動的プロキシ（model_proxy.py）連携と 500 Internal Server Error
* **課題:** ポート8000へのリクエストで `500 Server Error: httpx.ConnectError` が発生。
* **根本原因:** リクエストのJSONペイロードに `"model"` キーが含まれておらず、プロキシの `ensure_model()` が素通りされて裏の `llama-server`（ポート8080）が起動していなかった。
* **解決策:** `pipeline.py` 側で `"model": "Qwen2.5-32B-Instruct-Q4_K_M.gguf"` を明示し、プロキシ側でも未指定時のフォールバック先を設定。

### ④ コンテキスト上限不足とタイムアウト対策
* **課題:** 100分超の講義音声はテキスト化すると3万文字超（約2.5万トークン）に達し、従来の `-c 16384`（16k）ではコンテキスト長が不足してクラッシュする危険があった。
* **解決策:** プロキシ起動引数のコンテキスト長を `-c 32768`（32k）へ拡張。CPU推論時間を考慮し、プロキシおよびクライアントのタイムアウトを `1200秒`（20分）に設定。

### ⑤ 先行保存ガードによる成果物保護
* **課題:** LLM要約が万が一タイムアウトやエラーで落ちた場合、せっかく長時間かけて文字起こしした全文テキストが消去・消失してしまうリスク。
* **解決策:** Whisperの文字起こし完了直後、LLM要約APIを叩く前に `講義ノート/<科目名>/第<XX>回/文字起こし_全文.txt` を即座にNextcloudへアップロード。文字起こし結果を100%保護した状態で後段の要約処理へ移行するフェイルセーフを確立。

### ⑥ systemd timer による完全排他巡回
* **課題:** 講義音声が連続投入された際、前回の処理中に次のタイマーが発火してCPU・メモリが多重起動でパンクする。
* **解決策:** `lecture-pipeline.timer` に `OnUnitInactiveSec=5min` を採用。「前の処理が**完全に終了してから5分後**に次の巡回を行う」仕様とし、多重起動を物理的に遮断。
````

## File: docs/nextcloud-mcp.md
````markdown
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
````

## File: src/mcp_server.py
````python
#!/usr/bin/env python3
"""
mcp_server.py
Nextcloud WebDAV 連携 FastMCP (SSE) サーバー
LibreChat 等の MCP クライアントから講義ノートを安全に参照・検索するエンドポイントを提供
"""

import os
from mcp.server.fastmcp import FastMCP
from nc_client import NextcloudWebDAV

# === Nextcloud設定 (環境変数またはプレースホルダー) ===
NCP_HOST = os.getenv("NCP_HOST", "https://nextcloud.example.ts.net")
NCP_USER = os.getenv("NCP_USER", "your_username")
NCP_PASS = os.getenv("NCP_PASS", "your_app_password_here")
TARGET_BASE = os.getenv("TARGET_BASE", "/大学/2026_秋")

# FastMCPサーバー初期化 (ポート8000 / 0.0.0.0 で外部連携)
mcp = FastMCP("Nextcloud-Academic-Notes", host="0.0.0.0", port=8000)
nc = NextcloudWebDAV(NCP_HOST, NCP_USER, NCP_PASS)


@mcp.tool()
def list_course_notes(subpath: str = "") -> list[str]:
    """Nextcloud上の講義フォルダやファイルの一覧を取得します。
    引数 subpath: '科目名' や '科目名/第01回' などの相対パス。空なら科目一覧を返します。
    """
    clean_sub = subpath.strip("/")
    target = f"{TARGET_BASE}/{clean_sub}" if clean_sub else TARGET_BASE

    files = nc.list_dir(target)
    return files


@mcp.tool()
def read_course_note(relative_file_path: str) -> str:
    """指定された講義ノートや要約ファイルの中身を読み出します。
    引数 relative_file_path 例: '科目名/第01回/講義ノート_要約.md'
    """
    clean_path = relative_file_path.strip("/")
    target = f"{TARGET_BASE}/{clean_path}"

    # パストラバーサル防止ガード
    if not clean_path or ".." in clean_path:
        return "エラー: 不正なパス指定です。"

    try:
        content = nc.read_text(target)
        return content
    except Exception as e:
        return f"ファイル読み出し失敗: {e}"


if __name__ == "__main__":
    # 外部(LibreChat)から叩けるように SSE トランスポートで待受起動
    mcp.run(transport="sse")
````

## File: src/nc_client.py
````python
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
````

## File: src/pipeline.py
````python
import os
import re
import sys
import json
import time
import subprocess
import requests
import unicodedata
import urllib3
import xml.etree.ElementTree as ET
from urllib.parse import quote, unquote
from faster_whisper import WhisperModel

# SSL警告の完全抑制
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# === 設定項目 (環境変数から取得、未設定時はダミー/デフォルト値) ===
NC_HOST = os.getenv("NC_HOST", "https://nextcloud.example.ts.net")
NC_USER = os.getenv("NC_USER", "your_username")
NC_PASS = os.getenv("NC_PASS", "your_app_password_here")
DAV_BASE = f"{NC_HOST}/remote.php/dav/files/{NC_USER}"
TARGET_ROOT = os.getenv("TARGET_ROOT", "大学/2026_秋")  # MCPサーバー側の参照ルートと完全に一致させる
LLM_API_URL = os.getenv("LLM_API_URL", "http://127.0.0.1:8000/v1/chat/completions")
LLM_MODEL = os.getenv("LLM_MODEL", "Qwen2.5-32B-Instruct-Q4_K_M.gguf")

auth = (NC_USER, NC_PASS)

def dav_url(path: str) -> str:
    parts = [quote(p) for p in path.strip("/").split("/") if p]
    return f"{DAV_BASE}/" + "/".join(parts)

def list_inbox():
    res = requests.request("PROPFIND", dav_url("00_Inbox"), auth=auth, verify=False, headers={"Depth": "1"})
    if res.status_code not in [200, 207]:
        return []
    root = ET.fromstring(res.text)
    files = []
    target = f"/remote.php/dav/files/{NC_USER}/00_Inbox"
    for elem in root.findall(".//{DAV:}response"):
        href = unquote(elem.find("{DAV:}href").text.rstrip("/"))
        if href != target and href.lower().endswith((".aac", ".m4a", ".mp3", ".wav")):
            files.append(href.split("/")[-1])
    return files

def mkdir_p(path: str):
    cur = ""
    for p in path.strip("/").split("/"):
        if not p: continue
        cur += "/" + p
        requests.request("MKCOL", dav_url(cur), auth=auth, verify=False)

def move_file(src: str, dest: str):
    headers = {"Destination": dav_url(dest), "Overwrite": "T"}
    res = requests.request("MOVE", dav_url(src), auth=auth, headers=headers, verify=False)
    return res.status_code in [201, 204]

def download_file(src: str, local_path: str):
    res = requests.get(dav_url(src), auth=auth, verify=False, stream=True)
    res.raise_for_status()
    with open(local_path, "wb") as f:
        for chunk in res.iter_content(chunk_size=8192):
            f.write(chunk)

def upload_text(dest: str, content: str):
    res = requests.put(dav_url(dest), data=content.encode("utf-8"), auth=auth, verify=False)
    return res.status_code in [201, 204]

def parse_filename(filename: str):
    base = os.path.splitext(filename)[0]
    base = unicodedata.normalize('NFKC', base).strip()
    match = re.search(r"^(.*?)[_ -]*(?:第)?([0-9]{1,2})(?:回)?$", base)
    if match:
        subject = match.group(1).strip()
        num = int(match.group(2))
        return subject, f"第{num:02d}回"
    return base, "第01回"

def convert_to_wav(input_path: str, output_path: str):
    cmd = [
        "ffmpeg", "-y", "-i", input_path,
        "-ar", "16000", "-ac", "1", "-c:a", "pcm_s16le",
        output_path
    ]
    res = subprocess.run(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    if res.returncode != 0 or not os.path.exists(output_path) or os.path.getsize(output_path) == 0:
        raise RuntimeError(f"ffmpeg 変換に失敗しました: {input_path}")

def summarize_with_llm(subject: str, lecture_round: str, transcript: str) -> str:
    system_prompt = (
        "あなたは大学講義の専属ノートテイカーAIです。入力された講義音声の文字起こしテキストから、"
        "復習および試験対策に最適化された構造化された講義ノートをMarkdown形式で作成してください。\n"
        "専門用語は正確に記載し、重要な概念、箇条書き、定義、試験に出そうなポイントを明確に整理してください。"
    )
    user_prompt = f"# 講義科目: {subject} ({lecture_round})\n\n以下は本講義の文字起こしデータです。これを整理・構造化して講義ノートを作成してください。\n\n---\n{transcript}\n"
    payload = {
        "model": LLM_MODEL,
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        "temperature": 0.3,
        "max_tokens": 4096
    }
    res = requests.post(LLM_API_URL, json=payload, timeout=1200)
    res.raise_for_status()
    return res.json()["choices"][0]["message"]["content"]

def main():
    inbox_files = list_inbox()
    if not inbox_files:
        return
        
    print(f"[*] 検出された音声 ({len(inbox_files)}件): {inbox_files}")
    print("[*] faster-whisper (large-v3) をロード中...")
    model = WhisperModel("large-v3", device="cpu", compute_type="int8", cpu_threads=16)

    for fname in inbox_files:
        subject, lecture_round = parse_filename(fname)
        print(f"\n==========================================")
        print(f"[*] 対象: {fname} -> 科目: {subject}, 回数: {lecture_round}")
        
        src_path = f"00_Inbox/{fname}"
        proc_path = f"01_Processing/{fname}"
        print(f"[*] Processingへ移動中: {src_path} -> {proc_path}")
        move_file(src_path, proc_path)
        
        local_audio = f"/tmp/{fname}"
        local_wav = f"/tmp/{os.path.splitext(fname)[0]}.wav"
        target_dir = f"{TARGET_ROOT}/{subject}/{lecture_round}"
        
        try:
            print(f"[*] 音声をダウンロード中...")
            download_file(proc_path, local_audio)
            
            print(f"[*] ffmpeg で WAV 変換中 (16kHz mono)...")
            convert_to_wav(local_audio, local_wav)
            
            print("[*] 文字起こし開始 (CPU int8, beam_size=1 爆速モード)...")
            t0 = time.time()
            segments, info = model.transcribe(
                local_wav,
                language="ja",
                beam_size=1,
                best_of=1,
                vad_filter=True
            )
            full_text = [seg.text for seg in segments]
            transcript_raw = "\n".join(full_text).strip()
            
            if not transcript_raw:
                print(f"[!] 警告: 文字起こし結果が空です。ファイルを保護して中断します。")
                continue
                
            duration_min = info.duration / 60 if info.duration else 0
            proc_time = time.time() - t0
            print(f"[*] 文字起こし完了! (音声長: {duration_min:.1f}分, 処理時間: {proc_time:.1f}秒, 文字数: {len(transcript_raw)})")
            
            # 【先行保存ガード】LLMを呼ぶ前に文字起こしテキスト全文を即座にNextcloudへ保存
            mkdir_p(target_dir)
            upload_text(f"{target_dir}/文字起こし_全文.txt", transcript_raw)
            print(f"[*] 文字起こし全文を Nextcloud に先行保存完了: {target_dir}")
            
            print(f"[*] {LLM_MODEL} (llama-proxy経由) で講義ノート作成中...")
            t0 = time.time()
            summary_md = summarize_with_llm(subject, lecture_round, transcript_raw)
            print(f"[*] 要約完了! ({time.time() - t0:.1f}秒)")
            
            upload_text(f"{target_dir}/講義ノート_要約.md", summary_md)
            print(f"[*] 講義ノート要約を保存しました: {target_dir}")
            
            archive_dir = f"99_Archive/{subject}"
            mkdir_p(archive_dir)
            move_file(proc_path, f"{archive_dir}/{fname}")
            print(f"[*] 元音声をアーカイブへ移動完了: {archive_dir}/{fname}")
            
        except Exception as e:
            print(f"[!] エラー発生 ({fname}): {e}")
        finally:
            for p in [local_audio, local_wav]:
                if os.path.exists(p):
                    try:
                        os.remove(p)
                    except:
                        pass
                    
        print(f"[*] {fname} の処理サイクルが終了しました。")

if __name__ == "__main__":
    main()
````

## File: src/sorter_worker.py
````python
#!/usr/bin/env python3
"""
sorter_worker.py
Nextcloud 受動的ファイル自動仕分けワーカー
_Inbox を監視し、ローカル LLM (llama.cpp) を用いて科目・講義回を判定して自動移動
"""

import os
import json
import requests
from nc_client import NextcloudWebDAV

# === 設定 (環境変数またはプレースホルダー) ===
NCP_HOST = os.getenv("NCP_HOST", "https://nextcloud.example.ts.net")
NCP_USER = os.getenv("NCP_USER", "your_username")
NCP_PASS = os.getenv("NCP_PASS", "your_app_password_here")

LLM_HOST = os.getenv("LLM_HOST", "http://127.0.0.1:8080")
LLM_API_URL = f"{LLM_HOST}/v1/chat/completions"

INBOX_DIR = "/_Inbox"
TARGET_ROOT = os.getenv("TARGET_ROOT", "/大学/2026_秋")

# 登録科目リストのサンプル（運用環境に合わせてカスタマイズ）
SUBJECTS = [
    "分子生物学",
    "生化学Ⅰ",
    "有機化学Ⅱ",
    "物理化学実験",
    "情報科学基礎",
    "その他"
]


def classify_file(filename: str) -> dict:
    """llama.cpp にファイル名を渡して分類判定（JSON）"""
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

    # チャット推論等のキュー待ちに耐えるようタイムアウト300秒
    res = requests.post(LLM_API_URL, json=payload, timeout=300)
    res.raise_for_status()

    content = res.json()["choices"][0]["message"]["content"]
    return json.loads(content)


def process_inbox(nc: NextcloudWebDAV):
    """_Inbox 内のファイルを巡回して仕分け"""
    files = nc.list_dir(INBOX_DIR)
    if not files:
        print("[INFO] _Inbox は空です。")
        return

    print(f"[INFO] 処理対象ファイル: {files}")

    for filename in files:
        print(f"\n--- 処理開始: {filename} ---")
        try:
            # 1. LLMによる推論
            result = classify_file(filename)
            subject = result.get("subject", "その他")
            week = int(result.get("week", 0))
            print(f"[判定結果] 科目: {subject} / 回数: 第{week}回")

            # 2. 移動先ディレクトリの決定
            if week > 0:
                dest_dir = f"{TARGET_ROOT}/{subject}/第{week:02d}回"
            else:
                dest_dir = f"{TARGET_ROOT}/{subject}"

            # 3. ディレクトリ作成 & ファイル移動
            nc.mkdir_p(dest_dir)
            src_path = f"{INBOX_DIR}/{filename}"
            dest_path = f"{dest_dir}/{filename}"

            if nc.move(src_path, dest_path):
                print(f"[完了] 移動成功: {dest_path}")
            else:
                print(f"[警告] 移動に失敗しました: {filename}")

        except Exception as e:
            # 個別例外が発生してもワーカー全体は落とさず継続
            print(f"[エラー発生] {filename} の処理をスキップ: {e}")


if __name__ == "__main__":
    nc = NextcloudWebDAV(NCP_HOST, NCP_USER, NCP_PASS)
    process_inbox(nc)
````

## File: systemd/lecture-pipeline.service
````
[Unit]
Description=Automated Lecture Audio Transcription and Summarization Pipeline
After=network.target

[Service]
Type=oneshot
User=<username>
WorkingDirectory=/home/<username>
Environment=PYTHONUNBUFFERED=1
ExecStart=/home/<username>/whisper-env/bin/python3 /home/<username>/pipeline.py
TimeoutStartSec=infinity
Nice=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
````

## File: systemd/lecture-pipeline.timer
````
[Unit]
Description=Run Lecture Pipeline 5 minutes after last run finishes

[Timer]
OnBootSec=2min
OnUnitInactiveSec=5min
Persistent=true

[Install]
WantedBy=timers.target
````

## File: systemd/nextcloud-mcp.service
````
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
````

## File: .gitignore
````
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
.venv/
whisper-env/

# 環境変数・クレデンシャル
.env
*.token
*.pem
*.key

# 一時ファイル
/tmp/
*.wav
*.aac
*.mp3
````

## File: LICENSE
````
MIT License

Copyright (c) 2026 kuwafu

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
````

## File: README.md
````markdown
# mcp-file-agent

オンプレミス仮想化基盤（Proxmox VE）上で稼働する Nextcloud を中核とし、ローカル LLM（`llama.cpp`）および LibreChat の Model Context Protocol (MCP) を統合した、学術データ自律仕分け・音声文字起こし・講義ノート生成インフラです[cite: 14]。

---

## 1. システム概要 (Overview)

本リポジトリは、以下の2つの自律パイプラインを密結合して運用するための統合バックエンドです[cite: 14]。

1. **Nextcloud FastMCP Agent (閲覧・検索・仕分け基盤):**[cite: 14]
   * LibreChat の MCP クライアント機能から、WebDAV 経由で安全に Nextcloud 上の学術資料や講義ノートを横断検索・参照する FastMCP (SSE) サーバー[cite: 14]。
   * アップロードされた資料のファイル名から科目・回数をローカル LLM が自律推論し、該当ディレクトリへ自動分類・移動する受動的ワーカー[cite: 14]。
2. **講義音声 文字起こし & 構造化要約パイプライン (推論バッチ基盤):**[cite: 14]
   * スマホから Nextcloud の受信トレイ（`00_Inbox`）へ投入された音声ファイルを検知[cite: 14]。
   * オンプレミスの Xeon 計算基盤（Cascade Lake）上で CPU 推論（`faster-whisper` + `llama.cpp`）を実行し、文字起こし全文と試験対策用 Markdown 講義ノートを自律生成して Nextcloud へ再配置[cite: 14]。

---

## 2. システムアーキテクチャ (Architecture)

物理ホスト（Proxmox VE）内の VM / LXC コンテナ間を Tailscale メッシュネットワークで結び、宅内 LAN の IP 変動や瞬断に影響されない閉域通信を確立しています[cite: 14]。

```text
======================== [ オンプレミス物理ホスト (Proxmox VE) ] ========================
       │
       ├── [ ArcLight (Ubuntu VM) ] ── 推論ノード
       │    ├── CPU/RAM: デュアル Intel Xeon Gold（AVX-512 VNNI 最適化）[cite: 14]
       │    ├── llama-server (Qwen2.5-32B, コンテキスト長 32k)[cite: 14]
       │    ├── llama-proxy.service (FastAPI 動的プロキシ / ポート 8000)[cite: 14]
       │    └── lecture-pipeline (systemd timer 巡回 / faster-whisper large-v3)[cite: 14]
       │
       ├── [ NextcloudPi (Debian LXC) ] ── ストレージ & MCP ハブ
       │    ├── Nextcloud WebDAV API (ポート 443 / remote.php/dav/files/<user>/)[cite: 14]
       │    ├── nextcloud-mcp.service (FastMCP SSE サーバー / ポート 8000)[cite: 14]
       │    └── sorter_worker.py (LLM 連携受動的ファイル自動仕分け)[cite: 14]
       │
       └── [ LibreChat (Docker LXC) ] ── オーケストレーション UI
            └── mcpServers 連携 (Nextcloud 内のノートを対話形式で参照・RAG 活用)[cite: 14]
========================================================================================
```

---

## 3. 主要コンポーネント (Components)

### ① FastMCP サーバー (`src/mcp_server.py`)[cite: 14]
* **プロトコル:** Model Context Protocol (FastMCP 1.x 準拠, SSE トランスポート)[cite: 14]。
* **セキュリティ:** 意図しないファイル破壊やシステム領域参照を防ぐため、閲覧専用ツールに限定し、相対パス検証によるパストラバーサル防止ガードを実装[cite: 14]。
  * `list_course_notes(subpath)`: 科目・回数ごとのフォルダやノート一覧を取得[cite: 14]。
  * `read_course_note(relative_file_path)`: 講義ノート要約や文字起こしテキストの読み出し[cite: 14]。

### ② 受動的自動仕分けワーカー (`src/sorter_worker.py`)[cite: 14]
* **推論制御:** `_Inbox` に配置された未整理ファイルをスキャンし、ローカル LLM（`llama.cpp`）へ問い合わせ[cite: 14]。
* **構造化出力:** `response_format={"type": "json_object"}` を適用し、曖昧なファイル名から科目名と講義回（第XX回）を高精度に抽出[cite: 14]。
* **WebDAV 移動:** Nextcloud のデータベース整合性を保護するため、直接のファイルシステム操作を避け、公式 WebDAV `MKCOL` / `MOVE` API 経由で移動を実行[cite: 14]。

### ③ 講義音声 文字起こし & 要約パイプライン (`src/pipeline.py`)[cite: 14]
* **音声前処理:** スマホ録音時の ADTS パケット破損を吸収するため、`ffmpeg` により `16kHz mono WAV` へ事前サニタイズ変換[cite: 14]。
* **高速文字起こし:** Cascade Lake の `AVX-512 VNNI`（INT8 積和演算）および `beam_size=1`（Greedy 探索）を駆使し、CPU 実行でありながら RTF（リアルタイム係数）約 0.28 を達成[cite: 14]。
* **先行保存ガード:** LLM 要約時のタイムアウトや推論エラーに備え、Whisper 完了直後に文字起こし全文を Nextcloud へ即座に `PUT` アップロードするフェイルセーフを確立[cite: 14]。
* **排他巡回:** `lecture-pipeline.timer` に `OnUnitInactiveSec=5min` を設定し、多重起動によるメモリ・CPU バウンドのパンクを物理的に遮断[cite: 14]。

### ④ 共通 WebDAV 通信クライアント (`src/nc_client.py`)[cite: 14]
* `requests` ベースの軽量 WebDAV ラッパーモジュール[cite: 14]。
* HTTP `Destination` ヘッダー転送における latin-1 制限を回避するため、日本語ファイルパスを `quote(path, safe="/")` で自動エンコード処理[cite: 14]。

---

## 4. ドキュメント & 詳細設計 (Documentation)

各サブシステムの設計思想、トラブルシューティング記録、最適化ベンチマークは `docs/` 配下にまとめています[cite: 14]。

* **[講義音声 自動文字起こし & 構造化要約パイプライン仕様書](docs/lecture-pipeline.md)**[cite: 14]  
  ステートマシン設計、生 AAC 破損対策、PyAV バージョン互換性、AVX-512 VNNI チューニング、systemd timer 排他制御[cite: 14]。
* **[Nextcloud MCP & 自動仕分け連携仕様書](docs/nextcloud-mcp.md)**[cite: 14]  
  FastMCP SSE 設定、LibreChat SSRF 対策、WebDAV メタデータ保護、分類プロンプト設計[cite: 14]。
* **[自宅インフラ & ネットワーク構成仕様書](docs/infrastructure.md)**[cite: 14]  
  Proxmox VE 仮想化トポロジ、Tailscale MagicDNS 固定、デュアル Xeon NUMA 最適化、llama-proxy アーキテクチャ[cite: 14]。

---

## 5. セットアップ (Getting Started)

### 1. 依存ライブラリのインストール
```bash
pip install -r requirements.txt
```

### 2. 環境変数の設定
各スクリプトは環境変数経由で接続情報を読み込みます（未指定時はデフォルト値を使用）[cite: 14]。

```bash
export NCP_HOST="[https://your-nextcloud.ts.net](https://your-nextcloud.ts.net)"
export NCP_USER="your_nextcloud_username"
export NCP_PASS="your_nextcloud_app_password"
export LLM_HOST="http://your-arclight-ip:8080"
export TARGET_ROOT="/大学/2026_秋"
```

### 3. systemd による常駐・自動化

#### NextcloudPi 側: FastMCP サーバー常駐化
```bash
cp systemd/nextcloud-mcp.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now nextcloud-mcp.service
```

#### ArcLight (推論ノード) 側: 音声巡回タイマー有効化
```bash
cp systemd/lecture-pipeline.* /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now lecture-pipeline.timer
```

---

## 6. ディレクトリ構成 (Directory Structure)

```text
.
├── README.md                          # 本ドキュメント
├── LICENSE                            # MIT License
├── requirements.txt                   # 依存 Python パッケージ定義
├── .gitignore                         # Git 除外設定
├── docs/                              # 詳細技術仕様・障害対応記録
│   ├── infrastructure.md              # 仮想化基盤 & ネットワークトポロジ仕様
│   ├── lecture-pipeline.md            # 音声文字起こし・要約パイプライン仕様
│   └── nextcloud-mcp.md               # FastMCP & ファイル自動仕分け仕様
├── src/                               # アプリケーション実装本体
│   ├── mcp_server.py                  # Nextcloud 参照用 FastMCP (SSE) サーバー
│   ├── nc_client.py                   # 共通 WebDAV 通信クライアントモジュール
│   ├── pipeline.py                    # 音声文字起こし・Markdown 講義ノート生成
│   └── sorter_worker.py               # LLM 連携受動的ファイル自動仕分けワーカー
└── systemd/                           # 自動化・永続化ユニット定義
    ├── lecture-pipeline.service       # 音声処理パイプライン実行サービス
    ├── lecture-pipeline.timer         # 処理完了後 5 分間隔の排他実行タイマー
    └── nextcloud-mcp.service          # FastMCP サーバー永続常駐サービス
```

---

## 7. ライセンス (License)

[MIT License](LICENSE)
````

## File: requirements.txt
````
# FastMCP (1.x系で固定)
mcp<2

# HTTP & WebDAV 通信
requests>=2.31.0
urllib3>=2.0.0

# 音声文字起こし & パイプライン (ArcLight 推論側)
faster-whisper>=1.0.0
````
