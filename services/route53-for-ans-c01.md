# Amazon Route 53 ANS-C01対策

## 目的

ANS-C01で出題されるAmazon Route 53、Route 53 VPC Resolver、Private Hosted Zone、ハイブリッドDNS、DNS Firewall、ヘルスチェックを試験向けに整理する。

Route 53は単なるDNSサービスとしてではなく、**AWS内外の名前解決をどうつなぐか**、**どのDNS応答を返すか**、**DNSレベルでどう制御・監視するか**が問われる。

## 最初に覚える結論

| 要件 | 疑うもの |
| :--- | :--- |
| インターネット向けドメインを解決したい | Public Hosted Zone |
| VPC内だけで名前解決したい | Private Hosted Zone |
| オンプレからVPC内DNS名を解決したい | Resolver inbound endpoint |
| VPCからオンプレDNS名を解決したい | Resolver outbound endpoint + Resolver rule |
| 同じドメイン名で内部/外部の答えを変えたい | Split-horizon DNS |
| VPCのDNSクエリをログ化したい | Resolver query logging |
| Public Hosted ZoneへのDNS問い合わせをログ化したい | Public DNS query logging |
| VPCからの怪しいDNS問い合わせを止めたい | Route 53 Resolver DNS Firewall |
| DNSでActive/Passive切替したい | Failover routing + Health check |
| 複数リージョンで低遅延へ寄せたい | Latency routing |
| 比率で段階的に切替したい | Weighted routing |
| ユーザーの地域で振り分けたい | Geolocation / Geoproximity routing |

## Route 53とは

Amazon Route 53は、AWSのDNSサービスである。主な役割は次の3つ。

```text
1. ドメイン登録
2. DNSホスティング
3. ヘルスチェックとDNSルーティング
```

ANS-C01では、ドメイン登録よりも以下が重要。

```text
Public Hosted Zone
Private Hosted Zone
Route 53 VPC Resolver
Resolver inbound / outbound endpoint
Resolver rule
DNS Firewall
Routing policy
Health check
Alias record
```

## DNSの基本

DNSは、名前をIPアドレスなどに変換する仕組みである。

```text
www.example.com
  -> DNS問い合わせ
  -> 203.0.113.10
```

Route 53は、問い合わせに対してどのレコードを返すかを制御する。

| 用語 | 意味 |
| :--- | :--- |
| Hosted Zone | DNSレコードを入れる箱 |
| Record | DNS応答の中身 |
| A record | IPv4アドレスを返す |
| AAAA record | IPv6アドレスを返す |
| CNAME | 別のDNS名へ別名設定する |
| NS | そのゾーンの権威DNSサーバ |
| SOA | ゾーンの管理情報 |
| MX | メール配送先 |
| TXT | 文字列情報。SPF/DKIM検証など |
| TTL | Resolverが応答をキャッシュする時間 |

## Public Hosted Zone

Public Hosted Zoneは、インターネット向けのDNSゾーンである。

