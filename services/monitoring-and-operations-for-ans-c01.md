# Monitoring and Operations ANS-C01対策

## 目的

ANS-C01で出題されるAWSリソースの監視、ログ、監査、運用、トラブルシューティング系サービスを、問題文から正しいサービスを選べる形で整理する。

対象は主に次のサービスと機能。

```text
Amazon CloudWatch
CloudWatch Metrics
CloudWatch Alarms
CloudWatch Logs
CloudWatch Logs Insights
CloudWatch Metric Filters
CloudWatch Dashboards
Amazon CloudWatch Internet Monitor
Amazon CloudWatch Network Monitor
AWS CloudTrail
AWS Config
VPC Flow Logs
Route 53 Resolver Query Logging
Elastic Load Balancing Access Logs
CloudFront Logs
VPC Reachability Analyzer
VPC Network Access Analyzer
VPC Traffic Mirroring
AWS Network Manager
AWS Health
Amazon EventBridge
AWS Systems Manager
```

## 最初に覚える結論

| 要件 | 疑うサービス/機能 |
| :--- | :--- |
| メトリクス、しきい値、アラーム、ダッシュボード | CloudWatch Metrics / Alarms / Dashboards |
| ログを集約、検索、可視化したい | CloudWatch Logs / Logs Insights |
| ログの文字列をメトリクス化してアラームにしたい | CloudWatch Logs Metric Filter |
| 誰が、いつ、どのAPIを実行したか確認したい | CloudTrail |
| Security GroupやRoute Tableなどの変更履歴を見たい | CloudTrail + AWS Config |
| リソース設定がルールに準拠しているか確認したい | AWS Config Rules |
| 複数アカウント/複数リージョンの準拠状態を集約したい | AWS Config Aggregator |
| VPC内通信のACCEPT/REJECT、送信元/宛先IP、portを見たい | VPC Flow Logs |
| パケットの中身をIDSへコピーして解析したい | VPC Traffic Mirroring |
| DNS問い合わせを記録したい | Route 53 Resolver Query Logging |
| DNS問い合わせをブロックしたい | Route 53 Resolver DNS Firewall |
| ALB/CloudFrontへのHTTPアクセスログを見たい | ALB Access Logs / CloudFront Logs |
| 送信元から宛先まで通信可能か静的に確認したい | VPC Reachability Analyzer |
| 意図しない外部公開やネットワーク到達性を検出したい | VPC Network Access Analyzer |
| TGW、Cloud WAN、VPN、Direct Connectなどを一元的に可視化したい | AWS Network Manager |
| インターネット経路や利用者影響を監視したい | CloudWatch Internet Monitor |
| DX/VPNなどハイブリッド接続のネットワーク性能を監視したい | CloudWatch Network Monitor |
| AWS側の障害、メンテナンス、アカウント固有イベントを知りたい | AWS Health |
| イベントを契機に通知、Lambda、SSM Automationを起動したい | EventBridge |
| 運用手順や自動復旧を実行したい | Systems Manager Automation / OpsCenter |

## 全体像

```text
AWS Resources
  |
  +--> Metrics
  |      +--> CloudWatch Metrics
  |      +--> CloudWatch Alarms
  |      +--> CloudWatch Dashboards
  |
  +--> Logs
  |      +--> CloudWatch Logs
  |      +--> Logs Insights
  |      +--> Metric Filters
  |
  +--> API Audit
  |      +--> CloudTrail
  |      +--> CloudTrail Lake / S3 / CloudWatch Logs
  |
  +--> Configuration State
  |      +--> AWS Config
  |      +--> Config Rules
  |      +--> Aggregator
  |
  +--> Network Visibility
  |      +--> VPC Flow Logs
  |      +--> Resolver Query Logs
  |      +--> Reachability Analyzer
  |      +--> Network Access Analyzer
  |      +--> Traffic Mirroring
  |
  +--> Operations
         +--> EventBridge
         +--> Systems Manager
         +--> AWS Health
```

## 監視を5種類に分ける

ANS-C01では、監視対象を次の5種類に分けると判断しやすい。

| 種類 | 見たいもの | 代表サービス |
| :--- | :--- | :--- |
| メトリクス監視 | 数値、しきい値、異常傾向 | CloudWatch Metrics / Alarms |
| ログ監視 | アプリログ、OSログ、サービスログの検索 | CloudWatch Logs / Logs Insights |
| 監査ログ | API操作、変更者、変更時刻 | CloudTrail |
| 構成監視 | 現在設定、変更履歴、準拠/非準拠 | AWS Config |
| ネットワーク可視化 | 通信可否、経路、フロー、DNS問い合わせ | VPC Flow Logs / Reachability Analyzer / Resolver Query Logs |

覚え方:

```text
CloudWatch = 数値とログを見る
CloudTrail = API操作の証跡を見る
AWS Config = リソース設定の状態と準拠を見る
VPC Flow Logs = IP通信の流れを見る
Reachability Analyzer = 通れる設計か確認する
```

## Amazon CloudWatch

CloudWatchは、AWSリソースとアプリケーションのメトリクス、ログ、アラーム、ダッシュボードを扱う監視サービスである。

試験での見方:

