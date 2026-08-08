# TGW / VIF / VPN接続 チートシート

## 目的

ANS-C01で頻出のTransit Gateway（TGW）、Direct ConnectのVIF、Site-to-Site VPN接続を、問題文から判断しやすい形で整理する。

見直し後の対象範囲は、次の「拠点間ネットワーキング」まで含める。

```text
VPC間接続
複数アカウント間接続
複数リージョン間接続
オンプレミス - AWS接続
リモートユーザー - AWS接続
VPCからAWSサービスへのprivate接続
VPCから他アカウント/自社サービスへのprivate接続
集中検査VPC経由の接続
```

覚える軸は次の3つ。

```text
VIF = Direct Connect回線上の論理インターフェース
TGW = 複数VPC/オンプレ/検査VPCを集約するルーティングハブ
VPN = IPsecでオンプレとAWSをつなぐ暗号化トンネル
```

## 全体像

```text
オンプレミス / コーポレートネットワーク
  |
  | Direct Connect
  |   ├─ Public VIF  -> S3などAWSパブリックサービス
  |   ├─ Private VIF -> VGW / DXGW -> VPC
  |   └─ Transit VIF -> DXGW -> TGW -> 複数VPC
  |
  | Site-to-Site VPN
  |   ├─ VPN -> VGW -> 1つのVPC
  |   └─ VPN -> TGW -> 複数VPC
  |
Transit Gateway
  ├─ Spoke VPC
  ├─ Inspection VPC
  ├─ VPN attachment
  ├─ Direct Connect Gateway attachment
  ├─ Peering attachment
  └─ Connect attachment

VPC間 / AWS環境間
  ├─ VPC Peering
  ├─ Transit Gateway
  ├─ PrivateLink / VPC Endpoint
  ├─ Gateway Load Balancer Endpoint
  └─ AWS RAMによる共有
```

## 最初に覚える結論

| やりたいこと | 選ぶもの |
| :--- | :--- |
| Direct Connect経由でS3などAWSパブリックサービスへ行きたい | Public VIF |
| Direct Connect経由で特定VPCへプライベート接続したい | Private VIF |
| Direct Connect経由でTGW配下の複数VPCへ接続したい | Transit VIF + DXGW + TGW |
| インターネット経由で暗号化してオンプレとAWSを接続したい | Site-to-Site VPN |
| 多数VPC、複数アカウント、オンプレを集約したい | Transit Gateway |
| ステートフルFW/IDS/IPSでVPC間通信を検査したい | TGW + Inspection VPC + GWLB/GWLBE + Appliance Mode |
| SD-WANアプライアンスとTGWを動的接続したい | TGW Connect |
| 2つのVPCをシンプルにprivate接続したい | VPC Peering |
| 他アカウント/他VPCのサービスだけをprivate公開したい | AWS PrivateLink |
| VPCからS3/DynamoDBへprivate接続したい | Gateway VPC Endpoint |
| VPCからAWS APIへprivate接続したい | Interface VPC Endpoint |
| 複数アカウントでTGWやsubnetなどを共有したい | AWS RAM |
| リモートユーザーがAWS/オンプレへVPN接続したい | AWS Client VPN |
| 一対多のmulticast配信をしたい | Transit Gateway Multicast |

## 接続方式の選択表

| 接続したいもの | 第一候補 | 向いているケース | 注意点 |
| :--- | :--- | :--- | :--- |
| VPC - VPC | VPC Peering | 少数VPC、シンプル、低遅延 | 推移的ルーティング不可。多数VPCでは管理が複雑 |
| 多数VPC - 多数VPC | Transit Gateway | hub and spoke、複数アカウント、集中管理 | TGW route table設計が必要 |
| VPC - 他アカウントの特定サービス | PrivateLink | サービス単位で公開、consumer/provider分離 | VPC全体の双方向接続ではない |
| VPC - AWSサービス | VPC Endpoint | private subnetからAWS API/S3等へprivate接続 | Gateway endpointとInterface endpointを区別 |
| オンプレ - 1 VPC | VPN to VGW / Private VIF to VGW | 小規模、単一VPC | 拡張性はTGWより低い |
| オンプレ - 複数VPC | VPN to TGW / Transit VIF + DXGW + TGW | 大規模、複数VPC、複数アカウント | TGW/DXGW/route propagationが重要 |
| オンプレ - AWS public service | Public VIF | S3などpublic endpointへDX経由 | VPCへ入る用途ではない |
| リモート端末 - AWS/オンプレ | AWS Client VPN | ユーザー単位のリモートアクセス | Site-to-Site VPNとは用途が違う |
| 検査VPC経由通信 | TGW + GWLB/GWLBE + Appliance Mode | firewall/IDS/IPS集中検査 | 非対称ルーティングに注意 |
| 一対多配信 | TGW Multicast | 金融データ配信、映像配信、HPCなど | 通常のユニキャスト経路とは別に設計 |

