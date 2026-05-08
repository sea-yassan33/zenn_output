# 【2026/05】ローカルLLMブラッシュアップ

- オープンソースLLMの性能向上している傾向にあり、特にコーディング領域では商用モデルに匹敵するといわれております。
- GitHub Copilotの完全従量課金化やAnthropic「Claude Enterprise」の実質的な従量課金化により、企業のセキュリティ対策やコスト削減の切り札としてローカルLLMの検討のニーズが高まっています。
- Ollamaのインストール方法やコマンド操作方法について紹介します。

## GPUメモリから動くモデルについて

AI開発やローカルLLMの実行に役立つ、最新（2026年5月時点）の市場状況を反映したGPU比較表です。

> LLM/AI向けGPU比較表（2026年5月版）

| クラス | おすすめ型番 | VRAM容量 | 特徴 | 値段相場 (税込) | おすすめのモデル例 |
| --- | --- | --- | --- | --- | --- |
| **最高峰** | **RTX 4090** | 24GB | 現状の最強。大半のモデルが快適に動く。 | **約30万〜45万円** | **MSI SUPRIM X** (冷却性能高) / **ASUS ROG Strix** |
| **高コスパ** | **RTX 3090** | 24GB | 性能は4090に劣るが、VRAM 24GBが安く手に入る。 | **約15万〜20万円** | **MSI Gaming X Trio** / **ZOTAC Trinity** |
| **標準的** | **RTX 4070 Ti Super** | 16GB | 16GBあれば中規模モデルも視野。電力効率が良い。 | **約13万〜16万円** | **MSI Ventus 2X** (コンパクト) / **ZOTAC AMP** |
| **入門用** | **RTX 4060 Ti (16GB)** | 16GB | 安価に16GBを確保できる唯一の選択肢。消費電力が低い。 | **約7.5万〜9万円** | **MSI GAMING X SLIM** / **Palit JetStream** |
| **Mac派** | **M3 Max (32GB+)** | 共有メモリ | メインメモリをVRAMとして活用。Mac Studio/Proも選択肢。 | **約50万円〜** (本体) | **MacBook Pro** / **Mac Studio** |

> 【参考】RTX4060-Laptop(VRAM:8GB)で動くモデルについて

|項目|Qwen3.5:4B|Gemma3:4B|
|:----|:----|:----|
|パラメーター数|約40億|約40億|
|得意分野|コーディング・数学・論理推論。特にプログラミングのロジック構築に強い。|自然な日本語対話・要約。Googleの調整により、文章のニュアンスが非常に自然。|
|日本語の質|非常に高い。正確で簡潔な回答を好む。|極めて高い。より人間味のある、親しみやすい表現が得意。|
|思考機能 (Reasoning)|非常に高速な思考ステップ。|「Chain-of-Thought（思考の連鎖）」が強化され、ステップバイステップでの回答が得意。|
|推論速度 (4060 Laptop)|爆速。100 tps（秒間100文字以上）を超えることもある。|高速。ストレスなく、リアルタイムで文章が生成される。|
|VRAM使用量|約3.5GB 〜 4.5GB (4-bit量子化)|約3.8GB 〜 4.8GB (4-bit量子化)|
|開発環境との親和性|Next.jsやPythonのコード生成において、微細なバグ修正まで見抜く傾向。|ドキュメントの作成や、プロジェクトの企画構成案出しに最適。|

## 用途別モデル

|用途|推奨モデル|選定理由|
|:----|:----|:----|
|コーディング補完|Qwen3 or Qwen2.5-Coder|JSON 出力が安定、Apache 2.0 ライセンス|
|コーディング（中型）|Devstral Small 2|SWE-bench 68%、256K コンテキスト|
|汎用チャット|Llama 3.3|128k コンテキスト対応、幅広いサイズ展開|
|コスト効率重視|Qwen3-30B-A3B|MoE 構造で実質 3B 稼働、Apache 2.0|
|軽量・エッジ|gpt-oss-20b or Qwen3-1.7B|16GB/8GB で動作、推論モデル or 軽量汎用|
|マルチモーダル|Gemma 3-27B|テキスト・画像・動画対応、140 言語|
|数学・推論|Nemotron 3 Nano|AIME 89.1%、1M コンテキスト|
|推論タスク|DeepSeek-V3.2|GPT-5 レベル、推論・Agent 統合|

