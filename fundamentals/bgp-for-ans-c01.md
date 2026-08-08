# BGP基礎とANS-C01対策

## 目的

AWS Certified Advanced Networking - Specialty（ANS-C01）で問われるBGPの基本用語、経路選択、Direct Connect、Site-to-Site VPN、Transit Gatewayとの関係を整理する。

BGPは細かい仕様をすべて暗記するより、**どの経路が優先されるか**、**どの属性で経路を寄せるか**、**AWSサービスごとの制約は何か**を押さえることが重要である。

## BGPとは

BGP（Border Gateway Protocol）は、ネットワーク同士が「どの宛先ネットワークへ、どの経路で到達できるか」を教え合うためのルーティングプロトコルである。

インターネットやAWS Direct Connect、Site-to-Site VPNなどでは、企業ネットワーク、ISP、AWSのような大きな管理単位が、それぞれ経路情報を交換する。この管理単位をAS（Autonomous System）と呼び、ASにはASN（Autonomous System Number）という番号が付く。

BGPはデータそのものを運ぶ仕組みではなく、**経路情報を交換する制御プレーン**である。実際の通信パケットは、BGPで学習した経路に従って通常のIPルーティングで転送される。

```text
オンプレミスルーター
  AS65010
      |
      | BGPで経路交換
      |
AWS側ルーター / VGW / TGW / DXGW
  AS64512 / AS7224 など
```

## 最初に覚える用語

| 用語 | 意味 | 試験での見方 |
| :--- | :--- | :--- |
| BGP | 経路情報を交換するプロトコル | DX、VPN、TGW Connectで出る |
| AS | 1つの管理主体のネットワーク | 会社、ISP、AWSなど |
| ASN | ASに割り当てる番号 | AWS側ASN、Customer Gateway ASN |
| Prefix | 広告するネットワーク範囲 | 例: `10.0.0.0/16` |
| Advertise | 経路を相手に知らせること | 「AWSへ経路を広告する」 |
| Receive | 相手から経路を受け取ること | 「オンプレ経路を受信する」 |
| BGP peer / neighbor | BGPを張る相手 | オンプレルーターとAWS側 |
| eBGP | 異なるAS間のBGP | オンプレASとAWS側AS |
| iBGP | 同じAS内のBGP | TGW Connectで出ることがある |
| AS_PATH | その経路が通ってきたASの並び | 短い方が優先されやすい |
| Local Preference | 自AS内でどの出口を優先するか | 高い方が優先 |
| MED | 隣接ASに入口の優先度を伝える属性 | 低い方が優先 |
| ECMP | 同じ条件の複数経路を同時利用 | Active/Active構成 |
| BGP community | 経路に付けるタグ | DXでLocal Preferenceや広告範囲を制御 |

## 覚える数字

| 数字 | 意味 |
| :--- | :--- |
| TCP `179` | BGPセッションで使うポート |
| `7224` | AWSのASNとしてよく出る番号 |
| `64512-65534` | よく使われる16-bit Private ASN範囲 |
| `4200000000-4294967294` | 32-bit Private ASN範囲 |
| `7224:7100` | Direct ConnectのLow local preference |
| `7224:7200` | Direct ConnectのMedium local preference |
| `7224:7300` | Direct ConnectのHigh local preference |
| `7224:9100` | Public VIFでLocal AWS Regionへ広告 |
| `7224:9200` | Public VIFで同一大陸のAWS Regionsへ広告 |
| `7224:9300` | Public VIFで全Public AWS Regionsへ広告 |

## BGPが出るAWSサービス

| サービス/構成 | BGPの使われ方 |
| :--- | :--- |
| Site-to-Site VPN | 動的ルーティングでオンプレとAWSが経路交換 |
| Direct Connect Private VIF | オンプレとVPC/VGW/DXGW間で経路交換 |
| Direct Connect Public VIF | AWS public service向けにPublic prefixを広告 |
| Direct Connect Transit VIF | DXGW経由でTransit Gatewayへ接続 |
| Transit Gateway VPN attachment | VPN経由でオンプレ経路をTGWへ伝播 |
| Transit Gateway Connect | GREトンネル上でBGPを使い、SD-WAN/仮想アプライアンスと動的経路交換 |

