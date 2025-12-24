---
title: "AWS DevOps Agent はどこまで障害の原因を特定できるのか？re:Invent 2025 の新機能を検証"
emoji: "🔍"
type: "tech"
topics: ["aws", "devops", "reinvent", "mcp", "agent"]
published: false
publication_name: "10q89s"
---

:::message
本記事では、英文の日本語要約等で生成 AI を利用しています。
:::

## はじめに

こんにちは、URBAN HACKS サーバーサイドエンジニアの三好です。

AWS re:Invent 2025 で **AWS DevOps Agent** が発表されましたね！

**DevOps Agent** は、「自律的なオンコールエンジニア」として動作する Frontier Agent です。インシデント発生時に CloudWatch、GitHub、ServiceNow などのデータを横断的に分析し、根本原因の特定と緩和策の提案を行います。

本記事では、実際に Lambda や ECS で様々な障害シナリオを作り、DevOps Agent がどこまで原因を深掘りできるのかを検証します。

また、re:Invent の別セッション（DAT301）で体験した MCP ベースの自作 Agent との比較を通じて、自作 Agent での障害対応の可能性についても触れられればと思います。

### この記事で扱うこと

- DevOps Agent がインシデント調査で何を参照し、何を出力するのか（Root Cause / Mitigation Plan）の観察
- Lambda / ECS で用意した障害シナリオに対して、どこまで原因を絞り込めるかの検証
- MCP ベースの自作 Agent（DAT301 で体験した Mahavat）を踏まえた、できること・できないことの比較

### まず結論

- DevOps Agent は CloudWatch Logs/Metrics に加えて、ECR イメージ履歴、CloudFormation 履歴、Secrets Manager なども横断して相関を取れるため、一次対応の「状況整理」と「原因候補の絞り込み」に有効だと感じました。
- OOM / panic / 明確な接続エラーなど、症状がはっきりしていてログやメトリクスに根拠が残るケースでは、特に効果が出やすい印象でした。
- 一方で、ログが薄い・観測点が少ないケースでは観測可能な情報だけでは一意に特定できない場合があるため、対話しながら追加調査を促す（「他の可能性は？」「この時間帯のログを見て」など）使い方が現実的です。
- Lambda と ECS で「コードにどこまで近づけるか」の前提が異なり、ECS はコンテナイメージ運用のためログ依存になりやすい点に注意が必要です（詳細は後述）。
- 今回の検証では、調査完了までの所要時間は概ね 6〜21 分程度で、単一原因のシナリオは短く、複数の可能性を探索する必要があるケースは長くなる傾向がありました。

## 1. AWS DevOps Agent とは

### 1.1 Frontier Agent という新しい概念

AWS DevOps Agent は、re:Invent 2025 で発表された **Frontier Agent** の一つです。Frontier Agent は従来の AI アシスタントとは異なり、下記のような特徴を持ちます。

- **自律性**
  - 人間の介入なしに何時間・何日も動作可能
- **大規模スケーラビリティ**
  - 複数のデータソースを横断的に分析
- **継続的な学習**
  - 過去のインシデントパターンから改善

### 1.2 DevOps Agent の主要機能

| 機能 | 説明 |
|-----|------|
| **Topology 構築** | システムコンポーネントと依存関係のマップを自動生成 |
| **Root Cause （根本原因）分析** | メトリクス、ログ、トレースを相関分析して根本原因を特定 |
| **Mitigation Plan （緩和策）提案** | 具体的な緩和策を Kiro 等の開発ツール向け Spec として出力 |

### 1.3 連携可能なサービス

**観測系**
- Amazon CloudWatch
- Datadog
- Dynatrace
- New Relic
- Splunk

**デプロイ / CI/CD**
- GitHub Actions
- GitLab CI/CD

**インシデント管理**
- ServiceNow
- PagerDuty
  - Dev Ops Agent で Webhook を受け取って、調査開始を自動化できるかも？（未検証）

**拡張**:
- BYO MCP Server（Grafana、Prometheus 等を独自に統合可能）

### 1.4 現時点での制限事項（Public Preview）

DevOps Agent は現在 Public Preview 段階のため、以下の制限があります。

| 制限事項 | 内容 |
|---------|------|
| **利用可能リージョン** | us-east-1 のみ |
| **対応言語** | プロンプトは英語のみ（日本語は使用不可） |

GA 後に東京リージョンや日本語対応が追加されることを期待したいところです。

## 2. 検証環境のセットアップ

### 2.1 Agent Space の作成

Agent Space は、DevOps Agent がアクセスできるリソースのスコープを定義するものです。

検索ボックスから `DevOps Agent` を選択し、Agent Space を作成します。

![Agent Space 作成](/images/aws-devops-agent-investigation/agent_space/create_agent_space.png)

作成された Agent Space は下図のようなイメージになります。
`Capabilities` として様々な外部サービスとの連携がサポートされています。

![Agent Space](/images/aws-devops-agent-investigation/agent_space/agent_space.png)


### 2.2 GitHub 連携の設定

DevOps Agent が GitHub のデプロイ履歴を参照できるようにします。

![Add Pipeline](/images/aws-devops-agent-investigation/agent_space/add_pipeline.png)

`Add source` から GitHub を選択し、権限を与える Organization、User、Repository を指定します。**リポジトリ単位で絞り込める**ため、必要最小限のリポジトリのみを連携させることができます。
連携完了すると、GitHub 側に Authorized GitHub Apps として `AWS DevOps Agent` が追加されます。

![Authorized GitHub Apps](/images/aws-devops-agent-investigation/agent_space/authrized_gitHub_apps.png)

## 3. 検証シナリオ一覧

今回の検証では、以下の計 8 シナリオを用意しました。

### 3.1 シナリオ設計

#### 検証①: Lambda シナリオ

| # | シナリオ | 障害内容 | 期待する検知 |
|---|---------|---------|-------------|
| 1-1 | TypeError 例外 | `event.body` が null の場合に例外発生 | コードの問題箇所特定 |
| 1-2 | OOM クラッシュ | 128MB 設定で大量メモリ確保 | メモリ設定の問題指摘 |
| 1-3 | 依存サービス障害 | 存在しない DB ホストへの接続試行 | 下流サービスの問題切り分け |
| 1-4 | タイムアウト | 5 秒設定で 30 秒の処理を実行 | 遅延原因の特定 |

#### 検証②: ECS シナリオ

| # | シナリオ | 障害内容 | 期待する検知 |
|---|---------|---------|-------------|
| 2-1 | メモリリーク | リクエストごとに 10MB 確保し解放しない | OOM 検知 + デプロイ相関 |
| 2-2 | CPU スパイク | 16 goroutine で 60 秒間 CPU を使い切る | CPU 異常の原因特定 |
| 2-3 | panic | 意図的に panic を発生させクラッシュ | Exit Code + スタックトレース特定 |
| 2-4 | DB 接続エラー | 認証情報なしで DB 接続を試行 | 設定ミスの特定（ログなし） |