## ANS-C01での優先度

| 優先度 | 覚えるもの |
| :--- | :--- |
| 最重要 | TGW、VIF、DXGW、Site-to-Site VPN、BGP、VPC Peering、PrivateLink |
| 重要 | VPC Endpoint、Gateway Endpoint、Interface Endpoint、AWS RAM、Appliance Mode、GWLB/GWLBE |
| 出たら拾う | Client VPN、TGW Connect、Direct Connect SiteLink、Network Manager |

試験ガイド上も、Direct Connect Gateway、Transit Gateway、VIF、PrivateLink、VPC Peering、VPN、BGP、route table、CIDR重複、route limitsなどが同じ領域で問われる。

## VIFとは

VIF（Virtual Interface）は、Direct Connect物理回線の上に作る論理インターフェースである。

```text
Direct Connect物理回線
  ├─ Public VIF
  ├─ Private VIF
  └─ Transit VIF
```

読み方は「ヴィフ」または「ブイアイエフ」。

## VIF 3種類の比較

| 種類 | 接続先 | 主な用途 | 覚え方 |
| :--- | :--- | :--- | :--- |
| Public VIF | AWS public endpoints | S3、DynamoDBなどへDX経由で接続 | Public service用 |
| Private VIF | VGWまたはDXGW | VPCへプライベートIPで接続 | VPC用 |
| Transit VIF | DXGW | TGWへ接続し複数VPCへ到達 | TGW用 |

## Public VIF

Public VIFは、AWSのパブリックサービスへDirect Connect経由で接続するためのVIFである。

```text
オンプレ
  -> Direct Connect
  -> Public VIF
  -> S3 / DynamoDB / AWS public endpoints
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 接続先 | AWS public endpoints |
| 代表例 | S3、DynamoDB、CloudFrontなど |
| IP | Public IPをBGPで使う |
| VPC接続 | VPCに直接入る用途ではない |
| DXGW | Public VIFはDXGWへ作成しない |
| 試験キーワード | S3へDX経由、Public AWS service、Public endpoint |

## Private VIF

Private VIFは、Direct ConnectからVPCへプライベート接続するためのVIFである。

```text
オンプレ
  -> Direct Connect
  -> Private VIF
  -> VGW
  -> VPC
```

または、

```text
オンプレ
  -> Direct Connect
  -> Private VIF
  -> DXGW
  -> VGW
  -> VPC
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 接続先 | VGWまたはDXGW |
| 代表用途 | 1つまたは少数のVPC接続 |
| 経路交換 | BGP |
| MTU | Jumbo frame最大 `9001` |
| 重複CIDR | DXGW配下のVPC CIDR重複に注意 |
| 試験キーワード | VGW、VPCへプライベート接続、Direct ConnectでVPC |

## Transit VIF

Transit VIFは、Direct ConnectからTransit Gatewayへ接続するためのVIFである。

