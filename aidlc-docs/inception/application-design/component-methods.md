# Component Methods — わがママAI

**作成日**: 2026-05-08
**スコープ**: メソッドシグネチャ・入出力型・高レベル目的のみ。**詳細なビジネスルール・実装は Functional Design（Construction Phase per-unit）で扱う。**

> 言語規約：
> - Backend (TypeScript): Hono / Lambda / domain層は純 TypeScript
> - Agent Layer (Python): Nova Act / AgentCore SDK
> - Frontend (TypeScript / React Native)

---

## 1. Domain Layer（pure / TypeScript）

### 1.1 ChatMessage (Entity)

```typescript
class ChatMessage {
  static create(input: { userId: string; senderType: SenderType; text: string }): ChatMessage;
  // 純関数で生成。バリデーション失敗は throws DomainError。

  readonly id: string;
  readonly userId: string;
  readonly senderType: "user" | "mama";
  readonly text: string;
  readonly timestamp: number;

  isMamaMessage(): boolean;
}
```

### 1.2 UserProfile (Entity)

```typescript
class UserProfile {
  static create(input: { userId: string; nickname: string; consentScopes: ConsentScope[] }): UserProfile;

  readonly userId: string;
  readonly nickname: string;          // ママが呼びかける名前
  readonly consentScopes: ConsentScope[];  // calendar / location / health / purchase / chat
  readonly subPersonaFlags: SubPersonaFlags; // 自炊スキル等

  hasConsent(scope: ConsentScope): boolean;
  withConsent(scope: ConsentScope): UserProfile; // 不変更新
}
```

### 1.3 DamenessMetrics (Value Object)

```typescript
class DamenessMetrics {
  readonly mamaLove: number;        // 0..100, ママ溺愛度
  readonly mamaDelegation: number;  // 0..100, ママ任せ度
  readonly mamaInterference: number; // 0..100, ママ介入度
}
// 注：DamenessMetrics は computeMetrics() の出力。直接更新メソッドは持たない（純粋なスナップショット）。
// 状態の単調増加は MetricsCounters 側で表現する。
```

### 1.4 AdviceContent (Value Object)

```typescript
class AdviceContent {
  readonly category: "outfit" | "lunch";
  readonly text: string;       // ママ口調の本文
  readonly imageUrl?: string;  // 画像 URL (outfit のみ)
  readonly contextSnapshot: ContextSnapshot; // 天気・予定・履歴の参照スナップ
}
```

### 1.5 ReservationRequest (Value Object)

```typescript
class ReservationRequest {
  readonly userId: string;
  readonly category: "restaurant" | "delivery";
  readonly candidates: Candidate[]; // 上限 10 件
  readonly preference: PreferenceProfile;
  readonly timeSlot: { start: ISO8601; end: ISO8601 };
}
```

### 1.6 MetricsCounters (Value Object) — 生カウンタ

```typescript
type MetricsCounters = {
  // === 分子 ===
  selfInitiatedMessageCount: number;    // 自発的会話量
  thanksCount: number;                  // ありがとう数
  zeroTapActionCount: number;           // 無承認実行回数
  proactiveNotificationCount: number;   // 能動通知数

  // === 分母 ===
  totalUserMessageCount: number;        // ありがとう率の分母
  totalActionCount: number;             // ママ任せ度の分母
  passiveResponseCount: number;         // ママ介入度の分母

  // === 補助 ===
  sessionCount: number;                 // 利用頻度
};

function initialCounters(): MetricsCounters;
// 全フィールド 0 で初期化
```

### 1.7 MetricsEvent (Discriminated Union)

```typescript
type MetricsEvent =
  | { type: "user_message_sent"; isThanks: boolean; isInitiated: boolean }
  | { type: "session_opened" }
  | { type: "zero_tap_action_completed"; actionType: "reservation" | "advice_push" }
  | { type: "approved_action_completed"; actionType: string }
  | { type: "proactive_notification_pushed"; channel: "push" | "ses" | "chat" };
```

