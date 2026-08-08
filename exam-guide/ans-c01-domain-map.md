# ANS-C01ドメインマップ

## 目的

ANS-C01の試験範囲と学習成果物を対応付けるための索引。

頻出用語の詳細は [ANS-C01 頻出・必須用語集](./ans-c01-key-terms.md) を参照する。

## Domain 1: Network Design（30%）

- エッジネットワークとグローバル通信
- Public、Private、Hybrid DNS
- ロードバランシング
- ログ、監視、可視化設計
- オンプレミスとAWS間の接続設計
- マルチアカウント、マルチリージョン、マルチVPC設計

対応資料:

| Task | 主に見る資料 |
| :--- | :--- |
| Edge network / global traffic | [Edge Network Services](../services/edge-network-services-for-ans-c01.md) |
| Public / private / hybrid DNS | [Route 53](../services/route53-for-ans-c01.md) |
| Load balancing | [Edge Network Services](../services/edge-network-services-for-ans-c01.md)、[用語集](./ans-c01-key-terms.md) |
| Logging / monitoring design | [Monitoring and Operations](../services/monitoring-and-operations-for-ans-c01.md) |
| On-premises - AWS connectivity | [TGW/VIF/VPN](../fundamentals/tgw-vif-vpn-cheatsheet.md)、[BGP](../fundamentals/bgp-for-ans-c01.md) |
| Multi-account / multi-Region / multi-VPC | [TGW/VIF/VPN](../fundamentals/tgw-vif-vpn-cheatsheet.md)、[Network Automation](../services/network-automation-for-ans-c01.md) |

## Domain 2: Network Implementation（26%）

- Direct Connect、VPN、BGPの実装
- VPC Peering、Transit Gateway、PrivateLinkの実装
- Hybrid DNSとRoute 53 Resolver
- AWS CLI、CloudFormationなどによる自動化
- API Gateway private API / VPC Link、Cloud Map、EKS/ECSなどアプリ接続の実装

対応資料:

| Task | 主に見る資料 |
| :--- | :--- |
| Hybrid routing/connectivity | [TGW/VIF/VPN](../fundamentals/tgw-vif-vpn-cheatsheet.md)、[BGP](../fundamentals/bgp-for-ans-c01.md) |
| Multi-account / VPC connectivity | [TGW/VIF/VPN](../fundamentals/tgw-vif-vpn-cheatsheet.md)、[用語集](./ans-c01-key-terms.md) |
| Hybrid DNS | [Route 53](../services/route53-for-ans-c01.md) |
| Network automation | [Network Automation](../services/network-automation-for-ans-c01.md) |

## Domain 3: Network Management and Operation（20%）

- AWSおよびハイブリッド環境の経路管理
- Flow Logs、Reachability Analyzer、Traffic Mirroringによる調査
- 性能、可用性、コスト最適化
- Route Analyzer、AWS Network Manager、Trusted Advisor、Well-Architectedの使いどころ
- ENI、ENA、EFA、MTU、multicast、secondary CIDRなどの性能/拡張要素

対応資料:

| Task | 主に見る資料 |
| :--- | :--- |
| Routing and connectivity maintenance | [TGW/VIF/VPN](../fundamentals/tgw-vif-vpn-cheatsheet.md)、[Troubleshooting](../services/troubleshooting-for-ans-c01.md) |
| Monitoring / traffic analysis | [Monitoring and Operations](../services/monitoring-and-operations-for-ans-c01.md)、[Troubleshooting](../services/troubleshooting-for-ans-c01.md) |
| Performance / reliability / cost optimization | [暗記チートシート](./ans-c01-memorization-cheatsheet.md)、[用語集](./ans-c01-key-terms.md) |

## Domain 4: Network Security, Compliance, and Governance（24%）

- Security Group、Network ACL、Network Firewall
- WAF、Shield、Firewall Manager
- マルチアカウントガバナンス
- 通信暗号化、IPsec、TLS、証明書、DNSSEC

対応資料:

| Task | 主に見る資料 |
| :--- | :--- |
| Inbound / outbound / inter-VPC security | [Security Services](../services/security-services-for-ans-c01.md)、[Troubleshooting](../services/troubleshooting-for-ans-c01.md) |
| Security validation / audit | [Monitoring and Operations](../services/monitoring-and-operations-for-ans-c01.md)、[Network Automation](../services/network-automation-for-ans-c01.md) |
| Encryption / certificates / secure DNS | [Security Services](../services/security-services-for-ans-c01.md)、[Route 53](../services/route53-for-ans-c01.md) |

## 公式範囲で見落としやすい補助サービス

| サービス/機能 | 出題での見方 | 主に見る資料 |
| :--- | :--- | :--- |
| Amazon API Gateway | private API、VPC Link、CloudFront/WAF/ACM連携 | [用語集](./ans-c01-key-terms.md)、[Edge Network Services](../services/edge-network-services-for-ans-c01.md) |
| AWS Cloud Map | ECS/EKS/App Meshのservice discovery | [用語集](./ans-c01-key-terms.md)、[Route 53](../services/route53-for-ans-c01.md) |
| AWS Client VPN | ユーザー端末からAWS/オンプレへVPN接続 | [TGW/VIF/VPN](../fundamentals/tgw-vif-vpn-cheatsheet.md) |
| AWS Trusted Advisor | quota、fault tolerance、security gapの確認 | [Monitoring and Operations](../services/monitoring-and-operations-for-ans-c01.md) |
| AWS Well-Architected Tool | 設計レビュー、信頼性/性能/コスト最適化 | [Monitoring and Operations](../services/monitoring-and-operations-for-ans-c01.md) |
| ENA / EFA | EC2ネットワーク性能、HPC/ML低遅延通信 | [用語集](./ans-c01-key-terms.md) |
| TGW multicast | 1対多配信、multicast domain、IGMP | [TGW/VIF/VPN](../fundamentals/tgw-vif-vpn-cheatsheet.md) |

## 初期学習順序

1. VPC、Subnet、Route Table、SG、NACL
2. DNS、Route 53、Resolver
3. VPC Peering、Transit Gateway、PrivateLink
4. Site-to-Site VPN、Direct Connect、BGP
5. ELB、CloudFront、Global Accelerator
6. Flow Logs、Reachability Analyzer、Traffic Mirroring
7. Network Firewall、WAF、Shield、ガバナンス
8. 模擬問題、誤答分析、総復習
