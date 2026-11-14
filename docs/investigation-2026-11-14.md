# daily_stock_analysis（ZhuLinsen）調査メモ ― 日本株の私用ツール化

調査日: 2026-11-14
対象コミット: `ac9c33dfcf4cb84974d7bfee9c8661d88b8cfbff`（2026-11-14 時点の main）
前提: 公開・投資助言用途ではなく、藤本さん本人の保有銘柄チェック用の私用ツールとして使う。

## 結論（先に要点）

- **日本株は「suffix-only MVP」として正式対応している。** `7203.T` 形式で銘柄を渡せば、個股分析・履歴保存・レポート生成・大盤レビュー（日経225/TOPIX）まで既存パイプラインに乗る。コード・テスト・公式ドキュメント（`docs/market-support.md`）の3点で裏取り済み。
- ただし対応は最小構成で、**日本株のデータ源は yfinance（Yahoo Finance）のみ**。リアルタイム性・完全なファンダメンタルズ・全銘柄リストは「不承诺（保証しない）」と明記されている。
- **TDnet適時開示は完全に未対応**（リポジトリ全体を検索して `TDnet` の言及は0件）。日本語ニュース専用の仕組みもなく、ニュースは汎用Web検索API＋英語クエリが基本。藤本さんの既存のTDnet監視（強烈IR監視など）の代替にはならない。「引け後の保有銘柄まとめ読み」用途と割り切るのが正しい。
- **レポート出力言語は zh（中国語）/ en / ko のみで、日本語設定がない。** これが日本人ユーザーにとって最大の実用上の欠落で、設定では回避できない（後述）。
- AI分析には **LLMのAPIキーが最低1つ必須**（またはローカルLLMのOllama）。キーなしで動くのはデータ取得だけの `--dry-run` まで。

---

## 1. ライセンスと利用条件

- 本体コード: **MIT License**（`LICENSE`、Copyright (c) 2026 ZhuLinsen）。私的利用・改変・フォークは自由。
- リポジトリ自身の免責: 「本项目仅供学习和研究使用，不构成任何投资建议」。私用ツール化はこの範囲と整合する。
- データ側の条件は別途かかる:
  - 日本株データは yfinance 経由の Yahoo Finance データ。yfinance は非公式ライブラリで、Yahoo側の利用条件は個人利用前提。大量・高頻度取得はレート制限や遮断の対象になりうる。
  - ニュース検索API（SerpApi、Tavily 等）は各サービスの規約・無料枠に従う。
  - LLM API は各プロバイダの規約に従う（銘柄コード・ニュース・プロンプトが外部送信される）。
- サードパーティ依存の通知は `THIRD_PARTY_NOTICES.md` に整理されている。

## 2. アーキテクチャ概要

- 言語: **Python**（3.10+、GitHub Actions は 3.11 を使用）。LLM接続は **LiteLLM**（`litellm>=1.80.10,<1.99.0`）で統一。
- 主要コンポーネント:
  - `main.py` — CLIエントリ（分析実行、`--stocks`、`--dry-run`、`--schedule`、`--market-review`、`--webui`、`--serve-only` 等）
  - `data_provider/` — 行情フェッチャ群（優先度付きマネージャ `DataFetcherManager` が市場ごとにルーティング）
  - `src/search_service.py` — ニュース検索（複数プロバイダ、関連度スコアリング付き）
  - `src/core/pipeline.py` — 個股分析パイプライン
  - `server.py` / `api/` / `webui.py` — FastAPI + Web工作台（React製 `apps/dsa-web`）。手動分析、履歴レポート、バックテスト、持倉管理、Agent問股（チャット）
  - 保存先: SQLite（`./data/stock_analysis.db`）
  - `strategies/` — Agent問股用の戦略スキル（均線・纏論・波浪など15種、YAML）
  - `bot/` — Discord/Telegram等の対話ボット
