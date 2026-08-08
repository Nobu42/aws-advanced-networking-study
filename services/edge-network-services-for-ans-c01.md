# Edge Network Services ANS-C01対策

## 目的

ANS-C01で出題されるエッジネットワーク系サービスを、問題文から選択肢を切り分けられる形で整理する。

対象は主に次のサービス。

```text
Amazon CloudFront
AWS Global Accelerator
Lambda@Edge
CloudFront Functions
AWS WAF
AWS Shield
Route 53
```

## 最初に覚える結論

| 要件 | 疑うサービス |
| :--- | :--- |
| 静的/動的コンテンツを世界中へ低遅延配信したい | CloudFront |
| コンテンツをキャッシュしたい | CloudFront |
| ALB/NLB/EC2の前に固定IPが欲しい | Global Accelerator |
| 世界中からTCP/UDPアプリへ低遅延で接続したい | Global Accelerator |
| 企業Firewallに固定IPを許可リスト登録したい | Global Accelerator |
| HTTPリクエストを軽く書き換えたい | CloudFront Functions |
| 認証、A/Bテスト、動的オリジン選択など複雑な処理をエッジで行いたい | Lambda@Edge |
| Web攻撃をL7で防ぎたい | AWS WAF on CloudFront |
| DDoS対策を強化したい | AWS Shield Advanced |
| DNSで宛先を切り替えたい | Route 53 |

## 全体像

```text
Viewer / Client
  |
  | DNS
  v
Route 53
  |
  +--> CloudFront
  |      +--> CloudFront Functions
  |      +--> Lambda@Edge
  |      +--> AWS WAF
  |      +--> Origin: S3 / ALB / NLB / EC2 / custom origin
  |
  +--> Global Accelerator static anycast IPs
         +--> Listener
         +--> Endpoint group
         +--> Endpoint: ALB / NLB / EC2 / EIP
```

## CloudFront

Amazon CloudFrontは、AWSのCDNである。世界中のエッジロケーションにコンテンツをキャッシュし、ユーザーに近い場所から配信する。

```text
Viewer
  -> CloudFront edge location
  -> cache hitならedgeから応答
  -> cache missならoriginへ取得
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 主な役割 | CDN、キャッシュ、エッジ配信 |
| 対象プロトコル | HTTP/HTTPSが中心 |
| 代表オリジン | S3、ALB、NLB、EC2、Media系、custom origin |
| セキュリティ | AWS WAF、Shield、OAC、signed URL/cookie |
| カスタム処理 | CloudFront Functions、Lambda@Edge |
| 向いている要件 | コンテンツ配信、キャッシュ、Web高速化 |

試験キーワード:

```text
cache
CDN
static content
dynamic content acceleration
edge location
S3 origin
origin failover
signed URL
signed cookie
```

## CloudFrontの基本用語

| 用語 | 意味 |
| :--- | :--- |
| Distribution | CloudFront設定の本体 |
| Origin | CloudFrontが取りに行く元サーバ |
| Cache behavior | パスごとのキャッシュ/転送ルール |
| Cache key | キャッシュを区別するキー |
| Cache policy | キャッシュキーとTTLを制御 |
| Origin request policy | オリジンへ転送するheader/cookie/query stringを制御 |
| Viewer request | ユーザーからCloudFrontへのリクエスト |
| Origin request | CloudFrontからオリジンへのリクエスト |
| Viewer response | CloudFrontからユーザーへのレスポンス |
| Origin response | オリジンからCloudFrontへのレスポンス |

## Cache PolicyとOrigin Request Policy

CloudFrontでは、キャッシュキーに入れる値と、オリジンへ送る値を分けて考える。

```text
Cache policy
  = 何をキャッシュキーに含めるか

Origin request policy
  = 何をオリジンへ転送するか
```

例:

```text
Accept-Languageをcache keyに含める
  -> 言語別に別キャッシュになる

Authorization headerをoriginへ送るがcache keyに含めない
  -> オリジン認証には使えるが、キャッシュ設計に注意