```text
オンプレ
  -> Direct Connect
  -> Transit VIF
  -> DXGW
  -> TGW
  -> 複数VPC / VPN / Peering
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 接続先 | DXGW |
| TGW接続 | DXGWをTGWへ関連付ける |
| 代表用途 | 大規模、複数VPC、複数アカウント |
| 経路交換 | BGP |
| MTU | Jumbo frame最大 `8500` |
| 注意 | TGW ASNとDXGW ASNは異なる必要がある |
| 試験キーワード | Direct Connect + TGW、Transit VIF、DXGW association |

## VIFの覚え方

```text
Public VIF  = S3などPublic serviceへ
Private VIF = VPCへ
Transit VIF = TGWへ
```

迷ったら、問題文の「最終的にどこへ入りたいか」を見る。

| 問題文の宛先 | VIF |
| :--- | :--- |
| S3、DynamoDB、AWS public endpoint | Public VIF |
| 1つのVPC、VGW | Private VIF |
| TGW、複数VPC、ハブ&スポーク | Transit VIF |

## DXGWとは

DXGW（Direct Connect Gateway）は、Direct ConnectとVPC/TGWをつなぐ中継リソースである。

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| リソース種別 | グローバルリソース |
| 役割 | DXとVGW/TGWなどを中継 |
| データパス | DXGW自体が単一の物理中継点になるわけではない |
| Private VIF | DXGW経由でVGWへ接続できる |
| Transit VIF | DXGW経由でTGWへ接続する |
| Public VIF | DXGWには作らない |

## DXGW Allowed Prefix

DXGWのAllowed Prefixは、VGW関連付けとTGW関連付けで意味が違う。

| 関連付け先 | Allowed Prefixの意味 |
| :--- | :--- |
| VGW association | VPC CIDR広告のフィルター |
| TGW association | 入力したPrefix自体をオンプレへ広告 |

例:

```text
VGW association
VPC CIDR:       10.0.0.0/16
Allowed Prefix: 10.0.0.0/15
AWSから広告:    10.0.0.0/16
```

```text
TGW association
Allowed Prefix: 10.0.0.0/8
AWSから広告:    10.0.0.0/8
```

試験ではここが引っかけになりやすい。

## Direct Connectの冗長化

Direct Connectは専用線だが、1本だけでは単一障害点になる。

ANS-C01では、可用性要件に応じて複数接続を選ぶ。

| 要件 | 設計 |
| :--- | :--- |
| 最低限の冗長化 | 2本のDX接続 |
| 拠点障害にも備える | 異なるDXロケーションに接続 |
| デバイス障害にも備える | 異なるオンプレルーター/回線終端 |
| DX障害時に低コストでバックアップ | Site-to-Site VPNをbackup |
| Active/Activeで帯域利用 | 同じprefix長、同等BGP属性、ECMP |
| Active/Passiveで優先制御 | より具体的なprefix、local preference community、AS_PATH prepending |

覚えること:

```text
DX = 安定帯域、低遅延、専用接続
VPN = 暗号化、短期導入、DX backupに使いやすい
DX + VPN = よくあるハイブリッド冗長構成
```

## Direct Connect SiteLink

Direct Connect SiteLinkは、複数のオンプレミス拠点同士をAWSグローバルネットワーク経由で接続する機能である。

```text
拠点A
  -> DX location A
  -> AWS global network
  -> DX location B
  -> 拠点B
```

試験での見方:

| 問題文 | 疑う答え |
| :--- | :--- |
| オンプレ拠点間をAWS backboneで接続したい | Direct Connect SiteLink |
| VPCへ接続したい | Private VIF / Transit VIF |
| AWS public serviceへ接続したい | Public VIF |

注意:

```text
SiteLink = 拠点間接続
Transit VIF = DXからTGWへ接続
Public VIF = AWS public serviceへ接続
```

## TGWとは

TGW（Transit Gateway）は、複数のVPC、VPN、Direct Connect、Peering、Connect attachmentを集約するハブである。

```text
Spoke VPC A
Spoke VPC B
Inspection VPC
VPN
DXGW
TGW Peering
TGW Connect
    \ | /
     TGW
```

一言でいうと、

```text
TGW = AWSネットワークの中央ルーター
```

ただし、TGWは単に接続するだけでは通信できない。**TGW Route TableのAssociationとPropagation** が重要である。

## TGW Attachment

Attachmentは、TGWにぶら下がる接続点である。

| Attachment種別 | 用途 |
| :--- | :--- |
| VPC attachment | VPCをTGWへ接続 |
| VPN attachment | Site-to-Site VPNをTGWへ接続 |
| Direct Connect Gateway attachment | DXGW経由でDXをTGWへ接続 |
| Peering attachment | TGW同士を接続 |
| Connect attachment | SD-WAN/仮想アプライアンスをGRE+BGPで接続 |

## AssociationとPropagation

TGWで最重要。

```text
Association = 入ってきた通信が、どのTGW Route Tableを見るか
Propagation = そのAttachmentの経路を、どのTGW Route Tableへ載せるか
```

例:

```text
VPC A attachment
  Association: TGW-RT-Spoke
  Propagation: TGW-RT-Shared
