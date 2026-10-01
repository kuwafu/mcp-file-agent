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