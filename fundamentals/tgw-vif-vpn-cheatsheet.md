# TGW / VIF / VPN接続 チートシート

## 目的

ANS-C01で頻出のTransit Gateway（TGW）、Direct ConnectのVIF、Site-to-Site VPN接続を、問題文から判断しやすい形で整理する。

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
| 役割 | DXとVGW/TGW/Cloud WANを中継 |
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

## 公式参照

- [AWS Direct Connect virtual interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)
- [Create a transit virtual interface to the Direct Connect gateway](https://docs.aws.amazon.com/directconnect/latest/UserGuide/create-transit-vif-for-gateway.html)
- [Direct Connect gateways and Transit Gateway associations](https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-transit-gateways.html)
- [Allowed prefixes interactions for Direct Connect gateways](https://docs.aws.amazon.com/directconnect/latest/UserGuide/allowed-to-prefixes.html)
- [Transit Gateway VPC attachments and appliance mode](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-vpc-attachments.html)
- [Static and dynamic routing in AWS Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-static-dynamic.html)
- [Site-to-Site VPN route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)