```

意味:

```text
VPC AからTGWに入ってきた通信は TGW-RT-Spoke を見る
VPC AのCIDRは TGW-RT-Shared に伝播される
```

## Ingress / Egress

TGWを基準に見る。

```text
VPC A
  -> TGW   = Ingress
TGW
  -> VPC B = Egress
```

重要:

```text
IngressしたAttachmentにAssociationされたTGW Route Tableで、Egress先を決める
```

これを覚えると、TGW Route Table問題が読みやすくなる。

## TGW Route Table

TGW Route Tableは、TGW内部の経路表である。

| ルート種別 | 内容 |
| :--- | :--- |
| Static route | 手動で追加した経路 |
| Propagated route | Attachmentから伝播された経路 |
| Blackhole route | 意図的に破棄する経路 |

基本:

```text
より具体的なPrefixが優先
同じPrefixなら静的ルートが強い場面が多い
どのRoute Tableを見るかはAssociationで決まる
```

## TGW Peering

TGW Peeringは、TGW同士を接続する仕組みである。

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 用途 | 複数リージョン/複数TGW接続 |
| ルーティング | 静的ルートが必要 |
| 伝播 | TGW Peering経由の自動伝播には注意 |
| 推移的接続 | 設計上、意図しない推移的ルーティングを期待しない |

## Transit Gateway Multicast

Transit Gateway Multicastは、TGWを使ってVPC内のマルチキャスト通信を扱う機能である。

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 用途 | 一対多配信、金融データ配信、映像配信、HPCなど |
| 構成要素 | Multicast Domain、Group、Source、Member |
| メンバー管理 | IGMP、または静的なメンバー登録 |
| 経路 | 通常のユニキャストRoute Tableとは別に考える |

試験での注意:

- TGWの通常Attachmentを作れば自動的にマルチキャストできる、とは考えない
- Direct Connect、Site-to-Site VPN、TGW Peering、TGW Connectなどの接続先までマルチキャストを延伸できるかはサポート可否を確認する
- ほとんどのANS-C01問題では、単純なVPC間接続ではなく「一対多配信」「multicast」という単語が選択の合図になる

## VPC Peering

VPC Peeringは、2つのVPCをprivate IPで直接接続する仕組みである。

```text
VPC A
  <-> VPC Peering
VPC B
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 用途 | 少数VPC間のシンプルなprivate接続 |
| 接続範囲 | 同一リージョン、または異なるリージョン |
| アカウント | 同一アカウント、またはクロスアカウント |
| 経路 | 両側VPC route tableに相手CIDRへのrouteが必要 |
| CIDR | 重複CIDRは不可 |
| 推移的ルーティング | 不可 |
| 中央集約 | 多数VPCではTGWの方が向く |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 2つのVPCだけを接続したい | VPC Peering |
| 多数VPCをhub and spokeで集約 | Transit Gateway |
| VPC AからVPC B経由でVPC Cへ行きたい | VPC Peeringでは不可。TGWなどを検討 |
| CIDRが重複している | VPC Peeringでは接続不可 |

## VPC PeeringとTGWの使い分け

| 観点 | VPC Peering | Transit Gateway |
| :--- | :--- | :--- |
| 規模 | 少数VPC | 多数VPC |
| 構成 | point-to-point | hub and spoke |
| 推移的ルーティング | 不可 | 可能な設計にできる |
| ルート管理 | VPCごとに個別管理 | TGW route tableで集約管理 |
| コスト | 小規模なら低コストになりやすい | attachment/処理量課金を考慮 |
| 検査VPC | 設計しづらい | Inspection VPCを組み込みやすい |

覚え方:

```text
少数・単純 = VPC Peering
多数・集約・オンプレ連携 = Transit Gateway
```

## PrivateLink

AWS PrivateLinkは、VPCからサービスやリソースへprivate IPで接続するための仕組みである。

```text
Consumer VPC
  -> Interface Endpoint
  -> Endpoint Service
  -> NLB / GWLB / Service
  -> Provider VPC
```

重要:

