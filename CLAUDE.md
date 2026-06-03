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

### watchlistオーファン掃除（任意）
インポートで保有idが変わると、持ち主のいない古いwatchlistカードが残る（ASTS3枚問題）。
対策案: _syncAllShortToWatchlist 実行時に、fromDashboard:true かつ現在の短期保有idに
無いwatchlistドキュメントを削除する（手動追加カード fromDashboard:false は消さない）。
当面は手動で🗑削除で対応可。

---

## ✅ 2026-06 解決済み（直近修正）

### Firestoreデータ消失バグ (commit a5d114a)
- `_fbSaveHoldings` に削除スキップガードを追加:
  - `holdings.length === 0` または `existing.size > 5 && holdings.length < existing.size / 2` のとき削除パスをスキップ + toast警告
  - `force=true` 引数で強制削除可能（クラウド掃除ボタン用）
- NaN/Infinity も除去するよう保存前サニタイズを強化
- `save()` の `.catch(console.warn)` を `.catch(console.error)` に変更
- ヘッダーに「🧹 クラウド掃除」ボタンを追加（既存55件等のゴミdocを一掃できる）

### シンボル取り違えバグ (commit 2811878)
- `fetchCandles` / `fetchYahooPrice` を通貨で単一シンボルに確定
  - HKD → `.HK` のみ / それ以外 → `.T` のみ（競争させない）

### 評価損益金額表示 (commit 02ede7e)
- index.html カードに「評価損益 ¥XXX」を追加
  - qty × (現在値 - 買値) で算出、外貨は portfolio_v6_fx で円換算
  - プラス=緑 / マイナス=赤、qtyなし銘柄は非表示

---

## 🔴 未解決バグ（最優先・調査中）：インポート保有がFirestoreで増殖＆消失

### 症状
- ダッシュボードでSBI外国株を再インポート → 直後はメモリ上に正しく入る（特定30株＋NISA1株）
- タブを閉じて開き直すと、新規分（特定ASTS 30株）が消える
- トレードログ側ではASTSのカードが再インポートのたびに増殖（オーファン蓄積）

### 2026-06 の診断結果（重要な手がかり）
インポート確定時の検証トースト（executeImport末尾, ≈3050行）が
**「☁️Firestore 約55件 / メモリ 約1件」**と表示（ユーザー記憶ベース・要再確認）。
→ Firestoreの users/{uid}/holdings に**実際の保有数(約25)を遥かに超える約55ドキュメントが蓄積**。
→ さらに「メモリ ≈1件」が異常に少ない。保存時に holdings 配列が極小になっている疑い。
   もしそうなら _fbSaveHoldings の削除パス(ids に無いdocを削除)が大量削除/誤動作する温床。

### 既に試した対策（commit / いずれも未解決）
- 1695c1e: インポート重複判定キーに口座(特定/NISA)追加
- b4a900d: executeImportでFirestore保存をawait
- ead27e9: watchlist同期に account(特定/NISA) 付与
- 9e3aea7: _fbSaveHoldings の undefined除去・失敗を握り潰さずthrow＋トースト・_fbInit初回ロードガード
- d6a441a: インポート保存後にFirestoreを読み戻して件数を表示する診断トースト

### 次にやること（優先順）
1. **トースト数値を正確に再確認**：インポート直後の「☁️Firestore N件 / メモリ M件」を正確にメモ。
   さらに「閉じて開き直した直後」のholdings件数も確認。
2. **Firebaseコンソールで users/{uid}/holdings を直接確認**：ドキュメント数、ASTSの重複有無、id体系。
   約55件もあるならゴミが大量 → 一度コレクションを全削除して再インポートのクリーン再構築を検討。
3. **save() 呼び出し時に holdings がサブセットになっていないか全箇所点検**：
   silentFetchShortPrices / fetchPricesBatch / cycleHorizon / その他で holdings を
   フィルタ結果で上書きしていないか。「メモリ1件」が出る経路を特定する。
4. **_fbSaveHoldings の削除パスが危険**：ids に無いdocを消す設計。holdingsが一時的に小さい時に
   走ると大量削除する。削除を保守的にする（fromImport直後は削除しない/全削除ガード）か、
   holdingsが空/極小なら保存自体を中断するガードを入れる。
5. オーファン一掃（watchlist側も）＋ id を「ticker+口座」など安定キーにすることも検討。

### ターミナル版の分析（A/B/C・有力な残バグ候補）
前回修正(9e3aea7/d6a441a)で「undefinedでFirestore保存が無言失敗」は解消済み。残る怪しい点：
- **A. save() の失敗握り潰し**：save()内 `_fbSaveHoldings().catch(console.warn)` のため、
  executeImport以外の経路（銘柄編集・削除・タグ変更など）でFirestore保存が失敗しても画面に出ない。
  → これらの経路も失敗をユーザーに通知する／少なくともconsole.errorにする。
- **B. NaN がundefinedチェックを通過**：parseFloatがNaNを返した古いデータがあると、
  undefined除去では弾けず保存失敗の可能性が残る。→ 保存前サニタイズでNaN/Infinityも除去or0化する。
- **C.（最有力）holdingsが空/極小のとき _fbSaveHoldings が呼ばれると、削除パスで
  Firestoreのholdingsコレクションを全件削除してしまう**。今回の「☁️55件/メモリ1件」と整合。
  → _fbSaveHoldings冒頭に「holdingsが空(または直前より極端に減少)なら削除パスをスキップ/中断」ガードを入れる。
     これが30株消失＆カード増殖の主犯の可能性大。最優先で対処。

### 関連コード（dashboard.html）
- executeImport ≈3010-3060 / _fbSaveHoldings ≈718-745 / _fbInit ≈791-826
- save() ≈1114 / load() ≈1121 / startAlertPolling ≈3238 / fetchPricesBatch・silentFetchShortPrices

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
- ✅ Firestoreデータ消失バグ修正（commit a5d114a）… ガード追加・NaN除去・掃除ボタン
- ✅ シンボル取り違えバグ修正（commit 2811878）… fetchCandles/fetchYahooPrice を単一シンボル化
- ✅ 評価損益金額表示（commit 02ede7e）… index.htmlカードに ¥XXX 追加
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