## 基本の経路選択

ANS-C01では、BGP属性だけでなく、AWS側のルートテーブル評価と合わせて考える必要がある。

| 優先観点 | 覚えること |
| :--- | :--- |
| Longest Prefix Match | より具体的なCIDRが優先される |
| Local route | VPCのlocal routeは非常に強い |
| Static route | 同じ宛先なら伝播経路より静的ルートが優先される場面がある |
| Local Preference | 高い方が優先 |
| AS_PATH | 短い方が優先 |
| MED | 低い方が優先。ただしAWSでは優先度が低め |
| ECMP | 属性が同等なら複数経路を同時利用できることがある |
| Tunnel health | VPNではトンネルの正常性が他の属性より優先される |

## Direct Connectで覚えること

Direct ConnectのPrivate VIF/Transit VIFでは、AWS側の戻り経路を制御する問題がよく出る。

### 経路優先順位の考え方

AWS Direct ConnectのPrivate VIF/Transit VIFでは、概ね次の順で考える。

1. Longest Prefix Match
2. Local Preference
3. AS_PATH length
4. MED
5. ECMP

AWS公式では、Private VIF/Transit VIFでActive/Passiveにしたい場合、同じPrefix長ならLocal Preferenceの利用が推奨される。AS_PATHはLocal Preferenceが同じ場合に効く。MEDはさらに優先度が低いため、AWSはMEDへの依存を推奨していない。

### Local Preference community

オンプレからAWSへ広告するPrefixにBGP communityを付けることで、AWSからオンプレへ戻る通信の優先度を制御できる。

| Community | 意味 | 用途 |
| :--- | :--- | :--- |
| `7224:7100` | Low preference | バックアップ回線 |
| `7224:7200` | Medium preference | 標準 |
| `7224:7300` | High preference | メイン回線 |

例:

```text
10Gbps DX:  10.0.0.0/16 + 7224:7300
2Gbps DX:   10.0.0.0/16 + 7224:7100
```

この場合、AWSからオンプレへ戻る通信は10Gbps側を優先する。

### AS_PATH prepend

AS_PATH prependは、自分のAS番号をわざと複数回付けて、経路を「遠く」見せる方法である。

```text
優先したい回線:
AS_PATH: 65010

優先したくない回線:
AS_PATH: 65010 65010 65010
```

AS_PATHが長い経路は優先されにくい。

ただし、Direct ConnectではLocal Preference communityの方が優先度が高い。AWSからオンプレへ戻る通信を制御したい場合は、まずLocal Preferenceを疑う。

### ECMP

複数のPrivate VIFまたはTransit VIFで、Prefix、AS_PATH長、BGP属性が同じ場合、AWSはECMPで負荷分散できる。

Active/Activeにしたい場合:

```text
同じPrefix
同じLocal Preference
同じAS_PATH長
同じMED
```

Active/Passiveにしたい場合:

```text
メイン: 高いLocal Preference
予備:   低いLocal Preference
```

## Public VIFで覚えること

Public VIFは、S3などAWS public serviceへDirect Connect経由で到達するために使う。

| 項目 | 覚えること |
| :--- | :--- |
| AWS側ASN | Public VIFではAWS側ASNとして`7224`が出る |
| 広告Prefix | 自分が所有するPublic IPv4/IPv6 PrefixをBGPで広告する |
| Private ASN | Public VIFでPrivate ASNを使うと、外部向けにはAWS ASNへ置き換えられることがある |
| NO_EXPORT | Public VIFでAWSから広告される経路にはNO_EXPORTが付く |
| 広告範囲 | CommunityでLocal Region/Continent/Globalを制御できる |

### Public VIFのscope community

| Community | 広告範囲 |
| :--- | :--- |
| `7224:9100` | Local AWS Region |
| `7224:9200` | 同一大陸のAWS Regions |
| `7224:9300` | 全Public AWS Regions |
| 付与なし | デフォルトでGlobal |

## Site-to-Site VPNで覚えること

Site-to-Site VPNでは、静的ルーティングと動的ルーティング（BGP）を選べる。

