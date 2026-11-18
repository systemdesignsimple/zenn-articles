---
title: "MCPとAPI、同じタスクで直接5回・MCP経由9回"
emoji: "🔌"
type: "tech"
topics: ["mcp", "api", "ai-agents", "jsonrpc", "systemdesign"]
published: true
---
本記事は System Design Simple の英語記事の日本語版です。原文: https://systemdesignsimple.com/mcp-vs-api-when-ai-needs-mcp/

MCPとAPIの話では、1つのAIアプリとそのAPI連携だけを自分で管理するなら、直接のツール呼び出しのほうがたいていシンプルです。MCPは、対応するホストがそのツールを必要とするときに、共通の発見と呼び出しの仕組みを提供します。2026-11-15に私たちが回したスクリプトによるシミュレーションでは、同じタスクがAPIを直接呼ぶと5回の操作で済み、MCP経由では9回かかりました。


## モデルがツールを使うとき、実際にAPIを呼ぶのは誰か
**MCP (Model Context Protocol)**: JSON-RPC 2.0のプロトコルで、MCPサーバーがツール(名前、説明、入力のJSON Schema付き)を公開し、AIアプリ内のMCPクライアントが`tools/list`で一覧を取り、`tools/call`で実行します。

モデルは呼び出しをしません。モデルが作るのはツール呼び出しのリクエストで、ツール名と引数だけです。たとえば`get_open_orders`に`customer_id: "cust_017"`を付けたものです。ホストアプリがそれを実行します。このデモでは、そのリクエストをJSONテキストとして表します。MCPなしでは、アプリが自分のアダプターコードを動かしてAPIを呼びます。MCPありでは、アプリがリクエストをMCPクライアントに渡し、クライアントが`tools/call`をMCPサーバーへ送り、サーバーがAPIを呼びます。

デモではこの点を確かめました。モックAPIへのリクエストはすべて、誰が送ったかを記録しています。3回の実行すべてで、モデルが送った呼び出しは0件でした。直接の実行ではアプリが送り、MCPの実行ではMCPサーバーが送りました。MCPのツールのページにも同じ流れが書かれています。クライアントがツールを発見し、モデルが1つを選び、クライアントが実行します([MCP Tools, 2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18/server/tools))。