## 4. 検証結果：検証①（Lambda）

### 4.1 シナリオ 1-1: TypeError 例外

このシナリオでは、**CloudWatch Alarm → DevOps Agent 調査 → Mitigation Plan** の一連の流れを詳しく解説します。

#### 4.1.1 障害の概要

`user-profile-service` で `event.body` が undefined の場合に TypeError が発生。

```javascript
// バグのあるコード
const bodyData = JSON.parse(event.body);  // event.body が null → bodyData は null
const userName = bodyData.user.name;       // null.user で TypeError
```

#### 4.1.2 Step 1: 障害を発生させる

```bash
curl -X POST "${API_ENDPOINT}api/profile" \
  -H "x-api-key: ${API_KEY}"
```

#### 4.1.3 Step 2: CloudWatch Alarm の発火

Lambda のエラーメトリクスが閾値を超え、アラームが ALARM 状態に遷移します。

![CloudWatch Alarm が ALARM 状態に](/images/aws-devops-agent-investigation/scenario01/alarm.png)

#### 4.1.4 Step 3: DevOps Agent で調査を開始

![DevOps Agent の Incident Response タブ](/images/aws-devops-agent-investigation/scenario01/incident_response.png)

DevOps Agent の Incident Response タブを開き、以下のプロンプトで調査を依頼します。
（プリセットされている `Latest alarm` を押下すると下記プロンプトが用意されます）

**調査プロンプト**:
```text
Investigate my latest CloudWatch alarm that triggered and analyze the underlying metrics and logs to determine the root cause
```

![DevOps Agent の Incident Response タブから調査開始](/images/aws-devops-agent-investigation/scenario01/start_investigating.png)

#### 4.1.5 Step 4: DevOps Agent の調査プロセス

DevOps Agent は以下のステップで調査を進めます

1. アラーム関連情報を収集
1. メトリクスやログを分析
1. 根本原因を特定
1. 対応策（緩和策）を提示

:::details 調査プロセス（長いです）
![調査中](/images/aws-devops-agent-investigation/scenario01/investigating_process.png)
:::

#### 4.1.6 Step 5: 調査結果の確認

![Root Cause](/images/aws-devops-agent-investigation/scenario01/root_cause.png)

**要約**
> `user-profile-service` Lambda 関数の `index.js` 4 行目で、null オブジェクトの `user` プロパティにアクセスしようとして TypeError が発生しました。
> null チェックが欠如しているコード上の欠陥が原因です。

:::details DevOps Agent の原文出力（Root Cause）
> **Root cause**
> 1. Lambda function null pointer error in request handler
>
> The user-profile-service Lambda function has a code defect at line 4 of index.js (index.js:4:29) where the handler attempts to read the 'user' property from a null object without proper null checking. This causes a TypeError: 'Cannot read properties of null (reading 'user')' that immediately fails the Lambda invocation.
>
> The error occurred at 2025-12-16T05:45:37Z and was the sole cause of the CloudWatch alarm trigger. The function completed execution in 65ms, indicating fast failure early in the request processing lifecycle before any database operations could occur.
>
> The code lacks defensive programming practices such as null/undefined checks before property access.
:::

#### 4.1.7 Step 6: Mitigation Plan の確認

DevOps Agent が提案する緩和策を以下に示します。

![Mitigation Plan](/images/aws-devops-agent-investigation/scenario01/mitigation_plan.png)

**要約**
> 即時の運用対応（ロールバック等）では解決できません。コード修正が必要です。
> 具体的には、`index.js:4` で `user` プロパティにアクセスする前に null/undefined チェックを実装し、Lambda ハンドラの入力パラメータ全体の検証を追加します。

:::details DevOps Agent の原文出力（Mitigation Plan）
**提案された Mitigation Plan**

> No immediate operational mitigation action can be identified.
>
> Analysis of the root cause and deployment history indicates a pre-existing code defect in the user-profile-service Lambda function (arn:aws:lambda:us-east-1:651706755222:function:user-profile-service) in AWS account 651706755222 (us-east-1 region).
>
> The function fails with TypeError: Cannot read properties of null (reading 'user') at index.js:4:29 due to missing null/undefined validation before property access. This is a latent code defect that existed prior to the incident – the baseline period showed zero invocations, and when the function finally received an invocation on 2025-12-16T05:45:37Z with null input, it immediately failed.
>
> No deployment history was found indicating recent changes that could be rolled back. This issue requires code changes to add defensive null checking and input validation rather than immediate operational mitigation such as rollbacks or configuration adjustments. A code change specification has been generated to guide development work needed to resolve this defect permanently.
>
> **Next steps**
> Agent-ready spec
>
> Once the incident is resolved, implement these changes in your source code and infrastructure definitions. The below spec provides implementation guidance for development and automated agents.
>
> **Change requirements**
>
> THE SYSTEM SHALL implement null and undefined validation before accessing the 'user' property in the Lambda function handler at index.js:4.
>
> THE SYSTEM SHALL implement comprehensive input validation for all Lambda function handler parameters.
:::

#### 4.1.8 結果サマリー

| 評価項目 | 結果 | 備考 |
|---------|------|------|
| エラー種類の特定 | ✅ | TypeError |
| エラー発生箇所の特定 | ✅ | `index.js:4:29` |
| 根本原因の説明 | ✅ | null チェック不足を指摘 |
| 修正提案の有無 | ✅ | Next Steps を出力 |
| 調査所要時間 | 約6分 | |

#### 4.1.9 考察

DevOps Agent は**エラー発生箇所（`index.js:4:29`）を正確に特定**し、null チェック不足というコードレベルの問題まで指摘できました。また、「運用対応では解決できない」と正しく判断し、開発ツール向けの Agent-ready spec を出力する点も実用的です。

調査時間（約6分）は正直な感想としては速くはないと感じましたが、**人間が別作業をしている間にバックグラウンドで調査を進めてくれる非同期性**に価値があります。深夜のオンコール対応で「Agent に投げて、ユーザー対応等の別タスクを並行して進める」等の使い方が有効だと思います。

### 4.2 シナリオ 1-2: OOM

このシナリオでは、メモリ不足によるクラッシュを DevOps Agent がどこまで分析できるかを検証します。

#### 4.2.1 障害の概要

`order-processing-service` で大量のメモリを確保してクラッシュさせます。

#### 4.2.2 問題を引き起こす Lambda コード