| 項目 | 覚えること |
| :--- | :--- |
| VPN接続 | 1接続に2本のトンネル |
| 動的ルーティング | BGPでオンプレとAWSが経路交換 |
| 静的ルーティング | 手動でPrefixを設定 |
| 冗長化 | 両方のトンネルを設定する |
| 非対称ルーティング | AWSは対応できるCustomer Gatewayを推奨 |
| ECMP | Transit GatewayではVPNのECMPが使える |
| VGW | VGWではSite-to-Site VPNのECMPはサポートされない |

### VPNの経路優先

Virtual Private Gatewayで同じPrefixの経路がある場合、代表的には次の順で優先される。

1. Direct ConnectからBGPで伝播された経路
2. Site-to-Site VPNの静的ルート
3. Site-to-Site VPNからBGPで伝播された経路

同じVPN BGP経路同士では、AS_PATHが短い方が優先される。AS_PATH長が同じで、最初のASも同じ場合、MEDが比較され、低いMEDが優先される。

ただし、VPNではトンネルの正常性が経路属性より優先される。試験では「片方のトンネルだけを無理に固定する」より、「両トンネルを設定し、非対称ルーティングを許容する」設計が正解になりやすい。

## Transit Gatewayで覚えること

Transit Gatewayでは、BGPで受け取った経路がTGW Route Tableへ伝播される。TGWではAssociationとPropagationを分けて考える。

| 用語 | 意味 |
| :--- | :--- |
| Association | AttachmentがどのTGW Route Tableを参照するか |
| Propagation | Attachmentの経路をTGW Route Tableへ登録すること |
| Static route | 手動で追加するTGW経路 |
| Propagated route | VPN/DX/VPCなどから伝播された経路 |

### TGWの経路評価

Transit Gatewayでは、まず最も具体的な宛先Prefixが優先される。同じCIDRで複数のAttachment種別から経路がある場合、静的ルートやVPC伝播、Direct Connect Gateway伝播、VPN伝播などの優先順位がある。

試験対策では、まず次を覚える。

```text
より具体的なPrefixが強い
同じPrefixなら静的ルートが強い
伝播経路はAttachment種別ごとの優先順位がある
```

## Transit Gateway Connectで覚えること

Transit Gateway Connectは、SD-WANアプライアンスなどとTransit Gatewayを接続するための仕組みである。GREトンネルとBGPを使う。

| 項目 | 覚えること |
| :--- | :--- |
| カプセル化 | GRE |
| 経路交換 | BGP |
| Connect peer | GREトンネル |
| BGPセッション | 1つのConnect peerに2つのBGPセッション |
| eBGP | Peer ASNがTGW ASNと異なる場合。TTL 2のebgp-multihopが必要 |
| iBGP | Peer ASNがTGW ASNと同じ場合 |
| MP-BGP | IPv6 Prefix交換に関係する |
| BFD | TGW Connectではサポートされない |
| Graceful Restart | TGW Connectではサポートされない |
| ECMP | 複数Connect peerでAS_PATHとASNが一致すると使える |

## Direct Connect GatewayのAllowed Prefix

Direct Connect Gateway（DXGW）では、Allowed Prefixの扱いがVGW関連とTGW関連で違う。

| 関連付け先 | Allowed Prefixの意味 |
| :--- | :--- |
| VGW association | VPC CIDRを広告するためのフィルター。VPC CIDRと同じか、より広いPrefixが必要 |
| TGW association | Allowed Prefixに入力したPrefix自体がオンプレへ広告される |

例:

```text
VPC CIDR: 10.0.0.0/16
Allowed Prefix: 10.0.0.0/15
```

VGW associationでは、`10.0.0.0/16` が広告される。

```text
TGW association
Allowed Prefix: 10.0.0.0/8
```

TGW associationでは、`10.0.0.0/8` が広告される。

複数TGWをDXGWへ関連付ける場合、Allowed Prefixの重複はできない。

## Active/ActiveとActive/Passive

### Active/Active

複数回線を同時に使いたい構成。

```text
同じPrefix
同じLocal Preference
同じAS_PATH長
同じMED
ECMP有効
```

試験キーワード:

- 負荷分散
- 複数DXを同時利用
- ECMP
- 同じBGP属性

### Active/Passive

片方を主系、もう片方を待機系にしたい構成。

```text
主系: 高いLocal Preference
待機: 低いLocal Preference
```

