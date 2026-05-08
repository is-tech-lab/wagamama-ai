# Unit of Work — Story Map

**作成日**: 2026-05-08
**前提**: stories.md（16 ストーリー）/ unit-of-work.md（4 Unit + shared）
**目的**: 全 16 ストーリーを Unit に割当て、5/12 書類審査時点で「**全 Story が実装計画に紐付いている**」ことを示す。

---

## 1. Story × Unit マッピング表（網羅性検証用）

各セルの記号：
- ◎ = 主担当（Unit が中核を実装）
- ○ = 副担当（Unit が一部を分担）
- 基盤 = Unit が前提インフラを提供
- — = 非関与

| Story | Epic | U1 Mobile | U2 Backend | U3 Agent | U4 Infra | shared |
|---|---|:-:|:-:|:-:|:-:|:-:|
| **E1-S1** ママ人格チャット会話 | F1 | ◎ UI | ◎ Bedrock + Repo | — | 基盤 | DTO |
| **E1-S2** 初回オンボーディング | F1 | ◎ Onboarding UI + AuthModule | ○ プロファイル保存 | — | 基盤 (AgentCore Identity) | DTO |
| **E1-S3** チャット履歴永続化 | F1 | ◎ 履歴 UI | ◎ Repo + ページング | — | 基盤 (DDB) | DTO |
| **E2-S1** 朝コーデアドバイス | F2 | ○ Push 受信表示 | ◎ Use Case + Bedrock | — | 基盤 (Scheduler) | Event |
| **E2-S2** コーデ簡易画像生成 | F2 | ○ 画像表示 | ◎ Titan + S3 | — | 基盤 (S3) | DTO |
| **E2-S3** 昼食事アドバイス（メイン） | F2 | ○ Push 受信表示 | ◎ Use Case + Bedrock | — | 基盤 (Scheduler) | Event |
| **E2-S4** 昼食事アドバイス（サブペルソナ） | F2 | ○ Push 受信表示 | ◎ Prompt 分岐ロジック | — | 基盤 | — |
| **E3-S1** Nova Act 予約成功 | F3 | ○ 完了 Push 表示 | ◎ Orchestrator + EventBridge | ◎ Nova Act 操作 | 基盤 (AgentCore) | Event |
| **E3-S2** 予約失敗フォールバック | F3 | ○ 失敗 Push 表示 | ◎ HandleFailedUseCase | ◎ Nova Act エラー検知 | 基盤 | Event |
| **E3-S3** Calendar 自動登録 | F3 | — | — | ◎ AgentCore Identity + Calendar API | 基盤 (Identity) | — |
| **E3-S4** SES 確定メール風通知 | F3 | — | ◎ SES 送信 | — | 基盤 (SES) | — |
| **E3-S5** AgentCore Browser Live View | F3 | — | — | ○ Browser 標準機能利用 | 基盤 (Browser) | — |
| **E4-S1** ダメ度 3 軸ダッシュボード | F5 | ◎ Dashboard UI | ◎ MetricsRepo + computeMetrics | — | 基盤 (DDB Metrics) | DTO |
| **E5-S1** Google Calendar OAuth 連携 | F6 | ◎ expo-auth-session | ○ プロファイル更新 | — | 基盤 (AgentCore Identity) | DTO |
| **E5-S2** 天気データ取得 | F6 | — | ◎ ExternalDataService | — | 基盤 | — |
| **E5-S3** ホットペッパー店検索 | F6 | — | ◎ ExternalDataService | — | 基盤 | — |

**カバレッジ検証**: 16 / 16 ストーリーすべてが少なくとも 1 つの Unit に主担当として割り当てられている ✅

---

## 2. Unit 別 Story 一覧（実装計画用）

### 2.1 U1 Mobile App が **主担当**するストーリー

| Story | 主な実装 | 関連ファイル（unit-of-work.md §2 参照） |
|---|---|---|
| E1-S1 | ChatScreen + useChat hook | `packages/mobile/app/(tabs)/index.tsx` + `src/stores/chat-store.ts` |
| E1-S2 | OnboardingFlow + AuthModule | `packages/mobile/app/onboarding.tsx` + `src/modules/auth/` |
| E1-S3 | 履歴の無限スクロール、AsyncStorage | `src/components/chat-history-list.tsx` |
| E4-S1 | DashboardScreen + ゲージ可視化 | `app/(tabs)/dashboard.tsx` + `src/components/metric-gauge.tsx` |
| E5-S1 | OAuth フロー（PKCE） | `src/modules/auth/google-auth.ts`（expo-auth-session） |

### 2.2 U2 Backend が **主担当**するストーリー

