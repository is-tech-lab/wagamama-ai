# User Stories — わがママAI

**作成日**: 2026-05-08
**前提**: requirements.md（ハッカソン版改訂済み）, personas.md, story-generation-plan.md（承認済み）
**スコープ**: MVP 5 機能（F1, F2, F3, F5, F6）— F4 は MVP 外（requirements.md §4.3 / §6 参照）

---

## 0. このドキュメントの位置づけ

本ドキュメントは **実装契約**としての User Stories 集です。
- **プロダクトナラティブ・デモシナリオ**は別ドキュメント [`README.md`](../../../README.md) を参照（書類審査・予選デモ向けプレゼン資料）
- **要件・設計思想・NFR**は [`requirements.md`](../requirements/requirements.md) を参照
- **倫理境界・依存度調整・緊急停止**は requirements.md §2.4 に記述（本ドキュメントでは Story として独立化しない / Q7=D）
- **Security Baseline / Property-Based Testing** は Construction Phase の NFR Requirements / NFR Design で扱う（Q8=D, Q12=C）

ストーリーは **Hybrid: Journey × Feature**（Q5=E）で構成：朝 → 昼 → 夜 のシナリオ順に F1〜F6 をマッピング。

---

## 1. Demo Narrative（要約）

> 朝 7:30、田中翔太（28 歳・IT エンジニア）はスマホを見る。
> 👕 ママから「**今日は上着羽織りなさい**」のチャット + コーデ画像が届いている（F2）
> 🍱 「**ランチはサラダ多めにしな**」も決まっている（F2）
> 🍽 「**昨晩のうちに食べログで店予約しといた**よ。**Uber も頼んどいた**」（F3）
> 💖 そして **ママ溺愛度 97%** のゲージが上がっている（F5）
> 翔太は **何も決めていない、何も実行していない、なのに 1 日が回っている**。

詳細なナラティブ・スクリーンショット・GIF は [`README.md`](../../../README.md) を参照。

---

## 2. Story 一覧

| Epic | Feature | Story 数 | 5 設計選択カバー |
|---|---|:-:|---|
| **E1** | F1 ママ人格チャット UI (D1) | 3 | (4) キャラクター人格 |
| **E2** | F2 ママの先回りアドバイス (A1+A2) | 4 | (1) 能動的介入 |
| **E3** | F3 🔥 ママの代理ブラウザ操作 (B1) | 5 | (2) 実行代行 + (3) 判断機会の剥奪 + (5) 画面外への介入 |
| **E4** | F5 ママ溺愛度ダッシュボード (D3) | 1 | 評価軸の可視化 |
| **E5** | F6 スマホデータ統合 (D2) | 3 | 基盤 |
| | **合計** | **16** | 5 設計選択すべてカバー |

---

## 3. Stories

### Epic E1：ママ人格チャット UI（F1 / D1）

**Goal**: LINE 風 UI でママ人格と会話できる土台を提供する。

---

#### Story E1-S1：ママ人格との基本チャット会話

**As** メインペルソナ（田中翔太・佐藤美咲）
**I want** ママ人格と LINE 風 UI でテキストチャットしたい
**So that** 仕事帰りや朝、ママに話を聞いてもらえる安心感を得たい

**Acceptance Criteria**

- **Given** ユーザーがアプリを起動しチャット画面を開いている
  **When** ユーザーがテキストメッセージを送信する
  **Then** Bedrock Claude がママ口調で 5 秒以内に応答を返し、LINE 風吹き出しに表示される

- **Given** ユーザーが連続してメッセージを送信する
  **When** 直近 10 ターンの会話履歴がコンテキストに含まれる
  **Then** ママの応答は前後の文脈を踏まえた一貫性のある内容になる

- **Given** ママの応答が生成される
  **When** 応答テキストが UI にレンダリングされる
  **Then** ママらしい口調（「〜しなさいよ」「〜じゃない？」「あんた」等）が含まれる

**ダメ度メトリクス影響**: ママ溺愛度 +（自発的会話量、ありがとう率）
**関連要件**: requirements.md §4.2 F1, §5.2.1

---

#### Story E1-S2：初回オンボーディング（包括同意 + 基本プロファイル）

**As** 新規ユーザー
**I want** 初回起動時にママに自分の情報（名前・呼び方・PII 連携同意）を教えたい
**So that** ママがパーソナライズされた応答と先回り介入をできるようにしたい

**Acceptance Criteria**

