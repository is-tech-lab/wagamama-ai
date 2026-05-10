# Story Generation Plan — わがママAI

**作成日**: 2026-05-08
**前提**: requirements.md（ハッカソン版改訂済み）、user-stories-assessment.md（Execute = Yes）
**Stage**: INCEPTION - User Stories（Part 1: Planning）

---

## 0. 本プランの読み方

このドキュメントは User Stories の **生成計画**（Part 1: Planning）です。
ユーザーは下部の各 Question に `[Answer]:` タグで回答してください。回答が揃ったら本ドキュメント上部の「Plan 実行チェックリスト」に従って Part 2（生成）を実行します。

---

## 1. Plan 実行チェックリスト（Part 2 実行用）

> Part 2（生成）はユーザー回答 + Plan 承認後に実行されます。各ステップ完了時に `[ ]` → `[x]` へ更新します。

- [x] **STEP 1**: requirements.md と本プラン（回答済み）をロード
- [x] **STEP 2**: 採用したストーリー breakdown 戦略（Q5=E Hybrid）に沿ってストーリー骨格を構築
- [x] **STEP 3**: ペルソナ定義（Q1=B 主 2 体, Q2=C サブ 1 体）→ `aidlc-docs/inception/user-stories/personas.md` 生成
- [x] **STEP 4**: F1, F2, F3, F5, F6（MVP 5 機能、F4 カット）コアストーリーを生成（INVEST 準拠）
- [x] **STEP 5**: Acceptance Criteria を GWT 形式（Q4=A）で各ストーリーに付与
- [x] **STEP 6**: 倫理境界・Security Baseline は Q7=D / Q8=D / Q12=C により stories.md では触れず、requirements.md §2.4 / Construction Phase に委譲
- [x] **STEP 7**: 5/30 予選デモシナリオを stories.md §1 Demo Narrative として簡略記述、詳細は README.md（Q6=B）
- [x] **STEP 8**: ペルソナ ↔ ストーリーマッピング表を `personas.md` 末尾に追加
- [x] **STEP 9**: Q3=B（15〜20 本）に従いエピック分割、合計 16 本生成
- [x] **STEP 10**: `aidlc-docs/inception/user-stories/stories.md` 生成完了
- [x] **STEP 11**: `aidlc-state.md` を更新し User Stories ステージ進捗を記録
- [x] **STEP 12**: 完了メッセージ（user-stories.md Step 20 形式）を提示

---

## 2. ストーリー breakdown 戦略の候補（Q5 で選択）

| 戦略 | 概要 | 利点 | 欠点 | わがママAI への適合度 |
|---|---|---|---|---|
| **A. User Journey-Based** | 「朝のシナリオ」「夜のシナリオ」など時系列で並べる | デモシナリオ「おはよう…」と直結／審査員が体験を追いやすい | 機能の独立性が見えづらい／Unit 分解時に再整理が必要 | ◎ |
| **B. Feature-Based** | F1〜F6 ごとに独立してストーリー化 | requirements.md と 1:1 対応／Unit 分解が容易 | デモシナリオ全体の流れが見えづらい | ○ |
| **C. Persona-Based** | メイン／サブペルソナごとにストーリー群を分離 | ペルソナ間の体験差が明確 | 機能重複が発生／MVP 集中度が下がる | △ |
| **D. Domain-Based** | 機能カテゴリ A/B/C/D（思考代行・実行代行・先回り介入・人格関係性）で並べる | requirements.md §4.1 と整合／Unit 設計に直結 | カテゴリ C は Out of Scope なのでバランスが偏る | ○ |
| **E. Hybrid（Journey × Feature）** | デモシナリオを骨格に、各シーンを F1〜F6 にマッピング | デモ映え + Unit 分解の両立／審査・実装の双方を満たす | 整理コストがやや高い | ◎（推奨） |

---

## 3. Mandatory Artifacts（生成必須物）

- [ ] `aidlc-docs/inception/user-stories/stories.md` — INVEST 準拠の User Stories
- [ ] `aidlc-docs/inception/user-stories/personas.md` — ペルソナ archetypes
- [ ] 全ストーリーに Acceptance Criteria 付与
- [ ] ペルソナ ↔ ストーリーマッピング表
- [ ] 倫理境界・Security Baseline を横断 AC として埋め込み

---

## 4. Questions（ユーザー回答必須）

> 各質問に `[Answer]:` タグへ A/B/C... の文字、もしくは X) Other の場合は説明を記入してください。
> 回答完了後、「done」または「回答完了」とお知らせください。

---

### Question 1 — メインペルソナの解像度