```text
Internet user
  -> Route 53 Public Hosted Zone
  -> ALB / CloudFront / Global Accelerator / Public IP
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 対象 | インターネット上の利用者 |
| 代表用途 | Webサイト、API、公開サービス |
| レコード例 | `www.example.com -> ALB` |
| Query logging | CloudWatch Logsへ出力 |
| Health check | Public endpointの監視に使いやすい |

試験キーワード:

```text
internet-facing
public domain
external users
registered domain
```

## Domain Registration / Delegation

Route 53はDNSホスティングだけでなく、ドメイン登録にも使える。

ただし試験では、次の2つを分けて考える。

| 項目 | 意味 |
| :--- | :--- |
| Domain registration | `example.com` というドメインをレジストラで登録する |
| Public Hosted Zone | `example.com` のDNSレコードをRoute 53で管理する |
| Delegation | レジストラ側で、Route 53 Hosted ZoneのNSレコードを権威DNSとして指定する |

重要ポイント:

- Public Hosted Zoneを作成するとNSレコードが割り当てられる
- インターネットからRoute 53のレコードを見せるには、ドメインのNS委任がRoute 53 Hosted ZoneのNSへ向いている必要がある
- ドメイン登録がRoute 53以外でも、NSをRoute 53へ委任すればRoute 53でDNS管理できる
- 親ドメインから子ドメインへ委任する場合もNSレコードを使う

ひっかけ:

- Hosted Zoneにレコードを追加しただけでは、レジストラ側NSが別DNSを向いていると外部利用者には反映されない
- TTLが長いと変更反映に時間がかかる
- DNS移行では、古いDNSと新しいRoute 53 Hosted Zoneの両方に必要レコードが揃っているか確認する

## Private Hosted Zone

Private Hosted Zoneは、VPC内向けのDNSゾーンである。

```text
EC2 / ECS / Lambda in VPC
  -> Route 53 VPC Resolver
  -> Private Hosted Zone
  -> db.internal.example.com = 10.0.10.50
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 対象 | 関連付けられたVPC内 |
| 用途 | 内部サービス名、DB名、社内向け名前解決 |
| VPC関連付け | PHZは1つ以上のVPCに関連付ける |
| 必要設定 | `enableDnsHostnames=true`, `enableDnsSupport=true` |
| クロスアカウント | CLI/APIで認可して関連付ける |
| インターネット公開 | されない |

重要:

```text
Private Hosted ZoneはVPCに関連付けないと使えない
オンプレから直接VPC+2へ問い合わせるのは非推奨/不安定
オンプレから解決する場合はResolver inbound endpointを使う
```

## Split-horizon DNS

Split-horizon DNSは、同じドメイン名に対して内部と外部で違う答えを返す設計である。

```text
Public Hosted Zone:  example.com
  www.example.com -> public ALB

Private Hosted Zone: example.com
  www.example.com -> internal ALB
```

VPC内からはPrivate Hosted Zoneの答えを返し、インターネットからはPublic Hosted Zoneの答えを返す。

試験キーワード:

```text
same domain internally and externally
internal users need private IP
external users need public endpoint
split-view DNS
split-horizon DNS
```

注意:

```text
Private Hosted Zoneが一致したが、該当レコードがない場合、
public DNSへフォールバックせずNXDOMAINになる
```

## Route 53 VPC Resolver

Route 53 VPC Resolverは、VPC内でデフォルト利用できるDNS Resolverである。

```text
VPC CIDR: 10.0.0.0/16
Resolver: 10.0.0.2
```

一般に、VPCのベースアドレス + 2 のIPで使える。

解決できるもの:

| 対象 | 内容 |
| :--- | :--- |
| EC2 private DNS | EC2のVPC内DNS名 |
| Private Hosted Zone | VPCに関連付けられたPHZ |
| Public DNS | インターネット上の公開DNS名 |

公式では現在「Route 53 VPC Resolver」という表記があるが、試験や問題文では「Route 53 Resolver」と書かれることも多い。

## Inbound Resolver Endpoint

Inbound endpointは、オンプレミスや別ネットワークからVPC側のDNSを解決する入口である。

```text
オンプレDNS
  -> Direct Connect / VPN
  -> Resolver inbound endpoint
  -> Route 53 VPC Resolver
  -> Private Hosted Zone
```

使う場面:

| 要件 | 答え |
| :--- | :--- |
| オンプレから`app.aws.internal`を解決したい | Inbound endpoint |
| オンプレからPrivate Hosted Zoneを引きたい | Inbound endpoint |
| オンプレDNSにAWS向け条件付きフォワードを設定したい | Inbound endpointのIPを指定 |

設計ポイント:

```text
高可用性のため複数AZ/複数IPで作る
オンプレDNS側のforwarderにinbound endpointのIPを設定する
Security Group/NACLでTCP/UDP 53を許可する
```

## Outbound Resolver Endpoint

Outbound endpointは、VPCからオンプレミスDNSへ問い合わせる出口である。

```text
EC2 in VPC
  -> Route 53 VPC Resolver
  -> Resolver rule
  -> Resolver outbound endpoint
  -> Direct Connect / VPN
  -> オンプレDNS
```

使う場面:

| 要件 | 答え |
| :--- | :--- |
| VPCから`corp.example.local`を解決したい | Outbound endpoint + Resolver rule |
| AWSワークロードからオンプレDNSを引きたい | Outbound endpoint |
| 複数VPCで同じオンプレDNS転送を使いたい | Resolver ruleをRAMで共有 |

設計ポイント:

```text
Outbound endpointは複数VPCで共有しやすい
Resolver ruleに転送対象ドメインとオンプレDNSのIPを設定する
RulesをVPCに関連付ける
Endpointはリージョンをまたいで使えない
```

## Resolver Rule

Resolver ruleは、どのドメインをどこへ転送するかを決めるルールである。

```text
corp.example.local
  -> 192.168.10.10
  -> 192.168.10.11
```

覚えること:

| 種類 | 用途 |
| :--- | :--- |
| Forward rule | 指定ドメインをオンプレDNSなどへ転送 |
| System rule | 特定ドメインをResolverで解決し、転送を抑止 |
| Delegation rule | サブドメインの権威委任に使う |

重要:

```text
Private Hosted ZoneとResolver ruleが同じドメイン名で競合する場合、
Resolver ruleが優先される
```

## ハイブリッドDNSの基本パターン

### オンプレからAWSを引く

```text
オンプレサーバ
  -> オンプレDNS
  -> forward: aws.internal
  -> Resolver inbound endpoint
  -> Private Hosted Zone
```

答え:

```text
Inbound Resolver Endpoint
```

### AWSからオンプレを引く

```text
EC2 / ECS
  -> Route 53 VPC Resolver
  -> Resolver rule: corp.local
  -> Resolver outbound endpoint
  -> オンプレDNS
```

答え:

```text
Outbound Resolver Endpoint + Resolver Rule
```

## Query Logging

Route 53には大きく2種類のDNSログがある。

| 種類 | 対象 | 出力先 |
| :--- | :--- | :--- |
| Public DNS query logging | Public Hosted Zoneへの問い合わせ | CloudWatch Logs |
| Resolver query logging | VPC Resolver経由の問い合わせ | CloudWatch Logs / S3 / Firehose |

Resolver query loggingで記録できるもの:

```text
VPC内からのDNSクエリ
Inbound endpointを通ったクエリ
Outbound endpointを通ったクエリ
DNS Firewallでallow/block/alertされたクエリ
```

注意:

```text
ResolverがTTL内のキャッシュから応答した場合、同じ問い合わせが毎回ログに出るとは限らない
```

## DNS Firewall

Route 53 Resolver DNS Firewallは、VPCから出るDNS問い合わせをドメイン名で制御する機能である。

```text
EC2
  -> Route 53 VPC Resolver
  -> DNS Firewall
  -> allow / block / alert
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 制御対象 | VPC Resolverを通るDNSクエリ |
| 単位 | DNS Firewall rule groupをVPCに関連付け |
| アクション | Allow / Block / Alert |
| 防げるもの | 悪性ドメイン、DNS exfiltration、DGA、DNS tunnelingなど |
| ログ | Resolver query logsに記録可能 |
| 集中管理 | Firewall Managerで複数アカウントへ展開可能 |

重要:

```text
DNS Firewallはドメイン名ベースでDNS問い合わせを制御する
HTTPSやSSHなどDNS以外の通信そのものを検査/遮断するものではない
```

Network Firewallとの違い:

| サービス | 見るもの |
| :--- | :--- |
| DNS Firewall | DNSクエリのドメイン名 |
| Network Firewall | IP/TCP/UDP/TLS/SNI/HTTPなどのネットワーク通信 |

## Alias Record

Alias recordは、Route 53独自のレコードで、AWSリソースへDNS名で向けるために使う。

```text
example.com
  -> ALB
