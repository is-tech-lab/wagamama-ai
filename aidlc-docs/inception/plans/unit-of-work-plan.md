# Unit of Work Plan — わがママAI

**作成日**: 2026-05-08
**前提**: requirements.md / stories.md / personas.md / application-design 5 ドキュメント（承認済）
**Stage**: INCEPTION - Units Generation
**期限**: **2026-05-12 12:00 書類審査必須成果物**

---

## 0. 本プランの読み方

Units Generation の **分解計画**。Q1〜Q6 に `[Answer]:` タグで回答してください。完了後 `done` または `回答完了` で生成（unit-of-work.md / unit-of-work-dependency.md / unit-of-work-story-map.md）に進みます。

---

## 1. Plan 実行チェックリスト（Part 2 実行用）

- [x] **STEP 1**: 全上流ドキュメントをロード
- [x] **STEP 2**: Q1〜Q6 の回答（A/A/A/B/A/C 3人）を反映して Unit 境界確定（4 Unit + shared）
- [x] **STEP 3**: `unit-of-work.md` 生成完了（モノレポ構成・3 CDK スタック・3 人並行体制）
- [x] **STEP 4**: `unit-of-work-dependency.md` 生成完了（依存マトリックス・通信契約・5/8〜5/12 並行スケジュール）
- [x] **STEP 5**: `unit-of-work-story-map.md` 生成完了（16 / 16 Story 割当検証済）
- [x] **STEP 6**: 全 16 ストーリーが少なくとも 1 つの Unit に主担当として割当済を検証
- [x] **STEP 7**: `aidlc-state.md` 更新は次セクションで実施

---

## 2. 出発点：候補 Unit（application-design.md §8 より）

application-design では **7 つの Unit 候補**を提示しましたが、**Q1=A（モノ Hono Lambda）**を採用しているため、再整理が妥当です。

### 推奨案：**4 つの Unit**（Q1 で採用 OR 別案を選ぶ）

| Unit | 中身 | 言語/技術 | デプロイ単位 |
|---|---|---|---|
| **U1: Mobile App** | RN + Expo の全コンポーネント（ChatUI / DashboardUI / OnboardingUI / AuthModule / APIClient / NotificationService / Zustand Stores） | TypeScript + RN + Expo | TestFlight / Play Store / EAS Build |
| **U2: Backend (Mono Hono Lambda)** | Domain + Application（Use Cases / Services）+ Infrastructure（Repos / Clients / Handlers）。Clean Architecture 3 層 | TypeScript + Hono | CDK で Lambda Function としてデプロイ |
| **U3: Agent Runtime** | F3 ワークフロー（Nova Act + AgentCore Browser + AgentCore Identity 連携の Python ワークフロー） | Python + Nova Act SDK | CDK で AgentCore Runtime としてデプロイ（Docker 経由） |
| **U4: Infrastructure (CDK)** | API Gateway, Lambda 設定, DynamoDB（6 テーブル）, S3, EventBridge Bus & Scheduler, SES, KMS, IdempotencyStore, AgentCore リソース | TypeScript + AWS CDK | `cdk deploy` |

> **メリット**: 4 つのデプロイ境界が明確、それぞれ独立してビルド・デプロイ可能。
> **U5/U6/U7 の扱い**: Metrics、Data Integration、Advice Pipeline はすべて U2 内部のモジュールとして配置（モノ Lambda なので物理的に同じ実行環境）。

---

## 3. Mandatory Artifacts

- [ ] `aidlc-docs/inception/application-design/unit-of-work.md` — Unit 定義、責務、コード構成戦略
- [ ] `aidlc-docs/inception/application-design/unit-of-work-dependency.md` — 依存マトリックス
- [ ] `aidlc-docs/inception/application-design/unit-of-work-story-map.md` — Story 割当

---

## 4. Questions（ユーザー回答必須）

### Question 1 — Unit 境界（最重要）

§2 の推奨案（4 Unit）を採用しますか？

A) **採用：4 Unit**（U1 Mobile App / U2 Backend / U3 Agent Runtime / U4 Infrastructure）— Q1=A モノ Lambda と整合的、推奨
B) **より細かく分ける（5〜7 Unit）**：U2 Backend をさらに分解する（例：U2a Public API / U2b Workers / U2c Domain Library）
C) **より粗くする（3 Unit）**：U3 Agent Runtime と U4 Infrastructure を統合
D) Other（[Answer]: に記入）