または、

```text
主系: 短いAS_PATH
待機: 長いAS_PATH prepend
```

試験キーワード:

- バックアップ回線
- 優先経路
- フェイルオーバー
- Local Preference
- AS_PATH prepend

## 方向を間違えない

BGP問題では「どちら向きの通信を制御したいか」が重要である。

| 制御したい向き | 代表的な方法 |
| :--- | :--- |
| AWSからオンプレへ戻る通信 | AWSが受け取るPrefixにLocal Preference communityを付ける |
| オンプレからAWSへ向かう通信 | オンプレ側ルーターのBGPポリシーで経路選択する |
| バックアップ回線にしたい | 低いLocal Preference、またはAS_PATH prepend |
| 負荷分散したい | 同じPrefix、同じ属性でECMP |
| 特定宛先だけ別経路にしたい | より具体的なPrefixを広告する |

## よくある誤解

| 誤解 | 正しい理解 |
| :--- | :--- |
| BGPは帯域が大きい回線を自動で選ぶ | BGPは帯域ではなく属性で経路を選ぶ |
| AS_PATH prependすれば常に勝てる | Local Preferenceの方が優先される場面がある |
| MEDを使えば確実に制御できる | AWSではMEDは優先度が低く、依存しすぎない |
| VPNの片トンネルを固定すればよい | 両トンネルを設定し、非対称ルーティングを許容する設計が推奨されやすい |
| Static routeとBGP routeは同じ扱い | AWSのルートテーブルでは静的ルートが優先される場面がある |
| DXGWのAllowed Prefixは常にフィルター | VGW associationとTGW associationで意味が違う |
| ECMPはどこでも使える | TGWでは使える場面があるが、VGWのSite-to-Site VPNでは使えない |

## 試験での見抜き方

| 問題文の表現 | 疑う答え |
| :--- | :--- |
| 2本のDXをActive/Activeで使いたい | 同じPrefix、同じBGP属性、ECMP |
| 低速回線をバックアップにしたい | 低いLocal Preference、またはAS_PATH prepend |
| AWSからオンプレへの戻り経路を制御したい | Direct Connect local preference community |
| より具体的な宛先だけ別経路にしたい | Longest Prefix Match |
| VPNで片方のトンネルだけ通信が落ちる | 非対称ルーティング、MED、両トンネル設定 |
| SD-WANアプライアンスとTGWを動的接続 | Transit Gateway Connect、GRE、BGP |
| DXGWからTGWへ広いPrefixを広告したい | TGW associationのAllowed Prefix |
| VPC CIDRをDXGW経由で広告したい | VGW associationのAllowed Prefixはフィルター |

## 暗記チェック

- [ ] BGPはTCP `179` を使う
- [ ] ASはネットワークの管理単位、ASNはその番号
- [ ] Longest Prefix MatchはBGP属性より先に効く場面が多い
- [ ] Local Preferenceは高い方が優先
- [ ] AS_PATHは短い方が優先
- [ ] MEDは低い方が優先
- [ ] Direct ConnectのLocal Preference communityは `7224:7100`, `7224:7200`, `7224:7300`
- [ ] Active/Activeは同じPrefix・同じ属性・ECMP
- [ ] Active/PassiveはLocal Preference差、またはAS_PATH prepend
- [ ] Site-to-Site VPNは1接続2トンネル
- [ ] TGW ConnectはGRE + BGP
- [ ] TGW ConnectのConnect peerは2つのBGPセッションを持つ
- [ ] DXGWのAllowed PrefixはVGW associationとTGW associationで意味が違う

## 公式参照

- [Direct Connect routing policies and BGP communities](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)
- [Direct Connect virtual interfaces and hosted virtual interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)
- [Allowed prefixes interactions for Direct Connect gateways](https://docs.aws.amazon.com/directconnect/latest/UserGuide/allowed-to-prefixes.html)
- [AWS Site-to-Site VPN routing options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNRoutingTypes.html)
- [Route tables and AWS Site-to-Site VPN route priority](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-route-priority.html)
- [Customer gateway options for Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-options.html)
- [How AWS Transit Gateway works](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)
- [Transit Gateway Connect attachments and Connect peers](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-connect.html)
