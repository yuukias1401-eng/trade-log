# trade-log プロジェクトメモ

## 概要
GitHub Pages でホストしている2つの単一ファイル Web アプリ。
- `index.html` … トレードログ（スイングトレード用ウォッチリスト・スマホ向け）
- `dashboard.html` … 資産ダッシュボード（ポートフォリオ）
- 公開URL: https://yuukias1401-eng.github.io/trade-log/
- リポジトリ: https://github.com/yuukias1401-eng/trade-log
- データは Firebase Firestore（クラウド）に保存。HTML は画面と仕組みのみ。
- 認証は Google ログイン（両アプリ共通）。
- 同一オリジンの localStorage キー: `portfolio_v6` / `portfolio_v6_targets` / `portfolio_v6_fx`
  （ただし index.html は待機資金を Firestore `users/{uid}/holdings` + meta から直接購読する方式に変更済み）

## 開発フロー
PCで編集 → `git push` → GitHub Pages が反映（1〜2分）→ スマホで閲覧。
push しないとスマホに反映されない。

---

## やり残しタスク（後日実装したい）

### 1.【相談済み・設計確定】売り時サイン表示 ＋ 決済理由の選択式
ユーザーは「売るルールが曖昧で売り時に迷う」課題あり。買いの5条件チェックリストに対し、
売りの型が無いのが原因。以下2つを実装予定（ユーザー合意済み）。

**① 売り時サイン（各カードに常時表示・ゲージのすぐ上）**
価格・目標(target)・損切(loss)から自動判定するバッジ。優先順位：
| 状態 | 表示 | 色 |
|---|---|---|
| 現値 ≤ 損切 | 🚨 損切ライン割れ → ルール通り売り | 赤 |
| 現値 ≥ 目標 | 💰 利確圏 → 売り or 引き上げ判断 | 緑 |
| 達成度80%以上 | 🎯 利確接近（目標まで +X%） | 黄緑 |
| 含み益（買値超え） | 📈 保有継続（目標まで +X%） | 青 |
| 買値付近〜含み損 | 👀 様子見（損切まで −Y%） | グレー |
※ 既存の alertHTML（🔔目標到達 / 🚨損切割れ）はこのバッジに統合し二重表示を避ける。

**③ 決済時に理由を選ぶ（settleModal の自由入力 s-reason を選択式に変更）**
カテゴリ6つのワンタップボタン：
`💰利確(目標到達) / 🔒トレール(高値から下落) / 🚨損切(ルール通り) / 💥前提崩壊(決算・材料) / ⏳時間切れ(塩漬け) / ✏️その他`
- カードの売りサインに応じて初期選択を自動ハイライト（例: 目標超なら「利確」を既定選択）
- 選んだカテゴリは履歴(sellReason)に残す → 将来「どの理由で売ることが多いか」を統計で振り返れる

関連コード位置（index.html）:
- 決済モーダル HTML: `<div class="modal" id="settleModal">`（s-date / s-price / s-reason / s-memo）
- `window.openSettleModal` / `window.submitSettle`（sellReason は `submitSettle` で history に保存）
- カードの render() 内ゲージ直前に alertHTML を生成している箇所にサインを追加

### 2.【再挑戦したい】チャート表示の改善（特に日本株）
現状: index.html の「📊 チャートを表示」ボタンで市場別に出し分け。
- 米国株/香港株 → TradingView widgetembed の iframe 埋め込み（動作OK）
- 日本株(東証) → TradingView 無料埋め込みは JPX データライセンスで不可
  （「このシンボルはTradingView上でのみ利用可能です」と出て AAPL にフォールバックしてしまう）
  → 現状は Yahoo!ファイナンス / 株探 / TradingView本体 への外部リンクボタンで代替中。
調査済みでダメだった代替: Stooq（日本株チャート画像はデータなし・APIキー必須化）、株探(403)。
今後試す候補: 自前でローソク足を描く（価格データの取得元が課題）、investing.com 等の別ウィジェット、
有料 API。要相談。
関連コード: `function chartMarket(code, currency)` / `window.toggleChart`

---

## 注意点
- FleetView(このセッション)とターミナル版 Claude が同じファイルを触るとコンフリクトの恐れ。
  役割分担 or 「片方が push → もう片方が pull」の順守を推奨。作業前に必ず `git pull`。
- 過去に git push が IPv6(NAT64 `64:ff9b::`) 経路で60秒ハング→リセットする事象あり（IPv4は通る）。
  再発時は管理者で hosts に `20.27.177.113 github.com` を追記して IPv4 固定で回避。
- git の TLS バックエンドは schannel → openssl にグローバル設定変更済み（無害）。