[Answer]: A

---

### Question 2 — リポジトリ構成

GitHub のリポジトリ構造をどうしますか？（書類審査 5/12 では GitHub URL を提出）

A) **モノレポ + ワークスペース**（pnpm workspace / npm workspaces）。`packages/mobile/`, `packages/backend/`, `packages/agent/`, `packages/infra/`, `packages/shared/` 構成。型共有が容易
B) **モノレポ + フラットディレクトリ**（ワークスペースなし、単純な `apps/` `packages/` ディレクトリのみ）。設定簡素、型共有は手動コピー or git submodule
C) **マルチリポ**（Unit ごとに別リポジトリ）。ハッカソンには非現実的
D) Other（[Answer]: に記入）

> 💡 **指針**: ハッカソン速度なら **A 推奨**（pnpm workspace で 30 分セットアップ）。型共有（API 契約）が U1 Mobile と U2 Backend の連携で重要。

[Answer]: A

---

### Question 3 — 共有型ライブラリ

U1 Mobile と U2 Backend の間で API リクエスト/レスポンス型を共有しますか？

A) **shared パッケージで型共有**（DTO、API 契約を `packages/shared/` に集約。両側が import）
B) **OpenAPI スキーマから自動生成**（バックエンドが OpenAPI 出力 → openapi-typescript で型生成）
C) **共有なし**（手動で型を二重管理）
D) Other（[Answer]: に記入）

> 💡 **指針**: A が最速。Hono は `hc` クライアント（zod ベースの型推論）も提供するが、本格運用なら shared パッケージが安定。

[Answer]: A

---

### Question 4 — CDK スタック粒度

U4 Infrastructure の CDK をどう構成しますか？

A) **単一スタック**（すべてのリソースを 1 つの CDK Stack に。ハッカソンには十分）
B) **論理分離（複数スタック）**：例 `NetworkStack`, `DataStack`, `AppStack`, `AgentStack` で責務分離
C) **環境分離（dev/stg/prod）+ 単一スタック**：Stage を切って複数環境対応
D) Other（[Answer]: に記入）

> 💡 **指針**: 5/12 締切なら A、5/30 予選までなら C も視野。B はオーバーキル。

[Answer]: B

---

### Question 5 — Agent Runtime（U3）のリポジトリ配置

U3 は Python だが、TypeScript モノレポに同居させますか？

A) **同居（Python サブディレクトリ）**：`packages/agent/` に Python プロジェクト、Dockerfile 同梱、CDK が Docker ビルド経由でデプロイ
B) **同居だが別 Git リポでサブモジュール参照**：複雑度高、推奨せず
C) **別リポジトリ**：U3 だけ別管理、CDK が ECR / S3 経由でアーティファクトを参照
D) Other（[Answer]: に記入）

> 💡 **指針**: A 推奨。CDK の DockerImageAsset で Python プロジェクトを直接ビルドできる。

[Answer]: A

---

### Question 6 — チーム編成・並行開発

ハッカソンチームは何人体制で、どう分担しますか？（Unit 設計が分業に影響）

A) **1 人で全部**（全 Unit を一人で実装。Unit 境界は順次実装の指針として使う）
B) **2 人で分担**（例：1 人がフロント+CDK、もう 1 人がバックエンド+Agent）
C) **3 人以上で並行開発**（Unit ごとに担当者を分ける）
D) Other（[Answer]: に記入）

> 💡 **指針**: 並行開発するなら shared 型ライブラリ（Q3=A）が前提。1 人なら型の二重管理を許容しても進む。

[Answer]: C、3人で開発します

---

## 5. 回答後の流れ

1. ユーザーが Q1〜Q6 に `[Answer]:` で回答
2. AI が回答分析、曖昧さあれば clarification ファイル
3. AI が承認プロンプト提示
4. 承認 → Part 2 で 3 ドキュメント生成
5. 完了メッセージ → **5/12 書類審査の必須成果物が揃う**

---

## 6. 参照
- `aidlc-docs/inception/requirements/requirements.md`
- `aidlc-docs/inception/user-stories/stories.md`
- `aidlc-docs/inception/application-design/`（5 ドキュメント）
- `aidlc-docs/inception/plans/execution-plan.md`
- `.aidlc-rule-details/inception/units-generation.md`
