---
title: "概要"
---

ローカルLLMでLangChainを使用して、生成処理中の処理状況管理や生成コンテンツに対しての評価についてのドキュメントになります。

> 以下の管理や評価を行います。

- **管理：CallbackHandler**
  - CallbackHandlerを用いてLLMの呼び出し管理やトークン管理を行います。
- **1次評価**
  - ルールベース評価として文字数に対しての評価を行います。
- **2次評価**
  - セマンティック評価としてエンベンディングを使用したベクトル評価を行います。
- **3次評価**
  - 第2のモデルが生成コンテンツを評価する「LLM-as-a-judge」を用いて評価を行います。

今回、論文【[Amplifiers or Equalizers? A Longitudinal Study of LLM Evolution in Software Engineering Project-Based Learning](https://arxiv.org/abs/2511.23157)】を基にGoogle翻訳した内容をベースに「概要生成」・「1次評価」・「2次評価」・「3次評価」を行います。また、LLMが呼ばれた際のトークン管理を行います。

