# ANS-C01暗記チートシート

## 目的

AWS Certified Advanced Networking - Specialty（ANS-C01）で、問題文を読む速度と選択肢の切り分けに効く用語・数字をまとめる。

細かいサービスクォータは変更される可能性があるため、試験直前や実務利用時はAWS公式ドキュメントで確認する。

## 最優先で暗記する数字

| 項目 | 覚えること | 試験での使いどころ |
| :--- | :--- | :--- |
| RFC1918 | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | プライベートIP範囲、CIDR重複判定 |
| VPC CIDR | IPv4 VPCは `/16` から `/28` | VPC作成、CIDR追加、アドレス設計 |
| Subnet CIDR | IPv4 Subnetも `/16` から `/28` | サブネット分割、利用可能IP数 |
| AWS予約IP | サブネットの先頭4個と最後1個は使えない | 利用可能IP不足、サブネットサイズ計算 |
| `/16` | 65,536 IP | 大きめのVPC CIDR |
| `/24` | 256 IP、AWSでは251個利用可能 | よくあるサブネットサイズ |
| `/28` | 16 IP、AWSでは11個利用可能 | 最小サブネット、エンドポイント用サブネット設計 |
| Site-to-Site VPN | 1接続に2トンネル | 冗長化、フェイルオーバー |
| 標準VPNトンネル | 最大目安 `1.25 Gbps` | 帯域要件でDXや複数接続と比較 |
| Site-to-Site VPN MTU | `1446` | MTU/MSS/断片化問題 |
| Direct Connect Private VIF MTU | `9001` | Jumbo Frame |
| Direct Connect Transit VIF MTU | `8500` | TGW経由のJumbo Frame |
| Gateway Load Balancer | Geneve `UDP 6081` | GWLB/GWLBE/セキュリティアプライアンス |
| Global Accelerator | 固定Anycast IPv4を2つ | 固定IP、グローバル低遅延、許可リスト |
| NAT Gateway idle timeout | `350` 秒 | 長時間アイドル接続の切断、アプリ側keepalive |
| BGP | TCP `179` | DX/VPNのBGPセッション |
| DNS | TCP/UDP `53` | Route 53 Resolver、ハイブリッドDNS |
| HTTPS | TCP `443` | ALB、CloudFront、PrivateLink、エンドポイント |
| HTTP | TCP `80` | ALB、WAF、リダイレクト |
| IPsec IKE | UDP `500` | Site-to-Site VPN |
| IPsec NAT-T | UDP `4500` | NAT越しVPN、Accelerated VPN |
| ESP | IP protocol `50` | IPsec通信 |
| VPC Flow Logs protocol | `1=ICMP`, `6=TCP`, `17=UDP` | Flow Logs読解 |
| AWS ASN | `7224` | Direct ConnectのBGP communityでよく出る |
| DX local preference community | `7224:7100`, `7224:7200`, `7224:7300` | AWSからオンプレへの戻り経路優先度 |
| DX public VIF scope community | `7224:9100`, `7224:9200`, `7224:9300` | AWSパブリックプレフィックスの広告範囲 |

## 最優先で暗記する用語

| 用語 | 一言でいうと | 試験での見方 |
| :--- | :--- | :--- |
| Security Group | ENI/インスタンス単位のステートフル制御 | 戻り通信は自動許可。許可ルールのみ |
| Network ACL | サブネット単位のステートレス制御 | 戻り通信も明示許可。Allow/Denyあり。番号順 |
| Route Table | 宛先CIDRごとの転送先 | Public/Private判定、TGW/VGW/IGW/NATGW経路 |
| Longest Prefix Match | より具体的なCIDRが勝つ | `10.0.1.0/24` は `10.0.0.0/16` より優先 |
| NAT Gateway | Private Subnetから外へ出るためのNAT | インバウンド開始通信は不可。AZごと配置が基本 |
| Internet Gateway | VPCのインターネット出入口 | Public Subnetには `0.0.0.0/0 -> IGW` が必要 |
| Egress-only Internet Gateway | IPv6の外向き専用出口 | IPv6版の片方向インターネット出口 |
| ENI | 仮想ネットワークインターフェイス | SG、Private IP、Source/Destination Checkの単位 |
| ENA | EC2の高性能ネットワークアダプター | EC2性能不足、拡張ネットワーキング |
| EFA | HPC/ML向け低レイテンシー通信 | 分散処理、MPI、同一AZ配置 |
| VPC Endpoint | AWSサービスへプライベート接続 | Gateway型とInterface型を区別 |
| Gateway Endpoint | S3/DynamoDB向けエンドポイント | ルートテーブルにprefix list経路 |
| Interface Endpoint | PrivateLinkのENI型エンドポイント | サブネット内ENI、SGで制御 |
| PrivateLink | サービスをプライベートIPで公開/利用 | VPC間でCIDR重複していても使いやすい |
| VPC Peering | 2つのVPCを直接接続 | 推移的ルーティング不可 |
| Transit Gateway | 複数VPC/オンプレを集約するハブ | 大規模ネットワークの中継点 |
| TGW Association | AttachmentがどのTGW Route Tableを見るか | 転送判断に使うルート表 |
| TGW Propagation | Attachmentの経路をTGW Route Tableへ広げる | 経路の自動登録 |
| TGW Peering | TGW同士の接続 | 静的ルートが必要 |
| Appliance Mode | ステートフル検査で往復通信を同じAZ/装置に寄せる | Inspection VPC、GWLB、IDS/IPSで頻出 |
| Inspection VPC | 通信検査用VPC | Network Firewall、GWLB、IDS/IPSを配置 |
| AWS Network Firewall | マネージドL3-L7 Firewall | Stateful/Stateless、Suricata互換 |
| Suricata | オープンソースIDS/IPSエンジン | Network Firewallのstateful ruleで出る |
| Gateway Load Balancer | セキュリティアプライアンスを透過的に挟むLB | Geneve、GWLBE、Firewall挿入 |
| GWLBE | Gateway Load Balancer Endpoint | Consumer VPC側の入口 |
| VPC Traffic Mirroring | ENI通信を複製して監視装置へ送る | IDS/パケット解析。遮断ではなく監視寄り |

