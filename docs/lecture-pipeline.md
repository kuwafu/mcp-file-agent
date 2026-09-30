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
