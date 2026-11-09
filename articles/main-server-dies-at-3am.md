---
title: "サーバーフェイルオーバー: アクティブ・パッシブ vs アクティブ・アクティブ"
emoji: "🖥️"
type: "tech"
topics: ["failover", "reliability", "systemdesign", "nginx", "sre"]
published: true
---
本記事は System Design Simple の英語記事の日本語版です。原文: https://systemdesignsimple.com/main-server-dies-at-3am/

アクティブ・パッシブとアクティブ・アクティブのフェイルオーバーでは、ロードバランサーが死んだサーバーへの振り分けをやめるまでの間、そこへ送られたリクエストはすべて失敗します。つまり損失は、どれだけ早く障害に気づけるかと、誰が引き継げる状態にあるかで決まります。秒間 50 リクエストの私たちのシミュレーションでは、1 秒間隔のプローブを 3 回待ったアクティブ・パッシブ構成は中央値で 111 リクエストを失い、最初のリクエスト失敗でサーバーを外したアクティブ・アクティブ構成は 1 リクエストを失いました。

## ラボで測ったこと
コピーを[横に並べて](https://systemdesignsimple.com/two-servers-in-a-row/)置いても、切り替えが速くなければ意味がありません。空白の 1 秒 1 秒が[ダウンタイムの予算](https://systemdesignsimple.com/three-nines-8-hours-down/)から差し引かれます。`labs/sd-11/run.mjs` は、その空白を離散イベントシミュレーションで測ります。Node.js の 1 プロセス(v22.13.1、Apple M2 Pro)の中で仮想クロックを使い、実際のポートは開きません。これは効果の形を示すもので、ベンチマークではありません。コードと各実行の出力は GitHub にあるので、自分で動かせます: [failover-lab](https://github.com/kycsoftnepal/failover-lab)。

```text
load        50 req/s for 30 s; primary killed at 10 s
servers     2, each with 2 workers and a queue; service 8 ms + exp(mean 22 ms); 1 ms per hop
failure     requests in flight fail as "reset"; later ones are "refused" at once (closed port)
A-P         all traffic to the primary; probe every 1,000 ms; switch after 3 misses;
            cold standby (assumed +30 ms on its first request, fading over 50)
A-A         round robin over two warm servers; drop a server on its first failed request; no retries
runs        5 per mode, seeds 1100 to 1104; median (min to max)
```

| 指標 | アクティブ・パッシブ | アクティブ・アクティブ |
|---|---|---|
| 失敗リクエスト | 111 (102 to 148) | どの実行でも 1 |
| 最初の失敗から最初の成功まで | 2,217.2 ms (2,101.1 to 2,994.8) | 35.9 ms (0.9 to 80.3) |
| 停止前の p95 | 92.2 ms | 79.1 ms |
| 停止後 最初の 1 秒の p95 | 259.2 ms (コールドスタンバイ) | 78.3 ms |
| 安定後の p95 | 90.3 ms | 87.8 ms (生き残った側) |

![失敗数 111 の集計と、最初の失敗から最初の成功まで 2.2 s と書かれた暗いタイムラインの帯。私たちのシミュレーション、1 台のマシン、5 回の実行の中央値](https://systemdesignsimple.com/img/main-server-dies-at-3am/b04.webp)
*私たちのシミュレーションでのアクティブ・パッシブ: 失敗リクエスト 111 件、成功が 1 件もない時間が 2.2 秒(5 回の実行の中央値)。*

両構成は検知ポリシーが違うため、111 対 1 という数字はトポロジーと同じくらい検知の差を測っています。検知ポリシーは、死んだサーバーへトラフィックが流れ続ける時間を決めます。トポロジーは、どのサーバーがすでに応答しているか、どれくらい温まっているか、どれだけ余裕があるかを決めます。

## アクティブ・パッシブの検知にはどれくらいかかるか?
プローブが I 秒ごとの固定スケジュールで始まり、プローブのタイムアウト T は I より短く、障害はランダムなタイミングで起き、死んだサーバーは接続を即座に拒否するとします。しきい値が k 回の失敗なら、最初に失敗するプローブは次の I 秒のどこかで始まり、k 回目はその (k - 1) × I 後に始まります:

```text
best      (k - 1) × I      first probe fires right after the crash
worst     k × I            first probe fires just before the crash
expected  (k - 0.5) × I

lab, I = 1 s, k = 3:   2 s to 3 s, expected 2.5 s
failed ≈ rate × time:  50 × 2 = 100  to  50 × 3 = 150
median run:            50 × 2.2172 s ≈ 110.9    measured 111
```

5 回すべての間隔はこの範囲に収まり、そこにスタンバイの最初の応答までの数ミリ秒が加わりました。失敗のほぼすべては検知中の「refused」であり、障害の瞬間に処理中だったものは 1 回の実行あたり 0 から 4 件だけでした。

ハングしたサーバーでは、プローブのタイムアウト T が加わります。固定スケジュールなら T が 1 回分、各プローブが前のプローブの終了から I 後に始まる方式なら k × T です。I = 1 s、k = 3、T = 0.5 s なら、前者は 2.5 から 3.5 秒、後者は 3.5 から 4.5 秒になります。

デフォルトはもっと遅いです。HAProxy の [`inter` 2 s と `fall` 3](https://docs.haproxy.org/3.2/configuration.html#5.2-inter) では 4 から 6 秒かかります。[Kubernetes の readiness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) をデフォルト(10 秒、しきい値 3)のまま使うと、エンドポイントの更新がノードへ広がり始めるまでに 20 から 30 秒かかります。

プロセスが生きていることしか確認しないプローブは、本物の障害を見逃します。逆にデータベースへクエリするプローブは、データベースが落ちたときに全サーバーで同時に失敗します。プローブの結果は、実際のリクエストの失敗と比較してください。全サーバーが同時に失敗したら、まず共有の依存先を確認し、そのうえで全バックエンドを外すことがサービス改善になるのかを判断してください。

コールドスタンバイには独自のコストもありました。想定した +30 ms のもとでは、ワーカー 2 つで処理できるのは約 33 req/s のところに 50 req/s が到着したため、キューが積み上がり、最初の 1 秒間の p95 は 259.2 ms に達しました。

## なぜアクティブ・アクティブは 1 リクエストしか失わなかったのか?
ラウンドロビンにより、各サーバーには約 40 ms ごとに 1 リクエストが届きました。死んだプライマリへの最初のリクエストは拒否され、エラーは 1 ms のホップ 2 回分で戻ってきて、ロードバランサー([バックエンドの選び方](https://systemdesignsimple.com/load-balancer-vs-reverse-proxy/))はそのサーバーの次の順番が来るずっと前にそれを外しました。これはしきい値 1 のパッシブヘルスチェックです。実際のトラフィックがプローブの役割を果たし、接続拒否によって障害が即座に明らかになりました。

![アクティブ・アクティブのレーン: サーバー A は死んだものとして印が付いていて集計に赤いバツが 1 つ、サーバー B は白いチェック付きのリクエストの点を受け続けている](https://systemdesignsimple.com/img/main-server-dies-at-3am/b05.webp)
*アクティブ・アクティブ: ロードバランサーは最初のリクエスト失敗で死んだサーバーを外したため、失敗したのはその 1 リクエストだけでした。*

式に当てはめると、1 秒間隔のプローブを 3 回使うアクティブ・アクティブなら約 50 から 75 件(トラフィックの半分が 2 から 3 秒分)失い、逆に最初のエラーで外すルールを使うアクティブ・パッシブなら約 1 件で済みます。ただし、どちらも私たちは実行していません。Nginx の [`proxy_next_upstream`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream) によるリトライなら、その最後の 1 件の失敗も隠せたかもしれません。接続が拒否された場合は何も送信されていないため、POST でも次へ進めます。一方、POST が送信済みの場合、Nginx はデフォルトでは再送信しません。

## なぜ生き残ったサーバーは遅くなるのか?
私たちのアクティブ・アクティブの実行では、生き残ったサーバーが 2 倍の負荷を受け、その安定後の p95 は上がりました。使用率 U は「到着率 × 平均サービス時間 ÷ ワーカー数」です:

```text
S = 30 ms mean, c = 2 workers
before: 25 req/s each    U = 25 × 0.030 ÷ 2 = 0.375    p95 79.1 ms
after:  50 req/s on one  U = 50 × 0.030 ÷ 2 = 0.75     p95 87.8 ms

N servers at u each, one lost: survivors run at u × N ÷ (N - 1)
to keep survivors at or below 0.75:  N = 2: 0.375   N = 3: 0.50   N = 4: 0.5625
```

このラボの到着は等間隔なので、上昇は 8.7 ms にとどまりました。到着がランダムな場合、M/M/2 モデルでは平均待ち時間が U = 0.375 のときの 4.9 ms から U = 0.75 のときの 38.6 ms に増えます([満杯に近づくほど待ち時間が伸びる理由](https://systemdesignsimple.com/fast-alone-slow-crowded/))。余裕がなければ、生き残ったサーバー自身がチェックに失敗して次々に外れていきます。[カスケード障害に関する SRE 本](https://sre.google/sre-book/addressing-cascading-failures/)が説明している通りの展開です。

## Nginx の中身: 共有される障害状態、リトライ、ログ
私たちのアクティブ・アクティブのレーンは、`max_fails=1` でリトライなしのオープンソース版 Nginx アップストリームに似ていますが、違いが 2 つあります。シミュレーションでは死んだサーバーを永久に外しましたが、Nginx では [`fail_timeout`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) が過ぎるとそのピアは再び候補に戻ります。そのため、障害が続いていれば失敗する試行がさらに起きえます。もう 1 つは、[`zone`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#zone) ディレクティブがない場合、各ワーカープロセスが独自の失敗カウンターと「いつまで使えないか」の印を持つため、トラフィックを受けたワーカーがそれぞれ独立に障害を発見する可能性があることです。`zone app 64k;` と書けば、その状態は共有メモリに移ります。すでに死んだサーバーへ割り当てられたリクエストはそれでも失敗します。Nginx が失敗を記録するのは、失敗したピアを解放するときだからです([`ngx_http_upstream_free_round_robin_peer`](https://github.com/nginx/nginx/blob/master/src/http/ngx_http_upstream_round_robin.c))。

接続エラー、タイムアウト、不正なヘッダーは常に `max_fails` にカウントされます。500、502、503、504、あるいは 429 は、`proxy_next_upstream` にその `http_*` の値が書かれている場合にだけカウントされます。何にでも 500 を返すサーバーは、デフォルトではローテーションに残り続けます。

```nginx
# nginx.conf (http context): active-active, passive checks, bounded retry
upstream app {
    zone app 64k;   # share failure counters and down marks across workers
    server 10.0.0.1:8080 max_fails=1 fail_timeout=10s;
    server 10.0.0.2:8080 max_fails=1 fail_timeout=10s;
}
log_format upstream '$remote_addr "$request" status=$status rt=$request_time '
                    'ua="$upstream_addr" us="$upstream_status" '
                    'uct="$upstream_connect_time" urt="$upstream_response_time"';
server {
    listen 80;
    access_log /var/log/nginx/upstream.log upstream;
    location / {
        proxy_pass http://app;
        proxy_next_upstream error timeout;   # which failures are retried
        proxy_next_upstream_tries 2;         # at most 2 attempts (default 0 = no limit)
        proxy_next_upstream_timeout 3s;      # no new attempt after 3 s
        proxy_connect_timeout 1s;
        proxy_send_timeout 5s;               # inactivity between writes (default 60 s)
        proxy_read_timeout 5s;               # inactivity between reads (default 60 s)
    }
}
```

送信と読み取りのタイムアウトは、連続する 2 回の操作の間の無活動時間を測るため、応答が進み続けている限り無期限に続くことがあります。最初の接続が 1 秒でタイムアウトし、2 台目のバックエンドが応答バイトを何も送らなければ、読み取りタイムアウトで約 5 秒が加わります。これらの設定は応答全体の期限にはなりません([proxy_read_timeout](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout))。全体の期限が必要なら、クライアントか手前のレイヤーで強制してください。

リクエストがリトライされると、各 `$upstream_*` 変数には、試したサーバーごとにカンマ区切りのエントリが 1 つずつ入ります。以下は説明用の行で、私たちのラボで取得したものではありません:

```text
refused, retried:    status=200 ua="10.0.0.1:8080, 10.0.0.2:8080" us="502, 200" urt="0.000, 0.031"
connect timeout:     status=200 ua="10.0.0.1:8080, 10.0.0.2:8080" us="504, 200" urt="1.001, 0.029"
no response bytes:   status=504 ua="10.0.0.1:8080" us="504" urt="5.002"   (POST, not retried)
all marked down:     status=502 ua="app" us="502"   error log: no live upstreams
```

接続の拒否は明示的な拒絶で、多くの場合は何も待ち受けているプロセスがないためです。接続タイムアウトは、期限までに接続が完了しなかったことを意味します。バックエンドとネットワークを調べてください。`ua` にカンマがある場合は、アップストリームへの試行が複数回あったことを示します。リトライが成功したかどうかは、クライアントへの最終ステータスで確認してください([upstream 変数](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#variables))。

## データベース、仮想 IP、DNS
私たちのラボはデータを保持せず、アドレスも移動させません。非同期レプリケーションで送られるデータベースのスタンバイは、障害の直前に確認された書き込みを取りこぼすことがあり([レプリケーション遅延](https://systemdesignsimple.com/one-database-writes-ten-read/))、リースを過ぎても止まっていたノードが書き込みを完了させてしまうこともあります。そのため、ストレージ側で古いフェンシングトークンを拒否するか、昇格の前に旧プライマリを隔離しなければなりません([Kleppmann](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html))。VRRP の仮想 IP には独自のタイマーがあり、優先度 100 で間隔 1 秒の場合 3.609375 秒です([RFC 9568、6.1 節](https://www.rfc-editor.org/rfc/rfc9568.html#section-6.1))。DNS のフェイルオーバーも、検知、レコードの更新、残っているキャッシュの有効期間、クライアントがいつ再接続するかに依存します。

## アクティブ・パッシブとアクティブ・アクティブ、どちらを使うか
| 選択肢 | 向いている用途 | 払うコスト |
|---|---|---|
| アクティブ・パッシブ、アクティブチェック | データベースのプライマリなど、書き込みが 1 つだけのもの | 数秒間のエラー(私たちのシミュレーションでは 111 件)、遊ぶハードウェア、コールドスタート |
| アクティブ・アクティブ、パッシブチェック | ステートレスなアプリサーバー(私たちのシミュレーションでは失敗 1 件) | 全サーバーに N+1 の余裕が必要。生き残りのレイテンシが上がる |
| どちらか + リトライ | 最後の失敗リクエストを隠す | 冪等性キーがなければ副作用が重複する。リトライの嵐 |

## 私たちの見解
ステートレスなアプリサーバーであれば、私たちはアクティブ・アクティブで運用し、実際のリクエストエラーでバックエンドを外し、Nginx に `zone` を入れ、生き残った側を 75% 以下に保つ必要があるなら 2 台それぞれを使用率 37.5% 以下に抑えます。書き込みできるノードが 1 つだけの場所では、フェンシングとセットでアクティブ・パッシブを維持します。どちらの場合でも、実際の間隔、しきい値、タイムアウトから検知にかかる時間を計算し、ときどき意図的にフェイルオーバーさせてください。一度も応答したことのない予備は、コールドスタートや古い設定を隠していることがあるからです。

この記事は一般的なパターンと私たち自身の小さなシミュレーションを説明したものです。特定の企業の非公開システムを説明するものではありません。シミュレーションの数字はトレードオフの形を示すもので、本番環境の計測値ではありません。
