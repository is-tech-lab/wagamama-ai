# Unit of Work — わがママAI

**作成日**: 2026-05-08
**前提**: application-design.md / Q1〜Q6 回答（A / A / A / B / A / C 3人並行）
**配信期限**: **2026-05-12 12:00 書類審査必須成果物**

---

## 1. Unit 概要（4 Unit 構成）

| ID | Unit 名 | 言語 / 技術 | 責務（一行） | デプロイ単位 |
|---|---|---|---|---|
| **U1** | Mobile App | TypeScript + React Native + Expo | ユーザーが触るすべての UI（Chat / Dashboard / Onboarding） | EAS Build → TestFlight / Play Store |
| **U2** | Backend (Mono Hono Lambda) | TypeScript + Hono | Clean Architecture 3 層、F1/F2/F5/F6 の API + 内部イベントハンドラ | CDK で AWS Lambda Function |
| **U3** | Agent Runtime | Python + Nova Act SDK + AgentCore | F3 Nova Act ワークフロー（実 Web 操作） | CDK で AgentCore Runtime（Docker 経由） |
| **U4** | Infrastructure (CDK) | TypeScript + AWS CDK | 全 AWS リソースの IaC（**3 スタック構成**：Q4=B） | `cdk deploy --all` |
| **共通** | shared package | TypeScript（型のみ） | API DTO・EventBridge イベントスキーマ・共通型 | npm workspace 内ローカル参照 |

---

## 2. リポジトリ構成（Q2=A モノレポ + pnpm workspace）

