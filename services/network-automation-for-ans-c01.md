# Network Automation ANS-C01対策

## 目的

ANS-C01で出題される「ネットワークインフラ構築・構成の自動化」を、問題文から正しいサービスや手段を選べる形で整理する。

対象は主に次のサービスと概念。

```text
AWS CloudFormation
CloudFormation StackSets
CloudFormation Change Sets
CloudFormation Drift Detection
CloudFormation Guard
CloudFormation Hooks
AWS CLI
AWS SDK
AWS Config
AWS Config Rules
AWS Config Conformance Packs
AWS Config Aggregator
Systems Manager Automation
EventBridge
Lambda
```

## 最初に覚える結論

| 要件 | 疑うサービス/機能 |
| :--- | :--- |
| VPC、Subnet、Route Table、TGWなどをコードで作りたい | CloudFormation |
| 複数アカウント/複数リージョンへ同じネットワーク標準を展開したい | CloudFormation StackSets |
| 変更前に何が変わるか確認したい | CloudFormation Change Sets |
| 手動変更でテンプレートとの差分が出たか確認したい | CloudFormation Drift Detection |
| テンプレートが社内ルールに合うか事前チェックしたい | CloudFormation Guard |
| 非準拠リソースの作成をCloudFormation実行前に止めたい | CloudFormation Hooks |
| 繰り返し作業や確認をコマンド化したい | AWS CLI |
| アプリや運用ツールからAWS APIを呼びたい | AWS SDK |
| リソース設定の履歴、変更、準拠状況を見たい | AWS Config |
| Security GroupやVPC設定がルール違反か評価したい | AWS Config Rules |
| 複数ルールをまとめて配布/評価したい | AWS Config Conformance Packs |
| 複数アカウント/複数リージョンのConfig結果を集約したい | AWS Config Aggregator |
| 非準拠リソースを自動修復したい | AWS Config Remediation + Systems Manager Automation |
| Config違反やCloudTrailイベントを契機に処理したい | EventBridge + Lambda / Systems Manager |

## 全体像

```text
Code / Template
  |
  +--> CloudFormation
  |      +--> Change Sets
  |      +--> Drift Detection
  |      +--> StackSets
  |      +--> Guard / Hooks
  |
  +--> AWS CLI
  |      +--> deploy / validate / describe / detect drift
  |
  +--> AWS SDK
         +--> custom automation / internal tools / Lambda

Deployed AWS Resources
  |
  +--> AWS Config
         +--> Configuration Recorder
         +--> Config Rules
         +--> Conformance Packs
         +--> Aggregator
         +--> Remediation
```

## 2種類の自動化

ANS-C01では、自動化を2つに分けると判断しやすい。

| 種類 | 目的 | 代表サービス |
| :--- | :--- | :--- |
| 構築の自動化 | AWSリソースを標準化して作る | CloudFormation、StackSets、CLI、SDK |
| 構成/準拠の自動化 | 作られたリソースがルール通りか監視・修復する | AWS Config、Config Rules、Conformance Packs、Remediation |

覚え方:

```text
CloudFormation = 作る
AWS Config     = 見る・評価する・直す
AWS CLI        = 操作をコマンド化する
AWS SDK        = 操作をプログラム化する
```

## AWS CloudFormation

CloudFormationは、AWSリソースをテンプレートで定義し、スタックとして作成・更新・削除するIaCサービスである。

```text
template.yaml
  -> CloudFormation stack
  -> VPC / subnet / route table / security group / TGW / endpointなど
```

## CloudFormationの基本用語

| 用語 | 意味 |
| :--- | :--- |
| Template | YAML/JSONで書くリソース定義 |
| Stack | テンプレートから作られたリソース集合 |
| Resource | 作成対象のAWSリソース |
| Parameter | 環境ごとに変える入力値 |
| Output | 作成後に出力する値 |
| Mapping | リージョンや環境ごとの固定値表 |
| Condition | 条件付きでリソースを作る制御 |
| Change Set | 更新前の差分確認 |
| Drift Detection | 実リソースとテンプレートとの差分検出 |
| StackSet | 複数アカウント/リージョン展開 |

