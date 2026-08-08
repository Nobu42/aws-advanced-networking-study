# Security Services ANS-C01対策

## 目的

ANS-C01で出題されるネットワークセキュリティ系サービスを、問題文から正しいサービスを選べる形で整理する。

対象は主に次のサービスと概念。

```text
AWS Shield
AWS WAF
AWS Firewall Manager
AWS Network Firewall
IDS / IPS
VPC Traffic Mirroring
Route 53 Resolver DNS Firewall
AWS Certificate Manager
Gateway Load Balancer
Security Group / Network ACL
```

## 最初に覚える結論

| 要件 | 疑うサービス |
| :--- | :--- |
| DDoS対策を強化したい | AWS Shield Advanced |
| 追加料金なしの基本DDoS保護 | AWS Shield Standard |
| SQL injection / XSS / HTTP floodを防ぎたい | AWS WAF |
| HTTP/HTTPSリクエストをheader、cookie、query stringで制御したい | AWS WAF |
| 複数アカウントへWAF/Shield/SG/Network Firewall/DNS Firewallを展開したい | AWS Firewall Manager |
| VPC内/外向き通信をL3-L7でステートフル検査したい | AWS Network Firewall |
| Suricataルール、DPI、IPSが出てきた | AWS Network Firewall |
| IDSとしてミラーした通信を監視したい | VPC Traffic Mirroring + IDS appliance |
| インラインで通信を止めたいIPS | AWS Network Firewall または GWLB + security appliance |
| 悪性ドメインへのDNS問い合わせを止めたい | Route 53 Resolver DNS Firewall |
| DNS exfiltration / DNS tunnelingが出てきた | Route 53 Resolver DNS Firewall |
| ALB/NLB/API Gateway/CloudFrontでTLS証明書を使いたい | AWS Certificate Manager |
| CloudFront用のACM証明書 | us-east-1で発行またはimport |

## 全体像

```text
Internet / Client
  |
  +--> CloudFront
  |      +--> AWS WAF
  |      +--> AWS Shield
  |      +--> ACM certificate in us-east-1
  |
  +--> Global Accelerator / ALB / NLB
         +--> AWS Shield Advanced
         +--> AWS WAF on ALB
         +--> ACM certificate

VPC
  |
  +--> Route 53 Resolver DNS Firewall
  |      +--> outbound DNS query filtering
  |
  +--> AWS Network Firewall
  |      +--> stateless rules
  |      +--> stateful Suricata IPS rules
  |
  +--> VPC Traffic Mirroring
         +--> IDS appliance

AWS Organizations
  |
  +--> AWS Firewall Manager
         +--> WAF / Shield Advanced / SG / NACL / Network Firewall / DNS Firewall policies
```

## サービス選択の軸

| 軸 | L3/L4 | L7 HTTP | DNS | 証明書 | 複数アカウント管理 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| AWS Shield | DDoS | DDoS | Route 53 hosted zone保護 | - | Firewall Managerと連携可 |
| AWS WAF | - | Web攻撃制御 | - | - | Firewall Managerと連携可 |
| AWS Network Firewall | 通信制御 | 一部L7/DPI | 直接ResolverのDNSは見えない | - | Firewall Managerと連携可 |
| Route 53 Resolver DNS Firewall | - | - | DNS query制御 | - | Firewall Managerと連携可 |
| ACM | - | TLS終端で利用 | DNS検証あり | SSL/TLS証明書 | - |
| Firewall Manager | - | - | - | - | 中央管理 |

## AWS Shield

AWS ShieldはDDoS対策サービスである。

```text
DDoS = 大量通信や大量リクエストでサービスを使えなくする攻撃
```

## Shield StandardとShield Advanced

| 項目 | Shield Standard | Shield Advanced |
| :--- | :--- | :--- |
| 料金 | 追加料金なし | 有料 |
| 対象 | AWS利用時に自動適用される基本保護 | 指定した保護対象リソース |
| レイヤー | L3/L4中心の基本保護 | L3/L4/L7の高度なDDoS保護 |
| 代表機能 | 一般的なDDoS緩和 | 高度な検知、可視化、SRT連携、アプリケーション層DDoS緩和 |
| 試験での出方 | 「基本的なDDoS保護」 | 「重要サイト」「高度なDDoS」「SRT」「詳細な攻撃可視化」 |

## Shield Advancedの主な保護対象

覚える対象:

