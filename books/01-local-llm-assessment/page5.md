---
title: "2次評価"
---

# 2次評価で使用する評価について

## セマンティック評価（Semantic Evaluation）

**セマンティック評価**とはその語句が持つ「意味」や「文脈（コンテクスト）」を解釈・分析をし、関連性を評価する手法です。

Embeddingsを使用して文脈の類似度を評価します。

:::message
**Embeddings（エンベディング/埋め込み表現）**とは
単語、文章をその「意味」を捉えた多次元の数値ベクトルに変換する技術です。
:::

> 主なEmbeddingsのモデル

|実行環境|推奨モデル名|日本語精度|特徴・メリット|主な用途|
|:----|:----|:----|:----|:----|
|ローカル|ruri-v3-310m|最高水準|2025-26年の最新技術を反映。日本語特化型で非常に高い検索精度。|最新の日本語RAG、高精度な文書検索。|
|ローカル|multilingual-e5-large|優秀|世界的に標準的なモデル。日本語も安定して強く、バランスが良い。|一般的なセマンティック検索、多言語対応。|
|ローカル|sentence-bert-base-ja-en-mean-tokens-v2|標準的|軽量かつ安定。 日本語と英語のバイリンガル対応。過去の事例や記事が非常に多い。|軽量な分類タスク、低スペック環境での動作、古いライブラリとの互換性。|
|API|gemini-embedding-2|非常に高い|最新のマルチモーダル対応。テキスト・画像・動画を横断して検索可能。|画像・動画を含むRAG、Googleエコシステムでの開発。|
|API|text-embedding-004|非常に高い|検索精度が高く、次元数の調整（256〜768次元）が可能。|大規模文書検索、DBコストの最適化。|

## 評価用テンプレートを用いて類似度評価を分析

セマンティック評価だけですと、類似度のみしか出力されません。

そこで、類似度について分析を第２のLLMモデルに実施してもらいます。

評価用テンプレートを用意し、第２のLLMモデルが類似度に対して分析を行います。

# STEP0:準備
## モジュールの読み込み

- 以下のモジュールを読み込みます。

```py
import warnings
# 警告を非表示にする
warnings.filterwarnings('ignore')
import sys
sys.dont_write_bytecode = True
import os
from dotenv import load_dotenv
load_dotenv()
import common_func
from common_func import GoogleMetadataCallback
## LangChain系ライブラリ
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_classic.evaluation import load_evaluator, EmbeddingDistance
## 定数
meta_map = {}
```

# STEP1:生成された概要を準備

- 生成された概要と期待する概要（評価用）を準備します。

```py
## 生成された概要
with open("./data/test03-summary.md", "r", encoding="utf-8") as f:
  final_summary = f.read()
# 期待する概要
with open("./data/paper01_oberveiw.md", "r", encoding="utf-8") as f:
  expected_text = f.read()
```

# STEP2:セマンティック評価用の評価器を準備

```py
# from langchain_huggingface import HuggingFaceEmbeddings
# from langchain_classic.evaluation import load_evaluator, EmbeddingDistance

# Embeddingsモデルをセット
embeddings = HuggingFaceEmbeddings(
  model_name="sonoisa/sentence-bert-base-ja-en-mean-tokens-v2"
)

# 評価器をセット
evaluator = load_evaluator(
  "embedding_distance",    # Embeddingベースの類似度評価
  embeddings=embeddings,   # ローカルEmbeddingをセット
  distance_metric=EmbeddingDistance.COSINE  # 自然言語に最適
)
```

> load_evaluatorの第1引数(必須)について

|設定値|用途|
|:----|:----|
|embedding_distance|Embeddingベースの類似度評価|

:::message
load_evaluatorの第1引数の指定する値について
第３評価時もload_evaluatorを使用しますのでその際に第1引数に指定する値について説明します。
セマンティック評価では「embedding_distance」を使用します。
:::

> load_evaluatorの第3引数（distance_metric）について

|設定値|用途|メトリクス | 特徴 | スコア範囲 |
|:----|:----|:----|:----|:----|
|EmbeddingDistance.COSINE|デフォルト・自然言語に最適|`COSINE` | 方向の類似度・長さに依存しない | 0〜2（通常0〜1） |
|EmbeddingDistance.EUCLIDEAN|直線距離|`EUCLIDEAN` | ベクトル間の直線距離 | 0〜∞ |
|EmbeddingDistance.MANHATTAN|絶対値の差の合計|`MANHATTAN` | 各次元の差の絶対値合計 | 0〜∞ |
|EmbeddingDistance.CHEBYSHEV|各次元の最大差|`CHEBYSHEV` | 各次元の最大差 | 0〜∞ |


# STEP3:セマンティック評価実行

## セマティック評価を実行