```

重要:

```text
cache keyに含める値が増えるほど、cache hit率は下がりやすい
originへ送る必要がある値でも、必ずcache keyに入れるとは限らない
```

## CloudFront Origin Access Control

OAC（Origin Access Control）は、CloudFrontからS3オリジンへ安全にアクセスするための仕組みである。

```text
Viewer
  -> CloudFront
  -> OACで署名されたリクエスト
  -> S3 bucket
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 推奨 | OAIよりOACが推奨 |
| 対象 | S3 origin |
| 目的 | S3 URLへの直接アクセスを防ぎ、CloudFront経由に限定 |
| 対応 | SSE-KMS、動的リクエスト、最近のリージョンに強い |
| 注意 | S3 website endpointはcustom origin扱いでOAC対象外 |

試験キーワード:

```text
S3 bucket must not be publicly accessible
users must access content only through CloudFront
origin access control
OAI legacy
```

## CloudFront Origin Failover

CloudFront Origin Failoverは、プライマリオリジンが失敗した場合にセカンダリオリジンへ切り替える機能である。

```text
CloudFront
  -> Primary origin
  -> failover condition
  -> Secondary origin
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 構成 | Origin groupにPrimary/Secondaryを設定 |
| 条件 | 指定HTTPステータス、接続失敗、タイムアウトなど |
| 対象メソッド | 主にGET/HEAD/OPTIONS |
| 用途 | S3/ALB/custom originの可用性向上 |

## CloudFront Functions

CloudFront Functionsは、CloudFrontのエッジで軽量なJavaScriptを実行する機能である。

```text
Viewer
  -> CloudFront Function
  -> CloudFront
```

向いている処理:

| 用途 | 例 |
| :--- | :--- |
| 軽いURL書き換え | `/old`を`/new`へ |
| リダイレクト | HTTP 301/302 |
| Header操作 | セキュリティヘッダー追加 |
| 簡易認可 | Basic認証、軽いトークン確認 |
| A/B振り分け | cookie/headerベースの簡易分岐 |

制約:

| 項目 | 内容 |
| :--- | :--- |
| ランタイム | JavaScript |
| イベント | 主にviewer request / viewer response |
| Request body | 読めない |
| ネットワークアクセス | できない |
| ファイルシステム | 使えない |
| 環境変数 | 使えない |
| 用途 | 高速・軽量・低レイテンシー処理 |

補足:

```text
現在のCloudFront FunctionsにはmTLS向けのconnection requestもある。
ANS-C01ではまずviewer request/viewer responseの軽量処理として覚える。
```

試験での見方:

```text
軽い処理 = CloudFront Functions
複雑な処理 = Lambda@Edge
```

## Lambda@Edge

Lambda@Edgeは、CloudFrontイベントに応じてLambda関数をエッジで実行する仕組みである。

```text
Viewer
  -> CloudFront
  -> Lambda@Edge
  -> Origin
