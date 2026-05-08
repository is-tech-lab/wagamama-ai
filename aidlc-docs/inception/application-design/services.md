# Services & Orchestration — わがママAI

**作成日**: 2026-05-08
**スコープ**: サービスオーケストレーション、業務ワークフロー、サービス境界の定義
**前提**: Q3=B（AgentCore 非同期 + EventBridge）, Q4=B（メトリクス更新も EventBridge 経由）

---

## 1. サービスとは（本ドキュメントの定義）

ここでの「サービス」は、**複数の Use Case / Component を協調させて 1 つの業務ワークフローを成立させる単位**。

各サービスは：
- 入口（Trigger）が明確（HTTP / EventBridge / Scheduler / Push）
- 内部で複数の Use Case を呼び出す
- 1 つの**業務目的**に対応

> Application Layer の Use Case は「1 つの目的」のロジック。Service は「複数 Use Case を呼ぶマクロ的な orchestrator」。実装的には Hono Lambda の特定ルート + 内部呼び出しの全体。

---

## 2. サービス一覧

| Service | Trigger | 主目的 | 関連 Use Cases |
|---|---|---|---|
| **ChatService** | HTTPS POST `/chat/messages` | F1 ユーザー発話 → ママ応答 | SendChatMessageUseCase + UpdateMetricsUseCase（経由 EventBridge） |
| **OnboardingService** | HTTPS POST `/onboarding` | E1-S2 初回セットアップ | OnboardUserUseCase |
| **MorningAdviceService** | EventBridge Scheduler（毎日 6:30） | F2 朝の先回りアドバイス | GenerateMorningAdviceUseCase |
| **LunchAdviceService** | EventBridge Scheduler（毎日 11:30） | F2 昼前の食事アドバイス（ペルソナ別分岐） | GenerateLunchAdviceUseCase |
| **ReservationOrchestrationService** | EventBridge Scheduler / HTTPS POST `/reservations` | F3 代理予約全体 | TriggerReservationUseCase + HandleReservationCompletedUseCase |
| **MetricsAggregationService** | EventBridge `MetricsUpdated` イベント | F5 メトリクス計算と永続化 | UpdateMetricsUseCase |
| **DashboardService** | HTTPS GET `/metrics` | F5 ダッシュボード表示 | GetMetricsDashboardUseCase |
| **DataIntegrationService** | 他サービスから内部呼出（純粋ライブラリ的） | F6 外部 API 統合 | EnrichWithExternalDataUseCase |

---

## 3. サービス別ワークフロー

### 3.1 ChatService（F1）

**Trigger**: Mobile App → API Gateway → Hono Lambda `POST /chat/messages`

```mermaid
sequenceDiagram
    participant U as User (Mobile)
    participant L as Hono Lambda
    participant CR as ChatRepository
    participant BD as Bedrock Claude
    participant EB as EventBridge

    U->>L: POST /chat/messages {text}
    L->>CR: findRecent(userId, 10)
    CR-->>L: history[]
    L->>BD: chat(systemPrompt, [history, userMsg])
    BD-->>L: mama reply text
    L->>CR: save(userMsg)
    L->>CR: save(mamaReply)
    L->>EB: publish("MetricsUpdated", {type: "chat_sent"})
    L-->>U: 200 OK {mamaReply}
```

**ポイント**:
- Q4=B により、メトリクス更新は同期処理せず EventBridge にイベント発行のみ
- メトリクス計算は MetricsAggregationService が非同期で処理

---

### 3.2 OnboardingService（E1-S2、2026-05-08 改訂：レビュー指摘 3 対応）

**Trigger**: Mobile App → `POST /onboarding`（**OAuth code は同梱しない**、Hono Lambda は秘匿情報を一切扱わない）

