# AI生成物判定プロダクト - 技術設計書

作成日: 2026-05-05
最終更新: 2026-05-05
作成者: 開発部
ステータス: 第2版（技術検討深掘り版）

---

## 1. プロダクト概要

ユーザーが入力したコンテンツ（テキスト、画像、コード等）がAIによって生成されたものかどうかを判定し、
その確信度・判定根拠とともに結果を返すWebアプリケーション。

**プロダクト名（仮）**: DetectAI / AI鑑定士

### 1.1 解決する課題

- 教育機関: レポート・論文のAI生成チェック
- メディア・出版: 記事・原稿のオリジナリティ検証
- 企業: 採用選考・社内文書のAI利用状況の把握
- コンテンツプラットフォーム: AI生成コンテンツのラベリング義務対応（EU AI Act等）

### 1.2 差別化ポイント

- **日本語特化**: 日本語テキストの判定精度を最重視（既存ツールは英語中心）
- **マルチモーダル対応**: テキスト・画像・コードを統合的に判定
- **判定根拠の可視化**: 単なるスコアではなく「なぜAI生成と判断したか」を説明
- **API提供**: 外部サービスへの組み込みを前提とした設計

---

## 2. 技術アーキテクチャ設計方針

### 2.1 全体構成

```
[ユーザー / 外部サービス]
    |
    v
[CDN (CloudFront)] -- 静的アセット配信
    |
    v
[フロントエンド (Next.js SSR/SPA)]
    |  REST API / WebSocket
    v
[APIゲートウェイ / ロードバランサー (ALB + Nginx)]
    |
    +-- [認証・認可 (JWT + OAuth 2.0)]
    |
    v
[バックエンド API サーバー (FastAPI)]
    |
    +-- [判定エンジン (Detection Engine)]
    |       |
    |       +-- テキスト判定モジュール
    |       |     +-- 統計分析サブモジュール
    |       |     +-- ML分類サブモジュール
    |       |     +-- 透かし検出サブモジュール
    |       |
    |       +-- 画像判定モジュール
    |       |     +-- メタデータ解析サブモジュール
    |       |     +-- 周波数分析サブモジュール
    |       |     +-- CNN分類サブモジュール
    |       |
    |       +-- コード判定モジュール
    |       |     +-- AST分析サブモジュール
    |       |     +-- スタイル分析サブモジュール
    |       |
    |       +-- 統合スコアリングエンジン
    |
    +-- [非同期タスクキュー (Celery + Redis)]
    |       |
    |       +-- 重い判定処理のオフロード
    |       +-- バッチ処理（複数ファイル一括判定）
    |       +-- 定期的なモデル精度検証ジョブ
    |
    +-- [データベース (PostgreSQL 16)]
    |       |
    |       +-- ユーザー情報・認証情報
    |       +-- 判定履歴・判定詳細
    |       +-- フィードバックデータ（正誤報告）
    |       +-- APIキー管理
    |
    +-- [キャッシュ (Redis 7)]
    |       |
    |       +-- 判定結果キャッシュ（同一入力の再判定防止）
    |       +-- セッション管理
    |       +-- レートリミットカウンター
    |
    +-- [オブジェクトストレージ (S3)]
    |       |
    |       +-- アップロードファイル一時保存
    |       +-- MLモデルファイル（バージョン管理付き）
    |
    +-- [MLモデルサーバー (専用推論サーバー)]
            |
            +-- ONNX Runtime / TorchServe
            +-- GPU推論（重量モデル用）
            +-- モデルバージョン管理・ホットスワップ
```

### 2.2 設計原則

1. **Modular Monolith**: MVP段階ではモノリスだが、判定エンジンの各モジュールは明確なインターフェースで分離し、後からサービス分離可能にする。モジュール間の依存は抽象インターフェース（Protocol/ABC）経由とする
2. **非同期ファースト**: 判定処理は原則非同期。軽量な統計分析のみ同期で即時返却し、重いML推論は非同期で結果をWebSocket/SSEで通知
3. **プラグインアーキテクチャ**: 新しい判定手法を追加する際に `DetectorPlugin` インターフェースを実装するだけで組み込めるプラグイン設計
4. **API-First**: OpenAPI仕様を先に定義し、フロントエンド・バックエンド・外部連携を並行開発可能にする
5. **Observability**: 構造化ログ・分散トレーシング・メトリクスを最初から組み込む（後付けは困難）