- 実行形態: **GitHub Actions（推奨とされる）/ ローカルCLI / Docker / ローカル定時（`--schedule`）/ FastAPI常駐**。デスクトップアプリの打包もあり。
- GitHub Actions の標準ワークフロー `.github/workflows/00-daily-analysis.yml`: 平日 cron `0 10 * * 1-5` UTC = **日本時間19:00**（北京18:00）。東証の引け後なので日本株用途でも時刻は問題ない。デフォルトのタイムアウトは30分（変数で変更可）。

## 3. 対応市場とデータソース ― 日本株の検証結果

### 対応市場（README・コード共通）

A股、港股、米股、**日股**、韓股、台股、ETF。市場判定は `src/services/stock_code_utils.py` の `get_suffix_market()` 等に集約され、`7203.T` → `jp` になることが単体テストで固定されている（`tests/test_market_symbol_utils.py`、`tests/test_stock_code_utils.py`）。

### 日本株の入力形式（重要）

- **必ず Yahoo サフィックス形式 `7203.T`（4〜5桁＋`.T`）で書く。** 裸コードは危険: テスト上、裸の6桁数字は A股として扱われる（`detect_market("005930") == "cn"`）。4桁の日本コードを裸で入れると誤判定・別銘柄化の恐れがあるため、STOCK_LIST は全銘柄 `.T` 付きで統一すること。
- Web補完用の日本株シードは **30銘柄のみ**（`scripts/stock_index_seeds/stock_list_jp.csv`: トヨタ、ソニー、ソフトバンクG、三菱UFJ、キーエンス、日立、任天堂 等）。シード外の銘柄も `.T` 形式で手入力すれば分析対象になる（名称解決は yfinance 側の英語名に寄る）。

### 日本株のデータ経路（`docs/market-support.md` と `tests/test_jp_kr_market_support.py` で確認）

- 日線・基礎的なリアルタイム/準リアルタイム相場は **`YfinanceFetcher` のみ**。AkShare、Tushare、Efinance、Pytdx、Baostock 等の中国株専用源は日本株では一切呼ばれない（マネージャのルーティングをテストで検証済み）。
- ファンダメンタルズは yfinance の軽量経路（`data_provider/yfinance_fundamental_adapter.py`）を再利用。PERや出来高金額など欠ける項目があり、`data_quality=partial` / `missing_fields` として正直にメタデータ化される。
- A股専用概念（資金フロー、龍虎榜、板块、筹码分布）は日本株では `not_supported` に降級。レポート用プロンプトにも日本株専用ガイドラインがあり、A股の涨跌停・北向資金等の概念を套用しないよう明示されている（`src/market_context.py` の `get_market_guidelines`、テストで検証済み）。
- 取引カレンダー: `jp = XTKS / Asia/Tokyo` を登録済み。昼休み、15:25–15:30 の引け競売まで市場フェーズを判定する。取引日チェックも日本カレンダーで効く。
- 大盤レビュー: `MARKET_REVIEW_REGION=jp` で日経225（`^N225`）とTOPIX（`^TOPX`）のレビューが可能。ただし v1 は「主要指数＋ニュース＋LLMレビュー」のみで、**市場の広がり（騰落銘柄数）や業種別ランキングはなし**。
- Portfolio（持倉）: JP口座・保有スナップショットは通るが `data_quality=partial` で、JPYの為替・コスト・時価総額の完全な口径は未対応。Market Light アラートは cn/hk/us のみで **jp は対象外**。

### データ源一覧（市場別）

| データ源 | 対象 | キー |
|---|---|---|
| YFinance | **日股を含む全市場**（日股はこれのみ） | 不要 |
| AkShare / Baostock / Pytdx / Efinance | A股 | 不要 |
| TickPlus / TickFlow | A股・ETF | 任意（有料寄り） |
| Tushare | A股 | 任意（トークン） |
| Longbridge | 港股・米股 | 任意 |
| Finnhub / AlphaVantage | 米股 | 任意 |

## 4. 必要なLLMとAPIキー

### LLM（AI分析に最低1つ必須）

