---
title: "1通送っただけなのに、AIが「too many requests」と返す理由"
emoji: "🚦"
type: "tech"
topics: ["rate-limit", "llm", "api", "systemdesign"]
published: true
---
本記事は System Design Simple の英語記事の日本語版です。原文: https://systemdesignsimple.com/why-ai-says-too-many-requests-after-one-message/

AIのAPIは、1通のメッセージに「429 too many requests」を返すことがあります。トークン予算を超えたか、他のトラフィックと共有している制限が尽きたか、どちらかです。私たちのシミュレーションでは、長いチャットの2通目がトークン切れでブロックされました。そのとき、1分あたり6件のリクエスト枠のうち5.9件がまだ残っていました。


## 1通送っただけなのに、なぜ「too many requests」になるのか
**429 Too Many Requests**は、一定時間に送りすぎたクライアントに返すHTTPステータスで、RFC 6585で定義されています。サーバーは、どれくらい待てばよいかを示すRetry-Afterヘッダーを付けることがあります。

**トークン**は、言語モデルが読んだり書いたりするテキストの小さな断片で、たいていは単語の一部です。入力トークンは送るもの、出力トークンは返信です。

ステータスコードは「リクエスト」と言いますが、AIのAPIはたいてい1つ以上のものを数えています。[Anthropic API](https://platform.claude.com/docs/en/api/rate-limits)と[OpenAI API](https://developers.openai.com/api/docs/guides/rate-limits)の公開されているレート制限のページは、どちらも1分あたりのリクエスト数とトークン数の制限を説明していて、どれか1つを超えるとリクエストが拒否されます。制限はふつう組織やプロジェクトにかかるので、同じキーを使う他のトラフィックが、あなたのメッセージが届く前に制限を使い切ることもあります。どの予算が尽きても同じ429が返ってくるので、エラーの本文と制限のヘッダーを読んでください。

クライアントが履歴を管理するチャットでは、アプリが保持している会話の履歴を、モデルへのリクエストごとに含めます。その履歴は、プロバイダーのキャッシュの扱いにもよりますが、入力としてもう一度数えられます。アプリは古いターンを切り詰めたり要約したりすることもあります。私たちのラボは、45ターン前までの会話をすべて送り返します。エージェントで同じ効果が出ることは[AIエージェントが履歴を送り返す理由](https://systemdesignsimple.com/why-an-ai-agent-re-sends-its-history/)で、チャットが忘れる理由は[チャットが名前を忘れる理由](https://systemdesignsimple.com/my-chat-forgot-my-name/)で示しました。

## トークンバケットはどう判断するか
**トークンバケット**は、決まった数のクレジットを保持し、一定の速さで補充するレート制限の仕組みです。リクエストが通るのは、バケットにその分のクレジットがあるときだけで、通った分は使われます。

Anthropicのページでは、制限に[トークンバケット](https://en.wikipedia.org/wiki/Token_bucket)アルゴリズムを使い、1分の頭でリセットするのではなく容量を連続的に補充すると書かれています。私たちのシミュレーションは、予算ごとに1つのバケットを使い、既知の入力トークンと出力トークンを入場の前に差し引きます。

```text
requests bucket: capacity 6,      refill 6 / 60 s      = 0.1 credit per second
tokens bucket:   capacity 10,000, refill 10,000 / 60 s ≈ 166.7 tokens per second

a request passes only if  requests >= 1  AND  tokens >= input + output
```

実際のプロバイダーは数え方が違います。Claudeは入力と出力の制限を別々に文書化していて、キャッシュを意識した入力の数え方も含みます。OpenAIは、要求した出力の上限を入れた入場の見積もりを文書化しています。その違いが、長い履歴で予算が尽きるかどうかを決めます。キャッシュされた入力が数えられないなら、キャッシュされた接頭辞を持つ長いチャットは、私たちのラボが請求するよりもずっと少なく制限に食い込みます。

短い質問は、リクエストのクレジットを1つと、トークンのクレジットを数百使います。長いチャットのメッセージもリクエストのクレジットは1つですが、トークンのクレジットは数千使います。だから、リクエストのバケットがほぼ満杯でも、トークンのバケットは空になることがあります。

クレジットは `min(capacity, credits + elapsed × refill_rate)` で補充されるので、容量に収まるブロックされたメッセージは、他に何も変わらなくても、後になれば通るようになります。容量を超える請求は、どれだけ待っても通りません。10,000トークンのバケットに対する12,000トークンのリクエストは、待っても失敗します。対処は、リクエストを小さくする(履歴を切り詰めるか、出力の上限を下げる)か、アカウントの上限を上げるかのどちらかです。私たちのラボは、この場合を別に記録し、リトライはしません。

## 私たちのラボで測ったこと
![AI 21のカバー: 6件中5.9件のリクエスト枠が残っているのに1通がブロックされる様子、トークンバケット](https://systemdesignsimple.com/img/shorts/ai-21.jpg)

私たちのラボは、シミュレーション用の時計とシード21を使ったNode 22のシミュレーションです。スクリプト[`labs/ai-21/run.mjs`](https://github.com/systemdesignsimple/labs/blob/main/labs/ai-21/run.mjs)は毎回同じ数値を出します。公開している[ラボのリポジトリ](https://github.com/systemdesignsimple/labs/tree/main/labs/ai-21)にあります。制限は、効果が見えやすいように私たちが決めたもので、どのプロバイダーも使っていません。トークン数は固定の数え方によるもので、プロバイダーのトークナイザーによるものではないので、おおよその値として扱ってください。

| メッセージ | 入力 | 出力 | おおよそのトークン | リクエスト |
|---|---|---|---|---|
| 新しいチャットでの短い質問 | 130 | 344 | 約474 | 1 |
| 45ターン前の会話のあとの同じ質問 | 8,151 | 344 | 約8,495 | 1 |

長いチャットのメッセージは、短い質問の17.9倍のトークンを使いました(8,495 / 474)。8.94シミュレーション秒の間隔で2回送りました。

```text
send #1  t=0.00s  needs 8,495  tokens left before: 10,000  -> 200, tokens left 1,505
         refill over 8.94 s: about +1,490 tokens            -> about 2,995 left
send #2  t=8.94s  needs 8,495  tokens left: 2,995           -> 429 (tokens)
         requests left at that moment: 5.89 of 6
```

メッセージは約8,500トークンを必要とし、バケットには約3,000しかありませんでした。だから、リクエストの予算がほとんど残っているのに拒否されたのです。トークンの不足は5,500で、他のトラフィックがなければ、最も早い受け入れは33秒後です。5,500 / (10,000 / 60)。短い質問なら通りました。474は2,995を大きく下回るからです。古いターンを切り詰めるか要約すれば、この料金は小さくなります。ここでは測っていません。

## エージェントがリクエスト制限に当たる理由
エージェントは、1通のメッセージを何回ものモデル呼び出しに変えることがあります。ラボでは、1通のエージェントメッセージが、途中にツール呼び出しを挟んで10回のモデル呼び出しを計画しました。各呼び出しは増えていくコンテキストを送り返しましたが、長いチャットに比べれば1回1回は小さかったので、今度はリクエストのバケットが先に尽きました。リトライなしでは、7回目の呼び出しがリクエストの429を受けました。7件の送信のうち6件が成功です。どちらの予算も尽きる可能性があるので、クライアントは両方を追わなければなりません。ルーティングも呼び出しを増やします。[どのAIが答えるかを決める仕組み](https://systemdesignsimple.com/who-decides-which-ai-answers-your-message/)のとおりです。[セマンティックキャッシュ](https://systemdesignsimple.com/two-questions-look-alike-can-ai-reuse-the-answer/)は、呼び出しの一部をなくせます。

## 悪化させずにリトライする方法
一時的なスロットリングだけをリトライしてください。OpenAIのページでは、時間とともに解消するレート制限エラーと、口座側の対応が必要なクォータや課金のエラーを分けています。後者をリトライしても、同じ失敗を繰り返すだけです。スロットリングでも、すぐにリトライしても意味がありません。私たちのシミュレーションでは、バケットがまだ補充されていないからです。ふつうの対処は、ジッター付きの指数バックオフです。[AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)の説明のとおり、失敗のたびに上限が倍になる範囲からランダムな時間を待ちます。上限があります。ランダムにすることで、一緒に失敗したクライアントがばらけます。

[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after)では、Retry-Afterを秒数でもHTTP日付でも送れます。クライアントは両方を解析しなければなりません。

```js
// backoff.mjs: full jitter, base 1 s, cap 30 s (the lab's settings), plus a deadline
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

function retryAfterMs(value, now = Date.now()) {
  if (value == null) return null;
  const v = value.trim();
  if (/^\d+$/.test(v)) return Number(v) * 1000;      // delay-seconds
  const at = Date.parse(v);                           // HTTP-date
  return Number.isNaN(at) ? null : Math.max(0, at - now);
}

// call(signal): must abort its request when signal fires (pass it to fetch or the SDK)
// isTemporary(res): true for throttling, false for quota or billing errors (read the error body)
export async function withBackoff(call, isTemporary,
    { base = 1000, cap = 30000, retries = 8, deadlineMs = 60000 } = {}) {
  const deadline = Date.now() + deadlineMs;
  let res = null;
  for (let attempt = 0; ; attempt++) {
    const left = deadline - Date.now();
    if (left <= 0) return res;                          // no time for another attempt
    res = await call(AbortSignal.timeout(left));        // the attempt itself is bounded too
    if (res.status !== 429 || attempt === retries || !(await isTemporary(res))) return res;
    const hinted = retryAfterMs(res.headers.get('retry-after'));
    const wait = hinted !== null ? hinted : Math.random() * Math.min(cap, base * 2 ** attempt);
    if (Date.now() + wait >= deadline) return res;      // the wait alone would pass the deadline
    await sleep(wait);
  }
}
```

期限は、`call`がシグナルを守る場合にだけ有効です。シグナルを無視する呼び出しは、期限を過ぎても動き続けます。残り時間は毎回の試行の前に確認するので、遅れたタイマーが期限後に新しいリクエストを始めることはありません。多くのSDKも429を自分でリトライします。切るか(たとえば`maxRetries: 0`)、その内側のリトライも同じ試行回数と時間の予算に数えるか、どちらかにしてください。そうしないと、外側の1回の試行が実は何回ものリトライを隠すことになります。[OpenAIのレート制限ガイド](https://developers.openai.com/api/docs/guides/rate-limits)も、回数を区切ったバックオフを同じように勧めています。

ラボでは、バックオフで失敗したメッセージが7件中3件から0件になりました。代償は待ち時間です。全部で135シミュレーション秒、最も遅いメッセージで52.9秒でした。

## トレードオフ
| 選択肢 | 良いところ | 代償 | 使う場面 |
| --- | --- | --- | --- |
| 履歴をすべて送り返す | モデルがすべてを見る | ターンごとにトークンの料金が増える | 短いチャット |
| 古いターンを切り詰めるか要約する | 1通あたりのトークン料金が小さい | モデルが細部を失うことがある | トークン制限に近い長いチャット |
| ジッター付きのバックオフ | 一時的なスロットリングによる、ユーザーに見えるエラーを減らす | 数秒の待ち時間 | スロットリングを受ける可能性があるクライアントならどれでも |
| クライアント側のトークン予算 | 予測できるレート制限の拒否を減らす | プロバイダーごとの見積もりと、ワーカー間の調整が必要 | エージェントとバッチジョブ |

## 私たちの見解
どの制限が尽きたのかを、入力と出力のトークン数と一緒に記録してください。送る履歴は管理できる範囲に抑え、一時的なスロットリングだけを、バックオフとジッター付きでリトライしてください。私たちのシミュレーションは、待てばメッセージが回復することを示しています。本番のAPIで使えるリトライの予定表を検証したものではありません。[無料コース](https://systemdesignsimple.com/learn/)では、制限、キュー、リトライを順に説明しています。

注: このラボは、1台のマシンでのシミュレーションで、制限は作ったもの、トークン数もおおよそのものです。効果の形を示すもので、どのプロバイダーの実際の制限やタイミングでもありません。プロバイダーの挙動は、非公開の仕組みではなく、公開されているドキュメントから説明しています。