---

## 3. 技術スタック提案

### 3.1 バックエンド

| 要素 | 技術 | 選定理由 |
|------|------|----------|
| 言語 | Python 3.12+ | ML/NLPエコシステムが最も充実。判定エンジンとの親和性が高い |
| Webフレームワーク | FastAPI 0.115+ | 非同期対応、型ヒントによる自動ドキュメント(OpenAPI)生成、高パフォーマンス、Pydantic v2統合 |
| ORM | SQLAlchemy 2.0 | Python標準のORM、非同期対応(asyncio)、型ヒントとの統合が改善 |
| マイグレーション | Alembic | SQLAlchemyとの完全互換、自動マイグレーション生成 |
| タスクキュー | Celery 5.x + Redis | 重い判定処理の非同期実行、リトライ・デッドレターキュー対応 |
| バリデーション | Pydantic v2 | 入力テキストのサイズ制限、ファイル形式チェック等 |
| テスト | pytest + httpx + factory_boy | FastAPIとの相性、テストデータ生成の効率化 |
| リンター・フォーマッター | ruff | lintとformatを統合、高速 |

### 3.2 フロントエンド

| 要素 | 技術 | 選定理由 |
|------|------|----------|
| フレームワーク | Next.js 15 (App Router) | SSR/SSG対応、React Server Components、SEO対応、Vercel簡易デプロイ |
| 言語 | TypeScript 5.x (strict mode) | 型安全性による品質向上 |
| UIライブラリ | shadcn/ui + Tailwind CSS 4 | カスタマイズ性が高く、軽量。コピペベースで依存を最小化 |
| 状態管理 | Zustand | シンプルで軽量。判定結果の一時保持に使用 |
| API通信 | TanStack Query v5 | キャッシュ管理、リトライ、楽観的更新、SSRサポート |
| リアルタイム通信 | SSE (Server-Sent Events) | 非同期判定結果のリアルタイム通知。WebSocketより実装がシンプル |
| チャート | Recharts | 判定結果の可視化（パープレキシティヒートマップ等） |
| テスト | Vitest + Playwright | ユニットテストとE2Eテスト |
| リンター | Biome | ESLint+Prettier代替、高速 |

### 3.3 AI/MLライブラリ

| 要素 | 技術 | 用途 | 備考 |
|------|------|------|------|
| Transformers | Hugging Face Transformers 4.x | テキスト判定用の事前学習済みモデル利用 | RoBERTa, DeBERTa-v3ベースの分類器 |
| PyTorch | PyTorch 2.x | ML推論基盤 | torch.compileによる推論高速化 |
| scikit-learn | scikit-learn 1.x | 統計的特徴量による判定、アンサンブル学習 | 軽量判定の第一段階で使用 |
| spaCy | spaCy 3.x + GiNZA | テキストの言語学的特徴量抽出 | GiNZAで日本語形態素解析対応 |
| Pillow / OpenCV | Pillow, opencv-python | 画像前処理、メタデータ解析 | EXIF/C2PA解析 |
| ONNX Runtime | onnxruntime-gpu | モデル推論の高速化（本番環境） | INT8量子化で推論コスト削減 |
| tokenizers | Hugging Face tokenizers | 高速トークナイズ | Rust実装で高速 |
| datasets | Hugging Face datasets | 学習・評価データの管理 | ストリーミング読み込み対応 |

### 3.4 インフラ