| 問題文 | 選ぶもの |
| :--- | :--- |
| CPU、NetworkIn、TargetResponseTimeなどを監視 | CloudWatch Metrics |
| 閾値を超えたら通知 | CloudWatch Alarm + SNS |
| 複数メトリクスをまとめて可視化 | CloudWatch Dashboard |
| ログを検索したい | CloudWatch Logs Insights |
| ログの特定文字列を数値化したい | Metric Filter |
| 複数アラームを組み合わせたい | Composite Alarm |
| 通常と違う傾向を検出したい | Anomaly Detection |

## CloudWatch Metrics

CloudWatch Metricsは、時系列の数値データである。

代表例:

```text
EC2 CPUUtilization
EC2 NetworkIn / NetworkOut
ALB RequestCount
ALB TargetResponseTime
ALB HTTPCode_ELB_5XX_Count
NAT Gateway BytesOutToDestination
NAT Gateway ErrorPortAllocation
Transit Gateway BytesIn / BytesOut
VPN TunnelState
VPN TunnelDataIn / TunnelDataOut
Direct Connect ConnectionState
CloudFront 4xxErrorRate / 5xxErrorRate
Route 53 HealthCheckStatus
```

重要用語:

| 用語 | 意味 |
| :--- | :--- |
| Namespace | メトリクスの分類。例: `AWS/EC2`, `AWS/ApplicationELB` |
| Metric name | メトリクス名。例: `CPUUtilization` |
| Dimension | メトリクスを識別する属性。例: InstanceId、LoadBalancer |
| Statistic | Average、Sum、Minimum、Maximumなどの集計方法 |
| Period | 評価間隔。例: 1分、5分 |
| Datapoint | ある期間のメトリクス値 |

## CloudWatch Alarms

CloudWatch Alarmは、メトリクスが条件を満たしたときに状態を変える。

状態:

| 状態 | 意味 |
| :--- | :--- |
| OK | しきい値を超えていない |
| ALARM | しきい値を超えている |
| INSUFFICIENT_DATA | 評価に必要なデータが不足している |

よく出る設定:

| 設定 | 意味 |
| :--- | :--- |
| Threshold | しきい値 |
| Evaluation periods | 何期間分を評価するか |
| Datapoints to alarm | 何回条件を満たしたらALARMにするか |
| Treat missing data | 欠落データの扱い |
| Alarm action | SNS通知、Auto Scaling、EC2 actionなど |

試験で重要な考え方:

```text
CloudWatch Alarmは、基本的にメトリクスに対して作る。
ログそのものに直接アラームを作るのではなく、
ログをMetric Filterでメトリクス化してからAlarmにする。
```

## 欠落データの扱い

CloudWatch Alarmでは、データが来ない時間帯をどう扱うかを選ぶ。

| 設定 | 意味 | 使いどころ |
| :--- | :--- | :--- |
| notBreaching | しきい値未満として扱う | イベント発生時だけ出るメトリクス |
| breaching | しきい値超過として扱う | データ欠落自体を障害とみなす |
| ignore | 前回状態を維持する | 一時的な欠落で状態を変えたくない |
| missing | 不足データとして扱う | 明示的にデータ不足を見たい |

例:

```text
SecurityGroupChangeCount のように変更時だけ出るメトリクス
  -> 欠落データは notBreaching が自然

VPN TunnelState のように常時監視したいメトリクス
  -> 欠落データをどう扱うか慎重に設計する
```

## CloudWatch Logs

CloudWatch Logsは、ログイベントを保存、検索、フィルタリング、転送するサービスである。

基本用語:

| 用語 | 意味 |
| :--- | :--- |
| Log group | ログの入れ物。保持期間やKMS暗号化を設定する単位 |
| Log stream | 同じLog group内のログ発生元単位 |
| Log event | 1件のログ本文とタイムスタンプ |
| Retention | ログ保持期間 |
| Subscription filter | ログをLambda、Kinesis Data Streams、Firehoseなどへ転送する設定 |
| Metric filter | ログから特定パターンを検出してメトリクスを作る設定 |

試験での見方:

| 問題文 | 選ぶもの |
| :--- | :--- |
| ログを横断検索したい | CloudWatch Logs Insights |
| ログに特定文字列が出たら通知したい | Metric Filter + CloudWatch Alarm + SNS |
| ログをS3や分析基盤へ送信したい | Subscription filter / Firehose |
| ログ保持期間を制御したい | Log group retention |

## CloudWatch Logs Insights

Logs Insightsは、CloudWatch Logs内のログをクエリで検索・集計する機能である。

例:

```sql
fields @timestamp, @message
| filter eventSource = "ec2.amazonaws.com"
| filter eventName like /AuthorizeSecurityGroup/
| sort @timestamp desc
| limit 20
```

用途:

```text
障害調査
APIイベントの絞り込み
アプリケーションエラーの検索
VPC Flow Logsの集計
```

注意:

```text
Logs Insights = 調査・分析
Metric Filter = 検知・アラーム化
```

## CloudWatch Logs Metric Filter

Metric Filterは、ログイベントが条件に一致したときにCloudWatchメトリクスを生成する。

典型構成:

```text
CloudTrail
  -> CloudWatch Logs
  -> Metric Filter
  -> CloudWatch Metric
  -> CloudWatch Alarm
  -> SNS通知
```

Security Group変更検知の例:

