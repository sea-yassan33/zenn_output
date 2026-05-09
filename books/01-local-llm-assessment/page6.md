---
title: "3次評価"
---

# 3次評価で使用する評価について

- 3次評価は「LLM-as-a-judge」を使用して評価します。

## LLM-as-a-judgeについて

人間や従来のプログラムの代わりに、LLM（大規模言語モデル）を使って他のLLMの出力を評価・採点させる手法です

> LangChainを用いた「LLM-as-a-judge(load_evaluator)」使用例

```py
from langchain_classic.evaluation import load_evaluator

# 【labeled_criteria評価でパターン】
evaluator = load_evaluator(
  "labeled_criteria",　# 第一引数（必須）：評価器の種類
  llm=llm,　# モデルを指定
  criteria="correctness" # 「labeled_criteria」設定時に組み込み基準をいれる
)

# 【複数の評価指標を指定するパターン】
## 評価指標をmap形式で用意する
criteria_list = {
  "conciseness":  "概要は簡潔にまとめられているか？",
  "completeness": "重要なポイントが網羅されているか？",
}
## 評価結果を格納する箱(List形式)を用意
result_criteria_list = []
for name, description in criteria_list.items():
  evaluator = load_evaluator(
    "criteria",　# ←入力＋予測のみで評価
    llm=llm_api_gemini,
    criteria={name: description}# ←評価基準を格納
  )
  ## 設定評価基準毎に評価を実施
  result = evaluator.evaluate_strings(
    prediction=generated_summary,
    input=paper_text
  )
  ## 評価結果を格納
  result_criteria_list.append(result)
```

> 第一引数の値について

|設定値|用途|
|:----|:----|
|labeled_criteria|参照テキスト(reference)も使って評価|
|criteria|入力＋予測のみで評価（参照テキスト不要）|
|embedding_distance|Embeddingベースの類似度評価|
|pairwise_criteria|モデルA vs モデルBの出力比較|
|qa|Q&A形式の正確性評価,reference必須,参照テキストとの比較|
|cot_qa|Chain-of-Thought付きQ&A評価|

> 評価器の種類load_evaluator時、criteriaのパラメータについて

|設定値|用途|
|:----|:----|
|correctness|参照テキストと比較して正確か判断。期待されるデータを入れる必要あり。（labeled_criteria専用）|
|conciseness|簡潔にまとめられているか|
|completeness|重要なポイントが網羅されているか|
|relevance|入力に対して関連性があるか|
|coherence|論理的な流れ・一貫性があるか|
|helpfulness|役に立つ内容か|
|depth|内容に深みがあるか|
|harmfulness|有害な内容が含まれていないか|
|maliciousness|悪意のある内容が含まれていないか|
|controversiality|議論を呼ぶ内容か|


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
import pandas as pd
import common_func
from common_func import GoogleMetadataCallback
from common_func import OllamaMetadataCallback
## LangChain系ライブラリ
from langchain_community.chat_models import ChatOllama
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_classic.evaluation import load_evaluator
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
```

- ローカルLLM・定数を準備

```py
## model
llm = ChatOllama(model="gemma3:4b",base_url=os.environ['OLLAM_URL'],temperature=0.0)
## 定数
meta_map = {}
result_map ={}
```

# STEP1:論文の本文・期待する概要・生成された概要を読み込む

- 下記の様に必要なデータを読み込みます。

```py
## 論文の本文
with open("./data/paper01_maintext.md", "r", encoding="utf-8") as f:
  paper_text = f.read()
## 期待する概要
with open("./data/paper01_oberveiw.md", "r", encoding="utf-8") as f:
  reference_summary = f.read()
## 生成された概要
with open("./data/test03-summary.md", "r", encoding="utf-8") as f:
  generated_summary = f.read()