### 1.8 applyEvent / computeMetrics / calculateMetricsDelta (Pure Functions)

```typescript
// イベントを適用してカウンタを単調増加（純関数）
function applyEvent(c: MetricsCounters, e: MetricsEvent): MetricsCounters;

// カウンタから 3 軸スコアを計算（§2.3 計算式）
function computeMetrics(c: MetricsCounters): DamenessMetrics;
// ママ溺愛度 = normalize(sessionCount × selfInitiatedMessageCount × thanksCount/totalUserMessageCount)
// ママ任せ度 = zeroTapActionCount / totalActionCount   (0 除算は 0)
// ママ介入度 = proactiveNotificationCount / passiveResponseCount  (0 除算は 0)
// すべて [0, 100] にクランプ

// 合成関数：1 イベント受けて新カウンタとスコアを返す
function calculateMetricsDelta(
  c: MetricsCounters,
  e: MetricsEvent
): { newCounters: MetricsCounters; newMetrics: DamenessMetrics };
```

**PBT で検証する性質**：
- `applyEvent` は **単調増加**：`applyEvent(c, e).fieldX >= c.fieldX` （全フィールド）
- `computeMetrics` の出力は全フィールド **[0, 100] に収まる**
- `computeMetrics` は **NaN / Infinity を生まない**（分母 0 でも 0 を返す）
- 独立イベント間で **順序非依存**：`applyEvent(applyEvent(c, e1), e2) ≡ applyEvent(applyEvent(c, e2), e1)`

### 1.9 MamaPersonaPrompter (Pure Function)

```typescript
function buildMamaSystemPrompt(input: {
  nickname: string;
  toneLevel: ToneLevel;       // 1〜5（5/30 予選では固定 = 3）
  context: ContextSnapshot;
}): string;
```

### 1.10 AdvicePromptBuilder (Pure Function)

```typescript
function buildAdvicePrompt(input: {
  category: "outfit" | "lunch";
  weather: WeatherSnapshot;
  schedule: CalendarSnapshot;
  recentMeals?: MealHistory;
  subPersonaFlags?: SubPersonaFlags;
}): string;
// E2-S4（サブペルソナ）の分岐ロジックも含む
```

---

## 2. Application Layer（Use Cases / TypeScript）

各 Use Case は class または exported function。依存はコンストラクタ注入（DI）。

### 2.1 SendChatMessageUseCase

```typescript
class SendChatMessageUseCase {
  constructor(
    private chatRepo: ChatRepository,
    private bedrock: BedrockClaudeClient,
    private metrics: EventBridgePublisher
  ) {}

  async execute(input: { userId: string; text: string }): Promise<{ mamaReply: ChatMessage }>;
}
```

### 2.2 GetChatHistoryUseCase

```typescript
class GetChatHistoryUseCase {
  async execute(input: { userId: string; cursor?: string; limit: number }): 
    Promise<{ messages: ChatMessage[]; nextCursor?: string }>;
}
```

### 2.3 OnboardUserUseCase

```typescript
class OnboardUserUseCase {
  async execute(input: {
    userId: string;
    nickname: string;
    consentScopes: ConsentScope[];
    // googleAuthCode は受け取らない（CloudWatch Logs 漏洩経路を排除）
  }): Promise<{ profile: UserProfile; oauthRequired: boolean }>;
}
```

> **OAuth 連携の流れ（2026-05-08 改訂）**：
> 1. Mobile App は **AgentCore Identity の `streaming authorization URL`** を直接受け取って OAuth 認可フローを完了する（PKCE は Expo `expo-auth-session` が自動処理、RFC 8252 / OAuth 2.1 準拠）
> 2. AgentCore Identity が refresh token を Token Vault（KMS 暗号化）に自動保存
> 3. **Hono Lambda は OAuth code / token を一切見ない**（ログ漏洩経路ゼロ）
> 4. F3 実行時は AgentCore Runtime 内で `@requires_access_token` デコレータが Token Vault からトークンを自動注入