| 要素 | 技術 | 選定理由 |
|------|------|----------|
| コンテナ | Docker + Docker Compose | 開発環境の統一、デプロイの再現性 |
| IaC | Terraform | インフラのコード管理、再現性 |
| オーケストレーション | ECS Fargate (MVP) / EKS (将来) | MVP段階ではFargateで運用負荷を軽減、スケール時にEKS移行 |
| クラウド | AWS | GPU インスタンス (g5/g6) が利用可能、MLワークロードに適合 |
| GPU推論 | AWS Inferentia2 / g5 EC2 | Inferentia2でコスト最適化（対応モデルの場合） |
| CI/CD | GitHub Actions | コードリポジトリとの統合、セルフホストランナーでGPUテスト |
| 監視 | Prometheus + Grafana | メトリクス収集、ダッシュボード、アラート |
| ログ | Loki + Grafana | 構造化ログの集約・検索 |
| トレーシング | OpenTelemetry + Jaeger | 分散トレーシング、判定処理のボトルネック可視化 |
| CDN | CloudFront | 静的アセット配信 |
| シークレット管理 | AWS Secrets Manager | APIキー・DB接続情報等の安全な管理 |

---

## 4. AI生成物判定の技術的アプローチ

### 4.1 テキスト判定

#### A. 統計的特徴量分析（軽量・高速、第一段階の判定に使用）

| 手法 | 原理 | 実装方針 | 精度見込み |
|------|------|----------|------------|
| パープレキシティ分析 | AI生成テキストは人間の文章よりパープレキシティが低い傾向 | 小型言語モデル（GPT-2 small等）でトークンごとの対数確率を計算し、文単位のパープレキシティ分布を分析 | 単独では60-70%程度。他手法との組み合わせで有効 |
| バースティネス | 人間の文章は文の長さや複雑さにばらつきがある | 文ごとのパープレキシティの分散・標準偏差を特徴量とする | パープレキシティと組み合わせで5-10%精度向上 |
| エントロピー分析 | AI生成テキストは特定のトークンパターンに偏りやすい | ユニグラム・バイグラムのエントロピーを計測 | 補助的指標として有効 |
| 語彙多様性 | AI生成テキストは語彙の使い方が均一 | TTR (Type-Token Ratio)、hapax legomena比率、Yule's K等 | 長文で有効、短文では不安定 |
| 日本語固有特徴量 | 日本語特有の文章構造パターン | 文末表現の多様性、助詞の使用頻度分布、漢字/ひらがな/カタカナ比率の変動 | 日本語特化の差別化要素 |

**実装の具体的手順:**
```python
# 統計分析パイプラインの概念設計
class StatisticalAnalyzer:
    def analyze(self, text: str, lang: str) -> StatisticalFeatures:
        tokens = self.tokenize(text, lang)
        sentences = self.split_sentences(text, lang)
        return StatisticalFeatures(
            perplexity=self.calc_perplexity(tokens),
            burstiness=self.calc_burstiness(sentences),
            entropy=self.calc_entropy(tokens),
            ttr=self.calc_type_token_ratio(tokens),
            # 日本語の場合のみ
            jp_features=self.calc_japanese_features(text) if lang == "ja" else None,
        )
```

#### B. ML分類器（高精度、主力の判定手段）

| モデル | ベース | 用途 | 実装方針 |
|--------|--------|------|----------|
| テキスト分類器 (主力) | DeBERTa-v3-large | AI生成/人間作成の二値分類 | 最新AIモデル出力でファインチューニング。日本語は multilingual-DeBERTa または 日本語BERT(tohoku-nlp/bert-large-japanese-v2) を使用 |
| マルチモデル識別器 | RoBERTa-large | どのAIモデルが生成したかの多クラス分類 | GPT-4o, Claude 3.5/4, Gemini 2.x, Llama 3.x, Command R+等の出力を学習 |
| 軽量分類器 | DistilBERT | 高速スクリーニング用 | 精度は下がるが推論速度10倍。第一段階フィルタとして使用 |

**アンサンブル手法:**
- Stacking: 各分類器の出力を特徴量として、メタ分類器（ロジスティック回帰/LightGBM）で最終判定
- 信頼区間付きの出力: MCドロップアウトで推論を複数回行い、予測の不確実性を推定

**ファインチューニングのデータ収集戦略:**
1. 主要AIモデルのAPIを使い、多様なプロンプトでテキストを生成
2. 人間テキストはWikipedia、青空文庫、新聞記事コーパス等から収集
3. 日本語データは最低10万件を目標
4. パラフレーズ・部分編集されたテキストも学習データに含める（攻撃耐性向上）

#### C. 電子透かし (Watermark) 検出