```

# STEP2:labeled_criteria評価＜参照テキスト(reference)を用いる＞

> load_evaluatorの第1引数の値

|設定値|用途|
|:----|:----|
|labeled_criteria|参照テキスト(reference)も使って評価|

> load_evaluatorの第3引数(criteria)の値

|設定値|用途|
|:----|:----|
|correctness|参照テキストと比較して正確か判断。期待されるデータを入れる必要あり。（labeled_criteria専用）|

## labeled_criteria評価を実行

```py
## google用callbackを用意(初期化)
google_callback = GoogleMetadataCallback()
## 第2のLLMモデルを用意
llm_api_gemini = ChatGoogleGenerativeAI(
  model="gemini-2.5-flash",
  google_api_key=os.environ['GOOGLE_AI_ST_API'],
  temperature=0,
  callbacks=[google_callback]
)
## labeled_criteria評価用に各種パラメータをセット
labeled_evaluator = load_evaluator(
  "labeled_criteria", # 評価器の種類
  llm=llm_api_gemini, # 評価に使うLLM
  criteria="correctness" 
)
## labeled_criteria評価を実行
labeled_result = labeled_evaluator.evaluate_strings(
  prediction=generated_summary, # ←生成された概要
  input=paper_text, # ←論文の本文
  reference=reference_summary, # ←期待する概要
)
## callbackの情報をmap形式で格納
meta_map["llm_call3-01"] = google_callback
```

## labeled_criteria評価結果を翻訳

- 「labeled_evaluator.evaluate_strings」の結果（value）は英文で返される為、日本文にするは翻訳する処理を追加する必要があります。

> 翻訳用テンプレートを作成

```py
trans_prompt = ChatPromptTemplate.from_template(
  "以下を自然な日本語に翻訳してください:\n{text}"
)
```

> 翻訳を実行し、評価結果を格納

```py
## ollama用callbackを用意(初期化)
ollma_callback = OllamaMetadataCallback()
## ローカルLLM（gemma3:4b）でChainを作成
chain = trans_prompt | llm | StrOutputParser()
## 翻訳を実行
trans_result = chain.invoke(
  {"text": labeled_result['reasoning']},
  config={"callbacks": [ollma_callback]}
)
## callbackの情報をmap形式で格納
meta_map["llm_call3-02"] = ollma_callback

## スコアとlabeled_criteria評価結果の翻訳した値をmap形式で格納
result01 = [labeled_result['score'],labeled_result['value'],trans_result]
result_map["result01"] =result01
```

# STEP3:criteria評価＜入力＋予測のみで評価＞

> load_evaluatorの第1引数の値

|設定値|用途|
|:----|:----|
|criteria|入力＋予測のみで評価（参照テキスト不要）|

> load_evaluatorの第3引数(criteria)の値

|設定値|用途|
|:----|:----|
|conciseness|簡潔にまとめられているか|
|completeness|重要なポイントが網羅されているか|

## criteria評価を実行

```py
## google用callbackを用意(初期化)
google_callback = GoogleMetadataCallback()
## 第2のLLMモデルを用意(再度初期化、前回実施時のcalllbackが引き継がれる為)
llm_api_gemini = ChatGoogleGenerativeAI(
  model="gemini-2.5-flash",
  google_api_key=os.environ['GOOGLE_AI_ST_API'],
  temperature=0,
  callbacks=[google_callback]
)
## 評価基準をリスト形式で格納
criteria_list = {
  "conciseness":  "概要は簡潔にまとめられているか？",
  "completeness": "重要なポイントが網羅されているか？",
}
## 評価結果を格納する箱（List形式）を準備
result_criteria_list = []

for name, description in criteria_list.items():
  evaluator = load_evaluator("criteria",　# ←入力＋予測のみで評価
    llm=llm_api_gemini,
    criteria={name: description}　# ←評価基準を格納
  )
  # 評価基準毎に評価を実施
  result = evaluator.evaluate_strings(
    prediction=generated_summary,
    input=paper_text
  )
  result_criteria_list.append(result) # ←評価結果を格納

## callbackの情報をmap形式で格納
meta_map["llm_call3-03"] = google_callback
```

## criteria評価結果を翻訳

> 翻訳を実行

```py
## ollama用callbackを用意(初期化)
ollma_callback = OllamaMetadataCallback()
## ローカルLLM（gemma3:4b）でChainを作成
chain = trans_prompt | llm | StrOutputParser()
## 翻訳結果を格納する箱(List形式)を用意
trans_result_list = []
for result_criteria in result_criteria_list:
  ## 翻訳を実行
  trans_result = chain.invoke(
    {"text": result_criteria['reasoning']},
    config={"callbacks": [ollma_callback]}
  )
  ## 翻訳結果を格納
  trans_result_list.append(trans_result)

## callbackの情報をmap形式で格納
meta_map["llm_call3-04"] = ollma_callback
```

> 繰り返し処理で評価結果を格納

```py
## スコアとcriteria評価結果の翻訳した値をmap形式で格納
## concisenessとcompletenessの評価を行ったので繰り返し処理で格納
for i in range(len(result_criteria_list)):
  result_temp = [result_criteria_list[i]['score'],result_criteria_list[i]['value'],trans_result_list[i]]
  map_key = f"result0{i+2}"
  result_map[map_key] = result_temp
```

# STEP4:評価結果の出力とトークン量の確認

> 評価結果（map形式）をDataframeでまとめる

```py
# map型からDataframeに変換
df = pd.DataFrame.from_dict(
  result_map,
  orient='index',
  columns=['score', 'value', 'reasoning']
).reset_index().rename(columns={'index': 'result_id'})