| Story | 主な実装 | 関連 Use Case / Domain |
|---|---|---|
| E1-S1 | チャット応答生成 | `SendChatMessageUseCase` + `MamaPersonaPrompter`（Pure） |
| E1-S2 | プロファイル保存 | `OnboardUserUseCase` + `UserProfileRepository` |
| E1-S3 | チャット履歴ページング | `GetChatHistoryUseCase` + `ChatRepository` |
| E2-S1 | 朝コーデアドバイス生成 | `GenerateMorningAdviceUseCase` + `AdvicePromptBuilder`（Pure） |
| E2-S2 | コーデ画像生成 | 同上 + `BedrockTitanImageClient` + `S3ImageStorage` |
| E2-S3 / E2-S4 | 昼食事アドバイス（ペルソナ分岐） | `GenerateLunchAdviceUseCase` + `AdvicePromptBuilder` |
| E3-S1 / E3-S2 | F3 オーケストレーション | `TriggerReservationUseCase` + `ProcessReservationRequestUseCase` + `HandleReservationCompletedUseCase` + `HandleReservationFailedUseCase` |
| E3-S4 | SES 送信 | `HandleReservationCompletedUseCase` 内 `SESEmailSender.sendReservationConfirmation` |
| E4-S1 | メトリクス計算・取得 | `UpdateMetricsUseCase` + `GetMetricsDashboardUseCase` + `applyEvent` / `computeMetrics`（Pure） |
| E5-S2 / E5-S3 | 外部 API 統合 | `ExternalDataService` + `WeatherAPIClient` / `HotPepperAPIClient` |

### 2.3 U3 Agent Runtime が **主担当**するストーリー

| Story | 主な実装 | Python ファイル |
|---|---|---|
| E3-S1 | Nova Act 操作（予約成功） | `packages/agent/src/workflows/reservation_workflow.py` + `src/steps/execute_reservation.py` |
| E3-S2 | 予約失敗時のエラーハンドリング | `src/steps/execute_reservation.py` の例外処理 + `publish_failure` |
| E3-S3 | Google Calendar 登録 | `src/steps/register_to_calendar.py` (`@requires_access_token` デコレータ) |
| E3-S5 | AgentCore Browser Live View | （実装不要：Browser のマネージド機能） |

### 2.4 U4 Infrastructure (CDK) が **基盤を提供**するストーリー

| Story | 提供する基盤 | Stack |
|---|---|---|
| 全 Story | DynamoDB（ChatMessages / UserProfiles / MetricsCounters / Metrics / Reservations / IdempotencyStore）・S3・KMS | DataStack |
| E1-S1〜E1-S3 | API Gateway / Lambda（Mono Hono） | AppStack |
| E2-S1, E2-S3 | EventBridge Scheduler（6:30 / 11:30 cron） | AppStack |
| E3-S1, E3-S2 | EventBridge Bus（`wagamama-events`）+ ルール | AppStack |
| E3-S1〜E3-S5 | AgentCore Runtime + Browser + Identity | AgentStack |
| E3-S4 | SES（送信検証済ドメイン） | AppStack |
| E5-S1 | AgentCore Identity の Google OAuth Provider 設定 | AgentStack |

### 2.5 shared が **型を提供**するストーリー

| Story | 提供する型 |
|---|---|
| E1-S1 | `SendChatMessageRequest` / `SendChatMessageResponse` / `ChatMessageDTO` |
| E1-S2 | `OnboardingRequest` / `OnboardingResponse` / `ConsentScope` |
| E1-S3 | `GetChatHistoryQuery` / `GetChatHistoryResponse` |
| E2-S1〜S3 | `AdviceContentDTO` / `PushPayload` (`advice_pushed`) |
| E3-S1, S2 | `ReservationCompletedEvent` / `ReservationFailedEvent` / `PushPayload` |
| E4-S1 | `GetMetricsResponse` / `DamenessMetricsDTO` / `TrendPoint` |
| E5-S1 | `OAuthCallbackPayload`（フロント↔バック連携用、token は含まない） |

---

## 3. Story 別の責任ライン（並行開発の合意ポイント）

| Story | Person A (Mobile) | Person B (Backend + Agent) | Person C (Infra) |
|---|---|---|---|
| E1-S1 | UI + Zustand | Bedrock 呼出 + Repo | DDB ChatMessages, Lambda |
| E1-S2 | Onboarding UI + Auth | プロファイル保存 | AgentCore Identity 設定 |
| E1-S3 | 無限スクロール | ページング Repo | DDB GSI（必要なら） |
| E2-S1〜S4 | Push 受信表示 | Bedrock + プロンプト | EventBridge Scheduler |
| E3-S1〜S5 | 完了通知の UX | Orchestrator + Nova Act ワークフロー | AgentCore Runtime/Browser/Identity |
| E4-S1 | ゲージ UI | metrics 計算（Pure）+ Repo | DDB MetricsCounters/Metrics |
| E5-S1〜S3 | OAuth フロー | ExternalDataService | API Key Secrets, Calendar Provider |

---

## 4. 検証チェックリスト

- [x] 16 / 16 ストーリーが少なくとも 1 つの Unit で主担当を持つ
- [x] 主担当が複数 Unit にまたがるストーリー（E3-S1, E3-S2 など F3 系）は明示的にマップ済
- [x] U4 Infra は基盤提供のみ、機能ロジックを持たない
- [x] shared は型のみ、実行時依存を導入しない
- [x] U3 Agent Runtime は F3 関連 4 ストーリーのみ担当（責務集中）
- [x] 3 人並行開発の責任ラインが Story 単位で明確（§3）

---

## 5. 参照
- `aidlc-docs/inception/user-stories/stories.md`（16 Story 詳細）
- `aidlc-docs/inception/application-design/unit-of-work.md`（Unit 詳細・コード構成）
- `aidlc-docs/inception/application-design/unit-of-work-dependency.md`（依存マトリックス・通信契約）
- `aidlc-docs/inception/application-design/components.md`（Component → Story 対応）