## ネットワークでCloudFormation化しやすいもの

代表例:

```text
VPC
Subnet
Route Table
Internet Gateway
NAT Gateway
Security Group
Network ACL
VPC Endpoint
Transit Gateway
Transit Gateway Attachment
Route 53 Hosted Zone / Record
CloudWatch Log Group
VPC Flow Logs
AWS Network Firewall
AWS WAF Web ACL
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| ネットワーク構成を再現可能にしたい | CloudFormation |
| 手動構築ミスを減らしたい | CloudFormation |
| dev/stg/prodで同じ構成を作りたい | CloudFormation + Parameters |
| 複数アカウントへ標準VPC設定を配布したい | StackSets |

## Change Sets

Change Setは、CloudFormationの更新前に「何が追加・変更・削除されるか」を確認する機能である。

```text
new template
  -> create change set
  -> review changes
  -> execute change set
```

使いどころ:

| 要件 | 理由 |
| :--- | :--- |
| 本番変更前に影響を確認したい | 置換、削除、変更対象を事前に見られる |
| 変更承認の証跡が必要 | 変更内容をレビューしやすい |
| 予期しないリソース置換を避けたい | Replacementの有無を確認できる |

注意:

```text
Change Setは成功を保証しない。
実行時のクォータ、依存関係、サービス側制約で失敗することはある。
```

## Drift Detection

Drift Detectionは、CloudFormation管理下のリソースが、テンプレート定義とズレていないかを確認する機能である。

```text
CloudFormation template expected state
  vs
Actual resource state
```

主なステータス:

| Status | 意味 |
| :--- | :--- |
| IN_SYNC | テンプレートと一致 |
| DRIFTED | テンプレートと実リソースが不一致 |
| NOT_CHECKED | 未確認 |
| DELETED | 管理対象リソースが削除されている |
| MODIFIED | 管理対象リソースの設定が変更されている |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 誰かが手動でSecurity Groupを変更した疑い | Drift Detection |
| StackSet配下の各アカウントで差分確認 | StackSet drift detection |
| CloudFormation外の変更を見つけたい | Drift Detection |

## StackSets

StackSetsは、1つのCloudFormationテンプレートを複数アカウント/複数リージョンへ展開する機能である。

```text
Administrator account
  -> StackSet
  -> Account A / ap-northeast-1
  -> Account B / ap-northeast-1
  -> Account C / us-east-1
```

## StackSetsの重要用語

| 用語 | 意味 |
| :--- | :--- |
| Administrator account | StackSetを管理するアカウント |
| Target account | Stackを展開されるアカウント |
| Stack instance | 特定アカウント/リージョン内のStack参照 |
| Self-managed permissions | IAM Roleを自分で用意する方式 |
| Service-managed permissions | AWS Organizations連携でIAM Roleを自動管理する方式 |
| Automatic deployments | OUに追加された新規アカウントへ自動展開 |
| Failure tolerance | 失敗許容数/割合 |
| Maximum concurrent accounts | 同時展開するアカウント数/割合 |
| Region concurrency | リージョンを順次/並列で展開 |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| Organizations配下の全アカウントへ同じVPC Flow Logs設定 | StackSets service-managed permissions |
| 新規アカウント作成時にも自動展開 | StackSets automatic deployments |
| 複数リージョンへ同じConfig有効化 | StackSets |
| 1アカウント内だけで構築 | 通常のCloudFormation stack |

## CloudFormation Guard

CloudFormation Guardは、テンプレートや構成データがルールに合っているかをチェックするpolicy-as-codeツールである。

```text
template.yaml
  -> cfn-guard validate
  -> pass / fail
