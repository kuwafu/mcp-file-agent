# 講義音声 自動文字起こし & 構造化要約パイプライン

スマホからNextcloudへ講義音声（AAC）をアップロードするだけで、オンプレミスのXeon計算基盤上で文字起こし（CTranslate2 / faster-whisper）とLLM要約（llama.cpp / Qwen2.5）を自律実行し、講義ノートをWebDAV経由で自動配置するパイプライン。

---

## 1. 全体データフロー

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
---

## 2. ディレクトリ状態遷移（ステートマシン設計）

二重処理やスクリプト異常終了時の多重実行を防ぐため、ファイルのライフサイクルに応じたディレクトリ分離を実施。

* `00_Inbox/`: 音声ファイルの投入ポスト。
* `01_Processing/`: 処理中ディレクトリ。スクリプトが検知した瞬間にWebDAVの `MOVE` で即座に隔離し、次回巡回での重複実行を防止。
* `講義ノート/<科目名>/第<XX>回/`: 生成されたMarkdown要約ノートと文字起こし全文テキストの格納先。
* `99_Archive/<科目名>/`: 処理完了後の元音声ファイルを退避。万が一の再文字起こしにも対応。

---

## 3. ハマりどころと設計上の解決策（技術知見）

### ① スマホ生録音（AAC/ADTS）のパケット破損と ffmpeg サニタイズ
* **課題:** スマホ標準のボイスレコーダーが書き出す生AAC（ADTS形式）はパケットヘッダの境界が乱れていることが多く、Pythonの `av`（PyAV）で直接読み込もうとすると `InvalidDataError` が発生して音声長 `0.0秒` 判定で即死する。
* **解決策:** Python側で無理にパケット補正を行わず、OSネイティブの `ffmpeg` をサブプロセスで呼び出し、事前に `16kHz 16bit モノラル WAV` へサニタイズ変換してからWhisperへ投入する設計を採用。破損パケットを自動スキップして確実にデコードを完遂。

### ② PyAV 19.x 系と faster-whisper の世代間ギャップ
* **課題:** 最新環境（PyAV 19.0.0）では `metadata_errors="ignore"` 引数が非推奨化・削除されており、faster-whisper 呼び出し時に `TypeError` で爆死する。
* **解決策:** ライブラリ側の `audio.py` 内の不要な引数を `sed` でワンライナー削除、または `av<14,>=12.0.0` の安定世代に固定して整合性を確保。

### ③ メモリ帯域バウンドの回避（AVX-512 VNNI × beam_size=1）
* **課題:** `large-v3`（約15.5億パラメータ）をCPU推論させる際、デフォルトの `beam_size=5` では自己回帰デコーダのメモリ読み出し回数が激増し、長時間の音声で深刻な処理遅延が発生する。
* **解決策:** `beam_size=1, best_of=1`（Greedy探索）にチューニング。`large-v3` 自体の高い言語能力を活かすことで、専門用語の認識精度を落とすことなく探索オーバーヘッドを削減し、処理時間を1/2〜1/3へ短縮。

### ④ systemd timer による安全な排他自律稼働
* **課題:** 長時間の講義音声（60〜90分等）を処理している最中に次の定期実行が重なると、CPUとメモリが多重起動でパンクする。
* **解決策:** 
  * `lecture-pipeline.service`: `Type=oneshot`, `TimeoutStartSec=infinity`, `Nice=10` で実行。
  * `lecture-pipeline.timer`: `OnUnitActiveSec` ではなく `OnUnitInactiveSec=5min` を指定。「前の処理が**完全に終了してから5分後**に次の巡回を行う」設定にし、多重起動を物理的に遮断。