```text
AuthorizeSecurityGroupIngress
AuthorizeSecurityGroupEgress
RevokeSecurityGroupIngress
RevokeSecurityGroupEgress
CreateSecurityGroup
DeleteSecurityGroup
```

重要な注意:

```text
Metric Filterは作成後に取り込まれたログからメトリクスを作る。
過去ログにさかのぼってメトリクスを作る用途ではない。
```

## AWS CloudTrail

CloudTrailは、AWSアカウント内のAPI操作を記録する監査サービスである。

CloudTrailで見るもの:

| フィールド | 意味 |
| :--- | :--- |
| eventTime | API実行時刻 |
| eventSource | APIのサービス。例: `ec2.amazonaws.com` |
| eventName | API名。例: `AuthorizeSecurityGroupIngress` |
| userIdentity | 実行主体。IAM user、AssumedRole、AWS serviceなど |
| sourceIPAddress | 接続元IP、またはAWSサービス名 |
| awsRegion | APIが実行されたリージョン |
| requestParameters | APIに渡した値 |
| responseElements | APIの応答 |
| errorCode | 失敗時のエラーコード |

## CloudTrailのイベント種別

| 種別 | 内容 | 試験での見方 |
| :--- | :--- | :--- |
| Management events | コントロールプレーン操作。作成、変更、削除、参照 | ほぼ必須。Security Group変更、Route Table変更など |
| Data events | S3 object、Lambda Invokeなどデータプレーン操作 | 大量になりやすく、明示的に有効化が必要 |
| Insights events | 異常なAPIアクティビティ | 通常と違うAPI頻度を検出 |

CloudTrailの出力先:

```text
Event history
S3 bucket
CloudWatch Logs
CloudTrail Lake
```

試験で重要な判断:

| 要件 | 答え |
| :--- | :--- |
| 過去90日程度の管理イベントをコンソールで確認したい | CloudTrail Event history |
| 長期保存、証跡保全、監査証跡 | CloudTrail trail -> S3 |
| API操作を検知してアラーム通知 | CloudTrail -> CloudWatch Logs -> Metric Filter -> Alarm |
| 複数アカウントの証跡を中央管理 | Organization trail |
| 全リージョンのAPI操作を記録 | Multi-region trail |

## CloudTrailとCloudWatchの違い

| 観点 | CloudTrail | CloudWatch |
| :--- | :--- | :--- |
| 主目的 | API監査 | メトリクス/ログ監視 |
| 見るもの | 誰が、いつ、何のAPIを実行したか | リソース状態、数値、ログ |
| 例 | Security Groupを誰が変更したか | ALB 5xxが増えたか |
| アラーム | 直接ではなくCloudWatch Logs連携が一般的 | CloudWatch Alarmで実施 |

覚え方:

```text
CloudTrail = 操作履歴
CloudWatch = 監視
```

## AWS Config

AWS Configは、AWSリソースの設定履歴、構成変更、準拠状況を記録・評価するサービスである。

CloudTrailとの違い:

| 観点 | CloudTrail | AWS Config |
| :--- | :--- | :--- |
| 見るもの | API操作イベント | リソース設定の状態 |
| 得意な質問 | 誰が変更したか | 現在どう設定されているか |
| 例 | `AuthorizeSecurityGroupIngress`を実行したIAM Role | Security Groupに`0.0.0.0/0`があるか |
| 履歴 | APIイベント履歴 | 構成履歴 |
| 準拠評価 | 主目的ではない | Config Rulesで評価 |

AWS Configの構成要素:

| 用語 | 意味 |
| :--- | :--- |
| Configuration recorder | リソース設定を記録する仕組み |
| Configuration item | ある時点のリソース設定情報 |
| Delivery channel | 設定履歴をS3などへ配信する経路 |
| Config rule | 設定がルールに準拠しているか評価する |
| Managed rule | AWS管理のConfig Rule |
| Custom rule | Lambdaなどで作る独自ルール |
| Conformance pack | 複数ルールと修復設定のパッケージ |
| Aggregator | 複数アカウント/リージョンのConfig結果を集約 |
| Remediation | 非準拠時にSSM Automationなどで修復 |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| Security Groupがポリシー違反か継続評価 | AWS Config Rule |
| 複数アカウントの設定準拠を集約 | AWS Config Aggregator |
| ルールセットをまとめて配布 | Conformance Packs |
| 非準拠を自動修復 | Config Remediation + Systems Manager Automation |
| あるリソースの設定変更履歴を追跡 | AWS Config timeline |

## VPC Flow Logs

VPC Flow Logsは、VPC、Subnet、ENIを通過するIPトラフィックのメタデータを記録する機能である。

取得できる代表情報:

```text
version
account-id
interface-id
srcaddr
dstaddr
srcport
dstport
protocol
packets
bytes
start
end
action
log-status
```

読み方の例:

```text
2 123456789012 eni-abc123 10.0.1.10 10.0.2.20 51524 443 6 10 8400 1720000000 1720000060 ACCEPT OK
```

| フィールド | 意味 |
| :--- | :--- |
| `eni-abc123` | 対象ENI |
| `10.0.1.10` | 送信元IP |
| `10.0.2.20` | 宛先IP |
| `51524` | 送信元port |
| `443` | 宛先port |
| `6` | TCP |
| `ACCEPT` | SG/NACLで許可された通信 |
| `REJECT` | SG/NACLで拒否された通信 |
| `OK` | ログ記録成功 |