```text
PrivateLinkはVPC全体をつなぐ技術ではない。
サービス単位でprivate公開する技術。
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| Consumer | サービスを使う側 |
| Provider | サービスを提供する側 |
| Interface Endpoint | consumer VPCに作るENI型endpoint |
| Endpoint Service | providerが公開するサービス |
| NLB | endpoint serviceの代表的な背後LB |
| DNS | private DNS名でendpointへ解決させることが多い |
| アクセス方向 | 基本はconsumerからproviderのサービスへ |
| CIDR重複 | VPC Peering/TGWより許容しやすい設計にできる |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 他アカウントのサービスだけprivate公開したい | PrivateLink |
| VPC全体の双方向通信が必要 | TGWまたはVPC Peering |
| CIDRが重複しているが特定サービスだけ接続したい | PrivateLink |
| SaaS/Marketplace serviceへprivate接続 | PrivateLink |
| 検査アプライアンスへprivate転送 | GWLB Endpoint |

## VPC Endpoint

VPC Endpointは、VPCからAWSサービスやPrivateLink対応サービスへprivate接続する入口である。

| Endpoint種類 | 主な用途 | Route table設定 | 代表例 |
| :--- | :--- | :--- | :--- |
| Gateway Endpoint | S3/DynamoDB | 必要 | S3、DynamoDB |
| Interface Endpoint | AWS API/PrivateLink service | 通常はDNS利用 | EC2 API、CloudWatch、STS、KMSなど |
| Gateway Load Balancer Endpoint | Security appliance/GWLB | 必要 | firewall、IDS/IPS、DPI |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| private subnetからS3へNATなしで接続 | Gateway Endpoint for S3 |
| private subnetからAWS APIへprivate接続 | Interface Endpoint |
| 特定AWS APIだけendpoint policyで制御 | Interface Endpoint + endpoint policy |
| firewall applianceへ透過的に流したい | GWLB Endpoint |

注意:

```text
Gateway EndpointはS3/DynamoDB向け。
Interface EndpointはENIとprivate IPを持つ。
GWLB Endpointはroute tableのtargetにして検査経路へ使う。
ANS-C01ではまずGateway/Interface/GWLBEを優先して覚える。
```

## AWS RAMとクロスアカウント接続

AWS RAM（Resource Access Manager）は、AWSリソースを複数アカウントへ共有するサービスである。

ネットワークで出やすい共有対象:

```text
Transit Gateway
Subnet
Route 53 Resolver rule
IPAM pool
```

TGWのクロスアカウント利用:

```text
Network account
  -> TGWを作成
  -> AWS RAMで開発/本番/運用アカウントへ共有
  -> 各アカウントのVPCをTGW attachment
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 複数アカウントで中央TGWを使いたい | AWS RAMでTGW共有 |
| Organizations配下へ一貫した共有 | AWS RAM + Organizations |
| アカウントごとにTGWを重複作成したくない | AWS RAM |

## Client VPN

AWS Client VPNは、ユーザー端末からAWSやオンプレミスリソースへ接続するマネージドVPNである。

```text
User PC
  -> OpenVPN-based client
  -> Client VPN endpoint
  -> VPC / peered VPC / TGW / on-prem
```

Site-to-Site VPNとの違い:

| 観点 | Client VPN | Site-to-Site VPN |
| :--- | :--- | :--- |
| 接続元 | ユーザー端末 | 拠点ルーター |
| 目的 | リモートアクセス | 拠点間接続 |
| 暗号化 | TLS VPN | IPsec VPN |
| 認証 | AD、SAML、証明書など | PSK/証明書、IKE/IPsec |
| 試験キーワード | remote user、client-based、OpenVPN | branch office、data center、IPsec |

試験での見方:

```text
社員PCからVPCへ安全に接続 = Client VPN
オンプレ拠点からAWSへ接続 = Site-to-Site VPN
```

## Transit Gateway Network Manager

Transit Gateway Network Managerは、TGWを含むグローバルネットワークの可視化や運用確認で出る。

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| TGW中心のネットワークトポロジーを可視化 | Transit Gateway Network Manager |
| 複数リージョン/複数拠点の接続状態を把握 | Network Manager |
| TGW Route Tableの経路を確認 | Route Analyzer |
| VPC内のSG/NACL/Route込みで到達性を確認 | Reachability Analyzer |
| 実際の通信ログを確認 | Flow Logs |

### Route Analyzerとの違い

Route Analyzerは、TGW Route Table上の経路確認に向く。

```text
TGW Attachment
  -> TGW Route Table
  -> どのAttachmentへ出るか
```