```mermaid
sequenceDiagram
    participant U as User (Mobile)
    participant L as Hono Lambda
    participant UR as UserProfileRepository
    participant PN as PushNotificationSender
    participant ID as AgentCore Identity
    participant G as Google OAuth

    U->>L: POST /onboarding {nickname, consents}
    L->>UR: save(profile with consents)
    L->>PN: register device for push
    L-->>U: 200 OK {profile, oauthRequired: true}

    Note over U,G: ここから OAuth フェーズ（Hono Lambda は介在しない）
    U->>ID: streaming authorization URL 要求
    ID-->>U: Google 認可 URL (PKCE code_challenge 付き)
    U->>G: 認可ページへ（expo-auth-session が PKCE 処理）
    G->>U: 認可コード返却
    U->>ID: 認可コード + code_verifier を提示
    ID->>G: トークン交換
    G-->>ID: access_token + refresh_token
    ID->>ID: Token Vault に KMS 暗号化保存
    ID-->>U: OAuth 完了通知
```

**ポイント**：
- **Hono Lambda は OAuth code / token を一切見ない** → CloudWatch Logs 漏洩経路ゼロ
- Mobile 側は `expo-auth-session`（RFC 8252 / OAuth 2.1 準拠の PKCE）を使用、`client_secret` 不要
- AgentCore Identity が refresh token を Token Vault（KMS）に保存、F3 実行時は `@requires_access_token` デコレータで自動注入
- Google OAuth は `customParameters: {access_type: "offline"}` で refresh token を取得

---

### 3.3 MorningAdviceService（F2 / E2-S1, E2-S2）

**Trigger**: EventBridge Scheduler (cron: `30 6 * * *`) → Hono Lambda `POST /internal/scheduled/morning-advice`

```mermaid
sequenceDiagram
    participant SCH as EventBridge Scheduler
    participant L as Hono Lambda
    participant DI as DataIntegrationService
    participant BC as Bedrock Claude
    participant BT as Bedrock Titan Image
    participant S3 as S3
    participant CR as ChatRepository
    participant PN as Push
    participant EB as EventBridge

    SCH->>L: morning advice trigger {userId}
    L->>DI: fetchWeather(location)
    L->>DI: getCalendarEvents(window=today)
    DI-->>L: weather + schedule
    L->>BC: chat(advice prompt) → outfit text
    BC-->>L: "今日は上着羽織りなさいよ..."
    L->>BT: generate(outfit image prompt)
    BT-->>L: imageBase64
    L->>S3: upload + presign
    S3-->>L: imageUrl
    L->>CR: save(mama advice message with image)
    L->>PN: send "ママからアドバイス届いたよ"
    L->>EB: publish("MetricsUpdated", {type: "advice_pushed"})
```

---

### 3.4 LunchAdviceService（F2 / E2-S3, E2-S4）

**Trigger**: EventBridge Scheduler (cron: `30 11 * * *`)

E2-S3 と E2-S4 の分岐：

```typescript
// 内部ロジック（Service 内）
if (profile.subPersonaFlags.cookingSkill === "low") {
  // E2-S4 サブペルソナ向け：「コンビニで〇〇のサラダ買いな」
} else {
  // E2-S3 メイン向け：「ランチはサラダ多めにしなさい」
}
```

シーケンスは MorningAdviceService とほぼ同じだが、画像生成（Titan Image）はスキップ。

---

### 3.5 ReservationOrchestrationService（F3 / E3 全体）— 核（2026-05-08 改訂：レビュー指摘 1 対応）

**Trigger 1**: EventBridge Scheduler (cron: `0 22 * * *` 等、前夜トリガー)
**Trigger 2**: Mobile App → `POST /reservations`（手動依頼）

#### Phase A：受付（最小処理、〜200ms 想定 → 29 秒タイムアウトに完全に余裕）

```mermaid
sequenceDiagram
    participant T as Trigger (Mobile or Scheduler)
    participant L as Hono Lambda<br/>(public route)
    participant RS as ReservationStateRepository
    participant EB as EventBridge

    T->>L: POST /reservations {userId, prefs}
    L->>RS: create(sessionId, status: "queued")
    L->>EB: publish("ReservationRequested", {sessionId, userId, prefs})
    L-->>T: 202 Accepted {sessionId}
    Note over L,T: Lambda は ~200ms で終了。重い処理は Phase A.5 の<br/>EventBridge 経由 worker Lambda が引き受ける。
```

