# User Stories Assessment — わがママAI

**作成日**: 2026-05-08
**対象 Stage**: INCEPTION - User Stories
**前提**: requirements.md（ハッカソン版改訂済み）

---

## Request Analysis

- **Original Request**: AWS Summit Japan 2026 AI-DLC ハッカソン「人をダメにするサービスを考えよう！」テーマで、ユーザーの判断・実行・自己肯定を代行する「お母さん人格 AI（わがママAI）」を構築する。MVP（5/30 予選）→ AWS デプロイ済デモ（6/26 決勝）。
- **User Impact**: **Direct（強い直接影響）** — チャット UI でママとの会話、先回り通知、代理ブラウザ操作、ダメ度ダッシュボード等、すべてユーザー直結。
- **Complexity Level**: **Complex** — 6 つの MVP 機能（F1〜F6）、4 つの機能カテゴリ（A/B/C/D）、AgentCore + Nova Act のマルチサービス連携、複数ペルソナ。
- **Stakeholders**: メインペルソナ（決断疲れの 20〜30 代社会人）、サブペルソナ（親元を離れたばかりの若年層）、ハッカソン審査員（書類・予選・決勝）、開発チーム。

---

## Assessment Criteria Met

### High Priority（ALWAYS Execute）
- [x] **New User Features**: F1〜F6 すべて新規ユーザー向け機能
- [x] **Multi-Persona Systems**: メインペルソナ・サブペルソナの 2 種類が要件で明示済み
- [x] **Complex Business Logic**: 「人をダメにする」設計思想を実現する 5 つの非自明な設計選択（能動介入・実行代行・判断機会の剥奪・キャラクター人格・画面外介入）
- [x] **Cross-Team Projects**: ハッカソンチーム + 審査員 + 将来プロダクション昇格を見据えた設計が必要

### Medium Priority（Complexity Assessment）
- [x] **Scope**: F1〜F6 が複数 Unit（A/B/C/D カテゴリ）にまたがる
- [x] **Risk**: 「人をダメにする」設計のため、倫理境界・PII 取り扱い・依存度調整など審査でも問われる論点が多い
- [x] **Ambiguity**: 「ママ口調」「お節介度」「無承認実行の閾値」など、要件レベルでは抽象度が高くストーリーで具体化が必要
- [x] **Testing**: 予選デモ（5/30）の受け入れ基準を Acceptance Criteria に落とし込む必要

### Skip Criteria（該当なし）
- [ ] Pure Refactoring — 該当しない（Greenfield 新規構築）
- [ ] Isolated Bug Fixes — 該当しない
- [ ] Infrastructure Only — 該当しない（ユーザー向け機能含む）
- [ ] Documentation Only — 該当しない

### Expected Benefits
- **Acceptance Criteria の明文化**：5/30 予選デモ「おはよう、もう全部終わってるよ」シナリオの合格条件を定量化
- **ペルソナ定義**：メイン／サブペルソナの違いを明確化し、口調・通知頻度・ダメ度メトリクス目標値の設計指針を作る
- **ストーリー単位での独立実装**：F1〜F6 を Unit 分解可能にし、5/12 の application-design / unit-of-work 生成へ橋渡し
- **倫理境界の埋め込み**：依存度調整・緊急停止・公序良俗 NG をストーリーレベルで Acceptance Criteria に組み込み、審査リスクを低減
- **チーム共通理解**：ハッカソンチーム内で「人をダメにする UX」の合意形成

---

## Decision

**Execute User Stories**: ✅ **Yes**

**Reasoning**:
High Priority 6 軸中 4 軸ヒット（新規機能・マルチペルソナ・複雑ビジネスロジック・チーム連携）、Medium Priority も 4 軸ヒット。Skip 条件は完全に該当しない。さらにハッカソン書類審査（5/12）までに application-design / unit-of-work まで生成する必要があり、その前段である User Stories で機能の境界・ペルソナ・受入基準を明確化することはダウンストリームの品質を直接左右する。

---

## Expected Outcomes

1. **stories.md**: F1〜F6 を中核に、ペルソナ別の独立した INVEST ストーリー群（テストカバレッジ可能な Acceptance Criteria 付き）
2. **personas.md**: メインペルソナ・サブペルソナを掘り下げた 2〜3 体のユーザーアーキタイプ（ダメ度メトリクス目標値含む）
3. **倫理境界の組込み**: §2.4 の依存度調整・緊急停止を横断的 Acceptance Criteria として全ストーリーに反映
4. **5/30 予選デモの定量化**: 「おはよう、もう全部終わってるよ」シナリオの受入基準を Story Acceptance Criteria 形式で表現
5. **Unit 分解への橋渡し**: A/B/C/D カテゴリと F1〜F6 を Story → Unit にマップしやすい構造で出力

---

## 関連ドキュメント

- `aidlc-docs/inception/requirements/requirements.md`（ハッカソン版改訂済み）
- `aidlc-docs/aidlc-state.md`（Extension Configuration: Security Baseline ✅, Property-Based Testing ✅）
- `.aidlc-rule-details/inception/user-stories.md`（本 Stage 詳細手順）