ただし、VPC内Subnet Route Table、Security Group、Network ACL、OS Firewall、DNS応答までは判定しない。

## TGW Appliance Mode

Appliance Modeは、Inspection VPCのようなセキュリティ検査VPCを通すときに重要である。

```text
Spoke VPC A
  -> TGW
  -> Inspection VPC
  -> Firewall
  -> TGW
  -> Spoke VPC B
```

ステートフルFirewall/IDS/IPSでは、行きと戻りが同じ検査経路を通る必要がある。

```text
行き: A -> Firewall -> B
戻り: B -> 同じFirewall -> A
```

Appliance Modeを使う目的:

```text
往復通信を同じAZ/同じ検査経路へ寄せる
```

試験キーワード:

| キーワード | 疑う答え |
| :--- | :--- |
| Inspection VPC | Appliance Mode |
| Stateful inspection | Appliance Mode |
| IDS/IPS | Appliance Mode |
| GWLB/GWLBE | Appliance Mode |
| AZをまたぐ通信が断続的に失敗 | Appliance Mode |
| 非対称ルーティング | Appliance Modeまたはルート見直し |

## GWLB / GWLBEとTGW

GWLB（Gateway Load Balancer）は、Firewallなどのアプライアンス群へ通信を分散するロードバランサーである。

GWLBE（Gateway Load Balancer Endpoint）は、VPC内からGWLBへ向かう入口であり、ルートテーブルのターゲットにできる。

```text
Spoke VPC
  -> TGW
  -> Inspection VPC
  -> GWLBE
  -> GWLB
  -> Firewall appliance
  -> GWLB
  -> GWLBE
  -> TGW
```

覚えること:

| 用語 | 役割 |
| :--- | :--- |
| Inspection VPC | Firewall/GWLBを置くVPC |
| GWLBE | ルートテーブルのターゲット。GWLBへの入口 |
| GWLB | Firewallアプライアンスへフローを分散 |
| Geneve | GWLBで使うカプセル化。UDP `6081` |
| Appliance Mode | TGW側で往復通信を同じ検査経路へ寄せる |

## VPN接続とは

AWS Site-to-Site VPNは、オンプレミスとAWSをIPsecトンネルで接続するサービスである。

```text
オンプレミスルーター
  -> Internet
  -> IPsec VPN tunnel
  -> VGWまたはTGW
  -> VPC
```

## VPNの構成要素

| 用語 | 意味 |
| :--- | :--- |
| Customer Gateway（CGW） | オンプレ側ルーターを表すAWSリソース |
| Customer Gateway device | 実際のオンプレVPNルーター |
| Virtual Private Gateway（VGW） | VPC側のVPN/DX終端 |
| Transit Gateway（TGW） | 複数VPC向けのVPN終端 |
| VPN connection | AWSとオンプレ間のVPN接続 |
| VPN tunnel | VPN connection内の個別IPsecトンネル |
| IKE | IPsecの鍵交換 |
| IPsec | 暗号化通信 |
| BGP | 動的ルーティング |
| Static route | 静的ルーティング |

## VPNで覚える数字

| 数字 | 意味 |
| :--- | :--- |
| 2 | 1つのSite-to-Site VPN接続には2本のトンネル |
| UDP `500` | IKE |
| UDP `4500` | NAT-T |
| IP protocol `50` | ESP |
| TCP `179` | BGP |
| MTU `1446` | Site-to-Site VPNのMTU目安 |
| 最大目安 `1.25 Gbps` | 標準VPNトンネルの帯域目安 |

## VPN: VGW接続とTGW接続

| 観点 | VPN to VGW | VPN to TGW |
| :--- | :--- | :--- |
| 接続先 | 1つのVPC中心 | 複数VPC中心 |
| ルーティング | VPC route table + VGW | TGW route table |
| 拡張性 | 小さめ構成向き | 大規模/複数VPC向き |
| ECMP | Site-to-Site VPNでは基本使わない | ECMP利用可 |
| 設計キーワード | 単一VPC、シンプル | Hub and spoke、集約 |

## VPNの静的ルーティングとBGP

| 種類 | 内容 | 試験での見方 |
| :--- | :--- | :--- |
| Static routing | 経路を手動設定 | BGP非対応のCustomer Gateway |
| Dynamic routing | BGPで経路交換 | 推奨されやすい。フェイルオーバーに強い |