注意:

```text
VPC Flow Logsはパケットキャプチャではない。
HTTPヘッダー、URL、payloadの中身は見えない。
通信の結果がアプリケーションとして成功したかまでは分からない。
```

出力先:

```text
CloudWatch Logs
Amazon S3
Kinesis Data Firehose
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| Security Group/NACLで拒否された通信を調べたい | VPC Flow Logs |
| どのIPからどのIPへ通信しているか見たい | VPC Flow Logs |
| VPC内の通信量を確認したい | VPC Flow Logs |
| パケット本文をIDSで解析したい | VPC Traffic Mirroring |
| URLやHTTP headerを見たい | ALB/CloudFront logs、WAF logs、アプリログ |

## ACCEPTとREJECT

VPC Flow Logsの`action`には、主に`ACCEPT`または`REJECT`が入る。

| action | 意味 |
| :--- | :--- |
| ACCEPT | Security GroupまたはNetwork ACLで許可された |
| REJECT | Security GroupまたはNetwork ACLで拒否された |

注意:

```text
ACCEPT = アプリケーションが正常応答した、ではない。
REJECT = SG/NACLで落ちた可能性が高い。
タイムアウトやアプリエラーは別ログと合わせて見る。
```

## VPC Traffic Mirroring

VPC Traffic Mirroringは、ENIのトラフィックを別の監視アプライアンスへコピーする機能である。

用途:

```text
IDS
パケット解析
フォレンジック
詳細な通信解析
```

VPC Flow Logsとの違い:

| 観点 | VPC Flow Logs | VPC Traffic Mirroring |
| :--- | :--- | :--- |
| 見えるもの | IP/port/action/bytesなどのメタデータ | パケット内容を含むコピー |
| 主目的 | 通信の有無、許可/拒否、傾向確認 | IDS/詳細解析 |
| ブロック可否 | ブロックしない | ミラーのみ。単体ではブロックしない |
| コスト/データ量 | 比較的扱いやすい | 大量になりやすい |

試験での見方:

```text
IDSとして監視 = Traffic Mirroring + IDS appliance
IPSとしてインライン遮断 = Network Firewall または GWLB + security appliance
```

## Route 53 Resolver Query Logging

Route 53 Resolver Query Loggingは、VPC内リソースから発生するDNSクエリを記録する機能である。

用途:

```text
どのEC2がどのドメインを名前解決したか調べる
不審ドメインへの問い合わせを可視化する
DNSトラブルシュート
DNS exfiltrationの調査
```

出力先:

```text
CloudWatch Logs
Amazon S3
Kinesis Data Firehose
```

DNS Firewallとの違い:

| 観点 | Resolver Query Logging | Resolver DNS Firewall |
| :--- | :--- | :--- |
| 目的 | DNS問い合わせを記録する | DNS問い合わせを許可/ブロックする |
| 動作 | 可視化 | 制御 |
| 例 | `bad.example.com`へ問い合わせがあったか確認 | `bad.example.com`への問い合わせをブロック |

## ELB Access Logs

Elastic Load BalancingのAccess Logsは、ロードバランサーに届いたリクエストの詳細を記録する。

ALBで見やすいもの:

```text
client IP
target IP
request processing time
target processing time
response code
request line
user agent
TLS information
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| ALBの5xx増加原因を調べたい | CloudWatch Metrics + ALB Access Logs |
| 特定クライアントIPのHTTPアクセスを追いたい | ALB Access Logs |
| URL pathごとのアクセス状況を見たい | ALB Access Logs |
| NLBの接続情報を記録したい | NLB Access Logs |

## CloudFront Logs

CloudFront Logsは、エッジロケーションで受けたリクエスト情報を記録する。

代表:

| 種類 | 用途 |
| :--- | :--- |
| Standard logs | S3へ配信する基本的なアクセスログ |
| Real-time logs | 低遅延でKinesis Data Streamsへ配信 |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 世界中の閲覧者アクセスを分析 | CloudFront Logs |
| リアルタイム性の高いログ分析 | CloudFront Real-time logs |
| CloudFrontの4xx/5xx率を監視 | CloudWatch Metrics |
| Web攻撃ログを確認 | AWS WAF logs |

## VPC Reachability Analyzer

Reachability Analyzerは、VPC内の送信元から宛先までネットワーク的に到達可能かを静的に分析するサービスである。

確認対象の例:

```text
EC2 -> EC2
EC2 -> Internet Gateway
EC2 -> NAT Gateway
EC2 -> VPC Endpoint
EC2 -> Transit Gateway
```

見てくれる要素:

```text
Route Table
Security Group
Network ACL
Internet Gateway
NAT Gateway
VPC Peering
Transit Gateway
VPC Endpoint
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 通信できない原因を経路・SG・NACLから調べたい | Reachability Analyzer |
| 実際のパケットを送らずに経路可否を確認 | Reachability Analyzer |
| VPC設計上、宛先まで到達可能か確認 | Reachability Analyzer |

注意:

```text
Reachability Analyzerはネットワーク設定の到達性を分析する。
アプリケーションが起動しているか、認証が通るか、HTTP 200が返るかは確認しない。
```

## VPC Network Access Analyzer

Network Access Analyzerは、意図しないネットワークアクセスを検出するための分析サービスである。

Reachability Analyzerとの違い:

| 観点 | Reachability Analyzer | Network Access Analyzer |
| :--- | :--- | :--- |
| 主な質問 | AからBへ到達できるか | 望まない経路が存在しないか |
| 使い方 | 特定パスを指定して分析 | アクセス要件を定義して検出 |
| 例 | EC2からRDSへTCP 5432で到達可能か | Internetから到達可能な管理ポートがないか |
| 目的 | トラブルシュート | ガバナンス/監査/リスク検出 |

試験での見方:

```text
通信できない原因調査 = Reachability Analyzer
意図しない公開や到達性の検出 = Network Access Analyzer
```

## AWS Network Manager

AWS Network Managerは、AWS上およびハイブリッドネットワークを管理・可視化するサービスである。

対象になりやすいもの:

```text
Transit Gateway
AWS Cloud WAN
Site-to-Site VPN
Direct Connect
オンプレミス拠点
グローバルネットワーク
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 複数リージョンのTGWを一元的に可視化 | AWS Network Manager |
| グローバルネットワークのトポロジーを可視化 | AWS Network Manager |
| VPN/DX/TGWの運用状態を確認 | AWS Network Manager + CloudWatch |
| Cloud WANの中核的な管理 | AWS Network Manager |

## Route Analyzer

Route Analyzerは、Transit Gateway Route Tableの経路を分析する機能である。

確認できること:

```text
TGW attachment
TGW route table association
TGW route propagation
static route / propagated route
blackhole route
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| TGW Route Table内でどのAttachmentへ転送されるか確認したい | Route Analyzer |
| Association/Propagationの結果として経路があるか確認したい | Route Analyzer |
| VPC内のSecurity GroupやNACLまで含めて到達性を見たい | Reachability Analyzer |

注意:

```text
Route Analyzer = TGW内の経路分析
Reachability Analyzer = VPC内の到達性分析
Flow Logs = 実際に発生した通信ログ
```

## CloudWatch Internet Monitor

CloudWatch Internet Monitorは、インターネット経由でアプリケーションへ到達する利用者体験を監視するサービスである。

主な観点:

```text
availability
performance
client location
ASN
internet path
AWS edge / region
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 世界中の利用者から見たインターネット性能を知りたい | CloudWatch Internet Monitor |
| 特定地域/ISPからの遅延や可用性低下を検知 | CloudWatch Internet Monitor |
| CloudFront/ALB/NLB/WorkSpacesなどのインターネット影響を分析 | CloudWatch Internet Monitor |

## CloudWatch Network Monitor

CloudWatch Network Monitorは、AWSとオンプレミス間のハイブリッドネットワーク性能を監視するためのサービスである。

出題された場合の見方:

| 問題文 | 答え |
| :--- | :--- |
| Direct ConnectやVPNのネットワーク性能を継続監視 | CloudWatch Network Monitor |
| ハイブリッド接続の遅延、packet loss、可用性を見たい | CloudWatch Network Monitor |
| オンプレミスとの接続品質をCloudWatchで可視化 | CloudWatch Network Monitor |

関連:

```text
VPN TunnelState = VPNトンネルのUp/Down監視
DX ConnectionState = Direct Connect接続状態監視
Network Monitor = ハイブリッド経路の性能監視
```

## AWS Health

AWS Healthは、AWSサービス側のイベント、障害、メンテナンス、アカウント固有の影響を確認するサービスである。

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| AWS側の障害やメンテナンス通知を知りたい | AWS Health |
| 自アカウントのリソースに影響するイベントを知りたい | AWS Health Dashboard |
| AWS Healthイベントを運用通知へ流したい | AWS Health + EventBridge |

CloudWatchとの違い:

```text
CloudWatch = 自分のリソースやアプリのメトリクス/ログ
AWS Health = AWSサービス側やアカウント固有の運用イベント
```

## Trusted Advisor / Well-Architected Tool

Trusted AdvisorとWell-Architected Toolは、日々の通信ログというより、運用レビューと改善に使う。

| サービス | 役割 | ネットワークでの見方 |
| :--- | :--- | :--- |
| Trusted Advisor | AWS環境の健全性チェック | サービスクォータ、冗長化、セキュリティ、コスト、耐障害性 |
| Well-Architected Tool | フレームワークに沿った設計レビュー | 信頼性、性能効率、運用、セキュリティ、コストの改善計画 |

試験での見方:

- 「サービス制限に近い」「クォータ確認」「冗長化不足を見たい」ならTrusted Advisorを疑う
- 「設計をベストプラクティスに照らしてレビュー」「継続改善」ならWell-Architected Toolを疑う
- 個別の通信ログやAPI操作履歴を見たい場合は、Flow LogsやCloudTrailを選ぶ

## Amazon EventBridge

EventBridgeは、AWSサービスやSaaS、独自アプリから発生するイベントをルールで受け取り、ターゲットへ配送するサービスである。

監視・運用での使い方:

```text
CloudTrail API event
  -> EventBridge rule
  -> SNS / Lambda / Systems Manager Automation

AWS Health event
  -> EventBridge rule
  -> SNS / Chatbot / Lambda

AWS Config compliance change
  -> EventBridge rule
  -> remediation workflow
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| API操作を契機に自動処理したい | EventBridge + CloudTrail event |
| AWS Healthイベントで通知したい | EventBridge |
| スケジュール実行したい | EventBridge Scheduler / rule |
| 非準拠検出後に修復処理を起動 | EventBridge + Lambda/SSM |

## AWS Systems Manager

Systems Managerは、EC2やオンプレミスサーバーを含む運用管理サービスである。

ANS-C01では深掘りされにくいが、運用自動化の選択肢として出ることがある。

関連機能:

| 機能 | 用途 |
| :--- | :--- |
| Automation | 定型運用手順や修復処理を自動化 |
| Run Command | インスタンスへコマンド実行 |
| Patch Manager | パッチ適用管理 |
| Inventory | サーバー情報収集 |
| OpsCenter | 運用イベントや対応状況を集約 |
| Parameter Store | 設定値や秘密情報の管理 |

ネットワーク試験での見方:

```text
AWS Configで非準拠検出
  -> Systems Manager Automationで修復

CloudWatch Alarm発火
  -> Systems Manager OpsItem作成

多数のEC2へ設定確認コマンド
  -> Systems Manager Run Command
```

## 代表的なネットワーク系メトリクス

### NAT Gateway

| メトリクス | 見ること |
| :--- | :--- |
| BytesInFromSource / BytesOutToDestination | private subnetからインターネット向けの通信量 |
| ErrorPortAllocation | ポート枯渇の疑い |
| PacketsDropCount | パケットドロップ |
| ActiveConnectionCount | アクティブ接続数 |

試験での見方:

```text
private subnetから外部通信が遅い/失敗
  -> NAT Gatewayメトリクス
  -> VPC Flow Logs
  -> Route Table
  -> NACL/SG
```

### Site-to-Site VPN

| メトリクス | 見ること |
| :--- | :--- |
| TunnelState | トンネルUp/Down |
| TunnelDataIn / TunnelDataOut | トンネル通信量 |

試験での見方:

```text
VPN冗長化では2本のトンネルを使う。
TunnelStateをCloudWatch Alarmで監視する。
BGPを使う場合は経路広告と優先制御も確認する。
```

### Direct Connect

| メトリクス | 見ること |
| :--- | :--- |
| ConnectionState | DX接続状態 |
| ConnectionBpsIngress / ConnectionBpsEgress | 帯域使用量 |
| ConnectionPpsIngress / ConnectionPpsEgress | packet per second |
| ConnectionLightLevelTx / Rx | 光レベル |

試験での見方:

```text
DX接続の状態や帯域を監視 = CloudWatch Metrics
DXロケーションや回線冗長設計 = Direct Connect Resiliency Toolkitなど設計観点
```

### Transit Gateway

| メトリクス | 見ること |
| :--- | :--- |
| BytesIn / BytesOut | TGW attachmentの通信量 |
| PacketsIn / PacketsOut | packet数 |
| PacketDropCountBlackhole | blackhole routeによるdrop |
| PacketDropCountNoRoute | routeなしによるdrop |

試験での見方:

```text
TGW経由通信が落ちる
  -> TGW route table
  -> attachment association / propagation
  -> appliance mode
  -> VPC route table
  -> SG/NACL
  -> TGWメトリクス
```

### Elastic Load Balancing

| メトリクス | 見ること |
| :--- | :--- |
| RequestCount | リクエスト数 |
| TargetResponseTime | ターゲット応答時間 |
| HTTPCode_ELB_5XX_Count | LB側の5xx |
| HTTPCode_Target_5XX_Count | ターゲット側の5xx |
| HealthyHostCount / UnHealthyHostCount | ターゲット正常性 |

試験での見方:

```text
ALBの障害調査
  -> CloudWatch Metricsで傾向確認
  -> ALB Access Logsでリクエスト詳細確認
  -> Target Group health check確認
```

## よく出る比較

### CloudWatch vs CloudTrail vs Config vs Flow Logs

| 質問 | 選ぶサービス |
| :--- | :--- |
| CPUが高いか | CloudWatch |
| ALB 5xxが増えたか | CloudWatch |
| 誰がSecurity Groupを変更したか | CloudTrail |
| Security Groupが現在どんな設定か | AWS Config |
| Security Groupが社内ルール違反か | AWS Config Rules |
| どのIP通信が拒否されたか | VPC Flow Logs |
| DNSでどのドメインを引いたか | Route 53 Resolver Query Logging |
| EC2からRDSへ到達可能か | Reachability Analyzer |
| Internetから管理ポートへ到達できてしまうか | Network Access Analyzer |
| TGW Route Table上で経路があるか | Route Analyzer |
| クォータや冗長化の健全性を確認したい | Trusted Advisor |
| 設計全体をベストプラクティスでレビューしたい | Well-Architected Tool |

### Flow Logs vs ALB Logs vs CloudTrail

| ログ | 見えるもの | 見えないもの |
| :--- | :--- | :--- |
| VPC Flow Logs | IP、port、protocol、ACCEPT/REJECT、bytes | URL、HTTP header、payload |
| ALB Access Logs | HTTPリクエスト、client、target、status code、処理時間 | AWS API操作履歴 |
| CloudTrail | AWS API操作、実行主体、時刻 | 通信payload、アプリログ |

### Reachability Analyzer vs Flow Logs

| 観点 | Reachability Analyzer | VPC Flow Logs |
| :--- | :--- | :--- |
| 分析対象 | 設定上の到達可能性 | 実際に発生した通信 |
| パケット送信 | しない | 実通信のログ |
| 主な用途 | 事前確認、設計確認、疎通不可の原因調査 | 通信履歴、拒否確認、傾向分析 |

### Metric Filter vs Logs Insights

| 観点 | Metric Filter | Logs Insights |
| :--- | :--- | :--- |
| 目的 | ログからメトリクスを作りアラーム化 | ログを検索・集計して調査 |
| 常時監視 | 向いている | クエリ実行時の分析向け |
| 過去ログ | 作成前のログにはメトリクスを作らない | 保存済みログを検索できる |

## よくある問題文パターン

### Security Group変更を通知したい

選ぶ構成:

```text
CloudTrail management events
  -> CloudWatch Logs
  -> Metric Filter
  -> CloudWatch Alarm
  -> SNS
