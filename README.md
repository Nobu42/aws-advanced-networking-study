# AWS Advanced Networking Study

AWS Certified Advanced Networking - Specialty（ANS-C01）の学習用リポジトリ。

AWSネットワークの設計、実装、運用、トラブルシューティング、セキュリティを、試験対策とハンズオンの両面から整理する。

## 学習方針

- 公式試験ガイドのドメインに沿って学習する
- サービス名の暗記ではなく、要件に応じた設計判断を説明できるようにする
- 構成図、比較表、CLI、ハンズオンで理解を深める
- 問題演習では、正解理由と誤答理由の両方を記録する
- AWS認証情報、秘密鍵、個人情報は保存しない

## 試験ドメイン

| Domain | 内容 | 比率 |
| :--- | :--- | ---: |
| 1 | Network Design | 30% |
| 2 | Network Implementation | 26% |
| 3 | Network Management and Operation | 20% |
| 4 | Network Security, Compliance, and Governance | 24% |

詳細は [ANS-C01ドメインマップ](./exam-guide/ans-c01-domain-map.md) を参照する。

## ディレクトリ

| ディレクトリ | 用途 |
| :--- | :--- |
| `exam-guide/` | 試験範囲、学習計画、進捗管理 |
| `fundamentals/` | BGP、DNS、IPv6などの基礎知識 |
| `services/` | AWSネットワークサービス別ノート |
| `labs/` | ハンズオン手順、CLI、検証結果 |
| `practice-questions/` | 練習問題と解説 |
| `incorrect-answers/` | 誤答分析と弱点整理 |
| `diagrams/` | ネットワーク構成図 |

## 既存リポジトリとの役割分担

- `aws-reference`: AWS実務、案件対策、構築・変更手順、証跡取得
- `aws-advanced-networking-study`: ANS-C01資格試験対策

既存資料を利用する場合は丸ごと複製せず、試験範囲に必要な内容だけを資格向けに再編集する。

## 公式資料

- [AWS Certified Advanced Networking - Specialty](https://aws.amazon.com/certification/certified-advanced-networking-specialty/)
- [ANS-C01 Exam Guide](https://d1.awsstatic.com/training-and-certification/docs-advnetworking-spec/AWS-Certified-Advanced-Networking-Specialty_Exam-Guide.pdf)