AWS公式では、Customer Gateway deviceがBGPに対応しているなら動的ルーティングを選ぶ、という考え方でよい。

```text
BGP対応あり -> Dynamic routing
BGP対応なし -> Static routing
```

## VPNの2トンネル

Site-to-Site VPN接続は、冗長化のために2本のトンネルを持つ。

```text
VPN connection
  ├─ Tunnel 1
  └─ Tunnel 2
```

実務・試験の基本:

```text
両方のトンネルを設定する
片方だけに依存しない
非対称ルーティングを許容できる設計にする
```

## Accelerated Site-to-Site VPN

Accelerated VPNは、AWS Global Acceleratorを使ってVPNトラフィックを近くのAWSエッジからAWSグローバルネットワークへ入れる仕組みである。

```text
オンプレ
  -> Internet
  -> 最寄りのAWS edge
  -> AWS global network
  -> TGW
```

覚えること:

| 項目 | 内容 |
| :--- | :--- |
| 目的 | 遠距離インターネット経路の遅延/揺れを改善 |
| 背後の仕組み | AWS Global Accelerator |
| 接続先 | Transit Gateway |
| 注意 | 既存VPNを後から高速化ONではなく、新しいAccelerated VPNを作って切替 |

## DX / VPNの経路優先

ハイブリッド接続では、同じ宛先CIDRが複数経路から見えることがある。

基本原則:

```text
1. 最長一致が優先
2. static routeはpropagated routeより優先される場面が多い
3. VPN/DXではBGP属性で優先経路を調整する
4. tunnelやconnectionのhealthは経路属性より優先される
```

Site-to-Site VPN/VGWで同一prefixが競合する場合の代表的な優先順:

```text
Direct Connect BGP propagated route
  > Site-to-Site VPN static route
  > Site-to-Site VPN BGP propagated route
```

Direct ConnectのBGP経路制御で見る順序:

```text
prefix length
  -> local preference community
  -> AS_PATH length
  -> MED
```

試験での見方:

| 問題文 | 疑う答え |
| :--- | :--- |
| DXを主系、VPNをbackupにしたい | DX + VPN、BGP経路優先を調整 |
| 特定prefixだけ別回線へ流したい | より具体的なprefixを広告 |
| inbound to on-premの優先を変えたい | Direct Connect BGP community / local preference |
| 2本のDXでActive/Active | 同一prefix長、同等BGP属性、ECMP |
| VPNで複数トンネルを使いたい | TGW + ECMP。VGWではECMP不可 |

注意:

```text
経路制御は「AWSからオンプレへ」と「オンプレからAWSへ」で効く属性が違う。
問題文がどちら向きのtrafficを制御したいのかを読む。
```

## TGW + VPN + DXの使い分け

| 要件 | 選択肢 |
| :--- | :--- |
| 低コストですぐ接続したい | Site-to-Site VPN |
| 安定帯域、低遅延、閉域寄り | Direct Connect |
| DX障害時のバックアップ | Direct Connect + Site-to-Site VPN |
| 複数VPCへ集約接続 | TGW |
| DXからTGWへ接続 | Transit VIF + DXGW |
| VPNからTGWへ接続 | VPN attachment |
| グローバル拠点からVPN品質を改善 | Accelerated VPN |
| 少数VPC間を簡単に接続 | VPC Peering |
| 他VPC/他アカウントのサービスだけprivate公開 | PrivateLink |
| S3/DynamoDBへprivate接続 | Gateway VPC Endpoint |
| AWS APIへprivate接続 | Interface VPC Endpoint |
| リモートユーザーをAWSへ接続 | Client VPN |
| 複数アカウントで中央TGWを共有 | AWS RAM |
| 一対多配信、multicast | TGW Multicast |

## 試験での見抜き方