requirements.md §1.3 では「**一人暮らしの 20 代後半〜30 代社会人。仕事の決断疲れで、生活の判断まで自分でしたくない人**」と定義されています。ストーリー作成時、メインペルソナをどの解像度で描きますか？

A) 1 体の代表ペルソナ（名前・職業・1 日の流れ・ダメ度メトリクス目標値まで具体化）
B) 2 体（職業や生活パターンが異なる代表例を 2 つ作り、共通点と差分を示す）
C) 1 体 + 簡易補足（代表ペルソナ 1 体 + サブペルソナを軽く触れる）
D) 既存記述のまま（追加詳細は不要、抽象レベルで進める）
E) Other（[Answer]: 行に記入）

[Answer]: B

---

### Question 2 — サブペルソナの扱い

requirements.md §1.3 のサブペルソナ「**親元を離れたばかりの若年層。生活初心者で、何でもママに頼りたい層**」をどう扱いますか？

A) MVP では扱わない（決勝後フェーズで追加。stories.md でも触れない）
B) 言及のみ（personas.md に概要だけ記載、ストーリー作成にはほぼ反映しない）
C) MVP の一部ストーリーに登場させる（例: F2 食事アドバイスはサブペルソナ向けに「自炊できない若者」シナリオを追加）
D) メインと同等に扱う（全 MVP ストーリーにサブペルソナ視点も併記）
E) Other（[Answer]: 行に記入）

[Answer]: C

---

### Question 3 — ストーリー粒度（INVEST の Small）

ハッカソンの実装ボリュームと Unit 分解の容易さを踏まえ、ストーリーの粒度をどうしますか？

A) 機能 = 1 ストーリー（F1〜F6 の 6 ストーリー + 横断 2〜3 本）。粗めだがデモ直結
B) 機能 = エピック、それを 2〜4 個の子ストーリーに分割（合計 15〜20 本）
C) 機能 = エピック、それを細かい子ストーリーに分割（合計 25〜35 本、実装タスクに近い粒度）
D) Other（[Answer]: 行に記入）

[Answer]: B

---

### Question 4 — Acceptance Criteria の記述形式

各ストーリーの受入基準フォーマットを指定してください。

A) **Given-When-Then（GWT）形式**（BDD 親和性高、テスト連携しやすい）
B) **チェックリスト形式**（プレーンな箇条書き、可読性優先）
C) **GWT + メトリクス目標値**（GWT に加え、ダメ度メトリクス・SLO の数値目標も併記）
D) **シナリオ + チェックリストのハイブリッド**（短いシナリオ説明 + チェックリスト）
E) Other（[Answer]: 行に記入）

[Answer]: A

---

### Question 5 — ストーリー breakdown 戦略

§2 の表から戦略を選択してください。

A) User Journey-Based（時系列・デモシナリオ起点）
B) Feature-Based（F1〜F6 ベース）
C) Persona-Based（ペルソナ別）
D) Domain-Based（カテゴリ A/B/C/D ベース）
E) **Hybrid: Journey × Feature**（推奨：デモシナリオ骨格 + 各シーンを F1〜F6 にマップ）
F) Other（[Answer]: 行に記入）

[Answer]: E

---

### Question 6 — デモシナリオ「おはよう、もう全部終わってるよ」のストーリー化

5/30 予選デモシナリオを stories.md にどう含めますか？

A) **トップに 1 本のデモストーリー**として配置し、残りは個別機能ストーリー（推奨）
B) ストーリー化はせず、stories.md 冒頭で「Demo Narrative」として軽く触れるだけ
C) 各機能ストーリーの中に「デモでの役割」セクションを設ける（横断扱い）
D) ストーリー化せず、別ドキュメント（demo-script.md など）に切り出す
E) Other（[Answer]: 行に記入）

[Answer]: B（Part 2 生成時に stories.md 冒頭に README へのリンクを埋め込む形に）

---

### Question 7 — 倫理境界・依存度調整・緊急停止の表現

requirements.md §2.4 で定義された「依存度調整・緊急解除手段・公序良俗 NG」は、Story 上どう扱いますか？

A) 専用の **横断ストーリー**を 1〜2 本作る（例: 「ユーザーは依存度を調整できる」「緊急停止できる」）
B) 全ストーリーの Acceptance Criteria に **Cross-Cutting AC** として埋め込む（個別ストーリー化はしない）
C) 専用ストーリー + 重要機能には Cross-Cutting AC も併記（推奨：審査での説得力強化）
D) requirements.md に既述のため stories.md では触れない
E) Other（[Answer]: 行に記入）

[Answer]: D

---

### Question 8 — Security Baseline Extension のストーリー反映

