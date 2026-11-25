---
title: "GPT-6.2 AstraとSol: どちらを先に試すか、会話の先頭を触らない理由"
emoji: "🤖"
type: "tech"
topics: ["openai", "gpt", "responses-api", "prompt-caching", "ai"]
published: true
---
本記事は System Design Simple の英語記事の日本語版です。原文: https://systemdesignsimple.com/gpt-6-2-astra-vs-sol/

2026年11月17日、OpenAIのAPIにGPT-6.2のモデルが2つ加わりました。Astraは重い推論とコーディング、Solは日常のコーディングとエージェント向けです。制限もツールも同じです。ただ、今回のリリースにはもう1つの変化があります。Responses APIでは、会話の先頭を編集すると、モデルの推論が次のターンへ引き継がれなくなります。安定したプロンプトの先頭を、そのままにしておくことが大事になりました。


## 用語の定義
**推論アイテム(reasoning item)**: Responses APIの出力アイテムの1つで、モデルの推論の状態を持ちます。次のターンへ渡せます。推論そのものが、ふつうの読みやすいテキストとして返ってくるわけではありません。

**プロンプトプレフィックス(prompt prefix)**: レンダリングされたリクエストの先頭部分です。ツール定義、指示、会話アイテムのうち、以前のリクエストと同一の部分を指します。

**プロンプトキャッシュ(prompt cache)**: そうした先頭部分について、すでに済ませた処理をプロバイダー側で保存しておく仕組みです。あとからのリクエストは、もう一度処理する代わりに読み出せます。

## GPT-6.2 AstraとSolの新しいところ
OpenAIは、同じ制限とツールを持つ推論モデルを2つ加えました。2026年11月17日のAPI changelogと、2つのモデルページ(2026年11月19日確認)によります。

|  | GPT-6.2 Astra | GPT-6.2 Sol |
|---|---|---|
| モデルID | `gpt-6.2-astra` | `gpt-6.2-sol` |
| OpenAIの位置づけ | 重い推論とコーディング | 日常のコーディングとエージェント |
| コンテキストウィンドウ | 1,050,000トークン | 1,050,000トークン |
| 最大入力 / 出力 | 922,000 / 128,000トークン | 922,000 / 128,000トークン |
| 知識カットオフ | 2026年5月18日 | 2026年6月1日 |
| モダリティ | テキストと画像を入力、テキストを出力 | テキストと画像を入力、テキストを出力 |
| エンドポイント | Responses、Chat Completions | Responses、Chat Completions |

2つのページに載るResponses APIのツールは同じです。ウェブ検索、ファイル検索、画像生成、コードインタープリター、ホストシェル、パッチ適用、スキル、コンピューター利用、MCP、ツール検索の10種類です。changelogは、エージェントを作る人へのルールを1つ加えています。ツール呼び出しはResponses API経由でなければなりません。