```

見るイベント例:

```text
AuthorizeSecurityGroupIngress
AuthorizeSecurityGroupEgress
RevokeSecurityGroupIngress
RevokeSecurityGroupEgress
CreateSecurityGroup
DeleteSecurityGroup
```

誤答になりやすいもの:

| 選択肢 | 理由 |
| :--- | :--- |
| VPC Flow Logsのみ | SG変更というAPI操作は分からない |
| CloudWatch Metricsのみ | API操作の詳細は分からない |
| AWS Configのみ | 準拠評価や設定履歴には強いが、即時通知の典型構成はCloudTrail/EventBridge/CloudWatchと組み合わせる |

### Private subnetからインターネットへ出られない

確認順:

```text
Route Table
NAT Gateway配置と状態
NAT Gateway CloudWatch Metrics
Security Group
Network ACL
VPC Flow Logs
Reachability Analyzer
```

見るポイント:

| 観点 | 確認 |
| :--- | :--- |
| Route Table | `0.0.0.0/0`がNAT Gatewayへ向いているか |
| NAT Gateway | public subnetにあるか |
| Public subnet | Internet Gatewayへのdefault routeがあるか |
| NACL | ephemeral portが戻り通信で許可されているか |
| Flow Logs | REJECTが出ていないか |

### VPNトンネル障害を検知したい

選ぶ構成:

```text
VPN TunnelState
  -> CloudWatch Alarm
  -> SNS
```

関連して確認するもの:

```text
BGP session
Customer Gateway設定
ルート広告
オンプレ側ルーター状態
トンネル冗長性
```

### DNS問い合わせを監視したい

要件ごとの選択:

| 要件 | 答え |
| :--- | :--- |
| VPC内のDNS問い合わせを記録 | Resolver Query Logging |
| 悪性ドメインへの問い合わせをブロック | Resolver DNS Firewall |
| DNSルーティングポリシー | Route 53 Hosted Zone / Routing Policy |
| オンプレDNSとの名前解決 | Resolver inbound/outbound endpoint |

### 複数アカウントのネットワーク設定を統制したい

選ぶ構成:

```text
AWS Organizations
  -> AWS Config Aggregator
  -> Config Rules / Conformance Packs
  -> EventBridge
  -> Systems Manager Automation / Lambda
```

関連:

```text
Security系ポリシーの中央管理 = Firewall Manager
構成準拠の中央集約 = AWS Config Aggregator
IaCの複数アカウント展開 = CloudFormation StackSets
```

## 設計時の注意点

### ログ保存先

| 保存先 | 特徴 |
| :--- | :--- |
| CloudWatch Logs | 検索、Metric Filter、Alarm連携がしやすい |
| S3 | 長期保存、低コスト、Athena分析、監査保管に向く |
| Firehose | S3/OpenSearch/外部分析基盤などへ配送しやすい |

試験での考え方:

```text
すぐ検索・アラーム = CloudWatch Logs
長期保管・Athena分析 = S3
ストリーミング配送 = Firehose
```

### ログ保持

CloudWatch LogsはLog groupごとに保持期間を設定する。

```text
不要な無期限保持はコスト増になる。
監査要件がある場合はS3保管、ライフサイクル、暗号化も考える。
```

### マルチアカウント

マルチアカウントでは、個別アカウントだけで監視を完結させると運用が複雑になる。

代表構成:

```text
Organizations
  -> Organization CloudTrail
  -> Central logging account
  -> AWS Config Aggregator
  -> Firewall Manager
  -> CloudWatch cross-account observability
