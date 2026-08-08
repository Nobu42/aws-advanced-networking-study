# ANS-C01 頻出・必須用語集

## 目的

AWS Certified Advanced Networking - Specialty（ANS-C01）で頻出する用語を、試験で判断できる形に整理する。

単語の暗記ではなく、以下を説明できることを目標にする。

- 何をするサービス・機能か
- どの要件で選ぶか
- どの選択肢と迷いやすいか
- 障害時にどこを確認するか

## 試験注意

AWS公式ページでは、AWS Certified Advanced Networking - Specialty は **2026年8月25日が最終受験日** と案内されている。受験予定日、試験名、試験ガイドは必ず最新の公式ページで確認する。

公式資料:

- [AWS Certified Advanced Networking - Specialty](https://aws.amazon.com/certification/certified-advanced-networking-specialty/)
- [ANS-C01 Exam Guide](https://d1.awsstatic.com/training-and-certification/docs-advnetworking-spec/AWS-Certified-Advanced-Networking-Specialty_Exam-Guide.pdf)

## 試験ドメイン

| Domain | 内容 | 比率 |
| :--- | :--- | ---: |
| 1 | Network Design | 30% |
| 2 | Network Implementation | 26% |
| 3 | Network Management and Operation | 20% |
| 4 | Network Security, Compliance, and Governance | 24% |

## 1. VPC基礎

### Amazon VPC

Amazon VPCは、AWS上に作成する論理的に分離されたネットワーク空間。

試験では、VPCそのものよりも、Subnet、Route Table、Security Group、Network ACL、Endpoint、TGW、VPN、Direct Connectとの関係で問われる。

重要ポイント:

- VPCはリージョン単位のリソース
- Subnetは1つのAZに属する
- VPC内のCIDRは後続の接続設計に強く影響する
- CIDR重複はVPC Peering、TGW、VPN、DX設計で大きな問題になる

ひっかけ:

- VPCを作るだけではインターネット接続できない
- Public SubnetかどうかはSubnet名ではなく、Route Tableで判断する

### CIDR

CIDRはIPアドレス範囲を表す記法。例: `10.0.0.0/16`

試験では、アドレス設計、CIDR重複、オンプレミス接続、マルチVPC接続で頻出する。

重要ポイント:

- `/16` は広い、`/24` は狭い
- AWSでは各Subnetの先頭4個と最後1個のIPアドレスは予約される
- オンプレミス、他VPC、他リージョンと重複しない設計が重要

ひっかけ:

- CIDRが重複しているVPC同士は、通常のPeeringやTGWでは素直に接続できない
- IP重複がある場合はPrivateLinkやNATによる回避が選択肢になる

### Subnet

SubnetはVPC CIDRから切り出したIP範囲で、必ず1つのAZに属する。

重要ポイント:

- 高可用性のためには複数AZにSubnetを作る
- Public SubnetはInternet Gatewayへのデフォルトルートを持つSubnet
- Private SubnetはInternet Gatewayへ直接出ないSubnet

ひっかけ:

- Public IPがあるだけではPublic Subnetとは言わない
- SubnetのPublic/Privateは、関連付いたRoute Tableで判断する

### Route Table

Route Tableは、Subnetから出る通信の宛先を決める。

重要ポイント:

- Route TableはSubnetに関連付ける
- 明示的な関連付けがないSubnetはMain Route Tableを使う
- `local` routeはVPC内通信に使われ、自動的に存在する
- `0.0.0.0/0 -> igw-*` はインターネット向け
- `0.0.0.0/0 -> nat-*` はPrivate Subnetからの外向き通信向け
- `10.0.0.0/8 -> tgw-*` のような経路はTGW向け

ひっかけ:

- Security Groupが許可していてもRouteがなければ通信できない
- 片方向のRouteだけでは通信できない。戻り経路も必要

### Internet Gateway

Internet Gatewayは、VPCとインターネットの間のゲートウェイ。

重要ポイント:

- VPCにアタッチして使う
- Public SubnetのRoute Tableに `0.0.0.0/0 -> igw-*` を設定する
- EC2がインターネットから直接到達されるには、Public IPv4またはElastic IPも必要

ひっかけ:

- Internet GatewayだけではPrivate Subnetのインスタンスはインターネットへ出られない
- Private Subnetからの外向き通信には通常NAT Gatewayを使う

### NAT Gateway

NAT Gatewayは、Private Subnetのインスタンスがインターネットへアウトバウンド通信するためのマネージドNAT。

重要ポイント:

- Public NAT GatewayはPublic Subnetに配置する
- Elastic IPを関連付ける
- Private SubnetのRoute Tableで `0.0.0.0/0 -> nat-*` を設定する
- AZ単位のリソースなので、高可用性には各AZにNAT Gatewayを置く

ひっかけ:

- NAT Gatewayはインバウンド公開用ではない
- 別AZのPrivate Subnetから1つのNAT Gatewayへ集約すると、AZ障害やクロスAZ通信コストの論点になる

### NAT Instance

NAT Instanceは、EC2インスタンスでNATを実現する方式。

重要ポイント:

- Source/Destination Checkを無効化する必要がある
- インスタンスタイプ、OS、iptables、パッチ、スケーリングを自分で管理する
- セキュリティグループを設定できる

ひっかけ:

- NAT Gatewayより運用負荷が高い
- 性能不足の場合はインスタンスタイプ変更、帯域確認、NAT Gateway移行が候補になる

### Egress-only Internet Gateway

IPv6用の外向き専用ゲートウェイ。

重要ポイント:

- IPv6にはNAT GatewayのようなNAT変換は基本的に不要
- IPv6でPrivate Subnetから外向き通信だけを許可したい場合に使う

ひっかけ:

- IPv4のNAT Gatewayとは用途が違う
- IPv6の `::/0` routeと一緒に問われることが多い

### Elastic Network Interface

ENIはEC2や一部AWSサービスに接続される仮想ネットワークインターフェイス。

重要ポイント:

- Security GroupはENIに関連付く
- Private IP、Elastic IP、MACアドレス、Source/Destination Checkなどが関係する
- 複数ENIを持つEC2は、管理用/データ用など通信経路を分けられる
- Interface Endpoint、Lambda VPC接続、NAT Instance、Firewall Applianceの理解にも関係する

ひっかけ:

- EC2インスタンスそのものではなく、通信の入口はENIとして考える
- 複数ENIや複数Route Tableを使う構成では、戻り経路とSource/Destination Checkに注意する

### Elastic Network Adapter

ENAはEC2の高性能ネットワークアダプター。

重要ポイント:

- 高スループット、低レイテンシー、拡張ネットワーキングの文脈で出る
- EC2インスタンスタイプが対応している必要がある
- ネットワーク性能不足の切り分けでは、インスタンスタイプ、ENA対応、OSドライバー、PPS/帯域制限を確認する

ひっかけ:

- ENAは通信方式そのものではなく、EC2ネットワーク性能を支えるアダプター
- VPC接続サービスであるTGW、VPN、DXとは役割が違う

### Elastic Fabric Adapter

EFAはHPCや機械学習など、低レイテンシー・高スループットが必要な分散処理向けネットワークインターフェイス。

重要ポイント:

- EC2間の高性能通信に使う
- 対応インスタンス、対応AMI/ドライバー、同一AZ配置などが重要
- HPC、MPI、機械学習クラスターの問題文で出やすい

ひっかけ:

- 一般的なWebシステムの高速化ではなく、ノード間通信が重い計算基盤向け
- CloudFrontやGlobal Acceleratorのようなエッジ最適化サービスではない

### Secondary CIDR

既存VPCに追加できるCIDRブロック。

重要ポイント:

- IPアドレス不足への対応で出る
- 既存VPC CIDRや接続先ネットワークと重複してはいけない
- 追加後、Subnetを新しいCIDRから作成して利用する
- IPAMと組み合わせると、複数アカウント/複数リージョンのアドレス管理がしやすい

ひっかけ:

- CIDRを追加しても既存Subnetのサイズが自動で広がるわけではない
- Peering、TGW、VPN、DX先との重複確認が必要

### DHCP Options Set

VPC内インスタンスが使うDNSサーバーやドメイン名などのDHCP設定。

重要ポイント:

- AmazonProvidedDNS、オンプレミスDNS、独自ドメイン名の設定で関係する
- ハイブリッドDNS設計でRoute 53 Resolverと一緒に問われる

ひっかけ:

- Route 53 Hosted Zoneそのものではない
- 設定変更後、既存インスタンス側の反映タイミングにも注意する

## 2. セキュリティ境界

### Security Group

Security GroupはENI単位に適用されるステートフルな仮想ファイアウォール。

重要ポイント:

- Allowのみ設定できる
- Denyは書けない
- ステートフルなので、戻り通信を明示的に許可しなくてもよい
- EC2、ALB、RDS、Lambda ENI、Interface Endpointなどに関連する

ひっかけ:

- 明示的に拒否したい場合はNetwork ACLやNetwork Firewallを検討する
- Security Group参照は、許可元をCIDRではなく別SGで指定できる

### Network ACL

Network ACLはSubnet単位に適用されるステートレスなアクセス制御。

重要ポイント:

- AllowとDenyを設定できる
- Rule番号の小さい順に評価される
- ステートレスなので、InboundとOutboundの両方を考える
- Ephemeral Portの許可が必要になることがある

ひっかけ:

- Security Groupと違い、戻り通信も明示的に許可が必要
- NACLを変更するとSubnet内の複数リソースへ影響する

### Ephemeral Port

クライアント側が一時的に使う戻り通信用のポート範囲。

重要ポイント:

- NACLで戻り通信を制御するときに重要
- HTTP/HTTPSの通信でも、戻り通信は高い番号の一時ポートを使う

ひっかけ:

- Inboundの80/443だけ許可しても、戻り通信のEphemeral PortをNACLで落とすと通信できない

### AWS Network Firewall

VPC向けのマネージドネットワークファイアウォール。

重要ポイント:

- ステートフル、ステートレス両方のルールを扱える
- 集中検査構成で使われる
- TGW、Inspection VPC、Route Table設計とセットで問われる

ひっかけ:

- ステートフル検査では行きと戻りが同じ検査経路を通る設計が重要
- 非対称ルーティングは通信断の原因になる

### AWS WAF

HTTP/HTTPSのLayer 7でWebリクエストを保護するサービス。

重要ポイント:

- CloudFront、ALB、API Gatewayなどに関連付ける
- SQL Injection、XSS、IP制限、Rate-based Ruleなどで使う

ひっかけ:

- TCP/UDP全般の制御には向かない
- L3/L4のネットワーク制御はSecurity Group、NACL、Network Firewall、Shieldなどと比較する

### AWS Shield

DDoS対策サービス。

重要ポイント:

- Shield Standardは自動適用
- Shield Advancedは高度なDDoS保護、コスト保護、DRT連携などがある
- CloudFront、Route 53、Global Accelerator、ELBなどと関連する

ひっかけ:

- WAFはWebリクエスト制御、ShieldはDDoS対策

## 3. VPC接続パターン

### VPC Peering

2つのVPCを直接接続する機能。

重要ポイント:

- 非推移的
- 双方のRoute Tableに経路が必要
- CIDR重複は不可
- 小規模・単純なVPC間接続に向く

ひっかけ:

- A-B、B-CをPeeringしても、A-Cは通信できない
- 大規模な多VPC接続ではTransit Gatewayが選択肢になる

### AWS Transit Gateway

複数VPC、VPN、Direct Connect接続をハブアンドスポークで集約するサービス。

重要ポイント:

- VPC Attachment、VPN Attachment、Direct Connect Gateway連携で使う
- TGW Route Tableで通信範囲を分離できる
- Associationは、AttachmentがどのTGW Route Tableを使うか
- Propagationは、Attachmentの経路をTGW Route Tableへ広げること

ひっかけ:

- VPC側Route TableとTGW側Route Tableの両方が必要
- TGW Peeringは動的ルート伝播しないため、静的ルートが必要になるケースがある

### TGW Appliance Mode

Transit Gateway経由でステートフルな検査装置を使う場合に重要な設定。

重要ポイント:

- 共有サービスVPCやInspection VPCにIDS/IPS/Firewallを配置する構成で使う
- 同じ通信フローの行きと戻りが同じアプライアンス経路を通るようにする
- AZをまたぐ通信で戻り通信が別アプライアンスへ流れる問題を避ける

覚え方:

```text
TGW + 共有サービスVPC + ステートフル検査 + AZまたぎで不安定
= Appliance Mode
```

ひっかけ:

- Security GroupやNACLの問題に見えて、実は非対称ルーティングの問題であることがある

### AWS PrivateLink

サービス提供者VPCと利用者VPCを、プライベートIPで一方向的に接続する仕組み。

重要ポイント:

- 利用者側はInterface Endpointを作成する
- サービス提供者側はNLBなどを使ってEndpoint Serviceを公開する
- VPC間でCIDRが重複していても使える場合がある
- サービス単位の公開に向く

ひっかけ:

- VPC全体を接続するサービスではない
- 双方向の自由な通信が必要ならPeeringやTGWと比較する

### VPC Sharing

AWS Organizations内で、1つのVPC/Subnetを複数アカウントに共有する機能。

重要ポイント:

- ネットワークアカウントがVPCを管理し、アプリケーションアカウントがSubnetを利用する
- AWS RAMを使う
- IPアドレス管理とガバナンスに有効

ひっかけ:

- VPC PeeringやTGWのようにVPC同士を接続する機能ではない
- アカウント間のネットワークを増やすより、中央のネットワークアカウントでSubnetを共用したい要件に向く

### AWS RAM

AWS Resource Access Manager。AWSリソースを複数アカウントに共有するサービス。

重要ポイント:

- TGW、VPC Subnet、Route 53 Resolver Rule、Direct Connect Gateway関連などで登場する
- マルチアカウントネットワーク設計で頻出

ひっかけ:

- IAMポリシーだけでは共有できないリソースがある

## 4. VPC Endpoint

### Gateway Endpoint

S3やDynamoDBへVPC内からプライベートにアクセスするためのEndpoint。

重要ポイント:

- Route TableにPrefix List宛ての経路が追加される
- S3アクセスで非常に頻出
- NAT Gatewayを経由しないため、コストやセキュリティ面で有利な場合がある

ひっかけ:

- Security Groupは関連付けない
- 対応サービスは限定される

### Interface Endpoint

AWS PrivateLinkを利用するENI型のEndpoint。

重要ポイント:

- Subnet内にENIが作られる
- Security Groupを関連付ける
- Private DNSを有効化すると、通常のサービス名をプライベートIPへ解決できる
- 多くのAWSサービスに対応する

ひっかけ:

- DNS設定が誤ると、Endpointを作ったのにNAT経由のままになることがある
- AZごとにEndpointを置く設計が可用性・コスト面で重要

### Prefix List

AWS管理またはカスタマー管理のCIDR集合。

重要ポイント:

- S3 Gateway EndpointのRouteで使われる
- Security GroupやRoute Tableで宛先管理を簡素化できる

ひっかけ:

- AWS管理Prefix Listとカスタマー管理Prefix Listを混同しない

## 5. DNS

### Amazon Route 53

AWSのDNSサービス。

重要ポイント:

- Public Hosted Zoneはインターネット向け
- Private Hosted ZoneはVPC内向け
- Health CheckやRouting Policyでグローバルな名前解決を制御する

ひっかけ:

- Route 53のDNSルーティングは、通信経路そのものを転送するわけではない
- DNSのTTLにより切り替え反映に時間がかかる場合がある

### Hosted Zone

DNSレコードを管理する単位。

重要ポイント:

- Public Hosted ZoneはパブリックDNS名を管理する
- Private Hosted Zoneは関連付けたVPCからのみ解決できる

ひっかけ:

- Private Hosted ZoneはVPC関連付けが必要
- 同じドメイン名のPublic/Private Hosted Zoneがある場合、解決経路を整理する

### Alias Record

AWSリソース向けに使えるRoute 53独自のレコード。

重要ポイント:

- ALB、NLB、CloudFront、S3 Web Hostingなどに使う
- Zone Apexでも使える

ひっかけ:

- CNAMEはZone Apexに使えないが、Aliasは使える

### Route 53 Resolver

VPC内のDNS解決を担う仕組み。

重要ポイント:

- Inbound EndpointはオンプレミスからAWSのPrivate DNSを解決するために使う
- Outbound EndpointはVPCからオンプレミスDNSへ問い合わせるために使う
- Resolver Ruleで条件付き転送を設定する

ひっかけ:

- InboundとOutboundの向きを混同しやすい
- セキュリティグループでTCP/UDP 53を考慮する

### DNSSEC

DNS応答の正当性を検証するための仕組み。

重要ポイント:

- DNSの改ざん対策として問われる
- Route 53のPublic Hosted Zoneやドメイン登録と関連して出る

ひっかけ:

- DNSSECは通信の暗号化ではない
- 通信暗号化はTLS、IPsec、VPNなどと比較する

## 6. Load Balancing と Edge

### Application Load Balancer

Layer 7のロードバランサー。

重要ポイント:

- HTTP/HTTPS向け
- パスベース、ホストベースルーティングができる
- WAF連携、TLS終端、OIDC認証などと関連する

ひっかけ:

- TCP/UDPの低レイヤ要件ならNLBを検討する

### Network Load Balancer

Layer 4のロードバランサー。

重要ポイント:

- TCP、UDP、TLSなどに対応
- 高性能・低遅延
- 固定IP、PrivateLinkのEndpoint Serviceで使う

ひっかけ:

- HTTPヘッダーやパスベースルーティングはALBの領域

### Gateway Load Balancer

Firewall、IDS/IPSなどの仮想アプライアンスを透過的に挟むためのロードバランサー。

重要ポイント:

- GENEVEプロトコルを使う
- セキュリティアプライアンスのスケールアウトに使う
- 集中検査構成で問われる

ひっかけ:

- Webアプリの通常ロードバランシングにはALB/NLBを使う

### CloudFront

AWSのCDNサービス。

重要ポイント:

- エッジロケーションでコンテンツをキャッシュする
- S3、ALB、API Gatewayなどをオリジンにできる
- WAF、Shield、ACM、Origin Access Controlなどと関連する

ひっかけ:

- 動的アプリのグローバルTCP/UDP最適化ならGlobal Acceleratorと比較する

### AWS Global Accelerator

固定Anycast IPを提供し、AWSグローバルネットワークで最適なリージョンやエンドポイントへ誘導するサービス。

重要ポイント:

- TCP/UDPに対応
- ALB、NLB、EC2 Elastic IPなどをEndpointにできる
- ヘルスチェックに基づくリージョン切り替えができる

ひっかけ:

- CloudFrontはCDN、Global Acceleratorはグローバル入口・ネットワーク経路最適化
- DNSベースのRoute 53フェイルオーバーとは切り替え方式が違う

### Amazon API Gateway

APIの公開、認証、スロットリング、ルーティングを提供するマネージドサービス。

重要ポイント:

- REST API、HTTP API、WebSocket APIの文脈で出る
- CloudFront、WAF、ACM、Route 53と組み合わせることがある
- Private APIはVPC内からInterface Endpoint経由でアクセスさせる構成が重要

ひっかけ:

- API Gatewayはロードバランサーではない
- VPC内のプライベートなALB/NLB/サービスへ接続する場合はVPC Linkの要件を確認する

### API Gateway VPC Link

API GatewayからVPC内のプライベートなバックエンドへ接続するための仕組み。

重要ポイント:

- API Gatewayを入口にし、VPC内のNLB/ALBやプライベートサービスへ流す構成で使う
- 「APIは公開したいが、バックエンドはPrivate Subnetに置きたい」という問題で出やすい
- Security Group、NACL、バックエンドのリスナー、Route Tableを合わせて確認する

ひっかけ:

- Private APIとVPC Linkは逆向きに混同しやすい
- Private APIは「クライアントがVPCからAPI Gatewayへ入る」、VPC Linkは「API GatewayがVPC内バックエンドへ入る」

## 7. Hybrid Connectivity

### AWS Site-to-Site VPN

オンプレミスとAWSをIPsec VPNで接続するサービス。

重要ポイント:

- 2本のトンネルが作られる
- 静的ルートまたはBGPによる動的ルーティングを使う
- Direct Connectのバックアップ経路としても使う

ひっかけ:

- インターネット上の暗号化トンネルなので、帯域や遅延はインターネット品質に影響される

### Customer Gateway

オンプレミス側のVPN装置またはその定義。

重要ポイント:

- Public IP、BGP ASNなどを設定する
- Site-to-Site VPNのオンプレミス側を表す

ひっかけ:

- AWS側のゲートウェイであるVirtual Private GatewayやTransit Gatewayと混同しない

### Virtual Private Gateway

VPCにアタッチするVPN/DX接続用のゲートウェイ。

重要ポイント:

- 単一VPCとの接続に使う
- 大規模な複数VPC接続ではTGWが候補になる

ひっかけ:

- TGWとは役割が違う

### AWS Direct Connect

オンプレミスとAWSを専用線で接続するサービス。

重要ポイント:

- 安定した帯域、低遅延、一貫したネットワーク品質を得やすい
- 暗号化は標準では提供されない
- 暗号化が必要ならVPN over Direct ConnectやMACsecを検討する

ひっかけ:

- Direct Connectは「専用接続」だが「自動的に暗号化」ではない

### Direct Connect Gateway

Direct Connectを複数リージョンや複数VPC/TGWへ接続するためのゲートウェイ。

重要ポイント:

- Private VIFやTransit VIFと関連する
- マルチリージョン接続設計で出る

ひっかけ:

- VPCに直接アタッチするものではない

### Virtual Interface

Direct Connect上の論理インターフェイス。

重要ポイント:

- Public VIFはAWSパブリックサービス向け
- Private VIFはVPC/VGW向け
- Transit VIFはTransit Gateway向け

ひっかけ:

- S3のPublic EndpointへDX経由で行く要件ではPublic VIFが候補になる
- TGWへ接続するならTransit VIFを考える

### BGP

ネットワーク間で経路情報を交換する動的ルーティングプロトコル。

重要ポイント:

- VPN、Direct Connect、Transit Gateway Connectで頻出
- AS_PATH、Local Preference、MED、Communityなどで経路制御する
- Active/Passive、Load Sharing、Failover要件で問われる

ひっかけ:

- BGPは通信を暗号化しない
- 静的ルートより柔軟だが、経路制御の理解が必要

### ASN

Autonomous System Number。BGPで組織やネットワークを識別する番号。

重要ポイント:

- AWS側ASNとオンプレミス側ASNを設計する
- TGWやCustomer Gatewayで出る

ひっかけ:

- ASN重複や設計ミスはBGP接続設計の問題になる

### BGP Community

BGP経路に付与する属性情報。

重要ポイント:

- Direct Connectなどで経路制御に使う
- 経路の優先度や広報範囲の制御に関係する

ひっかけ:

- 単なるタグではなく、ルーティング動作に影響する情報として扱う

### Accelerated Site-to-Site VPN

Global Acceleratorを利用してVPN接続の性能を改善するSite-to-Site VPN。

重要ポイント:

- グローバルネットワークにより、遠距離接続の安定性改善が期待できる
- VPNの可用性・性能要件で問われる

ひっかけ:

- Direct Connectの代替ではなく、VPNの改善オプションとして理解する

### AWS Client VPN

利用者端末からAWSまたはオンプレミスへ安全に接続するためのマネージドリモートアクセスVPN。

重要ポイント:

- Site-to-Site VPNは拠点間、Client VPNはユーザー端末からのリモートアクセス
- 認証、認可ルール、クライアントCIDR、関連付けSubnet、ルートが重要
- 在宅勤務、運用担当者の管理アクセス、開発者アクセスの問題で出やすい

ひっかけ:

- オンプレミス拠点ルーターとAWSを結ぶ用途ならSite-to-Site VPN
- Client VPNを作るだけでは到達できず、Authorization RuleとRouteが必要

### Direct Connect SiteLink

Direct Connectロケーション間でオンプレミス拠点同士を接続できる機能。

重要ポイント:

- AWSリージョン内VPCを経由せず、DXロケーション間の拠点間通信に使う
- グローバルWANの一部としてAWSバックボーンを使いたい要件で出る

ひっかけ:

- VPC接続のためのPrivate VIF/Transit VIFとは目的が違う
- すべてのDX構成で自動的に有効になるわけではない

### Transit Gateway Multicast

Transit GatewayでVPC内のマルチキャスト通信を扱う機能。

重要ポイント:

- Multicast Domain、Group、Source、Memberを理解する
- 金融配信、映像配信、HPCなど一対多配信の問題文で出ることがある
- 通常のユニキャスト経路とは別物として考える

ひっかけ:

- TGWを使えばすべてのAttachmentへマルチキャストできる、とは考えない
- Direct Connect、Site-to-Site VPN、TGW Peeringなどでのサポート可否を確認する

## 8. 監視・調査

### VPC Flow Logs

VPC、Subnet、ENIのIP通信ログ。

重要ポイント:

- ACCEPT/REJECTを確認できる
- CloudWatch Logs、S3、Kinesis Data Firehoseへ出力できる
- Security GroupやNACLの切り分けに使う

ひっかけ:

- パケットの中身は見えない
- アプリケーションログではない

### Traffic Mirroring

ENIの通信をミラーリングして解析装置へ送る機能。

重要ポイント:

- IDS/IPS、パケット解析、詳細調査で使う
- セッションやパケット内容の調査が必要な場合に候補になる

ひっかけ:

- Flow Logsより詳細だが、設計・コスト・対象ENIに注意する

### Reachability Analyzer

VPC内の到達可能性を静的に分析するツール。

重要ポイント:

- Route Table、Security Group、NACLなどをもとに到達性を確認する
- 実トラフィックを流さずに経路分析できる

ひっかけ:

- アプリケーションの正常性やDNSの実応答までは保証しない

### Transit Gateway Network Manager

TGWを中心としたグローバルネットワークの可視化・管理サービス。

重要ポイント:

- TGW、オンプレミス、SD-WANなどを含むネットワーク可視化で問われる
- 監視・運用ドメインで出やすい

ひっかけ:

- 経路を自動修復する魔法の機能ではなく、可視化・管理の文脈で理解する

### Route Analyzer

Transit Gateway Route Tableの経路を分析し、TGW内で宛先へ届くかを確認する機能。

重要ポイント:

- TGW Route TableのAssociation、Propagation、Static Routeの確認に向く
- どのAttachmentを通るか、TGW内で経路が存在するかを見る

ひっかけ:

- VPC内のSecurity Group、NACL、Subnet Route Table、OS Firewallまでは検証しない
- VPCレベルの到達性はReachability Analyzerと使い分ける

### AWS Trusted Advisor

AWS環境のコスト、セキュリティ、耐障害性、パフォーマンス、サービスクォータなどをチェックするサービス。

重要ポイント:

- Direct ConnectやVPNなどの冗長化、サービス制限、公開リソースの検出などで候補になる
- 運用・継続改善・アカウント全体の健全性確認で使う

ひっかけ:

- 個別パケットの通信ログや経路ログを出すサービスではない
- 詳細な通信切り分けはFlow Logs、Reachability Analyzer、CloudWatchメトリクスなどを使う

### AWS Well-Architected Tool

Well-Architected Frameworkに沿ってアーキテクチャをレビューするためのツール。

重要ポイント:

- 信頼性、セキュリティ、パフォーマンス効率、コスト最適化、運用上の優秀性などを確認する
- ネットワーク設計では冗長化、帯域、レイテンシー、障害分離、監視の観点で出る

ひっかけ:

- 自動でネットワーク設定を変更するサービスではない
- 設計レビューや改善計画の文脈で選ぶ

### CloudWatch

メトリクス、ログ、アラーム、ダッシュボードのサービス。

重要ポイント:

- NAT Gateway、VPN、DX、ELBなどのメトリクス監視に使う
- Logs InsightsでFlow Logs分析もできる

ひっかけ:

- CloudTrailはAPI操作履歴、CloudWatchは監視・ログ・メトリクス

### CloudTrail

AWS API操作履歴を記録するサービス。

重要ポイント:

- Route Table変更、Security Group変更、TGW変更などの操作履歴を追う
- 誰が、いつ、何を変更したかの調査に使う

ひっかけ:

- 通信そのもののACCEPT/REJECTはFlow Logsの領域

## 9. 暗号化・証明書

### IPsec

IP層で通信を暗号化する技術。

重要ポイント:

- Site-to-Site VPNで使われる
- Direct Connect上で暗号化が必要な場合にVPN over Direct Connectとして登場する

ひっかけ:

- Direct Connect単体は暗号化ではない

### TLS

アプリケーション通信を暗号化する仕組み。

重要ポイント:

- ALB、NLB、CloudFront、API Gateway、アプリケーションサーバーで使う
- TLS終端、TLSパススルー、mTLSなどが問われる

ひっかけ:

- TLS終端の場所により、どこまで暗号化されるかが変わる

### AWS Certificate Manager

TLS証明書を管理するサービス。

重要ポイント:

- ALB、CloudFront、API Gatewayなどと連携する
- CloudFrontでは証明書リージョンの扱いに注意する

ひっかけ:

- Private CAが必要な場合はACM PCAを検討する

### AWS Private Certificate Authority

ACM Private CAは、社内向け・プライベート用途の証明書を発行するためのマネージドCA。

重要ポイント:

- 内部ALB、プライベートAPI、mTLS、社内サービス間TLSなどで使う
- パブリックに信頼される証明書ではなく、組織内の信頼基盤として考える
- ACMで証明書管理し、Private CAで発行元を管理する関係

ひっかけ:

- インターネット公開サイトの一般的な証明書は通常ACMのパブリック証明書
- Private CAはコストと運用設計も問われる

## 10. コンテナ・アプリケーションネットワーク

### Amazon ECS

コンテナを実行・管理するAWSのコンテナオーケストレーションサービス。

重要ポイント:

- EC2起動タイプとFargate起動タイプがある
- awsvpcネットワークモードではタスクにENIが割り当てられる
- ALB/NLB、Service Discovery、Cloud Map、Security Group、Subnet設計と関係する

ひっかけ:

- コンテナ通信でも、最終的にはVPC/Subnet/ENI/Security Groupの理解が必要

### AWS Fargate

サーバー管理なしでECS/EKSのコンテナを実行するコンピュート実行環境。

重要ポイント:

- EC2インスタンス管理をAWS側に任せられる
- タスク/Pod単位のENI、Security Group、Subnet選択が重要
- Private Subnet配置時は、イメージ取得や外向き通信のためにNAT GatewayまたはVPC Endpointを検討する

ひっかけ:

- サーバー管理不要でも、ネットワーク設計は不要にならない

### Amazon EKS

KubernetesをAWS上でマネージドに利用するサービス。

重要ポイント:

- AWS VPC CNI、Pod IP、Subnet IP消費、Load Balancer Controllerが試験範囲に絡む
- Ingress、Service、NLB/ALB連携が問われる

ひっかけ:

- Kubernetesの抽象概念とAWSネットワーク実体を対応付けて考える

### AWS Load Balancer Controller

EKSからALB/NLBを作成・管理するController。

重要ポイント:

- Kubernetes IngressやServiceからAWSロードバランサーを作る
- Target Group、Security Group、Subnetタグなどと関連する

ひっかけ:

- Kubernetes内だけの設定ではなく、VPC/Subnet/SG/IAMも必要

### AWS App Mesh

サービス間通信を制御・可視化するService Mesh。

重要ポイント:

- マイクロサービス間通信、トラフィック制御、可観測性の文脈で出る

ひっかけ:

- VPC間接続そのものを提供するサービスではない

### AWS Cloud Map

クラウドリソースやマイクロサービスの名前解決・サービスディスカバリを提供するサービス。

重要ポイント:

- ECS、EKS、App Mesh、Route 53と組み合わせて出る
- サービス名から動的にエンドポイントを発見する
- Private DNS Namespaceを使うとVPC内サービス名解決に使える

ひっかけ:

- Cloud Mapは名前解決/サービス発見であり、通信経路そのものを作るわけではない
- DNSが解決しても、Security GroupやRouteがなければ通信できない

## 11. 試験で迷いやすい比較

| 要件 | 第一候補 | 比較対象 |
| :--- | :--- | :--- |
| Private SubnetからS3へNATなしで接続 | Gateway Endpoint | NAT Gateway、Interface Endpoint |
| 多数のVPCを集約接続 | Transit Gateway | VPC Peering |
| 2つのVPCを単純接続 | VPC Peering | Transit Gateway |
| CIDR重複環境でサービスだけ公開 | PrivateLink | Peering、TGW |
| オンプレミスとAWSの暗号化接続 | Site-to-Site VPN | Direct Connect |
| 専用線で安定接続 | Direct Connect | Site-to-Site VPN |
| DXを暗号化したい | VPN over Direct Connect | DX単体 |
| グローバルWeb配信 | CloudFront | Global Accelerator |
| 固定Anycast IPでTCP/UDP最適化 | Global Accelerator | Route 53、CloudFront |
| HTTP/HTTPSのL7制御 | ALB | NLB |
| TCP/UDPのL4高性能LB | NLB | ALB |
| セキュリティアプライアンスを透過挿入 | Gateway Load Balancer | ALB、NLB |
| ステートフル検査の集中構成 | TGW Appliance Mode | 通常TGW Attachment |
| ユーザー端末からAWSへVPN接続 | Client VPN | Site-to-Site VPN |
| 拠点ルーターとAWSをVPN接続 | Site-to-Site VPN | Client VPN |
| VPC内からAPI GatewayをPrivateに呼ぶ | Private API + Interface Endpoint | Public API |
| API GatewayからVPC内バックエンドへ接続 | VPC Link | PrivateLink、NAT Gateway |
| サービス名で動的に宛先発見 | Cloud Map | 手動DNSレコード |
| HPC/MLの低レイテンシーEC2間通信 | EFA | ENA、Global Accelerator |
| EC2の一般的な高性能ネットワーク | ENA | EFA |
| TGW Route Tableの経路確認 | Route Analyzer | Reachability Analyzer |
| アカウント全体の健全性/クォータ確認 | Trusted Advisor | CloudWatch |
| 設計レビューと改善計画 | Well-Architected Tool | Config、Trusted Advisor |
| API操作履歴 | CloudTrail | Flow Logs |
| IP通信の許可/拒否ログ | VPC Flow Logs | CloudTrail |
| パケット内容の詳細解析 | Traffic Mirroring | Flow Logs |
| 静的な到達性分析 | Reachability Analyzer | ping、curl |
| Subnet単位のAllow/Deny | Network ACL | Security Group |
| ENI単位のステートフル許可 | Security Group | Network ACL |

## 12. 最短暗記フレーズ

```text
VPCはリージョン、SubnetはAZ。
Public SubnetはIGWへのRouteで決まる。
Private Subnetの外向き通信はNAT Gateway。
S3はVPC外のリージョンサービス。プライベート接続はVPC Endpoint。
Security Groupはステートフル、NACLはステートレス。
VPC Peeringは非推移的。
多数VPCはTransit Gateway。
サービス単位公開とCIDR重複回避はPrivateLink。
TGWでステートフル検査ならAppliance Mode。
Direct Connectは専用線だが暗号化ではない。
VPNはIPsecで暗号化。
BGPは動的経路交換。
CloudFrontはCDN、Global AcceleratorはAnycast入口。
CloudTrailはAPI操作履歴、Flow LogsはIP通信ログ。
Reachability Analyzerは実通信なしの到達性分析。
TGW内の経路確認はRoute Analyzer。
Client VPNは利用者端末、Site-to-Site VPNは拠点間。
Private APIはVPCからAPI Gateway、VPC LinkはAPI GatewayからVPC内バックエンド。
Cloud Mapはサービス発見。
ENAはEC2の高性能NIC、EFAはHPC/ML向け低レイテンシー通信。
```

## 13. 復習の進め方

1. 用語を見て、図を頭に描く
2. その用語を選ぶ要件を1つ言う
3. 似たサービスとの違いを1つ言う
4. 障害時に見る設定を1つ言う
5. 模擬問題で誤答した用語をこのファイルへ追記する