- [引用1](https://dev.classmethod.jp/articles/local-llm-guide-2026/#用途別おすすめモデル)

## モデル一覧

|モデル|得意領域|サイズ|ライセンス|特徴|
|:----|:----|:----|:----|:----|
|Qwen3-14B|汎用・日本語|14B|Apache 2.0|Qwen2.5-32B 相当の性能|
|Qwen3-30B-A3B|コスト効率|30B（稼働 3B）|Apache 2.0|MoE で軽量動作|
|Qwen2.5-Coder|コード生成・JSON|0.5B〜32B|Apache 2.0|29 言語対応|
|Qwen3-Coder|Agent・コード生成|480B（稼働 35B）|Apache 2.0|256K コンテキスト|
|Devstral Small 2|コーディング|24B|Apache 2.0|SWE-bench 68%、256K コンテキスト|
|gpt-oss-20b|推論|21B（稼働 3.6B）|Apache 2.0|OpenAI 初のオープンウェイトモデル|
|GLM-4.7-Flash|推論・高速生成|30B（稼働 3B）|MIT|24GB 推奨、安定性に課題|
|Gemma 3-27B|マルチモーダル|27B|Gemma 独自|140 言語、128K コンテキスト|
|Nemotron 3 Nano|数学・推論|31.6B（稼働 3.6B）|NVIDIA 独自|Hybrid Mamba-Transformer MoE、20 言語対応|
|DeepSeek-V3.2|推論・Agent 統合|671B（稼働 37B）|MIT|推論・Agent 統合、R1 後継|
|Kimi K2.5|Agent・マルチモーダル|1T（稼働 32B）|Modified MIT|Agent Swarm、256K コンテキスト|
|Llama 3.3|汎用チャット|1B〜405B|Meta 独自|128k コンテキスト|

- [引用2](https://dev.classmethod.jp/articles/local-llm-guide-2026#モデル詳細比較)

## Ollamaダウンロードと使用方法

### Ollamaのサイトでダウンロード
https://ollama.com/download

- インストーラーを実行
- 実行後、ターミナルで下記のコマンドを叩く
```sh
ollama --version
```

### モデルをインストール

```sh
## モデルをインストール
ollama pull gemma3:4b

```

### ollamaの各種コマンド
```sh
## ollama 起動　（GUIで動かしているなら不要）
ollama serve

## 実行状況の確認
ollama ps

## モデル実行
ollama run <モデル名>

## モデル停止 
ollama stop <モデル名>

## モデルリスト
ollama list

## モデル情報
llama show <モデル名>

## モデル削除
### 削除後：保存先パス内に残ってないか確認 %USERPROFILE%\.ollama\models
ollama rm <モデル名>
```

## 【参考】
- [AI使い放題の終焉？2026年のAIツール課金体系変更から見るCursorの強み](https://zenn.dev/knowledgework/articles/9c1379a60e57c9)
- [LLM APIコストまとめ](https://qiita.com/SH2/items/39314152c0a6f9a7b681)
- [2026年のローカルLLM事情を整理してみた](https://dev.classmethod.jp/articles/local-llm-guide-2026/)
- [ローカルLLMのおすすめモデルと導入の全貌！スペック・商用利用・RAG構築まで徹底解説](https://eques.co.jp/column/local-llm-recommendations/)
- [GPU メモリから逆算するローカル LLM モデル選定ガイド](https://qiita.com/vfuji/items/76a63ed27dc0845436b6)