```javascript
exports.handler = async () => {
  // 500万回ループで巨大な文字列配列を作成
  const hugeArray = [];
  for (let i = 0; i < 5000000; i++) {
    hugeArray.push('a'.repeat(500));
  }
  return { statusCode: 200, body: 'completed' };
};
```

#### 4.2.3 障害を発生させる

```bash
curl -X POST "${API_ENDPOINT}api/orders" \
  -H "x-api-key: ${API_KEY}"
```

#### 4.2.4 DevOps Agent の調査プロセス

:::details 調査中の思考プロセス（長いです）
![調査中](/images/aws-devops-agent-investigation/scenario02/investigating_process.png)
:::

#### 4.2.5 Root Cause

![Root Cause](/images/aws-devops-agent-investigation/scenario02/root_cause.png)

**要約**
> `order-processing-service` Lambda 関数が 128MB のメモリ設定では不足しており、実行時に 99.2%（127MB / 128MB）を消費しました。メモリ枯渇により処理が完了できず、30 秒のタイムアウトで終了しています。リソース名から意図的な OOM テストシナリオの可能性も指摘されています。

メモリの不足を指摘できていますね。

:::details DevOps Agent の原文出力（Root Cause）
> **Root cause**
> 1. Lambda function configured with insufficient memory (128 MB) leading to memory exhaustion and timeout
>
> The order-processing-service Lambda function is configured with only 128 MB of memory, which is insufficient for its workload.
>
> At 2025-12-17T01:38:00Z, the function exhausted 99.2% of its allocated memory (127 MB out of 128 MB) during execution. This memory exhaustion prevented the function from completing its work, causing it to run for the full 30-second timeout period before being terminated by AWS Lambda.
>
> The timeout resulted in an error that triggered the lambda-order-processing-service-errors CloudWatch alarm.
>
> The log group name OomLambdaLogGroup and IAM role name OomLambdaServiceRole suggest this may be a deliberate test scenario for out-of-memory conditions.
>
> No recent code deployments or configuration changes were detected, indicating the error was triggered by invocation of existing code that requires more than 128 MB of memory to execute successfully.
:::

#### 4.2.6 Mitigation Plan

**要約**
> `order-processing-service` Lambda 関数が **2025-12-17T01:38:00Z** にメモリ不足（128MB 中 127MB を消費、99.2%）を起こし、30 秒のタイムアウトで終了しました。メモリ割当を **128MB → 512MB（4倍）** に増加することを推奨しています。具体的な AWS CLI コマンド、検証手順、ロールバック手順も提示されました。

:::details Mitigation Plan のスクリーンショット
![Mitigation Plan](/images/aws-devops-agent-investigation/scenario02/mitigation_plan.png)
:::

#### 4.2.7 結果サマリー

| 評価項目 | 結果 | 備考 |
|---------|------|------|
| OOM の検知 | ✅ | |
| メモリ設定の問題指摘 | ✅ | |
| メモリサイズ増の提案 | ✅ | |
| 調査所要時間 | 約6分 | |

#### 4.2.8 考察

具体的なメモリ使用率を特定しており、期待通りの精度でした。

興味深いのは、「OomLambdaLogGroup」「OomLambdaServiceRole」というリソース名から「これは意図的な OOM テストシナリオかもしれない」と推測している点です。
今回の実験では安直な命名にしてしまいましたが、**AI にとってもリソース名は重要なコンテキスト**になることがわかりました。人間向けの可読性だけでなく、対 AI の観点でも命名規則を考える時代が来ているのかもしれません。

具体的な AWS CLI コマンドや Rollback 手順まで含めた完全な手順書が出力される点はありがたく、そのまま Runbook として使えそうだと感じました。

一方、シナリオ 1-1 では見てくれていた Lambda のスクリプトの中身を今回は見ていないようだったので、実装を踏まえて調査してほしい場合は明示的にプロンプトにその旨を含めるほうが良さそうです。

### 4.3 シナリオ 1-3: 依存サービスに接続できない

`notification-service` が存在しないホスト `invalid-db-host.local` への接続を試行。

#### 4.3.1 Root Cause

![Root Cause](/images/aws-devops-agent-investigation/scenario04/root_cause.png)

**要約**
> `notification-service` Lambda 関数が存在しない DB ホスト名 `invalid-db-host.local` への接続を試行し、DNS 解決エラー（ENOTFOUND）で失敗しました。VPC から到達不可能なホスト名が原因です。

:::details DevOps Agent の原文出力（Root Cause）
> **Root cause**
> 1. Lambda function configured with invalid database hostname causing DNS resolution failure
>
> The notification-service Lambda function is configured to connect to a database host named `invalid-db-host.local`, which cannot be resolved by DNS.
>
> When the function was invoked at 2025-12-17T05:31:07Z, it immediately failed with the error: `getaddrinfo ENOTFOUND invalid-db-host.local` (error code ENOTFOUND, errno -3008).
>
> This DNS resolution failure prevented the function from completing successfully, resulting in a Lambda error that triggered the CloudWatch alarm lambda-notification-service-errors at 05:32:35Z.
>
> The error occurs at the syscall level during DNS lookup (getaddrinfo), indicating the hostname does not exist in DNS or is unreachable from the Lambda's VPC (vpc-0b9dea157524aedae).
:::

#### 4.3.2 Mitigation Plan

**提案された Mitigation Plan（要約）**

> **No immediate operational mitigation action can be identified**
>
> 環境変数 `DB_HOST` には正しい RDS エンドポイントが設定されていますが、Lambda のコード内で**ハードコードされた無効なホスト名**を使用しているため、コード修正が必要です。
>
> **Next steps（Agent-ready spec）**:
> 1. `DB_HOST` 環境変数からデータベース接続先を取得するよう修正します
> 2. `DB_HOST` 環境変数が未設定の場合のエラーハンドリングを実装します

:::details Mitigation Plan のスクリーンショット
![Mitigation Plan](/images/aws-devops-agent-investigation/scenario03/mitigation_plan.png)
:::

#### 4.3.3 結果サマリー

| 評価項目 | 結果 | 備考 |
|---------|------|------|
| 接続エラーの検知 | ✅ | `ENOTFOUND` を特定 |
| 接続先ホストの特定 | ✅ | `invalid-db-host.local` |
| ネットワーク経路の確認 | ✅ | VPC ID まで特定 |
| 調査所要時間 | 約7分 | |

#### 4.3.4 考察

DevOps Agent は DNS 解決エラーを正確に特定し、さらに**コード側でハードコードされている**というところまで見抜けました。

シナリオ 1-1、1-2 と同様に「運用対応では解決できない、コード修正が必要」という判断は妥当で、Next steps として具体的な修正要件を提示してくれる点は一貫しています。

### 4.4 シナリオ 1-4: タイムアウト

`report-generator-service` が 5 秒のタイムアウト設定に対し、30 秒の処理を実行。

