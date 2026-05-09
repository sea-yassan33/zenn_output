---
title: "概要生成とトークン管理"
---
# 概要生成とトークン管理について

以下の本文からLLM（gemma3:4b）モデルを使用して概要を生成します。

https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/data/paper01_maintext.md

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
from common_func import OllamaMetadataCallback
## LangChain系ライブラリ
from langchain_community.chat_models import ChatOllama
from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
```

## model(gemma3:4b)・定数の設定

- 以下の様にmodelと定数の設定を行います。

```py
## model
llm = ChatOllama(
  model="gemma3:4b",　# ←ローカルLLMを設定
  base_url=os.environ['OLLAM_URL'], # ←OLLAMAを呼びだせるURLを設定
  temperature=0.0　# ←一貫性のバランスを調整(0~1)
)
## 定数(LLMのmeta情報を格納するmapを準備)
meta_map = {}
```

# STEP1:テンプレートを準備

- 対象のテキストを分割する処理をこの後に実施しますが、その前にテンプレートを準備します
- その分割したテキスト（チャンク：意味のある小さな塊）毎に要点を生成するテンプレートを用意します。
- 各チャンク毎にまとめたものをセットし概要を生成できるようにテンプレートを用意します。

```py
## 各チャンク毎に要点を生成する際のテンプレート
map_prompt = ChatPromptTemplate.from_template("""
以下のテキストの要点を簡潔にまとめてください：

{chunk}

要点：
""")
## 最終的に概要生成する際のテンプレート
reduce_prompt = ChatPromptTemplate.from_template("""
以下の複数の要点をまとめて、論文全体の概要を500字以内で作成してください：

{summaries}

最終概要：
""")
```

# STEP2:テキストの分割処理

## テキストを読み込み

- 「TextLoader」を使用して論文の本文を読み込み、Documentオブジェクトに成形します。

```py
# from langchain_community.document_loaders import TextLoader

loader = TextLoader("./data/paper01_maintext.md", encoding="utf-8")
docs = loader.load() #　←テキストをDocumentオブジェクトとして整形
```

## テキスト分割方法を設定

- 「RecursiveCharacterTextSplitter」を使用してテキスト分割の設定を行います。

```py
# from langchain_text_splitters import RecursiveCharacterTextSplitter 

## テキストを分割設定
text_splitter = RecursiveCharacterTextSplitter(
  chunk_size=3000,       # トークンではなく文字数に注意
  chunk_overlap=300,     # 文脈の連続性を確保
  separators=[
    "\n## ", "\n### ",  # 論文の章区切りを優先
    "\n\n", "\n", "。", ".", " ", ""
  ]
)
```

## 分割設定を基に文章を分割

- 分割設定を基に文章を分割します。
- 分割後、いくつ分割（チャンク）されたか確認します。
- チャンク数に応じてLLMの呼び出し数が増えます

```py
## ドキュメントを分割
splitter_docs = text_splitter.split_documents(docs)
print(f'【チャンク数】{len(splitter_docs)}') #　←分割された数（チャンク）を確認
```

# STEP3:各チャンク毎に要点を生成

## Chainの作成
- 生成された要点を格納するリスト（空箱）を作成します。
- パイプを用いて「要点生成用テンプレート」→「モデル」→「出力パーサー」と処理を渡すchainを作成します。
- 出力パーサーの1つであるStrOutputParserは生成結果を文字列だけ返すように指示します。（meta情報も除かれます）

```py
# from langchain_core.output_parsers import StrOutputParser

chunk_summaries = []
## 要点生成用のChainの作成
chain = map_prompt | llm | StrOutputParser()
```

## 要点を生成

- Ollama用のCallbackを用意します。（初期化）
- チャンク毎にLLMを呼び出し処理を実行します。
- 生成された要点を格納します。
- callbackの情報をmeta_map（map形式）に格納します。

```py
## Ollama用のCallbackを初期化
callback = OllamaMetadataCallback()
for i, chunk in enumerate(splitter_docs):
  ## LLMを呼び出し処理を実行
  summary = chain.invoke({"chunk": chunk},config={"callbacks": [callback]})
  ## 生成結果（要点）を格納
  chunk_summaries.append(summary)
  print(f"チャンク {i+1}/{len(splitter_docs)} 完了")