| 問題文の表現 | 疑う答え |
| :--- | :--- |
| Direct ConnectでS3に接続 | Public VIF |
| Direct ConnectでVPCに接続 | Private VIF |
| Direct ConnectでTGW配下の複数VPCへ接続 | Transit VIF + DXGW |
| 1つのVPCへVPN | VPN to VGW |
| 複数VPCへVPN | VPN to TGW |
| BGP対応ルーター | Dynamic routing |
| BGP非対応ルーター | Static routing |
| VPNの高可用性 | 2トンネル両方を設定 |
| VPNでActive/Active | TGW + ECMP |
| VPN経路品質を改善 | Accelerated VPN |
| 複数VPC間通信を集中検査 | TGW + Inspection VPC |
| GWLB配下Firewallで検査 | GWLBE + GWLB + Appliance Mode |
| 往復通信が別Firewallへ行き失敗 | Appliance Mode |
| TGWで通信できない | Association/Propagation/Route Tableを確認 |
| TGWへ入った通信の出口が不明 | Ingress attachmentのAssociationを見る |
| 2 VPCのみの単純接続 | VPC Peering |
| VPC PeeringでA-B-C接続したい | 推移的ルーティング不可。TGWを検討 |
| CIDR重複だが特定サービスだけ接続 | PrivateLink |
| private subnetからS3へNATなし | Gateway Endpoint |
| private subnetからCloudWatch/STS/KMS等へNATなし | Interface Endpoint |
| 複数アカウントでTGWを共用 | AWS RAM |
| ユーザー端末からAWSへVPN | Client VPN |
| オンプレ拠点間をAWS backboneで接続 | Direct Connect SiteLink |
| TGW内の経路を分析したい | Route Analyzer |
| 一対多配信、multicast domain | TGW Multicast |

## 最小暗記セット

### VIF

```text
Public VIF  = S3などAWS public service
Private VIF = VPC/VGW
Transit VIF = TGW/DXGW
```

### TGW

```text
Attachment  = TGWへの接続点
Association = 入ってきた通信が見るRoute Table
Propagation = 経路をRoute Tableへ載せる
Appliance Mode = ステートフル検査で往復を同じ経路へ寄せる
Route Analyzer = TGW Route Tableの経路確認
TGW Multicast = 一対多配信、Multicast Domain/Group/Source/Member
```

### VPN

```text
Site-to-Site VPN = IPsec
1接続 = 2トンネル
BGP対応あり = Dynamic routing
BGP非対応 = Static routing
複数VPC = TGW
単一VPC = VGW
```

### VPC間/AWS環境間

```text
VPC Peering = 2 VPCを直接接続、推移的ルーティング不可
TGW = 多数VPC/オンプレをhub and spokeで集約
PrivateLink = VPC全体ではなくサービス単位でprivate公開
Gateway Endpoint = S3/DynamoDB
Interface Endpoint = AWS API/PrivateLink service
GWLBE = security applianceへroute
AWS RAM = TGW/subnet/Resolver ruleなどをクロスアカウント共有
Client VPN = ユーザー端末からTLS VPN
```

## 公式参照

- [ANS-C01 Content Domain 3: Network Management and Operation](https://docs.aws.amazon.com/aws-certification/latest/advanced-networking-specialty-01/advanced-networking-specialty-01-domain3.html)
- [ANS-C01 In-Scope AWS Services](https://docs.aws.amazon.com/aws-certification/latest/advanced-networking-specialty-01/ans-01-in-scope-services.html)
- [AWS Direct Connect virtual interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)
- [Create a transit virtual interface to the Direct Connect gateway](https://docs.aws.amazon.com/directconnect/latest/UserGuide/create-transit-vif-for-gateway.html)
- [Direct Connect gateways and Transit Gateway associations](https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-transit-gateways.html)
- [Allowed prefixes interactions for Direct Connect gateways](https://docs.aws.amazon.com/directconnect/latest/UserGuide/allowed-to-prefixes.html)
- [Direct Connect routing policies and BGP communities](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)
- [Direct Connect SiteLink](https://docs.aws.amazon.com/directconnect/latest/UserGuide/dx-sitelink.html)
- [Transit Gateway VPC attachments and appliance mode](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html)
- [How AWS Transit Gateway works](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)
- [Multicast in AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-multicast-overview.html)
- [Route Analyzer for AWS Network Manager](https://docs.aws.amazon.com/network-manager/latest/tgwnm/route-analyzer.html)
- [Connect attachments and Connect peers in AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-connect.html)
- [What is VPC peering?](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- [What is AWS PrivateLink?](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html)
- [AWS PrivateLink concepts](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)
- [What is AWS Resource Access Manager?](https://docs.aws.amazon.com/ram/latest/userguide/what-is.html)
- [What is AWS Client VPN?](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/what-is.html)
- [Static and dynamic routing in AWS Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-static-dynamic.html)
- [Site-to-Site VPN route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)