- **Given** ユーザーがアプリを初めて起動する
  **When** オンボーディング画面が表示される
  **Then** 名前・呼び方・PII 連携範囲（カレンダー・位置・健康・購買）への同意取得が順次行われる

- **Given** ユーザーが PII 連携を承諾する
  **When** 同意フラグが DynamoDB に保存される
  **Then** Google Calendar OAuth フロー（E5-S1）への導線が表示される

- **Given** オンボーディング完了
  **When** ユーザーがチャット画面に遷移する
  **Then** ママから「**よろしくね、〇〇ちゃん**」のウェルカムメッセージが届く

**関連要件**: requirements.md §2.4（ユーザー同意）, §5.4

---

#### Story E1-S3：チャット履歴の永続化と再開

**As** メインペルソナ
**I want** 過去のチャット履歴をいつでも見返したい
**So that** ママとの関係性に積み上がりを感じたい

**Acceptance Criteria**

- **Given** ユーザーが過去にメッセージ送受信履歴を持っている
  **When** ユーザーがアプリを再起動しチャット画面を開く
  **Then** 直近 100 件の会話が時系列で表示される

- **Given** チャット履歴が 100 件を超える
  **When** ユーザーが画面上端にスクロールする
  **Then** 過去メッセージが追加でロードされる（無限スクロール）

- **Given** ユーザーがメッセージを送信する
  **When** 送信処理が完了する
  **Then** メッセージが DynamoDB に永続化される（送信失敗時はリトライ表示）

**ダメ度メトリクス影響**: ママ溺愛度 +（利用頻度カウント）
**関連要件**: requirements.md §4.2 F1, §5.2.1（DynamoDB 状態管理）

---

### Epic E2：ママの先回りアドバイス（F2 / A1+A2）

**Goal**: ユーザーが起きる前に、ママから服・食事のアドバイスがチャットに届く能動的介入を実現する。

---

#### Story E2-S1：朝のコーディネートアドバイス

**As** メインペルソナ（田中翔太・佐藤美咲）
**I want** 起きる前にママから今日の服装アドバイスを受け取りたい
**So that** 服を選ぶ判断疲れから解放されたい

**Acceptance Criteria**

- **Given** ユーザーが PII 連携同意済み（カレンダー・位置）
  **When** EventBridge Scheduler が起床想定時刻 30 分前にトリガーする
  **Then** その日の天気・予定・気温を踏まえたコーディネート文言が Bedrock Claude で生成される

- **Given** コーディネート文言が生成される
  **When** ママ口調にフォーマットされる
  **Then** 「**今日は上着羽織っていきなさいよ、肌寒いから**」のような押し付けがましいトーンになる

- **Given** ユーザーが朝起きる
  **When** ユーザーがアプリを開く
  **Then** チャット画面の最新メッセージとしてアドバイスが表示済みである

**ダメ度メトリクス影響**: ママ介入度 +（能動通知数カウント）
**関連要件**: requirements.md §4.1 A1, §4.2 F2, §5.2.1（EventBridge）

---

#### Story E2-S2：コーディネート簡易画像生成

**As** メインペルソナ
**I want** ママのコーデアドバイスに簡易イラスト画像が添えられていてほしい
**So that** 文字だけより直感的に「今日の服」がイメージできる

**Acceptance Criteria**

- **Given** E2-S1 のコーディネート文言が生成済み
  **When** Bedrock Titan Image Generator（または Stable Diffusion XL）に画像プロンプトが送信される
  **Then** 30 秒以内にイラスト的なコーデ案画像が生成される

- **Given** 画像生成完了
  **When** S3 に保存される
  **Then** プリサインド URL がチャットメッセージに添付される

- **Given** ユーザーがチャット画面を開く
  **When** 画像付きメッセージが表示される
  **Then** 画像はインライン表示され、タップで拡大可能である

**ダメ度メトリクス影響**: ママ溺愛度 +（コンテンツ濃度）
**関連要件**: requirements.md §4.2 F2, §5.2.1（Bedrock Titan Image / S3）

---

#### Story E2-S3：昼前の食事アドバイス（メインペルソナ向け）

**As** メインペルソナ（田中翔太・佐藤美咲）
**I want** 昼前にママから食事アドバイスを受け取りたい
**So that** ランチに何を食べるか考えたくない

**Acceptance Criteria**

- **Given** ユーザーが健康データ連携同意済み
  **When** EventBridge Scheduler が 11:30 にトリガーする
  **Then** 直近の食事傾向・栄養バランスを踏まえた食事アドバイスが Bedrock Claude で生成される