```

用途:

```text
Security Groupで0.0.0.0/0のSSHを禁止
S3 bucket encryption必須
VPC Flow Logs必須
タグ必須
許可されたCIDRだけ使う
```

重要:

```text
cfn-guardはテンプレート構文そのものの完全検査ツールではない。
構文やリソース仕様の検査にはvalidate-templateやcfn-lintを使う。
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| デプロイ前にテンプレートが社内ポリシーに合うか確認 | CloudFormation Guard |
| CI/CDで非準拠テンプレートを止めたい | CloudFormation Guard |
| 既存リソースの準拠状況を継続評価 | AWS Config |

## CloudFormation Hooks

CloudFormation Hooksは、CloudFormationがリソースを作成/更新/削除する前に設定を検査し、違反があれば警告または失敗させる仕組みである。

```text
CloudFormation operation
  -> Hook
  -> WARN or FAIL
  -> resource provisioning
```

Hookの実装方式:

| 方式 | 内容 |
| :--- | :--- |
| Proactive controls | AWS Control Towerのproactive controlを利用 |
| Guard Hook | CloudFormation Guardルールを利用 |
| Lambda Hook | Lambda関数で検査 |
| Custom Hook | CloudFormation CLIで独自拡張を実装 |

Failure mode:

| Mode | 動き |
| :--- | :--- |
| WARN | 警告するが継続 |
| FAIL | 違反時に失敗させ、作成/更新を止める |

Guardとの違い:

```text
CloudFormation Guard = ルール言語/CLIで事前評価
CloudFormation Hooks = CloudFormation実行時に検査して止める仕組み
```

## AWS CLI

AWS CLIは、AWS APIをコマンドラインから呼び出すためのツールである。

```text
aws ec2 describe-vpcs
aws cloudformation deploy
aws configservice describe-compliance-by-config-rule
```

## AWS CLIで覚えること

| 項目 | 内容 |
| :--- | :--- |
| profile | 複数アカウント/ロールを切り替える |
| region | 操作対象リージョンを明示する |
| output | json/table/text/yamlなど |
| query | JMESPathで結果を絞る |
| pagination | 大量結果は自動ページングされる |
| waiter | リソース状態が変わるまで待つ |
| dry-run | 対応APIでは権限や実行可否を確認 |
| skeleton | 入力JSON/YAMLの雛形生成 |

よく使うグローバルオプション:

```text
--profile
--region
--output
--query
--no-paginate
--page-size
--max-items
--debug
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 手順をスクリプト化したい | AWS CLI |
| 変更前後の確認コマンドを残したい | AWS CLI |
| アカウント/リージョンごとに実行したい | profile / regionを使う |
| 自動化コードの中からAWS操作したい | AWS SDK |

## AWS CLIの認証・設定

CLIは複数の場所から設定や認証情報を読み取る。

代表例:

```text
~/.aws/config
~/.aws/credentials
AWS_PROFILE
AWS_REGION
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
IAM role
IAM Identity Center
```

実務・試験の注意:

```text
長期アクセスキーをコードやGitに保存しない。
EC2/Lambda/ECS上の自動化ではIAM Roleを使う。
人の操作ではIAM Identity CenterやAssumeRoleを使う。
```

## AWS SDK

AWS SDKは、アプリケーションや自動化ツールからAWS APIを呼び出すためのライブラリである。

代表例:

```text
Boto3 for Python
AWS SDK for JavaScript
AWS SDK for Java
AWS SDK for Go
AWS SDK for .NET
```

## SDKで覚えること

| 項目 | 内容 |
| :--- | :--- |
| Client | サービスAPIを呼ぶオブジェクト |
| Region | 呼び出すサービスリージョン |
| Credential provider chain | 認証情報を探す仕組み |
| Pagination | APIのページング処理 |
| Waiter | リソース状態待ち |
| Retry | 一時エラーやスロットリングへの再試行 |
| Throttling | APIレート制限 |
| Exponential backoff with jitter | 再試行間隔を徐々に伸ばし、ランダム化する |

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 独自ポータルからVPC作成を自動化 | AWS SDK |
| APIを組み合わせて条件分岐したい | AWS SDK |
| 定期的に設定を取得してレポート化 | AWS SDK or CLI |
| LambdaからAWS APIを呼ぶ | AWS SDK |

## SDKとCLIの違い

| 観点 | AWS CLI | AWS SDK |
| :--- | :--- | :--- |
| 使い方 | コマンド | プログラム |
| 向いている用途 | 手順化、検証、運用スクリプト | アプリ組み込み、複雑な自動化 |
| 実行環境 | shell、CI/CD、踏み台 | Lambda、アプリ、内部ツール |
| ロジック | shellで組む | 言語で組む |
| 試験キーワード | command line, script, one-time operation | application, programmatically, custom automation |

## AWS Config

AWS Configは、AWSリソースの構成、変更履歴、関係性、準拠状況を記録・評価するサービスである。

```text
AWS Resource
  -> AWS Config records configuration item
  -> Config Rule evaluates compliance
  -> compliant / non_compliant
  -> optional remediation