### 2.4 GenerateMorningAdviceUseCase

```typescript
class GenerateMorningAdviceUseCase {
  async execute(input: { userId: string; triggeredAt: ISO8601 }): 
    Promise<{ advice: AdviceContent; pushed: boolean }>;
}
// 内部処理: DataIntegration → Bedrock Claude → Bedrock Titan → S3 → 保存 → Push
```

### 2.5 GenerateLunchAdviceUseCase

```typescript
class GenerateLunchAdviceUseCase {
  async execute(input: { userId: string; triggeredAt: ISO8601 }): 
    Promise<{ advice: AdviceContent; pushed: boolean }>;
}
// E2-S4 対応: profile.subPersonaFlags.cookingSkill === "low" で別プロンプト分岐
```

### 2.6 TriggerReservationUseCase（**最小処理に簡素化**、2026-05-08 改訂）

```typescript
class TriggerReservationUseCase {
  async execute(input: { userId: string; preferences: PreferenceProfile }): 
    Promise<{ sessionId: string; status: "queued" }>;
  // 内部処理（〜200ms 想定、29 秒タイムアウトに余裕）：
  //   1. ReservationStateRepository.create({sessionId, status: "queued"})
  //   2. EventBridgePublisher.publish("ReservationRequested", {sessionId, userId, preferences})
  //   3. 即 202 Accepted を返す
  // ⚠️ HotPepper 検索や AgentCore 起動は行わない（裏で別 Lambda が EventBridge 経由で実施）
}
```

### 2.6b ProcessReservationRequestUseCase（**新規追加**、内部ハンドラ）

```typescript
class ProcessReservationRequestUseCase {
  async execute(event: ReservationRequestedEvent): Promise<void>;
  // EventBridge 経由で起動される。Lambda 15 分タイムアウトまで使える。
  // 内部処理：
  //   1. ExternalDataService.searchRestaurants() → candidates[]
  //   2. ReservationStateRepository.update(sessionId, status: "in_progress")
  //   3. AgentCoreRuntimeInvoker.invokeAndWait() → result（30秒〜2分）
  //   4. 完了したら EventBridge.publish("ReservationCompleted") or "ReservationFailed"
}
// ⚠️ Powertools Idempotency でラップ：sessionId をキーに重複処理を防止
```

### 2.7 HandleReservationCompletedUseCase

```typescript
class HandleReservationCompletedUseCase {
  async execute(event: ReservationCompletedEvent): Promise<void>;
}
// EventBridge から呼ばれる：SES 送信 + Push + チャット投稿 + メトリクス更新
// ⚠️ Powertools Idempotency でラップ：sessionId をキーに二重発火を防止
//    （EventBridge は at-least-once 配信のため必須）
```

### 2.8 GetMetricsDashboardUseCase

```typescript
class GetMetricsDashboardUseCase {
  async execute(input: { userId: string; window?: "7d" | "30d" }): 
    Promise<{ current: DamenessMetrics; trend: TrendPoint[]; comment: string }>;
}
```

### 2.9 UpdateMetricsUseCase

```typescript
class UpdateMetricsUseCase {
  async execute(event: MetricsEvent): Promise<{ updated: DamenessMetrics }>;
}
```

### 2.10 ExternalDataService（**Use Case ではなく Application Service**、2026-05-08 改訂）

```typescript
// application/services/external-data-service.ts
class ExternalDataService {
  constructor(
    private weather: WeatherAPIClient,
    private hotpepper: HotPepperAPIClient,
    private calendar: GoogleCalendarClient
  ) {}

  async fetchWeather(input: { location: GeoPoint; date: ISO8601 }): Promise<WeatherSnapshot>;
  async searchRestaurants(input: { location: GeoPoint; preferences: PreferenceProfile }): 
    Promise<Candidate[]>;
  async getCalendarEvents(input: { userId: string; window: TimeWindow }): 
    Promise<CalendarSnapshot>;
}
```

