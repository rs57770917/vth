**Streamlit ダッシュボード仕様書**  
（Turso + Streamlit Cloud / 実績DB ⇔ バックテストDB 切替対応版）

### 1. 目的
- Tursoに蓄積されたデータを読み取り、インタラクティブなダッシュボードを提供する。
- **実績DB**（実取引結果）と**バックテストDB**（バックテスト結果＋条件）を切り替え可能にする。
- バックテスト側では「複数の条件セット」を切り替えられるようにする。
- ローソク足＋取引マーカーを正確に表示し、表形式の詳細も確認できるようにする。
- VPS側には表示系を一切置かない前提を維持する。

### 2. 全体構成（Streamlit側）
```
Streamlit Community Cloud
├── Secrets（接続情報）
├── サイドバー（データソース切替・条件選択・フィルタ）
└── メインエリア
    ├── サマリーカード
    ├── ローソク足チャート（Plotly）
    ├── エクイティ曲線
    ├── 取引一覧テーブル
    └── 条件詳細パネル（バックテスト時のみ）
```

### 3. データソース設計

#### 3.1 接続の切替
- **実績DB** と **バックテストDB** を2つの独立したTursoデータベースとして扱う（推奨）。
  - 理由: データ混在を防ぎ、権限管理・バックアップが容易。
- Streamlit Secrets に両方の接続情報を保持する。

```toml
# Streamlit Secrets 例
[connections.live]
url = "libsql://live-xxxx.turso.io"
token = "eyJ..."

[connections.backtest]
url = "libsql://backtest-xxxx.turso.io"
token = "eyJ..."
```

- アプリ起動時またはサイドバー操作で接続を切り替える。
- 接続オブジェクトは `@st.cache_resource` で保持し、切替時に再生成する。

#### 3.2 バックテストDBのデータモデル（必須）
バックテスト側は「条件」と「その条件で発生した取引」を分離する。

**`backtest_conditions`（条件マスター）**
```sql
CREATE TABLE IF NOT EXISTS backtest_conditions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,                  -- 表示名（例: "RSI14_overbought_70"）
    description TEXT,
    params TEXT,                         -- JSON文字列でパラメータを保存
    symbol TEXT,                         -- 対象通貨ペア
    timeframe TEXT,
    start_date TEXT,
    end_date TEXT,
    created_at TEXT DEFAULT (datetime('now'))
);
```

**`backtest_trades`（バックテスト取引）**
```sql
CREATE TABLE IF NOT EXISTS backtest_trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    condition_id INTEGER NOT NULL,       -- backtest_conditions.id への参照
    ticket TEXT,
    symbol TEXT NOT NULL,
    side TEXT NOT NULL,                   -- BUY / SELL
    open_time TEXT NOT NULL,
    close_time TEXT,
    open_price REAL NOT NULL,
    close_price REAL,
    lots REAL,
    profit REAL,
    comment TEXT,
    FOREIGN KEY (condition_id) REFERENCES backtest_conditions(id)
);
```

**`ohlc`** は実績・バックテスト共通で持つか、バックテスト側にも同様のテーブルを置く（推奨は共通化または両方に持つ）。

実績DB側は前回仕様の `trades` / `ohlc` / `equity_snapshots` をそのまま使用。

### 4. UI/UX仕様

#### 4.1 サイドバー（必須要素）
1. **データソース選択**
   - ラジオボタン or セレクトボックス  
     `実績` / `バックテスト`
2. **バックテスト条件選択**（データソースがバックテストのときのみ表示）
   - `backtest_conditions` から取得した一覧をセレクトボックスで表示
   - 選択した `condition_id` で以降のデータをフィルタ
3. **共通フィルタ**
   - 通貨ペア（symbol）
   - 期間（開始日〜終了日）
   - タイムフレーム（ローソク足用）
   - 売買方向（BUY / SELL / 両方）
4. **表示オプション**
   - ローソク足に取引マーカーを重ねる ON/OFF
   - エクイティ曲線表示 ON/OFF

#### 4.2 メインエリア
- **ヘッダー**: 現在選択中のデータソース名 ＋ 条件名（バックテスト時）を明示
- **サマリーカード**（横並び）
  - 総損益 / 勝率 / 取引回数 / 平均利益・平均損失 / 最大ドローダウン など
- **ローソク足チャート**（Plotly `go.Candlestick`）
  - 選択した symbol + timeframe のOHLCを表示
  - 取引のエントリー・決済位置にマーカーをプロット（利益=緑、損失=赤）
  - ズーム・ホバー対応
- **エクイティ曲線**（任意表示）
- **取引一覧テーブル**
  - ソート・検索可能な `st.dataframe`
  - バックテスト時は選択中の条件に紐づく取引のみ表示
- **条件詳細パネル**（バックテスト時のみ）
  - 選択中条件の `params`（JSON）を見やすく表示
  - 期間・対象シンボルなども併記

### 5. 機能要件詳細

| 機能 | 内容 |
|------|------|
| DB切替 | 実績 ⇔ バックテストを即時切替。切替時にキャッシュを適切にクリアまたは再取得 |
| 条件切替 | バックテスト内で複数条件をセレクトで切替。切替後にチャート・テーブル・サマリーを再描画 |
| データ取得 | 選択中の接続情報を使ってTursoからクエリ。`libsql` クライアントを使用 |
| キャッシュ | `@st.cache_data(ttl=60〜300)` でクエリ結果をキャッシュ。DB/条件変更時はキーを変えて再取得 |
| エラーハンドリング | 接続失敗・データなしの場合に分かりやすいメッセージを表示 |
| レスポンシブ | ワイドレイアウト推奨（`st.set_page_config(layout="wide")`） |

### 6. 技術実装方針

- **接続管理**
  ```python
  def get_connection(source: str):
      if source == "live":
          conf = st.secrets["connections"]["live"]
      else:
          conf = st.secrets["connections"]["backtest"]
      return libsql.connect(database=conf["url"], auth_token=conf["token"])
  ```

- **条件一覧取得**（バックテスト時）
  - `SELECT id, name, description FROM backtest_conditions ORDER BY created_at DESC`

- **取引データ取得**
  - 実績: `trades` テーブルから期間・symbolでフィルタ
  - バックテスト: `backtest_trades` を `condition_id` で絞り込み

- **チャート**
  - Plotlyの `Candlestick` + `Scatter` マーカー
  - 時間軸はUTCで統一し、必要に応じて表示時にJST変換

- **ページ構成**
  - 単一ページで完結を基本とする（サイドバーで全て制御）
  - 将来的に `pages/` で「詳細分析」「条件比較」などを追加可能

### 7. セキュリティ・運用
- Secretsにトークンを保存（Gitにコミットしない）
- 読み取り専用トークンの使用を推奨
- Streamlit Community Cloudのアプリは必要に応じてパスワード保護
- データ量が増えた場合は `ttl` や取得件数制限で負荷を制御

### 8. 今後の拡張ポイント
- 複数条件の同時比較（オーバーレイ表示）
- 条件パラメータの編集・再実行トリガー（高度）
- 実績とバックテストの成績を並べて比較するビュー
- TradingView Lightweight Chartsへの切り替え