```

## AWS Configの基本用語

| 用語 | 意味 |
| :--- | :--- |
| Configuration Recorder | どのリソースタイプを記録するかを定義 |
| Delivery Channel | S3/SNSなどへの配信先 |
| Configuration Item | ある時点のリソース構成 |
| Configuration History | リソース構成の履歴 |
| Configuration Snapshot | 記録対象リソースのスナップショット |
| Config Rule | 準拠/非準拠を評価するルール |
| Managed Rule | AWS提供のルール |
| Custom Rule | LambdaまたはGuardで作る独自ルール |
| Remediation | 非準拠リソースの修復 |
| Conformance Pack | Config RulesとRemediationのまとまり |
| Aggregator | 複数アカウント/リージョンのConfig情報集約 |

## Config Rules

Config Ruleは、リソース設定がルールに合っているか評価する。

評価結果:

| Result | 意味 |
| :--- | :--- |
| COMPLIANT | 準拠 |
| NON_COMPLIANT | 非準拠 |
| ERROR | パラメータ不正など |
| NOT_APPLICABLE | 対象外 |

トリガー:

| Trigger | 内容 |
| :--- | :--- |
| Configuration changes | リソース変更時に評価 |
| Periodic | 定期評価 |
| Hybrid | 変更時 + 定期評価 |

評価モード:

| Mode | 内容 |
| :--- | :--- |
| Detective | 作成済みリソースを評価 |
| Proactive | デプロイ前のリソース設定を評価 |

重要:

```text
ConfigのProactive evaluationは、評価するだけ。
それ自体がリソース作成を止めたり修復したりするわけではない。
```

## ネットワークで出やすいConfig Rule例

試験で想定しやすいルール:

```text
Security Groupで0.0.0.0/0のSSHを禁止
VPC Flow Logsが有効であること
default Security Groupが使用されていないこと
restricted portsが公開されていないこと
EIPが未使用でないこと
CloudTrailが有効であること
S3 bucket public accessが禁止されていること
```

Configの選び方:

| 問題文 | 答え |
| :--- | :--- |
| 現在の設定が社内標準に準拠しているか確認 | AWS Config Rules |
| 過去にSecurity Groupがどう設定されていたか確認 | AWS Config configuration history |
| 複数アカウント/リージョンの準拠状況を集約 | AWS Config Aggregator |
| 非準拠Security Groupを自動修復 | AWS Config Remediation |

## Conformance Packs

Conformance Packは、複数のConfig RulesとRemediationをまとめて展開・監視する単位である。

```text
conformance-pack.yaml
  -> Config rules
  -> remediation actions
  -> compliance dashboard