**重要**：
- 公開 API ハンドラ（`POST /reservations`）は **DDB 書込 + EventBridge 発行のみ**。HotPepper API 呼出や AgentCore 起動は行わない
- **API Gateway 29 秒タイムアウトの心配は完全になくなる**（200ms で終わるため）
- 公式パターン：[Process events asynchronously with API Gateway and Lambda](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/process-events-asynchronously-with-amazon-api-gateway-and-aws-lambda.html)

#### Phase A.5：実処理（EventBridge → 同 Lambda の内部ルート、最大 15 分）

```mermaid
sequenceDiagram
    participant EB as EventBridge
    participant L as Hono Lambda<br/>(internal route)
    participant DI as ExternalDataService
    participant RS as ReservationStateRepository
    participant AI as AgentCoreRuntimeInvoker
    participant ACR as AgentCore Runtime

    EB->>L: ReservationRequested {sessionId, userId, prefs}
    Note over L: Powertools Idempotency でラップ<br/>(eventKeyJmesPath: detail.sessionId)
    L->>DI: searchRestaurants(prefs) [HotPepper, ~5s]
    DI-->>L: candidates[]
    L->>RS: updateStatus(sessionId, "in_progress")
    L->>AI: invokeAndWait({runtimeArn, payload}) [同期、最大 8 分]
    AI->>ACR: invoke_agent_runtime (streaming sync)
    ACR-->>AI: 完了 (30s〜2min)
    AI-->>L: result
    Note over L,ACR: Lambda 15 分タイムアウト ＞ F3 最大 2 分なので安全
```

**ポイント**：
- AWS SDK の `invoke_agent_runtime` は streaming sync API（fire-and-forget なし）。Lambda が同期で待つ必要があり、worker Lambda（API Gateway 経由しない）が 15 分の余裕で完遂する
- `invokeAndWait` の中身は `invoke_agent_runtime` を呼んでレスポンスストリームを最後まで読む。完了したら結果を返す

#### Phase B：AgentCore Runtime 実行（Phase A.5 の Lambda が同期で待つ間に裏で動く）

```mermaid
sequenceDiagram
    participant ACR as AgentCore Runtime (Python)
    participant ACB as AgentCore Browser
    participant NA as Nova Act
    participant ACI as AgentCore Identity
    participant GC as Google Calendar
    participant EB as EventBridge

    ACR->>ACB: browser_session() → CDP endpoint
    ACR->>NA: NovaAct(cdp_endpoint_url=ws_url, ...)
    NA->>NA: act("野菜が多いランチを 19 時に予約")
    Note over NA: 食べログ等を実操作
    NA-->>ACR: ReservationResult
    
    alt 成功
        ACR->>ACI: requires_access_token(google calendar)
        ACI-->>ACR: access token
        ACR->>GC: createEvent(reservation event)
        GC-->>ACR: eventId
        ACR->>EB: publish("ReservationCompleted", {sessionId, result, eventId})
    else 失敗（CAPTCHA / 在庫切れ等）
        ACR->>EB: publish("ReservationFailed", {sessionId, reason})
    end
```

#### Phase C：完了処理（EventBridge → Hono Lambda の内部ルート、Idempotency 保護）

```mermaid
sequenceDiagram
    participant EB as EventBridge
    participant L as Hono Lambda<br/>(Idempotency wrapped)
    participant RS as ReservationStateRepository
    participant SES as SES
    participant CR as ChatRepository
    participant PN as Push
    participant MEB as EventBridge (再発火)

    EB->>L: ReservationCompleted {sessionId, result, eventId}
    Note over L: Powertools Idempotency が eventId をキーに<br/>重複処理を IdempotencyStore で抑止
    L->>RS: markCompleted(sessionId, result, eventId)
    L->>SES: sendReservationConfirmation
    L->>CR: save(mama message: "予約しといたよ")
    L->>PN: send "ママが予約しといたよ"
    L->>MEB: publish("MetricsUpdated", {type: "zero_tap_action_completed", actionType: "reservation"})
```