#### 4.4.1 問題を引き起こす Lambda コード

```javascript
exports.handler = async () => {
  await new Promise(r => setTimeout(r, 30000)); // 30秒待機
  return { statusCode: 200, body: 'done' };
};
```

**設定**:
- タイムアウト: 5秒（意図的に短く設定）
- 実際の処理時間: 30秒

#### 4.4.2 Root Cause

![Root Cause](/images/aws-devops-agent-investigation/scenario04/root_cause.png)

**要約**
> `report-generator-service` Lambda 関数が過去 3 回の実行すべてで 5000ms のタイムアウトで失敗し、アプリケーションログが一切出力されていません。ハンドラの設定ミス、初期化時のブロッキング処理、または依存関係の欠如が原因と推測されます。デプロイパッケージサイズが 247 バイトと異常に小さく、不完全なデプロイの可能性も指摘されています。

デプロイパッケージサイズまで見られるとは思ってなかったです。
ただ、やはり Lambda の場合はスクリプトの内容を見てほしい旨をプロンプトに含めるほうが調査の精度を高められそうです。

:::details DevOps Agent の原文出力（Root Cause）
> **Root cause**
> Lambda function code fails to execute due to handler misconfiguration or blocking initialization code
>
> The `report-generator-service` Lambda function has a fundamental code-level issue that prevents the handler from executing any application logic.
>
> The function consistently times out after exactly **5000 ms** (the configured timeout limit) across all 3 historical invocations spanning 2 days (Dec 15–17), with **zero application logs** produced in any execution.
>
> Evidence indicates the handler is affected by one or more of the following issues:
> 1. The handler is misconfigured (the handler path `index.handler` points to a non-existent function)
> 2. The handler contains blocking or infinite loop code at module initialization
> 3. The handler has missing dependencies, causing the runtime to hang during initialization
>
> The deployed package size is only **247 bytes**, which is suspiciously small for a `report-generator` service, suggesting incomplete or minimal code deployment.
>
> Aurora RDS metrics confirm **zero database connections** were ever established, indicating the function never reaches database connection logic.
:::

#### 4.4.3 Mitigation Plan

**提案された Mitigation Plan（要約）**

> **No immediate operational mitigation action can be identified**
>
> 初回デプロイ以降、全ての実行で 5000ms タイムアウト・アプリケーションログなしという状態が続いており、ロールバック可能な過去バージョンも存在しません。コード修正が必要です。
>
> **Next steps（Agent-ready spec）**:
> 1. `index.handler` に有効なハンドラ関数を実装します
> 2. 適切なタイムアウト・エラーハンドリングを持つ DB 接続処理を実装します
> 3. ハンドラ実行ライフサイクル全体にわたるロギングを実装します
> 4. レポート生成処理に十分な時間を確保できるよう Lambda タイムアウト設定を延長します

:::details Mitigation Plan のスクリーンショット
![Mitigation Plan](/images/aws-devops-agent-investigation/scenario04/mitigation_plan.png)
:::

#### 4.4.4 結果サマリー

| 評価項目 | 結果 | 備考 |
|---------|------|------|
| タイムアウト検知 | ✅ | |
| タイムアウト設定の確認 | ✅ | |
| 処理時間の問題指摘 | ✅ | |
| 調査所要時間 | 約13分 | |

#### 4.4.5 考察

単に「タイムアウトした」だけでなく、**パッケージサイズ（247 bytes）が不自然に小さい**点から、より深い原因（ハンドラの設定ミス or 初期化時のブロッキング）まで推測しています。

また、Aurora RDS のメトリクスも確認し「DB 接続が一度も確立されていない」ことから「ハンドラがビジネスロジックに到達していない」と結論づけており、複数のデータソースを横断した分析ができていることが確認できました。

## 5. 検証結果：検証②（ECS）

### 5.1 シナリオ 2-1: メモリリーク

#### 5.1.1 障害の概要

`api-gateway-service`（ECS Fargate）でメモリリークを発生させ、OOM によるコンテナ再起動を引き起こします。

```go
// メモリリーク用のグローバル変数（解放されない）
var memoryLeakStorage [][]byte
var memoryLeakMutex sync.Mutex

// memoryLeakHandler はメモリリークを発生させる
// 呼び出すたびに10MBのメモリを確保し、解放しない
func memoryLeakHandler(w http.ResponseWriter, r *http.Request) {
	memoryLeakMutex.Lock()
	defer memoryLeakMutex.Unlock()

	// 10MB のメモリを確保（解放されない）
	leak := make([]byte, 10*1024*1024)
	for i := range leak {
		leak[i] = byte(i % 256)
	}
	memoryLeakStorage = append(memoryLeakStorage, leak)
	// ...
}
```

**設定**:
- メモリ: 512MB
- CPU: 0.25 vCPU

#### 5.1.2 Root Cause

**要約**
> `api-gateway-service` の ECS Fargate タスクが **2025-12-18T01:30:08Z** のデプロイ以降、8〜15 分で 512MB メモリを使い果たし Exit Code 137（OOM Kill）で終了しています。メモリ使用率が起動時から段階的に上昇（69〜91%）する一貫したパターンが確認されました。CPU 使用率は低く（0.3〜5%）、コードレベルのメモリリークが原因と判断されました。繰り返しのコンテナ再起動により ALB で 502 エラーが発生しています。

:::details DevOps Agent の原文出力（Root Cause）
> **Root cause**
> Memory leak in api-gateway-service application causing OOM container terminations
>
> The `api-gateway-service` application deployed at **2025-12-18T01:30:08Z** contains a **memory leak** that causes ECS Fargate containers to exhaust their allocated **512 MB memory** within **8–15 minutes of startup**, resulting in container termination with **exit code 137 (Out Of Memory)**.
>
> CloudWatch metrics show a consistent pattern where memory utilization grows steadily from baseline (<1%) to critical levels (**70–91%**) within 10–15 minutes, immediately followed by container termination and restart.
>
> Observed memory usage patterns prior to OOM kills:
> - **81% at 01:54**
> - **69% at 01:41**
> - **69% at 01:02**
> - **73% at 05:15**
>
> Application logs show **only startup messages** with no runtime logs, indicating that the container is either killed before handling requests or memory is exhausted during startup/initialization.
>
> CPU utilization remains consistently low (**0.3–5%**), ruling out CPU-related causes.
>
> The ECR container image was pushed at **01:01:49**, confirming a **new deployment** that introduced the memory leak. No metrics exist before the deployment, confirming this issue did not occur prior to the new version.
>
> These repeated OOM container terminations cause cascading **502 errors** at the Application Load Balancer (ALB) layer, as backend connections cannot be established during container restart transitions.
:::

#### 5.1.3 Mitigation Plan