```

用途:

| 要件 | 答え |
| :--- | :--- |
| ネットワーク標準ルールをまとめて配布 | Conformance Pack |
| 複数アカウントへ準拠ルールセットを展開 | Organization Conformance Pack |
| 監査用に統一ルールで評価 | Conformance Pack |

注意:

```text
Conformance PackはCloudFormation stackではない。
YAMLテンプレートでConfig RulesとRemediationをまとめるAWS Configの機能。
```

## AWS Config Aggregator

Aggregatorは、複数アカウント/複数リージョンのConfig設定・準拠情報を1つのアカウント/リージョンへ集約する。

```text
Account A / Region 1
Account B / Region 2
Account C / Region 3
  -> Config Aggregator account
```

重要:

```text
Aggregatorは集約して見るための仕組み。
Aggregatorから直接、送信元アカウントの設定を変更するものではない。
```

試験での見方:

| 問題文 | 答え |
| :--- | :--- |
| 全アカウント/全リージョンの準拠状態を中央で確認 | AWS Config Aggregator |
| 非準拠を自動修復 | AggregatorではなくConfig Remediation |
| ルールを組織展開 | Organization Conformance Pack / StackSets |

## Remediation

AWS Config Remediationは、非準拠リソースに対して修復アクションを実行する。

多くの場合、Systems Manager Automationドキュメントを使う。

```text
Config Rule
  -> NON_COMPLIANT
  -> Remediation action
  -> Systems Manager Automation
  -> fix resource
```

例:

```text
公開SSHのSecurity Group ruleを削除
S3 Block Public Accessを有効化
CloudTrailを有効化
タグ不足を修正
```

注意:

```text
自動修復は影響が大きい。
試験では「まず検知/通知」「承認後に修復」「最小権限Role」が安全な選択になることがある。
```

## EventBridge + Lambda / Systems Manager

EventBridgeは、AWSサービスイベントを契機に自動処理を起動できる。

```text
AWS Config compliance change
CloudTrail API event
CloudFormation stack status change
  -> EventBridge rule
  -> Lambda / Systems Manager Automation / SNS
```

使いどころ:

| 要件 | 構成 |
| :--- | :--- |
| Config違反を通知 | Config + EventBridge + SNS |
| Security Group変更を検知して戻す | CloudTrail/EventBridge + Lambda or Config Remediation |
| Stack失敗時に通知 | CloudFormation event + EventBridge + SNS |
| 定期的な棚卸し | EventBridge Scheduler + Lambda/CLI |

## CloudFormationとAWS Configの違い

| 観点 | CloudFormation | AWS Config |
| :--- | :--- | :--- |
| 主目的 | 構築・変更 | 記録・評価・監査 |
| タイミング | デプロイ時 | デプロイ後/変更時/定期 |
| 単位 | Stack | Resource / Rule |
| 差分 | Change Set / Drift Detection | Configuration history / compliance |
| 複数アカウント | StackSets | Aggregator / Organization Conformance Pack |
| 修復 | Stack updateで修正 | Remediationで修正 |
| 試験キーワード | IaC, template, stack, deploy | compliance, history, noncompliant, audit |

覚え方:

```text
CloudFormation = 正しい形で作る
AWS Config     = 作られた後も正しいか見続ける
```

## CloudFormation Guard / Hooks / AWS Configの違い

| 観点 | CloudFormation Guard | CloudFormation Hooks | AWS Config |
| :--- | :--- | :--- | :--- |
| 主目的 | テンプレート/構成をルール評価 | CloudFormation実行時に事前検査 | 既存リソースの継続評価 |
| タイミング | CI/CDやローカル | Create/Update/Delete前 | 作成後、変更時、定期 |
| 違反時 | CI/CDで止める設計にできる | WARNまたはFAIL | NON_COMPLIANTにする |
| 修復 | しない | 作成を止める | Remediation可能 |
| 典型問題 | policy-as-code | 非準拠リソースのprovisioning防止 | 監査、履歴、準拠状況 |

## AWS CLI / SDK / CloudFormationの違い

| 観点 | AWS CLI | AWS SDK | CloudFormation |
| :--- | :--- | :--- | :--- |
| 操作単位 | コマンド | プログラム内API | テンプレート |
| 状態管理 | 基本的に自分で管理 | 自分で管理 | Stackとして管理 |
| 再現性 | スクリプト次第 | 実装次第 | 高い |
| 差分確認 | 自分でdescribe比較 | 実装次第 | Change Set / Drift Detection |
| 向いている用途 | 運用手順、調査、単発変更 | 複雑な自動化、アプリ連携 | 標準構成の構築 |

試験での判断:

```text
長期運用する標準ネットワーク = CloudFormation
大量の確認/手順化 = AWS CLI
条件分岐の多い独自処理 = AWS SDK
準拠監査 = AWS Config
```

## ネットワーク自動化の設計ポイント

## 1. 冪等性

冪等性とは、同じ処理を何度実行しても結果が同じになる性質である。

```text
良い自動化:
  既に存在するなら再利用
  設定が正しければ何もしない
  設定が違えば安全に更新