> **設計意図（2026-05-08 改訂、レビュー指摘 5 対応）**：
> 旧 `EnrichWithExternalDataUseCase` は「1 クラス = 1 Use Case = 1 execute メソッド」原則に違反していた。複数の独立した外部 API 統合は **横断的なサービス**として `application/services/` に配置し直し、他の Use Case（`GenerateMorningAdviceUseCase`、`ProcessReservationRequestUseCase` 等）が DI で受け取って利用する。

---

## 3. Infrastructure Layer

### 3.1 ChatRepository

```typescript
interface ChatRepository {
  save(message: ChatMessage): Promise<void>;
  findByUser(userId: string, opts: { cursor?: string; limit: number }): 
    Promise<{ messages: ChatMessage[]; nextCursor?: string }>;
  findRecent(userId: string, count: number): Promise<ChatMessage[]>; // コンテキスト用
}
```

### 3.2 UserProfileRepository

```typescript
interface UserProfileRepository {
  save(profile: UserProfile): Promise<void>;
  findById(userId: string): Promise<UserProfile | null>;
  updateConsent(userId: string, scope: ConsentScope, granted: boolean): Promise<void>;
}
```

### 3.3 MetricsRepository

```typescript
interface MetricsRepository {
  // === Counters（生データ、source of truth） ===
  getCounters(userId: string): Promise<MetricsCounters>;
  saveCounters(userId: string, counters: MetricsCounters, eventId: string): Promise<void>;
  // 内部で DDB の atomic ADD と eventId 冪等性チェックを使用

  // === Snapshot（DamenessMetrics、計算結果のキャッシュ） ===
  getLatestSnapshot(userId: string): Promise<DamenessMetrics>;
  saveSnapshot(userId: string, metrics: DamenessMetrics, ts: ISO8601): Promise<void>;
  getTrend(userId: string, window: "7d" | "30d"): Promise<TrendPoint[]>;
}
```

### 3.4 ReservationStateRepository

```typescript
interface ReservationStateRepository {
  create(input: { sessionId: string; userId: string; request: ReservationRequest }): Promise<void>;
  updateStatus(sessionId: string, status: "in_progress" | "completed" | "failed"): Promise<void>;
  // ⚠️ 2026-05-08 改訂：eventId 必須化（冪等性のため、レビュー指摘 2 対応）
  markCompleted(sessionId: string, result: ReservationResult, eventId: string): Promise<void>;
  markFailed(sessionId: string, reason: FailureReason, eventId: string): Promise<void>;
  findBySession(sessionId: string): Promise<ReservationState | null>;
}
// 注: markCompleted / markFailed は内部で DDB 条件付き書込（attribute_not_exists(processedEventIds[eventId])）
// により、同じ eventId での 2 回目の呼出を no-op にする。
// さらに上位 Use Case 層では Powertools Idempotency で完全多重防止。
```

### 3.5 BedrockClaudeClient

```typescript
interface BedrockClaudeClient {
  chat(input: { systemPrompt: string; messages: BedrockMessage[]; maxTokens?: number }): 
    Promise<{ text: string; usage: TokenUsage }>;
}
```

### 3.6 BedrockTitanImageClient

```typescript
interface BedrockTitanImageClient {
  generate(input: { prompt: string; size?: "512x512" | "1024x1024" }): 
    Promise<{ imageBase64: string }>;
}
```

### 3.7 S3ImageStorage

```typescript
interface S3ImageStorage {
  uploadBase64(key: string, base64: string, contentType: string): Promise<{ s3Uri: string }>;
  presignGet(s3Uri: string, ttlSeconds: number): Promise<string>;
}
```

### 3.8 AgentCoreRuntimeInvoker