## デモで何を数えたか
スクリプト[`labs/ai-32/compare-mcp-vs-direct-api-steps-and-tokens.mjs`](https://github.com/systemdesignsimple/labs/blob/main/labs/ai-32/compare-mcp-vs-direct-api-steps-and-tokens.mjs)は、「未処理の注文を取って、要約をメールで送る」という1つのタスクを、同じモックのショップAPIに対して3通りの方法で実行します。モデルはスクリプトで決められた動きをするスタブなので、どの実行も同じ選択をします。Node 22の1台のMacで、シード32で動かしました。出力は毎回同じです。

| 経路 | モデル呼び出し | API呼び出し | MCPリクエスト | シミュレーションしたリクエスト/返信の操作数 |
|---|---|---|---|---|
| A. 固定ワークフロー: アプリのコードがAPIを呼び、モデルは要約だけを書く | 1 | 2 | 0 | 3 |
| C. モデルがツールを選び、アプリがAPIを直接呼ぶ | 3 | 2 | 0 | 5 |
| B. モデルがツールを選び、呼び出しはMCP経由 | 3 | 2 | 4 | 9 |

数えていないのは、輸送の確認応答です。CとBが公平な比較で、ツール定義もプロンプトもターンも同じです。スクリプトは、毎ターン、プロンプトがバイト単位で同一であることを確認しています。だからモデルが読むテキストはどちらの経路でも同じで、変わるのは呼び出しの経路だけです。

![ラボのターミナル出力: モデルが送ったAPI呼び出しは0件。往復は固定ワークフローで3回、直接で5回、MCP経由で9回。モデルの文章量は推定で282、2,157、2,157単位。MCPは往復を4回、JSON-RPCを2,628バイト増やします](https://systemdesignsimple.com/img/mcp-vs-api-when-ai-needs-mcp/lab-run.webp)

*ラボの出力の要約です。1台のMacでNode 22を使い、プロセス内でシミュレーションしました。ネットワークも遅延の測定もありません。2026-11-15にChromeでキャプチャしました。ソース: labs/ai-32/capture-output-in-chrome.mjs。*

## MCPが増やす4つのリクエスト
**往復**: 返信を待つ1回のリクエスト。このデモでは、モデル呼び出し、APIへのHTTP呼び出し、MCPクライアントとサーバー間のJSON-RPCリクエストのいずれかです。

| リクエスト | 存在する理由 | 頻度 |
|---|---|---|
| `initialize` | プロトコルのバージョンと能力について合意します。仕様では、最初のやり取りでなければならないとされています | セッションの初期化時に1回 |
| `notifications/initialized` | クライアントの準備ができたことをサーバーに伝えます。通知には返信がないので、数えていません | セッションの初期化時に1回 |
| `tools/list` | ツールとそのスキーマを発見します | このデモでは1回。実際のクライアントはさらにページを取ったり、`notifications/tools/list_changed`のあとに更新したりすることもあります |
| `tools/call` | ツールを1つ実行します | ツールを使うたびに1回(ここでは2回) |

ライフサイクルの規則は[MCPのライフサイクルのページ](https://modelcontextprotocol.io/specification/2025-06-18/basic/lifecycle)から来ています。クライアントが`initialize`を送り、サーバーが能力を返し、クライアントが`notifications/initialized`を送ります。このやり取りのJSON-RPCトラフィックは、シミュレーションで合計2,628バイトでした。

`initialize`と`tools/list`はセットアップのコストです。`initialize`はセッションの初期化時に1回だけ走ります。MCPセッションはHTTPのトランスポート接続と同じものではありません。このデモでは初期化のあと、ページ分割なしの`tools/list`を1回だけ送ります。実際のクライアントは、さらにページを取ったり、一覧を更新したりすることもあります。初期化が済めば、1つのセッションで多くのタスクを処理できます。ただツールを使うたびに`tools/call`が1回かかり、アプリ内で動く直接のアダプターより1ホップ多くなります。

## ツール選択がモデルの文章量を増やした理由
AとCを比べます。どちらもMCPなしで、APIを直接呼びます。モデルにツールを選ばせると、モデル呼び出しが2回増え、推定のモデル文章量は282から2,157単位に上がりました。増えた1,875単位のうち、1,227単位はツール定義で、3回のモデル呼び出しのたびに送り直されました(1回あたり409単位)。残りの648単位は、変わったプロンプトと出力の差し引きで、繰り返されたシステムテキスト、履歴、生成されたツール呼び出しを含みます。

| | 固定ワークフロー(A) | モデルがツールを選ぶ(C) | MCP経由(B) |
|---|---|---|---|
| モデルの文章量、推定単位 | 282 | 2,157 | 2,157 |
| うちツール定義 | 0 | 1,227 | 1,227 |

この単位は、組み込みの簡易な見積もりによるもので、実際のトークナイザーではありません。だから意味があるのは経路どうしの比較だけです。MCP経由に変えても、モデルの文章量は0単位しか変わりませんでした。増えたのは、モデルにツールを選ばせたことによるもので、このスクリプトは両方の定義を毎ターン送り直します。「注文を取って、メールを送る」という決まったタスクでは、固定ワークフローのほうが推定の文章量は少なく済みました。課金は測っていません。エージェントが履歴を送り直す理由は[AIエージェントが履歴を送り直す理由](https://systemdesignsimple.com/why-an-ai-agent-re-sends-its-history/)で扱いました。

## 私たちが実行したとき
2026-11-15 06:23 UTCに、私たちのMacで`node labs/ai-32/capture-output-in-chrome.mjs`を実行しました。スクリプトはChromeで撮った画面を1400x808の画像として保存しました。ラボフォルダにPNGで1枚、WebPで1枚です。

![私たちのai-32実行の出力: node capture-output-in-chrome.mjs](https://systemdesignsimple.com/img/mcp-vs-api-when-ai-needs-mcp/lab-run.webp)

*2026-11-15にChromeでキャプチャスクリプトが保存した画像。*

![公式ページのスクリーンショット: Model Context Protocol specification 2025-06-18](https://systemdesignsimple.com/img/posts/mcp-vs-api-when-ai-needs-mcp/source-1.webp)

*ソースとして挙げている公式のModel Context Protocol仕様ページ。*

このページが、私たちの定義の土台を裏付けています。MCPクライアントとサーバーはJSON-RPCメッセージでやり取りします。

## 最小のシミュレーションMCPクライアント
これはデモのクライアント側で、短くしたものです。シミュレーション用のコードです。`pipe.send`は同じプロセス内のサーバーにJSON-RPCメッセージを1つ渡し、返信を返します。

```js
// mcp-client.mjs (simulation code from labs/ai-32, trimmed)
class McpClient {
  constructor(pipe) { this.pipe = pipe; this.nextId = 1; }
  request(method, params) {
    const reply = this.pipe.send({ jsonrpc: '2.0', id: this.nextId++, method, params });
    if (reply.error) throw new Error(`${method}: ${reply.error.message}`);
    return reply.result;
  }
  connect() {                                   // once per session
    this.request('initialize', { protocolVersion: '2025-06-18', capabilities: {},
      clientInfo: { name: 'lab-host-app', version: '0.1.0' } });
    this.pipe.send({ jsonrpc: '2.0', method: 'notifications/initialized' }); // no id, no reply
  }
  listTools() { return this.request('tools/list', {}).tools; }        // discovery; may paginate or refresh
  callTool(name, args) { return this.request('tools/call', { name, arguments: args }); } // per tool use
}
```

`node compare-mcp-vs-direct-api-steps-and-tokens.mjs`を[`labs/ai-32`](https://github.com/systemdesignsimple/labs/tree/main/labs/ai-32)で実行してください(Node 22、インストール不要、ネットワーク不要。先に`git clone https://github.com/systemdesignsimple/labs`)。トレースが誰と誰が話すかを段階ごとに表示します。「JSON-RPC tools/call get_open_orders」と「calls made by the model: 0」という行を探してください。

本番のクライアントは、JSON-RPCの`error`フィールドだけでは足りません。`initialize`のあと、返ってきた`protocolVersion`が自分の対応バージョンと合うかを比べ、合わなければ切断すべきだと[ライフサイクルのページ](https://modelcontextprotocol.io/specification/2025-06-18/basic/lifecycle)は言います。`tools/list`を呼ぶのは、サーバーが`tools`の能力を宣言している場合だけです。実行して失敗したツールは、`isError: true`付きの通常の結果を返します([ツールのエラー](https://modelcontextprotocol.io/specification/2025-06-18/server/tools#error-handling))。このスニペットは`isError`を調べずに結果を返します。ホストはそのフラグを見て、失敗の状態を保ったままモデルへ結果を渡さなければなりません。すべてのリクエストにはタイムアウトも必要です。`send_email`では、タイムアウトすると結果がわからなくなります。メールがすでに送られているかもしれないからです。だから再試行する前に、API側に冪等キーなどの重複送信防止が必要です。

## MCPとAPI、どちらを使うべきか
直接の経路は、ほかのアプリがそのツールを必要としない場合にシンプルです。MCPの余分なリクエストは、共有インターフェースの対価です。対応するホストは、設定と認可のあと、サーバーのツールインターフェースを再利用でき、アプリごとに新しいアダプターコードを書く必要がありません。私たちの[ツール発見の記事](https://systemdesignsimple.com/how-ai-learns-new-apps-tool-discovery/)では、クライアントが`tools/list`だけで新しいアプリを取り込む様子を示しました。

| 状況 | APIを直接呼ぶ | MCPサーバーを立てる |
|---|---|---|
| 1つのAIアプリ、1つのAPI、どちらも自分で持っている | いちばんシンプルで、ホップも最少 | クライアントとサーバーの境界が増えます。このシミュレーションでは4回のリクエストが増えました |
| 対応する複数のAIアプリが同じツールを必要とする | アプリごとにアダプターが必要 | 1つのサーバーをみんなで再利用できます |
| 自分のAIでないアプリがロジックを再利用する | 共有のAPIクライアントライブラリ | このためには要りません |
| ワークフローがいつも同じ手順 | 固定ワークフローは、このシミュレーションでモデル呼び出しも推定の文章量も少なく済みました | このためには要りません |

ほかのアプリがそのロジックを再利用するだけなら、共有のAPIクライアントライブラリで十分で、MCPは要りません。MCPが加えるのは、自分が書いていない対応AIアプリもそのツールを使えることです。

どちらの経路を選んでも、ツールのページは、人間がツール呼び出しを拒否できる仕組みと、センシティブな操作の前の確認プロンプトを求めています。そのあたりは[AIエージェントは私のファイルを消すのか](https://systemdesignsimple.com/would-an-ai-agent-delete-my-files/)をご覧ください。

## このデモでは見えないこと
このシミュレーションでは、対応するホスト間での再利用は試していません。

## 私たちの見解
自分で管理する1つのAIアプリが1つのAPIと話すなら、APIを直接呼んでください。このデモでは5回の操作で済み、9回ではありませんでした。対応する複数のAIアプリが同じツールを必要とするなら、MCPサーバーを立ててください。クライアントとトランスポートが対応していれば、初期化済みのMCPセッションをタスク間で再利用してください。どちらを選ぶにしても、まずモデルにツールを選ばせる必要があるかを考えてください。その選択のほうが、MCPよりも文章量を増やしました。

この記事は、小さなシミュレーションによる一般的なパターンの説明です。特定の企業の非公開システムを説明するものではなく、AI製品を測定したものでもありません。数字は、スクリプトで決められたモデルによる2026-11-15の1回の実行から来ています。
