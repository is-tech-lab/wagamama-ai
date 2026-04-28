# Requirements Clarification Questions - Ghost Host

Ghost Hostアプリの要件を明確にするための質問です。
各質問の `[Answer]:` タグの後に選択肢の文字を記入してください。
選択肢に合うものがない場合は最後の選択肢（X/E など）を選び、説明を追記してください。

---

## Question 1
このプロジェクトのゴールは何ですか？

A) AWSハッカソンのデモ用PoC（動けばOK、本番品質不要）
B) ハッカソン後も実際に使い続けるプロダクト
C) ハッカソン用だが、そのままプロダクションに昇格させたい
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 2
ハッカソンのデモで最優先に動かしたい機能はどれですか？

A) イベント検知 → メッセージ自動生成・送信（コアフロー）
B) ギフト選定・発注まで含めたエンドツーエンドの自律実行
C) ダッシュボードUI（ユーザーが状況を確認できる画面）
D) 上記すべてをデモできる状態にしたい
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 3
「ターゲット（相手）」のデータソースとして、ハッカソンで実際につなぎたいものはどれですか？

A) Googleカレンダー（記念日・予定の取得）
B) Gmail / メール（近況・イベント検知）
C) モックデータ（実際のAPI連携は不要、ダミーデータで動作確認）
D) A＋B両方
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 4
メッセージ送信先として想定しているチャネルはどれですか？

A) メール（Amazon SES）
B) SMS（Amazon Pinpoint）
C) Slack / LINE / その他チャットツール
D) 送信はモック（実際には送らない、ログに記録するだけ）
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 5
ギフト選定・発注機能の扱いはどうしますか？

A) Amazon.co.jpなど実際のECサイトAPIと連携する
B) ギフト選定まで自動化し、発注は人間が確認してから行う
C) ギフト選定はAIが提案するだけ（モック）、実際の発注はスコープ外
D) ギフト機能はハッカソンスコープ外
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 6
フロントエンド（UI）はどうしますか？

A) WebアプリUI（React / Next.js など）を作る
B) AWS ConsoleやStep Functions GUIで操作・確認できればOK
C) CLIやAPIのみでOK（UIは不要）
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 7
「あなたらしい」メッセージ生成に使うパーソナライズ情報は何を使いますか？

A) ユーザーが事前に登録したプロフィール・文体サンプル
B) 過去のメール・チャット履歴を学習させる
C) ハッカソンではプロンプトに文体指示を書くだけ（簡易実装）
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 8
AWSリージョンはどこを使いますか？

A) ap-northeast-1（東京）
B) us-east-1（バージニア）
C) その他（Bedrockの利用可能リージョンに合わせる）
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 9
インフラのデプロイ方法はどうしますか？

A) AWS CDK（TypeScript）
B) AWS SAM
C) Terraform
D) マネジメントコンソールから手動
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 10
バックエンドの実装言語はどれにしますか？

A) Python（Lambda等で最もシンプル）
B) TypeScript / Node.js
C) Java
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question: Security Extensions
このプロジェクトにセキュリティ拡張ルールを適用しますか？

A) Yes — すべてのSECURITYルールをブロッキング制約として適用（本番グレード推奨）
B) No — SECURITYルールをスキップ（PoC・プロトタイプ・実験的プロジェクト向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question: Property-Based Testing Extension
プロパティベーステスト（PBT）ルールを適用しますか？

A) Yes — すべてのPBTルールをブロッキング制約として適用
B) Partial — 純粋関数とシリアライゼーションのラウンドトリップのみに適用
C) No — PBTルールをスキップ（シンプルなCRUD・UIのみのプロジェクト向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: 