```text
wagamama-ai/                                         # GitHub リポジトリルート
├── package.json                                     # workspace ルート
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── README.md                                        # プロダクトナラティブ + デモシナリオ
├── packages/
│   ├── mobile/                                      # 🟢 U1: Mobile App
│   │   ├── app/                                     # Expo Router screens
│   │   │   ├── (tabs)/index.tsx                     # ChatUI
│   │   │   ├── (tabs)/dashboard.tsx                 # DashboardUI
│   │   │   └── onboarding.tsx                       # OnboardingUI
│   │   ├── src/
│   │   │   ├── components/                          # 共通 UI コンポーネント
│   │   │   ├── modules/auth/                        # AuthModule (expo-auth-session)
│   │   │   ├── modules/api/                         # APIClient
│   │   │   ├── modules/notifications/               # NotificationService
│   │   │   ├── stores/                              # Zustand
│   │   │   └── types/                               # ローカル型（shared を補完）
│   │   ├── app.json                                 # Expo 設定
│   │   ├── eas.json                                 # EAS Build 設定
│   │   └── package.json
│   │
│   ├── backend/                                     # 🟢 U2: Backend (Mono Hono Lambda)
│   │   ├── src/
│   │   │   ├── domain/                              # ⭐ PBT 主対象（Pure Logic）
│   │   │   │   ├── chat-message.ts
│   │   │   │   ├── user-profile.ts
│   │   │   │   ├── dameness-metrics.ts
│   │   │   │   ├── metrics-counters.ts
│   │   │   │   ├── metrics-event.ts
│   │   │   │   ├── advice-content.ts
│   │   │   │   ├── reservation-request.ts
│   │   │   │   ├── pure-functions/                  # applyEvent / computeMetrics 等
│   │   │   │   │   ├── apply-event.ts
│   │   │   │   │   ├── compute-metrics.ts
│   │   │   │   │   ├── mama-persona-prompter.ts
│   │   │   │   │   └── advice-prompt-builder.ts
│   │   │   │   └── errors.ts
│   │   │   ├── application/
│   │   │   │   ├── use-cases/                       # 10 Use Cases
│   │   │   │   │   ├── send-chat-message.ts
│   │   │   │   │   ├── get-chat-history.ts
│   │   │   │   │   ├── onboard-user.ts
│   │   │   │   │   ├── generate-morning-advice.ts
│   │   │   │   │   ├── generate-lunch-advice.ts
│   │   │   │   │   ├── trigger-reservation.ts
│   │   │   │   │   ├── process-reservation-request.ts
│   │   │   │   │   ├── handle-reservation-completed.ts
│   │   │   │   │   ├── handle-reservation-failed.ts
│   │   │   │   │   ├── get-metrics-dashboard.ts
│   │   │   │   │   └── update-metrics.ts
│   │   │   │   └── services/                        # Application Service (横断)
│   │   │   │       └── external-data-service.ts     # ExternalDataService
│   │   │   ├── infrastructure/
│   │   │   │   ├── repos/                           # 4 Repositories
│   │   │   │   │   ├── chat-repo.ts
│   │   │   │   │   ├── user-profile-repo.ts
│   │   │   │   │   ├── metrics-repo.ts
│   │   │   │   │   └── reservation-state-repo.ts
│   │   │   │   ├── clients/                         # AWS / 外部 API
│   │   │   │   │   ├── bedrock-claude-client.ts
│   │   │   │   │   ├── bedrock-titan-image-client.ts
│   │   │   │   │   ├── s3-image-storage.ts
│   │   │   │   │   ├── agentcore-runtime-invoker.ts
│   │   │   │   │   ├── eventbridge-publisher.ts
│   │   │   │   │   ├── ses-email-sender.ts
│   │   │   │   │   ├── push-notification-sender.ts
│   │   │   │   │   ├── weather-api-client.ts
│   │   │   │   │   ├── hotpepper-api-client.ts
│   │   │   │   │   └── google-calendar-client.ts
│   │   │   │   ├── handlers/                        # Hono routes
│   │   │   │   │   ├── public/                      # API GW 経由
│   │   │   │   │   │   ├── chat.ts
│   │   │   │   │   │   ├── onboarding.ts
│   │   │   │   │   │   ├── reservations.ts
│   │   │   │   │   │   └── metrics.ts
│   │   │   │   │   ├── internal/                    # EventBridge / Scheduler 経由
│   │   │   │   │   │   ├── morning-advice.ts
│   │   │   │   │   │   ├── lunch-advice.ts
│   │   │   │   │   │   ├── reservation-requested.ts
│   │   │   │   │   │   ├── reservation-completed.ts
│   │   │   │   │   │   ├── reservation-failed.ts
│   │   │   │   │   │   └── metrics-update.ts
│   │   │   │   │   └── eventbridge-adapter.ts       # EventBridge イベント → Hono ルート
│   │   │   │   └── idempotency/                     # Powertools Idempotency 設定
│   │   │   │       └── config.ts
│   │   │   └── lambda-entry.ts                      # Lambda handler エントリポイント
│   │   ├── tests/
│   │   │   ├── unit/                                # Vitest
│   │   │   ├── property/                            # ⭐ fast-check (PBT)
│   │   │   └── integration/                         # localstack / aws-sdk-mock
│   │   └── package.json
│   │
│   ├── agent/                                       # 🟢 U3: Agent Runtime (Python)
│   │   ├── pyproject.toml                           # uv / poetry
│   │   ├── Dockerfile                               # AgentCore Runtime デプロイ用
│   │   ├── src/
│   │   │   ├── workflows/
│   │   │   │   └── reservation_workflow.py          # F3 メインワークフロー
│   │   │   ├── steps/
│   │   │   │   ├── execute_reservation.py           # Nova Act 操作
│   │   │   │   ├── register_to_calendar.py          # Google Calendar 登録
│   │   │   │   └── publish_completion.py            # EventBridge 通知
│   │   │   └── lib/
│   │   │       ├── eventbridge_notifier.py
│   │   │       └── types.py
│   │   └── tests/
│   │
│   ├── infra/                                       # 🟢 U4: Infrastructure (CDK)
│   │   ├── bin/wagamama.ts                          # CDK エントリ
│   │   ├── lib/
│   │   │   ├── data-stack.ts                        # DataStack
│   │   │   ├── app-stack.ts                         # AppStack
│   │   │   ├── agent-stack.ts                       # AgentStack
│   │   │   └── constructs/                          # 共通コンストラクト
│   │   ├── cdk.json
│   │   ├── tests/                                   # cdk-nag / cfn-lint テスト
│   │   └── package.json
│   │
│   └── shared/                                      # 🟢 共通：型・契約のみ
│       ├── src/
│       │   ├── api/                                 # API DTO
│       │   │   ├── chat.ts
│       │   │   ├── onboarding.ts
│       │   │   ├── reservations.ts
│       │   │   └── metrics.ts
│       │   ├── events/                              # EventBridge イベントスキーマ
│       │   │   ├── reservation-requested.ts
│       │   │   ├── reservation-completed.ts
│       │   │   ├── reservation-failed.ts
│       │   │   └── metrics-updated.ts
│       │   └── types/                               # 共通プリミティブ
│       │       ├── consent-scope.ts
│       │       ├── persona-flags.ts
│       │       └── geo.ts
│       └── package.json
│
├── aidlc-docs/                                      # AI-DLC 成果物（既存）
└── .github/
    └── workflows/                                   # CI: lint, test, build, deploy
```