## Hybrid Connectivity

| 用語 | 覚えること |
| :--- | :--- |
| Site-to-Site VPN | インターネット経由のIPsec VPN。1接続2トンネル |
| Client VPN | 利用者端末からAWSへ接続するリモートアクセスVPN |
| Accelerated VPN | Global Acceleratorを使うVPN。TGW接続で利用。既存VPNに後からONではなく新規作成して切替 |
| Direct Connect | 専用線接続。安定した帯域/低遅延/閉域寄り |
| DX Connection | 物理的なDirect Connect接続 |
| VIF | Direct Connect上の論理インターフェース |
| Private VIF | VPC/VGW/DXGWへプライベートIPで接続 |
| Public VIF | S3などAWS public serviceへ接続 |
| Transit VIF | Direct Connect Gateway経由でTransit Gatewayへ接続 |
| DXGW | Direct Connect Gateway。DXと複数VPC/TGWをつなぐ中継 |
| VGW | Virtual Private Gateway。VPC側のVPN/DX接続先 |
| CGW | Customer Gateway。オンプレ側ルーターを表すAWSリソース |
| BGP | 動的経路交換。DX/VPNで頻出 |
| AS_PATH prepend | AS_PATHを長くして、その経路を優先されにくくする |
| Local Preference | Direct Connectの戻り経路制御で優先度を上げ下げする |
| MED | BGP属性。優先度はLocal PreferenceやAS_PATHより低め |
| ECMP | 同じコストの複数経路で負荷分散 |
| BFD | BGP障害検知を速くする仕組み |
| SiteLink | Direct Connectロケーション間で拠点間通信を行う機能 |
| TGW Multicast | TGWで一対多配信を扱う機能。Multicast Domain/Group/Source/Member |

## DNS

| 用語 | 覚えること |
| :--- | :--- |
| Route 53 Public Hosted Zone | インターネット向けDNSゾーン |
| Route 53 Private Hosted Zone | VPC内向けDNSゾーン |
| Route 53 Resolver | VPC標準DNS。AmazonProvidedDNS |
| Inbound Resolver Endpoint | オンプレからVPC内DNSへ問い合わせる入口 |
| Outbound Resolver Endpoint | VPCからオンプレDNSへ問い合わせる出口 |
| Resolver Rule | どのドメインをどのDNSへ転送するかのルール |
| Conditional Forwarding | 特定ドメインだけ別DNSへ転送 |
| Split-horizon DNS | 同じ名前を内部/外部で違う答えにする |
| DNS Firewall | Route 53 Resolverでドメイン単位にブロック/許可 |
| DNSSEC | DNS応答の正当性検証 |
| Cloud Map | サービスディスカバリ。サービス名から動的に宛先を発見 |

## Edge / Load Balancing

| 用語 | 覚えること |
| :--- | :--- |
| ALB | L7。HTTP/HTTPS、パス/ホストベースルーティング |
| NLB | L4。TCP/UDP/TLS、高性能、固定IPやPrivateLinkで使う |
| GWLB | L3/L4でアプライアンス挿入。Geneve |
| CloudFront | CDN。キャッシュ、エッジ配信、WAF連携 |
| Global Accelerator | キャッシュしない。固定Anycast IP、経路最適化、ALB/NLB/EC2/EIPへ |
| Anycast | 同じIPを複数拠点から広告し、近い入口へ誘導 |
| API Gateway Private API | VPC内からInterface Endpoint経由でAPIを呼ぶ |
| API Gateway VPC Link | API GatewayからVPC内のPrivateなALB/NLB/バックエンドへ接続 |
| Listener | LBの待ち受けプロトコル/ポート設定 |
| Target Group | LBの転送先グループ |
| X-Forwarded-For | 元クライアントIPをHTTPヘッダーで伝える |