## callbackの情報をmap形式で格納
meta_map["llm_call01"] = callback

## 各チャンクで生成した要点を統合
combined = "\n\n".join(chunk_summaries)
```

# STEP4:最終の概要を生成

- パイプを用いて「概要生成用テンプレート」→「モデル」→「出力パーサー」と処理を渡すchainを作成します。

```py
## 概要生成用のChainの作成
chain02 = reduce_prompt | llm | StrOutputParser()

## Ollama用のCallbackを初期化
callback = OllamaMetadataCallback()
## LLMを呼び出し処理を実行し、概要を生成
final_summary = chain02.invoke({"summaries": combined},config={"callbacks": [callback]})
## callbackの情報をmap形式で格納
meta_map["llm_call02"] = callback
```

# STEP5:生成コンテンツの出力とトークン量の確認

## 生成コンテンツを出力

- 生成されたコンテンツを次のフェーズ（評価）で使用できるようにMarkDown形式で出力します。
- [common_func.py](https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/common_func.py)内の「res_output_md」関数を使用して出力します


> common_func.py - res_output_md

```py
# from pathlib import Path
# import time

def res_output_md(response,dir_str=None,file_name=None):
  """
  markdownファイル出力
  """
  ## 格納するディレクトリを設定
  if (dir_str==None):
    dir_str = "out"
    dir_path = Path(f"./{dir_str}/")
    dir_path.mkdir(parents=True, exist_ok=True)
  else:
    dir_path = Path(f"./{dir_str}/")
    dir_path.mkdir(parents=True, exist_ok=True)
  ## ファイル名を設定
  if (file_name==None):
    now = datetime.now()
    formatted_time = now.strftime("%y%m%d-%H%M")
    file_name = f'test-{formatted_time}'
    ## markdown形式で出力（ファイル名の指定がない場合は時間を文字列にして保存）
    with open(f"./{dir_str}/{file_name}.md", "w", encoding="utf-8") as f:
      f.write(response)
  else:
    ## markdown形式で出力（指定したファイル名で保存）
    with open(f"./{dir_str}/{file_name}.md", "w", encoding="utf-8") as f:
      f.write(response)
```

```py
## 生成コンテンツ出力（評価時に使用）
common_func.res_output_md(response=final_summary,dir_str="data",file_name=f"test03-summary")
```

## 要約と概要を生成時に発生したトークン量を確認

- [common_func.py](https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/common_func.py)内の「token_check」を使用してトークン量のサマリーを出力

> common_func.py - token_check

```py
# import yfinance as yf

# 料金テーブル（USD / 1Mトークン）
PRICING = {
  # Google Gemini
  "gemini-3.1-pro":        {"input": 2.00,  "output": 12.00},
  "gemini-2.5-pro":        {"input": 1.25,  "output": 10.00},
  "gemini-2.5-flash":      {"input": 0.30,  "output": 2.50},
  "gemini-2.5-flash-lite": {"input": 0.10,  "output": 0.40},
  # OpenAI
  "gpt-5.4":               {"input": 2.50,  "output": 15.00},
  "gpt-5.4-mini":          {"input": 0.75,  "output": 4.50},
  "gpt-5.4-nano":          {"input": 0.20,  "output": 1.25},
  "gpt-4.1":               {"input": 5.00,  "output": 15.00},
  "gpt-4.1-mini":          {"input": 0.40,  "output": 1.60},
  # Anthropic Claude
  "claude-opus-4-7":       {"input": 5.00,  "output": 25.00},
  "claude-opus-4-6":       {"input": 5.00,  "output": 25.00},
  "claude-sonnet-4-6":     {"input": 3.00,  "output": 15.00},
  "claude-haiku-4-5":      {"input": 1.00,  "output": 5.00},
}
## LLMトークン量確認
def token_check (meta_map):
  """
  call_backのmeta情報からトークン量を出力
  """
  for call_neme, meta_data in meta_map.items():
    print("="*40)
    print(f"{call_neme}")
    meta_data.summary()
    cost = calc_cost(
      input_tokens=meta_data.meta_data().get("prompt_tokens"),
      output_tokens=meta_data.meta_data().get("eval_tokens"),
      model="gemini-2.5-flash"
    )
    print_cost_report(cost)