```typescript
interface AgentCoreRuntimeInvoker {
  startAsync(input: { 
    runtimeArn: string; 
    payload: ReservationRequest; 
    userId: string;
  }): Promise<{ sessionId: string }>;
}
// Q3=B: 非同期 (InvokeAgentRuntime + ResponseStream を待たずに即返却するラッパー)
```

### 3.9 EventBridgePublisher

```typescript
interface EventBridgePublisher {
  publish(input: { source: string; detailType: EventType; detail: object }): Promise<void>;
  // EventType: "MetricsUpdated" | "ReservationCompleted" | "ReservationFailed" | ...
}
```

### 3.10 SESEmailSender

```typescript
interface SESEmailSender {
  sendReservationConfirmation(input: {
    to: string;
    nickname: string;
    reservation: ReservationResult;
  }): Promise<{ messageId: string }>;
}
```

### 3.11 PushNotificationSender

```typescript
interface PushNotificationSender {
  sendToUser(input: { 
    userId: string; 
    title: string; 
    body: string; 
    data?: Record<string, string>;
  }): Promise<void>;
}
```

### 3.12 WeatherAPIClient

```typescript
interface WeatherAPIClient {
  getDailyForecast(location: GeoPoint, date: ISO8601): Promise<WeatherSnapshot>;
}
```

### 3.13 HotPepperAPIClient

```typescript
interface HotPepperAPIClient {
  search(input: { location: GeoPoint; genre?: string; budget?: BudgetRange; limit: number }): 
    Promise<Candidate[]>;
}
```

### 3.14 GoogleCalendarClient

```typescript
interface GoogleCalendarClient {
  listEvents(input: { userId: string; window: TimeWindow }): Promise<CalendarSnapshot>;
  createEvent(input: { userId: string; event: CalendarEvent }): Promise<{ eventId: string }>;
}
// 実体: AgentCore Identity 経由の OAuth トークンを使用（コードに埋め込まない）
```

### 3.15 REST Handlers（Hono）

```typescript
// 公開 API (API Gateway 経由)
const app = new Hono();
app.post("/chat/messages", sendMessageHandler);
app.get("/chat/messages", getChatHistoryHandler);
app.post("/onboarding", onboardHandler);
app.post("/reservations", triggerReservationHandler);
app.get("/metrics", getMetricsHandler);

// 内部ルート (EventBridge / Scheduler 経由のみ、API Gateway 非公開)
app.post("/internal/scheduled/morning-advice", morningAdviceHandler);
app.post("/internal/scheduled/lunch-advice", lunchAdviceHandler);
app.post("/internal/events/reservation-completed", reservationCompletedHandler);
app.post("/internal/events/metrics-update", metricsUpdateHandler);

// EventBridge → Lambda 起動時のペイロードを内部 path にルーティング
function eventBridgeAdapter(event: EventBridgeEvent): Request;
```

---

## 4. Frontend Components（TypeScript / React Native）

### 4.1 ChatUI Component

```typescript
function ChatScreen(props: { userId: string }): JSX.Element;

// 内部 hooks
const { messages, send, loadMore } = useChat(userId);
```

### 4.2 DashboardUI Component

```typescript
function DashboardScreen(props: { userId: string }): JSX.Element;

const { metrics, trend, comment } = useMetrics(userId);
```

### 4.3 OnboardingUI Component

```typescript
function OnboardingFlow(props: { onComplete: () => void }): JSX.Element;

// ステップ: Welcome → Nickname → ConsentScopes → GoogleOAuth → Done
```

### 4.4 AuthModule

```typescript
interface AuthModule {
  loginWithGoogle(): Promise<{ token: string; profile: GoogleProfile }>;
  logout(): Promise<void>;
  getCurrentToken(): string | null;
  refreshIfNeeded(): Promise<void>;
}
```

### 4.5 APIClient

```typescript
class APIClient {
  constructor(private auth: AuthModule, private baseURL: string) {}

  async get<T>(path: string): Promise<T>;
  async post<T>(path: string, body: object): Promise<T>;
  // auth.getCurrentToken() を Authorization header に自動付与
}
```

