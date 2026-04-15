# プロジェクト共通ルール

## バックテストコード作成時の必須制約

Pythonバックテストコードを作成するときは、以下の制約をすべて守ること。

### 定数定義（コード冒頭に必ず記載）

```python
SLIPPAGE_BPS = 5        # 購入スリッページ（5bps = 0.05%）
TAX_RATE     = 0.20315  # 譲渡益税率（特定口座・源泉徴収あり）
```

### 手数料計算ルール

- 手数料は1日の約定合計金額（daily_turnover）で判定する
- エントリー・イグジット両方で手数料を計上する
- daily_turnover はその日のエントリー＋イグジット約定額の累計とする
- 当日の手数料増分 = `calc_fee(累計後) - calc_fee(累計前)` として計上する

```python
def calc_fee(daily_turnover):
    for threshold, fee in FEE_TABLE:
        if daily_turnover <= threshold:
            return fee
    over = daily_turnover - 3_000_000
    extra = ceil(over / 1_000_000) * FEE_OVER_3M_PER_1M
    return FEE_OVER_3M_BASE + extra
```

### 先読みバイアス防止

- シグナルは必ず `.shift(1)` して翌日に反映すること
- 当日の close 確定後にシグナル生成 → 翌日の open でエントリー
- インジケーター計算に当日未来データを使わないこと

### エントリー処理

```python
actual_entry_price = open_price * (1 + SLIPPAGE_BPS / 10000)
shares             = floor(cash / actual_entry_price)
entry_amount       = shares * actual_entry_price
slippage_amount    = (actual_entry_price - open_price) * shares
entry_fee          = calc_fee(daily_turnover + entry_amount) - calc_fee(daily_turnover)
daily_turnover    += entry_amount
cash              -= (entry_amount + entry_fee)
```

- `shares` が 0、または `cash < entry_amount` の場合はエントリーしないこと

### イグジット処理

```python
exit_amount  = shares * exit_price
exit_fee     = calc_fee(daily_turnover + exit_amount) - calc_fee(daily_turnover)
daily_turnover += exit_amount
profit       = exit_amount - entry_amount - entry_fee - exit_fee
tax          = max(0, profit * TAX_RATE)
cash        += exit_amount - exit_fee - tax
```

### トレードログ（DataFrame で記録）

各トレードを以下の列で記録すること:

```
date, ticker, action, open_price, actual_price, shares,
entry_amount, exit_amount, slippage_amount,
entry_fee, exit_fee, profit_before_tax, tax, net_profit,
cash_before, cash_after
```

### バックテスト終了後の出力

以下をすべて表示すること:

1. トレードログ先頭10件
2. コスト集計サマリー（総手数料 / 総税額 / 総スリッページ / コスト控除前後損益の比較）
3. サニティチェック（cash マイナス / 購入金額が cash_before 超 / shares が 0 のトレードを表示）

---

## バックテスト レビュー観点

バックテスト結果をレビューするとき、以下の観点をすべて確認すること。問題が見つかった場合は必ずユーザーに報告する。

### 1. 先読みバイアス（Look-ahead Bias）
- [ ] シグナル生成に当日の close/high/low を使っていないか（翌日 open 執行なら当日 close までが上限）
- [ ] MA・ボリンジャー等のインジケーターが未来データを参照していないか（`rolling` の `min_periods` 確認）
- [ ] Volume Profile・POC の計算がエントリー時点以前のデータのみ使用しているか（`df.iloc[:i+1]`）
- [ ] 週足・月足 MA のリサンプル時に `ffill` で未来の値が混入していないか

### 2. サバイバーシップバイアス
- [ ] 検証銘柄が「現在も上場している勝ち組」に偏っていないか
- [ ] 上場廃止・合併・分割した銘柄を意図的に除外していないか

### 3. 過学習・カーブフィッティング
- [ ] パラメータ最適化をした期間と検証期間が同一になっていないか
- [ ] パラメータ数に対してトレード数が少なすぎないか（目安: トレード数 ≥ パラメータ数 × 10）
- [ ] 最適化後に Out-of-Sample（未見期間）で再検証しているか
- [ ] 特定の銘柄・期間だけ突出して良い結果になっていないか

### 4. コスト・執行の現実性
- [ ] スリッページ（SLIPPAGE_BPS）が計上されているか
- [ ] 手数料（calc_fee）がエントリー・イグジット両方で計上されているか
- [ ] 譲渡益税（TAX_RATE = 0.20315）が控除されているか
- [ ] エントリーは翌日 open で執行しているか（当日 close 執行は先読みバイアス）
- [ ] 単元未満株（S株）の約定タイミング制約を考慮しているか

### 5. 統計的信頼性
- [ ] トレード数が十分か（最低 30 件以上を目安、少ないほど偶然性が高い）
- [ ] 勝率・損益が特定の数件のトレードに依存していないか
- [ ] 最大ドローダウン（MDD）が許容範囲内か（目安: -20% 以内）
- [ ] プロフィットファクター（PF）が 1.5 以上か
- [ ] 勝ちトレードの平均損益 ÷ 負けトレードの平均損益（損益比）が 1.5 以上か

### 6. パラメータの堅牢性
- [ ] 最適値の前後（±1段階）でも結果が大きく崩れないか
- [ ] 銘柄を変えても同様の結果が出るか（銘柄依存になっていないか）
- [ ] 相場環境（上昇・下落・横ばい）ごとに分けて結果を確認したか

### 7. リスク管理
- [ ] ハードストップ（stop_loss_pct）が設定されているか
- [ ] 1トレードの最大損失が初期資金の 5% 以内に収まっているか
- [ ] 連続損失（最大連敗数）が許容できる範囲か
- [ ] MDD の計算が「初期資本ベース」で正規化されているか（累積損益ベースは過小評価になる）

### 8. 指標計算の正確性
- [ ] Total Return = 総損益 ÷ 初期資本（position_size）で計算されているか
- [ ] CAGR の期間が「最初のエントリー〜最後のエグジット」になっているか（データ全期間は過小評価）
- [ ] Sharpe Ratio の年率換算係数が保有日数ベースになっているか
- [ ] MDD が equity（初期資本 + 累積損益）の peak からの下落率で計算されているか