```

使う場面:

| 宛先 | Alias利用 |
| :--- | :--- |
| ALB/NLB | よく使う |
| CloudFront | よく使う |
| S3 static website endpoint | よく使う |
| API Gateway | よく使う |
| Global Accelerator | よく使う |

重要:

```text
Zone apexにはCNAMEを置けない
Zone apexでALBなどへ向けたい場合はAlias recordを使う
```

例:

```text
example.com      -> ALB  # Aliasなら可能
www.example.com  -> ALB  # CNAMEでもAliasでも可能
```

## Routing Policy

Route 53のRouting Policyは、DNS問い合わせに対してどのレコードを返すかを決める。

| Policy | 使う場面 | 覚え方 |
| :--- | :--- | :--- |
| Simple | 単一の宛先 | 普通 |
| Weighted | 比率で振り分け | 70:30、段階移行、Blue/Green |
| Latency | 最低レイテンシーのリージョンへ | 速いところへ |
| Failover | Active/Passive | 障害時切替 |
| Geolocation | ユーザーの地理的位置で振り分け | 国/地域ベース |
| Geoproximity | リソース位置とbiasで振り分け | 近さ + 重み調整 |
| Multivalue answer | 最大8個の正常レコードを返す | 簡易分散 |
| IP-based | 送信元IP範囲で振り分け | CIDRベース |

試験での見抜き方:

| 問題文 | Policy |
| :--- | :--- |
| 一部ユーザーだけ新環境へ | Weighted |
| 低遅延のリージョンへ | Latency |
| 障害時にDRサイトへ | Failover |
| 日本ユーザーは東京、欧州ユーザーは欧州 | Geolocation |
| 地理的近さを調整したい | Geoproximity |
| 複数IPを返して簡易分散 | Multivalue answer |
| 特定CIDRのユーザーを専用エンドポイントへ | IP-based |

Private Hosted Zoneでは使えるRouting Policyに制限がある。少なくとも試験では、Public/Privateで常に同じことができると決めつけない。

## Health Check

Route 53 Health Checkは、DNS応答に使う宛先の正常性を判断するために使う。

```text
Route 53 health checker
  -> HTTP / HTTPS / TCP
  -> endpoint
```

使いどころ:

| 要件 | 答え |
| :--- | :--- |
| Active/Passive切替 | Failover routing + Health check |
| 正常な宛先だけ返したい | Health check |
| ELBの状態を見たい | Alias + Evaluate Target Health |
| Private IPの内部エンドポイントを見たい | CloudWatch alarm based health checkを検討 |

重要:

```text
Route 53 health checkersはVPC外にいる
Private IPや非ルーティングアドレスを直接ヘルスチェックできない
Private Hosted Zoneで内部リソースを判定したい場合はCloudWatch metric/alarm based health checkを使う選択肢がある
```

## Evaluate Target Health

Alias recordでAWSリソースへ向ける場合、Evaluate Target Healthを使うと、Alias先の状態を見てDNS応答を制御できる。

```text
example.com
  -> Alias to ALB
  -> Evaluate Target Health = Yes