### 4.6 NotificationService

```typescript
interface NotificationService {
  registerDevice(): Promise<{ pushToken: string }>;
  onReceive(handler: (notification: PushPayload) => void): () => void;
  onTap(handler: (notification: PushPayload) => void): () => void;
}
```

### 4.7 Zustand Stores

```typescript
// stores/chat-store.ts
interface ChatStore {
  messages: ChatMessage[];
  appendMessage(msg: ChatMessage): void;
  prependHistory(msgs: ChatMessage[]): void;
  reset(): void;
}
const useChatStore = create<ChatStore>(...);

// stores/metrics-store.ts
interface MetricsStore {
  current: DamenessMetrics | null;
  trend: TrendPoint[];
  set(metrics: DamenessMetrics, trend: TrendPoint[]): void;
}
const useMetricsStore = create<MetricsStore>(...);
```

---

## 5. Agent Layer（Python / Nova Act + AgentCore）

### 5.1 ReservationWorkflow

```python
@app.entrypoint  # AgentCore Runtime decorator
def reservation_workflow(payload: dict) -> dict:
    """
    payload: { userId, candidates, preference, timeSlot }
    return: { sessionId, status }  # 完了は EventBridge で通知
    """
    ...
```

### 5.2 Nova Act 操作（Workflow 内部）

```python
from bedrock_agentcore.tools.browser_client import browser_session
from nova_act import NovaAct

def execute_reservation(candidates, preference, time_slot) -> ReservationResult:
    with browser_session(region="ap-northeast-1") as client:
        ws_url, headers = client.generate_ws_headers()
        with NovaAct(
            cdp_endpoint_url=ws_url,
            cdp_headers=headers,
            nova_act_api_key=...,
            starting_page="https://tabelog.com/",
        ) as nova_act:
            result = nova_act.act(f"{preference} を {time_slot} に予約して")
            return ReservationResult(...)
```

### 5.3 CalendarRegistrationStep

```python
@requires_access_token(provider="google", scopes=["calendar.events"])
def register_to_calendar(reservation_result: ReservationResult, *, access_token: str) -> str:
    """Google Calendar API でイベント追加。AgentCore Identity が token 注入。"""
    ...
    return event_id
```

### 5.4 EventBridge Notifier

```python
def publish_completion(session_id: str, result: ReservationResult) -> None:
    """EventBridge へ ReservationCompleted イベントを発行。"""
    ...

def publish_failure(session_id: str, reason: str) -> None:
    """EventBridge へ ReservationFailed イベントを発行。"""
    ...
```

---

## 6. メソッドカウントサマリー

| 層 | メソッド／関数数（概算） |
|---|---|
| Domain | 8 純関数 + 各 Entity/VO のファクトリ・更新メソッド |
| Application（Use Cases）| 10 Use Case の `execute()` |
| Infrastructure（Repos）| 4 リポジトリ × 平均 4 メソッド = 16 |
| Infrastructure（Clients）| 10 クライアント × 平均 2 メソッド = 20 |
| REST Handlers | 9 ハンドラ |
| Frontend | 7 コンポーネント / モジュール × 平均 3 メソッド ≈ 21 |
| Agent Layer | 4 関数 |
| **合計** | **約 90 メソッド／関数** |

---

## 7. 詳細設計の委譲先

以下は **Construction Phase の Functional Design（Unit ごと）** で扱う：
- 各純関数の具体的な計算式・ガード節（例: `calculateMetricsDelta` の重み付け）
- ママ口調のシステムプロンプト本文（実テンプレート）
- 各 Repository の DynamoDB スキーマ詳細
- エラーハンドリング・リトライ戦略
- テストデータ・PBT のプロパティ定義

---

## 8. 参照
- `aidlc-docs/inception/application-design/components.md`
- `aidlc-docs/inception/application-design/services.md`
- `aidlc-docs/inception/application-design/component-dependency.md`