![OpenAIの開発者ドキュメントにあるGPT-6.2 Astraのモデルページ。ソフトウェアの構築と改修、コンピューターの利用、テキストと画像の入力、1,050,000トークンのコンテキストウィンドウ、最大出力128,000トークン、知識カットオフ2026年5月18日](https://systemdesignsimple.com/img/posts/gpt-6-2-astra-vs-sol/model-limits.webp)

## 会話を編集すると推論はどうなるか
先頭のターンを編集すると、Responses APIは推論アイテムを先へ運ぶのをやめます。[GPT-6.2移行ガイド](https://developers.openai.com/api/docs/guides/latest-model?model=gpt-6.2-astra#gpt-6.2-astra-migration-quickstart)によります。きっかけは3つです。先頭の履歴への編集、ダウングレードを含む一部のモデル変更、エンドユーザーや組織の変更です。推論が落ちたあとは、モデルが先の作業をもう一度やるために、トークンを余分に使うことがあります。

コードレビューのエージェントで考えてみましょう。ターン1では、レビューのルールを書いたdeveloperメッセージを送り、つぎにdiffを送ります。モデルは推論し、ツールを呼び、答えます。ターン2では、先の推論を2つの方法で引き継げます。保存されたターン1のレスポンスを`previous_response_id`で指すか、推論アイテムを含む先の入出力アイテムを、元の順番のままコード側で送るかです。あとでdeveloperメッセージのタイプミスを直して履歴を作り直すと、それは先頭への編集です。ガイドのルールにより、ターン1の推論は落ちます。

プロンプトキャッシュは、また別の層です。リクエストの先頭について計算済みのキーと値の状態を保存しますが、推論アイテムは会話の中の出力アイテムです。[キャッシュのガイド](https://developers.openai.com/api/docs/guides/prompt-caching)によれば、リクエストは使えるブレークポイントをたどりながら、いちばん長いキャッシュ済みの先頭を探します。再利用できるのは、そこまでのレンダリング内容が一致するときだけです。つまりタイプミスの修正は、変わったところから先の再利用を止めます。より前のブレークポイントは使えるままなので、ヒットする可能性は残ります。保証はありません。

|  | 推論アイテム | プロンプトキャッシュ |
|---|---|---|
| 残る条件 | 先頭の履歴が変わらず、そのまま渡され、モデル・ユーザー・組織の変更がない | レンダリングされた先頭が、使えるブレークポイントまで一致する |
| 先頭を編集したあと | 落ちる | 編集したところから先は再利用なし。より前のブレークポイントはまだ合うことがある |
| 読める信号 | ガイドに直接の信号は載っていない | `usage.input_tokens_details.cached_tokens` |

会話は、追加だけのログとして扱うのが無難です。新しいアイテムは末尾に足し、古いものはそのままにします。誤って貼ったシークレットを消すなど、編集がどうしても必要なら、ガイドのルールが適用されます。推論アイテムは落ち、キャッシュの再利用は、より前の一致するブレークポイントまで届くことがあります。

もう1つの変化は、置き場所です。GPT-6.2から、トップレベルの`instructions`は`input`のあとに付けられ、キャッシュできる先頭の外に出ます。ガイドでは、再利用したい指示は、最初の入力アイテムとして`developer`メッセージに入れるよう勧めています。繰り返しのコンテキストを毎回処理し直す理由は、[ロングコンテキストとRAG](https://systemdesignsimple.com/long-context-vs-search-rag/)にあります。

## APIでの使い方
モデルIDを変え、ツールにはResponses APIを使い、推論の努力量を設定します。移行ガイドの手順です。

1. `model`を`gpt-6.2-sol`か`gpt-6.2-astra`にします。
2. ツールを使う呼び出しはResponses APIにします。
3. 推論の努力量を設定します。新しい統合は`medium`から、以前に`none`や`minimal`を使っていたなら`low`にします。フィールドはResponsesでは`reasoning.effort`、Chat Completionsでは`reasoning_effort`です。
4. Responsesでは`temperature`、`top_p`、`top_logprobs`を省き、`include`から`message.output_text.logprobs`を外します。Chat Completionsでも`logprobs`を省きます。
5. 安定した指示は、最初の`developer`メッセージへ移します。
6. GPT-5.5以前から移る場合は、`prompt_cache_retention`を`prompt_cache_options.ttl`に置き換え、`"30m"`に設定します。

`POST /v1/responses`への最初のターンです。

```json
{
  "model": "gpt-6.2-sol",
  "reasoning": { "effort": "medium" },
  "tools": [{ "type": "web_search" }],
  "input": [
    { "role": "developer", "content": "You review pull requests. Reply with a numbered list." },
    { "role": "user", "content": "Review this diff: <paste the diff>" }
  ]
}
```

2ターン目は、保存されたレスポンスから続きます。先のアイテムは、推論を含めて変わらず、順番もそのままです。

```json
{
  "model": "gpt-6.2-sol",
  "reasoning": { "effort": "medium" },
  "tools": [{ "type": "web_search" }],
  "previous_response_id": "<id from the first response>",
  "input": [{ "role": "user", "content": "Now check the tests." }]
}
```

この短い例は続き方を示すもので、キャッシュに当たるとは限りません。キャッシュのガイドによれば、再利用できる先頭には、GPT-5.6以降で少なくとも1,024の可視トークンが必要で(2026年11月19日確認)、一致するキャッシュエントリがなければなりません。トップレベルの`instructions`は`previous_response_id`経由では引き継がれません。[Responses APIのリファレンス](https://developers.openai.com/api/reference/resources/responses/methods/create)によります。developerメッセージに入れておきましょう。`cached_tokens`が測るのはキャッシュの再利用だけで、推論アイテムが残ったかどうかではありません。

## どちらを先に試すか
エージェントのループやふだんのコーディングは、まずSolで試すのがいいでしょう。むずかしい計画やデバッグのステップにAstraを使う、というのがOpenAIの位置づけです。どちらのページも、自分の仕事でどうなるかは言っていません。実際のリクエストを10件から20件、両方で動かしてみてください。ガイドによれば、一部のモデル変更で推論アイテムが落ちるので、Astraのステップは新しい会話で始めるほうが、設計としては単純です。仕事を別々のコンテキストに分ける考え方は、[サブエージェント](https://systemdesignsimple.com/claude-code-subagents-explained/)と同じです。

## いまのAPIの制限
両モデルともテキストと画像を入力し、テキストだけを返します。推論は切れません。リクエストとトークンのレート制限は、アカウントの利用ティアで決まり、各モデルページに載っています。

- 音声や動画の入出力はありません。
- 推論の努力量`none`はAPIが拒否します。
- Chat Completionsは動きますが、ツール呼び出しはできません。
- ダウングレード以外に、どのモデル変更が推論アイテムを落とすか、ガイドは言っていません。

## 私たちの見解
GPT-6.2は、モデルを2つに分けたというより、会話の扱い方を変えたリリースです。Solから試し、Astraは重いステップに取っておく。10件から20件の自分のリクエストで両方を比べる。そして会話は追加だけのログにする。先頭を触るたびに、推論とキャッシュの両方が、そこから先を使えなくなります。トップレベルの`instructions`に頼っていたコードは、最初のdeveloperメッセージへ移すところから直しましょう。

## さらに読む
- [OpenAI API changelog、2026年11月17日](https://developers.openai.com/api/docs/changelog)は、GPT-6.2 AstraとSolの追加を載せています。
- [GPT-6.2 Astraのモデルページ](https://developers.openai.com/api/docs/models/gpt-6.2-astra)と[Solのモデルページ](https://developers.openai.com/api/docs/models/gpt-6.2-sol)には、制限、知識カットオフ、ツールが載っています。
- [GPT-6.2移行ガイド](https://developers.openai.com/api/docs/guides/latest-model?model=gpt-6.2-astra#gpt-6.2-astra-migration-quickstart)は、推論アイテムが落ちる条件と、移行の手順を説明しています。
- [プロンプトキャッシュのガイド](https://developers.openai.com/api/docs/guides/prompt-caching)は、キャッシュ済みの先頭の探し方と、ブレークポイントの条件を説明しています。

出典: 2026年11月17日のOpenAI API changelog、GPT-6.2 AstraとSolのモデルページ、GPT-6.2移行ガイド、プロンプトキャッシュのガイド、Responses APIのリファレンス。すべて2026年11月19日に確認しました。OpenAIのシステムが推論やプロンプトのキャッシュをどう保持するかについての記述は、これらのページによります。詳細は公開後に変わることがあります。