```

CloudFront Functionsより重い処理に向く。

代表用途:

| 用途 | 例 |
| :--- | :--- |
| 認証/認可 | JWT検証、外部認証呼び出し |
| A/Bテスト | cookie/headerでオリジンやパスを変更 |
| 動的オリジン選択 | headerや地域でoriginを差し替える |
| SEO/デバイス最適化 | User-Agentに応じて返す内容を変更 |
| エラーレスポンス加工 | origin responseでエラーを整形 |
| HTTPレスポンス生成 | viewer/origin requestで直接応答 |

## Lambda@Edgeの4イベント

| イベント | 実行タイミング | 試験での使い分け |
| :--- | :--- | :--- |
| Viewer request | CloudFrontがリクエストを受け取った直後、キャッシュ確認前 | cache key変更、認証、URL書き換え |
| Origin request | cache miss後、originへ送る前 | origin選択、origin向けheader追加 |
| Origin response | originから応答後、キャッシュ前 | origin応答を書き換えてキャッシュしたい |
| Viewer response | viewerへ返す直前 | response header追加など |

重要:

```text
viewer request/response = 基本的に毎回実行されやすい
origin request/response = cache miss時だけ実行される
```

ただし、viewer responseはオリジンがHTTP 400以上を返した場合などに実行されないケースがある。

## CloudFront FunctionsとLambda@Edgeの比較

| 観点 | CloudFront Functions | Lambda@Edge |
| :--- | :--- | :--- |
| 主目的 | 軽量・高速なリクエスト/レスポンス加工 | 複雑なエッジ処理 |
| ランタイム | JavaScript | Node.js / Python |
| イベント | viewer request / viewer response | viewer request / origin request / origin response / viewer response |
| Request body | 読めない | 一部読めるがサイズ制限あり |
| ネットワーク呼び出し | 不可 | 可能 |
| Origin操作 | 不向き | 動的origin選択に使える |
| レイテンシー | より低い | Functionsより重い |
| 典型用途 | リダイレクト、header操作、軽い認証 | 認証、A/B、origin選択、外部API連携 |

覚え方:

```text
CloudFront Functions = 軽い・速い・viewer側
Lambda@Edge          = 重い・柔軟・origin側も可能
```

## Lambda@Edgeの注意点

| 注意点 | 内容 |
| :--- | :--- |
| 作成リージョン | Lambda関数はUS East (N. Virginia)、つまりus-east-1に作成する |
| 関数バージョン | `$LATEST`やaliasではなく番号付きversionを使う |
| レプリケーション | CloudFrontに関連付けると世界中へ複製される |
| DNS | origin request関数の前にCloudFrontがorigin名をDNS解決する |
| viewer response | HTTP status codeを変更できない制約がある |
| body | viewer requestは40 KB、origin requestは1 MBなど制限がある |
| WAF順序 | AWS WAF適用後にviewer requestのLambda@Edgeが実行される |

## Global Accelerator

AWS Global Acceleratorは、固定Anycast IPを入口にして、AWSグローバルネットワーク経由で最適なリージョン/エンドポイントへ転送するサービスである。

```text
Client
  -> static anycast IP
  -> nearest AWS edge
  -> AWS global network
  -> ALB / NLB / EC2 / EIP
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 主目的 | 固定IP、低遅延、可用性向上 |
| キャッシュ | しない |
| 対象 | TCP/UDP |
| IPv4 | 固定Anycast IPv4を2つ |
| Dual-stack | IPv4 2つ + IPv6 2つ |
| Standard endpoint | ALB、NLB、EC2、Elastic IP |
| Custom routing endpoint | VPC subnet |
| ルーティング | client location、health、weight、traffic dial |

試験キーワード:

```text
static IP
anycast
allow list
global users
lowest latency
TCP/UDP
ALB endpoint
NLB endpoint
regional failover
```

## Global Acceleratorの構成要素

| 用語 | 意味 |
| :--- | :--- |
| Accelerator | GAの本体 |
| Static IP address | クライアントが接続する固定Anycast IP |
| Listener | TCP/UDPとポートを待ち受ける設定 |
| Endpoint group | リージョン単位のエンドポイント集合 |
| Endpoint | ALB/NLB/EC2/EIPなど実際の宛先 |
| Traffic dial | リージョンへの流量割合 |
| Endpoint weight | 同一endpoint group内の重み |
| Client affinity | 同じクライアントIPを同じendpointへ寄せる |

## Global AcceleratorのClient IP Preservation

Global Acceleratorは、endpoint種類によってクライアントIPの扱いが違う。

| Endpoint | Client IP preservation |
| :--- | :--- |
| ALB | 対応。ALBでは`X-Forwarded-For`で確認 |
| EC2 | 対応 |
| NLB with security groups | 対応 |
| NLB without security groups | 非対応 |
| Elastic IP | 非対応 |
| Internal ALB | 常に有効 |

重要:

```text
Internal ALBをGlobal Acceleratorのendpointにすると、
ALBはprivateでもGlobal Accelerator経由で到達できる。
ただしSGでは元クライアントIPを許可する必要がある。
```

試験での引っかけ:

```text
private ALB + GA + SG 0.0.0.0/0 on 443
```

これは「ALBをインターネット公開している」とは限らない。ALB自体にInternet Gateway経路がなければ、直接インターネットからは到達できない。GA経由で届く元クライアントIPをSGで許可するために必要な場合がある。

