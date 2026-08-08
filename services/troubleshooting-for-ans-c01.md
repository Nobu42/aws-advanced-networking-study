# Troubleshooting ANS-C01対策

## 目的

ANS-C01で出題されるAWSネットワークのトラブルシューティングを、問題文から原因候補、確認ポイント、使うサービスを選べる形で整理する。

対象は主に次の領域。

```text
VPC routing
Security Group
Network ACL
VPC Flow Logs
Reachability Analyzer
Network Access Analyzer
Transit Gateway
VPC Peering
PrivateLink
NAT Gateway
Internet Gateway
Route 53 / Resolver
Site-to-Site VPN
Direct Connect
BGP
ALB / NLB / GWLB
CloudFront / Global Accelerator
Network Firewall
CloudWatch
CloudTrail
AWS Config
```

## 最初に覚える結論

| 症状 | まず疑うもの |
| :--- | :--- |
| EC2へ接続できない | Route Table、Security Group、NACL、対象OSのlisten、Reachability Analyzer |
| private subnetからインターネットへ出られない | NAT Gateway、Route Table、NACL、DNS、VPC Flow Logs |
| public subnetなのにインターネットから入れない | IGW、public IP、Route Table、Security Group、NACL |
| VPC Peering先へ通信できない | 両側Route Table、SG/NACL、CIDR重複、DNS解決、推移的ルーティング不可 |
| TGW経由で通信できない | TGW route table association/propagation、VPC route、attachment subnet、戻り経路 |
| Inspection VPCで通信が断続的 | 非対称ルーティング、TGW appliance mode、AZごとの経路 |
| VPNトンネルが上がらない | IKE、IPsec、PSK、暗号設定、UDP 500/4500、Customer Gateway |
| VPNは上がるが経路が流れない | BGP peer、ASN、prefix広告、route propagation、優先経路 |
| Direct ConnectのBGPが張れない | VLAN、peer IP、ASN、MD5、TCP 179、prefix limit |
| DNS名が解決できない | VPC DNS属性、Route 53 Resolver、Private Hosted Zone関連付け、Resolver rule |
| ALB targetがunhealthy | Health check path/port/code、SG、NACL、アプリlisten、戻り通信 |
| ALB/CloudFrontで502/503/504 | target不在、target応答不正、timeout、SG/NACL、origin到達性 |
| CloudFrontで403 | S3/OAC/OAI権限、WAF、署名URL、オブジェクト名、origin設定 |
| Global Accelerator経由だけ失敗 | Endpoint health、ALB/NLB SG、client IP許可、endpoint group、listener |
| Network Firewallで期待通り許可/遮断されない | ルート、stateless/stateful rule、非対称ルーティング、ログ |
| API変更の原因を知りたい | CloudTrail |
| 現在の設定と変更履歴を知りたい | AWS Config |
| 通信が許可/拒否されたか見たい | VPC Flow Logs |

## トラブルシューティングの基本手順

いきなりサービス名を選ぶのではなく、次の順で切り分ける。

```text
1. どこからどこへ通信したいか
2. 名前解決は成功しているか
3. 送信元と宛先IPは想定通りか
4. ルートは往復で存在するか
5. SG/NACL/firewallは許可しているか
6. 宛先サービスはlistenしているか
7. 実ログとメトリクスは何を示しているか
8. 直前に誰が何を変更したか
```

覚え方:

```text
名前解決
  -> 経路
  -> 通信制御
  -> アプリ
  -> ログ
  -> 変更履歴
```

## 使う道具の選び方

| 見たいこと | 使うもの |
| :--- | :--- |
| 設定上、通信できる経路か | Reachability Analyzer |
| 実際に通信が発生し、ACCEPT/REJECTされたか | VPC Flow Logs |
| どのAPI操作で変更されたか | CloudTrail |
| 現在の設定、過去の設定、準拠状態 | AWS Config |
| 数値として劣化や障害を見たい | CloudWatch Metrics / Alarms |
| ログを検索・集計したい | CloudWatch Logs Insights |
| DNS問い合わせを見たい | Route 53 Resolver Query Logging |
| パケット内容をIDSで見たい | VPC Traffic Mirroring |
| 意図しない公開経路を検出したい | Network Access Analyzer |

## まず見るべきAWSネットワーク構成要素

### VPC通信の基本チェック