> **U2 サブモジュール（参考用、独立 Unit ではない）**：旧 application-design.md §8 で挙げた U3 Advice Pipeline / U5 Metrics & Events / U6 Data Integration は U2 内部の `application/use-cases/`, `application/services/`, `domain/` に分散する論理モジュール。

---

## 3. 各 Unit の責務詳細

### 3.1 U1: Mobile App

| 項目 | 内容 |
|---|---|
| **言語/Framework** | TypeScript / React Native (Expo SDK) / Zustand / expo-auth-session / expo-secure-store / expo-notifications |
| **責務** | ユーザータッチポイント全体（Chat / Dashboard / Onboarding） |
| **対応 Story** | E1-S1, E1-S2, E1-S3, E4-S1（UI 部分）、F2/F3/F5 の Push 受信表示 |
| **デプロイ** | EAS Build → iOS Simulator（5/30 予選） / TestFlight or Play Store Internal Testing（6/26 決勝） |
| **ビルド** | `pnpm --filter mobile expo export` / `eas build --profile preview` |
| **テスト** | Jest + React Native Testing Library + Detox (E2E) |
| **依存先 Unit** | `@wagamama/shared`（型のみ）、U2 Backend（REST API 経由） |

### 3.2 U2: Backend (Mono Hono Lambda)

