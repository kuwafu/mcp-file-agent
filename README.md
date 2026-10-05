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