接続は LiteLLM 経由。`.env.example` と Actions Secrets の定義で確認した選択肢:

- **アグリゲータ（プロジェクト推奨）**: Modelflare（`MODELFLARE_KEY`）、AIHubMix（`AIHUBMIX_KEY`）— 1キーで GPT/Claude/Gemini 等を切替。
- **直接API**: Gemini（`GEMINI_API_KEY`、無料枠あり）、OpenAI互換（`OPENAI_API_KEY` ＋ `OPENAI_BASE_URL`/`OPENAI_MODEL`）、DeepSeek（`DEEPSEEK_API_KEY`、低価格）、Anthropic（`ANTHROPIC_API_KEY`）、Moonshot/Kimi、通义千问（DashScope）、智譜GLM、MiniMax、火山方舟、SiliconFlow、OpenRouter 等。中国系モデルが厚い。
- **キー不要の代替**: Ollama ローカルモデル（`OLLAMA_API_BASE`、例 `ollama/qwen3:8b`）。ただし GitHub Actions では使えず、ローカル/Docker 実行専用。
- 実験的opt-inとして、ローカルCLIバックエンド（`codex_cli` / `claude_code_cli` / `opencode_cli`）も選べる（`GENERATION_BACKEND`）。ログイン済みCLIを使う形で、APIキーの新規発行なしに試せる可能性があるが「experimental/limited」扱い。
- 画像からの銘柄取込（Vision）を使う場合のみ、Vision対応モデルのキーが別途必要。私用では不要。
- **キーなしでできるのは `--dry-run`（データ取得のみ、AI分析なし）まで。** レポート生成には必ずどれか1つのLLM経路が要る。

### ニュース検索キー（任意だが実質推奨）

SerpApi（推奨）、Tavily、Brave、Bocha、MiniMax、Anspire、SearXNG（自建ならキー不要）。**どれも未設定だとニュース検索プロバイダが0になり、ニュース面は空になる**（SearXNG公共インスタンスの自動利用はレート制限のためデフォルト無効）。最低1つは入れたい。

### その他

- 通知先の設定（Webhook URL や SMTP パスワード等）が通知を使う場合に必要（第5節）。
- 行情の有料キー（Tushare 等）は **日本株には効かない**ので不要。

## 5. 出力形態

- **決策ダッシュボード（Markdownレポート）**: 銘柄ごとに結論・スコア・トレンド・売買ポイント・リスク警告・催化要因・チェックリスト。大盤レビュー（指数・市場概況）も別レポートで出る。レポート種別は `REPORT_TYPE=simple/full/brief`。
- **レポート言語: `REPORT_LANGUAGE` は `zh`（デフォルト）/ `en` / `ko` の3択のみ。日本語はない。**
- **通知チャネル（複数同時可）**: 企業微信、飛書、釘釘、Telegram、Discord、Slack、メール（SMTP）、カスタムWebhook（Bark等も可）、Pushover、ntfy、Gotify、PushPlus、Server酱、AstrBot。LINE は無い。日本で現実的なのは **メール / Telegram / Discord / カスタムWebhook** あたり。
- **Web工作台**: 手動分析、タスク進捗、過去レポートの閲覧、バックテスト（過去の推奨の事後検証）、持倉管理、Agent問股チャット。`http://127.0.0.1:8000`。
- **定時実行**: GitHub Actions（平日19:00 JST）、ローカル `--schedule`（`SCHEDULE_TIME`、デフォルト18:00）、Docker常駐。非取引日はデフォルトでスキップ（強制実行フラグあり）。
- 履歴は SQLite に蓄積され、Web/レポートから再読できる。

## 6. ニュース源 ― 日本語ニュースとTDnet