```py
res_assesment = evaluator.evaluate_strings(
  prediction=final_summary, # ←生成した概要
  reference=expected_text　 # ←期待する概要
)
```

## セマティック評価用の出力関数を作成

```py
def evaluation_analysis01(assesment):
  """
  評価出力関数
  assesment:セマンティック評価の結果
  """
  # --- 2次評価: セマンティック評価 ---
  print("📋 セマンティック評価結果")
  print("="*50)
  distance = assesment["score"]　# ←セマンティック評価結果のスコアを取り出す
  similarity = 1 - distance　# ←類似スコアを算出
  print(f"コサイン距離:   {distance:.4f}  （0に近いほど良い）")
  print(f"類似度スコア:   {similarity:.4f}  （1に近いほど良い）")
  print(f"評価結果:       {'✅ 良好' if similarity >= 0.8 else '⚠️ 要改善' if similarity >= 0.6 else '❌ 不十分'}")
  print("\n" + "="*50)
  print("📋 評価の分析Part1")
  print("="*50)
  return similarity # ←LLMに渡す用の類似スコアを返す
```

> セマティック評価結果を出力

```py
# evaluation_analysis01関数を実行
similarity = evaluation_analysis01(res_assesment)
```

> 出力結果

```txt
==================================================
📋 セマンティック評価結果
==================================================
コサイン距離:   0.3133  （0に近いほど良い）
類似度スコア:   0.6867  （1に近いほど良い）
評価結果:       ⚠️ 要改善
```

# STEP4:セマティック評価に対する分析

- セマティック結果より「要改善」と出ましたが、何を改善すべきか分析する必要があります。
- 今回は第2のLLM（gemini-2.5-flash）で分析してもらいます。

## 評価用テンプレートを用意

- 「生成した概要(generated)」・「期待する概要(reference)」・「類似度（similarity）」を渡せるようテンプレートを作成します。

```py
## 分析用テンプレート
analysis_prompt = ChatPromptTemplate.from_template("""
あなたは文章評価の専門家です。
以下の「生成された概要」と「参照概要」を比較し、
類似度が低い原因を日本語で簡潔に指摘してください。

生成された概要:
{generated}

参照概要:
{reference}

類似度スコア: {similarity:.2f}（1.0が最高）

指摘（箇条書き）:
""")
```

## 評価用Chainの作成

- パイプを用いて**分析用テンプレート**→**モデル**→**出力パーサー**と処理を渡すchainを作成します。

```py
## 第2のLLMモデルを用意
llm_api_gemini = ChatGoogleGenerativeAI(
  model="gemini-2.5-flash",
  google_api_key=os.environ['GOOGLE_AI_ST_API'],
  temperature=0
)
## 評価用Chainを作成
chain = analysis_prompt | llm_api_gemini | StrOutputParser()
```

## 第2のLLMによる評価を実行

- ChatGoogleGenerativeAIプロバイダー用のCallbackクラスを準備します。

> [common_func.py](https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/common_func.py)内の「GoogleMetadataCallback」クラス

