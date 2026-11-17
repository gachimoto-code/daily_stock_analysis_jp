# daily_stock_analysis_jp

[ZhuLinsen/daily_stock_analysis](https://github.com/ZhuLinsen/daily_stock_analysis)（MIT License）を、日本株の私用チェックに使うための作業リポジトリ。

本体は上流の実装をそのまま使う（分析エンジン・バックエンドは Python + FastAPI、Web工作台のフロントは React + Vite + TypeScript）。日本株向けの設定と、日本株のニュース検索を日本語化する改修を加えている。

## 状態

- 2026-11-14: 調査完了。詳細は [docs/investigation-2026-11-14.md](docs/investigation-2026-11-14.md)（上流コミット `ac9c33d` 時点）
- 日本株は `7203.T` 形式で正式対応していることを、コード・テスト・実地の取得確認で裏取り済み
- 2026-11-17: 本分析の実走まで完了（トヨタ 7203.T で複数モデルを比較）。検索API（Brave）のキーを入れたことでニュース取得も動作
- 2026-11-17: ニュース検索を日本語化（下記「日本株向けの改修」）

## 日本株向けの改修（このフォークの差分）

- **ニュース検索の日本語化**（`src/search_service.py`、`src/data/stock_mapping.py`）
  - 日本株（`.T`）は、日本語社名＋証券コード＋日本語キーワードで検索クエリを構築する。日本語社名は `STOCK_JP_NAME_MAP`（7203 トヨタ自動車、6758 ソニーグループ、9984 ソフトバンクグループ、8306 三菱UFJフィナンシャル・グループ）が真源で、未登録銘柄は表示名＋日本語キーワードにフォールバックする
  - Brave検索には `search_lang=ja` / `country=JP` を渡し、結果はカナ判定で日本語記事を優先表示する
  - イベント検索・多角的情報検索にも日本語分岐を追加（決算・業績予想・適時開示などの日本語クエリ）
  - 改修前は英語社名＋中国語キーワードのクエリで、海外の二次ソースしか拾えなかった

## セットアップ（実施済みの内容）

実行環境は Ubuntu 24.04 / Python 3.12.3。以下の手順で構築し、実走まで確認済み。

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env              # 下記のキーを記入（キーの実値はコミットしない）
```

`.env` の設定（2026-11-17 時点）:

- LLM: `OPENAI_API_KEY` と `ANTHROPIC_API_KEY` を設定し、`LITELLM_MODEL=openai/gpt-6.1-sol` を既定にしている（Claude 系モデルとも差し替え可能）
- ニュース検索: `BRAVE_API_KEYS` を設定。キーが無い場合、分析は動くがニュース欄は空になる
- 対象銘柄: `STOCK_LIST=7203.T,6758.T,9984.T,8306.T`
- レポート: `REPORT_LANGUAGE=en`、`REPORT_TYPE=simple`
- 大盤レビュー: `MARKET_REVIEW_ENABLED=true`、`MARKET_REVIEW_REGION=jp`

実行:

```bash
python main.py --dry-run --stocks 7203.T        # データ取得だけ確認（キー不要）
python main.py --stocks 7203.T --no-notify      # LLMを入れて本分析
```

注意（この環境固有）: プロキシ環境変数 `NO_PROXY` / `no_proxy` に括弧付きIPv6エントリが含まれていると HTTP クライアント（httpx）が起動時に落ちるため、実行時は `NO_PROXY=localhost,127.0.0.1 no_proxy=localhost,127.0.0.1` のように整理して起動する。

## 注意（調査で判明した制約）

- **銘柄コードは必ず `.T` 付き**（例: `7203.T`）。裸コードは A股と誤判定される
- **TDnet適時開示は未対応**。既存のTDnet監視の代替にはならない。「引け後のまとめ読み」専用
- レポート言語は zh / en / ko のみで、日本語設定はない（当面 `REPORT_LANGUAGE=en`）
- 日本株の相場は yfinance（Yahoo Finance）経由のみ。財務項目は欠けることがある
- 資金フロー（マネーフロー）は日本株では取得できず、買い判定の確認材料にできない
- `.env` は絶対にコミットしない（`.gitignore` 済み）