| 透かし方式 | 対象 | 検出方法 | 実装難易度 |
|------------|------|----------|------------|
| C2PA メタデータ | OpenAI, Adobe等の出力 | メタデータパーサーで検出 | 低（ライブラリあり） |
| SynthID テキスト透かし | Google系AIの出力 | トークン選択の統計的偏りパターンを検出。公開仕様に基づく | 中（論文ベースの再実装） |
| Kirchenbauer方式透かし | 学術的な透かし手法 | green/red listのトークン分布の偏りを統計検定 | 中（論文実装あり） |
| 独自透かし検出 | 未知の透かし方式 | z-testベースの統計的異常検出 | 高（研究開発要素） |

**実装の優先度:** C2PA > SynthID > Kirchenbauer方式 > 独自検出

#### D. ゼロショット検出（モデル非依存の検出手法）

| 手法 | 原理 | 長所 | 短所 |
|------|------|------|------|
| DetectGPT | テキストの対数確率に微小摂動を加え、確率曲面の曲率から判定 | ファインチューニング不要 | 計算コストが高い（摂動を100回程度必要）、最新モデルへの対応が不確実 |
| Fast-DetectGPT | DetectGPTの高速化版。摂動にサンプリングモデルを使用 | DetectGPTの10倍高速 | それでもML分類器より遅い |
| Binoculars | 2つのLLMの対数確率比から判定 | 高精度でファインチューニング不要 | 2つのモデルのロードが必要でメモリ消費大 |
| GLTR | トークンごとのランク分布を可視化 | 直感的で説明性が高い | 精度は他手法に劣る |

**MVP段階での採用:** Fast-DetectGPTまたはBinocularsを補助手法として実装。主力はML分類器とする。

### 4.2 画像判定

#### A. メタデータ解析（最も確実だが回避も容易）
- **EXIF / C2PA メタデータ**: AI生成画像に付与されるメタデータの有無と内容を解析。`c2pa-python` ライブラリで実装
- **Content Credentials**: Adobe Content Authenticity Initiative (CAI) 準拠のメタデータ検出
- **ステガノグラフィ解析**: 不可視の透かし情報の検出

#### B. 周波数領域分析
- **フーリエ変換 (FFT)**: AI生成画像はGANやDiffusionモデル特有の周波数パターンを持つ。高周波成分の分布を分析。特にGAN生成画像は特徴的なスペクトルピークが現れる
- **DCT分析**: 離散コサイン変換による周波数特性の分析。JPEG圧縮との相互作用に注意
- **ウェーブレット分析**: 多解像度分析による局所的な周波数特性の抽出

#### C. CNN/ViT分類器
- **CLIP ベース分類器**: CLIP の画像エンコーダを特徴抽出に使用し、分類ヘッドを追加
- **GAN特徴検出**: GANが生成する画像特有のアーティファクト（チェッカーボードパターン等）を検出
- **Diffusion特徴検出**: Stable Diffusion, DALL-E 3, Midjourney v6, Flux等の出力特性を学習
- **UnivFD (Universal Fake Detector)**: 事前学習済みの汎用AI画像検出器をベースに利用

#### D. SynthID検出
- **Google SynthID**: Googleが提供する電子透かし技術への対応（DeepMindが開発）

### 4.3 コード判定

| 分析手法 | 詳細 | 特徴量例 |
|----------|------|----------|
| コーディングスタイル分析 | 変数命名の一貫性、コメントパターン | 命名規則の統一度、コメント密度、コメントの定型表現 |
| AST構造分析 | 抽象構文木の構造的特徴量 | ネストの深さ分布、制御構造の使用パターン、関数の平均行数 |
| パターンマッチング | AI特有のコード生成パターン | 定型的なエラーハンドリング、過剰なドキュメンテーション、典型的なインポート順序 |
| コード複雑度分析 | 循環的複雑度等のメトリクス分布 | McCabe複雑度の均一性（AI生成コードは複雑度が均一になりやすい） |

**注意:** コード判定はテキスト・画像に比べて精度が不安定。MVPスコープからは除外し、将来フェーズで対応する。

### 4.4 統合スコアリング

