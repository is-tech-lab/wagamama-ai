# Application Design Plan — わがママAI

**作成日**: 2026-05-08
**前提**: requirements.md / stories.md / personas.md / execution-plan.md（全て承認済）
**Stage**: INCEPTION - Application Design

---

## 0. 本プランの読み方

Application Design の **設計計画**。Q1〜Q8 に `[Answer]:` タグで回答してください。回答完了後 `done` または `回答完了` をお知らせいただくと、`components.md` / `component-methods.md` / `services.md` / `component-dependency.md` / `application-design.md`（統合版）を生成します。

> **Note**: 詳細なビジネスロジック（純粋関数の処理、メトリクス計算式の実装等）は **Construction Phase の Functional Design**（Unit ごと）で扱います。本ステージは **コンポーネント境界・責務・インタフェース・依存関係**に絞ります。

---

## 1. Plan 実行チェックリスト（Part 2 実行用）

- [x] **STEP 1**: requirements.md / stories.md / personas.md / 本プラン回答済みをロード
- [x] **STEP 2**: コンポーネント境界を確定（Q1=A モノ Lambda, Q5=D Google OAuth 反映）
- [x] **STEP 3**: `components.md` 生成（62 コンポーネント識別）
- [x] **STEP 4**: `component-methods.md` 生成（約 90 メソッド／関数）
- [x] **STEP 5**: `services.md` 生成（8 サービス、Q3=B / Q4=B 反映）
- [x] **STEP 6**: `component-dependency.md` 生成（依存マトリックス・データフロー 4 シナリオ・Q2=D / Q6=B 反映）
- [x] **STEP 7**: `application-design.md`（統合版）生成
- [x] **STEP 8**: Mermaid 図を Validate（記法チェック完了）
- [x] **STEP 9**: `aidlc-state.md` 更新は次セクションで実施

---

## 2. Mandatory Artifacts

- [ ] `aidlc-docs/inception/application-design/components.md`
- [ ] `aidlc-docs/inception/application-design/component-methods.md`
- [ ] `aidlc-docs/inception/application-design/services.md`
- [ ] `aidlc-docs/inception/application-design/component-dependency.md`
- [ ] `aidlc-docs/inception/application-design/application-design.md`（統合版）

---

## 3. Questions（ユーザー回答必須）

### Question 1 — バックエンド Lambda の粒度

Hono / TypeScript で書く API レイヤを **どの粒度で Lambda 化**しますか？

A) **モノ Lambda**（単一の Hono アプリ = 単一 Lambda、ルーティングは Hono Router で全機能を捌く）
B) **機能別 Lambda**（F1 chat, F2 advice, F3 reservation orchestrator, F5 metrics, F6 data integration 等で分割）
C) **境界別 Lambda**（同期 API 用 Lambda + EventBridge 駆動の非同期 Lambda の 2 種類に分割）
D) Other（[Answer]: に記入）

> 💡 **指針**: ハッカソン速度重視なら A、観測性・コールドスタート最小化なら B/C。MVP では A 推奨。

[Answer]: A

---

### Question 2 — フロントエンド ↔ バックエンドの通信パターン（チャット）

F1（ママ人格チャット UI）でリアルタイム性をどう実現しますか？

A) **REST のみ**（送信は POST、応答は同期で返す。新規ママ通知は Push 通知で取得後 GET）
B) **REST + WebSocket**（送信は POST、ママ応答ストリーミング・能動通知は WebSocket）
C) **REST + Server-Sent Events (SSE)**（応答は SSE でストリーム、送信は REST）
D) **REST + プッシュ通知のみ**（応答は POST 同期、能動通知は APNs/FCM のみ）
E) Other（[Answer]: に記入）

> 💡 **指針**: ハッカソン速度なら A or D。デモ映え（応答ストリーミング）狙うなら B or C。

[Answer]: D

---

### Question 3 — AgentCore Runtime（F3 / Nova Act）の起動方式

Hono Lambda から AgentCore Runtime をどう起動しますか？

A) **同期起動**（Lambda が `InvokeAgentRuntimeCommand` で AgentCore Runtime 完了を待ち、結果を返す。Lambda 15 分 limit に注意）
B) **非同期起動 + EventBridge**（Lambda は AgentCore Runtime をキック後すぐ返却、完了は EventBridge → Lambda → ユーザー通知）
C) **非同期起動 + SQS**（Lambda はメッセージキューに投入、別 Lambda がワーカーとして AgentCore Runtime を起動）
D) **AgentCore 内完結**（EventBridge Scheduler から直接 AgentCore Runtime をキック、Hono Lambda 経由しない）
E) Other（[Answer]: に記入）