- **TDnet適時開示: 未対応（断定可）。** リポジトリ全体のコード検索で `TDnet` / `tdnet` / `適時開示` のヒットは **0件**。日本株の「公告」に相当する取得経路も存在しない（A股の公告取得は中国側データ源依存で、日本株ルートには接続されていない）。
- **日本語ニュース専用源: なし。** 組み込みの情報源（NewsNow経由の中国金融メディア等、`docs/intelligence-sources.md`）に日本メディアは含まれない（同ドキュメント内の日本関連キーワード検索も0件）。
- 個股ニュースは汎用Web検索APIで都度検索する方式（`src/search_service.py`）。外国株のクエリは英語で組まれる設計（`"{英語社名} {コード} stock latest news"`）だが、英語別名マップ（`src/data/stock_mapping.py` の `STOCK_ENGLISH_NAME_MAP`）には **米股・港股しか登録がなく、日本株は未登録**。したがって日本株のニュース検索は:
  - シード30銘柄: 表示名が中国語名（例: 丰田汽车）になるため、中国語クエリ寄りになり、日本語ニュースはほぼ拾えない。
  - シード外の銘柄: yfinance 由来の英語社名で検索されれば英語ニュース（Reuters等）は拾える可能性があるが、日本語記事の体系的な取得にはならない。
- つまり **ニュース・開示の鮮度と網羅性は、このツールの日本株対応の最弱点**。藤本さんの既存TDnet監視（30分間隔）が担っている領域は置き換えられない。DSA は「テクニカル＋ファンダ中心の夕方のまとめ読み」専用と位置づけるべき。

## 7. キーなし実地確認（今回実施したもの）

1. **yfinance で `.T` 取得（このサンドボックスから直接実行）**:
   - `7203.T`（トヨタ）: 直近5営業日の OHLCV を取得成功。2026-11-13 終値 3,418円、通貨 JPY。
   - `9984.T`（ソフトバンクG）: 同様に取得成功。2026-11-13 終値 3,547円。
   - → 日本株の日線データ経路そのものはキーなしで実際に動くことを確認済み。
2. **コード読解での裏取り**（推測ではなく実装で確認した点）:
   - 市場判定・正規化: `get_suffix_market("7203.t") == "jp"`（テスト）、`normalize_stock_code("7203.t") == "7203.T"`（テスト）。
   - ルーティング: JP/KR の日線は YfinanceFetcher のみに流れる（テスト `test_data_fetcher_manager_routes_jp_kr_daily_only_to_yfinance`）。
   - 取引カレンダー: `MARKET_EXCHANGE["jp"] == "XTKS"`、`MARKET_TIMEZONE["jp"] == "Asia/Tokyo"`（テスト）。
   - プロンプト: 日本株ガイドラインに「A股の概念を套用しない」旨が含まれる（テスト）。
   - TDnet 言及 0 件（リポジトリ全体検索）。
   - レポート言語の選択肢 zh/en/ko のみ（`.env.example` の定義）。

## 8. 日本株私用化アダプト計画

### 最小スモークテスト（キーなし〜最小キー）

1. **環境**: Python 3.11 の venv を用意（このVMの `/tmp` は 512MB の tmpfs で依存一式が入らない恐れがあるため、ワークスペース側ディスクか Docker を使う）。`pip install -r requirements.txt`。
2. **設定（キーなし版）**: `.env` に `STOCK_LIST=7203.T,6758.T,9984.T,8306.T`、`MARKET_REVIEW_REGION=jp` のみ。
3. **データ経路の確認**: `python main.py --dry-run --stocks 7203.T,6758.T` → ログで YfinanceFetcher 経由の日線取得と SQLite への保存を確認。あわせて `python main.py --market-review` 系で日経225/TOPIX の取得を確認。
4. **LLMを1つだけ入れて本分析**: `python main.py --stocks 7203.T --no-notify` を1銘柄で実行し、レポートが生成されること・日本株ガイドラインが効いていること（A股用語が出ないこと）・言語（zh既定）を確認。ここで品質とコスト感を判定する。
5. **対象を広げる**: 問題なければ保有のうち流動性の高い10〜20銘柄に拡大し、通知（メール等）を接続。定時化はその後。

### 必要な改修・設定上の工夫（想定）