```text
CloudFront distributions
Route 53 hosted zones
Global Accelerator standard accelerators
Elastic IP addresses
EC2 instances associated with protected EIP
Application Load Balancers
Classic Load Balancers
Network Load Balancers through protected EIP
```

注意:

```text
Shield Advancedは、保護対象として明示したリソースを保護する。
何も指定しなくても全リソースを自動で高度保護するわけではない。
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| DDoS対策、CloudFront、Route 53、ALBが出る | Shield Advanced |
| L7 HTTP floodをWAFルールで自動緩和したい | Shield Advanced + AWS WAF |
| 組織内の複数アカウントへShield Advancedを展開 | Firewall Manager Shield Advanced policy |
| NAT Gatewayのアウトバウンド保護 | ShieldではなくNetwork Firewallを検討 |

## AWS WAF

AWS WAFはWeb Application Firewallである。HTTP/HTTPSリクエストを検査して、Webアプリケーションを保護する。

```text
Client
  -> CloudFront / ALB / API Gateway
  -> AWS WAF web ACL
  -> Application
```

## WAFで防ぐもの

代表例:

```text
SQL injection
Cross-site scripting
HTTP flood
bad bot
IP reputation
country match
rate-based request
header / cookie / query string / body match
regex match
```

WAFの基本単位:

| 用語 | 意味 |
| :--- | :--- |
| Web ACL | WAF設定の本体。保護対象リソースに関連付ける |
| Rule | 条件とアクションの組み合わせ |
| Rule group | 複数ルールの再利用単位 |
| Managed rule group | AWSまたはMarketplace提供の管理ルール |
| IP set | 許可/拒否したいIPアドレス集合 |
| Regex pattern set | 正規表現パターン集合 |
| WCU | Web ACL capacity unit。WAFルールの容量単位 |
| Label | ルール一致結果に付与される内部ラベル |

## WAFのアクション

| Action | 意味 |
| :--- | :--- |
| Allow | 許可 |
| Block | 拒否 |
| Count | 検知だけして許可。新ルールの事前検証に使う |
| CAPTCHA | CAPTCHAを要求 |
| Challenge | サイレントなチャレンジを要求 |

試験では、いきなりBlockではなく、まずCountで影響確認する選択肢が安全側として出ることがある。

## WAFの保護対象

代表的な保護対象:

```text
CloudFront distribution
Application Load Balancer
API Gateway REST API
AppSync GraphQL API
Cognito user pool
App Runner service
Verified Access instance
Amplify
```

重要なリージョンの違い:

| 対象 | WAFのScope / Region |
| :--- | :--- |
| CloudFront | Global扱い。実体はus-east-1で作成 |
| ALB / API Gatewayなど | 対象リソースと同じリージョン |

覚え方:

```text
WAF + CloudFront = global / us-east-1
WAF + ALB        = regional
```

## WAFでできないこと

| できないこと | 使う候補 |
| :--- | :--- |
| 任意のTCP/UDP通信を検査 | Network Firewall / GWLB appliance |
| VPC内のすべての通信を経路上で検査 | Network Firewall / GWLB appliance |
| DNS問い合わせ先ドメインをResolverで制御 | Route 53 Resolver DNS Firewall |
| TLS証明書を発行/更新 | ACM |
| DDoS専用の高度保護 | Shield Advanced |

## AWS Firewall Manager

AWS Firewall Managerは、AWS Organizations配下の複数アカウントに対して、セキュリティポリシーを中央管理するサービスである。

```text
Security administrator account
  -> Firewall Manager policy
  -> Organization accounts / OUs
  -> in-scope resources