| 観点 | 確認 |
| :--- | :--- |
| CIDR | 送信元/宛先VPCやオンプレと重複していないか |
| Subnet | 対象リソースが想定subnetにいるか |
| Route Table | 宛先CIDRに対する最長一致ルートが正しいか |
| Security Group | stateful。必要なinbound/outboundがあるか |
| Network ACL | stateless。往復のport、特にephemeral portを許可しているか |
| ENI | 正しいENIにFlow LogsやSGが付いているか |
| DNS | 名前解決結果が想定IPか |
| Return path | 戻り通信のルートがあるか |

重要:

```text
AWSのルーティングも戻り経路が必要。
Security Groupはstatefulだが、Route TableとNACLは戻り方向も考える。
```

## Security GroupとNetwork ACLの切り分け

### Security Group

Security GroupはENIに付くstateful firewallである。

確認ポイント:

```text
inbound rule
outbound rule
protocol
port
source/destination
Security Group参照
IPv4/IPv6の違い
```

試験での見方:

| 症状 | 疑う設定 |
| :--- | :--- |
| ALBからtargetへ到達できない | target SGがALB SGを許可しているか |
| EC2へSSHできない | EC2 SG inbound TCP 22 |
| 戻り通信だけ落ちるように見える | SGよりNACLやrouteを疑う |

### Network ACL

Network ACLはsubnet単位のstateless firewallである。

確認ポイント:

```text
inbound rule
outbound rule
rule number
allow/deny
ephemeral port
IPv4/IPv6
```

重要:

```text
NACLはstatelessなので戻り通信も明示的に許可する。
クライアント側のephemeral portを閉じると、ALBやEC2の通信がtimeoutしやすい。
```

## VPC Flow Logsの読み方

VPC Flow Logsは、VPC、subnet、ENIのIPトラフィックメタデータを記録する。

デフォルト形式:

```text
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

例:

```text
2 123456789012 eni-abc123 10.0.1.10 10.0.2.20 51524 443 6 10 8400 1720000000 1720000060 ACCEPT OK
```

読み方:

| フィールド | 意味 |
| :--- | :--- |
| `10.0.1.10` | 送信元IP |
| `10.0.2.20` | 宛先IP |
| `51524` | 送信元port |
| `443` | 宛先port |
| `6` | TCP |
| `ACCEPT` | SG/NACLで許可 |
| `REJECT` | SG/NACLなどで拒否 |
| `OK` | 正常に記録 |

注意:

```text
ACCEPT = アプリが正常応答した、ではない。
REJECT = SG/NACLで拒否された可能性が高い。
NODATA = その集計期間に通信がなかった。
SKIPDATA = 一部ログがスキップされた。
```

## Reachability Analyzerの使いどころ

Reachability Analyzerは、実パケットを送らずに、設定上の到達性を静的分析する。

得意なこと:

```text
Route Tableの不足
Security Groupの不足
Network ACLの拒否
Internet Gateway/NAT Gateway/TGW/Peering/Endpoint経路の確認
どのコンポーネントで止まるかの特定
```

向かないこと:

```text
アプリが起動しているか
HTTP 200が返るか
認証が成功するか
一時的なpacket lossがあるか
実際の遅延が何msか
```

試験での判断:

```text
通信できない原因をルート/SG/NACLから調べる = Reachability Analyzer
実際に通信が発生したかを見る = VPC Flow Logs
```

## Public subnetへ接続できない

症状:

```text
インターネットからEC2/ALBへ接続できない
public subnetのつもりだが外部から到達しない
```

確認順:

| 順序 | 確認 |
| :--- | :--- |
| 1 | VPCにInternet Gatewayがattachされているか |
| 2 | subnet route tableに`0.0.0.0/0 -> igw-...`があるか |
| 3 | EC2にpublic IPv4またはElastic IPがあるか |
| 4 | ALBならinternet-facingか |
| 5 | Security Group inboundがclient IP/portを許可しているか |
| 6 | NACL inbound/outboundが往復を許可しているか |
| 7 | OS firewallとアプリlistenを確認 |

よくある誤解:

```text
public subnet = 自動でインターネットから到達可能、ではない。
IGWへのrouteとpublic IP、SG/NACL、listenが必要。
```

## Private subnetからインターネットへ出られない

症状:

```text
private subnetのEC2がyum/apt、外部API、OS updateに失敗する
```

確認順:

| 順序 | 確認 |
| :--- | :--- |
| 1 | private subnet route tableに`0.0.0.0/0 -> nat-...`があるか |
| 2 | NAT Gatewayがpublic subnetにあるか |
| 3 | NAT Gatewayのsubnet route tableに`0.0.0.0/0 -> igw-...`があるか |
| 4 | NAT GatewayにElastic IPがあるか |
| 5 | EC2のSecurity Group outboundが許可されているか |
| 6 | NACLでephemeral portを含む往復が許可されているか |
| 7 | DNS解決できているか |
| 8 | NAT Gatewayメトリクスでport枯渇やdropがないか |

NAT Gatewayで覚える数字:

```text
idle timeout: 350秒
1つのIPv4アドレスあたり同一宛先へ約55,000同時接続
```

試験での見方:

| 問題文 | 疑うこと |
| :--- | :--- |
| pingしてもNAT Gatewayが応答しない | NAT Gatewayはping応答確認対象ではない |
| 長時間idle後に切れる | NAT Gateway idle timeout |
| 接続数が多く新規接続できない | port枯渇。NAT GW追加、AZ分散、secondary IP |
| tracerouteにNAT GW private IPが出ない | 別経路を通っている可能性 |

## VPC Peeringのトラブルシューティング

VPC Peeringで通信できない場合の確認ポイント。

| 観点 | 確認 |
| :--- | :--- |
| Peering状態 | activeか |
| CIDR | VPC CIDRが重複していないか |
| Route Table | 両側に相手CIDR -> pcx routeがあるか |
| Security Group | 相手CIDRまたはpeer SGを許可しているか |
| NACL | 往復通信を許可しているか |
| DNS | Private DNS解決やDNS resolution設定 |
| 推移的ルーティング | VPC Peeringはtransitive routing不可 |

重要:

```text
VPC A -- Peering -- VPC B -- Peering -- VPC C