# 評価種類のカラムを追加
label_map = {
  'result01': 'reference評価',
  'result02': 'criteria評価（簡潔さ）',
  'result03': 'criteria評価（完全性）',
}
df['evaluation_label'] = df['result_id'].map(label_map)
```

> 評価結果出力サマリ用(marakdown)の関数を用意

```py
def print_evaluation_report_markdown(df: pd.DataFrame) -> str:
  """
  サマリーをmarkdown形式で出力
  """
  lines = []
  # ============================================================
  # ヘッダー
  # ============================================================
  lines.append("# 📋 論文概要 評価レポート\n")
  # 合否サマリーテーブル
  lines.append("## 評価サマリー\n")
  lines.append("| 評価項目 | スコア | 合否 | 判定 |")
  lines.append("|----------|--------|------|------|")
  for _, row in df.iterrows():
    judge = '✅ 合格' if row['score'] == 1 else '❌ 不合格'
    lines.append(
      f"| {row['evaluation_label']} "
      f"| {row['score']} "
      f"| {row['value']} "
      f"| {judge} |"
    )
  total = df['score'].sum()
  lines.append(f"\n**総合スコア: {total} / {len(df)}**\n")
  lines.append("---\n")
  # ============================================================
  # 各評価の詳細
  # ============================================================
  lines.append("## 評価詳細\n")
  for _, row in df.iterrows():
    label     = row['evaluation_label']
    score     = row['score']
    value     = row['value']
    reasoning = row['reasoning']
    # reasoningの前置き文を除去
    skip_prefixes = [
      "以下のように自然な日本語に翻訳しました",
      "以下は、提示された内容をより自然な日本語に翻訳したものです",
      "以下は、提供されたテキストをより自然な日本語に翻訳したものです",
      "---",
      "**注記:**",
    ]
    cleaned_lines = []
    for line in reasoning.strip().split('\n'):
      if any(line.strip().startswith(p) for p in skip_prefixes):
        continue
      cleaned_lines.append(line)
    cleaned_reasoning = '\n'.join(cleaned_lines).strip()
    judge = '✅ 合格' if score == 1 else '❌ 不合格'
    lines.append(f"### 【{label}】{judge}\n")
    lines.append(f"| 項目 | 内容 |")
    lines.append(f"|------|------|")
    lines.append(f"| 評価スコア | `{score}` |")
    lines.append(f"| 合格/不合格 | `{value}` |")
    lines.append(f"\n#### 評価内容\n")
    lines.append(cleaned_reasoning)
    lines.append("\n---\n")
  return '\n'.join(lines)
```

> 評価結果出力サマリ用(marakdown)の関数を実行

```py
## サマリーを出力
md_text = print_evaluation_report_markdown(df)
## marakdown形式でファイル出力
common_func.res_output_md(response=md_text,file_name=f"test04-assesment_summary")
```

> 出力結果

https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/out/test04-assesment_summary.md

## トークン量を確認

> token_checkを呼び出す

```py
## LLMトークン量確認（callbackを格納したMap形式を引数に入れて実行）
common_func.token_check(meta_map)
```

> token_checkの出力結果

```txt
========================================
llm_call3-01
【📊 トークン集計】
呼び出し回数    : 1
合計処理時間    : 0.17分
入力トークン合計: 3,036
出力トークン合計: 1,745
総トークン合計  : 4,781
========================================
===================================
入力トークン:   3,036
出力トークン:   1,745
為替レート:     1 USD = ¥156.37
ベースモデル:    gemini-2.5-flash
料金 (USD):     $0.005273
料金 (JPY):     ¥0.82
===================================
========================================
llm_call3-02
【📊 トークン集計】
呼び出し回数    : 1
合計処理時間    : 0.29分
入力トークン合計: 340
出力トークン合計: 400
総トークン合計  : 740
========================================
===================================
入力トークン:   340
出力トークン:   400
為替レート:     1 USD = ¥156.37
ベースモデル:    gemini-2.5-flash
料金 (USD):     $0.001102
料金 (JPY):     ¥0.17
===================================
========================================
llm_call3-03
【📊 トークン集計】
呼び出し回数    : 2
合計処理時間    : 0.47分
入力トークン合計: 5,591
出力トークン合計: 5,016
総トークン合計  : 10,607
========================================
===================================
入力トークン:   5,591
出力トークン:   5,016
為替レート:     1 USD = ¥156.37
ベースモデル:    gemini-2.5-flash
料金 (USD):     $0.014217
料金 (JPY):     ¥2.22
===================================
========================================
llm_call3-04
【📊 トークン集計】
呼び出し回数    : 2
合計処理時間    : 0.48分
入力トークン合計: 1,421
出力トークン合計: 1,362
総トークン合計  : 2,783
========================================
===================================
入力トークン:   1,421
出力トークン:   1,362
為替レート:     1 USD = ¥156.37
ベースモデル:    gemini-2.5-flash
料金 (USD):     $0.003831
料金 (JPY):     ¥0.60
===================================
```

## 今回LLM呼び出された際のトークン量

|呼び出し回数|入力トークン|出力トークン|料金(JPY)|
|:----|:----|:----|:----|
|6|10,388|8,523|￥3.81|