```

## Firewall Managerで管理できる代表ポリシー

| Policy type | 管理対象 |
| :--- | :--- |
| AWS WAF policy | Web ACL / rule group適用 |
| Shield Advanced policy | Shield Advanced保護 |
| Security group policy | SGの監査、共通ルール適用 |
| Network ACL policy | NACLの共通ルール適用 |
| Network Firewall policy | VPCへのNetwork Firewall適用 |
| Route 53 Resolver DNS Firewall policy | VPCへのDNS Firewall rule group関連付け |

## Firewall Managerの前提

覚える前提:

```text
AWS Organizations
Firewall Manager administrator account
AWS Config
```

Network FirewallやDNS Firewallのポリシーでは、リソース共有のためにAWS RAMが関係することがある。

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 複数アカウントに同じWAFルールを強制 | Firewall Manager |
| 新規アカウント/新規リソースにも自動適用 | Firewall Manager |
| Organization全体のSecurity Groupを監査 | Firewall Manager |
| 単一ALBだけにWAFを付けたい | WAF単体でよい |
| 単一VPCだけにFirewallを入れたい | Network Firewall単体でよい |

## AWS Network Firewall

AWS Network Firewallは、VPC向けのマネージドネットワークファイアウォールである。

主に次の用途で出る。

```text
egress filtering
east-west inspection
centralized inspection
stateful firewall
Suricata compatible IPS rules
domain filtering over HTTP/TLS SNI
deep packet inspection
```

## Network Firewallのルール

| 種類 | 内容 |
| :--- | :--- |
| Stateless rules | パケット単位。送信元/宛先IP、ポート、プロトコルなど |
| Stateful rules | 通信フロー単位。Suricata互換ルール、DPI、ドメイン/アプリ層の検査 |

処理の大枠:

```text
packet
  -> stateless rules
  -> default stateless action
  -> stateful rules
  -> pass / drop / reject / alert
```

## SuricataとIPS

Network Firewallのstateful rule groupでは、Suricata互換のIPS仕様を使える。

```text
Suricata = オープンソースのIDS/IPSエンジン
IPS      = Intrusion Prevention System
IDS      = Intrusion Detection System
DPI      = Deep Packet Inspection
```

試験での意味:

| 用語 | 動き |
| :--- | :--- |
| IDS | 検知・通知・ログ取得。通信は基本的に止めない |
| IPS | 検知したうえで通信を止める |
| alert | ログ/警告。遮断しない |
| drop | 通信を破棄して遮断 |
| reject | 通信を拒否して遮断 |
| pass | 通信を通す |

覚え方:

```text
IDS = 見る
IPS = 止める
Network Firewall = IPSとして使える
Traffic Mirroring = IDS向けにコピーを渡す
```

## IDS構成とVPC Traffic Mirroring

VPC Traffic Mirroringは、EC2インスタンスのENIに出入りする通信コピーを、監視アプライアンスへ送る仕組みである。

```text
EC2 ENI
  -> mirrored traffic
  -> IDS appliance
```

特徴:

| 項目 | 内容 |
| :--- | :--- |
| 目的 | 監視、分析、IDS |
| 通信 | コピーを送る。元通信の経路上にはいない |
| 遮断 | 基本的にできない |
| ターゲット | ENI、NLB、GWLB endpointなど |
| カプセル化 | VXLAN |

試験での切り分け:

```text
検知だけでよい / out-of-band = Traffic Mirroring + IDS
通信を止めたい / inline = Network Firewall or GWLB appliance
```

## Gateway Load Balancerとセキュリティアプライアンス

Gateway Load Balancerは、サードパーティ製ファイアウォール、IDS/IPS、NGFWなどのアプライアンス群をスケールさせるために使う。

```text
Spoke VPC
  -> GWLBE
  -> GWLB
  -> security appliance fleet
  -> GWLB
  -> destination
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| GWLB | アプライアンス群への入口 |
| GWLBE | VPC側からGWLBへ到達するエンドポイント |
| Geneve | GWLBで使われるカプセル化プロトコル |
| 代表用途 | centralized inspection、third-party firewall、IDS/IPS |
| 注意 | ステートフル検査では往復経路を同じアプライアンスへ通す設計が重要 |

TGWと組み合わせる場合:

```text
Transit Gateway
  -> Inspection VPC attachment
  -> appliance mode support
```

ステートフル検査では、戻り通信が別アプライアンスへ流れるとセッションが壊れる。TGWのappliance mode supportは、このような非対称ルーティングを避けるために重要になる。

## Route 53 Resolver DNS Firewall

Route 53 Resolver DNS Firewallは、VPC内のリソースが行うアウトバウンドDNS問い合わせを制御する。

```text
EC2 / workload in VPC
  -> VPC Resolver
  -> DNS Firewall rule group
  -> allowed / blocked / alerted
```

主な目的:

```text
malicious domain block
DNS exfiltration protection
DNS tunneling detection
DGA domain detection
許可済みドメインだけ問い合わせ許可
悪性ドメインだけ拒否
```

## DNS Firewallの構成要素

| 用語 | 意味 |
| :--- | :--- |
| Rule group | DNS Firewallルールの集合 |
| Rule | domain listやAdvanced protectionに対するaction |
| Domain list | ドメイン名のリスト。AWS managedまたは自前 |
| Rule priority | 小さい数字から評価 |
| Association | Rule groupをVPCへ関連付けること |