```py
# ▼▼▼ GoogleGenerativeAI_Callback_Class ▼▼▼
class GoogleMetadataCallback(BaseCallbackHandler):
  """
  ChatGoogleGenerativeAI専用 Callback
  Ollamaの OllamaMetadataCallback と同等の出力を提供する
  usage_metadata キー:
    input_tokens  : プロンプトのトークン数
    output_tokens : 生成トークン数
    total_tokens  : 合計トークン数
  """
  def __init__(self, window: int = 5):
    self.call_count        = 0
    self.str_count         = 0
    self.total_prompt_tokens = 0
    self.total_eval_tokens   = 0
    self.elapsed_min       = 0
    self.start_time        = None
    self.all_records       = []
    self.durations         = []     # 各呼び出しの所要時間（分）
    self.window            = window #←　経過時間を測る際に使用（デフォルト：直近5回）
    self._call_start_time  = None   # 1回の呼び出し開始時刻
  # -------------------------------------------------------
  def on_llm_start(self, serialized, prompts, **kwargs):
    """LLM呼び出し開始時"""
    if self.call_count == 0:
      self.start_time = time.time()     #←開始時間をセット
    self._call_start_time = time.time() # 今回の呼び出し開始
    self.str_count += len(prompts[0])   #←LLMモデルが呼ばれた回数をカウント
  # -------------------------------------------------------
  def on_llm_end(self, response: LLMResult, **kwargs):
    """LLM呼び出し終了時"""
    self.call_count += 1
    # 今回の呼び出し所要時間（分）
    call_elapsed = (time.time() - self._call_start_time) / 60
    self.durations.append(call_elapsed)
    self.elapsed_min = (time.time() - self.start_time) / 60
    for generations in response.generations:
      for gen in generations:
        # ── usage_metadata からトークン情報を取得 ──
        usage = getattr(gen.message, "usage_metadata", {}) or {}
        self.all_records.append(usage)
        prompt_tokens = usage.get("input_tokens", 0) #← 入力トークン量
        eval_tokens   = usage.get("output_tokens", 0) #←　出力トークン量
        self.total_prompt_tokens += prompt_tokens
        self.total_eval_tokens   += eval_tokens
        # ── 直近 window 回の平均 ──
        recent = self.durations[-self.window:]
        avg    = sum(recent) / len(recent)
        progress = f"{self.call_count:2d}"
        print(
            f"[呼び出し #{self.call_count}] 入力: {prompt_tokens} / 出力: {eval_tokens}\n"
            f"【合計文字数】{self.str_count}"
            f"【{progress}】今回: {call_elapsed:.2f}分 "
            f"| 経過: {self.elapsed_min:.2f}分 "
            f"| 直近{len(recent)}回平均: {avg:.2f}分/回"
        )
  # -------------------------------------------------------
  def on_llm_error(self, error, **kwargs):
    """エラー発生時"""
    print(f"[ERROR] {error}")
  # -------------------------------------------------------
  def summary(self) -> dict:
    """summaryメソッドが呼ばれた際の処理（カスタム関数）"""
    total_tokens = self.total_prompt_tokens + self.total_eval_tokens
    # summary = {
    #   "call_count"          : self.call_count,
    #   "total_str_count"     : self.str_count,
    #   "total_prompt_tokens" : self.total_prompt_tokens,
    #   "total_eval_tokens"   : self.total_eval_tokens,
    #   "total_tokens"        : total_tokens,
    #   "elapsed_min"         : round(self.elapsed_min, 3),
    # }
    print("【📊 トークン集計】")
    print(f"呼び出し回数    : {self.call_count}")
    print(f"合計処理時間    : {self.elapsed_min:.2f}分")
    print(f"入力トークン合計: {self.total_prompt_tokens:,}")
    print(f"出力トークン合計: {self.total_eval_tokens:,}")
    print(f"総トークン合計  : {total_tokens:,}")
    print("="*40)
  # -------------------------------------------------------
  def meta_data(self):
    """
    meta_dataメソッドが呼ばれた際の処理（カスタム関数）
    LLMのmeta情報を辞書形式で返す
    meta_data: リスト形式のmeta情報
    prompt_tokens: int形式の入力トークン合計量
    eval_tokens: int形式の出力トークン合計量
    total_tokens: int軽視の総トークン合計量
    """
    prompt_tokens = self.total_prompt_tokens
    eval_tokens = self.total_eval_tokens
    total_tokens = self.total_prompt_tokens + self.total_eval_tokens
    return {
      "meta_data": self.all_records,
      "prompt_tokens": prompt_tokens,
      "eval_tokens": eval_tokens,
      "total_tokens": total_tokens
    }
```

:::message
- 使用しているプロバイダーによって、出力されるmeta情報が異なります。
- Callbackカスタマイズを作成前に出力されるmeta情報の確認が必要になります。
- Ollama（ChatOllama）とGoogle（ChatGoogleGenerativeAI）ではtoken量を取り出す方法が異なります。
:::

> 第2のLLM（gemini-2.5-flash）による評価を実行

```py
## Google用のCallbackをセット
google_callback = GoogleMetadataCallback()

## LLM（gemini-2.5-flash）で評価を分析を実行
analysis = chain.invoke({
  "generated":  final_summary, # ←生成された概要
  "reference":  expected_text, # ←期待する概要
  "similarity": similarity # ←類似度
},config={"callbacks": [google_callback]})

## callbackの情報をmap形式で格納
meta_map["llm_call2-01"] = google_callback
```

# STTEP5:セマティック評価分析結果出力とトークン量を確認

## セマティック評価分析結果
```py
## summary内容出力
common_func.res_output_md(response=analysis,file_name=f"test03-summary-analysis")
```

> 分析結果を出力結果

https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/out/test03-summary-analysis.md

## トークン量を確認

> token_checkを呼び出す

```py
## LLMトークン量確認（callbackを格納したMap形式を引数に入れて実行）
common_func.token_check(meta_map)
```

> token_checkの出力結果

```txt
========================================
llm_call2-01
【📊 トークン集計】
呼び出し回数    : 1
合計処理時間    : 0.81分
入力トークン合計: 762
出力トークン合計: 1,747
総トークン合計  : 2,509
========================================
===================================
入力トークン:   762
出力トークン:   1,747
為替レート:     1 USD = ¥156.21
ベースモデル:    gemini-2.5-flash
料金 (USD):     $0.004596
料金 (JPY):     ¥0.72
===================================
```

## 2次評価でのトークン量と料金試算

|呼び出し回数|入力トークン|出力トークン|料金(JPY)|
|:----|:----|:----|:----|
|1|762|1747|￥0.72|