- **Given** 食事アドバイスが生成される
  **When** ママ口調にフォーマットされる
  **Then** 「**最近野菜足りてないわよ、ランチはサラダ多めにしなさい**」のようなトーンになる

- **Given** アドバイスが生成完了
  **When** ユーザーへチャット通知される
  **Then** 12:00 までに必ずユーザーへ届く

**ダメ度メトリクス影響**: ママ介入度 +、ママ溺愛度 +
**関連要件**: requirements.md §4.1 A2, §4.2 F2

---

#### Story E2-S4：食事アドバイス（サブペルソナ向け：自炊できない若者）

**As** サブペルソナ（山田結菜・新卒上京組）
**I want** 自炊スキルがなくても買えるコンビニ食材レベルでアドバイスがほしい
**So that** 「外食しろ」「自炊しろ」と言われても困らない

**Acceptance Criteria**

- **Given** ユーザープロファイルが「自炊スキル: 低」「居住期間: 1 年未満」と設定されている
  **When** E2-S3 と同じスケジューラがトリガーする
  **Then** 食事アドバイスが「**コンビニで〇〇のサラダ買いな、200 円台で野菜摂れるから**」など具体的な購入導線を含むトーンになる

- **Given** サブペルソナ向けコンテンツが生成される
  **When** ユーザーへ届く
  **Then** メイン向けコンテンツ（E2-S3）とは異なるトーン・粒度になっている

**ダメ度メトリクス影響**: ママ溺愛度 +（情緒依存強化）
**関連要件**: requirements.md §1.3 サブペルソナ, §4.1 A2

---

### Epic E3：ママの代理ブラウザ操作（F3 / B1）🔥

**Goal**: Nova Act × AgentCore Browser で実 Web サイトをママが代理操作し、ユーザーから「自分で予約・注文する」機会を完全に奪う。**本サービスの核**。

---

#### Story E3-S1：Nova Act による飲食店予約（成功フロー）

**As** メインペルソナ
**I want** 夕食の店をママに勝手に予約しておいてほしい
**So that** 「今日どこ行こう」と考えなくていい

**Acceptance Criteria**

- **Given** ユーザーのスケジュール・気分・食事傾向が把握できる
  **When** EventBridge Scheduler が前日夜にトリガーする
  **Then** Hono Lambda が `InvokeAgentRuntimeCommand` で AgentCore Runtime（Python・Nova Act ワークフロー）を起動する

- **Given** AgentCore Runtime 起動
  **When** `browser_session()` で AgentCore Browser から CDP endpoint と headers を取得し、Nova Act が `cdp_endpoint_url` 経由で食べログに接続する
  **Then** Nova Act が自然言語指示「**野菜が多いランチを 19 時に予約して**」で予約フォームを操作完了する

- **Given** 予約完了
  **When** Nova Act が結果（店名・時間・確認番号）を JSON で返す
  **Then** Hono Lambda がママ口調メッセージを生成しチャットに通知する：「**昨晩のうちに〇〇予約しといたよ、19 時ね**」

**ダメ度メトリクス影響**: ママ任せ度 +（無承認実行数カウント、最重要）
**関連要件**: requirements.md §4.2 F3, §5.2.2（AgentCore + Nova Act 連携パターン）

---

#### Story E3-S2：予約失敗 / CAPTCHA 検知時のフォールバック

**As** メインペルソナ
**I want** ママが予約に失敗した時も適切に通知してほしい
**So that** 当日「予約してたはず」のトラブルを避けたい

**Acceptance Criteria**

- **Given** Nova Act が予約フォーム操作中
  **When** CAPTCHA・ログイン要求・在庫切れなどで操作が完了できない
  **Then** AgentCore Browser の Human-in-the-loop takeover メカニズムが発動する、または Nova Act がエラー結果を返す

- **Given** Nova Act がエラーで終了
  **When** Hono Lambda がエラー結果を受信する
  **Then** ママ口調で「**予約できなかったわ、別の店探しとくね**」のフォールバック通知がチャットに届く

- **Given** 連続 3 回失敗
  **When** ユーザーへエスカレートする
  **Then** 「**ごめん、今日はママもお手上げ。〇〇か△△で予約してくれる？**」と候補を提示する

**ダメ度メトリクス影響**: ママ任せ度（失敗時もカウント、ただし達成度は減）
**関連要件**: requirements.md §4.2 F3, §5.2.2（AgentCore Browser ライブビュー / takeover）

---

#### Story E3-S3：予約完了後の Google Calendar 自動登録