**提案された Mitigation Plan（要約）**

> **Increase ECS task memory allocation from 512 MB to 1024 MB to reduce OOM restart frequency while code fix is developed**
>
> 根本原因は **2025-12-18T01:30:08Z のデプロイで導入されたコードレベルのメモリリーク**です。恒久対応にはコード修正が必要ですが、**即時の運用緩和策**として メモリ割当を 512MB → 1024MB に増加することを推奨しています。
>
> これにより OOM 発生までの時間が約 8-15 分から約 15-30 分に延長され、再起動頻度と ALB での 502 エラーを軽減できます。
>
> **この緩和策はメモリリークを解決するものではなく、一時的な措置としてのみ扱うべきです。**

:::details Mitigation Plan のスクリーンショット
![Mitigation Plan](/images/aws-devops-agent-investigation/scenario05/mitigation_plan.png)
:::


#### 5.1.4 結果サマリー

| 評価項目 | 結果 | 備考 |
|---------|------|------|
| OOM の検知 | ✅ | Exit Code 137 を特定 |
| メモリ使用パターンの分析 | ✅ | 時系列での使用率推移を把握 |
| デプロイとの相関 | ✅ | ECR push 時刻との関連を特定 |
| 一時対応の提案 | ✅ | メモリ増加（512→1024MB） |
| 恒久対応の必要性指摘 | ✅ | コード修正が必要と明記 |
| 調査所要時間 | 約11分 | |

#### 5.1.5 考察

**ECS でも高精度な分析が可能**

Lambda シナリオと同様に、DevOps Agent は ECS Fargate 環境でも高い精度で原因を特定できました。

- **Exit Code 137（OOM Kill）** を検出
- メモリ使用率の時系列推移（81%、69%、73% など）を具体的に提示
- **ECR への push 時刻**と障害発生の相関を自動的に分析
- CPU 使用率が低いことから CPU 起因を除外する切り分け

**一時対応と恒久対応の明確な区別**

「メモリを増やせば延命できるが、根本解決にはコード修正が必要」という判断は妥当です。
「とりあえず再起動」「とりあえずスケールアップ」があくまでも一時対応であることを明確に示してくれる点が良いですね。

**GitHub 連携**

今回 GitHub と連携させましたが、**デプロイパイプラインの履歴参照**であって、**ソースコードの内容を読んでくれるわけではない**ようです。「そのデプロイでどんなコード変更があったか」までは踏み込めていませんでした（私が仕様を誤解していました）。

ソースコードレベルの分析が必要な場合は、別途 MCP サーバーを用意して連携させる等の対応が必要そうです（未検証）。

:::message
Lambda と ECS では「DevOps Agent がコードにどこまで近づけるか」の前提が異なります。Lambda は関数コードが AWS 上に（スクリプトとして）デプロイされているため、DevOps Agent がスクリプトの内容にアクセスできる前提で調査が進みます。一方 ECS はコンテナイメージをデプロイするため、DevOps Agent がソースコードを直接参照できず、コード位置の特定はログ出力（panic のスタックトレース等）に依存します。
:::

### 5.2 シナリオ 2-2: CPU スパイク

#### 5.2.1 障害の概要

`api-gateway-service`（ECS Fargate）で CPU を使い切る処理を実行し、CPU スパイクを発生させます。

```go
// cpuSpikeHandler はCPUスパイクを発生させる
// 複数のgoroutineで60秒間ビジー状態を維持
func cpuSpikeHandler(w http.ResponseWriter, r *http.Request) {
	duration := 60 * time.Second
	numWorkers := 16 // 16個のgoroutineで負荷をかける

	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(idx int) {
			defer wg.Done()
			endTime := time.Now().Add(duration)
			for time.Now().Before(endTime) {
				// 素数判定、行列計算、フィボナッチ等の重い処理
				isPrimeHeavy(1000000)
				matrixMultiply(50)
				fibonacciSlow(30)
			}
		}(i)
	}
	wg.Wait()
	// ...
}
```

**設定**:
- メモリ: 512MB
- CPU: 0.25 vCPU（256 units）

#### 5.2.2 Root Cause

![Root Cause](/images/aws-devops-agent-investigation/scenario06/root_cause.png)

**要約**
> `api-gateway-service` が起動後約 17 秒で「Server starting on port 8080」ログを出力した直後、無限ループまたはブロッキング処理に入り CPU を 99.9%（255.83 / 256 units）消費し続けています。2025年12月15日以降の全 7 タスクで同じパターンが発生しています。アプリケーションは初期化を完了せず、ヘルスチェックにも応答せず、エラーログも出力されていません。無限ループ、タイムアウトなしのブロッキング処理、CPU バウンドな計算、または終了条件のない再帰呼び出しが原因と推測されます。

:::details DevOps Agent の原文出力（Root Cause）
> **Root cause**
> Application code contains infinite loop or blocking operation in startup sequence causing 99.9% CPU consumption
>
> The true root cause is an **application-level bug in the container's startup code**.
>
> The application successfully logs `Server starting on port 8080` approximately **17 seconds after container launch**, and then immediately enters a **silent, CPU-intensive operation** that consumes **255.83 CPU units (99.9% of allocated 256 units)** continuously for **10+ minutes**.
>
> This pattern is **systematic** and affects **all 7 task deployments since December 15, 2025**. Each task produces exactly **one log message**, then goes silent while consuming maximum CPU. The application:
> - Never completes initialization
> - Never responds to health checks
> - Never handles HTTP requests
> - Produces **zero error logs**
>
> The most probable causes include:
> 1. An **infinite loop** in configuration loading or dependency initialization
> 2. A **blocking synchronous operation without timeout**
> 3. A **CPU-bound computation that never completes**
> 4. A **recursive function call without a proper termination condition**
:::

#### 5.2.3 Mitigation Plan

**提案された Mitigation Plan（要約）**

> **Roll back ECS service to pre-incident container image from December 12, 2025**
>
> 2025年12月15日頃に導入されたコード欠陥により、全てのコンテナタスクが `Server starting on port 8080` ログ出力直後に**無限ループまたはブロッキング処理**に入り、**CPU 使用率 99.9%** が 10 分以上継続しています。
>
> ECR イメージ履歴から **2025年12月12日** の正常イメージが確認できたため、そのイメージへのロールバックを推奨しています。具体的なロールバック手順（AWS CLI コマンド）も提示されました。

:::details Mitigation Plan のスクリーンショット
![Mitigation Plan](/images/aws-devops-agent-investigation/scenario06/mitigation_plan.png)
:::

#### 5.2.5 結果サマリー