**E3-S2（失敗フォールバック）**は同様の流れで `ReservationFailed` イベントを処理（Idempotency 保護込み）。

**冪等性保証の二重化（レビュー指摘 2 対応）**：
1. **Powertools Idempotency**：handler 全体を `makeIdempotent` でラップ、IdempotencyStore テーブルに結果キャッシュ
2. **ReservationStateRepository.markCompleted**：内部で DDB 条件付き書込により eventId 重複を no-op 化

```typescript
// 実装イメージ
import { makeIdempotent, IdempotencyConfig } from '@aws-lambda-powertools/idempotency';
import { DynamoDBPersistenceLayer } from '@aws-lambda-powertools/idempotency/dynamodb';

const persistenceStore = new DynamoDBPersistenceLayer({ tableName: 'IdempotencyStore' });
const config = new IdempotencyConfig({ eventKeyJmesPath: 'detail.eventId' });

export const reservationCompletedHandler = makeIdempotent(
  async (event) => { /* SES + Push + DDB 更新 */ },
  { persistenceStore, config }
);
```

---

### 3.6 MetricsAggregationService（F5 / Q4=B、2026-05-08 改訂：レビュー指摘 4 対応）

**Trigger**: EventBridge カスタムバス `wagamama-events` の `MetricsUpdated` イベント

```mermaid
sequenceDiagram
    participant EB as EventBridge
    participant L as Hono Lambda<br/>(Idempotency wrapped)
    participant MR as MetricsRepository
    participant CN as MetricsCounters<br/>+ Pure Functions

    EB->>L: MetricsUpdated {userId, event, eventId}
    Note over L: Powertools Idempotency<br/>(eventKeyJmesPath: detail.eventId)
    L->>MR: getCounters(userId)
    MR-->>L: currentCounters
    L->>CN: applyEvent(currentCounters, event)
    CN-->>L: newCounters
    L->>CN: computeMetrics(newCounters)
    CN-->>L: newMetrics (DamenessMetrics)
    L->>MR: saveCounters(userId, newCounters, eventId)
    L->>MR: saveSnapshot(userId, newMetrics, ts)
```

**ポイント**:
- メトリクス計算は **2 段の純関数**（PBT 対象）：
  - `applyEvent(counters, event) → newCounters`（単調増加性を保証）
  - `computeMetrics(counters) → DamenessMetrics`（[0, 100] 範囲、ゼロ除算耐性）
- Counters は source of truth（atomic ADD で永続化）、Snapshot は読取キャッシュ
- 冪等性は Powertools Idempotency + `saveCounters` の eventId による条件付き書込で二重保証

---

### 3.7 DashboardService（F5 / E4-S1）

**Trigger**: Mobile App → `GET /metrics?window=7d`

シンプルな読み取り：
- MetricsRepository から最新値とトレンドを取得
- `Get​MetricsDashboardUseCase` がママ口調コメントを生成（"ヤバいランク" 表示）

---

### 3.8 DataIntegrationService（F6）

他のサービスから内部関数として呼ばれる（独立した Lambda ではない、共通ライブラリ的）。

| メソッド | 対応 Story | 外部 API |
|---|---|---|
| `fetchWeather` | E5-S2 | 天気 API（無料公開） |
| `searchRestaurants` | E5-S3 | ホットペッパー API |
| `getCalendarEvents` | E5-S1 | Google Calendar API（OAuth 経由） |

---

## 4. サービス境界（責務マトリックス）

