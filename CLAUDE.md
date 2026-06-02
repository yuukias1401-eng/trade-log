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

## 後日やりたい（未着手）

### 儲け額（金額）表示
トレードログのカードは現状％だけで、いくら儲かっているか「金額」が無い。
儲け額 = (現在値 − 買値) × 株数。watchlistドキュメントに buy/current/qty があるので算出可。
- カードに「評価損益 ¥XXX」を表示（プラス緑/マイナス赤）。qtyが無い銘柄は非表示。
- 外貨建ては円換算も出せると尚良い（fxは dashboard 側 portfolio_v6_fx 参照）。

### watchlistオーファン掃除（任意）
インポートで保有idが変わると、持ち主のいない古いwatchlistカードが残る（ASTS3枚問題）。
対策案: _syncAllShortToWatchlist 実行時に、fromDashboard:true かつ現在の短期保有idに
無いwatchlistドキュメントを削除する（手動追加カード fromDashboard:false は消さない）。
当面は手動で🗑削除で対応可。

---

## 🐛 既知のバグ（要修正・優先）

### A. チャート/株価が「別の銘柄」を表示する（シンボル取り違え）
症状: サムコ(3436)のチャートで、足を切り替えると一部の足が別会社のチャートになる。
     株価取得でも昔から「別の株を読み込む」現象あり。
原因: index.html の fetchCandles(≈1285-1320行) と fetchYahooPrice(≈680-692行) が、
     4〜5桁コードの時に `code+'.T'`(東証) と `code+'.HK'`(香港) の両シンボルを
     **同時にPromise.anyで投げて、最速で返った方を採用**している。
     3436.T=サムコ だが 3436.HK=全くの別会社。プロキシの速さ次第で別銘柄を拾う。
     足切替ごとにfetchするため「ある足はサムコ/別の足は別会社」の混在が起きる。
修正方針: 通貨で取引所を確定し**単一シンボルのみ**使う。
     - currency が 'HKD' → `code+'.HK'` のみ
     - それ以外(JPY等) → `code+'.T'` のみ
     - 既に `.` 付き(AAPL/0700.HK等)はそのまま
     複数シンボルを競争させない。競争してよいのは同一シンボルの proxy/endpoint だけ。
     fetchCandles と fetchYahooPrice の両方を同様に直すこと。
     ※ chartMarket(code,currency) が既に市場判定を持っているので流用可。

---

## 完了済み（2026-06 実装）
- ✅ 売り時サインバッジ＋決済理由の選択式（commit 829b4c0）… 下記「1.」の設計で実装済み
- ✅ カード内ローソク足チャート 全市場対応（commit 1b76c03）… 下記「2.」の方針で実装済み。
     Yahoo chart API（query1/2 finance/chart, interval=1d&range=1y）を既存PROXIESで取得し、
     TradingView Lightweight Charts(v4 オープンソース版)でローソク足を自前描画。
     買値/目標/損切ラインを createPriceLine で重ね表示。取得失敗時は外部リンクにフォールバック。
     ※「埋め込みウィジェット」ではなく「データ取得→自前描画」だから日本株もOK。

---

## 過去の検討記録（実装済み・参考用）

### 1.【実装済み】売り時サイン表示 ＋ 決済理由の選択式
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