- **銘柄リスト整備**: 楽天の保有CSV（約200銘柄）から `コード.T` 形式の一覧を機械生成する。名前解決はシード外だと英語名頼みになるため、レポートの見栄え用に主要銘柄だけでも名称対応を足す余地あり。
- **ニュースの日本語化（やるなら中規模改修）**: `STOCK_ENGLISH_NAME_MAP` の日本株版を作るか、`src/search_service.py` のクエリ生成に日本語社名＋「株価」「決算」等の日本語クエリ分岐を足す。SerpApi は日本語検索と相性が良い。ただし TDnet の代替にはならない（検索は記事ベース）。
- **レポートの日本語化（やるなら改修必須）**: `REPORT_LANGUAGE` に ja が無いため、プロンプト/テンプレート側の改修が必要。安易な代替は `REPORT_LANGUAGE=en`。中国語のままでも数値中心なら読めるが、毎日の常用には摩擦が大きい。ここが継続利用の分かれ目。
- **全銘柄一括は避ける**: 200銘柄×（データ取得＋ニュース検索＋LLM呼び出し）は、Actions 無料枠（30分タイムアウト既定）と yfinance のレート制限の両面で現実的でない。日常はコア銘柄に絞り、全量は週1回ローカル実行などの階層化が要る。

### 詰まりそうな箇所

- **裸コード誤判定**: 保有リストを `.T` なしで入れると A股扱い等で静かに壊れる。投入前の形式チェックを必ず挟む。
- **yfinance の安定性**: 非公式経路のため、フィールド欠落（PER・出来高金額など）や一時的な取得失敗が前提。`data_quality=partial` は通常運転と考える。
- **GitHub Actions の公開性**: 公開リポジトリのフォークでは Actions のログも公開されうる。保有銘柄リストや分析対象がログに出るのが気になる場合は、**ローカル/Docker/このVMでの定時実行**を本命にし、Actions は試用に留める。
- **プロンプトの投資哲学**: 既定の分析観点（追高回避、MA配列、纏論など）は A股個人投資家向けに調整されており、藤本さんの売買メソッド（+5%利確、−10%/−20%追加 等）とは別物。レポートは「別人の視点の参考メモ」として読む前提にし、売買判断そのものの委託先にはしない。
- **コスト**: LLM 呼び出しは銘柄数に比例する。無料枠（Gemini）でもレート制限があり、`ANALYSIS_DELAY` 等の調整が要る場合がある。

## 9. 藤本さんに決めてほしい点

- **LLM の選択とキー**: どれを1つ用意するか。
  - 候補A: Gemini APIキー（無料枠あり、Google AI Studio で発行。既に Google AI Pro 契約あり）
  - 候補B: DeepSeek APIキー（低価格・従量）
  - 候補C: OpenAI APIキー、または Anthropic APIキー
  - 候補D: 集約系（Modelflare / AIHubMix）— プロジェクト推奨だが新規サービス登録が必要
  - 候補E: キー新規発行なしで、ローカルCLIバックエンド（Codex / Claude Code CLI、実験的）か Ollama ローカルモデルで試す
- **ニュース検索キー**: SerpApi（推奨）か Tavily のキーを用意するか、当面ニュースなし（データ＋ファンダのみ）で割り切るか。
- **最初の対象銘柄**: スモークテストは `7203.T`（トヨタ）単体でよいか。広げる場合の初期セット（例: 保有のうち時価総額上位10〜20銘柄に `.T` を付けたもの）も決めたい。
- **実行環境**: (1) このVM（Muse のワークスペース）で定時実行、(2) 藤本さんの手元マシンでローカル/Docker、(3) GitHub Actions（公開フォークになる点に注意）のどれを本命にするか。
- **レポート言語の扱い**: 英語（`REPORT_LANGUAGE=en`）で割り切るか、中国語のまま試すか、日本語化の改修までやるか。
- **通知先**: メール / Telegram / Discord / カスタムWebhook のどれに送るか（未定なら最初は通知なしで Web/ファイル確認のみでも可）。