## CloudFrontとGlobal Acceleratorの違い

| 観点 | CloudFront | Global Accelerator |
| :--- | :--- | :--- |
| 主目的 | コンテンツ配信/CDN | 固定IP/経路最適化 |
| キャッシュ | する | しない |
| プロトコル | HTTP/HTTPS中心 | TCP/UDP |
| 入口 | DNS名 | 固定Anycast IP |
| 宛先 | S3/ALB/HTTP originなど | ALB/NLB/EC2/EIP |
| L7処理 | 強い。cache behavior、headers、WAF、edge functions | 弱い。L4寄り |
| 静的IP | 標準では固定IP目的ではない | 固定IPが主目的 |
| 代表要件 | Web高速化、キャッシュ、S3配信 | 企業FW許可リスト、ゲーム/VoIP/TCP/UDP、multi-region failover |

覚え方:

```text
CloudFront = キャッシュするHTTPエッジ
Global Accelerator = 固定IPのグローバル入口
```

## Route 53との違い

| サービス | 役割 |
| :--- | :--- |
| Route 53 | DNSで名前から宛先を返す |
| CloudFront | エッジでコンテンツを配信/キャッシュする |
| Global Accelerator | 固定Anycast IPでAWS global networkへ入れる |

試験での切り分け:

```text
DNS名で宛先を選ぶ = Route 53
コンテンツをキャッシュ = CloudFront
IP固定が必要 = Global Accelerator
```

## AWS WAF on CloudFront

AWS WAFは、HTTP/HTTPSのWebリクエストをL7で検査する。

CloudFrontにWAFを関連付けると、エッジでWeb攻撃をブロックできる。

代表的な検査:

```text
SQL injection
XSS
IP set
country match
rate-based rule
managed rule group
header/cookie/query/body
```

試験キーワード:

| 要件 | 答え |
| :--- | :--- |
| SQL injectionを防ぎたい | AWS WAF |
| XSSを防ぎたい | AWS WAF |
| 特定国をブロックしたい | AWS WAFまたはCloudFront geo restriction |
| HTTP floodを緩和したい | WAF rate-based rule + Shield |
| 複数アカウントへWAFルールを配布 | Firewall Manager |

## CloudFront Geo RestrictionとWAF country match

どちらも国単位制御に使える。

| 機能 | 向いている用途 |
| :--- | :--- |
| CloudFront geographic restriction | 単純な国別allow/block |
| AWS WAF country match | 他条件と組み合わせた柔軟な制御 |

## AWS Shield

AWS ShieldはDDoS対策サービスである。

| 種類 | 内容 |
| :--- | :--- |
| Shield Standard | 追加料金なしで基本DDoS対策 |
| Shield Advanced | 高度なDDoS対策、保護強化、DRT連携など |

ANS-C01では、CloudFront、Route 53、Global Accelerator、ALBなどと組み合わせてDDoS対策として出る。

試験キーワード:

```text
DDoS
AWS Shield Advanced
CloudFront
Route 53
Global Accelerator
AWS WAF
```

## セキュアなS3配信

S3コンテンツをCloudFront経由だけにしたい場合:

```text
Viewer
  -> CloudFront
  -> OAC
  -> S3 private bucket
```

使うもの:

```text
CloudFront
OAC
S3 bucket policy
Block Public Access
```

使わない/注意:

```text
S3 bucketをpublicにしない
S3 website endpointの場合はOAC対象外
OAIはlegacy。基本はOACを優先
```

## プライベートコンテンツ配信

CloudFrontで認可済みユーザーだけに配信したい場合、signed URLまたはsigned cookieを使う。

| 機能 | 使いどころ |
| :--- | :--- |
| Signed URL | 個別ファイルへのアクセス制御 |
| Signed Cookie | 複数ファイルへのアクセス制御 |

覚え方:

```text
1ファイル = signed URL
複数ファイル/URLを変えたくない = signed cookie
```

## Field-Level Encryption

CloudFront Field-Level Encryptionは、POSTリクエスト内の特定フィールドをエッジで暗号化する機能である。