```

試験では、ELBや別レコードに向けるAliasとHealth Checkの組み合わせで出やすい。

## Route 53とGlobal Accelerator / CloudFrontの違い

| サービス | 役割 |
| :--- | :--- |
| Route 53 | DNSで名前を解決し、どの宛先を返すか決める |
| CloudFront | コンテンツをエッジでキャッシュ/配信する |
| Global Accelerator | 固定Anycast IPでAWSグローバルネットワークへ入れる |

試験での見方:

```text
DNS名で複数宛先を返す = Route 53
キャッシュしたい = CloudFront
固定IP + 低遅延 + TCP/UDP入口最適化 = Global Accelerator
```

## DNSSEC / DNS Delegationの注意

DNSSECは、DNS応答が改ざんされていないことを検証する仕組みである。

覚えること:

| 用語 | 意味 |
| :--- | :--- |
| Zone signing | Hosted Zone内のDNSレコードに署名する |
| KSK | Key Signing Key。DNSSECの信頼の起点側で使う鍵 |
| DS record | 親ゾーンに登録し、子ゾーンの鍵を信頼させるためのレコード |
| Validation | Resolver側が署名を検証すること |

試験での見方:

- 「DNS応答の改ざん検知」「DNSの信頼性」「署名」という表現ならDNSSECを疑う
- Public Hosted ZoneのDNSSECでは、親ゾーン/レジストラ側のDSレコード登録が必要になる
- Private Hosted Zoneの名前解決や転送設計は、DNSSECよりResolver Endpoint/Resolver Ruleの論点になりやすい

## よくある失敗パターン

| 症状 | 原因候補 |
| :--- | :--- |
| Public Hosted Zoneにレコードを作ったが外部から解決できない | レジストラ/親ゾーンのNS委任がRoute 53 Hosted Zoneを向いていない |
| PHZの名前が解決できない | VPC関連付け漏れ、enableDnsHostnames/enableDnsSupport無効 |
| オンプレからPHZを引けない | Inbound endpoint未構成、オンプレDNS forwarder未設定 |
| VPCからオンプレDNSを引けない | Outbound endpointまたはResolver rule未設定 |
| PrivateとPublicで同じ名前なのにPublic側へ行かない | PHZが一致し、NXDOMAINになっている |
| Resolver ruleを作ったらPHZが引けない | 同一ドメインではResolver ruleがPHZより優先 |
| DNSログに全クエリが出ない | Resolverキャッシュにより再問い合わせされていない |
| DNS FirewallでHTTPS通信が止まらない | DNS FirewallはDNS問い合わせだけを制御する |
| Private IPのヘルスチェックが失敗する | Route 53 health checkerはVPC外にいる |
| 複数VPC/複数アカウントでDNS運用が煩雑 | Resolver rule共有、PHZ関連付け、Route 53 Profilesを検討 |

## 最小暗記セット

```text
Public Hosted Zone  = インターネット向けDNS
Private Hosted Zone = VPC内向けDNS
Inbound endpoint    = オンプレ -> AWS DNS
Outbound endpoint   = AWS -> オンプレDNS
Resolver rule       = どのドメインをどこへ転送するか
Split-horizon DNS   = 同じ名前で内部/外部の答えを変える
DNS Firewall        = VPCのDNS問い合わせをドメイン名で制御
Alias record        = Zone apexでもAWSリソースへ向けられる
Health check        = DNSフェイルオーバー判断
Delegation          = 親ゾーン/レジストラからNSで委任
DNSSEC              = DNS応答の署名/検証、KSKとDS
```

## 試験直前チェック

- [ ] Public Hosted ZoneとPrivate Hosted Zoneの違いを説明できる
- [ ] Public Hosted Zone作成とドメインNS委任の違いを説明できる
- [ ] Inbound endpointとOutbound endpointの向きを間違えない
- [ ] Resolver ruleはOutbound endpointとセットで考える
- [ ] オンプレからVPC+2へ直接問い合わせる設計を選ばない
- [ ] PHZには`enableDnsHostnames`と`enableDnsSupport`が必要
- [ ] PHZが一致してレコードがない場合はPublic DNSへフォールバックしない
- [ ] 同一ドメインではResolver ruleがPHZより優先される
- [ ] Alias recordとCNAMEの違いを説明できる
- [ ] Weighted / Latency / Failover / Geolocation / Geoproximityを選び分けられる
- [ ] Route 53 Health CheckはPrivate IPを直接チェックできない
- [ ] DNS FirewallはDNS問い合わせを制御し、HTTPS通信そのものを検査するものではない
- [ ] Resolver query loggingとPublic DNS query loggingの対象を区別できる
- [ ] DNSSECは署名/検証、KSK、DSレコードの関係で覚える

## 公式参照

- [Amazon Route 53 concepts](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/route-53-concepts.html)
- [What is Amazon Route 53?](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Registering and managing domains using Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/registrar.html)
- [Registering a new domain](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-register.html)
- [Choosing a routing policy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [Working with hosted zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html)
- [Working with private hosted zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html)
- [Private hosted zone considerations](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-considerations.html)
- [What is Route 53 VPC Resolver?](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html)
- [Resolver endpoints and forwarding rules](https://docs.aws.amazon.com/whitepapers/latest/hybrid-cloud-dns-options-for-vpc/route-53-resolver-endpoints-and-forwarding-rules.html)
- [Resolver query logging](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-query-logs.html)
- [Resolver DNS Firewall](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall-overview.html)
- [Enabling DNSSEC signing and establishing a chain of trust](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec-enable-signing.html)
- [Configuring failover in a private hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-private-hosted-zones.html)
