# Components — わがママAI

**作成日**: 2026-05-08
**前提**: requirements.md / stories.md / application-design-plan.md（承認済 Q1〜Q8）

---

## 1. システム全体のコンポーネント分布

```
┌────────────────────────────────────────────────────────────────┐
│                     Mobile App (RN + Expo)                     │
│  [ChatUI] [DashboardUI] [OnboardingUI] [AuthModule] [APIClient]│
└────────────────────────────────────────────────────────────────┘
                              │ HTTPS (REST + APNs/FCM)
                              ▼
┌────────────────────────────────────────────────────────────────┐
│        Backend (Mono Hono Lambda, ap-northeast-1)              │
│                                                                │
│  Clean Architecture：                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ infrastructure: REST handlers, Repos, AWS clients       │   │
│  │ application: Use Cases (orchestration)                  │   │
│  │ domain: Entities, Value Objects, Pure Functions         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
        │                                     ▲
        │ InvokeAgentRuntime (async)           │ EventBridge
        ▼                                     │ (events back)
┌────────────────────────────────────────────────────────────────┐
│   Agent Layer (AgentCore Runtime, Python, ap-northeast-1)      │
│  [ReservationWorkflow (Nova Act)] [CalendarRegistration]       │
│                                                                │
│  ・AgentCore Browser (CDP endpoint 払い出し)                   │
│  ・AgentCore Identity (Google OAuth トークン注入)              │
└────────────────────────────────────────────────────────────────┘
                              │ CDP via AWS backbone
                              ▼ (Nova Act SDK)
                      Nova Act 推論 (us-east-1)
```

---

## 2. Frontend Components（Mobile App）

### 2.1 ChatUI Component
- **責務**: F1 のチャット画面（LINE 風 UI）。メッセージ送受信、履歴スクロール、ママ口調表示
- **インタフェース**: `props { userId: string }` / `events: onMessageSent, onScrollToHistory`
- **依存**: APIClient, NotificationService, ChatStore（Zustand）
- **対応 Story**: E1-S1, E1-S3

### 2.2 DashboardUI Component
- **責務**: F5 ダメ度メトリクス 3 軸ダッシュボード。円形ゲージ、トレンドグラフ、ママ口調コメント
- **インタフェース**: `props { userId: string }` / `dataSource: useMetricsQuery()`
- **依存**: APIClient, MetricsStore（Zustand）
- **対応 Story**: E4-S1

### 2.3 OnboardingUI Component
- **責務**: 初回起動時のセットアップフロー（名前・呼び方・PII 同意・Google OAuth）
- **インタフェース**: `props { onComplete: () => void }`
- **依存**: AuthModule, APIClient
- **対応 Story**: E1-S2, E5-S1

### 2.4 AuthModule
- **責務**: Google OAuth 2.0 ログイン処理（Q5=D）。Calendar OAuth と同一クライアントを使用
- **インタフェース**: `login(): Promise<AuthToken>` / `logout()` / `getToken(): AuthToken | null`
- **依存**: Google OAuth SDK, AsyncStorage（トークン保存）
- **対応 Story**: E1-S2, E5-S1

### 2.5 APIClient
- **責務**: バックエンド REST API 呼び出しのラッパー（fetch + auth header 自動付与）
- **インタフェース**: `get(path)`, `post(path, body)`, etc.
- **依存**: AuthModule（トークン取得）
- **対応 Story**: 全 Story（横断）

### 2.6 NotificationService
- **責務**: APNs / FCM プッシュ通知の受信・表示・タップハンドリング（Q2=D）
- **インタフェース**: `register()`, `onReceive(handler)`, `onTap(handler)`
- **依存**: Expo Notifications API
- **対応 Story**: E2-S1, E2-S3, E3-S1, E3-S2

### 2.7 ChatStore / MetricsStore（Zustand）
- **責務**: フロント側のクライアントステート保管（Q7=A）
- **インタフェース**: Zustand 標準（`get`, `set`, hooks）
- **対応 Story**: E1-S1, E1-S3, E4-S1

---

## 3. Backend Components（Mono Hono Lambda / Clean Architecture）

> Q1=A, Q8=B を反映。`src/domain/`, `src/application/`, `src/infrastructure/` の 3 層に分離。

### 3.1 Domain Layer（純ロジック・PBT 対象）

