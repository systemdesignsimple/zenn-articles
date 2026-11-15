---
title: "Claude Codeのアドバイザー: Opus 5.5が作業し、Fable 5.2が助言する"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "ai", "claude", "agent", "devtools"]
published: true
---
本記事は System Design Simple の英語記事の日本語版です。原文: https://systemdesignsimple.com/claude-code-advisor-opus-fable/

Claude Codeのアドバイザーを使うと、1つのモデルが作業を進めながら、結果を左右する場面だけ、もっと強いモデルに相談できます。Opus 5.5を本体、Fable 5.2をアドバイザーにすれば、計画、繰り返すエラー、最後のチェックにFableの判断が入ります。毎ターンFableを走らせる必要はありません。

## Claude Codeのアドバイザーとは

アドバイザーは、Anthropic側で動く[サーバーツール](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool)です。大事な場面で本体のモデルがこれを呼びます。アドバイザーのモデルは会話全体を受け取ります。ツール呼び出しも、その結果も、すべてです。そして助言を返します。本体のモデルはその助言を適用して、作業を続けます。アドバイザー自身がファイルを編集することはありません。本体のモデルの証拠が助言と食い違うとき(勧められた手順が失敗した、ファイルの内容が違うと言っている、など)は、盲目的に従うのではなく、その対立を表面に出します([Claude Codeのドキュメント](https://code.claude.com/docs/en/advisor))。

## Fable 5.2をアドバイザーにする方法

やり方は3つ、ドキュメントの通りです。

```bash
# 1. セッションの中で設定する。既定として保存される
/advisor fable

# 2. 設定ファイル(~/.claude/settings.json)に書く。ずっと有効な既定になる
{
  "advisorModel": "fable"
}

# 3. 1回だけのセッション。保存した設定は変えない
claude --advisor fable
```

`fable`、`opus`、`sonnet`は、Claude Codeの各ファミリーの現在の既定への別名です。`claude-opus-5-5`のような完全なモデルIDも使えます。何が有効かを確認するには、引数なしで`/advisor`を実行します。私たちのMac(Claude Code 2.1.322)では、設定を追加する前と後で、こう表示されました。

```text
$ claude -p "/advisor"
Advisor: off
Usage: /advisor <fable|opus|sonnet|off>

# ~/.claude/settings.json に "advisorModel": "fable" を追加したあと
$ claude -p "/advisor"
Advisor: Fable 5.2
Usage: /advisor <fable|opus|sonnet|off>
```

プランによっては、Fableをアドバイザーにする前に1回だけの同意が必要です。`/model fable`を実行して、Fableで1回続けてから、Fableをアドバイザーに選びます。それまではClaude Codeが`/model fable`を指し示し、Fableをアドバイザーとして保存しません。

## どのアドバイザーがどのモデルと組めるか

ルールは1つです。アドバイザーは、セッションの本体モデルと同格以上でなければなりません。Claude Codeはリクエストのたびに組み合わせをチェックし、APIももう一度チェックします。

![本体モデルごとに受け付けるアドバイザーモデルを示したClaude Codeドキュメントの表](https://systemdesignsimple.com/img/claude-code-advisor-opus-fable/pairing-table.webp)
*受け付けられる組み合わせ。Claude Codeのドキュメントより。2026年11月10日に取得。[出典](https://code.claude.com/docs/en/advisor)*

| 本体モデル | 使えるアドバイザー | 備考 |
|---|---|---|
| Opus 5 / Opus 5.5 | Fable、Opus 5以降 | Opusが作業し、Fableが助言する |
| Sonnet 5.5 | Fable、Opus 5以降、Sonnet 5.5 | Fableを常時走らせずに助言だけ受ける |
| Fable 5.2 | Fable 5.2のみ | Claude CodeはFableにOpusやSonnetのアドバイザーを付けない |
| Haiku 4.5 | Fable、Opus、Sonnet、Haiku 5.5 | Haiku 4.5はアドバイザーを呼べるが、アドバイザーにはなれない |

サブエージェントはアドバイザーを引き継ぎ、自分のモデルに対して同じチェックを受けます。本体モデルより下のアドバイザーを選ぶと、Claude Codeは設定を保存しますが、付けません。

## Claudeがアドバイザーに聞くタイミング

決めるのはClaudeで、固定のルールではありません。実際には、こういう場面で相談する傾向があります。

- 方針を決める前に
- 同じエラーが繰り返し返ってくるときに
- タスクが終わったと宣言する前に

直接頼むこともできます(「続ける前にアドバイザーに相談して」)。呼び出し回数を制限する設定も、強制する設定もありません。もっと多く、もっと少なくしたければ、指示に書きます。呼び出しの最中は、トランスクリプトに`Advising`という行が出ます。`Ctrl+O`を押すと助言が読めます。セッションの途中でアドバイザーをオン・オフしても、本体モデルのプロンプトキャッシュは壊れません。

## アドバイザーとopusplan、サブエージェントの違い

![アドバイザーツール、opusplan、サブエージェント、モデル切り替えを比べたClaude Codeドキュメントの表](https://systemdesignsimple.com/img/claude-code-advisor-opus-fable/compare-table.webp)
*強いモデルがどこで動くか。アプローチごとの違い。Claude Codeのドキュメントより。2026年11月10日に取得。[出典](https://code.claude.com/docs/en/advisor)*

アドバイザーは、長いタスクに向いています。ほとんどのターンは定型作業でも、計画が結果を決めるようなタスクです。大きなリファクタ、戻ってくるバグ、終わりと呼ぶ前にチェックしてほしい作業がそうです。[Claude Codeの横でCodex CLIを動かしている](https://systemdesignsimple.com/claude-code-vs-codex-cli/)なら、アドバイザーはClaudeのセッションの中でセカンドオピニオンをくれます。短いタスクや、毎ターン最強のモデルが要る作業なら、`/model`で本体モデルを切り替えたほうがいいです。サブタスク全体を別のモデルに渡したいなら、自分のモデルを持つサブエージェントを使います。

## 要件と制限

- 実験的機能で、Anthropic APIかAWS上のClaude Platformでのみ使えます(AWS上ではClaude Code 2.1.300以降が必要)。Amazon Bedrock、Google CloudのAgent Platform、Microsoft Foundryでは使えません([要件](https://code.claude.com/docs/en/advisor))。
- Fable 5.2をアドバイザーにするには、Claude Code 2.1.321以降とFableへのアクセスが必要です。
- アドバイザーは呼び出しのたびに会話全体を読みます。エージェントの[履歴はターンごとに増えていきます](https://systemdesignsimple.com/why-an-ai-agent-re-sends-its-history/)。この読み取りはキャッシュされないので、呼び出し1回ごとに、本体モデルに加えてFableの使用枠を使います。プランによっては、Fableのアドバイザーが使用クレジットに課金されます([週次制限の仕組み](https://systemdesignsimple.com/claude-ultra-no-5-hour-limit/))。
- AWS上のClaude Platformの外では、機能フラグの取得がオフだと(たとえば`DISABLE_TELEMETRY`を設定していると)アドバイザーはオフのままです。
- `/advisor off`でオフにできます。`CLAUDE_CODE_DISABLE_ADVISOR_TOOL=1`でツール自体を無効化できます。

## なぜこの設計がうまくいくのか

これはコーディネーターパターンをひっくり返したものです。強いモデルがたくさんのワーカーを指揮するのではなく、1人の有能なワーカーが、判断の場面だけ強いレビュアーに聞きます。コーディングタスクのほとんどのターンは定型作業です。ファイルを読む、テストを走らせる、小さな修正をする。そういうターンに最強のモデルは要りません。方向を決める数ターンにこそ要ります。結果を変える場所にだけ判断の対価を払う。そういうトレードオフです。[1つのAIが書き、もう1つのAIがレビューする](https://systemdesignsimple.com/one-ai-writes-one-ai-reviews/)のと同じ考え方を、1つのセッションの中に持ち込んだものです。Fable 5.2自体に何ができるかは、[Claude Fable 5.2のレビュー](https://systemdesignsimple.com/claude-fable-5-2-mythos-5-2/)をご覧ください。長いセッションを安定させる残りの設定は、[Claudeをプロのように使う方法](https://systemdesignsimple.com/claude-like-a-pro/)にまとめています。

## 私たちの見解

私たちのMacで確かめた限り、設定はドキュメントの通り動きました。`"advisorModel": "fable"`を追加すると、`/advisor`は`Advisor: Fable 5.2`と表示します。仕組みとして気に入っているのは、強いモデルを常駐させない点です。毎ターンのコストを払わず、判断の場面だけ借りてくる。ただし、呼び出しのたびに会話全体を読むので、長いセッションではその読み取りもFableの枠を使います。使い始める前に、自分のプランでどう課金されるかを確認しておくといいでしょう。