```

覚え方:

```text
証跡の中央化 = Organization CloudTrail
設定準拠の中央化 = Config Aggregator
セキュリティポリシーの中央化 = Firewall Manager
ログ保管の中央化 = Central logging account
```

## トラブルシューティングの型

### 1. 事象を数値で見る

```text
CloudWatch Metrics
CloudWatch Dashboard
CloudWatch Alarm history
```

例:

```text
ALB 5xxが増えた
NAT Gatewayでport allocation errorが出た
VPN TunnelStateが0になった
```

### 2. 実際のログを見る

```text
CloudWatch Logs
VPC Flow Logs
ALB Access Logs
CloudFront Logs
Resolver Query Logs
```

例:

```text
REJECTがあるか
どのclient IPから来ているか
どのDNS名を問い合わせているか
```

### 3. API操作履歴を見る

```text
CloudTrail
```

例:

```text
Route Tableが変更された
Security Groupが開けられた
NAT Gatewayが削除された
Trailが停止された
```

### 4. 現在構成と準拠を見る

```text
AWS Config
```

例:

```text
現在のSecurity Group設定
過去のRoute Table設定
非準拠リソース一覧
```

### 5. 経路として通れるか見る

```text
Reachability Analyzer
Network Access Analyzer
```

例:

```text
EC2からRDSへ到達できるか
InternetからSSH到達可能な経路がないか
```

## 試験で狙われる落とし穴

| 落とし穴 | 正しい理解 |
| :--- | :--- |
| CloudTrailを有効にすれば自動でCloudWatch Alarmが鳴る | AlarmにはCloudWatch Logs連携やEventBridgeなどの設計が必要 |
| VPC Flow LogsでHTTPのURLが見える | Flow LogsはIP/port/protocol中心。URLは見えない |
| ACCEPTならアプリ正常 | ACCEPTはネットワーク制御で許可された意味。アプリ成功ではない |
| Reachability Analyzerは実際に疎通試験する | 設定を静的分析する。実パケット送信ではない |
| Route AnalyzerでSG/NACLまで確認できる | Route AnalyzerはTGW Route Table中心。VPC内到達性はReachability Analyzer |
| Trusted Advisorは通信ログを分析する | Trusted Advisorは健全性チェック。通信ログはFlow Logsなどを見る |
| AWS Configで誰が変更したか完全に分かる | 実行主体はCloudTrailが得意 |
| Metric Filterは過去ログも自動でメトリクス化する | 作成後に取り込まれるログが対象 |
| Traffic Mirroringで通信をブロックできる | ミラーするだけ。遮断はNetwork FirewallやGWLB構成など |
| DNS Query Loggingでブロックできる | Loggingは記録。ブロックはDNS Firewall |

## 重要キーワード

```text
metric
namespace
dimension
alarm
evaluation period
datapoint
treat missing data
log group
log stream
metric filter
subscription filter
Logs Insights
management event
data event
organization trail
multi-region trail
configuration item
Config rule
conformance pack
aggregator
remediation
flow log
ACCEPT
REJECT
interface-id
srcaddr
dstaddr
resolver query log
traffic mirroring
reachability analyzer
route analyzer
network access analyzer
AWS Health event
Trusted Advisor
Well-Architected Tool
EventBridge rule
Systems Manager Automation
```

## 試験前チェックリスト

- CloudWatch、CloudTrail、AWS Config、VPC Flow Logsの違いを説明できる
- Security Group変更検知の典型構成を言える
- Flow Logsの`ACCEPT`と`REJECT`の意味を説明できる
- Flow LogsではpayloadやURLが見えないことを説明できる
- Metric FilterとLogs Insightsの違いを説明できる
- CloudTrail management eventsとdata eventsの違いを説明できる
- AWS Config Rules、Conformance Packs、Aggregatorの役割を説明できる
- Reachability AnalyzerとNetwork Access Analyzerの違いを説明できる
- Route AnalyzerはTGW Route Table中心の経路分析だと説明できる
- Trusted AdvisorとWell-Architected Toolは運用レビュー/改善向けだと説明できる
- Traffic MirroringとIPS/IDSの関係を説明できる
- VPN、Direct Connect、TGW、ALB、NAT Gatewayの代表メトリクスを知っている
- マルチアカウント監視でOrganization CloudTrail、Config Aggregator、中央ログアカウントを連想できる

## 公式ドキュメント

- [Amazon CloudWatch metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html)
- [Using Amazon CloudWatch alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [Monitoring log data with metric filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)
- [Analyzing log data with CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
- [What is AWS CloudTrail?](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
- [CloudTrail record contents](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference.html)
- [Sending events to CloudWatch Logs](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/send-cloudtrail-events-to-cloudwatch-logs.html)
- [What is AWS Config?](https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [Flow log records](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)
- [Route 53 Resolver query logging](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-query-logs.html)
- [What is VPC Reachability Analyzer?](https://docs.aws.amazon.com/vpc/latest/reachability/what-is-reachability-analyzer.html)
- [What is Network Access Analyzer?](https://docs.aws.amazon.com/vpc/latest/network-access-analyzer/what-is-network-access-analyzer.html)
- [Route Analyzer for AWS Network Manager](https://docs.aws.amazon.com/network-manager/latest/tgwnm/route-analyzer.html)
- [What is Amazon CloudWatch Internet Monitor?](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-InternetMonitor.html)
- [What is Amazon CloudWatch Network Monitor?](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Network-Monitor.html)
- [What is AWS Health?](https://docs.aws.amazon.com/health/latest/ug/what-is-aws-health.html)
- [What is AWS Trusted Advisor?](https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor.html)
- [Trusted Advisor service quotas](https://docs.aws.amazon.com/awssupport/latest/user/service-limits.html)
- [AWS Well-Architected Tool](https://docs.aws.amazon.com/wellarchitected/latest/userguide/waf.html)
- [What is Amazon EventBridge?](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html)
- [What is AWS Systems Manager?](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html)
