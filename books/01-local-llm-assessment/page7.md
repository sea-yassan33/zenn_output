---
title: "まとめ"
---

# 総括

> 今回、各セクションで使用されたトークン量と料金（gemini-2.5-flashベース）は下記のとおりです。

|セクション|呼び出し回数|入力トークン|出力トークン|料金(JPY)|
|:----|:----|:----|:----|:----|
|概要生成|3|2,908|1,048|￥0.55|
|2次評価|1|762|1,747|￥0.72|
|3次評価|6|10,388|8,523|￥3.81|
||||||
|合計|10|14,058|11,318|￥4.37|


> 今回、論文の概要生成から3次評価までLLMモデルを10回呼び出し、**￥4.37**かかる試算となりました。

> 1次評価でテンプレートは500文字以内と書いてるのに実際は820文字出力されている結果となりました。テンプレートの調整や文章校正用のLLMが必要と思われます

> 3次評価の結果において参照テキストと比較して正確性の観点で「合格」判定となりました。

> 様々な評価指標があるため、何を目的に評価をするか明確にし、それに応じて評価方法（パラメータ）を調整する必要と思われます。

>　最後まで読んで頂きありがとうございます。ローカルLLM開発のヒントになれば幸いです。 

# 作成したコード

> 共通関数

https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/common_func.py

> 概要生成

https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/01_%E6%A6%82%E8%A6%81%E7%94%9F%E6%88%90_%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E7%AE%A1%E7%90%86.py

> 1・2次評価

https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/02_1-2%E6%AC%A1%E8%A9%95%E4%BE%A1.py

> 3次評価

https://github.com/sea-yassan33/output_creativity/blob/main/python/03_local_llm_assessment/03_3%E6%AC%A1%E8%A9%95%E4%BE%A1.py


