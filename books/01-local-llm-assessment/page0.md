---
title: "概要"
---
ローカルLLMでLangChainを使用して、生成処理中の処理状況管理や生成コンテンツに対しての評価についてまとめます。

> 以下の管理や評価を行います。

- **CallbackHandler**
  - LLMの呼び出し管理やトークン管理を実装する事が可能になります。

- **一次評価**
  - ルールベース評価として文字数に対しての評価を行います。
- **二次評価**
  - セマンティック評価を行い、エンベンディングを使用してベクトル評価を行います。
- **三次評価**
  - 第二のモデルが生成コンテンツを評価する「LLM-as-a-judge」を用いて評価を行います。

今回、論文【[Amplifiers or Equalizers? A Longitudinal Study of LLM Evolution in Software Engineering Project-Based Learning](https://arxiv.org/abs/2511.23157)】を基にGoogle翻訳した内容をベースに概要生成から三次評価を行います。