```
# 階層的スコアリングアーキテクチャ

第1層: 高速スクリーニング（<100ms）
  - 統計分析スコア（パープレキシティ、バースティネス、エントロピー）
  - 透かし検出スコア（C2PAメタデータの有無）
  → 即時結果を返却（暫定スコア）

第2層: ML判定（1-5秒）
  - 軽量分類器（DistilBERT）による判定
  → 暫定スコアを更新

第3層: 高精度判定（5-30秒、非同期）
  - 重量分類器（DeBERTa-v3-large）による判定
  - ゼロショット手法（Binoculars等）
  → 最終スコアとして確定、SSEで通知

最終スコア算出:
  score = sigmoid(
    w1 * statistical_score +
    w2 * lightweight_classifier_score +
    w3 * heavyweight_classifier_score +
    w4 * watermark_score +
    w5 * zero_shot_score +
    bias
  )

  重み w1-w5 と bias は検証データセットでの cross-validation で最適化
```

- 各手法のスコアを正規化して0.0〜1.0の範囲に統一
- **段階的な結果返却**: ユーザーは待たずに暫定結果を確認でき、精度の高い結果は後から更新される
- 閾値ごとに「AI生成の可能性: 高(>0.8)/中(0.4-0.8)/低(<0.4)」のラベルを付与
- **判定根拠の説明**: どの特徴量が判定に最も寄与したかをSHAP値で可視化
- **文単位の判定**: テキストを文単位で判定し、どの部分がAI生成の可能性が高いかをハイライト表示

---

## 5. MVP実装の機能と技術的実現可能性

### 5.1 MVPスコープ

| 機能 | 優先度 | 実現可能性 | 技術的詳細 |
|------|--------|------------|------------|
| テキスト判定 (日本語/英語) | P0: 必須 | **高** | 既存モデル(DeBERTa等) + 日本語データでのファインチューニングで実現可能。英語は既存の公開モデルを初期利用し、日本語は独自学習が必要 |
| 段階的結果返却 | P0: 必須 | **高** | 統計分析(即時) → 軽量ML(数秒) → 高精度ML(非同期) の3段階。SSEで結果をプッシュ |
| 判定根拠の可視化 | P0: 必須 | **高** | 文単位のAI生成確率ヒートマップ、パープレキシティの折れ線グラフ、寄与度の高い特徴量の表示 |
| ユーザー認証 | P0: 必須 | **高** | NextAuth.js v5 / OAuth 2.0 (Google, GitHub)。メールアドレス認証も対応 |
| 判定履歴の保存・管理 | P0: 必須 | **高** | PostgreSQLに保存。ユーザーごとの履歴一覧、検索、エクスポート(CSV) |
| REST API提供 | P0: 必須 | **高** | FastAPI の自動OpenAPIドキュメントを活用。APIキー認証。レート制限付き |
| C2PAメタデータ検出 | P1: 望ましい | **高** | `c2pa-python`ライブラリで実装可能。テキスト・画像両方に対応 |
| 画像判定 (基本) | P1: 望ましい | **中** | メタデータ解析は容易。CNN分類は精度を担保するために追加学習データが必要 |
| バッチ処理 | P1: 望ましい | **高** | Celery で実装可能。複数テキストの一括判定、結果のCSVダウンロード |
| フィードバック機能 | P1: 望ましい | **高** | 「正しい/間違い」の二択ボタン。収集データをモデル改善に活用（Active Learning） |
| コード判定 | P2: 将来 | **中** | ASTベースの分析は可能だが、精度の保証が困難。研究開発フェーズが必要 |
| リアルタイム判定 | P2: 将来 | **中** | SSE対応で入力中の逐次判定。デバウンス処理が必要 |
| 多言語対応 | P2: 将来 | **中** | 中国語・韓国語等。各言語のNLPリソース確保が課題 |

### 5.2 MVP技術要件