| 項目 | 内容 |
|---|---|
| **言語/Framework** | TypeScript / Hono / @aws-sdk/* / @aws-lambda-powertools/idempotency / fast-check |
| **責務** | F1/F2/F5/F6 の全機能 + F3 のオーケストレーション層（Nova Act 起動・完了処理） |
| **対応 Story** | 全 16 ストーリーのバックエンド層 |
| **デプロイ** | esbuild でバンドル → CDK が Lambda Function にデプロイ |
| **ビルド** | `pnpm --filter backend build`（esbuild） |
| **テスト** | Vitest（unit）+ fast-check（PBT、Domain 層）+ aws-sdk-client-mock（integration） |
| **依存先 Unit** | `@wagamama/shared` |
| **被依存 Unit** | U1 Mobile（REST 経由）、U3 Agent Runtime（InvokeAgentRuntime / EventBridge） |
| **内部モジュール** | Domain / Application（Use Cases + Services）/ Infrastructure（Repos + Clients + Handlers） |

### 3.3 U3: Agent Runtime (Python)

| 項目 | 内容 |
|---|---|
| **言語/Framework** | Python 3.11+ / nova-act / bedrock-agentcore SDK |
| **責務** | F3 のブラウザ操作部分（Nova Act + AgentCore Browser + Calendar API） |
| **対応 Story** | E3-S1, E3-S2, E3-S3, E3-S5 |
| **デプロイ** | CDK が Dockerfile を AgentCore Runtime にビルド・デプロイ（DockerImageAsset） |
| **ビルド** | `docker build -t wagamama-agent packages/agent/` |
| **テスト** | pytest + ローカル Nova Act SDK |
| **依存先 Unit** | （Python 単独、shared 型は使わず Python 側で定義） |
| **被依存 Unit** | U2 Backend が `InvokeAgentRuntime` で起動 |
| **特殊事項** | AgentCore Identity の `@requires_access_token` デコレータで Google OAuth トークン自動注入 |

### 3.4 U4: Infrastructure (CDK)

**Q4=B 複数 CDK スタック構成**：3 スタックで責務分離（VPC 不要のため NetworkStack はなし）。

| Stack | 責務 | 主なリソース |
|---|---|---|
| **DataStack** | 永続化リソース（**先にデプロイ、最後に削除**） | DynamoDB（6 テーブル：ChatMessages / UserProfiles / MetricsCounters / Metrics / Reservations / IdempotencyStore）、S3（画像）、KMS Key |
| **AppStack** | API + Compute + Events（DataStack に依存） | API Gateway (REST), Hono Lambda Function, EventBridge Bus（`wagamama-events`）, EventBridge Scheduler (cron: 6:30 / 11:30 / 22:00), SES, Lambda Permissions |
| **AgentStack** | エージェント実行環境（DataStack + AppStack に依存） | AgentCore Runtime (DockerImageAsset), AgentCore Browser, AgentCore Identity (Google OAuth Provider), IAM Roles for Nova Act |

| 項目 | 内容 |
|---|---|
| **言語/Framework** | TypeScript / AWS CDK v2 / cdk-nag |
| **対応 Story** | 全 Story の基盤 |
| **デプロイ** | `cdk deploy --all`（依存関係に従って Data → App → Agent の順にデプロイ） |
| **テスト** | `cdk synth` + cdk-nag（セキュリティ静的解析）+ `cdk diff`（CI で確認） |
| **依存先 Unit** | U2 Backend（Lambda コード）、U3 Agent Runtime（Docker イメージ） |

### 3.5 共通：shared package

| 項目 | 内容 |
|---|---|
| **責務** | U1 と U2 の間で共有する型のみ（実行時コードなし） |
| **内容** | API DTO（Request/Response）、EventBridge イベントスキーマ、共通プリミティブ（ConsentScope, GeoPoint 等） |
| **ビルド** | `tsc` で declaration のみ出力（または ESM ソース直接 import） |
| **使用方法** | `import type { ChatMessageRequest } from '@wagamama/shared'` |

---

## 4. チーム編成（Q6=C 3 人並行）

| 担当 | 主担当 Unit | 副担当 | 主な責務 |
|---|---|---|---|
| **Person A**（フロント担当） | U1 Mobile App | shared（API 契約） | UI 実装、UX、Push 通知、OAuth フロー（expo-auth-session） |
| **Person B**（バックエンド担当） | U2 Backend、U3 Agent Runtime | shared（イベント契約） | Hono Lambda、Domain 層、Nova Act ワークフロー、PBT |
| **Person C**（インフラ担当） | U4 Infrastructure | CI/CD | CDK 3 スタック、IAM、KMS、Observability、デプロイパイプライン |

### 並行開発の競合回避

- **shared パッケージは事前合意ファースト**：API DTO とイベントスキーマを 5/8 中に Person B が定義 → A/C は import のみ
- **CDK 3 スタック分離**（Q4=B）：A/B/C それぞれが触る AWS リソースが Stack 単位で分離 → CloudFormation コンフリクト最小化
- **PR 戦略**：
  - feature ブランチ → `develop` への小さい PR を継続的に
  - 大きな構造変更は事前 Slack 共有
  - shared の変更は **必ず PR レビュー**（A/B/C 全員確認）

---

## 5. ビルド・テスト・デプロイの全体フロー

```text
[ローカル開発]
pnpm install
pnpm --filter shared build     # 型を最初に build
pnpm --filter mobile dev       # Person A：Expo dev server
pnpm --filter backend dev      # Person B：Hono local（sst-dev or aws-lambda-runtime-interface-emulator）
docker compose up agent        # Person B：Agent Runtime のローカル検証
pnpm --filter infra cdk synth  # Person C：CDK template 検証

[CI（GitHub Actions）]
1. Lint + Type check（全パッケージ並列）
2. Unit Test + PBT（backend）+ E2E（mobile）
3. cdk synth + cdk-nag（infra）
4. develop ブランチへの merge で：
   - Mobile: EAS Build (preview)
   - Backend: esbuild bundle
   - Agent: Docker build
   - Infra: cdk deploy --all（dev 環境）

[デプロイ順序（CDK 依存）]
DataStack → AppStack → AgentStack
```

---

## 6. ハッカソンマイルストーンとの対応

| 期日 | 必要 Unit | 状態目標 |
|---|---|---|
| **2026-05-12 12:00 書類審査** | 設計のみ（U1〜U4 計画） | INCEPTION 全成果物完成 + GitHub リポ提出 |
| **2026-05-30 予選デモ** | U1 + U2 + U3 + U4 動作 MVP | F1〜F3, F5, F6 が動く状態（フィクスチャユーザー） |
| **2026-06-26 決勝デモ** | U1〜U4 + AgentCore 拡張統合 + AWS デプロイ | OAuth 本番化、Memory/Gateway/Policy/Observability 統合（§5.2.3） |

---

## 7. Quality Gates

- [x] 全 16 ストーリーが Unit に割当済（→ unit-of-work-story-map.md 参照）
- [x] Unit 境界が言語・実行環境・デプロイ単位で明確
- [x] モノレポ + workspace で型共有可能（Q3=A）
- [x] CDK 3 スタックで並行開発のコンフリクト回避（Q4=B）
- [x] 3 人並行開発の責務分担明示（Q6=C）

---

## 8. 参照
- `aidlc-docs/inception/application-design/application-design.md`（コンポーネント詳細）
- `aidlc-docs/inception/application-design/unit-of-work-dependency.md`（依存マトリックス）
- `aidlc-docs/inception/application-design/unit-of-work-story-map.md`（Story 割当）
- `aidlc-docs/inception/plans/unit-of-work-plan.md`（本プラン）