| 評価項目 | 結果 | 備考 |
|---------|------|------|
| CPU スパイクの検知 | ✅ | 99.9% 消費を特定 |
| 原因の推定 | ✅ | 無限ループ/ブロッキング処理と推定 |
| デプロイとの相関 | ✅ | 12/15 以降の全タスクで発生と特定 |
| ロールバック提案 | ✅ | 12/12 のイメージへのロールバック |
| 調査所要時間 | 約21分 | 再指示を含む |

#### 5.2.6 考察

DevOps Agent は最初「CloudFormation デプロイ時のタスク置き換え」という期待と異なる Root Cause を提示しましたが、Chat UI から「それ以外の根本原因を探して」と再指示したところ、無限ループ/ブロッキング処理の可能性にたどり着きました。

このシナリオの事象はあくまで **CPU 使用率が 99.9% 近くまで張り付く CPU スパイク**ですが、DevOps Agent はログのパターン（起動ログが 1 行だけで以降沈黙）と CPU の継続的な高止まりから、原因候補として「無限ループ」や「タイムアウトのないブロッキング処理」といった **CPU バウンドなコード不具合を推測**していました。ここで重要なのは、これはソースコードを直接見た断定ではなく、観測された状況からの仮説である点です。

実運用でこの仮説の確度を上げるなら、CPU 上昇の時間帯に `GET /scenarios/cpu-spike` のような負荷系エンドポイントへのアクセス有無（ALB アクセスログ等）と、アプリケーション側の「リクエスト到達ログ」を突合し、
「リクエスト起因の CPU スパイク」なのか「起動直後から継続する CPU 消費（無限ループ等）」なのかを切り分けるのが有効だと感じました。

DevOps Agent だけでなく他の AI ツールも同様ですが、一発で正解を出すツールではなく **対話しながら調査を深掘りできるツール**として使うのが正しいようです。人間のエンジニアが「いや、それは違うと思う。他の可能性は？」と聞くような使い方ができる点は、実運用で大きな価値があります。

ソースコードが見えない状況でも CPU 使用率とログのパターンから「無限ループの可能性」まで推測できている点は興味深いです。

### 5.3 シナリオ 2-3: panic

#### 5.3.1 障害の概要

`api-gateway-service`（ECS Fargate）で意図的に panic を発生させ、コンテナをクラッシュさせます。

```go
// panicHandler は意図的にパニックを発生させる
func panicHandler(w http.ResponseWriter, r *http.Request) {
	log.Println("Panic scenario triggered - application will crash")
	panic("Intentional panic for testing - DevOps Sandbox")
}
```

**設定**:
- メモリ: 512MB
- CPU: 0.25 vCPU

#### 5.3.2 Root Cause

![Root Cause](/images/aws-devops-agent-investigation/scenario07/root_cause.png)

**要約**
> `api-gateway-service` が `/app/main.go:315` の意図的な panic テストコードによりクラッシュしました。コンテナは exit code 2 で終了し、2 つのタスク（2025-12-19T00:16:59Z と 00:13:28Z）がクラッシュしました。CPU 使用率（<1%）、メモリ使用率（<0.4%）が低く、インフラの問題ではなくコードレベルの問題と判断されています。

:::details DevOps Agent の原文出力（Root Cause）
> **Root cause**
> Intentional panic test code in application causing ECS task crashes
>
> The `api-gateway-service` application contains **intentional panic test code** located at `/app/main.go:315`. This code triggers a panic with the message `Intentional panic for testing - DevOps Sandbox`.
>
> When executed, this test scenario causes the Go application to crash with a panic, resulting in the container exiting with **exit code 2 (application error)**.
>
> Two ECS tasks crashed as a result of this panic:
> - Task `9e47ae6049fd44a78eb6f9dbab667c9a` at **2025-12-19T00:16:59Z**
> - Task `487a697364284f2f8396caebc3c51add` at **2025-12-19T00:13:28Z**
>
> Both tasks logged **"Panic scenario triggered – application will crash"** immediately before the panic and stack trace.
>
> This is a **code-level issue**, not an infrastructure problem:
> - CPU utilization remained extremely low (**<1%**)
> - Memory usage remained low (**<0.4%**)
> - No infrastructure changes occurred
:::

#### 5.3.3 Mitigation Plan

**提案された Mitigation Plan（要約）**

> **Roll back ECS service to previous container image without panic test code**
>
> `api-gateway-service` の ECS タスクが `/app/main.go:315` の**意図的な panic テストコード**によりクラッシュしています。コンテナは **exit code 2** で終了し、タスクの再起動が繰り返されています。
>
> panic テストコードを含まない**以前の安定したコンテナイメージへのロールバック**を推奨しています。具体的なロールバック手順と、ロールバック後の検証コマンドも提示されました。

:::details Mitigation Plan のスクリーンショット
![Mitigation Plan](/images/aws-devops-agent-investigation/scenario07/mitigation_plan.png)
:::

#### 5.3.4 結果サマリー

| 評価項目 | 結果 | 備考 |
|---------|------|------|
| panic の検知 | ✅ | Exit Code 2 を特定 |
| panic メッセージの特定 | ✅ | ログから抽出 |
| コード位置の特定 | ✅ | `/app/main.go:315` |
| 意図的なテストコードと認識 | ✅ | メッセージから推測 |
| ロールバック提案 | ✅ | 以前のイメージへ |
| 調査所要時間 | 約6分 | |

#### 5.3.5 考察

DevOps Agent は CloudWatch Logs から panic メッセージとスタックトレースを正確に抽出し、panic が発生したファイルと行番号（`/app/main.go:315`）、panic メッセージの内容、クラッシュした具体的なタスク ID と時刻を特定しました。なお、このファイルパスと行番号はソースコードを直接読んだのではなく、**ログに出力されたスタックトレースから取得したもの**です。

特筆すべきは、panic メッセージに「Intentional」「testing」「Sandbox」という単語が含まれていたことから、「意図的に仕込まれたテストコードが本番に残っている」と正しく判断した点です。Root Cause のサマリーでも「Intentional panic test code left enabled in production」と明記されており、単なるバグではなくテストコードの消し忘れという状況を理解しています。
（やはり実験とはいえ現実に即した命名にしておくべきだったとは思います）

また、CPU 使用率（<1%）、メモリ使用率（<0.4%）が低いことから、「これはインフラの問題ではなくコードの問題」と明確に切り分けている点も良かったです。

### 5.4 シナリオ 2-4: DB 接続エラー

#### 5.4.1 障害の概要

`api-gateway-service`（ECS Fargate）で DB 接続エラーを発生させます。今回は**意図的にエラーログを出力しない**ようにしており、DevOps Agent がログなしでどこまで原因を特定できるかを検証します。