```text
Viewer
  -> CloudFront encrypts sensitive fields
  -> Origin
  -> 必要なアプリだけprivate keyで復号
```

試験キーワード:

```text
sensitive form field
credit card
encrypt at edge
only specific applications can decrypt
```

## エッジサービス選択表

| 問題文 | 選ぶもの |
| :--- | :--- |
| S3の静的Webコンテンツを世界配信 | CloudFront |
| ALB配下のWebをキャッシュして高速化 | CloudFront |
| Web APIでキャッシュせず固定IPが必要 | Global Accelerator |
| UDP/TCPゲームサーバを世界中から低遅延化 | Global Accelerator |
| 顧客Firewallに固定宛先IPを登録したい | Global Accelerator |
| S3 bucketをprivateのままCloudFront配信 | CloudFront OAC |
| CloudFront経由のみでprivate content配信 | OAC + signed URL/cookie |
| 軽いURL rewriteやredirect | CloudFront Functions |
| 外部認証APIを呼ぶ認証処理 | Lambda@Edge |
| ユーザー属性でoriginを動的に変えたい | Lambda@Edge origin request |
| SQLi/XSSを防ぎたい | AWS WAF |
| DDoS対策を強化したい | Shield Advanced |
| DNSでActive/Passive failover | Route 53 failover routing |

## よくある誤解

| 誤解 | 正しい理解 |
| :--- | :--- |
| CloudFrontとGlobal Acceleratorは同じ | CloudFrontはCDN、GAは固定IP/経路最適化 |
| GAはキャッシュする | GAはキャッシュしない |
| CloudFront Functionsで何でもできる | Body/ネットワーク/ファイルシステム不可。軽量処理向き |
| Lambda@Edgeは通常Lambdaと同じ制約 | `$LATEST`不可、version必須などEdge固有制約がある |
| OAIが最新推奨 | 現在はOAC推奨 |
| WAFはL3/L4 Firewall | WAFはHTTP/HTTPSのL7制御 |
| DNS国別制御は必ずWAF | 単純な国別制御ならCloudFront geo restrictionも候補 |
| Route 53だけで低遅延経路になる | DNS応答制御であり、実データ経路最適化はGA/CloudFrontの領域 |

## 試験直前チェック

- [ ] CloudFrontはキャッシュするCDN、Global Acceleratorはキャッシュしない固定IP入口
- [ ] Global AcceleratorはIPv4で固定Anycast IPを2つ提供する
- [ ] Dual-stackのGlobal AcceleratorはIPv4 2つ + IPv6 2つ
- [ ] CloudFront Functionsは軽量JavaScriptでviewer request/response向き
- [ ] Lambda@Edgeは4イベントで使える
- [ ] Lambda@Edgeは番号付きversionが必要で、`$LATEST`やaliasは使わない
- [ ] OACはS3 originをCloudFront経由に限定する推奨方式
- [ ] Signed URLは個別ファイル、Signed Cookieは複数ファイル
- [ ] Cache policyはcache key、Origin request policyはoriginへ送る値
- [ ] CloudFront origin failoverは主にGET/HEAD/OPTIONSで使う
- [ ] AWS WAFはSQLi/XSS/rate-based ruleなどL7防御
- [ ] Shield AdvancedはDDoS対策強化

## 公式参照

- [Customize at the edge with functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions.html)
- [CloudFront events that can trigger Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-cloudfront-trigger-events.html)
- [Get started with Lambda@Edge functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-edge-how-it-works.html)
- [Restrictions on Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-edge-function-restrictions.html)
- [Restrictions on all edge functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-function-restrictions-all.html)
- [Restrictions on CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-function-restrictions.html)
- [Understand cache policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cache-key-understand-cache-policy.html)
- [Control origin requests with a policy](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-origin-requests.html)
- [Restrict access to an Amazon S3 origin](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [Optimize high availability with CloudFront origin failover](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html)
- [What is AWS Global Accelerator?](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html)
- [AWS Global Accelerator components](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-components.html)
- [Preserve client IP addresses in Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/preserve-client-ip-address.html)