def calc_cost(input_tokens: int, output_tokens: int, model: str) -> dict:
  """
  トークン・コストデータ取得
  """
  price = PRICING.get(model, PRICING["gemini-2.5-flash"])
  input_cost_usd  = (input_tokens  / 1_000_000) * price["input"]
  output_cost_usd = (output_tokens / 1_000_000) * price["output"]
  total_cost_usd  = input_cost_usd + output_cost_usd
  # 為替
  usd_to_jpy = get_usd_to_jpy_yfinance()
  ## 直近の為替データから日本円で出力
  total_cost_jpy  = total_cost_usd * usd_to_jpy
  ## 結果を辞書形式で返す
  return {
    "referenc_model": model,
    "input_tokens":    input_tokens,
    "output_tokens":   output_tokens,
    "input_cost_usd":  input_cost_usd,
    "output_cost_usd": output_cost_usd,
    "total_cost_usd":  total_cost_usd,
    "usd_to_jpy":      usd_to_jpy,
    "total_cost_jpy":  total_cost_jpy,
  }
def get_usd_to_jpy_yfinance() -> float:
  """
  日米為替レート取得
  """
  try:
    # yfinanceのモジュールを使用して為替データを取得
    ticker = yf.Ticker("USDJPY=X")
    # 直近1日のデータを取得
    hist = ticker.history(period="1d")
    return round(float(hist["Close"].iloc[-1]), 2)
  except Exception as e:
    print(f"取得失敗: {e}")
    return 160.0
def print_cost_report(cost: dict):
  """
  トークン・コストレポート出力
  """
  print(f"{'='*35}")
  print(f"入力トークン:   {cost['input_tokens']:,}")
  print(f"出力トークン:   {cost['output_tokens']:,}")
  print(f"為替レート:     1 USD = ¥{cost['usd_to_jpy']:.2f}")
  print(f"ベースモデル:    {cost['referenc_model']}")
  print(f"料金 (USD):     ${cost['total_cost_usd']:.6f}")
  print(f"料金 (JPY):     ¥{cost['total_cost_jpy']:.2f}")
  print(f"{'='*35}")
```

> token_checkを呼び出す

```py
## LLMトークン量確認（callbackを格納したMap形式を引数に入れて実行）
common_func.token_check(meta_map)
```

> token_checkの出力結果

```
========================================
llm_call01 ←　1・2回目の呼び出し
【📊 トークン集計】
呼び出し回数    : 2
合計処理時間    : 0.38分
入力トークン合計: 2,277
出力トークン合計: 598
総トークン合計  : 2,875
========================================
===================================
入力トークン:   2,277
出力トークン:   598
為替レート:     1 USD = ¥156.41
ベースモデル:    gemini-2.5-flash
料金 (USD):     $0.002178
料金 (JPY):     ¥0.34
===================================
========================================
llm_call02　←　3回目の呼び出し
【📊 トークン集計】
呼び出し回数    : 1
合計処理時間    : 0.19分
入力トークン合計: 631
出力トークン合計: 450
総トークン合計  : 1,081
========================================
===================================
入力トークン:   631
出力トークン:   450
為替レート:     1 USD = ¥156.41
ベースモデル:    gemini-2.5-flash
料金 (USD):     $0.001314
料金 (JPY):     ¥0.21
===================================
```

## 今回LLM呼び出された際のトークン量

|呼び出し回数|入力トークン|出力トークン|料金(JPY)|
|:----|:----|:----|:----|
|3|2,908|1,048|￥0.55|