**As** メインペルソナ
**I want** 予約された予定が自動的に自分のカレンダーに入っていてほしい
**So that** 「あれ予約したっけ？」を忘れない

**Acceptance Criteria**

- **Given** E3-S1 で予約成功し、店名・時間・予約番号が取得できている
  **When** Nova Act ワークフロー内で AgentCore Identity の `@requires_access_token` デコレータが Google OAuth トークンを注入する
  **Then** Google Calendar API でユーザーのカレンダーに予約イベントが作成される

- **Given** Calendar 登録成功
  **When** イベント詳細が反映される
  **Then** タイトル「**ママが予約: 〇〇（19:00）**」、場所、メモ（予約番号）が登録されている

- **Given** Calendar 登録失敗（OAuth トークン期限切れ等）
  **When** AgentCore Identity がリフレッシュフローで自動再取得する
  **Then** ユーザーへの再認証要求は不要、もしくは最小限になる

**関連要件**: requirements.md §4.2 F3, §5.2.2（AgentCore Identity）

---

#### Story E3-S4：予約完了後の SES 確定メール風通知

**As** メインペルソナ
**I want** ママから「予約確定メール」っぽい通知も届くようにしたい
**So that** 「ちゃんとやってくれてる」感が増す

**Acceptance Criteria**

- **Given** E3-S1 で予約成功
  **When** Hono Lambda が SES SendEmail を実行する
  **Then** ユーザーの登録メールアドレスに「**予約確定のお知らせ（ママより）**」件名のメールが届く

- **Given** メール本文
  **When** メール内容が表示される
  **Then** 店名・時間・場所・予約番号と、ママ口調の一文「**ちゃんと行きなさいよ**」が含まれている

**ダメ度メトリクス影響**: ママ介入度 +（能動通知数）
**関連要件**: requirements.md §4.2 F3, §5.2.2（SES）

---

#### Story E3-S5：AgentCore Browser Live View（デモ専用）

**As** ハッカソン審査員
**I want** 予選・決勝デモで「ママが食べログを操作している様子」を視覚的に確認したい
**So that** 「画面外への貫通」が実演されていることを納得したい

**Acceptance Criteria**

- **Given** デモ実行中
  **When** プレゼンターが AWS Console で AgentCore Browser の Live View を開く
  **Then** Nova Act が食べログ・Uber Eats Web を実際に操作している様子がライブ配信される

- **Given** Live View が表示される
  **When** Nova Act がフォーム入力・ボタンクリック等を実行する
  **Then** 各操作ステップがリアルタイムで視認できる

- **Given** デモ終了後
  **When** 審査員が録画を確認する
  **Then** 操作履歴が CloudWatch Logs に残っている（決勝向けに AgentCore Observability を統合する場合）

**関連要件**: requirements.md §4.2 F3, §5.2.2（AgentCore Browser Live View）, §5.2.3（決勝向け Observability）

---

### Epic E4：ママ溺愛度ダッシュボード（F5 / D3）

**Goal**: ダメ度メトリクス 3 軸を可視化し、ユーザー本人が「自分のダメ度」をエンタメ的に確認できる。

> **Note**: Q10=D により F5 は 1 ストーリーで完結。各メトリクスの**更新条件**は他 Story の AC（「ダメ度メトリクス影響」セクション）に分散して記述済み。

---

#### Story E4-S1：ダメ度メトリクス 3 軸ダッシュボード

**As** メインペルソナ・サブペルソナ
**I want** 自分の「ママ溺愛度・ママ任せ度・ママ介入度」をリアルタイムで見たい
**So that** 自分がどれだけママに依存しているかをエンタメ的に把握したい

**Acceptance Criteria**

- **Given** ユーザーがダッシュボード画面を開く
  **When** メトリクス取得 API が DynamoDB から最新値を取得する
  **Then** 3 軸（💖 ママ溺愛度・🙏 ママ任せ度・👁 ママ介入度）が円形ゲージで表示される

- **Given** 各メトリクスが計算式（requirements.md §2.3）に従って算出済み
  **When** ゲージが描画される
  **Then** 各メトリクスのパーセンテージ・直近 7 日のトレンド・「ヤバいランク」（コメント）が表示される

- **Given** メトリクス更新条件が他 Story の実行で発生
  **When** 該当アクション（チャット送受信・F2 通知受信・F3 無承認実行等）が起きる
  **Then** メトリクスがほぼリアルタイム（最大 1 分以内）で更新される