> 💡 **指針**: F3 完了まで 30 秒〜2 分の見込み（requirements.md §5.3）。同期 (A) なら API Gateway 29 秒 timeout を超えるリスクあり → **B or D 推奨**。

[Answer]: B

---

### Question 4 — メトリクス更新のタイミング

ダメ度メトリクス（ママ溺愛度・ママ任せ度・ママ介入度）はいつ更新しますか？

A) **イベント駆動リアルタイム**（各機能ストーリーのアクション完了時に同期で DynamoDB 更新）
B) **EventBridge 経由非同期**（アクション完了時に EventBridge にイベント発行 → 別 Lambda が DynamoDB 更新）
C) **バッチ（5 分ごと）**（各種ログを集計して定期更新、ダッシュボード表示は 5 分前の値）
D) **ハイブリッド**（重要メトリクスはリアルタイム、細かいカウンタは EventBridge 経由）
E) Other（[Answer]: に記入）

> 💡 **指針**: デモ映え（ゲージがリアルタイムに上がる）狙うなら A or B。実装単純さなら A。

[Answer]: B

---

### Question 5 — 認証・認可方式

MVP のユーザー認証をどうしますか？

A) **Amazon Cognito User Pool**（標準的、将来運用化を見据える）
B) **認証なし（デモ用固定ユーザー）**（5/30 予選デモ時はフィクスチャユーザーで動作、認証 UI は実装しない）
C) **Magic Link / OTP**（メールアドレスのみ、パスワード不要、Cognito + SES）
D) **OAuth 2.0（Google ログイン）**（Google Calendar 連携と同一の OAuth クライアントを再利用）
E) Other（[Answer]: に記入）

> 💡 **指針**: 5/30 予選デモは固定ユーザー (B) で十分。決勝以降に切り替える前提なら B 推奨。
> 6/26 決勝までに切り替えるなら D（Google ログイン）が OAuth 連携と統一できて有利。

[Answer]: D

---

### Question 6 — DynamoDB アクセスパターン

データアクセス層の抽象化レベルは？

A) **直接 SDK 呼出**（Lambda 内で `@aws-sdk/client-dynamodb` を直接利用、最小実装）
B) **薄い Repository レイヤ**（テーブルごとにリポジトリクラス、テストで mock しやすい）
C) **Data Mapper / ORM 風**（DynamoDB Toolbox 等のライブラリで型安全・スキーマ管理）
D) Other（[Answer]: に記入）

> 💡 **指針**: PBT（純ロジック）と分離する観点では B 推奨。ハッカソン速度なら A も可。

[Answer]: B

---

### Question 7 — フロントエンドの State 管理

React Native + Expo の状態管理ライブラリは？

A) **Zustand**（軽量、ボイラープレート少、デモ向け）
B) **Redux Toolkit + RTK Query**（堅牢、API キャッシュ機能、学習コスト高）
C) **React Context + useReducer のみ**（外部ライブラリ不要、規模小なら十分）
D) **TanStack Query（React Query）+ Zustand**（API は TQ、UI 状態は Zustand）
E) Other（[Answer]: に記入）

> 💡 **指針**: ハッカソン規模なら A or C。API キャッシュ（チャット履歴等）を意識するなら D。

[Answer]: A

---

### Question 8 — アーキテクチャパターン（バックエンド）

バックエンドの内部構造は？

A) **シンプル層分離**（Lambda handler → service → repository の 3 層、最小限）
B) **Hexagonal / Clean Architecture**（domain / application / infrastructure を明確分離、PBT との親和性高）
C) **機能ベース構造**（feature/ ディレクトリ単位で全部入り、横展開しやすい）
D) Other（[Answer]: に記入）

> 💡 **指針**: PBT を効かせるなら B（純ロジックを domain に隔離）。ハッカソン速度なら A or C。

[Answer]: B

---

## 4. 回答後の流れ

1. ユーザーが Q1〜Q8 に `[Answer]:` で回答
2. AI が回答分析、曖昧さあれば clarification ファイル
3. AI が承認プロンプト提示
4. 承認 → Part 2（生成）実行 → 5 つの設計ドキュメント生成
5. 完了メッセージ → Units Generation ステージへ

---

## 5. 参照
- `aidlc-docs/inception/requirements/requirements.md`
- `aidlc-docs/inception/user-stories/stories.md`
- `aidlc-docs/inception/user-stories/personas.md`
- `aidlc-docs/inception/plans/execution-plan.md`
- `.aidlc-rule-details/inception/application-design.md`