```
MVP構成（最小限で動作する構成）:

[Vercel]
  └── Next.js フロントエンド

[AWS]
  ├── ECS Fargate
  │   └── FastAPI バックエンドコンテナ
  │
  ├── RDS (PostgreSQL 16)
  │   └── ユーザー情報、判定履歴
  │
  ├── ElastiCache (Redis 7)
  │   └── キャッシュ、セッション、Celeryブローカー
  │
  ├── EC2 g5.xlarge (or Inferentia2)
  │   └── ML推論サーバー (ONNX Runtime)
  │
  └── S3
      └── MLモデルファイル、アップロード一時保存

推論サーバーの構成:
  - 軽量モデル (DistilBERT): CPU推論可能 → Fargateコンテナ内で実行
  - 重量モデル (DeBERTa-v3-large): GPU必須 → 専用GPU EC2インスタンス
  - 統計分析: CPU処理 → Fargateコンテナ内で実行
```

### 5.3 MVP開発工数見積もり

| フェーズ | 期間 (目安) | 内容 | 成果物 |
|----------|-------------|------|--------|
| Phase 0: PoC | 1週間 | テキスト判定エンジンの概念実証。公開モデルを使い、日本語テキストでの精度を検証 | PoC結果レポート、Go/No-Go判断 |
| Phase 1: 基盤構築 | 2週間 | プロジェクト雛形(monorepo構成)、認証、DB設計(マイグレーション)、CI/CD(GitHub Actions) | 動作するプロジェクト骨格、dev環境 |
| Phase 2: 判定エンジン | 3週間 | テキスト判定の実装（統計分析+ML分類器+透かし検出）、日本語データ収集・ファインチューニング | 判定APIエンドポイント、学習済みモデル |
| Phase 3: フロントエンド | 2週間 | UI実装（入力画面、結果表示画面、履歴画面）、結果可視化（ヒートマップ、チャート） | ユーザー向けWebアプリ |
| Phase 4: 統合・テスト | 1週間 | E2Eテスト、負荷テスト(k6)、セキュリティテスト(OWASP ZAP) | テスト報告書、品質保証 |
| Phase 5: デプロイ | 1週間 | 本番AWS環境構築(Terraform)、監視設定(Prometheus/Grafana)、ドメイン・SSL設定 | 本番稼働環境 |
| **合計** | **約10週間** | | |

### 5.4 データベース設計（主要テーブル）