## DNS Firewallのアクション

| Action | 動き |
| :--- | :--- |
| Allow | 許可して検査終了 |
| Alert | 許可するがResolver query logに記録 |
| Block | DNS問い合わせをブロック |

Block時の応答:

| Response | 意味 |
| :--- | :--- |
| NODATA | 名前はあるが該当レコードなしのように返す |
| NXDOMAIN | ドメインが存在しないように返す |
| OVERRIDE | 指定したCNAMEを返す |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| VPCから悪性ドメインへの名前解決を止めたい | DNS Firewall |
| DNS exfiltrationを防ぎたい | DNS Firewall |
| DNS tunneling / DGAを検知・ブロック | DNS Firewall Advanced |
| HTTP HostやTLS SNIで通信を止めたい | Network Firewall |
| Web request headerで止めたい | WAF |

重要:

```text
DNS FirewallはDNS問い合わせを見る。
Network Firewallは通信経路上のネットワーク/アプリケーション通信を見る。
DNS Firewallだけでは、IP直打ち通信は止められない。
```

## AWS Certificate Manager

AWS Certificate Managerは、SSL/TLS証明書の発行、保存、更新、利用を管理するサービスである。

```text
ACM certificate
  -> ALB / NLB / CloudFront / API Gatewayなど
  -> HTTPS / TLS
```

## ACMで覚えること

| 項目 | 内容 |
| :--- | :--- |
| 証明書種別 | Public certificate、Private certificate、Imported certificate |
| 検証方式 | DNS validation、Email validation |
| 自動更新 | ACM発行証明書は条件を満たすと管理更新される |
| Imported certificate | 自動更新されない。利用者が更新/importする |
| リージョン性 | ACM証明書はリージョナルリソース |
| CloudFront | Viewer向け証明書はus-east-1で発行/import |
| ELB | ALB/NLBと同じリージョンのACM証明書を使う |
| Wildcard | `*.example.com`のようなワイルドカード証明書を利用可能 |

## ACMとDNS検証

DNS validationでは、ACMが提示するCNAMEレコードをDNSに登録して、ドメイン所有を証明する。

```text
ACM
  -> validation CNAME
  -> Route 53 public hosted zone / external DNS
  -> certificate issued
```

覚えること:

```text
DNS validationのCNAMEを残しておくと、自動更新に有利
public certificateのDNS検証はpublic DNSで解決できる必要がある
private hosted zoneだけではpublic certificateの検証に使えない
```

## ACM Private CA

ACM Private CAは、組織内向けのプライベート証明書を発行するマネージドCAである。