```go
// dbErrorHandler はDB接続エラーを発生させる
func dbErrorHandler(w http.ResponseWriter, r *http.Request) {
	dbHost := os.Getenv("DB_HOST")
	if dbHost == "" {
		dbHost = "invalid-db-host.local"
	}

	// DB接続を試行（認証情報なしで失敗する）
	connStr := fmt.Sprintf("host=%s port=5432 dbname=testdb sslmode=disable", dbHost)
	db, err := sql.Open("postgres", connStr)
	if err != nil {
		// 標準出力にはエラーログを出力しない（わざと調査を難しくするため）
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}
	defer db.Close()
	// ...
}
```

**設定**:
- メモリ: 512MB
- CPU: 0.25 vCPU
- DB_HOST: Aurora エンドポイント（認証情報なし）

#### 5.4.2 Root Cause

![Root Cause](/images/aws-devops-agent-investigation/scenario08/root_cause.png)

**要約**
> ECS タスク定義に DB 認証情報（ユーザー名とパスワード）が含まれておらず、Aurora PostgreSQL に接続できません。Aurora メトリクスでは 17 分以上タスクが実行されているにもかかわらず接続数ゼロ、アプリケーションログにも DB 接続試行やエラーが一切出力されていない、ECS から Aurora へのネットワークトラフィックも最小限（~0.47 bytes/sec）という状況から判断されました。CloudFormation で Secrets Manager に認証情報は作成されていますが、タスク定義でコンテナ環境に注入されていないことが原因です。

:::details DevOps Agent の原文出力（Root Cause）
> **Root cause**
> ECS task definition missing database credentials (username and password)
>
> The ECS task definition for `api-gateway-service` is configured with the database host via the `DB_HOST` environment variable. However, the task definition contains **no database username or password credentials** in either environment variables or secrets configuration.
>
> Without credentials, the application cannot authenticate to the Aurora PostgreSQL database. This explains the observed behavior:
> 1. **Aurora metrics show zero connections** despite ECS tasks running for over **17 minutes**.
> 2. **Application logs show zero database connection attempts or errors**.
> 3. **Network traffic from ECS to Aurora is minimal** (~0.47 bytes/sec), indicating no connection attempts reach the database.
> 4. The ECS task **passes ALB health checks** and reports **HEALTHY**, because the health check endpoint does not validate database connectivity.
>
> CloudFormation created database credentials in **AWS Secrets Manager** at **2025-12-19T01:52:11 UTC**. However, the ECS task definition was **not configured to inject these secrets** into the container environment.
:::

#### 5.4.3 Mitigation Plan

**提案された Mitigation Plan（要約）**

> **Update ECS task definition to inject database credentials from AWS Secrets Manager and grant IAM permissions for credential retrieval**
>
> ECS サービスが Aurora PostgreSQL に接続できないのは、タスク定義に DB 認証情報（ユーザー名とパスワード）が含まれていないためです。AWS Secrets Manager に認証情報は作成されていますが、**コンテナ環境に注入されていません**。
>
> **緩和策**:
> 1. `DB_USER` と `DB_PASSWORD` を Secrets Manager から注入するようタスク定義を更新します
> 2. ECS タスク実行ロールに Secrets Manager へのアクセス権限を付与します
> 3. サービスを再デプロイします
>
> 具体的な AWS CLI コマンド、IAM ポリシー、ロールバック手順まで提示されました。

:::details Mitigation Plan のスクリーンショット
![Mitigation Plan](/images/aws-devops-agent-investigation/scenario08/mitigation_plan.png)
:::

#### 5.4.4 結果サマリー

| 評価項目 | 結果 | 備考 |
|---------|------|------|
| DB 接続問題の検知 | ✅ | Aurora メトリクスから推測 |
| 認証情報不足の特定 | ✅ | タスク定義を分析 |
| Secrets Manager の存在確認 | ✅ | CloudFormation 履歴から特定 |
| 修正手順の提案 | ✅ | IAM + タスク定義更新 |
| 調査所要時間 | 約16分 | |

#### 5.4.5 考察

今回のシナリオでは意図的にエラーログを出力しない設定にしていました。
通常、エラーログがなければ原因特定は困難ですが、DevOps Agent は Aurora の DatabaseConnections メトリクスが 0 であること、ECS からのネットワークトラフィックが極めて少ないこと、タスク定義の環境変数に `DB_HOST` はあるが認証情報がないこと、Secrets Manager に認証情報が存在することを CloudFormation 履歴から特定するなど、複数のデータソースを横断的に分析して原因に迫っていました。

CloudFormation の履歴から「Secrets Manager に認証情報が作成されたがタスク定義に注入されていない」という設定ミスを特定できた点は興味深く、IaC のリソース間の依存関係を理解した上での分析と言えます。

また、「ALB ヘルスチェックは HEALTHY だが、それは DB 接続性を検証していないから」という指摘も的確で、ヘルスチェックの設計に関する改善提案とも受け取れました。

## 6. 自作 Agent（Mahavat Agent）との比較


### 6.1 Mahavat Agent の概要

re:Invent 2025 の別セッション「DAT301 - AI Powered PostgreSQL: Incident Detection & MCP Integration」で体験した Agent です。
DevOps Agent の発表直後に参加したワークショップだったのですが「これって DevOps Agent 的な Agent を自前で実装したやつでは？」と感じたので、DevOps Agent と比較する視点で見てみようと思います。

ソースコードは GitHub に公開されています。
https://github.com/aws-samples/sample-dat301-PG-IDR

![Mahavat Agent](/images/aws-devops-agent-investigation/mahavat/mahavat.png)

なぜかワークショップ中の画面キャプチャがこれしか残っていなかったのですが（大失態）、DevOps Agent と似た画面構成になっており、 CloudWatch Alarm からインシデントを検知するとこの画面に表示され、それを起点に Agent に調査を依頼することができます。
特徴的な点は、種々の MCP サーバと連携し、Aurora PostgreSQL に特化した調査を行うエージェントになっているということです。

**アーキテクチャ**
- **フレームワーク**: Strands Agent Framework + MCP
- **LLM**: Amazon Bedrock（Claude Sonnet 4）
- **ベクトル検索**: pgvector（Aurora PostgreSQL）
- **対象**: PostgreSQL のパフォーマンス問題

**主な機能**
- CloudWatch アラームからのインシデント検知
- pgvector による類似インシデント検索
- Runbook の自動提案
- Performance Insights との MCP 連携

### 6.2 機能比較

| 観点 | DevOps Agent | Mahavat Agent（自作） |
|------|--------------|---------------------|
| **対象サービス** | AWS 全般 | PostgreSQL に特化 |
| **データソース** | CloudWatch + 外部ツール + GitHub | CloudWatch + Performance Insights |
| **調査の深さ** | Lambda 内部状態、X-Ray トレース | DB メトリクス、slow query |
| **調査の広さ** | マルチサービス相関、デプロイ履歴 | 単一サービスだが拡張可能 |
| **透明性** | ブラックボックス | ホワイトボックス（コード参照可） |
| **カスタマイズ** | BYO MCP で拡張 | 自由に拡張可能 |
| **セットアップ** | GUI | コーディングが必要 |