| Component | 責務 | 種別 |
|---|---|---|
| **ChatMessage** | チャットメッセージ Entity（不変） | Entity |
| **UserProfile** | ユーザープロファイル Entity | Entity |
| **DamenessMetrics** | ダメ度メトリクス VO（3 軸スコア、`computeMetrics` の出力） | Value Object |
| **MetricsCounters** | カウンタ VO（分子／分母を保持する生データ） | Value Object |
| **MetricsEvent** | メトリクス更新イベント（Discriminated Union） | Sum Type |
| **AdviceContent** | アドバイス VO（コーディネート/食事の中間表現） | Value Object |
| **ReservationRequest** | 予約リクエスト VO（店候補・時間・人数等） | Value Object |
| **applyEvent** | `MetricsCounters × MetricsEvent → MetricsCounters` 純関数（単調増加） | Pure Function |
| **computeMetrics** | `MetricsCounters → DamenessMetrics` 純関数（§2.3 計算式、[0..100] クランプ） | Pure Function |
| **calculateMetricsDelta** | `applyEvent` + `computeMetrics` を合成した便利関数 | Pure Function |
| **MamaPersonaPrompter** | Bedrock システムプロンプト生成（ママ口調・お節介度） | Pure Function |
| **AdvicePromptBuilder** | F2 アドバイスのプロンプト構築（天気・予定・履歴を結合） | Pure Function |

**対応 Story**: 全 Story（中核ロジック）。**PBT Extension の主対象**。

> **Note (2026-05-08 改訂)**: メトリクス計算は **2 段階**に分離（レビュー指摘 4 対応）。
> 1. `applyEvent`：イベントから生カウンタを単調増加（PBT: 単調性）
> 2. `computeMetrics`：カウンタから 3 軸スコアを計算（PBT: [0..100] 範囲、ゼロ除算耐性）

### 3.2 Application Layer（Use Case Orchestration）

| Component | 責務 | 対応 Story |
|---|---|---|
| **SendChatMessageUseCase** | F1: メッセージ受信 → Bedrock 呼出 → 保存 → 応答 | E1-S1 |
| **GetChatHistoryUseCase** | F1: チャット履歴取得（ページング） | E1-S3 |
| **OnboardUserUseCase** | E1-S2: ユーザー作成 + 同意保存 + OAuth 連携導線 | E1-S2, E5-S1 |
| **GenerateMorningAdviceUseCase** | F2: コーデアドバイス生成 + 画像生成 + Push | E2-S1, E2-S2 |
| **GenerateLunchAdviceUseCase** | F2: 食事アドバイス生成 + Push（ペルソナ別バリエーション） | E2-S3, E2-S4 |
| **TriggerReservationUseCase** | F3: 候補抽出 → AgentCore Runtime 非同期キック → 202 Accepted | E3-S1 |
| **HandleReservationCompletedUseCase** | F3: EventBridge 経由 AgentCore 完了イベント受信 → SES 送信 → Push → メトリクス更新 | E3-S1, E3-S2, E3-S4 |
| **GetMetricsDashboardUseCase** | F5: 3 軸メトリクス取得・トレンド整形 | E4-S1 |
| **UpdateMetricsUseCase** | F5: EventBridge 経由メトリクスイベント受信 → 計算 → 永続化 | F5（横断、Q4=B） |
| ~~EnrichWithExternalDataUseCase~~ | **削除**：単一責任原則違反（複数メソッドを持つ Use Case） → `ExternalDataService` に降格（下記 Application Services 参照） | — |

#### 3.2.1 Application Services（Use Case ではないが Application 層に配置する横断サービス）

| Component | 責務 | 対応 Story |
|---|---|---|
| **ExternalDataService** | F6: 天気・ホットペッパー店検索・Calendar 取得を統合した横断サービス。Use Case ではなく、他 Use Case が DI で受け取って利用する | E5-S2, E5-S3（横断） |

### 3.3 Infrastructure Layer（外部 I/O）

#### Repositories（Q6=B）

| Component | 責務 | DynamoDB Table |
|---|---|---|
| **ChatRepository** | チャット履歴 CRUD | `ChatMessages` |
| **UserProfileRepository** | プロファイル CRUD | `UserProfiles` |
| **MetricsRepository** | ダメ度メトリクス CRUD（**スナップショット = `DamenessMetrics`** と **生カウンタ = `MetricsCounters`** を分けて管理） | `Metrics`, `MetricsCounters` |
| **ReservationStateRepository** | 予約状態（queued/in_progress/done/failed）。`markCompleted` / `markFailed` に **eventId 引数追加**（冪等性確保） | `Reservations` |
| **IdempotencyStoreRepository** | AWS Lambda Powertools Idempotency 用ストア（直接コードでは触らないが CDK で作成） | `IdempotencyStore` |

#### AWS / 外部サービスクライアント

| Component | 責務 |
|---|---|
| **BedrockClaudeClient** | Claude モデル呼出（テキスト応答・アドバイス生成） |
| **BedrockTitanImageClient** | Titan Image / SDXL 呼出（コーデ画像） |
| **S3ImageStorage** | 画像アップロード・プリサインド URL 発行 |
| **AgentCoreRuntimeInvoker** | `InvokeAgentRuntimeCommand` 非同期キック（Q3=B） |
| **EventBridgePublisher** | `MetricsUpdated`, `ReservationCompleted` 等のイベント発行 |
| **SESEmailSender** | F3 確定メール風通知 |
| **PushNotificationSender** | APNs / FCM プッシュ送信 |
| **WeatherAPIClient** | 天気 API（無料）呼出 |
| **HotPepperAPIClient** | ホットペッパー店検索 API |
| **GoogleCalendarClient** | Google Calendar API（OAuth トークンは AgentCore Identity 経由） |

