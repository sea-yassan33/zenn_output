---
title: "管理：CallbackHandler"
---
# CallbackHandlerとは
- LangChainのCallbackHandlerは、LLMアプリケーションの実行中に発生する様々なイベントに対して独自の処理をします。
- プログラム内部で何が起きているかをリアルタイムで監視・記録が可能になります。

# CallbackHandlerの主な役割

主な役割は以下の通りになります。
- ロギングと監視
- ストリーミング
- デバッグ
- コスト管理
- BaseCallbackHandlerを使用してカスタマイズ

# メソッドについて

CallbackHandlerで使用するメソッドは下記の通りになります。

|メソッド名|実行されるタイミング|
|:----|:----|
|on_llm_start|LLMが処理を開始したとき|
|on_llm_new_token|新しいトークンが生成されたとき（ストリーミング時）|
|on_llm_end|LLMの処理が完了したとき|
|on_chain_start|Chain（一連の処理）が開始されたとき|
|on_tool_start|外部ツール（検索など）が呼び出されたとき|
|on_error|エラーが発生したとき|

> その他、カスタマイズでメソッドを追加する事が可能です。

# カスタマイズについて

[common_func.py](https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/common_func.py)内の「OllamaMetadataCallbackクラス」を例にカスタマイズ方法を説明します。

```py
from langchain_core.callbacks.base import BaseCallbackHandler　#←ライブラリを読み込みます

# ▼▼▼ Ollam_Callback_Class ▼▼▼
class OllamaMetadataCallback(BaseCallbackHandler):
  """
  Ollama専用callback
  """
  ## 初期化時の処理
  def __init__(self, window=5):
    self.call_count = 0
    self.str_count = 0
    self.total_prompt_tokens = 0
    self.total_eval_tokens = 0
    self.elapsed_min = 0
    self.start_time = None
    self.all_records = []
    self.durations = []
    self.window = window　#←　経過時間を測る際に使用（デフォルト：直近5回）
  ## LLMが処理を開始時の処理
  def on_llm_start(self, serialized, prompts, **kwargs):
    if self.call_count == 0:
      self.start_time = time.time()　#←開始時間をセット
    self.str_count += len(prompts[0]) 
  ## LLMの処理が完了時の処理
  def on_llm_end(self, response, **kwargs):
    self.call_count += 1
    for generations in response.generations:
      for gen in generations:
        ## meta情報
        meta = gen.message.response_metadata
        self.all_records.append(meta) #←LLM処理完了時のmeta情報を格納
        ## トークン情報
        prompt_tokens = meta.get("prompt_eval_count", 0) #← 入力トークン量
        eval_tokens   = meta.get("eval_count", 0) #←　出力トークン量
        self.total_prompt_tokens += prompt_tokens
        self.total_eval_tokens   += eval_tokens
        ## 処理時間
        duration_min = meta.get("total_duration", 0) / 1_000_000_000 / 60　#←　処理時間を分単位で取得
        self.durations.append(duration_min)
        self.elapsed_min = (time.time() - self.start_time) / 60　#←　経過時間を分単位で取得（複数LLMが呼ばれる際に必要）
        recent = self.durations[-self.window:]
        avg = sum(recent) / len(recent)
        progress   = f"{self.call_count:2d}"
        print(
          f"[呼び出し #{self.call_count}] 入力: {prompt_tokens} / 出力: {eval_tokens}\n"
          f"【合計文字数】{self.str_count}"
          f"【{progress}】今回: {duration_min:.2f}分 |経過: {self.elapsed_min:.2f}分 | 直近{len(recent)}回平均: {avg:.2f}分/回"
        )
  ## summaryメソッドが呼ばれた際の処理（カスタム関数）
  def summary(self):
    """
    トークン量の使用量を出力
    """
    total = self.total_prompt_tokens + self.total_eval_tokens
    print("\n" + "="*40)
    print("📊 トークン集計")
    print("="*40)
    print(f"呼び出し回数    : {self.call_count}")
    print(f"合計処理時間    : {self.elapsed_min:.2f}分")
    print(f"入力トークン合計: {self.total_prompt_tokens:,}")
    print(f"出力トークン合計: {self.total_eval_tokens:,}")
    print(f"総トークン合計  : {total:,}")
    print("="*40)
  ## meta_dataメソッドが呼ばれた際の処理（カスタム関数）
  def meta_data(self):
    """
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

# Callbackの使用例

下記の様にどのタイミングで管理するかで、Callbackをセットするパターンが異なります。

```py
# パターン1:LLM呼び出し時に組み込む
callback = OllamaMetadataCallback()
result = chain.invoke(
  {"input_documents": sp_docs}
  ,config={"callbacks": [callback]} #← ここにCallbackをセット
)

#パターン2:LLM初期化時に組み込む
google_callback = GoogleMetadataCallback()
llm_api_gemini = ChatGoogleGenerativeAI(
  model="gemini-2.5-flash",
  google_api_key=os.environ['GOOGLE_AI_ST_API'],
  temperature=0,
  callbacks=[google_callback] #← ここにCallbackをセット
)
```

:::message
本プログラム内ではLLM呼び出し時直前にCallbackをセットします。（パターン1）
状況によってはパターン2を使用します。
:::