```sql
-- ユーザー
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    hashed_password VARCHAR(255),
    oauth_provider VARCHAR(50),
    oauth_id VARCHAR(255),
    api_key VARCHAR(64) UNIQUE,
    plan VARCHAR(20) DEFAULT 'free',  -- free, pro, enterprise
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- 判定リクエスト
CREATE TABLE detection_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    content_type VARCHAR(20) NOT NULL,  -- text, image, code
    input_text TEXT,
    input_file_url VARCHAR(500),
    language VARCHAR(10),  -- ja, en, etc.
    status VARCHAR(20) DEFAULT 'pending',  -- pending, processing, completed, failed
    created_at TIMESTAMP DEFAULT NOW()
);

-- 判定結果
CREATE TABLE detection_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id UUID REFERENCES detection_requests(id),
    overall_score DECIMAL(5,4),  -- 0.0000 ~ 1.0000
    label VARCHAR(20),  -- high, medium, low
    statistical_score DECIMAL(5,4),
    ml_classifier_score DECIMAL(5,4),
    watermark_score DECIMAL(5,4),
    zero_shot_score DECIMAL(5,4),
    detail_json JSONB,  -- 文単位のスコア、特徴量詳細等
    model_version VARCHAR(50),
    processing_time_ms INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

-- フィードバック
CREATE TABLE feedbacks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    result_id UUID REFERENCES detection_results(id),
    user_id UUID REFERENCES users(id),
    is_correct BOOLEAN NOT NULL,
    actual_label VARCHAR(20),  -- ai_generated, human_written, mixed
    comment TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 6. 技術的リスクと課題

### 6.1 判定精度に関するリスク

| リスク | 影響度 | 発生確率 | 対策 |
|--------|--------|----------|------|
| AIモデルの進化で検出が困難になる | 高 | 高 | 継続的なモデル更新パイプライン構築、フィードバックループ |
| 日本語テキストの判定精度が低い | 高 | 中 | 日本語データセットでのファインチューニング、日本語固有の言語特徴量追加 |
| パラフレーズによる検出回避 | 中 | 高 | 複数の検出手法のアンサンブルで耐性向上 |
| 人間とAIの共同作成物の判定が曖昧 | 中 | 高 | 部分的AI利用率の推定、文単位の判定を実装 |

### 6.2 技術的課題

| 課題 | 詳細 | 対応方針 |
|------|------|----------|
| GPU推論コスト | MLモデルの推論にはGPUが必要で、コストが高い | ONNX Runtime での最適化、バッチ推論、キャッシュ活用でコスト削減 |
| レイテンシ | 複数の判定手法を実行すると応答時間が長くなる | 軽量手法を先行実行し即時結果を返し、重い手法は非同期で後から更新 |
| モデルサイズ | Transformerモデルは数GBに達する | 量子化 (INT8/INT4)、蒸留モデルの活用 |
| データセット | 最新AIモデルの出力データ収集が必要 | 主要AIサービスのAPIを利用してデータ収集パイプラインを構築 |
| 偽陽性/偽陰性 | 誤判定による信頼性低下 | 確信度とともに結果を返す設計、閾値の調整機能 |

### 6.3 法的・倫理的リスク

| リスク | 対応 |
|--------|------|
| プライバシー: ユーザーが入力したテキストの取り扱い | 判定後のデータ保持ポリシーを明確化、オプトイン方式でのデータ利用 |
| 著作権: 判定のためにAIモデルの出力を収集する行為 | 法務部と連携して適法性を確認 |
| 誤判定の責任: AI生成と誤判定された場合の影響 | 免責事項の明示、判定結果は「参考情報」として提供 |

### 6.4 運用リスク

| リスク | 対応 |
|--------|------|
| トラフィック急増 | オートスケーリング設定、レート制限の実装 |
| モデルのドリフト | 定期的な精度検証、A/Bテスト基盤の構築 |
| セキュリティ | 入力バリデーション徹底、ファイルアップロードのサニタイズ、WAF導入 |

---

## 7. 推奨ディレクトリ構成 (MVP)

```
ai-content-detector/
├── frontend/                  # Next.js アプリ
│   ├── src/
│   │   ├── app/              # App Router ページ
│   │   ├── components/       # UIコンポーネント
│   │   ├── lib/              # ユーティリティ
│   │   └── types/            # 型定義
│   ├── package.json
│   └── tsconfig.json
│
├── backend/                   # FastAPI アプリ
│   ├── app/
│   │   ├── api/              # APIルート
│   │   │   └── v1/
│   │   │       ├── detect.py
│   │   │       ├── auth.py
│   │   │       └── history.py
│   │   ├── core/             # 設定、セキュリティ
│   │   ├── models/           # SQLAlchemy モデル
│   │   ├── schemas/          # Pydantic スキーマ
│   │   ├── services/         # ビジネスロジック
│   │   └── detection/        # 判定エンジン
│   │       ├── text/
│   │       │   ├── perplexity.py
│   │       │   ├── classifier.py
│   │       │   ├── watermark.py
│   │       │   └── ensemble.py
│   │       ├── image/
│   │       │   ├── metadata.py
│   │       │   ├── frequency.py
│   │       │   └── classifier.py
│   │       └── scoring.py    # 統合スコアリング
│   ├── tests/
│   ├── requirements.txt
│   └── Dockerfile
│
├── ml/                        # MLモデル管理
│   ├── training/             # 学習スクリプト
│   ├── evaluation/           # 評価スクリプト
│   └── data/                 # データセット管理
│
├── docker-compose.yml
├── .github/
│   └── workflows/            # CI/CD
└── docs/                     # 技術ドキュメント
```

---

## 8. 次のアクション

1. **PM室と連携**: 開発スケジュール・マイルストーンの策定
2. **デザイン部と連携**: UI/UXデザインの検討（判定結果の可視化方法）
3. **法務部と連携**: データ取り扱いポリシー、利用規約の策定
4. **マーケティング部と連携**: ターゲットユーザーの明確化（個人向け/企業向け/教育機関向け）
5. **プロトタイプ着手**: テキスト判定エンジンの概念実証 (PoC) を最優先で開発開始
