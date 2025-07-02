# コーディングエージェント実装詳細調査レポート

## 概要

本レポートでは、2つの主要なコーディングエージェント実装である「gemini-cli」と「opencode」を詳細に調査し、それぞれの実装アプローチ、精度向上のための工夫、および技術的特徴を比較分析しました。

## 目次

1. [プロジェクト概要](#プロジェクト概要)
2. [アーキテクチャ比較](#アーキテクチャ比較)
3. [コア実装の詳細](#コア実装の詳細)
4. [ツール実装の分析](#ツール実装の分析)
5. [精度向上のための工夫](#精度向上のための工夫)
6. [実装から学ぶベストプラクティス](#実装から学ぶベストプラクティス)
7. [結論と推奨事項](#結論と推奨事項)

## プロジェクト概要

### gemini-cli
- **言語**: TypeScript/JavaScript
- **フレームワーク**: React (Ink) for TUI
- **特徴**: Google Gemini APIに特化した高度な最適化
- **パッケージ構造**: Monorepo (cli + core)
- **主要な強み**: 自動エラー修正、高度なプロンプトエンジニアリング

### opencode
- **言語**: Go
- **フレームワーク**: Bubble Tea for TUI
- **特徴**: マルチプロバイダー対応、LSP統合
- **アーキテクチャ**: クリーンアーキテクチャ
- **主要な強み**: 権限管理、プロバイダー汎用性

## アーキテクチャ比較

### gemini-cli アーキテクチャ

```
packages/
├── cli/          # UIとCLIインターフェース
│   ├── src/
│   │   ├── ui/   # Reactコンポーネント
│   │   └── config/
└── core/         # ビジネスロジック
    ├── src/
    │   ├── core/     # メインロジック
    │   ├── tools/    # ツール実装
    │   └── services/ # 外部サービス
```

**主要コンポーネント**:
- `GeminiChat`: チャット管理の中核
- `CoreToolScheduler`: ツール実行の調整
- `ContentGenerator`: コンテンツ生成の抽象化
- `Turn`: 単一会話ターンの管理

### opencode アーキテクチャ

```
internal/
├── app/         # アプリケーション層
├── llm/         # LLM関連実装
│   ├── agent/   # エージェントロジック
│   ├── provider/ # プロバイダー抽象化
│   ├── prompt/  # プロンプト管理
│   └── tools/   # ツール実装
├── tui/         # ターミナルUI
└── session/     # セッション管理
```

**主要コンポーネント**:
- `App`: 依存性注入とサービス管理
- `Agent`: ストリーミング対応のエージェント
- `Session`: 階層的セッション管理
- `TUI`: ElmアーキテクチャベースのUI

## コア実装の詳細

### gemini-cli のコア実装

#### 1. チャット機能（geminiChat.ts）
```typescript
class GeminiChat {
  // 高度な履歴管理
  private extractCuratedHistory(history: Content[]) {
    // 無効な応答を除外し、有効な履歴のみを保持
  }
  
  // リトライ機構
  private async sendWithRetry() {
    // エクスポネンシャルバックオフ + ジッター
    // Flashモデルへの自動フォールバック
  }
}
```

**特徴**:
- 洗練された履歴フィルタリング
- 思考（thought）コンテンツの特別処理
- 429エラー時の自動フォールバック

#### 2. ツールスケジューラー（coreToolScheduler.ts）
```typescript
class CoreToolScheduler {
  // ステートマシンによる状態管理
  private toolCallStates: Map<string, ToolCallState>
  
  // 並列実行制御
  async executeTools(toolCalls: ToolCall[]) {
    // 承認フローの管理
    // ライブ出力のサポート
  }
}
```

**特徴**:
- 詳細な状態遷移管理
- ユーザー承認フローの統合
- モディファイ機能（エディタ統合）

### opencode のコア実装

#### 1. エージェント実装（agent.go）
```go
type Agent struct {
    llm     provider.LLM
    tools   map[string]tools.Tool
    session *session.Session
}

func (a *Agent) GenerateResponse(ctx context.Context) {
    // ストリーミング応答処理
    // ツール実行の順次管理
    // コンテキスト伝播
}
```

**特徴**:
- イベント駆動アーキテクチャ
- 権限管理の統合
- LSP診断情報の活用

#### 2. セッション管理（session.go）
```go
type Session struct {
    ID           string
    ParentID     *string
    Title        string
    TokensUsed   int
    Cost         float64
}
```

**特徴**:
- 階層的セッション構造
- コスト追跡機能
- 要約による長期記憶

## ツール実装の分析

### gemini-cli のツール実装

#### Edit Tool（edit.ts）- 最も洗練された実装
```typescript
class EditTool {
  // エラー修正機能
  private async correctOldString(fileContent: string, oldString: string) {
    // LLMベースの文字列補正
    // キャッシュによる高速化
  }
  
  // エスケープ文字の自動修正
  private unescapeStringForGeminiBug(str: string) {
    // Gemini特有のバグ対策
  }
}
```

**精度向上の工夫**:
1. **editCorrector**: LLMを使用した自動修正
2. **LruCache**: 修正結果のキャッシング
3. **Diff生成**: 変更内容の可視化
4. **期待置換回数**: 正確性の検証

#### その他の主要ツール
- **Shell Tool**: プロセスグループ管理、リアルタイム出力
- **Grep Tool**: 3段階フォールバック（git grep → system grep → JS実装）
- **Memory Tool**: GEMINI.mdファイルへの構造化保存
- **Web Search**: Gemini APIのグラウンディング機能活用

### opencode のツール実装

#### Edit Tool（edit.go）- 信頼性重視の実装
```go
func (e *EditTool) Execute(args map[string]interface{}) {
    // 厳密な文字列一致検証
    // ファイル変更検出
    // LSP診断の自動取得
}
```

**信頼性向上の工夫**:
1. **一意性チェック**: 置換対象の重複検出
2. **タイムスタンプ検証**: 最終読み取り後の変更検出
3. **LSP統合**: 編集後の診断情報取得
4. **差分表示**: すべての変更を可視化

#### その他の主要ツール
- **Bash Tool**: 永続的シェルセッション、禁止コマンドリスト
- **Patch Tool**: 複数ファイルの一括編集
- **View Tool**: LSP診断情報付きファイル表示
- **Agent Tool**: 再帰的エージェント呼び出し

## 精度向上のための工夫

### 1. エラー修正機能の比較

| 項目 | gemini-cli | opencode |
|------|------------|----------|
| アプローチ | 積極的な自動修正 | 厳密な検証 |
| 実装方法 | LLMベース修正 + キャッシュ | 文字列完全一致 |
| エラー処理 | 自動リトライ + フォールバック | 明確なエラーメッセージ |
| 特徴 | Gemini特有のバグ対策 | LSP診断統合 |

### 2. プロンプトエンジニアリング

#### gemini-cli の戦略
```typescript
const systemPrompt = `
You are Claude Code, a small language model...
[詳細な役割定義]
[ワークフロー指示]
[具体例による学習]
`;
```

**特徴**:
- 包括的で教育的
- 環境依存の動的調整
- 豊富な例示

#### opencode の戦略
```go
func getSystemPrompt(provider string) string {
    switch provider {
    case "anthropic":
        return anthropicOptimizedPrompt
    case "openai":
        return openaiOptimizedPrompt
    }
}
```

**特徴**:
- プロバイダー別最適化
- 簡潔で直接的
- CLIでの表示を考慮

### 3. コンテキスト管理

#### gemini-cli
- **Curated History**: 有効な履歴のみを保持
- **Comprehensive History**: 完全な履歴を保存
- **思考プロセス**: 特別な処理で管理
- **圧縮機能**: トークン制限対策

#### opencode
- **セッションベース**: 階層的な管理
- **要約機能**: 長期記憶の保持
- **データベース**: 永続化サポート
- **タスクセッション**: 専用の実行環境

### 4. リトライとエラー処理

#### gemini-cli
```typescript
// 高度なリトライ機構
async function retryWithBackoff(fn, options) {
  const backoff = options.initialDelay * Math.pow(2, attempt);
  const jitter = backoff * 0.1 * Math.random();
  await sleep(backoff + jitter);
}
```

#### opencode
```go
// シンプルで効果的なリトライ
for i := 0; i < maxRetries; i++ {
    if err := tryExecute(); err == nil {
        return nil
    }
    time.Sleep(time.Second * time.Duration(i+1))
}
```

### 5. パフォーマンス最適化

| 最適化手法 | gemini-cli | opencode |
|-----------|------------|----------|
| キャッシング | LruCache（50件） | ファイル履歴 |
| 並列処理 | Promise.all | goroutine |
| イベント処理 | AsyncGenerator | channel |
| 状態管理 | React State | Bubble Tea Model |

## 実装から学ぶベストプラクティス

### 1. エラー処理とユーザビリティ
- **gemini-cli**: 自動修正による開発体験の向上
- **opencode**: 明確なエラーメッセージと権限管理による信頼性

### 2. 拡張性とメンテナビリティ
- **gemini-cli**: モノレポによる統一的な管理
- **opencode**: クリーンアーキテクチャによる疎結合設計

### 3. パフォーマンスとスケーラビリティ
- **gemini-cli**: 積極的なキャッシングと並列処理
- **opencode**: Goの並行性を活用した効率的な実装

### 4. セキュリティとプライバシー
- **gemini-cli**: ファイルパス検証、サンドボックス対応
- **opencode**: 詳細な権限管理システム

## 結論と推奨事項

### gemini-cli の強み
1. **高度な自動化**: エラー修正、リトライ、フォールバック
2. **Gemini最適化**: モデル特有の問題への対策
3. **開発者体験**: 詳細なロギング、デバッグ情報
4. **TypeScript**: 型安全性と豊富なエコシステム

### opencode の強み
1. **マルチプロバイダー**: 幅広いLLMサポート
2. **信頼性**: 厳密な検証と権限管理
3. **LSP統合**: リアルタイムコード診断
4. **Go言語**: 高性能と並行処理

### 実装時の推奨事項

#### 精度向上のために
1. **エラー修正**: LLMベースの修正とキャッシングの組み合わせ
2. **プロンプト**: モデル特性に応じた最適化
3. **コンテキスト**: 履歴の適切なフィルタリングと要約
4. **検証**: 厳密な文字列マッチングとLSP診断の活用

#### アーキテクチャ設計
1. **モジュラー設計**: ツールとコアロジックの分離
2. **イベント駆動**: 非同期処理とリアルタイム更新
3. **プラグイン可能**: 新しいツールやプロバイダーの追加容易性
4. **テスト可能性**: モックとインターフェースの活用

#### ユーザー体験
1. **権限管理**: 明示的な承認フローの実装
2. **進捗表示**: リアルタイムのフィードバック
3. **エラーメッセージ**: 明確で実行可能な指示
4. **カスタマイズ**: 設定ファイルとメモリ機能

### 最終的な推奨

コーディングエージェントを実装する際は、以下の要素のバランスを考慮することが重要です：

1. **自動化 vs 制御**: ユーザーの期待に応じた適切なレベル
2. **精度 vs 性能**: キャッシングと検証のトレードオフ
3. **汎用性 vs 最適化**: ターゲットモデルに応じた設計
4. **複雑性 vs 保守性**: 長期的なメンテナンスを考慮

両プロジェクトから学べる最も重要な教訓は、明確な設計哲学に基づいた一貫した実装が、優れたコーディングエージェントの基盤となることです。