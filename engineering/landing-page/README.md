# AI鑑定士 — ランディングページ (プロトタイプ)

AI生成コンテンツ判定プロダクト「AI鑑定士 / AI Judge」の、動きのあるLP（1ファイル完結）。

## 見方

`index.html` をブラウザで開くだけ。ビルド不要。

```bash
open engineering/landing-page/index.html          # macOS
xdg-open engineering/landing-page/index.html      # Linux
```

## 使用技術

- **anime.js v3.2.1**（CDN読み込み）— 全アニメーションの駆動
- Vanilla JS + Canvas（背景のパーティクル網）
- Google Fonts（Noto Sans JP / Inter / JetBrains Mono）

## 仕込んだ「動き」

| 箇所 | 動き |
|------|------|
| ヒーロー見出し | 1文字ずつ stagger でフェードイン＋回転落下 |
| 背景 | Canvas のパーティクルが浮遊し、近接すると線で接続 |
| カーソル | 追従するグロー（マウス環境のみ） |
| ライブスキャン | スキャンライン走査 → 文単位でAI/人間をハイライト → ゲージが81%まで充填＆カウントアップ → 判定バッジがバウンド表示。`もう一度スキャン`で再生 |
| 統計値 | スクロール到達でカウントアップ |
| 特長カード | スクロールで stagger 出現＋ホバーで3Dチルト |
| CTAボタン | 発光のパルスループ |

## 配慮

- `prefers-reduced-motion` 対応（動きを止めて静的表示）
- anime.js / フォントがCDNから読めない場合もレイアウトと最終状態は崩れないフォールバックを実装
- レスポンシブ（〜860px / 〜720px でレイアウト調整）

## 補足

ブランド方針（`design/brand-direction.md`）のネイビー＋ブルーを基調に、
LPとしての訴求のためシアン/バイオレットのアクセントとモーションを加えています。
コピーは技術設計書（`engineering/architecture-design.md`）の機能に基づく仮テキストです。