## Security / Logging / Troubleshooting

| 用語 | 覚えること |
| :--- | :--- |
| VPC Flow Logs | ENI/Subnet/VPCの通信メタデータ。パケット中身は見ない |
| Reachability Analyzer | 到達性の静的解析 |
| Route Analyzer | TGW Route Tableの経路解析 |
| Traffic Mirroring | 実パケットを複製して解析 |
| CloudTrail | API操作履歴 |
| CloudWatch | メトリクス、ログ、アラーム |
| GuardDuty | 脅威検知 |
| AWS Config | リソース設定履歴と準拠性 |
| Firewall Manager | 複数アカウントにWAF/Firewallポリシーを展開 |
| WAF | HTTP/HTTPSのWeb攻撃対策 |
| Shield | DDoS対策 |
| Network Firewall | VPC単位のネットワークFirewall |
| Kinesis Data Firehose | ログをS3/OpenSearch/Splunkなどへ配送 |
| Athena | S3上のログをSQLで分析 |
| Trusted Advisor | クォータ、冗長化、セキュリティ、コストなどの健全性確認 |
| Well-Architected Tool | 設計レビューと改善計画 |

## 暗記優先順位

1. CIDR、RFC1918、AWS予約IP
2. Security GroupとNetwork ACLの違い
3. Route Table、Longest Prefix Match、NAT Gateway、Internet Gateway
4. Transit GatewayのAssociation、Propagation、Appliance Mode
5. Direct ConnectのVIF 3種類
6. BGP属性: AS_PATH、Local Preference、MED、ECMP
7. Route 53 ResolverのInbound/Outbound
8. Global AcceleratorとCloudFrontの違い
9. Network Firewall、GWLB、Geneve、Suricata
10. API Gateway Private API / VPC Link、Cloud Map
11. Client VPN、SiteLink、TGW Multicast
12. Flow Logs、Reachability Analyzer、Route Analyzer、Traffic Mirroring、Athena

## 試験での見抜き方

| 問題文のキーワード | 疑うサービス/設計 |
| :--- | :--- |
| 固定IP、世界中、低遅延、ALBの前段 | Global Accelerator |
| キャッシュ、静的コンテンツ、エッジ配信 | CloudFront |
| 複数VPC、オンプレ、集約、ハブ | Transit Gateway |
| VPC間でCIDR重複 | PrivateLink |
| S3/DynamoDBへプライベート接続 | Gateway Endpoint |
| その他AWSサービスへプライベート接続 | Interface Endpoint |
| 利用者端末からAWSへVPN | Client VPN |
| APIをVPC内だけから呼ばせたい | API Gateway Private API + Interface Endpoint |
| API GatewayからPrivate Subnetのバックエンドへ | VPC Link |
| サービス名で動的に宛先を見つける | AWS Cloud Map |
| オンプレからVPC内DNS | Route 53 Resolver inbound endpoint |
| VPCからオンプレDNS | Route 53 Resolver outbound endpoint |
| IDS/IPS、Suricata、ドメイン検査 | AWS Network Firewall |
| サードパーティFirewallを挟む | Gateway Load Balancer |
| 往復通信が別装置を通って落ちる | TGW Appliance Mode |
| VPNの遅延や不安定なインターネット経路 | Accelerated Site-to-Site VPN |
| 専用線、安定帯域、BGP | Direct Connect |
| 経路を優先/非優先にしたい | BGP属性、Local Preference、AS_PATH prepend |
| TGW Route Tableの経路を確認したい | Route Analyzer |
| VPC内のSG/NACL/Route込みで到達性確認 | Reachability Analyzer |
| HPC、MPI、機械学習クラスター | EFA |
| サービスクォータや冗長化の健全性確認 | Trusted Advisor |

## 公式参照

- [AWS Certified Advanced Networking - Specialty (ANS-C01)](https://docs.aws.amazon.com/aws-certification/latest/advanced-networking-specialty-01/advanced-networking-specialty-01.html)
- [VPC CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)
- [Subnet CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html)
- [Direct Connect virtual interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)
- [Direct Connect routing policies and BGP communities](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)
- [Site-to-Site VPN quotas](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-limits.html)
- [AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html)
- [Transit Gateway appliance mode](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)
- [AWS Network Firewall stateful rule groups](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups-ips.html)