| Story | 関与するサービス |
|---|---|
| E1-S1 チャット会話 | ChatService |
| E1-S2 オンボーディング | OnboardingService |
| E1-S3 履歴永続化 | ChatService（含む） |
| E2-S1 朝コーデアドバイス | MorningAdviceService + DataIntegrationService |
| E2-S2 コーデ画像 | MorningAdviceService（内蔵） |
| E2-S3 昼食事アドバイス（メイン） | LunchAdviceService + DataIntegrationService |
| E2-S4 昼食事アドバイス（サブ） | LunchAdviceService（分岐） |
| E3-S1 予約成功 | ReservationOrchestrationService Phase A〜C |
| E3-S2 予約失敗 | ReservationOrchestrationService（失敗フロー） |
| E3-S3 Calendar 登録 | ReservationOrchestrationService Phase B |
| E3-S4 SES 通知 | ReservationOrchestrationService Phase C |
| E3-S5 Live View | AgentCore Browser 標準機能（実装不要） |
| E4-S1 ダッシュボード | DashboardService（読取） + MetricsAggregationService（書込） |
| E5-S1 OAuth 連携 | OnboardingService（初回） + AgentCore Identity（運用時） |
| E5-S2 天気取得 | DataIntegrationService（横断） |
| E5-S3 店検索 | DataIntegrationService（横断） |

---

## 5. オーケストレーション原則（2026-05-08 改訂）

1. **公開 API ハンドラは「受付係」に徹する**：HTTP 同期パスは DDB 書込 + EventBridge 発行のみ（〜200ms）。重い処理は EventBridge 経由の内部 worker Lambda へ委譲。**API Gateway 29 秒タイムアウトを越える処理は同期パスに置かない**（レビュー指摘 1 対応）
2. **イベント駆動で疎結合**：メトリクス更新・F3 完了処理は EventBridge 経由（Q4=B）。送信側も受信側も互いを直接知らない
3. **冪等性は二重防御**（レビュー指摘 2 対応）：
   - **Layer 1**：すべての非同期ハンドラを `@aws-lambda-powertools/idempotency` の `makeIdempotent` でラップ。`IdempotencyStore` DynamoDB テーブルに結果キャッシュ（PK: `id`、TTL: `expiration`）
   - **Layer 2**：Repository 層で `eventId` 引数を必須化、DDB 条件付き書込で重複を no-op 化
4. **OAuth 認証情報は Lambda で扱わない**（レビュー指摘 3 対応）：Mobile App ↔ AgentCore Identity ↔ Token Vault の直結フロー。Hono Lambda は `googleAuthCode` も `access_token` も見ない（CloudWatch Logs 漏洩経路ゼロ）
5. **メトリクス計算は 2 段純関数**（レビュー指摘 4 対応）：`applyEvent` → `computeMetrics` の合成。Counters（source of truth）と Snapshot（キャッシュ）を分離
6. **Use Case と Service の区別**（レビュー指摘 5 対応）：1 クラス = 1 execute メソッドが Use Case、複数メソッドの横断統合は `application/services/` の Service として配置
7. **責務分離**：Bedrock 呼出は Lambda、ブラウザ操作は AgentCore Runtime、SES は Lambda（IAM 権限集約）
8. **Failure handling**：EventBridge DLQ + CloudWatch アラームで不達検知（Construction Phase で詳細設計）

### 5.1 非同期ハンドラ一覧（Powertools Idempotency 必須）

| Handler | Trigger | Idempotency Key |
|---|---|---|
| `ProcessReservationRequestUseCase` | EventBridge `ReservationRequested` | `detail.sessionId` |
| `HandleReservationCompletedUseCase` | EventBridge `ReservationCompleted` | `detail.eventId` |
| `HandleReservationFailedUseCase` | EventBridge `ReservationFailed` | `detail.eventId` |
| `UpdateMetricsUseCase` | EventBridge `MetricsUpdated` | `detail.eventId` |
| `morningAdviceHandler` | EventBridge Scheduler | `<userId>#<date>` |
| `lunchAdviceHandler` | EventBridge Scheduler | `<userId>#<date>` |

---

## 6. 参照
- `aidlc-docs/inception/application-design/components.md`
- `aidlc-docs/inception/application-design/component-methods.md`
- `aidlc-docs/inception/application-design/component-dependency.md`
- `aidlc-docs/inception/user-stories/stories.md`
