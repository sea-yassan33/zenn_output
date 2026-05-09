---
title: "1次評価"
---
# 1次評価について

- 1次評価は機械学習やLLMを使用せずにあらかじめ定義したルールベースを基に評価します。
- 今回は簡略的に文字数で評価します。

# STEP0:準備
## モジュールの読み込み

- 以下のモジュールを読み込みます。

```py
import warnings
# 警告を非表示にする
warnings.filterwarnings('ignore')
import sys
sys.dont_write_bytecode = True
```

# STEP1：ルールベースの評価関数を用意

- 下記の様にルールベースの評価関数を作成します。

```py
def evaluation_analysis01(target_text: str, expected_text: str):
  """
  target_text: 生成したコンテンツ
  expected_text: 期待されるコンテンツ
  """
  # --- １次評価: テキスト長の比較 ---
  gen_len = len(target_text)
  ref_len = len(expected_text)
  ratio   = gen_len / ref_len if ref_len > 0 else 0
  print(f"\n【長さの比較】")
  print(f"  生成: {gen_len}文字 / 参照: {ref_len}文字 (比率: {ratio:.2f})")
  if ratio < 0.5:
    print("  ⚠️ 原因候補: 生成が短すぎる → 情報が欠落している可能性")
  elif ratio > 2.0:
    print("  ⚠️ 原因候補: 生成が長すぎる → 余分な情報が含まれている可能性")
  else:
    print("  ✅ 長さは適切")
  print("\n" + "="*50)
```

# STEP2:生成された概要と評価用の概要を準備

- 生成された概要と評価用の概要を準備します。

```py
## 生成された概要
with open("./data/test03-summary.md", "r", encoding="utf-8") as f:
  final_summary = f.read()
## 評価用の概要
with open("./data/paper01_oberveiw.md", "r", encoding="utf-8") as f:
  expected_text = f.read()
```

# STEP3:評価結果を出力

```py
## 評価を実行し、結果を出力
evaluation_analysis01(final_summary,expected_text)
```

> 出力内容

```txt
==================================================
📋 評価の分析Part1
==================================================

【長さの比較】
  生成: 820文字 / 参照: 454文字 (比率: 1.81)
  ✅ 長さは適切　
```

- 比率が0.5～2.0の範囲に収まるため「適切」と判定としております。（比率範囲を～（500/454）までにすれば不適合になります。）
- テンプレートは500文字以内と書いてるのに実際は820文字出力されている為、テンプレートの調整が必要になります。
- テンプレートでも調整が効かない場合はモデルの変更や新たに文章校正用のLLMを作成して添削してもらう必要があります。