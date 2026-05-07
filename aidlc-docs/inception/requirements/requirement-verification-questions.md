# Requirements Clarification Questions - わがママAI

わがママAIアプリの要件を明確にするための質問です。
各質問の `[Answer]:` タグの後に選択肢の文字を記入してください。
選択肢に合うものがない場合は最後の選択肢（X/E など）を選び、説明を追記してください。

---

## Question 1
このプロジェクトのゴールは何ですか？

A) AWSハッカソンのデモ用PoC（動けばOK、本番品質不要）
B) ハッカソン後も実際に使い続けるプロダクト
C) ハッカソン用だが、そのままプロダクションに昇格させたい
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Question 2
ハッカソンのデモで最優先に動かしたい機能はどれですか？

A) コーディネート提案（毎朝着る服をAIが選んでくれる）
B) 料理手配（Uber Eats等を自動で注文してくれる）
C) プレゼント選定（友人・恋人へのギフトをAIが選んでくれる）
D) 上記すべてをデモできる状態にしたい
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 3
「ママ」がユーザーを理解するためにアクセスするスマホ情報の範囲はどこまでですか？

A) カレンダー（予定・スケジュール）のみ
B) カレンダー ＋ 位置情報
C) カレンダー ＋ 購買履歴
D) すべて（カレンダー、位置情報、購買履歴、ヘルスデータ等）
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 4
「ママ」がアクションを起こしたときの通知・インタラクション方法はどれですか？

A) プッシュ通知のみ（シンプル）
B) チャット形式（LINE・Slack風）でメッセージを送ってくれる
C) Slack / LINE / その他チャットツールに送信する
D) 音声（テキスト読み上げ）で話しかけてくれる
X) Other (please describe after [Answer]: tag below)

[Answer]: B
Dへの拡張の可能性はあります
---

## Question 5
プレゼント選定・発注機能の扱いはどうしますか？

A) AIが選ぶだけ（提案のみ、発注はユーザーが手動）
B) 選定まで自動化し、発注は人間が確認してから行う
C) 承認不要で完全自動発注
D) ハッカソンスコープ外
X) Other (please describe after [Answer]: tag below)

[Answer]: B
承認後をしたら発注できるようにしてほしい
選定の際は相手の欲しい物リストなどをデータソースにしたい

---

## Question 6
フロントエンド（UI）はどうしますか？

A) WebアプリUI（React / Next.js など）を作る
B) AWS ConsoleやStep Functions GUIで操作・確認できればOK
C) CLIやAPIのみでOK（UIは不要）
X) Other (please describe after [Answer]: tag below)

[Answer]: X
スマホアプリがいいと思います
スマホアプリにするならReactNativeで実装したい

---

## Question 7
「ママ」のパーソナライズ（ユーザーの好みを覚える）はどう実装しますか？

A) ユーザーが事前に好み・プロフィールを手動登録する
B) スマホデータから自動的に学習する
C) ハッカソンではプロンプトへの指示で簡易実装（固定パーソナリティ）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 8
AWSリージョンはどこを使いますか？

A) ap-northeast-1（東京）
B) us-east-1（バージニア）
C) その他（Bedrockの利用可能リージョンに合わせる）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 9
インフラのデプロイ方法はどうしますか？

A) AWS CDK（TypeScript）
B) AWS SAM
C) Terraform
D) マネジメントコンソールから手動
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 10
バックエンドの実装言語はどれにしますか？

A) Python（Lambda等で最もシンプル）
B) TypeScript / Node.js
C) Java
X) Other (please describe after [Answer]: tag below)

[Answer]: B
バックエンドのフレームワークとしてはHonoを使って欲しい

---

## Question 11
起床サポート（アラーム）機能の扱いはどうしますか？

A) 指定時刻に通知を送るだけ（シンプルなリマインダー）
B) カレンダーの予定を見て、逆算して起こしてくれる
C) ハッカソンスコープ外
X) Other (please describe after [Answer]: tag below)

[Answer]: X
スマホのアラームと同期したい
---

## Question 12
スマート家電連携（家事サポート）の実装範囲はどうしますか？

A) 既存のスマート家電サービス（Nature Remo等）のAPIと連携する
B) モック実装（実際のAPI連携は不要、動作確認のみ）
C) 対応家電の種類を絞って実装（洗濯機・掃除ロボットのみなど）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question: Security Extensions
このプロジェクトにセキュリティ拡張ルールを適用しますか？

A) Yes — すべてのSECURITYルールをブロッキング制約として適用（本番グレード推奨）
B) No — SECURITYルールをスキップ（PoC・プロトタイプ・実験的プロジェクト向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question: Property-Based Testing Extension
プロパティベーステスト（PBT）ルールを適用しますか？

A) Yes — すべてのPBTルールをブロッキング制約として適用
B) Partial — 純粋関数とシリアライゼーションのラウンドトリップのみに適用
C) No — PBTルールをスキップ（シンプルなCRUD・UIのみのプロジェクト向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 今後の展望（ハッカソンスコープ外）

以下は将来的な機能アイデアとして記録しておく。ハッカソンでは実装せず、ドキュメントへの記載のみ。

- **人間味・失敗**: たまに微妙な服を提案したり、ゲームの旧バージョンを買ってきたり、起こすのを諦めたりする「完璧ではないお母さん感」
