---
name: document_ja
description: >
  モジュラリティレビュードキュメントを Markdown と HTML の両形式で生成する。
  モジュラリティ分析の最終レビュー出力を書くときに使用する。
user-invocable: false
allowed-tools: Read, Write
---

# ドキュメント生成

モジュラリティレビューの最終ドキュメントを2つの形式で生成する: Markdown（`.md`）と HTML（`.html`）。両ファイルの内容は同一だ。

## ドキュメント構造

### ヘッダー

```markdown
# モジュラリティレビュー

**スコープ**: [何を分析したか]
**日付**: [日付]
```

### エグゼクティブサマリー

以下を網羅する短い段落（3〜5文）:

- プロジェクトが何をするか（コア機能）
- モジュラリティの総合ステータス（健全、要注意、重大な問題あり）
- レビューで最も重要な発見

### 結合概要テーブル（Coupling Overview Table）

すべての主要な統合をまとめる。テーブルのヘッダーは必ず coupling.dev にリンクすること:

```markdown
| 統合 | [強度](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/) | [距離](https://coupling.dev/posts/dimensions-of-coupling/distance/) | [変動性](https://coupling.dev/posts/dimensions-of-coupling/volatility/) | [均衡?](https://coupling.dev/posts/core-concepts/balance/) |
| ---- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------- |
| A -> B | ...                                                                           | ...                                                                 | ...                                                                     | ...                                                       |
```

### 問題（Issues）

特定された各不均衡について、以下のセクションを書く:

```markdown
## 問題: [短い説明的なタイトル]

**統合**: [コンポーネントA] -> [コンポーネントB]
**重大度**: [Critical（重大） / Significant（重要） / Minor（軽微）]

### 知識の漏洩（Knowledge Leakage）

共有されるべきでない知識、または明示的であるべきなのに暗黙的に共有されている知識。境界を越えて漏洩している具体的な実装の詳細、ビジネスルール、ドメインモデルの概念を特定する。

### 複雑性の影響（Complexity Impact）

この不均衡が変更の結果をどのように予測不可能にするか。開発者が一方のコンポーネントを変更するとき、もう一方にどんな予期しない影響が連鎖しうるか。これが認知容量（作業記憶の4±1単位）をどのように超えるか。

### 連鎖的変更（Cascading Changes）

一方のコンポーネントの変更が他方への変更を強制する具体的なシナリオ。どのようなビジネスや技術的変更が連鎖的修正を引き起こすか。現在のコンポーネント間の距離を考慮したとき、そうした連鎖変更はどれほどコストがかかるか。

### 改善提案（Recommended Improvement）

結合を再バランスするための具体的で実行可能な提案。モデルに基づいて提案を根拠づける:

- **強度を下げる**: 統合コントラクト、Anti-corruption layer、Facade、Published language を導入する
- **距離を下げる**: コンポーネントを同じモジュール/サービス/境界づけられたコンテキストに配置する
- **不均衡な結合を許容する**: 変動性が不均衡を許容できるほど低いことを示す

提案される変更のトレードオフも含める――どんなコストをもたらすか、なぜそのトレードオフが価値あるのか。
```

### フッター

すべてのレビュードキュメントの下部に、この正確なテキストを逐語的に含める――言い換え、拡張、注釈を加えないこと:

```markdown
---

_この分析は [Vlad Khononov](https://vladikk.com) による [Balanced Coupling](https://coupling.dev) モデルを使用して実施されました。_
```

### 重大度の基準

- **Critical（重大）**: 高強度 + 高距離 + 高変動性。複雑性の活発な源。この領域での変更は頻繁で、コストが高く、予測不可能。最優先で対処する。
- **Significant（重要）**: 中程度の変動性領域での不均衡な結合、または統合ポイントを隠す暗黙的な結合。システムが進化するにつれて問題を引き起こす。
- **Minor（軽微）**: 低変動性領域での不均衡な結合、または認知的負荷を増やすが連鎖変更を引き起こさない低凝集度。機会を見て対処する。

## coupling.dev へのリンク（必須）

ドキュメントが均衡結合の概念に言及するたびに、**coupling.dev へのハイパーリンクを追加しなければならない**。これは Markdown と HTML の両出力に適用される。どのリンクを使うか考える必要はない――この対照表を使うこと:

| 言及する概念 | リンク先 |
| ------------ | -------- |
| Balanced coupling（均衡結合）、バランスルール、バランス式、`STRENGTH XOR DISTANCE`、モジュラリティ vs 複雑性 | https://coupling.dev/posts/core-concepts/balance/ |
| Integration strength（統合強度）、共有される知識、結合強度のレベル | https://coupling.dev/posts/dimensions-of-coupling/integration-strength/ |
| Intrusive coupling（侵入的結合）、プライベートインターフェース、実装の詳細の漏洩 | https://coupling.dev/posts/dimensions-of-coupling/integration-strength/ |
| Functional coupling（機能的結合）、重複したビジネスロジック、共有された機能要件 | https://coupling.dev/posts/dimensions-of-coupling/integration-strength/ |
| Model coupling（モデル結合）、共有ドメインモデル、共有ビジネスモデル | https://coupling.dev/posts/dimensions-of-coupling/integration-strength/ |
| Contract coupling（コントラクト結合）、統合コントラクト、Facade、DTO、Published language | https://coupling.dev/posts/dimensions-of-coupling/integration-strength/ |
| Distance（距離）、変更のコスト、物理的/論理的分離 | https://coupling.dev/posts/dimensions-of-coupling/distance/ |
| Lifecycle coupling（ライフサイクル結合）、デプロイ結合、共同デプロイ | https://coupling.dev/posts/dimensions-of-coupling/distance/ |
| Socio-technical distance（ソシオテクニカル距離）、チーム境界、組織構造 | https://coupling.dev/posts/dimensions-of-coupling/distance/ |
| Runtime coupling（ランタイム結合）、同期/非同期統合 | https://coupling.dev/posts/dimensions-of-coupling/distance/ |
| Volatility（変動性）、変更の確率、変更率 | https://coupling.dev/posts/dimensions-of-coupling/volatility/ |
| Core subdomain（コアサブドメイン）、競争優位、高変動性ドメイン | https://coupling.dev/posts/dimensions-of-coupling/volatility/ |
| Supporting subdomain（サポーティングサブドメイン）、Generic subdomain（ジェネリックサブドメイン）、サブドメイン分類 | https://coupling.dev/posts/dimensions-of-coupling/volatility/ |
| Essential vs accidental volatility（本質的 vs 偶発的変動性） | https://coupling.dev/posts/dimensions-of-coupling/volatility/ |
| Complexity（複雑性）、Cynefin、予測不可能な結果 | https://coupling.dev/posts/core-concepts/complexity/ |
| Modularity（モジュラリティ）、モジュラ設計、明確な変更結果 | https://coupling.dev/posts/core-concepts/modularity/ |
| Coupling（結合）（一般概念）、なぜ結合が重要か | https://coupling.dev/posts/core-concepts/coupling/ |
| Tight coupling（密結合）、分散モノリスト | https://coupling.dev/posts/core-concepts/balance/ |
| Loose coupling（疎結合）、High cohesion（高凝集度）、Low cohesion（低凝集度） | https://coupling.dev/posts/core-concepts/balance/ |
| Connascence（コナセンス）、結合の度合い | https://coupling.dev/posts/related-topics/connascence/ |
| Module coupling（モジュール結合）、古典的な結合モデル | https://coupling.dev/posts/related-topics/module-coupling/ |
| Domain-driven design（DDD）、境界づけられたコンテキスト、集約 | https://coupling.dev/posts/related-topics/domain-driven-design/ |

**ルール:**

- ドキュメント内のすべての結合概念は少なくとも1回リンクしなければならない。迷ったらリンクする。
- 結合概要テーブルにおいて: ヘッダーは常にリンクする（上のテンプレートを参照）。テーブルのセル内の強度値（Contract, Model, Functional, Intrusive）もリンクしなければならない。
- 問題セクション: 各概念の最初の出現にリンクする。
- Markdown では: `[概念テキスト](url)`
- HTML では: `<a href="url">概念テキスト</a>`

## 出力

両ファイルを `docs/modularity-review/$(date +%Y-%m-%d)/` に書き込む:

1. `docs/modularity-review/$(date +%Y-%m-%d)/modularity-review.md` — Markdown バージョン
2. `docs/modularity-review/$(date +%Y-%m-%d)/modularity-review.html` — HTML バージョン

### HTML 生成

**HTML を生成する前に必ずテンプレートファイルを読むこと。** テンプレートは `${CLAUDE_SKILL_DIR}/assets/template.html` にある。このファイルを読んで、以下を置き換える:

- `{{TITLE}}` をレビュータイトル（例: "Modularity Review"）に
- `{{CONTENT}}` をレビュー本文の HTML に

独自の HTML 構造、スタイル、ボイラープレートを生成しないこと。テンプレートがすべてのスタイリング、テーマ切り替え（時刻に基づくライト/ダーク）、レイアウトを提供する。テンプレートをそのまま使い、2つのプレースホルダーだけを置き換えること。

**HTML には Markdown バージョンと同じ coupling.dev リンクが含まれていなければならない。** Markdown のすべての `[テキスト](url)` を HTML の `<a href="url">テキスト</a>` に変換する。変換中にリンクを削除しないこと。

テンプレートの以下の CSS クラスを使用する:

- `.meta` — スコープ/日付のメタデータブロック
- `.issue` — 各問題セクションを `<div class="issue">` で囲む
- `.issue-meta` — 問題内の統合/重大度行
- `.severity` + `.severity-critical` / `.severity-significant` / `.severity-minor` — 重大度バッジ
- `.footer` — 帰属フッター