#### REST Handlers（Hono Router）

| Route | Handler | UseCase |
|---|---|---|
| `POST /chat/messages` | sendMessageHandler | SendChatMessageUseCase |
| `GET /chat/messages` | getChatHistoryHandler | GetChatHistoryUseCase |
| `POST /onboarding` | onboardHandler | OnboardUserUseCase |
| `POST /reservations` | triggerReservationHandler | TriggerReservationUseCase |
| `GET /metrics` | getMetricsHandler | GetMetricsDashboardUseCase |
| **Internal Routes**（API Gateway 公開せず、EventBridge / Scheduler から） | | |
| `POST /internal/scheduled/morning-advice` | morningAdviceHandler | GenerateMorningAdviceUseCase |
| `POST /internal/scheduled/lunch-advice` | lunchAdviceHandler | GenerateLunchAdviceUseCase |
| `POST /internal/events/reservation-completed` | reservationCompletedHandler | HandleReservationCompletedUseCase |
| `POST /internal/events/metrics-update` | metricsUpdateHandler | UpdateMetricsUseCase |

> EventBridge ターゲットは Lambda Function なので、ペイロードを内部ルートにルーティングする `eventBridgeAdapter` で内部 path を解決。

---

## 4. Agent Layer Components（AgentCore Runtime / Python）

### 4.1 ReservationWorkflow（Nova Act）
- **責務**: F3 の核。Hono Lambda から `InvokeAgentRuntimeCommand` で起動され、Nova Act で食べログ等を操作
- **入力**: `{ userId, candidates, preference, timeSlot }`
- **出力**: EventBridge イベント `reservation_completed`（成功時）or `reservation_failed`（失敗時）
- **依存**: AgentCore Browser（CDP endpoint）, Nova Act SDK, AgentCore Identity（Google OAuth）, EventBridge
- **対応 Story**: E3-S1, E3-S2, E3-S5

### 4.2 CalendarRegistrationStep
- **責務**: F3 完了後の Google Calendar イベント追加
- **入力**: 予約結果（店名・時刻）
- **依存**: Google Calendar API（AgentCore Identity がトークン注入）
- **対応 Story**: E3-S3

### 4.3 EventBridge Notifier（Workflow 内部関数）
- **責務**: Workflow の状態を EventBridge に発行（`reservation_completed`, `reservation_failed`）
- **対応 Story**: E3-S1, E3-S2

> **Note**: SES 送信は Hono Lambda 側で行う（責務集約・SES IAM 権限を Hono 側に集約）。AgentCore Runtime → EventBridge → Hono Lambda → SES の流れ。

---

## 5. AWS インフラ Components（CDK 管理対象）

| Component | 役割 |
|---|---|
| **API Gateway (REST)** | Mobile App ↔ Hono Lambda |
| **Hono Lambda** | バックエンド API + 内部イベント処理 |
| **AgentCore Runtime** | F3 ワークフロー実行環境（Python・最大 8 時間） |
| **AgentCore Browser** | フルマネージドブラウザ + Live View（デモ用） |
| **AgentCore Identity** | Google OAuth トークン管理 |
| **DynamoDB**（6 テーブル） | チャット・プロファイル・メトリクス（snapshot）・メトリクスカウンタ（counters）・予約状態・**IdempotencyStore**（Powertools Idempotency 永続層、PK: `id`, TTL: `expiration`） |
| **S3** | 生成画像保存 |
| **EventBridge Scheduler** | F2 朝/昼の能動通知トリガー |
| **EventBridge Bus** | 非同期イベント（メトリクス更新・予約完了） |
| **SES** | F3 確定メール風通知 |
| **KMS** | DynamoDB / S3 の保存時暗号化 |
| **Cognito Identity Pool**（オプション）| Google OAuth トークン → AWS 一時認証情報変換（必要な場合） |

> **Note**: 詳細な IaC 設計は Construction Phase の Infrastructure Design で扱う。本ドキュメントはコンポーネント識別のみ。

---

## 6. コンポーネント数サマリー

| 層 | コンポーネント数 |
|---|---|
| Frontend | 7 |
| Backend - Domain | 8 |
| Backend - Application（Use Cases） | 10 |
| Backend - Infrastructure | 4 (Repos) + 10 (Clients) + 9 (Handlers) = 23 |
| Agent Layer | 3 |
| AWS Infrastructure | 11 |
| **合計** | **62 コンポーネント** |

---

## 7. 参照
- `aidlc-docs/inception/application-design/component-methods.md`（メソッドシグネチャ）
- `aidlc-docs/inception/application-design/services.md`（オーケストレーション）
- `aidlc-docs/inception/application-design/component-dependency.md`（依存関係）
- `aidlc-docs/inception/requirements/requirements.md`
- `aidlc-docs/inception/user-stories/stories.md`
