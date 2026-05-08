---
title: "技術概要"
---
**【GPU】**
- NVIDIA GeForce RTX 4060 Laptop
- 標準メモリ構成: 8 GB

> 参考：上記GPUで動かせるモデル

|モデルサイズ|量子化（圧縮）|動作状況|備考|
|:----|:----|:----|:----|
|1B〜3B (Llama-3.2-3B等)|なし（FP16）|非常に高速|チャット用途でサクサク動きます。|
|7B〜9B (Llama-3, Gemma-2等)|4-bit〜Q8|快適|VRAM 8GBにちょうど収まり、実用的な速度が出ます。|
|12B〜14B (Mistral NeMo等)|Q4_K_M|ギリギリ|VRAMをほぼ使い切ります。他のアプリを開くと速度が落ちます。|
|20B〜|4-bit以上|低速|VRAMから溢れた分がメインメモリ（RAM）に回るため、極端に遅くなります。|

**【ローカルLLMモデル】**
- gemma3:4b
  - Googleが2025年後半にリリース
  - 軽量で高性能なローカルLLM（大規模言語モデル）
  - ローカルPC環境で高速かつ多機能に動作するモデル

**【API:LLMモデル】**
- Gemini 2.5 Flash

> 参考：上記2つのモデルの比較

| 項目 | Gemini 2.5 Flash | Gemma 3 4B |
|------|-----------------|------------|
| 提供形態 | クラウドAPI（有料） | オープンウェイト（無料・ローカル可） |
| コンテキスト長 | **100万トークン** | 12.8万トークン |
| 出力上限 | 65,000トークン | 約8,192トークン |
| 学習データ | 2025年1月まで | 2024年8月まで |
| マルチモーダル | テキスト・画像・動画・音声 | テキスト・画像 |

**【python: LangChain関連のライブラリ】**

- 今回使用するLangChain関連は下記の通りです。

|ライブラリ名|URL<deps.dev>|
|:----|:----|
|langchain|https://deps.dev/pypi/langchain|
|langchain-google-genai|https://deps.dev/pypi/langchain-google-genai|
|langchain-huggingface|https://deps.dev/pypi/langchain-huggingface|
|langchain-classic|https://deps.dev/pypi/langchain-classic|

:::message
- LangChainのバージョンは指定しませんので、インストール前に確認をお願いします。
- LangChain関連以外のライブラリに関して各々インストールしてください。
:::