```

ネットワークでは特に重要。

```text
Route table
Security Group rule
VPC endpoint
Transit Gateway route
DNS record
```

## 2. 変更前確認

変更前に確認するもの:

```text
account ID
region
VPC ID
CIDR
route table association
existing routes
existing security group rules
tags
CloudFormation stack status
Config compliance status
```

CLI例:

```text
aws sts get-caller-identity
aws configure list
aws ec2 describe-vpcs
aws ec2 describe-route-tables
aws cloudformation describe-stacks
```

## 3. 変更後確認

変更後に確認するもの:

```text
expected resources exist
route target is correct
security group rule is correct
VPC Flow Logs enabled
CloudFormation stack status is CREATE_COMPLETE / UPDATE_COMPLETE
Config rule is COMPLIANT
```

## 4. ロールバック

CloudFormationは更新失敗時にロールバックできる。

ただし、次は別途注意が必要。

```text
手動変更されたリソース
Retainされたリソース
DeletionPolicyが設定されたリソース
外部依存があるリソース
IAMやRoute 53など影響範囲が広いリソース
```

本番変更では、Change Set確認とrollback planをセットで考える。

## 5. 権限

自動化の権限は最小権限にする。

```text
CloudFormation execution role
StackSets administration/execution role
Config remediation role
Lambda execution role
SSM Automation assume role
```

避けること:

```text
AdministratorAccess前提
長期アクセスキーをGit保存
個人ユーザーのcredentialで本番自動化
```

## 頻出シナリオ

## 1. 複数アカウントに標準ネットワーク設定を配布

要件:

```text
AWS Organizations
複数アカウント
複数リージョン
標準VPC Flow Logs / Config / IAM Roleを配布
新規アカウントにも適用
```

答え:

```text
CloudFormation StackSets
service-managed permissions
automatic deployments
```

## 2. 本番CloudFormation更新前に影響確認

要件:

```text
本番スタック
変更前に追加/変更/削除/置換を確認
承認後に実行
```

答え:

```text
CloudFormation Change Sets
```

## 3. 手動変更されたSecurity Groupを検出

要件:

```text
CloudFormationで作ったSecurity Group
誰かが手動でルール変更
テンプレートとの差分を知りたい
```

答え:

```text
CloudFormation Drift Detection
```

または、継続的な準拠監視なら:

```text
AWS Config Rule
```

## 4. 既存環境の準拠状況を監査

要件:

```text
既にあるVPC/Security Group/Flow Logsを評価
履歴も見たい
複数アカウントで集約
```

答え:

```text
AWS Config
AWS Config Rules
AWS Config Aggregator
Conformance Packs
```

## 5. 非準拠リソースを自動修復

要件:

```text
Config RuleでNON_COMPLIANT
自動で設定を戻す
```

答え:

```text
AWS Config Remediation
Systems Manager Automation
```

## 6. テンプレート段階で社内ポリシー違反を止める

要件:

```text
CloudFormationテンプレート
Security Groupの公開SSHを禁止
CI/CDで検査
```

答え:

```text
CloudFormation Guard
```

CloudFormation実行時に止めるなら:

```text
CloudFormation Hooks with FAIL mode
```

## よくある誤解

| 誤解 | 正しい理解 |
| :--- | :--- |
| CloudFormationは変更前確認なしで安全 | Change Setsで影響確認する |
| Change Setは必ず成功を保証する | 事前差分確認であり、実行成功保証ではない |
| Drift DetectionはすべてのAWSリソース差分を完全検出 | 対応リソース/プロパティに依存する |
| StackSetsは1リージョンだけの機能 | 複数アカウント/複数リージョン展開に使う |
| AWS Configはリソースを作るサービス | 記録・評価・監査が主目的 |
| Config Aggregatorでリソース修正できる | Aggregatorは集約ビューであり、変更権限を持つ仕組みではない |
| Config Proactive evaluationは作成を止める | 評価するだけ。止めるならHooksやCI/CD制御 |
| CLIは状態管理してくれる | CLIはAPI実行手段。状態管理は自分で行う |
| SDKなら自動で安全な権限になる | 権限設計はIAM Role/Policy次第 |
| 自動修復は常に最善 | 誤修復や通信断のリスクがあるため、影響評価が必要 |

## 試験直前チェック

- [ ] CloudFormationはIaCでAWSリソースをStack管理する
- [ ] Change Setsは更新前の差分確認
- [ ] Drift DetectionはCloudFormation外の手動変更検出
- [ ] StackSetsは複数アカウント/複数リージョン展開
- [ ] StackSets service-managed permissionsはAWS Organizationsと連携する
- [ ] StackSets automatic deploymentsは新規アカウント/OUsへの自動展開
- [ ] CloudFormation Guardはpolicy-as-codeでテンプレートを評価する
- [ ] CloudFormation Hooksはprovisioning前にWARN/FAILできる
- [ ] AWS CLIは運用手順や確認のコマンド化
- [ ] AWS SDKはアプリ/自動化コードからAWS APIを呼ぶ
- [ ] SDK/CLIではcredential provider chain、profile、regionを意識する
- [ ] SDKの再試行はthrottlingや一時エラー対策として重要
- [ ] AWS Configは構成履歴、関係性、準拠状況を記録/評価する
- [ ] Config Ruleの結果はCOMPLIANT/NON_COMPLIANT/ERROR/NOT_APPLICABLE
- [ ] Config RuleはManaged RuleとCustom Ruleがある
- [ ] Conformance Packは複数Config RulesとRemediationのまとまり
- [ ] Config Aggregatorは複数アカウント/複数リージョンの集約ビュー
- [ ] Config Remediationは多くの場合SSM Automationで修復する
- [ ] 作る自動化はCloudFormation、見る自動化はConfig

## 公式参照

- [Managing AWS resources as a single unit with CloudFormation stacks](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacks.html)
- [Update CloudFormation stacks using change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)
- [Detect unmanaged configuration changes with drift detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html)
- [Managing stacks across accounts and Regions with StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html)
- [StackSets concepts](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html)
- [Enable automatic deployments for StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-manage-auto-deployment.html)
- [CloudFormation best practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [What are CloudFormation Hooks?](https://docs.aws.amazon.com/cloudformation-cli/latest/hooks-userguide/what-is-cloudformation-hooks.html)
- [What is AWS CloudFormation Guard?](https://docs.aws.amazon.com/cfn-guard/latest/ug/what-is-guard.html)
- [Command line options in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-options.html)
- [Using pagination options in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-pagination.html)
- [AWS SDKs standardized credential providers](https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html)
- [AWS SDKs retry behavior](https://docs.aws.amazon.com/sdkref/latest/guide/feature-retry-behavior.html)
- [What is AWS Config?](https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html)
- [AWS Config terminology and concepts](https://docs.aws.amazon.com/config/latest/developerguide/config-concepts.html)
- [Evaluating Resources with AWS Config Rules](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config.html)
- [Remediating Noncompliant Resources with AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/remediation.html)
- [Conformance Packs for AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html)