```text
ACM Private CA
  -> private certificate
  -> internal ALB / private API / service-to-service TLS / mTLS
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 用途 | 社内サービス、内部ALB、プライベートAPI、mTLS、サービス間TLS |
| 信頼範囲 | 組織内/管理対象端末/管理対象サービス |
| ACMとの関係 | ACMで証明書を管理し、Private CAで発行元を管理する |
| Public certificateとの違い | インターネット一般利用者に標準で信頼される証明書ではない |

試験での見方:

- 「内部サービスに証明書を発行」「private certificate」「mTLS」「社内CA」という表現ならACM Private CAを疑う
- インターネット公開Webサイトなら通常はACM public certificate
- CloudFront viewer certificateは引き続きus-east-1のACM証明書が論点になる

## CloudFrontとACMの頻出ポイント

CloudFrontで独自ドメインのHTTPSを使う場合:

```text
ACM certificate must be in us-east-1
certificate SAN must cover alternate domain name
CloudFront distribution uses the certificate
Route 53 alias or CNAME points domain to CloudFront
```

試験での引っかけ:

| 状況 | 原因 |
| :--- | :--- |
| CloudFrontで証明書が選べない | us-east-1に証明書がない |
| CloudFrontでCNAMEを追加できない | 証明書のSANに対象ドメインがない |
| ALBでは使える証明書がCloudFrontで使えない | ALB用リージョンにあり、us-east-1にない |
| imported certificateが期限切れ | ACM自動更新対象ではない |

## Security GroupとNetwork ACL

ANS-C01では基本問題として、Security GroupとNetwork ACLの違いも出る。

| 項目 | Security Group | Network ACL |
| :--- | :--- | :--- |
| 適用単位 | ENI / Instance | Subnet |
| 状態 | Stateful | Stateless |
| ルール | Allowのみ | Allow / Deny |
| 戻り通信 | 自動許可 | 明示的に許可が必要 |
| 評価 | すべてのルール | ルール番号順 |
| 典型用途 | インスタンス単位の防御 | サブネット単位の粗い制御 |

覚え方:

```text
SG   = stateful / allow only / ENI
NACL = stateless / allow and deny / subnet
```

## 重要比較

## WAF vs Network Firewall

| 観点 | AWS WAF | AWS Network Firewall |
| :--- | :--- | :--- |
| 主対象 | Webアプリ | VPC通信 |
| レイヤー | L7 HTTP/HTTPS | L3-L7 |
| 保護対象 | CloudFront、ALB、API Gatewayなど | VPCのルート経路上の通信 |
| 代表検査 | SQLi、XSS、header、cookie、query | IP、port、protocol、domain、Suricata、DPI |
| 遮断単位 | Web request | packet / flow |
| 典型要件 | Web攻撃対策 | egress制御、east-west inspection、IPS |

## Shield vs WAF

| 観点 | Shield | WAF |
| :--- | :--- | :--- |
| 主目的 | DDoS対策 | Web request制御 |
| レイヤー | L3/L4/L7 DDoS | L7 HTTP/HTTPS |
| 代表機能 | DDoS検知・緩和 | SQLi/XSS/rate-based/IP/country |
| 組み合わせ | Shield Advanced + WAF | L7 DDoS緩和で連携 |

## DNS Firewall vs Network Firewall

| 観点 | DNS Firewall | Network Firewall |
| :--- | :--- | :--- |
| 見るもの | VPC Resolver経由のDNS問い合わせ | 経路上の通信 |
| 主目的 | DNS exfiltration、悪性ドメイン制御 | egress/east-west inspection |
| IP直打ち通信 | 止められない | 止められる |
| DNS名ベース制御 | domain list | HTTP Host / TLS SNIなど |
| 複数アカウント展開 | Firewall Manager | Firewall Manager |

## IDS/IPS構成の選び方

| 要件 | 構成 |
| :--- | :--- |
| 通信をコピーして分析したい | VPC Traffic Mirroring + IDS |
| 通信をインラインで遮断したい | Network Firewall |
| サードパーティNGFWを使いたい | GWLB + appliance |
| 複数VPCの通信を中央検査したい | TGW + Inspection VPC + Network Firewall/GWLB |
| ステートフル検査で戻り経路が不安定 | TGW appliance mode supportを確認 |
| 内部サービス向け証明書を発行したい | ACM Private CA |

## 試験での頻出シナリオ

## 1. グローバルWebアプリのDDoS対策

要件:

```text
世界中のユーザー
CloudFrontまたはGlobal Accelerator
DDoSから保護
L7攻撃も考慮
```

答え:

```text
Shield Advanced
AWS WAF
CloudFront / Global Accelerator
```

## 2. WebアプリのSQLi/XSS対策

要件:

```text
ALB配下のWebアプリ
SQL injection / XSSを防ぎたい
HTTPリクエストを検査したい
```

答え:

```text
AWS WAF web ACLをALBに関連付ける
managed rule groupを利用する
必要に応じてCountで検証してからBlock
```

## 3. 複数アカウントに同じWAFルールを強制

要件:

```text
AWS Organizations
複数アカウント
新規ALB/CloudFrontにも自動適用
中央管理
```

答え:

```text
AWS Firewall Manager
AWS WAF policy
```

## 4. VPCから悪性ドメインへの通信を止めたい

要件:

```text
EC2がマルウェア感染
悪性ドメインへDNS問い合わせ
DNS exfiltrationを防止
VPC単位で適用
```

答え:

```text
Route 53 Resolver DNS Firewall
DNS Firewall rule groupをVPCへ関連付ける
必要に応じてFirewall Managerで組織展開
```

## 5. IDS/IPSアプライアンスを中央に置きたい

要件:

```text
複数VPC
共有Inspection VPC
サードパーティIDS/IPS
ステートフル検査
戻り通信が別経路になると困る
```

答え:

```text
TGW + Inspection VPC + GWLB/GWLBE
Transit Gateway VPC attachmentでappliance mode supportを有効化
```

## 6. CloudFrontの独自ドメインHTTPS

要件:

```text
CloudFront
独自ドメイン
HTTPS
ACM certificate
```

答え:

```text
us-east-1でACM証明書を発行/import
証明書SANに独自ドメインを含める
CloudFront distributionへ関連付ける
Route 53 aliasでCloudFrontへ向ける
```

## よくある誤解

| 誤解 | 正しい理解 |
| :--- | :--- |
| WAFはすべての通信を守る | WAFはHTTP/HTTPSのWeb request向け |
| Shield Advancedは全AWSリソースを自動で高度保護 | 指定した保護対象リソースを保護する |
| Firewall ManagerはFirewallそのもの | 中央管理サービス。実際の防御はWAF/Shield/SG/Network Firewallなど |
| DNS Firewallで全アウトバウンド通信を止められる | DNS問い合わせを制御する。IP直打ちは別対策が必要 |
| IDSとIPSは同じ | IDSは検知、IPSは遮断 |
| Traffic Mirroringで通信をブロックできる | ミラーはコピーなので通常はブロックしない |
| ACM証明書はリージョンをまたいでそのまま使える | ACM証明書はリージョナル。CloudFront用はus-east-1 |
| Private CA証明書はインターネット利用者にそのまま信頼される | Private CAは組織内の信頼基盤。公開WebはPublic certificateを使う |
| imported certificateもACMが自動更新する | imported certificateは利用者が更新する |
| Network FirewallはResolver DNS FirewallのDNS queryを見られる | Resolver経由DNSの制御はDNS Firewall |

## 試験直前チェック

- [ ] DDoS = Shield、Web攻撃 = WAF
- [ ] Shield Standardは自動/追加料金なし、Shield Advancedは有料/高度保護
- [ ] Shield Advancedの対象にCloudFront、Route 53 hosted zone、Global Accelerator standard accelerator、ALB、EIPが含まれる
- [ ] WAFはHTTP/HTTPSのL7制御
- [ ] WAF for CloudFrontはus-east-1 / Global扱い
- [ ] WAF for ALBはRegional扱い
- [ ] 複数アカウントへの一括適用はFirewall Manager
- [ ] Firewall Managerの前提はAWS Organizations、管理者アカウント、AWS Config
- [ ] Network Firewallのstateful ruleはSuricata互換IPS
- [ ] IDSは検知、IPSは遮断
- [ ] Traffic MirroringはコピーをIDSへ送る。遮断用途ではない
- [ ] DNS FirewallはVPC Resolver経由のアウトバウンドDNS問い合わせを制御
- [ ] DNS FirewallのBlock応答はNODATA、NXDOMAIN、OVERRIDE
- [ ] ACM証明書はリージョナル
- [ ] 内部証明書、mTLS、社内CA要件ではACM Private CAを検討する
- [ ] CloudFront用ACM証明書はus-east-1
- [ ] imported certificateは自動更新対象ではない
- [ ] ステートフルinspectionでは対称ルーティングが重要

## 公式参照

- [What are AWS WAF, AWS Shield Advanced, and AWS Firewall Manager?](https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html)
- [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html)
- [Resources that you can protect with AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works-resources.html)
- [How AWS Shield and Shield Advanced work](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-overview.html)
- [List of resources that AWS Shield Advanced protects](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-protections-by-resource-type.html)
- [AWS Firewall Manager](https://docs.aws.amazon.com/waf/latest/developerguide/fms-chapter.html)
- [AWS Firewall Manager prerequisites](https://docs.aws.amazon.com/waf/latest/developerguide/fms-prereq.html)
- [Using AWS Firewall Manager policies](https://docs.aws.amazon.com/waf/latest/developerguide/working-with-policies.html)
- [Working with stateful rule groups in AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups-ips.html)
- [How AWS Network Firewall filters network traffic](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-policy-processing.html)
- [Managing evaluation order for Suricata compatible rules](https://docs.aws.amazon.com/network-firewall/latest/developerguide/suricata-rule-evaluation-order.html)
- [Using DNS Firewall to filter outbound DNS traffic](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall.html)
- [Rule actions in DNS Firewall](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-rule-actions.html)
- [DNS Firewall Foundational Rules](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-domain-lists.html)
- [What is AWS Certificate Manager?](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
- [What is AWS Private CA?](https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html)
- [Conditions for using AWS Private CA to sign ACM private certificates](https://docs.aws.amazon.com/acm/latest/userguide/ca-access.html)
- [AWS Certificate Manager DNS validation](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html)
- [Managed certificate renewal in AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/managed-renewal.html)
- [Requirements for using SSL/TLS certificates with CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html)
- [How Traffic Mirroring works](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-how-it-works.html)