- **Given** メトリクスが目標値を超える
  **When** ダッシュボードが描画される
  **Then** 「**🎉 ママ溺愛度 95%！ あんた、ママなしじゃ生きていけないわね**」のママ口調コメントが表示される

**関連要件**: requirements.md §2.3, §4.2 F5, §5.2.1（DynamoDB）

---

### Epic E5：スマホデータ統合（F6 / D2）

**Goal**: F2/F3 を成立させる基盤として、カレンダー・天気・店検索データを統合する。

---

#### Story E5-S1：Google Calendar OAuth 連携

**As** メインペルソナ
**I want** 自分の Google Calendar をママに見せて勝手に書き込みも許可したい
**So that** 予約や予定をママが自動で扱えるようにしたい

**Acceptance Criteria**

- **Given** E1-S2 オンボーディング完了
  **When** ユーザーが Google Calendar 連携ボタンをタップする
  **Then** Google OAuth 2.0 認可フロー（scope: calendar.events）が起動する

- **Given** ユーザーが認可画面で承諾する
  **When** リフレッシュトークンが取得される
  **Then** AgentCore Identity に保存され、コードに認証情報を埋め込まずに F3 から再利用可能になる

- **Given** トークンが期限切れ
  **When** Nova Act ワークフローが Calendar API を呼ぶ
  **Then** AgentCore Identity が自動リフレッシュし、ユーザー操作は不要

**関連要件**: requirements.md §4.2 F6, §5.2.2（AgentCore Identity）, §5.4（OAuth 集中管理）

---

#### Story E5-S2：天気データ取得（F2 のアドバイス精度向上）

**As** メインペルソナ
**I want** ママのコーデアドバイスが今日の天気と整合してほしい
**So that** 「上着羽織りな」と言われた日が実際に肌寒い日であってほしい

**Acceptance Criteria**

- **Given** EventBridge Scheduler が朝のコーデアドバイス（E2-S1）を起動する
  **When** Hono Lambda が天気 API（無料公開 API）にユーザー位置で問い合わせる
  **Then** 当日最高/最低気温・降水確率・天気概況が取得される

- **Given** 天気データが取得される
  **When** Bedrock Claude のプロンプトに含まれる
  **Then** 生成されるコーデアドバイスが当日の気温に整合する（最高 18 度未満なら「上着」が含まれる、など）

**関連要件**: requirements.md §4.2 F6, §5.2.1（外部 API）

---

#### Story E5-S3：ホットペッパー店検索（F3 の予約候補絞り込み）

**As** メインペルソナ
**I want** F3 の予約対象が「**ユーザーが好きそうな店**」に絞られていてほしい
**So that** ママが選ぶ店がハズレばかりだと困る

**Acceptance Criteria**

- **Given** F3 が予約処理を起動する
  **When** Hono Lambda がホットペッパー API（無料）でユーザー位置・好み（過去予約傾向）に基づき店検索する
  **Then** 候補店リスト（最大 10 件）が取得される

- **Given** 候補店リストが取得される
  **When** AgentCore Runtime に渡される
  **Then** Nova Act が候補店リストの中から最適な 1 店を選択し食べログで予約する

**関連要件**: requirements.md §4.2 F6, §5.2.1（ホットペッパー API）

---

## 4. ストーリー粒度・本数の検証

| 観点 | 値 |
|---|---|
| 総ストーリー数 | **16 本** |
| Q3 (15-20 範囲) | ✅ 範囲内 |
| Q14 (12-20 範囲) | ✅ 範囲内 |
| INVEST Independent | ✅ 各 Story が独立して実装・テスト可能 |
| INVEST Negotiable | ✅ AC は実装合意の起点 |
| INVEST Valuable | ✅ 各 Story がペルソナ価値に紐づく |
| INVEST Estimable | ✅ 1 Story = 1〜3 日想定 |
| INVEST Small | ✅ 平均 2 日相当 |
| INVEST Testable | ✅ GWT 形式の AC 完備 |
| 5 設計選択カバレッジ | ✅ (1) F2 / (2)(3) F3 / (4) F1 / (5) F3 |

---

## 5. 参照

- `aidlc-docs/inception/requirements/requirements.md`（要件・設計思想・NFR・倫理境界 §2.4）
- `aidlc-docs/inception/user-stories/personas.md`（ペルソナ詳細 + マッピング表）
- `aidlc-docs/inception/plans/story-generation-plan.md`（生成計画・Q&A）
- `README.md`（プロダクトナラティブ・デモシナリオ・スクリーンショット）
