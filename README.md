# daily_stock_analysis_jp

[ZhuLinsen/daily_stock_analysis](https://github.com/ZhuLinsen/daily_stock_analysis)（MIT License）を、日本株の私用チェックに使うための作業リポジトリ。

本体は上流の実装をそのまま使う（分析エンジン・バックエンドは Python + FastAPI、Web工作台のフロントは React + Vite + TypeScript）。このリポジトリには、まず日本株向けの設定ファイルと調査メモを置く。

## 状態

- 2026-11-14: 調査完了。詳細は [docs/investigation-2026-11-14.md](docs/investigation-2026-11-14.md)（上流コミット `ac9c33d` 時点）
- 日本株は `7203.T` 形式で正式対応していることを、コード・テスト・実地の取得確認で裏取り済み
- 次のステップ: APIキーを `.env` に入れて、1銘柄でスモークテスト（`--dry-run` → 本分析）

## セットアップ（予定）

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt   # 上流の requirements.txt を取り込んでから
cp .env.example .env              # 使うLLMのキー等を記入（キーは各自で用意）
python main.py --dry-run --stocks 7203.T        # データ取得だけ確認（キー不要）
python main.py --stocks 7203.T --no-notify      # LLMを入れて本分析
```

## 注意（調査で判明した制約）

- **銘柄コードは必ず `.T` 付き**（例: `7203.T`）。裸コードは A股と誤判定される
- **TDnet適時開示は未対応**。既存のTDnet監視の代替にはならない。「引け後のまとめ読み」専用
- レポート言語は zh / en / ko のみで、日本語設定はない（当面 `REPORT_LANGUAGE=en`）
- 日本株の相場は yfinance（Yahoo Finance）経由のみ。財務項目は欠けることがある
- `.env` は絶対にコミットしない（`.gitignore` 済み）