### 6.3 DevOps Agent の強み

1. **セットアップの容易さ**: Agent Space を作るだけで開始可能
2. **調査の広さ**: 複数サービス、デプロイ履歴まで横断的に分析
3. **外部ツール連携**: Datadog, Splunk 等との統合が簡単

### 6.4 自作 Agent の強み

1. **透明性**: Agent の思考プロセスをコードレベルで理解可能
2. **カスタマイズ性**: ドメイン固有のロジックを自由に組み込める
3. **コスト管理**: Bedrock API の料金のみで運用可能
4. **特化型の深さ**: 特定サービス（例: PostgreSQL）に対する深い分析

### 6.5 使い分けの指針

今回の検証を通じて、DevOps Agent と自作 Agent の使い分けについて、以下のような指針が見えてきました。

DevOps Agent の最大の強みは、Agent Space を作るだけで即座に使い始められる手軽さです。「とりあえず全サービスに Agent Space を作っておけば、いざという時に助かる」という使い方もできますし、Datadog や Splunk などの外部ツールとの連携が簡単なので、既存の監視基盤と統合しやすい点も魅力です。特別な設計や準備をしなくても、CloudWatch Alarm が発火した時点で「とりあえず Agent に投げてみる」だけで、一次対応の工数を削減できます。

一方、自作 Agent は課題をはっきりさせてから、それ専用に尖らせた設計にする形になると思いました。例えば、「PostgreSQL の slow query 問題が頻発しているから、Performance Insights と pgvector を使った専用 Agent を作る」とか、「チーム固有のデプロイフローに合わせた調査手順を組み込む」といった、プロジェクト特化型・タスク特化型の Agent です。最近では、Agent Guardrails（Agent の動作範囲を制限する仕組み）や Amazon Bedrock Agentcore MCP Server（Bedrock Agent を MCP から呼び出せる公式サーバー）などのツールが登場し、業務で自作 Agent を使用するハードルが下がってきています。課題が明確で、Agent をメンテナンスする手間に見合うなら、自作 Agent も十分に選択肢に入ってくると感じました。

両者の一番大きな違いはカスタマイズ性です。DevOps Agent は汎用的で手軽ですが、組織固有のナレッジベースや特殊な調査フローを組み込むことは難しそうです。自作 Agent なら、例えば「社内 Wiki の過去インシデント記事を RAG で検索」「特定の Runbook を自動実行」といった、組織特化の機能を自由に実装できます。

## 7. まとめ

### 7.1 DevOps Agent の価値

今回の検証を通じて確認できた DevOps Agent の価値は以下のとおりです。

#### **1. 横断的なデータソース分析**

CloudWatch メトリクス、ログ、ECR イメージ履歴、Secrets Manager、CloudFormation 履歴など、複数のデータソースを横断して相関分析できます。シナリオ 2-4 のようにエラーログがない状況でも、Aurora のメトリクスやタスク定義から原因を特定できた点は印象的でした。

#### **2. 一次対応と恒久対応の明確な区別**

「とりあえずメモリを増やして延命する」といった一時的な緩和策と、「コード修正が必要」という恒久対応を明確に分けて提案してくれます。運用現場で「これは一時対応です」と明示してくれるのは、後続の対応を判断する上で非常に助かります。

#### **3. 非同期での調査**

人間が別の作業をしている間にバックグラウンドで調査を進めてくれます。深夜のオンコール対応で「Agent に投げて、ユーザー対応等の別タスクを並行して進める」といった使い方ができる点は、実運用で大きな価値があると感じました。

### 7.2 運用上の Tips

今回の検証で得られた、DevOps Agent を効果的に使うための Tips をまとめます。

- **リソース名は AI にとっても重要なコンテキスト**
  - 「OomLambdaLogGroup」という名前から「OOM テストかも」と推測されたように、リソースの命名規則は AI の分析精度にも影響します。

- **対話しながら深掘りする**
  - 最初の回答が期待と異なる場合もあります。思考の過程も随時アップデートしてくれるので、様子を見ながら「それ以外の原因を探して」「コードの中身も見て」など、人間のエンジニアに指示するように対話することで精度が上がります。

- **非同期で活用する**
  - 調査に 6〜20 分かかるため、「Agent に投げて別タスクを進める」という使い方が効果的です。

- **調査時間は 6〜21 分程度のばらつきがある**
  - 今回の検証では、シンプルな単一原因（TypeError、panic）は 6〜7 分、複数の可能性を探索する必要があるケース（タイムアウト、CPU スパイク）は 13〜21 分かかりました。
  - ログが明確でスタックトレースがある場合は速く、ログがない・複数のデータソースを横断する必要がある場合は時間がかかる傾向があります。
  - 初回の回答が期待と異なり再指示が必要な場合（シナリオ 2-2）は、その分だけ追加の時間がかかります。

### 7.3 今後への期待

公式ブログによると、近日中に以下の機能が追加予定です。

- **コードバグの分析**
  - 障害原因がコードバグの場合の詳細分析
- **テストカバレッジの改善提案**
  - テスト不足の検知と改善提案

これらが実装されれば、「一次対応」から「恒久対応」への接続がよりスムーズになることが期待されます。

### 7.4 最後に

DevOps Agent は Preview 段階ですが、今回の検証を通じて実用的なレベルに達していると感じました。Lambda や ECS の障害に対して、6〜20 分程度で根本原因を特定し、具体的な緩和策まで提案してくれます。GA になって東京リージョンに展開されたら、ぜひプロジェクトで活用したいと考えています。

一方で、今回の検証で自作 Agent という選択肢も改めて意識するようになりました。課題が明確で、特定のサービスやワークフローに特化した調査が必要な場合は、自作 Agent の方が適しているケースもあると思いました。Agent Guardrails や MCP SDK など、自作 Agent を構築するためのエコシステムも充実してきているので、「汎用的な初動対応は DevOps Agent」「特化型の深い分析は自作 Agent」という使い分けを検討していきたいです。

## 参考リンク

- [AWS DevOps Agent 公式ページ](https://aws.amazon.com/devops-agent)
- [AWS News Blog - AWS DevOps Agent helps you accelerate incident response](https://aws.amazon.com/blogs/aws/aws-devops-agent-helps-you-accelerate-incident-response-and-improve-system-reliability-preview/)
- [DAT301 Workshop - AI Powered PostgreSQL](https://github.com/aws-samples/sample-dat301-PG-IDR)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Strands Agent Framework](https://strandsagents.com/)