PII 取扱・OAuth トークン管理・KMS 暗号化など Security Baseline Extension の要件をストーリーに反映する方法を選んでください。

A) 専用の **NFR ストーリー**を作る（例: 「PII を暗号化して保存する」「OAuth トークンを安全に管理する」）
B) 各機能ストーリーの Acceptance Criteria に **Security AC** として埋め込む
C) A + B の併用（NFR ストーリー + 重要機能には Security AC も明示）
D) Construction Phase の NFR Requirements / Design に委ねる（stories.md では深追いしない）
E) Other（[Answer]: 行に記入）

[Answer]: D

---

### Question 9 — F3「Nova Act 代理ブラウザ操作」のストーリー深掘り度

F3 は本サービスの核（5 つの設計選択のうち 3 つを体現）。ストーリー粒度をどうしますか？

A) **F3 を 1 エピックとして 3〜5 子ストーリー**に分割（予約成功・予約失敗 / CAPTCHA・カレンダー登録・SES 通知・Live View デモ）
B) F3 を 1 ストーリーで簡潔に（残りはタスクとして開発時対応）
C) F3 + Calendar 登録 + SES 通知の 3 ストーリーに分割
D) Other（[Answer]: 行に記入）

[Answer]: A

---

### Question 10 — ダメ度メトリクス（F5 ダッシュボード）の表現

§2.3 のダメ度メトリクス（ママ溺愛度／ママ任せ度／ママ介入度）を Story にどう落とすか。

A) F5 を **可視化ストーリー 1 本**にし、メトリクス計算式は requirements.md 参照のみ
B) F5 を **可視化ストーリー + メトリクス計算ストーリー** の 2 本に分割
C) 各メトリクスを 3 本のストーリー（ママ溺愛度 / ママ任せ度 / ママ介入度）として独立化
D) F5 = 1 ストーリー + 各機能ストーリーの AC にメトリクス更新条件を埋め込む（推奨）
E) Other（[Answer]: 行に記入）

[Answer]: D

---

### Question 11 — 「ママ口調」「お節介度」の調整可能性

requirements.md §5.6 で「ママ人格（D1）の口調・距離感はパラメータで調整可能」と定義済み。この調整可能性を Story にどう反映？

A) 専用ストーリーを 1 本（「ユーザーはママの口調・お節介度を調整できる」）
B) D1 ストーリー内の Acceptance Criteria として表現
C) MVP 外（5/30 予選では固定値、決勝以降に調整 UI を実装）
D) Other（[Answer]: 行に記入）

[Answer]: C

---

### Question 12 — Property-Based Testing Extension の Story 反映

Extension で「純ロジック層は PBT 必須」が確定しています。Story には？

A) 各純ロジック系ストーリーの AC に **PBT 適用** を明記
B) 専用 NFR ストーリーで横断的に表現
C) Story では触れず、Construction Phase で Test 設計に委ねる
D) Other（[Answer]: 行に記入）

[Answer]: C

---

### Question 13 — ~~F4「ゼロタップ代理返信」の同意フロー~~（**N/A**）

⚠️ **2026-05-08 更新**: F4（代理コミュニケーション / B2）が MVP からカットされたため、本質問は**不要**となりました。requirements.md §4.3 Out of Scope および §6 将来展望に移動済み。

[Answer]: N/A（F4 カットに伴い不要）

---

### Question 14 — Story 数の目安（参考、強い制約ではない）

最終的な stories.md のストーリー本数の目安を教えてください（最終調整は AI 側で行います）。

A) 8〜12 本（コンパクト、デモ直結のみ）
B) 12〜20 本（標準、F1〜F6 + 横断 + デモストーリー）
C) 20〜30 本（詳細、エピック + 子ストーリー分割）
D) AI 側に任せる（推奨数で生成）
E) Other（[Answer]: 行に記入）

[Answer]: B

---

## 5. 回答後の流れ

1. ユーザーが全 14 問に `[Answer]:` で回答
2. AI が回答を分析し、曖昧さがあれば clarification ファイルを作成
3. すべて明確化 → AI が Plan 承認プロンプトを提示
4. ユーザー承認後、Part 2（生成）実行 → `stories.md` / `personas.md` 生成
5. 完了メッセージ提示 → Workflow Planning ステージへ

---

## 6. 参照

- `aidlc-docs/inception/requirements/requirements.md`
- `aidlc-docs/inception/plans/user-stories-assessment.md`
- `.aidlc-rule-details/inception/user-stories.md`
- `.aidlc-rule-details/extensions/security/baseline/security-baseline.md`（Extension Enabled）
- `.aidlc-rule-details/extensions/testing/property-based/property-based-testing.md`（Extension Enabled）