AからCへは直接通信できない。
VPC Peeringは推移的ルーティングをサポートしない。
```

## Transit Gatewayのトラブルシューティング

TGWでは、VPCルートとTGWルートの両方を見る。

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| Attachment | VPC/VPN/DXGW attachmentがavailableか |
| Association | attachmentが正しいTGW route tableに関連付いているか |
| Propagation | 必要な経路がTGW route tableへ伝播しているか |
| Static route | blackholeや誤った静的routeがないか |
| VPC route table | subnet route tableが宛先CIDRをTGWへ向けているか |
| Return path | 戻り経路もTGWへ向いているか |
| Attachment subnet | 対象AZにTGW attachment subnetがあるか |
| Appliance mode | stateful appliance経由なら有効化されているか |

よくある失敗:

```text
VPC route tableだけ設定して、TGW route tableを設定していない。
TGW route propagationを有効にしたが、association先が違う。
戻り経路が別attachmentへ向いている。
blackhole routeが最長一致している。
```

## TGW Appliance Modeと非対称ルーティング

Inspection VPCやFirewall VPCをTGWに接続し、stateful applianceで検査する場合は、非対称ルーティングが問題になりやすい。

問題の形:

```text
往路: VPC A -> TGW -> Inspection VPC AZ-a -> VPC B
復路: VPC B -> TGW -> Inspection VPC AZ-c -> VPC A
```

stateful firewallやIDS/IPSは、往路と復路が同じ装置、または同じflow stateを持つ経路を通る必要がある。

対策:

```text
Inspection VPCのTGW VPC attachmentでappliance mode supportを有効化する。
AZごとのroute tableを対称に設計する。
GWLB/Network FirewallのendpointをAZごとに配置する。
```

試験でのキーワード:

```text
stateful inspection
intermittent connectivity
cross-AZ traffic
inspection VPC
shared services VPC
appliance mode
asymmetric routing
```

## PrivateLink / VPC Endpointのトラブルシューティング

### Interface Endpoint

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| Endpoint状態 | availableか |
| Subnet/AZ | 利用するAZにendpoint ENIがあるか |
| Security Group | endpoint ENIのSGが送信元を許可しているか |
| Private DNS | 有効化されているか |
| DNS解決 | public IPではなくendpoint private IPへ解決しているか |
| Endpoint policy | 必要なAPI操作が許可されているか |
| Service側承認 | endpoint serviceで承認待ちになっていないか |

### Gateway Endpoint

対象:

```text
S3
DynamoDB
```

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| Route Table | 対象subnet route tableにprefix list routeがあるか |
| Endpoint policy | S3/DynamoDB操作が許可されているか |
| Bucket policy | `aws:sourceVpce`など条件が一致しているか |
| Region | 対象サービスとendpointのリージョンが一致しているか |

試験での見方:

```text
AWSサービスへprivate接続 = VPC Endpoint
自社/他社サービスをprivate公開 = Endpoint Service + PrivateLink
DNSがpublic endpointへ向く = Private DNS設定を疑う
```

## Route 53 / Resolver / DNSのトラブルシューティング

DNS障害では、名前解決と通信経路を分ける。

確認順:

| 順序 | 確認 |
| :--- | :--- |
| 1 | `dig` / `nslookup`で期待する名前が引けるか |
| 2 | 返ってきたIPがpublic/privateどちらか |
| 3 | VPCの`enableDnsSupport`と`enableDnsHostnames` |
| 4 | Private Hosted Zoneが正しいVPCに関連付いているか |
| 5 | 同名のpublic/private hosted zoneがないか |
| 6 | Resolver outbound/inbound endpointのSG |
| 7 | Resolver ruleのdomain、target IP、関連付けVPC |
| 8 | オンプレDNSのforwarder設定 |
| 9 | Resolver Query Loggingで問い合わせの実態 |

よくある失敗:

```text
Private Hosted Zoneを作ったがVPC関連付けがない。
オンプレからAWS private zoneを引きたいがInbound Resolver Endpointがない。
AWSからオンプレDNSを引きたいがOutbound Resolver EndpointとResolver ruleがない。
DNS Firewallでブロックされている。
TTLが残っていて変更がすぐ反映されない。
```

## Site-to-Site VPNのトラブルシューティング

VPNはレイヤーごとに見る。

```text
1. Internet reachability
2. IKE
3. IPsec
4. BGPまたはstatic route
5. VPC/TGW route
6. SG/NACL
7. アプリ通信
```

### トンネルがDOWN

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| Customer Gateway IP | AWS側設定とオンプレ側public IPが一致しているか |
| PSK | 事前共有鍵が一致しているか |
| IKE version | IKEv1/IKEv2が一致しているか |
| Encryption/Auth/PFS | AWS設定ファイルと一致しているか |
| UDP port | 500/4500が途中で遮断されていないか |
| NAT-T | NAT配下ならNAT Traversalを確認 |
| Firewall | ESPまたはUDP encapsulationが許可されているか |

### トンネルはUPだが通信不可

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| BGP peer | establishedか |
| ASN | Customer ASN / Amazon ASNが正しいか |
| Prefix advertisement | 必要なCIDRを広告しているか |
| Route propagation | VGW/TGW route tableへ伝播しているか |
| Static route | static VPNならAWS側routeがあるか |
| Return route | オンプレ側にVPC CIDRへの戻り経路があるか |
| SG/NACL | 宛先VPC側でオンプレCIDRを許可しているか |

CloudWatchで見るもの:

```text
TunnelState
TunnelDataIn
TunnelDataOut
```

## Direct Connectのトラブルシューティング

Direct ConnectはLayer 1から順に見る。

```text
Layer 1: physical
Layer 2: VLAN
Layer 3: IP/BGP
Layer 4: TCP 179
Routing: prefix advertisement / route preference
```

### BGPが確立しない

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| VIF状態 | availableか |
| VLAN ID | LOA-CFA/プロバイダ設定と一致しているか |
| Peer IP | AWS側/顧客側peer IPが正しいか |
| ASN | local ASN / Amazon ASNが一致しているか |
| MD5 | 認証キーが一致しているか |
| TCP 179 | firewall/ACLで遮断されていないか |
| Prefix limit | 広告経路数上限を超えていないか |

### BGPはUPだが経路がおかしい

確認ポイント:

```text
受信prefix
広告prefix
AS_PATH
local preference
BGP community
最長一致
VGW/DXGW/TGW association
VPNとの優先順位
```

試験での見方:

| 問題文 | 疑うこと |
| :--- | :--- |
| DXよりVPNが使われる | BGP属性、AS_PATH、local preference、prefix長 |
| 一部prefixだけ届かない | prefix広告漏れ、prefix limit、route filter |
| public VIFでAWS public serviceへ行けない | public prefix広告、BGP policy、NAT/Firewall |
| transit VIFでTGWへつながらない | DXGW association、allowed prefix、TGW route |

## MTU / Jumbo Frame / Path MTU Discovery

ANS-C01では、MTU、jumbo frame、fragmentation、DF bitが出ることがある。

基本:

| 用語 | 意味 |
| :--- | :--- |
| MTU | 1フレーム/パケットで送れる最大サイズ |
| Jumbo frame | 通常の1500 bytesを超える大きなMTU |
| PMTUD | Path MTU Discovery。経路上で通る最大MTUを見つける仕組み |
| DF bit | Don't Fragment。fragmentation禁止 |
| MSS | TCP segmentの最大サイズ |

症状:

```text
小さい通信は通るが大きい通信だけ失敗する
TLS handshakeや大きいPOSTだけ失敗する
VPN/DX/TGW経由だけ一部アプリがtimeoutする
```

確認ポイント:

```text
経路上の最小MTU
ICMP Fragmentation Neededが遮断されていないか
DF bit付きpacket
VPN encapsulationによるoverhead
jumbo frame対応範囲
MSS clamping
```

## ALBのトラブルシューティング

### Targetがunhealthy

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| Health check path | 存在するpathか |
| Health check port | targetがlistenしているportか |
| Success code | アプリが期待コードを返すか |
| Target SG | ALB SGからの通信を許可しているか |
| NACL | ALB subnetとtarget subnet間の往復を許可しているか |
| アプリ | processが起動し、応答時間がtimeout内か |

よくある失敗:

```text
health check pathが認証必須で302/401を返す。
targetは起動しているがhealth check portが違う。
target SGがclient IPだけ許可し、ALB SGを許可していない。
NACLでephemeral portが閉じている。
```

### HTTP 502 / 503 / 504

| Status | 代表原因 |
| :--- | :--- |
| 502 | targetから不正応答、RST、SSL handshake失敗、header不正 |
| 503 | registered targetなし、healthy targetなし |
| 504 | target接続timeout、target応答timeout、NACLでephemeral port遮断 |

確認するログ/メトリクス:

```text
ALB CloudWatch Metrics
ALB Access Logs
Target Group health check reason
Application logs
VPC Flow Logs
```

## NLBのトラブルシューティング

NLBはL4ロードバランサーである。

確認ポイント:

```text
target health
listener port/protocol
target group protocol
cross-zone load balancing
client IP preservation
Security Group/NACL
targetがclient IPを許可しているか
```

注意:

```text
NLBではclient IP preservationにより、target側から見る送信元がclient IPになる構成がある。
target SG/NACLでNLBだけを許可したつもりでも、client IP許可が必要になる場合がある。
```

## GWLB / Security Applianceのトラブルシューティング

Gateway Load Balancerは、firewall、IDS/IPS、DPI applianceを透過的に挟むためのL3ロードバランサーである。

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| Route Table | GWLB Endpointへ向けるrouteが往復であるか |
| Appliance health | target applianceがhealthyか |
| GENEVE | applianceがGENEVE UDP 6081に対応しているか |
| Symmetry | 往路/復路が同じ検査経路を通るか |
| Source/Destination Check | appliance要件に応じて設定されているか |
| Logging | appliance側ログ、Flow Logs |

試験での見方:

```text
IDS/IPS applianceをスケールさせたい = GWLB
GENEVE UDP 6081 = GWLB
通信が断続的 = 非対称ルーティングやappliance healthを疑う
```

## AWS Network Firewallのトラブルシューティング

Network Firewallはstateful inspectionを行うため、非対称ルーティングに弱い。

確認ポイント:

```text
Firewall endpointがAZごとにあるか
subnet route tableがfirewall endpointへ向いているか
戻り経路も同じfirewall endpointを通るか
stateless rule default action
stateful rule group
Suricata rule
flow log / alert log
```

よくある失敗:

```text
往路だけNetwork Firewallを通し、復路が別経路で戻る。
TGW集中検査でappliance modeを有効にしていない。
default actionでstateful rule groupへforwardしていない。
ルールの優先順位やdomain listの向きが誤っている。
```

## CloudFrontのトラブルシューティング

CloudFrontでは、viewer側、edge側、origin側を分ける。

```text
Viewer -> CloudFront Edge -> Origin
```

確認ポイント:

| 症状 | 疑うもの |
| :--- | :--- |
| 403 | S3権限、OAC/OAI、WAF、署名URL/Cookie、オブジェクト名 |
| 404 | originにオブジェクトがない、path違い、大文字小文字 |
| 502 | origin TLS、DNS、port、protocol、Lambda@Edge/Function |
| 503 | origin過負荷、CloudFront capacity、Lambda@Edge error |
| 504 | origin timeout、SG/NACL、firewall、origin応答遅延 |

S3 originでよくある確認:

```text
OACまたはOAI設定
bucket policy
Block Public Access
object keyの大文字小文字
default root object
origin path
```

ALB originでよくある確認:

```text
ALB security groupでCloudFrontからの通信を許可
origin protocol policy
ACM certificate
Host header
target health
```

## Global Acceleratorのトラブルシューティング

Global Acceleratorは、固定エニーキャストIPで最寄りのAWS edgeへ入り、AWSグローバルネットワーク経由でALB/NLB/EC2/EIPなどへ転送する。

確認ポイント:

| 観点 | 確認 |
| :--- | :--- |
| Accelerator | enabledか |
| Listener | port/protocolが正しいか |
| Endpoint group | region、traffic dial、health check |
| Endpoint | ALB/NLB/EC2/EIPがhealthyか |
| Security Group | ALBなどで必要な送信元を許可しているか |
| Client IP preservation | 有効時、送信元IPの扱いを理解しているか |

試験での見方:

```text
固定IPでグローバル最適化 = Global Accelerator
企業FWに許可リスト登録 = Global Acceleratorの固定Anycast IP
ALB直アクセスを防ぐ = private ALB + GA構成、またはSGでGA経由を想定した制御
```

## CloudTrail / AWS Configで変更原因を追う

通信障害は、直前の設定変更が原因のことが多い。

### CloudTrail

CloudTrailで見るもの:

```text
eventTime
eventSource
eventName
userIdentity
sourceIPAddress
requestParameters
errorCode
```

代表イベント:

```text
AuthorizeSecurityGroupIngress
RevokeSecurityGroupIngress
CreateRoute
ReplaceRoute
DeleteRoute
AssociateRouteTable
CreateTransitGatewayRoute
EnableTransitGatewayRouteTablePropagation
ModifyVpcEndpoint
ModifyLoadBalancerAttributes
UpdateDistribution
ChangeResourceRecordSets
```

### AWS Config

AWS Configで見るもの:

```text
現在のリソース設定
設定変更の時系列
準拠/非準拠
変更前後の差分
```

使い分け:

| 質問 | 使うもの |
| :--- | :--- |
| 誰がAPIを実行したか | CloudTrail |
| 現在の設定がどうなっているか | AWS Config |
| いつ設定が変わったか | CloudTrail + AWS Config |
| 社内ルール違反か | AWS Config Rules |

## 試験でよく出る判断パターン

### 「通信できない」

最優先で確認:

```text
Route Table
Security Group
Network ACL
DNS
Reachability Analyzer
VPC Flow Logs
```

### 「通信が断続的」

疑うもの:

```text
非対称ルーティング
AZごとのルート差分
TGW appliance mode未設定
NAT GatewayのAZ依存
NACLのephemeral port
target healthの揺れ
DNSの複数回答
```

### 「一部の宛先だけ失敗」

疑うもの:

```text
最長一致route
prefix advertisement漏れ
NACL
DNS回答差分
MTU/PMTUD
WAF/Firewall rule
```

### 「小さい通信は通るが大きい通信が失敗」

疑うもの:

```text
MTU
DF bit
PMTUD
ICMP blocked
VPN encapsulation overhead
MSS clamping
```

### 「名前では失敗、IPでは成功」

疑うもの:

```text
Route 53 record
Private Hosted Zone association
Resolver rule
VPC DNS attributes
DNS Firewall
TTL/cache
```

### 「IPでは失敗、名前解決は成功」

疑うもの:

```text
Route Table
Security Group
Network ACL
Firewall
target listen
return path
```

## トラブルシューティング早見表

| 領域 | 症状 | 確認 |
| :--- | :--- | :--- |
| VPC | EC2へ入れない | route、SG、NACL、public IP、listen |
| NAT | private subnetから外へ出られない | NAT GW、IGW、route、NACL、DNS |
| Peering | peer VPCへ行けない | 両側route、CIDR重複、SG/NACL、推移的不可 |
| TGW | VPC間通信不可 | association、propagation、blackhole、VPC route |
| Inspection | 通信が断続的 | appliance mode、非対称routing、AZ |
| VPN | tunnel down | IKE、IPsec、PSK、UDP 500/4500 |
| VPN | tunnel upだが通信不可 | BGP/static route、propagation、戻り経路 |
| DX | BGP down | VLAN、peer IP、ASN、MD5、TCP 179 |
| DNS | 名前解決不可 | PHZ、Resolver endpoint/rule、VPC DNS属性 |
| ALB | target unhealthy | health check、SG、NACL、app |
| CloudFront | 403 | S3権限、OAC/OAI、WAF、object key |
| Firewall | 期待通り遮断しない | route、rule priority、stateful/stateless、logs |
| Audit | 変更者不明 | CloudTrail |
| Config | 現在設定不明 | AWS Config |

## 誤答になりやすい選択肢

| 誤答 | なぜ違うか |
| :--- | :--- |
| 通信不可の原因をCloudTrailだけで調べる | CloudTrailはAPI操作履歴。通信経路やSG/NACLの到達性はReachability AnalyzerやFlow Logs |
| HTTP URLをVPC Flow Logsで調べる | Flow LogsはIP/port/protocol中心。URLやheaderは見えない |
| VPC PeeringでA-B-Cの推移的通信をする | VPC Peeringはtransitive routing不可 |
| NAT Gatewayへpingして疎通確認する | NAT Gatewayはping応答確認の対象ではない |
| SGだけ見て戻り通信を判断する | SGはstateful。戻りで詰まるならNACL、route、firewallも見る |
| Network Firewallで非対称routingでもstateful inspectionできる | stateful機能は対称経路が前提 |
| Reachability Analyzerでアプリ応答を確認する | 静的なネットワーク到達性の分析であり、HTTP応答確認ではない |
| DNS Query LoggingでDNSをブロックする | Query Loggingは記録。ブロックはDNS Firewall |

## 試験前チェックリスト

- 通信不可時にRoute Table、SG、NACL、DNS、戻り経路を順番に確認できる
- VPC Flow Logsの`ACCEPT`、`REJECT`、`NODATA`、`SKIPDATA`を説明できる
- Reachability AnalyzerとVPC Flow Logsの違いを説明できる
- CloudTrailとAWS Configの使い分けを説明できる
- public subnetとprivate subnetのインターネット経路を説明できる
- NAT Gatewayのidle timeout 350秒を覚えている
- VPC Peeringの推移的ルーティング不可を説明できる
- TGW route table association/propagationを説明できる
- Inspection VPCでappliance modeが必要になる理由を説明できる
- VPNのIKE、IPsec、BGPの切り分け順を説明できる
- DXのLayer 1/2/3とBGP確認項目を説明できる
- MTU/DF bit/PMTUD/MSS clampingの関係を説明できる
- ALBの502/503/504の代表原因を説明できる
- CloudFrontの403/502/504の代表原因を説明できる
- DNS障害でPHZ、Resolver endpoint、Resolver rule、TTLを確認できる

## 公式ドキュメント

- [Content Domain 3: Network Management and Operation](https://docs.aws.amazon.com/aws-certification/latest/advanced-networking-specialty-01/advanced-networking-specialty-01-domain3.html)
- [Troubleshoot reachability issues using Reachability Analyzer](https://docs.aws.amazon.com/vpc/latest/userguide/reachability-analyzer.html)
- [Troubleshoot VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-troubleshooting.html)
- [Flow log records](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)
- [Troubleshoot NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html)
- [Troubleshoot a VPC peering connection](https://docs.aws.amazon.com/vpc/latest/peering/troubleshoot-vpc-peering-connections.html)
- [Amazon VPC attachments in AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html)
- [How AWS Transit Gateway works](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)
- [Avoiding asymmetric routing with AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/asymmetric-routing.html)
- [Troubleshooting general issues in AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/troubleshooting-general-issues.html)
- [Troubleshooting AWS Site-to-Site VPN customer gateway device](https://docs.aws.amazon.com/vpn/latest/s2svpn/Troubleshooting.html)
- [Troubleshoot AWS Site-to-Site VPN connectivity when using BGP](https://docs.aws.amazon.com/vpn/latest/s2svpn/Generic_Troubleshooting.html)
- [Troubleshoot Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Troubleshooting.html)
- [Troubleshoot layer 3/4 issues in Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/ts-layer-3.html)
- [Troubleshoot your Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html)
- [Troubleshooting error response status codes in CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/troubleshooting-response-errors.html)
- [What is Route 53 VPC Resolver?](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html)
- [AWS PrivateLink concepts](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)
